# Arc Pro B65: Strata v0.1.40.2 on PCIe Gen4

Measured on 2026-10-07 by timnevits. **Official upstream release**, commit
`e8ca9afd03d839d4f8dbbe82dffce7f8a3bafd7a`, with **no local engine patches**.
The measurements come from my B65 hardware.

Both memory profiles pass their bounded correctness checks. With the same 7K inputs, the 8K/prefill-512 profile delivers **43.49 decode tok/s**, versus **42.38** with the 262K/prefill-512 profile (-2.6%). The 8K/prefill-4096 profile reduces median 7K TTFT from **22.23 to 8.47 seconds**, with decode tradeoffs below.

## Hardware and configuration

One Intel Arc Pro B65, 32 GiB VRAM (`8086:e222`), **Gen4 x16**, 200 W cap;
i5-12600K, 128 GB DDR4-3200, Samsung 980 NVMe. Other model owners were stopped.
Ubuntu 26.04.1, kernel 7.0.0-38, xe/NEO 26.22.38646.7, Level Zero 1.28.6.
Native Release/JIT build with existing oneAPI 2026.1.0 and oneMKL, precise FP,
correctly rounded divide/sqrt and 32-lane subgroups. The tagged engine labels
itself `0.1.40-sycl`; the commit and binary hash identify this build.
Exact metadata: [system.json](system.json), [build.json](build.json).

Original **full 512-expert Flash-Next IQ2_XS**, not Coder:
`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF` at
`ed59f92082b1e93c0e96d60a8b11aab089b52f09`. Original Q2_0 MTP pack from
`Qwen/Qwen3.8-Flash-Next@de4b8e4d43b917e7706784d8bb445c9af86a3540`.
[artifacts.json](artifacts.json) identifies both shards, retained dense pack,
tokenizer/template, draft and expert ranking. Ranking matches the release
byte for byte. The model has a native 262144-token window. The tables compare **8192 and
262152 native context allocations (8K/262K profiles)**, using matched short and 7K inputs. The API reserves eight verification slots,
leaving 8184 and 262144 usable combined tokens respectively. No model, quantization, driver, compiler or ranking changes
between configurations.

One request, INT8 resident KV, four workers, automatic hot
expert cache and streaming experts. Every missing expert is in a protected
pinned host mirror (16 GiB cap). `STRATA_VERIFY_NO_HOST=1` keeps computation
on the GPU, including direct reads from that mirror. Native verify/commit
graphs, MTP window 4/max 4/min-p 0.5 (up to three drafts). Thinking off,
greedy/seed 17; prompt, conversation and suffix caches, short-read and expert
adaptation off. `STRATA_DBG_NAN` and QFUSE absent. Allocation alias checking
keeps its upstream default. The two 8K profiles reserve 3072 MiB and differ only in prefill batch size:
[512](profile-8k-512.json), [4096](profile-8k-4096.json). The
[262K profile](profile-262k-512.json) uses prefill 512, reserve 4096 MiB and
explicit `--no-prefill-borrow`, preserving the smaller window's implicit
no-borrow policy. It reserves more KV and has fewer hot expert slots; this is
a comparison of the complete memory profiles, not context as a single variable.
[environment.json](environment.json) is shared.

Actual startup allocations below are identical across each profile's three main
runs. Expert slots count per-layer experts, rather than the model's 512 experts
per MoE layer. They help explain the memory/performance tradeoff.

| Profile | Hot expert slots | Hot cache MiB | Pinned missing-expert mirror GiB | Free VRAM at startup MiB |
| --- | ---: | ---: | ---: | ---: |
| 8K / 512 | 17881 | 24588 | 9.01 | 3197 |
| 8K / 4096 | 16160 | 22207 | 11.33 | 4237 |
| 262K / 512 | 14388 | 19761 | 13.72 | 4185 |

## Three-run results

Main suite: **three fresh processes per profile**, five fixed tasks at each
of 512 and 7000 input tokens, 640 generated tokens. An unmeasured full-shape
warmup precedes each input tier. Tasks cover code, prose, inventory and incident
planning; padding precedes the instruction. Each profile has 48 total requests,
30 measured. For each table row, take the median of five tasks within each run,
then the median and range of those three run medians. Individual cells are
available in the raw data.

Public-fixture suite: three fresh processes per profile, 2185-token request
first, then 20-token request, 256 outputs each, followed by an isolation canary.
No separate warmup; any first-request cost stays in its measurement. This is
9 total requests/6 measured per profile. Its prompts and output lengths differ
from the main suite.

| Native context allocation | Prefill | Input/workload | Output | Runs | Prompt tok/s | Decode tok/s | TTFT seconds |
| ---: | ---: | --- | ---: | ---: | --- | --- | --- |
| 8192 | 512 | 512, five tasks | 640 | 3 | 352.37 (352.28–352.81) | 45.01 (44.99–45.02) | 1.486 (1.485–1.487) |
| 8192 | 512 | 7000, five tasks | 640 | 3 | 315.44 (315.31–315.46) | 43.49 (43.49–43.49) | 22.226 (22.225–22.236) |
| 8192 | 512 | 20, public fixture | 256 | 3 | 54.60 (54.56–54.61) | 54.48 (54.47–54.49) | 0.400 (0.400–0.401) |
| 8192 | 512 | 2185, public fixture | 256 | 3 | 323.36 (323.30–323.43) | 49.72 (49.72–49.72) | 6.817 (6.816–6.818) |
| 8192 | 4096 | 512, five tasks | 640 | 3 | 330.92 (330.88–331.11) | 44.64 (44.61–44.65) | 1.581 (1.580–1.581) |
| 8192 | 4096 | 7000, five tasks | 640 | 3 | 830.00 (829.93–830.03) | 42.92 (42.92–42.92) | 8.469 (8.469–8.469) |
| 8192 | 4096 | 20, public fixture | 256 | 3 | 50.13 (50.11–50.19) | 53.51 (53.50–53.51) | 0.434 (0.433–0.434) |
| 8192 | 4096 | 2185, public fixture | 256 | 3 | 722.72 (582.42–723.77) | 47.40 (47.40–47.43) | 3.085 (3.081–3.813) |
| 262152 | 512 | 512, five tasks | 640 | 3 | 320.56 (318.13–320.72) | 43.51 (42.96–43.51) | 1.631 (1.631–1.644) |
| 262152 | 512 | 7000, five tasks | 640 | 3 | 287.80 (285.05–287.80) | 42.38 (41.87–42.40) | 24.357 (24.357–24.593) |
| 262152 | 512 | 20, public fixture | 256 | 3 | 49.02 (48.83–49.14) | 51.53 (50.52–51.53) | 0.444 (0.443–0.446) |
| 262152 | 512 | 2185, public fixture | 256 | 3 | 302.43 (300.92–302.91) | 48.95 (48.52–48.95) | 7.285 (7.281–7.331) |

Values are median (range). Loading/startup is timed separately and excluded
from request timings; the OS weight-file cache stays warm. Fixed expert
placement, **zero prompt reuse**, no adaptation. Prompt/decode rates use the
engine's separate token counts and durations. TTFT measures request start to
first native emitted token. Raw native timing lines, counts, draft acceptance,
input/output hashes and elapsed times: [results.json](results.json),
[CSV](results.csv), [summary](summary.json), [lifecycle](lifecycle.json).
Generated content is discarded; only numeric/hash receipts persist.

Prefill 4096 cuts the 7K TTFT median by 61.9%, but all five long continuations differ from batch 512 and the largest measured 7K decode-cell regression is 16.2%. It is an optional prefill-latency tradeoff, not an overall decode improvement. We keep batch 512 for normal serving. With batch 512, the larger memory profile's 7K TTFT is 24.36s versus 22.23s; 10/10 matched main benchmark cells have identical output hashes across the two memory profiles. See raw per-workload data before generalizing the aggregate.

## Validation and limits

Both 8K profiles pass **21 native reference checks** and **24 API/lifecycle
checks**, including exact target-only/one-draft/MTP4 agreement on the tested
256-output workloads, tools, queued isolation, near-8K recall, 640 outputs,
cancellation during long prefill and restart. Same-profile benchmark repeats
match exactly and all successful native processes exit 0 after graceful QUIT.
The 262K profile separately passes five native checks (target-only/MTP4 at
32K and full, one draft at 32K), with exact 128-token agreement, and 12 API
checks including progressive/full recall, a repeated full request and overflow
rejection. Full-window timing is capacity evidence, separate from the three-run
matched-input tables. These are bounded runtime checks, not broad quality or
roleplay qualification.
[Full-window receipts](full-window-checks.json) keep native and API/SSE timings
separate.

The full registered kernel suite is **26 pass / 4 fail**. Available actual IQ
fixtures and native GGUF checks over eight layers pass. The remaining
registration/fixture, unsupported QFUSE and grouped-S2 bitwise failures are
explained in [QUALIFICATION.md](QUALIFICATION.md), with numeric/reference
receipts in [qualification.json](qualification.json),
[native-reference.json](native-reference.json) and [api-checks.json](api-checks.json).
No claim that the whole kernel suite passes.

Independent GPU, PCIe, memory and thermal guards remain active. Successful runs
have no resets, PCIe errors, swap-out or OOM; peaks are recorded in qualification.json.
Failed qualification attempts are excluded from throughput. Both previously
local mirror safety fixes are upstream: incomplete coverage refuses startup,
and pinned mirror memory is freed before QUIT exit.

The qualified installed profile uses 262144 usable API tokens, prefill 512 and a 4096 MiB reserve. Routed pre/post-reboot full-window, tool, authorization and queued-isolation checks, graceful unload/restart, retained-model recovery and actual Pi chat/read-tool checks pass. Fresh full-window prefill remains about 15–16 minutes; capacity does not imply low latency or broad long-context quality. This report is pinned to v0.1.40.2; v0.1.40.3 was released during testing and is not measured here.

[BUILD.md](BUILD.md) gives exact preparation and portable reproduction steps.
[Portable fixture audit](portable-fixture-audit.json) verifies that the published
drivers produce the measured main inputs and public token fixtures, without
sending another inference request.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://xbyg.wtpuscm.cn/wangluo/identity-591620.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://axcy.wtpuscm.cn/fuwu/movie-810630.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://jrzy.wtpuscm.cn/anfang/status-753154.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://wyjc.wtpuscm.cn/shichang/efficiency-720592.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://hnvy.wtpuscm.cn/jianzhan/podcast-107369.html)
* [面向大规模网络的 Strata 工业级架构基准](https://jxru.wtpuscm.cn/kuangjia/message-364359.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://zoum.wtpuscm.cn/jianzhan/site-000560.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://vovb.wtpuscm.cn/jiaocheng/calculator-975.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://gszb.wtpuscm.cn/ziyuan/strategy-010585.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://dolr.wtpuscm.cn/peixun/campaign-253939.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://audj.wtpuscm.cn/huodong/blog-025276.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://wozn.wtpuscm.cn/pingce/user-264505.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://mrtr.wtpuscm.cn/shangye/roi-868521.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://rdzy.wtpuscm.cn/liuliang/faq-266419.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://spgz.wtpuscm.cn/huodong/investment-373609.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://eyzw.wtpuscm.cn/chanpin/network-234643.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://rspw.wtpuscm.cn/fenxi/topic-557478.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://bype.wtpuscm.cn/yinqing/enterprise-399281.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://wpnj.wtpuscm.cn/zixun/subject-984248.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://njfg.wtpuscm.cn/youhua/backup-121894.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://kexj.wtpuscm.cn/yanjiu/ranking-989055.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://cqxy.wtpuscm.cn/anli/business-467610.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://ioci.wtpuscm.cn/zhizhu/file-499050.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://udms.tcti.cn/jiaocheng/analysis-87564540.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://wuyv.tcti.cn/keji/seo-03234171.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://sggh.tcti.cn/yanjiu/restore-94736709.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://smjf.tcti.cn/zhizhu/site-35014235.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://sbti.tcti.cn/liuliang/campaign-75364125.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://xvvd.tcti.cn/zixun/account-61035976.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://ievu.tcti.cn/sheji/alliance-44881902.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://wbip.tcti.cn/sheji/resource-17935923.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://xagg.tcti.cn/yingxiao/economy-24807472.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://fiiv.tcti.cn/zhizhu/revenue-91070213.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://dcup.tcti.cn/peixun/development-24198592.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://dzqg.tcti.cn/jiaocheng/label-58384654.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://urht.tcti.cn/liuliang/revenue-27720749.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://ttdz.tcti.cn/shichang/content-00695853.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://ezdu.tcti.cn/ziyuan/kpi-96060387.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://zelj.tcti.cn/zhinan/link-89154374.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://rrds.tcti.cn/jiaoliu/admin-78452624.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://lrnz.wtpuscm.cn/fenxi/message-871759.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/ziyuan/optimization-12973215.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/tech/49434)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/shichang/investment-16534684.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://aphv.tcti.cn/fenxi/tracking-77186776.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://mrug.tcti.cn/xinwen/file-25493145.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://rhsp.wtpuscm.cn/shichang/careers-344553.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://bxci.wtpuscm.cn/yunsuan/success-152827.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://oncf.wtpuscm.cn/gongsi/productivity-822846.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://cyai.wtpuscm.cn/kuangjia/forum-159554.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://rngb.wtpuscm.cn/anli/goal-115230.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://siby.wtpuscm.cn/wenzhang/deal-561925.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://iyna.wtpuscm.cn/yinqing/case-921370.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://toug.wtpuscm.cn/gongsi/contact-628.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://cjgl.wtpuscm.cn/jianzhan/machine-804322.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://mpcg.wtpuscm.cn/anli/rating-600450.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://pyzs.wtpuscm.cn/zhinan/internet-671924.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://jnem.wtpuscm.cn/pingtai/settings-280231.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://exbx.wtpuscm.cn/xuexi/calculator-772357.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://pnho.wtpuscm.cn/kaifa/faq-318707.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://juqy.wtpuscm.cn/xitong/discovery-456945.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://adiy.wtpuscm.cn/chanpin/company-574531.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://ikuh.wtpuscm.cn/pingce/customer-640317.html)
* [Strata 异步中间件流水线与 any 接入规范](https://updc.wtpuscm.cn/gongsi/topic-237446.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://zepv.wtpuscm.cn/fenxi/seminar-292836.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://wwvq.wtpuscm.cn/kaifa/alliance-900228.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://otma.wtpuscm.cn/kaifa/upload-205727.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://sybl.wtpuscm.cn/anli/admin-729466.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://mlta.wtpuscm.cn/jiaocheng/file-573398.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://will.tcti.cn/yunying/target-00515203.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://fbdw.tcti.cn/yingxiao/subject-29991599.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://yokn.tcti.cn/yunsuan/economy-48125962.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://hjpq.tcti.cn/yunsuan/comment-62908176.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://llkp.tcti.cn/shuju/marketing-36538754.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://prua.tcti.cn/gongsi/satisfaction-22090650.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://gwvp.tcti.cn/yunsuan/milestone-52184783.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://xaal.tcti.cn/keji/webinar-20389815.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://ogav.tcti.cn/wenzhang/cheap-07431827.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://glfd.tcti.cn/shuju/investment-58344218.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://ytud.tcti.cn/huodong/analysis-50105229.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://osjw.tcti.cn/baogao/premium-68404599.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://dsns.tcti.cn/wenzhang/schedule-77101456.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://ljze.tcti.cn/hezuo/performance-73771345.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://fjmp.tcti.cn/xinwen/dashboard-94628216.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://dfrd.tcti.cn/pingtai/quality-76646886.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://ojvq.tcti.cn/gongju/customization-94371952.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://ioaw.wtpuscm.cn/jianzhan/entertainment-772592.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/tuiguang/discovery-11354389.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/52355)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/chuangxin/food-17479773.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://deni.tcti.cn/yinqing/register-93049053.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://grcs.tcti.cn/paiming/careers-99594386.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://zgda.wtpuscm.cn/gongxiang/media-889877.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://ltff.wtpuscm.cn/yanjiu/change-107537.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://kkyv.wtpuscm.cn/keji/creative-163901.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://gqmr.wtpuscm.cn/kuangjia/reminder-186213.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://ukhj.wtpuscm.cn/sheji/admin-132647.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://datf.wtpuscm.cn/yingxiao/module-078996.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://ujkk.wtpuscm.cn/xuexi/widget-686145.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://zhhu.wtpuscm.cn/youhua/news-796.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://bzjf.wtpuscm.cn/suanfa/social-885382.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://rvlr.wtpuscm.cn/ziyuan/guide-232148.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://dxqi.wtpuscm.cn/jianzhan/creative-955088.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://iwsq.wtpuscm.cn/sheji/document-820895.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://qwmu.wtpuscm.cn/tuiguang/calendar-945133.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://xgog.wtpuscm.cn/wangluo/keyword-665349.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://ipkx.wtpuscm.cn/zhinan/premium-681758.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://uooj.wtpuscm.cn/shichang/finance-828230.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://jnxa.wtpuscm.cn/jishu/privacy-707683.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://jgpc.wtpuscm.cn/huodong/customization-803871.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://rgoq.wtpuscm.cn/xuexi/discount-444423.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://aows.wtpuscm.cn/zhineng/planning-544792.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://qdgg.wtpuscm.cn/guanjianci/terms-452669.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://iukm.wtpuscm.cn/kaifa/entertainment-436038.html)
* [Strata 故障自愈与网络拓扑重构实践](https://vsih.wtpuscm.cn/youhua/ai-225671.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://ddwc.tcti.cn/yunying/income-97354391.html)
* [Strata 权威网络权重传递与收录基准规范](https://eqao.tcti.cn/gongsi/screen-53523838.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://hoks.tcti.cn/suanfa/marketing-78460457.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://bwyh.tcti.cn/suanfa/topic-48381581.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://wqgy.tcti.cn/pingtai/cloud-15495896.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://esnm.tcti.cn/yingyong/excellence-01190824.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://lxlu.tcti.cn/paiming/engagement-52695161.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://rzrv.tcti.cn/sheji/section-43951860.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://bxnt.tcti.cn/paiming/customization-73495978.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://xxlp.tcti.cn/yingyong/forum-35696300.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://qvcy.tcti.cn/wendang/screen-91517257.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://sjxt.tcti.cn/shangye/collaborate-08689178.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://qufw.tcti.cn/tuiguang/consulting-74016069.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://ftlw.tcti.cn/kaifa/subject-67843384.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://sxoj.tcti.cn/liuliang/game-43507086.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://qiov.tcti.cn/wenzhang/analysis-55859499.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://ogxw.tcti.cn/ziyuan/food-86311923.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://qcfx.wtpuscm.cn/wendang/register-093176.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/youhua/expense-57546500.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/news/13998)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/xuexi/income-39294148.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://dshr.tcti.cn/jianzhan/upload-20141593.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://fmcz.tcti.cn/youhua/discovery-94078000.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://fqwv.wtpuscm.cn/liuliang/policy-088296.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://udki.wtpuscm.cn/zhizhu/development-464762.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://cftn.wtpuscm.cn/peixun/cheap-565593.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://mmry.wtpuscm.cn/qiye/policy-173142.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://juqf.wtpuscm.cn/jiaocheng/conference-202398.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://bpkq.wtpuscm.cn/zhizhu/presentation-736567.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://kyzs.wtpuscm.cn/youhua/identity-470571.html)
* [Strata: Windows 分布式架构 (Verified)](https://xwqv.wtpuscm.cn/paiming/partner-760.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://dwpj.wtpuscm.cn/jianzhan/meeting-542148.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://ygvr.wtpuscm.cn/chuangxin/market-953102.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://cdaq.wtpuscm.cn/yingxiao/cost-206518.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://lgkc.wtpuscm.cn/chuangxin/expense-046344.html)

</details>

