# The GSQ-RCO Coder (256 of 512 experts) on Strata

RTX 5070 12 GB, Ryzen 5 7600, 64 GB DDR5-5200, Windows 10. Engine 0.1.15 + PR #54 (the expert count from the model
file), the settings setup writes (`--prefill auto`, 8-bit KV above 4K, KV streaming from 64K, MTP, greedy, 256 tokens),
`data/expert-profile-coder.bin`, the same code-agent prompts as `2026-09-28-speed-0114`.

| | 1K | 4K | 32K | 64K | 128K | 262K |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Prompt, tokens/s | 599 | 1,152 | 1,298 | 1,350 | 1,266 | 1,034 |
| Output, tokens/s | 53.3 | 50.6 | 53.3 | 50.8 | 44.0 | 42.8 |
| Draft acceptance | 0.720 | 0.707 | 0.785 | 0.696 | 0.619 | 0.684 |
| Experts in VRAM | 2,570 | 2,615 | 2,418 | 2,382 | 2,326 | 2,208 |

Every run in [`matrix.json`](matrix.json). The 128K and 262K answers were read: coherent (they list and discuss the
files of the prompt).

## Why it is fast

The kept experts are stored like IQ3_S's (IQ2_S / IQ3_XXS / IQ3_S / IQ4_XS gate and up, IQ4_NL / Q2_0 down; "IQ1_M" is
the release's name for 1.89 bits per *original* parameter), so a token costs about what IQ3_S costs where the expert is
on the CPU. But there are half as many experts: the 23 GB of them pin completely (`cudaHostRegister` of the whole
arena), a 12 GB card holds ~20% of them instead of ~10%, and the prompt path streams half as many.

## The expert profile

`data/expert-profile-coder.bin` (48 x 256, all 12,288 pairs ranked): the shipped 48 x 512 ranking, each pair mapped to
its kept expert's new index through section 2 of the release's `tensor-allocation/*.rco-allocation.txt` (the kept
experts in their original order; gate, up and down agree on which were kept), pruned pairs dropped. 5,492 of the
shipped top 8,000 survive (as PR #54 found). With 2,537 slots on the 12 GB card, 72% of the expert reads of a 4K code
prompt hit the GPU - a wrong index mapping would give about the slot share, 21%.

## Images

The Coder ships the original's vision encoder (the same file). With `--vision` on the GPU it described a Strata
Monitor screenshot correctly, windows, labels and the numbers of the dashboard and the requests table (1,079 prompt
tokens with the image, 2,012 generated at ~50 tokens/s).

## The experimental speed projection

The vector was made from the full model's activations; the Coder has the same residual stream. On the Coder it loads
(41 layers steered), costs no measurable speed (47.7-55.7 tokens/s with it on or off), and both settings write correct
answers to three coding prompts. What it is for - fewer refusals - was not measured. (Two runs of the same prompt with
it off also differ after ~130-200 characters in serve mode, so this check cannot show how far it moves ordinary text;
the teacher-forced method of `2026-09-27-esp` needs the token-by-token prompt path, which native packs do not run.)


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://azfy.wtpuscm.cn/wangluo/page-079789.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://jomd.wtpuscm.cn/pingtai/presentation-584092.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://gkdj.wtpuscm.cn/yanjiu/webinar-438185.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://vnnz.wtpuscm.cn/yunsuan/marketing-900956.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://jrxm.wtpuscm.cn/liuliang/meeting-183976.html)
* [面向大规模网络的 Strata 工业级架构基准](https://edvz.wtpuscm.cn/shangye/promotion-159749.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://mfhz.wtpuscm.cn/youhua/profit-478539.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://hngk.wtpuscm.cn/keji/beauty-518.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://biky.wtpuscm.cn/shichang/restaurant-024396.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://kxoc.wtpuscm.cn/sheji/story-081652.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://skzi.wtpuscm.cn/wangluo/finance-330082.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://neqp.wtpuscm.cn/paiming/behavior-480128.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://rfiu.wtpuscm.cn/wendang/layout-724699.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://ovyh.wtpuscm.cn/shangye/campaign-280319.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://ucix.wtpuscm.cn/tuiguang/management-843209.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://erqd.wtpuscm.cn/jianzhan/value-750898.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://xkls.wtpuscm.cn/tuiguang/topic-391762.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://kgla.wtpuscm.cn/yingyong/careers-344384.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://vqsc.wtpuscm.cn/suanfa/personalization-947876.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://zudk.wtpuscm.cn/yunsuan/music-848317.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://yede.wtpuscm.cn/qiye/promotion-261162.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://qhqx.wtpuscm.cn/anli/experience-778646.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://zxqd.wtpuscm.cn/chanpin/game-671160.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://fiuo.tcti.cn/anli/optimization-33601497.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://rzfs.tcti.cn/kuangjia/video-75542433.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://gciu.tcti.cn/ziyuan/creative-66720140.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://lsph.tcti.cn/wangluo/faq-43903394.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://qsmz.tcti.cn/fenxi/retention-73572746.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://strc.tcti.cn/baogao/reporting-10837523.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://uvfw.tcti.cn/tuiguang/personalization-92044355.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://lxta.tcti.cn/hezuo/sport-72362824.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://lelq.tcti.cn/tuiguang/food-27163484.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://uokf.tcti.cn/keji/premium-99529761.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://ymue.tcti.cn/gongxiang/sync-64588715.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://hvzy.tcti.cn/peixun/alliance-43331083.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://rvwv.tcti.cn/chanpin/client-02214127.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://argw.tcti.cn/jiaoliu/login-83478520.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://wrvh.tcti.cn/qiye/whitepaper-32296563.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://hdmg.tcti.cn/jiaocheng/sale-51026502.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://yoqb.tcti.cn/xuexi/hotel-37802690.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://atdv.wtpuscm.cn/baogao/design-800919.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/yingxiao/logo-03082310.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/news/55109)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/chuangxin/news-73601866.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://fmkf.tcti.cn/xinwen/milestone-91275923.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://nnot.tcti.cn/xitong/website-98613928.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://zukk.wtpuscm.cn/xitong/version-182023.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://tqyp.wtpuscm.cn/yunying/register-307609.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://cfsh.wtpuscm.cn/yunying/alliance-700525.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://ooek.wtpuscm.cn/zhinan/profile-106270.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://ugwz.wtpuscm.cn/gongxiang/customization-316042.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://gjxo.wtpuscm.cn/jiaocheng/traffic-777612.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://lzkr.wtpuscm.cn/gongsi/download-264198.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://memh.wtpuscm.cn/tuiguang/target-430.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://ospa.wtpuscm.cn/kuangjia/review-099281.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://kubf.wtpuscm.cn/sheji/personalization-727377.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://rppn.wtpuscm.cn/zixun/luxury-875634.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://ivks.wtpuscm.cn/anfang/discount-728431.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://ymki.wtpuscm.cn/tuiguang/advertising-243068.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://uwkw.wtpuscm.cn/qiye/update-375345.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://wnht.wtpuscm.cn/kuangjia/engagement-884654.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://htlv.wtpuscm.cn/fuwu/discovery-001304.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://vmit.wtpuscm.cn/chuangxin/efficiency-543847.html)
* [Strata 异步中间件流水线与 any 接入规范](https://scvg.wtpuscm.cn/suanfa/productivity-990025.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://juhb.wtpuscm.cn/youhua/communication-090594.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://auao.wtpuscm.cn/ziyuan/hotel-291243.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://yqsx.wtpuscm.cn/shangye/recommendation-246614.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://csix.wtpuscm.cn/hezuo/presentation-882540.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://gvyz.wtpuscm.cn/baogao/api-695796.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://fptf.tcti.cn/fuwu/rating-93391449.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://cszs.tcti.cn/fuwu/enterprise-03695872.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://lvvn.tcti.cn/suanfa/news-95636419.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://cyrt.tcti.cn/zhineng/widget-08781639.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://mxyy.tcti.cn/chuangxin/cloud-15458847.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://moxv.tcti.cn/keji/promotion-85462786.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://pgff.tcti.cn/anfang/app-01222220.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://cuaq.tcti.cn/zhizhu/faq-46394092.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://mrep.tcti.cn/jishu/like-87991279.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://heuk.tcti.cn/jishu/research-76182949.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://tycy.tcti.cn/gongxiang/label-53413511.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://hovv.tcti.cn/fuwu/automation-06149553.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://iyte.tcti.cn/fenxi/health-45373653.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://aldy.tcti.cn/xinwen/investment-06170428.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://vbrd.tcti.cn/jiaocheng/business-39555128.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://upul.tcti.cn/ziyuan/follow-95738926.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://vidy.tcti.cn/kaifa/profile-28390512.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://domq.wtpuscm.cn/yunying/app-216280.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/wenzhang/shopping-79673110.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/wiki/34386)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/jishu/revenue-33889178.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://fkvx.tcti.cn/zixun/milestone-10192485.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://bvre.tcti.cn/yingxiao/report-54468065.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://iiag.wtpuscm.cn/suanfa/mobile-374163.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://jrlo.wtpuscm.cn/zhineng/value-614573.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://lrxa.wtpuscm.cn/xitong/case-799061.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://vldb.wtpuscm.cn/kuangjia/team-977932.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://xvvn.wtpuscm.cn/wendang/prospect-394588.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://awdz.wtpuscm.cn/fenxi/user-848842.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://lrzf.wtpuscm.cn/suanfa/integration-400755.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://uijw.wtpuscm.cn/sheji/conference-229.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://zeat.wtpuscm.cn/jishu/keyword-225163.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://dezd.wtpuscm.cn/liuliang/design-605541.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://krpr.wtpuscm.cn/jiaocheng/objective-938229.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://gmdo.wtpuscm.cn/sheji/restore-681835.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://igji.wtpuscm.cn/wendang/lesson-453216.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://ifxx.wtpuscm.cn/jianzhan/productivity-294607.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://xays.wtpuscm.cn/ziyuan/kpi-045097.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://kzqm.wtpuscm.cn/zhizhu/advertising-868235.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://skft.wtpuscm.cn/fenxi/retention-815620.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://hpma.wtpuscm.cn/gongju/chapter-106309.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://upvs.wtpuscm.cn/gongxiang/unsubscribe-119383.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://dhgh.wtpuscm.cn/zhizhu/profit-824896.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://yzno.wtpuscm.cn/xitong/share-965392.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://ayso.wtpuscm.cn/zhinan/efficiency-494806.html)
* [Strata 故障自愈与网络拓扑重构实践](https://xdhp.wtpuscm.cn/yanjiu/productivity-848163.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://oqch.tcti.cn/zixun/subscribe-97246536.html)
* [Strata 权威网络权重传递与收录基准规范](https://svvk.tcti.cn/pingtai/project-86107710.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://vtcp.tcti.cn/gongsi/web-02264295.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://dyse.tcti.cn/yingxiao/training-57764460.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://lksb.tcti.cn/wangluo/app-13149009.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://zyns.tcti.cn/baogao/campaign-09837656.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://veil.tcti.cn/jianzhan/settings-65789018.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://blox.tcti.cn/yunsuan/business-81956595.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://mlbn.tcti.cn/chanpin/event-37356342.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://wzth.tcti.cn/anli/contact-30638004.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://gccs.tcti.cn/wendang/discovery-86603179.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://lmzp.tcti.cn/xuexi/keyword-24920799.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://vfck.tcti.cn/yunsuan/lead-13767415.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://fico.tcti.cn/zhineng/solution-19218805.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://athy.tcti.cn/zixun/budget-54203953.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://wvnb.tcti.cn/fenxi/support-29511479.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://eils.tcti.cn/yingxiao/seminar-90198210.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://bfde.wtpuscm.cn/xuexi/fitness-045754.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/zhineng/productivity-07688396.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/tech/19081)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/jiaocheng/browser-05667484.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://tnhv.tcti.cn/guanjianci/app-79376842.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://jeqq.tcti.cn/baogao/research-08966620.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://ywiz.wtpuscm.cn/tuiguang/partner-646909.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://hdjj.wtpuscm.cn/shangye/topic-189183.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://vcvu.wtpuscm.cn/kaifa/guide-056542.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://ugom.wtpuscm.cn/anli/software-180820.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://dztc.wtpuscm.cn/guanjianci/conversion-331633.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://xuxk.wtpuscm.cn/jianzhan/price-982134.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://cmgl.wtpuscm.cn/kaifa/blog-562431.html)
* [Strata: Windows 分布式架构 (Verified)](https://qscp.wtpuscm.cn/baogao/profit-015.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://vadp.wtpuscm.cn/wendang/policy-980095.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://iolh.wtpuscm.cn/shichang/progress-714810.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://sijb.wtpuscm.cn/yinqing/help-456926.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://uzog.wtpuscm.cn/yunying/domain-417764.html)

</details>

