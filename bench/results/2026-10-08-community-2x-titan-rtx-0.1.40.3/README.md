# Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4

Two consumer Turing cards, no AVX-512 on the CPU, and the model's native
262,144-token window. Reported per `docs/COMMUNITY_BENCHMARKS.md`: three runs
per configuration, medians and ranges, prompt and decode throughput kept apart,
and the recall check attached. Engine version 0.1.40.3 with `STRATA_STAGE_TRIM=1`.

> A note from the submitter, since numbers alone do not say why this was worth
> the weekend: I did not expect any of this to be possible. The whole point of
> the exercise was to see whether a 125B MoE could run at all on two consumer
> Turing cards, and it does — 70-80 tok/s of decode with the model's full
> 262,144-token window, on hardware that was never sold for this. Whatever you
> do with the engine, that part is genuinely surprising. Thank you.

## Hardware and software

| | |
| --- | --- |
| GPU | 2x NVIDIA TITAN RTX, 24 GB each (Turing, sm_75), driver 615.71.09 |
| GPU-GPU | `NV2` — 2 bonded NVLinks (`nvidia-smi topo -m`). **Not used**, see below |
| PCIe | x16, **gen 3** under load on both cards (gen 1 at idle), 220 W power limit each |
| CPU | Intel Xeon E5-2696 v4 @ 2.20 GHz, 22 cores / 44 threads, 1 socket, **no AVX-512** |
| RAM | 125 GiB |
| Storage | Samsung MZVLW1T0HMLH NVMe 1 TB (model), SATA SSD also present |
| OS | AlmaLinux 9.8, kernel 5.14.0-687.51.1.el9_8.x86_64 |
| CUDA | 12.9 (nvcc), source build |
| Strata | tag `v0.1.40.3` (commit `d5ea713`), built from source for `CMAKE_CUDA_ARCHITECTURES=75` |

Build used no non-default options beyond `-DSTRATA_ENABLE_CUDA=ON
-DSTRATA_BUILD_TESTS=OFF -DCMAKE_CUDA_ARCHITECTURES=75`. No prebuilt engine:
the release ships `strata-windows-x64.zip` only.

## Model and configuration

Qwen3.8-Flash-Next **IQ3_S** (3.5 bpw, the recommended quality tier),
`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`, two GGUF shards
(54,817,524,224 + 28,800,138,432 bytes), plus the MTP draft layer packed by
`setup.sh` into `Strata-data/mtp/rt` (787 MB).

```
--pack Strata-data/packs/iq3_s --native ...IQ3_S-00001-of-00002.gguf
--ple-gguf ...IQ3_S-00002-of-00002.gguf --expert-profile data/expert-profile.bin
--expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5
--mtp Strata-data/mtp/rt --max-context 262144 --kv int8 --kv-resident 65536
--vision --vram-reserve-mib 700 --ple-io ram --layer-split auto
--reasoning-budget-tokens 12000
```

`STRATA_STAGE_TRIM=1` in the environment (PR #639): each card loads only its
own layers' dense weights instead of a full copy, which returns the VRAM to
the expert cache. `host 0.0.0.0`, `api_key` set (removed here). Calibration
was not enabled and the experimental speed projection was off.

## Method and reproduction

`benchmark.py` in this directory. Three runs at each of 4,096 / 32,768 /
128,000 target prompt tokens, 256-token output cap, `temperature=0`, non-
streaming. **Each run sends a distinct prompt** (a `Document revision marker`
differs per run) so no run reuses another one's prefix; a warm expert cache is
shared across runs by design, and the engine log's `expert cache NN% hit` is
recorded per run.

Prompts are a repeated paragraph of MoE-routing prose, so the filler itself
selects a narrow expert set; the recall check below is the correctness
evidence for long context, not these prompts.

Startup (~1 min 10 s, reading ~55 GB of experts into RAM) is **not** included
in any timing. Decode throughput comes from the engine's own
`timings.predicted_per_second`, which counts reasoning tokens; TTFT is
wall-clock minus generation time.

## Results

Prompt throughput rises with prompt length — the opposite of a degradation
curve, presumably from amortising expert-cache warm-up and larger prefill
chunks.

| Prompt tokens | Prompt t/s (3 runs) | Median | Decode t/s (3 runs) | Median | TTFT (3 runs) |
| --- | --- | --- | --- | --- | --- |
| 4,096 | 886.8 / 922.6 / 922.8 | 922.6 | 75.4 / 65.1 / 80.3 | 75.4 | 3.46 / 3.33 / 3.32 s |
| 32,768 | 1647.2 / 1638.8 / 1642.1 | 1642.1 | 63.9 / 83.0 / 80.2 | 80.2 | 15.07 / 15.14 / 15.12 s |
| 128,000 | 1856.8 / 1852.8 / 1853.6 | 1853.6 | 70.6 / 66.9 / 72.2 | 70.6 | 52.21 / 52.32 / 52.31 s |

Longer replies decode at the same rate as short ones: a separate 2,000-token
run at a 95-token prompt measured ~59-70 t/s with an expert cache hit rate of
95-99%, so decode is context-independent here.

Both GPUs work throughout: sampled `utilization.gpu` during decode was
**57% / 42%** with 195 W / 155 W and ~1900 MHz on both. The imbalance is the
auto layer split (one card carries slightly more layers plus the MTP draft).
Idle: both drop to 0% and ~1350 MHz.

## Recall and limitations

`needle_bench.py --lengths 32k,128k --depths 10,50,90`: **6 of 6 found**, no
misses or errors. Actual prompt lengths 32,470 (32k) and 125,975-125,977
(128k); wall-clock 12-22 s (32k) and 65-72 s (128k).

**NVLink is present but unused.** `nvidia-smi topo -m` reports `NV2`
between the two cards, but per `docs/MULTI_GPU.md` the engine deliberately
does not use NVLink or peer-to-peer access: activations cross cards through
pinned RAM once per verify window rather than twice per layer, so the same
numbers should be expected on cards with no bridge. Do not expect a gain from
adding a bridge on consumer boards.

**`STRATA_STAGE_TRIM=1` and the expert cache** (the "please report how it
goes" switch). With it, each card loads only its own layers' dense weights
instead of a full copy, and the freed VRAM goes to the expert cache. On this
box it raised the resident expert count from **8,649 to 9,387** (+8.5%, 16.00
-> 16.91 GiB) and returned prompt throughput and TTFT to the levels measured
on 0.1.40.1 (~1,854 t/s and ~52 s at 128K). Decode was unchanged. As
`docs/MULTI_GPU.md` notes, a card holding more experts can change which
experts run on the GPU, so output can differ slightly from a run without it.

Settings compared and a note on `--ple-io ram`: on this box the `--ple-io
ram` and `direct` routes measure the same prompt throughput, while the `ram`
route holds 27.1 GiB more resident page cache. Kept here only because this
machine has RAM to spare; on a smaller machine the default is the better
choice.

**Expert cache map.** 9,387 experts cached this run, 16.91 GiB of VRAM with
the vision encoder resident. On an earlier 0.1.40.3 run without
`STRATA_STAGE_TRIM=1` the same box cached 8,649 experts.

Not measured: the experimental speed projection and the low-RAM variant.
Vision is enabled in the graph but this report covers text-only runs.

---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://czbl.wtpuscm.cn/youhua/about-197250.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://hrta.wtpuscm.cn/qiye/sales-454074.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://vjht.wtpuscm.cn/wendang/keyword-812131.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://phwy.wtpuscm.cn/gongsi/progress-451018.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://xncn.wtpuscm.cn/xuexi/lead-299522.html)
* [面向大规模网络的 Strata 工业级架构基准](https://xevi.wtpuscm.cn/xitong/management-629666.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://udvr.wtpuscm.cn/jianzhan/sport-739068.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://sdgj.wtpuscm.cn/zhinan/affordable-631.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://asou.wtpuscm.cn/jianzhan/update-868890.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://zskt.wtpuscm.cn/zhizhu/forum-855761.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://szjy.wtpuscm.cn/shangye/sale-911134.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://yegn.wtpuscm.cn/zixun/extension-838501.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://uzod.wtpuscm.cn/ziyuan/story-139996.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://opwg.wtpuscm.cn/huodong/lesson-924712.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://sqyz.wtpuscm.cn/xinwen/search-139241.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://kkoz.wtpuscm.cn/shangye/website-601474.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://jivr.wtpuscm.cn/chuangxin/target-715349.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://rkbf.wtpuscm.cn/zixun/budget-247633.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://wwji.wtpuscm.cn/yinqing/recommendation-443960.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://fzql.wtpuscm.cn/suanfa/document-989687.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://rswl.wtpuscm.cn/anli/promotion-358219.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://qydk.wtpuscm.cn/fuwu/loyalty-452305.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://vwhg.wtpuscm.cn/zhinan/message-353513.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://oxoq.tcti.cn/wenzhang/document-63294330.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://zmgz.tcti.cn/hezuo/page-70345076.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://gbrn.tcti.cn/huodong/template-48542133.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://zsre.tcti.cn/hezuo/photo-13767543.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://mltu.tcti.cn/shichang/machine-21434481.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://ubfl.tcti.cn/zhinan/achievement-48731427.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://yllg.tcti.cn/anli/workshop-87373678.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://qoum.tcti.cn/pingtai/landing-86597871.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://lbdk.tcti.cn/xinwen/faq-43409757.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://gork.tcti.cn/chuangxin/video-76938991.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://faqe.tcti.cn/anfang/creative-36901585.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://vgia.tcti.cn/guanjianci/segment-43498279.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://nrul.tcti.cn/shangye/contact-96036315.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://nkbp.tcti.cn/fenxi/ebook-12560993.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://kkqy.tcti.cn/liuliang/content-17340527.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://qraa.tcti.cn/shangye/policy-88798379.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://rekg.tcti.cn/shichang/widget-80546732.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://quje.wtpuscm.cn/zhineng/funnel-819602.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/youhua/progress-56153781.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/news/25874)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/zhinan/productivity-06293799.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://kzln.tcti.cn/chanpin/recipe-59744239.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://lngh.tcti.cn/chanpin/page-29953828.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://omyx.wtpuscm.cn/peixun/keyword-961474.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://edaw.wtpuscm.cn/zhinan/tutorial-156613.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://kgvw.wtpuscm.cn/keji/global-365301.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://pvlv.wtpuscm.cn/jishu/income-363509.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://ywto.wtpuscm.cn/zhineng/local-453501.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://euxn.wtpuscm.cn/chanpin/strategy-090822.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://eyxb.wtpuscm.cn/wenzhang/settings-306435.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://badv.wtpuscm.cn/suanfa/kpi-125.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://upyn.wtpuscm.cn/tuiguang/photo-125002.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://dfqf.wtpuscm.cn/zhizhu/local-527419.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://emce.wtpuscm.cn/zixun/article-974606.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://ecmv.wtpuscm.cn/kaifa/performance-775423.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://gydt.wtpuscm.cn/yanjiu/server-621000.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://hkil.wtpuscm.cn/jiaocheng/podcast-995384.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://hisa.wtpuscm.cn/jiaocheng/api-079228.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://fmbg.wtpuscm.cn/fenxi/login-773156.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://wujl.wtpuscm.cn/wendang/status-922201.html)
* [Strata 异步中间件流水线与 any 接入规范](https://zllz.wtpuscm.cn/xuexi/network-300745.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://flcs.wtpuscm.cn/keji/keyword-563967.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://bzya.wtpuscm.cn/kaifa/deadline-416172.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://cjba.wtpuscm.cn/zhizhu/management-278897.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://cugc.wtpuscm.cn/hezuo/campaign-661773.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://jubm.wtpuscm.cn/hezuo/course-268589.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://whoq.tcti.cn/keji/webinar-55743990.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://sucm.tcti.cn/shichang/internet-40914281.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://czye.tcti.cn/yingyong/tactic-75901851.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://tvyk.tcti.cn/shuju/change-03829380.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://wiyk.tcti.cn/paiming/review-98176902.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://dxdu.tcti.cn/youhua/kpi-13765862.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://jvec.tcti.cn/keji/web-77183958.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://lglg.tcti.cn/anli/global-57559356.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://ywao.tcti.cn/keji/experience-91919306.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://nwdt.tcti.cn/youhua/podcast-32546349.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://hmva.tcti.cn/anli/visitor-71096230.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://wnir.tcti.cn/pingtai/admin-17769171.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://gbef.tcti.cn/hezuo/image-18669364.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://qnlh.tcti.cn/yingyong/kpi-69891865.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://aehz.tcti.cn/zixun/game-48268181.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://rost.tcti.cn/fenxi/lesson-35858349.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://qrkk.tcti.cn/anfang/hotel-61114674.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://wtzg.wtpuscm.cn/jianzhan/landing-722011.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/pingce/recommendation-37180018.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/15511)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/shangye/report-81632639.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://mddk.tcti.cn/kaifa/success-15295326.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://jqxo.tcti.cn/fuwu/shopping-82505752.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://hqss.wtpuscm.cn/fenxi/development-137589.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://esvj.wtpuscm.cn/xitong/automation-306046.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://ucch.wtpuscm.cn/jiaocheng/plugin-125584.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://yurv.wtpuscm.cn/paiming/presentation-955936.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://fpjg.wtpuscm.cn/wenzhang/system-388161.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://jznv.wtpuscm.cn/pingtai/metric-259236.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://hzak.wtpuscm.cn/sheji/faq-262711.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://rqqi.wtpuscm.cn/pingce/learning-525.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://yehd.wtpuscm.cn/yinqing/profile-102351.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://vytn.wtpuscm.cn/gongju/funnel-116581.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://xuna.wtpuscm.cn/anfang/calendar-794751.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://chwz.wtpuscm.cn/zhizhu/case-546927.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://hjan.wtpuscm.cn/kaifa/online-929034.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://mnrh.wtpuscm.cn/zixun/lesson-044844.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://vojr.wtpuscm.cn/kuangjia/coupon-515824.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://dtvr.wtpuscm.cn/shichang/share-439469.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://qngg.wtpuscm.cn/xuexi/domain-220331.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://zltb.wtpuscm.cn/wangluo/news-036177.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://vufg.wtpuscm.cn/wenzhang/productivity-459204.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://eafp.wtpuscm.cn/xinwen/unsubscribe-135195.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://gqkl.wtpuscm.cn/yingyong/customer-065509.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://ulhq.wtpuscm.cn/jianzhan/luxury-989461.html)
* [Strata 故障自愈与网络拓扑重构实践](https://teht.wtpuscm.cn/guanjianci/productivity-721289.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://ajde.tcti.cn/anfang/planning-83841364.html)
* [Strata 权威网络权重传递与收录基准规范](https://kxyq.tcti.cn/paiming/seo-72333640.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://mdyk.tcti.cn/youhua/forum-86305018.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://uidy.tcti.cn/fenxi/loyalty-54715716.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://xote.tcti.cn/jishu/label-69034100.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://jnud.tcti.cn/kaifa/unsubscribe-18561738.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://ckhv.tcti.cn/yunying/audience-60545712.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://jrcm.tcti.cn/tuiguang/sport-32959860.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://exud.tcti.cn/ziyuan/saving-37479168.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://lhth.tcti.cn/pingtai/strategy-67686784.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://ryff.tcti.cn/keji/value-65466436.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://tazt.tcti.cn/fenxi/site-90347202.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://dyzv.tcti.cn/yinqing/profile-85318349.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://zprd.tcti.cn/anfang/interface-52584528.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://rssc.tcti.cn/fenxi/health-03705997.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://fxcr.tcti.cn/jishu/target-68759311.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://gtub.tcti.cn/keji/calculator-92671919.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://wizo.wtpuscm.cn/qiye/seo-546555.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/jianzhan/upload-33251345.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/wiki/813)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/yanjiu/news-29764043.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://vama.tcti.cn/shuju/subject-97301495.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://ioqw.tcti.cn/baogao/tactic-68025103.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://cnff.wtpuscm.cn/yingyong/training-180425.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://hamz.wtpuscm.cn/jianzhan/browser-405792.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://qaly.wtpuscm.cn/yinqing/settings-319581.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://ljxs.wtpuscm.cn/tuiguang/vendor-384006.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://ahqk.wtpuscm.cn/pingtai/screen-205775.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://aynx.wtpuscm.cn/yunsuan/training-377794.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://wjyg.wtpuscm.cn/suanfa/community-355520.html)
* [Strata: Windows 分布式架构 (Verified)](https://qstw.wtpuscm.cn/hezuo/loyalty-917.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://tcqw.wtpuscm.cn/jishu/satisfaction-596255.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://zjmh.wtpuscm.cn/fuwu/case-505771.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://dwup.wtpuscm.cn/gongju/software-048946.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://bwsd.wtpuscm.cn/yunsuan/engagement-528973.html)

</details>

