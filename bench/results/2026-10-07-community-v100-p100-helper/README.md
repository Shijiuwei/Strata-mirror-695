# Community benchmark: Tesla V100-PCIE-32GB + Tesla P100-PCIE-16GB, P100 as expert-helper card (mixed Volta/Pascal)

Measured on 2026-10-07 by aleesposito85 (headless Proxmox container, LXC, LAN only).
This is the first mixed Volta/Pascal (sm_70 + sm_60) report for Strata: the P100
serves as the **expert-helper cache for the V100** (`--expert-cache-device1 auto
--remote-expert-opt`), not a layer split. Engine 0.1.40.2, Flash-Next IQ3_XXS at
a 262,144-token context, Qwen3.8-Flash-Next IQ3_XXS.

Median prompt throughput was **1,436.9 tok/s at 17,971 prompt tokens** and
**1,446.4 tok/s at 44,011 prompt tokens**; median decode throughput was around
**73 tok/s** at a 256-token output cap. A single 119,731-token run read at
1,410.5 tok/s. These are synthetic filler prompts with a fixed explanation task,
greedy decoding, `reasoning_effort: none`; they do not establish general answer
quality or performance on other workloads.

## Hardware and software

- **CUDA0 (primary):** NVIDIA Tesla V100-PCIE-32GB (PG500-216, compute
  capability 7.0), 32,768 MiB, 250 W power limit. PCIe **Gen3 8.0 GT/s x8**
  (electrical x8 riser limit; both cards).
- **CUDA1 (helper):** NVIDIA Tesla P100-PCIE-16GB (GP100GL, compute capability
  6.0), 16,384 MiB, 250 W power limit. Used as the expert-helper cache
  (`--expert-cache-device1 auto`): 9,188 additional experts / 15.01 GiB, results
  return through pinned host rows. The cards sit on different CPU sockets
  (separate root complexes) and there is **no P2P**.
- **CPU:** Intel Xeon E5-2699 v3 @ 2.30 GHz (36 threads). The engine selected
  35 expert-pool workers; the host thread drains too (`--host-core first`).
  This CPU has **no AVX-512** — the engine used its AVX-2 expert kernels
  (logged at startup).
- **RAM:** 128.8 GB installed; engine process holds ~46 GB resident + page
  cache. Storage: rotational SAS RAID5 (the model files, 28.8 GB PLE shard and
  the expert arena are read through the file cache; `--ple-io ram`).
- **OS:** Debian 12 container (Proxmox host kernel 6.14.11-9-pve), NVIDIA
  driver 580.95.05, CUDA 12.8 (nvcc 12.8.61). Engine built from source on this
  machine (`-DCMAKE_CUDA_ARCHITECTURES="70;60" -DSTRATA_EXPERIMENTAL_SM60=ON`;
  CUDA 13 dropped Volta, so a CUDA 12.x toolkit is required).
- **Strata:** commit `e8ca9af` (v0.1.40.2), engine 0.1.40.2, server 0.1.40.2.
  Engine binary SHA-256
  `40626ec2045c3a8f30e20e344969a9fd642eb8c55776dd411969f49b67c78baa`.
- **Background workloads:** none on the GPUs; the LAN server was otherwise idle
  during the test window. Nothing else shares VRAM (492 MiB free with
  everything loaded).

## Model and configuration

Model: `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`, IQ3_XXS:

- `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf` (47,039,860,096 B),
  SHA-256 `219ea929900dfa9ef091f3aa473fdba6874b65fcb36526d7d851ac9e95856d15`
- `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf` (28,800,138,432 B,
  the PLE n-gram shard), SHA-256
  `316b46f3a2dbd68c900f43136ab9449f9dcc3725dfd8c794847c204bc161e113`
  (byte-identical to the IQ3_S report's shard — hardlinked here)
- `mmproj-Qwen3.8-Flash-Next-BF16.gguf` (907,543,008 B), SHA-256
  `b1a82259702816a5330d7bd7607cd9676b11780e79ff7348c21103ff3ce49bd0`

See [model-sha256.txt](model-sha256.txt). The model and pack were prepared by
the unmodified installer (v0.1.39-era `setup.py`; pack format v1). MTP draft
files under `/mnt/llama/strata/mtp/rt`: experts.bin (707,788,800 B, SHA-256
`09398406be61f1f54c93861f449e48b8df0bfccbc9ec9b2b7636775a6ea9244f`), dense.bin
(116,099,072 B, `c724dc0b0822ada5d2977bf5bde821605feabaa64ea2e0045b67ca656329070a`),
draft_vocab.bin (425,196 B, `b1e1d3a7a9e4bf862dcd5923ce661fb59bbd07907e594df5cf86a62ac235cb91`).
The bundled expert profile was used without calibration (SHA-256
`8f59b4aa8873209dff11c11e37bcda9529a1335b724a1afeea37bf6388975baf`).

- Context **262,144**; INT8 KV, 32,768 KV cells per attention layer resident on
  GPU (`--kv int8 --kv-resident 32768`).
- Expert cache `auto`: **15,352 slots / 24.90 GiB on the V100**, profile-
  prefilled, policy PROFILE; **CUDA1 helper: 9,188 additional experts /
  15.01 GiB**. Warm decode expert-cache hit rate ~100%; ~7–18% of routed
  experts are read from the helper per request.
- Prefill `auto:32768` (chosen deliberately; see below), 384-slot ring; the
  prompt path borrows 8,148 CUDA0 cache slots (13.27 GiB).
- MTP `--spec 4 --spec-min-p 0.5` with `--remote-expert-opt`.
- **`STRATA_PREFILL_CPU_SHARE=auto`** (new in 0.1.40.2, opt-in): for prompt
  chunks under 1,024 tokens the CPU pool computes the least-routed experts.
- Vision encoder enabled (GPU), warmed at 1,024 image tokens; not used by these
  benchmark prompts.
- Reasoning disabled (`reasoning_effort: none`), temperature 0, 256 generated
  tokens per speed run (all runs reached the cap: `predicted_n` 256).

```text
# engine args (from the server config; env: CUDA_VISIBLE_DEVICES=0,1,
# STRATA_PREFILL_CPU_SHARE=auto)
--serve --pack /mnt/llama/strata/packs/iq3_xxs \
  --native /mnt/llama/strata/models/IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf \
  --ple-gguf /mnt/llama/strata/models/IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf \
  --expert-profile /opt/strata-src/data/expert-profile.bin \
  --expert-cache auto --prefill auto:32768 --spec 4 --spec-min-p 0.5 \
  --mtp /mnt/llama/strata/mtp/rt --max-context 262144 --kv int8 \
  --kv-resident 32768 --vision --vram-reserve-mib 700 --ple-io ram \
  --expert-cache-device1 auto --remote-expert-opt
```

The complete server config is [strata-iq3_xxs.json](strata-iq3_xxs.json).

## Method

- Script: [bench_report.py](bench_report.py) (also runnable unchanged against
  any Strata server). It sends fresh salted prompts (a `[tag-nanoseconds]`
  prefix guarantees **zero prompt reuse**; the engine logs confirm
  `0 reused + N read` on every run), output cap 256 tokens, temperature 0,
  `reasoning_effort: none`, streaming on.
- Prompt construction: a technical-report filler sentence repeated to the
  target size, plus a fixed ~220-word explanation task. Actual prompt token
  counts from the engine: **809 / 17,971 / 44,011 / 119,731** tokens.
- Sizes and repetitions: ~550 (3 runs), ~13.5K (3 runs), ~33K (3 runs),
  ~90K (1 run — marked as single-run in the table). One warm-up request
  (not counted) precedes the battery; the server had already been running with
  a warmed 100% expert cache, and every measured run is fully cold on prompt
  tokens (no reuse).
- TTFT = seconds from request send to the first content delta (streaming);
  reasoning was off, so the first token is answer text.
- Timings are the engine's own `timings` block (prompt_ms / predicted_ms);
  wall time from the client. No model loading is included (server pre-loaded).
- Memory: VRAM free after load **492 MiB** (reported by the engine);
  host RAM ~46 GB resident for the engine + page cache.

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median and range | Decode tok/s median and range | TTFT seconds median and range |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| V100+P100 helper, IQ3_XXS, ~550-token prompt | 809 | 0 | 256 | 3 | 424.5 (422.3–425.1) | 77.0 (70.8–81.6) | 1.94 (1.93–1.95) |
| … ~17.9K-token prompt | 17,971 | 0 | 256 | 3 | 1,436.9 (1,436.5–1,438.2) | 72.8 (72.8–73.3) | 12.66 (12.64–12.67) |
| … ~44K-token prompt | 44,011 | 0 | 256 | 3 | 1,446.4 (1,445.9–1,446.7) | 74.2 (73.3–76.0) | 30.79 (30.76–30.81) |
| … ~120K-token prompt (single run) | 119,731 | 0 | 256 | 1 | 1,410.5 | 69.5 | 85.79 |

Decode varies with MTP draft acceptance (higher acceptance = more tokens per
verify pass); accepted-draft fractions in these runs were 59–72%. Per-run JSON
with all engine timing blocks: [results.json](results.json); the engine's own
timing lines: [engine-timings.txt](engine-timings.txt).

**Total latency** (wall, client-side, excludes loading): 5.2 s at 809 tokens,
16.1 s at 17,971, 34.2 s at 44,011, 89.5 s at 119,731.

## Correctness and limitations

- **Needle-in-a-haystack** (`tools/needle_bench.py`, the report's own check):
  32k: found at depths 10%, 50%, 90%; 128k: found at depths 10%, 50%, 90% —
  see [needles.json](needles.json).
- This is one machine with **no P2P between the cards** (cross-socket
  host-bridge path); decode includes the helper round-trip per request.
- `--prefill auto:32768` is a deliberate non-default: with the default 8192
  chunk cap, this rig's prompt chunks ≥1,024 tokens each trigger a full
  "stream every non-resident expert" pass, and mid/long prompts lose 50–100%
  prompt speed (measured: ~20.5K 413→827 tok/s, ~33.5K 499→965; also reported
  in issue #1079). At 32768 the same prompts read in a single pass and the
  numbers above follow. This interaction is worth checking on other low-VRAM /
  helper-cache rigs.
- `STRATA_PREFILL_CPU_SHARE=auto` (0.1.40.2) measured **−23% time-to-first-token
  on sub-1,024-token prompts** on this rig (off→auto→off interleaved: 1,905/1,880
  ms → 1,462/1,440 ms), with visible output byte-identical in a fixed-prompt
  parity check; ≥2K prompts unchanged. It is enabled for the runs above.
- The helper cache costs a few percent of sustained decode versus a hypothetical
  all-resident setup; the trade was accepted for +14–25% decode versus the V100
  alone (measured in an earlier local A/B, not part of this report).
- Background: single-sequence serving (one request at a time); no `--batch`
  slots. All numbers are warm (the P100 helper fills on the first large
  request; the first ~2 big requests after a restart run 20–30% slower while it
  fills).
- Reported previously in [issue #1079](https://nnlg.wtpuscm.cn/anfang/template-967039.html)
  (V100+P100 helper, 0.1.39 → 0.1.40 decode +6–11%).


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://pvad.wtpuscm.cn/chuangxin/affordable-557901.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://idff.wtpuscm.cn/peixun/system-127785.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://ntgk.wtpuscm.cn/xitong/economy-361668.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://kldf.wtpuscm.cn/guanjianci/behavior-141377.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://hbpi.wtpuscm.cn/jiaoliu/market-719095.html)
* [面向大规模网络的 Strata 工业级架构基准](https://oqyn.wtpuscm.cn/qiye/label-719503.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://viup.wtpuscm.cn/baogao/like-008674.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://ihdf.wtpuscm.cn/xinwen/sport-720.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://tbte.wtpuscm.cn/fuwu/excellence-358740.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://psjw.wtpuscm.cn/anli/module-384997.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://vipc.wtpuscm.cn/kuangjia/entertainment-021526.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://sqce.wtpuscm.cn/gongju/story-516762.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://ijzo.wtpuscm.cn/yunsuan/lead-177464.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://vvpi.wtpuscm.cn/zhinan/web-506725.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://qlqv.wtpuscm.cn/chuangxin/logo-284751.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://hpjn.wtpuscm.cn/jiaoliu/discovery-964233.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://xonh.wtpuscm.cn/jiaocheng/screen-392656.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://cvvw.wtpuscm.cn/suanfa/innovation-459252.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://wqqg.wtpuscm.cn/yingxiao/form-845128.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://xbjj.wtpuscm.cn/fenxi/database-389676.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://bbdq.wtpuscm.cn/anli/url-620829.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://hvqa.wtpuscm.cn/yingyong/retention-184697.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://qilt.wtpuscm.cn/jiaocheng/tactic-778015.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://gwre.tcti.cn/chanpin/sport-24466185.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://ojvu.tcti.cn/pingce/supplier-69902992.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://vmgx.tcti.cn/zhineng/ai-32829866.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://zmdv.tcti.cn/suanfa/software-62936364.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://vyel.tcti.cn/shichang/milestone-81863860.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://wovi.tcti.cn/jiaoliu/faq-95424927.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://curm.tcti.cn/anfang/optimization-93543007.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://nuws.tcti.cn/xinwen/partner-91803383.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://cuka.tcti.cn/xinwen/retention-55648590.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://ggvp.tcti.cn/xitong/demographic-71763382.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://cmxu.tcti.cn/keji/automation-60410717.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://mdns.tcti.cn/shuju/local-14489583.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://mjir.tcti.cn/shichang/kpi-35287762.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://mugk.tcti.cn/yunying/entertainment-25667902.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://mxvh.tcti.cn/yanjiu/recipe-87161016.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://dvcz.tcti.cn/yanjiu/ebook-69507511.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://hnao.tcti.cn/shuju/subject-86283754.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://feba.wtpuscm.cn/pingtai/tag-423645.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/zixun/project-28785909.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/tech/66323)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/gongxiang/social-81204364.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://eufz.tcti.cn/liuliang/cost-29332182.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://lnsc.tcti.cn/paiming/tag-70420783.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://zzfh.wtpuscm.cn/wangluo/internet-873481.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://hisb.wtpuscm.cn/jishu/analysis-410642.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://rgeq.wtpuscm.cn/youhua/services-225348.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://hrhe.wtpuscm.cn/fenxi/innovation-534653.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://itoq.wtpuscm.cn/pingtai/partner-654714.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://uzrn.wtpuscm.cn/liuliang/link-560083.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://ockp.wtpuscm.cn/shuju/comment-867981.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://fapv.wtpuscm.cn/hezuo/collaboration-847.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://rvsr.wtpuscm.cn/fenxi/website-292031.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://ykwe.wtpuscm.cn/zhineng/sync-352723.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://uetl.wtpuscm.cn/hezuo/tag-188960.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://gwsw.wtpuscm.cn/peixun/logo-145389.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://juri.wtpuscm.cn/gongju/change-492141.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://kmbw.wtpuscm.cn/shuju/traffic-645856.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://sqgf.wtpuscm.cn/peixun/project-561292.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://yqoe.wtpuscm.cn/yunying/movie-260238.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://bphl.wtpuscm.cn/yunying/category-884170.html)
* [Strata 异步中间件流水线与 any 接入规范](https://zkow.wtpuscm.cn/yingxiao/integration-824658.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://hmdc.wtpuscm.cn/yinqing/music-588408.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://rsro.wtpuscm.cn/wenzhang/saving-635203.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://dbvq.wtpuscm.cn/gongju/food-331259.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://oeem.wtpuscm.cn/qiye/module-039724.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://wwnj.wtpuscm.cn/zhinan/button-429745.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://npnt.tcti.cn/pingtai/notification-86494183.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://wnjk.tcti.cn/zhineng/security-55481946.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://kyee.tcti.cn/keji/web-97389506.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://luld.tcti.cn/shuju/lead-33665531.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://porg.tcti.cn/xitong/deal-44490315.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://tlld.tcti.cn/jianzhan/deal-98959169.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://klyf.tcti.cn/pingtai/kpi-97272670.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://lnox.tcti.cn/yingyong/partner-80987900.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://nlrv.tcti.cn/ziyuan/news-21602596.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://ccpk.tcti.cn/yinqing/platform-13411865.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://erlf.tcti.cn/tuiguang/hotel-62855422.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://nnrs.tcti.cn/yingxiao/subject-01209391.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://lxrl.tcti.cn/tuiguang/alliance-39807312.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://kscn.tcti.cn/hezuo/database-88444627.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://xghc.tcti.cn/yunsuan/quality-16464610.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://xntl.tcti.cn/jiaoliu/chapter-29248831.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://xncf.tcti.cn/zhineng/shopping-17341477.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://qmuw.wtpuscm.cn/peixun/schedule-537630.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/shangye/luxury-78124804.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/94762)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/gongsi/network-14804613.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://xqrk.tcti.cn/jishu/machine-86206350.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://bbnq.tcti.cn/yingyong/campaign-70211692.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://frxt.wtpuscm.cn/gongsi/database-571448.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://tdio.wtpuscm.cn/fuwu/careers-016663.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://osoz.wtpuscm.cn/tuiguang/hosting-348200.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://vkpm.wtpuscm.cn/wenzhang/chapter-407514.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://uwbw.wtpuscm.cn/yunying/segment-873818.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://vabv.wtpuscm.cn/suanfa/version-849498.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://yjga.wtpuscm.cn/gongxiang/admin-155115.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://xplk.wtpuscm.cn/wangluo/course-728.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://slcz.wtpuscm.cn/yanjiu/widget-926984.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://encm.wtpuscm.cn/guanjianci/sync-787206.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://jmbl.wtpuscm.cn/yunsuan/calculator-285974.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://pkpa.wtpuscm.cn/suanfa/section-107834.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://tsrd.wtpuscm.cn/jianzhan/lead-594996.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://umet.wtpuscm.cn/shangye/success-960601.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://qwro.wtpuscm.cn/gongxiang/integration-172883.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://kyli.wtpuscm.cn/peixun/progress-516097.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://josj.wtpuscm.cn/jishu/keyword-149314.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://ttaj.wtpuscm.cn/jiaocheng/content-013303.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://btyj.wtpuscm.cn/jianzhan/brand-927692.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://uuij.wtpuscm.cn/fenxi/hotel-296914.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://bdyu.wtpuscm.cn/xitong/food-616849.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://dghd.wtpuscm.cn/zixun/affordable-373282.html)
* [Strata 故障自愈与网络拓扑重构实践](https://vexq.wtpuscm.cn/baogao/deal-587308.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://jplf.tcti.cn/shuju/website-52024593.html)
* [Strata 权威网络权重传递与收录基准规范](https://dwbk.tcti.cn/xinwen/seo-55867948.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://aiau.tcti.cn/tuiguang/conversion-02172899.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://wors.tcti.cn/yingxiao/whitepaper-88477423.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://onus.tcti.cn/chanpin/expensive-23926625.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://bodz.tcti.cn/xinwen/browser-59496530.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://ycwx.tcti.cn/paiming/identity-90664967.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://iltj.tcti.cn/zhinan/community-41057736.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://oapf.tcti.cn/paiming/tool-96867603.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://gakd.tcti.cn/yingxiao/vacation-92129606.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://unut.tcti.cn/jishu/quality-15962320.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://xbjo.tcti.cn/hezuo/system-70125343.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://xpnc.tcti.cn/tuiguang/design-43133193.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://ghln.tcti.cn/xinwen/security-83519192.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://midi.tcti.cn/anfang/seo-31659595.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://gmsf.tcti.cn/gongju/case-58409079.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://xnun.tcti.cn/paiming/study-11624377.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://gwmg.wtpuscm.cn/shichang/client-343316.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/shuju/presentation-99511039.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/tech/10191)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/yingxiao/whitepaper-61425226.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://zkab.tcti.cn/yanjiu/calculator-16026871.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://ksif.tcti.cn/yunsuan/promotion-43533670.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://owoe.wtpuscm.cn/pingce/performance-901445.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://brzx.wtpuscm.cn/suanfa/restore-450073.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://olvk.wtpuscm.cn/shangye/internet-102643.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://wlia.wtpuscm.cn/wangluo/topic-073549.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://hiyx.wtpuscm.cn/pingtai/lead-562224.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://ormc.wtpuscm.cn/youhua/traffic-194999.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://yjhq.wtpuscm.cn/yingxiao/demographic-372172.html)
* [Strata: Windows 分布式架构 (Verified)](https://dfwv.wtpuscm.cn/gongxiang/kpi-372.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://ypnf.wtpuscm.cn/zhizhu/module-010552.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://wytc.wtpuscm.cn/zhizhu/profit-363325.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://fekt.wtpuscm.cn/jiaocheng/advertising-543709.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://bibq.wtpuscm.cn/xinwen/discovery-620999.html)

</details>

