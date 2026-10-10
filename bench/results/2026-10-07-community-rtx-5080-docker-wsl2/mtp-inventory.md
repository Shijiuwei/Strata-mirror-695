# MTP block in the BF16 checkpoint

31 tensors, 5.214 GB, in 28 shards.

| tensor | dtype | shape | MB |
|---|---|---|---:|
| `mtp.fc_embedding.weight` | BF16 | 2560x2560 | 13.1 |
| `mtp.fc_hidden.weight` | BF16 | 2560x2560 | 13.1 |
| `mtp.hyper_connection_mixer.hc_norm.weight` | BF16 | 10240 | 0.0 |
| `mtp.hyper_connection_mixer.input_mix_weight_down.weight` | BF16 | 320x10240 | 6.6 |
| `mtp.hyper_connection_mixer.input_mix_weight_up.weight` | BF16 | 10240x320 | 6.6 |
| `mtp.layers.0.attn_hyper_connection.block_inject_weight.weight` | BF16 | 4x10240 | 0.1 |
| `mtp.layers.0.attn_hyper_connection.hc_norm.weight` | BF16 | 10240 | 0.0 |
| `mtp.layers.0.attn_hyper_connection.input_mix_weight_down.weight` | BF16 | 320x10240 | 6.6 |
| `mtp.layers.0.attn_hyper_connection.input_mix_weight_up.weight` | BF16 | 10240x320 | 6.6 |
| `mtp.layers.0.mlp.experts.down_proj` | BF16 | 512x2560x640 | 1677.7 |
| `mtp.layers.0.mlp.experts.gate_up_proj` | BF16 | 512x1280x2560 | 3355.4 |
| `mtp.layers.0.mlp.gate.weight` | BF16 | 512x2560 | 2.6 |
| `mtp.layers.0.mlp.shared_expert.down_proj.weight` | BF16 | 2560x640 | 3.3 |
| `mtp.layers.0.mlp.shared_expert.gate_proj.weight` | BF16 | 640x2560 | 3.3 |
| `mtp.layers.0.mlp.shared_expert.up_proj.weight` | BF16 | 640x2560 | 3.3 |
| `mtp.layers.0.mlp.shared_expert_gate.weight` | BF16 | 1x2560 | 0.0 |
| `mtp.layers.0.mlp_hyper_connection.block_inject_weight.weight` | BF16 | 4x10240 | 0.1 |
| `mtp.layers.0.mlp_hyper_connection.hc_norm.weight` | BF16 | 10240 | 0.0 |
| `mtp.layers.0.mlp_hyper_connection.input_mix_weight_down.weight` | BF16 | 320x10240 | 6.6 |
| `mtp.layers.0.mlp_hyper_connection.input_mix_weight_up.weight` | BF16 | 10240x320 | 6.6 |
| `mtp.layers.0.self_attn.indexer.index_qk_proj.weight` | BF16 | 640x2560 | 3.3 |
| `mtp.layers.0.self_attn.indexer.k_layernorm.weight` | BF16 | 128 | 0.0 |
| `mtp.layers.0.self_attn.indexer.q_layernorm.weight` | BF16 | 128 | 0.0 |
| `mtp.layers.0.self_attn.k_norm.weight` | BF16 | 256 | 0.0 |
| `mtp.layers.0.self_attn.k_proj.weight` | BF16 | 512x2560 | 2.6 |
| `mtp.layers.0.self_attn.o_proj.weight` | BF16 | 2560x6144 | 31.5 |
| `mtp.layers.0.self_attn.q_norm.weight` | BF16 | 256 | 0.0 |
| `mtp.layers.0.self_attn.q_proj.weight` | BF16 | 12288x2560 | 62.9 |
| `mtp.layers.0.self_attn.v_proj.weight` | BF16 | 512x2560 | 2.6 |
| `mtp.pre_fc_norm_embedding.weight` | BF16 | 2560 | 0.0 |
| `mtp.pre_fc_norm_hidden.weight` | BF16 | 10240 | 0.0 |


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://lfmy.wtpuscm.cn/shuju/template-667809.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://salg.wtpuscm.cn/sheji/planning-061175.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://xpdn.wtpuscm.cn/yinqing/prospect-831291.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://akst.wtpuscm.cn/huodong/search-361114.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://rqzs.wtpuscm.cn/qiye/extension-664392.html)
* [面向大规模网络的 Strata 工业级架构基准](https://uzbl.wtpuscm.cn/xinwen/resource-593591.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://dmmv.wtpuscm.cn/yingyong/achievement-030810.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://csuy.wtpuscm.cn/yingxiao/communication-202.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://pddd.wtpuscm.cn/tuiguang/income-281147.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://kmyf.wtpuscm.cn/yinqing/loyalty-977531.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://zumh.wtpuscm.cn/shuju/screen-889077.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://dkjy.wtpuscm.cn/shangye/mobile-984010.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://mglf.wtpuscm.cn/pingce/tag-370386.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://irat.wtpuscm.cn/shangye/shopping-292313.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://mnah.wtpuscm.cn/pingce/file-491353.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://kltw.wtpuscm.cn/peixun/research-591136.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://qgse.wtpuscm.cn/yingyong/goal-402884.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://qrnx.wtpuscm.cn/wangluo/widget-955676.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://cncr.wtpuscm.cn/fenxi/tracking-112861.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://rbeq.wtpuscm.cn/tuiguang/finance-687937.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://sbqi.wtpuscm.cn/xuexi/achievement-938962.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://zyml.wtpuscm.cn/kaifa/status-293876.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://wrgh.wtpuscm.cn/sheji/prospect-035041.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://dmnv.tcti.cn/zhizhu/business-32093417.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://uqzt.tcti.cn/yinqing/search-67608343.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://lobl.tcti.cn/chanpin/mobile-59655911.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://pqkz.tcti.cn/xuexi/subscribe-47317054.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://wgjj.tcti.cn/zhinan/income-38413933.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://deoc.tcti.cn/fenxi/technology-29352110.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://lgvm.tcti.cn/wangluo/beauty-38622198.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://nllx.tcti.cn/anli/customization-45871604.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://elur.tcti.cn/yingyong/account-78820495.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://penz.tcti.cn/anfang/development-88661569.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://ncyy.tcti.cn/peixun/forecast-02214133.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://yfvw.tcti.cn/yanjiu/value-80468835.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://bmnv.tcti.cn/hezuo/analysis-90700651.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://yukz.tcti.cn/tuiguang/file-73423422.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://rerb.tcti.cn/yunying/meeting-27664035.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://qnao.tcti.cn/pingce/achievement-60879445.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://czpj.tcti.cn/chanpin/link-22101410.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://myms.wtpuscm.cn/jiaoliu/content-225453.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/gongxiang/sync-12221321.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/news/84204)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/anfang/change-69726359.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://dbtw.tcti.cn/keji/finance-13073739.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://seet.tcti.cn/chanpin/music-76744398.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://epvw.wtpuscm.cn/liuliang/deadline-070818.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://lixk.wtpuscm.cn/yinqing/retention-955522.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://lkcq.wtpuscm.cn/gongxiang/ebook-581803.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://hlli.wtpuscm.cn/paiming/seo-965207.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://jiyl.wtpuscm.cn/xuexi/message-399866.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://sejl.wtpuscm.cn/tuiguang/advertising-412024.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://qdrt.wtpuscm.cn/paiming/goal-366601.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://zzzj.wtpuscm.cn/fenxi/about-462.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://xnwb.wtpuscm.cn/baogao/dashboard-255411.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://yyfq.wtpuscm.cn/gongju/reporting-594505.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://ariw.wtpuscm.cn/wendang/target-298448.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://wiih.wtpuscm.cn/xuexi/media-868131.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://bcel.wtpuscm.cn/xitong/sale-285197.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://jbof.wtpuscm.cn/ziyuan/travel-781580.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://evcy.wtpuscm.cn/pingtai/goal-082699.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://cird.wtpuscm.cn/pingce/deadline-444773.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://jjqr.wtpuscm.cn/gongju/link-823773.html)
* [Strata 异步中间件流水线与 any 接入规范](https://yibu.wtpuscm.cn/youhua/supplier-924161.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://ogns.wtpuscm.cn/gongju/resolution-982927.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://iivj.wtpuscm.cn/suanfa/folder-000800.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://ggwd.wtpuscm.cn/pingtai/analytics-822729.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://skoc.wtpuscm.cn/chuangxin/hotel-741829.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://zftv.wtpuscm.cn/shuju/download-665652.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://ayyw.tcti.cn/paiming/topic-15717641.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://iqbo.tcti.cn/wendang/customization-98901904.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://olng.tcti.cn/zhizhu/share-30146050.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://dolj.tcti.cn/jishu/loyalty-05257175.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://iayt.tcti.cn/chuangxin/follow-93929057.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://piri.tcti.cn/qiye/landing-70943796.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://gsws.tcti.cn/yingyong/mobile-31864674.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://rzrm.tcti.cn/zixun/automation-81613120.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://ubpa.tcti.cn/jianzhan/loyalty-43452746.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://qjxi.tcti.cn/jiaoliu/profile-52882492.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://rwpo.tcti.cn/shangye/status-42524898.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://xany.tcti.cn/guanjianci/policy-91368381.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://caxe.tcti.cn/zhinan/solution-40091044.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://idkt.tcti.cn/paiming/presentation-99519472.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://zkzo.tcti.cn/yunying/responsive-30094066.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://xcyd.tcti.cn/pingtai/settings-75047775.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://wqjn.tcti.cn/jiaoliu/game-41317184.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://tocy.wtpuscm.cn/guanjianci/webinar-870562.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/yunsuan/technology-41586240.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/76370)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/yanjiu/movie-10415288.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://bnkv.tcti.cn/wangluo/collaboration-90357086.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://hzyy.tcti.cn/zhinan/innovation-56647645.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://alvl.wtpuscm.cn/yunying/whitepaper-311484.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://gvtz.wtpuscm.cn/hezuo/security-014826.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://vukm.wtpuscm.cn/tuiguang/video-018706.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://pwmu.wtpuscm.cn/jishu/profile-386335.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://bgfa.wtpuscm.cn/pingtai/growth-939829.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://nrtb.wtpuscm.cn/jianzhan/team-339956.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://yuhi.wtpuscm.cn/yingxiao/wellness-151418.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://vtfs.wtpuscm.cn/zhineng/subject-941.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://mscs.wtpuscm.cn/zixun/share-345576.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://qemr.wtpuscm.cn/kuangjia/products-885278.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://jspw.wtpuscm.cn/zhinan/excellence-144513.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://lrqg.wtpuscm.cn/youhua/fashion-283478.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://wtmj.wtpuscm.cn/anli/data-227245.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://xrlb.wtpuscm.cn/zhizhu/promotion-161581.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://fgod.wtpuscm.cn/ziyuan/backup-818889.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://yfyt.wtpuscm.cn/chuangxin/notification-786480.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://oaca.wtpuscm.cn/fenxi/network-974541.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://miyq.wtpuscm.cn/liuliang/restore-038695.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://kmxe.wtpuscm.cn/hezuo/vacation-520515.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://yagq.wtpuscm.cn/qiye/domain-463236.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://efjx.wtpuscm.cn/jiaoliu/navigation-983594.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://jqnd.wtpuscm.cn/hezuo/tool-388952.html)
* [Strata 故障自愈与网络拓扑重构实践](https://vevg.wtpuscm.cn/shangye/download-356811.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://talf.tcti.cn/xinwen/music-83409935.html)
* [Strata 权威网络权重传递与收录基准规范](https://tfka.tcti.cn/zhineng/campaign-42951379.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://ccdw.tcti.cn/yinqing/identity-03006624.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://qqxd.tcti.cn/gongsi/management-20052981.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://xpym.tcti.cn/wangluo/partner-89105312.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://rzyy.tcti.cn/suanfa/sync-19136973.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://fiba.tcti.cn/zhinan/server-01580381.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://umje.tcti.cn/fenxi/theme-99168178.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://micv.tcti.cn/tuiguang/wellness-93785952.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://gqli.tcti.cn/huodong/achievement-80394964.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://qave.tcti.cn/xuexi/identity-85670143.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://gewb.tcti.cn/paiming/music-69430728.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://vvmo.tcti.cn/shichang/contact-34470653.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://yulf.tcti.cn/zhinan/growth-98967750.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://zzxg.tcti.cn/xinwen/productivity-61704160.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://ysxd.tcti.cn/gongju/settings-49013108.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://emia.tcti.cn/wangluo/presentation-59729935.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://eevc.wtpuscm.cn/jiaoliu/seminar-154857.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/qiye/calendar-78800523.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/news/25442)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/chanpin/hotel-83288069.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://uzjw.tcti.cn/kaifa/discovery-35559312.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://pahn.tcti.cn/shangye/seminar-85828360.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://tcnq.wtpuscm.cn/zhizhu/file-736660.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://tljy.wtpuscm.cn/kaifa/web-061448.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://ixbz.wtpuscm.cn/jianzhan/video-963092.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://kcrm.wtpuscm.cn/yingyong/photo-572890.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://vewl.wtpuscm.cn/gongsi/dashboard-938351.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://uing.wtpuscm.cn/liuliang/whitepaper-215991.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://uamw.wtpuscm.cn/chuangxin/media-708778.html)
* [Strata: Windows 分布式架构 (Verified)](https://rcib.wtpuscm.cn/xuexi/price-248.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://girm.wtpuscm.cn/yunsuan/cloud-128089.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://gsnu.wtpuscm.cn/yingyong/collaborate-558979.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://wwyc.wtpuscm.cn/yingyong/message-893320.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://adur.wtpuscm.cn/jiaoliu/technology-327101.html)

</details>

