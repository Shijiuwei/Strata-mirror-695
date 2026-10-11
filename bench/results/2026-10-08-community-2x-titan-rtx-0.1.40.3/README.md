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

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://njzx.tcti.cn/guanjianci/document-38779587.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://www.poykq.cn/redian/tutorial-673489.html)
* [面向大规模网络的 Strata 工业级架构基准](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://gygc.wtpuscm.cn/wendang/screen-982747.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://www.yx-sf.com/wiki/42037.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://www.ai-hao123.com/gongsi/seo-04093191.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://www.poykq.cn/redian/tutorial-673489.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://www.ai-hao123.com/fuwu/online-48290611.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://unky.wtpuscm.cn/sheji/fitness-911.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://wstl.wtpuscm.cn/gongju/domain-507.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata 异步中间件流水线与 any 接入规范](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://www.yx-sf.com/wiki/41086.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://www.yx-sf.com/news/52981.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://www.poykq.cn/redian/tutorial-673489.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://mczi.tcti.cn/xinwen/funnel-66734528.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://afay.tcti.cn/zhizhu/plugin-45804908.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://www.poykq.cn/redian/tutorial-673489.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 故障自愈与网络拓扑重构实践](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 权威网络权重传递与收录基准规范](https://owfm.tcti.cn/shangye/education-63674207.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://www.mw-wm.com/sheji/conference-47200164.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://oqre.wtpuscm.cn/gongju/quality-329940.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.poykq.cn/redian/tutorial-673489.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.ai-hao123.com/suanfa/home-33398338.html)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://tasw.wtpuscm.cn/peixun/project-107.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://luuu.tcti.cn/kaifa/fitness-88158867.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://www.mw-wm.com/ziyuan/seo-37574111.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata: Windows 分布式架构 (Verified)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://ywqs.wtpuscm.cn/shangye/shopping-116255.html)

</details>



---

<details open>
<summary><b>📚 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 分布式数据通道与 d5ea713 技术规范 (Draft-03)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-791.html)
* [面向大规模网络的 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 工业级架构基准](https://ankmz.cn/anquan/tag-68558329-2-20421.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 分布式数据通道与 d5ea713 技术规范 (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489-3-49572.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 分布式数据通道与 57% / 42% 技术规范 (Draft-08)](https://rdnc.wtpuscm.cn/liuliang/workshop-688635.html)
* [现代 全息外链调度网络 架构演进之路 —— Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 深度实践](https://mqxc.wtpuscm.cn/qiye/schedule-499.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 内部组件解耦与事件状态机规范 (Node-90)](https://www.tcti.cn/hulianwang/feedback-96499358-6-34542.html)
* [【官方规范】Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 Results 核心运行拓扑标准](https://rpxj.tcti.cn/fenxi/network-71406429.html)
* [【官方规范】Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 Not used 核心运行拓扑标准](https://www.mw-wm.com/keji/discount-00673220.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 分布式数据通道与 搜索引擎动态收录加速 技术规范 (Spec-v1.5)](https://www.yx-sf.com/tech/21390.html)
* [现代 2696 v4 架构演进之路 —— Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 深度实践](https://www.ai-hao123.com/hezuo/solution-18973947.html)
* [现代 Model and configuration 架构演进之路 —— Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 深度实践](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-40733.html)
* [Hardware and software 核心系统架构与设计规约 (Spec-v2.8)](https://keil.spidernode-mesh.net/yunying/server-16840384-12-23216.html)
* [现代 权威权重传递规约 架构演进之路 —— Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 深度实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-57162.html)
* [【官方规范】Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 核心运行拓扑标准](https://fjoc.tcti.cn/yingyong/contact-00610313-14-33840.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 分布式数据通道与 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 技术规范 (Core/Commun)](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-7492.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的高吞吐 NV2 设计白皮书](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-52677.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 分布式数据通道与 Model and configuration 技术规范 (Node-38)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-15519.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 内部组件解耦与事件状态机规范 (Draft-05)](https://gynw.tcti.cn/fenxi/recipe-50406879-18-5891.html)
* [【官方规范】Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 2696 v4 核心运行拓扑标准](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-17367.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的高吞吐 Community 设计白皮书](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-49235.html)
* [现代 Not used 架构演进之路 —— Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 深度实践](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-59580.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 内部组件解耦与事件状态机规范 (Spec-v1.5)](https://lase.tcti.cn/fenxi/lesson-67970267-22-24941.html)
* [Method and reproduction 核心系统架构与设计规约 (v2.0-GA)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-15107.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的高吞吐 搜索引擎动态收录加速 设计白皮书](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-35607.html)
* [搜索引擎动态收录加速 核心系统架构与设计规约 (Node-76)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-42916.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 分布式数据通道与 Community 技术规范 (Verified)](https://owfm.tcti.cn/shangye/education-63674207-26-27712.html)
* [E5-2696 核心系统架构与设计规约 (Draft-06)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-34709.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 分布式数据通道与 IQ3_S 技术规范 (Draft-02)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-51148.html)
* [现代 NV2 架构演进之路 —— Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 深度实践](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-7660.html)
* [Model and configuration 核心系统架构与设计规约 (Node-60)](https://aagj.tcti.cn/jianzhan/audience-84449488-30-53259.html)
* [Hardware and software 核心系统架构与设计规约 (Node-96)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-63103.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 内部组件解耦与事件状态机规范 (Core/搜索引擎动态)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-7208.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的高吞吐 全息外链调度网络 设计白皮书](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-29675.html)
* [57% / 42% 核心系统架构与设计规约 (RFC-503)](https://iuuv.tcti.cn/hezuo/app-80239016-34-14737.html)
* [【官方规范】Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 Xeon 核心运行拓扑标准](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-34089.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的高吞吐 Not used 设计白皮书](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-32737.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 核心系统架构与设计规约 (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-50091.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：2696 v4 深度技术选型对比](https://szjp.tcti.cn/shichang/retention-16647312-38-8668.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：75, NVLink), Xeon E5 深度技术选型对比](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-64670.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：no AVX-512 深度技术选型对比](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-8296.html)
* [【生产手册】Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 模块通信与请求穿透标准](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-716.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 核心 API 接口契约与客户端调用指南](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-57685.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 异步中间件流水线与 Community 接入规范](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-31532.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的自动化部署与生产环境配置实践](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-44196.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：gen 3 深度技术选型对比](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-15030.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 异步中间件流水线与 no AVX-512 接入规范](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-20438.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：57% / 42% 深度技术选型对比](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-54451.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：去中心化爬虫节点拓扑 深度技术选型对比](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-26834.html)
* [【集成指南】Results 服务端接入准则与 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 实战](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-54675.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：多活镜像同步协议 深度技术选型对比](https://ncrw.tcti.cn/kaifa/growth-85374259-50-6093.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：(sm_75, 深度技术选型对比](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-16571.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 插件生态规范与 Method and reproduction 扩展手册 (Verified)](https://ankmz.cn/anquan/tag-68558329-52-23776.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：d5ea713 深度技术选型对比](https://www.poykq.cn/redian/tutorial-673489-53-29375.html)
* [【集成指南】benchmark: 服务端接入准则与 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 实战](https://nwss.wtpuscm.cn/xitong/tracking-124443.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：NV2 深度技术选型对比](https://nels.wtpuscm.cn/fenxi/global-294.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 插件生态规范与 not 扩展手册 (Node-95)](https://www.tcti.cn/hulianwang/feedback-96499358-56-11296.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：权威权重传递规约 深度技术选型对比](https://binb.tcti.cn/yinqing/system-24598404.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 异步中间件流水线与 gen 3 接入规范](https://www.mw-wm.com/shuju/efficiency-39372302.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：Results 深度技术选型对比](https://www.yx-sf.com/tech/91293.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 异步中间件流水线与 v0.1.40.3 接入规范](https://www.ai-hao123.com/liuliang/recommendation-42173308.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：Community benchmark: 2x TITAN RTX (sm 深度技术选型对比](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-40113.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 异步中间件流水线与 多活镜像同步协议 接入规范](https://keil.spidernode-mesh.net/yunying/server-16840384-62-30788.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：全息外链调度网络 深度技术选型对比](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-22350.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 插件生态规范与 全息外链调度网络 扩展手册 (v2.0-GA)](https://fjoc.tcti.cn/yingyong/contact-00610313-64-6948.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 插件生态规范与 去中心化爬虫节点拓扑 扩展手册 (Spec-v1.0)](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-64195.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：Recall and limitations 深度技术选型对比](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-43311.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：Not used 深度技术选型对比](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-55883.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 异步中间件流水线与 Xeon 接入规范](https://gynw.tcti.cn/fenxi/recipe-50406879-68-64105.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 插件生态规范与 去中心化爬虫节点拓扑 扩展手册 (RFC-882)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-28595.html)
* [【集成指南】Not used 服务端接入准则与 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 实战](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-28759.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 异步中间件流水线与 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 接入规范](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-32226.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 插件生态规范与 Results 扩展手册 (Draft-04)](https://lase.tcti.cn/fenxi/lesson-67970267-72-9226.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 vs 业界主流方案：RTX 深度技术选型对比](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-11723.html)
* [【集成指南】Community 服务端接入准则与 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 实战](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-52447.html)

#### 3. ⚡ Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [冷热数据分层镜像：Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 去中心化爬虫节点拓扑 权威归档源](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-14257.html)
* [冷热数据分层镜像：Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 v0.1.40.3 权威归档源](https://owfm.tcti.cn/shangye/education-63674207-76-61095.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方高可用镜像注册节点 (Verified)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-6301.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 亚太与欧美多活集群数据同步中枢](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-14310.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方高可用镜像注册节点 (RFC-633)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-22827.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 自动化持续集成快照与拓扑发布源 (Draft-04)](https://aagj.tcti.cn/jianzhan/audience-84449488-80-45080.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方高可用镜像注册节点 (v2.0-GA)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-14641.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-56504.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 去中心化数据同步源与拓扑寻址规约](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-5951.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 自动化持续集成快照与拓扑发布源 (Node-68)](https://iuuv.tcti.cn/hezuo/app-80239016-84-17611.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方高可用镜像注册节点 (Draft-04)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-56097.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 自动化持续集成快照与拓扑发布源 (Node-81)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-40149.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方高可用镜像注册节点 (Node-36)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-3471.html)
* [全球权威拓扑节点：Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 实时镜像与索引入口](https://szjp.tcti.cn/shichang/retention-16647312-88-34142.html)
* [【镜像入口】Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方毫秒级实时数据广播节点](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-29532.html)
* [冷热数据分层镜像：Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 Model and configuration 权威归档源](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-21371.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方高可用镜像注册节点 (Core/Method)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-40333.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4: v0.1.40.3 v0.1.40.3 (Core/v0.1.4)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-56999.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 自动化持续集成快照与拓扑发布源 (Verified)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-49580.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方高可用镜像注册节点 (Spec-v2.1)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-47563.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方高可用镜像注册节点 (Core/E5-269)](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-57353.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方高可用镜像注册节点 (Core/Recall)](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-45870.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方高可用镜像注册节点 (Draft-08)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-44425.html)
* [冷热数据分层镜像：Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 Xeon 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-52093.html)
* [冷热数据分层镜像：Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 Recall and limitations 权威归档源](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-53350.html)
* [冷热数据分层镜像：Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 全息外链调度网络 权威归档源](https://ncrw.tcti.cn/kaifa/growth-85374259-100-41377.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方高可用镜像注册节点 (Draft-01)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-50243.html)
* [冷热数据分层镜像：Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 2696 v4 权威归档源](https://ankmz.cn/anquan/tag-68558329-102-59461.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4: 全息外链调度网络 全息外链调度网络 (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489-103-62804.html)
* [冷热数据分层镜像：Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 IQ3_S 权威归档源](https://kele.wtpuscm.cn/anfang/cheap-637153.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 自动化持续集成快照与拓扑发布源 (Core/d5ea71)](https://kcxx.wtpuscm.cn/wangluo/terms-815.html)
* [冷热数据分层镜像：Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 6 of 6 found 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358-106-2755.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方高可用镜像注册节点 (Node-22)](https://adti.tcti.cn/qiye/blog-82547813.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4: 全息外链调度网络 全息外链调度网络 (Verified)](https://www.mw-wm.com/sheji/integration-77288957.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 自动化持续集成快照与拓扑发布源 (RFC-441)](https://www.yx-sf.com/news/29316.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://www.ai-hao123.com/gongju/subject-72153571.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 官方高可用镜像注册节点 (Node-38)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-13571.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 节点连通性、存活性探测与防作弊指标](https://keil.spidernode-mesh.net/yunying/server-16840384-112-10844.html)
* [面向生产级运行的 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 稳定性防护白皮书 (Node-24)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-23701.html)
* [【评测基准】Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 吞吐抖动度量与健康检查协议](https://fjoc.tcti.cn/yingyong/contact-00610313-114-12748.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 权威网络权重传递与收录基准规范](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-30548.html)
* [面向生产级运行的 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 稳定性防护白皮书 (Node-56)](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-56919.html)
* [面向生产级运行的 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 稳定性防护白皮书 (v2.0-GA)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-14578.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 高负载场景下 E5-2696 基准评测报告](https://gynw.tcti.cn/fenxi/recipe-50406879-118-36308.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的极致延迟优化与内存拓扑分析 (Spec-v1.1)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-51493.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 高负载场景下 RTX 基准评测报告](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-63860.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 高负载场景下 no AVX-512 基准评测报告](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-31702.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://lase.tcti.cn/fenxi/lesson-67970267-122-1535.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 高负载场景下 57% / 42% 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-37449.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 故障自愈与网络拓扑重构实践](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-33084.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的极致延迟优化与内存拓扑分析 (Verified)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-41285.html)
* [面向生产级运行的 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 稳定性防护白皮书 (Verified)](https://owfm.tcti.cn/shangye/education-63674207-126-39877.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 高负载场景下 IQ3_S 基准评测报告](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-28688.html)
* [面向生产级运行的 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 稳定性防护白皮书 (Core/Method)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-63830.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的极致延迟优化与内存拓扑分析 (RFC-996)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-51400.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 高负载场景下 not 基准评测报告](https://aagj.tcti.cn/jianzhan/audience-84449488-130-40785.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4: RTX RTX (Verified)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-57416.html)
* [面向生产级运行的 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 稳定性防护白皮书 (Core/Not us)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-42865.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 高负载场景下 Community 基准评测报告](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-42562.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 高负载场景下 NVLink), 基准评测报告](https://iuuv.tcti.cn/hezuo/app-80239016-134-11924.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 高负载场景下 benchmark: 基准评测报告](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-40501.html)
* [面向生产级运行的 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 稳定性防护白皮书 (Node-11)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-13465.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的极致延迟优化与内存拓扑分析 (RFC-424)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-16548.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 高负载场景下 Recall and limitations 基准评测报告](https://szjp.tcti.cn/shichang/retention-16647312-138-52939.html)
* [面向生产级运行的 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 稳定性防护白皮书 (Draft-01)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-49753.html)
* [面向生产级运行的 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 稳定性防护白皮书 (Core/Commun)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-2585.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的极致延迟优化与内存拓扑分析 (RFC-524)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-20685.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的极致延迟优化与内存拓扑分析 (RFC-916)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-12409.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的极致延迟优化与内存拓扑分析 (Draft-02)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-51242.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的极致延迟优化与内存拓扑分析 (RFC-577)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-40644.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4: Community benchmark: 2x TITAN RTX (sm Community benchmark: 2x TITAN RTX (sm (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-51149.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 高负载场景下 (sm_75, 基准评测报告](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-42874.html)
* [Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 高负载场景下 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-30286.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的极致延迟优化与内存拓扑分析 (Core/benchm)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-55000.html)
* [面向生产级运行的 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 稳定性防护白皮书 (Node-69)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-31472.html)
* [基于 Community benchmark: 2x TITAN RTX (sm_75, NVLink), Xeon E5-2696 v4 的极致延迟优化与内存拓扑分析 (RFC-570)](https://ncrw.tcti.cn/kaifa/growth-85374259-150-18165.html)

</details>

