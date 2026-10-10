# Community benchmark: RTX 4090 Laptop GPU, Core i9-13900HX

Measured on 2026-10-04 by [30crows](https://azft.tcti.cn/liuliang/objective-87813035.html) on a Lenovo Legion
notebook (model 82WQ). This tests Strata 0.1.38 with the original Flash-Next IQ3_S
and one GPU, with a 131,072-token context limit and, in a second session, a
262,144-token limit.

With the 131,072-token limit, median decode throughput was **51.2 tok/s at 4,096
prompt tokens, 49.9 tok/s at 32,768, and 48.1 tok/s at 128,000**; median prompt
throughput was 1,345 / 2,506 / 2,386 tok/s. With the 262,144-token limit, a
**250,000-token prompt was read at 2,116 tok/s (118.5 s to the first token) and
decoded at 40.3 tok/s**; at the shorter lengths decode was 13-17% slower than with
the 131,072-token limit ([below](#262144-token-context)). These are synthetic
code-explanation requests with greedy decoding and a 256-token output cap. They do
not establish general answer quality or performance on other workloads.

## Hardware and software

- NVIDIA GeForce RTX 4090 Laptop GPU (AD103); 16,376 MiB reported VRAM. nvidia-smi
  reported a 175 W current and maximum power limit (default 80 W), with
  `nvidia-powerd` (Dynamic Boost) active. PCIe capability: Gen 4, x16. The engine's
  startup transfer probe reported 19.3 GB/s host-to-device. GPU clocks were not fixed.
- Intel Core i9-13900HX; 32 logical CPUs. No AVX-512: the engine used AVX2 with
  23 expert-pool workers plus its host thread. CPU limits from the Lenovo firmware
  attributes: PL1 125 W, PL2 190 W.
- 64 GB installed RAM; Linux reported 61.28 GiB usable. Storage: WD_BLACK SN850X
  2 TB NVMe (model files and data directory). 512 GiB swap file.
- Platform profile `performance` (the `lenovo-wmi-gamezone` driver also offers
  `max-power` and `custom`; they were not used).
- Ubuntu 26.04.1 LTS, kernel 7.0.0-38-generic, NVIDIA driver 610.57.04 (reports
  CUDA 13.3).
- Source commit `99f3dbd0b21d1401b3769e0c0d963913607f380b`; engine 0.1.38.
  **Source build:** setup found no ready-made engine for v0.1.38 (the release
  download returned HTTP 404) and compiled one locally for CUDA architecture 89 with
  CUDA Toolkit 13.4 (NVCC V13.4.92) and GCC 15.2.0. Python 3.14.4. Hashes of the
  engine binary and the expert profile are in [artifact-hashes.json](artifact-hashes.json).
- During the run, Firefox and other large applications were closed; the desktop
  session stayed running. The machine was not otherwise isolated.

## Model and configuration

Model: `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`, revision
`ed59f92082b1e93c0e96d60a8b11aab089b52f09` (setup's pinned revision):

- `IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf` (54,817,524,224 bytes)
- `IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf` (28,800,138,432 bytes)

The files were downloaded and checked by the unmodified installer; they were not
hashed again for this report. The MTP draft layer was fetched by the installer from
`Qwen/Qwen3.8-Flash-Next`. No vision encoder.

Setup command:

```bash
./setup.sh --yes --family qwen --model IQ3_S --context 131072 --kv int8 --vision no --no-start
```

- Context 131,072; INT8 KV. The launch arguments contain no `--kv-resident`, so KV
  streaming was off (setup's choice for this RAM size).
- Expert cache `auto`: 3,865 slots, 7.38 GiB VRAM, profile-prefilled, no eviction.
  The bundled expert profile was used without calibration.
- Prefill `auto` selected chunks of 8,192 tokens, borrowing 2,333 expert-cache
  slots (4.43 GiB) during prompt processing.
- 46.84 GiB of experts loaded into RAM at 4.96 GiB/s; 4 KB pages (no huge pages).
- MTP `--spec 4 --spec-min-p 0.5`.
- Low-RAM mode off; vision off; experimental speed projection off.
- Reasoning disabled, temperature 0, maximum 256 generated tokens per speed run.

The complete configuration is [strata-iq3_s.json](strata-iq3_s.json). The engine log
of the measured session (from engine start to the last recall check) is
[engine.log](engine.log). The server was started with `BROWSER=true ./run-iq3_s.sh`
so that it opened no browser.

## Method and reproduction

The benchmark and monitor scripts are the ones from the
[RTX 5090 report](../2026-09-30-community-rtx-5090/README.md), unchanged, run from
the repository root against the installed server:

```bash
B=bench/results/2026-09-30-community-rtx-5090
.venv/bin/python $B/monitor.py ~/strata-bench/telemetry.jsonl &
.venv/bin/python $B/benchmark.py --root . --pack ../Strata-data/packs/iq3_s \
  --url http://127.0.0.1:8080 --out ~/strata-bench
kill %1
.venv/bin/python tools/needle_bench.py --url http://127.0.0.1:8080 \
  --lengths 32k,128k --depths 10,50,90 --out ~/strata-bench/needles.json
```

Before the benchmark, the same loaded engine answered one 1,000-token request
(a power check, not part of the results). One short warm-up request was then
excluded. Three runs at each length were executed serially in increasing-length
order. All nine speed requests processed their entire prompt: **zero reused
tokens**. The GPU expert cache was filled at startup and kept between requests;
loading time is excluded.

TTFT and total latency are measured by the client over loopback. Engine prompt
throughput is freshly read tokens divided by `prompt_ms`; decode throughput is
`engine_generated / decode_ms`.

## Results

Each cell is the median **[minimum–maximum]** of three runs. Every request generated
256 tokens and stopped at the output limit, with no failed or cancelled requests.

| Prompt tokens | Reused | Prompt tok/s | Decode tok/s | TTFT seconds | Total seconds |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 1,345.0 [1,280.0–1,367.7] | 51.2 [42.7–53.9] | 3.084 [3.032–3.235] | 8.004 [7.802–9.205] |
| 32,768 | 0 | 2,505.5 [2,496.1–2,511.8] | 49.9 [49.0–53.1] | 13.150 [13.135–13.201] | 18.305 [17.919–18.345] |
| 128,000 | 0 | 2,385.8 [2,376.5–2,412.7] | 48.1 [47.6–48.8] | 53.818 [53.217–54.043] | 59.164 [58.432–59.333] |

Raw per-run records: [results.json](results.json); aggregates:
[summary.json](summary.json). Decode expert-cache hit rates were 68.2–80.3%, and
the model accepted 136–162 of 203–234 MTP drafts per request. The lowest decode
run (42.7 tok/s) is the first 4K run, which also had the lowest hit rate (68.2%).

[telemetry.jsonl](telemetry.jsonl) has 267 samples over 273.5 seconds, covering the
speed suite only (not the recall checks). Observed peaks
([memory-summary.json](memory-summary.json)):

- GPU memory: **15,698 MiB** of 16,376 MiB, system-wide.
- Host RAM: **54.74 GiB** used (`MemTotal - MemAvailable`, system-wide, not process
  RSS); MemAvailable never fell below 6.55 GiB.
- Swap: 698.5 MiB allocated at the start and at the peak (no increase during the
  run). Swap I/O was not measured.
- GPU power: median **170 W** over the 250 samples with utilization above 50%, peak
  176 W; peak temperature 86 °C. Prompt processing ran at the power limit.

A separate power check on the same session (one 1,000-token answer, 25-token
prompt, not part of the table) decoded at 57.0 tok/s with a median GPU draw of
116 W (peak 141 W) over 39 busy samples: decode did not reach the power limit.

## Recall

The unchanged `tools/needle_bench.py` found **all six needles** at depths 10%,
50% and 90%. Actual prompt lengths were 31,969–31,970 tokens for `32k` and
122,753–122,755 for `128k`. See [needles.json](needles.json). The third case of
each length reused 16,384 prompt tokens (engine `/metrics`); the others reused zero.

## 262,144-token context

Measured later the same day (after a reboot) on the same machine, commit, engine
build, model files and power mode. Only the context changed:

```bash
./setup.sh --setup --yes --family qwen --model IQ3_S --context 262144 --kv int8 --vision no --no-start
```

The resulting configuration differs from the 131,072-token one only in
`--max-context 262144` ([context-262144/strata-iq3_s.json](context-262144/strata-iq3_s.json)).
KV streaming stayed off, so the larger INT8 KV cache sits in VRAM and the expert
cache shrank to **2,856 slots (5.47 GiB)** from 3,865 (7.38 GiB). Prefill still used
8,192-token chunks and borrowed 2,319 slots (4.43 GiB). The PCIe probe reported
20.6 GB/s. Engine log of this session: [context-262144/engine.log](context-262144/engine.log).

The same benchmark script ran with `--targets 4096,32768,128000,250000`, and the
monitor ran alongside it. No other request preceded the warm-up. All twelve speed
requests read their entire prompt (**zero reused tokens**), generated 256 tokens
and stopped at the output limit.

| Prompt tokens | Reused | Prompt tok/s | Decode tok/s | TTFT seconds | Total seconds |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 1,343.0 [1,266.5–1,397.2] | 43.4 [37.4–44.1] | 3.097 [2.973–3.278] | 9.056 [8.961–9.783] |
| 32,768 | 0 | 2,457.5 [2,434.0–2,469.2] | 43.3 [43.0–43.3] | 13.412 [13.349–13.539] | 19.295 [19.228–19.457] |
| 128,000 | 0 | 2,291.6 [2,290.5–2,327.3] | 39.7 [39.5–42.6] | 56.027 [55.166–56.054] | 62.031 [61.612–62.448] |
| 250,000 | 0 | 2,116.4 [2,115.7–2,132.3] | 40.3 [40.2–41.0] | 118.457 [117.552–118.483] | 124.661 [123.891–124.798] |

Raw records: [context-262144/results.json](context-262144/results.json); aggregates:
[context-262144/summary.json](context-262144/summary.json). Decode expert-cache hit
rates were 67.1–74.6% (68.2–80.3% with the 131,072-token limit), and the model
accepted 134–155 of 203–246 MTP drafts per request. Against the 131,072-token
configuration, median decode was 15% lower at 4,096 tokens, 13% lower at 32,768
and 17% lower at 128,000; prompt throughput was within 4%.

The telemetry ([context-262144/telemetry.jsonl](context-262144/telemetry.jsonl),
662 samples over 679.5 seconds, speed suite only) shows peaks of **15,718 MiB** GPU
memory and **55.11 GiB** host RAM used; MemAvailable never fell below 6.17 GiB, and
swap stayed at 0.7 MiB. GPU power had a median of **172 W** over the 626 samples with
utilization above 50%, one sample at 184 W, and a peak temperature of 89 °C
([context-262144/memory-summary.json](context-262144/memory-summary.json)).

The needle test ran with `--lengths 32k,128k,256k` and found **all nine needles**.
Actual prompt lengths were 32,034–32,035 tokens for `32k`, 122,798–122,800 for
`128k` and 249,309–249,310 for `256k`
([context-262144/needles.json](context-262144/needles.json)). All three `256k`
cases and the third `32k` and `128k` cases reused 16,384 prompt tokens; the others
reused zero.

## Limitations

This is one machine, one quantization, two context limits and a small synthetic
workload. Long output, sampled decoding, thinking, coding-task correctness, tool
use and a sustained thermal run were not evaluated. A
notebook's power and thermal behaviour depends on its power mode, cooling and
ambient temperature; other power modes were not tested. The README's numbers for
other machines are not a controlled comparison.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://uutb.wtpuscm.cn/shichang/upload-026922.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://xazn.wtpuscm.cn/xuexi/objective-310333.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://cmmi.wtpuscm.cn/gongju/promotion-077099.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://brve.wtpuscm.cn/gongsi/restaurant-895500.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://rvkv.wtpuscm.cn/yanjiu/services-520729.html)
* [面向大规模网络的 Strata 工业级架构基准](https://jwwi.wtpuscm.cn/yingyong/coupon-340605.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://pxlj.wtpuscm.cn/peixun/home-395751.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://cxlu.wtpuscm.cn/yunsuan/page-934.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://qxij.wtpuscm.cn/yingyong/chapter-687440.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://mydg.wtpuscm.cn/youhua/study-830555.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://zvsh.wtpuscm.cn/yunying/creative-908833.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://iqwx.wtpuscm.cn/chanpin/landing-357752.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://wwwn.wtpuscm.cn/zhinan/system-425439.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://zzvq.wtpuscm.cn/wendang/customer-852940.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://wdjh.wtpuscm.cn/kuangjia/template-087596.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://bwfm.wtpuscm.cn/chanpin/productivity-763623.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://rhrd.wtpuscm.cn/yingxiao/recipe-022535.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://lqkn.wtpuscm.cn/fuwu/backup-740084.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://ehcs.wtpuscm.cn/kuangjia/health-912085.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://cvoe.wtpuscm.cn/kuangjia/training-083077.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://ywwk.wtpuscm.cn/jiaocheng/seminar-154448.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://rwdd.wtpuscm.cn/keji/objective-552465.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://mpqg.wtpuscm.cn/huodong/recipe-820105.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://scrb.tcti.cn/chuangxin/analytics-99779888.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://dwjg.tcti.cn/yingxiao/dashboard-89871176.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://uyyx.tcti.cn/anfang/case-32626337.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://rxvi.tcti.cn/gongju/hotel-92459906.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://wfci.tcti.cn/zhineng/system-09002381.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://hnjb.tcti.cn/yanjiu/machine-19614607.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://bmpi.tcti.cn/pingtai/status-51698523.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://hipf.tcti.cn/pingtai/report-33138520.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://ziaa.tcti.cn/youhua/supplier-67978311.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://ydjy.tcti.cn/qiye/tutorial-30070931.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://tkye.tcti.cn/liuliang/landing-53991595.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://klpd.tcti.cn/zhizhu/network-02529802.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://glor.tcti.cn/chuangxin/automation-55828984.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://irem.tcti.cn/jiaoliu/investment-32173131.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://nljt.tcti.cn/youhua/networking-82555345.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://phca.tcti.cn/peixun/sale-88127999.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://hhfm.tcti.cn/shangye/metric-56710346.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://hibb.wtpuscm.cn/liuliang/conference-157473.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/pingtai/domain-90117653.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/news/22717)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/keji/status-19843321.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://jurl.tcti.cn/pingtai/platform-18738367.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://yrlu.tcti.cn/keji/story-56585776.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://djcm.wtpuscm.cn/hezuo/unsubscribe-114434.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://ovcw.wtpuscm.cn/zhineng/analysis-556267.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://agpq.wtpuscm.cn/zhizhu/demographic-002039.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://wwuk.wtpuscm.cn/chuangxin/products-948845.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://eosh.wtpuscm.cn/youhua/recommendation-775352.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://jmjq.wtpuscm.cn/pingtai/resolution-650692.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://nwze.wtpuscm.cn/gongsi/shopping-147099.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://ulut.wtpuscm.cn/gongxiang/topic-685.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://fsus.wtpuscm.cn/pingce/fitness-017457.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://tldm.wtpuscm.cn/yingxiao/folder-245993.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://gasu.wtpuscm.cn/yinqing/partner-608187.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://gwzc.wtpuscm.cn/yingxiao/admin-020546.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://ahxl.wtpuscm.cn/gongju/alert-682647.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://qvbq.wtpuscm.cn/fuwu/sales-269341.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://zvle.wtpuscm.cn/gongxiang/update-492585.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://dsbs.wtpuscm.cn/gongju/tag-454814.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://lvxx.wtpuscm.cn/zhineng/tutorial-629440.html)
* [Strata 异步中间件流水线与 any 接入规范](https://qwhh.wtpuscm.cn/hezuo/technology-369141.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://wmmi.wtpuscm.cn/yingxiao/technology-742636.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://cetl.wtpuscm.cn/pingtai/register-126599.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://oeyn.wtpuscm.cn/qiye/update-633625.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://hwta.wtpuscm.cn/wenzhang/module-742833.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://emvt.wtpuscm.cn/shichang/widget-496590.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://flwg.tcti.cn/zhizhu/mobile-39842400.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://dgmg.tcti.cn/zhizhu/enterprise-41672100.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://eqgd.tcti.cn/baogao/notification-86839240.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://qkvt.tcti.cn/jiaocheng/demographic-52984259.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://gvaf.tcti.cn/xinwen/community-01994054.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://occa.tcti.cn/jiaocheng/audience-87940170.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://nmza.tcti.cn/zixun/lesson-94686536.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://uhei.tcti.cn/yingyong/game-77047947.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://vzba.tcti.cn/qiye/kpi-21740901.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://fzmz.tcti.cn/yanjiu/mobile-66174813.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://xvit.tcti.cn/kuangjia/help-76553859.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://drtc.tcti.cn/xinwen/ebook-04081666.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://lrli.tcti.cn/baogao/deadline-00348909.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://aecs.tcti.cn/fuwu/fitness-76565940.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://uvbx.tcti.cn/jiaoliu/reporting-44696134.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://xocv.tcti.cn/zhineng/software-86089118.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://euca.tcti.cn/fenxi/ranking-98951297.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://xprj.wtpuscm.cn/yinqing/website-798091.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/jiaocheng/calendar-95437124.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/tech/54779)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/jianzhan/mobile-84192701.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://rvfy.tcti.cn/keji/subject-62044638.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://uebp.tcti.cn/yinqing/premium-26978149.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://zmyg.wtpuscm.cn/peixun/sport-406904.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://czzs.wtpuscm.cn/shuju/shopping-833503.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://ulxy.wtpuscm.cn/yingyong/deadline-203583.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://apcg.wtpuscm.cn/jiaocheng/presentation-509798.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://pluh.wtpuscm.cn/gongxiang/strategy-837876.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://irvc.wtpuscm.cn/hezuo/forum-137099.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://zwcx.wtpuscm.cn/pingce/customer-729488.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://eqqq.wtpuscm.cn/huodong/expensive-403.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://bjlz.wtpuscm.cn/kaifa/discount-806675.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://gwqe.wtpuscm.cn/jianzhan/terms-676546.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://qknt.wtpuscm.cn/yunsuan/marketing-001355.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://wrko.wtpuscm.cn/wangluo/audience-364065.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://qkoi.wtpuscm.cn/peixun/keyword-324316.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://yvpn.wtpuscm.cn/keji/forum-856122.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://lkha.wtpuscm.cn/yunying/goal-207319.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://nujk.wtpuscm.cn/xuexi/policy-437119.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://yrad.wtpuscm.cn/xuexi/webinar-379279.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://gjgz.wtpuscm.cn/yingxiao/resource-013243.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://taxz.wtpuscm.cn/anli/update-543895.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://zdvr.wtpuscm.cn/huodong/notification-160212.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://knpa.wtpuscm.cn/peixun/platform-026778.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://pwig.wtpuscm.cn/chuangxin/profit-745832.html)
* [Strata 故障自愈与网络拓扑重构实践](https://tsvh.wtpuscm.cn/baogao/demographic-976442.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://vydz.tcti.cn/huodong/report-09473734.html)
* [Strata 权威网络权重传递与收录基准规范](https://czfy.tcti.cn/paiming/innovation-44539039.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://cagg.tcti.cn/pingce/image-03709255.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://pkdb.tcti.cn/jiaocheng/milestone-91436528.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://xirc.tcti.cn/pingce/price-02934807.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://lhqp.tcti.cn/baogao/health-10142943.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://oxkg.tcti.cn/jianzhan/health-62262828.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://absx.tcti.cn/yanjiu/economy-42878355.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://cpaf.tcti.cn/yanjiu/demographic-94624513.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://cmqb.tcti.cn/wenzhang/study-34403988.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://tztx.tcti.cn/jishu/document-47722069.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://rfao.tcti.cn/chuangxin/extension-68810032.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://vsmn.tcti.cn/anfang/technology-55541296.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://euem.tcti.cn/xitong/visitor-58234458.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://adar.tcti.cn/pingtai/subscribe-79864670.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://ohol.tcti.cn/liuliang/workshop-23157980.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://rvsx.tcti.cn/youhua/cost-21873152.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://pbhk.wtpuscm.cn/zhineng/interface-679293.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/xuexi/promotion-01855952.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/tech/31417)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/yinqing/support-00332211.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://rkti.tcti.cn/anfang/luxury-69252404.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://pdsb.tcti.cn/guanjianci/communication-50330292.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://aqre.wtpuscm.cn/youhua/search-391505.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://mhlz.wtpuscm.cn/keji/alert-360662.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://ucsx.wtpuscm.cn/peixun/responsive-457735.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://tbog.wtpuscm.cn/zhineng/fashion-251133.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://gaul.wtpuscm.cn/pingtai/traffic-421212.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://xnkf.wtpuscm.cn/youhua/calendar-734620.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://oint.wtpuscm.cn/peixun/saving-246727.html)
* [Strata: Windows 分布式架构 (Verified)](https://jiug.wtpuscm.cn/wenzhang/health-851.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://icuj.wtpuscm.cn/zhineng/creative-289738.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://uoul.wtpuscm.cn/gongju/help-724297.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://lrgz.wtpuscm.cn/wenzhang/shopping-104361.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://cyep.wtpuscm.cn/zhizhu/recommendation-118196.html)

</details>

