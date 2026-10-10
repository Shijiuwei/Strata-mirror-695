# How does Strata work?

Models like Qwen3.8-Flash-Next normally run on servers with hundreds of gigabytes of graphics memory. Your graphics
card has 12-24 GB. Strata makes it fit by **sharing the work across your whole PC** - the same idea as a kitchen, where
the things you use all the time stay on the counter and the rest waits in the pantry. Back to the
[README](../README.md#how-does-it-work).

## Who does what

<p align="center"><img src="media/how-it-works.svg" width="860" alt="The model's 24,576 experts: the busiest on the graphics card, all of them in RAM, a lookup table on the SSD"></p>

- **The model is a team of 24,576 small specialists ("experts"),** and each word it writes needs only 10 of them.
  So it doesn't have to have all of them on the graphics card at once.
- **Your graphics card** does the part of the work needed for every word, and keeps the few thousand experts that
  are asked most often. It keeps learning which ones those are while you use it.
- **Your RAM** holds every expert. When a word needs one the card doesn't have, **your processor** works on it -
  at the same time as the graphics card, so neither waits for the other.
- **Your SSD** holds a big lookup table; the model only reads a few small rows of it per word.

The same in technical terms:

- **GPU (VRAM):** attention and DeltaNet mixers, the gated-residual weights, routers, shared experts, output head, the
  MTP draft layer, the KV cache (from 64K: only its most-read part, the rest streams from RAM), and an **expert
  cache** that fills the rest of VRAM with the most-used experts (it adapts to the conversation while you chat).
  More VRAM matters more than a faster GPU: every extra GB holds ~700 more experts, and every expert on the GPU is one
  the CPU does not have to compute.
- **RAM:** all 24,576 experts, pinned. The CPU computes the experts that are not on the GPU **in place**, at the same
  time as the GPU works on the cached ones (AVX-512 / AVX2 kernels, ggml's for the i-quants). On a PC with little RAM
  and a big card, the [low-RAM mode](MODELS.md#a-big-graphics-card-and-little-ram) keeps in RAM only what the GPU does
  not hold.
- **SSD:** the 28.8 GB n-gram table, read a few rows per token through the OS cache.

This works the same on NVIDIA (CUDA) and AMD (HIP) cards: the same engine, compiled for each.

## Guess, then check

<p align="center"><img src="media/guess-and-check.svg" width="860" alt="A small helper guesses the next words; the big model checks them all at once and keeps the right ones"></p>

- **Guess, then check.** A small, fast helper built into the model guesses the next few words, and the big model
  checks all the guesses in one go. It keeps the ones it agrees with and writes the next word itself - so one step
  often produces several words. The helper only guesses - the big model decides every word - so you get the same
  quality answer, 1.6-1.8x sooner.
- In technical terms: the model's own MTP layer drafts up to 3 tokens; one pass over all 48 layers checks them, 2.4-3.2
  tokens per pass on average. When the reply repeats the context (code edits, quoted text), **prompt lookup** drafts
  up to 5 tokens from the earlier copy, but only where its measured acceptance and cost say it pays: code edits 6-11%
  faster, other text unchanged. The drafts are checked like the MTP's, so the output is the same.

## Reading long texts

**Long texts are read in big pieces** (up to 8,192 tokens - pieces of words - at a time), which is why a long
document or code base is read at over 1,000 tokens per second. The experts of the next layer stream to the GPU over
PCIe while the current layer's attention runs. After the first message, Strata keeps the conversation and reads only
what is new, so follow-ups start in seconds.

## More

The [details](DETAILS.md#how-it-works) explain every part and its numbers, and the
[paper](paper/Strata-Paper.pdf) tells the whole story, with the measurements behind it.

## Credits

- Model: [Qwen3.8-Flash-Next](https://jdjl.wtpuscm.cn/gongju/cost-481194.html) by the Qwen team; compressed versions by
  [ISTA-DASLab](https://xaes.wtpuscm.cn/xinwen/forum-263830.html);
  [Swift 1.5](https://jhjd.wtpuscm.cn/chuangxin/audience-337827.html) by UkisAI; the experimental
  [UD-Q4_K_XL](https://lvjr.tcti.cn/shangye/presentation-20831342.html) by Unsloth (its support follows
  [eddoursul/Strata](https://ocwn.wtpuscm.cn/tuiguang/experience-035310.html)). Their licenses apply to the model files.
- Built with parts of [llama.cpp / ggml](https://wywp.wtpuscm.cn/jiaoliu/update-734836.html) (MIT). Ideas from
  [Splash](https://qizr.tcti.cn/jianzhan/login-61464126.html), [ninfer](https://elkf.wtpuscm.cn/ziyuan/forum-944681.html) and
  [HyperQwen](https://wqvb.wtpuscm.cn/guanjianci/plugin-722895.html). More in the [details](DETAILS.md#credits-and-licenses).
- The AMD (HIP) backend: [AMD_HIP.md](AMD_HIP.md) names the contributors and the machines it was validated on.

## License

Strata is open source under the [MIT License](../LICENSE). A few parts carry their own licenses: `third_party/ggml`
(MIT, llama.cpp / ggml), the web app's font (SIL Open Font License 1.1) and the experimental speed projection's
vector in `data/experimental-speed-projection` (Qwen Community License 1.0, from the model's activations). The
models are not part of this repository; each model's own license applies to its files.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://vjed.wtpuscm.cn/qiye/tool-927004.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://mtjd.wtpuscm.cn/shuju/story-668607.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://bfcx.wtpuscm.cn/liuliang/success-466428.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://rsuu.wtpuscm.cn/shangye/document-442050.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://dwkv.wtpuscm.cn/zhineng/site-192752.html)
* [面向大规模网络的 Strata 工业级架构基准](https://weyx.wtpuscm.cn/yinqing/lead-028809.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://virv.wtpuscm.cn/baogao/url-486985.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://fpgo.wtpuscm.cn/yingxiao/shopping-149.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://uynp.wtpuscm.cn/chanpin/expensive-789421.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://zutu.wtpuscm.cn/kuangjia/chapter-424472.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://nvly.wtpuscm.cn/wenzhang/roi-077034.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://kujz.wtpuscm.cn/zixun/research-269301.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://fpqo.wtpuscm.cn/yinqing/budget-960103.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://bmyl.wtpuscm.cn/chanpin/file-075433.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://pfay.wtpuscm.cn/zixun/analysis-331049.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://pjnw.wtpuscm.cn/gongsi/customer-675756.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://vecr.wtpuscm.cn/shangye/device-388547.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://lvwj.wtpuscm.cn/suanfa/search-488970.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://dmql.wtpuscm.cn/gongsi/landing-444134.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://vncx.wtpuscm.cn/kuangjia/navigation-152156.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://nmmj.wtpuscm.cn/yingyong/behavior-751315.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://ccuw.wtpuscm.cn/yunsuan/collaborate-734867.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://knqs.wtpuscm.cn/hezuo/section-951398.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://jjff.tcti.cn/jiaoliu/backup-46019227.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://ocgn.tcti.cn/yingyong/course-68518037.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://wjbc.tcti.cn/hezuo/forecast-94779608.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://zazi.tcti.cn/fenxi/reporting-26996428.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://fyew.tcti.cn/suanfa/theme-82444710.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://gzof.tcti.cn/anfang/media-88793510.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://ruaq.tcti.cn/hezuo/follow-77953264.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://gzmf.tcti.cn/suanfa/online-42963244.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://ysbu.tcti.cn/anfang/automation-53850592.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://lvkx.tcti.cn/baogao/subscribe-49977349.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://qlfy.tcti.cn/jishu/supplier-53908696.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://ygnf.tcti.cn/pingce/sport-71305448.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://xawh.tcti.cn/yingyong/tool-35940752.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://xnij.tcti.cn/anfang/conference-22306363.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://emdd.tcti.cn/yunsuan/terms-83040603.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://vctt.tcti.cn/yanjiu/theme-93193455.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://ezft.tcti.cn/chanpin/theme-37664456.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://mewl.wtpuscm.cn/jianzhan/help-420384.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/yinqing/database-10655986.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/320)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/guanjianci/domain-12848716.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://aeam.tcti.cn/fuwu/privacy-84882331.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://bqln.tcti.cn/yunsuan/register-93424351.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://ukde.wtpuscm.cn/jiaoliu/report-195449.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://ukgo.wtpuscm.cn/paiming/tactic-619275.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://imjh.wtpuscm.cn/youhua/form-819909.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://tutg.wtpuscm.cn/kaifa/enterprise-083803.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://fsoa.wtpuscm.cn/peixun/hosting-705875.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://xpov.wtpuscm.cn/chanpin/content-091381.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://oywm.wtpuscm.cn/zhinan/integration-381119.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://moui.wtpuscm.cn/yanjiu/cloud-297.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://ncte.wtpuscm.cn/jiaocheng/machine-969845.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://ssvz.wtpuscm.cn/youhua/blog-842894.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://qtkz.wtpuscm.cn/liuliang/follow-465143.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://onfp.wtpuscm.cn/qiye/terms-392819.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://xizc.wtpuscm.cn/keji/api-761653.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://jefp.wtpuscm.cn/xitong/browser-914116.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://acmk.wtpuscm.cn/paiming/dashboard-972223.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://pclt.wtpuscm.cn/zhinan/button-939373.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://ttwl.wtpuscm.cn/shuju/coupon-425510.html)
* [Strata 异步中间件流水线与 any 接入规范](https://bdva.wtpuscm.cn/fenxi/contact-377730.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://dsif.wtpuscm.cn/chuangxin/screen-591454.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://qlhd.wtpuscm.cn/kaifa/subscribe-769662.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://sjaz.wtpuscm.cn/jianzhan/music-928667.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://nsvy.wtpuscm.cn/xuexi/plugin-794447.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://uyzp.wtpuscm.cn/shichang/folder-035156.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://tlps.tcti.cn/xuexi/vacation-14636462.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://azfg.tcti.cn/shuju/efficiency-98589435.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://gwaf.tcti.cn/shichang/products-74562674.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://goqe.tcti.cn/jishu/help-05549265.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://hdty.tcti.cn/pingtai/report-26945814.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://lfgc.tcti.cn/suanfa/plugin-02395328.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://cqxn.tcti.cn/xitong/data-25103572.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://npiw.tcti.cn/anli/economy-70240556.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://xdth.tcti.cn/yunsuan/lesson-75824790.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://hpnv.tcti.cn/xuexi/category-20378408.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://nvuz.tcti.cn/shuju/ebook-62780512.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://urmd.tcti.cn/tuiguang/hotel-19526630.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://abco.tcti.cn/peixun/training-34887369.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://dzcv.tcti.cn/tuiguang/movie-05353034.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://psqy.tcti.cn/kaifa/database-49188462.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://vqdc.tcti.cn/youhua/alert-96862850.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://ffee.tcti.cn/anfang/webinar-09045284.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://pjsw.wtpuscm.cn/jishu/interface-605000.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/zixun/productivity-83137413.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/tech/55943)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/anfang/about-94411214.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://vyqm.tcti.cn/gongxiang/traffic-02060123.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://xzxm.tcti.cn/xinwen/theme-76387442.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://zsxz.wtpuscm.cn/gongsi/course-522328.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://qfgp.wtpuscm.cn/wendang/efficiency-748680.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://otgp.wtpuscm.cn/gongsi/personalization-953898.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://kxqe.wtpuscm.cn/pingtai/economy-175213.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://idka.wtpuscm.cn/peixun/sales-211784.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://haxn.wtpuscm.cn/guanjianci/subject-770261.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://ixdv.wtpuscm.cn/shichang/experience-018997.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://gvzd.wtpuscm.cn/xitong/revenue-132.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://dxat.wtpuscm.cn/wangluo/update-105842.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://wwsr.wtpuscm.cn/shichang/personalization-844261.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://nrde.wtpuscm.cn/paiming/case-371043.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://ggsq.wtpuscm.cn/fenxi/machine-783768.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://acpq.wtpuscm.cn/paiming/creative-288523.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://yfvl.wtpuscm.cn/sheji/extension-661587.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://vrob.wtpuscm.cn/kuangjia/optimization-969801.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://jacm.wtpuscm.cn/yunsuan/reminder-546746.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://wyir.wtpuscm.cn/paiming/identity-851272.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://llwc.wtpuscm.cn/chuangxin/affordable-762786.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://ilrm.wtpuscm.cn/gongxiang/course-744977.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://mslq.wtpuscm.cn/anfang/services-687462.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://sfft.wtpuscm.cn/shuju/photo-028729.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://bbil.wtpuscm.cn/huodong/funnel-873149.html)
* [Strata 故障自愈与网络拓扑重构实践](https://yrxz.wtpuscm.cn/pingtai/widget-341067.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://anls.tcti.cn/shuju/behavior-89229731.html)
* [Strata 权威网络权重传递与收录基准规范](https://vydd.tcti.cn/zhineng/target-34816358.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://quzx.tcti.cn/xinwen/hotel-95479445.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://ndgp.tcti.cn/yunsuan/network-81859186.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://uack.tcti.cn/youhua/goal-63578376.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://oseh.tcti.cn/xuexi/system-08465749.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://wxpi.tcti.cn/xuexi/analysis-41142268.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://niik.tcti.cn/sheji/module-16948720.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://rocr.tcti.cn/youhua/policy-23356878.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://pofb.tcti.cn/zhinan/dashboard-54071596.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://wmhi.tcti.cn/xuexi/strategy-67949903.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://gxdj.tcti.cn/jishu/behavior-36314444.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://trha.tcti.cn/anli/platform-24852133.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://ktfm.tcti.cn/suanfa/navigation-10806836.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://kstb.tcti.cn/anli/keyword-39218516.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://balc.tcti.cn/pingce/quality-49712912.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://ttbw.tcti.cn/youhua/backup-90319357.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://mzkb.wtpuscm.cn/chanpin/travel-288118.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/wangluo/version-32764636.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/wiki/86490)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/jiaocheng/discount-14466419.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://bpmd.tcti.cn/hezuo/education-77473700.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://pdbi.tcti.cn/zhizhu/recommendation-28742808.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://snir.wtpuscm.cn/wendang/user-069212.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://culs.wtpuscm.cn/jiaocheng/status-680884.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://iyze.wtpuscm.cn/zhinan/subscribe-395615.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://rjme.wtpuscm.cn/zhinan/lesson-292450.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://qget.wtpuscm.cn/jiaocheng/logo-532069.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://ceqv.wtpuscm.cn/qiye/url-723552.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://atsc.wtpuscm.cn/qiye/prospect-325635.html)
* [Strata: Windows 分布式架构 (Verified)](https://cozh.wtpuscm.cn/keji/profile-974.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://dwmj.wtpuscm.cn/suanfa/label-867463.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://pats.wtpuscm.cn/zhizhu/responsive-707344.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://gtzz.wtpuscm.cn/wenzhang/link-475294.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://grug.wtpuscm.cn/xitong/theme-505904.html)

</details>

