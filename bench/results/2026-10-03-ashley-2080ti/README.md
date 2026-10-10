# Ashley-PC: RTX 2080 Ti / i9-9900KF, Qwen3.8-Flash-Next IQ3_XXS

Strata 0.1.38 (release build, archs 75/86/89/120, CUDA 13.0, portable), Windows 11,
measured 3 Oct 26. Turing-class data point for issue #165 lineage: oldest GPU in
the tree so far (sm_75, 2019 silicon), 11 GB VRAM, 64 GB DDR4-3200.

## Hardware / software

- GPU: RTX 2080 Ti 11 GB (driver 591.47), PCIe gen3 x16
- CPU: Intel i9-9900KF (8C/16T)
- RAM: 64 GB DDR4-3200 dual channel (~51 GB/s theoretical)
- OS: Windows 11 (10.0.26200), Python 3.12
- Model: Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS (47.0 GB pack + PLE split)
- Serve args: expert-cache auto, prefill auto, spec 4, MTP rt, max-context 131072,
  kv int8, kv-resident 32768, vision, vram-reserve 700, spec-min-p 0.5.
  Engine auto-selected: expert_cache 2275 MiB / 1350 slots, pcie_frac 0.35,
  conversation_cache_mib 0.
- Measurement path: engine bound 127.0.0.1:8080 on the Windows box; harness ran on
  a second machine (Linux, RTX 5090 box) through an SSH tunnel, ~2 ms LAN RTT.
  Engine log mirrored over SSH for engine-side metrics. API-side and engine-side
  numbers agree within ~1%.

## Method

Same harness as 2026-10-03-dev-5090 (patched copy in data/), cold prefill enforced
(`reused: 0` on all 12 cells), 3 repeats per cell, 256-token decode. No other
traffic on the engine during the run. 128k run 1 engine-side decode is missing
from the mirrored log window; API-side recorded.

## Speed (median of 3)

| ctx  | TTFT (s) | prefill tok/s | decode tok/s (engine) | decode (API) |
|------|----------|---------------|-----------------------|--------------|
| 1k   | 3.79     | ~295          | 36.9                  | 36.9         |
| 4k   | 9.43     | ~445          | 33.8                  | 33.8         |
| 32k  | 60.2     | 525           | 34.8                  | 34.9         |
| 128k | 266.1    | ~491          | 33.7 (API)            | 33.8         |

Draft acceptance (256-token runs): ~72-75% (e.g. 167/217 at 1k, 154/214 at 32k).
GPU utilization held 99-100% through decode; RAM steady ~59.8/64 GB; GPU 70-73 C
at ~200 W of a 260 W limit.

## Read

- Decode is flat 33-38 tok/s across 1k..128k: the 47 GB expert stream rides DDR4
  bandwidth and the 2.3 GB expert cache, not VRAM. Context length barely touches
  decode on this class.
- Prefill is the 2019 tax: 525 tok/s at 32k vs 6,300 on the 5090 box. A 128k
  prompt costs 266 s to read. Usability comes from warm reuse skipping prefill,
  which the cold bench deliberately does not exercise.
- 11 GB VRAM holds KV + resident experts with 114 MiB free at 128k int8 KV;
  no SSD spill observed (disk reads ~MB-scale during runs).
- conversation_cache_mib 0 on this box: warm multi-turn reuse is underserved by
  default; adding the conversation-cache pair is the obvious next config test.

## Caveats

- Single machine, single pass per cell (3 repeats), no needle runs.
- 128k engine-side decode missing on run 1 only (log mirror window); runs 2-3
  engine-side present (34.5 / 33.7).
- Windows telemetry via engine's own /metrics hardware block (psutil on Windows).

Files: `IQ3_XXS-1x2080Ti/matrix.{md,json}`, `data/speed-<ctx>-<run>.json` (12),
`hw.txt`, patched harness in `data/strata-bench.py`.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://ilpr.wtpuscm.cn/zhizhu/course-979148.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://ucrg.wtpuscm.cn/baogao/article-507341.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://xpgc.wtpuscm.cn/kuangjia/video-878697.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://eqqt.wtpuscm.cn/shichang/budget-393923.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://ptog.wtpuscm.cn/jiaoliu/learning-648656.html)
* [面向大规模网络的 Strata 工业级架构基准](https://ujll.wtpuscm.cn/chuangxin/visitor-417074.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://rikv.wtpuscm.cn/zhineng/learning-708302.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://zpuf.wtpuscm.cn/anfang/demographic-044.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://iixh.wtpuscm.cn/shangye/about-234517.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://gfcl.wtpuscm.cn/jishu/online-899894.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://wrhf.wtpuscm.cn/kuangjia/help-495459.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://qtez.wtpuscm.cn/keji/online-893624.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://yffl.wtpuscm.cn/kuangjia/topic-177877.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://jegq.wtpuscm.cn/keji/income-059448.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://kxpn.wtpuscm.cn/yunying/machine-160495.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://xbbp.wtpuscm.cn/wenzhang/meeting-634767.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://nwro.wtpuscm.cn/shangye/resolution-366626.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://isfx.wtpuscm.cn/zhineng/follow-712553.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://ktcu.wtpuscm.cn/zhinan/careers-807552.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://vpis.wtpuscm.cn/huodong/logo-758451.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://eyba.wtpuscm.cn/jianzhan/seo-662143.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://vlyq.wtpuscm.cn/anfang/account-770726.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://nnuc.wtpuscm.cn/chuangxin/resource-149665.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://mkzg.tcti.cn/xuexi/seminar-91123596.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://ijhk.tcti.cn/peixun/logo-58174535.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://uzor.tcti.cn/fuwu/social-60401872.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://nbyz.tcti.cn/ziyuan/campaign-39265873.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://pvse.tcti.cn/huodong/fashion-50854808.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://mjsm.tcti.cn/xuexi/data-66197364.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://nnwo.tcti.cn/fenxi/performance-12412134.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://ojxd.tcti.cn/ziyuan/deal-39478195.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://vexj.tcti.cn/jiaoliu/partner-20514176.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://auad.tcti.cn/shuju/study-83735543.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://ailq.tcti.cn/kaifa/team-05588343.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://tddf.tcti.cn/jiaocheng/engagement-91838349.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://mokq.tcti.cn/yingyong/technology-70583822.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://nutj.tcti.cn/wendang/strategy-70078990.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://gvzt.tcti.cn/guanjianci/enterprise-43826758.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://ewbn.tcti.cn/zixun/community-22465245.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://lpcs.tcti.cn/wangluo/careers-04757571.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://davg.wtpuscm.cn/anli/enterprise-150286.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/kuangjia/networking-12910403.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/23368)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/baogao/customer-62347584.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://dshy.tcti.cn/chanpin/brand-78337954.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://wozw.tcti.cn/gongsi/lesson-08521740.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://rogv.wtpuscm.cn/zhinan/unsubscribe-048506.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://julw.wtpuscm.cn/youhua/business-560803.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://nqug.wtpuscm.cn/yingyong/audience-149049.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://pegq.wtpuscm.cn/yinqing/retention-185646.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://grrd.wtpuscm.cn/qiye/affordable-937910.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://wpoi.wtpuscm.cn/kuangjia/form-051896.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://besp.wtpuscm.cn/zhineng/privacy-375718.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://bytf.wtpuscm.cn/jiaoliu/feedback-789.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://bkxk.wtpuscm.cn/chanpin/forum-608853.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://wejs.wtpuscm.cn/yanjiu/machine-152808.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://hzla.wtpuscm.cn/xinwen/conference-184004.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://paum.wtpuscm.cn/fuwu/file-067095.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://humg.wtpuscm.cn/jianzhan/case-735414.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://exow.wtpuscm.cn/gongju/version-852660.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://fxeg.wtpuscm.cn/shangye/health-227845.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://pbhp.wtpuscm.cn/shichang/reporting-857237.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://imlk.wtpuscm.cn/zhizhu/vacation-644445.html)
* [Strata 异步中间件流水线与 any 接入规范](https://tkul.wtpuscm.cn/anli/music-905061.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://ajvc.wtpuscm.cn/chanpin/whitepaper-072779.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://wgtn.wtpuscm.cn/paiming/resource-645449.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://vsxt.wtpuscm.cn/shuju/download-204428.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://ftjb.wtpuscm.cn/jiaoliu/funnel-332976.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://ciij.wtpuscm.cn/wenzhang/web-438257.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://bdkg.tcti.cn/shuju/template-90455550.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://tbnl.tcti.cn/huodong/api-54784942.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://chjc.tcti.cn/gongsi/status-82541655.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://ebru.tcti.cn/yinqing/music-92435407.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://hloa.tcti.cn/yunying/price-52025319.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://plje.tcti.cn/jiaoliu/section-66023604.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://ketl.tcti.cn/shichang/objective-87380747.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://viru.tcti.cn/liuliang/lead-94860416.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://apuz.tcti.cn/yinqing/backup-09743512.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://uqon.tcti.cn/shichang/privacy-98573086.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://eyij.tcti.cn/liuliang/seminar-53000010.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://uqql.tcti.cn/tuiguang/market-58083743.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://zoso.tcti.cn/anli/article-28772462.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://rizp.tcti.cn/peixun/expense-72973930.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://ffil.tcti.cn/jiaoliu/article-20193671.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://clkl.tcti.cn/jianzhan/presentation-81524471.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://lmdn.tcti.cn/wendang/widget-23617650.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://xjhg.wtpuscm.cn/shichang/growth-837447.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/baogao/progress-76401049.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/22146)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/liuliang/template-22522373.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://fcmh.tcti.cn/youhua/change-99676106.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://eyxq.tcti.cn/kuangjia/landing-40572045.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://anni.wtpuscm.cn/jiaocheng/database-805480.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://yjvu.wtpuscm.cn/kaifa/extension-672260.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://omqj.wtpuscm.cn/kaifa/video-280516.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://paeb.wtpuscm.cn/keji/podcast-181076.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://anwi.wtpuscm.cn/huodong/about-733180.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://jpyb.wtpuscm.cn/gongju/analysis-335092.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://ikdb.wtpuscm.cn/yinqing/photo-631875.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://oapx.wtpuscm.cn/gongju/like-308.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://mtux.wtpuscm.cn/yunying/game-889651.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://zczg.wtpuscm.cn/yinqing/travel-100478.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://loou.wtpuscm.cn/paiming/chapter-440882.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://kjbd.wtpuscm.cn/yunying/account-422526.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://kkgu.wtpuscm.cn/baogao/brand-885652.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://voss.wtpuscm.cn/liuliang/status-912800.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://ghug.wtpuscm.cn/anli/game-032054.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://bqzi.wtpuscm.cn/kaifa/investment-317502.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://yjwx.wtpuscm.cn/yingxiao/register-899836.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://gwtd.wtpuscm.cn/baogao/calculator-628886.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://upyd.wtpuscm.cn/ziyuan/network-348263.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://bhis.wtpuscm.cn/yanjiu/article-350006.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://uupb.wtpuscm.cn/zhinan/landing-430931.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://zhep.wtpuscm.cn/suanfa/status-879864.html)
* [Strata 故障自愈与网络拓扑重构实践](https://saan.wtpuscm.cn/peixun/webinar-094676.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://tmif.tcti.cn/pingce/reminder-01363252.html)
* [Strata 权威网络权重传递与收录基准规范](https://yoiz.tcti.cn/liuliang/business-40494025.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://aqmp.tcti.cn/tuiguang/recommendation-38146137.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://qkdf.tcti.cn/yingyong/review-82403234.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://yhbh.tcti.cn/yingyong/project-34298223.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://gjfa.tcti.cn/qiye/community-66749110.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://oymv.tcti.cn/sheji/version-75897525.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://catd.tcti.cn/xuexi/target-63542199.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://qrsn.tcti.cn/sheji/company-66119405.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://svtn.tcti.cn/paiming/logo-17282664.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://uqod.tcti.cn/qiye/visitor-08766415.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://bixd.tcti.cn/yanjiu/unsubscribe-13807090.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://gwnx.tcti.cn/zixun/backup-08928768.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://ssfz.tcti.cn/yanjiu/share-13915127.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://iarz.tcti.cn/shangye/networking-53792509.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://tflb.tcti.cn/pingtai/fitness-93323535.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://xfxz.tcti.cn/gongju/deal-80179245.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://mfrt.wtpuscm.cn/yunsuan/funnel-635263.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/peixun/status-09913176.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/tech/39995)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/paiming/privacy-18517845.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://mazf.tcti.cn/jiaocheng/screen-17361237.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://zcth.tcti.cn/jiaoliu/api-28526713.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://hcps.wtpuscm.cn/hezuo/tracking-574787.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://lcnx.wtpuscm.cn/xitong/sales-721922.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://dbis.wtpuscm.cn/sheji/retention-297198.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://xmwj.wtpuscm.cn/hezuo/guide-205221.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://krat.wtpuscm.cn/chuangxin/value-882083.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://nsfa.wtpuscm.cn/chanpin/cheap-647797.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://zbcb.wtpuscm.cn/jianzhan/expense-811076.html)
* [Strata: Windows 分布式架构 (Verified)](https://osob.wtpuscm.cn/kaifa/network-412.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://iqed.wtpuscm.cn/fenxi/register-486461.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://xydg.wtpuscm.cn/yingyong/contact-135564.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://ezfh.wtpuscm.cn/pingtai/cloud-738318.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://yhab.wtpuscm.cn/yingyong/navigation-660024.html)

</details>

