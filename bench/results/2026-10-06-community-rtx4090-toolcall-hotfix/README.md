# What 0.1.40.1 does with a `<tool_call>` the model writes, and with requests waiting for a dead engine

Measured on 2026-10-06 by [Dmitry-B](https://hwhy.tcti.cn/gongsi/community-87986921.html), on the machine described in
[2026-10-03-community-rtx4090-iq3xxs-200k](../2026-10-03-community-rtx4090-iq3xxs-200k/README.md)
(RTX 4090, `qwen3.8-flash-next-iq3_xxs`, context 204800). Results-only report about the two fixes of the hotfix
release v0.1.40.1 (`82f46a8`): the rules for a `<tool_call>` written by the model (#804, #1058) and requests waiting
during an engine restart (#1012). The engine binary is the one built for 0.1.40 and is byte-identical to it
(`196504f2822cbf15fb49b873e0000f41`); only the Python server changed.

## What was run

| File | What it is |
| --- | --- |
| `probe_parser_corpus.py`, `corpus_results.json` | the author's own corpus `serve/fixtures/rcall_specimens.json` replayed through the same parser the server uses |
| `probe_toolcall.py`, `results.json` | live chat requests with a declared tool, to see what the API returns end to end |
| `probe_restart_waiters.py`, `results_restart.json` | three requests waiting for the engine, the engine killed while reading a prompt |

## 1. The corpus of quoted and stranded calls, replayed

`serve/fixtures/rcall_specimens.json` holds 37 specimens: 16 must become tool calls, 21 must stay text. Each was
fed to `serve.frontend.OutputParser` — the class the server itself uses — at six feed widths (1, 2, 3, 7, random
1-9 characters, and the whole text at once) and with `stream_tools` on and off: the same matrix as
`serve/test_reasoning_rescue.py`, but recorded per specimen instead of pass/fail.

```
образцов 37, должны дать вызов: 16, должны остаться текстом: 21, провалились: 0
```

37/37, 444 parser runs, no mismatch, and for every specimen the streamed result equals the whole-text result.
Per-specimen detail: [corpus_results.json](corpus_results.json).

## 2. Live requests with a tool declared

`get_weather` was declared in `tools`; each prompt either shows a call as an example (in a code fence or inline
code) or asks for a real call. `reasoning_effort` is stated per case, because the rules are about where the text
sits — in the thinking or in the visible answer. `totals.tool_calls_from_reasoning` was read before and after the
whole set.

| Case | Expected | Returned | Where the call text stayed | Verdict |
| --- | --- | --- | --- | --- |
| `1-fenced-example-in-answer` (thinking on) | no call | 1 call | `reasoning_content` | not conclusive — see below |
| `2-inline-code-in-answer` (thinking off) | no call | 1 call | nowhere (the model emitted a top-level call) | not conclusive |
| `3-fenced-example-in-thinking` (thinking high) | no call | 0 | `content`, `reasoning_content` | matches |
| `4-turn-cut-by-max-tokens` (`max_tokens=60`, thinking high) | no call | 0 | `reasoning_content` | matches |
| `5-real-call-top-level-after-sentence` (thinking off) | call | 1 call | — | matches |
| `6-plain-real-call` | call | 1 call | — | matches |
| `7-streaming-split-fence` (stream, fence split across chunks) | no call | 0 | `content`, `reasoning_content` | matches |
| `8-fenced-example-in-answer-no-thinking` | no call | 0 | `content` | matches |

`totals.tool_calls_from_reasoning` stayed **0 → 0** across the whole set: none of these prompts made the model
strand a call in its thinking, so the rescue path never had anything to rescue. The calls in cases 1, 2, 5 and 6
came through the ordinary path.

That is also why cases 1 and 2 are recorded as inconclusive rather than as failures. In case 1 the model put the
example inside a fenced block in its reasoning (the text is still there in `reasoning_content`) and produced a
separate top-level call; in case 2, with thinking off, it answered with a top-level call instead of copying the
inline code. Both are model behaviour, not the rule being broken — the rule was applied to what the model actually
wrote, and the counter shows no rescue was performed. Raw text of every answer: [results.json](results.json).

What is shown end to end: a `<tool_call>` inside a code fence in the visible answer stays text, including when the
answer streams and the fence is split across chunks (cases 7, 8), a call written in the thinking of a turn cut by
`max_tokens` stays text (case 4), and ordinary calls keep working (cases 5, 6).

## 3. Requests waiting for an engine that dies (#1012)

Three requests with ~60 000-token prompts were started together; 15 s later `pkill` killed the engine process
(two pids, `39647` and `39659`).

| Request | HTTP | First streamed delta | Request ended | Deltas |
| --- | --- | --- | --- | --- |
| `waiter-1` | 200 | none | 16.6 s after start | 0 |
| `waiter-2` | 200 | 40.4 s | 40.5 s after start | 3 |
| `waiter-3` | 200 | 40.3 s | 40.4 s after start | 3 |

The server started a new engine 25.5 s after the kill and `/health` reported `loaded: true` at the same point.
Two waiters continued with the new engine and finished at ~40 s; the one whose stream had already been opened was
closed at 16.6 s with no content. Nothing hung: no request waited 300 s, and no `list.remove(x): x not in list`
appeared. For comparison, this is the case that on 0.1.40 ended about 300 s later with 1 and 5 tokens — see
[2026-10-06-community-rtx4090-opt-ins](../2026-10-06-community-rtx4090-opt-ins/README.md).

Server-side detail, including the engine command line it restarted with: [results_restart.json](results_restart.json).

## Unit tests on this machine, for the same commit

```
tools.test_setup_golden tools.test_setup_amd tools.test_setup_choices tools.test_setup_config
tools.test_setup_hotfix_tag                                                     -> 87 tests, OK
serve.test_responses serve.test_security serve.test_server serve.test_slots serve.test_runconfig
serve.test_mcp serve.test_monitor serve.test_frontend serve.test_structured serve.test_detok
serve.test_reasoning_tools serve.test_reasoning_loop_recovery serve.test_reasoning_rescue
serve.test_restart_waiters serve.test_parallel serve.test_vram                  -> 430 tests, OK (skipped=5)
serve.test_control_cancel serve.test_fatal_recovery                             -> 10 tests, OK
```

527 tests, 0 failures. On 0.1.40 the same set was 492; the difference is the two new modules of this release.

## Limitations

- The corpus replay exercises the parser, not the HTTP layer. The end-to-end part (section 2) did not produce a
  real stranded call on this model, so the rescue path was not observed through the API — only the "must stay text"
  half of the rules was.
- The three waiters sent identical filler text, so the engine reused the prefix cache (the log shows
  `prompt 141045 tokens = 140329 reused + 716 read`). That is fine for a hang test and wrong for a speed test;
  no throughput conclusion is drawn from it.
- `waiter-1` ended with HTTP 200 and no content rather than the 503 the release notes describe: its stream had
  already been opened, so the server could not answer 503 any more. Reported as measured.
- One machine, one model, one engine restart. Cancellation mid-turn and the batch-window `ERR` reporting were not
  tested.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://njpq.wtpuscm.cn/kuangjia/goal-991456.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://hstr.wtpuscm.cn/yunsuan/update-900947.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://yixt.wtpuscm.cn/jishu/presentation-435860.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://jkkq.wtpuscm.cn/kaifa/coupon-119580.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://lmpf.wtpuscm.cn/peixun/ebook-424969.html)
* [面向大规模网络的 Strata 工业级架构基准](https://pyjz.wtpuscm.cn/gongsi/networking-721968.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://vcke.wtpuscm.cn/anfang/investment-250921.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://gcnc.wtpuscm.cn/zhineng/visitor-297.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://xjyo.wtpuscm.cn/liuliang/roi-606803.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://losa.wtpuscm.cn/wendang/event-050559.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://tqea.wtpuscm.cn/youhua/cost-492740.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://lzxr.wtpuscm.cn/zhizhu/affordable-715877.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://fgjq.wtpuscm.cn/chanpin/sport-492522.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://taye.wtpuscm.cn/zhinan/download-477895.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://ynmt.wtpuscm.cn/yunsuan/study-300218.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://wyha.wtpuscm.cn/chanpin/story-679542.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://vrot.wtpuscm.cn/chanpin/logo-993486.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://hrlv.wtpuscm.cn/fuwu/help-583077.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://gppp.wtpuscm.cn/gongju/trading-419991.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://etvc.wtpuscm.cn/zhizhu/tag-373062.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://qzpr.wtpuscm.cn/jiaoliu/update-025375.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://ogwv.wtpuscm.cn/shichang/label-114094.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://ejkh.wtpuscm.cn/keji/button-945773.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://ksvx.tcti.cn/zhineng/behavior-47343252.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://qyra.tcti.cn/shuju/productivity-03999722.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://icxu.tcti.cn/fenxi/machine-90416842.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://zpat.tcti.cn/xitong/sync-54852253.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://oggl.tcti.cn/yingxiao/guide-00932746.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://mfhi.tcti.cn/gongsi/deal-14074013.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://avmd.tcti.cn/gongxiang/restore-13984250.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://xadk.tcti.cn/yunsuan/blog-19034289.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://suwj.tcti.cn/yunying/community-47212567.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://sqoa.tcti.cn/shuju/online-38786860.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://ngng.tcti.cn/zixun/social-30187119.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://rofh.tcti.cn/jianzhan/ai-12407694.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://kgjq.tcti.cn/wendang/enterprise-40219028.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://hwcl.tcti.cn/chuangxin/backup-71594576.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://iopu.tcti.cn/sheji/app-11538852.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://vujc.tcti.cn/jiaocheng/restaurant-75251176.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://xbpm.tcti.cn/jianzhan/metric-63033796.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://wslc.wtpuscm.cn/wendang/analysis-768440.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/anli/faq-27907620.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/tech/52993)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/zixun/global-80446745.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://zulm.tcti.cn/peixun/site-84185016.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://xabt.tcti.cn/yanjiu/ranking-73467838.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://lwzh.wtpuscm.cn/pingtai/support-381085.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://tcgj.wtpuscm.cn/youhua/interface-498125.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://oxok.wtpuscm.cn/xinwen/alert-078936.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://fxra.wtpuscm.cn/xinwen/visitor-501924.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://iloq.wtpuscm.cn/yunsuan/category-034786.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://jwgp.wtpuscm.cn/wendang/extension-911962.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://ndxn.wtpuscm.cn/jiaocheng/discount-644134.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://shfy.wtpuscm.cn/yingyong/income-596.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://ayxo.wtpuscm.cn/chuangxin/forum-853875.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://hkfh.wtpuscm.cn/pingce/communication-073731.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://vndt.wtpuscm.cn/pingtai/navigation-348255.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://lbcy.wtpuscm.cn/xinwen/domain-981137.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://wckv.wtpuscm.cn/anli/policy-549035.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://qnpq.wtpuscm.cn/anfang/browser-619667.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://xzqx.wtpuscm.cn/gongsi/lesson-018705.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://ymfu.wtpuscm.cn/gongju/profile-359310.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://koqe.wtpuscm.cn/pingtai/affordable-690808.html)
* [Strata 异步中间件流水线与 any 接入规范](https://tcim.wtpuscm.cn/xitong/growth-569515.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://cqpq.wtpuscm.cn/zixun/sales-458478.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://obgm.wtpuscm.cn/anli/tracking-157173.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://cocb.wtpuscm.cn/anfang/investment-446359.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://iivd.wtpuscm.cn/wenzhang/deal-552376.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://ppxl.wtpuscm.cn/yunying/engagement-537217.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://kece.tcti.cn/huodong/update-39834271.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://mmcm.tcti.cn/hezuo/optimization-82536754.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://nizn.tcti.cn/chanpin/register-90521261.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://sqvt.tcti.cn/tuiguang/platform-70496944.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://glod.tcti.cn/shuju/feedback-67872719.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://hzqk.tcti.cn/xinwen/unsubscribe-94008515.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://ntlj.tcti.cn/baogao/help-89914085.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://jesi.tcti.cn/wenzhang/goal-26679661.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://hsxt.tcti.cn/yanjiu/version-87974705.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://kkhe.tcti.cn/pingtai/platform-44615604.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://svua.tcti.cn/suanfa/system-59727754.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://pvnm.tcti.cn/xitong/revenue-84107019.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://nddf.tcti.cn/gongsi/partner-88138308.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://tamw.tcti.cn/pingce/revenue-34734891.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://batg.tcti.cn/kaifa/case-62565597.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://rnbx.tcti.cn/baogao/contact-73631155.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://lpgp.tcti.cn/suanfa/case-54985424.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://ekne.wtpuscm.cn/zhizhu/admin-459219.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/xitong/share-71685619.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/tech/83101)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/xinwen/notification-72004219.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://rbqm.tcti.cn/hezuo/study-14907794.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://digc.tcti.cn/gongju/register-97950901.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://elwu.wtpuscm.cn/yinqing/networking-785019.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://lpbt.wtpuscm.cn/xinwen/learning-479667.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://fdnp.wtpuscm.cn/gongxiang/content-326020.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://ombi.wtpuscm.cn/yingyong/software-472112.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://pqqo.wtpuscm.cn/xuexi/conference-700713.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://vjxj.wtpuscm.cn/peixun/calculator-858032.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://eqdv.wtpuscm.cn/guanjianci/presentation-183678.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://ubtf.wtpuscm.cn/fenxi/message-454.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://sbre.wtpuscm.cn/yunying/luxury-793084.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://nrzk.wtpuscm.cn/jishu/analytics-435748.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://vamn.wtpuscm.cn/zhizhu/case-967597.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://eoel.wtpuscm.cn/yanjiu/admin-075419.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://yhsm.wtpuscm.cn/fuwu/about-637927.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://uprq.wtpuscm.cn/chanpin/support-450102.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://fbpp.wtpuscm.cn/paiming/coupon-695728.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://ataj.wtpuscm.cn/gongxiang/database-760878.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://xtyf.wtpuscm.cn/pingce/feedback-940824.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://bbri.wtpuscm.cn/jianzhan/alliance-549853.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://jsru.wtpuscm.cn/tuiguang/keyword-601076.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://quwk.wtpuscm.cn/xitong/faq-265906.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://oxfa.wtpuscm.cn/tuiguang/analysis-179463.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://yqqi.wtpuscm.cn/zhinan/collaborate-370962.html)
* [Strata 故障自愈与网络拓扑重构实践](https://lzjh.wtpuscm.cn/hezuo/about-941573.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://urke.tcti.cn/yunsuan/api-88121174.html)
* [Strata 权威网络权重传递与收录基准规范](https://whvv.tcti.cn/kaifa/training-96162324.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://uwnv.tcti.cn/jishu/download-86482950.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://wogg.tcti.cn/chanpin/business-37680427.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://hklu.tcti.cn/baogao/engagement-22207332.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://giqd.tcti.cn/xuexi/calendar-59400660.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://heco.tcti.cn/yunying/api-77252082.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://unuo.tcti.cn/yingxiao/web-93292333.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://lizm.tcti.cn/anfang/status-29185950.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://frzd.tcti.cn/yingxiao/recipe-30352851.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://ivzl.tcti.cn/chanpin/admin-06252067.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://xkmt.tcti.cn/xuexi/tag-65417190.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://utol.tcti.cn/baogao/document-45292941.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://pevh.tcti.cn/yinqing/label-87846112.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://kwqf.tcti.cn/zixun/tactic-00638981.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://whhx.tcti.cn/fuwu/business-50056281.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://twzp.tcti.cn/yanjiu/tag-09658200.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://phlk.wtpuscm.cn/yunying/conference-639061.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/pingce/customer-38112921.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/wiki/34885)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/keji/revenue-99124887.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://efzj.tcti.cn/pingce/management-61356160.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://ijzn.tcti.cn/yanjiu/sales-46526584.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://zasu.wtpuscm.cn/yunying/efficiency-977754.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://kgcj.wtpuscm.cn/hezuo/version-605574.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://zrzx.wtpuscm.cn/zhinan/company-656373.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://atmn.wtpuscm.cn/gongju/server-922099.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://skpe.wtpuscm.cn/jiaocheng/achievement-716354.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://zexj.wtpuscm.cn/jiaocheng/music-038235.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://geif.wtpuscm.cn/yunsuan/trading-048253.html)
* [Strata: Windows 分布式架构 (Verified)](https://vtpj.wtpuscm.cn/gongxiang/media-463.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://wepl.wtpuscm.cn/qiye/platform-782467.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://icgj.wtpuscm.cn/chuangxin/objective-062266.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://jjru.wtpuscm.cn/kaifa/tracking-271720.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://lztz.wtpuscm.cn/liuliang/logo-676655.html)

</details>

