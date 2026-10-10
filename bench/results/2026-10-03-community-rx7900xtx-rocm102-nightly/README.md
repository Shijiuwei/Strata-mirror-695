# Community benchmark on AMD Radeon RX 7900 XTX 24 GB — ROCm 7.1.1 vs ROCm 10.2 nightly (serve path, Qwen3.8-Flash-Next IQ3_S)

Measured 2026-10-03/04 by xyzzing. What was tested: the same serving config on two ROCm stacks —
the packaged ROCm 7.1.1 and a ROCm 10.2 nightly SDK — measuring warm steady-state decode on a
repeated identical request. Main limitation: the nightly build has no matching gfx1100 hipBLASLt
tuning table (see the confound under Results), and per-request raw server logs were lost to a log
truncation on restart — the decode values below were captured at request time from the API's
`timings` field.

## Hardware and software

- GPU: AMD Radeon RX 7900 XTX 24 GB (gfx1100, RDNA3); CPU: Ryzen 9 7900X (12C/24T); RAM: 96 GB DDR5; storage: NVMe ext4; PCIe Gen4 x16:
- OS: Fedora 44 (kernel 7.2.7); driver: amdgpu (in-tree); ROCm: packaged 7.1.1 vs a ROCm 10.2 nightly SDK (`libamdhip64.so.7.17.26392`, verified loaded via /proc/PID/maps):
- Strata: HIP source builds of the xyzzing integration line — engine 0.1.38 (`99f3dbd` base) for both stacks (one binary per ROCm stack):
- Background workloads: KDE/Wayland desktop + browser (the reason for the VRAM reserve below); no power limits:

## Model and configuration

- Model: Qwen3.8-Flash-Next GSQ-RCO (ISTA), IQ3_S GGUF (2 shards, hash `692a40d0…` shard 1):
- No vision; expert cache `auto` pre-filled from the shipped routing profile; `--vram-reserve-mib 3072`:
- Context 65536; KV int8 with `--kv-resident 32768`; prefill chunk 8192; MTP (`--mtp`, draft layer loaded); sampling: greedy (temperature 0); no calibration; no low-RAM mode:

```text
# serve (identical args on both stacks; only ROCM_PATH/LD_LIBRARY_PATH differ)
<venv python> serve/server.py --engine strata --config <cfg>.json --port <8081|8082> --host 127.0.0.1
# cfg args: --pack packs/iq3_s --native <shard1> --ple-gguf <shard2> --mtp mtp/rt
#   --expert-profile expert-profile.bin --expert-cache auto --spec 3 --kv int8 --kv-resident 32768
#   --prefill 8192 --vram-reserve-mib 3072 --max-context 65536
#   env: STRATA_PREFILL_RING=96 STRATA_GR_V3=1 STRATA_SELECT_WMMA=1 (see confound)
# request: POST /v1/chat/completions {"messages":[{"role":"user","content":
#   "Name the capital of France in one word."}],"max_tokens":200,"temperature":0}
```

## Method

One identical request repeated back-to-back per stack (≈61 prompt tokens incl. template, 200
generated tokens, greedy), served over loopback by `serve/server.py`; decode tok/s read from the
response's `timings.predicted_per_second` field (committed tokens / decode interval — not derived
from total request time). The request sequence intentionally measures the warm-up curve: request 1
is cold (fresh engine + cold expert tier), later requests are warm. Stack order: all ROCm 7
requests, then engine stop, then all ROCm 10.2 requests. No prompt reuse beyond the engine's own
cache (the same short prompt each time). Memory: engine VRAM 22.8/24 GiB with the 3072 MiB
desktop reserve (compositor + browser co-resident). **Raw server logs were truncated by a restart
between stacks; the decode values below were captured at request time** (see `runs.json`, which
also marks one stalled request).

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Decode tok/s median and range |
| --- | ---: | ---: | ---: | ---: | --- |
| ROCm 7.1.1, warm (requests 4–9) | ~61 | not recorded per run | 200 | 6 | 80.4, 78.8–81.6 |
| ROCm 7.1.1, request 1 (cold-warm-up) | ~61 | 0 | 200 | 1 | 59.3 |
| ROCm 10.2 nightly, warm (requests 6–10) | ~61 | not recorded per run | 200 | 5 | 88.4, 83.8–88.9 |
| ROCm 10.2 nightly, request 1 (cold restart) | ~61 | 0 | 200 | 1 | 58.6 |
| → toolchain delta at warm steady state | | | | | **+10.0% (88.4 vs 80.4)** |

Secondary, 128K-context one-shot (131071 prompt tokens, 256 generated, one run each — reported for
completeness, not a median): decode 62.0 (ROCm 7.1.1) vs 61.0 (nightly) tok/s — decode is O(1) in
context on this model and toolchain-neutral at this shape. **Prefill at 128K is confounded**: the
nightly's hipBLASLt is version 100500 and no gfx1100 table ships for it (100100/100200 only), so
the engine falls back to plain hipBLAS (prefill 921 vs 1535 tok/s; the version-mismatch mechanism
is documented in the AMD HIP docs). Prompt throughput 1K/4K tiers on 0.1.38: 850 / 1,366 tok/s
(earlier sweep, engine 0.1.31; `runs.json` has the 128K pair).

Failures: one stalled request on the nightly (29.0 tok/s, request 2 after a cold restart — the
documented background-GPU stall pattern; marked and excluded, kept in `runs.json`).

## Correctness and limitations

Correctness: every request returned the identical greedy answer ("Paris", finish=stop, 200-token
cap) on both stacks. Limitations: single GPU; a desktop co-resident (that is the point of the VRAM
reserve); the request is short (~10 content tokens + template), so absolute tok/s reflects decode
of a chat-sized answer, not long-completion behavior — decode is flat in context for this model
(GDN linear attention), measured 1K→128K elsewhere. Untested: k8v4 KV (incompatible with
`--kv-resident` on this line), the rocWMMA select arm (compiled out of the nightly build), long
multi-turn conversations, and prompt-processing comparisons at other hipBLASLt versions.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://lxzu.wtpuscm.cn/gongju/collaborate-274661.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://guvx.wtpuscm.cn/huodong/update-461675.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://fftl.wtpuscm.cn/peixun/engagement-172653.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://uqch.wtpuscm.cn/tuiguang/report-696988.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://bwit.wtpuscm.cn/paiming/traffic-918910.html)
* [面向大规模网络的 Strata 工业级架构基准](https://tcsv.wtpuscm.cn/gongju/expense-696251.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://qrhr.wtpuscm.cn/yunying/rating-267833.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://aida.wtpuscm.cn/peixun/prospect-362.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://wvio.wtpuscm.cn/zhinan/innovation-390496.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://nswe.wtpuscm.cn/fuwu/investment-232326.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://lyqg.wtpuscm.cn/kaifa/market-399615.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://tmln.wtpuscm.cn/tuiguang/segment-951220.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://amye.wtpuscm.cn/zhinan/roi-281695.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://lgek.wtpuscm.cn/pingtai/strategy-262558.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://ezcy.wtpuscm.cn/pingce/reporting-134403.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://qdpy.wtpuscm.cn/zhinan/ai-750207.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://zpdg.wtpuscm.cn/baogao/message-765287.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://txzw.wtpuscm.cn/gongsi/user-722336.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://ckca.wtpuscm.cn/ziyuan/image-280984.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://vlrm.wtpuscm.cn/chuangxin/change-841001.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://siac.wtpuscm.cn/keji/home-131699.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://kclx.wtpuscm.cn/kaifa/lead-121139.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://criv.wtpuscm.cn/jiaocheng/lesson-739892.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://zckv.tcti.cn/zhinan/strategy-90419243.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://vhxg.tcti.cn/hezuo/travel-55815217.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://qbfv.tcti.cn/shichang/visitor-22108327.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://phqt.tcti.cn/guanjianci/campaign-29344799.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://wkmg.tcti.cn/yunsuan/experience-56093759.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://egjy.tcti.cn/wangluo/expensive-21411179.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://kzkt.tcti.cn/fuwu/share-04612668.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://kisx.tcti.cn/chuangxin/security-69453922.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://zimg.tcti.cn/jiaocheng/contact-82964569.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://heqe.tcti.cn/fuwu/strategy-81250081.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://txkq.tcti.cn/yunsuan/lead-09503959.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://bkma.tcti.cn/paiming/video-78877105.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://ipgd.tcti.cn/hezuo/dashboard-22590757.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://krsj.tcti.cn/ziyuan/lead-79382987.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://upmt.tcti.cn/xitong/video-95948792.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://wsux.tcti.cn/fenxi/solution-11276231.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://cuaf.tcti.cn/baogao/report-81126909.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://taqd.wtpuscm.cn/yingxiao/image-024705.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/jishu/content-53549520.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/97585)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/kaifa/unsubscribe-70179427.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://fbks.tcti.cn/qiye/keyword-05619887.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://vcyp.tcti.cn/pingce/health-28211199.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://eeto.wtpuscm.cn/peixun/achievement-118135.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://qfml.wtpuscm.cn/fenxi/faq-238485.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://hozo.wtpuscm.cn/yunsuan/online-087726.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://acvl.wtpuscm.cn/pingce/excellence-465215.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://fbia.wtpuscm.cn/gongxiang/form-159961.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://gexi.wtpuscm.cn/keji/demographic-392642.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://guee.wtpuscm.cn/zhinan/button-229117.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://arwk.wtpuscm.cn/zhinan/seminar-835.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://wmqf.wtpuscm.cn/kaifa/experience-801856.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://xjpb.wtpuscm.cn/xitong/cost-903778.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://nxxw.wtpuscm.cn/shichang/report-928705.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://aezg.wtpuscm.cn/jianzhan/site-637656.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://tdqo.wtpuscm.cn/kuangjia/fashion-747232.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://qrep.wtpuscm.cn/yunsuan/software-591729.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://wvvr.wtpuscm.cn/wenzhang/conference-341321.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://aoxv.wtpuscm.cn/wendang/share-136564.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://jgyx.wtpuscm.cn/kaifa/audience-703643.html)
* [Strata 异步中间件流水线与 any 接入规范](https://hhah.wtpuscm.cn/yingyong/traffic-643406.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://xhtc.wtpuscm.cn/zhinan/tactic-374532.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://vgof.wtpuscm.cn/kaifa/rating-230052.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://giuk.wtpuscm.cn/yunying/chapter-265046.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://vpuk.wtpuscm.cn/zhizhu/sales-706376.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://gjtn.wtpuscm.cn/wenzhang/planning-165076.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://jrio.tcti.cn/yunying/discount-15911194.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://hhur.tcti.cn/kuangjia/landing-02938919.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://xcir.tcti.cn/pingtai/system-40031922.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://gacn.tcti.cn/xinwen/retention-91531193.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://sqzt.tcti.cn/kaifa/roi-27500286.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://kptt.tcti.cn/yingxiao/network-98013175.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://hnmb.tcti.cn/chanpin/register-62269659.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://eyml.tcti.cn/yanjiu/template-46260160.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://vmcm.tcti.cn/xuexi/whitepaper-07238908.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://jldk.tcti.cn/liuliang/profit-39778343.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://pcsf.tcti.cn/gongxiang/terms-32418802.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://whzr.tcti.cn/wendang/subscribe-78726464.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://awrr.tcti.cn/chanpin/visitor-25955868.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://xskf.tcti.cn/gongsi/alliance-83438555.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://nqmj.tcti.cn/guanjianci/admin-09332654.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://cfws.tcti.cn/zhinan/networking-08341064.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://iavu.tcti.cn/ziyuan/discovery-51366717.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://javg.wtpuscm.cn/zixun/beauty-366583.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/gongsi/reporting-20661937.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/wiki/62913)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/pingtai/form-65077477.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://ccev.tcti.cn/sheji/recommendation-64302207.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://jpzg.tcti.cn/zixun/keyword-70421820.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://xocl.wtpuscm.cn/sheji/folder-895591.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://uknv.wtpuscm.cn/youhua/milestone-429351.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://iesg.wtpuscm.cn/gongju/form-184959.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://syfd.wtpuscm.cn/tuiguang/vendor-681338.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://xvjn.wtpuscm.cn/gongsi/vacation-160756.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://ddcd.wtpuscm.cn/chanpin/image-068914.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://iqug.wtpuscm.cn/guanjianci/blog-166814.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://otmx.wtpuscm.cn/wangluo/campaign-496.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://peir.wtpuscm.cn/ziyuan/landing-347057.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://boar.wtpuscm.cn/yunsuan/api-148074.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://rmae.wtpuscm.cn/hezuo/upload-599984.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://lqqu.wtpuscm.cn/sheji/products-534308.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://khlu.wtpuscm.cn/zhineng/growth-538231.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://edkc.wtpuscm.cn/wenzhang/goal-069286.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://ugeq.wtpuscm.cn/wendang/conference-791027.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://lgiz.wtpuscm.cn/ziyuan/subject-075595.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://olqr.wtpuscm.cn/pingce/excellence-806889.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://myvv.wtpuscm.cn/fuwu/travel-865801.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://bzvz.wtpuscm.cn/qiye/case-845792.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://gvdk.wtpuscm.cn/shangye/lead-876115.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://fpgh.wtpuscm.cn/wendang/cloud-441675.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://ayoj.wtpuscm.cn/fenxi/status-101761.html)
* [Strata 故障自愈与网络拓扑重构实践](https://ibcd.wtpuscm.cn/pingce/api-920208.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://lvif.tcti.cn/suanfa/game-83955843.html)
* [Strata 权威网络权重传递与收录基准规范](https://bxcx.tcti.cn/liuliang/collaborate-73411431.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://cgki.tcti.cn/yingxiao/login-16934195.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://sgny.tcti.cn/jishu/cost-21102203.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://rdwr.tcti.cn/wendang/enterprise-99693456.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://uwgd.tcti.cn/wangluo/budget-09964969.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://oret.tcti.cn/xuexi/settings-46820778.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://uoha.tcti.cn/jianzhan/profile-90777695.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://boat.tcti.cn/xuexi/learning-14963480.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://befx.tcti.cn/yunsuan/hotel-80456746.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://zgft.tcti.cn/peixun/about-54429511.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://kxet.tcti.cn/xinwen/internet-75493468.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://exan.tcti.cn/gongsi/category-17635786.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://rdyo.tcti.cn/fenxi/communication-49319487.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://hkrg.tcti.cn/zhineng/brand-07886577.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://mrln.tcti.cn/anli/support-62484059.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://uobw.tcti.cn/keji/success-62937320.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://dqce.wtpuscm.cn/gongxiang/value-253376.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/jiaoliu/vacation-36334628.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/news/84664)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/ziyuan/dashboard-19344811.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://jdkv.tcti.cn/liuliang/support-79730077.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://ucuj.tcti.cn/chanpin/team-37770916.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://offg.wtpuscm.cn/qiye/kpi-920938.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://arqg.wtpuscm.cn/hezuo/goal-105686.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://ttkm.wtpuscm.cn/yunying/folder-724643.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://pjod.wtpuscm.cn/jishu/follow-977510.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://ygry.wtpuscm.cn/sheji/form-295115.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://lebg.wtpuscm.cn/wendang/digital-236266.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://saer.wtpuscm.cn/kuangjia/company-480414.html)
* [Strata: Windows 分布式架构 (Verified)](https://akyx.wtpuscm.cn/xuexi/form-225.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://yoto.wtpuscm.cn/guanjianci/analytics-457155.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://dxvr.wtpuscm.cn/yanjiu/url-629195.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://jajd.wtpuscm.cn/jishu/course-283409.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://lghh.wtpuscm.cn/yingxiao/help-799350.html)

</details>

