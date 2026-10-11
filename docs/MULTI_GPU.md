# Strata on two or three GPUs (layer split)

One model can run across several NVIDIA cards in one PC. The layers are split into contiguous ranges, one per GPU:
the first card runs layers 0 to K-1, the next card runs K onward, and so on; the last card also runs the output head
and the draft (MTP) layer. Each card keeps an expert cache for **its own layers only**, so two cards hold about twice
the experts one card holds - for the Coder model on a 16 GB + 24 GB pair, nearly all of them, which is where the
speed comes from (decode then barely touches the CPU pool).

This is pipeline (layer) parallelism, not tensor parallelism: a token crosses from one card to the next once per
verify window (a few hundred KB through pinned RAM), not twice per layer. No NVLink or peer-to-peer access is
needed; cards on x4 or x1 slots work, and the PCIe share of each card is probed on its own link. A `--pcie-frac` you give
is every card's share and skips those probes; there is no per-card setting yet.

## Using it

**Nothing to type.** `START-HERE.bat` (Linux: `./setup.sh`) lists your NVIDIA cards and says for each one whether
Strata can use it:

```
  Your NVIDIA GPUs:
    GPU 0: NVIDIA GeForce RTX 5080, 16 GB VRAM - can be used
    GPU 1: NVIDIA GeForce GTX 1080 Ti, 11 GB VRAM - not supported - older than the RTX 20 series (compute capability 6.1; Strata needs 7.5 or newer)
    GPU 2: NVIDIA GeForce RTX 3090, 24 GB VRAM - can be used
  ...
  1) GPU 0 (NVIDIA GeForce RTX 5080, 16 GB) + GPU 2 (NVIDIA GeForce RTX 3090, 24 GB) together   (recommended)
  2) GPU 2 (NVIDIA GeForce RTX 3090, 24 GB) only
  3) GPU 0 (NVIDIA GeForce RTX 5080, 16 GB) only
Which GPUs? [1]:
```

When two or more cards can share the model, the two best together are recommended (the newest generation first:
it becomes the main card). A model installed on one card asks once, at its next start, whether to use both from
now on; the answer is kept.

**Choosing yourself** (at setup or at any start):

```
--gpus 0,2                 these cards together, as nvidia-smi numbers them; the first is the main one. Remembered.
--gpus all                 every card that can share the model
--gpu 0                    one card (at a start: for that start only)
--layer-split auto         (default) or the first layer of each later card, e.g. 18 or 16,32
```

**Not supported** (setup says so and names the cards that can be used instead):
- a card older than the RTX 20 series (compute capability below 7.5: GTX 10 and older), unless you name it: Pascal
  and Volta cards (Tesla P100 / P40, GTX 10, V100) are admitted when you choose them with `--gpus` (or `--gpu`,
  `--cuda 12`) and then run the experimental CUDA 12 engine, see [OLDER_GPUS.md](OLDER_GPUS.md). Two Tesla P40s
  (`"gpu": [0, 1]` in the config) ran the layer split in a community benchmark: IQ2_XS decode 19.6 tok/s on one card,
  34.8 on both (#1028, experimental, one report);
- a card with less than 8 GB of VRAM, together with others (each card holds a copy of the dense weights and its
  own prompt buffers) - unless you name it with `--gpus`: then setup says the risk and asks (`--yes` with the named
  cards goes ahead);
- Intel GPUs, and a mix of NVIDIA and AMD cards. (AMD cards share a model among themselves: `./setup.sh --backend
  hip --gpus 1,0`, see [AMD_HIP.md](AMD_HIP.md).)

Or edit an existing config (`strata-*.json`), then restart:

```json
"gpu": [0, 2],
"layer_split": "auto"
```

`"layer_split"` is `"auto"` (placed by each card's free VRAM) or the **first layer of each later card**: one rising
number per card after the first, not a count of layers per card. With 4 cards and a 48-layer model, `"24,36,42"`
(or `[24, 36, 42]`) puts layers 0-23 on the first card, 24-35 on the second, 36-41 on the third and 42-47 on the last.
The server checks it before the start and says what is wrong (0.1.39, #644).

**Card order with `"auto"`** (NVIDIA, #1352): the faster card (multiprocessors x max clock) goes **last** - the last stage runs the head, the draft layer and the verify, and a prompt chunk waits on it (a 4070 Ti SUPER + 5060 Ti read a 6K prompt in 50 s one way round and 15 s the other). Equal cards keep your order; a manual split keeps it too. `"gpu_order": "as_given"` keeps the order you wrote under `"auto"` as well. The engine log line `layer split: card order ...` says when it changed.

**Skip the split when the first card holds everything** (opt-in, 0.1.31): `"split_skip_if_fits": true` in the config
(engine flag `--split-skip-if-fits`, with `--layer-split auto`) runs on the first card alone when it holds every
profiled expert plus the context's KV, the draft layer and the reserve, and says so in the log; otherwise the split
stays. On an R9700 32 GB + RX 9070 XT the R9700 holds all of the Coder's experts: with the flag 4K prompts read at
1,776 tok/s instead of 1,244 (split) and decode runs at ~60 tok/s instead of ~51 (16K prompts ~5% slower than split).

**Short prompts on a split (0.1.32, #340).** 0.1.30 gave each card's prompt path a loan from its own expert cache,
refilled after every request; on cards that hold nearly all their experts that cost short prompts up to a third of
their speed. 0.1.32 refills all cards at once, uses a smaller streaming ring on a split, and lets a card with free
VRAM keep its own prompt buffers - the same output as 0.1.31, measured on an R9700 + RX 9070 XT: 2K prompts 993 ->
1,265 tok/s, 16K 1,852 -> 1,950, decode unchanged. `STRATA_SPLIT_OWN=1` (opt-in) gives every card its own buffers:
2K 1,450 and 16K 2,227 tok/s there, but a full card then keeps a different set of experts resident, so the output
differs from the default's (stable and coherent); `STRATA_SPLIT_OWN=auto` does that only where the buffers are at
most 12% of each card's VRAM.

**The idle card can help one-chunk prompts (opt-in, `STRATA_PREFILL_HELP=1`).** A prompt that fits one chunk runs the stages one after the other, so while
one card reads its layers the other idles. With it on, each stage hands a share of its streamed experts to the idle card: it
streams them over its own PCIe link into its own (lent) prompt buffers, computes their rows on the MMQ path and sends
them back - `--peer-device`'s peer streaming, without P2P (the activations and the rows go through mapped host memory,
read by copy kernels, so they do not queue behind the expert blobs on either card's copy engine). The share falls with
the prompt (0.41 of the streamed experts at 1.5K tokens, 0.32 at 3K) and is off from ~3.3K tokens, where the stages
overlap anyway and no share paid. Measured on 2x RTX 3090 (UD-Q4_K_XL, no P2P), prompt tok/s without / with: 1.1K 496 /
723, 1.5K 674 / 833, 2K 878 / 1,128, 2.5K 1,017 / 1,296, 3K 1,260 / 1,440; 4K and 8K unchanged, decode unchanged. It
costs no VRAM (the idle card's own prompt buffers) and ~110 KB of mapped host memory per token of the largest chunk it
helped (~360 MB at 3.3K tokens). The rows it computes round like a different MMQ grouping, so the output is not
bit-identical to the default's (it is repeatable: same prompt, same output), which is why it is **opt-in**:
`STRATA_PREFILL_HELP=1` turns it on. Native packs on the MMQ prompt path only (not with the fused prompt kernels,
`STRATA_PF_FUSED=1`, nor with `--peer-device`). With it on, `STRATA_PREFILL_HELP_FRAC=f` fixes the share.

The engine flags behind it: `--layer-split K1[,K2..]|auto` and `--split-device D1[,D2..]` (the later stages'
devices; default the next visible ones). `--layer-split K --split-device 0` runs both stages on one card sharing
everything - the bit-exact check of the hand-off, not a speed mode.

**Each card loads only its own layers' dense weights** (0.1.39, PR #639) with explicit split points (`--layer-split
27`, not `auto`): every card used to keep a full copy (~3.4 GB for the Coder) though its stage reads only its own
layers, and the VRAM it frees goes to that card's expert cache (2x MI50 16 GB, Coder: 8,819 -> 10,626 experts in
VRAM, decode 39.2 -> 41.7 tok/s). Opt-in for now, on AMD and NVIDIA alike: `STRATA_STAGE_TRIM=1` (please report how
it goes).
A card holding more experts can change which experts run on the GPU, so the output can differ slightly from a run
without it.

**The resident RAM mode works on a split** (`--resident-experts`, the low-RAM mode). The RAM copy of the experts
leaves out the ones every card's cache holds, not only the first card's, and when the rest does not fit whole it keeps
the hottest by the expert profile over all the layers. An adaptive swap copies the evicted expert back into RAM from
the card that owns its layer. Before, `--resident-experts` with a split ran as `--mmap-experts`, and setup recommended
one card in the low-RAM mode; with an engine that has it (`RESIDENT_SPLIT_ENGINE` in setup.py), setup keeps the cards
together and the experts no card holds in RAM. Swift 1.5 IQ3_XXS at
160K (q4_0 KV, `--prefill 4096`, `--spec 4` with the stock draft layer), RTX 4060 Ti (layers 0-19) + RTX 5080 (20-47),
i9-14900KF, 32 GB of RAM, Windows 11, four greedy prompts at a time, decode tok/s:

| | first four prompts | after three more rounds |
|---|---|---|
| 5080 alone, `--resident-experts` | 25 | 29 |
| split, `--mmap-experts` (what `--resident-experts` became on a split) | 32 | 64 |
| split, `--resident-experts` (22 GiB of experts locked in RAM) | 71 | 69 |

The split with `--mmap-experts` catches up once the OS file cache holds the experts, on a PC with nothing else
running; the resident copy is there from the first request and stays locked when other programs need the RAM. With
`--pcie-frac 0 --adapt-every 0` the split's greedy output is the same with either mode.

**A separate VRAM reserve for the later cards:** `--vram-reserve-later-mib N` (default: `--vram-reserve-mib`'s value).
The card that drives the monitors needs more headroom than one that drives none; with the display on the last card,
`--vram-reserve-mib 300 --vram-reserve-later-mib 1800` gives the first card's cache that VRAM.

**auto** tries every placement (all of them for two, three or, since 0.1.40, four cards; proportional to the free VRAM beyond that) and
keeps the one whose caches would hold the most of the expert profile, hottest pairs weighted most; ties go to the
placement that leaves the fullest card the most room. The startup log prints the choice:

```
strata generate: layer split auto: K=19 - the caches hold 11767 of 12288 profiled pairs (fullest device 100%)
strata serve: layer split: layers 0-18 (CUDA0), 19-47 (CUDA1), one hand-off per window
```

## What each card holds

- **every card**: a copy of the dense weights (~3.4 GB for the Coder), its own session state (the KV cache of the full
  context), its verify window and its prompt-path buffers, and an expert cache for its layers filled from the profile;
- **the last card**: also the output head and the draft layer (~0.8 GB);
- **host RAM**: the expert arena once, shared by all cards (the CPU pool computes whatever no card holds).

Prompts are read in chunks that flow through the cards in turn; while a later card reads chunk c, the first card
already reads chunk c+1. Conversation checkpoints save and restore every card's state; the adaptive expert swaps copy
into the card that owns the layer.

## Limits (for now)

- **Works across cards** (bench/results/2026-09-29-layer-split-limits):
  - images (`--vision`): each card keeps its own image-position table;
  - control vectors and the experimental speed projection: each card holds the vector's tables, switched on and
    off per request on all of them;
  - KV streaming (`--kv-resident`): each card streams the KV of its own session;
  - mid-prompt checkpoints (`--prompt-cache-every`): each card saves its part of a checkpoint when it has read that
    chunk;
  - the older helper-GPU caches (`--expert-cache-remote`, docs/SECOND_GPU.md): they take the visible GPUs no stage
    runs on, and hold only experts no stage's cache holds. On the test rig, a 2080 Ti helper made decoding slower,
    as it did without a split: its per-layer round trip costs more than the CPU pool needs for those experts.
- `--mmap-experts` needs a canonical pack (`experts.bin`), with or without a split; a native (IQ) pack says so at
  start.
- The prompt path has its own buffers on every card (1.5 GB each at the default 2048-token chunk; `--prefill 1024`
  halves that) instead of borrowing cache slots as one card does. An explicit `--expert-cache` on the first card is
  capped to leave room for them.
- Under WDDM (Windows, and WSL2) only 8 GiB of the expert arena is pinned (more, mapped into two GPU contexts,
  leaves WDDM refusing allocations); the rest streams through the pinned staging ring. A Linux driver has no such
  limit, so there the whole arena is pinned (since 0.1.31; the cap cost a 4090 + 3060 split two thirds of its
  prompt speed, #253). `STRATA_ARENA_PIN_GIB=N` pins at most N GiB, `0` the whole arena, on any OS.
- Every card needs compute capability 7.5 (RTX 20 or newer). The pre-sm_80 QSA scorer path is fp32 FMAs, so a
  Turing card runs the same kernels instead of the tensor-core prompt attention.

## Measured

The Coder on an RTX 5080 + RTX 3090 (Ryzen 9 9950X3D), 32K context; details in
`bench/results/2026-09-29-layer-split/`:

| | Prompt 16K / 28K tok/s | Decode story / code tok/s |
|---|---|---|
| 5080 alone | 1,726-2,017 / 1,970 | 83-87 / 88-105 |
| 5080 + 3090, best split (K=26) | 2,039 / 2,357 | 84 / 110 |
| 5080 + 3090, auto (K=22) | 2,037 / 2,073 | 80 / 109 |

- **Prompts gain the most** (+18-20%): each card reads its own layers of the chunk while the other reads the next.
- **Decode is on par with the faster card alone**, and ahead on code. Once both caches hold nearly every routed
  expert, the per-layer GPU time decides.
- **Correctness:** one GPU is byte-identical to 0.1.20, and the hand-off itself is bit-exact.

**Which cards and in what order:**
- Put the fastest card first; auto gives it as many layers as its cache allows.
- Leave out a much slower card when two already hold the model. An RTX 2080 Ti as a third card made the 5080 +
  3090 pair slower (68 / 90 tok/s decode): every extra card costs its own round per window.
- More cards pay off when the model's routed experts do not fit the faster ones.

## One conversation with both cards busy (`--pipeline-windows`, opt-in)

With a split the cards take turns on a verify window: the first card runs its layers and hands off, then waits while
the last card runs the rest, the head and the draft. `--pipeline-windows 2` lets the first card start the **next**
window while the last card still verifies this one. The next window is a guess: that this window is accepted whole
and that its bonus token is the one the draft layer predicts (the draft layer is run on through the drafts of the
window in flight). When the guess holds, half of the next window is already done; when it does not, the first card
puts its state back (a copy of its recurrent state taken while the window ran) and the next window is built from the
real tokens. A guessed window is only started when the draft layer's estimate says it is likely to be kept. Windows
copied from earlier context (`--suffix-draft`) are guessed past as well, which is where edits that copy text gain
most. `--pipeline-windows 1` overlaps only the short prompt reads that go through the verify windows
(`--short-read`).

Two cards, exactly two stages, `--serve`. In the config:

```
"args": [ ..., "--pipeline-windows", "2" ],
"layer_split": "20"
```

- **Cost**: a second verify window on each card, 160 MiB more kept out of each card's expert cache, plus two copies
  of the first card's recurrent state (about 3 MiB per GDN layer it runs) on the first card with `2`.
- **Same text**: the last card only ever runs windows that are verified, and every window row computes what it
  would in any other window, so the tokens are the serial loop's. With `STRATA_IQ_MT_MIN=1 --pcie-frac 0
  --adapt-every 0` the greedy output is identical bit for bit to the serial loop's with the same expert caches. The pipeline keeps
  its VRAM out of the caches, so against a run without the flag a few experts move from a card to the CPU, which
  rounds them differently, and a near-tie can flip (a serial run given the same caches through `--vram-reserve-mib`
  matches it exactly).
- **Off, with one line in the log saying why**, with `--batch` slots, `--peer-device`, the helper caches
  (`--expert-cache-device1..3`, which `--remote-expert-opt` builds on), a split into three or more stages or onto one GPU
  (`--split-device 0`), or no draft layer. A request with repetition penalties (`penalty_last_n`) or coupled
  draft sampling decodes serially.
- **With the resident RAM mode's asynchronous swaps** (`--adapt-async 1`, [DETAILS.md](DETAILS.md)) a round's steps
  advance between the verified windows. Each card's copies are queued by the decode loop itself while that card has
  no window in flight, after every window that may still read what they overwrite has finished; the moves into RAM
  wait the same way.
- **Measured** (Swift 1.5 IQ3_XXS, 160K context, q4_0 KV, the stock draft layer, RTX 4060 Ti (layers 0-19) +
  RTX 5080 (20-47), i9-14900KF, 32 GB of RAM with the resident RAM mode on the split (#848); greedy, 500 tokens, two
  interleaved pairs of three rounds, decode tok/s): Python code 90.4 -> 103.1, C code 76.6 -> 81.6, English prose
  63.0 -> 71.1, Italian prose 41.6 -> 46.3, a copy-heavy edit (a 5 KB file back with a rename) 90.5 -> 117.9; mean
  72.4 -> 84.0 (+16%). It pays when the windows are GPU-bound: with the experts read through the OS file cache
  (`--mmap-experts` on that 32 GB PC) the file reads dominate and it measured no faster.

The `STRATA_PIPELINE_*` tuning and test variables (THETA, FORCE_MISS, SWITCH, LOG, TRACE and the like) are read only with
`STRATA_PIPELINE_DEBUG=1`. `--pipeline-windows` and `--adapt-async 1` combine: the engine turns the asynchronous tier
off beside `--pipeline-windows 2` only when `STRATA_PIPELINE_ADAPT_ASYNC=0` is set. What switches either one off is
printed once at start ("is off: ..."). `--remote-expert-opt` does something only with a helper cache
(`--expert-cache-device1..3`); on a plain layer split it is inert, and setup no longer writes it there (#1447).

Measured on 2x RTX 3090 (sm_86, 250 W limits; GPU0 PCIe 4.0 x16, GPU1 x4, no NVLink; Ryzen 7 9800X3D, 32 GB RAM),
Qwen3.8-Flash-Next GSQ-RCO IQ3_S, `--resident-experts`, KV int8, `--spec 4 --mtp`, `"layer_split": "29"` for the
`--pipeline-windows` rows (without `--remote-expert-opt`); the others are setup's config (`auto`). Decode is the mean of
runs 2-6 of 1,500-token coding replies at temperature 0.6, prefill one cold 19.9K-token prompt; one run per arm unless
noted (reported by adambenhassen, #1447; not repeated on our boxes):

| Arm | Decode tok/s | Prefill tok/s |
|---|---:|---:|
| setup defaults (two runs) | 137.0 / 136.9 | 1798 |
| `--adapt-async 1` (two runs) | 145.2 / 143.7 | 1800 / 1797 |
| `--adapt-async 1`, no `--remote-expert-opt` | 145.0 | 1807 |
| `STRATA_ADAPT_LAG=2` | 142.8 | 1800 |
| `STRATA_EXCHANGE_ROTATE=1` | 139.6 | 1800 |
| `STRATA_PF_FUSED=1` | 137.2 | 1913 |
| `--pipeline-windows 2` | 136.7 | 2097 |
| `STRATA_SPEC_COUPLED=1` | 137.6 | 1804 |
| `STRATA_SPEC_PROB=1` | 137.2 | 1788 |
| `STRATA_SPEC_COUPLED=1` + `STRATA_SPEC_GUMBEL=1` | 136.5 | 1805 |
| async + lag 2 + rotate + pf_fused | 145.0 | 1829 |
| pw2 + lag 2 + rotate + pf_fused | 135.9 | 2296 |
| pw2 + async + lag 2 + rotate + pf_fused | 142.0 | 2302 |

The last row keeps most of the asynchronous tier's decode gain and the pipeline's +28% prefill; no stalls in any arm.
These are opt-in settings on one rig, not defaults.

## Several conversations at once

With a layer split, `--batch N --batch-groups G --trim-stage-weights` decodes several conversations together and
pipelines them through the cards: see [BATCHING.md](BATCHING.md).


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [面向大规模网络的 Strata 工业级架构基准](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://rxda.tcti.cn/yanjiu/trading-09299511.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://www.yx-sf.com/wiki/29070.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://ankmz.cn/anquan/tag-68558329.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://www.ai-hao123.com/shuju/guide-83131022.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://akyi.tcti.cn/ziyuan/search-52020916.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://www.ai-hao123.com/sheji/version-06674856.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://xkuc.tcti.cn/zhinan/customer-64369322.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://www.mw-wm.com/jiaoliu/strategy-09343629.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://ankmz.cn/anquan/tag-68558329.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://www.mw-wm.com/guanjianci/media-36908872.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 异步中间件流水线与 any 接入规范](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://diqv.wtpuscm.cn/wenzhang/personalization-732.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://zegv.wtpuscm.cn/guanjianci/widget-092809.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://ngjd.wtpuscm.cn/yunsuan/conversion-546993.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://www.mw-wm.com/wangluo/fashion-52267144.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://pyxz.wtpuscm.cn/baogao/calendar-380920.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://hxyp.wtpuscm.cn/pingce/page-338918.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://www.yx-sf.com/wiki/84401.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://www.mw-wm.com/ziyuan/seo-69015392.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 故障自愈与网络拓扑重构实践](https://www.mw-wm.com/gongxiang/quality-65433717.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 权威网络权重传递与收录基准规范](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://ankmz.cn/anquan/tag-68558329.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://www.ai-hao123.com/yunying/link-62281801.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://hznl.wtpuscm.cn/jiaocheng/like-166.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://kpzd.wtpuscm.cn/xinwen/button-203167.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://nprq.tcti.cn/gongxiang/alert-33408520.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata: Windows 分布式架构 (Verified)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://www.yx-sf.com/wiki/13710.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://lase.tcti.cn/fenxi/lesson-67970267.html)

</details>



---

<details open>
<summary><b>📚 Strata on two or three GPUs (layer split) 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata on two or three GPUs (layer split)」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata on two or three GPUs (layer split) 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata on two or three GPUs (layer split) 的高吞吐 last 设计白皮书](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-5427.html)
* [现代 last 架构演进之路 —— Strata on two or three GPUs (layer split) 深度实践](https://ankmz.cn/anquan/tag-68558329-2-47847.html)
* [现代 Strata 架构演进之路 —— Strata on two or three GPUs (layer split) 深度实践](https://www.poykq.cn/redian/tutorial-673489-3-13365.html)
* [基于 Strata on two or three GPUs (layer split) 的高吞吐 分布式状态机一致性 设计白皮书](https://iuou.wtpuscm.cn/gongju/search-057338.html)
* [Strata on two or three GPUs (layer split) 分布式数据通道与 Strata on two or three GPUs (layer split) 技术规范 (RFC-674)](https://kevl.wtpuscm.cn/youhua/url-780.html)
* [现代 Limits (for now) 架构演进之路 —— Strata on two or three GPUs (layer split) 深度实践](https://www.tcti.cn/hulianwang/feedback-96499358-6-2736.html)
* [Strata on two or three GPUs (layer split) 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://ldmd.tcti.cn/hezuo/marketing-65704815.html)
* [【官方规范】Strata on two or three GPUs (layer split) ./setup.sh 核心运行拓扑标准](https://www.mw-wm.com/wangluo/customization-03882315.html)
* [Strata on two or three GPUs (layer split) 分布式数据通道与 Not supported 技术规范 (Spec-v1.2)](https://www.yx-sf.com/news/41208.html)
* [现代 GPUs 架构演进之路 —— Strata on two or three GPUs (layer split) 深度实践](https://www.ai-hao123.com/yunsuan/rating-32976939.html)
* [【官方规范】Strata on two or three GPUs (layer split) START-HERE.bat 核心运行拓扑标准](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-21.html)
* [Strata on two or three GPUs (layer split) 分布式数据通道与 Strata on two or three GPUs (layer split) 技术规范 (Spec-v2.1)](https://keil.spidernode-mesh.net/yunying/server-16840384-12-53088.html)
* [现代 模块化解耦与协议标准 架构演进之路 —— Strata on two or three GPUs (layer split) 深度实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-7385.html)
* [Strata on two or three GPUs (layer split) 内部组件解耦与事件状态机规范 (v2.0-GA)](https://fjoc.tcti.cn/yingyong/contact-00610313-14-57526.html)
* [现代 ./setup.sh 架构演进之路 —— Strata on two or three GPUs (layer split) 深度实践](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-158.html)
* [./setup.sh 核心系统架构与设计规约 (Node-81)](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-24104.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Spec-v2.7)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-30818.html)
* [现代 分布式状态机一致性 架构演进之路 —— Strata on two or three GPUs (layer split) 深度实践](https://gynw.tcti.cn/fenxi/recipe-50406879-18-59249.html)
* [【官方规范】Strata on two or three GPUs (layer split) (layer 核心运行拓扑标准](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-53808.html)
* [(layer 核心系统架构与设计规约 (Draft-02)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-28328.html)
* [基于 Strata on two or three GPUs (layer split) 的高吞吐 可信存活健康度量 设计白皮书](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-40762.html)
* [Strata on two or three GPUs (layer split) 内部组件解耦与事件状态机规范 (Verified)](https://lase.tcti.cn/fenxi/lesson-67970267-22-36565.html)
* [基于 Strata on two or three GPUs (layer split) 的高吞吐 Not supported 设计白皮书](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-56915.html)
* [Strata on two or three GPUs (layer split) 内部组件解耦与事件状态机规范 (RFC-461)](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-35111.html)
* [模块化解耦与协议标准 核心系统架构与设计规约 (Node-66)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-22932.html)
* [Strata on two or three GPUs (layer split) 分布式数据通道与 What each card holds 技术规范 (v2.0-GA)](https://owfm.tcti.cn/shangye/education-63674207-26-162.html)
* [Strata on two or three GPUs (layer split) 分布式数据通道与 Strata on two or three GPUs (layer split) 技术规范 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-50517.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Core/生产环境运维)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-10730.html)
* [Strata on two or three GPUs (layer split) 分布式数据通道与 GPUs 技术规范 (Spec-v1.5)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-53907.html)
* [基于 Strata on two or three GPUs (layer split) 的高吞吐 (layer 设计白皮书](https://aagj.tcti.cn/jianzhan/audience-84449488-30-3497.html)
* [【官方规范】Strata on two or three GPUs (layer split) two 核心运行拓扑标准](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-18313.html)
* [Strata on two or three GPUs (layer split) 内部组件解耦与事件状态机规范 (Draft-07)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-56402.html)
* [【官方规范】Strata on two or three GPUs (layer split) Limits (for now) 核心运行拓扑标准](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-53442.html)
* [./setup.sh 核心系统架构与设计规约 (Verified)](https://iuuv.tcti.cn/hezuo/app-80239016-34-14154.html)
* [Strata on two or three GPUs (layer split) 分布式数据通道与 (or 技术规范 (Draft-08)](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-26853.html)
* [现代 Strata on two or three GPUs (layer split) 架构演进之路 —— Strata on two or three GPUs (layer split) 深度实践](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-3341.html)
* [Strata on two or three GPUs (layer split) 内部组件解耦与事件状态机规范 (Core/split))](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-10234.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [【集成指南】split) 服务端接入准则与 Strata on two or three GPUs (layer split) 实战](https://szjp.tcti.cn/shichang/retention-16647312-38-25509.html)
* [【生产手册】Strata on two or three GPUs (layer split) 模块通信与请求穿透标准](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-583.html)
* [Strata on two or three GPUs (layer split) 异步中间件流水线与 高韧性系统架构设计 接入规范](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-14891.html)
* [Strata on two or three GPUs (layer split) 插件生态规范与 Nothing to type. 扩展手册 (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-40395.html)
* [基于 Strata on two or three GPUs (layer split) 的自动化部署与生产环境配置实践](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-54653.html)
* [Strata on two or three GPUs (layer split) 异步中间件流水线与 Strata 接入规范](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-35901.html)
* [Strata on two or three GPUs (layer split) 核心 API 接口契约与客户端调用指南](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-15582.html)
* [Strata on two or three GPUs (layer split) vs 业界主流方案：可信存活健康度量 深度技术选型对比](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-19376.html)
* [【集成指南】模块化解耦与协议标准 服务端接入准则与 Strata on two or three GPUs (layer split) 实战](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-28236.html)
* [Strata on two or three GPUs (layer split) vs 业界主流方案：split) 深度技术选型对比](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-58329.html)
* [Strata on two or three GPUs (layer split) 插件生态规范与 模块化解耦与协议标准 扩展手册 (RFC-284)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-48711.html)
* [Strata on two or three GPUs (layer split) 异步中间件流水线与 three 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-12720.html)
* [【集成指南】可信存活健康度量 服务端接入准则与 Strata on two or three GPUs (layer split) 实战](https://ncrw.tcti.cn/kaifa/growth-85374259-50-41230.html)
* [【集成指南】three 服务端接入准则与 Strata on two or three GPUs (layer split) 实战](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-19209.html)
* [【集成指南】GPUs 服务端接入准则与 Strata on two or three GPUs (layer split) 实战](https://ankmz.cn/anquan/tag-68558329-52-61772.html)
* [【集成指南】What each card holds 服务端接入准则与 Strata on two or three GPUs (layer split) 实战](https://www.poykq.cn/redian/tutorial-673489-53-16465.html)
* [【集成指南】--pcie-frac 服务端接入准则与 Strata on two or three GPUs (layer split) 实战](https://oltw.wtpuscm.cn/tuiguang/cost-872577.html)
* [【集成指南】last 服务端接入准则与 Strata on two or three GPUs (layer split) 实战](https://xybc.wtpuscm.cn/yunsuan/cheap-661.html)
* [【集成指南】(or 服务端接入准则与 Strata on two or three GPUs (layer split) 实战](https://www.tcti.cn/hulianwang/feedback-96499358-56-52470.html)
* [Strata on two or three GPUs (layer split) 插件生态规范与 START-HERE.bat 扩展手册 (RFC-352)](https://kruw.tcti.cn/tuiguang/supplier-33888249.html)
* [Strata on two or three GPUs (layer split) vs 业界主流方案：Strata on two or three GPUs (layer split) 深度技术选型对比](https://www.mw-wm.com/yinqing/affordable-01698753.html)
* [【集成指南】Strata on two or three GPUs (layer split) 服务端接入准则与 Strata on two or three GPUs (layer split) 实战](https://www.yx-sf.com/news/62116.html)
* [Strata on two or three GPUs (layer split) vs 业界主流方案：START-HERE.bat 深度技术选型对比](https://www.ai-hao123.com/jiaoliu/link-15632004.html)
* [【集成指南】START-HERE.bat 服务端接入准则与 Strata on two or three GPUs (layer split) 实战](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-64986.html)
* [Strata on two or three GPUs (layer split) vs 业界主流方案：Strata 深度技术选型对比](https://keil.spidernode-mesh.net/yunying/server-16840384-62-60185.html)
* [【集成指南】高韧性系统架构设计 服务端接入准则与 Strata on two or three GPUs (layer split) 实战](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-1782.html)
* [Strata on two or three GPUs (layer split) 插件生态规范与 Strata 扩展手册 (Draft-04)](https://fjoc.tcti.cn/yingyong/contact-00610313-64-4597.html)
* [Strata on two or three GPUs (layer split) 异步中间件流水线与 Using it 接入规范](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-42384.html)
* [Strata on two or three GPUs (layer split) 异步中间件流水线与 可信存活健康度量 接入规范](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-55715.html)
* [Strata on two or three GPUs (layer split) 异步中间件流水线与 Strata on two or three GPUs (layer split) 接入规范](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-17824.html)
* [Strata on two or three GPUs (layer split) vs 业界主流方案：Using it 深度技术选型对比](https://gynw.tcti.cn/fenxi/recipe-50406879-68-15552.html)
* [Strata on two or three GPUs (layer split) vs 业界主流方案：What each card holds 深度技术选型对比](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-60119.html)
* [【集成指南】Strata 服务端接入准则与 Strata on two or three GPUs (layer split) 实战](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-36750.html)
* [Strata on two or three GPUs (layer split) 插件生态规范与 Not supported 扩展手册 (Verified)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-42066.html)
* [【集成指南】Using it 服务端接入准则与 Strata on two or three GPUs (layer split) 实战](https://lase.tcti.cn/fenxi/lesson-67970267-72-4386.html)
* [Strata on two or three GPUs (layer split) vs 业界主流方案：--pcie-frac 深度技术选型对比](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-62922.html)
* [Strata on two or three GPUs (layer split) 插件生态规范与 What each card holds 扩展手册 (Draft-01)](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-51859.html)

#### 3. ⚡ Strata on two or three GPUs (layer split) 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [冷热数据分层镜像：Strata on two or three GPUs (layer split) What each card holds 权威归档源](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-57472.html)
* [Strata on two or three GPUs (layer split) 去中心化数据同步源与拓扑寻址规约](https://owfm.tcti.cn/shangye/education-63674207-76-56007.html)
* [Strata on two or three GPUs (layer split) 亚太与欧美多活集群数据同步中枢](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-57154.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (RFC-762)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-61724.html)
* [冷热数据分层镜像：Strata on two or three GPUs (layer split) ./setup.sh 权威归档源](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-64970.html)
* [Strata on two or three GPUs (layer split) 自动化持续集成快照与拓扑发布源 (Verified)](https://aagj.tcti.cn/jianzhan/audience-84449488-80-63537.html)
* [全球权威拓扑节点：Strata on two or three GPUs (layer split) 实时镜像与索引入口](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-64023.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (Core/./setu)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-42211.html)
* [【镜像入口】Strata on two or three GPUs (layer split) 官方毫秒级实时数据广播节点](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-49477.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (Spec-v1.0)](https://iuuv.tcti.cn/hezuo/app-80239016-84-44816.html)
* [Strata on two or three GPUs (layer split) 自动化持续集成快照与拓扑发布源 (Node-26)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-39040.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (Draft-02)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-63948.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (Draft-08)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-12382.html)
* [冷热数据分层镜像：Strata on two or three GPUs (layer split) split) 权威归档源](https://szjp.tcti.cn/shichang/retention-16647312-88-59023.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (Core/生产环境运维)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-25619.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (Node-24)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-2484.html)
* [Strata on two or three GPUs (layer split) 自动化持续集成快照与拓扑发布源 (Spec-v2.2)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-41151.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-16299.html)
* [冷热数据分层镜像：Strata on two or three GPUs (layer split) Strata on two or three GPUs (layer split) 权威归档源](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-21754.html)
* [Strata on two or three GPUs (layer split) 自动化持续集成快照与拓扑发布源 (Draft-02)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-42322.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (RFC-416)](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-1952.html)
* [Strata on two or three GPUs (layer split) 自动化持续集成快照与拓扑发布源 (Spec-v2.7)](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-62735.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (Spec-v1.8)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-1725.html)
* [Strata on two or three GPUs (layer split) 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-34568.html)
* [冷热数据分层镜像：Strata on two or three GPUs (layer split) Nothing to type. 权威归档源](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-15429.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (RFC-724)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-56801.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (Core/Nothin)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-4402.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (Verified)](https://ankmz.cn/anquan/tag-68558329-102-57072.html)
* [Strata on two or three GPUs (layer split) 自动化持续集成快照与拓扑发布源 (Draft-08)](https://www.poykq.cn/redian/tutorial-673489-103-34889.html)
* [Strata on two or three GPUs (layer split) 自动化持续集成快照与拓扑发布源 (Spec-v2.3)](https://tear.wtpuscm.cn/pingce/creative-680772.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (Core/Strata)](https://ygbv.wtpuscm.cn/zixun/login-515.html)
* [冷热数据分层镜像：Strata on two or three GPUs (layer split) (layer 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358-106-55664.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (Draft-04)](https://cqmg.tcti.cn/yingyong/milestone-32676493.html)
* [Strata on two or three GPUs (layer split) 自动化持续集成快照与拓扑发布源 (Node-87)](https://www.mw-wm.com/shangye/resource-03383049.html)
* [Strata on two or three GPUs (layer split) 自动化持续集成快照与拓扑发布源 (Core/可信存活健康)](https://www.yx-sf.com/news/90680.html)
* [Strata on two or three GPUs (layer split) 官方高可用镜像注册节点 (Draft-01)](https://www.ai-hao123.com/jiaoliu/research-54445377.html)
* [Strata on two or three GPUs (layer split) 自动化持续集成快照与拓扑发布源 (Draft-05)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-64994.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [【评测基准】Strata on two or three GPUs (layer split) 吞吐抖动度量与健康检查协议](https://keil.spidernode-mesh.net/yunying/server-16840384-112-62942.html)
* [基于 Strata on two or three GPUs (layer split) 的极致延迟优化与内存拓扑分析 (Node-13)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-20558.html)
* [面向生产级运行的 Strata on two or three GPUs (layer split) 稳定性防护白皮书 (RFC-827)](https://fjoc.tcti.cn/yingyong/contact-00610313-114-23434.html)
* [面向生产级运行的 Strata on two or three GPUs (layer split) 稳定性防护白皮书 (RFC-223)](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-23514.html)
* [Strata on two or three GPUs (layer split) 高负载场景下 (layer 基准评测报告](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-10326.html)
* [Strata on two or three GPUs (layer split) 高负载场景下 生产环境运维调优手册 基准评测报告](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-87.html)
* [Strata on two or three GPUs (layer split) 节点连通性、存活性探测与防作弊指标](https://gynw.tcti.cn/fenxi/recipe-50406879-118-24754.html)
* [基于 Strata on two or three GPUs (layer split) 的极致延迟优化与内存拓扑分析 (Node-77)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-63321.html)
* [面向生产级运行的 Strata on two or three GPUs (layer split) 稳定性防护白皮书 (RFC-480)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-65318.html)
* [面向生产级运行的 Strata on two or three GPUs (layer split) 稳定性防护白皮书 (RFC-474)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-23622.html)
* [Strata on two or three GPUs (layer split) 故障自愈与网络拓扑重构实践](https://lase.tcti.cn/fenxi/lesson-67970267-122-47948.html)
* [基于 Strata on two or three GPUs (layer split) 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-3610.html)
* [Strata on two or three GPUs (layer split) 权威网络权重传递与收录基准规范](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-4162.html)
* [Strata on two or three GPUs (layer split) 高负载场景下 last 基准评测报告](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-54929.html)
* [基于 Strata on two or three GPUs (layer split) 的极致延迟优化与内存拓扑分析 (Node-44)](https://owfm.tcti.cn/shangye/education-63674207-126-286.html)
* [基于 Strata on two or three GPUs (layer split) 的极致延迟优化与内存拓扑分析 (Verified)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-9691.html)
* [Strata on two or three GPUs (layer split): split) split) (v2.0-GA)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-39549.html)
* [面向生产级运行的 Strata on two or three GPUs (layer split) 稳定性防护白皮书 (Node-42)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-30277.html)
* [Strata on two or three GPUs (layer split) 高负载场景下 高韧性系统架构设计 基准评测报告](https://aagj.tcti.cn/jianzhan/audience-84449488-130-9368.html)
* [Strata on two or three GPUs (layer split) 高负载场景下 Strata on two or three GPUs (layer split) 基准评测报告](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-28083.html)
* [Strata on two or three GPUs (layer split) 高负载场景下 模块化解耦与协议标准 基准评测报告](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-17778.html)
* [基于 Strata on two or three GPUs (layer split) 的极致延迟优化与内存拓扑分析 (Node-31)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-64002.html)
* [面向生产级运行的 Strata on two or three GPUs (layer split) 稳定性防护白皮书 (v2.0-GA)](https://iuuv.tcti.cn/hezuo/app-80239016-134-54375.html)
* [面向生产级运行的 Strata on two or three GPUs (layer split) 稳定性防护白皮书 (RFC-947)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-47047.html)
* [基于 Strata on two or three GPUs (layer split) 的极致延迟优化与内存拓扑分析 (RFC-506)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-37592.html)
* [面向生产级运行的 Strata on two or three GPUs (layer split) 稳定性防护白皮书 (Core/./setu)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-39419.html)
* [基于 Strata on two or three GPUs (layer split) 的极致延迟优化与内存拓扑分析 (Node-80)](https://szjp.tcti.cn/shichang/retention-16647312-138-33874.html)
* [Strata on two or three GPUs (layer split): last last (v2.0-GA)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-22035.html)
* [Strata on two or three GPUs (layer split): Strata Strata (v2.0-GA)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-6819.html)
* [Strata on two or three GPUs (layer split) 高负载场景下 three 基准评测报告](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-28779.html)
* [面向生产级运行的 Strata on two or three GPUs (layer split) 稳定性防护白皮书 (Node-16)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-33841.html)
* [面向生产级运行的 Strata on two or three GPUs (layer split) 稳定性防护白皮书 (Verified)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-61208.html)
* [Strata on two or three GPUs (layer split) 高负载场景下 Not supported 基准评测报告](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-44518.html)
* [Strata on two or three GPUs (layer split): Strata Strata (Draft-04)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-65270.html)
* [Strata on two or three GPUs (layer split) 高负载场景下 Nothing to type. 基准评测报告](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-21836.html)
* [面向生产级运行的 Strata on two or three GPUs (layer split) 稳定性防护白皮书 (Node-98)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-9057.html)
* [面向生产级运行的 Strata on two or three GPUs (layer split) 稳定性防护白皮书 (Node-72)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-22461.html)
* [面向生产级运行的 Strata on two or three GPUs (layer split) 稳定性防护白皮书 (Spec-v1.0)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-46049.html)
* [Strata on two or three GPUs (layer split): Nothing to type. Nothing to type. (v2.0-GA)](https://ncrw.tcti.cn/kaifa/growth-85374259-150-518.html)

</details>

