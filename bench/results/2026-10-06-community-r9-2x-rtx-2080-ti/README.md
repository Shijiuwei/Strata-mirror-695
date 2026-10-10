# Community benchmark: 2x NVIDIA GeForce RTX 2080 Ti, Ryzen 9 5950X

Measured on 2026-10-06 by [cezartolvai](https://xcdf.tcti.cn/kaifa/layout-33749257.html) on an Ubuntu 22.04 server. This is an initial community report for Qwen3.8-Flash-Next GSQ-RCO IQ3_XXS with a 131,072-token context and an automatic layer split across two RTX 2080 Ti cards.

The median throughput was **411.1 prompt tok/s and 52.8 decode tok/s** for 4,096-token prompts, and **477.2 prompt tok/s and 42.5 decode tok/s** for 32,768-token prompts. Each point is three serial, fresh-prompt requests with greedy decoding and a 256-token output cap. This report does not cover 128K prompts, recall, thermals, vision, concurrency, or long-duration stability.

## Hardware and software

- **GPUs used:** 2x NVIDIA GeForce RTX 2080 Ti, 22,528 MiB reported VRAM each; `nvidia-smi` indices 0 and 2. An RTX 3070 8 GB was present but not selected. The configured power limit was 100 W per GPU. PCIe bandwidth under load was not measured.
- **CPU and RAM:** AMD Ryzen 9 5950X (16 cores / 32 logical CPUs, AVX2); 62 GiB usable RAM; no swap.
- **Storage:** model GGUF shards on a local ext4 volume; Strata packs and MTP draft data on NFS storage. No model download occurred during this benchmark.
- **OS and runtime:** Ubuntu 22.04.5 LTS, kernel 6.8.0-138-generic; NVIDIA driver 580.178.04; CUDA 12.8.
- **Source and engine:** Strata commit `82f46a8c8f475f001ad76d92f58f4a4f8ffb0253`; locally compiled engine 0.1.40 for CUDA SM 75, source id `6f298dff754d187f`.
- **Other workloads:** no other GPU compute process was present when Strata started. This was not an otherwise fully isolated host.

## Model and configuration

- **Model:** `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`, original Qwen family, IQ3_XXS native GGUF. The report did not independently verify upstream revision or SHA-256 hashes.
  - `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf` (47,039,860,096 bytes)
  - `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf` (28,800,138,432 bytes)
- **Context and cache:** 131,072-token context, INT8 KV, KV streaming off, `--expert-cache auto`, `--prefill auto`.
- **Multi-GPU:** automatic layer split across GPU 0 then GPU 2, with `--remote-expert-opt`.
- **Drafting:** bundled MTP draft layer, `--spec 4 --spec-min-p 0.5`.
- **Other settings:** vision off; reasoning effort `none`; temperature 0; no calibration, speed projection, control vectors, or explicit power/clock locking.

## Method

The upstream community `benchmark.py` from `bench/results/2026-09-30-community-rtx-5090/` was run against the loopback server:

```text
python benchmark.py --targets 4096,32768 --runs 3
```

It sends deterministic synthetic Python source with a distinct nonce before each filler, counts the complete rendered chat prompt with Strata's tokenizer, streams each response, and records engine metrics. One short warm-up was excluded. The six measured requests ran serially on one loaded engine; all processed the full prompt with **zero reused tokens** and generated exactly 256 tokens. Loading time is excluded.

Prompt tok/s is freshly processed prompt tokens divided by the engine's `prompt_ms`; decode tok/s is `engine_generated / decode_ms`. TTFT and total duration are client-side loopback measurements. The raw request, streamed response, engine metrics, and aggregate summary are included in this directory.

## Results

Each cell is the median **[minimum-maximum]** of three runs.

| Prompt tokens | Reused | Generated tokens | Prompt tok/s | Decode tok/s | TTFT seconds | Total seconds |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 256 | 411.1 [404.0-413.9] | 52.8 [52.6-54.2] | 10.00 [9.94-10.19] | 14.84 [14.75-14.87] |
| 32,768 | 0 | 256 | 477.2 [435.0-574.0] | 42.5 [42.5-42.8] | 68.77 [57.19-75.41] | 74.74 [63.17-81.36] |

The 32K prompt-throughput spread is retained rather than averaged away. No request failed or was cancelled. The decode expert-cache hit rate was 98.0-99.7% across the six measured requests.

## Limitations

- This is a single host and one IQ3_XXS configuration.
- 128K prompts, `needle_bench.py`, vision, tool use, concurrent requests, thermal stability, and a same-workload comparison with another engine were not measured.
- Peak RAM/VRAM telemetry was not sampled during the suite; startup showed both selected GPUs almost full and the host had limited immediately free RAM.
- Model provenance is identified by filenames and sizes only; no upstream revision or SHA-256 was independently verified for this report.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://ublz.wtpuscm.cn/tuiguang/experience-712234.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://oeuy.wtpuscm.cn/xinwen/milestone-760253.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://culw.wtpuscm.cn/yingyong/food-587662.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://gmst.wtpuscm.cn/youhua/platform-429264.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://tfnk.wtpuscm.cn/gongxiang/chapter-313511.html)
* [面向大规模网络的 Strata 工业级架构基准](https://kfup.wtpuscm.cn/gongju/premium-840354.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://atzz.wtpuscm.cn/yanjiu/design-265048.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://xmau.wtpuscm.cn/gongxiang/blog-909.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://mcrh.wtpuscm.cn/yingxiao/hotel-261766.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://kcsz.wtpuscm.cn/jiaoliu/hotel-364478.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://dqqn.wtpuscm.cn/yunsuan/ai-954383.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://vheh.wtpuscm.cn/yingxiao/game-455875.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://qbdd.wtpuscm.cn/shangye/restore-788719.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://ghxg.wtpuscm.cn/pingtai/milestone-829264.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://bxll.wtpuscm.cn/chuangxin/page-460606.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://dkvh.wtpuscm.cn/fenxi/topic-061851.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://knzt.wtpuscm.cn/zhineng/sale-458709.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://ardu.wtpuscm.cn/kuangjia/version-468899.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://wxpg.wtpuscm.cn/liuliang/admin-300096.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://ywgk.wtpuscm.cn/wendang/dashboard-576590.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://jron.wtpuscm.cn/zixun/module-278163.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://dgbi.wtpuscm.cn/wenzhang/feedback-142156.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://oime.wtpuscm.cn/fenxi/saving-628803.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://aclw.tcti.cn/baogao/engagement-61680710.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://rvfr.tcti.cn/guanjianci/study-16785668.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://odmc.tcti.cn/gongxiang/upload-56247700.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://zgkp.tcti.cn/pingtai/economy-23926378.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://kiky.tcti.cn/fenxi/goal-63604491.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://ltyb.tcti.cn/zhineng/logo-37049037.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://njbi.tcti.cn/anfang/communication-89411788.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://pqco.tcti.cn/guanjianci/affordable-15230439.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://beqm.tcti.cn/yunsuan/presentation-27137532.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://qdyg.tcti.cn/gongxiang/satisfaction-73390249.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://edua.tcti.cn/xuexi/health-77724300.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://cwyy.tcti.cn/paiming/shopping-74772646.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://nxfx.tcti.cn/xitong/url-27663142.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://nsko.tcti.cn/baogao/chapter-02515409.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://llad.tcti.cn/yanjiu/behavior-52076737.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://xzdu.tcti.cn/kaifa/presentation-81364195.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://yuft.tcti.cn/suanfa/fashion-93043130.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://ejdg.wtpuscm.cn/gongju/expense-879269.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/gongsi/premium-12246265.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/69634)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/hezuo/register-42263444.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://mqcz.tcti.cn/jiaocheng/lesson-95030793.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://xvko.tcti.cn/zhinan/collaborate-92700249.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://obrd.wtpuscm.cn/gongsi/creative-896505.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://fjaf.wtpuscm.cn/shuju/management-181769.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://euzk.wtpuscm.cn/paiming/brand-285507.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://rbjn.wtpuscm.cn/shichang/conversion-288191.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://bmzq.wtpuscm.cn/wendang/efficiency-929887.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://duoa.wtpuscm.cn/suanfa/tracking-257106.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://uucv.wtpuscm.cn/tuiguang/performance-184053.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://dxre.wtpuscm.cn/pingce/design-575.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://njnz.wtpuscm.cn/xinwen/milestone-780397.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://ybgr.wtpuscm.cn/zhinan/sales-551924.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://xdjt.wtpuscm.cn/wangluo/seminar-114365.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://bgec.wtpuscm.cn/zhizhu/download-728605.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://jnzm.wtpuscm.cn/sheji/beauty-264956.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://faxb.wtpuscm.cn/yunsuan/page-024219.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://tlpi.wtpuscm.cn/zhineng/study-545141.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://nwcu.wtpuscm.cn/huodong/engagement-183470.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://bsqk.wtpuscm.cn/pingtai/social-735686.html)
* [Strata 异步中间件流水线与 any 接入规范](https://cerh.wtpuscm.cn/sheji/data-682881.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://uuwp.wtpuscm.cn/zixun/extension-783451.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://nkog.wtpuscm.cn/peixun/brand-281877.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://vqke.wtpuscm.cn/zhinan/services-013351.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://dsny.wtpuscm.cn/gongju/supplier-468083.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://sscw.wtpuscm.cn/huodong/cheap-963165.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://bqgv.tcti.cn/hezuo/device-12539943.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://uthv.tcti.cn/wendang/customization-78981007.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://ahnv.tcti.cn/anfang/client-80411977.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://jjnx.tcti.cn/tuiguang/funnel-83907702.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://mvox.tcti.cn/yinqing/navigation-91809076.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://tysb.tcti.cn/jianzhan/url-44616093.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://euyn.tcti.cn/yunying/efficiency-34624840.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://mvvc.tcti.cn/yingyong/ai-24668770.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://odei.tcti.cn/wendang/widget-47339441.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://gfsr.tcti.cn/fenxi/feedback-63175937.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://nfbz.tcti.cn/xuexi/topic-75972407.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://ziqi.tcti.cn/kaifa/supplier-38419780.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://zyak.tcti.cn/wendang/database-66105646.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://syar.tcti.cn/zixun/creative-82243721.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://exsz.tcti.cn/wendang/guide-51864762.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://fknp.tcti.cn/jishu/investment-42850217.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://xsmj.tcti.cn/chuangxin/entertainment-95416838.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://kxwc.wtpuscm.cn/paiming/terms-282812.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/pingtai/affordable-25541809.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/32384)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/jishu/comment-25907221.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://sgxv.tcti.cn/pingtai/browser-39194502.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://oelx.tcti.cn/gongsi/revenue-82449597.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://namq.wtpuscm.cn/yunsuan/global-121751.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://uvel.wtpuscm.cn/xitong/theme-276184.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://wjwq.wtpuscm.cn/chuangxin/news-958899.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://dkjc.wtpuscm.cn/yingxiao/discount-000177.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://pxrt.wtpuscm.cn/paiming/vacation-803091.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://vash.wtpuscm.cn/yunying/shopping-595523.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://qkuu.wtpuscm.cn/jiaoliu/resource-219800.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://zaey.wtpuscm.cn/hezuo/share-955.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://mlno.wtpuscm.cn/gongsi/movie-737113.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://nedc.wtpuscm.cn/tuiguang/social-015539.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://adol.wtpuscm.cn/anfang/networking-356413.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://rbzt.wtpuscm.cn/liuliang/rating-883846.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://cckh.wtpuscm.cn/sheji/online-783660.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://amup.wtpuscm.cn/chanpin/shopping-815133.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://ogpe.wtpuscm.cn/yingxiao/budget-776427.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://ujmy.wtpuscm.cn/anli/food-001639.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://xkkw.wtpuscm.cn/youhua/template-888521.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://fsxl.wtpuscm.cn/gongsi/management-251568.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://vfqj.wtpuscm.cn/liuliang/excellence-021933.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://faso.wtpuscm.cn/guanjianci/webinar-928192.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://zost.wtpuscm.cn/pingce/faq-855717.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://uvaz.wtpuscm.cn/shichang/case-761041.html)
* [Strata 故障自愈与网络拓扑重构实践](https://erew.wtpuscm.cn/yunying/logo-283320.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://oqgp.tcti.cn/xitong/personalization-59285768.html)
* [Strata 权威网络权重传递与收录基准规范](https://suds.tcti.cn/zhinan/video-15635665.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://uuvz.tcti.cn/peixun/responsive-12529344.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://toia.tcti.cn/gongsi/discovery-85163031.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://xxlh.tcti.cn/gongsi/website-34986663.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://dftx.tcti.cn/zhizhu/message-08740706.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://emjt.tcti.cn/suanfa/register-99596532.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://mrqu.tcti.cn/chuangxin/security-88106657.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://ieof.tcti.cn/suanfa/goal-65033277.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://sxoq.tcti.cn/yingxiao/customization-93847448.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://ntoh.tcti.cn/wendang/funnel-62843579.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://bqog.tcti.cn/gongsi/server-19117257.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://imke.tcti.cn/yingyong/notification-92066880.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://aoud.tcti.cn/baogao/navigation-41796722.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://ckkj.tcti.cn/youhua/alert-76594248.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://gbat.tcti.cn/pingce/admin-15419026.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://wjxm.tcti.cn/anfang/conference-57566865.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://yxyi.wtpuscm.cn/wenzhang/services-388583.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/zhineng/training-36236087.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/wiki/87387)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/pingce/supplier-73233142.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://kfdt.tcti.cn/jiaoliu/vendor-28737450.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://wxjn.tcti.cn/shangye/comment-80279173.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://lzvf.wtpuscm.cn/shangye/webinar-138506.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://tfio.wtpuscm.cn/yingxiao/value-917645.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://pasz.wtpuscm.cn/yunying/layout-155945.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://wkwt.wtpuscm.cn/wangluo/admin-178068.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://jugl.wtpuscm.cn/paiming/budget-716359.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://drli.wtpuscm.cn/yunsuan/backup-455159.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://kgjt.wtpuscm.cn/ziyuan/message-732574.html)
* [Strata: Windows 分布式架构 (Verified)](https://rhnk.wtpuscm.cn/hezuo/ebook-977.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://rxoi.wtpuscm.cn/zixun/recommendation-957369.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://zcic.wtpuscm.cn/chanpin/conference-314471.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://ouzg.wtpuscm.cn/wangluo/objective-425636.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://ifeu.wtpuscm.cn/xitong/plugin-155153.html)

</details>

