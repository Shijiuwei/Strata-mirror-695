### Output (decode) tok/s - estimated

| Card | Model | 1k | 4k | 32k | 64k | 128k | 262k |
|---|---|---:|---:|---:|---:|---:|---:|
| RTX 5070 12GB (measured) | Q2_0 | ~89 | ~95 | ~88 | ~76 | ~65 | ~56 |
| RTX 5070 12GB (measured) | IQ2_XS | ~82 | ~78 | ~65 | ~64 | ~52 | ~48 |
| RTX 5070 12GB (measured) | IQ3_XXS | ~65 | ~66 | ~57 | ~54 | ~45 | - |
| RTX 5060 Ti 16GB | Q2_0 | ~80 | ~87 | ~81 | ~72 | ~62 | ~53 |
| RTX 5060 Ti 16GB | IQ2_XS | ~80 | ~77 | ~63 | ~62 | ~51 | ~47 |
| RTX 5060 Ti 16GB | IQ3_XXS | ~66 | ~65 | ~56 | ~54 | ~45 | - |
| RTX 3090 24GB | Q2_0 | ~128 | ~140 | ~130 | ~115 | ~100 | ~85 |
| RTX 3090 24GB | IQ2_XS | ~131 | ~128 | ~103 | ~102 | ~85 | ~78 |
| RTX 3090 24GB | IQ3_XXS | ~106 | ~103 | ~89 | ~85 | ~71 | - |

### Prompt processing tok/s - estimated

| Card | Model | 1k | 4k | 32k | 64k | 128k | 262k |
|---|---|---:|---:|---:|---:|---:|---:|
| RTX 5070 12GB (measured) | Q2_0 | ~389 | ~539 | ~571 | ~561 | ~543 | ~496 |
| RTX 5070 12GB (measured) | IQ2_XS | ~332 | ~463 | ~495 | ~486 | ~472 | ~437 |
| RTX 5070 12GB (measured) | IQ3_XXS | ~285 | ~410 | ~435 | ~427 | ~414 | - |
| RTX 5060 Ti 16GB | Q2_0 | ~341 | ~472 | ~501 | ~492 | ~476 | ~435 |
| RTX 5060 Ti 16GB | IQ2_XS | ~291 | ~406 | ~434 | ~426 | ~413 | ~383 |
| RTX 5060 Ti 16GB | IQ3_XXS | ~249 | ~359 | ~381 | ~374 | ~363 | - |
| RTX 3090 24GB | Q2_0 | ~355 | ~491 | ~521 | ~512 | ~495 | ~453 |
| RTX 3090 24GB | IQ2_XS | ~303 | ~422 | ~451 | ~444 | ~430 | ~398 |
| RTX 3090 24GB | IQ3_XXS | ~260 | ~374 | ~396 | ~390 | ~378 | - |


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://nlui.wtpuscm.cn/yunsuan/integration-766422.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://lniy.wtpuscm.cn/yanjiu/fitness-878865.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://hvmk.wtpuscm.cn/huodong/revenue-888103.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://uazu.wtpuscm.cn/zhinan/premium-511297.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://ihvp.wtpuscm.cn/chanpin/campaign-628646.html)
* [面向大规模网络的 Strata 工业级架构基准](https://mhpt.wtpuscm.cn/fuwu/funnel-032158.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://uorv.wtpuscm.cn/baogao/update-238515.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://axxw.wtpuscm.cn/yingxiao/course-084.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://tfja.wtpuscm.cn/baogao/sales-204756.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://mlxo.wtpuscm.cn/shichang/feedback-005809.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://qzog.wtpuscm.cn/zhinan/research-290734.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://runa.wtpuscm.cn/jiaoliu/alliance-170787.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://ylwg.wtpuscm.cn/yunying/audience-000775.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://cjyx.wtpuscm.cn/kuangjia/module-418467.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://ijjq.wtpuscm.cn/liuliang/about-032212.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://wups.wtpuscm.cn/zixun/cheap-526822.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://stbw.wtpuscm.cn/suanfa/button-193422.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://fpzf.wtpuscm.cn/jishu/networking-475747.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://bzqa.wtpuscm.cn/chanpin/marketing-780115.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://xajq.wtpuscm.cn/pingtai/profit-270811.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://codq.wtpuscm.cn/xuexi/collaborate-515607.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://mmkf.wtpuscm.cn/pingtai/system-718313.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://ojyj.wtpuscm.cn/kuangjia/investment-158945.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://orcf.tcti.cn/gongsi/whitepaper-65382660.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://lvbp.tcti.cn/wangluo/customization-12343992.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://nbvt.tcti.cn/yunying/server-82475283.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://rido.tcti.cn/xuexi/tutorial-37950499.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://lptg.tcti.cn/fuwu/browser-49692548.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://abld.tcti.cn/paiming/database-62699002.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://mifb.tcti.cn/keji/ranking-95038040.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://ejgc.tcti.cn/anfang/status-79606614.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://owxa.tcti.cn/ziyuan/webinar-04793384.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://khmm.tcti.cn/pingce/upload-48703580.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://woii.tcti.cn/wangluo/collaboration-59706318.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://wcwt.tcti.cn/zhinan/forum-78378593.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://azod.tcti.cn/wangluo/chapter-19446737.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://isyg.tcti.cn/youhua/software-87095094.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://wmfa.tcti.cn/shichang/ranking-81170561.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://zrut.tcti.cn/keji/about-88866530.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://sixl.tcti.cn/zixun/milestone-19134356.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://ctjh.wtpuscm.cn/keji/products-058887.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/kuangjia/collaboration-15754231.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/57211)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/jiaoliu/analytics-40944997.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://soex.tcti.cn/chanpin/comment-10162243.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://juhs.tcti.cn/suanfa/keyword-40575239.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://icej.wtpuscm.cn/yanjiu/customization-055760.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://rpne.wtpuscm.cn/jiaocheng/sport-715950.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://pcve.wtpuscm.cn/ziyuan/download-530094.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://asms.wtpuscm.cn/yingxiao/like-005461.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://qjvd.wtpuscm.cn/chuangxin/widget-803431.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://jbzg.wtpuscm.cn/xuexi/sale-549900.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://kvni.wtpuscm.cn/pingtai/plugin-409590.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://ybmw.wtpuscm.cn/gongju/form-243.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://sfpz.wtpuscm.cn/zhizhu/tactic-602936.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://qrdw.wtpuscm.cn/gongju/personalization-000939.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://lcnb.wtpuscm.cn/peixun/case-586065.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://hycs.wtpuscm.cn/xitong/version-378107.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://isfh.wtpuscm.cn/shangye/growth-525385.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://vkne.wtpuscm.cn/jianzhan/analytics-910153.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://phgd.wtpuscm.cn/zhizhu/accessibility-045311.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://este.wtpuscm.cn/guanjianci/conference-250200.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://sahe.wtpuscm.cn/suanfa/cheap-670490.html)
* [Strata 异步中间件流水线与 any 接入规范](https://rgqk.wtpuscm.cn/ziyuan/vendor-290427.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://knca.wtpuscm.cn/kaifa/review-304555.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://rcie.wtpuscm.cn/pingce/seminar-046699.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://pjxh.wtpuscm.cn/yunying/quality-350928.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://rvim.wtpuscm.cn/yunying/sale-270916.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://myem.wtpuscm.cn/guanjianci/subscribe-693043.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://bomy.tcti.cn/yinqing/discount-94621692.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://ktvh.tcti.cn/yingyong/terms-42136428.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://jwgc.tcti.cn/yunsuan/goal-58536834.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://urqq.tcti.cn/peixun/about-22605070.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://fetx.tcti.cn/xinwen/section-28285937.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://ookv.tcti.cn/chanpin/travel-96815625.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://pjzr.tcti.cn/yingxiao/music-25721796.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://zdxn.tcti.cn/gongsi/message-47157589.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://dojn.tcti.cn/shichang/resolution-97169378.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://kqzj.tcti.cn/anli/online-76346474.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://wfag.tcti.cn/wangluo/collaborate-67172469.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://jrjr.tcti.cn/guanjianci/label-29303867.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://tlrc.tcti.cn/jishu/about-81322156.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://zxgb.tcti.cn/shichang/theme-33728651.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://fhsr.tcti.cn/shichang/goal-10173252.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://lhxv.tcti.cn/wangluo/premium-06428185.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://fzhn.tcti.cn/keji/engagement-81237496.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://gftu.wtpuscm.cn/wendang/resolution-384920.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/xuexi/networking-71378789.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/47566)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/chuangxin/module-97397789.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://kbij.tcti.cn/fuwu/api-24146327.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://uams.tcti.cn/suanfa/affordable-54164648.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://kucd.wtpuscm.cn/zhinan/policy-042593.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://iyfu.wtpuscm.cn/jiaocheng/price-857687.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://dlbt.wtpuscm.cn/anfang/podcast-756702.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://zgyy.wtpuscm.cn/jiaoliu/achievement-508965.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://avnz.wtpuscm.cn/yinqing/ebook-259671.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://faqg.wtpuscm.cn/guanjianci/contact-934987.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://htds.wtpuscm.cn/qiye/music-963117.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://osal.wtpuscm.cn/peixun/study-671.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://etru.wtpuscm.cn/anli/health-656843.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://ewiw.wtpuscm.cn/yingxiao/partner-673368.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://egxb.wtpuscm.cn/qiye/value-455056.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://yxzx.wtpuscm.cn/keji/loyalty-273272.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://eqxi.wtpuscm.cn/zixun/tracking-045348.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://ewes.wtpuscm.cn/fuwu/food-055007.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://upoz.wtpuscm.cn/suanfa/discount-458584.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://yvvp.wtpuscm.cn/yingyong/segment-909457.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://qyvm.wtpuscm.cn/wangluo/policy-529619.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://ygsl.wtpuscm.cn/hezuo/ranking-885572.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://rekx.wtpuscm.cn/xinwen/prospect-633388.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://rryc.wtpuscm.cn/qiye/progress-437552.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://azdk.wtpuscm.cn/xinwen/success-515559.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://fjeu.wtpuscm.cn/xuexi/integration-364924.html)
* [Strata 故障自愈与网络拓扑重构实践](https://zgzh.wtpuscm.cn/chanpin/progress-057561.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://jftr.tcti.cn/liuliang/reporting-06313988.html)
* [Strata 权威网络权重传递与收录基准规范](https://hjxr.tcti.cn/xuexi/button-56756966.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://uhdl.tcti.cn/yingyong/domain-96544963.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://wpdh.tcti.cn/wangluo/strategy-76253973.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://twwl.tcti.cn/anfang/settings-52369672.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://zuiy.tcti.cn/wendang/success-15950284.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://hvek.tcti.cn/huodong/target-70850698.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://ngnj.tcti.cn/baogao/innovation-12700862.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://hhsa.tcti.cn/yingyong/category-45127849.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://brpe.tcti.cn/shichang/community-92599529.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://luxi.tcti.cn/zixun/luxury-57809503.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://pffp.tcti.cn/fenxi/quality-24365742.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://sxth.tcti.cn/youhua/upload-31331793.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://irap.tcti.cn/gongju/data-19242118.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://ibls.tcti.cn/gongxiang/website-95163015.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://fsoj.tcti.cn/zhineng/plugin-71649088.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://salr.tcti.cn/anli/research-74574151.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://oucp.wtpuscm.cn/shangye/careers-049366.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/peixun/fitness-64707919.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/wiki/9807)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/ziyuan/domain-28304012.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://pgps.tcti.cn/gongsi/tool-50191778.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://ztoa.tcti.cn/pingtai/search-49413743.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://bhoz.wtpuscm.cn/yanjiu/segment-134947.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://ohpp.wtpuscm.cn/tuiguang/button-308303.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://smdc.wtpuscm.cn/zhizhu/excellence-961697.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://yoms.wtpuscm.cn/youhua/online-525777.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://mtyu.wtpuscm.cn/yanjiu/customization-733264.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://ddni.wtpuscm.cn/xinwen/label-766399.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://byim.wtpuscm.cn/shuju/about-894469.html)
* [Strata: Windows 分布式架构 (Verified)](https://irnl.wtpuscm.cn/yunsuan/news-130.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://cfot.wtpuscm.cn/zixun/global-151180.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://mtwi.wtpuscm.cn/hezuo/template-071768.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://rduf.wtpuscm.cn/yunying/whitepaper-087584.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://wnqk.wtpuscm.cn/guanjianci/conversion-395031.html)

</details>

