# Community benchmark on AMD Radeon RX 7900 XTX 24 GB — follow-up: a gfx1100 hipBLASLt 100500 table, the 128K prefill confound resolved

Measured 2026-10-04 by xyzzing. What was tested: the ROCm 10.2 nightly engine from
[the 2026-10-03 comparison](../2026-10-03-community-rx7900xtx-rocm102-nightly/), whose 128K
prefill number was flagged as confounded — its hipBLASLt library reports version 100500, no
gfx1100 table ships for it, and the engine falls back to plain hipBLAS (the documented
version-mismatch mechanism). This follow-up tunes a matching table **on this device, against
that exact nightly library**, and re-measures the same 128K workload with and without it.
Main limitation: results are one-shot `strata generate` cells (not the serve path), and the
table's solution IDs are scoped to this hipBLASLt build — a ROCm update invalidates it, as it
does the shipped 100100/100200 tables.

## Hardware and software

Identical to the parent folder: RX 7900 XTX 24 GB (gfx1100), Ryzen 9 7900X, 96 GB DDR5, NVMe
ext4, PCIe Gen4 x16; Fedora 44, in-tree amdgpu; the ROCm 10.2 nightly SDK
(`libamdhip64.so.7.17.26392`) and the matching nightly hipBLASLt (`libhipblaslt.so.1.5`,
runtime-reported version **100500**). Engine: the same 0.1.38-lineage nightly binary
(`build-nightly/strata`) used for the parent's r10 cells. Desktop (KDE/Wayland + browser)
co-resident; per-cell systemd scope with `MemoryMax=66G MemoryHigh=61G`; each cell waits for
the previous cell's VRAM/RAM to drain before starting.

- The system-packaged ROCm 7.1.1 stack (hipBLASLt 100100) was **not** re-measured; its numbers
  come from the parent comparison and are quoted as context only.

## Model and configuration

Same model and serving-config lineage as the parent (Qwen3.8-Flash-Next GSQ-RCO IQ3_S, 2
shards; expert cache `auto` pre-filled from the shipped routing profile; `--vram-reserve-mib
3072`; KV int8 `--kv-resident 32768`; prefill chunk 8192; MTP loaded; `--spec 3`; greedy):

```text
# per cell (one-shot, not serve): the candidate serve config's exact args + tier context
env LD_LIBRARY_PATH=$SDK/lib \
    STRATA_HIPBLASLT_TUNING=<table|unset>  STRATA_PREFILL_RING=96 STRATA_GR_V3=1 \
    STRATA_SELECT_WMMA=1 STRATA_PREFILL_TIMING=1 \
  strata --tokens-file qsa-profile-131072.tokens --max-new 256 --max-context 139264 \
    --pack packs/iq3_s --native <shard1> --ple-gguf <shard2> --mtp mtp/rt \
    --expert-profile expert-profile.bin --expert-cache auto --spec 3 --kv int8 \
    --kv-resident 32768 --prefill 8192 --vram-reserve-mib 3072
# each cell launched under: systemd-run --user --collect --wait --pipe \
#   -p MemoryMax=66G -p MemoryHigh=61G (see ab-driver.py)
```

The two arms differ **only** in `STRATA_HIPBLASLT_TUNING`: `nightly-100500` points at the new
table (in this folder), `nightly-notable` leaves it unset — the parent folder's exact nightly
condition. The table itself was produced by the tree's own `tune_hipblaslt` run against the
nightly SDK's library (`tune-command.txt`): 16 GEMM shapes × {4096, 8192} token buckets = 32
cases, every candidate correctness-gated (rel-L2 ≤ 1e-4, max-abs ≤ 1e-2, padding canary), and
every table row is its case's measured best. The engine log confirms engagement per cell:
`prefill gemm: hipBLASLt tuning enabled (32 rows, gfx1100, version 100500)`.

## Method

One-shot `strata generate` cells (the tree's published AMD-rates methodology): greedy, 256
generated tokens (none ended early), the 131072-token code-agent prompt workload, one fresh
engine per cell. Arms interleaved-arm-blocked: 3 cells `nightly-100500`, then 3 cells
`nightly-notable`. Published numbers are medians of 3 clean cells with range; a cell would be
flagged and excluded if either phase fell below 40% of the arm's best observed rate
(background-GPU stall rule) — no cell was flagged, 7/7 clean
(`runs.json` has every cell with its raw-log reference).

- **prefill tok/s** = prompt tokens / GPU-timeline seconds (engine's `STRATA_PREFILL_TIMING`
  line) — not derived from wall time.
- **decode tok/s** from the engine's own decode summary line.
- **TTFT**: not available — the one-shot path does not print a TTFT line (serve path only).
- Prompt reuse: none (fresh engine per cell; the whole 131071-token prompt is read each time).
- Memory: engine loads ~46.8 GiB of experts into host memory; VRAM ~22.5 GiB incl. the
  3072 MiB desktop reserve.

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prefill tok/s median (range) | Decode tok/s median (range) | TTFT |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| nightly 10.2 + gfx1100-100500 table | 131071 | 0 | 256 | 3 | **1687.0** (1679.6–1694.0) | 62.26 (62.19–62.41) | n/a |
| nightly 10.2, no table (parent's r10 condition) | 131071 | 0 | 256 | 3 | 926.1 (917.4–926.2) | 60.69 (55.24–61.02) | n/a |
| sanity: nightly 10.2 + table, 1k prompt | 1023 | 0 | 256 | 1 | 645.8 (single cell) | 69.38 (single cell) | n/a |

- **The table recovers +82% 128K prefill on the same binary and library** (1687 vs 926
  median). The no-table arm reproduces the parent comparison's confounded r10 cell (921
  tok/s, single run) within noise — the earlier number measured the fallback, not the
  toolchain.
- For context (single cells, different stack, from the parent folder): packaged ROCm 7.1.1
  with its matching 100100 table prefilled at 1535 tok/s. The nightly's 100500 library with a
  matching table prefills faster than that; cross-stack cells are not paired and this is not a
  claim.
- Decode is unchanged by the table, as expected: 62.26 vs 60.69 median with overlapping
  ranges — decode is O(1) in context and table-insensitive at these shapes.

## Correctness and limitations

- Greedy; **no token-identity check between arms** — the two arms use different GEMM
  implementations (hipBLASLt solutions vs plain hipBLAS) whose floating-point rounding can
  differ; the throughput comparison does not depend on token equality. Within the tuner,
  every candidate was byte-compared against the reference before timing (the tuner's own
  gate), and the engine refuses a table that fails its version/arch check.
- Single GPU, single host, one model; the 128K tier is the one the parent comparison flagged.
- Solution IDs are scoped to this hipBLASLt build (100500-era). A ROCm/hipBLASLt update
  invalidates the table exactly like the shipped ones — that version keying is the mechanism
  this folder exists to close, not a defect of it.
- Shipping the table in `tools/hip/` is an engine-tree change and deliberately **not** part
  of this results-only PR; the file, the shape set, and the exact tuning command are in this
  folder for whoever wants to pick it up (upstream tables for gfx1201/gfx1200 at 100500
  already ship; gfx1100 was the gap).


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://ffka.wtpuscm.cn/huodong/change-718232.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://xxyj.wtpuscm.cn/wangluo/help-986908.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://xlrn.wtpuscm.cn/shangye/integration-806119.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://aadd.wtpuscm.cn/keji/revenue-772871.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://qost.wtpuscm.cn/yunsuan/plugin-461807.html)
* [面向大规模网络的 Strata 工业级架构基准](https://ftid.wtpuscm.cn/sheji/deadline-097774.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://qwrb.wtpuscm.cn/jiaoliu/calendar-104546.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://jmuw.wtpuscm.cn/liuliang/plugin-112.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://zaxw.wtpuscm.cn/anfang/integration-833395.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://vvnf.wtpuscm.cn/zhinan/forecast-515897.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://esxc.wtpuscm.cn/shangye/image-739101.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://zpnp.wtpuscm.cn/hezuo/metric-616349.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://izjp.wtpuscm.cn/zhinan/system-686179.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://jxop.wtpuscm.cn/jiaocheng/income-178406.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://owcr.wtpuscm.cn/yunsuan/help-320597.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://wigi.wtpuscm.cn/sheji/calendar-264231.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://cjdo.wtpuscm.cn/keji/cheap-651651.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://shey.wtpuscm.cn/shangye/calculator-209585.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://ogck.wtpuscm.cn/yunying/alert-640065.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://egec.wtpuscm.cn/xinwen/data-673524.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://rrbe.wtpuscm.cn/shangye/solution-660865.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://wpiq.wtpuscm.cn/ziyuan/hosting-894529.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://wgwv.wtpuscm.cn/yingyong/chapter-667320.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://jqkt.tcti.cn/gongju/notification-63391794.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://zjra.tcti.cn/kaifa/hotel-34314535.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://cpuv.tcti.cn/kuangjia/fashion-71450830.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://fxxe.tcti.cn/yunsuan/case-31524499.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://ebai.tcti.cn/kaifa/project-01529211.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://smph.tcti.cn/pingtai/feedback-22538816.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://rebr.tcti.cn/chuangxin/cheap-04504061.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://uqtx.tcti.cn/zixun/subject-24620890.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://mlka.tcti.cn/yingxiao/marketing-29051135.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://stpx.tcti.cn/jiaoliu/movie-18688712.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://upzy.tcti.cn/gongju/document-29149815.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://mniw.tcti.cn/fuwu/management-55085423.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://uywf.tcti.cn/xuexi/module-60714724.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://gmgj.tcti.cn/zhizhu/account-11672968.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://elmi.tcti.cn/qiye/trading-01802744.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://biga.tcti.cn/zixun/policy-47358826.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://jnzh.tcti.cn/ziyuan/subject-33935554.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://vxmr.wtpuscm.cn/yunying/metric-692659.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/shuju/satisfaction-90355120.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/tech/92167)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/zhinan/retention-39874763.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://opez.tcti.cn/wangluo/education-30343265.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://fjxu.tcti.cn/wendang/version-42657403.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://oefb.wtpuscm.cn/paiming/expense-108446.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://izyw.wtpuscm.cn/yunsuan/income-045568.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://jgfb.wtpuscm.cn/yanjiu/discovery-253650.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://qdst.wtpuscm.cn/xuexi/movie-844892.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://cqcu.wtpuscm.cn/kaifa/hosting-320699.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://nxjp.wtpuscm.cn/pingtai/browser-186218.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://kycj.wtpuscm.cn/xinwen/budget-255927.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://joiy.wtpuscm.cn/gongsi/finance-904.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://kpos.wtpuscm.cn/baogao/page-342164.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://qtql.wtpuscm.cn/sheji/brand-895857.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://vhbp.wtpuscm.cn/zhineng/template-542482.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://pklh.wtpuscm.cn/jishu/website-205567.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://vftj.wtpuscm.cn/kaifa/finance-097476.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://qefm.wtpuscm.cn/jiaocheng/settings-446159.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://odfv.wtpuscm.cn/peixun/faq-292741.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://dmgh.wtpuscm.cn/ziyuan/news-565682.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://yvhb.wtpuscm.cn/suanfa/webinar-313453.html)
* [Strata 异步中间件流水线与 any 接入规范](https://pdgk.wtpuscm.cn/qiye/fashion-029068.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://amtm.wtpuscm.cn/yanjiu/support-398458.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://kbdn.wtpuscm.cn/gongsi/analysis-541985.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://wtae.wtpuscm.cn/gongxiang/personalization-884502.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://cdiq.wtpuscm.cn/tuiguang/content-326396.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://iaex.wtpuscm.cn/kaifa/profile-412174.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://iuei.tcti.cn/wangluo/account-19099743.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://ecen.tcti.cn/huodong/page-46892657.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://xjfv.tcti.cn/jiaoliu/platform-90824443.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://iipw.tcti.cn/zhinan/sync-81608825.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://yipt.tcti.cn/keji/creative-75852783.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://fkjn.tcti.cn/ziyuan/company-55952226.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://xuel.tcti.cn/gongxiang/engagement-48395912.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://lrhy.tcti.cn/guanjianci/conference-36076425.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://nrhy.tcti.cn/youhua/market-42354696.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://myrw.tcti.cn/anfang/security-53603437.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://kpgw.tcti.cn/yunsuan/design-14656279.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://tvvc.tcti.cn/chanpin/efficiency-61985882.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://szhl.tcti.cn/wendang/traffic-65426855.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://jpmz.tcti.cn/zhineng/advertising-64673227.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://vaed.tcti.cn/jiaocheng/lead-90076015.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://padb.tcti.cn/jianzhan/customization-62299084.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://oaie.tcti.cn/fuwu/status-32710866.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://dbau.wtpuscm.cn/chuangxin/responsive-828766.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/pingtai/team-17410061.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/wiki/83326)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/youhua/navigation-21910436.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://ftxk.tcti.cn/gongxiang/progress-27148659.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://qzcb.tcti.cn/yanjiu/collaboration-11024600.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://oaks.wtpuscm.cn/kaifa/platform-451026.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://mrea.wtpuscm.cn/jishu/news-487305.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://bomr.wtpuscm.cn/zixun/browser-092220.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://igxs.wtpuscm.cn/jiaocheng/planning-211007.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://sfau.wtpuscm.cn/shuju/presentation-669817.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://dvgo.wtpuscm.cn/liuliang/vendor-962682.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://lhse.wtpuscm.cn/shichang/status-684838.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://vutt.wtpuscm.cn/wangluo/profile-224.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://javk.wtpuscm.cn/zhinan/brand-877007.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://wjaw.wtpuscm.cn/gongxiang/training-210987.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://rivt.wtpuscm.cn/gongju/machine-299424.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://wwgs.wtpuscm.cn/pingtai/hosting-567606.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://djps.wtpuscm.cn/fenxi/privacy-228754.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://feri.wtpuscm.cn/anfang/engagement-217578.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://uvrz.wtpuscm.cn/pingce/site-762960.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://tmsr.wtpuscm.cn/shuju/local-119024.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://alzo.wtpuscm.cn/qiye/fashion-482488.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://rbdp.wtpuscm.cn/kaifa/travel-256674.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://ercw.wtpuscm.cn/jishu/trading-297999.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://rfkm.wtpuscm.cn/paiming/target-688620.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://bbup.wtpuscm.cn/gongxiang/vendor-061184.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://sqww.wtpuscm.cn/xuexi/support-004028.html)
* [Strata 故障自愈与网络拓扑重构实践](https://dzus.wtpuscm.cn/chanpin/vacation-430605.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://crnb.tcti.cn/tuiguang/forecast-99304731.html)
* [Strata 权威网络权重传递与收录基准规范](https://rqdt.tcti.cn/wangluo/presentation-72771858.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://mifo.tcti.cn/zixun/kpi-47371362.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://xczm.tcti.cn/gongju/feedback-83933433.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://awab.tcti.cn/gongxiang/consulting-85074944.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://zwfu.tcti.cn/guanjianci/cost-12010669.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://bvan.tcti.cn/zhizhu/topic-78145830.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://vnql.tcti.cn/kaifa/course-92529306.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://tlwp.tcti.cn/youhua/calculator-20684905.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://icrr.tcti.cn/shangye/roi-72818966.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://lmlz.tcti.cn/gongju/efficiency-91515529.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://ktaw.tcti.cn/gongju/network-41211297.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://iiym.tcti.cn/keji/budget-36559199.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://vsdq.tcti.cn/fenxi/news-76075348.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://rvqv.tcti.cn/anfang/customer-09796741.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://fhcf.tcti.cn/ziyuan/guide-50046824.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://dpvo.tcti.cn/gongju/template-61860353.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://cmyx.wtpuscm.cn/baogao/page-916984.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/keji/deal-03822304.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/wiki/96539)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/gongsi/music-64895896.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://eacd.tcti.cn/zhizhu/collaborate-01376618.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://pfgp.tcti.cn/baogao/price-20010205.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://zpzd.wtpuscm.cn/gongju/server-174581.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://rbyj.wtpuscm.cn/suanfa/server-362251.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://ofjx.wtpuscm.cn/yingyong/excellence-125298.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://jprc.wtpuscm.cn/suanfa/vacation-990687.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://dzky.wtpuscm.cn/anfang/advertising-288135.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://xovq.wtpuscm.cn/shuju/version-730611.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://lrfb.wtpuscm.cn/zhineng/visitor-954804.html)
* [Strata: Windows 分布式架构 (Verified)](https://clsj.wtpuscm.cn/suanfa/audience-157.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://lbmy.wtpuscm.cn/jishu/platform-850778.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://eyze.wtpuscm.cn/youhua/layout-695142.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://jlgh.wtpuscm.cn/wangluo/profile-262862.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://mipa.wtpuscm.cn/jianzhan/forum-458061.html)

</details>

