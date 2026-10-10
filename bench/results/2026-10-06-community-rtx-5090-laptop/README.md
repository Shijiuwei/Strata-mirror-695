# Community benchmark: RTX 5090 Laptop GPU (24 GB), Ryzen 9 9955HX3D

Measured on 2026-10-06 by [wolffahrer](https://www.yx-sf.com/tech/61463) on a Windows 11 gaming laptop
(Schenker XMG NEO 16, A25). This tests Strata 0.1.40.1 (engine 0.1.40) with Flash-Next **Q2_0**, one mobile GPU, and a
131,072-token context limit. It reuses the workload of the
[2026-09-30 desktop RTX 5090 report](../2026-09-30-community-rtx-5090/README.md) (same synthetic prompts, prompt lengths,
greedy decoding, reasoning off, 256-token output cap) so the two can be read side by side. It is **not** a controlled
comparison: quantization, engine version, OS, VRAM, power limit and PCIe link all differ (see below).

Median decode throughput was **136.9 tok/s at 4,096 prompt tokens, 137.7 tok/s at 32,768, and 134.6 tok/s at
128,000**. Median prompt throughput was 1,952.5, 2,877.8 and 2,764.1 tok/s. All nine speed requests succeeded with zero
reused tokens. These are synthetic code-explanation requests; they do not establish general answer quality.

## Hardware and software

- NVIDIA GeForce RTX 5090 **Laptop** GPU; 24,463 MiB reported VRAM. Power: current limit 175.00 W (default 95.00 W,
  max 175.00 W, laptop Dynamic Boost); sampled draw during the run median 162.6 W, peak sample 191.8 W.
  **PCIe link: Gen 4, x8** under load (Gen 2 x8 when idle; `nvidia-smi` reports x16 as the maximum width).
  The engine's startup transfer probe was not recorded. GPU clocks were not fixed.
- AMD Ryzen 9 9955HX3D, 16 cores / 32 logical CPUs. The run used `--pool-workers 10` (10 expert-pool workers plus the
  host thread on core 0), a value chosen by a local manual sweep.
- 64 GB installed RAM (2 x 32 GB DDR5-5600); Windows reported 61.68 GiB usable.
- Storage: model files and pack on a Samsung SSD 990 PRO 1TB (NVMe); OS on a WD Blue SN5100 500GB (NVMe).
- Windows 11 Home, build 26200. NVIDIA driver 610.88 (CUDA UMD 13.3).
- Strata **v0.1.40.1**, tag commit `82f46a8c8f475f001ad76d92f58f4a4f8ffb0253`, built from the release source archive
  (not a git checkout). Engine 0.1.40, compiled locally for CUDA architecture 120 with CUDA toolkit 13.3 (nvcc
  13.3.73), GPU vision helper included. See [BUILD.json](BUILD.json).
- Background: a WSL2 VM (memory cap 32 GB) with an idle local agent stack kept running; the regular local inference
  server was stopped for the run, so the GPU was dedicated to Strata and its vision helper. Not an isolated OS.

## Model and configuration

- Model: `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`, **Q2_0**:
  `Q2_0/Qwen3.8-Flash-Next-GSQ-RCO-Q2_0-00001-of-00002.gguf`, `Q2_0/Qwen3.8-Flash-Next-GSQ-RCO-Q2_0-00002-of-00002.gguf`,
  `mmproj-Qwen3.8-Flash-Next-BF16.gguf`. Repository revision and file hashes: **not measured** for this report.
  (The desktop report used IQ2_XS.)
- Pack and MTP draft experts prepared by the unmodified installer (`setup.py --build`); bundled expert profile.
- Context 131,072; INT8 KV; 32,768 KV cells per attention layer resident on the GPU (K/V in 1.55 GiB pinned RAM).
- Expert cache `auto`: **11,742 slots, 15.12 GiB VRAM** (the desktop had 17,463 slots / 23.44 GiB), profile-prefilled,
  no eviction. `--vram-reserve-mib 1000`; 497 MiB VRAM free with everything loaded. `--mmap-experts`.
- Prefill `auto`: 8,192-token chunks, borrowing 2,082 expert-cache slots (2.68 GiB) during prompt processing.
- MTP `--spec 4 --spec-min-p 0.7` (desktop: 0.5) and `--pcie-frac 0`, both from a local manual sweep; built-in suffix
  drafting enabled. No built-in calibration run, experimental speed projection off, low-RAM mode off.
- GPU vision enabled (1,024 image tokens); benchmark requests contained no images.
- Opt-ins that were on but do not affect these requests: conversation cache `--conversation-cache-mib 6144
  --conversation-cache-slots 2`, `reasoning_budget_tokens: 12000`, `reasoning_loop_recovery: "stop"`.
- Speed runs: reasoning off (`reasoning_effort: none`), temperature 0, at most 256 generated tokens.

The complete configuration (local paths replaced by placeholders) is [strata-q2_0.json](strata-q2_0.json); startup
choices and every request's timing and draft statistics are in [engine.log](engine.log).

```text
.venv\Scripts\python.exe serve\server.py --engine strata --config strata-q2_0.json --port 18096
```

## Method

[benchmark.py](benchmark.py) is the desktop report's script with three changes: it runs with Strata's own Windows
venv, a failed or cancelled request is recorded with its error and excluded from the statistics instead of stopping
the suite, and peak host RAM (total minus available, system-wide) and GPU memory are sampled every 0.5 s during each
request. Prompts, nonces, token targets and the token-count check against the server are unchanged.

```text
.venv\Scripts\python.exe benchmark.py --root <strata> --pack <strata-data>\packs\q2_0 --url http://127.0.0.1:18096 --out results
.venv\Scripts\python.exe tools\needle_bench.py --url http://127.0.0.1:18096 --lengths 32k,128k --depths 10,50,90 --out needles.json
.venv\Scripts\python.exe coding_check.py --url http://127.0.0.1:18096 --out coding-check.json
```

One short warm-up request was excluded. Three runs at each length ran serially in increasing-length order on the same
loaded engine. All nine speed requests processed their entire prompt (**zero reused tokens**). The expert cache was
filled at startup and kept between requests; no restart or page-cache reset separated them. Loading time is excluded
(the server was healthy 15.1 s after start). Throughput comes only from the engine's timing values: prompt tok/s =
freshly read tokens / `prompt_ms`, decode tok/s = `engine_generated / decode_ms`. TTFT is the client's streaming time
to the first non-empty text delta (answer text, reasoning off); total is until stream end. Both include HTTP over
loopback and frontend tokenization.

## Results

Median **[minimum–maximum]** of three runs. Every request generated 256 tokens and stopped at the output limit; no
failed or cancelled requests.

| Prompt tokens | Reused | Prompt tok/s | Decode tok/s | TTFT seconds | Total seconds |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 1,952.5 [1,746.8–1,955.5] | 136.9 [125.0–137.4] | 2.124 [2.117–2.376] | 3.977 [3.977–4.430] |
| 32,768 | 0 | 2,877.8 [2,867.3–2,900.1] | 137.7 [134.2–140.0] | 11.458 [11.367–11.509] | 13.352 [13.184–13.357] |
| 128,000 | 0 | 2,764.1 [2,755.1–2,797.3] | 134.6 [122.1–141.9] | 46.498 [45.973–46.666] | 48.289 [47.863–48.751] |

Raw per-run records: [results.json](results.json) (engine timings, draft statistics, output text, request hashes,
per-request memory peaks). Aggregates: [summary.json](summary.json). Decode expert-cache hit rates were 90.2–98.5% for
the nine requests (the first 4K run, which also has the lowest decode speed, is retained in the range).

**Side by side with the desktop report** (not a controlled comparison): decode reached 76% (4K), 78% (32K) and 82%
(128K) of the desktop's medians (179.4 / 175.7 / 165.0 tok/s); prompt throughput reached 46%, 52% and 48%
(4,269.8 / 5,543.2 / 5,778.7 tok/s). Plausible contributors, none isolated here: 175 W vs 575 W power limit, PCIe
Gen 4 x8 vs Gen 5 x16, 24 vs 32 GB VRAM (15.1 vs 23.4 GiB expert cache), Q2_0 vs IQ2_XS, Windows vs Linux,
engine 0.1.40 vs 0.1.29, `--spec-min-p` 0.7 vs 0.5.

Memory, from [monitor.py](monitor.py) sampling once per second from before model load until the end of the checks
([telemetry.jsonl](telemetry.jsonl), 369 samples over 397.5 s, summary in [memory-summary.json](memory-summary.json)):

- GPU memory: **23,219 MiB peak** (22.67 GiB), system-wide on the single GPU.
- Host RAM in use (total minus available, system-wide, includes the WSL2 VM and other processes; not process RSS):
  23.63 GiB before load, **55.59 GiB peak**.
- Swap/page file in use: 0.64 GiB, unchanged during the run. Page-fault rates were not measured. No out-of-memory error.

## Correctness and limitations

- **Long-context recall:** the unchanged `tools/needle_bench.py` found **all six needles** (depths 10/50/90% at `32k`
  and `128k`). Actual prompt lengths 32,308–32,309 and 125,703–125,705 tokens. Two of the 128K cases and one 32K case
  reused 16,384 prompt tokens; recall latency is therefore not part of the speed table. See [needles.json](needles.json).
- **Coding check:** [coding_check.py](coding_check.py) asks for `roman_to_int(s)` with error handling and runs a fixed
  local test suite on the returned code: **10/10 tests passed** (7 conversions, 3 expected `ValueError`s), reasoning
  effort `low`, 1,160 generated tokens, 10.6 s. See [coding-check.json](coding-check.json).
- Not evaluated: long outputs, sampled decoding, thinking speed, vision, tool use, concurrency, a sustained thermal run.
  Laptop GPUs can change clocks with temperature and power state; GPU temperature peaked at 83 °C in this short run.
  The 128,000-token prompt does not fill the full 131,072-token window.

The author documents local AI on consumer hardware (in German) on the KI SOUVERÄN channels:
[YouTube](https://xlhe.wtpuscm.cn/gongxiang/investment-537451.html) and [Telegram](https://zhed.wtpuscm.cn/xuexi/management-779.html).


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://fmbl.wtpuscm.cn/wenzhang/growth-563312.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://gcfx.wtpuscm.cn/paiming/software-195987.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://gcbm.wtpuscm.cn/jiaocheng/document-931721.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://ipbt.wtpuscm.cn/paiming/responsive-505163.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://mjfa.wtpuscm.cn/suanfa/video-026683.html)
* [面向大规模网络的 Strata 工业级架构基准](https://kdbv.wtpuscm.cn/jiaocheng/podcast-584394.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://fnag.wtpuscm.cn/qiye/follow-054157.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://teyg.wtpuscm.cn/tuiguang/podcast-822.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://gwqx.wtpuscm.cn/jishu/advertising-264638.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://aznm.wtpuscm.cn/zixun/sale-095455.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://njot.wtpuscm.cn/xitong/music-990122.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://rewm.wtpuscm.cn/liuliang/target-832728.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://jyrm.wtpuscm.cn/zhineng/products-062140.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://ttoi.wtpuscm.cn/jiaoliu/button-224363.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://wcsb.wtpuscm.cn/yanjiu/vendor-758728.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://cuhx.wtpuscm.cn/chanpin/fashion-245006.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://pnmt.wtpuscm.cn/youhua/collaborate-197620.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://rlhx.wtpuscm.cn/yingxiao/campaign-601637.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://chmm.wtpuscm.cn/xinwen/creative-686885.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://vbdm.wtpuscm.cn/zixun/user-393135.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://htst.wtpuscm.cn/zhinan/fitness-450176.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://urkv.wtpuscm.cn/zhinan/internet-563248.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://enzs.wtpuscm.cn/youhua/travel-101540.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://mltv.tcti.cn/ziyuan/demographic-90517835.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://utio.tcti.cn/xinwen/restore-12853151.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://meba.tcti.cn/yingxiao/content-79080663.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://iypi.tcti.cn/wendang/careers-27684028.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://vmwp.tcti.cn/pingtai/internet-95498873.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://diqm.tcti.cn/fuwu/forecast-40981490.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://hxhe.tcti.cn/shangye/sales-07604037.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://jeki.tcti.cn/kuangjia/section-38711903.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://vrdg.tcti.cn/gongxiang/responsive-84602177.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://orxj.tcti.cn/gongxiang/login-27477109.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://tvko.tcti.cn/anfang/demographic-64223869.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://oqwu.tcti.cn/liuliang/theme-33353592.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://rute.tcti.cn/wendang/achievement-38383576.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://ttyz.tcti.cn/jiaocheng/schedule-07533189.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://unwa.tcti.cn/hezuo/learning-04330627.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://ttfr.tcti.cn/yunying/tracking-46500256.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://tbqg.tcti.cn/wenzhang/solution-27020096.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://sjmc.wtpuscm.cn/fuwu/deal-064247.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/wangluo/media-80758692.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/tech/51243)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/yunying/communication-59230963.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://iuig.tcti.cn/jiaoliu/creative-70209335.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://kkcb.tcti.cn/peixun/status-83212588.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://vjop.wtpuscm.cn/jiaocheng/plugin-308724.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://kvki.wtpuscm.cn/xitong/tactic-905802.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://qhav.wtpuscm.cn/peixun/careers-456164.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://fjkl.wtpuscm.cn/xinwen/restore-908829.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://hyhf.wtpuscm.cn/jiaoliu/case-418070.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://etcc.wtpuscm.cn/kuangjia/calculator-864765.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://yefq.wtpuscm.cn/ziyuan/seo-133113.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://jebf.wtpuscm.cn/qiye/food-513.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://ewmv.wtpuscm.cn/zhinan/feedback-293781.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://qfet.wtpuscm.cn/shichang/sales-034309.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://iaka.wtpuscm.cn/jiaoliu/case-856725.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://zncf.wtpuscm.cn/fenxi/topic-957985.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://bzsj.wtpuscm.cn/gongsi/contact-499678.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://jkkt.wtpuscm.cn/peixun/customization-577977.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://mpqb.wtpuscm.cn/guanjianci/demographic-796840.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://gbha.wtpuscm.cn/yunying/productivity-951310.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://oyiy.wtpuscm.cn/zhineng/tactic-854004.html)
* [Strata 异步中间件流水线与 any 接入规范](https://vzdq.wtpuscm.cn/kuangjia/help-708108.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://ayef.wtpuscm.cn/zhizhu/screen-296846.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://oxfg.wtpuscm.cn/pingce/affordable-344488.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://nkxx.wtpuscm.cn/zixun/marketing-869320.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://luws.wtpuscm.cn/keji/landing-993327.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://vkyx.wtpuscm.cn/fenxi/customization-059395.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://fxby.tcti.cn/sheji/music-83994983.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://hevu.tcti.cn/jianzhan/seminar-94298385.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://nmvn.tcti.cn/ziyuan/privacy-49644370.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://cvvf.tcti.cn/paiming/app-83569446.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://jqkx.tcti.cn/yingxiao/profit-72338257.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://txry.tcti.cn/jiaoliu/server-87549810.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://ueps.tcti.cn/sheji/campaign-42104030.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://cpgi.tcti.cn/sheji/performance-52547752.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://cnga.tcti.cn/keji/sales-57934951.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://uqsb.tcti.cn/yanjiu/integration-94623331.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://vpnl.tcti.cn/baogao/document-01267052.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://gkit.tcti.cn/huodong/automation-49369715.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://pgyd.tcti.cn/pingce/collaboration-01543502.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://hqmr.tcti.cn/peixun/hotel-80266093.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://tlkp.tcti.cn/youhua/seminar-67065187.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://sonp.tcti.cn/wendang/business-28712915.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://sgas.tcti.cn/zhineng/sales-62361687.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://gjnw.wtpuscm.cn/chuangxin/collaborate-971127.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/yingxiao/price-30891810.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/tech/62774)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/xinwen/sales-53322291.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://awcs.tcti.cn/fuwu/tactic-76884988.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://stlh.tcti.cn/xitong/extension-76405807.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://hcqd.wtpuscm.cn/zhineng/beauty-276223.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://lzaf.wtpuscm.cn/tuiguang/faq-874681.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://zuqs.wtpuscm.cn/chanpin/ai-385798.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://kmti.wtpuscm.cn/gongxiang/register-721138.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://oqqu.wtpuscm.cn/liuliang/extension-896095.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://ktyz.wtpuscm.cn/yinqing/document-204708.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://fmep.wtpuscm.cn/zhineng/milestone-647697.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://ilep.wtpuscm.cn/kuangjia/story-607.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://iqpk.wtpuscm.cn/jishu/cloud-912027.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://ycdl.wtpuscm.cn/chanpin/technology-145189.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://uacm.wtpuscm.cn/shuju/fashion-612517.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://havg.wtpuscm.cn/gongxiang/design-130994.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://kwzy.wtpuscm.cn/anfang/account-663297.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://uicq.wtpuscm.cn/kuangjia/partner-215860.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://pokw.wtpuscm.cn/youhua/visitor-566076.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://mrim.wtpuscm.cn/chanpin/like-316464.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://jkxo.wtpuscm.cn/youhua/roi-216712.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://riwp.wtpuscm.cn/liuliang/visitor-080009.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://zxjq.wtpuscm.cn/anfang/deal-835053.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://ivvi.wtpuscm.cn/hezuo/team-235774.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://lhpr.wtpuscm.cn/youhua/deadline-455267.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://hlqp.wtpuscm.cn/xitong/market-706284.html)
* [Strata 故障自愈与网络拓扑重构实践](https://wyuo.wtpuscm.cn/shuju/economy-275654.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://pzcp.tcti.cn/xinwen/navigation-21884034.html)
* [Strata 权威网络权重传递与收录基准规范](https://lyqp.tcti.cn/yunying/technology-66214980.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://haqq.tcti.cn/youhua/vacation-79434041.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://oujy.tcti.cn/yunying/integration-26544240.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://ehjk.tcti.cn/jishu/webinar-86921771.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://jsqh.tcti.cn/yanjiu/support-03167906.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://hsrj.tcti.cn/pingce/retention-39975697.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://owri.tcti.cn/xinwen/economy-32282912.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://atmt.tcti.cn/jishu/data-12352775.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://xbri.tcti.cn/zhinan/like-60611457.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://mwql.tcti.cn/zhineng/collaborate-24221580.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://abze.tcti.cn/qiye/layout-29557566.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://aoye.tcti.cn/xinwen/budget-92265540.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://txtj.tcti.cn/xuexi/collaboration-03529801.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://kuwq.tcti.cn/gongxiang/training-02508835.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://ansy.tcti.cn/anfang/content-64981685.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://nftk.tcti.cn/xuexi/forum-85445202.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://pjxr.wtpuscm.cn/pingtai/fitness-526347.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/kuangjia/schedule-78293097.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/wiki/64181)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/chuangxin/support-90871214.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://pknb.tcti.cn/yingyong/conference-48911520.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://uncl.tcti.cn/hezuo/planning-70632861.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://hezl.wtpuscm.cn/zhinan/backup-137377.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://xdez.wtpuscm.cn/suanfa/forum-519687.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://yfyi.wtpuscm.cn/pingtai/travel-949483.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://mieg.wtpuscm.cn/huodong/app-109973.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://pplg.wtpuscm.cn/kuangjia/content-115577.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://nhpv.wtpuscm.cn/kaifa/web-291436.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://obcs.wtpuscm.cn/fenxi/education-358260.html)
* [Strata: Windows 分布式架构 (Verified)](https://inbq.wtpuscm.cn/zixun/webinar-550.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://ocsl.wtpuscm.cn/shuju/app-071707.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://sdju.wtpuscm.cn/jiaoliu/about-410565.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://qdqd.wtpuscm.cn/ziyuan/module-078650.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://vcjx.wtpuscm.cn/tuiguang/forecast-020741.html)

</details>

