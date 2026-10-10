# Community benchmark on 2x RTX PRO 4500 Blackwell, Swift IQ3_XXS, engine 0.1.40.1

Measured on 2026-10-06 by qni-live. Layer split over two 32 GB cards, Swift 1.5 IQ3_XXS. Two parts: (1) single stream, 1k to 256k prompt tokens, engine 0.1.40.1 against 0.1.39 measured right after it on the same setup; (2) several concurrent requests with and without `--batch` (the all-resident batch fix from 0.1.40, issue #776). One prompt family only (code explanation). Limits are listed at the end.

The tests were run by an AI agent (Claude Code) on my machine, with me deciding the plan and reviewing the results and this report.

## Hardware and software

- GPU: 2x NVIDIA RTX PRO 4500 Blackwell, 32 GB GDDR7 each (GB203, SM 12.0, 200 W limit, unchanged), no NVLink, PCIe Gen 5 x16 to each card (maximum link as reported by `nvidia-smi`)
- CPU: AMD Ryzen Threadripper 7960X, 24 cores / 48 threads. RAM: 64 GB DDR5 (61 GiB usable)
- Storage: model on a 3.7 TB NVMe at PCIe 4.0 x4. The disk only matters at start.
- OS: Ubuntu 26.04.1 LTS, kernel 7.0.0-38-generic. Driver 595.91.07. CUDA 13.4 toolkit
- Strata: tag v0.1.40.1, commit `82f46a8c8f475f001ad76d92f58f4a4f8ffb0253`, **source build** for SM 120 (`build.txt`). Control: v0.1.39, commit `6f32ec070f23ced9f50e704d854d775da52591ab`, source build
- Background workloads: none besides the benchmark. The server was stopped and restarted between configurations.

## Model and configuration

- Model: `ukisai/Swift-1.5-Qwen3.8-Flash-Next-GSQ-RCO-GGUF`, IQ3_XXS, files `Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-0000{1,2}-of-00002.gguf` (39.8 GB + 36.2 GB). Pack built by setup, shipped expert profile, MTP runtime from `Strata-data/mtp/rt`
- Context 262144, KV `k8v4`, expert cache auto, prefill auto, `--spec 5`, `--spec-min-p 0.5`, layer split 25 over GPUs 0 and 1
- Solo runs: vision encoder loaded (no images sent), `--vram-reserve-mib 700`
- Batch runs: no vision, `--vram-reserve-mib 700`, `--batch 2` or `--batch 4`, each with `--batch-groups 2 --trim-stage-weights`. **No `STRATA_*` environment overrides**; in particular no `STRATA_VERIFY_ALL_RESIDENT=0`
- With 2 slots 100% of the experts are VRAM-resident; with 4 slots 95% (the startup log says so)
- Sampling: server `presence_penalty` set to **0.0** in all runs (our daily config uses 0.2; batch windows apply no penalties, so all arms use the same value). `bench.py`: `temperature 0`. `parlast.py`: `temperature 0.7, top_p 0.95, top_k 20`. Reasoning at the server default. No calibration, no experimental speed projection

Configs as run: `config-solo.json`, `config-batch2.json`, `config-batch4.json`. Start: `serve/server.py --engine strata --config <file> --port 8080`.

## Method

- Single stream: `bench.py` (`temperature 0`, `max_tokens 256`, the reasoning text counts toward the 256). Prompt = Strata source and docs cut to the target length plus an "explain and propose three refactorings" task, with a random nonce first so nothing is reused from the cache (`reused` is 0). One warm-up per length, then 3 measured runs; speeds are read from the server log line. The 256k row is new (target 245,000 tokens)
- Concurrency: `parlast.py`, 8,000-token prompts (a different one per request, nonce first), 400 tokens out, streaming through the server, 1-4 concurrent requests, 3 rounds, median. "Total tok/s" = all generated tokens / wall time of the round, **including** prompt reading (about 2.5 s per request). It is not decode speed. The 32k check used 32,000-token prompts and 300 tokens, 2 rounds.
- Needle recall: `tools/needle_bench.py --lengths 32k,128k --depths 10,50,90`, run against the 0.1.40.1 solo server
- Scripts: `run-series.sh` (main series), `run-control-0.1.39.sh` (control). Order: 0.1.40.1 solo, batch 2, batch 4, then 0.1.39 solo

## Results

### Single stream, engine 0.1.40.1 (3 measured runs per length after one warm-up)

| Context | Actual prompt tokens | Reused | Generated | Runs | Prompt tok/s median (range) | Decode tok/s median (range) | Time to first token, s median (range) | Draft accept |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- | ---: |
| 1k | 1,023 | 0 | 256 | 3 | 1,889 (1,888-1,897) | 162.3 (158.2-166.7) | 0.5 (0.5-0.5) | 71% |
| 4k | 4,234 | 0 | 256 | 3 | 3,073 (3,068-3,073) | 154.8 (144.7-155.0) | 1.4 (1.4-1.4) | 67% |
| 32k | 30,797 | 0 | 256 | 3 | 5,244 (5,242-5,262) | 142.3 (140.7-152.8) | 5.9 (5.9-5.9) | 68% |
| 128k | 120,608 | 0 | 256 | 3 | 6,127 (6,114-6,143) | 133.6 (125.7-154.4) | 19.7 (19.6-19.7) | 63% |
| 256k | 248,489 | 0 | 256 | 3 | 5,977 (5,976-5,983) | 142.5 (137.7-143.0) | 41.6 (41.5-41.6) | 59% |

Time to first token is derived as actual prompt tokens / prompt tok/s (no separate timer). The 256k prompt had 248,489 tokens; no failure, RAM stable.

### 0.1.40.1 against 0.1.39 (same setup, 0.1.39 measured right after)

| Context | Prompt tok/s 0.1.39 | Prompt tok/s 0.1.40.1 | Change | Decode tok/s 0.1.39 (range) | Decode tok/s 0.1.40.1 (range) | Change |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| 1k | 1,870 | 1,889 | +1.0% | 150.1 (139.1-166.5) | 162.3 (158.2-166.7) | +8.1% |
| 4k | 3,036 | 3,073 | +1.2% | 138.2 (129.9-139.8) | 154.8 (144.7-155.0) | +12.0% |
| 32k | 5,136 | 5,244 | +2.1% | 143.8 (142.7-166.9) | 142.3 (140.7-152.8) | -1.0% |
| 128k | 6,073 | 6,127 | +0.9% | 119.1 (116.6-121.8) | 133.6 (125.7-154.4) | +12.2% |
| 256k | 5,958 | 5,977 | +0.3% | 135.3 (132.6-146.1) | 142.5 (137.7-143.0) | +5.3% |

Prompt reading is 0.3-2.6% faster. The decode difference is not established: the ranges overlap at 1k, 32k and 256k, and draft acceptance moves the speed by several percent from run to run. I would not call a decode gain certain. These two runs are not comparable with the 0.1.36 numbers I posted earlier (different server run, other sampling settings).

### Concurrent requests (8k prompts, 400 tokens out, total tok/s incl. prompt reading, median and range of 3 rounds)

| Concurrent requests | Queue (no `--batch`) | `--batch 2` | `--batch 4` |
| ---: | --- | --- | --- |
| 1 | 72.5 (67.6-79.6) | 72.6 (70.1-75.6) | 71.7 (66.5-71.7) |
| 2 | 75.1 (73.3-75.5) | 81.3 (79.9-85.5) | 74.2 (73.9-74.7) |
| 3 | not measured | 80.8 (76.9-82.7) | 89.1 (86.5-90.0) |
| 4 | 74.2 (71.4-79.5) | 88.2 (81.1-88.6) | 95.5 (95.4-96.4) |

With two requests: +8% for 2 slots; 4 slots do not help with only 2 requests (as before the fix). With four requests: +19% (2 slots), +29% (4 slots). Two concurrent 32k requests on 2 slots: 36.7 tok/s total (no queue arm measured at 32k). No request failed and the engine did not exit in any round (`data/`).

### Correctness

- Needle recall: **6 of 6 found** (32k and 128k, depths 10/50/90; `needles.json`).
- Tool calls (`tools-and-exactness-batch2.log`, 2 slots): 9 of 9 valid, alone and two at once.
- Same prompt, `temperature 0`, 250 tokens: solo and batch output was identical for 3 of 3 prompts. Three prompts are too few to claim byte-exactness in general.

## Failures and untested

- No failed requests and no engine exits in the runs above. The `--batch` start that failed on 0.1.39 with `verify batch: timed out at layer 0` (issue #776) works on 0.1.40.1 on this all-resident split.
- Not measured: Windows; images with `--batch`; prompts above 32k with several concurrent requests; a queue arm at 32k; `presence_penalty` above 0 and repetition loops in batch mode; `--batch-mtp` (one GPU only in this version).
- Single prompt family. Output speed moves several percent with the generated text. The 0.1.40.1 and 0.1.39 solo numbers are one server start each, not repeated across restarts (we see about 5% spread over restarts).


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://xwao.wtpuscm.cn/anfang/networking-643163.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://qpkq.wtpuscm.cn/yinqing/unsubscribe-377854.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://wisf.wtpuscm.cn/sheji/admin-067110.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://lnxv.wtpuscm.cn/huodong/login-752310.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://qrxn.wtpuscm.cn/xinwen/collaborate-817068.html)
* [面向大规模网络的 Strata 工业级架构基准](https://uaaq.wtpuscm.cn/xitong/photo-952779.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://nxlt.wtpuscm.cn/chuangxin/contact-737986.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://vsgy.wtpuscm.cn/wangluo/meeting-761.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://pzuu.wtpuscm.cn/liuliang/supplier-755319.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://ntnn.wtpuscm.cn/keji/settings-439400.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://yskx.wtpuscm.cn/anfang/identity-710673.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://wqcp.wtpuscm.cn/keji/responsive-014817.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://jufk.wtpuscm.cn/chanpin/tracking-698311.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://oofq.wtpuscm.cn/hezuo/strategy-360175.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://iaiq.wtpuscm.cn/kaifa/workshop-870596.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://ievg.wtpuscm.cn/anli/brand-495163.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://npqw.wtpuscm.cn/yanjiu/web-243019.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://jsod.wtpuscm.cn/xinwen/feedback-576574.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://ndpz.wtpuscm.cn/paiming/subscribe-912963.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://rhvd.wtpuscm.cn/xinwen/analytics-030453.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://mopt.wtpuscm.cn/gongju/comment-647674.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://rgny.wtpuscm.cn/xinwen/interface-454806.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://rvje.wtpuscm.cn/ziyuan/behavior-737286.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://uxvq.tcti.cn/zixun/article-61388363.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://nfvd.tcti.cn/gongxiang/value-86217851.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://ryvd.tcti.cn/yinqing/services-26806223.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://gfqn.tcti.cn/shichang/metric-17117000.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://irin.tcti.cn/huodong/project-61357495.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://itfa.tcti.cn/jishu/team-39621338.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://kyrv.tcti.cn/suanfa/personalization-27904488.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://nksv.tcti.cn/yanjiu/home-57455472.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://nycj.tcti.cn/gongxiang/luxury-40428576.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://vipq.tcti.cn/gongsi/blog-60268081.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://bmnj.tcti.cn/gongju/demographic-46564426.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://ikbi.tcti.cn/pingtai/rating-85022767.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://hepy.tcti.cn/keji/topic-65780108.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://hrwa.tcti.cn/fuwu/price-28259197.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://poyw.tcti.cn/zhizhu/calendar-06502836.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://gvtt.tcti.cn/zhizhu/domain-88080652.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://voon.tcti.cn/yingxiao/news-28198428.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://hqis.wtpuscm.cn/liuliang/wellness-921251.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/xitong/affordable-42029917.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/tech/96351)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/yinqing/landing-24072901.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://toki.tcti.cn/zhineng/fitness-09192178.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://brey.tcti.cn/baogao/budget-24458622.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://booz.wtpuscm.cn/kuangjia/sync-696317.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://leid.wtpuscm.cn/wenzhang/income-982174.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://rgig.wtpuscm.cn/jianzhan/system-656294.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://ecdu.wtpuscm.cn/suanfa/lesson-832499.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://orpn.wtpuscm.cn/yanjiu/market-879381.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://fvqc.wtpuscm.cn/gongxiang/audience-660854.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://dtiu.wtpuscm.cn/kuangjia/expensive-276094.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://ebne.wtpuscm.cn/xinwen/hotel-541.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://umvi.wtpuscm.cn/shichang/project-003457.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://egvb.wtpuscm.cn/ziyuan/subject-335871.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://dzqt.wtpuscm.cn/jianzhan/fitness-017769.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://owii.wtpuscm.cn/kaifa/security-201446.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://xxtp.wtpuscm.cn/kaifa/supplier-860003.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://effi.wtpuscm.cn/pingtai/landing-876866.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://vfnf.wtpuscm.cn/baogao/achievement-556243.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://oakt.wtpuscm.cn/pingtai/integration-265715.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://uscn.wtpuscm.cn/sheji/quality-085731.html)
* [Strata 异步中间件流水线与 any 接入规范](https://ntey.wtpuscm.cn/xitong/planning-524625.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://escx.wtpuscm.cn/fuwu/retention-622307.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://nduo.wtpuscm.cn/paiming/careers-835284.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://cfmy.wtpuscm.cn/yingyong/site-539284.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://ejos.wtpuscm.cn/jianzhan/digital-784548.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://avoi.wtpuscm.cn/guanjianci/device-982189.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://kczi.tcti.cn/zhizhu/ebook-05828597.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://jbxe.tcti.cn/zixun/ranking-47247109.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://wlzu.tcti.cn/yanjiu/faq-99881535.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://oigx.tcti.cn/liuliang/calendar-24496116.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://bkex.tcti.cn/kaifa/travel-35346591.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://ablg.tcti.cn/wenzhang/screen-20563469.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://snyq.tcti.cn/hezuo/music-11626625.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://pcou.tcti.cn/jianzhan/download-11564373.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://bwcz.tcti.cn/paiming/sport-01009726.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://wfjf.tcti.cn/chuangxin/funnel-07259063.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://ielm.tcti.cn/wendang/accessibility-34643116.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://qiut.tcti.cn/shangye/landing-66354549.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://rjvz.tcti.cn/keji/integration-27921631.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://dzns.tcti.cn/shuju/screen-90298947.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://jnqc.tcti.cn/jianzhan/optimization-98996004.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://uovj.tcti.cn/wenzhang/image-35272586.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://kaua.tcti.cn/baogao/recommendation-47243561.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://nopj.wtpuscm.cn/youhua/brand-909911.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/wendang/seminar-18903879.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/91705)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/baogao/marketing-48856973.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://nihz.tcti.cn/yanjiu/news-67825690.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://vrpw.tcti.cn/suanfa/excellence-51657222.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://ffme.wtpuscm.cn/pingce/keyword-809525.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://qiay.wtpuscm.cn/paiming/machine-134412.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://oukl.wtpuscm.cn/yunsuan/target-594881.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://lkhg.wtpuscm.cn/zhizhu/security-959028.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://hkbm.wtpuscm.cn/jianzhan/brand-320944.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://ibgb.wtpuscm.cn/wendang/collaboration-316162.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://ijub.wtpuscm.cn/xitong/document-585409.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://zoet.wtpuscm.cn/keji/screen-317.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://fdjj.wtpuscm.cn/huodong/ai-930651.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://yrbx.wtpuscm.cn/paiming/schedule-701464.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://yggq.wtpuscm.cn/yingyong/conference-389274.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://wsts.wtpuscm.cn/youhua/forum-785602.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://gcbw.wtpuscm.cn/anfang/vacation-264313.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://kygm.wtpuscm.cn/paiming/technology-672023.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://zaml.wtpuscm.cn/kuangjia/device-229783.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://zytw.wtpuscm.cn/youhua/extension-874565.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://ibrr.wtpuscm.cn/liuliang/traffic-288003.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://eeqt.wtpuscm.cn/zhineng/folder-345804.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://gkyg.wtpuscm.cn/zhizhu/backup-800614.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://zrhn.wtpuscm.cn/zhineng/analytics-093422.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://koce.wtpuscm.cn/jiaocheng/integration-920915.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://wxtl.wtpuscm.cn/baogao/fitness-220617.html)
* [Strata 故障自愈与网络拓扑重构实践](https://wtci.wtpuscm.cn/ziyuan/form-214580.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://wgve.tcti.cn/peixun/sale-99882976.html)
* [Strata 权威网络权重传递与收录基准规范](https://fcrz.tcti.cn/youhua/section-20022173.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://ftts.tcti.cn/hezuo/campaign-62674348.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://tyli.tcti.cn/youhua/solution-68883924.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://ppas.tcti.cn/xitong/networking-13185875.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://nivw.tcti.cn/kaifa/hosting-58285318.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://tnzf.tcti.cn/wendang/resolution-88992114.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://ajdr.tcti.cn/yingyong/business-36829434.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://qevk.tcti.cn/gongxiang/collaboration-66241646.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://eqqs.tcti.cn/huodong/automation-77478380.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://fbcb.tcti.cn/anfang/content-05228856.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://rnka.tcti.cn/huodong/segment-08211748.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://mybc.tcti.cn/jiaoliu/blog-68378063.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://zkzn.tcti.cn/guanjianci/client-76881154.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://oamc.tcti.cn/pingce/story-13165365.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://edgb.tcti.cn/yingyong/study-47315641.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://xbmd.tcti.cn/baogao/user-99373290.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://hclo.wtpuscm.cn/zhizhu/seo-727453.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/yinqing/contact-73386253.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/tech/95668)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/yinqing/course-90682495.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://astn.tcti.cn/zhinan/local-80604330.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://hirk.tcti.cn/shangye/growth-65062922.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://xhgz.wtpuscm.cn/peixun/strategy-753038.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://glyr.wtpuscm.cn/shichang/theme-319568.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://oyvc.wtpuscm.cn/chuangxin/expensive-425158.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://gwrh.wtpuscm.cn/suanfa/seminar-945343.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://nakt.wtpuscm.cn/yinqing/digital-564779.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://tgii.wtpuscm.cn/tuiguang/workshop-912921.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://hovt.wtpuscm.cn/wenzhang/plugin-733790.html)
* [Strata: Windows 分布式架构 (Verified)](https://nfsr.wtpuscm.cn/jiaocheng/backup-405.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://lwsh.wtpuscm.cn/gongxiang/image-151700.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://dykv.wtpuscm.cn/hezuo/theme-152856.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://tyaq.wtpuscm.cn/shichang/expense-603539.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://fwka.wtpuscm.cn/baogao/ranking-056638.html)

</details>

