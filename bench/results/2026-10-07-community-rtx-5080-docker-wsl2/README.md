# Community benchmark: RTX 5080 in Docker Desktop on Windows (WSL2)

Measured on 2026-10-07 by [xWinIcex](https://www.yx-sf.com/tech/26914), on a Windows 10 host
using the repository's own Dockerfile and entrypoint — no bare-metal Linux, no
`setup.sh`, no hand-written launch command. Coder IQ1_M, one RTX 5080, two context
limits (131,072 and 65,536) on the same loaded model.

Median decode throughput was **75.7–80.8 tok/s at 4,096–32,768 prompt tokens and 72.2
tok/s at 130,560** with the 131,072-token limit, and **71.5–78.5 tok/s at 4,096–32,768
and 73.8 at 65,024** with the 65,536-token limit. These are synthetic code-explanation
requests with greedy decoding and a 256-token output cap; they do not establish general
answer quality. The main limitation is the host: everything ran inside Docker Desktop's
WSL2 VM on Windows, which changes both the memory the installer sees and the storage path.

This is, as far as we could tell from the existing reports, the first report from that
environment, so three things that are specific to it are written up in
[Notes for users of this environment](#notes-for-users-of-this-environment).

## Hardware and software

- NVIDIA GeForce RTX 5080; 16,303 MiB reported VRAM; power limit 360 W (maximum 380 W).
  PCIe link **Gen 5, x8**, while the card reports a maximum width of x16. The engine's own
  startup probe measured **28.9 GB/s host-to-device** (`pcie_frac 0.55`, the default).
  Clocks were not fixed and no thermal soak was run; the highest GPU temperature observed
  was 68 °C and the highest power 273 W.
- AMD Ryzen 9 9950X, 16 cores / 32 threads, AVX-512. The engine chose the AVX-512 path,
  48 pool tasks per phase, 15 expert-pool workers plus the host thread on core 0.
- 64 GB installed RAM; the container sees **54.9 GiB** (see the memory note below).
  Storage: `C:` XPG MARS 980 PRO 1908 GB NVMe SSD — the WSL2 VHDX, and therefore the
  image, the model files and `/data` all live there. The machine's other disk is a
  3,726 GB HDD and was not used.
- Windows 10 Pro 26H2 build 26300; WSL 2.7.10.0, kernel 6.18.33.2-2; Docker Desktop
  4.84.0 (client and server 29.6.2, buildx v0.35.0-desktop.2); NVIDIA driver 617.14.
- Strata commit `d5ea7133741e67743c0e886bb426c0ce8d69cf6c` ("Version 0.1.40.3"); engine
  **0.1.40.3**, `source=local`, CUDA architectures `[120]`, vision compiled for CUDA —
  [BUILD.json](BUILD.json).
- CUDA 13.0.48 (`nvcc` V13.0.48); GCC 13.3.0; Python 3.12.3. Installed Python packages:
  [python-packages.txt](python-packages.txt).
- The whole environment is in [environment.txt](environment.txt), including the exact
  `docker run` lines, the WSL2 memory setting and the base image digest.

## Model and configuration

Model: `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-Coder-GGUF` at the revision the installer
pins, `5348543e0147355ac9cbcb031184a3546350988e` (2026-09-29):

- `IQ1_M/Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00001-of-00002.gguf` — 29,608,446,496 B
- `IQ1_M/Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00002-of-00002.gguf` — 28,800,138,432 B
- `mmproj-Qwen3.8-Flash-Next-BF16.gguf` — 907,543,008 B

All three local SHA-256 hashes matched the repository's published LFS oids
([sha256.txt](sha256.txt), [model-provenance.json](model-provenance.json)). The MTP draft
layer was fetched from `Qwen/Qwen3.8-Flash-Next` by the installer (31 tensors, 5.214 GB,
per-tensor hashes in [mtp-manifest.json](mtp-manifest.json)); the full BF16 shards were not
downloaded. The unmodified installer prepared the pack (`/data/packs/coder-iq1_m`, 1.4 GB
on disk) and the Q2_0 MTP experts.

Both configurations were produced by the image's own entrypoint from the same
`strata-data` volume; only `CONTEXT` and `REINSTALL` differ between them. The recorded
engine configuration is [config-131072.json](config-131072.json) and
[config-65536.json](config-65536.json):

```text
# 131,072-token limit
docker run -d --name strata --gpus all --network strata-net \
  -p 127.0.0.1:8080:8080 --ulimit memlock=-1 --stop-timeout 60 \
  -v strata-data:/data \
  -e FAMILY=coder -e MODEL=IQ1_M -e CONTEXT=131072 -e VISION=cpu -e GPU=0 strata

# 65,536-token limit, same volume
  ... -e CONTEXT=65536 -e VISION=cpu -e GPU=0 -e REINSTALL=1 strata
```

Settings both configurations share: `--expert-cache auto`, `--prefill auto`, `--kv int8`,
`--spec 4 --spec-min-p 0.5`, MTP on, `--vram-reserve-mib 700`, low-RAM mode off, the
experimental speed projection off, no calibration, no custom expert profile, temperature 0,
`reasoning_effort: none`, 256 generated tokens per speed run. Vision is on with the encoder
on the **CPU** (`VISION=cpu`, 16 threads, 300 image tokens), which is the first thing worth
noting: nothing about this configuration was tuned by hand.

What differs, from the engine's startup log:

| | 131,072 limit | 65,536 limit |
| --- | --- | --- |
| Expert cache | 3,554 slots, 6.78 GiB VRAM | 4,060 slots, 7.73 GiB VRAM |
| Free VRAM at load | 152 MiB | 178 MiB |
| KV | INT8 | INT8, streaming off, resident in VRAM |
| Prefill chunk | 8,192 tokens, borrows 2,172 cache slots | 8,192 tokens |

At 131,072 the engine warned about its own configuration:

```text
strata serve: 152 MiB of VRAM free with everything loaded - LOW: requests may stall;
add --vram-reserve-mib 1060 to the config's args (or lower --max-context)
```

So a 16 GB card at a 128K context leaves almost nothing spare, and the engine says so
itself. Nothing below failed because of it: no request was refused for VRAM and none
stalled, at either limit, including the 130,560-token prompt.

## Method

The benchmark is the script from
[bench/results/2026-09-30-community-rtx-5090/benchmark.py](../2026-09-30-community-rtx-5090/benchmark.py),
attached here as [benchmark.py](benchmark.py), **unchanged except for four defaults**
(`--root /opt/strata`, `--pack /data/packs/coder-iq1_m`, `--url http://127.0.0.1:8080`,
`--out /data/bench`), so the numbers stay comparable with that report. It builds
deterministic synthetic Python functions, puts a different nonce near the start of every
request so no prompt prefix can be reused, counts the complete rendered chat prompt with
Strata's own tokenizer, and verifies its count against the engine afterwards.

One warm-up request was excluded. Three runs were executed at each length, serially, in
increasing-length order, on the same loaded engine. Every measured request read its entire
prompt: **zero reused tokens** in all 33 runs (`reused: 0`, `prompt_read == prompt_total`).
Loading time is not included. Both context limits were measured in their own session on a
freshly loaded model; see the limitations.

Streaming TTFT is measured from immediately before the HTTP request until the first
non-empty text delta, ignoring empty role chunks; with reasoning off that delta is answer
text. Total latency ends at the end of the stream; both include HTTP and front-end
tokenization on loopback. Prompt throughput is fresh tokens divided by the engine's
`prompt_ms`; decode throughput is `engine_generated / decode_ms`. Per-run records — the
engine's timings and draft statistics, the client's TTFT and total latency, the output
text and a SHA-256 of each request body — are in `data-131072/results.json` and
`data-65536/results.json`. The request bodies themselves are not committed because
`benchmark.py` rebuilds them deterministically; compare the recorded `request_sha256`
after a re-run to confirm it reproduced the same prompt.

Memory was sampled once per second from the engine's own `/metrics` by
[monitor.py](monitor.py) and summarised by
[summarize-telemetry.py](summarize-telemetry.py). Every memory number in this report comes
from the engine's view, not from `nvidia-smi`, which inside a container reports the whole
GPU and not the container's share.

## Results

Each cell is the median **[minimum–maximum]** of three runs. Every request generated 256
tokens and stopped at the output limit; there were no failed or cancelled speed requests.

**Context limit 131,072** (`data-131072/`, session of 2026-10-07 ~22:22):

| Prompt tokens | Reused | Prompt tok/s | Decode tok/s | TTFT seconds | Total seconds |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 2,252.8 [2,138.8–2,267.5] | 75.7 [72.6–77.9] | 1.849 [1.839–1.948] | 5.306 [5.100–5.350] |
| 8,192 | 0 | 3,009.2 [2,998.9–3,063.1] | 79.9 [78.9–80.8] | 2.772 [2.706–2.780] | 5.929 [5.913–5.940] |
| 16,384 | 0 | 3,197.3 [3,184.2–3,304.9] | 79.1 [77.6–81.0] | 5.166 [5.002–5.189] | 8.402 [8.136–8.442] |
| 32,768 | 0 | 3,304.3 [3,271.8–3,311.9] | 80.8 [75.6–82.4] | 9.981 [9.960–10.081] | 13.223 [13.057–13.316] |
| 65,536 | 0 | 3,299.4 [3,269.5–3,310.9] | 73.8 [73.8–77.5] | 19.958 [19.884–20.137] | 23.400 [23.166–23.584] |
| 130,560 | 0 | 3,076.5 [2,942.9–3,200.6] | 72.2 [71.0–76.1] | 42.612 [40.959–44.532] | 46.131 [44.292–48.114] |

**Context limit 65,536** (`data-65536/`, session of 2026-10-07 ~22:37):

| Prompt tokens | Reused | Prompt tok/s | Decode tok/s | TTFT seconds | Total seconds |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 2,194.1 [2,113.8–2,221.6] | 73.1 [72.1–78.2] | 1.904 [1.882–1.970] | 5.379 [5.127–5.495] |
| 8,192 | 0 | 2,781.6 [2,760.3–2,804.9] | 76.4 [75.2–76.8] | 2.981 [2.963–3.010] | 6.332 [6.288–6.336] |
| 16,384 | 0 | 3,001.3 [2,997.2–3,016.8] | 78.5 [68.4–78.6] | 5.523 [5.480–5.539] | 8.741 [8.707–9.241] |
| 32,768 | 0 | 3,034.9 [3,016.7–3,084.7] | 71.5 [70.3–73.6] | 10.873 [10.702–10.945] | 14.419 [14.148–14.550] |
| 65,024 | 0 | 3,027.7 [3,003.6–3,197.0] | 73.8 [73.7–79.9] | 21.582 [20.441–21.762] | 25.028 [23.619–25.203] |

The two limits are **not** in the expected order: at every shared prompt length the
131,072-token configuration measured equal or slightly faster (decode 80.8 vs 71.5 tok/s at
32,768; prefill 3,304 vs 3,035 tok/s), even though it has the *smaller* expert cache
(3,554 vs 4,060 experts, 6.78 vs 7.73 GiB of VRAM) and only 152 MiB of free VRAM. Decode
ranges at 32,768 do not overlap (128K: 75.6–82.4; 64K: 70.3–73.6), so this is not obviously
run-to-run noise, but with one session per configuration and no interleaving we did **not**
isolate the cause and do not claim that a smaller context is slower in general. It is
reported because it is what was measured, and because the cache sizes alone do not predict it.

Decode expert-cache hit rates were 81.0–87.0% across the 33 measured runs (`hit_rate` in
`results.json`); draft acceptance was 140–162 of the 196–237 drafts offered.

### Memory

From the one-second samples (peaks and ranges; a sampled peak can miss a shorter spike):

| | 131,072 limit | 65,536 limit |
| --- | ---: | ---: |
| GPU memory used, peak (of 15.92 GiB) | 15.82 GiB | 15.74 GiB |
| Host RAM used, peak (of 54.92 GiB) | 29.77 GiB | 29.49 GiB |
| Free VRAM as the engine reports it | 152 MiB | 178 MiB |
| GPU temperature, peak | 68 °C | 65 °C |
| GPU power, peak (limit 360 W) | 267 W | 268 W |
| Disk read, peak | 230 MB/s (recall run) / 3.7 MB/s (speed run) | 3.6 MB/s |

`gpu_mem_used` comes from the engine's `/metrics`, so it is the process-plus-context
allocation rather than the card's total. The 230 MB/s peak was recorded during the 128K
recall run: on very long prompts the experts that the prompt path's borrowed cache slots do
not cover are read from the SSD, which is the behaviour the engine documents.

## Correctness

The repository's unchanged `tools/needle_bench.py` found **all nine needles** on the
131,072-token configuration — depths 10%, 50% and 90% at lengths 32k, 64k and 128k. Actual
prompt lengths were 32,343–32,344, 62,625–62,627 and 125,919–125,920 tokens; every answer
matched the expected code word. On the 65,536-token configuration the same command found
**6 of 6** (32k and 64k; 128k skipped because the server's context is 65,536), and the skip
is recorded in the output rather than silently dropped. See
[data-131072/needles.json](data-131072/needles.json) and
[data-65536/needles.json](data-65536/needles.json).

The vision path was checked with [image-check.py](image-check.py), which draws a code word
with Pillow and sends it as an OpenAI `image_url` data URL. With the encoder on the CPU the
model answered `K7QX-4291` for the drawn `K7QX-4291` in 1.1 s at 261 prompt tokens
([data-131072/image-check.json](data-131072/image-check.json),
[image-check.png](data-131072/image-check.png)). The encoder falling back to the CPU is
visible in the log as `ggml_cuda_init: failed to initialize CUDA` followed by
`strata-vision: on the CPU, 16 threads`, which is what `VISION=cpu` asks for.

## Notes for users of this environment

1. **Docker Desktop's default RAM makes the installer choose the low-RAM mode.** With no
   `%UserProfile%\.wslconfig`, Docker Desktop caps the WSL2 VM at 50% of host RAM, so the
   container saw ~31 GB of the host's 64 GB. `setup.py` reads RAM from `/proc/meminfo`, so it
   did not error — it selected the low-RAM mode ("the GPU holds ~38% of IQ1_M's experts, the
   other ~15 GB stay in RAM, read once from a copy in the model folder"). A 32 GB PC is a
   documented low-RAM case, but on a 64 GB PC it is silent and unwanted. Adding
   `[wsl2]` / `memory=56GB` ([wslconfig.txt](wslconfig.txt)) and restarting
   (`wsl --shutdown`) gives the container 54.9 GiB and the normal mode. Both measured runs
   used the normal mode.
2. **A request needs 8 tokens of headroom.** `prompt + max_tokens + 8 ≤ max_context`, and
   the engine does not truncate. A 65,280-token prompt with `max_tokens: 256` on the
   65,536-token limit was refused with HTTP 400: *"prompt (65280 tokens) + max tokens (256)
   exceeds the context (65536); requests are never truncated. Send a smaller max_tokens (at
   most 248 here)"*. [probe-request.py](probe-request.py) reproduces it. The largest prompt
   measured here therefore leaves 512 tokens: 130,560 of 131,072 and 65,024 of 65,536.
3. **The port is published on `127.0.0.1` only**, so no API key was set
   (`"api_key": false` in `/health`). Other containers reach the same server over a
   user-defined Docker network at `http://strata:8080`; nothing was exposed to the LAN.

## Limitations

- **One session per configuration.** The warm-up state differs between them, and the
  128K-vs-64K comparison in the results is a comparison of two sessions, not of two
  interleaved settings. The engine's own log shows CUDA-graph capture
  (`strata verify: capturing the N-token window`) happening during the first requests of a
  session, which is one candidate explanation for the ordering; it was not tested. Treat the
  difference as an observation to reproduce, not a result.
- **One machine, one quantization, one GPU, one synthetic workload.** Long output, sampled
  decoding, thinking, multi-request concurrency (`parallel`), tool use and a sustained
  thermal run were not evaluated. The needle and image checks measure recall and reading on
  those specific inputs, not general model quality.
- **The PCIe link ran at x8 of a possible x16**, and the card reports a maximum width of
  x16. The engine measured 28.9 GB/s host-to-device; the RTX 5090 report on a Gen5 x16 link
  measured 50.0 GB/s. Whether the narrower link costs anything here was not tested — the
  link is a property of this machine's slot wiring, so absolute numbers may move on a host
  that runs the card at x16.
- **Windows and WSL2 are between Strata and the hardware.** File I/O and page-locked memory
  go through the WSL2 VM; `--ulimit memlock=-1` was passed and the engine reported
  `MAP_HUGETLB unavailable` with transparent huge pages requested instead. Numbers from a
  bare-metal Linux host with the same card are not guaranteed to match.
- **Timings include the whole client path** (HTTP, front-end tokenization, loopback) and are
  measured at the client, not inside the engine.
- The 131,072-token configuration logged the engine's own VRAM warning quoted above. It did
  not cause a failure in any measured run, but it is a real margin of 152 MiB and a reason
  to prefer `--vram-reserve-mib 1060` (or a smaller context) for unattended use.
- One telemetry file, `data-131072/telemetry-speed-overlapped.jsonl`, was written by two
  samplers that overlapped for about eleven minutes (819 of 820 lines parse; the timestamps
  contain one discontinuity). Only peaks and ranges are taken from it. The clean runs are
  `data-131072/telemetry-checks.jsonl` and `data-65536/telemetry-speed.jsonl`.

## Files

| File | What it is |
| --- | --- |
| `benchmark.py`, `monitor.py`, `summarize-telemetry.py`, `image-check.py`, `probe-request.py` | the scripts, runnable as documented above |
| `data-131072/`, `data-65536/` | `results.json` (per-run engine timings, draft statistics, output text, request hashes) and `summary.json`, the telemetry and its memory summary, `needles.json`, the image check, and the engine's status at startup |
| `config-131072.json`, `config-65536.json`, `engine-131072.log`, `engine-65536.log` | the recorded configuration and the unabridged engine log of each session |
| `environment.txt`, `python-packages.txt`, `wslconfig.txt` | host and container environment |
| `model-provenance.json`, `sha256.txt`, `mtp-manifest.json`, `mtp-inventory.md` | what was downloaded and its hashes |
| `BUILD.json` | the engine the image compiled |


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://okan.wtpuscm.cn/fenxi/url-880993.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://pczu.wtpuscm.cn/gongsi/machine-378733.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://djsl.wtpuscm.cn/shangye/landing-511776.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://vfej.wtpuscm.cn/qiye/webinar-168568.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://lffd.wtpuscm.cn/guanjianci/follow-581853.html)
* [面向大规模网络的 Strata 工业级架构基准](https://hwzp.wtpuscm.cn/jianzhan/design-548828.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://botu.wtpuscm.cn/wendang/change-840094.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://kcks.wtpuscm.cn/anli/plugin-302.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://vrxm.wtpuscm.cn/qiye/technology-383142.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://pvyy.wtpuscm.cn/yinqing/register-542066.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://edmy.wtpuscm.cn/yinqing/loyalty-091078.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://afcd.wtpuscm.cn/guanjianci/sales-647351.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://xqll.wtpuscm.cn/gongsi/travel-165011.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://eemc.wtpuscm.cn/baogao/game-900593.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://cbxe.wtpuscm.cn/zhineng/deadline-869970.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://uwgs.wtpuscm.cn/youhua/support-583033.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://zykj.wtpuscm.cn/jiaocheng/news-058041.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://ilgv.wtpuscm.cn/kuangjia/page-703420.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://aaki.wtpuscm.cn/zixun/analysis-342143.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://ehzc.wtpuscm.cn/youhua/account-250585.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://xrjf.wtpuscm.cn/baogao/alert-253839.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://tfqg.wtpuscm.cn/fenxi/performance-964646.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://time.wtpuscm.cn/huodong/solution-833948.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://qbtq.tcti.cn/wendang/site-80753813.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://bbvz.tcti.cn/liuliang/luxury-50599865.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://zpdz.tcti.cn/kuangjia/resource-58599109.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://bejq.tcti.cn/zhineng/digital-11877139.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://zlch.tcti.cn/jianzhan/rating-72853751.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://kggm.tcti.cn/jiaoliu/subject-00402712.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://uzdd.tcti.cn/xinwen/article-06076325.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://aayg.tcti.cn/tuiguang/coupon-75288903.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://ljwp.tcti.cn/shuju/module-48974323.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://patc.tcti.cn/liuliang/affordable-02191520.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://hsrz.tcti.cn/yunying/analytics-18787072.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://zbrn.tcti.cn/peixun/sale-49520291.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://tqwz.tcti.cn/yinqing/cheap-15954515.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://xdak.tcti.cn/yanjiu/alliance-51728686.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://oane.tcti.cn/xuexi/income-12604238.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://xxsf.tcti.cn/hezuo/status-61952014.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://babz.tcti.cn/jiaocheng/api-46705509.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://ajcm.wtpuscm.cn/pingtai/restaurant-582629.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/gongsi/integration-67894117.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/tech/86303)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/jiaocheng/management-58585175.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://rqjx.tcti.cn/anli/revenue-24032171.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://umfb.tcti.cn/pingce/page-28386846.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://njgr.wtpuscm.cn/yinqing/price-283381.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://ctut.wtpuscm.cn/suanfa/follow-409297.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://ojcv.wtpuscm.cn/anli/health-913764.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://nxbm.wtpuscm.cn/youhua/whitepaper-788482.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://twig.wtpuscm.cn/huodong/accessibility-351751.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://cxbl.wtpuscm.cn/fuwu/unsubscribe-957202.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://pnzw.wtpuscm.cn/huodong/shopping-515577.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://gbww.wtpuscm.cn/xitong/coupon-053.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://paab.wtpuscm.cn/shuju/management-595591.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://sqlx.wtpuscm.cn/tuiguang/sync-014699.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://lmxy.wtpuscm.cn/yunsuan/demographic-797605.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://agbq.wtpuscm.cn/qiye/technology-909199.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://qqab.wtpuscm.cn/liuliang/products-274394.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://xspo.wtpuscm.cn/xitong/notification-221619.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://eexm.wtpuscm.cn/zhinan/message-522219.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://kgon.wtpuscm.cn/paiming/traffic-202691.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://yuxz.wtpuscm.cn/guanjianci/topic-092450.html)
* [Strata 异步中间件流水线与 any 接入规范](https://uwuf.wtpuscm.cn/baogao/software-459742.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://agoz.wtpuscm.cn/guanjianci/expensive-246592.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://hgvf.wtpuscm.cn/paiming/webinar-491249.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://alwr.wtpuscm.cn/kuangjia/video-469794.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://hpjx.wtpuscm.cn/paiming/conversion-805236.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://wrgd.wtpuscm.cn/yanjiu/education-686835.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://axpp.tcti.cn/liuliang/ai-71903582.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://nrlo.tcti.cn/shuju/global-69560481.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://pvjm.tcti.cn/jiaoliu/restore-66801441.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://nnab.tcti.cn/anfang/kpi-54523148.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://pyci.tcti.cn/yunsuan/products-36033085.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://umes.tcti.cn/kuangjia/expense-47966844.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://dhkg.tcti.cn/sheji/course-49307046.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://hqsj.tcti.cn/guanjianci/budget-77072320.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://fuva.tcti.cn/shichang/collaborate-74891201.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://mdcu.tcti.cn/ziyuan/value-81995369.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://cnak.tcti.cn/jiaocheng/login-24706204.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://oyca.tcti.cn/pingtai/performance-33685864.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://ltpy.tcti.cn/zhinan/home-11537214.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://osbv.tcti.cn/anli/vacation-16651363.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://ryha.tcti.cn/keji/deadline-46813800.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://cspd.tcti.cn/qiye/demographic-28258913.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://orcp.tcti.cn/liuliang/online-68327540.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://liwr.wtpuscm.cn/zhizhu/section-679355.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/shichang/collaborate-68941440.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/27768)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/xuexi/change-72402006.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://drgb.tcti.cn/huodong/segment-58518245.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://iwph.tcti.cn/youhua/presentation-43774225.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://uxpe.wtpuscm.cn/kaifa/premium-243077.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://stan.wtpuscm.cn/kaifa/url-572010.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://chaf.wtpuscm.cn/zixun/reminder-129341.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://fxrm.wtpuscm.cn/yunsuan/development-760969.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://tuoy.wtpuscm.cn/chanpin/recipe-395020.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://gtph.wtpuscm.cn/fenxi/restaurant-737808.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://ntew.wtpuscm.cn/jiaocheng/page-253701.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://oask.wtpuscm.cn/suanfa/landing-121.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://pjea.wtpuscm.cn/wangluo/metric-703701.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://libz.wtpuscm.cn/zhizhu/audience-602607.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://joxb.wtpuscm.cn/shichang/change-453894.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://ugvo.wtpuscm.cn/yunying/economy-865478.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://cjlj.wtpuscm.cn/jiaocheng/seo-097813.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://sxyf.wtpuscm.cn/hezuo/faq-463451.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://ykgp.wtpuscm.cn/sheji/web-674502.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://rrbh.wtpuscm.cn/kuangjia/alliance-867627.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://lqnl.wtpuscm.cn/yunying/software-596269.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://qgyc.wtpuscm.cn/kuangjia/consulting-912944.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://qbhl.wtpuscm.cn/chanpin/about-415536.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://skju.wtpuscm.cn/yanjiu/economy-924435.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://jjoa.wtpuscm.cn/sheji/training-768393.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://fuua.wtpuscm.cn/liuliang/metric-131805.html)
* [Strata 故障自愈与网络拓扑重构实践](https://fgsx.wtpuscm.cn/youhua/advertising-695925.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://ogas.tcti.cn/fenxi/navigation-22406683.html)
* [Strata 权威网络权重传递与收录基准规范](https://ewqp.tcti.cn/huodong/trading-40627910.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://uony.tcti.cn/chuangxin/tag-78117665.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://cnfj.tcti.cn/gongsi/brand-23980173.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://rojf.tcti.cn/xuexi/keyword-37132287.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://dpsc.tcti.cn/chanpin/careers-25450771.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://cfwc.tcti.cn/suanfa/visitor-76042381.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://zcgj.tcti.cn/fenxi/dashboard-15090949.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://tafg.tcti.cn/huodong/team-82983239.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://smfi.tcti.cn/ziyuan/alert-49849535.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://pcwl.tcti.cn/keji/domain-19783358.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://wely.tcti.cn/suanfa/internet-68262565.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://hhim.tcti.cn/kaifa/promotion-58089188.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://obwk.tcti.cn/youhua/subject-15501116.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://hgtn.tcti.cn/xuexi/system-08536352.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://lrfz.tcti.cn/zhineng/upload-44728002.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://izmm.tcti.cn/jishu/partner-13618202.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://qasl.wtpuscm.cn/youhua/communication-462033.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/sheji/domain-37959926.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/tech/45857)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/keji/income-89399306.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://oszm.tcti.cn/shuju/cloud-82541317.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://gvht.tcti.cn/yunying/affordable-13275539.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://qtue.wtpuscm.cn/anfang/premium-676681.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://kgwh.wtpuscm.cn/jishu/document-899181.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://rbsn.wtpuscm.cn/kuangjia/hotel-901946.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://udhl.wtpuscm.cn/yunying/finance-332150.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://ydlw.wtpuscm.cn/qiye/case-190148.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://tupy.wtpuscm.cn/zhizhu/finance-417625.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://kmfp.wtpuscm.cn/huodong/review-191311.html)
* [Strata: Windows 分布式架构 (Verified)](https://adqc.wtpuscm.cn/gongsi/navigation-300.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://adkl.wtpuscm.cn/liuliang/forecast-990059.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://qpkd.wtpuscm.cn/zhizhu/feedback-731907.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://ddpr.wtpuscm.cn/kaifa/status-707447.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://nzrt.wtpuscm.cn/youhua/fashion-910011.html)

</details>

