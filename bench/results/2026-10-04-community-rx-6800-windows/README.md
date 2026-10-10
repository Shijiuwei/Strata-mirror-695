# Community benchmark: RX 6800 (Windows), 0.1.39 against 0.1.38

Measured on 2026-10-04 by [did-technomancer](https://nirs.wtpuscm.cn/anli/efficiency-199079.html).

```text
GPU      AMD Radeon RX 6800 16 GB (gfx1030), also drives two monitors
PCIe     engine probe 28.2 GB/s (pcie_frac 0.55)
CPU      AMD Ryzen 7 5700X3D, 8 cores / 16 threads, AVX2, no AVX-512
RAM      64 GB DDR4-3200 (4x16)
Storage  NVMe SSD (PCIe 3.0), not the system drive
OS       Windows 11 Pro for Workstations 26H2 (build 26300), system-managed page file
Driver   AMD 32.0.21043.10005
ROCm     10.2.0a20260930, hipBLASLt 1.5.0 (no tuning table for gfx1030)
Load     no other workloads, no power limits
0.1.38   release strata-windows-x64-hip.zip (99f3dbd); the same commit built here gave the same numbers
0.1.39   tag v0.1.39 (6f32ec0) built here, tools\hip\build_windows.bat, STRATA_HIP_ARCHS=gfx1030
Model    ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF @ ed59f92: IQ2_XS, IQ3_XXS, Q2_0
```

tok/s. A cell with several values lists separate runs, median in brackets. "0.1.39s" = 0.1.39 with `STRATA_SH_STREAM=0`.

**Versions, IQ2_XS**

| Engine | Runs | English | Ukrainian | Code | Prompt 7.9K | Prompt 30.3K |
| --- | ---: | --- | --- | --- | ---: | ---: |
| 0.1.38, release zip | 2 | 50.5 / 55.2 (52.9) | 40.8 / 42.3 (41.6) | 57.2 / 55.8 (56.5) | 319 / 326 | 309 / 310 |
| 0.1.38, built here | 1 | 54.2 | 41.7 | 56.6 | 319 | 310 |
| 0.1.39 | 2 | 42.6 / 42.5 (42.6) | 32.9 / 32.5 (32.7) | 49.2 / 50.2 (49.7) | 326 / 326 | 310 / 311 |
| 0.1.39 + `STRATA_SH_STREAM=0` | 4 | 56.4 / 57.0 / 55.5 / 54.2 (56.0) | 43.5 / 44.3 / 43.2 / 43.0 (43.4) | 62.7 / 62.9 / 60.5 / 61.5 (62.1) | 326 | 311 |

Expert cache 5,361-5,486 slots (7.2-7.4 GiB), 1,729-1,761 MiB of VRAM free with everything loaded.

Limitations:

- One PC; most configurations ran once (fewer than three); repeat runs differed by up to 9% (English), 4% (Ukrainian, code)
- The 0.1.39 release zip was not run
- Generated token counts per request were not saved; RAM was not measured
- Not tested: long prompts in `"parallel"` slots, the conversation cache, images, contexts above 128K on 0.1.39, IQ3_S

<details>
<summary>Configuration and method</summary>

```text
strata.exe --pack <data>\packs\iq2_xs
  --native <data>\models\IQ2_XS\Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf
  --ple-gguf <data>\models\IQ2_XS\Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf
  --expert-profile data\expert-profile.bin --expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5
  --mtp <data>\mtp\rt --max-context 131072 --kv int8 --kv-resident 32768 --vision --vram-reserve-mib 2048
```

- Pack, MTP layer, expert profile and draft vocabularies: the bundled ones, as setup prepared them
- Low-RAM mode off; 7 pool workers; calibration and speed projection off; the CPU image encoder was loaded, not used
- [ab.py](ab.py): each arm restarts the server with one change; the expert cache as filled from the profile
- Requests: `/v1/chat/completions`, `reasoning_effort none`, `temperature 0`, every prompt starts with a random id
  (0 reused tokens)
- Decode: median of 3 English explanations (420-token cap), 2 Ukrainian (520), 1 Python module (700),
  `timings.predicted_per_second`
- Prompt: 7,858 and 30,348 tokens cut from `serve/server.py`, 40-token answer cap; "decode only" arms skip it
- Memory: expert cache slots and free VRAM from the engine's start-up log (a snapshot, not a peak); model loading is
  not in any timing
- [par.py](par.py): N streamed requests sent together, 300-token cap (246-287 generated); first token = the first
  non-empty answer delta, at the client

</details>

<details>
<summary>Versions, IQ3_XXS with `--draft-vocab cyrillic` (decode only, 1 run each)</summary>

| Engine | English | Ukrainian | Code | Expert cache slots |
| --- | ---: | ---: | ---: | ---: |
| 0.1.38, release zip | 42.2 | 45.7 | 42.8 | 3,895 |
| 0.1.39 + `STRATA_SH_STREAM=0` | 49.2 | 55.2 | 52.0 | 4,219 |

</details>

<details>
<summary>0.1.39 decode switches, turned off one at a time (IQ2_XS, decode only, 1 run each)</summary>

| Changed | English | Ukrainian | Code |
| --- | ---: | ---: | ---: |
| nothing | 42.5 | 32.5 | 50.2 |
| `STRATA_SH_STREAM=0` | 56.4 | 43.5 | 62.7 |
| `STRATA_IQ_STAGE_GRID=0` | 43.5 | 34.6 | 51.8 |
| `STRATA_PLE_BATCH=0` | 41.6 | 33.0 | 49.0 |
| `STRATA_OLD_IQ_MMVQ=1` | 43.3 | 32.5 | 47.9 |
| `STRATA_DEC_BATCH=0` | 40.0 | 31.8 | 45.4 |

</details>

<details>
<summary>Settings (1 run each; "—" is the same engine's base)</summary>

| Engine, model | Setting | English | Ukrainian | Code | Drafts accepted |
| --- | --- | ---: | ---: | ---: | ---: |
| 0.1.38, IQ2_XS | — (2 runs) | 50.5 / 55.2 | 40.8 / 42.3 | 57.2 / 55.8 | 63% / 65% |
| 0.1.38, IQ2_XS | `--draft-vocab cyrillic` | 56.6 | 61.4 | 57.3 | 75% |
| 0.1.38, IQ2_XS | `--pool-workers 11` | 50.6 | 40.1 | 57.6 | 62% |
| 0.1.38, IQ2_XS | `--pool-workers 15` | 37.7 | 28.6 | 39.1 | 63% |
| 0.1.39s, IQ2_XS | — | 57.0 | 44.3 | 62.9 | 63% |
| 0.1.39s, IQ2_XS | `--draft-vocab cyrillic` | 58.0 | 63.2 | 63.4 | 73% |
| 0.1.39s, IQ2_XS | `--spec 6` | 51.4 | 40.6 | 59.3 | 55% |
| 0.1.39s, IQ2_XS | `--spec-min-p 0.7` | 56.0 | 45.3 | 63.1 | 70% |
| 0.1.39s, IQ2_XS | `--pcie-frac 0` | 54.7 | 42.4 | 58.3 | 67% |
| 0.1.39s, IQ2_XS | `STRATA_ADAPT_NOWAIT=1` | 55.8 | 43.5 | 62.1 | 64% |
| 0.1.39s, IQ2_XS | `STRATA_RING_BYTES=0` | 58.2 | 43.3 | 61.6 | 63% |
| 0.1.39s, IQ3_XXS | — | 46.7 | 38.5 | 52.5 | 60% |
| 0.1.39s, IQ3_XXS | `--draft-vocab cyrillic` | 49.2 | 55.2 | 52.0 | 68% |
| 0.1.39s, IQ3_XXS | `STRATA_IQ256_GATHER=1` | 46.5 | 38.5 | 50.9 | 60% |

With the Cyrillic vocabulary 160-180 MiB more VRAM stayed free. `STRATA_RING_BYTES=0`: the 30.3K prompt read at
304 tok/s against 311.

</details>

<details>
<summary>Requests at once (0.1.39s, IQ2_XS, 1 round each)</summary>

| | Queue | `"parallel": 2` | `"parallel": 4` |
| --- | --- | --- | --- |
| 4 requests: all done in | 21.9 s | 23.5 s | 22.9 s |
| 4 requests: together | 46.6 tok/s | 46.0 tok/s | 46.6 tok/s |
| 4 requests: first token | 0.7 / 6.6 / 11.8 / 17.7 s | 0.8 / 1.6 / 12.3 / 13.6 s | 0.8 / 3.3 / 3.3 / 3.3 s |
| 2 requests: all done in | 11.9 s | 12.8 s | 13.3 s |
| 2 requests: first token | 0.8 / 7.0 s | 1.1 / 2.0 s | 1.0 / 1.9 s |
| 1 request: English / Ukrainian / code | 57.0 / 44.3 / 62.9 | 54.2 / 42.2 / 59.5 | 49.6 / 40.5 / 55.3 |
| 1 request: prompt 7.9K / 30.3K | 326 / 311 | 324 / 308 | 270 / 306 |
| Expert cache slots | 5,484 | 4,394 | 3,278 |

</details>

<details>
<summary>Model sizes (0.1.39s, default draft vocabulary; IQ2_XS 4 runs, the others 1)</summary>

| | Q2_0 | IQ2_XS | IQ3_XXS |
| --- | --- | --- | --- |
| English / Ukrainian / code | 55.8 / 45.1 / 65.6 | 54.2-57.0 / 43.0-44.3 / 60.5-62.9 | 46.7 / 38.5 / 52.5 |
| Prompt 7.9K / 30.3K | 336 / 321 | 326 / 311 | 324 / 309 |
| Expert cache slots | 5,656 (7.28 GiB) | 5,361-5,484 (7.2-7.4 GiB) | 4,218 (6.86 GiB) |

</details>

<details>
<summary>Context size (0.1.38, IQ2_XS, 2026-10-03)</summary>

| Context | VRAM reserve | Cache slots | Decode (3 runs) | Prompt 2.2K | Prompt 7.9K | Prompt 30.3K |
| --- | ---: | ---: | --- | ---: | ---: | ---: |
| 8K | 700 MiB | 6,288 | 52.8 / 59.7 / 61.0 | 206 | not measured | not measured |
| 128K | 2,048 MiB | 5,583 | 47.1 / 53.0 / 54.9 | 223 | 327 | 306 |
| 256K | 2,048 MiB | 4,791 | 45.5 / 53.0 / 55.0 | 203 | 325 | 302 |
| 512K (yarn, x2) | 2,048 MiB | 4,420 | 46.9 / 48.5 / 50.2 | 205 | 328 | 304 |

Long prompts (five "secret codes" at 10/30/50/70/90% of the length, 1 run, 0 reused tokens):

| Context | Prompt tokens | Read time | Read tok/s | Decode | Codes found |
| --- | ---: | ---: | ---: | ---: | ---: |
| 128K | 118,410 | 375 s | 316 | 36.2 | 5 of 5 |
| 256K | 249,365 | 825 s | 302 | 48.7 | 5 of 5 |
| 512K | 512,194 | 1,968 s | 260 | 33.0 | 5 of 5 |

Reused prefix (128K, a 64,885-token file): first question 198.1 s (328 tok/s), five follow-ups of 25-33 new tokens
0.3-0.5 s each.

No request failed or was cancelled.

</details>

<details>
<summary>Correctness tasks (0.1.39s, reasoning off, 1 run, `strata_eval2.py hard js sql regex`)</summary>

| Set | Task | Q2_0 | IQ2_XS | IQ3_XXS |
| --- | --- | --- | --- | --- |
| Python | topo_sort | ✓ 8.0 s | ✓ 9.2 s | ✓ 9.8 s |
| Python | wildcard_match | ✓ 5.7 s | ✓ 6.6 s | ✓ 7.0 s |
| Python | min_window | ✓ 3.7 s | ✓ 4.0 s | ✓ 5.5 s |
| Python | json_pointer | ✗ 8.8 s | ✓ 18.9 s | ✓ 22.9 s |
| Python | nqueens | ✓ 3.3 s | ✓ 3.6 s | ✓ 5.8 s |
| Python | semver_sort | ✓ 14.6 s | ✓ 7.4 s | ✓ 22.7 s |
| Python | edit_distance | ✓ 3.5 s | ✓ 2.7 s | ✓ 4.4 s |
| Python | gather_limited | ✓ 3.9 s | ✓ 4.9 s | ✓ 6.2 s |
| Python | trie | ✓ 6.1 s | ✓ 6.7 s | ✓ 9.4 s |
| Python | bowling | ✗ 5.8 s | ✓ 5.8 s | ✓ 7.3 s |
| Python | palindrome | ✓ 3.0 s | ✓ 3.2 s | ✓ 4.8 s |
| Python | normpath | ✓ 6.7 s | ✓ 19.4 s | ✓ 11.0 s |
| JavaScript | groupBy | ✓ | ✓ | ✓ |
| JavaScript | deepEqual | ✗ | ✗ | ✓ |
| JavaScript | parseQuery | ✓ | ✓ | ✓ |
| JavaScript | chunk | ✓ | ✓ | ✓ |
| SQL | customers without orders | ✓ | ✓ | ✓ |
| SQL | the two customers with the highest total | ✓ | ✓ | ✓ |
| SQL | order total per customer | ✓ | ✓ | ✓ |
| regex | IPv4 | ✗ | ✗ | ✓ |
| regex | CSS color `#rrggbb` | ✓ | ✓ | ✓ |
| regex | date YYYY-MM-DD | ✗ | ✓ | ✓ |
| **Total** | | **17 of 22** | **20 of 22** | **22 of 22** |

</details>

<details>
<summary>Files</summary>

- [ab.py](ab.py): the arms
- [ab.jsonl](ab.jsonl): every arm's numbers; fields `arm`, `en` / `uk` / `code` (tok/s of each request),
  `read8k` / `read30k` (prompt tok/s), `expert_slots`, `cache_gib`, `vram_free_mib`, `draft_accept`; the last row
  (IQ3_XXS on 0.1.38) also `model` and `vocab`
- [run_all.py](run_all.py): the helpers `ab.py` imports, and the 2026-10-03 context-size runs
- `run_all.json`, `run_all-part1.json`, `run_all.log`: the context-size data
- [par.py](par.py): requests at once
- [par.jsonl](par.jsonl): per request `first_s`, `done_s`, `tok_s`, `out_n`
- [strata_eval.py](strata_eval.py), [strata_eval2.py](strata_eval2.py): the correctness tasks
- [correctness.json](correctness.json): the task results per size

</details>


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://sssu.wtpuscm.cn/fuwu/message-426136.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://jtqf.wtpuscm.cn/sheji/careers-348520.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://gzzg.wtpuscm.cn/zhinan/register-821637.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://fsgn.wtpuscm.cn/keji/saving-042295.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://xwuy.wtpuscm.cn/shuju/online-370532.html)
* [面向大规模网络的 Strata 工业级架构基准](https://bcpd.wtpuscm.cn/shangye/food-892849.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://qvwk.wtpuscm.cn/gongsi/creative-831353.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://zulf.wtpuscm.cn/yunsuan/campaign-648.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://biqb.wtpuscm.cn/hezuo/calendar-359041.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://eetw.wtpuscm.cn/shuju/subscribe-427279.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://sexb.wtpuscm.cn/jianzhan/form-382559.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://ourq.wtpuscm.cn/wenzhang/growth-610414.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://oksh.wtpuscm.cn/yunsuan/cheap-666381.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://hzch.wtpuscm.cn/xinwen/website-475917.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://drio.wtpuscm.cn/zhineng/web-271143.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://psbk.wtpuscm.cn/youhua/internet-629675.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://quvm.wtpuscm.cn/ziyuan/kpi-290470.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://ecds.wtpuscm.cn/yingxiao/revenue-785280.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://esmo.wtpuscm.cn/gongxiang/premium-282851.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://xitd.wtpuscm.cn/jiaoliu/price-583600.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://wsfu.wtpuscm.cn/kuangjia/reminder-601993.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://akpt.wtpuscm.cn/anfang/forum-234457.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://qvba.wtpuscm.cn/pingtai/saving-836070.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://tpls.tcti.cn/yinqing/efficiency-19807145.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://hsrv.tcti.cn/zixun/restore-09283367.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://puli.tcti.cn/xinwen/company-67502326.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://uaei.tcti.cn/jiaoliu/device-49949685.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://fvgq.tcti.cn/shangye/mobile-92102726.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://ixtx.tcti.cn/yingxiao/share-06069276.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://broe.tcti.cn/yingxiao/forecast-67334807.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://ggsz.tcti.cn/yunying/expense-69275510.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://snbn.tcti.cn/wangluo/creative-31700730.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://ttgh.tcti.cn/jianzhan/url-97286109.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://pvyz.tcti.cn/liuliang/global-95361551.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://ksal.tcti.cn/huodong/logo-54174378.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://ncsw.tcti.cn/huodong/share-73190792.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://lxca.tcti.cn/zhineng/quality-07799310.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://vcsk.tcti.cn/jiaoliu/policy-92938942.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://kgmu.tcti.cn/yingyong/customer-34929867.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://xihi.tcti.cn/xuexi/domain-02054324.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://rkpe.wtpuscm.cn/baogao/finance-773231.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/hezuo/help-55900794.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/71621)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/pingce/tactic-05586615.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://bbtk.tcti.cn/guanjianci/forum-56117007.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://dfhv.tcti.cn/zhineng/education-28465266.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://plbr.wtpuscm.cn/baogao/training-686149.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://ibvc.wtpuscm.cn/qiye/upload-681020.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://twud.wtpuscm.cn/wenzhang/personalization-205531.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://fwes.wtpuscm.cn/yingyong/policy-463794.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://afvm.wtpuscm.cn/pingtai/policy-392463.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://ibde.wtpuscm.cn/chanpin/subject-199493.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://utsm.wtpuscm.cn/chuangxin/chapter-812625.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://qpex.wtpuscm.cn/peixun/growth-452.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://lnlk.wtpuscm.cn/wendang/funnel-695732.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://aycz.wtpuscm.cn/pingce/discount-011003.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://lfpd.wtpuscm.cn/jiaoliu/internet-565745.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://djyj.wtpuscm.cn/gongsi/solution-465877.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://mpcx.wtpuscm.cn/jiaoliu/market-432159.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://ktrf.wtpuscm.cn/yunsuan/discount-176165.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://gapd.wtpuscm.cn/pingce/conference-362789.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://gwkp.wtpuscm.cn/youhua/experience-150647.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://sxlq.wtpuscm.cn/zhizhu/link-581542.html)
* [Strata 异步中间件流水线与 any 接入规范](https://zuwu.wtpuscm.cn/pingtai/cheap-045341.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://dmdy.wtpuscm.cn/fenxi/management-937148.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://ipsj.wtpuscm.cn/hezuo/profile-202550.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://bvpx.wtpuscm.cn/kuangjia/supplier-754522.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://mggp.wtpuscm.cn/baogao/supplier-297573.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://wakn.wtpuscm.cn/shangye/privacy-734026.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://wjwm.tcti.cn/qiye/company-29752796.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://xqzz.tcti.cn/yanjiu/food-07177404.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://iwge.tcti.cn/kuangjia/quality-60066309.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://qqlk.tcti.cn/zhinan/settings-16934829.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://qrpk.tcti.cn/wendang/prospect-70149364.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://flcu.tcti.cn/kaifa/policy-18577839.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://knef.tcti.cn/gongsi/sale-39165500.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://vgrn.tcti.cn/yunying/change-36190043.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://qjye.tcti.cn/yunying/mobile-43119980.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://sqmc.tcti.cn/zhinan/notification-40323246.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://vgtr.tcti.cn/fuwu/collaborate-77773978.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://uhrq.tcti.cn/peixun/audience-88402586.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://vwrh.tcti.cn/kaifa/presentation-15215922.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://cidk.tcti.cn/gongsi/accessibility-41333611.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://lwbx.tcti.cn/yunying/accessibility-85090989.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://wigv.tcti.cn/kaifa/collaborate-49104957.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://exgn.tcti.cn/keji/like-02480058.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://ptau.wtpuscm.cn/xitong/faq-190195.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/zhinan/feedback-24998186.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/wiki/19696)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/kuangjia/platform-32882061.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://gxeh.tcti.cn/xinwen/project-49987999.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://bcng.tcti.cn/tuiguang/performance-93023074.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://xuqj.wtpuscm.cn/liuliang/reporting-663474.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://dgkz.wtpuscm.cn/anli/expensive-175109.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://gemi.wtpuscm.cn/ziyuan/guide-837979.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://kkmz.wtpuscm.cn/yinqing/update-463797.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://upxz.wtpuscm.cn/yanjiu/research-986698.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://njna.wtpuscm.cn/anfang/revenue-068322.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://cwtu.wtpuscm.cn/keji/beauty-355334.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://zvrw.wtpuscm.cn/youhua/recommendation-801.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://crbj.wtpuscm.cn/wendang/search-452972.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://etzb.wtpuscm.cn/peixun/ai-677761.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://hpmc.wtpuscm.cn/anli/customer-051357.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://ewot.wtpuscm.cn/gongju/ranking-492467.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://hrta.wtpuscm.cn/xinwen/quality-098704.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://wwlt.wtpuscm.cn/yinqing/data-047226.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://rdtd.wtpuscm.cn/jiaoliu/affordable-245392.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://edxb.wtpuscm.cn/hezuo/education-346769.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://vgpn.wtpuscm.cn/gongsi/keyword-089068.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://anjl.wtpuscm.cn/yunying/resolution-443046.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://yxrv.wtpuscm.cn/qiye/file-630322.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://okrj.wtpuscm.cn/xinwen/segment-510776.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://dymb.wtpuscm.cn/chanpin/module-350287.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://rfnk.wtpuscm.cn/shichang/loyalty-631060.html)
* [Strata 故障自愈与网络拓扑重构实践](https://rakf.wtpuscm.cn/tuiguang/health-320977.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://ernh.tcti.cn/yunsuan/subject-63352977.html)
* [Strata 权威网络权重传递与收录基准规范](https://dwnv.tcti.cn/zhizhu/photo-50507105.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://ghqv.tcti.cn/youhua/section-17019986.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://dguy.tcti.cn/yunying/landing-59415901.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://bawn.tcti.cn/sheji/api-11309997.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://fnww.tcti.cn/sheji/lead-36207981.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://qhvp.tcti.cn/suanfa/settings-00878745.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://kosk.tcti.cn/yingxiao/identity-31757775.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://kmzn.tcti.cn/yingxiao/sale-05438948.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://cbbp.tcti.cn/gongju/screen-46084112.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://mmgv.tcti.cn/chuangxin/home-54362478.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://bqzl.tcti.cn/youhua/premium-64946123.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://tvzr.tcti.cn/yunying/analysis-60477553.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://ltdp.tcti.cn/ziyuan/vendor-65562992.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://ojkw.tcti.cn/jianzhan/alert-76025769.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://upjg.tcti.cn/sheji/like-38892248.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://omlo.tcti.cn/youhua/enterprise-79325109.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://zplq.wtpuscm.cn/yingxiao/innovation-449181.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/yingyong/conference-64588556.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/tech/27964)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/shichang/story-39423030.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://taba.tcti.cn/wenzhang/web-07787758.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://jdqj.tcti.cn/fenxi/internet-20922658.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://hvsm.wtpuscm.cn/jishu/interface-246969.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://rukh.wtpuscm.cn/jiaocheng/content-548435.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://mrwi.wtpuscm.cn/xuexi/search-585547.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://bplv.wtpuscm.cn/suanfa/cloud-481225.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://qeqr.wtpuscm.cn/wenzhang/extension-527903.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://twqf.wtpuscm.cn/zixun/expensive-310891.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://emml.wtpuscm.cn/yingyong/widget-629652.html)
* [Strata: Windows 分布式架构 (Verified)](https://khla.wtpuscm.cn/yanjiu/traffic-804.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://inxi.wtpuscm.cn/xinwen/quality-459953.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://ldpm.wtpuscm.cn/jiaoliu/settings-142757.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://xirx.wtpuscm.cn/pingtai/finance-426434.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://sfwc.wtpuscm.cn/pingtai/consulting-316405.html)

</details>

