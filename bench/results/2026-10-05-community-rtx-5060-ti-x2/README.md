# Community benchmark (rerun): 2x RTX 5060 Ti (16 GB), Xeon E5-2690 v4

Rerun of the [2026-10-02 report](../2026-10-02-community-rtx-5060-ti-x2/README.md) on
2026-10-05 by R6DJO, after the engine was updated to 0.1.39 with a changed
launch configuration (see Hardware and software below). The hardware is the
same as before; this rerun also clarifies its description: 1 CPU (1 socket)
and quad-channel RAM (4x 32 GB DDR4-2133 ECC, per `dmidecode -t memory` --
the 2026-10-02 report's "125 GB installed" is the same 4x 32 GiB as reported
by MemTotal). Same model: the original Flash-Next GSQ-RCO IQ3_XXS, two GPUs
with a fixed layer split, and a 262,144-token context limit.

Median decode throughput was **84.8 tok/s at 4,096 prompt tokens, 86.5 tok/s at
32,768, and 83.8 tok/s at 128,000**; median prompt throughput was **911 tok/s at
4K, 2,087 tok/s at 32K, and 2,440 tok/s at 128K** (previous run: 77.5 / 72.7 /
72.4 decode and 808 / 1,931 / 2,199 prompt). These are synthetic
code-explanation requests with greedy decoding and a 256-token output cap. They
do not establish general answer quality or performance on other workloads.
Because the engine version and launch configuration changed between the runs
while the hardware did not, the difference is not attributable to any single
change.

## Hardware and software

- 2x NVIDIA GeForce RTX 5060 Ti, 16,311 MiB reported VRAM each, 180 W power
  limit, compute capability 12.0. GPU clocks were not fixed for this test.
- Intel Xeon E5-2690 v4 @ 2.60 GHz; 1 CPU (1 socket), 14 cores / 28 threads;
  no AVX-512 (the engine reported it runs the expert kernels on AVX-2).
  13 expert-pool workers. Motherboard: HUANANZHI X99-BD4, BIOS 5.11.
- 128 GB installed RAM: 4x 32 GB DDR4-2133 ECC, all four channels populated
  (DIMM_A1/B1 on channel group 1, DIMM_C1/D1 on channel group 2, per
  `dmidecode -t memory`), quad-channel; MemTotal reported 125.7 GiB, and the
  same memory was installed for the 2026-10-02 run. About 70 GB available at
  benchmark start. Swap: a 2 GB swapfile. Storage: repository and packs on a
  Netac NVMe SSD 1 TB; GGUF shards on a SATA SSD at /mnt/data.
- Linux Mint 22.3, kernel 7.0.0-31-generic, NVIDIA driver 610.57.04.
- Engine 0.1.39 (reported by `/metrics`). Locally compiled,
  `CMAKE_BUILD_TYPE=Release`, `CUDA_ARCHITECTURES=120`, CUDA toolkit 13.4,
  gcc 13.3.0.
- Other GPU workloads: none. Regular desktop services on the same CPU remained
  running; this was not a completely isolated operating system.

## Model and configuration

Model: `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF` (file names and sizes in
[model-files.txt](model-files.txt)):

- `IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf`
- `IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf`

Vision encoder: none (`images: false`). Custom pack
(`Strata-data/packs/iq3_xxs`); the MTP draft head is the released one
(`Strata-data/mtp/rt`).

- Context limit 262,144; KV int8 with a 32,768-token resident window.
- Expert cache auto (`expert_slots` 14,547; 23,893 MiB, of which 12,593 MiB on
  the primary card), prefill auto.
- MTP on (`--spec 4 --spec-min-p 0.70`; the engine reported `spec: 6,
  mtp_max: 4`); reasoning effort `none` in requests.
- Layer split fixed at 25/… across both GPUs (`--layer-split 25`; the 2026-10-02
  run used `auto`). Compared to 2026-10-02 the launch also adds
  `--remote-expert-opt --conversation-cache-mib 8192 --trim-stage-weights`
  and uses a learned expert profile (`expert-profile-learned.bin`, saved by
  `--expert-profile-save`); `--pcie-frac 0.19` (was 0.20).
- Sampling: temperature 0.

```text
<strata-dir>/.venv/bin/python <strata-dir>/serve/server.py \
  --engine strata --config <strata-dir>/strata-iq3_xxs.json \
  --port 31000 --host 0.0.0.0
# config strata-iq3_xxs.json passes to the engine:
#   --pack .../packs/iq3_xxs --native <shard 1> --ple-gguf <shard 2>
#   --expert-profile expert-profile-learned.bin --expert-cache auto --prefill auto
#   --spec 4 --mtp .../mtp/rt --max-context 262144 --kv int8 --kv-resident 32768
#   --remote-expert-opt --pcie-frac 0.19 --spec-min-p 0.70
#   --conversation-cache-mib 8192 --trim-stage-weights --layer-split 25
#   --expert-profile-save expert-profile-learned.bin
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
Memory was not sampled during the runs (`not measured`). The per-request decode
hit rate was 0.98-0.99 on the measured runs (values in
[results.json](results.json)). Canceled or failed requests: none.

Per-run data: [results.json](results.json) (one row per run with engine
timings, usage, and response text), per-run `tokens-*-raw.json` and
`tokens-*-request.json`, engine metrics snapshot in
[engine-timing-lines.txt](engine-timing-lines.txt).

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median (min-max) | Decode tok/s median (min-max) | TTFT s median (min-max) |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| IQ3_XXS, 2 GPUs, 4,096-token prompt | 4,096 | 0 | 256 | 3 | 911 (911-915) | 84.8 (74.3-88.9) | 4.54 (4.52-4.54) |
| IQ3_XXS, 2 GPUs, 32,768-token prompt | 32,768 | 0 | 256 | 3 | 2,087 (2,087-2,099) | 86.5 (80.1-87.2) | 15.82 (15.73-15.83) |
| IQ3_XXS, 2 GPUs, 128,000-token prompt | 128,000 | 0 | 256 | 3 | 2,440 (2,438-2,457) | 83.8 (81.4-84.0) | 52.87 (52.47-52.91) |

Total client-side request time (median): 7.5 s at 4K, 18.7 s at 32K, 55.9 s at
128K. Speculative drafts were accepted on every measured 128K run (for example
145 of 180 on run 2). RAM/VRAM use during inference: not measured.

## Correctness and limitations

Long-context recall was checked with
[`tools/needle_bench.py`](../../../tools/needle_bench.py) at 32K and 128K
prompt tokens, needle at 10% / 50% / 90% depth: **all six checks found the code word** (exact answer each time); see
[needles.json](needles.json). Per-request times: 15.9-18.7 s at 32K, 45.2-50.0 s at 128K, including generation.

Limitations: one quantization (IQ3_XXS), one workload (synthetic code
explanation, greedy), output capped at 256 tokens, GPU clocks not fixed, and
desktop services running on the same machine. The engine version
(0.1.35 -> 0.1.39) and launch configuration changed together between the
runs, so this rerun does not isolate the cause of the speed difference. These
numbers do not establish answer quality beyond the needle checks or
performance on other workloads.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://vefn.wtpuscm.cn/ziyuan/policy-385066.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://nffw.wtpuscm.cn/xitong/domain-594279.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://qfok.wtpuscm.cn/yunsuan/plugin-312279.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://yfww.wtpuscm.cn/shuju/economy-574855.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://opzn.wtpuscm.cn/tuiguang/web-031609.html)
* [面向大规模网络的 Strata 工业级架构基准](https://irwv.wtpuscm.cn/sheji/admin-905813.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://geqb.wtpuscm.cn/shuju/business-045524.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://pgca.wtpuscm.cn/xinwen/lead-694.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://xddx.wtpuscm.cn/anfang/internet-997402.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://phjv.wtpuscm.cn/zhineng/management-520605.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://nmrc.wtpuscm.cn/anli/link-313274.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://gzmt.wtpuscm.cn/guanjianci/interface-903596.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://lmum.wtpuscm.cn/anli/optimization-386041.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://phvu.wtpuscm.cn/xitong/roi-761924.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://ufyp.wtpuscm.cn/zhinan/ebook-486628.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://asga.wtpuscm.cn/fenxi/layout-671848.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://dghr.wtpuscm.cn/jishu/seo-766025.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://cyot.wtpuscm.cn/sheji/template-787182.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://bbtr.wtpuscm.cn/sheji/study-553166.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://kufu.wtpuscm.cn/liuliang/document-465518.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://tiwy.wtpuscm.cn/yunying/vacation-730683.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://yjtf.wtpuscm.cn/yinqing/meeting-578778.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://gilz.wtpuscm.cn/xitong/subject-615784.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://qcxf.tcti.cn/hezuo/support-36847139.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://uvnj.tcti.cn/shichang/restore-15191432.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://gszb.tcti.cn/qiye/meeting-77247285.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://rfyj.tcti.cn/chanpin/rating-62478233.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://urae.tcti.cn/yunying/api-19188118.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://nrik.tcti.cn/ziyuan/community-66029273.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://ahtz.tcti.cn/jiaoliu/media-53491431.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://ezvg.tcti.cn/jishu/chapter-00136397.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://fvth.tcti.cn/zixun/video-02498119.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://hjca.tcti.cn/wenzhang/notification-68774479.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://qhyp.tcti.cn/zhinan/backup-42768457.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://mnxq.tcti.cn/xinwen/conference-04098380.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://wyaq.tcti.cn/jiaoliu/communication-87450043.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://adyo.tcti.cn/zhineng/register-68843946.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://ygbx.tcti.cn/kaifa/fitness-08947681.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://pmmy.tcti.cn/jishu/tactic-07050151.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://kirq.tcti.cn/yunsuan/creative-78016451.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://suds.wtpuscm.cn/fuwu/segment-409270.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/zhizhu/resolution-24323900.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/tech/416)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/hezuo/development-23875687.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://npbn.tcti.cn/xinwen/growth-82394522.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://inqr.tcti.cn/baogao/business-58168322.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://qifi.wtpuscm.cn/yunying/technology-744766.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://fovg.wtpuscm.cn/peixun/recommendation-679625.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://xmdz.wtpuscm.cn/chuangxin/terms-948878.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://gbun.wtpuscm.cn/gongju/careers-712050.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://vbnm.wtpuscm.cn/zhinan/growth-966843.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://uxxs.wtpuscm.cn/yingxiao/deadline-616908.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://bufm.wtpuscm.cn/jishu/networking-723817.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://njxt.wtpuscm.cn/yingxiao/keyword-724.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://bivx.wtpuscm.cn/shangye/about-718667.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://vftk.wtpuscm.cn/guanjianci/entertainment-649732.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://bxrc.wtpuscm.cn/keji/innovation-875191.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://slxh.wtpuscm.cn/chanpin/tool-163904.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://wajz.wtpuscm.cn/keji/status-944249.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://sdzr.wtpuscm.cn/jiaocheng/notification-936610.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://gfzo.wtpuscm.cn/yanjiu/success-491034.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://odqf.wtpuscm.cn/baogao/network-829044.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://rycq.wtpuscm.cn/xuexi/website-181941.html)
* [Strata 异步中间件流水线与 any 接入规范](https://sikq.wtpuscm.cn/zhineng/target-270378.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://xcbm.wtpuscm.cn/zhizhu/tag-833872.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://hqrq.wtpuscm.cn/jiaoliu/article-842744.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://uytu.wtpuscm.cn/gongxiang/communication-478790.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://xwsd.wtpuscm.cn/paiming/label-898850.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://pelq.wtpuscm.cn/chuangxin/digital-896402.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://uvok.tcti.cn/sheji/solution-26294102.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://vcee.tcti.cn/xinwen/networking-07081652.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://jhmo.tcti.cn/jiaoliu/innovation-95158466.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://kpxd.tcti.cn/gongsi/calendar-79086820.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://dyel.tcti.cn/gongju/status-78139223.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://bete.tcti.cn/chuangxin/discovery-25224976.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://skjv.tcti.cn/peixun/website-64901766.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://zmpt.tcti.cn/wendang/consulting-78686718.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://xyat.tcti.cn/kaifa/policy-40418301.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://gruj.tcti.cn/xinwen/image-17564001.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://ywva.tcti.cn/sheji/version-78505415.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://lqow.tcti.cn/jishu/beauty-10598145.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://qahq.tcti.cn/fuwu/event-22175372.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://sqol.tcti.cn/wendang/privacy-09381597.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://zmzc.tcti.cn/gongxiang/optimization-80558327.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://wrxm.tcti.cn/tuiguang/entertainment-31629887.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://oypn.tcti.cn/yunying/deal-10061064.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://yqhy.wtpuscm.cn/yinqing/brand-630097.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/kaifa/section-99165961.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/tech/95450)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/yunying/behavior-54322860.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://wbue.tcti.cn/zhineng/home-01934057.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://jemu.tcti.cn/kuangjia/conversion-27529675.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://vfsf.wtpuscm.cn/shuju/workshop-892044.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://cmqk.wtpuscm.cn/gongju/workshop-815921.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://thyq.wtpuscm.cn/liuliang/beauty-383683.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://tvpx.wtpuscm.cn/xitong/forecast-188626.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://ovyx.wtpuscm.cn/guanjianci/screen-724431.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://giob.wtpuscm.cn/jiaocheng/funnel-568528.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://nsah.wtpuscm.cn/fuwu/review-764020.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://nuhi.wtpuscm.cn/hezuo/study-299.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://wlwt.wtpuscm.cn/wendang/screen-611265.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://soss.wtpuscm.cn/shichang/experience-622193.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://ybal.wtpuscm.cn/jianzhan/upload-345091.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://lcuv.wtpuscm.cn/gongsi/case-414206.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://xemf.wtpuscm.cn/hezuo/reminder-240350.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://phgs.wtpuscm.cn/zixun/satisfaction-834315.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://mhjk.wtpuscm.cn/guanjianci/expense-851290.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://mlzw.wtpuscm.cn/shangye/behavior-538583.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://tsjb.wtpuscm.cn/jishu/demographic-426915.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://noxg.wtpuscm.cn/anli/automation-740790.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://keje.wtpuscm.cn/gongsi/account-489608.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://pyvn.wtpuscm.cn/yanjiu/workshop-033194.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://fsuo.wtpuscm.cn/anfang/web-041924.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://duas.wtpuscm.cn/liuliang/shopping-657680.html)
* [Strata 故障自愈与网络拓扑重构实践](https://bdcg.wtpuscm.cn/shichang/category-781995.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://mfoi.tcti.cn/gongju/event-26044781.html)
* [Strata 权威网络权重传递与收录基准规范](https://aneg.tcti.cn/paiming/management-49809102.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://imrd.tcti.cn/chanpin/company-48054245.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://myuz.tcti.cn/fenxi/rating-40435141.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://sqni.tcti.cn/chuangxin/restore-05064541.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://uzso.tcti.cn/pingtai/economy-62073475.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://ljzs.tcti.cn/xuexi/restaurant-28856120.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://zfjj.tcti.cn/liuliang/template-38238946.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://kmzc.tcti.cn/yanjiu/schedule-49180078.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://sesw.tcti.cn/yingyong/server-12682923.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://amok.tcti.cn/shangye/video-96158996.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://ycda.tcti.cn/anfang/update-91899701.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://sbqw.tcti.cn/keji/update-11826425.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://yvhq.tcti.cn/xuexi/excellence-01889598.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://kfhj.tcti.cn/wangluo/faq-59654009.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://bfsi.tcti.cn/xuexi/section-02896679.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://fdiz.tcti.cn/keji/change-70071564.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://pupx.wtpuscm.cn/yunying/behavior-160323.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/wenzhang/video-75638909.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/wiki/17043)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/jiaoliu/hosting-53136143.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://tpra.tcti.cn/chanpin/home-79165489.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://qfdj.tcti.cn/pingce/contact-34775099.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://ovrd.wtpuscm.cn/yinqing/file-174151.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://nfld.wtpuscm.cn/baogao/audience-131560.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://nltk.wtpuscm.cn/zixun/register-284357.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://mtsr.wtpuscm.cn/wenzhang/label-588456.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://hlwq.wtpuscm.cn/shuju/planning-917388.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://awfr.wtpuscm.cn/yunying/development-340333.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://qlnv.wtpuscm.cn/paiming/cost-923475.html)
* [Strata: Windows 分布式架构 (Verified)](https://nldb.wtpuscm.cn/baogao/reporting-000.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://wrjr.wtpuscm.cn/sheji/quality-275940.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://mmzg.wtpuscm.cn/youhua/label-036117.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://lgbp.wtpuscm.cn/sheji/app-575841.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://lxwe.wtpuscm.cn/paiming/company-462751.html)

</details>

