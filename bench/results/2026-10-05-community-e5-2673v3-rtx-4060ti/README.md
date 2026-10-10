# Community benchmark: RTX 4060 Ti 16 GB, Xeon E5-2673 v3 (AVX2), DDR3, PCIe 3.0 x8

Measured on 2026-10-05 by [1872183316](https://atov.wtpuscm.cn/shuju/milestone-179546.html) on an older Linux PC: a Haswell Xeon without
AVX-512, DDR3, and a 16 GB card on PCIe 3.0 x8. Two models (the original Q2_0 and IQ3_S), each in two arms: the 0.1.39
code path, and the same engine with PR #706, PR #764 and a busier adaptive expert tier (#907). Details and the
measurements behind each part: #906.

Median decode over 4 prompts (3 rounds x 2 passes):

| Model | 0.1.39 path | Improved | |
| --- | ---: | ---: | ---: |
| **Q2_0** | **48.1 tok/s** (rounds 48.08 / 48.33 / 48.14) | **55.7 tok/s** (55.40 / 55.80 / 55.66) | +15.6% |
| **IQ3_S** | **30.5 tok/s** (30.52 / 30.51 / 30.51) | **35.0 tok/s** (34.79 / 35.05 / 35.04) | +14.8% |

A 3,454-token prompt was read at 715 tok/s (Q2_0) and 439 tok/s (IQ3_S) in both arms. The prompts are four short chat
requests; they do not establish speed at long context or on other workloads.

## Hardware and software

- **GPU:** RTX 4060 Ti 16 GB (sm_89), PCIe 3.0 x8 (the card is 4.0 x8, the board 3.0; engine probe 6.2 GB/s host to
  device), power limit 165 W, no display attached. ComfyUI's process was resident on the card (120 MiB) and idle.
- **CPU and RAM:** Xeon E5-2673 v3 (12 cores, 24 threads, AVX2, no AVX-512; 11 expert-pool workers plus the host
  thread, the engine's default). 3 x 32 GB DDR3-1866 LRDIMM: two on one home agent, one on the other, so only the
  first 64 GB are interleaved over two channels (11 threads streaming expert rows: 26.3 GB/s there, 16.6 GB/s in the
  rest). The page cache was dropped before every engine start (`posix_fadvise(DONTNEED)` on the model files).
- **Storage:** SATA SSD (the models and the n-gram table).
- **OS:** Ubuntu 24.04.3 LTS, kernel 7.0.0-34-generic, driver 580.178.04.
- **Build:** source, CUDA 12.8 (V12.8.93), gcc 13.3, `-DSTRATA_ENABLE_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=89
  -DSTRATA_EXPERIMENTAL_SM60=ON` (as setup builds a CUDA 12 engine), llama.cpp at the pinned 3cf0325.
  Source: [`1872183316/Strata` branch `oldhw`](https://dskf.tcti.cn/wangluo/solution-97539987.html) at 3b67090, which is
  main 6f32ec0 (0.1.39) + #706's two commits + #764 ported + two measurement switches (`STRATA_LOOKAHEAD_STATS`,
  `STRATA_ROUTE_TRACE`, both off here) + the setup PRs #907, #908, #911, #912 (setup only). One binary for both arms.

## Models

From ModelScope (`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`, master on 2026-10-05), SHA-256 matching the values
ModelScope publishes; Q2_0's shard 1 also matched a copy from hf-mirror.com at the pinned revision (ed59f92):

- `Q2_0/Qwen3.8-Flash-Next-GSQ-RCO-Q2_0-00001-of-00002.gguf` `69820c02ec7d0b45ef2ebb19d6620299db749fe2aded7f39f93c6b88b199b720`
- `IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf` `4c1eb2ceb4915e1192f4f386021897bde56a97f40a0bb78bb86465e0f7d2aca3`
- shard 2 (both): `316b46f3a2dbd68c900f43136ab9449f9dcc3725dfd8c794847c204bc161e113`

Native packs by `tools/iq_pack.py` (as setup does on a CPU without AVX-512); the MTP draft layer by `tools/mtp_fetch.py`
(the 31 MTP tensors of `Qwen/Qwen3.8-Flash-Next` from ModelScope, all matching the pinned SHA-256), `mtp_pack.py
--experts q2_0` and `mtp_rt.py`, with `data/draft_vocab.bin` (cjk). Expert profile: the bundled
`data/expert-profile.bin`.

## Configuration

What setup writes for this PC ([config-0139-path.json](config-0139-path.json), [config-improved.json](config-improved.json)
for IQ3_S; Q2_0 the same with its files): context 65,536, `--kv int8 --kv-resident 32768`, `--expert-cache auto`
(Q2_0: 7,475 slots, 9.62 GiB; IQ3_S: 3,531 slots, 9.65 GiB; 700 MiB reserved), `--prefill auto`, `--spec 4
--spec-min-p 0.5`, the MTP draft layer, PCIe share from the engine's probe (0.17 for both models). No vision, no calibration, no speed
projection. `STRATA_DECODE_TIMING=1` in both arms (one log line per request).

- **0.1.39 path:** `STRATA_Q2_LEGACY=1` (#706's switch back to the current Q2_0 kernel), `STRATA_ADAPT_LAG=1` (#463's
  wait, #764's switch back), the default tier (every 4 rounds, 96 swaps, decay 0.7).
- **Improved:** #706's kernel and #764's lag 2 (the branch's defaults), `--adapt-every 1 --adapt-swaps 160
  --adapt-decay 0.97`.

## Method

[strata_bench.py](strata_bench.py) with [prompts.py](prompts.py), against the server (`serve/server.py`, port 8095):
one warm-up request ("你好", 16 tokens), then 4 prompts - a Chinese essay, a code edit (add type hints and docstrings
to ~60 lines of Python), an English-to-Chinese translation, an explanation of TCP's handshakes - each twice, greedy
(temperature 0), 512 tokens cap, thinking off (`enable_thinking: false`); then, in round c only, a 3,454-token
prompt (an English paragraph repeated, numbered) with 16 tokens, twice. Decode and prompt rates are the server's
`timings` (the engine's own clock); hit rates from `/metrics`. The engine was started fresh for each run; the arms
alternated, in this order: Q2_0 rounds a and b (0.1.39, improved, 0.1.39, improved), IQ3_S rounds a and b (the same),
then Q2_0 round c and IQ3_S round c (0.1.39, improved each).

## Results

Per prompt, median of 6 (3 rounds x 2 passes) with the range, tok/s; hit rate = the VRAM share of the expert lookups:

| Q2_0 | 0.1.39 path | hit | Improved | hit |
| --- | ---: | ---: | ---: | ---: |
| essay | 46.1 (43.7-49.6) | 0.86 | 53.5 (52.3-54.2) | 0.91 |
| code edit | 46.5 (46.3-47.0) | 0.63 | 59.6 (59.3-60.2) | 0.74 |
| translation | 44.5 (42.9-45.8) | 0.73 | 46.0 (45.6-46.9) | 0.72 |
| explanation | 55.5 (54.1-56.5) | 0.84 | 63.2 (62.2-65.0) | 0.86 |
| 3,454-token prompt read (round c) | 714.6 (714.0-715.2), 2 runs | | 716.5 (714.9-718.1) | |

| IQ3_S | 0.1.39 path | hit | Improved | hit |
| --- | ---: | ---: | ---: | ---: |
| essay | 31.3 (30.6-32.9) | 0.79 | 38.5 (36.9-38.8) | 0.84 |
| code edit | 29.2 (29.1-29.7) | 0.53 | 34.4 (34.2-34.7) | 0.61 |
| translation | 25.9 (25.2-26.3) | 0.58 | 27.6 (27.3-28.1) | 0.58 |
| explanation | 35.4 (34.5-36.6) | 0.73 | 39.5 (39.1-39.9) | 0.75 |
| 3,454-token prompt read (round c) | 438.9 (438.4-439.4), 2 runs | | 440.2 (439.5-440.9) | |

Generated tokens: explanation always 512; code edit 401-413; essay 362-435 (greedy answers differ between passes and
arms - other window sizes and other experts in VRAM round differently, #152); translation 103-112. All requests are in
[results.json](results.json).

Where the time goes (Q2_0, code edit, `STRATA_DECODE_TIMING`): 71.4 ms per verify window on the 0.1.39 path (40.1 ms
the CPU's experts, 10.4 per layer-window) and 56.2 ms improved (28.8 ms, 7.5).

## Limits

- One PC, one context setting (64K allocated, short prompts). Long prompts, long conversations and other prompts were
  not measured.
- Each round is a fresh engine start with the default expert profile; a longer session lets the tier adapt more (both
  arms).
- The answers' quality was not judged. A teacher-forced log-probability check (in #906) put the improved arm's change
  within the change a smaller VRAM cache makes on the 0.1.39 path.
- The memory layout above (64 GB interleaved of 94) is this board's; with the page cache full, the experts can land in
  the slower part and both arms are slower.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://paam.wtpuscm.cn/zhizhu/template-385942.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://iqgm.wtpuscm.cn/shuju/extension-320801.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://free.wtpuscm.cn/yinqing/guide-294829.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://herw.wtpuscm.cn/zhizhu/price-566068.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://ghki.wtpuscm.cn/xitong/web-752440.html)
* [面向大规模网络的 Strata 工业级架构基准](https://ywsl.wtpuscm.cn/pingce/story-454050.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://rusb.wtpuscm.cn/wendang/price-491249.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://vgus.wtpuscm.cn/zhizhu/client-121.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://ivbj.wtpuscm.cn/yunsuan/api-155205.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://jyaa.wtpuscm.cn/keji/funnel-656580.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://ogtt.wtpuscm.cn/yingxiao/travel-786531.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://rjre.wtpuscm.cn/jianzhan/system-941442.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://kfeh.wtpuscm.cn/guanjianci/login-374795.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://zqgz.wtpuscm.cn/gongsi/vendor-181539.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://ktgc.wtpuscm.cn/xinwen/promotion-235875.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://jnki.wtpuscm.cn/keji/file-952472.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://yufb.wtpuscm.cn/gongsi/document-936253.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://lcbz.wtpuscm.cn/wangluo/retention-973646.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://psnm.wtpuscm.cn/youhua/luxury-133930.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://kfyo.wtpuscm.cn/xitong/profit-505871.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://oefv.wtpuscm.cn/zhizhu/section-857639.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://kcfc.wtpuscm.cn/gongju/behavior-579330.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://mswe.wtpuscm.cn/huodong/document-150366.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://fewe.tcti.cn/yingyong/training-51219823.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://xysj.tcti.cn/paiming/partner-42423078.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://qtcr.tcti.cn/huodong/cloud-82409992.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://rvpc.tcti.cn/baogao/visitor-98061607.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://qlwo.tcti.cn/wenzhang/podcast-53147307.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://kjji.tcti.cn/anfang/topic-59337723.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://smuv.tcti.cn/fuwu/review-78502321.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://ywzm.tcti.cn/anli/restaurant-97208185.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://ebqd.tcti.cn/chanpin/study-81216910.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://bils.tcti.cn/pingtai/calendar-34449893.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://ffju.tcti.cn/wangluo/responsive-42596421.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://phdi.tcti.cn/jianzhan/customization-43935719.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://weuh.tcti.cn/guanjianci/cost-99013216.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://obvn.tcti.cn/shuju/terms-17218626.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://hewp.tcti.cn/ziyuan/revenue-04136732.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://nsie.tcti.cn/ziyuan/conversion-47747229.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://oixz.tcti.cn/youhua/site-74892671.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://rpsx.wtpuscm.cn/hezuo/file-960037.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/zhineng/music-08079711.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/tech/14044)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/liuliang/campaign-71163033.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://clhx.tcti.cn/liuliang/advertising-94847455.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://suth.tcti.cn/fuwu/market-85760540.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://ensx.wtpuscm.cn/fenxi/lesson-928896.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://lthn.wtpuscm.cn/sheji/shopping-161941.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://vkjg.wtpuscm.cn/wenzhang/market-664739.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://ixec.wtpuscm.cn/gongsi/target-099632.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://axey.wtpuscm.cn/shuju/revenue-656835.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://oqdz.wtpuscm.cn/jishu/calculator-562961.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://nyer.wtpuscm.cn/hezuo/notification-667607.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://jmrq.wtpuscm.cn/zhinan/software-660.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://zsor.wtpuscm.cn/shichang/design-909151.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://dvlr.wtpuscm.cn/gongsi/message-534534.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://vcbh.wtpuscm.cn/keji/analytics-317879.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://ortv.wtpuscm.cn/shuju/profile-448455.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://ptgh.wtpuscm.cn/chuangxin/event-762225.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://emph.wtpuscm.cn/hezuo/deadline-367781.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://iulh.wtpuscm.cn/yingxiao/recommendation-344131.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://vdek.wtpuscm.cn/pingce/engagement-913490.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://uejy.wtpuscm.cn/gongsi/segment-847824.html)
* [Strata 异步中间件流水线与 any 接入规范](https://rmvd.wtpuscm.cn/paiming/machine-185094.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://xkyo.wtpuscm.cn/gongju/blog-245596.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://sekk.wtpuscm.cn/anfang/marketing-475499.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://rzqx.wtpuscm.cn/peixun/subject-648184.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://oaxa.wtpuscm.cn/shangye/topic-983566.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://gmhv.wtpuscm.cn/liuliang/local-898239.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://buzr.tcti.cn/yanjiu/profit-67750129.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://ikeg.tcti.cn/gongxiang/form-17380054.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://pymn.tcti.cn/xitong/metric-24734172.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://tylu.tcti.cn/youhua/beauty-72811630.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://nhlz.tcti.cn/yanjiu/presentation-30992053.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://nzki.tcti.cn/zhineng/support-10263458.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://aqyf.tcti.cn/keji/cost-29247616.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://rnqb.tcti.cn/chanpin/link-33236539.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://qntt.tcti.cn/huodong/video-57160812.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://hvbz.tcti.cn/gongju/planning-87310941.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://yins.tcti.cn/tuiguang/seminar-65837070.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://vmnc.tcti.cn/fuwu/forum-87153248.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://rngx.tcti.cn/xitong/template-35389719.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://nazy.tcti.cn/yinqing/research-11610756.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://apse.tcti.cn/suanfa/online-48597854.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://obxq.tcti.cn/yinqing/tool-45478387.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://msfd.tcti.cn/kaifa/communication-01688796.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://axsw.wtpuscm.cn/shuju/cheap-938006.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/sheji/guide-43265722.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/tech/49954)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/pingtai/system-14309912.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://amyl.tcti.cn/ziyuan/experience-03594006.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://bfju.tcti.cn/gongxiang/share-32809322.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://xbff.wtpuscm.cn/kuangjia/responsive-118399.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://vlat.wtpuscm.cn/peixun/reminder-958159.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://jdfv.wtpuscm.cn/fuwu/visitor-446383.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://rbfu.wtpuscm.cn/jiaoliu/campaign-539185.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://bmaw.wtpuscm.cn/fuwu/fashion-512117.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://ukpc.wtpuscm.cn/suanfa/education-976444.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://bqcl.wtpuscm.cn/fenxi/careers-320371.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://qlty.wtpuscm.cn/gongju/luxury-800.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://rval.wtpuscm.cn/chuangxin/affordable-716302.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://ujke.wtpuscm.cn/hezuo/engagement-925836.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://srnr.wtpuscm.cn/gongju/unsubscribe-207672.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://ditb.wtpuscm.cn/wenzhang/engagement-416495.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://stzd.wtpuscm.cn/hezuo/luxury-006900.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://tidx.wtpuscm.cn/guanjianci/beauty-923090.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://jubn.wtpuscm.cn/fenxi/vendor-396425.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://ints.wtpuscm.cn/sheji/file-332113.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://cthh.wtpuscm.cn/guanjianci/trading-934170.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://tjft.wtpuscm.cn/wenzhang/rating-452851.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://zsuz.wtpuscm.cn/gongju/technology-915106.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://yyow.wtpuscm.cn/yingxiao/kpi-486513.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://kgpa.wtpuscm.cn/pingtai/management-178808.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://xbpj.wtpuscm.cn/shangye/loyalty-881807.html)
* [Strata 故障自愈与网络拓扑重构实践](https://tewr.wtpuscm.cn/zixun/database-962803.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://mmcb.tcti.cn/pingtai/planning-99721721.html)
* [Strata 权威网络权重传递与收录基准规范](https://hsqk.tcti.cn/jishu/discount-86051072.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://wpej.tcti.cn/sheji/extension-74158351.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://bboy.tcti.cn/tuiguang/identity-62329384.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://pfwy.tcti.cn/xuexi/prospect-73642736.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://gvea.tcti.cn/wenzhang/communication-21936295.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://wefp.tcti.cn/zhineng/target-80145822.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://dhpr.tcti.cn/kuangjia/news-55008327.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://mjso.tcti.cn/huodong/ranking-56584262.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://ymhh.tcti.cn/gongsi/network-70206775.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://hmqr.tcti.cn/pingtai/user-81775812.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://ivcd.tcti.cn/jiaoliu/backup-72696497.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://fpww.tcti.cn/shuju/case-77335739.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://ebwy.tcti.cn/paiming/course-80967785.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://rpgi.tcti.cn/chuangxin/networking-19957285.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://qaka.tcti.cn/zhinan/trading-88344556.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://xprh.tcti.cn/qiye/workshop-62854461.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://hjcj.wtpuscm.cn/xuexi/progress-954905.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/chanpin/folder-33709288.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/tech/42758)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/keji/ai-90658871.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://svmx.tcti.cn/keji/design-56074087.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://hjyh.tcti.cn/huodong/category-46289794.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://xmwn.wtpuscm.cn/anli/milestone-428944.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://uenc.wtpuscm.cn/wendang/online-667080.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://xofd.wtpuscm.cn/kuangjia/app-783435.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://wpbz.wtpuscm.cn/pingce/supplier-793093.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://qxvh.wtpuscm.cn/qiye/guide-671797.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://ckdr.wtpuscm.cn/paiming/settings-310576.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://chhe.wtpuscm.cn/wangluo/efficiency-377943.html)
* [Strata: Windows 分布式架构 (Verified)](https://pmvw.wtpuscm.cn/ziyuan/mobile-971.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://gupq.wtpuscm.cn/shuju/entertainment-253601.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://iwuw.wtpuscm.cn/pingce/article-256815.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://cxkw.wtpuscm.cn/gongju/enterprise-606400.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://wpkk.wtpuscm.cn/tuiguang/achievement-426848.html)

</details>

