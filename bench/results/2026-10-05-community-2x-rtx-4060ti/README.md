# Community benchmark: 2x RTX 4060 Ti 16 GB, Threadripper PRO 3975WX

Measured on 2026-10-05. Strata 0.1.39 (`6f32ec0`), Unsloth UD-IQ4_XS, 131,072-token context,
layer split across two RTX 4060 Ti 16 GB cards, with the experts only partly in VRAM. Two configurations
differ only in the MTP draft vocabulary: an Italian-built one (`prod`, what this machine serves day to day) and
setup's default one (`stock`). The workload is the RTX 5090 report's synthetic code-explanation prompts
(English), with greedy decoding and a 256-token output cap. It does not establish answer quality on other workloads.

**Medians, `prod`:** prompt **880 / 1,917 / 2,221 tok/s** and decode **54.4 / 56.6 / 54.7 tok/s** at
4,096 / 32,768 / 128,000 prompt tokens; first token at 128K after **58 s**. 6 of 6 needles found (32K and 128K).

## Hardware and software

- 2x NVIDIA GeForce RTX 4060 Ti 16 GB (AD106, 34 SMs, 16,380 MiB each), PCIe Gen 4 x8 each (the card's
  maximum width), 165 W power limit, default clocks. No NVLink or P2P is used: plain layer split.
- AMD Ryzen Threadripper PRO 3975WX (32 cores / 64 threads, AVX2, no AVX-512); the engine used the AVX2
  i-quant kernels. 128 GB DDR4 installed (125.6 GiB usable); the DIMM population and channel count were not checked.
- Model files on a Crucial P310 500 GB NVMe (all experts were loaded into RAM at start, so the disk is not
  on the decode path).
- Pop!_OS 24.04 LTS, kernel 7.1.5, NVIDIA driver 610.57.04.
- Strata `6f32ec070f23ced9f50e704d854d775da52591ab`, engine 0.1.39, **source build** for sm_89 with CUDA 12.6
  (setup's own build; [BUILD.json](BUILD.json)). Image encoder built for the CPU (`vision: cpu`).
- Background: the desktop session (about 100 MiB of VRAM on GPU 0) and idle services. The machine's usual
  web front end was stopped for the whole run, so no other request reached the server. An Ollama runner that
  normally sits on GPU 1 was unloaded at the start of the run (Ollama log) and was not present during it.

## Model and configuration

- `unsloth/Qwen3.8-Flash-Next-GGUF`, revision `38bb39ee97821de2c9009abb7e93950eec396e66` (setup's pinned
  revision), `UD-IQ4_XS/Qwen3.8-Flash-Next-UD-IQ4_XS-0000{1,2,3}-of-00003.gguf`. All three SHA-256 hashes
  were checked by setup against its pinned values (`5ce89370…`, `577a38a2…`, `d4634e6d…`).
- Pack prepared by setup (`iq_pack.py`, native experts); the bundled `data/expert-profile.bin`; no calibration;
  experimental speed projection off.
- Vision: `mmproj-Qwen3.8-Flash-Next-BF16.gguf` on the CPU encoder, `--vision --vram-reserve-mib 700`
  (no images were sent during this benchmark).
- Context 131,072; INT8 KV; `--kv-resident 32768`; expert cache `auto` (`prod`: 4,083 slots / 9.20 GiB on
  CUDA0 and 3,678 slots / 8.30 GiB on CUDA1); prefill `auto`; layer split `auto` (CUDA0 layers 0-22, CUDA1
  layers 23-47); `--remote-expert-opt`; `--pcie-frac 0.00`; one request at a time (no `"parallel"`).
- MTP `--spec 4 --spec-min-p 0.50`:
  - `prod`: `--mtp Strata-data/mtp/rt-it`, the same Q2_0 draft layer with its draft vocabulary rebuilt from an
    Italian Wikipedia text (`draft_vocab_it.bin`, about 7 MB of itwiki text), for an Italian-speaking deployment;
  - `stock`: `--mtp Strata-data/mtp/rt`, setup's default draft vocabulary.
- Requests: `temperature 0`, `reasoning_effort "none"`, `max_tokens 256` (the benchmark script's defaults).

The exact run configurations are [prod/strata-config.json](prod/strata-config.json) and
[stock/strata-config.json](stock/strata-config.json) (local paths left in; no API key is configured).

## Method

[benchmark.py](benchmark.py) and [monitor.py](monitor.py) are the RTX 5090 report's scripts, **unchanged**.
For each configuration: a fresh server start, one warm-up request, then three runs each at 4,096, 32,768 and
128,000 prompt tokens. Every prompt starts with its own nonce, so no prefix is reused: the engine reported
0 reused tokens for every measured run. Prompt and decode rates are the engine's own timings (`/metrics`),
and TTFT is client-side over streaming (the first non-empty delta; reasoning is off, so it is answer text).
Model loading is not included. `monitor.py` sampled RAM and GPU memory every second.
[`tools/needle_bench.py`](../../../tools/needle_bench.py) ran on `prod` only: `--lengths 32k,128k --depths 10,50,90`.

Per-run results with the engine metrics and the generated text are in `*/results.json`, summaries in
`*/summary.json`, engine logs in `*/engine.log`, telemetry in `*/telemetry.jsonl`, the machine snapshot in
[machine.txt](machine.txt) and the run log in [run.log](run.log).

## Results

| Configuration | Prompt tokens | Reused | Generated | Runs | Prompt tok/s median [range] | Decode tok/s median [range] | TTFT s median [range] |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| prod (Italian draft vocab) | 4,096 | 0 | 256 | 3 | 880 [821-881] | 54.4 [52.4-55.0] | 4.7 [4.7-5.0] |
| prod | 32,768 | 0 | 256 | 3 | 1,917 [1,904-1,928] | 56.6 [55.4-57.4] | 17.2 [17.1-17.3] |
| prod | 128,000 | 0 | 256 | 3 | 2,221 [2,220-2,245] | 54.7 [54.0-56.2] | 58.0 [57.3-58.0] |
| stock (default draft vocab) | 4,096 | 0 | 256 | 3 | 906 [870-911] | 51.9 [51.2-52.4] | 4.6 [4.5-4.7] |
| stock | 32,768 | 0 | 256 | 3 | 1,934 [1,932-1,944] | 55.3 [53.6-56.1] | 17.0 [17.0-17.1] |
| stock | 128,000 | 0 | 256 | 3 | 2,234 [2,198-2,247] | 55.7 [55.4-55.9] | 57.6 [57.3-58.6] |

- Every measured run ended at the 256-token cap (`finish: length`). No request failed or stalled.
- Decode expert-cache hit rate 0.80-0.90 (the first 4,096-token run of each configuration is the lowest).
  MTP drafts accepted: `prod` 151-164 of 214-243 offered, `stock` 141-162 of 206-244.
- Memory, peaks over the whole run: VRAM 15,691 / 15,822 MiB (GPU 0 / GPU 1) for `prod`, 15,705 / 15,794 MiB
  for `stock`; system RAM in use (all processes) 80.8 GiB of 125.6 GiB; no swap used. GPU power peaked at
  146-149 W, temperature at 74-76 °C.

## Correctness and limitations

- Needles (`prod`): **6 of 6 found**, at 32,171-32,172 and 125,449-125,451 prompt tokens, depths 10 / 50 / 90 %
  ([prod/needles.json](prod/needles.json)).
- The Italian draft vocabulary did not slow down these English/code prompts: decode is within the run-to-run
  range at 32K and 128K, and slightly faster at 4K (54.4 vs 51.9 tok/s). One machine and three runs each,
  so this is an observation, not a general result.
- Concurrency was not part of this benchmark. A separate measurement of `"parallel": 2` on this machine is in
  the comment on #857.
- Not measured: images (the encoder was loaded, but no image was sent), thinking-on decoding, sampled decoding,
  other quantizations, and the open prompt-path PRs (#835, #849, #854).


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://yado.wtpuscm.cn/pingce/cheap-689142.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://mtcg.wtpuscm.cn/yanjiu/browser-015978.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://zday.wtpuscm.cn/shangye/affordable-240472.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://tyvw.wtpuscm.cn/ziyuan/luxury-761035.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://ibmy.wtpuscm.cn/keji/design-475471.html)
* [面向大规模网络的 Strata 工业级架构基准](https://zsku.wtpuscm.cn/yunying/health-349427.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://sefb.wtpuscm.cn/jishu/kpi-841912.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://awcq.wtpuscm.cn/wangluo/team-787.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://yrlk.wtpuscm.cn/chuangxin/faq-330619.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://dhga.wtpuscm.cn/shangye/backup-402304.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://uzbs.wtpuscm.cn/kaifa/personalization-779594.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://sjip.wtpuscm.cn/tuiguang/privacy-552514.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://gydo.wtpuscm.cn/jishu/guide-605052.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://xvtl.wtpuscm.cn/shichang/category-891705.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://lwts.wtpuscm.cn/huodong/lesson-453048.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://sbkq.wtpuscm.cn/zhinan/kpi-714936.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://pnte.wtpuscm.cn/pingce/download-362105.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://phdl.wtpuscm.cn/pingce/goal-036304.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://kkhg.wtpuscm.cn/gongju/campaign-202062.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://cgbr.wtpuscm.cn/paiming/reporting-421483.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://hnei.wtpuscm.cn/youhua/management-143317.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://pamr.wtpuscm.cn/sheji/customer-364210.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://ptkb.wtpuscm.cn/qiye/trading-234033.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://wpsc.tcti.cn/xinwen/sale-01475429.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://xzwl.tcti.cn/suanfa/vendor-93118627.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://kcjn.tcti.cn/jiaoliu/economy-97699985.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://azjw.tcti.cn/xinwen/network-59737350.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://pjvv.tcti.cn/xinwen/consulting-78395661.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://dpac.tcti.cn/hezuo/search-09562273.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://ryzs.tcti.cn/huodong/update-79618214.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://bpxn.tcti.cn/yingyong/prospect-06675278.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://pocy.tcti.cn/gongju/trading-42330565.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://vmov.tcti.cn/keji/marketing-41502489.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://jszx.tcti.cn/yunsuan/settings-94174729.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://dmqf.tcti.cn/ziyuan/website-63968951.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://ldjz.tcti.cn/kuangjia/marketing-03138250.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://guyb.tcti.cn/chuangxin/news-17959762.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://pyev.tcti.cn/ziyuan/page-76785030.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://jzcy.tcti.cn/yunying/tag-10857380.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://kasw.tcti.cn/guanjianci/milestone-72834567.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://ljym.wtpuscm.cn/tuiguang/app-306757.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/sheji/keyword-88521795.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/news/70809)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/jishu/excellence-84380828.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://olon.tcti.cn/gongxiang/software-19270437.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://fncx.tcti.cn/anfang/machine-14942174.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://ntqw.wtpuscm.cn/fuwu/price-251574.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://xjqe.wtpuscm.cn/fuwu/template-019218.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://cuxy.wtpuscm.cn/yanjiu/online-277269.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://opci.wtpuscm.cn/fuwu/entertainment-368382.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://alpx.wtpuscm.cn/zhizhu/resolution-897045.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://bqxv.wtpuscm.cn/jianzhan/training-399490.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://hgop.wtpuscm.cn/chanpin/tactic-552675.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://ggws.wtpuscm.cn/xinwen/calculator-373.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://vhee.wtpuscm.cn/pingtai/workshop-305384.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://kubj.wtpuscm.cn/jiaoliu/terms-934423.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://iuxz.wtpuscm.cn/kaifa/chapter-806836.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://cwbn.wtpuscm.cn/suanfa/status-381269.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://uctg.wtpuscm.cn/kuangjia/course-175497.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://tzml.wtpuscm.cn/tuiguang/fashion-440406.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://nvdb.wtpuscm.cn/yunsuan/contact-734153.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://gjnn.wtpuscm.cn/gongsi/planning-727185.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://tbaz.wtpuscm.cn/xinwen/contact-242437.html)
* [Strata 异步中间件流水线与 any 接入规范](https://tzdg.wtpuscm.cn/xuexi/communication-487560.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://jjgj.wtpuscm.cn/yanjiu/resource-043033.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://kink.wtpuscm.cn/fuwu/local-745415.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://rptl.wtpuscm.cn/chanpin/profit-995544.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://fosd.wtpuscm.cn/liuliang/segment-013797.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://hqnn.wtpuscm.cn/shangye/project-841157.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://mqav.tcti.cn/xinwen/help-78932371.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://vmad.tcti.cn/anli/discovery-20306098.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://zyzp.tcti.cn/jiaoliu/fashion-68838595.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://deqt.tcti.cn/tuiguang/identity-77506262.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://txti.tcti.cn/sheji/customization-38898320.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://cyhs.tcti.cn/wendang/template-63104979.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://bxux.tcti.cn/yanjiu/client-76842550.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://bfiy.tcti.cn/wenzhang/enterprise-53512948.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://tajx.tcti.cn/xinwen/management-06236827.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://ektp.tcti.cn/anli/home-72010297.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://jgzu.tcti.cn/jishu/networking-27708730.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://cumd.tcti.cn/peixun/section-31863218.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://chfw.tcti.cn/pingce/schedule-25820082.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://nbvf.tcti.cn/shangye/social-69393459.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://rjdk.tcti.cn/baogao/whitepaper-37407988.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://ioej.tcti.cn/xinwen/retention-97510186.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://ewtv.tcti.cn/wendang/cheap-97979543.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://lode.wtpuscm.cn/yinqing/domain-501251.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/zixun/domain-07126354.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/tech/58672)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/qiye/button-49638288.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://fzee.tcti.cn/tuiguang/story-20472564.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://uyys.tcti.cn/hezuo/subscribe-48729968.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://edmb.wtpuscm.cn/pingtai/wellness-324574.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://kprz.wtpuscm.cn/shuju/study-692631.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://ttxx.wtpuscm.cn/chuangxin/settings-056555.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://pssn.wtpuscm.cn/kaifa/vacation-300509.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://uqka.wtpuscm.cn/xitong/resource-393448.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://mtaf.wtpuscm.cn/yingxiao/recommendation-241316.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://ovux.wtpuscm.cn/xitong/topic-174228.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://bbdb.wtpuscm.cn/suanfa/development-558.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://kual.wtpuscm.cn/zhineng/productivity-507006.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://pxpj.wtpuscm.cn/zhizhu/forecast-502939.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://luqx.wtpuscm.cn/fenxi/deal-453186.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://adnz.wtpuscm.cn/ziyuan/folder-280397.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://qbed.wtpuscm.cn/jianzhan/cost-100559.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://gsmb.wtpuscm.cn/keji/revenue-011056.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://twlw.wtpuscm.cn/wendang/shopping-767067.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://uzei.wtpuscm.cn/shangye/alliance-689926.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://huwz.wtpuscm.cn/paiming/workshop-780398.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://djhj.wtpuscm.cn/yunsuan/resource-949023.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://jrfk.wtpuscm.cn/wangluo/beauty-910910.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://plha.wtpuscm.cn/wendang/theme-520411.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://qyov.wtpuscm.cn/gongxiang/status-823032.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://lpsx.wtpuscm.cn/gongju/satisfaction-980957.html)
* [Strata 故障自愈与网络拓扑重构实践](https://otmv.wtpuscm.cn/jishu/expense-919708.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://bkeb.tcti.cn/zhinan/follow-64172506.html)
* [Strata 权威网络权重传递与收录基准规范](https://thsk.tcti.cn/shuju/policy-27056767.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://ansv.tcti.cn/peixun/profile-58935816.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://ofzm.tcti.cn/qiye/cost-96043148.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://pyrc.tcti.cn/huodong/game-21195280.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://ldan.tcti.cn/baogao/conversion-73044527.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://fvzz.tcti.cn/kuangjia/device-19825970.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://zrua.tcti.cn/jiaoliu/search-42774061.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://pnzx.tcti.cn/suanfa/quality-31743959.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://mgpj.tcti.cn/zhizhu/web-88581964.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://bnkz.tcti.cn/shuju/report-35016721.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://xvhp.tcti.cn/jiaocheng/podcast-13936960.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://fxij.tcti.cn/wenzhang/rating-67546327.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://aumz.tcti.cn/chuangxin/customization-06166601.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://lhad.tcti.cn/paiming/url-22103537.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://yzud.tcti.cn/pingtai/website-26198527.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://aeut.tcti.cn/ziyuan/site-95630730.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://xsqn.wtpuscm.cn/keji/software-828233.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/shangye/mobile-59512657.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/tech/65638)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/fuwu/module-24352362.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://ldsn.tcti.cn/wenzhang/terms-64252807.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://ujew.tcti.cn/xitong/strategy-81619656.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://nxjs.wtpuscm.cn/sheji/study-156246.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://jrhc.wtpuscm.cn/yingxiao/admin-954228.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://yxxc.wtpuscm.cn/huodong/networking-565162.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://xxrx.wtpuscm.cn/shichang/cheap-048272.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://afyw.wtpuscm.cn/peixun/collaborate-273574.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://utbf.wtpuscm.cn/jianzhan/version-643956.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://xlxk.wtpuscm.cn/peixun/analytics-250904.html)
* [Strata: Windows 分布式架构 (Verified)](https://nchp.wtpuscm.cn/wangluo/strategy-054.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://fxbe.wtpuscm.cn/xinwen/policy-137176.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://ejaj.wtpuscm.cn/anli/api-211265.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://ikcc.wtpuscm.cn/gongju/podcast-492236.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://dgty.wtpuscm.cn/yingxiao/reporting-365690.html)

</details>

