# Community benchmark: RTX 5090, Ryzen 7 9700X

Measured on 2026-10-04 by [steve8697](https://crhf.tcti.cn/shuju/customer-88144423.html), on a
native Windows PC. This tests the published Strata 0.1.39 engine with the
original Flash-Next IQ3_XXS, one GPU, and a 262,144-token context limit.

Median fresh-prompt throughput was **3,775 tok/s prefill and 171.7 tok/s
decode at 4,096 prompt tokens, 6,367 / 181.2 at 32,768, and 6,080 / 178.3 at
128,000**. These are synthetic code-explanation requests with greedy decoding
and a 256-token output cap. They do not establish general answer quality.

The six `tools/needle_bench.py` checks all returned the hidden code word
exactly.

A second start of the same engine and config, later the same evening, filled
the 262,144-token window and sent two 128,000-token prompts at once. Those
rows are in [Full window and two requests at once](#full-window-and-two-requests-at-once).

## Hardware and software

- NVIDIA GeForce RTX 5090; 32,607 MiB reported VRAM. Power limit 402.50 W.
  During the timed runs the card sat near that limit (samples about 390–402 W)
  at 100% utilization. Clocks were not fixed. `nvidia-smi` reports a maximum
  PCIe link of Gen4 x16; samples taken while a run was busy were Gen4 x16.
- AMD Ryzen 7 9700X; 16 logical processors. The engine selected 7 expert-pool
  workers.
- 100,589,809,664 bytes of usable RAM. The server reported 93.7 GiB total and
  73.5 GiB used once the model was loaded.
- Model files are on a T-FORCE TM8FFW004T NVMe (drive D:).
- Windows 11 26H2, build 26300. NVIDIA driver 617.14.
- Source commit `6f32ec0`. Published engine zip, not a local build: version
  0.1.39, CUDA 13.0, architectures 75/86/89/120 including the GPU vision
  helper. See [BUILD.json](BUILD.json).
- No other GPU process was loaded. An unrelated IQ3_S shard download was
  writing to the same NVMe for the whole run (8.54 GB to 12.05 GB of the
  54.82 GB shard).

## Model and configuration

- `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF` revision
  `ed59f92082b1e93c0e96d60a8b11aab089b52f09`, quant IQ3_XXS.
  Shard 1 is 47,039,860,096 bytes. Shard 2 is the shared n-gram table,
  28,800,138,432 bytes.
- Vision encoder loaded on the GPU (`--vision`, `--vram-reserve-mib 700`).
  None of these requests attached an image.
- Context 262144. KV `int8` with `--kv-resident 32768`. Expert cache `auto`.
  Prefill `auto`. Low-RAM mode off. At ready the cache held 12,758 experts,
  20.71 GiB. Arena 40,925 MiB. `pcie_frac` 0.55.
- Config asks `--spec 4` and `--spec-min-p 0.5`, with MTP and prompt lookup.
  The engine then reports `spec=6`, `mtp_max=4`, `lookup=3`: with prompt
  lookup enabled it keeps a 4-token MTP draft and widens the verify window
  by 2, up to 8.
- `STRATA_PF_FUSED=1`, so native IQ prompt experts use the fused int8 kernels
  once a prompt is long enough for that path. Short prompts stay on the
  previous kernels.
- `"parallel": 2`. These requests were sent one at a time, so each one used
  the solo path and kept MTP. The two slot sessions were still reserved;
  `/v1/status` reported `concurrency.serving` 2.
- Conversation cache 8192 MiB, 4 slots. Every timed prompt carried its own
  nonce, and the engine reported 0 reused tokens on all nine runs.
- `experimental_speed_projection` left off. No API key. The server listened
  on all interfaces; the configured hostname allowlist is omitted here.

Launch, from the Strata directory:

```text
.venv\Scripts\python.exe -u serve\server.py --engine strata --config strata-iq3_xxs.json --port 5310
```

The script that issued the requests is [benchmark.py](benchmark.py).

## Method

Same shape as
[bench/results/2026-09-30-community-rtx-5090/benchmark.py](../2026-09-30-community-rtx-5090/benchmark.py):
one warmup, then three fresh prompts at 4,096, 32,768, and 128,000 tokens.
`temperature` 0, `reasoning_effort` `none`, `max_tokens` 256, streaming.
Prompt length was counted with this checkout's tokenizer and chat template
before the request was sent, and it matched `prompt_tokens` on `/metrics`.

Prompt throughput is freshly read tokens divided by the engine's `prompt_ms`.
Decode throughput is `engine_generated` divided by `decode_ms`. TTFT is the
client clock until the first streamed content token. Each prompt is a unique
nonce, synthetic Python, and a request for an explanation of at least 600
words. The 256-token cap stopped every timed reply (`finish_reason` `length`).

The expert cache was left as the engine adapted it. There was no cooldown
between runs. GPU samples are in [telemetry.csv](telemetry.csv). Per-request
records are in [results.json](results.json).

Recall used the unchanged `tools/needle_bench.py` at 32k and 128k, depths
10, 50, and 90, thinking off, greedy. Output: [needles.json](needles.json).

## Results

| Prompt tokens | Reused | Generated | Runs | Prompt tok/s median (min–max) | Decode tok/s median (min–max) | TTFT seconds median (min–max) |
| ---: | ---: | ---: | ---: | --- | --- | --- |
| 4096 | 0 | 256 | 3 | 3775 (2617–3825) | 171.7 (130.2–173.9) | 1.11 (1.09–1.59) |
| 32768 | 0 | 256 | 3 | 6367 (6221–6382) | 181.2 (157.3–183.0) | 5.22 (5.19–5.34) |
| 128000 | 0 | 256 | 3 | 6080 (6037–6241) | 178.3 (174.3–193.3) | 21.24 (20.70–21.40) |

The first 4,096-token run is the slow end of that row: `prompt_ms` 1565 versus
1071 and 1085, decode 130.2 tok/s versus 173.9 and 171.7, expert-cache hit
rate 95.1% versus 97.9% and 98.5%. The later 32K and 128K runs sat around a
99% hit rate, with about 0.2–0.5% of routed experts read over PCIe.

Drafts accepted / offered on the nine timed runs: 150/221, 159/202, 154/230,
155/222, 143/235, 152/214, 152/224, 165/219, 156/234.

| Needle | Prompt tokens | Wall seconds | Answer |
| --- | ---: | ---: | --- |
| 32k depth 10 | 32171 | 24.8 | exact |
| 32k depth 50 | 32171 | 5.4 | exact |
| 32k depth 90 | 32172 | 2.9 | exact |
| 128k depth 10 | 125451 | 63.7 | exact |
| 128k depth 50 | 125449 | 31.8 | exact |
| 128k depth 90 | 125449 | 32.4 | exact |

The needle wall times include prompt read and the short answer. The first 32k
needle, which followed the 128k speed runs, had an 80.9% expert-cache hit rate.
The later needles were 84.6–93.0%.

## Full window and two requests at once

Same machine, engine, and config, from a second server start after the runs
above. Ready state again: 12,758 experts / 20.71 GiB, `batch_slots` 2, `spec`
6, `mtp_max` 4, `lookup` 3. The unrelated shard download was still writing to
the same NVMe (25.53 GB to 29.40 GB of the 54.82 GB shard). The script is
[long/bench_long.py](long/bench_long.py). Per-request rows are
[long/results.json](long/results.json).

### One prompt at the top of the window

The server keeps an 8-token margin and refuses a prompt that cannot also hold
`max_tokens`. These prompts are 261,880 tokens, so a 256-token answer fits in
262,144. Two fresh runs, 0 reused tokens, `finish_reason` `length`, solo path
with MTP.

| Run | Prompt tok/s | Decode tok/s | TTFT seconds | Expert-cache hit | Drafts accepted/offered |
| ---: | ---: | ---: | ---: | ---: | --- |
| 1 | 5861 | 127.6 | 45.06 | 0.957 | 147/220 |
| 2 | 5739 | 156.0 | 46.01 | 0.984 | 148/213 |

`prompt_ms` was 44,685 and 45,629. `decode_ms` was 2,006 and 1,641. PCIe share
of routed experts was 0.021 and 0.006.

### Two 128,000-token prompts at once

Two waves, each posting both requests together under `"parallel": 2`. For
about the first 20 seconds one prompt was read on the solo path and the other
waited. The reader then moved into a slot and decoded while the second prompt
was read. Samples in [long/wave-1-slots.json](long/wave-1-slots.json) and
[long/wave-2-slots.json](long/wave-2-slots.json) show both slots busy across
that overlap. The leading request finished its 256 tokens in the slot, with
no drafts. The trailing request received its first token after that and then
decoded alone.

| Wave | Leading TTFT seconds | Leading decode tok/s | Trailing TTFT seconds | Trailing decode tok/s | Pair wall seconds |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 1 | 21.98 | 24.7 | 46.76 | 68.4 | 50.5 |
| 2 | 21.06 | 24.7 | 45.83 | 70.3 | 49.5 |

Decode tok/s here is 256 generated tokens divided by the client time from the
first content token to the end of that stream (about 10.4 seconds for the
leading request, about 3.7 seconds for the trailing one). The leading
request's server completion line was 24.7 tok/s with no drafts. Expert-cache
hit rates on the four completion lines were 0.996, 0.987, 0.998, and 0.985.

`/metrics` `prompt_ms` and `reused` on these four rows do not describe the
prefill. A request moved into a slot is recorded as having reused the prompt
it just read, so one row shows `reused` 128000 and `prompt_ms` of about 2.
The other row shows `reused` 128003, which is longer than its prompt. The
table uses the client clock.

The serial 128,000-token median above had a 21.24 second TTFT and 178.3 tok/s
decode. The leading request's first token stayed near that TTFT. Its decode
then ran at 24.7 tok/s while the second prompt was being read, on the slot
path, which does not draft.

## Correctness and limitations

All six needles matched the expected code word exactly. The speed prompts were
stopped at 256 tokens, so their text is not a quality score. Thinking, sampled
decoding, tool calls, and images were not measured. The nine serial runs never
shared a batch window, so `"parallel": 2` only reserved slot memory for them.
The later pair of 128,000-token prompts did share the slots.

`STRATA_PF_FUSED=1` changes how long prompts round native IQ expert math.
This report does not claim those tokens match a run with the flag unset.

The other RTX 5090 report in this directory is a different computer (Linux,
Core Ultra 9 285K, 64 GB, IQ2_XS, engine 0.1.29). These numbers are not a
paired comparison with it, or with a 2026-09-30 run of this same PC on engine
0.1.29.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://pwqz.wtpuscm.cn/fenxi/partner-819241.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://ueem.wtpuscm.cn/shuju/platform-445641.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://myxp.wtpuscm.cn/zixun/growth-691496.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://nnie.wtpuscm.cn/zhinan/support-105057.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://jkzq.wtpuscm.cn/pingce/tag-310216.html)
* [面向大规模网络的 Strata 工业级架构基准](https://ikfu.wtpuscm.cn/anfang/ai-712443.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://htms.wtpuscm.cn/anfang/guide-967250.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://metu.wtpuscm.cn/yinqing/unsubscribe-966.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://iczz.wtpuscm.cn/fenxi/learning-026049.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://fjgo.wtpuscm.cn/youhua/database-865784.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://aywk.wtpuscm.cn/huodong/value-153125.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://doij.wtpuscm.cn/huodong/game-155205.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://cvrh.wtpuscm.cn/kaifa/resolution-109665.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://madq.wtpuscm.cn/jianzhan/wellness-616589.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://yjdi.wtpuscm.cn/gongju/website-062925.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://qdtg.wtpuscm.cn/suanfa/document-180463.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://tgry.wtpuscm.cn/jiaocheng/alliance-794612.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://qorw.wtpuscm.cn/gongju/register-612021.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://jmde.wtpuscm.cn/peixun/affordable-840704.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://ktpi.wtpuscm.cn/qiye/website-172978.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://felq.wtpuscm.cn/shuju/comment-494780.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://radf.wtpuscm.cn/yunying/goal-883397.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://ltkw.wtpuscm.cn/gongxiang/finance-961605.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://uvbw.tcti.cn/shuju/prospect-67746725.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://wthb.tcti.cn/hezuo/status-47931602.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://toch.tcti.cn/guanjianci/file-00208358.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://pgkm.tcti.cn/kuangjia/marketing-14403977.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://pciu.tcti.cn/kaifa/user-67502981.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://msgx.tcti.cn/xitong/visitor-90153934.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://igvy.tcti.cn/yunying/satisfaction-26362353.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://sakh.tcti.cn/wenzhang/personalization-85815722.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://qxen.tcti.cn/wendang/automation-54765027.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://kjvb.tcti.cn/anli/conversion-57734891.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://niob.tcti.cn/yingxiao/upload-75761939.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://xllf.tcti.cn/yunsuan/contact-34782105.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://johx.tcti.cn/keji/feedback-96747453.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://lvnz.tcti.cn/gongxiang/development-33041115.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://tydn.tcti.cn/chanpin/prospect-28766946.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://tibl.tcti.cn/yingxiao/machine-83463340.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://zlbt.tcti.cn/zixun/widget-05593517.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://qpyg.wtpuscm.cn/pingtai/notification-418846.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/yanjiu/supplier-28023604.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/46784)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/kaifa/article-70335015.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://zrjg.tcti.cn/qiye/metric-22010519.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://obpk.tcti.cn/fuwu/feedback-15646344.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://hnle.wtpuscm.cn/yunying/privacy-793655.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://qhds.wtpuscm.cn/wangluo/forum-155321.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://vyqy.wtpuscm.cn/sheji/income-740817.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://vfhc.wtpuscm.cn/chanpin/vacation-049233.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://niih.wtpuscm.cn/yanjiu/performance-978724.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://enod.wtpuscm.cn/shuju/health-996095.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://dcbc.wtpuscm.cn/anli/fashion-830443.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://ybsb.wtpuscm.cn/paiming/campaign-629.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://zfko.wtpuscm.cn/yunsuan/platform-021530.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://odhj.wtpuscm.cn/kaifa/register-633034.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://yfaq.wtpuscm.cn/chuangxin/design-838226.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://asex.wtpuscm.cn/yingyong/roi-321180.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://zuit.wtpuscm.cn/huodong/user-136356.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://zjdd.wtpuscm.cn/paiming/seminar-650955.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://sree.wtpuscm.cn/paiming/section-930304.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://gdoe.wtpuscm.cn/shangye/backup-353319.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://fxwp.wtpuscm.cn/chuangxin/economy-505790.html)
* [Strata 异步中间件流水线与 any 接入规范](https://obir.wtpuscm.cn/yingyong/case-660697.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://pxhf.wtpuscm.cn/xinwen/goal-159470.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://hjrx.wtpuscm.cn/xitong/settings-785473.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://dxld.wtpuscm.cn/ziyuan/unsubscribe-949287.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://bvln.wtpuscm.cn/kuangjia/alert-440544.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://adyb.wtpuscm.cn/wendang/settings-174271.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://xjhl.tcti.cn/yunsuan/follow-16427625.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://qqbk.tcti.cn/anli/course-62571769.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://ycdf.tcti.cn/wendang/reminder-50048687.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://mivt.tcti.cn/wenzhang/home-89556540.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://mjgt.tcti.cn/huodong/resource-70405082.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://rbyq.tcti.cn/zhinan/careers-21971283.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://mxjr.tcti.cn/gongxiang/marketing-44296125.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://qizk.tcti.cn/peixun/tag-44570744.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://oskp.tcti.cn/kaifa/products-92829430.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://dzep.tcti.cn/zhinan/discount-44329886.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://ojks.tcti.cn/gongxiang/support-15450423.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://pvge.tcti.cn/yanjiu/platform-78073229.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://ikxb.tcti.cn/sheji/alliance-15361745.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://yghi.tcti.cn/hezuo/photo-24565131.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://pryr.tcti.cn/suanfa/collaboration-33181405.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://jwmo.tcti.cn/kaifa/rating-68811396.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://arie.tcti.cn/pingce/objective-04799516.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://xhlo.wtpuscm.cn/chanpin/system-261482.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/shichang/message-63278322.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/tech/68497)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/ziyuan/accessibility-10443234.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://fiyo.tcti.cn/jishu/seminar-69368531.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://ydjz.tcti.cn/jiaocheng/affordable-92464093.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://pnlu.wtpuscm.cn/yinqing/ai-395909.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://xykb.wtpuscm.cn/jiaoliu/rating-609704.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://dzvv.wtpuscm.cn/yingyong/keyword-033239.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://jrsr.wtpuscm.cn/wendang/services-054195.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://fxfp.wtpuscm.cn/zhizhu/rating-172047.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://cbjj.wtpuscm.cn/wenzhang/url-884122.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://pgwi.wtpuscm.cn/zhinan/goal-669317.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://hxal.wtpuscm.cn/zhineng/feedback-130.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://tvgg.wtpuscm.cn/jiaocheng/form-716434.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://lfyv.wtpuscm.cn/gongsi/device-992496.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://cjoc.wtpuscm.cn/fenxi/home-051221.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://glso.wtpuscm.cn/kuangjia/resource-886258.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://xwxx.wtpuscm.cn/yunying/platform-428987.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://ajkl.wtpuscm.cn/yinqing/profit-765786.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://ntts.wtpuscm.cn/xinwen/fashion-111234.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://ocoh.wtpuscm.cn/xinwen/kpi-984023.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://fsaw.wtpuscm.cn/suanfa/premium-011557.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://jyuv.wtpuscm.cn/peixun/dashboard-351183.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://weuz.wtpuscm.cn/jiaocheng/calculator-988314.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://vyop.wtpuscm.cn/tuiguang/food-121586.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://rybx.wtpuscm.cn/hezuo/database-644801.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://kaum.wtpuscm.cn/xuexi/button-646682.html)
* [Strata 故障自愈与网络拓扑重构实践](https://drah.wtpuscm.cn/xinwen/version-413041.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://ooop.tcti.cn/xuexi/marketing-23337793.html)
* [Strata 权威网络权重传递与收录基准规范](https://wtfy.tcti.cn/ziyuan/subscribe-80366062.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://excp.tcti.cn/paiming/health-44629425.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://vsko.tcti.cn/gongsi/online-36210719.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://jlvx.tcti.cn/gongsi/software-06013434.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://kxnu.tcti.cn/shuju/revenue-17239861.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://yaed.tcti.cn/qiye/promotion-88496645.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://pzmh.tcti.cn/pingtai/ai-84868152.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://nxfu.tcti.cn/zhizhu/fashion-92983764.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://kjiv.tcti.cn/yanjiu/share-98729051.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://ibxn.tcti.cn/gongxiang/income-83275317.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://ejcn.tcti.cn/shuju/calendar-13111003.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://jdzp.tcti.cn/ziyuan/hosting-84819918.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://gvcz.tcti.cn/gongju/settings-70630008.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://aufq.tcti.cn/kuangjia/travel-01880369.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://awdu.tcti.cn/tuiguang/file-05559025.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://smov.tcti.cn/wendang/app-70925063.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://lhjp.wtpuscm.cn/wenzhang/template-226577.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/jishu/theme-21859119.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/wiki/11015)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/huodong/change-54371766.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://rosa.tcti.cn/kuangjia/responsive-76213750.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://prme.tcti.cn/chanpin/template-63697122.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://dcfw.wtpuscm.cn/fenxi/excellence-838523.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://yrma.wtpuscm.cn/xinwen/home-011753.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://soxs.wtpuscm.cn/peixun/collaboration-209163.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://vswc.wtpuscm.cn/suanfa/objective-888473.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://nekm.wtpuscm.cn/suanfa/sport-769907.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://bbyc.wtpuscm.cn/yunying/engagement-502025.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://epzu.wtpuscm.cn/ziyuan/seminar-082645.html)
* [Strata: Windows 分布式架构 (Verified)](https://vtwz.wtpuscm.cn/yunsuan/home-782.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://kqhd.wtpuscm.cn/yinqing/budget-587627.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://kgmt.wtpuscm.cn/liuliang/topic-507563.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://bfqv.wtpuscm.cn/wendang/widget-753561.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://pewb.wtpuscm.cn/huodong/theme-450086.html)

</details>

