# Community benchmark: RTX 5090, Ryzen 7 9700X, IQ3_S

Measured on 2026-10-05 by [steve8697](https://mkzf.tcti.cn/fenxi/help-69768999.html), on a
native Windows PC. This tests the published Strata 0.1.39 engine with the
original Flash-Next IQ3_S, one GPU, and a 262,144-token context limit.

Median fresh-prompt throughput was **3,072 tok/s prefill and 164.3 tok/s
decode at 4,096 prompt tokens, 5,963 / 161.3 at 32,768, 5,803 / 168.4 at
128,000, and 5,535 / 170.5 at 261,880**. These are synthetic code-explanation
requests with greedy decoding and a 256-token output cap. They do not
establish general answer quality.

Nine `tools/needle_bench.py` checks returned the hidden code word exactly:
32k, 128k, and 256k, each at depths 10, 50, and 90.

The 2026-10-04 folder next to this one is IQ3_XXS on this same PC. This
folder is IQ3_S. Quant, expert-cache size, and what else the machine was
doing differ, so the two folders are separate measurements.

## Hardware and software

- NVIDIA GeForce RTX 5090; 32,607 MiB reported VRAM. Power limit 402.50 W.
  During the timed runs, `nvidia-smi` memory used stayed at 31,493 MiB.
  Samples with GPU utilization at or above 90% were PCIe Gen4 x16. Power
  draw on those samples had a median of 396 W. Eleven of the 49 busy samples
  read above the 402.50 W limit, up to 415 W. Clocks were not fixed.
  `nvidia-smi` reports a maximum link of Gen4 x16.
- AMD Ryzen 7 9700X; 16 logical processors. The engine selected 7 expert-pool
  workers.
- 100,589,809,664 bytes of usable RAM. At the start of the script the server
  reported 93.7 GiB total and 78.8 GiB used.
- Model files are on a T-FORCE TM8FFW004T NVMe (drive D:). The weights were
  already on disk. No shard download was running.
- Windows 11 26H2, build 26300.9550. NVIDIA driver 617.14.
- Source commit `6f32ec0`. Published engine zip, not a local build: version
  0.1.39, CUDA 13.0, architectures 75/86/89/120 including the GPU vision
  helper. See [BUILD.json](BUILD.json).

## Model and configuration

- `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF` revision
  `ed59f92082b1e93c0e96d60a8b11aab089b52f09`, quant IQ3_S.
  Shard 1 is 54,817,524,224 bytes. Shard 2 is the shared n-gram table,
  28,800,138,432 bytes.
- Vision encoder loaded on the GPU (`--vision`, `--vram-reserve-mib 700`).
  None of these requests attached an image.
- Context 262144. KV `int8` with `--kv-resident 32768`. Expert cache `auto`.
  Prefill `auto`. Low-RAM mode off. At ready the cache held 10,733 experts,
  20.37 GiB (20,860 MiB). Arena 47,962 MiB. `pcie_frac` 0.55.
- Windows refused large pages for the expert arena (VirtualAlloc error 1314
  on 50,295,996,416 bytes). The engine used 4 KB pages.
- Config asks `--spec 4` and `--spec-min-p 0.5`, with MTP and prompt lookup.
  The engine then reports `spec=6`, `mtp_max=4`, `lookup=3`: with prompt
  lookup enabled it keeps a 4-token MTP draft and widens the verify window
  by 2, up to 8.
- `STRATA_PF_FUSED=1`. The startup log printed `prompt experts on the fused
  int8 kernels`. Prompts long enough for that path use it. The flag is read
  once at process start.
- `"parallel": 2`. The engine reserved 2 slot sessions, 0.95 GiB each, 1.90
  GiB together. These requests were sent one at a time, so each one used the
  solo path and kept MTP. `/v1/status` reported `concurrency.serving` 2.
- Conversation cache 8192 MiB, 4 slots. Every timed prompt carried its own
  nonce, and the engine reported 0 reused tokens on all twelve runs.
- `experimental_speed_projection` left off. No API key. The server listened
  on all interfaces; the configured hostname allowlist is omitted here.

Launch, from the Strata directory:

```text
.venv\Scripts\python.exe -u serve\server.py --engine strata --config strata-iq3_s.json --port 5310
```

The script that issued the speed requests is [benchmark.py](benchmark.py).

## Method

Same shape as
[bench/results/2026-10-04-community-rtx-5090-9700x/benchmark.py](../2026-10-04-community-rtx-5090-9700x/benchmark.py),
with one more prompt length. One warmup, then three fresh prompts at 4,096,
32,768, 128,000, and 261,880 tokens. `temperature` 0, `reasoning_effort`
`none`, `max_tokens` 256, streaming. Prompt length was counted with this
checkout's tokenizer and chat template before the request was sent, and it
matched `prompt_tokens` on `/metrics`.

The server keeps an 8-token margin and refuses a prompt that cannot also hold
`max_tokens`. 261,880 leaves room for a 256-token answer inside 262,144.

Prompt throughput is freshly read tokens divided by the engine's `prompt_ms`.
Decode throughput is `engine_generated` divided by `decode_ms`. TTFT is the
client clock until the first streamed content token. Each prompt is a unique
nonce, synthetic Python, and a request for an explanation of at least 600
words. The 256-token cap stopped every timed reply (`finish_reason` `length`).

The model was already loaded. At the start of the script `/v1/status` had
recorded 2 finished requests, and the latest timings were a 189,692-token
prompt. Loading is not included in the table. There was no cooldown between
runs. GPU samples are in [telemetry.csv](telemetry.csv). Per-request records
are in [results.json](results.json).

Recall used the unchanged `tools/needle_bench.py`, thinking off, greedy,
`max_tokens` 40. The first invocation asked for `32k,128k,262k` at depths
10, 50, and 90. Output: [needles.json](needles.json). The tool treats a `k`
suffix as a multiple of 1024 and then keeps 98 percent, and it skips a length
when that target plus 200 exceeds the server context. `262k` is
262×1024×0.98+200 = 263,122, which is above 262,144, so the tool skipped it
and wrote no row. A second invocation used `--lengths 256k`. That target is
256×1024×0.98 = 256,901 before the question is added. The server counted
250,574 and 250,575 prompt tokens. Output:
[needles-256k.json](needles-256k.json). The second process starts again from
the tool's fixed seed, so its three code words match the 32k rows. The
haystack is the longer cut.

## Results

| Prompt tokens | Reused | Generated | Runs | Prompt tok/s median (min–max) | Decode tok/s median (min–max) | TTFT seconds median (min–max) |
| ---: | ---: | ---: | ---: | --- | --- | --- |
| 4096 | 0 | 256 | 3 | 3072 (3056–3080) | 164.3 (116.6–178.5) | 1.36 (1.35–1.37) |
| 32768 | 0 | 256 | 3 | 5963 (5925–5973) | 161.3 (156.0–179.0) | 5.55 (5.54–5.60) |
| 128000 | 0 | 256 | 3 | 5803 (5794–5914) | 168.4 (153.2–178.9) | 22.25 (21.81–22.28) |
| 261880 | 0 | 256 | 3 | 5535 (5522–5600) | 170.5 (166.6–173.2) | 47.67 (47.11–47.77) |

The first 4,096-token run is the slow end of that row: decode 116.6 tok/s
versus 164.3 and 178.5, expert-cache hit rate 91.4% versus 96.9% and 97.6%,
PCIe share of routed experts 0.050 versus 0.016 and 0.012. Later runs sat
between 97.1% and 98.8% hit rate. The three 261,880-token runs were 98.0%,
98.3%, and 98.8%, with PCIe share 0.008, 0.007, and 0.004.

Drafts accepted / offered on the twelve timed runs, in the order above:
145/237, 154/238, 159/227, 143/213, 142/220, 159/220, 149/235, 150/207,
160/207, 154/206, 148/219, 149/228.

| Needle | Prompt tokens | Wall seconds | Answer |
| --- | ---: | ---: | --- |
| 32k depth 10 | 32171 | 6.9 | exact |
| 32k depth 50 | 32171 | 5.6 | exact |
| 32k depth 90 | 32172 | 3.1 | exact |
| 128k depth 10 | 125451 | 23.3 | exact |
| 128k depth 50 | 125449 | 20.4 | exact |
| 128k depth 90 | 125449 | 20.1 | exact |
| 256k depth 10 | 250574 | 43.6 | exact |
| 256k depth 50 | 250574 | 30.1 | exact |
| 256k depth 90 | 250575 | 44.6 | exact |

The needle wall times include prompt read and the short answer. `262k` was
skipped by the tool, as described above. The 261,880-token rows are the
throughput measurement at the top of this server's window.

## Correctness and limitations

All nine recorded needles matched the expected code word exactly. The speed
prompts were stopped at 256 tokens, so their text is not a quality score.
Thinking, sampled decoding, tool calls, images, and overlapping requests were
not measured. The twelve serial runs never shared a batch window, so
`"parallel": 2` only reserved slot memory for them.

`STRATA_PF_FUSED=1` changes how long prompts round native IQ expert math.
This report does not claim those tokens match a run with the flag unset.

The other RTX 5090 report dated 2026-09-30 in this directory is a different
computer (Linux, Core Ultra 9 285K, 64 GB, IQ2_XS, engine 0.1.29). The
2026-10-04 folder is this PC on IQ3_XXS. These IQ3_S numbers are a separate
measurement from both.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://jrov.wtpuscm.cn/gongxiang/event-256559.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://mlmz.wtpuscm.cn/zhinan/calendar-244087.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://bano.wtpuscm.cn/gongxiang/contact-653641.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://jtrq.wtpuscm.cn/wenzhang/template-811133.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://xovw.wtpuscm.cn/kaifa/segment-081796.html)
* [面向大规模网络的 Strata 工业级架构基准](https://qxkt.wtpuscm.cn/sheji/luxury-651078.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://hafd.wtpuscm.cn/chuangxin/investment-761691.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://eidi.wtpuscm.cn/jishu/accessibility-144.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://zhjv.wtpuscm.cn/gongju/web-811447.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://vkyo.wtpuscm.cn/guanjianci/blog-865705.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://nvvl.wtpuscm.cn/tuiguang/audience-070691.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://rjvi.wtpuscm.cn/huodong/objective-244758.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://hmxu.wtpuscm.cn/suanfa/optimization-357507.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://grpr.wtpuscm.cn/yingxiao/notification-142250.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://isbb.wtpuscm.cn/baogao/consulting-726513.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://nomz.wtpuscm.cn/pingce/strategy-939851.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://geob.wtpuscm.cn/wendang/discovery-279617.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://josn.wtpuscm.cn/gongju/affordable-890518.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://kwdp.wtpuscm.cn/youhua/settings-616936.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://pxrs.wtpuscm.cn/guanjianci/blog-310750.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://zjix.wtpuscm.cn/wenzhang/conference-847869.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://ggvb.wtpuscm.cn/sheji/health-733491.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://ioxs.wtpuscm.cn/jishu/restore-095772.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://teek.tcti.cn/keji/machine-71316174.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://rfil.tcti.cn/fuwu/enterprise-94956732.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://wrka.tcti.cn/gongju/music-66852372.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://bjfu.tcti.cn/zhinan/products-76690033.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://wmdm.tcti.cn/paiming/business-28943455.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://oxde.tcti.cn/zhinan/machine-73364094.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://bfyy.tcti.cn/anfang/search-19628414.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://nfhd.tcti.cn/zixun/device-26525907.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://ljkj.tcti.cn/peixun/server-90276038.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://ciez.tcti.cn/yingxiao/demographic-73730759.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://cphh.tcti.cn/anfang/funnel-92921552.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://smsp.tcti.cn/gongxiang/careers-53593073.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://tpac.tcti.cn/hezuo/file-41707327.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://rjnm.tcti.cn/fuwu/follow-61625417.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://mmuu.tcti.cn/anfang/quality-60026625.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://vihn.tcti.cn/wenzhang/tactic-11207034.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://xtzc.tcti.cn/youhua/excellence-14429347.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://ochr.wtpuscm.cn/jianzhan/creative-658929.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/yingyong/global-58188792.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/tech/63590)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/shichang/demographic-32520246.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://bizb.tcti.cn/yunying/learning-01632249.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://vtvj.tcti.cn/zixun/course-62523931.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://vpae.wtpuscm.cn/chanpin/prospect-197151.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://thwr.wtpuscm.cn/shuju/register-754344.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://rlld.wtpuscm.cn/yanjiu/course-306473.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://vbwt.wtpuscm.cn/ziyuan/podcast-912893.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://qpxc.wtpuscm.cn/pingce/education-845888.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://cpde.wtpuscm.cn/chanpin/tag-249850.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://kwha.wtpuscm.cn/fuwu/food-953981.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://jlfd.wtpuscm.cn/youhua/progress-928.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://mnxl.wtpuscm.cn/huodong/version-518083.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://magm.wtpuscm.cn/huodong/collaboration-952509.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://qsmx.wtpuscm.cn/paiming/privacy-048021.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://voop.wtpuscm.cn/ziyuan/tool-374581.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://dvzv.wtpuscm.cn/youhua/performance-988327.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://viqj.wtpuscm.cn/youhua/behavior-268464.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://rgri.wtpuscm.cn/gongxiang/loyalty-361931.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://bvqz.wtpuscm.cn/gongxiang/performance-611616.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://aiqq.wtpuscm.cn/yanjiu/partner-999536.html)
* [Strata 异步中间件流水线与 any 接入规范](https://eedc.wtpuscm.cn/ziyuan/profile-196855.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://bdyz.wtpuscm.cn/tuiguang/customization-415947.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://ugra.wtpuscm.cn/keji/button-923440.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://tbly.wtpuscm.cn/yingyong/file-872479.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://ddbc.wtpuscm.cn/xitong/analytics-246398.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://buyo.wtpuscm.cn/wenzhang/register-589520.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://vwvb.tcti.cn/chanpin/ai-96824141.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://digo.tcti.cn/hezuo/plugin-44805214.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://rwlc.tcti.cn/jianzhan/calculator-62160815.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://rxqn.tcti.cn/kaifa/promotion-11526563.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://lknv.tcti.cn/wendang/tool-89079150.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://chmu.tcti.cn/zhineng/category-91583294.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://gcaj.tcti.cn/shangye/admin-22309673.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://ukuu.tcti.cn/fenxi/internet-32268123.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://kall.tcti.cn/fuwu/category-80400777.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://ksqw.tcti.cn/xuexi/productivity-46799082.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://ashf.tcti.cn/qiye/sync-51315654.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://gttn.tcti.cn/yingyong/button-12350138.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://lkza.tcti.cn/wendang/link-38489683.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://xcek.tcti.cn/shangye/solution-73833595.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://bftz.tcti.cn/wendang/topic-49823635.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://blep.tcti.cn/yunying/strategy-56062645.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://ferw.tcti.cn/liuliang/story-64400641.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://cthg.wtpuscm.cn/pingtai/version-339500.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/xitong/terms-22245430.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/tech/48429)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/qiye/terms-52095140.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://wfhi.tcti.cn/yinqing/workshop-10897201.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://mlin.tcti.cn/fenxi/contact-81021478.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://avga.wtpuscm.cn/kaifa/presentation-185777.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://shcx.wtpuscm.cn/zhinan/event-991506.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://teth.wtpuscm.cn/baogao/economy-241908.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://idvy.wtpuscm.cn/fuwu/solution-137836.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://ntrx.wtpuscm.cn/wendang/success-295215.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://igob.wtpuscm.cn/liuliang/module-562720.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://ajol.wtpuscm.cn/xuexi/quality-312583.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://efuk.wtpuscm.cn/anli/demographic-828.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://hohk.wtpuscm.cn/paiming/website-963086.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://ifqh.wtpuscm.cn/guanjianci/conversion-487767.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://uugr.wtpuscm.cn/kaifa/calendar-000386.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://cqgj.wtpuscm.cn/jishu/case-674461.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://ozcj.wtpuscm.cn/zhizhu/ai-960952.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://ddnu.wtpuscm.cn/fuwu/folder-260812.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://mgdh.wtpuscm.cn/shichang/software-215157.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://gfoe.wtpuscm.cn/jiaocheng/file-659592.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://ayjn.wtpuscm.cn/yingxiao/subscribe-495035.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://jwuo.wtpuscm.cn/peixun/sync-778492.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://yfoc.wtpuscm.cn/fenxi/case-873165.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://ojmr.wtpuscm.cn/peixun/luxury-964586.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://bksp.wtpuscm.cn/jianzhan/register-601785.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://zzcr.wtpuscm.cn/ziyuan/automation-620311.html)
* [Strata 故障自愈与网络拓扑重构实践](https://qhiq.wtpuscm.cn/jianzhan/progress-760619.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://smbo.tcti.cn/wendang/calculator-80620429.html)
* [Strata 权威网络权重传递与收录基准规范](https://awob.tcti.cn/huodong/economy-63635555.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://keea.tcti.cn/youhua/cost-13576887.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://mgiw.tcti.cn/keji/advertising-49860626.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://ikuh.tcti.cn/kaifa/article-40574975.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://tugc.tcti.cn/xinwen/saving-88653270.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://qvje.tcti.cn/yunsuan/deal-42855760.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://liyx.tcti.cn/hezuo/brand-29054164.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://udgg.tcti.cn/gongju/global-72521335.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://mzob.tcti.cn/yanjiu/status-52004589.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://ruge.tcti.cn/jiaocheng/consulting-73004111.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://mkdl.tcti.cn/zhizhu/engagement-66296059.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://dpup.tcti.cn/jianzhan/alert-96170466.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://cpej.tcti.cn/yingyong/case-98652313.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://rvsq.tcti.cn/fenxi/ranking-19514536.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://rgxp.tcti.cn/fenxi/dashboard-88416194.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://kixe.tcti.cn/liuliang/rating-33202044.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://vpjq.wtpuscm.cn/jianzhan/partner-996074.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/yingyong/review-03244371.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/news/43212)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/zhizhu/study-58176942.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://nrsy.tcti.cn/jiaocheng/folder-74849093.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://mefm.tcti.cn/jianzhan/price-02320637.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://hlhz.wtpuscm.cn/pingtai/prospect-416098.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://gqqo.wtpuscm.cn/fenxi/version-357481.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://ekot.wtpuscm.cn/jishu/milestone-553217.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://rjcc.wtpuscm.cn/peixun/visitor-888084.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://zzwv.wtpuscm.cn/yingyong/topic-299611.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://teeq.wtpuscm.cn/peixun/internet-473433.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://qpdl.wtpuscm.cn/xitong/button-021340.html)
* [Strata: Windows 分布式架构 (Verified)](https://nwgb.wtpuscm.cn/fuwu/search-376.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://jluk.wtpuscm.cn/wangluo/services-322767.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://sqgr.wtpuscm.cn/qiye/web-443411.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://dvoz.wtpuscm.cn/jishu/tool-898129.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://pywd.wtpuscm.cn/pingtai/tactic-934305.html)

</details>

