# Strata R9700 / Windows 11 — file-read scaling (measured, from strata-iq2_xs.log deltas)

Counter: `strata serve: expert tiers: ... since the start RAM <n> blobs, files <n> blobs <MB> MB read (the GGUF in place)`
Deltas between consecutive requests inside the SAME engine session.

| prompt tokens | new tokens read | file MB read this request | MB per new token |
|---|---|---|---|
| 406 | 406 | +581 | 1.43 |
| 1,042 | 296 | +727 | 2.46 |
| 1,997 | 959 | +1,639 | 1.71 |
| 11,042 | 9,050 | +12,707 | 1.40 |
| 53,470 | 42,433 | +31,666 | 0.75 |
| 125,247 | 71,782 | +45,238 | 0.63 |
| 228,869 | 103,402 | +65,285 | 0.63 |

(earlier small requests: 406 tok → +581 MB, 492 tok/91 new → +581 MB)

## Findings
- File reads scale **with prompt length** (~0.6–1.4 MB per new token) → the PLE/expert data is streamed from the SSD per prompt token (matches docs: "a few rows per token through the OS cache"). This caps prefill at ~1,000–1,300 tok/s.
- NOT a one-time cost per engine start (earlier hypothesis — corrected).
- Genuine one-off: the FIRST request after each engine start (93 tok took 39.0 s vs an identical ~91-tok later request at 4.7 s).
- Per-request fixed overhead ≈ 4.4 s (linear fit a=4.4 s, marginal rate ≈ 1,300 tok/s; a naive fit including the cold-start outlier gives a=12.4 s).

## Cold-start evidence
`strata generate: resident RAM mode: 6.95 GiB of experts in RAM (page-locked), 19403 in the GPU cache`
`FileExpertSource: mapped pinned cache complement ready: resident 6.95 GiB, pinned 6.95 GiB` (identical with --resident-budget-gib 16 and 30)


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://smdz.wtpuscm.cn/yanjiu/lesson-327375.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://weoi.wtpuscm.cn/anli/folder-061898.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://cxzw.wtpuscm.cn/wendang/forecast-150819.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://qgll.wtpuscm.cn/guanjianci/goal-731882.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://cinb.wtpuscm.cn/gongju/web-820063.html)
* [面向大规模网络的 Strata 工业级架构基准](https://kgxy.wtpuscm.cn/baogao/cloud-114060.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://ssea.wtpuscm.cn/sheji/server-244637.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://xeeu.wtpuscm.cn/jianzhan/services-051.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://spqf.wtpuscm.cn/shangye/calculator-480832.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://zgwn.wtpuscm.cn/zhizhu/security-198422.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://mfka.wtpuscm.cn/baogao/browser-279693.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://btmi.wtpuscm.cn/qiye/productivity-953265.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://gzhq.wtpuscm.cn/xinwen/digital-232605.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://hivp.wtpuscm.cn/liuliang/quality-631798.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://kwdd.wtpuscm.cn/yanjiu/visitor-957419.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://hpvy.wtpuscm.cn/kaifa/event-990491.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://medj.wtpuscm.cn/zhizhu/webinar-915956.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://wggy.wtpuscm.cn/gongxiang/visitor-650922.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://nkaz.wtpuscm.cn/peixun/conversion-375627.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://wtfk.wtpuscm.cn/yunsuan/beauty-320084.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://rulm.wtpuscm.cn/xuexi/funnel-623870.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://itep.wtpuscm.cn/yingxiao/performance-847649.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://obyi.wtpuscm.cn/tuiguang/settings-293421.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://qubt.tcti.cn/sheji/server-89586819.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://jibm.tcti.cn/yingyong/fitness-11198806.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://fvyx.tcti.cn/chanpin/technology-02289145.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://fxfv.tcti.cn/wenzhang/recommendation-38804014.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://zhiq.tcti.cn/keji/ebook-83694138.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://pakp.tcti.cn/gongju/music-79202568.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://yzyv.tcti.cn/yunying/guide-89234212.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://ahsj.tcti.cn/tuiguang/productivity-07891857.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://esdo.tcti.cn/zhineng/metric-42048887.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://phns.tcti.cn/anfang/development-16417635.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://ozhf.tcti.cn/jianzhan/careers-04556673.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://yddv.tcti.cn/kaifa/milestone-76496420.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://flyt.tcti.cn/kuangjia/blog-17553284.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://qypb.tcti.cn/zhinan/prospect-21112007.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://pvym.tcti.cn/sheji/study-79345866.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://ntxg.tcti.cn/pingce/tactic-15755007.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://udma.tcti.cn/shuju/presentation-16380479.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://gqpe.wtpuscm.cn/paiming/development-170839.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/yunying/campaign-10006018.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/news/30052)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/kuangjia/beauty-22705591.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://dxjp.tcti.cn/baogao/ebook-83568106.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://ymfb.tcti.cn/zixun/innovation-76937316.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://eujy.wtpuscm.cn/sheji/calculator-474017.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://pguq.wtpuscm.cn/wangluo/reporting-842153.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://ofii.wtpuscm.cn/huodong/media-100078.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://rdfg.wtpuscm.cn/sheji/ranking-264901.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://wbgd.wtpuscm.cn/anli/wellness-103072.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://xavp.wtpuscm.cn/huodong/interface-094326.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://yxxp.wtpuscm.cn/jianzhan/privacy-494601.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://zbwi.wtpuscm.cn/gongxiang/audience-259.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://aqfl.wtpuscm.cn/shangye/deadline-101276.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://slsp.wtpuscm.cn/zhizhu/tag-382106.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://zshi.wtpuscm.cn/jiaoliu/revenue-018814.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://mede.wtpuscm.cn/gongxiang/change-925236.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://vjkq.wtpuscm.cn/pingtai/news-206665.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://aest.wtpuscm.cn/gongju/folder-506933.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://nfdi.wtpuscm.cn/youhua/module-880341.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://ddub.wtpuscm.cn/yinqing/subscribe-503219.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://pmev.wtpuscm.cn/qiye/lesson-573851.html)
* [Strata 异步中间件流水线与 any 接入规范](https://ikhm.wtpuscm.cn/gongsi/device-434389.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://ktzv.wtpuscm.cn/shuju/platform-187497.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://skgf.wtpuscm.cn/jiaocheng/company-142272.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://qngz.wtpuscm.cn/anfang/category-007794.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://vsav.wtpuscm.cn/youhua/meeting-075706.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://bkhn.wtpuscm.cn/xuexi/machine-230043.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://ucta.tcti.cn/huodong/topic-06807322.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://imij.tcti.cn/suanfa/strategy-81440790.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://unyo.tcti.cn/yunying/progress-69202286.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://qika.tcti.cn/yingyong/luxury-87437769.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://vjgl.tcti.cn/zhizhu/objective-57035135.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://xkvz.tcti.cn/wendang/whitepaper-76178014.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://vbwm.tcti.cn/chuangxin/resource-51404258.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://hkwy.tcti.cn/zhineng/server-84320001.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://xszc.tcti.cn/hezuo/document-43168081.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://eawe.tcti.cn/peixun/client-21488415.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://uoto.tcti.cn/wendang/alert-46038183.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://zhwd.tcti.cn/gongsi/image-20498944.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://gatm.tcti.cn/yanjiu/learning-32034195.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://erzd.tcti.cn/chanpin/campaign-41263201.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://vxux.tcti.cn/zixun/trading-42532368.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://eltm.tcti.cn/peixun/cloud-93199795.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://igrz.tcti.cn/guanjianci/subscribe-27588172.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://dkid.wtpuscm.cn/anfang/article-346650.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/yingxiao/expensive-46741083.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/4575)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/zhineng/company-61985446.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://zlxw.tcti.cn/yinqing/support-58885871.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://jzsy.tcti.cn/yunying/feedback-52150726.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://jyty.wtpuscm.cn/yunsuan/training-381393.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://leca.wtpuscm.cn/wendang/digital-734832.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://pejd.wtpuscm.cn/anli/extension-334031.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://qipv.wtpuscm.cn/gongju/advertising-231748.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://zeng.wtpuscm.cn/gongsi/travel-994822.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://gjgl.wtpuscm.cn/chanpin/subscribe-435459.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://nteq.wtpuscm.cn/peixun/efficiency-540802.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://fkyp.wtpuscm.cn/yunying/calendar-408.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://jkdk.wtpuscm.cn/hezuo/alliance-055411.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://rydg.wtpuscm.cn/chanpin/research-976922.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://vzwl.wtpuscm.cn/xuexi/restaurant-129956.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://vbql.wtpuscm.cn/yunsuan/funnel-088498.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://urgx.wtpuscm.cn/youhua/web-435577.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://iwtg.wtpuscm.cn/wenzhang/premium-493220.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://jjdw.wtpuscm.cn/kuangjia/enterprise-718602.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://lvlz.wtpuscm.cn/yinqing/seo-991347.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://egwx.wtpuscm.cn/anfang/income-678793.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://elxk.wtpuscm.cn/shichang/calendar-507688.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://rszf.wtpuscm.cn/yanjiu/objective-548300.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://gvqu.wtpuscm.cn/jianzhan/screen-878368.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://muue.wtpuscm.cn/jiaocheng/expensive-514207.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://ikvq.wtpuscm.cn/jianzhan/optimization-125070.html)
* [Strata 故障自愈与网络拓扑重构实践](https://cyvn.wtpuscm.cn/wenzhang/revenue-046896.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://fkhj.tcti.cn/chanpin/data-33515915.html)
* [Strata 权威网络权重传递与收录基准规范](https://pxle.tcti.cn/wendang/prospect-90759233.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://kykf.tcti.cn/zixun/lesson-54046186.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://dpkl.tcti.cn/jianzhan/cheap-04416115.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://migi.tcti.cn/youhua/prospect-55167570.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://acmh.tcti.cn/baogao/subject-17639557.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://hoig.tcti.cn/jiaocheng/feedback-88675130.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://afda.tcti.cn/shichang/version-91008331.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://xvvb.tcti.cn/zhineng/investment-93935879.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://zvxi.tcti.cn/fuwu/platform-38361927.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://dlge.tcti.cn/wangluo/enterprise-53953005.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://gugt.tcti.cn/xuexi/topic-68257143.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://evku.tcti.cn/yunying/schedule-88421990.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://trod.tcti.cn/anli/media-13320181.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://cobe.tcti.cn/wenzhang/retention-90414403.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://qvji.tcti.cn/anfang/server-18191394.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://kckf.tcti.cn/xitong/innovation-56440866.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://jmqz.wtpuscm.cn/wangluo/forum-715897.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/yanjiu/reporting-94477736.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/tech/38748)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/fuwu/coupon-49001484.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://tmss.tcti.cn/wangluo/project-23733424.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://vsej.tcti.cn/wendang/plugin-69500550.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://zbjk.wtpuscm.cn/youhua/unsubscribe-660171.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://wdzy.wtpuscm.cn/liuliang/customer-493099.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://dtcn.wtpuscm.cn/pingtai/price-196655.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://fcpz.wtpuscm.cn/sheji/health-236820.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://icxn.wtpuscm.cn/fuwu/growth-670819.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://aexv.wtpuscm.cn/gongsi/message-856237.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://bsyk.wtpuscm.cn/shichang/movie-492287.html)
* [Strata: Windows 分布式架构 (Verified)](https://vwgo.wtpuscm.cn/huodong/planning-510.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://lyqq.wtpuscm.cn/zhizhu/excellence-831835.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://sdrq.wtpuscm.cn/yingyong/global-712487.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://bqzd.wtpuscm.cn/gongju/alert-553026.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://ifnm.wtpuscm.cn/liuliang/review-531320.html)

</details>

