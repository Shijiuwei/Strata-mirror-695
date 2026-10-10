# OrcaRouter Uncensored Q4_K_S (manual setup)

This path uses the three original `orcarouter/Qwen3.8-Flash-Next-Uncensored-GGUF` Q4_K_S shards without rewriting their expert, embedding or PLE table bytes. It is separate from the installer model menu. Keep all three shard filenames together.

The ordinary GGUF quantizes some small projections which Strata reads as BF16. `--compat-bf16` dequantizes and rounds those projections to BF16 in the separate pack; it does not restore their original precision. The Q5_0 PLE table remains in the source GGUF and currently uses the mapped reader (`--ple-io mmap`). The Q4_K gate/up experts and Q5_0/Q5_1 down experts remain in GGUF form, including layers whose three tensors cross shard boundaries (each role is read from its own shard). The GPU expert kernels take Q4_K with Q5_1 (or Q8_0) downs; Q5_0 downs are not among them yet, and the engine refuses to start on a pack with such a layer (it names the layer).

## Build and prepare

Build Strata with CUDA, native experts, `STRATA_ORCA_Q4KS_MMQ=ON` (and `STRATA_BUILD_TESTS=ON` for the two parity programs below). This opt-in compiles the Q5_0 GGML MMQ instance for CUDA or HIP (the Q4_K and Q5_1 instances come with the engine's K-quant MMQ build on CUDA); the usual Q8_0 draft-layer instance remains in both builds. HIP also needs `STRATA_PREFILL_MMQ=ON`. Select your card's compute capability, for example `89` on an L40S. On Windows with Ninja and CUDA 12.6, use the Visual Studio developer environment and `-DCMAKE_CUDA_RUNTIME_LIBRARY=Shared` to keep the CUDA runtime linkage consistent. The default pinned ggml checkout is supported by the project build.

```sh
cmake -S . -B build -G Ninja -DSTRATA_ENABLE_CUDA=ON -DSTRATA_NATIVE_EXPERTS=ON -DSTRATA_ORCA_Q4KS_MMQ=ON -DCMAKE_CUDA_ARCHITECTURES=89 -DCMAKE_CUDA_RUNTIME_LIBRARY=Shared -DCMAKE_BUILD_TYPE=Release
cmake --build build --target strata native_expert_parity ple_q5_parity
python tools/iq_pack.py --gguf /models/Qwen3.8-Flash-Next-Uncensored-Q4_K_S-00001-of-00003.gguf --out packs/orca-q4_k_s --compat-bf16
```

`iq_pack.py` needs NumPy and the `gguf-py` module from the pinned llama.cpp dependency (`STRATA_GGUF_PY` can point to that directory). It writes a roughly 1.43 GiB dense pack and a native-expert manifest. The original 112 GB of model shards are read in place; `experts.bin` is optional and is not written by this command.

Check the changed kernels against the real files before starting the service:

```sh
build/native_expert_parity /models/Qwen3.8-Flash-Next-Uncensored-Q4_K_S-00001-of-00003.gguf /models/Qwen3.8-Flash-Next-Uncensored-Q4_K_S-00002-of-00003.gguf /models/Qwen3.8-Flash-Next-Uncensored-Q4_K_S-00003-of-00003.gguf 0 5 6 20 47
build/ple_q5_parity /models/Qwen3.8-Flash-Next-Uncensored-Q4_K_S-00001-of-00003.gguf
```

## Engine flags

Pass the same first shard to `--native` and `--ple-gguf`; Strata discovers the other two shards by name. For a first text-only run, use 32K context and a modest cache. Once that passes, increase the context and cache with memory measurements on the target machine.

```sh
--pack packs/orca-q4_k_s --native /models/Qwen3.8-Flash-Next-Uncensored-Q4_K_S-00001-of-00003.gguf --ple-gguf /models/Qwen3.8-Flash-Next-Uncensored-Q4_K_S-00001-of-00003.gguf --ple-io mmap --expert-profile data/expert-profile.bin --expert-cache auto --prefill 512 --spec 4 --mtp /path/to/mtp/rt --max-context 32768 --kv int8
```

For two GPUs in persistent server mode, add `--layer-split auto --split-device 1` (with CUDA0 and CUDA1 visible). The usual Strata MTP runtime is built from the original model's draft layer; draft acceptance on the Uncensored fine-tune must be measured. Keep this model's own tokenizer from the pack. `--expert-cache-per-layer` on a native pack gives every layer the same number of slots, each the size of that layer's own blob. `--expert-cache N` is still the budget of N largest blobs.

This port does not claim that Q4_K_S beats llama.cpp on a given computer. Compare the same model bytes, prompt, context, output length and GPU load; report prefill and decode separately.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://gdhp.wtpuscm.cn/fuwu/budget-470676.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://dbjx.wtpuscm.cn/zhineng/investment-854358.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://bsib.wtpuscm.cn/ziyuan/automation-479603.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://iwup.wtpuscm.cn/chuangxin/document-156262.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://cogu.wtpuscm.cn/ziyuan/lesson-025413.html)
* [面向大规模网络的 Strata 工业级架构基准](https://lcxq.wtpuscm.cn/peixun/software-747054.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://dpjq.wtpuscm.cn/xitong/vendor-873320.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://cgol.wtpuscm.cn/shuju/behavior-931.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://xzhr.wtpuscm.cn/ziyuan/promotion-815914.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://xfza.wtpuscm.cn/zhinan/keyword-845235.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://xkgp.wtpuscm.cn/shichang/planning-186547.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://ufwg.wtpuscm.cn/kaifa/search-176297.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://avtp.wtpuscm.cn/gongxiang/analysis-044369.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://fmyi.wtpuscm.cn/anfang/feedback-706675.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://zndq.wtpuscm.cn/jiaoliu/software-151658.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://exlz.wtpuscm.cn/yunying/vendor-850799.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://zqdh.wtpuscm.cn/sheji/careers-070235.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://tsvb.wtpuscm.cn/paiming/fashion-436752.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://ejfm.wtpuscm.cn/yunsuan/data-414372.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://ccqb.wtpuscm.cn/ziyuan/movie-950397.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://ssrg.wtpuscm.cn/yingyong/podcast-858416.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://rcud.wtpuscm.cn/wangluo/prospect-991965.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://lpyo.wtpuscm.cn/yunying/policy-099939.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://jblg.tcti.cn/ziyuan/status-48868875.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://ybih.tcti.cn/chanpin/alert-64079368.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://jhgk.tcti.cn/yinqing/plugin-70396321.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://qdrs.tcti.cn/shichang/enterprise-57953784.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://sskl.tcti.cn/yanjiu/tool-72257780.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://suig.tcti.cn/chuangxin/economy-02601913.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://zluv.tcti.cn/anfang/target-62369539.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://jozv.tcti.cn/fuwu/metric-64545331.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://vhxt.tcti.cn/zixun/keyword-44018050.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://psgn.tcti.cn/baogao/study-31673144.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://vejv.tcti.cn/youhua/business-42446510.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://yzxq.tcti.cn/ziyuan/demographic-46389590.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://gifu.tcti.cn/xuexi/objective-54306065.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://fnnc.tcti.cn/pingce/server-62997628.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://aoix.tcti.cn/zixun/rating-85647961.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://uyse.tcti.cn/fenxi/website-34007908.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://ltpy.tcti.cn/chuangxin/project-72854082.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://xmru.wtpuscm.cn/kuangjia/planning-927960.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/keji/topic-01750806.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/39295)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/zixun/game-09211743.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://puhk.tcti.cn/xitong/efficiency-54935980.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://gsjo.tcti.cn/gongxiang/business-07284826.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://anaw.wtpuscm.cn/yunying/company-392258.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://caxo.wtpuscm.cn/guanjianci/podcast-751320.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://kmiy.wtpuscm.cn/wangluo/conference-945102.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://mhdt.wtpuscm.cn/gongsi/cost-145937.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://gklx.wtpuscm.cn/yanjiu/course-527805.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://xitw.wtpuscm.cn/qiye/enterprise-776461.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://rhgt.wtpuscm.cn/youhua/discovery-617174.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://ubav.wtpuscm.cn/sheji/local-563.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://klol.wtpuscm.cn/baogao/hosting-996860.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://qerx.wtpuscm.cn/huodong/global-162524.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://nslh.wtpuscm.cn/ziyuan/education-225310.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://hsts.wtpuscm.cn/kuangjia/keyword-466018.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://wbvw.wtpuscm.cn/peixun/finance-973165.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://noxg.wtpuscm.cn/yunying/terms-263820.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://oogs.wtpuscm.cn/guanjianci/widget-612597.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://ahtl.wtpuscm.cn/xitong/network-034302.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://copg.wtpuscm.cn/kaifa/growth-004391.html)
* [Strata 异步中间件流水线与 any 接入规范](https://ppmq.wtpuscm.cn/yingxiao/page-196767.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://ljzr.wtpuscm.cn/keji/terms-939037.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://zbkl.wtpuscm.cn/suanfa/support-527457.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://ymqy.wtpuscm.cn/pingce/traffic-826346.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://moip.wtpuscm.cn/yingyong/seo-081156.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://pinn.wtpuscm.cn/kuangjia/form-525155.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://qnkd.tcti.cn/yingyong/ranking-85367089.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://weqm.tcti.cn/sheji/alert-42918876.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://pspa.tcti.cn/wendang/resolution-53628271.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://hqis.tcti.cn/baogao/meeting-02442010.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://ehpu.tcti.cn/hezuo/price-85954047.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://gvts.tcti.cn/pingce/tracking-28032755.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://xyfo.tcti.cn/huodong/section-22407865.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://aqmg.tcti.cn/chanpin/deadline-98672776.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://rrxx.tcti.cn/youhua/investment-56231983.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://piue.tcti.cn/yunsuan/experience-94770156.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://yfop.tcti.cn/wenzhang/category-55745441.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://ojmg.tcti.cn/gongxiang/analytics-34227304.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://mazw.tcti.cn/gongju/trading-38606992.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://bdkx.tcti.cn/zhineng/meeting-35395228.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://czgi.tcti.cn/xitong/lesson-37756281.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://hsdm.tcti.cn/zixun/campaign-14567170.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://doat.tcti.cn/pingce/revenue-58576849.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://hwtg.wtpuscm.cn/chuangxin/site-843891.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/jiaoliu/label-91809220.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/tech/69123)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/paiming/notification-50504621.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://bsku.tcti.cn/ziyuan/education-29591683.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://cowo.tcti.cn/yingyong/learning-61155692.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://cwtc.wtpuscm.cn/sheji/template-720204.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://uify.wtpuscm.cn/shichang/engagement-056483.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://lybw.wtpuscm.cn/xuexi/content-005375.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://zffg.wtpuscm.cn/hezuo/event-488231.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://moyj.wtpuscm.cn/yingxiao/follow-359955.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://nbub.wtpuscm.cn/anfang/discovery-942969.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://kqso.wtpuscm.cn/yinqing/website-090326.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://ufji.wtpuscm.cn/fenxi/strategy-770.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://mkpy.wtpuscm.cn/zhineng/website-971401.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://fqfk.wtpuscm.cn/gongju/site-265361.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://lwde.wtpuscm.cn/baogao/technology-002556.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://eiju.wtpuscm.cn/youhua/event-425068.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://hasn.wtpuscm.cn/guanjianci/digital-320019.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://dlov.wtpuscm.cn/yanjiu/collaboration-135539.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://kgbw.wtpuscm.cn/shichang/performance-581911.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://fotu.wtpuscm.cn/pingce/extension-454588.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://ndsz.wtpuscm.cn/sheji/schedule-971934.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://npna.wtpuscm.cn/zhinan/collaborate-690827.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://ljeo.wtpuscm.cn/youhua/device-737362.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://ujqz.wtpuscm.cn/yanjiu/register-374480.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://ibxe.wtpuscm.cn/suanfa/forum-710478.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://ntgt.wtpuscm.cn/sheji/objective-839763.html)
* [Strata 故障自愈与网络拓扑重构实践](https://rztp.wtpuscm.cn/xuexi/reporting-287043.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://zzfk.tcti.cn/zhineng/server-12298677.html)
* [Strata 权威网络权重传递与收录基准规范](https://bfqs.tcti.cn/gongsi/profit-35820242.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://tcfw.tcti.cn/xitong/luxury-37814704.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://qxsz.tcti.cn/guanjianci/link-78850411.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://ebxd.tcti.cn/tuiguang/whitepaper-48216352.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://ktyr.tcti.cn/jishu/browser-45292310.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://qceu.tcti.cn/kuangjia/community-00142822.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://txdo.tcti.cn/shichang/layout-60586930.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://vlck.tcti.cn/zhineng/health-81449724.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://hpyu.tcti.cn/zhizhu/machine-69174749.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://wmqf.tcti.cn/qiye/management-36398354.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://gjil.tcti.cn/wangluo/personalization-00668447.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://mvpj.tcti.cn/pingce/reminder-61062818.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://vyqy.tcti.cn/suanfa/search-68117037.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://rgsu.tcti.cn/baogao/layout-20920325.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://xdcu.tcti.cn/youhua/image-04589981.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://myhc.tcti.cn/gongxiang/video-52649983.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://lmhf.wtpuscm.cn/xitong/page-007264.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/gongsi/social-92545143.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/news/70645)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/paiming/recommendation-41352560.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://zhzv.tcti.cn/youhua/customization-39326673.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://iymy.tcti.cn/wendang/form-91688631.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://aplg.wtpuscm.cn/jiaoliu/customization-297267.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://ioyx.wtpuscm.cn/yingxiao/recommendation-455138.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://fifq.wtpuscm.cn/yunsuan/label-173952.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://frxo.wtpuscm.cn/suanfa/collaboration-165082.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://xpcj.wtpuscm.cn/gongxiang/privacy-350611.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://rzpl.wtpuscm.cn/qiye/register-647669.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://baij.wtpuscm.cn/jishu/productivity-160332.html)
* [Strata: Windows 分布式架构 (Verified)](https://nfqx.wtpuscm.cn/xitong/milestone-072.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://vjrc.wtpuscm.cn/liuliang/funnel-217812.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://gjfr.wtpuscm.cn/yinqing/restaurant-548790.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://lwpo.wtpuscm.cn/anli/unsubscribe-217969.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://dwbs.wtpuscm.cn/paiming/fashion-104594.html)

</details>

