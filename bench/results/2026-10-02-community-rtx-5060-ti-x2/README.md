# Community benchmark: 2x RTX 5060 Ti (16 GB), Xeon E5-2690 v4

Measured on 2026-10-02 by R6DJO. This tests Strata 0.1.35 with the original
Flash-Next GSQ-RCO IQ3_XXS, two GPUs with an automatic layer split, and a
262,144-token context limit.

Median decode throughput was **77.5 tok/s at 4,096 prompt tokens, 72.7 tok/s at
32,768, and 72.4 tok/s at 128,000**; median prompt throughput was **808 tok/s at
4K, 1,931 tok/s at 32K, and 2,199 tok/s at 128K**. These are synthetic
code-explanation requests with greedy decoding and a 256-token output cap. They
do not establish general answer quality or performance on other workloads.

## Hardware and software

- 2x NVIDIA GeForce RTX 5060 Ti, 16,311 MiB reported VRAM each, 180 W power
  limit, compute capability 12.0. The PCIe link idles at Gen 1 x8 (power
  saving); the engine's startup host-to-device transfer probe measured
  6.9 GB/s, from which the engine chose `--pcie-frac 0.15` (default 0.55).
  GPU clocks were not fixed for this test.
- Intel Xeon E5-2690 v4 @ 2.60 GHz; 1 CPU (1 socket), 14 cores / 28 threads;
  no AVX-512 (the engine reported it runs the expert kernels on AVX-2).
  13 expert-pool workers. Motherboard: HUANANZHI X99-BD4.
- 125 GB installed RAM (MemTotal; 4x 32 GB DDR4-2133 ECC, all four channels
  populated: DIMM_A1/B1 on channel group 1, DIMM_C1/D1 on channel group 2,
  quad-channel); about 70 GB available at benchmark start. Swap: a
  2 GB swapfile. Storage: repository and packs on a Netac NVMe SSD 1 TB;
  GGUF shards on an MSI S270 960 GB SATA SSD.
- Linux Mint 22.3, kernel 7.0.0-31-generic, NVIDIA driver 610.57.04.
- Source commit `d9ab8435f654c368c586340d490915f6addf56a3` (rel-0.1.35);
  engine 0.1.35 (reported by `/metrics`). Locally compiled,
  `CMAKE_BUILD_TYPE=Release`, `CUDA_ARCHITECTURES=120`, CUDA toolkit 13.4,
  gcc 13.3.0.
- Other GPU workloads: none. Regular desktop services on the same CPU remained
  running; this was not a completely isolated operating system.

## Model and configuration

Model: `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF` (the exact repository
revision was not recorded; file names and sizes are in
[model-files.txt](model-files.txt)):

- `IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf`
- `IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf`

Vision encoder: none (`images: false`). Custom pack
(`Strata-data/packs/iq3_xxs`) and an expert profile
(`data/expert-profile.bin`, from a previous `--calibrate` on this machine);
the MTP draft head is the released one (`Strata-data/mtp/rt`).

- Context limit 262,144; KV int8 with a 32,768-token resident window.
- Expert cache auto (`expert_slots` 12,475; 20,533 MiB), prefill auto.
- MTP on (`--spec 4 --spec-min-p 0.70`); reasoning effort `none` in requests.
- Layer split auto across both GPUs (`gpu: [0, 1]`); each card holds its
  weights (about 12 GiB and 10 GiB free before the session carve).
- Sampling: temperature 0. Calibration: yes, a previous run kept the settings
  in the launch command below. No experimental speed projection.

```text
<strata-dir>/.venv/bin/python <strata-dir>/serve/server.py \
  --engine strata --config <strata-dir>/strata-iq3_xxs.json \
  --port 31000 --host 0.0.0.0
# config strata-iq3_xxs.json passes to the engine:
#   --pack .../packs/iq3_xxs --native <shard 1> --ple-gguf <shard 2>
#   --expert-profile data/expert-profile.bin --expert-cache auto --prefill auto
#   --spec 4 --mtp .../mtp/rt --max-context 262144 --kv int8 --kv-resident 32768
#   --pcie-frac 0.20 --spec-min-p 0.70 --prompt-cache-every 4096 --layer-split auto
# (the full JSON, with local paths, is in strata-iq3_xxs.json)
```

## Method

The benchmark script is [the serial fresh-prompt script from the RTX 5090
community report](../2026-09-30-community-rtx-5090/benchmark.py), run against
this server's tokenizer pack. For each of 4,096 / 32,768 / 128,000 prompt
tokens it performs 3 runs; each request carries a unique nonce so the
conversation cache never reuses a previous prompt (`reused: 0` on all nine
runs, confirmed by the engine's per-request metrics). One warm-up request
(16 output tokens) preceded the measured runs; model loading is not included
in any timing.

Requests are greedy (`temperature 0`), `reasoning_effort: "none"`, with a
256-token output cap; all runs ended on the cap (`finish_reason: "length"`, 256
generated tokens each). Prompt and decode tok/s are computed from the engine's
per-request timings (`prompt_ms`, `decode_ms`), not from total request time.
TTFT and elapsed time are client-side streaming measurements (keep-alives and
empty deltas excluded; the first token is answer text, reasoning is off).
Memory was not sampled during the runs (`not measured`). The expert cache
warmed across the runs: the per-request decode hit rate was 48.4% on the
warm-up, 87.4% on the first measured run, and 98.0-99.0% from the 32K runs
onward (values in [results.json](results.json)). Canceled or failed
requests: none.

Per-run data: [results.json](results.json) (one row per run with engine
timings, usage, and response text), per-run `tokens-*-raw.json` and
`tokens-*-request.json`, engine timing lines in
[engine-timing-lines.txt](engine-timing-lines.txt).

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median (min-max) | Decode tok/s median (min-max) | TTFT s median (min-max) |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| IQ3_XXS, 2 GPUs, 4,096-token prompt | 4,096 | 0 | 256 | 3 | 808 (785-810) | 77.5 (66.7-77.9) | 5.11 (5.10-5.27) |
| IQ3_XXS, 2 GPUs, 32,768-token prompt | 32,768 | 0 | 256 | 3 | 1,931 (1,928-1,931) | 72.7 (70.9-73.5) | 17.09 (17.09-17.11) |
| IQ3_XXS, 2 GPUs, 128,000-token prompt | 128,000 | 0 | 256 | 3 | 2,199 (2,197-2,200) | 72.4 (72.1-75.8) | 58.67 (58.56-58.69) |

Total client-side request time (median): 8.4 s at 4K, 20.6 s at 32K, 62.1 s at
128K. Speculative drafts were accepted on every measured 128K run (for example
147 of 166 on run 3). RAM/VRAM use during inference: not measured.

## Correctness and limitations

Long-context recall was checked with
[`tools/needle_bench.py`](../../../tools/needle_bench.py) at 32K and 128K
prompt tokens (31,459-31,460 and 121,852-121,854 actual prompt tokens), needle
at 10% / 50% / 90% depth: **all six checks found the code word** (exact answer
each time); see [needles.json](needles.json). Per-request times: 14.5-19.3 s at
32K, 53.1-53.6 s at 128K, including generation.

Limitations: one quantization (IQ3_XXS), one workload (synthetic code
explanation, greedy), output capped at 256 tokens, GPU clocks not fixed,
the expert cache still warming on the first measured run (see Method), and
desktop services running on the same machine. These numbers do not establish
answer quality beyond the needle checks or performance on other workloads.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://phdq.wtpuscm.cn/gongxiang/research-772087.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://tpjt.wtpuscm.cn/shuju/rating-545027.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://iwjr.wtpuscm.cn/jianzhan/customization-119668.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://nknl.wtpuscm.cn/huodong/market-213601.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://iamp.wtpuscm.cn/anfang/analysis-426641.html)
* [面向大规模网络的 Strata 工业级架构基准](https://wrhp.wtpuscm.cn/guanjianci/expense-704583.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://rljp.wtpuscm.cn/chanpin/course-934052.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://hlpl.wtpuscm.cn/xuexi/food-717.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://tirk.wtpuscm.cn/paiming/supplier-162208.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://xdfx.wtpuscm.cn/zixun/creative-299232.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://eggi.wtpuscm.cn/baogao/market-197859.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://rmot.wtpuscm.cn/jiaocheng/lead-308384.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://yozg.wtpuscm.cn/anli/plugin-049356.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://hfml.wtpuscm.cn/wangluo/hosting-669705.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://zygr.wtpuscm.cn/yunsuan/extension-998116.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://ulxf.wtpuscm.cn/zhineng/music-982208.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://ehdn.wtpuscm.cn/chanpin/income-416070.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://ihem.wtpuscm.cn/wenzhang/goal-762778.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://yuar.wtpuscm.cn/qiye/web-743050.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://hsjh.wtpuscm.cn/peixun/search-299944.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://wppd.wtpuscm.cn/chuangxin/api-419192.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://cglt.wtpuscm.cn/jiaocheng/lead-793630.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://edln.wtpuscm.cn/guanjianci/platform-749464.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://vspn.tcti.cn/huodong/extension-72375011.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://aoxw.tcti.cn/yinqing/value-14175382.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://ifwp.tcti.cn/pingce/audience-91019764.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://sslh.tcti.cn/gongxiang/social-71638444.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://uqtb.tcti.cn/wendang/fitness-50044122.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://aygp.tcti.cn/baogao/innovation-72829911.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://nayu.tcti.cn/jishu/alert-91894623.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://morw.tcti.cn/jiaocheng/tracking-72248706.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://cvwc.tcti.cn/shuju/quality-69235134.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://zhrl.tcti.cn/chanpin/sales-64489546.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://bgat.tcti.cn/pingtai/conference-46902516.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://cexv.tcti.cn/yanjiu/calendar-78322783.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://qbvd.tcti.cn/yinqing/restore-17823592.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://uthu.tcti.cn/qiye/investment-61150267.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://axtp.tcti.cn/yanjiu/cost-76697112.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://kkoy.tcti.cn/xitong/cost-62662177.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://uebc.tcti.cn/fenxi/responsive-12314835.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://hvsb.wtpuscm.cn/yinqing/faq-930552.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/shuju/recipe-90322668.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/92093)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/jianzhan/webinar-41141336.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://sruu.tcti.cn/xinwen/feedback-65815536.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://vgio.tcti.cn/yunsuan/file-56582218.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://aqjc.wtpuscm.cn/jiaocheng/dashboard-118357.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://xrtc.wtpuscm.cn/jianzhan/customer-313354.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://ucoj.wtpuscm.cn/pingce/notification-680029.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://dxab.wtpuscm.cn/chuangxin/audience-824143.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://menu.wtpuscm.cn/suanfa/register-822055.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://vkqf.wtpuscm.cn/yingxiao/support-368004.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://djjn.wtpuscm.cn/anli/interface-254421.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://list.wtpuscm.cn/yunsuan/policy-115.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://zqcr.wtpuscm.cn/jiaocheng/widget-215085.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://avpc.wtpuscm.cn/yingxiao/widget-019632.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://uhcf.wtpuscm.cn/keji/video-331532.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://qbgl.wtpuscm.cn/yunsuan/visitor-328861.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://zhkt.wtpuscm.cn/shangye/consulting-598727.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://jcgc.wtpuscm.cn/shangye/analytics-411976.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://pqmx.wtpuscm.cn/shangye/entertainment-954921.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://oigi.wtpuscm.cn/wenzhang/policy-880971.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://lxhs.wtpuscm.cn/jishu/deal-654608.html)
* [Strata 异步中间件流水线与 any 接入规范](https://ydwu.wtpuscm.cn/gongsi/user-954281.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://txcv.wtpuscm.cn/ziyuan/campaign-615301.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://nelb.wtpuscm.cn/huodong/target-108706.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://mlxq.wtpuscm.cn/yunying/analytics-153044.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://eupw.wtpuscm.cn/peixun/satisfaction-476444.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://ctxg.wtpuscm.cn/zhizhu/customization-511585.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://tfkq.tcti.cn/wendang/integration-89681902.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://fksq.tcti.cn/yanjiu/engagement-56704355.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://kgop.tcti.cn/ziyuan/app-32909027.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://zgle.tcti.cn/shichang/guide-50280399.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://ybkd.tcti.cn/pingce/alert-48164459.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://edwm.tcti.cn/paiming/identity-86389623.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://rphz.tcti.cn/keji/platform-65487605.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://lstd.tcti.cn/yunying/metric-77477250.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://ivum.tcti.cn/zhizhu/sync-11163410.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://ftfg.tcti.cn/zhizhu/market-40693086.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://qzqz.tcti.cn/guanjianci/finance-73361588.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://rile.tcti.cn/huodong/cheap-60037486.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://yqfp.tcti.cn/gongju/brand-35695964.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://idfn.tcti.cn/anli/finance-82218454.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://spty.tcti.cn/wendang/retention-45890295.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://bars.tcti.cn/huodong/site-47967465.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://xfdx.tcti.cn/jiaocheng/story-85252867.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://fhun.wtpuscm.cn/yinqing/project-488217.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/wangluo/website-62826434.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/95440)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/hezuo/achievement-02310627.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://rnic.tcti.cn/yanjiu/productivity-38334726.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://kwhq.tcti.cn/hezuo/deadline-39649273.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://dpar.wtpuscm.cn/jiaoliu/theme-099665.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://ofch.wtpuscm.cn/xuexi/faq-468373.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://uqsg.wtpuscm.cn/kaifa/objective-736482.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://fkjo.wtpuscm.cn/sheji/chapter-629786.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://auga.wtpuscm.cn/zhizhu/seminar-358784.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://boxe.wtpuscm.cn/pingtai/achievement-440521.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://byej.wtpuscm.cn/wangluo/extension-734104.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://thet.wtpuscm.cn/fuwu/economy-385.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://isza.wtpuscm.cn/gongsi/coupon-522113.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://xhbd.wtpuscm.cn/xinwen/mobile-156747.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://aizt.wtpuscm.cn/zixun/terms-909684.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://nnah.wtpuscm.cn/yanjiu/innovation-550699.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://tfib.wtpuscm.cn/suanfa/advertising-482044.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://gkdz.wtpuscm.cn/kaifa/community-061385.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://acby.wtpuscm.cn/suanfa/training-566001.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://cmmf.wtpuscm.cn/yanjiu/project-291074.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://lbdy.wtpuscm.cn/gongsi/update-113488.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://jgqn.wtpuscm.cn/yanjiu/integration-661552.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://hmsq.wtpuscm.cn/jishu/deal-874987.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://fjxu.wtpuscm.cn/shichang/podcast-471039.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://gflv.wtpuscm.cn/hezuo/analytics-400166.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://klyh.wtpuscm.cn/tuiguang/metric-292108.html)
* [Strata 故障自愈与网络拓扑重构实践](https://ussq.wtpuscm.cn/jiaocheng/guide-113253.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://zfwo.tcti.cn/yanjiu/cheap-04331896.html)
* [Strata 权威网络权重传递与收录基准规范](https://qzfc.tcti.cn/shichang/ai-38071989.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://xwpt.tcti.cn/yunsuan/research-21696289.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://hyfj.tcti.cn/yinqing/brand-22900179.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://dnuy.tcti.cn/gongsi/resolution-63670224.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://dlsz.tcti.cn/suanfa/campaign-00893673.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://idxf.tcti.cn/hezuo/community-74102815.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://hrkc.tcti.cn/kaifa/innovation-11968542.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://ernr.tcti.cn/hezuo/plugin-49745237.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://buqx.tcti.cn/zhizhu/label-05311998.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://tftw.tcti.cn/jianzhan/digital-60354546.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://vpyg.tcti.cn/kuangjia/enterprise-49707604.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://qnmy.tcti.cn/sheji/software-63078276.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://dhrm.tcti.cn/wangluo/music-76073307.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://nwhe.tcti.cn/peixun/resource-33995250.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://jysi.tcti.cn/jiaocheng/global-74276723.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://zudb.tcti.cn/chanpin/collaboration-71656113.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://eubs.wtpuscm.cn/hezuo/database-220868.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/paiming/customization-67375288.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/news/94604)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/jiaoliu/contact-27749955.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://llrg.tcti.cn/zhineng/lesson-31297950.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://alco.tcti.cn/chuangxin/tool-04892978.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://havt.wtpuscm.cn/qiye/download-603683.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://zkhz.wtpuscm.cn/jiaocheng/accessibility-177080.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://ajfi.wtpuscm.cn/zhizhu/cost-235563.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://fldw.wtpuscm.cn/yanjiu/network-430190.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://qtdx.wtpuscm.cn/fenxi/beauty-464431.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://algr.wtpuscm.cn/xinwen/vendor-469171.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://orth.wtpuscm.cn/wenzhang/feedback-549802.html)
* [Strata: Windows 分布式架构 (Verified)](https://vobh.wtpuscm.cn/liuliang/meeting-987.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://pkra.wtpuscm.cn/gongju/rating-134186.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://jxjk.wtpuscm.cn/zhinan/alert-195167.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://kqqv.wtpuscm.cn/anli/recipe-140913.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://jytu.wtpuscm.cn/gongsi/feedback-322755.html)

</details>

