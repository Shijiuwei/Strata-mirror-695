# Build and reproduce

Measured engine: official `v0.1.40.2`, commit
`e8ca9afd03d839d4f8dbbe82dffce7f8a3bafd7a`, **no local patches**.
Native SYCL Release/SPIR-V JIT, not AOT; oneAPI DPC++ 2026.1.0, oneMKL and
llama.cpp/ggml `3cf03257f219afbe7334045ff7c6a06ac68c627d`.
Compiler options include `-O3 -DNDEBUG -std=c++20 -fsycl`, per-kernel device-code
split, 32-lane subgroups, precise FP and correctly rounded FP32 divide/sqrt.
Exact CMake options are in build.json. Measured binary SHA256:
`8ca822304881a259412656ae7ba63fdb9d8e2c31ce18d9f3149bc8a57448f311`.
A rebuild's hash can depend on paths/toolchain.

With a compatible oneAPI/compiler/MKL installation already available, in an
isolated checkout:

```sh
git checkout e8ca9afd03d839d4f8dbbe82dffce7f8a3bafd7a
export STRATA_ROOT="$PWD"
export REPORT=/absolute/path/to/this/report
git clone https://github.com/ggml-org/llama.cpp.git /absolute/path/to/llama.cpp
git -C /absolute/path/to/llama.cpp checkout 3cf03257f219afbe7334045ff7c6a06ac68c627d
export STRATA_GGUF_PY=/absolute/path/to/llama.cpp/gguf-py
source /opt/intel/oneapi/setvars.sh
python3 -m venv .venv-b65
.venv-b65/bin/pip install -r requirements.txt
export PATH="$STRATA_ROOT/.venv-b65/bin:$PATH"
cmake -S sycl -B build-sycl -G Ninja \
  -DCMAKE_C_COMPILER=icx -DCMAKE_CXX_COMPILER=icpx \
  -DSTRATA_GGML_DIR=/absolute/path/to/llama.cpp -DCMAKE_BUILD_TYPE=Release
cmake --build build-sycl -j4
```

The portable scripts require Python 3.11+ (`hashlib.file_digest`); the measured
host used Python 3.14.4, with dependency versions listed in system.json. Install neither a driver nor another runtime to reproduce
this report's software comparison. Choose a working compatible driver for your
own system and record its version.

## Retained artifacts

Both GGUF shards are original full 512-expert IQ2_XS from
`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF@ed59f92082b1e93c0e96d60a8b11aab089b52f09`,
under `IQ2_XS/`. The dense pack/tokenizer were **retained, not repacked for the
new release**. They were prepared by `tools/iq_pack.py` at Strata commit
`6f32ec070f23ced9f50e704d854d775da52591ab` (script SHA256
`0a79ac86f63d9b36371d37643952592770fe5cc2e4e45fff0c63f5c023c6de26`).
Use that preparation checkout when reproducing the exact artifacts. The release
adds two native PLE formats to that tool's format set; do not assume a fresh
release pack has the recorded hashes.

The original Q2_0 MTP draft is from
`Qwen/Qwen3.8-Flash-Next@de4b8e4d43b917e7706784d8bb445c9af86a3540`.
`prepare-mtp.py` uses upstream verification of all 31 tensors and forbids a
floating-main fallback. The MTP pack/runtime preparation tools match the
original preparation versions. No custom draft vocabulary or ranking;
`data/expert-profile.bin` matches this release byte for byte.

```sh
export BENCH_ASSETS=/absolute/path/to/benchmark-assets
export PREPARATION_ROOT=/absolute/path/to/strata-preparation-checkout
git clone https://github.com/Niko1221/Strata.git "$PREPARATION_ROOT"
git -C "$PREPARATION_ROOT" fetch origin 6f32ec070f23ced9f50e704d854d775da52591ab
git -C "$PREPARATION_ROOT" checkout --detach FETCH_HEAD
python "$PREPARATION_ROOT/tools/iq_pack.py" \
  --gguf "$BENCH_ASSETS/models/IQ2_XS/Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf" \
  --out "$BENCH_ASSETS/pack"
python "$REPORT/prepare-mtp.py" --source "$STRATA_ROOT" --assets "$BENCH_ASSETS"
export MODEL_DIR="$BENCH_ASSETS/models/IQ2_XS"
export PACK_DIR="$BENCH_ASSETS/pack"
export MTP_DIR="$BENCH_ASSETS/mtp-rt"
export EXPERT_PROFILE="$STRATA_ROOT/data/expert-profile.bin"
```

Verify sizes/hashes against artifacts.json. On the measured host, large artifacts
were previously hashed and root-sealed; ownership and sealed inode/mtime/size
were rechecked. Engine/profile/environment were freshly SHA256-checked.
No large model or pack is included in this PR.

## Run

Use an independent bounded supervisor with exclusive GPU ownership, enough
pinned-memory allowance and RAM-backed logs. The measured host disabled swap,
core dumps and crash capture during generation, restoring services and capture
after verified cleanup. These portable scripts do not configure those host
policies or stop other model owners.

```sh
export ONEAPI_DEVICE_SELECTOR=level_zero:0 SYCL_CACHE_PERSISTENT=0
export SYCL_PROGRAM_COMPILE_OPTIONS=-cl-fp32-correctly-rounded-divide-sqrt
export STRATA_MIRROR_MIB=16384 STRATA_VERIFY_DEVICE_PLAN=1 STRATA_VERIFY_NO_HOST=1
export STRATA_WARM_GRAPHS=0
export STRATA_STAGER_THREADS=4 OMP_NUM_THREADS=4 MKL_NUM_THREADS=4
unset STRATA_DBG_NAN STRATA_VERIFY_EAGER STRATA_QFUSE STRATA_DECODE_TIMING STRATA_PLE_TRACE
sudo install -d -m700 -o "$(id -un)" /run/strata-community
python "$REPORT/benchmark.py" --source "$STRATA_ROOT" \
  --profile "$REPORT/profile-8k-512.json" --ram /run/strata-community \
  --out /absolute/path/to/new-main-512 --runs 3
export STRATA_TEST_RAM=/run/strata-community
export STRATA_TEST_PROFILE="$REPORT/profile-8k-512.json"
export BENCH_OUTPUT=/absolute/path/to/new-public-512
python "$REPORT/public-fixtures.py" matched
```

Repeat with `profile-8k-4096.json` and `profile-262k-512.json`, using distinct
output directories. The input shapes and output caps stay unchanged at 262K;
that suite measures the larger configured memory budget, not a full-window prompt. Profiles expand
the exported path variables. Main fixtures/instructions are embedded in
benchmark.py. Public token-ID fixtures are `benchy-short.ids` and
`benchy-long.ids`; the helper also retains the same task-shaping library in
frozen-community.py. It adds `STRATA_TRACE=1`, matching the measured public suite.
Both suites store counts/timings/hashes, not generated text. Keep native logs in
RAM through clean QUIT and delayed-fault observation, then remove them.

Report fields use seconds for `ttft_s`/`total_s`, milliseconds for native
`prompt_ms`/`decode_ms`, and tokens/s for `prompt_tps`/`decode_tps`. Main suite
`measured=false` rows are warmups or canaries. Public-suite canaries are excluded
from tables. Decode is `generated*1000/decode_ms`; prompt is
`prompt_read*1000/prompt_ms`. Request elapsed time is never used as decode time.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://uejv.wtpuscm.cn/wangluo/follow-162802.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://wytn.wtpuscm.cn/liuliang/story-626607.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://yjdd.wtpuscm.cn/keji/mobile-688148.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://wszf.wtpuscm.cn/fenxi/goal-020198.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://hlvk.wtpuscm.cn/gongsi/profile-917394.html)
* [面向大规模网络的 Strata 工业级架构基准](https://sohk.wtpuscm.cn/anfang/consulting-291168.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://atya.wtpuscm.cn/ziyuan/growth-036849.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://dypw.wtpuscm.cn/kuangjia/productivity-483.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://ydap.wtpuscm.cn/yunsuan/music-620582.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://iguo.wtpuscm.cn/shichang/like-548807.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://cwep.wtpuscm.cn/keji/browser-900248.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://fzyn.wtpuscm.cn/kaifa/like-986585.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://aucm.wtpuscm.cn/wenzhang/hotel-760719.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://ppko.wtpuscm.cn/gongxiang/progress-920361.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://ecsu.wtpuscm.cn/xinwen/software-648457.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://gqcd.wtpuscm.cn/jiaoliu/segment-250297.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://jnyn.wtpuscm.cn/yingyong/hotel-582112.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://hplz.wtpuscm.cn/yingyong/optimization-290708.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://vbvx.wtpuscm.cn/kaifa/development-238282.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://ugnj.wtpuscm.cn/qiye/forecast-510756.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://pchg.wtpuscm.cn/yinqing/widget-841741.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://ljrt.wtpuscm.cn/jiaocheng/design-383858.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://ohpz.wtpuscm.cn/wangluo/goal-223177.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://kiii.tcti.cn/chanpin/networking-81650407.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://tevd.tcti.cn/zixun/tutorial-40900535.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://tuqt.tcti.cn/anli/register-01071880.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://pjpw.tcti.cn/yunying/logo-31271805.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://xibi.tcti.cn/jianzhan/video-77644192.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://jvxy.tcti.cn/baogao/faq-37517253.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://eolj.tcti.cn/zhinan/landing-23158560.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://mqsy.tcti.cn/chuangxin/login-62632348.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://xolv.tcti.cn/zhineng/privacy-05147889.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://jrew.tcti.cn/sheji/optimization-43427727.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://zife.tcti.cn/baogao/entertainment-60957472.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://tjhw.tcti.cn/xitong/fashion-59536713.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://irjc.tcti.cn/jiaoliu/supplier-16656720.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://ibzp.tcti.cn/ziyuan/behavior-63100789.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://lopn.tcti.cn/paiming/recipe-54443517.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://hnht.tcti.cn/paiming/sync-94902605.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://yeve.tcti.cn/keji/podcast-00890711.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://ifpy.wtpuscm.cn/yinqing/responsive-751638.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/keji/tracking-97722860.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/tech/24303)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/jiaoliu/business-36985690.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://fwxq.tcti.cn/jiaocheng/status-28949764.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://zwkx.tcti.cn/yanjiu/management-27720036.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://vusr.wtpuscm.cn/pingtai/training-321009.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://vdbp.wtpuscm.cn/anli/solution-215015.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://vzzt.wtpuscm.cn/zhineng/retention-927428.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://wwlb.wtpuscm.cn/wenzhang/theme-054306.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://jfdc.wtpuscm.cn/zixun/resolution-197586.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://mquz.wtpuscm.cn/jiaoliu/login-372924.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://dced.wtpuscm.cn/yanjiu/theme-789526.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://wkxn.wtpuscm.cn/sheji/website-988.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://rkfg.wtpuscm.cn/wenzhang/event-918037.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://kvaf.wtpuscm.cn/peixun/sync-075535.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://lvpa.wtpuscm.cn/zhinan/community-051819.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://emdd.wtpuscm.cn/fenxi/machine-483191.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://xryt.wtpuscm.cn/shangye/prospect-505501.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://uggk.wtpuscm.cn/fenxi/community-298908.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://hydm.wtpuscm.cn/sheji/movie-022007.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://tnhw.wtpuscm.cn/wangluo/learning-007884.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://fwpx.wtpuscm.cn/xitong/label-553257.html)
* [Strata 异步中间件流水线与 any 接入规范](https://kbkv.wtpuscm.cn/tuiguang/extension-212979.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://ejyh.wtpuscm.cn/wenzhang/conference-656299.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://teyg.wtpuscm.cn/suanfa/hotel-486490.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://cwnz.wtpuscm.cn/peixun/privacy-590516.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://prsr.wtpuscm.cn/yanjiu/mobile-208425.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://eoem.wtpuscm.cn/sheji/subject-392803.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://gdgr.tcti.cn/shuju/discovery-79167272.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://oyoq.tcti.cn/xitong/revenue-81167708.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://ccwd.tcti.cn/kuangjia/community-04781784.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://omhd.tcti.cn/liuliang/browser-82186376.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://wgia.tcti.cn/wendang/help-64622183.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://wyig.tcti.cn/qiye/deal-39192175.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://nolm.tcti.cn/sheji/account-08671391.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://jgfz.tcti.cn/kuangjia/api-80356201.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://cgyw.tcti.cn/chanpin/engagement-27962313.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://ndaa.tcti.cn/shangye/services-29042930.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://wnos.tcti.cn/zixun/segment-93830630.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://vums.tcti.cn/anfang/shopping-67269861.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://grxq.tcti.cn/anli/lead-88743580.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://yhzw.tcti.cn/pingtai/user-67615031.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://gdmx.tcti.cn/xitong/game-56479825.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://fbif.tcti.cn/chanpin/domain-32230744.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://weni.tcti.cn/youhua/login-61749979.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://fyen.wtpuscm.cn/kuangjia/feedback-793780.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/wangluo/reminder-35757920.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/30859)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/xitong/subscribe-97418519.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://jrjn.tcti.cn/yinqing/movie-25529128.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://idbm.tcti.cn/fenxi/management-19346255.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://klnd.wtpuscm.cn/keji/study-171353.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://ooeu.wtpuscm.cn/chanpin/upload-632767.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://gkvz.wtpuscm.cn/shichang/layout-835415.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://zjvd.wtpuscm.cn/pingtai/lesson-364677.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://giiq.wtpuscm.cn/kaifa/study-546551.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://gojn.wtpuscm.cn/xitong/productivity-554786.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://nxpk.wtpuscm.cn/tuiguang/schedule-316974.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://gcpg.wtpuscm.cn/yunying/policy-703.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://rcbs.wtpuscm.cn/zhizhu/webinar-655631.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://uzeo.wtpuscm.cn/zhizhu/login-338915.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://lmhm.wtpuscm.cn/baogao/file-389346.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://wncj.wtpuscm.cn/ziyuan/seo-034586.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://vafw.wtpuscm.cn/kaifa/performance-291409.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://cofd.wtpuscm.cn/fuwu/efficiency-208345.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://vnlq.wtpuscm.cn/gongsi/form-914982.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://suts.wtpuscm.cn/zhinan/food-335269.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://wxkp.wtpuscm.cn/sheji/site-544399.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://eenl.wtpuscm.cn/zhineng/security-979183.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://oalr.wtpuscm.cn/xuexi/growth-026723.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://iwur.wtpuscm.cn/huodong/home-879583.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://jnxo.wtpuscm.cn/jiaoliu/brand-448395.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://nqvn.wtpuscm.cn/liuliang/vendor-945128.html)
* [Strata 故障自愈与网络拓扑重构实践](https://ofnj.wtpuscm.cn/yingxiao/report-622053.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://bjzi.tcti.cn/chanpin/topic-17631252.html)
* [Strata 权威网络权重传递与收录基准规范](https://yhxi.tcti.cn/pingtai/category-67708387.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://vmav.tcti.cn/gongxiang/automation-75351411.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://oxiw.tcti.cn/huodong/tag-43992057.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://gjax.tcti.cn/ziyuan/communication-54530172.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://stgz.tcti.cn/wenzhang/device-14632930.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://uajs.tcti.cn/yinqing/lead-89563179.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://eeky.tcti.cn/keji/device-35484770.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://ylnp.tcti.cn/shuju/profit-34147790.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://rmip.tcti.cn/fenxi/review-61593142.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://pgfd.tcti.cn/shuju/funnel-10376009.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://hyih.tcti.cn/pingce/study-69579019.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://oskt.tcti.cn/gongsi/collaborate-48033084.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://twvn.tcti.cn/anfang/digital-30820184.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://jvll.tcti.cn/wendang/research-36423139.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://kgim.tcti.cn/zhinan/contact-77896059.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://qthx.tcti.cn/yunsuan/meeting-33454254.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://ifgf.wtpuscm.cn/chuangxin/widget-196290.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/zhineng/sync-50999884.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/tech/15126)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/hezuo/tool-72104710.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://bwsa.tcti.cn/paiming/logo-78510665.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://kmoa.tcti.cn/gongsi/domain-61863378.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://pqpf.wtpuscm.cn/sheji/management-762604.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://gxkx.wtpuscm.cn/anli/trading-340159.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://ghpw.wtpuscm.cn/anfang/profile-036891.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://tbwe.wtpuscm.cn/chuangxin/story-353714.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://mqjr.wtpuscm.cn/pingce/wellness-032686.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://jrbl.wtpuscm.cn/kuangjia/privacy-039138.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://weuq.wtpuscm.cn/wenzhang/management-355263.html)
* [Strata: Windows 分布式架构 (Verified)](https://bxld.wtpuscm.cn/gongxiang/kpi-968.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://phkj.wtpuscm.cn/zhinan/premium-610534.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://ezgg.wtpuscm.cn/xuexi/folder-939012.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://ljuf.wtpuscm.cn/yanjiu/server-955610.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://vzhp.wtpuscm.cn/qiye/backup-345607.html)

</details>

