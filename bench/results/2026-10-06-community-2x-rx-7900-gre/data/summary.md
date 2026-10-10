
## interleave500 (500 greedy tokens per request, one restart per request)

| arm | runs | prompt tok/s median (range) | decode tok/s median (range) | TTFT s median (range) | wall s median (range) | finish |
| --- | ---: | --- | --- | --- | --- | --- |
| A | 10 | 961.0 (841.5..1270.1) | 69.3 (66.4..74.3) | 2.9 (1.9..3.6) | 10.1 (6.9..10.7) | ['length', 'stop'] |
| B | 10 | 959.9 (839.6..1271.9) | 63.8 (59.7..69.7) | 2.9 (1.9..3.7) | 10.3 (6.0..11.5) | ['length', 'stop'] |
- pair 1: A 66.4 -> B 60.5 tok/s (-8.9%), prompt 926 -> 923 tok/s
- pair 2: A 72.6 -> B 59.7 tok/s (-17.8%), prompt 842 -> 854 tok/s
- pair 3: A 67.8 -> B 68.3 tok/s (+0.7%), prompt 972 -> 972 tok/s
- pair 4: A 68.8 -> B 68.1 tok/s (-1.0%), prompt 1270 -> 1267 tok/s
- pair 5: A 73.2 -> B 63.3 tok/s (-13.5%), prompt 1231 -> 1251 tok/s
- pair 6: A 67.4 -> B 63.5 tok/s (-5.8%), prompt 919 -> 913 tok/s
- pair 7: A 66.9 -> B 63.8 tok/s (-4.6%), prompt 852 -> 840 tok/s
- pair 8: A 71.0 -> B 69.7 tok/s (-1.8%), prompt 950 -> 948 tok/s
- pair 9: A 74.3 -> B 64.6 tok/s (-13.1%), prompt 1270 -> 1272 tok/s
- pair 10: A 69.9 -> B 63.9 tok/s (-8.6%), prompt 1205 -> 1154 tok/s
- B vs A decode change: median -7.2% (-17.8%..+0.7%) over 10 pairs
- prompt tokens: 1757..4238 (median 2913)
- reused: 0..0 (median 0)
- fresh: 1757..4238 (median 2913)
- generated: 256..500 (median 500)

## short arms (bench_prefill.py: 128 tokens, warmup + 4 fresh + 4 follow-up per arm)

| arm | side | kind | runs | prompt tok/s | decode tok/s | prompt read ms | decode ms |
| --- | --- | --- | ---: | --- | --- | --- | --- |
| batch2 | A | fresh | 4 | 1542.7 (1301.4..1712.5) | 79.1 (74.1..82.0) | 4195.5 (3055.0..5172.0) | 1295.5 (926.0..1727.0) |
| batch2 | A | followup | 4 | 227.3 (188.4..311.4) | 70.1 (62.3..71.3) | 668.0 (595.0..796.0) | 1827.0 (1796.0..2055.0) |
| batch2 | B | fresh | 4 | 1404.0 (1155.0..1552.9) | 68.3 (66.2..69.4) | 4665.5 (3348.0..5695.0) | 1874.0 (1844.0..1934.0) |
| batch2 | B | followup | 4 | 288.7 (174.1..300.7) | 62.8 (59.6..69.1) | 833.5 (655.0..877.0) | 2044.0 (1852.0..2147.0) |
| batch4 | A | fresh | 4 | 1452.3 (956.0..1798.1) | 71.2 (56.7..72.3) | 4657.5 (3505.0..5183.0) | 1798.0 (1771.0..2257.0) |
| batch4 | A | followup | 4 | 213.9 (141.3..298.2) | 61.0 (57.8..68.0) | 699.0 (569.0..832.0) | 2096.5 (1883.0..2211.0) |
| batch4 | B | fresh | 4 | 1117.7 (854.2..1238.9) | 68.5 (66.0..74.9) | 6027.5 (4196.0..7166.0) | 1869.5 (1709.0..1939.0) |
| batch4 | B | followup | 4 | 147.1 (144.7..256.3) | 60.9 (59.0..67.6) | 780.0 (771.0..967.0) | 2103.5 (1894.0..2168.0) |
| pipeline-off | A | fresh | 4 | 1582.8 (1302.1..1756.8) | 77.1 (70.9..83.9) | 4129.5 (2962.0..5062.0) | 1663.5 (1526.0..1804.0) |
| pipeline-off | A | followup | 4 | 185.8 (182.3..315.1) | 66.2 (60.0..73.2) | 622.0 (603.0..787.0) | 1933.5 (1750.0..2134.0) |
| pipeline-off | B | fresh | 4 | 1568.0 (1174.4..1738.6) | 70.0 (63.1..73.6) | 4332.0 (2997.0..5100.0) | 1829.5 (1738.0..2029.0) |
| pipeline-off | B | followup | 4 | 246.8 (182.7..313.1) | 62.8 (56.9..64.6) | 708.0 (592.0..819.0) | 2029.5 (1841.0..2249.0) |
| resident | A | fresh | 4 | 1558.5 (1235.2..1736.0) | 72.1 (63.9..78.5) | 4247.0 (3007.0..5143.0) | 1778.5 (968.0..2002.0) |
| resident | A | followup | 4 | 186.6 (181.1..301.3) | 63.4 (61.3..70.1) | 619.0 (598.0..823.0) | 2019.0 (1826.0..2086.0) |
| resident | B | fresh | 4 | 1550.0 (1313.5..1718.9) | 69.0 (58.0..72.1) | 4171.0 (3036.0..5155.0) | 1854.0 (1775.0..2208.0) |
| resident | B | followup | 4 | 243.9 (179.6..312.5) | 64.6 (60.5..66.6) | 714.5 (606.0..823.0) | 1981.0 (1921.0..2115.0) |
| stage-trim-off | A | fresh | 4 | 1582.9 (1229.7..1751.3) | 75.7 (67.6..84.7) | 4232.5 (2965.0..5058.0) | 1690.5 (1512.0..1892.0) |
| stage-trim-off | A | followup | 4 | 186.1 (180.4..193.3) | 73.5 (68.7..79.1) | 612.5 (585.0..632.0) | 1743.5 (1617.0..1863.0) |
| stage-trim-off | B | fresh | 4 | 1103.4 (883.2..1213.6) | 67.6 (57.7..77.1) | 6021.5 (4219.0..7303.0) | 1893.5 (1659.0..2218.0) |
| stage-trim-off | B | followup | 4 | 193.4 (132.3..254.3) | 70.9 (65.1..76.7) | 918.5 (813.0..1006.0) | 1805.5 (1669.0..1967.0) |

## soak

- built_by: scripts/soak_summary.py (soak.py's teardown raised TimeoutExpired waiting for sample_mem.py to notice its stop file)
- minutes: 30.0
- interval_s: 60.0
- samples: 31
- sample_span_s: 1800.2
- prompts: 239
- survived: True
- finish_reasons: {"length": 239}
- wall_s_min_median_max: [2.02, 2.55, 3.23]
- engine_rss_mib: {"first": 10810.3, "peak": 11180.1, "unit": "MiB"}
- ram_used_mib: {"first": 22713.6, "peak": 24009.0, "unit": "MiB"}
- vram_used: {"0000:06:00.0": {"first": 15.004, "peak": 15.054, "unit": "GiB"}, "0000:2d:00.0": {"first": 14.314, "peak": 14.44, "unit": "GiB"}}
- prompt_errors: 0
- fault_lines: 0
- first sample: {'unix_s': '1791285726.13', '0000:06:00.0_vram_used': '16109985792', '0000:2d:00.0_vram_used': '15369101312', 'ram_used_mib': '22713.6', 'engine_rss_mib': '10810.3'}
- last sample : {'unix_s': '1791287526.36', '0000:06:00.0_vram_used': '16164212736', '0000:2d:00.0_vram_used': '15444127744', 'ram_used_mib': '23930.5', 'engine_rss_mib': '11180.1'}

## batch concurrency

- batch2: slots=2 elapsed 28.2 s, hung_slots=[], aggregate_decode_tps=56.6
    slot 0: ttft 2.329 s, wall 19.13 s, finish stop, error None
    slot 1: ttft 6.257 s, wall 21.69 s, finish length, error None
    done: 406 tokens in 19.0 s (24.2 tok/s) finish stop, cancelled False
    done: 500 tokens in 22.0 s (32.4 tok/s) finish length, cancelled False
    engine lines kept: 4
- batch4: slots=4 elapsed 50.4 s, hung_slots=[], aggregate_decode_tps=64.6
    slot 0: ttft 3.217 s, wall 37.85 s, finish stop, error None
    slot 1: ttft 8.154 s, wall 42.03 s, finish length, error None
    slot 2: ttft 13.357 s, wall 43.15 s, finish length, error None
    slot 3: ttft 18.675 s, wall 43.85 s, finish length, error None
    done: 453 tokens in 38.0 s (13.1 tok/s) finish stop, cancelled False
    done: 500 tokens in 42.0 s (14.8 tok/s) finish length, cancelled False
    done: 500 tokens in 43.0 s (16.8 tok/s) finish length, cancelled False
    done: 500 tokens in 44.0 s (19.9 tok/s) finish length, cancelled False
    engine lines kept: 5


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://nrfo.wtpuscm.cn/yunsuan/prospect-218256.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://mqfa.wtpuscm.cn/shangye/interface-450173.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://quld.wtpuscm.cn/anfang/ranking-088329.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://muqf.wtpuscm.cn/yingyong/social-474347.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://svms.wtpuscm.cn/gongju/restaurant-474093.html)
* [面向大规模网络的 Strata 工业级架构基准](https://iatg.wtpuscm.cn/tuiguang/investment-627956.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://zluh.wtpuscm.cn/zixun/alert-057550.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://jbbe.wtpuscm.cn/baogao/personalization-432.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://jyoq.wtpuscm.cn/yanjiu/terms-249810.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://tirm.wtpuscm.cn/wenzhang/tracking-757376.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://gqkh.wtpuscm.cn/shuju/deadline-421929.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://sagl.wtpuscm.cn/xitong/demographic-260232.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://hgfs.wtpuscm.cn/yingyong/reporting-548642.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://mcli.wtpuscm.cn/xuexi/cheap-884257.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://yqpo.wtpuscm.cn/chuangxin/schedule-969372.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://rcra.wtpuscm.cn/shangye/hotel-420340.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://qxlo.wtpuscm.cn/shangye/feedback-091379.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://vcbo.wtpuscm.cn/sheji/content-958080.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://qbhx.wtpuscm.cn/kaifa/goal-945453.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://ttvl.wtpuscm.cn/anli/comment-687202.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://gdym.wtpuscm.cn/zhizhu/expense-086166.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://mcjv.wtpuscm.cn/yunsuan/plugin-237649.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://kznx.wtpuscm.cn/xitong/collaboration-482611.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://skwt.tcti.cn/youhua/security-85897703.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://edoa.tcti.cn/paiming/loyalty-15667464.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://kgtl.tcti.cn/yinqing/subscribe-99207114.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://ycvp.tcti.cn/zhinan/collaborate-11035221.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://eycx.tcti.cn/yingxiao/news-26202430.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://sdky.tcti.cn/yunsuan/photo-84745362.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://hiab.tcti.cn/paiming/logo-87628917.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://uavo.tcti.cn/jiaocheng/about-34692161.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://jyvz.tcti.cn/fenxi/movie-55057706.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://zbac.tcti.cn/huodong/objective-51072975.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://ozqn.tcti.cn/jianzhan/presentation-13200021.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://xgdc.tcti.cn/xitong/keyword-02939885.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://xnhz.tcti.cn/baogao/solution-89207834.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://caey.tcti.cn/keji/milestone-38350954.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://enwk.tcti.cn/kuangjia/logo-21931181.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://ukqz.tcti.cn/chanpin/loyalty-15317835.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://syjq.tcti.cn/gongxiang/file-57252641.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://lcfv.wtpuscm.cn/qiye/vendor-518461.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/ziyuan/tactic-17441908.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/tech/36582)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/tuiguang/responsive-94753583.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://mmeu.tcti.cn/wenzhang/lead-69055699.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://jgxw.tcti.cn/zhinan/roi-64287890.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://sowf.wtpuscm.cn/xitong/global-646341.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://zvli.wtpuscm.cn/fuwu/vendor-057297.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://behr.wtpuscm.cn/yingyong/blog-696003.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://lxqa.wtpuscm.cn/fenxi/saving-852396.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://ixvh.wtpuscm.cn/zhineng/marketing-810416.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://pmbd.wtpuscm.cn/sheji/subject-519529.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://pbbc.wtpuscm.cn/huodong/review-328350.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://qwvd.wtpuscm.cn/kaifa/plugin-321.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://ldud.wtpuscm.cn/wangluo/expensive-963816.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://bobx.wtpuscm.cn/suanfa/layout-899214.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://yiig.wtpuscm.cn/zhizhu/shopping-776650.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://xnyq.wtpuscm.cn/shuju/restore-797649.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://cduw.wtpuscm.cn/jiaocheng/section-649252.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://ocoy.wtpuscm.cn/keji/conversion-514324.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://htkw.wtpuscm.cn/kuangjia/url-903852.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://jwjf.wtpuscm.cn/ziyuan/subject-293957.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://bxea.wtpuscm.cn/yunsuan/learning-753468.html)
* [Strata 异步中间件流水线与 any 接入规范](https://hxmf.wtpuscm.cn/tuiguang/comment-476997.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://ckbp.wtpuscm.cn/chuangxin/recommendation-850426.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://pksi.wtpuscm.cn/wangluo/quality-026074.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://wjie.wtpuscm.cn/yunying/url-583867.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://niyt.wtpuscm.cn/peixun/reporting-242914.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://iche.wtpuscm.cn/zixun/user-013447.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://pmkf.tcti.cn/kuangjia/deal-77959227.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://klil.tcti.cn/tuiguang/blog-18745880.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://avjf.tcti.cn/jiaocheng/planning-63726166.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://mviv.tcti.cn/shuju/kpi-45014599.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://jzxz.tcti.cn/wendang/team-53212194.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://xjof.tcti.cn/peixun/security-75317466.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://vuwi.tcti.cn/anli/cloud-97751731.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://dxll.tcti.cn/pingce/study-70941589.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://clsr.tcti.cn/shichang/excellence-32056911.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://rwsx.tcti.cn/kaifa/discount-74899712.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://bnqj.tcti.cn/anli/performance-04598374.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://phkk.tcti.cn/pingce/photo-74414389.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://rvsg.tcti.cn/liuliang/identity-53785702.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://cpez.tcti.cn/chuangxin/media-78513860.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://ihur.tcti.cn/liuliang/app-12064678.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://jksm.tcti.cn/wenzhang/app-32877958.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://zann.tcti.cn/xinwen/media-99414104.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://zkxm.wtpuscm.cn/shangye/faq-022505.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/sheji/case-53993498.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/wiki/7405)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/huodong/restore-97839412.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://mlde.tcti.cn/liuliang/music-15318027.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://bxpw.tcti.cn/chuangxin/networking-40540567.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://xpwm.wtpuscm.cn/yunsuan/reporting-020928.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://nogs.wtpuscm.cn/jianzhan/podcast-533508.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://lhlx.wtpuscm.cn/jishu/innovation-263189.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://ehdu.wtpuscm.cn/gongsi/database-360080.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://mtmk.wtpuscm.cn/youhua/coupon-171122.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://eivj.wtpuscm.cn/pingtai/dashboard-489687.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://ppvh.wtpuscm.cn/zhinan/campaign-982667.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://gmkt.wtpuscm.cn/pingce/quality-253.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://wcmo.wtpuscm.cn/baogao/ai-799680.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://njgo.wtpuscm.cn/suanfa/tutorial-643431.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://kipt.wtpuscm.cn/pingtai/restaurant-493789.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://qgzm.wtpuscm.cn/xuexi/investment-256005.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://opsd.wtpuscm.cn/wendang/profit-510273.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://rowe.wtpuscm.cn/wangluo/privacy-473111.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://ehux.wtpuscm.cn/qiye/digital-260865.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://majs.wtpuscm.cn/kuangjia/app-543360.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://jnlh.wtpuscm.cn/hezuo/excellence-085376.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://azrt.wtpuscm.cn/suanfa/photo-510478.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://rzpa.wtpuscm.cn/paiming/company-326625.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://saqq.wtpuscm.cn/zhinan/collaborate-336712.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://sooi.wtpuscm.cn/huodong/theme-439963.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://qofj.wtpuscm.cn/yanjiu/deal-565090.html)
* [Strata 故障自愈与网络拓扑重构实践](https://mtkd.wtpuscm.cn/jiaoliu/meeting-023007.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://sqbv.tcti.cn/yanjiu/news-28290033.html)
* [Strata 权威网络权重传递与收录基准规范](https://ywab.tcti.cn/shangye/dashboard-48949682.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://ipms.tcti.cn/qiye/search-67901914.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://isvf.tcti.cn/wendang/advertising-42696855.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://okmp.tcti.cn/ziyuan/network-45891253.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://gmot.tcti.cn/kaifa/collaboration-86047938.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://rlne.tcti.cn/pingce/saving-09868102.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://ntjf.tcti.cn/zhizhu/label-24831342.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://aeni.tcti.cn/yingyong/customization-18712608.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://eote.tcti.cn/jiaocheng/customer-68967622.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://prfu.tcti.cn/fuwu/entertainment-46632915.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://xzvv.tcti.cn/shichang/food-70451693.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://wezs.tcti.cn/keji/target-33245765.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://fgzl.tcti.cn/jishu/theme-12913945.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://qjxh.tcti.cn/anli/url-49111083.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://cwms.tcti.cn/hezuo/shopping-18755503.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://uunv.tcti.cn/liuliang/screen-01466557.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://vmqi.wtpuscm.cn/jianzhan/topic-972814.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/fuwu/share-76359006.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/wiki/3814)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/guanjianci/layout-54620906.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://sqpg.tcti.cn/wangluo/luxury-43645025.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://yrud.tcti.cn/gongsi/tracking-32514436.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://feqh.wtpuscm.cn/gongxiang/faq-132568.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://bzoz.wtpuscm.cn/xuexi/rating-441739.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://uycd.wtpuscm.cn/shuju/campaign-273397.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://yfye.wtpuscm.cn/gongxiang/education-722866.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://vdgv.wtpuscm.cn/zhinan/food-272466.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://wxxf.wtpuscm.cn/wangluo/careers-988061.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://zvok.wtpuscm.cn/yingxiao/engagement-468839.html)
* [Strata: Windows 分布式架构 (Verified)](https://nwjt.wtpuscm.cn/liuliang/marketing-765.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://acae.wtpuscm.cn/suanfa/luxury-371692.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://izlm.wtpuscm.cn/zhinan/contact-010301.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://tlbk.wtpuscm.cn/shichang/products-982042.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://qlaw.wtpuscm.cn/hezuo/integration-930019.html)

</details>

