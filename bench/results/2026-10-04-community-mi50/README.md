# Community benchmark: AMD MI50 (gfx906), Intel i5-12400F

Measured on 2026-10-04 by [xxDoman](https://www.ai-hao123.com/xitong/accessibility-44571411.html), on the Linux
machine **LianLi**. This tests Strata 0.1.38 with the original Flash-Next
IQ2_XS, one GPU (the MI50), at the model's full **131,072-token context** with
KV streaming.

Median decode throughput was **36.0 tok/s at 4,096 prompt tokens, 34.0 tok/s at
32,768, and 33.5 tok/s at 128,000** — it barely drops as the context grows.
Prompt processing held **~330 tok/s at 4K/32K and ~305 tok/s at 128K**. These are
synthetic code-explanation requests with greedy decoding and a 256-token output
cap. They do not establish general answer quality or performance on other
workloads.

This is, to our knowledge, the first measured community run of Strata on an AMD
MI50 (Vega 20, gfx906). It is an experimental, unsupported architecture: the
build carries no vendor tuning tables, and the numbers reflect that. It also
shows that the MI50 — a 32 GB card — can serve the model's full 128K context
with KV streaming, at a decode cost of only ~7% versus a 4K prompt.

## Hardware and software

- AMD Radeon Instinct MI50 32GB (Vega 20, gfx906); 34,342,961,152 bytes VRAM
  reported; PCIe Gen4 x16 (`LnkSta` 16 GT/s, width x16). The TDC GFX limit is
  pinned to 150 W at boot (a host service); clocks were not fixed otherwise.
- 12th Gen Intel Core i5-12400F; 12 logical CPUs. The engine selected an AVX2
  build and 5 expert-pool workers plus its host thread.
- 64 GB installed RAM; Linux reported 61.50 GiB total. Local NVMe storage:
  Lexar SSD NM790 2TB. 15 GiB system swap.
- Ubuntu 26.04.1 LTS, kernel 7.0.0-38-generic. AMD **ROCm 7.2.1** (`hipcc`
  reports AMD clang 22.0.0).
- Source commit `99f3dbd0b21d1401b3769e0c0d963913607f380b`; engine 0.1.38.
  Built on the host for **gfx906** with the ROCm 7.2.1 toolchain and packaged
  into a Docker image (`strata-mi50:0.1.38`); the server runs inside the
  container with `/dev/kfd` and `/dev/dri` passed through. See [BUILD.json](BUILD.json).
- The GPU was dedicated to Strata during the run (the co-located Ollama server
  held no model). Other CPU services remained running; this was not a
  completely isolated operating system.

## Model and configuration

Model: `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`, revision
`ed59f92082b1e93c0e96d60a8b11aab089b52f09`:

- `IQ2_XS/Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf`
- `IQ2_XS/Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf`

Both local GGUF SHA-256 hashes matched the revision's published LFS hashes
(see [model-sha256.txt](model-sha256.txt) and [provenance.json](provenance.json)).
`mmproj-Qwen3.8-Flash-Next-BF16.gguf` is present on disk but was **not** passed
to the engine: vision was off for this run, so this report does not cover it.

- **Context 131,072; KV streaming `--kv-resident 32768`** — 32,768 KV cells per
  QSA layer resident in VRAM, with the full K/V in pinned RAM. This is what
  makes the full 128K context fit the card while leaving VRAM for experts.
- KV type INT8.
- Expert cache `auto`: 19,304 slots, profile-prefilled with no eviction. Engine
  log: cache hit path ON; per-request `hit_rate` 0.97–0.995.
- Prefill 2,048-token chunks.
- MTP `--spec 3 --spec-min-p 0.9`.
- Vision off; temperature 0; maximum 256 generated tokens per speed run.

The complete measured configuration is [strata-iq2_xs.json](strata-iq2_xs.json)
and the engine's own startup and per-request lines are in [engine.log](engine.log).

## Method and reproduction

This uses an adaptation of the repository's community harness
([`bench/results/2026-09-30-community-rtx-5090/benchmark.py`](../2026-09-30-community-rtx-5090/benchmark.py)).
The only changes are the target port/pack path for the container and a required
`Authorization` header ([benchmark_mi50.py](benchmark_mi50.py)). It runs inside
the container (imports `strata_tokenizer` and `serve.frontend` from `/opt/strata`).

```bash
docker exec strata python3 /work/bench/benchmark_mi50.py --api-key <key> \
  --url http://127.0.0.1:8085 --out /work/bench/results --targets 4096,32768,128000 --runs 3
```

The script generates deterministic synthetic Python functions, adds a different
nonce near the start of each request, and counts the complete rendered chat
prompt using Strata's tokenizer, adjusting the filler to the target and
verifying the count against the server afterward. One warm-up request was
excluded. Three runs at each length were executed serially in increasing-length
order on the same loaded engine. All nine speed requests processed their entire
prompt: **zero reused tokens**. Loading time is excluded; the expert cache was
kept between requests.

TTFT is measured client-side from immediately before the HTTP request until the
first nonempty text delta. Engine prompt throughput uses freshly read tokens
divided by `prompt_ms`; decode throughput uses `engine_generated / decode_ms`.

System memory and GPU memory were sampled once per second by
[monitor_mi50.py](monitor_mi50.py) (host `rocm-smi` adaptation), before load
through the last run; raw samples in [telemetry.jsonl](telemetry.jsonl).

## Results

Each cell is the median **[minimum–maximum]** of three runs. Every request
generated 256 tokens and stopped at the output limit; no failed or cancelled
speed requests.

| Prompt tokens | Reused | Prompt tok/s | Decode tok/s | TTFT seconds | Total seconds |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 331.9 [330.1–332.3] | 36.0 [35.9–37.4] | 12.4 [12.4–12.5] | 19.4 [19.3–19.5] |
| 32,768 | 0 | 327.2 [327.2–327.3] | 34.0 [33.9–34.1] | 100.2 [100.2–100.2] | 107.7 [107.7–107.7] |
| 128,000 | 0 | 305.2 [304.7–305.2] | 33.5 [32.4–34.0] | 419.6 [419.4–419.7] | 427.2 [426.8–430.0] |

Raw per-run records: [results.json](results.json). Calculated aggregates:
[summary.json](summary.json). Per-request engine values are also preserved in
`tokens-<N>-run-<R>-raw.json`.

The very first request of a session (a previous, discarded invocation) showed a
one-off prefill overhead (~35 s for 14 prompt tokens) before the cache settled;
the warm-up in this run was excluded, and steady-state prefill was a flat
~305–330 tok/s across all three lengths.

Memory, from [telemetry.jsonl](telemetry.jsonl):

- Host RAM: peak **40.96 GiB**, calculated as `MemTotal - MemAvailable` over all
  processes (not Strata's RSS alone). The ~41 GiB is the expert set in RAM plus
  the streamed K/V for the 128K context.
- Swap: peak **~2.9 GiB used**, up from ~0 — Strata pages during the run.
- GPU memory: the MI50 was filled to essentially all available VRAM
  (~33.5 GB, 32 GB class card); the engine reported `964 MiB` free with the
  window up and `1024 MiB` reserved for the draft head.
- No out-of-memory error occurred.

## Correctness and limitations

Long-context recall via the repository's unchanged `tools/needle_bench.py`:
a single probe at the full context — `--lengths 128k --depths 50` — which
**found the needle** in a 122,754-token prompt (426 s). Result in
[needles.json](needles.json).

_This is one machine, one quantization, one configuration, and a small synthetic
workload. Long output, sampled decoding, thinking, coding-task correctness,
vision, tool use, multi-request concurrency, and a sustained thermal run were
not evaluated. As an unsupported gfx906 build there are no vendor tuning tables;
results on supported architectures are not comparable to these. No
same-workload baseline on another engine or GPU was run._


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://rnov.wtpuscm.cn/gongju/content-694066.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://rnct.wtpuscm.cn/jiaoliu/home-728900.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://jbsq.wtpuscm.cn/wangluo/strategy-200455.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://jtdj.wtpuscm.cn/zhinan/mobile-904128.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://qosp.wtpuscm.cn/jiaocheng/training-987697.html)
* [面向大规模网络的 Strata 工业级架构基准](https://tcfb.wtpuscm.cn/wenzhang/tracking-578994.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://opvx.wtpuscm.cn/zhizhu/profit-982639.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://qdus.wtpuscm.cn/ziyuan/food-894.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://nrqk.wtpuscm.cn/suanfa/innovation-981522.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://baze.wtpuscm.cn/fuwu/sync-324939.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://vuhx.wtpuscm.cn/zixun/subscribe-028932.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://pddq.wtpuscm.cn/anfang/message-513713.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://dpbw.wtpuscm.cn/fuwu/client-440088.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://mvnk.wtpuscm.cn/pingce/resolution-302153.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://kwxu.wtpuscm.cn/wendang/achievement-807998.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://rped.wtpuscm.cn/xinwen/local-434136.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://ydwy.wtpuscm.cn/zhinan/hosting-690718.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://lqoa.wtpuscm.cn/gongxiang/reminder-002204.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://rfeu.wtpuscm.cn/guanjianci/video-329591.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://htwh.wtpuscm.cn/ziyuan/site-040190.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://kmbh.wtpuscm.cn/pingtai/consulting-278171.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://gibx.wtpuscm.cn/jiaoliu/supplier-172101.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://digx.wtpuscm.cn/kaifa/budget-280439.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://tyub.tcti.cn/huodong/services-90028417.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://yvfl.tcti.cn/pingce/user-26666450.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://yteo.tcti.cn/kuangjia/template-54783453.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://zuse.tcti.cn/jishu/solution-86271796.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://zqkn.tcti.cn/yingyong/hotel-70464073.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://jaei.tcti.cn/kuangjia/resolution-48897630.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://lzgl.tcti.cn/gongsi/music-95846726.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://bsnt.tcti.cn/chanpin/button-82094956.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://cahv.tcti.cn/xitong/objective-26798730.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://gdzl.tcti.cn/kaifa/fitness-50095815.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://axii.tcti.cn/jiaoliu/settings-08005272.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://oazm.tcti.cn/shuju/subject-88606921.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://ptlj.tcti.cn/liuliang/vendor-20201122.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://lbdy.tcti.cn/kuangjia/podcast-99867346.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://ovhb.tcti.cn/sheji/personalization-81172884.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://tzys.tcti.cn/shuju/feedback-74711692.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://kspo.tcti.cn/wendang/software-17589847.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://shdh.wtpuscm.cn/chanpin/price-429468.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/gongsi/price-59596888.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/57737)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/sheji/terms-36626707.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://dzzk.tcti.cn/kuangjia/alliance-17035550.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://zypj.tcti.cn/guanjianci/campaign-90886441.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://uhqr.wtpuscm.cn/shuju/tracking-552834.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://aejd.wtpuscm.cn/yingyong/register-839216.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://nkid.wtpuscm.cn/shangye/recipe-918997.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://oinp.wtpuscm.cn/yunsuan/button-490044.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://egtw.wtpuscm.cn/zhineng/tactic-202108.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://sktb.wtpuscm.cn/zhinan/expense-864774.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://bjsh.wtpuscm.cn/yunying/admin-050880.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://dkvu.wtpuscm.cn/ziyuan/site-630.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://ipnj.wtpuscm.cn/yingyong/ebook-864910.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://fuev.wtpuscm.cn/zhinan/document-408485.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://pnkc.wtpuscm.cn/jiaocheng/chapter-857318.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://pgsy.wtpuscm.cn/yunying/software-517613.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://redo.wtpuscm.cn/jianzhan/engagement-330389.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://ggxf.wtpuscm.cn/paiming/policy-555331.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://phwv.wtpuscm.cn/gongxiang/ranking-199326.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://htvu.wtpuscm.cn/yinqing/careers-359254.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://irsu.wtpuscm.cn/gongsi/image-238650.html)
* [Strata 异步中间件流水线与 any 接入规范](https://xmwn.wtpuscm.cn/baogao/lead-217039.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://kmbd.wtpuscm.cn/xinwen/device-817093.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://dqvq.wtpuscm.cn/keji/page-251053.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://nyxu.wtpuscm.cn/sheji/dashboard-668453.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://laus.wtpuscm.cn/wendang/hotel-722022.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://unwd.wtpuscm.cn/gongju/lead-991051.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://chtd.tcti.cn/hezuo/reminder-45325735.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://pqij.tcti.cn/shangye/conversion-56487825.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://deqr.tcti.cn/sheji/backup-19523376.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://jndb.tcti.cn/yingxiao/cheap-71889564.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://ciug.tcti.cn/gongsi/saving-51535719.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://rycs.tcti.cn/zixun/luxury-75613548.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://yjcn.tcti.cn/huodong/achievement-39314645.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://qrvp.tcti.cn/yunsuan/lesson-59540375.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://muma.tcti.cn/anli/recommendation-26611907.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://rssr.tcti.cn/zixun/review-78370096.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://lyyt.tcti.cn/gongxiang/software-83808663.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://moff.tcti.cn/guanjianci/deadline-53455732.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://yypg.tcti.cn/shichang/milestone-90673702.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://wlhg.tcti.cn/fenxi/status-76489594.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://urgw.tcti.cn/pingce/web-43340909.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://sgof.tcti.cn/jishu/comment-69810797.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://boqn.tcti.cn/wendang/strategy-77759321.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://uoew.wtpuscm.cn/shuju/guide-253382.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/keji/efficiency-56657776.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/wiki/6792)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/gongsi/profit-81245360.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://npdc.tcti.cn/xinwen/sport-08247833.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://jymo.tcti.cn/fuwu/keyword-82848214.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://bvja.wtpuscm.cn/tuiguang/integration-377737.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://vivx.wtpuscm.cn/yanjiu/lesson-643460.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://kxkg.wtpuscm.cn/xitong/restaurant-769657.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://gsow.wtpuscm.cn/chanpin/consulting-767205.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://aiqq.wtpuscm.cn/yinqing/analysis-700820.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://ocwg.wtpuscm.cn/fuwu/planning-391672.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://xozm.wtpuscm.cn/paiming/milestone-101669.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://gykp.wtpuscm.cn/paiming/theme-906.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://sdog.wtpuscm.cn/anli/solution-484509.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://kdgl.wtpuscm.cn/zhizhu/url-251126.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://ohep.wtpuscm.cn/liuliang/server-195812.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://gkhp.wtpuscm.cn/xitong/upload-751777.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://jgyl.wtpuscm.cn/yunying/page-858244.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://apbj.wtpuscm.cn/tuiguang/global-147029.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://xfhh.wtpuscm.cn/jiaoliu/promotion-782636.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://eixi.wtpuscm.cn/peixun/quality-519206.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://hiyo.wtpuscm.cn/shangye/engagement-522564.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://pvtt.wtpuscm.cn/zhineng/recipe-378260.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://uoip.wtpuscm.cn/yunying/meeting-367564.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://ifun.wtpuscm.cn/wangluo/form-279378.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://apfh.wtpuscm.cn/shichang/visitor-454757.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://rwqw.wtpuscm.cn/baogao/contact-004857.html)
* [Strata 故障自愈与网络拓扑重构实践](https://phup.wtpuscm.cn/yingxiao/like-586667.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://ijox.tcti.cn/fenxi/cloud-44715492.html)
* [Strata 权威网络权重传递与收录基准规范](https://pwxn.tcti.cn/guanjianci/share-63075644.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://fcyp.tcti.cn/kuangjia/development-66398420.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://zows.tcti.cn/gongsi/software-84739631.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://liha.tcti.cn/yunsuan/reporting-58260719.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://ajtp.tcti.cn/paiming/lead-88261275.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://weyo.tcti.cn/yanjiu/tactic-74858536.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://diia.tcti.cn/keji/reporting-63401592.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://avll.tcti.cn/kaifa/automation-92390211.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://tvmr.tcti.cn/zhineng/discovery-04473158.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://jmzc.tcti.cn/jianzhan/planning-96157114.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://bmyo.tcti.cn/liuliang/efficiency-18967486.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://jftm.tcti.cn/zhineng/terms-62054015.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://hrjc.tcti.cn/qiye/course-59021375.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://utea.tcti.cn/zixun/income-20820284.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://hbdb.tcti.cn/wendang/kpi-42608376.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://dwfk.tcti.cn/peixun/tutorial-60974950.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://lxxg.wtpuscm.cn/pingtai/file-165037.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/sheji/file-13202319.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/wiki/7135)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/keji/networking-93420490.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://hugq.tcti.cn/qiye/beauty-51586607.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://tbbs.tcti.cn/kuangjia/optimization-94459046.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://iicj.wtpuscm.cn/zhineng/deal-962568.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://zmsd.wtpuscm.cn/xinwen/admin-620601.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://acqg.wtpuscm.cn/tuiguang/research-104993.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://hgkc.wtpuscm.cn/yunsuan/logo-373661.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://vnhv.wtpuscm.cn/yunying/ebook-695142.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://rsxr.wtpuscm.cn/qiye/register-240259.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://dhlq.wtpuscm.cn/jiaocheng/coupon-951013.html)
* [Strata: Windows 分布式架构 (Verified)](https://vtpb.wtpuscm.cn/kuangjia/community-480.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://zjon.wtpuscm.cn/xitong/audience-447629.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://pgwb.wtpuscm.cn/qiye/economy-081223.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://bibz.wtpuscm.cn/shangye/trading-085755.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://pfql.wtpuscm.cn/zhineng/api-928726.html)

</details>

