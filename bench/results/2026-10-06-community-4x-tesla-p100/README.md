# Community benchmark on 4x Tesla P100 16 GB (Pascal, sm_60)

Measured on 2026-10-06 by apollo-mg. This is the experimental CUDA 12 engine on Pascal, which docs/OLDER_GPUS.md lists as "not measured" for the P100.
- **Model and setup:** the installer's configuration for the GSQ-RCO IQ3_XXS model across all four cards.
- **What was measured:** short, ~4.4K and ~36.4K token prompts.
- **Main limitation:** a single machine with the GPU clocks pinned at 1063 MHz, and the engine built with CUDA 12.4, not the 12.8 / 12.9 you test with.

## Hardware and software

- **GPUs:** 4x Tesla P100-PCIE-16GB (compute capability 6.0), 150 W power limit.
  - SM clock during every run was 1063 MHz, from per-second nvidia-smi samples.
- **CPU and RAM:** 2x Intel Xeon E5-2650 v3 (AVX2, no AVX-512); 128 GB DDR4-2133 ECC (121 GB visible).
- **Storage:** Samsung SSD 860 (SATA), about 76 MB/s on large sequential reads during this session. That only affects the first
  load.
- **PCIe:** not measured.
- **OS and driver:** Ubuntu 26.04 LTS (kernel 7.0.0-34-generic), NVIDIA driver 580.173.02.
- **CUDA:** 12.4 (`/usr/bin/nvcc`, V12.4.131), with gcc-13 as the CUDA host compiler. The system default is gcc 15,
  which CUDA 12.4 does not support.
- **Strata:** v0.1.39 (tag at `a1641e9`), source build by setup.
  - There was no ready-made CUDA 12 engine for this setup ("no ready-made engine ... compiling instead").
  - Engine: `engine-cuda12/strata`, built by setup with `STRATA_NVCC=/usr/bin/nvcc CUDAHOSTCXX=/usr/bin/g++-13`,
    `--cuda 12`.
- **Background workloads:** none. The machine was idle apart from the benchmark.

## Model and configuration

- **Model:** `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF` IQ3_XXS at the revision setup pins (`ed59f92`).
  - Shards `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf` (sha256 `219ea929...56d15`) and `-00002`
    (`316b46f3...1e113`), both verified.
  - The files were already on disk and passed with `--gguf-dir`.
- **Vision:** off (`--vision no`).
- **Packs and profiles:** the pack and the MTP draft layer (q2_0 experts) were built by setup, with its default
  expert profile.
- **Context and KV:** max context 65536, `--kv int8`, KV streaming on (`--kv-resident 32768`).
- **Expert cache:** auto, with `--remote-expert-opt` and a layer split across GPUs 0-3 (auto). `--prefill auto`.
- **MTP and sampling:** `--spec 4 --spec-min-p 0.5`; reasoning effort `none`; temperature 0.
- **Not enabled:** calibration and the experimental speed projection.

```text
./setup.sh --setup --family qwen --model IQ3_XXS --gguf-dir <folder with both shards> --gpus 0,1,2,3 --cuda 12 --vision no --yes
engine args (from strata-iq3_xxs.json):
--pack .../packs/iq3_xxs --native ...IQ3_XXS-00001-of-00002.gguf --ple-gguf ...IQ3_XXS-00002-of-00002.gguf
--expert-profile data/expert-profile.bin --expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5
--mtp .../mtp/rt --max-context 65536 --kv int8 --kv-resident 32768 --remote-expert-opt
```

## Method

- **Every request is fresh:** each request starts with a unique nonce line, so no prefix is reused. The engine log
  reports `0 reused` for every request.
- **Request settings:** output capped at 256 tokens; streaming; `reasoning_effort: none` (no reasoning tokens were
  produced).
- **Short prompts:** the two LocalMaxxing canonical prompts (`canonical_reasoning-v1.txt`, `canonical_code-v1.txt`),
  sent by the LocalMaxxing `lmx` v0.1.48 CLI.
  - One untimed warm-up request, then three timed requests, per server start.
  - reasoning-v1 over two server starts, code-v1 over one.
- **Long prompts:** Strata's own docs (`docs/*.md` + `README.md` at the measured commit) plus one question, built by
  `strata_bench.py`.
  - Three runs each, with no separate warm-up; the first ~4.4K run is the first request after load.
- **Timings:**
  - Prompt and decode throughput come from the engine's own `strata serve: prompt ... generated ...` lines, all of
    which are in `runs.json`.
  - TTFT is client-side: from sending the request to the first streamed token, which is answer text.
- **Memory:** a snapshot after load (`free`, nvidia-smi).

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median and range | Decode tok/s median and range | TTFT seconds median and range |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| reasoning-v1 (short) | 301-307 | 0 | 256 | 6 | 94.0 (92.9-94.7) | 33.5 (32.0-35.3) | 3.30 (3.27-3.34) |
| code-v1 (short) | 239-242 | 0 | 256 | 3 | 86.1 (86.0-88.9) | 32.6 (32.2-33.2) | 2.83 (2.75-2.86) |
| Strata docs, ~4.4K | 4441 | 0 | 256 | 3 | 239.4 (222.3-240.8) | 29.0 (27.5-29.4) | 18.63 (18.52-20.07) |
| Strata docs, ~36.4K | 36356-36359 | 0 | 233-256 | 3 | 593.5 (583.5-594.9) | 28.0 (27.6-29.4) | 61.59 (61.38-62.58) |

- **MTP drafts accepted / drafted:**
  - short reasoning 167-174 / 215-240;
  - short code 167-168 / 228-238;
  - 4.4K 135-146 / 208-226;
  - 36.4K 131-148 / 205-209.
- **Decode expert cache hit rate:** 98.7-99.7 % over the 12 short-prompt requests. ~0 % of routed experts went over
  PCIe; the whole IQ3_XXS expert arena fits in the four cards.
- **Memory after load:** 49 GB of RAM used (72 GB available); VRAM 13.1 / 13.4 / 15.5 / 15.8 GB on GPUs 0-3.
- **Two 36.4K runs ended before the cap** (233 and 235 tokens; the engine generated fewer than 256, so the model stopped on its own; finish reason not recorded).
- **Failures:** none. Every request completed.

**For context only (a different engine):** the same weights on llama.cpp (a fork at `0b2789f23`, tensor split, the
Flash-Next MTP head at draft 3) decode the two short canonical prompts at a similar speed (33.4 tok/s median,
measured by the same `lmx` CLI).

## Correctness and limitations

- Every answer was coherent text that addressed the prompt. Answer heads are in the per-run data; a short-prompt
  example begins "### Part 1: Budget Calculation".
- No accuracy check, needle test, tool call or image test was run.
- **Untested:**
  - other packs, including IQ3_S and the Unsloth 4-bit;
  - prompts above ~36K;
  - concurrency;
  - unpinned clocks;
  - the CUDA 12.8 / 12.9 toolkits.
- Pascal emulates `__dp4a` (docs/OLDER_GPUS.md). This report does not separate that cost.

Files: `runs.json` (engine timing lines and the table rows), `client_timing_long_prompts.jsonl`, `strata_bench.py`,
and the two canonical prompts.

(Test designed and run with my agent, Claude; I reviewed it before posting.)


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://kknq.wtpuscm.cn/keji/luxury-919493.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://lxrh.wtpuscm.cn/anli/about-732285.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://qcyv.wtpuscm.cn/chuangxin/case-898290.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://fknf.wtpuscm.cn/anli/login-792977.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://wmbt.wtpuscm.cn/hezuo/reminder-968020.html)
* [面向大规模网络的 Strata 工业级架构基准](https://zpjc.wtpuscm.cn/jianzhan/responsive-911650.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://arfc.wtpuscm.cn/guanjianci/products-814631.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://lhto.wtpuscm.cn/anfang/cheap-770.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://sexh.wtpuscm.cn/gongxiang/tool-404825.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://xluc.wtpuscm.cn/pingce/domain-202201.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://hflv.wtpuscm.cn/peixun/resource-895422.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://gvww.wtpuscm.cn/jiaocheng/category-207255.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://hazl.wtpuscm.cn/gongxiang/software-572335.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://ippw.wtpuscm.cn/baogao/enterprise-339008.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://octu.wtpuscm.cn/fuwu/platform-397436.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://cafl.wtpuscm.cn/peixun/hosting-394275.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://tsdr.wtpuscm.cn/tuiguang/contact-152700.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://jwxg.wtpuscm.cn/suanfa/workshop-355659.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://gada.wtpuscm.cn/huodong/recipe-774333.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://hvme.wtpuscm.cn/jiaoliu/platform-514779.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://odos.wtpuscm.cn/chuangxin/module-904332.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://eema.wtpuscm.cn/chuangxin/prospect-023051.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://cjgb.wtpuscm.cn/yingyong/download-527164.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://jlsi.tcti.cn/kuangjia/category-46228043.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://aqwu.tcti.cn/sheji/button-21268960.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://xinm.tcti.cn/chanpin/planning-28138219.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://olhn.tcti.cn/jianzhan/blog-95148913.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://gnje.tcti.cn/baogao/settings-06071820.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://fulw.tcti.cn/paiming/target-80779939.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://fvir.tcti.cn/ziyuan/technology-51825374.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://okmj.tcti.cn/xitong/customer-64533402.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://hrrp.tcti.cn/jianzhan/roi-55067108.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://eixc.tcti.cn/peixun/project-20091051.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://ipqo.tcti.cn/yinqing/tool-89622224.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://fsfb.tcti.cn/liuliang/recommendation-84443749.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://errb.tcti.cn/fuwu/cloud-02881181.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://ehlw.tcti.cn/jianzhan/navigation-36188940.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://bwsd.tcti.cn/shangye/investment-28935210.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://yxhm.tcti.cn/yingyong/discovery-12708782.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://banx.tcti.cn/kaifa/hosting-11403776.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://yeqd.wtpuscm.cn/zhineng/efficiency-182873.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/wendang/event-86423357.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/news/24967)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/yunying/home-92889222.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://nwij.tcti.cn/chuangxin/partner-68668456.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://xazg.tcti.cn/huodong/behavior-68835452.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://wxlo.wtpuscm.cn/wangluo/experience-152610.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://hawq.wtpuscm.cn/xitong/link-003321.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://veib.wtpuscm.cn/yingxiao/status-757310.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://wqab.wtpuscm.cn/jianzhan/website-470097.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://wqtz.wtpuscm.cn/jianzhan/chapter-761102.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://ifhk.wtpuscm.cn/anli/optimization-288558.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://ewdf.wtpuscm.cn/guanjianci/conversion-574858.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://rlwz.wtpuscm.cn/chuangxin/hosting-467.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://ccuc.wtpuscm.cn/zhinan/system-178828.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://chni.wtpuscm.cn/jiaocheng/cost-417919.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://trsx.wtpuscm.cn/gongju/register-525932.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://cqzs.wtpuscm.cn/zhizhu/online-475384.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://upwz.wtpuscm.cn/chuangxin/case-306389.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://yfhq.wtpuscm.cn/peixun/plugin-269920.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://tydj.wtpuscm.cn/xitong/label-296778.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://ggjz.wtpuscm.cn/shuju/collaboration-128613.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://dwgs.wtpuscm.cn/shichang/quality-596747.html)
* [Strata 异步中间件流水线与 any 接入规范](https://qdce.wtpuscm.cn/zixun/tactic-443256.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://pekp.wtpuscm.cn/zhizhu/tool-276048.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://bhng.wtpuscm.cn/shuju/plugin-432616.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://djho.wtpuscm.cn/anli/cloud-090484.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://zihy.wtpuscm.cn/kaifa/roi-175085.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://lcep.wtpuscm.cn/zhineng/movie-560749.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://dtgx.tcti.cn/zhinan/screen-53006598.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://pamf.tcti.cn/sheji/machine-69670455.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://cdmx.tcti.cn/huodong/audience-13150296.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://pjcg.tcti.cn/suanfa/ai-74681202.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://aojc.tcti.cn/gongju/experience-21324490.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://ipwz.tcti.cn/liuliang/news-32025689.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://bfcv.tcti.cn/gongxiang/traffic-45990307.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://qaod.tcti.cn/yanjiu/game-51559155.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://bhif.tcti.cn/chuangxin/research-09683468.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://jrtp.tcti.cn/huodong/seo-82056635.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://yyvg.tcti.cn/hezuo/software-76043112.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://lnhs.tcti.cn/hezuo/module-47400689.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://mzni.tcti.cn/ziyuan/analysis-82817530.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://lzqf.tcti.cn/gongsi/profit-40387486.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://kkcj.tcti.cn/jiaoliu/research-23026731.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://rukr.tcti.cn/fuwu/subscribe-00981763.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://tvuq.tcti.cn/chuangxin/terms-99123453.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://eomy.wtpuscm.cn/yingxiao/workshop-840156.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/fenxi/analysis-43409831.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/52820)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/anli/prospect-47654091.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://pkjd.tcti.cn/jishu/saving-72198957.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://gahf.tcti.cn/shichang/beauty-56765918.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://yqev.wtpuscm.cn/fuwu/learning-604534.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://bscu.wtpuscm.cn/gongju/topic-123474.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://fnyq.wtpuscm.cn/hezuo/resolution-165065.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://yvzp.wtpuscm.cn/yinqing/backup-556366.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://fwov.wtpuscm.cn/suanfa/site-108259.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://exmb.wtpuscm.cn/jiaoliu/conversion-895510.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://qukx.wtpuscm.cn/liuliang/game-924896.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://ksqr.wtpuscm.cn/xuexi/saving-197.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://lpfo.wtpuscm.cn/chuangxin/subscribe-710094.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://rkaf.wtpuscm.cn/keji/photo-991834.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://ffiu.wtpuscm.cn/anfang/tactic-886519.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://wbei.wtpuscm.cn/baogao/revenue-355433.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://yjyt.wtpuscm.cn/gongsi/trading-430583.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://kwdq.wtpuscm.cn/ziyuan/progress-680788.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://pkrb.wtpuscm.cn/huodong/behavior-407333.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://bmfo.wtpuscm.cn/hezuo/module-450522.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://kdsh.wtpuscm.cn/zixun/progress-272731.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://xzkf.wtpuscm.cn/sheji/partner-889771.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://ncqe.wtpuscm.cn/kuangjia/experience-943632.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://plzd.wtpuscm.cn/xitong/deal-829040.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://surv.wtpuscm.cn/zhineng/notification-682235.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://qfbd.wtpuscm.cn/gongju/collaboration-323102.html)
* [Strata 故障自愈与网络拓扑重构实践](https://optu.wtpuscm.cn/zixun/ai-927885.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://dokv.tcti.cn/shangye/cloud-78332660.html)
* [Strata 权威网络权重传递与收录基准规范](https://qjmu.tcti.cn/xitong/course-01450114.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://tkhl.tcti.cn/zhineng/fashion-23132250.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://bwrm.tcti.cn/yingxiao/performance-98922508.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://ojfm.tcti.cn/yingxiao/like-31742146.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://czkm.tcti.cn/youhua/system-48027664.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://kwud.tcti.cn/zhineng/global-69847991.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://kebz.tcti.cn/suanfa/collaboration-22339305.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://chwm.tcti.cn/wendang/lead-69088525.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://efrb.tcti.cn/qiye/profit-43318561.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://pdjm.tcti.cn/chanpin/analysis-29446992.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://viyp.tcti.cn/shangye/budget-37846889.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://rsfo.tcti.cn/huodong/presentation-89245836.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://ncfa.tcti.cn/yunying/study-41980574.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://kqtq.tcti.cn/gongxiang/seminar-74515711.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://eooj.tcti.cn/wenzhang/reminder-51479215.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://juzx.tcti.cn/paiming/promotion-09701597.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://byoc.wtpuscm.cn/hezuo/internet-841210.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/chanpin/tag-61304705.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/tech/65185)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/baogao/segment-39842341.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://sgow.tcti.cn/qiye/webinar-30813353.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://esoe.tcti.cn/chanpin/topic-97496193.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://adyb.wtpuscm.cn/gongsi/shopping-780282.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://dxyt.wtpuscm.cn/xitong/landing-761181.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://xyng.wtpuscm.cn/gongxiang/report-948430.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://eaeq.wtpuscm.cn/ziyuan/performance-617092.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://jxpj.wtpuscm.cn/gongxiang/price-250877.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://irgn.wtpuscm.cn/qiye/travel-216058.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://hnwu.wtpuscm.cn/keji/change-823137.html)
* [Strata: Windows 分布式架构 (Verified)](https://xpsv.wtpuscm.cn/yingxiao/faq-004.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://ebgt.wtpuscm.cn/paiming/content-326093.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://nqjy.wtpuscm.cn/jishu/interface-438177.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://fquy.wtpuscm.cn/gongxiang/funnel-326453.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://byhd.wtpuscm.cn/tuiguang/hotel-122763.html)

</details>

