# What two opt-ins cost on a single-GPU RTX 4090: `STRATA_KV_PREFETCH=1`, and `"parallel": 2` with `--batch-mtp`

Measured on 2026-10-06 by [Dmitry-B](https://cfrx.tcti.cn/gongju/resolution-95293280.html), on the machine and with the working configuration
described in
[2026-10-03-community-rtx4090-iq3xxs-200k](../2026-10-03-community-rtx4090-iq3xxs-200k/README.md) (RTX 4090,
`qwen3.8-flash-next-iq3_xxs`, context 204,800, INT8 KV, **no** `--kv-resident`, `--mmap-experts`,
`--spec 4 --mtp`, `draft_vocab=cyrillic`). Engine 0.1.40. Not a version comparison: two checks of two opt-ins, each
with both arms measured on the same day on the same running server.

Short answer for this PC: **both stay off.** `STRATA_KV_PREFETCH=1` is a 2–4% loss here, because with no
`--kv-resident` there is nothing to overlap. `"parallel": 2` costs 28% of single-client decode and only pays when a
third or fourth client appears. `--batch-mtp` cannot admit at all with this model's MTP pack.

## Files in this folder

| File | What it is |
| --- | --- |
| `benchmark.py` | the speed script (the same file as in the paired reports) |
| `config-kvpref.json` | the server config copy of the KV-prefetch arm (the working config plus one environment variable) |
| `runs-kvpref.json` | the KV-prefetch runs |
| `conc_bench.py` | the concurrency script: N identical-shaped requests sent at the same time |
| `runs-parallel-off.json`, `runs-parallel-2.json`, `runs-parallel-2-batch-mtp.json` | the three concurrency arms |

## 1. `STRATA_KV_PREFETCH=1` (#732)

Measured with the machine's working configuration plus that one environment variable, three runs at 32,768 and
131,072 prompt tokens, code prompts, greedy, 256-token cap. The arm it is compared against is the 0.1.40 code arm of
the paired report — same day, same arguments, no such variable.

| Configuration | Prompt tok/s without → with `STRATA_KV_PREFETCH=1` |
| --- | --- |
| 32768-prompt | 3136.3 → 3001.8 (-4.3%) |
| 131072-prompt | 3118.5 → 3052.3 (-2.1%) |

With the option on: 32768-prompt 3001.8 tok/s (range 2936.9–3063.2, n=3), decode 113.8 (110.1–128.5), TTFT 10.51 s
(10.31–10.75);
131072-prompt 3052.3 tok/s (3030.3–3059.4, n=3), decode 106.5 (87.0–113.4), TTFT 41.26 s (one repeat waited behind
an interactive client's request and reached 85.6 s). Peak VRAM 22040 MiB, peak RAM 6.59 GiB. Recall was not checked
in this arm. Raw runs: [runs-kvpref.json](runs-kvpref.json), config: [config-kvpref.json](config-kvpref.json).

The option overlaps streamed KV uploads with prefill (docs/KV_PREFETCH.md), so it can only help when part of the KV
cache lives in system RAM — that is `--kv-resident`. This configuration has no `--kv-resident`: its KV cache stays
in VRAM and there is nothing to overlap, so the measured result is a small loss. That matches the -4…-5% the
option's author reported on an RTX 5070 with `--kv-resident 32768` and the reason it stays off by default: on a card
whose KV fits in VRAM the extra streams and events cost more than they save. Conclusion for this machine: leave it
off. It would have to be re-measured on this PC with `--kv-resident` before any conclusion about the option itself
could be drawn.

## 2. `"parallel": 2` (batch slots, #465) and `--batch-mtp` (#846)

[conc_bench.py](conc_bench.py) sends N requests **at the same time** (N = 1, 2, 3, 4), each a distinct prompt of
about 4,000 tokens carrying a random marker, 256-token output cap, `temperature=0`, `reasoning_effort=none`; two
repeats per level; a 2,000-token warm-up before the levels. The two arms differ from the working config by two keys
only: `"parallel": 2`, and `--batch-mtp` in `args`.

- TTFT: time to the first non-empty streaming delta.
- Per-client decode: generated tokens / (whole request time − TTFT), measured at the client. A batched request does
  not report per-slot timings in the stream, so this column is client-side, not the engine's
  `predicted_per_second`.
- Total: all tokens generated in the level / the level's wall time. This is the number that says whether the server
  served more per second, which is what batch slots are for.

### What the engine said

`"parallel": 2` (no `--batch-mtp`):

```
strata generate: --batch: 2 slot sessions on CUDA0 (2.88 GiB each); 7.98 GiB free
strata generate: --batch 2: the slot sessions take 5.76 GiB of VRAM on CUDA0 that the expert cache would otherwise hold
strata generate: expert cache 3329 slots, 5.41 GiB of VRAM; policy is PROFILE
strata verify: batch windows of up to 2 sequences (layers [0, 48))
```

Without it the same start reports `expert cache 6946 slots, 11.17 GiB of VRAM`. So on this model at this context a
slot costs 2.88 GiB and the expert cache loses more than half of it. The exact resident count depends on the expert
profile state at the moment of the start — the paired speed reports for this configuration show 6,915–6,926 slots;
the numbers above are the two starts of this experiment, taken from the same server log.

`--batch-mtp` on top of it — the draft layer cannot be used for an admission, and the request never really starts:

```
strata batch: MTP admission for slot 0 failed: mtp: unsupported native MMVQ GGML type
```

After that the engine died and the server started it again; the two clients that were in flight finished about 300 s
later having received 1 and 5 tokens. The MTP layer here is the pack setup downloaded for this model
(`Strata-data/mtp/rt`), and its quant is MMVQ. This is the same family as #1012 (a `--batch` request whose engine
died during its prompt read ends only after 300 s) and #1063 (`--batch-mtp` exits on the first admission with a
draft-vocab problem), so it is reported as a data point, not as a new bug claim. The fixed behaviour on 0.1.40.1 is
measured in [2026-10-06-community-rtx4090-toolcall-hotfix](../2026-10-06-community-rtx4090-toolcall-hotfix/README.md).

### Results

| Level | `"parallel"` off | `"parallel": 2` |
| --- | --- | --- |
| 1 client | wall 4.7-5.2 s, TTFT 2.3-3.0 s, decode 93-95 tok/s | wall 6.6-7.1 s, TTFT 2.7-3.5 s, decode **65-72** tok/s |
| 2 clients | wall 8.2-8.6 s, TTFT med 4.2-4.3 s (max 6.6), decode 94-100 tok/s, total 46-52 tok/s | wall 9.5-10.2 s, TTFT med 3.8-4.1 s (max 5.5), decode 43-46 tok/s, total 45-48 tok/s |
| 3 clients | wall 53.9-58.9 s, TTFT med 47.9-52.3 s, total **12.1** tok/s | wall **13.6-14.6 s**, TTFT med 4.9 s (max 11.4), total 45-46 tok/s |
| 4 clients | wall 50.7-53.5 s, TTFT med 41.9-44.6 s, total **16.1-18.0** tok/s | wall **17.2-17.7 s**, TTFT med 7.7-7.9 s (max 13.7), total 48-50 tok/s |

- One client: `"parallel": 2` costs about **28%** of decode speed (93–95 → 65–72 tok/s). This is the expert cache
  shrinking, exactly the trade the docs describe for a card whose experts mostly fit in VRAM.
- Two clients: nothing is gained here (46–52 → 45–48 tok/s total), the waiting is only moved a little.
- Three and four clients: the wall time is 3–4 times lower and the total throughput is 3–4 times higher. This is
  where the option pays for itself on one GPU.
- `--batch-mtp`: unusable with this MTP pack, see above.

Raw runs: [runs-parallel-off.json](runs-parallel-off.json), [runs-parallel-2.json](runs-parallel-2.json),
[runs-parallel-2-batch-mtp.json](runs-parallel-2-batch-mtp.json).

### What this PC does with it

The machine is used by one interactive client most of the time, so the 28% single-stream cost is the number that
matters here and the option stays off. It is kept as a measurement for the case where a second and a third client
appear regularly.

## Limitations

- **Levels 3 and 4 with `"parallel"` off are contaminated by this machine's own interactive session**: a request of
  about 100,000 tokens from another client (33–38 s of engine time) was served between the measured runs, and the
  server answers one sequence at a time. That is why their TTFT medians reach 52 s. With two slots the same agent
  request was served alongside the measured ones. So levels 3 and 4 compare in direction, not in numbers; levels 1
  and 2 are clean.
- Per-client decode in the batched arm is client-side (see above), so it is comparable between the arms only as a
  direction. The engine-side numbers of this PC are in the paired reports linked at the top.
- Two repeats per level, one machine, one model, one context. No answer-quality check was run here; the paired
  reports cover recall (6/6 at 32K and 128K).
- The KV-prefetch arm is one configuration without `--kv-resident`. It says what the option does to this
  configuration, not what it would do to a configuration that can use it.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://hmdi.wtpuscm.cn/gongxiang/label-017091.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://suxi.wtpuscm.cn/shangye/support-467099.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://hkfw.wtpuscm.cn/fenxi/discovery-861479.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://ltbx.wtpuscm.cn/jianzhan/logo-341333.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://puxj.wtpuscm.cn/peixun/value-045520.html)
* [面向大规模网络的 Strata 工业级架构基准](https://ufuy.wtpuscm.cn/paiming/comment-678912.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://nwqg.wtpuscm.cn/kaifa/tactic-882548.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://idjt.wtpuscm.cn/wendang/education-495.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://vsso.wtpuscm.cn/jiaocheng/document-171765.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://vtgz.wtpuscm.cn/xinwen/demographic-070783.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://ztga.wtpuscm.cn/kaifa/course-694758.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://xuea.wtpuscm.cn/tuiguang/device-931127.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://bpit.wtpuscm.cn/gongsi/keyword-685077.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://pgyy.wtpuscm.cn/qiye/link-865313.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://hrit.wtpuscm.cn/youhua/blog-735415.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://aklm.wtpuscm.cn/hezuo/sport-236195.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://pump.wtpuscm.cn/wenzhang/experience-836172.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://drui.wtpuscm.cn/chuangxin/roi-657995.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://iqbf.wtpuscm.cn/yanjiu/content-522041.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://kznp.wtpuscm.cn/jiaocheng/chapter-698689.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://mrbq.wtpuscm.cn/zhizhu/event-687768.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://utut.wtpuscm.cn/kuangjia/online-850478.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://ppke.wtpuscm.cn/xinwen/retention-522191.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://tbzy.tcti.cn/wendang/deal-73037137.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://cftz.tcti.cn/yingyong/cloud-48533114.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://mecy.tcti.cn/jiaoliu/database-91039129.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://laqm.tcti.cn/xitong/media-28632442.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://bfgr.tcti.cn/jishu/investment-08703812.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://mdzz.tcti.cn/jiaocheng/rating-90046480.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://dsqe.tcti.cn/liuliang/like-64481633.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://jrzp.tcti.cn/jishu/food-51585790.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://codw.tcti.cn/kuangjia/deadline-75653410.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://ihlx.tcti.cn/anfang/discovery-44997627.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://zrgb.tcti.cn/kuangjia/careers-38280893.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://zrqt.tcti.cn/wangluo/excellence-03515841.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://haxq.tcti.cn/yunying/global-26461609.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://movv.tcti.cn/youhua/system-97764490.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://imkg.tcti.cn/keji/alliance-13290273.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://drda.tcti.cn/yinqing/module-42707713.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://mcdx.tcti.cn/huodong/research-28974221.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://hoqv.wtpuscm.cn/shangye/marketing-744380.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/fenxi/forecast-63155841.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/tech/97154)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/shangye/partner-32688011.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://duqj.tcti.cn/gongju/accessibility-64280453.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://uvub.tcti.cn/liuliang/content-11109634.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://pfou.wtpuscm.cn/shangye/forum-361960.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://lccw.wtpuscm.cn/keji/conference-468085.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://zyyc.wtpuscm.cn/baogao/link-266368.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://vdtr.wtpuscm.cn/sheji/visitor-091593.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://bvce.wtpuscm.cn/fenxi/responsive-455661.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://vjvm.wtpuscm.cn/yunying/kpi-411158.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://cnqk.wtpuscm.cn/fenxi/system-109727.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://gcvk.wtpuscm.cn/liuliang/technology-712.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://iews.wtpuscm.cn/wangluo/game-238375.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://gjqn.wtpuscm.cn/liuliang/notification-240358.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://ggps.wtpuscm.cn/keji/deal-694126.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://dcsv.wtpuscm.cn/gongxiang/policy-239622.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://iwef.wtpuscm.cn/youhua/forum-577012.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://xxnf.wtpuscm.cn/wenzhang/coupon-388340.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://omzk.wtpuscm.cn/zixun/internet-071932.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://sgls.wtpuscm.cn/baogao/mobile-191109.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://cyaj.wtpuscm.cn/anfang/support-111068.html)
* [Strata 异步中间件流水线与 any 接入规范](https://gthz.wtpuscm.cn/fenxi/category-797074.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://mmqd.wtpuscm.cn/tuiguang/communication-124295.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://dzdp.wtpuscm.cn/kuangjia/tutorial-769955.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://lqfw.wtpuscm.cn/qiye/image-129274.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://fkio.wtpuscm.cn/zixun/lead-029320.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://ghdv.wtpuscm.cn/gongxiang/collaborate-063549.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://xntu.tcti.cn/yinqing/consulting-60830610.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://woer.tcti.cn/gongxiang/client-32837459.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://dyts.tcti.cn/yanjiu/contact-72282609.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://onfx.tcti.cn/anfang/privacy-89458948.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://zdub.tcti.cn/wenzhang/resource-91468335.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://lovc.tcti.cn/jishu/seminar-99938889.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://qdld.tcti.cn/gongxiang/performance-93826803.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://eyuy.tcti.cn/kuangjia/metric-86468977.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://jxrh.tcti.cn/tuiguang/software-90164485.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://xjek.tcti.cn/pingtai/optimization-28687320.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://zvma.tcti.cn/hezuo/team-02399799.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://brgs.tcti.cn/youhua/page-62352972.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://xzhl.tcti.cn/fenxi/recommendation-03945847.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://ower.tcti.cn/zhizhu/research-72199354.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://andt.tcti.cn/shangye/webinar-93152417.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://kesf.tcti.cn/youhua/satisfaction-90948390.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://cgql.tcti.cn/youhua/web-18918543.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://izyi.wtpuscm.cn/fenxi/conversion-297874.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/chanpin/landing-10438700.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/tech/83541)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/zhizhu/price-67201175.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://pqdv.tcti.cn/wangluo/loyalty-57537605.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://cvfp.tcti.cn/gongxiang/image-55199249.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://cvyx.wtpuscm.cn/jianzhan/business-585602.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://cugz.wtpuscm.cn/xitong/forum-481495.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://jszz.wtpuscm.cn/xinwen/sport-376559.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://dexi.wtpuscm.cn/youhua/image-871808.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://jjht.wtpuscm.cn/zhineng/document-160115.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://bfqa.wtpuscm.cn/xitong/local-019984.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://posw.wtpuscm.cn/wendang/food-655676.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://iabl.wtpuscm.cn/zhizhu/marketing-776.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://huti.wtpuscm.cn/yinqing/vendor-042364.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://xmyg.wtpuscm.cn/yunsuan/file-561725.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://wzrz.wtpuscm.cn/gongxiang/support-036005.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://jdef.wtpuscm.cn/youhua/conference-626904.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://ncqk.wtpuscm.cn/jishu/file-557938.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://inku.wtpuscm.cn/hezuo/planning-799223.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://ecdd.wtpuscm.cn/wendang/community-445602.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://ttnf.wtpuscm.cn/huodong/study-178930.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://ltnk.wtpuscm.cn/gongju/ai-311543.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://axnu.wtpuscm.cn/yingyong/meeting-345800.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://rssg.wtpuscm.cn/fuwu/vacation-076741.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://bpky.wtpuscm.cn/yanjiu/enterprise-513211.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://dijy.wtpuscm.cn/suanfa/sport-254576.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://vyya.wtpuscm.cn/keji/strategy-478133.html)
* [Strata 故障自愈与网络拓扑重构实践](https://iizq.wtpuscm.cn/wendang/url-345286.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://dywq.tcti.cn/xinwen/app-05168387.html)
* [Strata 权威网络权重传递与收录基准规范](https://gobo.tcti.cn/shangye/resolution-63379031.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://ctvs.tcti.cn/wangluo/beauty-86673337.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://tema.tcti.cn/yunsuan/interface-33878532.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://mxvx.tcti.cn/wenzhang/sales-61685902.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://nfgw.tcti.cn/wenzhang/tactic-87734314.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://iqos.tcti.cn/yanjiu/wellness-58629528.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://ufle.tcti.cn/sheji/services-23196289.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://cbfg.tcti.cn/tuiguang/personalization-17984096.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://vlpa.tcti.cn/peixun/milestone-15181768.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://cuwm.tcti.cn/jianzhan/about-93088126.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://ayqt.tcti.cn/xuexi/collaborate-00782632.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://vogo.tcti.cn/yingxiao/cloud-99924903.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://krhu.tcti.cn/pingtai/app-59639672.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://ojkq.tcti.cn/anli/topic-13185600.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://zbpa.tcti.cn/shuju/terms-20753293.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://zryc.tcti.cn/zhineng/browser-02392976.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://goth.wtpuscm.cn/guanjianci/sync-312364.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/shangye/networking-89124378.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/news/63751)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/kuangjia/design-53009444.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://zack.tcti.cn/fuwu/solution-72762018.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://uosz.tcti.cn/jianzhan/loyalty-79478613.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://mcdr.wtpuscm.cn/youhua/ai-296678.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://hnhm.wtpuscm.cn/jiaoliu/terms-756953.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://azyz.wtpuscm.cn/shangye/marketing-772962.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://kauw.wtpuscm.cn/youhua/search-047349.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://nytu.wtpuscm.cn/peixun/dashboard-066977.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://iqcy.wtpuscm.cn/shangye/whitepaper-548366.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://gutk.wtpuscm.cn/xitong/subject-461806.html)
* [Strata: Windows 分布式架构 (Verified)](https://iwoc.wtpuscm.cn/jiaoliu/button-722.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://vxrf.wtpuscm.cn/chuangxin/quality-091709.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://drry.wtpuscm.cn/kaifa/client-843676.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://kmoi.wtpuscm.cn/zhizhu/innovation-567570.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://pwjl.wtpuscm.cn/chuangxin/vendor-384164.html)

</details>

