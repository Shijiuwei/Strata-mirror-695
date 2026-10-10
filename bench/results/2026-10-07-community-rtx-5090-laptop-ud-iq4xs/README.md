# Community benchmark: RTX 5090 Laptop, Core Ultra 9 275HX, 188 GB RAM — Unsloth UD-IQ4_XS (first NVIDIA measurement)

Measured on 2026-10-06/07 on a laptop called s18. This tests Strata 0.1.40 (commit
`1735d64`) with **Unsloth UD-IQ4_XS** (`unsloth/Qwen3.8-Flash-Next-GGUF`, revision `38bb39e`),
one GPU, 262,144-token context. Per docs/UNSLOTH_Q4.md this pack had "not been measured on
NVIDIA yet"; this is a first data point.

**Headline:** 22–33 tok/s decode on the Chinese greedy prompt (cold starts at the low end;
see the raw lines), versus 53–59 tok/s for IQ3_S on the same machine and settings — the
~4-bit pack is about **2x slower** here, and the decode-time
breakdown says the GPU itself is the bottleneck (its Q8_0 dense side and larger expert blobs),
not the CPU expert pool, not PCIe, and not RAM residency.

## Hardware and software

- NVIDIA GeForce RTX 5090 Laptop GPU; 24,463 MiB VRAM; PCIe Gen 5, x16. Driver 595.91.07.
- Intel Core Ultra 9 275HX: 24 cores (8 P + 16 E, no HT). The engine picks AVX2 (no AVX-512).
- 192 GB RAM installed; 188 GiB usable. NVMe SSDs (the GGUFs on one, the OS on another).
- Ubuntu 24.04.5 LTS, kernel 7.0.0-38-generic; CUDA 13.3 toolkit; locally compiled engine
  (`CMAKE_CUDA_ARCHITECTURES=120`). Strata commit `1735d64`, engine 0.1.40.

## Model and configuration

Three shards, 93.7 GB, all SHA-256-verified against the table in docs/UNSLOTH_Q4.md
(`5ce89370…`, `577a38a2…`, `d4634e6d…`). The installer prepared the pack with `--compat-bf16`
and reused the shared MTP draft layer with the CJK draft vocabulary (106,299 ids, 212.9 MiB
draft head). Launch configuration: [strata-unsloth-ud-iq4_xs.json](strata-unsloth-ud-iq4_xs.json).

- Context 262,144; INT8 KV; 32,768 KV cells per QSA layer resident on GPU.
- Expert cache `auto`: 6,030 slots, 13.59 GiB VRAM (IQ3_S on the same box: 8,000 slots, 15.17 GiB).
- `--resident-budget-gib 55`: 41.84 GiB of experts pinned in RAM (every expert the GPU cache
  does not hold); adaptive swaps exchange them with the VRAM tier.
- `--pool-workers 15` chosen by setup for this hybrid CPU (8P+16E).
- As installed: `--spec 4 --spec-min-p 0.5`, no `--pcie-frac`. After the sweep below I set
  `--pcie-frac 0.55 --spec-min-p 0.70` by hand.

Startup layout (from [bench.log](bench.log)):

```
expert cache 6030 slots, 13.59 GiB of VRAM
resident RAM mode: 41.84 GiB of experts in RAM (page-locked), 6030 in the GPU cache
mtp: draft head over 106299 tokens (212.9 MiB)
213 MiB of VRAM free with everything loaded - LOW: requests may stall
```

## Workload

OpenAI chat requests against the local server, one at a time, reasoning default (high),
`max_tokens=512`. Two prompt types, several runs each after a warm-up run (the first request
after every server start is ~20% slower here):

- Chinese prose: "请用中文写一份关于上海城市发展的详细介绍，分五个部分展开，每部分至少一百五十字。" (temperature 1.0, and greedy runs of the same prompt)
- English code: "Write a Python module implementing a red-black tree with insert, delete and search, with detailed comments explaining each step." (temperature 1.0)

All tok/s figures are the engine's own `strata serve: prompt …` log lines, decode only
(prefill reported separately there). Raw lines in [bench.log](bench.log).

## Results

As installed (`--spec-min-p 0.5`, no `--pcie-frac`):

| Run | Decode tok/s | Drafts accepted |
| --- | --- | --- |
| Chinese, temp 1.0 | 29.6 | 241/423 (57%) |
| English code, temp 1.0 | 21.3 | 287/406 (71%) |
| Chinese, greedy (warm 1) | 21.6 | 251/414 |
| Chinese, greedy (warm 2) | 28.1 | 234/424 |

Calibration sweep (`./setup.sh --calibrate`, engine 0.1.40, this pack):

| PCIe share | tok/s | | draft floor | tok/s |
| --- | --- | --- | --- | --- |
| 0.00 | 26.6 | | 0.30 | 23.3 |
| 0.20 | 27.6 | | 0.50 | 26.4 |
| 0.25 | 25.1 | | 0.70 | 27.3 |
| 0.35 | 27.5 | | | |
| 0.55 | **28.2** | | | |
| 0.75 | 21.9 | | | |

With `--pcie-frac 0.55 --spec-min-p 0.70` applied, warm greedy runs of the Chinese prompt:
**27.0, 28.8, 33.0 tok/s**.

Decode-time breakdown (`STRATA_DECODE_TIMING=1`, warm run at 33.0 tok/s):

```
291 windows, 1.76 tokens/window, 53.39 ms/window =
  verify 36.85 (GPU-reach wait 27.02 + per-layer host 8.30 [CPU experts 7.17] + stage 0.21)
  + commit/emit 0.06 + draft 4.10;
per layer-window: CPU experts 1.16, VRAM hits 18.68, PCIe 0.81
```

i.e. most of the window is spent waiting on the GPU (27 ms of 53 ms), while the CPU expert
pool contributes ~7 ms and PCIe expert copies ~0.8 ms per layer-window. That points at the
pack's larger GPU-side work (Q8_0 dense projections, IQ4_NL/IQ3_S experts) rather than at
expert delivery.

IQ3_S measured on the same machine for contrast (same protocol, calibrated settings
`--pcie-frac 0.35 --spec-min-p 0.70 --pool-workers 15`): 53.1–59.1 tok/s warm on the same
two prompts, 8,000 VRAM expert slots, ~1.6 GiB VRAM free.

## Issues and notes worth acting on

1. **`--calibrate` aborted mid-sweep on this configuration.** During the "23 CPU workers"
   restart the MTP draft head no longer fit VRAM (212.9 MiB needed, ~213–274 MiB free), so the
   whole tuning run failed and kept defaults:

   ```
   strata mtp: the draft head over 106299 tokens needs 213 MiB of VRAM and 274 MiB is free.
   strata serve: mtp: the draft head does not fit): the default settings stay
   ```

   Workaround: apply the sweep winners by hand (they are printed before the failure), or
   start once with a smaller `--draft-vocab` and re-run calibration. It may be worth making
   the sweep treat a non-fitting restart as "this setting loses" instead of failing the run.

2. **`STRATA_EXCHANGE_ROTATE=1` is unavailable for this pack** ("requires equal-size expert
   blocks and a fully mapped/pinned RAM complement" — the blocks are IQ3_S/IQ4_NL mixes), so
   swaps keep the host copy path. Fine, just noted since the doc says rotation helps swap
   churn; something to know for mixed-format packs.

3. **213 MiB of VRAM free** with everything loaded (262K context, CJK draft head). The engine
   warns requests may stall and suggests `--vram-reserve-mib ~999` or a smaller context. I did
   not hit an actual stall in ~30 short requests, but agentic long-context use has little
   headroom; on 24 GB cards the 262K context + CJK draft vocab combination leaves almost
   nothing.

4. **Adaptive swap counts grow during use** (5,369 → 20,938 exchanges over four requests),
   and decode warms up over the first requests (21.6 → 28.1 → 33.0 tok/s in consecutive
   runs). Short-lived servers will sit at the low end of the range above.

5. Quality side: on a small 12-item suite (math, logic, code execution, Chinese knowledge,
   instruction following, a long-context needle), UD-IQ4_XS passed 12/12, indistinguishable
   from IQ3_S at that difficulty — the speed/quality trade here is about capability on hard
   long-horizon work, which this report does not measure.

## Reproduce

```bash
./setup.sh --setup --family unsloth --model UD-IQ4_XS --yes --no-start --context 262144 --kv int8
# start script: run-unsloth-ud-iq4_xs.sh (config strata-unsloth-ud-iq4_xs.json)
# bench: POST /v1/chat/completions, the two prompts above, max_tokens 512;
# read decode tok/s from strata-unsloth-ud-iq4_xs.log ("strata serve: prompt ...")
```

Trial logs: [bench.log](bench.log) (startup + bench lines + decode timing), config:
[strata-unsloth-ud-iq4_xs.json](strata-unsloth-ud-iq4_xs.json).


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://cbjs.wtpuscm.cn/pingtai/api-202821.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://wiaz.wtpuscm.cn/yingxiao/communication-781474.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://utfy.wtpuscm.cn/gongju/affordable-055735.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://bjxs.wtpuscm.cn/zhizhu/device-331155.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://pzin.wtpuscm.cn/yunying/login-303062.html)
* [面向大规模网络的 Strata 工业级架构基准](https://viog.wtpuscm.cn/jiaoliu/button-675067.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://tttj.wtpuscm.cn/yingxiao/tracking-150253.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://hste.wtpuscm.cn/keji/marketing-122.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://niaj.wtpuscm.cn/xuexi/food-293950.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://hyow.wtpuscm.cn/yunsuan/analysis-101080.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://kunk.wtpuscm.cn/guanjianci/story-766741.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://bniu.wtpuscm.cn/yunsuan/update-659602.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://mxii.wtpuscm.cn/chuangxin/expensive-705838.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://alcs.wtpuscm.cn/xuexi/supplier-853012.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://tsjd.wtpuscm.cn/sheji/screen-786543.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://fapz.wtpuscm.cn/pingtai/tracking-289712.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://xjtl.wtpuscm.cn/guanjianci/link-625424.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://xpkc.wtpuscm.cn/yingxiao/wellness-987643.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://lcxj.wtpuscm.cn/peixun/change-179637.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://nzie.wtpuscm.cn/tuiguang/whitepaper-904926.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://nkzs.wtpuscm.cn/suanfa/objective-256074.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://zqzy.wtpuscm.cn/yinqing/topic-645488.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://bnfk.wtpuscm.cn/xuexi/update-852071.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://wgvg.tcti.cn/keji/help-02891460.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://hiud.tcti.cn/yingxiao/download-51991697.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://qurb.tcti.cn/zhizhu/blog-09606751.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://qkap.tcti.cn/jianzhan/content-35549410.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://gxen.tcti.cn/yunying/promotion-54371093.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://wido.tcti.cn/chuangxin/unsubscribe-98266609.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://cieh.tcti.cn/xuexi/web-63311244.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://sqyx.tcti.cn/liuliang/roi-42704567.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://ippd.tcti.cn/jiaocheng/comment-69522913.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://lvom.tcti.cn/paiming/collaborate-79315855.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://smam.tcti.cn/shangye/design-47993302.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://wuiw.tcti.cn/suanfa/entertainment-53031270.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://fzdk.tcti.cn/pingce/music-11313935.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://olco.tcti.cn/anli/premium-72020107.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://xoac.tcti.cn/paiming/chapter-15858503.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://ygso.tcti.cn/wendang/user-72072872.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://dbnh.tcti.cn/yunying/contact-05794455.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://gopn.wtpuscm.cn/wenzhang/chapter-409797.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/peixun/excellence-95595102.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/1752)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/zhinan/ranking-89598556.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://tiil.tcti.cn/wenzhang/navigation-21683120.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://xklk.tcti.cn/shuju/success-90842086.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://lelk.wtpuscm.cn/youhua/terms-515742.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://jdoj.wtpuscm.cn/suanfa/faq-758021.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://utwv.wtpuscm.cn/anfang/global-295206.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://zxhx.wtpuscm.cn/gongju/page-885525.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://xxpz.wtpuscm.cn/anli/social-628718.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://mguh.wtpuscm.cn/yanjiu/retention-569976.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://ixul.wtpuscm.cn/wenzhang/share-604454.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://wcub.wtpuscm.cn/zhineng/podcast-236.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://lfwb.wtpuscm.cn/gongju/efficiency-300653.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://gbnu.wtpuscm.cn/zhineng/food-758862.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://jhll.wtpuscm.cn/suanfa/deal-670142.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://htoa.wtpuscm.cn/qiye/layout-269774.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://alct.wtpuscm.cn/jishu/satisfaction-030006.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://pkjo.wtpuscm.cn/huodong/audience-426931.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://qpod.wtpuscm.cn/shichang/feedback-948925.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://ltub.wtpuscm.cn/yunying/admin-484735.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://iguj.wtpuscm.cn/gongju/health-172055.html)
* [Strata 异步中间件流水线与 any 接入规范](https://ghce.wtpuscm.cn/kuangjia/profit-308041.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://zyym.wtpuscm.cn/fenxi/sale-162682.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://yjfx.wtpuscm.cn/wendang/theme-204934.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://ayrf.wtpuscm.cn/shangye/social-959318.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://spme.wtpuscm.cn/yingyong/milestone-205821.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://lkut.wtpuscm.cn/huodong/calculator-762248.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://xthz.tcti.cn/pingtai/keyword-15572095.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://svsx.tcti.cn/wenzhang/extension-29867988.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://pgzr.tcti.cn/wenzhang/products-10227347.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://xkmf.tcti.cn/tuiguang/demographic-41957425.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://zoqk.tcti.cn/baogao/faq-70923882.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://itmh.tcti.cn/guanjianci/schedule-53773076.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://kdcz.tcti.cn/suanfa/meeting-27853997.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://gcbc.tcti.cn/shuju/products-41336955.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://tdkv.tcti.cn/kuangjia/label-71121062.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://cxyk.tcti.cn/liuliang/community-04951678.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://juch.tcti.cn/shangye/digital-30632241.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://qgrg.tcti.cn/zhineng/behavior-80087222.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://ncry.tcti.cn/kaifa/management-93450288.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://ioog.tcti.cn/suanfa/careers-25763480.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://qnrn.tcti.cn/yingxiao/ai-10223783.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://svrc.tcti.cn/huodong/api-20726866.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://wqou.tcti.cn/shichang/subject-62845190.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://yfxt.wtpuscm.cn/fenxi/partner-610303.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/yunying/behavior-09318937.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/wiki/92298)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/gongju/reporting-11934843.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://hgdq.tcti.cn/baogao/analytics-67729134.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://ortz.tcti.cn/ziyuan/image-79860291.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://vkkj.wtpuscm.cn/qiye/download-139452.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://evem.wtpuscm.cn/chuangxin/discount-107811.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://wavb.wtpuscm.cn/tuiguang/security-718700.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://bpnm.wtpuscm.cn/chanpin/project-510549.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://wqpl.wtpuscm.cn/xitong/wellness-004040.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://ujcf.wtpuscm.cn/peixun/data-096201.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://brwh.wtpuscm.cn/tuiguang/mobile-028639.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://hgxz.wtpuscm.cn/zhinan/api-943.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://crab.wtpuscm.cn/jiaoliu/kpi-748696.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://bcjt.wtpuscm.cn/shuju/profit-382090.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://cnhv.wtpuscm.cn/yingyong/luxury-331589.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://lwhn.wtpuscm.cn/yunying/metric-328186.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://ydtw.wtpuscm.cn/fenxi/plugin-427562.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://rtjq.wtpuscm.cn/xuexi/identity-072308.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://vlmy.wtpuscm.cn/fenxi/conference-122968.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://oqhf.wtpuscm.cn/yinqing/tool-615424.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://tugv.wtpuscm.cn/yingxiao/data-278266.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://esjt.wtpuscm.cn/wendang/conversion-167658.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://yqct.wtpuscm.cn/yinqing/app-248188.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://ckil.wtpuscm.cn/anfang/integration-583936.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://ftmz.wtpuscm.cn/youhua/tactic-768271.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://rgdc.wtpuscm.cn/pingce/economy-502174.html)
* [Strata 故障自愈与网络拓扑重构实践](https://bclt.wtpuscm.cn/shichang/alert-852238.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://ukol.tcti.cn/jiaoliu/collaboration-99428263.html)
* [Strata 权威网络权重传递与收录基准规范](https://vcdo.tcti.cn/zixun/innovation-62524468.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://mllp.tcti.cn/yinqing/progress-29417776.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://babg.tcti.cn/zixun/seminar-21469561.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://eimv.tcti.cn/yunsuan/follow-97581986.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://cyas.tcti.cn/yinqing/market-10735669.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://pngd.tcti.cn/jishu/webinar-47687856.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://ejlt.tcti.cn/qiye/brand-12140368.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://kucn.tcti.cn/fenxi/category-78661337.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://nrem.tcti.cn/zhineng/faq-03032624.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://btdv.tcti.cn/huodong/presentation-10462661.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://rajn.tcti.cn/baogao/domain-32493779.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://zrei.tcti.cn/fenxi/cheap-23411965.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://hrlt.tcti.cn/sheji/supplier-33962042.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://wifl.tcti.cn/zixun/optimization-26339711.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://eruh.tcti.cn/huodong/success-61834397.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://ivhe.tcti.cn/yanjiu/customer-40417227.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://elft.wtpuscm.cn/suanfa/register-802632.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/tuiguang/hotel-58038103.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/news/22498)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/zhizhu/collaborate-88521402.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://wnnf.tcti.cn/zhinan/theme-70151643.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://fbbe.tcti.cn/yunsuan/data-81252757.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://fepp.wtpuscm.cn/yingyong/comment-728974.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://lfru.wtpuscm.cn/yanjiu/version-374354.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://tuty.wtpuscm.cn/baogao/deal-740309.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://rdnd.wtpuscm.cn/xitong/tool-999398.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://peqd.wtpuscm.cn/yunying/download-306020.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://dduy.wtpuscm.cn/chuangxin/forecast-358014.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://tsbj.wtpuscm.cn/jiaoliu/change-815685.html)
* [Strata: Windows 分布式架构 (Verified)](https://rfkk.wtpuscm.cn/suanfa/resolution-756.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://hhxl.wtpuscm.cn/tuiguang/identity-425821.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://cvzo.wtpuscm.cn/peixun/trading-348210.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://vpbw.wtpuscm.cn/yunsuan/feedback-233791.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://mvmz.wtpuscm.cn/liuliang/sales-627990.html)

</details>

