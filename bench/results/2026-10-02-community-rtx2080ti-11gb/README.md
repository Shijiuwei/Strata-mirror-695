# Community benchmark on RTX 2080 Ti 11 GB

Measured on 2026-10-02 by [homeofe](https://fdxu.wtpuscm.cn/jiaoliu/ai-635882.html). This tests Strata
engine 0.1.33 (source build, CUDA 12.0, sm_75) with Qwen3.8-Flash-Next IQ3_S at a
**262,144-token context** on an 11 GB Turing card, a 24-core Zen 2 CPU and 128 GB of
DDR4-2133. It is a single-GPU, low-VRAM configuration: KV streaming (32,768 cells
per layer on the GPU) and a 3.25 GiB expert cache, with the image encoder on the CPU.

Median results over three runs: prompt read of **497 tok/s at 4,096 tokens, 637
tok/s at 32,768, and 566 tok/s at 128,000**. Decode ran at **39.6, 38.0 and
34.1 tok/s**. One 250,000-token prompt read at 466.6 tok/s (536.8 s to first
token) and then decoded at 31.8 tok/s. A follow-up turn on that conversation started
answering after 4.6 s. Every recall check found its code word, and all three image
questions were answered exactly.

The main limitation is that everything was measured through the server that runs on
this PC. That server is a shared service, not a dedicated benchmark instance: it was
not restarted for this report, and an agent also uses it. Each request was checked
for overlap with other traffic (see Method), and none overlapped.

## Hardware and software

- **GPU:** NVIDIA GeForce RTX 2080 Ti, 11,264 MiB, compute capability 7.5, VBIOS
  90.02.17.00.47. Power limit 260 W (default, not changed); clocks not fixed.
  PCIe 3.0 x16 capable; nvidia-smi shows Gen 1 x16 at idle (power management). The
  engine's startup probe measured `13.1 GB/s host->device -> pcie_frac 0.28`.
- **CPU:** AMD Ryzen Threadripper 3960X, 24 cores / 48 threads, Zen 2, AVX2, no
  AVX-512. Governor `schedutil`. The engine uses 23 expert-pool workers plus the host
  thread; the image encoder uses 24 threads.
- **RAM:** 128 GB installed, 8 x 16 GB DDR4 (rated DDR4-3200, running at 2133 MT/s);
  Linux reports 125.7 GiB. Swap: an 8 GiB swap file, **already full (8.0 GiB used)
  before and during all runs**. vmstat showed no swap-in or swap-out during a prompt
  read, so the swap was occupied but not actively paging.
- **Storage:** model files on an ext4 LVM volume spanning two Samsung SSD 970 EVO
  Plus 1 TB NVMe drives.
- **OS:** Ubuntu 24.04.4 LTS, kernel 6.8.0-139-generic, NVIDIA driver 595.84 (CUDA
  13.2 driver API).
- **Strata:**
  - Engine 0.1.33, built from source on this PC with CUDA 12.0 (nvcc 12.0.140), GCC
    13.3.0, arch 75 ([BUILD.json](BUILD.json)).
  - The server runs `main` at `aeb35be` (tag `v0.1.33` plus one HIP-only commit)
    with the server-side change from PR #436 applied (`serve/server.py`: a request is
    cancelled when its client disconnects). That change does not touch the engine
    and does not affect requests whose client stays connected, which covers every
    measurement here.
  - Python 3.12.3 (the install's venv).
- **Background workloads:**
  - Docker, an OpenClaw agent gateway and other services ran on the CPU.
  - The OpenClaw agent sends requests to this same Strata server from time to time.
  - No other process used the GPU; the systemd unit stops the PC's other GPU model
    services when it starts.

## Model and configuration

- Model: `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`, quantization IQ3_S,
  downloaded by `setup.sh` on 2026-10-01. Setup pins revision
  `ed59f92082b1e93c0e96d60a8b11aab089b52f09`; the file hashes were not re-verified
  for this report.
  - `IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf` (54,817,524,224 bytes)
  - `IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf` (28,800,138,432 bytes)
  - `mmproj-Qwen3.8-Flash-Next-BF16.gguf` (907,543,008 bytes)
- Native pack `packs/iq3_s` prepared by setup (`tools/iq_pack.py`); MTP draft layer
  `mtp/rt` from setup; the bundled `data/expert-profile.bin`. No calibration, no
  custom profile, no control vectors.
- **Vision:** `strata-vision` on the CPU (`"gpu": false`, 24 threads, at most 1,024
  image tokens). The engine binary was built with GPU vision support; the config
  moves the encoder to the CPU by hand (see the encoder note below).
- **Context and memory:**
  - Context 262,144 with INT8 KV. KV streaming keeps 32,768 cells per QSA layer in
    VRAM (`--kv-resident 32768`) and the rest in 3.09 GiB of pinned RAM.
  - Expert cache `auto`: 1,692 slots, 3.25 GiB of VRAM, profile-prefilled, no eviction.
  - Prefill `auto` chose 3,072-token prompt chunks; the prompt path borrows 1,361
    cache slots (2.61 GiB).
  - VRAM reserve 700 MiB. Low-RAM mode off.
- **Decoding:** MTP `--spec 4 --spec-min-p 0.5` (plus the built-in suffix drafts).
  Requests used `reasoning_effort: "none"` and `temperature: 0`. Experimental speed
  projection off (`projection: null` in every request record).
- The server unloads the model after 1,800 s idle (`idle_unload_s`). It stayed
  loaded throughout; every run checked `/health` `loaded: true` first.

Configuration with the API key removed and the install path replaced by `<workspace>`:
[strata-iq3_s.json](strata-iq3_s.json). The launch, from the systemd user unit
(`LimitMEMLOCK=16867729408`), was:

```text
.venv/bin/python serve/server.py --engine strata --config strata-iq3_s.json --port 8080
# the engine command it starts (from the config):
engine/strata --pack <data>/packs/iq3_s --native <data>/models/IQ3_S/...-00001-of-00002.gguf \
  --ple-gguf <data>/models/IQ3_S/...-00002-of-00002.gguf --expert-profile data/expert-profile.bin \
  --expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5 --mtp <data>/mtp/rt \
  --max-context 262144 --kv int8 --kv-resident 32768 --vision --vram-reserve-mib 700
```

The engine's startup lines for this server process are in
[engine-startup.log](engine-startup.log). The first line, `ggml_cuda_init: failed
to initialize CUDA`, comes from the CPU image encoder, which runs with CUDA hidden.
The relevant startup lines:

```text
strata generate: KV streaming: 32768 of 262144 cells per QSA layer in VRAM, the K/V in 3.09 GiB of pinned RAM
strata generate: loaded 46.84 GiB at 2.25 GiB/s
strata generate: expert cache auto: 4.15 GiB free, 700 MiB reserved (+218 MiB for the draft head) -> 1313 slots
strata generate: expert cache 1692 slots, 3.25 GiB of VRAM; policy is ... PROFILE, ranked by routing frequency, no eviction.
strata serve: prompt chunk auto: 3072 tokens
strata serve: the prompt path borrows 1361 CUDA0 cache slots (2.61 GiB)
strata serve: 421 MiB of VRAM free with everything loaded
```

## Method

The scripts are in this folder. Run them with the install's Python, which provides
`regex` for the tokenizer. Each script reads the API key at run time from
`STRATA_API_KEY` or from `--key-config`; the key is never written.

- [bench_server.py](bench_server.py): fresh-prompt and follow-up speed.
  - **Prompts:** the haystack from `tools/needle_bench.py` (this repository's docs
    and sources, in fixed order; `third_party/llama.cpp/docs` was not present in the
    checkout used, which was at `aeb35be`). It is cut to an exact token count with the pack's
    tokenizer and chat template, then followed by a request for a 600-word prose
    overview.
  - **Output:** 400 tokens maximum; every speed run generated exactly 400 tokens
    (`finish: length`) of English prose. Streaming with `stream_options.include_usage`.
  - **No reuse:** every prompt starts with its own random nonce, so no earlier
    prompt prefix can be reused. The engine log confirms **0 reused tokens** for
    every fresh run.
  - **Needle:** prompts of 100K tokens or more carry a code word at 40% depth.
  - **Follow-up:** after the third 128K run and after the 250K run, a second user
    turn is sent on the same conversation (the long prefix plus the first answer).
    It asks for the code word and about 400 more words of prose.
  - Commands:
    - `--targets 4096,32768,128000 --runs 3 --followup --warmup`
    - then `--targets 250000 --runs 1 --followup --seed 2027`
- [run_needles.py](run_needles.py): runs the unchanged `tools/needle_bench.py` with
  `--lengths 32k,128k --depths 10,50,90` and keeps the engine log lines.
- [image_check.py](image_check.py): three newly drawn 768x512 PNGs, so the server's
  encoded-image cache cannot serve them. Each has a code in large type and a known
  number of red circles and blue squares. The answer must match `CODE; circles;
  squares` exactly.
- [monitor.py](monitor.py): once per second, nvidia-smi memory/utilization/power,
  `/proc/meminfo` and the RSS of `strata` and `strata-vision`, written to
  [data/telemetry.jsonl](data/telemetry.jsonl). It ran through all measurements.
- [summarize.py](summarize.py): builds [runs.csv](runs.csv) and
  [summary.json](summary.json), including per-run memory peaks.

**Foreign-traffic check:**
- Before every request, the scripts wait until `GET /status` reports `busy: false,
  queued: 0` for 3 s.
- After it, the run is discarded and repeated if `GET /metrics` counted more than
  one finished request or the engine log gained more than one `strata serve:
  prompt` line.
- No run was discarded. Each JSON file records `totals_before` / `totals_after`.

**Warm-up and state:**
- One 8-token warm-up request was excluded.
- The server had been started about 6 minutes before the first run. It had served
  two short requests and was not restarted between runs.
- The expert cache is profile-filled with no eviction, so its contents do not depend
  on earlier requests. Model loading is excluded.
- Runs were serial, in increasing size.

**Timing boundaries:**
- Prompt tok/s is the freshly read tokens divided by the engine's `prompt_ms`.
- Decode tok/s is the engine-generated tokens (draft-verified tokens included)
  divided by `decode_ms`.
- Both come from `GET /metrics`, which has 0.1 ms resolution and matches the engine
  log lines.
- TTFT is client wall time from just before the HTTP request to the first non-empty
  text delta. With reasoning off, that is answer text. Keep-alive comments and the
  empty role chunk are ignored.
- Total is client time to the end of the stream.
- All client times are over loopback and include the server's tokenization.

**Memory:**
- GPU memory is nvidia-smi `memory.used` for the whole card.
- Host RAM is `MemTotal - MemAvailable` for the whole system, including other
  services. It is not process RSS.

Per-run JSON in [data/](data/) holds the request hash, client timings, usage, the
`/metrics` record, the parsed engine timing, the output text, and that request's
engine and server log lines. Paths in these files are shortened to `<workspace>` (`common.py`, `STRATA_SCRUB_PREFIX`).

## Results

Each cell is the median **[minimum-maximum]** of three runs, except where marked
"1 run". Every speed request generated 400 tokens and stopped at the output cap. No
speed request failed or was cancelled.

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median and range | Decode tok/s median and range | TTFT seconds median and range |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| Fresh 4K | 4,096 | 0 | 400 | 3 | 496.9 [483.7-498.6] | 39.6 [39.3-41.2] | 8.30 [8.27-8.52] |
| Fresh 32K | 32,768 | 0 | 400 | 3 | 636.9 [635.8-638.5] | 38.0 [37.7-40.0] | 51.61 [51.48-51.70] |
| Fresh 128K | 128,000 | 0 | 400 | 3 | 566.2 [566.0-566.9] | 34.1 [33.7-34.3] | 226.59 [226.34-226.67] |
| Fresh 250K | 250,000 | 0 | 400 | **1 run** | 466.6 | 31.8 | 536.80 |
| Follow-up at 128K depth | 128,463 | 128,400 | 400 | 1 run | 63 tokens read in 1.28 s | 35.0 | 1.85 |
| Follow-up at 250K depth | 250,463 | 249,993 | 400 | 1 run | 470 tokens read in 3.56 s | 34.6 | 4.55 |

Further per-configuration values. Draft acceptance is MTP drafts accepted /
proposed, from the engine line `drafts accepted A of B`. Hit rate and KV hits are
from `decode expert cache hit rate` and `KV streaming: N% of block reads hit VRAM`.

| Configuration | Total s | Draft acceptance | Decode expert-cache hit rate | KV block reads that hit VRAM | KV read from RAM |
| --- | --- | --- | --- | --- | --- |
| Fresh 4K | 18.34 [17.97-18.67] | 65.6% [62.2-67.0] (240/366, 234/349, 227/365) | 55.0% [50.2-55.1] | 99.59% | 54 MiB |
| Fresh 32K | 62.18 [61.45-62.19] | 63.7% [62.6-67.2] (238/354, 228/364, 226/355) | 54.0% [52.9-54.6] | 97.71% [97.69-97.81] | 295-308 MiB |
| Fresh 128K | 238.32 [238.03-238.42] | 55.7% [54.3-58.4] (195/350, 189/348, 187/320) | 55.2% [54.9-55.6] | 95.62% [95.54-95.73] | 596-616 MiB |
| Fresh 250K (1 run) | 549.33 | 53.4% (195/365) | 54.9% | 94.25% | 825 MiB |
| Follow-up 128K | 13.25 | 59.5% (209/351) | 54.2% | 95.86% | 1,177 MiB |
| Follow-up 250K | 16.08 | 59.3% (214/361) | 57.2% | 94.66% | 1,498 MiB |

The engine's lines for the long runs, as logged:

```text
strata serve: prompt 128000 tokens = 0 reused + 128000 read in 225806 ms (566.9 tok/s), 400 generated in 11727 ms (34.1 tok/s), drafts accepted 195 of 350, 6 checkpoints
strata serve: prompt 250000 tokens = 0 reused + 250000 read in 535798 ms (466.6 tok/s), 400 generated in 12571 ms (31.8 tok/s), drafts accepted 195 of 365, 6 checkpoints
strata serve: decode expert cache hit rate: 54.9% (135616 hits / 247239 lookups)
strata serve: KV streaming: 94.25% of 3561648 block reads hit VRAM, 824.9 MiB read from RAM
strata serve: prompt 250463 tokens = 249993 reused + 470 read in 3556 ms (132.2 tok/s), 400 generated in 11568 ms (34.6 tok/s), drafts accepted 214 of 361, 6 checkpoints
```

The complete lines for every request are in its JSON file. Per run:
[runs.csv](runs.csv); aggregates: [summary.json](summary.json).

**Follow-up reuse:** at 128K, the follow-up reused the entire previous prompt plus
the previous answer (128,000 + 400 = 128,400) and read only the 63 new tokens. At
250K, it reused 249,993 tokens and read 470. That is 7 tokens of the original
prompt, the 400-token answer and the new turn, so the previous answer was not reused
in that case. Both runs are recorded as observed; this report does not investigate
the difference.

**Memory during the runs** (sampled once per second; peaks over all measurements):

- GPU: **10,525 MiB of 11,264 MiB** used, constant from the first to the last
  sample. The engine allocates everything at startup (`421 MiB of VRAM free with
  everything loaded`), and VRAM use does not grow with prompt length.
- `strata` RSS: 52.3 GiB at 4K, rising to **52.9 GiB** at 128K and 250K. That
  includes the expert arena (`loaded 46.84 GiB`) and 3.09 GiB of pinned KV.
  `strata-vision` RSS: about 1.0 GiB.
- Host RAM used (`MemTotal - MemAvailable`, whole system): 62.7-65.6 GiB. The
  remaining RAM was mostly page cache (the PLE table is read from the mapped
  second GGUF shard).
- Swap: 8.0 of 8.0 GiB occupied throughout, with no swap I/O seen during a prompt
  read (see Hardware). No out-of-memory failure.

### Image questions (CPU encoder)

| Run | Expected | Answer | Prompt tokens | Prompt read | Client TTFT | Approx. encode (TTFT - prompt_ms) |
| --- | --- | --- | ---: | ---: | ---: | ---: |
| 1 | `KAZ-14; 2; 4` | `KAZ-14; 2; 4` | 441 | 3.44 s | 5.02 s | 1.58 s |
| 2 | `NCC-91; 3; 4` | `NCC-91; 3; 4` | 441 | 3.43 s | 5.08 s | 1.65 s |
| 3 | `JTT-57; 4; 3` | `JTT-57; 4; 3` | 441 | 3.39 s | 4.95 s | 1.56 s |

All three are exact. Each image was new, so each was encoded fresh by
`strata-vision` on 24 CPU threads. The encode estimate also contains HTTP and
tokenization time. The images are in [data/](data/).

### Recall: tools/needle_bench.py

`python tools/needle_bench.py --url http://127.0.0.1:8080 --api-key <redacted> --lengths 32k,128k --depths 10,50,90`
found **6 of 6** code words, each answer exactly the code word
([data/needles.json](data/needles.json); engine lines in
[data/needles-run.json](data/needles-run.json)).

| Length | Depth | Prompt tokens | Reused | Seconds | Found |
| --- | ---: | ---: | ---: | ---: | --- |
| 32k | 10% | 31,075 | 0 | 50.5 | yes |
| 32k | 50% | 31,075 | 0 | 50.2 | yes |
| 32k | 90% | 31,076 | 0 | 50.3 | yes |
| 128k | 10% | 122,151 | 0 | 214.7 | yes |
| 128k | 50% | 122,149 | 0 | 214.9 | yes |
| 128k | 90% | 122,149 | 55,296 | 129.9 | yes |

The 128k/90% case reused the first 55,296 tokens, the part of the text it shares
with the 50% case, so its time is not a fresh read. The code words at 40% depth in
the 128K and 250K speed runs were also found, by the follow-up turns
(`lantern-falcon-213` at 128K and `glacier-falcon-113` at 250K).

### Note: GPU image encoder vs CPU image encoder (earlier measurements, same day)

Earlier on 2026-10-02, before this report, the same engine 0.1.33 build and model
were run on this PC with `strata-vision` on the GPU. The resident GPU encoder takes
VRAM away from the engine on this 11 GB card, so `--prefill auto` chose smaller
prompt chunks and the expert cache shrank. Fresh reads of 26K-33K-token prompts
**fell to about 243 tok/s**. These numbers are earlier measurements, not repeated
for this report. Only the current configuration (CPU encoder, 256K) was re-measured
above. Log excerpts: [earlier-encoder-comparison.log](earlier-encoder-comparison.log),
with original log line numbers.

| Encoder | Context, KV streaming | Expert cache | Prompt chunk | Fresh prompt read |
| --- | --- | --- | --- | --- |
| GPU | 131,072, 32,768 resident | 1,169 slots, 2.26 GiB | 1,024 | 33,038 tokens: 243.3 tok/s; 25,918: 242.8; 33,046: 242.5 |
| GPU | 131,072, KV not streamed (no KV-streaming line in that start) | 509 slots, 0.99 GiB | 512 | 25,918: 162.6 tok/s; 33,046: 318.3 |
| CPU | 131,072, 32,768 resident | 1,811 slots, 3.48 GiB | 4,096 | 25,918: 750.0 tok/s; 33,046: 774.5 |
| CPU | 262,144, 32,768 resident | 1,694 slots, 3.26 GiB | 3,072 | 25,920: 621.9 tok/s; 33,048: 634.4 |

The last row is the configuration measured in this report. A restart later gave
1,692 slots / 3.25 GiB, and its 32,768-token reads (636.9 tok/s median) agree with
the earlier 622-634 tok/s. In this setup the CPU encoder costs about 1.6 s per new
768x512 image (above). The GPU encoder would cost the engine more than half its
prompt-read speed for every request.

## Correctness and limitations

- **Checks:**
  - Recall: 6 of 6 needles found at 31K and 122K tokens, plus 2 of 2 code words
    found at 40% depth after 128K and 250K prompts.
  - Images: 3 of 3 exact answers.
  - Every speed request produced 400 tokens of coherent English. Outputs are stored
    per run.
  - These checks measure recall and reading on these inputs, not general model
    quality.
- **Single runs:** the 250K fresh read and both follow-up turns ran once each. The
  250K run takes about 9 minutes.
- **Not measured:**
  - Sampled decoding (temperature > 0) and reasoning on.
  - Tool calls, coding tasks and long outputs (over 400 tokens).
  - Concurrency, a sustained thermal run and the Anthropic endpoint.
  - The low-RAM mode, other quantizations, and a comparison against another engine.
- **Same-session state:** speed runs, recall and image checks all ran on the same
  loaded engine without restarts. OS page cache was not dropped.
- **Shared service:**
  - The server is the PC's live service. Its configuration was not tuned or changed
    for this report.
  - It runs `main` plus the #430 server fix (PR #436), not a tagged release.
  - CPU-side services (Docker, an agent gateway) were running.
- **Memory peaks:** values are 1 s samples. Brief spikes between samples may be
  missed.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://zgga.wtpuscm.cn/fenxi/terms-879630.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://stwn.wtpuscm.cn/baogao/home-082718.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://sabl.wtpuscm.cn/suanfa/development-364234.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://bnzr.wtpuscm.cn/gongxiang/discount-579784.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://xezr.wtpuscm.cn/guanjianci/metric-213170.html)
* [面向大规模网络的 Strata 工业级架构基准](https://json.wtpuscm.cn/suanfa/accessibility-669889.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://kjpo.wtpuscm.cn/jiaocheng/recommendation-399662.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://oxzd.wtpuscm.cn/shichang/device-266.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://rwtf.wtpuscm.cn/wenzhang/partner-449327.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://fnfb.wtpuscm.cn/yanjiu/tactic-765952.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://zlha.wtpuscm.cn/zhinan/design-245769.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://pmil.wtpuscm.cn/shichang/status-109274.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://gjsg.wtpuscm.cn/youhua/wellness-866423.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://ktcy.wtpuscm.cn/shichang/management-796855.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://asbu.wtpuscm.cn/yunsuan/reminder-806232.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://uohv.wtpuscm.cn/shuju/excellence-199052.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://jqdh.wtpuscm.cn/kaifa/rating-768480.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://iwvv.wtpuscm.cn/fuwu/app-786740.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://bgbb.wtpuscm.cn/huodong/education-261882.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://oaxg.wtpuscm.cn/jishu/trading-651338.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://remw.wtpuscm.cn/jiaoliu/music-276709.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://wtzd.wtpuscm.cn/suanfa/unsubscribe-986513.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://mdkk.wtpuscm.cn/qiye/support-440898.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://cvme.tcti.cn/chanpin/url-69124615.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://rpfi.tcti.cn/pingce/forecast-41803813.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://dfws.tcti.cn/pingce/feedback-27374418.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://asug.tcti.cn/kaifa/saving-69579617.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://rafc.tcti.cn/paiming/entertainment-50581246.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://qwia.tcti.cn/yingxiao/user-19290970.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://vzmr.tcti.cn/jianzhan/topic-19531206.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://fbms.tcti.cn/baogao/label-93095708.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://cpcm.tcti.cn/peixun/goal-99569456.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://wisb.tcti.cn/jianzhan/vacation-91303437.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://lvff.tcti.cn/wenzhang/efficiency-01412846.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://fqha.tcti.cn/ziyuan/tracking-65918658.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://emxy.tcti.cn/kaifa/milestone-51599211.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://kiah.tcti.cn/wendang/food-75171772.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://qbmr.tcti.cn/qiye/trading-07256872.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://cgyf.tcti.cn/shangye/guide-59916432.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://vibo.tcti.cn/shichang/terms-63682242.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://jeaq.wtpuscm.cn/hezuo/extension-519791.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/sheji/account-64425246.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/4438)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/chuangxin/goal-22219917.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://hzdn.tcti.cn/wangluo/personalization-59885537.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://kxbl.tcti.cn/youhua/efficiency-14313304.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://ogsp.wtpuscm.cn/zhinan/expensive-867092.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://apkz.wtpuscm.cn/paiming/automation-352234.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://scxs.wtpuscm.cn/shuju/careers-668948.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://qynq.wtpuscm.cn/yunsuan/browser-858694.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://wtjh.wtpuscm.cn/zhizhu/value-735712.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://kbfm.wtpuscm.cn/hezuo/excellence-357874.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://xews.wtpuscm.cn/yunsuan/reminder-076849.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://hkyp.wtpuscm.cn/xitong/products-252.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://chnj.wtpuscm.cn/zhizhu/development-742219.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://rqqo.wtpuscm.cn/gongju/strategy-890921.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://smdi.wtpuscm.cn/jishu/audience-560473.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://shas.wtpuscm.cn/guanjianci/design-107368.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://zdhc.wtpuscm.cn/yingyong/brand-126521.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://ndzl.wtpuscm.cn/youhua/update-208708.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://axij.wtpuscm.cn/wendang/trading-038162.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://qtvi.wtpuscm.cn/sheji/policy-637848.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://gebo.wtpuscm.cn/ziyuan/trading-286334.html)
* [Strata 异步中间件流水线与 any 接入规范](https://niqn.wtpuscm.cn/jishu/collaborate-987243.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://bhtv.wtpuscm.cn/fuwu/discovery-595343.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://scmy.wtpuscm.cn/jishu/economy-228521.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://rinp.wtpuscm.cn/xuexi/tool-554396.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://xoht.wtpuscm.cn/gongsi/rating-228318.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://xwhv.wtpuscm.cn/yingyong/server-511878.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://nkvm.tcti.cn/shangye/extension-87682723.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://rkih.tcti.cn/jiaoliu/growth-21290781.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://qsmt.tcti.cn/gongxiang/presentation-53997960.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://yzsc.tcti.cn/suanfa/extension-96920821.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://xvcu.tcti.cn/paiming/global-43833958.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://jznr.tcti.cn/huodong/analysis-16167769.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://uacx.tcti.cn/yunsuan/profile-73349153.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://tdts.tcti.cn/peixun/upload-03465144.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://epdj.tcti.cn/pingtai/calculator-26838274.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://wnwi.tcti.cn/zhinan/network-06782132.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://leot.tcti.cn/zhineng/share-80364905.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://pqbt.tcti.cn/xitong/ebook-02313279.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://lgut.tcti.cn/xuexi/course-26599733.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://bdgm.tcti.cn/jiaoliu/growth-34989898.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://smwb.tcti.cn/shichang/dashboard-58632561.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://cfzq.tcti.cn/keji/wellness-70673129.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://paob.tcti.cn/anfang/local-83734891.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://xxeh.wtpuscm.cn/xitong/budget-083695.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/yingyong/luxury-70236855.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/wiki/92628)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/kuangjia/api-47856208.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://qyjx.tcti.cn/zhineng/machine-88328428.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://ztcv.tcti.cn/ziyuan/site-89833432.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://qwhc.wtpuscm.cn/ziyuan/economy-339947.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://dbsw.wtpuscm.cn/shuju/section-673697.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://syxy.wtpuscm.cn/paiming/objective-606372.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://mnzx.wtpuscm.cn/youhua/software-487887.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://nokk.wtpuscm.cn/gongxiang/terms-194266.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://dnic.wtpuscm.cn/ziyuan/interface-311997.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://mswf.wtpuscm.cn/zhizhu/resource-599795.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://cuvk.wtpuscm.cn/shichang/technology-151.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://ejea.wtpuscm.cn/gongsi/whitepaper-283751.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://dvca.wtpuscm.cn/anli/lead-083455.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://cbtt.wtpuscm.cn/ziyuan/experience-827743.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://ciqg.wtpuscm.cn/pingtai/backup-421224.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://aczt.wtpuscm.cn/xuexi/comment-312995.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://aggy.wtpuscm.cn/yanjiu/accessibility-364535.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://axol.wtpuscm.cn/wangluo/server-319464.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://kcwj.wtpuscm.cn/yanjiu/widget-038970.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://sxqw.wtpuscm.cn/peixun/tool-907847.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://wkgd.wtpuscm.cn/yunying/webinar-586075.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://xbkw.wtpuscm.cn/xuexi/backup-075681.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://cyzo.wtpuscm.cn/pingce/food-853066.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://byhn.wtpuscm.cn/shangye/profile-019895.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://xkfk.wtpuscm.cn/shichang/collaboration-024385.html)
* [Strata 故障自愈与网络拓扑重构实践](https://iypv.wtpuscm.cn/anli/market-426315.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://xdwy.tcti.cn/guanjianci/collaborate-46332802.html)
* [Strata 权威网络权重传递与收录基准规范](https://ktvn.tcti.cn/guanjianci/user-40538919.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://bpaz.tcti.cn/xitong/software-67380163.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://kdhl.tcti.cn/jiaoliu/resource-14483024.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://cevi.tcti.cn/paiming/management-44516529.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://qwsp.tcti.cn/gongju/privacy-19162529.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://gmxa.tcti.cn/tuiguang/browser-21538615.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://rqkp.tcti.cn/ziyuan/campaign-47859376.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://boby.tcti.cn/zixun/hotel-76560381.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://rode.tcti.cn/yunsuan/game-25146551.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://bnkv.tcti.cn/yanjiu/landing-33154510.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://mwwa.tcti.cn/wangluo/marketing-76392998.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://csth.tcti.cn/zhineng/module-49415165.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://futf.tcti.cn/zhizhu/game-28265740.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://vjwg.tcti.cn/hezuo/device-45686744.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://bqrs.tcti.cn/gongju/api-39685138.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://ytzo.tcti.cn/peixun/file-60490447.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://mvyz.wtpuscm.cn/anli/register-791979.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/pingce/unsubscribe-69879567.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/news/38166)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/guanjianci/photo-89427154.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://grec.tcti.cn/baogao/deal-01171194.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://vnad.tcti.cn/yingyong/achievement-94384070.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://fvoz.wtpuscm.cn/shuju/goal-247890.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://fexm.wtpuscm.cn/kuangjia/conference-950313.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://zirr.wtpuscm.cn/pingce/food-401305.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://vdfc.wtpuscm.cn/shuju/layout-366054.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://amgu.wtpuscm.cn/yunying/plugin-847569.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://rans.wtpuscm.cn/shuju/value-794785.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://alaz.wtpuscm.cn/gongxiang/progress-232003.html)
* [Strata: Windows 分布式架构 (Verified)](https://rwxz.wtpuscm.cn/jianzhan/advertising-888.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://orow.wtpuscm.cn/zixun/movie-987314.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://dgoc.wtpuscm.cn/zhinan/kpi-689967.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://utcz.wtpuscm.cn/shangye/file-465885.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://dved.wtpuscm.cn/kuangjia/course-445135.html)

</details>

