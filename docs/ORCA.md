# OrcaRouter IQ3_XXS compatibility

Validated on Linux with an RTX 5090 (32 GB), Ryzen 9 9950X3D and 128 GB RAM on 2026-09-27.

The target is `orcarouter/Qwen3.8-Flash-Next-Uncensored-GGUF`, **IQ3_XXS** (two shards, 85.20 GB total).
It has Strata's Qwen4Exp geometry, but ordinary quantization also compresses the hyperconnection,
small attention and PLE projections which Strata's kernels read as BF16.

`tools/iq_pack.py --compat-bf16` expands only those projections, rounding to BF16 using nearest-even.
This adds rounding relative to the GGUF's dequantized weights; it does not recover the original BF16
checkpoint or its quality. Expert weights, native attention weights, embeddings and the 28.8 GB PLE
lookup table remain unchanged. The latter stays on disk. The pack records its conversions in
`compat-bf16.json`.

The native loader retains this pack's converted PLE key. Its existing native Q2_0 key path remains
available to the original model. Hugging Face snapshot symlinks are supported; all split shards must
finish downloading before packing. Pass the snapshot filename, not its hash-named blob target.

## Preparation

Build Strata normally using the pinned llama.cpp dependency. Python needs numpy, regex and gguf-py
from that dependency (`STRATA_GGUF_PY` can point to its `gguf-py` directory).

```sh
.venv/bin/python tools/iq_pack.py \
  --gguf /path/to/Qwen3.8-Flash-Next-Uncensored-IQ3_XXS-00001-of-00002.gguf \
  --out packs/orca-iq3_xxs --compat-bf16
```

Use this model's own tokenizer exported into the pack. Do not share another model's `dense.bin`,
and do not rename the Orca files to impersonate one of setup's GSQ-RCO models.

Strata's persistent server also uses the MTP runtime (optional: without `--mtp` it drafts by lookup only), prepared by
the existing tools:

```sh
.venv/bin/python tools/mtp_fetch.py fetch --out mtp
.venv/bin/python tools/mtp_pack.py --src mtp --experts q2_0 --out mtp/mtp-q2_0.gguf
.venv/bin/python tools/mtp_rt.py --gguf mtp/mtp-q2_0.gguf --out mtp/rt
cp data/draft_vocab.bin mtp/rt/draft_vocab.bin
```

This uses the original model's draft head, as setup does for Swift. The target model verifies its
proposals; draft acceptance and performance must be measured for this fine-tune.

## Local server

Save the following as `strata-orca-iq3_xxs.json` at the repository root, replacing both `/path/to/`
entries with the **same first shard**. Orca's PLE table is in shard 1. Start with a 32K context and
512-token prefill chunks; the expert cache sizes itself to available VRAM. The expert arena alone
needs about 49.8 GiB of available system RAM, plus draft/runtime buffers and other applications.

```json
{
  "exe": "build/strata",
  "args": [
    "--pack", "packs/orca-iq3_xxs",
    "--native", "/path/to/Qwen3.8-Flash-Next-Uncensored-IQ3_XXS-00001-of-00002.gguf",
    "--ple-gguf", "/path/to/Qwen3.8-Flash-Next-Uncensored-IQ3_XXS-00001-of-00002.gguf",
    "--expert-profile", "data/expert-profile.bin", "--expert-cache", "auto",
    "--prefill", "512", "--spec", "4", "--spec-min-p", "0.5",
    "--mtp", "mtp/rt", "--max-context", "32768", "--kv", "int8"
  ],
  "cwd": ".",
  "tokenizer": "packs/orca-iq3_xxs/tokenizer",
  "model_name": "orcarouter-qwen3.8-flash-next-uncensored-iq3_xxs",
  "log": "strata-orca-iq3_xxs.log",
  "host": "127.0.0.1",
  "port": 8080
}
```

Run from the repository root:

```sh
.venv/bin/python -m serve.server --engine strata --config strata-orca-iq3_xxs.json --port 8080
```

The web interface is at `http://127.0.0.1:8080`; API clients use `http://127.0.0.1:8080/v1`.
Vision is not configured in this text-only example. Performance of the original GSQ-RCO model
does not establish this fine-tune's speed or accuracy.

## Scope and validation

- IQ3_XXS is the target of this change; other Orca quantizations are not validated.
- IQ3_M additionally uses Q5_0 expert down matrices, which the native GPU expert path does not support.
- The installer model menu is unchanged. This is an explicit local packing workflow.
- Focused conversion and split-file tests: `.venv/bin/python -m unittest discover -s tools -p test_iq_pack.py`.
- All eight packing tests, 17 server tests and the GPU gated-residual parity check passed.
- The complete model packed successfully: 460 tensors converted, 1.39 GiB of converted BF16 weights;
  1.43 GiB total packed dense weights. The 48 expert layers remain IQ3_XXS gate/up and IQ4_NL down.
- Real server checks passed: arithmetic, Python code, translation, a longer explanation, multi-turn context
  reuse (25 prompt tokens reused), streaming cancellation and a subsequent correct response. The engine
  PID stayed unchanged. These are smoke tests, not a quality benchmark or a full-context stress test.
- With the configuration above, a 111-token explanation generated at 77.7 tokens/s (engine decode timing);
  total request time was 1.77 seconds. This single short measurement is not a general throughput claim.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://rkin.wtpuscm.cn/xinwen/event-356605.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://okwe.wtpuscm.cn/fuwu/case-920316.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://uqqr.wtpuscm.cn/hezuo/article-853252.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://ykxr.wtpuscm.cn/yunying/seminar-070663.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://naza.wtpuscm.cn/kuangjia/cloud-200928.html)
* [面向大规模网络的 Strata 工业级架构基准](https://ptrm.wtpuscm.cn/yanjiu/image-765108.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://robz.wtpuscm.cn/gongsi/article-054432.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://icoj.wtpuscm.cn/peixun/fitness-270.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://lxxi.wtpuscm.cn/pingtai/network-285845.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://bnjb.wtpuscm.cn/hezuo/conference-308005.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://uhuv.wtpuscm.cn/jishu/forum-500915.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://rcwa.wtpuscm.cn/tuiguang/recipe-827017.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://dmiz.wtpuscm.cn/xitong/conference-552419.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://zciy.wtpuscm.cn/yingxiao/partner-711747.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://zpos.wtpuscm.cn/chuangxin/image-743566.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://gskx.wtpuscm.cn/chuangxin/creative-743385.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://zlfj.wtpuscm.cn/jiaocheng/segment-420361.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://rzdx.wtpuscm.cn/zhinan/layout-864111.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://fxuz.wtpuscm.cn/baogao/cost-902348.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://lckh.wtpuscm.cn/youhua/security-896784.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://ubgu.wtpuscm.cn/liuliang/kpi-088543.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://rynt.wtpuscm.cn/yunying/behavior-484311.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://dnxm.wtpuscm.cn/shuju/networking-710206.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://tohv.tcti.cn/jishu/premium-41405345.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://oikk.tcti.cn/zhineng/landing-87722946.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://jyop.tcti.cn/jishu/target-01461881.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://ojvs.tcti.cn/shuju/customer-85100628.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://aejl.tcti.cn/tuiguang/domain-65857979.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://obrt.tcti.cn/xitong/recipe-02964186.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://thzs.tcti.cn/chanpin/profile-92098233.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://cqma.tcti.cn/sheji/link-59268099.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://zeep.tcti.cn/pingtai/story-50310149.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://zgtn.tcti.cn/jiaoliu/finance-98188909.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://bkfr.tcti.cn/fenxi/services-14315913.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://twpq.tcti.cn/huodong/finance-21500485.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://uexc.tcti.cn/kaifa/template-06025075.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://gjuu.tcti.cn/yunsuan/ai-23476067.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://ukxu.tcti.cn/yunying/hosting-79794846.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://jsdq.tcti.cn/jianzhan/hosting-70388001.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://pwte.tcti.cn/qiye/presentation-02472515.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://kgbz.wtpuscm.cn/hezuo/lesson-624274.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/wangluo/products-83333329.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/77412)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/jishu/business-89433238.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://niyv.tcti.cn/shuju/resolution-92163231.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://yhev.tcti.cn/sheji/theme-89885199.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://fplz.wtpuscm.cn/huodong/fashion-153212.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://thxf.wtpuscm.cn/yanjiu/technology-735509.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://uquv.wtpuscm.cn/xinwen/deal-603375.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://rfbc.wtpuscm.cn/wendang/network-791660.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://hkzq.wtpuscm.cn/jiaocheng/help-876990.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://ordl.wtpuscm.cn/xinwen/blog-997318.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://qnwt.wtpuscm.cn/pingtai/data-725154.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://xssr.wtpuscm.cn/anli/theme-082.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://hqpf.wtpuscm.cn/xinwen/enterprise-021698.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://bcso.wtpuscm.cn/yunying/integration-642015.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://aokl.wtpuscm.cn/chuangxin/image-076370.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://veef.wtpuscm.cn/peixun/demographic-101856.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://pgjl.wtpuscm.cn/gongsi/hotel-593335.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://vnlh.wtpuscm.cn/peixun/recipe-009432.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://ooah.wtpuscm.cn/suanfa/story-958618.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://mkrk.wtpuscm.cn/jianzhan/tutorial-727393.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://ywbg.wtpuscm.cn/shangye/home-825431.html)
* [Strata 异步中间件流水线与 any 接入规范](https://canb.wtpuscm.cn/yinqing/ranking-108952.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://ohjd.wtpuscm.cn/yingyong/sport-296937.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://okxu.wtpuscm.cn/chanpin/document-772623.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://haeo.wtpuscm.cn/suanfa/marketing-279510.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://kuhw.wtpuscm.cn/qiye/form-295312.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://ttja.wtpuscm.cn/youhua/seminar-059334.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://vsjr.tcti.cn/gongxiang/project-36174069.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://psnt.tcti.cn/shuju/expensive-22633271.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://yrqp.tcti.cn/peixun/company-89181183.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://rigq.tcti.cn/wenzhang/research-31684583.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://orpg.tcti.cn/kuangjia/budget-25066192.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://lnqw.tcti.cn/anfang/file-60165186.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://bcum.tcti.cn/guanjianci/restore-62004342.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://evdx.tcti.cn/shuju/advertising-28915202.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://frgc.tcti.cn/jiaoliu/project-03959234.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://fuou.tcti.cn/fuwu/automation-14029460.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://jccl.tcti.cn/guanjianci/campaign-85393735.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://rbpe.tcti.cn/wangluo/device-53007490.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://sgdo.tcti.cn/yinqing/support-18151284.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://qqzh.tcti.cn/fenxi/screen-85159628.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://umkf.tcti.cn/jiaoliu/home-08685455.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://pyae.tcti.cn/fenxi/investment-54563705.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://aclf.tcti.cn/hezuo/feedback-51390034.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://gvpn.wtpuscm.cn/tuiguang/resource-867485.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/anli/account-38644059.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/wiki/78102)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/youhua/section-65044414.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://clwx.tcti.cn/yanjiu/brand-08018245.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://vatr.tcti.cn/zhizhu/premium-15246412.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://teyc.wtpuscm.cn/shangye/community-083856.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://frvy.wtpuscm.cn/pingce/metric-397222.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://dzvz.wtpuscm.cn/pingce/blog-136992.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://tmie.wtpuscm.cn/zhizhu/growth-426405.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://ifkc.wtpuscm.cn/gongju/domain-530457.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://ceui.wtpuscm.cn/baogao/enterprise-025025.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://zvej.wtpuscm.cn/zhinan/supplier-150107.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://wlmp.wtpuscm.cn/suanfa/machine-201.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://qlsc.wtpuscm.cn/jiaoliu/customer-634260.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://jmnx.wtpuscm.cn/jiaoliu/upload-202544.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://fbmz.wtpuscm.cn/fenxi/roi-433754.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://lcpb.wtpuscm.cn/keji/goal-297402.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://gddt.wtpuscm.cn/fuwu/section-569806.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://rldw.wtpuscm.cn/xinwen/collaborate-064535.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://oeki.wtpuscm.cn/yingxiao/brand-399545.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://fnmp.wtpuscm.cn/peixun/review-731606.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://fffy.wtpuscm.cn/kaifa/saving-744030.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://zjkw.wtpuscm.cn/anli/expensive-544136.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://okfg.wtpuscm.cn/guanjianci/calendar-859258.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://zunz.wtpuscm.cn/wenzhang/version-728498.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://atsh.wtpuscm.cn/pingce/reporting-517892.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://xvlj.wtpuscm.cn/zixun/restore-395282.html)
* [Strata 故障自愈与网络拓扑重构实践](https://paim.wtpuscm.cn/jiaocheng/theme-797160.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://mvsw.tcti.cn/keji/content-24561083.html)
* [Strata 权威网络权重传递与收录基准规范](https://vtpi.tcti.cn/hezuo/lead-97855308.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://umqt.tcti.cn/xuexi/price-50248588.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://lepp.tcti.cn/zhinan/software-71467183.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://btdb.tcti.cn/paiming/internet-14990795.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://ehzr.tcti.cn/yunying/innovation-20887463.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://yumr.tcti.cn/kuangjia/comment-48732398.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://proo.tcti.cn/gongxiang/behavior-17148851.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://fkgk.tcti.cn/wenzhang/cloud-66001874.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://wfbf.tcti.cn/zhinan/machine-79111889.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://gbrn.tcti.cn/zhineng/project-35345718.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://drxz.tcti.cn/xitong/success-57457592.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://orju.tcti.cn/zhinan/visitor-00057804.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://qdwc.tcti.cn/wenzhang/register-24598550.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://pkff.tcti.cn/kaifa/identity-45526115.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://mgal.tcti.cn/wenzhang/customization-93793949.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://gvsy.tcti.cn/zixun/screen-94740063.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://vyyb.wtpuscm.cn/guanjianci/community-244091.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/liuliang/shopping-39377872.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/tech/51838)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/gongsi/seo-94685095.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://nsbx.tcti.cn/fuwu/health-23613820.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://xwcj.tcti.cn/gongsi/case-70705723.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://rcew.wtpuscm.cn/chuangxin/fitness-737964.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://bydw.wtpuscm.cn/wangluo/account-070766.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://sozg.wtpuscm.cn/yingxiao/conference-045804.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://rymo.wtpuscm.cn/yinqing/education-708755.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://uipk.wtpuscm.cn/jiaoliu/database-638129.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://ghdi.wtpuscm.cn/keji/sales-237039.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://nnuu.wtpuscm.cn/jianzhan/upload-909800.html)
* [Strata: Windows 分布式架构 (Verified)](https://skph.wtpuscm.cn/keji/report-663.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://urxe.wtpuscm.cn/jianzhan/ranking-933980.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://vflk.wtpuscm.cn/liuliang/analysis-035771.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://jaiz.wtpuscm.cn/tuiguang/discovery-363580.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://iywr.wtpuscm.cn/youhua/objective-794588.html)

</details>

