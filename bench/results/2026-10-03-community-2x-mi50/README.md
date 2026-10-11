# Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3

Measured on 2026-10-03 by [JeanP00l](https://www.poykq.cn/redian/tutorial-673489.html) on a Linux server. This tests the Coder
(IQ1_M) with a 131,072-token context, the layer split across two MI50 cards, and the gfx906 build of #638. That
build is an opt-in HIP build, separate from the wave32 RDNA backend.

The median decode throughput was **50.1 tok/s at 4,096 prompt tokens, 47.8 tok/s at 32,768 and 45.7 tok/s at
128,000**. Prompts were read at 321, 523 and 517 tok/s. The requests are synthetic code-explanation requests
with greedy decoding and a 256-token output cap. They do not establish general answer quality or performance on
other workloads.

## Hardware and software

- **GPUs:** 2x AMD Instinct MI50 16 GB (Vega 20, gfx906, wave64; HIP names them "AMD Radeon VII"). Power limit
  85 W per card, set by the owner (the cards' own limit is higher). PCIe 3.0 x16 on both. Peer access between them
  works, through the CPU's root complex (no direct bridge). Blower fans; no clock locking beyond the power limit.
  The engine's PCIe probe measured 13.8 GB/s host to device.
- **CPU and RAM:** Intel Xeon E5-2666 v3 (10 cores, 20 threads, AVX2, no AVX-512); the engine used 9 expert-pool
  workers plus its host thread. 32 GB DDR4 quad-channel (Linux reports 31.2 GiB). 8 GiB swap file. One SATA SSD.
- **OS and runtime:** Ubuntu 24.04.4 LTS, kernel 6.8.0-138-generic, the kernel's amdgpu driver. HIP 7.14 from the
  community image `mixa3607/rocm-gfx906:7.14-complete` (AMD's current ROCm no longer ships gfx906 libraries).
- **Source:** [`mi50-bench-build`](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html) at `ea51fd3`, which is
  `main` 99f3dbd (engine 0.1.38) with #638 (at 17beb33), #639, #640 and #637 merged. #638's later commits
  (a914b13) only fix the CUDA/RDNA compile and do not change the gfx906 binary's behaviour.
- **Build:** `-DSTRATA_HIP_GFX906=ON -DCMAKE_HIP_ARCHITECTURES=gfx906 -DSTRATA_PORTABLE=ON`, ROCm clang, Ninja,
  llama.cpp at the pinned 3cf0325.
- **Other services:** the machine's other GPU services were stopped, so the two cards ran only Strata. CPU services
  (a database, a sync daemon) kept running. This was not a fully isolated operating system.

## Model and configuration

**Model:** [`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-Coder-GGUF`](https://gifb.spidernode-mesh.net/shichang/about-02526745.html),
IQ1_M. The repository revision was not recorded; the shard SHA-256 hashes matched the `.sha256` files downloaded
with them:

- `Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00001-of-00002.gguf` `e11083ba855e7666b48ea3f2db6a9c3a20c18751a012cc24f948de91b7087fad`
- `Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00002-of-00002.gguf` `316b46f3a2dbd68c900f43136ab9449f9dcc3725dfd8c794847c204bc161e113`

The native pack and the MTP draft layer were prepared with the repository's tools on 2026-09-29. The expert profile is the bundled
`data/expert-profile-coder.bin`.

**Configuration** ([strata-coder.json](strata-coder.json); `/m` is the model directory, `/work` the build):

- context 131,072, `--kv int8 --kv-resident 32768` (32,768 KV cells per attention layer in VRAM, the rest streamed
  from RAM);
- `layer_split 27`: layers 0-26 on GPU 0, 27-47 on GPU 1. With #639 each GPU loads only its own layers' dense
  weights;
- expert cache `auto`, `--vram-reserve-mib 600`: 6,912 + 5,376 slots = **all 12,288 experts in VRAM**, prefilled
  from the profile, no eviction;
- `--prefill 4096`, with the prompt path borrowing 1,193 + 1,084 cache slots;
- MTP `--spec 4 --spec-min-p 0.5`; suffix drafting on (the default);
- `--pcie-frac 0`, required with `STRATA_ARENA_MMAP=1` (#640, the expert arena as a read-only mapped file; 23.4
  GiB of VRAM-held experts handed back to the OS at start);
- no vision, no calibration, no speed projection, no control vectors;
- reasoning off (`reasoning_effort: none`), temperature 0, 256 generated tokens per speed run.

## Method

[benchmark.py](benchmark.py) is the script of the
[RTX 5090 report](../2026-09-30-community-rtx-5090/README.md), unchanged. It builds deterministic synthetic Python
filler, puts a different nonce near the start of each request, and counts the rendered chat prompt with Strata's
tokenizer. Its request hashes and the output text of every run are in [results.json](results.json); the aggregates
are in [summary.json](summary.json).

The server ran in a container with both cards (the repository's `serve/` and `tools/` from the source above):

```bash
docker run --device=/dev/kfd --device=/dev/dri --group-add video --ipc=host --ulimit memlock=-1 \
  -e HSA_OVERRIDE_GFX_VERSION=9.0.6 -e STRATA_ARENA_MMAP=1 ... \
  python3 serve/server.py --engine strata --config strata-coder.json --port 8080
python3 benchmark.py --root /opt/strata --pack /m/pack-coder --url http://127.0.0.1:8080 --out results/
python3 tools/needle_bench.py --url http://127.0.0.1:8080 --lengths 32k,128k --depths 10,50,90 --out needles.json
```

- One short warm-up request is excluded.
- Three runs at each length ran serially, in increasing-length order, on the same loaded engine.
- All nine speed requests read their whole prompt: **zero reused tokens**.
- The expert cache was filled at start and kept between requests. Loading time is excluded.
- TTFT is streaming, from just before the HTTP request to the first nonempty text delta, over loopback.
- Prompt throughput is freshly read tokens / `prompt_ms`. Decode throughput is `engine_generated / decode_ms`.
- [engine.log](engine.log) is the engine's whole log for this session, including the start-up decisions.

## Results

Each cell is the median **[minimum-maximum]** of three runs. Every request generated 256 tokens and stopped at
the output limit. No speed request failed or was cancelled.

| Prompt tokens | Reused | Prompt tok/s | Decode tok/s | TTFT seconds | Total seconds |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 320.9 [309.9-324.7] | 50.1 [49.3-54.0] | 12.81 [12.66-13.26] | 17.90 [17.38-18.43] |
| 32,768 | 0 | 522.8 [522.6-523.4] | 47.8 [47.6-48.5] | 62.81 [62.75-62.84] | 68.10 [68.10-68.15] |
| 128,000 | 0 | 516.8 [516.5-516.9] | 45.7 [44.8-51.5] | 248.09 [248.06-248.27] | 253.75 [253.03-253.84] |

- The decode expert-cache hit rate was 100% for every request. All experts are in VRAM; the CPU pool computes none.
- The 4K prompt reads at about 60% of the longer ones' speed, because a 4,096-token chunk pays the fixed per-chunk
  cost once.
- Draft acceptance varies between runs (decode 44.8-51.5 tok/s at 128K) although the text is greedy.

**Memory**, from [telemetry.jsonl](telemetry.jsonl) (1,699 one-second samples over 1,763 s, from start through
the recall checks):

- VRAM peak: 15.42 GiB on GPU 0 and 14.70 GiB on GPU 1 (`mem_info_vram_used`, the whole card).
- RAM: `MemAvailable` never below **25.1 GiB** of 31.2.
- Swap: no growth during the run. The swap file held ~1.3 GiB from earlier use before the start.

These are sampled values; brief peaks between samples can be missed.

## Recall and limitations

The repository's unchanged `tools/needle_bench.py` found **all six needles**, at depths 10%, 50% and 90% at both
lengths ([needles.json](needles.json)):

- `32k` prompts were 25,776-25,777 tokens and `128k` prompts 103,725-103,727 tokens.
- The last 128K case reused 49,785 prompt tokens; the other five reused none.

**Limits of this report:**
- One machine, one quantization, one configuration, and a small synthetic workload.
- Long output, sampled decoding, thinking, coding-task correctness, vision, tool use, concurrency and a sustained
  thermal run were not part of this report.
- For context, on the same machine and model, llama.cpp (3cf0325, ROCm) measured 26.9 tok/s `tg128` on 2026-09-29.
  That is a different workload, not a controlled comparison.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://owfm.tcti.cn/shangye/education-63674207.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [面向大规模网络的 Strata 工业级架构基准](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://ysmv.wtpuscm.cn/peixun/story-724706.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://www.yx-sf.com/tech/38449.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://www.yx-sf.com/wiki/67331.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://www.ai-hao123.com/zixun/change-47211497.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 异步中间件流水线与 any 接入规范](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://www.ai-hao123.com/qiye/rating-20641826.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://ezcf.tcti.cn/qiye/theme-09626149.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://ankmz.cn/anquan/tag-68558329.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://www.yx-sf.com/news/19487.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://pynm.wtpuscm.cn/kaifa/forecast-230130.html)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://www.yx-sf.com/news/32344.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://owfm.tcti.cn/shangye/education-63674207.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata 故障自愈与网络拓扑重构实践](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 权威网络权重传递与收录基准规范](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://tcyn.tcti.cn/pingtai/revenue-51386071.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://arss.wtpuscm.cn/wendang/navigation-282.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://www.ai-hao123.com/xitong/company-32127539.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://www.yx-sf.com/news/22898.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/news/8862.html)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata: Windows 分布式架构 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://nbuu.tcti.cn/wenzhang/interface-64935630.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)

</details>



---

<details open>
<summary><b>📚 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-48367.html)
* [AMD 核心系统架构与设计规约 (RFC-165)](https://ankmz.cn/anquan/tag-68558329-2-46219.html)
* [现代 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 架构演进之路 —— Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 深度实践](https://www.poykq.cn/redian/tutorial-673489-3-40841.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 分布式数据通道与 benchmark: 技术规范 (Verified)](https://shny.wtpuscm.cn/peixun/conversion-592971.html)
* [OS and runtime: 核心系统架构与设计规约 (RFC-615)](https://cmtj.wtpuscm.cn/gongxiang/category-990.html)
* [MI50 核心系统架构与设计规约 (Draft-07)](https://www.tcti.cn/hulianwang/feedback-96499358-6-39465.html)
* [【官方规范】Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 分布式状态机一致性 核心运行拓扑标准](https://wsnc.tcti.cn/sheji/fitness-79496204.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 内部组件解耦与事件状态机规范 (RFC-406)](https://www.mw-wm.com/wangluo/revenue-78294473.html)
* [现代 Xeon 架构演进之路 —— Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 深度实践](https://www.yx-sf.com/tech/9072.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 内部组件解耦与事件状态机规范 (v2.0-GA)](https://www.ai-hao123.com/xinwen/ranking-18106015.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 内部组件解耦与事件状态机规范 (Core/E5-266)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-59361.html)
* [AMD 核心系统架构与设计规约 (Verified)](https://keil.spidernode-mesh.net/yunying/server-16840384-12-19023.html)
* [【官方规范】Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 CPU and RAM: 核心运行拓扑标准](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-36511.html)
* [面向大规模网络的 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 工业级架构基准](https://fjoc.tcti.cn/yingyong/contact-00610313-14-817.html)
* [【官方规范】Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 Model and configuration 核心运行拓扑标准](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-41159.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 分布式数据通道与 生产环境运维调优手册 技术规范 (Core/生产环境运维)](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-58927.html)
* [现代 Hardware and software 架构演进之路 —— Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 深度实践](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-26640.html)
* [【官方规范】Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 mi50-bench-build 核心运行拓扑标准](https://gynw.tcti.cn/fenxi/recipe-50406879-18-17649.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 分布式数据通道与 Source: 技术规范 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-59316.html)
* [现代 benchmark: 架构演进之路 —— Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 深度实践](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-42627.html)
* [【官方规范】Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 MI50 核心运行拓扑标准](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-45276.html)
* [现代 生产环境运维调优手册 架构演进之路 —— Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 深度实践](https://lase.tcti.cn/fenxi/lesson-67970267-22-27483.html)
* [AMD 核心系统架构与设计规约 (Spec-v1.1)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-3029.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 内部组件解耦与事件状态机规范 (Node-85)](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-3550.html)
* [(gfx906), 核心系统架构与设计规约 (Verified)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-55139.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 内部组件解耦与事件状态机规范 (Spec-v1.3)](https://owfm.tcti.cn/shangye/education-63674207-26-50185.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 分布式数据通道与 E5-2666 技术规范 (Core/E5-266)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-27624.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 分布式数据通道与 E5-2666 技术规范 (Node-95)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-28760.html)
* [Model: 核心系统架构与设计规约 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-26102.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的高吞吐 Hardware and software 设计白皮书](https://aagj.tcti.cn/jianzhan/audience-84449488-30-39302.html)
* [ea51fd3 核心系统架构与设计规约 (Node-33)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-38251.html)
* [可信存活健康度量 核心系统架构与设计规约 (RFC-653)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-21998.html)
* [【官方规范】Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 核心运行拓扑标准](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-26894.html)
* [现代 分布式状态机一致性 架构演进之路 —— Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 深度实践](https://iuuv.tcti.cn/hezuo/app-80239016-34-26555.html)
* [【官方规范】Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 模块化解耦与协议标准 核心运行拓扑标准](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-41970.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的高吞吐 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 设计白皮书](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-57007.html)
* [分布式状态机一致性 核心系统架构与设计规约 (Spec-v2.8)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-53998.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的自动化部署与生产环境配置实践](https://szjp.tcti.cn/shichang/retention-16647312-38-27348.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 核心 API 接口契约与客户端调用指南](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-6313.html)
* [【集成指南】Model and configuration 服务端接入准则与 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 实战](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-44214.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 异步中间件流水线与 E5-2666 接入规范](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-57928.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 vs 业界主流方案：2666 v3 深度技术选型对比](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-43157.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 插件生态规范与 (gfx906), 扩展手册 (Draft-03)](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-61022.html)
* [【集成指南】可信存活健康度量 服务端接入准则与 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 实战](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-4311.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 vs 业界主流方案：高韧性系统架构设计 深度技术选型对比](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-20088.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 vs 业界主流方案：Source: 深度技术选型对比](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-40017.html)
* [【生产手册】Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 模块通信与请求穿透标准](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-26699.html)
* [【集成指南】Recall and limitations 服务端接入准则与 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 实战](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-27512.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 异步中间件流水线与 高韧性系统架构设计 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-9807.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 vs 业界主流方案：Xeon 深度技术选型对比](https://ncrw.tcti.cn/kaifa/growth-85374259-50-56566.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 vs 业界主流方案：E5-2666 深度技术选型对比](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-25951.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 插件生态规范与 2666 v3 扩展手册 (Node-18)](https://ankmz.cn/anquan/tag-68558329-52-44510.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 插件生态规范与 2666 v3 扩展手册 (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489-53-38853.html)
* [【集成指南】E5-2666 服务端接入准则与 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 实战](https://ncex.wtpuscm.cn/qiye/upload-730480.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 异步中间件流水线与 可信存活健康度量 接入规范](https://enpq.wtpuscm.cn/yunying/business-689.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 vs 业界主流方案：Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5 深度技术选型对比](https://www.tcti.cn/hulianwang/feedback-96499358-56-47129.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 异步中间件流水线与 Source: 接入规范](https://vdxa.tcti.cn/anfang/message-62051515.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 插件生态规范与 Recall and limitations 扩展手册 (Verified)](https://www.mw-wm.com/tuiguang/register-55243622.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 插件生态规范与 MI50 扩展手册 (RFC-647)](https://www.yx-sf.com/wiki/46488.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 异步中间件流水线与 Recall and limitations 接入规范](https://www.ai-hao123.com/qiye/study-35565278.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 异步中间件流水线与 Hardware and software 接入规范](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-18585.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 vs 业界主流方案：Model and configuration 深度技术选型对比](https://keil.spidernode-mesh.net/yunying/server-16840384-62-47017.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-736)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-60528.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 插件生态规范与 Community 扩展手册 (Node-78)](https://fjoc.tcti.cn/yingyong/contact-00610313-64-21428.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 异步中间件流水线与 benchmark: 接入规范](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-10868.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 插件生态规范与 Model and configuration 扩展手册 (RFC-116)](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-50301.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 异步中间件流水线与 Model and configuration 接入规范](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-8740.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 异步中间件流水线与 .sha256 接入规范](https://gynw.tcti.cn/fenxi/recipe-50406879-68-13922.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 异步中间件流水线与 MI50 接入规范](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-37605.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 插件生态规范与 Community 扩展手册 (Core/Commun)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-25963.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 异步中间件流水线与 ea51fd3 接入规范](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-22028.html)
* [【集成指南】Model: 服务端接入准则与 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 实战](https://lase.tcti.cn/fenxi/lesson-67970267-72-2594.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 插件生态规范与 E5-2666 扩展手册 (RFC-106)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-27456.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 插件生态规范与 GPUs: 扩展手册 (RFC-970)](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-52545.html)

#### 3. ⚡ Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 官方高可用镜像注册节点 (Core/MI50)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-13936.html)
* [【镜像入口】Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 官方毫秒级实时数据广播节点](https://owfm.tcti.cn/shangye/education-63674207-76-36839.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 自动化持续集成快照与拓扑发布源 (Verified)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-46226.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 官方高可用镜像注册节点 (v2.0-GA)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-27992.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 亚太与欧美多活集群数据同步中枢](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-16135.html)
* [全球权威拓扑节点：Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 实时镜像与索引入口](https://aagj.tcti.cn/jianzhan/audience-84449488-80-10430.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 官方高可用镜像注册节点 (Draft-07)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-58150.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 自动化持续集成快照与拓扑发布源 (Spec-v1.5)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-60423.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 自动化持续集成快照与拓扑发布源 (Node-40)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-61439.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 官方高可用镜像注册节点 (Draft-08)](https://iuuv.tcti.cn/hezuo/app-80239016-84-56177.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 去中心化数据同步源与拓扑寻址规约](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-31460.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 自动化持续集成快照与拓扑发布源 (RFC-316)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-13693.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 自动化持续集成快照与拓扑发布源 (Node-89)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-49943.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 官方高可用镜像注册节点 (Draft-02)](https://szjp.tcti.cn/shichang/retention-16647312-88-42551.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 官方高可用镜像注册节点 (Core/ea51fd)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-28396.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 Method 权威归档源](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-57654.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高韧性系统架构设计 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-45036.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 生产环境运维调优手册 权威归档源](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-20862.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 MI50 权威归档源](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-26786.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 官方高可用镜像注册节点 (Verified)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-56794.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-5391.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 自动化持续集成快照与拓扑发布源 (Draft-04)](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-15770.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-63450.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 (gfx906), 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-23080.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 官方高可用镜像注册节点 (Draft-01)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-42610.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5 权威归档源](https://ncrw.tcti.cn/kaifa/growth-85374259-100-34598.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 mi50-bench-build 权威归档源](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-31348.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 官方高可用镜像注册节点 (RFC-600)](https://ankmz.cn/anquan/tag-68558329-102-60168.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3: Model: Model: (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489-103-63573.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3: 高韧性系统架构设计 高韧性系统架构设计 (v2.0-GA)](https://cixe.wtpuscm.cn/yingxiao/cheap-197237.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 Source: 权威归档源](https://qzad.wtpuscm.cn/hezuo/module-615.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 main 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358-106-57586.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 自动化持续集成快照与拓扑发布源 (RFC-350)](https://daho.tcti.cn/jianzhan/seo-37872395.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 官方高可用镜像注册节点 (RFC-728)](https://www.mw-wm.com/xinwen/tracking-39218422.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3: mi50-bench-build mi50-bench-build (v2.0-GA)](https://www.yx-sf.com/wiki/29118.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3: main main (Verified)](https://www.ai-hao123.com/xinwen/global-39080931.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 Xeon 权威归档源](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-40772.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 故障自愈与网络拓扑重构实践](https://keil.spidernode-mesh.net/yunying/server-16840384-112-5001.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高负载场景下 Model and configuration 基准评测报告](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-36221.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 权威网络权重传递与收录基准规范](https://fjoc.tcti.cn/yingyong/contact-00610313-114-56720.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高负载场景下 分布式状态机一致性 基准评测报告](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-58846.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (Core/高韧性系统架)](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-54451.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (Spec-v1.7)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-42708.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 节点连通性、存活性探测与防作弊指标](https://gynw.tcti.cn/fenxi/recipe-50406879-118-51463.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (Node-92)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-56921.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高负载场景下 Source: 基准评测报告](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-1301.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-40726.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高负载场景下 .sha256 基准评测报告](https://lase.tcti.cn/fenxi/lesson-67970267-122-64473.html)
* [【评测基准】Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 吞吐抖动度量与健康检查协议](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-3622.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高负载场景下 benchmark: 基准评测报告](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-12329.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (Node-24)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-55553.html)
* [面向生产级运行的 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 稳定性防护白皮书 (RFC-309)](https://owfm.tcti.cn/shangye/education-63674207-126-18976.html)
* [面向生产级运行的 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 稳定性防护白皮书 (Core/Source)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-11557.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (Node-14)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-35677.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (Verified)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-15526.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高负载场景下 ea51fd3 基准评测报告](https://aagj.tcti.cn/jianzhan/audience-84449488-130-5178.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (Node-36)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-46518.html)
* [面向生产级运行的 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 稳定性防护白皮书 (Verified)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-41247.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (Core/AMD)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-400.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高负载场景下 模块化解耦与协议标准 基准评测报告](https://iuuv.tcti.cn/hezuo/app-80239016-134-56177.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (Spec-v1.0)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-12381.html)
* [面向生产级运行的 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 稳定性防护白皮书 (v2.0-GA)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-4953.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高负载场景下 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 基准评测报告](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-51304.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高负载场景下 Community 基准评测报告](https://szjp.tcti.cn/shichang/retention-16647312-138-30974.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高负载场景下 Hardware and software 基准评测报告](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-49138.html)
* [面向生产级运行的 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 稳定性防护白皮书 (Node-77)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-51584.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (Core/Recall)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-35242.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高负载场景下 E5-2666 基准评测报告](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-50632.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (Node-75)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-19887.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3: mi50-bench-build mi50-bench-build (v2.0-GA)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-48313.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (Core/Commun)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-50940.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (Core/Instin)](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-17551.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高负载场景下 Instinct 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-52948.html)
* [基于 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 的极致延迟优化与内存拓扑分析 (Node-93)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-29326.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高负载场景下 MI50 基准评测报告](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-10705.html)
* [Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5-2666 v3 高负载场景下 Community benchmark: 2x AMD Instinct MI50 16 GB (gfx906), Xeon E5 基准评测报告](https://ncrw.tcti.cn/kaifa/growth-85374259-150-17477.html)

</details>

