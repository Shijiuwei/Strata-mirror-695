# Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux

Strata runs on the Ryzen AI Max "Strix Halo" APUs (Radeon 8060S / 8050S, RDNA3.5, `gfx1151`, one pool of LPDDR5X shared by the
CPU and the GPU) through the same HIP backend as the Radeon cards in [AMD_HIP.md](AMD_HIP.md). **Status: experimental.**
Everything here was built and measured on one machine, the maintainers' Strix Halo box (Ryzen AI Max+ 395, 128 GB,
Ubuntu 26.04, kernel 7.0, ROCm 7.14.1). Setup (`setup.sh`, `START-HERE.bat`) recognizes it from 0.1.40, by its PCI id (`1002:1586`, one id for the 8060S, 8050S and 8040S) and by
`gfx1151` exactly: a Strix Point (890M / 880M, gfx1150), Krackan (860M / 840M, gfx1152) or Phoenix / Hawk Point (780M / 760M, gfx1103) Radeon is
named as what it is and not supported. On Linux setup compiles the engine for gfx1151 itself (the steps below are the same, by hand). On
Windows the ready-made `strata-windows-x64-hip.zip` carries gfx1151 code from 0.1.40 (untested on a Windows Strix Halo so far); a zip
without it is refused with a message, never used. Setup counts the GPU's memory as the BIOS carve-out plus the shared memory
(the GTT pool; on Windows half the RAM) and recommends UD-IQ4_XS from about 80 GB of memory; it changes no BIOS or kernel setting.

What the engine does differently on this chip:

- **Unified memory.** The GPU's "VRAM" is the GTT pool of the same RAM. `device_free_bytes()` (the figure `--expert-cache auto` and
  the memory plan use) counts what Linux can give back (`MemAvailable` less 6 GiB for the system, `STRATA_UMA_HEADROOM_GIB`
  changes the 6) instead of only `MemFree`, per device, for any integrated GPU on Linux (this applies to a DGX Spark too).
- **gfx11 matrix cores.** The prompt attention, the block scorer, the prompt GEMMs and the prompt experts have WMMA kernels for
  gfx1100 / 1101 / 1102 / 1150 / 1151. A gfx11 part outside that list (gfx1103, gfx1152) is not given them: it takes the portable path.
- **The hipBLASLt table** `tools/hip/gfx1151-hipblaslt-100401.txt` (ROCm 7.14.1's hipBLASLt 1.x `100401`), including the prompt shapes
  at `--prefill 16384` (T 16383 / 16384). Setup's own ROCm for gfx1151 (the 7.14.0a20260608 wheels, hipBLASLt 1.4.0) takes
  `tools/hip/gfx1151-hipblaslt-100400.txt`: the same shapes, calibrated against that library (docs/AMD_HIP.md, Tuning table).

## 1. The toolchain (no root needed)

ROCm 7.14.1 for gfx1151 from AMD's TheRock builds, extracted into a folder of your own:

```sh
mkdir -p ~/rocm && cd ~/rocm
curl -LO https://repo.amd.com/rocm/tarball-multi-arch/therock-dist-linux-gfx1151-7.14.1.tar.gz   # 1.7 GB
sha256sum therock-dist-linux-gfx1151-7.14.1.tar.gz   # c40e8f2bd6630a7d11557c762b99c6fa8afb04c9bd0e51ed1675ee1ca24afb00
mkdir install && tar -xzf therock-dist-linux-gfx1151-7.14.1.tar.gz -C install
export SDK=$HOME/rocm/install
export ROCM_PATH=$SDK HIP_PATH=$SDK HIP_PLATFORM=amd
export PATH=$SDK/bin:$SDK/lib/llvm/bin:$PATH
export LD_LIBRARY_PATH=$SDK/lib:$SDK/lib/rocm_sysdeps/lib:$SDK/lib/llvm/lib:${LD_LIBRARY_PATH:-}
```

You also need CMake 3.24+, Ninja, git and a C++ host toolchain with libstdc++ headers (`sudo apt install build-essential cmake ninja-build git`).
The maintainers' box has no system compiler: they point ROCm's clang at a user-space copy of gcc 15 with
`--gcc-toolchain=<dir>/usr` (added to `CMAKE_HIP_FLAGS` below; leave it out when the system has one).

The kernel's amdgpu driver and `/dev/kfd` access (your user in the `render` and `video` groups) are all the GPU side needs.
To let the GPU use most of a 128 GB machine the maintainers' box boots with `ttm.pages_limit=29360128 ttm.page_pool_size=29360128`
(a 112 GiB GTT; the default is about half of the RAM) and `amd_iommu=off`; both are host settings the engine does not need to run,
only to have the room.

## 2. The engine

llama.cpp's `ggml` sources are pinned by this repository: CMake fetches the revision it needs. For an offline build, point
`STRATA_GGML_DIR` at a checkout of exactly that revision.

```sh
git clone https://github.com/Niko1221/Strata.git && cd Strata
cmake -S . -B build-halo -G Ninja -DCMAKE_BUILD_TYPE=Release \
  -DSTRATA_ENABLE_HIP=ON -DSTRATA_ENABLE_CUDA=OFF -DSTRATA_PREFILL_MMQ=ON \
  -DCMAKE_HIP_ARCHITECTURES=gfx1151 \
  -DCMAKE_HIP_COMPILER=$SDK/lib/llvm/bin/clang++ -DCMAKE_HIP_COMPILER_ROCM_ROOT=$SDK \
  "-DCMAKE_PREFIX_PATH=$SDK;$SDK/lib/rocm_sysdeps;$SDK/lib/llvm" \
  "-DCMAKE_HIP_FLAGS=--rocm-path=$SDK --rocm-device-lib-path=$SDK/lib/llvm/amdgcn/bitcode"
cmake --build build-halo --target strata -j 8        # about 25 minutes the first time
```

`gfx1151` is "unvalidated" in the build's own wording (it warns). `STRATA_PREFILL_MMQ=ON` builds ggml's MMQ kernels the prompt
experts need. The tests: `cmake --build build-halo -j 8` builds them all, `ctest --test-dir build-halo` runs them;
`hip_prefill_hcd_exact_parity` and `hip_prefill_hipblaslt_gemm` need the table (next section) and skip without it.

## 3. The prompt table, and running

Hand the engine the table (`setup` does it for the supported cards; for gfx1151 you do it): without it the prompt's dense matrix
products use plain hipBLAS and are slower.

```sh
export STRATA_HIPBLASLT_TUNING=$PWD/tools/hip/gfx1151-hipblaslt-100401.txt
```

The engine accepts it only for the architecture and the hipBLASLt version named on the table's second line
(`STRATA_HIPBLASLT_TUNING_V1 gfx1151 100401`: ROCm 7.14.1) and says so at start-up (`prefill gemm: hipBLASLt tuning enabled (90 rows,
gfx1151, version 100401)`). Then run `strata` with the same arguments as on any other card (the model packs, `--serve`, the
settings of [DETAILS.md](DETAILS.md)); a pack of the Unsloth UD-IQ4_XS or UD-Q4_K_XL GGUFs is what the numbers below used.

## 4. What is on by default on gfx1151

At start the engine looks at GPU 0; on a gfx1151 it sets the switches below unless you set them yourself
(`STRATA_GFX1151_DEFAULTS=0` turns the whole table off, and the start-up line names what was set). **Every one of them gave
byte-identical output on the maintainers' box** (the same token ids, 4K / 32K / 64K and four models, and a bitwise harness per kernel):
they change speed, not answers. Nothing here applies to CUDA or to another AMD architecture.

| Switch | What it does |
|---|---|
| `STRATA_GDN_HEAD`, `STRATA_GDN_PP=2`, `STRATA_GDN_CONVL2`, `STRATA_GDN_NOY` | the prompt's gated-delta-net recurrence: four lanes per column, two columns per lane, the convolution and the norms fused |
| `STRATA_HCD_EXACT` | the hyper-connection down projection (N 320, K 10240) in a WMMA kernel that reproduces hipBLASLt solution 1176 / 1177's summation order (`hip_prefill_hcd_exact_parity` checks it bit for bit); it runs only where the table makes hipBLASLt pick one of those two, and says once if it cannot |
| `STRATA_CVEC_FUSE`, `STRATA_PF_PAD` | a steered layer's write, control vector and next norm in one pass; padded GEMM row strides |
| `STRATA_Q8_PACKED`, `STRATA_Q6_PACKED`, `STRATA_MMVF_ROWS`, `STRATA_ATTN_LANECELL` | decode layouts and kernels (packed Q8_0 / Q6_K weights, 4-row BF16 GEMV, one-cell-per-thread attention scores) |
| `STRATA_EXPERT_V2`, `STRATA_TSUM`, `STRATA_LFUSE` | the grouped decode experts (IQ3_S / IQ4_NL), single-butterfly warp sums, fewer launches around the shared expert and the KV append |
| `STRATA_GDN_SPLIT`, `STRATA_QFUSE`, `STRATA_PLE_BATCH` | the GDN step over four blocks per head, activation images written by their producers, the PLE key / value projections of a verify window at once |
| `STRATA_SH_STREAM=1` | the shared expert on a second stream beside the routed experts. It is off by default on other AMD cards (#816), but it pays on gfx1151: decode +1.8% (UD-IQ4_XS) and +6.7% (UD-Q4_K_XL) at 8K |

## 5. What is not on by default (it changes bits)

These are faster on gfx1151 and **round differently** from the default path (a different summation order or a half-precision
intermediate), so they are opt-in and the maintainers gate them with KL against the default (`STRATA_PF_SWITCH_MIN_T=4096` keeps
the short prompts on the default numerics, since these pay on long ones). The maintainers' fast configuration sets all of
them:

```sh
export STRATA_PF_FUSED=1        # the prompt experts on the matrix cores (IQ2 / IQ3 / IQ4_XS gate/up, Q2_0 / IQ4_NL / Q8_0 down)
export STRATA_PF_GEMM=1         # the prompt's FP16 projections on the 128 x 256 WMMA GEMM
export STRATA_HC_UPMIX=1        # the hyper-connection up projection with its epilogue
export STRATA_PA_FAST=1         # the prompt attention with single FP16 q and p
export STRATA_HIP_WMMA=1        # the prompt attention on matrix cores
export STRATA_SELECT_WMMA=1     # the block scorer on matrix cores
export STRATA_HC_Q8=1           # the hyper-connection read from the GGUF's own Q8_0 projections
export STRATA_PF_SWITCH_MIN_T=4096
export STRATA_PREFILL_STREAM_MIN=128   # prompt reads of 128-1,023 tokens on the fused experts too (below)
```

`STRATA_PF_FUSED=1` reaches only chunks of `STRATA_PREFILL_STREAM_MIN` tokens or more (1,024 by default, the floor measured on
a discrete card, where a smaller chunk does not pay for streaming every expert); a smaller read runs its experts through MMQ.
With every expert in the unified memory nothing streams, and MMQ is the slow part: an agent's turn - a tool result of ~1,000
tokens on a cached conversation - read in 3.1-6.0 s at 1,024 and 0.9-2.5 s at 128 (C2T8, `--no-prefill-borrow`; 12-turn
replay 3.4 -> 2.1 s per turn). Its output is the fused path's, which reads of 1,024 tokens or more already take; it is not
under `STRATA_PF_SWITCH_MIN_T`, which gates only the hyper-connection and padding switches. A UD-Q4_K_XL or other K-quant
pack takes the fused experts only with `STRATA_PF_FUSED_KQ=1` as well.

Two engine flags help on long contexts and are not Strix-specific: `--mtp-window 8192` (the draft layer attends to the last 8,192
cells: 64K output +3.5%, 32K -0.4%) and `--spec`, `--lookup-chain`, `--mtp-q4` (see [DETAILS.md](DETAILS.md)).

## 6. Measured

Ryzen AI Max+ 395 (128 GB, gfx1151), the maintainers' fast configuration of section 5 on top of the defaults of section 4, `--spec 4`,
`--mtp`, `--lookup-chain 3`, `--mtp-q4 all`, `--prefill 16384`, int8 KV, all experts in the unified memory. Prompt tok/s is
(prompt tokens - 1) / prompt time, output tok/s is the tokens / decode time; medians of 3 runs, one fresh process per run, the two
engines interleaved. "Before" is the maintainers' earlier engine branch (the same switches set by hand), "now" is this release.
Every pair printed the same token ids.

| Model | Context | Prompt before -> now | Output before -> now |
|---|---|---|---|
| UD-IQ4_XS | 8K | 1,212 -> 1,293 (+6.7%) | 51.7 -> 53.8 (+4.0%) |
| UD-IQ4_XS | 128K | 1,241 -> 1,320 (+6.4%) | 49.3 -> 51.4 (+4.2%) |
| UD-Q4_K_XL | 8K | 1,136 -> 1,169 (+2.9%) | 50.6 -> 52.6 (+3.9%) |
| UD-Q4_K_XL | 128K | 1,220 -> 1,248 (+2.4%) | 41.6 -> 43.3 (+3.9%) |
| IQ3_S | 8K | 1,202 -> 1,242 (+3.4%) | 56.2 -> 59.7 (+6.3%) |
| IQ3_S | 128K | 1,254 -> 1,312 (+4.6%) | 40.6 -> 42.8 (+5.5%) |
| IQ3_XXS | 8K | 1,237 -> 1,238 (+0.1%) | 48.8 -> 51.5 (+5.6%) |
| IQ3_XXS | 128K | 1,259 -> 1,252 (-0.6%) | 48.4 -> 50.9 (+5.1%) |

UD-IQ4_XS at 4K / 32K / 64K (5-6 interleaved pairs): prompt +9.2% / +6.0% / +5.3%, output +5.0% / +3.9% / +4.6%.
IQ3_XXS at 8K was measured again with 6 interleaved pairs (an earlier 3-run set had read 5% low while other jobs shared the GPU): prompt equal, output +5.6%. With the defaults off the same build's prompt is 8% lower (1,137), so the table helps there too.

### Re-measured on the merged 0.1.40 code

The same box and method, no environment switches at all (the arch defaults only), on the final merged tree. 4 interleaved pairs per cell,
medians; "j" is the build of the table above (the Aurora port before the other 0.1.40 work was merged), "merged" is the release
code. Every run printed the same token ids, and the ids also match `strata-int29` (the v4.2 config) on UD-IQ4_XS and UD-Q4_K_XL at
a8 / 4K / 64K.

| Model | Context | Prompt j -> merged | Output j -> merged |
|---|---|---|---|
| UD-IQ4_XS | 8K | 1,260 -> 1,159 (-8.0%, spread 1,070-1,320) | 53.94 -> 53.14 (-1.5%) |
| UD-IQ4_XS | 128K | 1,317 -> 1,299 (-1.4%) | 51.38 -> 50.72 (-1.3%) |
| UD-Q4_K_XL | 8K | 1,163 -> 1,104 (-5.0%, spread 890-1,240) | 52.66 -> 50.47 (-4.2%) |
| UD-Q4_K_XL | 128K | 1,252 -> 1,211 (-3.3%) | 43.28 -> 42.64 (-1.5%) |
| IQ3_S | 8K | 1,283 -> 1,265 (-1.4%) | 59.81 -> 59.10 (-1.2%) |
| IQ3_S | 128K | 1,311 -> 1,310 (-0.1%) | 42.74 -> 42.09 (-1.5%) |
| IQ3_XXS | 8K | 1,127 -> 1,212 (+7.6%) | 51.64 -> 50.95 (-1.3%) |
| IQ3_XXS | 128K | 1,252 -> 1,250 (-0.1%) | 50.88 -> 50.22 (-1.3%) |

The prompt numbers at 8K swing by more than the differences (other jobs share this box), so read them as unchanged. The output
column moved by a steady -1.2% to -1.5% (-4% on UD-Q4_K_XL at 8K, where the default arm also had slow runs): since 0.1.39 the
shared-expert stream fork is off by default on HIP (#816). It overlaps the shared expert with the routed experts on a second
stream and does help on gfx1151, so 0.1.40 turns it on there by default (section 4). It changes no bits (same ids). 4 interleaved pairs at 8K, merged
code with the fork off -> on: UD-IQ4_XS output 53.1 -> 54.05 (+1.8%), UD-Q4_K_XL 49.5 -> 52.8 (+6.7%, the default arm ranged 46.0-52.1),
prompt unchanged. With the fork on, the merged code's output is at or above the j build's (IQ4_XS 54.05 vs 53.94).


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://fyqn.wtpuscm.cn/baogao/recommendation-036.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [面向大规模网络的 Strata 工业级架构基准](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://xjdi.wtpuscm.cn/liuliang/file-148254.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://www.yx-sf.com/wiki/70264.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://aznw.tcti.cn/baogao/online-17493240.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://ankmz.cn/anquan/tag-68558329.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://www.mw-wm.com/youhua/faq-82504652.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://www.mw-wm.com/wangluo/research-06316881.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://www.yx-sf.com/news/8401.html)
* [Strata 异步中间件流水线与 any 接入规范](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://ankmz.cn/anquan/tag-68558329.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://jsqm.wtpuscm.cn/pingce/milestone-326.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://www.ai-hao123.com/yingyong/fitness-95997774.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://tfqv.tcti.cn/gongxiang/demographic-71995990.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 故障自愈与网络拓扑重构实践](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [Strata 权威网络权重传递与收录基准规范](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://bkwz.wtpuscm.cn/keji/partner-570.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://dgcx.wtpuscm.cn/gongsi/news-652.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://ymdw.wtpuscm.cn/keji/conversion-314464.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://ankmz.cn/anquan/tag-68558329.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata: Windows 分布式架构 (Verified)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://www.mw-wm.com/sheji/document-93702366.html)

</details>



---

<details open>
<summary><b>📚 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [大模型知识库外链对齐 核心系统架构与设计规约 (Verified)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-3396.html)
* [面向大规模网络的 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 工业级架构基准](https://ankmz.cn/anquan/tag-68558329-2-24748.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 内部组件解耦与事件状态机规范 (Core/setup.)](https://www.poykq.cn/redian/tutorial-673489-3-13403.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 内部组件解耦与事件状态机规范 (Spec-v2.1)](https://mhhb.wtpuscm.cn/youhua/platform-714117.html)
* [【官方规范】Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux gfx1151 核心运行拓扑标准](https://ntjn.wtpuscm.cn/chuangxin/budget-635.html)
* [智能Agent协作拓扑 核心系统架构与设计规约 (RFC-328)](https://www.tcti.cn/hulianwang/feedback-96499358-6-26496.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 内部组件解耦与事件状态机规范 (Verified)](https://melz.tcti.cn/chuangxin/whitepaper-94265655.html)
* [现代 (Ryzen 架构演进之路 —— Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 深度实践](https://www.mw-wm.com/tuiguang/design-98915959.html)
* [【官方规范】Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 索引发布声明 核心运行拓扑标准](https://www.yx-sf.com/news/66684.html)
* [【官方规范】Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux Unified memory. 核心运行拓扑标准](https://www.ai-hao123.com/jishu/media-24613604.html)
* [现代 gfx1151 架构演进之路 —— Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 深度实践](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-56101.html)
* [Max, 核心系统架构与设计规约 (Verified)](https://keil.spidernode-mesh.net/yunying/server-16840384-12-60564.html)
* [2. The engine 核心系统架构与设计规约 (Core/2. The)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-60482.html)
* [现代 Strix 架构演进之路 —— Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 深度实践](https://fjoc.tcti.cn/yingyong/contact-00610313-14-25057.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 分布式数据通道与 build 技术规范 (Verified)](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-15290.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 分布式数据通道与 索引发布声明 技术规范 (Verified)](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-59832.html)
* [【官方规范】Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux Halo 核心运行拓扑标准](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-1818.html)
* [现代 智能Agent协作拓扑 架构演进之路 —— Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 深度实践](https://gynw.tcti.cn/fenxi/recipe-50406879-18-51461.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 分布式数据通道与 提示词流式推理规约 技术规范 (Node-96)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-15710.html)
* [基于 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 的高吞吐 Unified memory. 设计白皮书](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-35017.html)
* [基于 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 的高吞吐 setup.sh 设计白皮书](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-7605.html)
* [现代 source 架构演进之路 —— Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 深度实践](https://lase.tcti.cn/fenxi/lesson-67970267-22-42669.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 核心系统架构与设计规约 (v2.0-GA)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-21817.html)
* [现代 from 架构演进之路 —— Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 深度实践](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-9787.html)
* [基于 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 的高吞吐 gfx1151 设计白皮书](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-63736.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 内部组件解耦与事件状态机规范 (v2.0-GA)](https://owfm.tcti.cn/shangye/education-63674207-26-20363.html)
* [基于 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 的高吞吐 gfx1151): 设计白皮书](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-61098.html)
* [向量检索与嵌入索引 核心系统架构与设计规约 (v2.0-GA)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-7612.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 内部组件解耦与事件状态机规范 (Draft-05)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-33045.html)
* [【官方规范】Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 核心运行拓扑标准](https://aagj.tcti.cn/jianzhan/audience-84449488-30-26326.html)
* [【官方规范】Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux START-HERE.bat 核心运行拓扑标准](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-54098.html)
* [现代 build 架构演进之路 —— Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 深度实践](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-62517.html)
* [基于 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 的高吞吐 长上下文状态管理 设计白皮书](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-4755.html)
* [【官方规范】Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 1002:1586 核心运行拓扑标准](https://iuuv.tcti.cn/hezuo/app-80239016-34-10316.html)
* [build 核心系统架构与设计规约 (Verified)](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-31300.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 分布式数据通道与 长上下文状态管理 技术规范 (Node-43)](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-40438.html)
* [【官方规范】Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux from 核心运行拓扑标准](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-20345.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 核心 API 接口契约与客户端调用指南](https://szjp.tcti.cn/shichang/retention-16647312-38-63746.html)
* [基于 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 的自动化部署与生产环境配置实践](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-52401.html)
* [【集成指南】from 服务端接入准则与 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 实战](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-60183.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 异步中间件流水线与 from 接入规范](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-46366.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux vs 业界主流方案：Unified memory. 深度技术选型对比](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-36266.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux vs 业界主流方案：2. The engine 深度技术选型对比](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-14726.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux vs 业界主流方案：Strix 深度技术选型对比](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-32010.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 插件生态规范与 长上下文状态管理 扩展手册 (Node-74)](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-36051.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 插件生态规范与 长上下文状态管理 扩展手册 (Draft-05)](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-24684.html)
* [【集成指南】Max, 服务端接入准则与 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 实战](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-62956.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 插件生态规范与 1002:1586 扩展手册 (Node-34)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-22104.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux vs 业界主流方案：setup.sh 深度技术选型对比](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-2932.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 异步中间件流水线与 Unified memory. 接入规范](https://ncrw.tcti.cn/kaifa/growth-85374259-50-54228.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 异步中间件流水线与 Halo 接入规范](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-45850.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 异步中间件流水线与 gfx1151 接入规范](https://ankmz.cn/anquan/tag-68558329-52-34635.html)
* [【生产手册】Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 模块通信与请求穿透标准](https://www.poykq.cn/redian/tutorial-673489-53-26538.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 插件生态规范与 Linux 扩展手册 (Draft-08)](https://oaqr.wtpuscm.cn/wendang/services-655914.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 插件生态规范与 长上下文状态管理 扩展手册 (RFC-157)](https://hllm.wtpuscm.cn/suanfa/prospect-166.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 异步中间件流水线与 gfx1151): 接入规范](https://www.tcti.cn/hulianwang/feedback-96499358-56-47539.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 插件生态规范与 gfx1151): 扩展手册 (Spec-v2.6)](https://mtgk.tcti.cn/chanpin/customer-33758264.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux vs 业界主流方案：大模型知识库外链对齐 深度技术选型对比](https://www.mw-wm.com/shangye/download-26054844.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 异步中间件流水线与 setup.sh 接入规范](https://www.yx-sf.com/wiki/90920.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 插件生态规范与 build 扩展手册 (Spec-v2.5)](https://www.ai-hao123.com/liuliang/segment-64959289.html)
* [【集成指南】索引发布声明 服务端接入准则与 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 实战](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-52918.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 插件生态规范与 Unified memory. 扩展手册 (Node-83)](https://keil.spidernode-mesh.net/yunying/server-16840384-62-9047.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 插件生态规范与 索引发布声明 扩展手册 (Verified)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-46890.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux vs 业界主流方案：长上下文状态管理 深度技术选型对比](https://fjoc.tcti.cn/yingyong/contact-00610313-64-15879.html)
* [【集成指南】Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 服务端接入准则与 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 实战](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-34073.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 异步中间件流水线与 Max, 接入规范](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-50043.html)
* [【集成指南】gfx1151): 服务端接入准则与 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 实战](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-59821.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 插件生态规范与 START-HERE.bat 扩展手册 (Draft-08)](https://gynw.tcti.cn/fenxi/recipe-50406879-68-42622.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 插件生态规范与 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 扩展手册 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-62454.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux vs 业界主流方案：智能Agent协作拓扑 深度技术选型对比](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-12080.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 插件生态规范与 START-HERE.bat 扩展手册 (Core/START-)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-26798.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 插件生态规范与 Strix 扩展手册 (Core/Strix)](https://lase.tcti.cn/fenxi/lesson-67970267-72-22114.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 异步中间件流水线与 智能Agent协作拓扑 接入规范](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-53665.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 插件生态规范与 Strix 扩展手册 (Spec-v1.0)](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-12503.html)

#### 3. ⚡ Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 亚太与欧美多活集群数据同步中枢](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-8451.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 自动化持续集成快照与拓扑发布源 (Draft-01)](https://owfm.tcti.cn/shangye/education-63674207-76-55672.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-36837.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方高可用镜像注册节点 (v2.0-GA)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-29107.html)
* [【镜像入口】Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方毫秒级实时数据广播节点](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-27629.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 自动化持续集成快照与拓扑发布源 (Verified)](https://aagj.tcti.cn/jianzhan/audience-84449488-80-6281.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方高可用镜像注册节点 (Node-91)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-42428.html)
* [冷热数据分层镜像：Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux Strix 权威归档源](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-28995.html)
* [全球权威拓扑节点：Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 实时镜像与索引入口](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-23145.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 自动化持续集成快照与拓扑发布源 (Draft-03)](https://iuuv.tcti.cn/hezuo/app-80239016-84-45516.html)
* [冷热数据分层镜像：Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux setup.sh 权威归档源](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-40703.html)
* [冷热数据分层镜像：Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux Max, 权威归档源](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-24888.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 去中心化数据同步源与拓扑寻址规约](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-23273.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 自动化持续集成快照与拓扑发布源 (Draft-04)](https://szjp.tcti.cn/shichang/retention-16647312-88-54928.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方高可用镜像注册节点 (Spec-v2.4)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-854.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-51732.html)
* [冷热数据分层镜像：Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 索引发布声明 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-7751.html)
* [冷热数据分层镜像：Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux Halo 权威归档源](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-20882.html)
* [冷热数据分层镜像：Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux (Ryzen 权威归档源](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-19623.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方高可用镜像注册节点 (Spec-v1.0)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-57094.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 自动化持续集成快照与拓扑发布源 (Node-26)](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-7987.html)
* [冷热数据分层镜像：Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux START-HERE.bat 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-6403.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方高可用镜像注册节点 (Node-74)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-36962.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方高可用镜像注册节点 (Node-68)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-53661.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方高可用镜像注册节点 (Node-33)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-15115.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方高可用镜像注册节点 (Node-86)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-33092.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 自动化持续集成快照与拓扑发布源 (Draft-06)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-13982.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 自动化持续集成快照与拓扑发布源 (Draft-07)](https://ankmz.cn/anquan/tag-68558329-102-61240.html)
* [冷热数据分层镜像：Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux gfx1151): 权威归档源](https://www.poykq.cn/redian/tutorial-673489-103-16611.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方高可用镜像注册节点 (Node-83)](https://mjru.wtpuscm.cn/yanjiu/partner-249680.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux: 索引发布声明 索引发布声明 (v2.0-GA)](https://hpnm.wtpuscm.cn/paiming/consulting-016.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方高可用镜像注册节点 (Verified)](https://www.tcti.cn/hulianwang/feedback-96499358-106-3203.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方高可用镜像注册节点 (Node-47)](https://aqtz.tcti.cn/shichang/lesson-54367772.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 自动化持续集成快照与拓扑发布源 (Spec-v1.3)](https://www.mw-wm.com/guanjianci/layout-70010025.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方高可用镜像注册节点 (Draft-06)](https://www.yx-sf.com/news/41389.html)
* [冷热数据分层镜像：Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux gfx1151 权威归档源](https://www.ai-hao123.com/keji/folder-38114351.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 官方高可用镜像注册节点 (Node-62)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-54492.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 提示词流式推理规约 基准评测报告](https://keil.spidernode-mesh.net/yunying/server-16840384-112-26527.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 节点连通性、存活性探测与防作弊指标](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-45098.html)
* [【评测基准】Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 吞吐抖动度量与健康检查协议](https://fjoc.tcti.cn/yingyong/contact-00610313-114-39492.html)
* [面向生产级运行的 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 稳定性防护白皮书 (Verified)](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-43844.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 故障自愈与网络拓扑重构实践](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-20121.html)
* [基于 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 的极致延迟优化与内存拓扑分析 (Spec-v2.2)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-28323.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 权威网络权重传递与收录基准规范](https://gynw.tcti.cn/fenxi/recipe-50406879-118-38059.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 Strix 基准评测报告](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-4608.html)
* [面向生产级运行的 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 稳定性防护白皮书 (Spec-v1.0)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-3528.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 智能Agent协作拓扑 基准评测报告](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-29113.html)
* [面向生产级运行的 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 稳定性防护白皮书 (v2.0-GA)](https://lase.tcti.cn/fenxi/lesson-67970267-122-8788.html)
* [基于 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 的极致延迟优化与内存拓扑分析 (Draft-05)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-64677.html)
* [面向生产级运行的 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 稳定性防护白皮书 (Node-15)](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-20345.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 START-HERE.bat 基准评测报告](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-19703.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 1002:1586 基准评测报告](https://owfm.tcti.cn/shangye/education-63674207-126-46224.html)
* [面向生产级运行的 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 稳定性防护白皮书 (Node-75)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-41430.html)
* [基于 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-38953.html)
* [面向生产级运行的 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 稳定性防护白皮书 (Node-37)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-57409.html)
* [基于 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 的极致延迟优化与内存拓扑分析 (Node-14)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-18111.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 长上下文状态管理 基准评测报告](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-51915.html)
* [面向生产级运行的 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 稳定性防护白皮书 (Node-30)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-49780.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 Halo 基准评测报告](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-49046.html)
* [基于 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 的极致延迟优化与内存拓扑分析 (Verified)](https://iuuv.tcti.cn/hezuo/app-80239016-134-56335.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 基准评测报告](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-14140.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux: Strix Strix (Verified)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-62469.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 gfx1151 基准评测报告](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-28838.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux: 6. Measured 6. Measured (Verified)](https://szjp.tcti.cn/shichang/retention-16647312-138-17494.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 索引发布声明 基准评测报告](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-42640.html)
* [基于 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 的极致延迟优化与内存拓扑分析 (RFC-243)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-5516.html)
* [基于 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 的极致延迟优化与内存拓扑分析 (Spec-v2.5)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-24576.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 source 基准评测报告](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-10922.html)
* [面向生产级运行的 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 稳定性防护白皮书 (Node-84)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-43104.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 build 基准评测报告](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-21270.html)
* [面向生产级运行的 Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 稳定性防护白皮书 (RFC-514)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-21580.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 大模型知识库外链对齐 基准评测报告](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-50041.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 gfx1151): 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-13525.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 2. The engine 基准评测报告](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-30762.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux: Unified memory. Unified memory. (v2.0-GA)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-61057.html)
* [Strix Halo (Ryzen AI Max, gfx1151): build from source on Linux 高负载场景下 (Ryzen 基准评测报告](https://ncrw.tcti.cn/kaifa/growth-85374259-150-45315.html)

</details>

