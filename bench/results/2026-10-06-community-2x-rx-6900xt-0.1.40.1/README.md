# Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1

Measured on 2026-10-06 on a Linux desktop. This tests the original Qwen3.8-Flash-Next (GSQ-RCO IQ3_S) with a
131,072-token context in the three ways two cards can be used - one card alone, a layer split across both, and one card
with the other as an expert helper (`--remote-expert-opt`) - on engine 0.1.40.1 as released, on 0.1.40.1 with the two
switches it ships for this card family (`STRATA_HIP_PROMPT_F16=1 STRATA_SH_STREAM=1`), and on 0.1.40.1 with the three
open gfx103x pull requests #1150, #1151 and #1167 applied on top of those switches. The adaptive expert swaps were
**on** (the default) in every configuration; the 0.1.39 report from this machine had them off because of #884.

On stock 0.1.40.1, one card reads a prompt at about **460 tok/s** and decodes at 47-48 tok/s; with the two switches the
same card reads at **811 / 973 / 913 tok/s** (4K / 32K / 128K) and decodes at 49-51; with the pull requests it reads at
**969 / 1,152 / 1,126**. The layer split with the pull requests reads a 128,000-token prompt at **2,003 tok/s** (64 s
to the first token, against 156 s on the stock split and 279 s on one stock card) and decodes at 70-75 tok/s; the helper
mode decodes at 64-75 tok/s in every arm. All 54 recall checks (six needles per configuration) were found. The requests
are synthetic code-explanation requests with greedy decoding and a 256-token output cap; they do not establish answer
quality or performance on other workloads.

**One thing to know before reading the two-card numbers:** with the adaptive swaps on and the GPU's GFX power-gating
(GFXOFF) at the kernel's default, the stock and the switches-only layer split both stalled on their first 128,000-token
request (the engine stopped making progress while reading the prompt; the server restarted it). The two-card
configurations below were therefore measured with GFXOFF disabled through the amdgpu debugfs knob, which on this machine
removes the stall and does not change speed (the stalled runs' 4K and 32K rows are kept in the `*-gfxoffon` folders and
match the GFXOFF-off rows to the token). The one-card configurations ran at the default and did not stall. Details in
"Stalls" below; the issue is #884.

## Hardware and software

- **GPUs:** 2x AMD Radeon RX 6900 XT 16 GB (Navi 21, gfx1030, wave32, no matrix cores), reference cards, 245 W power
  cap, no clock changes, no display attached. Both on CPU PCIe 4.0 lanes of an X570 board, which gives each card x8 (the
  engine's own probe: 13.6-14.1 GB/s host to device per card). No P2P between the cards is used by any mode below.
- **CPU and RAM:** AMD Ryzen 5 5600X (6 cores / 12 threads, AVX2, no AVX-512; the engine ran 5 expert-pool workers plus
  its host thread). 128 GB DDR4-3200 (4 DIMMs). 8 GiB swap file.
- **Storage:** the model files and the pack on one NVMe SSD (Samsung 970 EVO 500 GB), also the system disk. The PLE
  table is read from the GGUF shard on it (`--ple-io direct`, the default).
- **OS and runtime:** Ubuntu 26.04, kernel 7.0.0-38-generic with its own amdgpu driver (no DKMS), ROCm 10.0.0 (HIP
  7.15, rocBLAS 5.6.0.8d1ae90e). hipBLASLt ships no gfx1030 kernels, so the plain hipBLAS/rocBLAS path runs.
- **Source, stock:** [Niko1221/Strata](https://www.mw-wm.com/anli/demographic-53344234.html) `main` at `82f46a8` (engine 0.1.40.1).
- **Source, "PRs":** `82f46a8` with #1150 (PR #540's attention kernel on gfx103x, bit-exact), #1151 (the QSA block
  scores on rocBLAS SGEMM, opt-in `STRATA_SELECT_SGEMM=1`) and #1167 (a rocBLAS solution table for the FP16 prompt
  GEMMs, `STRATA_ROCBLAS_TUNING`) merged locally, plus #1148 (a test skip) and #1149 (tests and an off-by-default
  range counter for the FP16 route); the tree is `341922e` in each `BUILD.txt`. The engine source of #1148 and #1149
  does not change any number here.
- **Build:** both trees built by hand with cmake and ROCm's clang: `-DCMAKE_BUILD_TYPE=Release -DSTRATA_ENABLE_HIP=ON
  -DCMAKE_HIP_ARCHITECTURES=gfx1030 -DSTRATA_PREFILL_MMQ=ON` (the options setup uses for AMD), llama.cpp at the pinned
  commit. Each configuration's `BUILD.txt` has the tree, the commit, the binary's SHA-256, the environment and the
  GFXOFF knob's reading at start.
- **Other load:** nothing else used the GPUs during the runs (telemetry, below). The machine's other services (an idle
  llama-swap router, a home-automation stack, the LACT fan-curve daemon) kept running on the CPU.

## Model and configuration

**Model:** [`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`](https://api.pvucttcv.cn/yinyue/innovation-37390655.html),
IQ3_S - the same two files setup downloads for its IQ3_S choice, packed with the repository's `tools/iq_pack.py` (no
`--compat-bf16`); the SHA-256 hashes match the repository's `.sha256` files (they are listed in the 2026-10-05 report
from this machine). The MTP draft layer was prepared with the repository's tools. The expert profile is the repository's
own `data/expert-profile.bin`. No vision encoder, no calibration, no speed projection.

**Configurations** (each one's `config.json` is in its folder; `/m` is the model directory, `/work/stock` and
`/work/prs` the two trees). Common to all: context 131,072, `--kv int8 --kv-resident 32768` (32,768 K/V cells per
attention layer in VRAM, the rest streamed from pinned RAM), `--expert-cache auto`, `--prefill auto` (8,192-token
chunks), MTP with `--spec 4 --spec-min-p 0.5`, suffix drafting on, and the adaptive expert swaps at their default
(`--adapt-every 4`, not passed).

| name | cards | engine | environment |
| --- | --- | --- | --- |
| `single-stock` | one (PCI 0f) | stock | none |
| `single-sw` | one | stock | `STRATA_HIP_PROMPT_F16=1 STRATA_SH_STREAM=1` |
| `single-prs` | one | PRs | the two above + `STRATA_ROCBLAS_TUNING=/work/prs/tools/hip STRATA_SELECT_SGEMM=1 STRATA_SELECT_SGEMM_VERBOSE=1 STRATA_IO_THREADS=128` |
| `split-stock` | both, `layer_split: auto` | stock | none |
| `split-sw` | both | stock | the two switches |
| `split-prs` | both | PRs | as `single-prs` |
| `helper-stock` | one + the other as helper (`--expert-cache-device1 auto --remote-expert-opt`), the probed PCIe share 0.39 | stock | none |
| `helper-sw` | the same | stock | the two switches |
| `helper-prs` | the same | PRs | as `single-prs` |
| `split-stock-gfxoffon`, `split-sw-gfxoffon` | both | stock | as the two split arms above, GFXOFF at the kernel default; stalled at 128K (below) |

The engine's own log confirms each switch: `STRATA_HIP_PROMPT_F16=1: the prompt's 16-bit GEMMs run FP16 in and out`
(once per card), `prefill gemm: rocBLAS tuning enabled (112 rows, 76 with a solution, gfx1030, rocBLAS 5.6.0.8d1ae90e)`
and the `qsa select: SGEMM reach<=... ` measurement lines (`engine.log` in each folder).

**GFXOFF.** `/sys/kernel/debug/dri/{0,1}/amdgpu_gfxoff` (4 bytes, little-endian) was 1 (the default) for the three
one-card arms and for the two `*-gfxoffon` arms, and 0 for the six two-card arms, written as root before `split-stock`
and written back to 1 after `helper-prs`; the knob's reading at each start is in `BUILD.txt`. The knob is a reference
count in amdgpu (`amdgpu_gfx_off_ctrl`): a 0 is undone only by a later 1.

## Method

[benchmark.py](benchmark.py) is the script of the
[RTX 5090 report](../2026-09-30-community-rtx-5090/README.md), unchanged. It builds deterministic synthetic Python
filler, puts a different nonce near the start of each request, and counts the rendered chat prompt with Strata's
tokenizer. [monitor.py](monitor.py) samples RAM, swap and both cards' VRAM, GTT, busy percentage, power and temperature
from sysfs once per second.

Every configuration started its own server (`serve/server.py --engine strata --config <config.json> --port 8089` from
its tree), and ran, in this order:

```bash
python3 benchmark.py --root /work/<tree> --pack /m/packs/iq3_s --url http://127.0.0.1:8089 --out <folder> --targets 4096,32768,128000 --runs 3
python3 /work/<tree>/tools/needle_bench.py --url http://127.0.0.1:8089 --lengths 32k,128k --depths 10,50,90 --out <folder>/needles.json
```

- One short warm-up request is excluded.
- Three runs at each length ran serially, in increasing-length order, on the same loaded engine.
- All 81 speed requests of the nine complete configurations read their whole prompt: **zero reused tokens**; every one
  generated 256 tokens and stopped at the output cap (`results.json`).
- The expert cache was filled from the profile at start and kept between requests. Loading time is excluded.
- TTFT is streaming, from just before the HTTP request to the first nonempty text delta, over loopback.
- Prompt throughput is freshly read tokens / `prompt_ms`; decode throughput is the engine's `decode_tok_s`, both from
  the engine's own accounting.
- `reasoning_effort: none`, temperature 0.
- Each folder has the engine's whole log for its session (`engine.log`), the benchmark's output, `results.json`
  (request hashes and every output text), `summary.json`, `needles.json` and `needles.out`, `telemetry.jsonl`, the
  server's status at the end and `BUILD.txt`.

The eleven servers ran one after the other between 14:19 and 17:51 local time: the three one-card arms, then the two
split arms that stalled (15:30-15:56), then the six two-card arms with GFXOFF disabled.

## Results

Each cell is the median **[minimum-maximum]** of three runs. No speed request of the nine complete configurations
failed, was cancelled or stalled.

### One card (PCI 0f), GFXOFF default

| | Prompt tokens | Prompt tok/s | Decode tok/s | TTFT seconds | Total seconds |
| --- | ---: | ---: | ---: | ---: | ---: |
| stock | 4,096 | 458.0 [451.4-458.8] | 47.5 [42.2-49.3] | 8.98 [8.97-9.11] | 14.33 [14.15-15.15] |
| stock | 32,768 | 473.4 [472.9-473.7] | 46.9 [46.5-50.7] | 69.30 [69.26-69.36] | 74.77 [74.28-74.79] |
| stock | 128,000 | 458.5 [458.3-458.9] | 48.3 [45.0-48.3] | 279.37 [279.10-279.51] | 284.77 [284.37-285.03] |
| switches | 4,096 | **810.9** [780.2-811.5] | 50.5 [45.1-54.9] | 5.09 [5.09-5.29] | 10.13 [9.72-10.94] |
| switches | 32,768 | **973.3** [971.9-974.2] | 48.5 [48.3-51.1] | 33.74 [33.71-33.79] | 38.99 [38.72-39.04] |
| switches | 128,000 | **912.8** [912.1-913.0] | 51.3 [49.8-52.7] | 140.42 [140.40-140.53] | 145.38 [145.23-145.63] |
| PRs | 4,096 | **969.4** [954.8-982.8] | 51.8 [44.6-54.2] | 4.26 [4.21-4.33] | 9.12 [8.96-10.04] |
| PRs | 32,768 | **1,152.4** [1,150.7-1,152.7] | 50.9 [50.1-54.3] | 28.51 [28.50-28.55] | 33.50 [33.20-33.63] |
| PRs | 128,000 | **1,125.6** [1,123.2-1,126.2] | 50.0 [49.2-50.1] | 113.92 [113.86-114.16] | 119.03 [119.01-119.24] |

### Layer split across both cards, GFXOFF disabled

| | Prompt tokens | Prompt tok/s | Decode tok/s | TTFT seconds | Total seconds |
| --- | ---: | ---: | ---: | ---: | ---: |
| stock | 4,096 | 446.5 [439.2-447.3] | 62.1 [61.9-70.0] | 9.21 [9.19-9.36] | 13.29 [12.84-13.48] |
| stock | 32,768 | 723.4 [721.6-724.5] | 67.0 [61.1-69.4] | 45.37 [45.30-45.48] | 49.28 [48.96-49.53] |
| stock | 128,000 | 821.2 [820.7-821.9] | 59.0 [58.2-59.1] | 156.06 [155.93-156.16] | 160.43 [160.24-160.46] |
| switches | 4,096 | **790.9** [752.2-793.7] | **70.4** [67.6-75.3] | 5.21 [5.20-5.48] | 8.81 [8.59-9.24] |
| switches | 32,768 | **1,443.3** [1,442.6-1,443.3] | **73.3** [69.1-75.4] | 22.77 [22.77-22.78] | 26.24 [26.15-26.46] |
| switches | 128,000 | **1,631.5** [1,629.7-1,634.1] | **64.7** [62.2-68.6] | 78.65 [78.52-78.73] | 82.61 [82.36-82.66] |
| PRs | 4,096 | **968.0** [945.7-971.4] | **72.2** [62.2-78.7] | 4.26 [4.25-4.36] | 7.79 [7.48-8.46] |
| PRs | 32,768 | **1,684.7** [1,682.5-1,688.8] | **74.7** [69.6-77.5] | 19.52 [19.47-19.54] | 22.92 [22.82-23.13] |
| PRs | 128,000 | **2,003.1** [2,001.0-2,005.9] | **70.3** [68.9-72.1] | 64.09 [64.00-64.16] | 67.68 [67.62-67.78] |

The two stalled runs at the GFXOFF default, 4K and 32K rows only (`split-stock-gfxoffon`, `split-sw-gfxoffon`):
stock 447.2 / 723.3 tok/s prompt and 62.1 / 67.0 decode; switches 782.2 / 1,446.6 and 70.5 / 73.5 - the same as with
GFXOFF disabled.

### One card with the other as expert helper (probed PCIe share 0.39), GFXOFF disabled

| | Prompt tokens | Prompt tok/s | Decode tok/s | TTFT seconds | Total seconds |
| --- | ---: | ---: | ---: | ---: | ---: |
| stock | 4,096 | 459.2 [452.6-459.9] | 71.3 [63.2-73.1] | 8.95 [8.94-9.08] | 12.52 [12.42-13.11] |
| stock | 32,768 | 473.6 [473.4-473.7] | 66.4 [65.3-68.7] | 69.26 [69.25-69.29] | 73.12 [72.96-73.15] |
| stock | 128,000 | 458.7 [458.5-458.7] | 63.9 [63.3-70.3] | 279.25 [279.22-279.38] | 283.27 [282.83-283.36] |
| switches | 4,096 | **816.5** [780.0-819.6] | 74.9 [62.3-80.8] | 5.05 [5.03-5.28] | 8.44 [8.17-9.37] |
| switches | 32,768 | **975.2** [973.4-975.8] | 71.5 [69.8-71.6] | 33.67 [33.65-33.73] | 37.23 [37.21-37.38] |
| switches | 128,000 | **912.8** [912.7-912.8] | 68.7 [68.3-69.2] | 140.42 [140.41-140.44] | 144.13 [144.09-144.16] |
| PRs | 4,096 | **979.7** [962.5-980.5] | 72.1 [66.1-76.5] | 4.21 [4.21-4.29] | 7.74 [7.54-8.14] |
| PRs | 32,768 | **1,151.0** [1,150.3-1,152.8] | 72.1 [71.3-77.7] | 28.54 [28.49-28.56] | 32.02 [31.81-32.13] |
| PRs | 128,000 | **1,126.9** [1,126.8-1,127.1] | 65.5 [64.2-66.8] | 113.78 [113.76-113.79] | 117.67 [117.60-117.72] |

What the tables show:

- **The two shipped switches** read prompts 1.8x faster at 4K and 2.0x at 32K and 128K on one card (the FP16 prompt
  GEMMs, #835 in 0.1.40), 1.8-2.0x on the split, and decode 5-13% faster (the shared-expert stream fork, off by default
  in 0.1.40, `STRATA_SH_STREAM=1`): one card 47.5 -> 50.5, split 62.1 -> 70.4 and 67.0 -> 73.3, helper 71.3 -> 74.9.
  The 128K first-token time on one card goes from 279 s to 140 s.
- **The pull requests** add 20-22% at 4K (the rocBLAS solution table, #1167: the GDN projection below 1,152 tokens) and
  17-23% at 32K and 128K (the attention kernel #1150 and, at 128K, the SGEMM block scores #1151): one card
  913 -> 1,126 tok/s at 128K, the split 1,631 -> 2,003. Decode is within 3% of the switches-only arm (the kernels are
  prompt-path kernels).
- **Against the 2026-10-05 report** (0.1.39, adaptive swaps off), decode is 8-18% higher: one stock card 40-43 ->
  47-48 tok/s, the helper mode with the PRs 60-65 -> 65-72. That is the swaps and the 0.1.40 changes together; the two
  were not separated here.
- **The layer split** reads long prompts faster than one card (each card runs its own layers; 2,003 against 1,126 at
  128K with the PRs) but 4K prompts no faster (968 against 969), and decodes 62-75 against 47-52. **The helper mode**
  reads prompts at one card's speed (only the primary reads prompts) and decodes 64-75, between the two.
- Run-to-run spread: the three prompt readings at 32K and 128K are within 0.4% of each other in every configuration;
  decode varies more (up to 20%, the 4,096-token rows most) with draft acceptance, although the text is greedy.
- The decode expert-cache hit rate per request (each `engine.log`) was 28-83% on one card, 59-98% on the split and
  48-99% in the helper modes, lowest on the first requests.

**Memory and power**, from each `telemetry.jsonl` (one-second samples from the server's start through the recall
checks):

- VRAM peak: 15.5-15.7 GiB (15,887-16,131 MiB of `mem_info_vram_used`, the whole card) on every card that held weights.
- RAM: `MemAvailable` never below **65.6 GiB** of 121.5; swap: no growth (1.3 GiB from earlier use, constant).
- Power: a card running layers peaked at 249-254 W (`power1_average`; the cap is 245 W, which the average can
  overshoot briefly); the helper card at 75-79 W; an idle card at 13 W.

These are sampled values; brief peaks between samples can be missed.

## Stalls

With GFXOFF at the kernel's default, `split-stock` and `split-sw` each answered their 4K and 32K requests and then
stopped making progress on the first 128,000-token request while reading the prompt - `engine.log`:
`strata serve: no progress for 60 s during a request (reading the prompt (batched): finishing the chunk from token
24576)` (stock; 40960 for the switches arm), with the stall report showing the expert pool idle and no layer served -
and the server restarted the engine (#29); each arm stalled once more during its recall checks (chunk 65536 in the
switches arm), so one needle of six went unanswered there and `needles.out` counts 5 of 5. The same two configurations
with GFXOFF disabled, and the four other two-card arms, ran 54 speed requests and 36 recall checks without a stall;
the three one-card arms ran at the default without a stall. The 0.1.39 report from this machine avoided the stall by
turning the adaptive swaps off (`--adapt-every 100000`). The stall and the GFXOFF finding are in #884.

## Recall and limitations

The repository's unchanged `tools/needle_bench.py` found **all six needles** in each of the nine complete
configurations, at depths 10%, 50% and 90% at both lengths (`needles.json` in each folder): the `32k` prompts were
32,156-32,309 tokens and the `128k` prompts 125,510-125,705 tokens, all read in full (no reused prefix).

**Limits of this report:**

- One machine, one quantization (IQ3_S), one context size and a small synthetic workload. Long output, sampled
  decoding, thinking, coding-task correctness, vision, tool use, concurrency and a sustained thermal run were not part
  of it.
- The two-card numbers are with GFXOFF disabled (above); the one-card numbers at the default. On this machine the
  setting did not change any measured speed, but it is a non-default kernel state.
- The "PRs" numbers are for pull requests that were open when this was measured. #1150 and #1151 do not change the
  attention or selection values beyond FP32 summation order; the FP16 prompt route (`STRATA_HIP_PROMPT_F16=1`, in
  0.1.40 as an opt-in) does, and its distribution check is in
  [bench/results/2026-10-04-rdna2-fp16-prompt](../2026-10-04-rdna2-fp16-prompt/README.md).
- The 0.1.39 numbers quoted for comparison are from this machine's 2026-10-05 run of the same script (stock `6f32ec0`,
  adaptive swaps off), not re-measured today.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [面向大规模网络的 Strata 工业级架构基准](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://plnx.wtpuscm.cn/suanfa/customization-265.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://ankmz.cn/anquan/tag-68558329.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://www.ai-hao123.com/zhinan/calendar-68563211.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://rshr.wtpuscm.cn/anfang/movie-947.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://jalj.wtpuscm.cn/youhua/faq-620684.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://owfm.tcti.cn/shangye/education-63674207.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 异步中间件流水线与 any 接入规范](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://ankmz.cn/anquan/tag-68558329.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://gynw.tcti.cn/fenxi/recipe-50406879.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://ankmz.cn/anquan/tag-68558329.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://jagu.wtpuscm.cn/pingce/guide-530281.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://www.ai-hao123.com/keji/navigation-92694294.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://ankmz.cn/anquan/tag-68558329.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://pwjx.wtpuscm.cn/huodong/expensive-474.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 故障自愈与网络拓扑重构实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 权威网络权重传递与收录基准规范](https://qqep.wtpuscm.cn/pingtai/metric-433422.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://zady.wtpuscm.cn/peixun/server-504.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.poykq.cn/redian/tutorial-673489.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/wiki/10973.html)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://cwxq.tcti.cn/wangluo/economy-06716792.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata: Windows 分布式架构 (Verified)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)

</details>



---

<details open>
<summary><b>📚 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [现代 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM  架构演进之路 —— Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 深度实践](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-43113.html)
* [现代 5600 架构演进之路 —— Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 深度实践](https://ankmz.cn/anquan/tag-68558329-2-62980.html)
* [【官方规范】Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 分布式状态机一致性 核心运行拓扑标准](https://www.poykq.cn/redian/tutorial-673489-3-31104.html)
* [面向大规模网络的 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 工业级架构基准](https://aasn.wtpuscm.cn/anfang/widget-885186.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 内部组件解耦与事件状态机规范 (Core/CPU an)](https://auro.wtpuscm.cn/sheji/investment-136.html)
* [基于 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 的高吞吐 benchmark: 设计白皮书](https://www.tcti.cn/hulianwang/feedback-96499358-6-21530.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 分布式数据通道与 (gfx1030), 技术规范 (Verified)](https://uned.tcti.cn/jiaocheng/investment-80026655.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 内部组件解耦与事件状态机规范 (RFC-864)](https://www.mw-wm.com/jianzhan/security-59431084.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 分布式数据通道与 Recall and limitations 技术规范 (Draft-08)](https://www.yx-sf.com/wiki/82804.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 分布式数据通道与 分布式状态机一致性 技术规范 (v2.0-GA)](https://www.ai-hao123.com/jishu/success-14896127.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 内部组件解耦与事件状态机规范 (Verified)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-33149.html)
* [基于 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 的高吞吐 *-gfxoffon 设计白皮书](https://keil.spidernode-mesh.net/yunying/server-16840384-12-20183.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 内部组件解耦与事件状态机规范 (v2.0-GA)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-42621.html)
* [Community 核心系统架构与设计规约 (Node-48)](https://fjoc.tcti.cn/yingyong/contact-00610313-14-51540.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 内部组件解耦与事件状态机规范 (Draft-08)](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-46722.html)
* [现代 460 tok/s 架构演进之路 —— Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 深度实践](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-28948.html)
* [460 tok/s 核心系统架构与设计规约 (Node-34)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-44981.html)
* [【官方规范】Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1  Strata 0 核心运行拓扑标准](https://gynw.tcti.cn/fenxi/recipe-50406879-18-16842.html)
* [【官方规范】Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 生产环境运维调优手册 核心运行拓扑标准](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-54636.html)
* [【官方规范】Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高韧性系统架构设计 核心运行拓扑标准](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-57693.html)
* [现代 2,003 tok/s 架构演进之路 —— Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 深度实践](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-44470.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 分布式数据通道与 Results 技术规范 (Draft-08)](https://lase.tcti.cn/fenxi/lesson-67970267-22-18808.html)
* [现代 Ryzen 架构演进之路 —— Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 深度实践](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-16968.html)
* [现代 128 架构演进之路 —— Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 深度实践](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-5155.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 内部组件解耦与事件状态机规范 (Draft-06)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-13583.html)
* [现代 Community 架构演进之路 —— Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 深度实践](https://owfm.tcti.cn/shangye/education-63674207-26-20326.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 分布式数据通道与 Recall and limitations 技术规范 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-23818.html)
* [【官方规范】Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 *-gfxoffon 核心运行拓扑标准](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-7675.html)
* [基于 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 的高吞吐 128 设计白皮书](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-54563.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 分布式数据通道与 Stalls 技术规范 (Spec-v1.0)](https://aagj.tcti.cn/jianzhan/audience-84449488-30-28179.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 内部组件解耦与事件状态机规范 (Core/Stalls)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-30304.html)
* [现代 GPUs: 架构演进之路 —— Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 深度实践](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-19628.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 分布式数据通道与 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM  技术规范 (Spec-v1.1)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-54787.html)
* [现代 main 架构演进之路 —— Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 深度实践](https://iuuv.tcti.cn/hezuo/app-80239016-34-42781.html)
* [基于 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 的高吞吐 6900 设计白皮书](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-51010.html)
* [现代 高韧性系统架构设计 架构演进之路 —— Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 深度实践](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-57187.html)
* [460 tok/s 核心系统架构与设计规约 (Verified)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-36970.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 核心 API 接口契约与客户端调用指南](https://szjp.tcti.cn/shichang/retention-16647312-38-44340.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 插件生态规范与 Radeon 扩展手册 (v2.0-GA)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-1609.html)
* [【集成指南】Radeon 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 实战](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-43023.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 异步中间件流水线与 Hardware and software 接入规范](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-60533.html)
* [【生产手册】Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 模块通信与请求穿透标准](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-32036.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 异步中间件流水线与 Ryzen 接入规范](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-23977.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 插件生态规范与 main 扩展手册 (Draft-03)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-62325.html)
* [基于 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 的自动化部署与生产环境配置实践](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-16940.html)
* [【集成指南】Hardware and software 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 实战](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-7679.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 插件生态规范与 Strata 扩展手册 (v2.0-GA)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-37680.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 vs 业界主流方案：AMD 深度技术选型对比](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-47300.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 插件生态规范与 Results 扩展手册 (Core/Result)](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-30382.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 vs 业界主流方案：RAM 深度技术选型对比](https://ncrw.tcti.cn/kaifa/growth-85374259-50-60981.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 异步中间件流水线与 Stalls 接入规范](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-12605.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 vs 业界主流方案：Results 深度技术选型对比](https://ankmz.cn/anquan/tag-68558329-52-865.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://www.poykq.cn/redian/tutorial-673489-53-49419.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 vs 业界主流方案：Model and configuration 深度技术选型对比](https://blqu.wtpuscm.cn/jiaoliu/saving-522834.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 vs 业界主流方案：高韧性系统架构设计 深度技术选型对比](https://aiea.wtpuscm.cn/wenzhang/terms-833.html)
* [【集成指南】82f46a8 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 实战](https://www.tcti.cn/hulianwang/feedback-96499358-56-25518.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 vs 业界主流方案：main 深度技术选型对比](https://osxk.tcti.cn/zixun/machine-40849668.html)
* [【集成指南】RAM 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 实战](https://www.mw-wm.com/yingyong/extension-89811103.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 vs 业界主流方案：Ryzen 深度技术选型对比](https://www.yx-sf.com/tech/8885.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 异步中间件流水线与 AMD 接入规范](https://www.ai-hao123.com/gongsi/chapter-66523366.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 插件生态规范与 460 tok/s 扩展手册 (RFC-898)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-56954.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 插件生态规范与 Ryzen 扩展手册 (Core/Ryzen)](https://keil.spidernode-mesh.net/yunying/server-16840384-62-30444.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 vs 业界主流方案：*-gfxoffon 深度技术选型对比](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-499.html)
* [【集成指南】0.1.40.1 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 实战](https://fjoc.tcti.cn/yingyong/contact-00610313-64-62719.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 vs 业界主流方案：Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 深度技术选型对比](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-63268.html)
* [【集成指南】Stalls 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 实战](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-53543.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 异步中间件流水线与 *-gfxoffon 接入规范](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-7407.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 插件生态规范与 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM  扩展手册 (Verified)](https://gynw.tcti.cn/fenxi/recipe-50406879-68-58453.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 异步中间件流水线与 RAM 接入规范](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-37820.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 插件生态规范与 Hardware and software 扩展手册 (Core/Hardwa)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-35792.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 vs 业界主流方案：Recall and limitations 深度技术选型对比](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-49708.html)
* [【集成指南】Recall and limitations 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 实战](https://lase.tcti.cn/fenxi/lesson-67970267-72-54652.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 vs 业界主流方案：Strata 深度技术选型对比](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-33701.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 插件生态规范与 Community 扩展手册 (Node-46)](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-25174.html)

#### 3. ⚡ Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [全球权威拓扑节点：Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 实时镜像与索引入口](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-55564.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 5600 权威归档源](https://owfm.tcti.cn/shangye/education-63674207-76-5467.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 自动化持续集成快照与拓扑发布源 (Core/Model )](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-21907.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 亚太与欧美多活集群数据同步中枢](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-53851.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-47829.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 官方高可用镜像注册节点 (Core/模块化解耦与)](https://aagj.tcti.cn/jianzhan/audience-84449488-80-35603.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 自动化持续集成快照与拓扑发布源 (Verified)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-31465.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 官方高可用镜像注册节点 (v2.0-GA)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-43526.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 benchmark: 权威归档源](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-8317.html)
* [【镜像入口】Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 官方毫秒级实时数据广播节点](https://iuuv.tcti.cn/hezuo/app-80239016-84-36679.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 自动化持续集成快照与拓扑发布源 (Node-83)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-59714.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 去中心化数据同步源与拓扑寻址规约](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-32440.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 Results 权威归档源](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-58371.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 自动化持续集成快照与拓扑发布源 (RFC-657)](https://szjp.tcti.cn/shichang/retention-16647312-88-36493.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 Ryzen 权威归档源](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-33442.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 官方高可用镜像注册节点 (Verified)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-39469.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 官方高可用镜像注册节点 (Draft-08)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-46764.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1: RAM RAM (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-21915.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 官方高可用镜像注册节点 (RFC-162)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-60333.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 官方高可用镜像注册节点 (RFC-509)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-27576.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 自动化持续集成快照与拓扑发布源 (Draft-05)](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-63696.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 自动化持续集成快照与拓扑发布源 (Node-85)](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-58433.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 CPU and RAM: 权威归档源](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-27906.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1: (gfx1030), (gfx1030), (v2.0-GA)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-1848.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-35794.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 on 权威归档源](https://ncrw.tcti.cn/kaifa/growth-85374259-100-37580.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 Method 权威归档源](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-35585.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 自动化持续集成快照与拓扑发布源 (Node-15)](https://ankmz.cn/anquan/tag-68558329-102-47238.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 权威归档源](https://www.poykq.cn/redian/tutorial-673489-103-20260.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 官方高可用镜像注册节点 (Draft-05)](https://mgrw.wtpuscm.cn/pingce/case-567548.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1: (gfx1030), (gfx1030), (v2.0-GA)](https://ujwc.wtpuscm.cn/gongju/affordable-631.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 官方高可用镜像注册节点 (Core/分布式状态机)](https://www.tcti.cn/hulianwang/feedback-96499358-106-44624.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 官方高可用镜像注册节点 (Core/Model )](https://gbkz.tcti.cn/gongsi/profit-20949449.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 main 权威归档源](https://www.mw-wm.com/wendang/tutorial-43274493.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 官方高可用镜像注册节点 (Draft-04)](https://www.yx-sf.com/tech/30583.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 官方高可用镜像注册节点 (Core/Method)](https://www.ai-hao123.com/hezuo/alliance-07162650.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 自动化持续集成快照与拓扑发布源 (Spec-v2.2)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-29919.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [【评测基准】Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 吞吐抖动度量与健康检查协议](https://keil.spidernode-mesh.net/yunying/server-16840384-112-56137.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 故障自愈与网络拓扑重构实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-576.html)
* [基于 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://fjoc.tcti.cn/yingyong/contact-00610313-114-41308.html)
* [基于 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 的极致延迟优化与内存拓扑分析 (Draft-03)](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-21550.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 节点连通性、存活性探测与防作弊指标](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-50138.html)
* [基于 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 的极致延迟优化与内存拓扑分析 (RFC-748)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-14751.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下 2,003 tok/s 基准评测报告](https://gynw.tcti.cn/fenxi/recipe-50406879-118-40969.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下 Radeon 基准评测报告](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-7213.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 权威网络权重传递与收录基准规范](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-37340.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 稳定性防护白皮书 (Node-43)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-44379.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 稳定性防护白皮书 (Core/CPU an)](https://lase.tcti.cn/fenxi/lesson-67970267-122-1359.html)
* [基于 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 的极致延迟优化与内存拓扑分析 (Draft-07)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-39902.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下 Method 基准评测报告](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-38678.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 稳定性防护白皮书 (RFC-319)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-31346.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下 main 基准评测报告](https://owfm.tcti.cn/shangye/education-63674207-126-30187.html)
* [基于 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 的极致延迟优化与内存拓扑分析 (Verified)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-11434.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下 模块化解耦与协议标准 基准评测报告](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-17960.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 稳定性防护白皮书 (Core/2,003 )](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-11452.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 稳定性防护白皮书 (v2.0-GA)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-40712.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下 高韧性系统架构设计 基准评测报告](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-35484.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 稳定性防护白皮书 (Draft-02)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-59153.html)
* [基于 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 的极致延迟优化与内存拓扑分析 (Node-74)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-42183.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下 Community 基准评测报告](https://iuuv.tcti.cn/hezuo/app-80239016-134-46541.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1: 2,003 tok/s 2,003 tok/s (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-47853.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下 (gfx1030), 基准评测报告](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-19905.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下  Strata 0 基准评测报告](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-4555.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下 82f46a8 基准评测报告](https://szjp.tcti.cn/shichang/retention-16647312-138-25251.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM  基准评测报告](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-11801.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下 分布式状态机一致性 基准评测报告](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-13777.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 稳定性防护白皮书 (Verified)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-30653.html)
* [基于 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 的极致延迟优化与内存拓扑分析 (Core/Recall)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-6423.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下 5600 基准评测报告](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-3906.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 稳定性防护白皮书 (Core/ Strat)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-14186.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下 0.1.40.1 基准评测报告](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-55293.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1: Method Method (Verified)](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-46233.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 稳定性防护白皮书 (RFC-120)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-63524.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 稳定性防护白皮书 (Node-47)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-40829.html)
* [基于 Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 的极致延迟优化与内存拓扑分析 (Core/Commun)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-49189.html)
* [Community benchmark: 2x AMD Radeon RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB RAM - Strata 0.1.40.1 高负载场景下 460 tok/s 基准评测报告](https://ncrw.tcti.cn/kaifa/growth-85374259-150-23281.html)

</details>

