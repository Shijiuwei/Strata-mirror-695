# AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030)

Strata runs on AMD Radeon cards through its HIP backend, the same engine as on NVIDIA compiled for AMD. This page
covers the build on Linux (on Windows a ready-made engine, see [Windows](#windows)) for the RX 7900 XT / XTX (RDNA3, gfx1100) and the
RX 9070 / 9070 XT / Radeon AI PRO R9700 (RDNA4, gfx1201; see [RDNA4](#rdna4-gfx1201)). The RX 7800 XT / 7700 XT
(gfx1101) and the RX 9060 XT (gfx1200) were validated by their owners (see [Community-validated
cards](#community-validated-cards)); the RX 6800 / 6900 series (RDNA2, gfx1030) builds and runs too, reported by a community machine and not yet validated by the maintainers (see [RDNA2](#rdna2-gfx1030)). Setup chooses it by itself on a PC with no NVIDIA card Strata can use (`--backend hip` on a PC with both); the
install steps for users are in [INSTALL.md](INSTALL.md#amd-cards). gfx906 (Instinct MI50 / MI60, Radeon VII; wave64) has a separate
opt-in build, see [gfx906](#gfx906-instinct-mi50--mi60-radeon-vii-wave64-built-from-source). The Ryzen AI Max "Strix Halo" APU (gfx1151, RDNA3.5, unified memory) is built from source on Linux: see [STRIX_HALO.md](STRIX_HALO.md). Other AMD architectures and mixed
AMD/NVIDIA execution in one run are not supported.

The backend maps the CUDA-shaped runtime and BLAS calls to HIP/hipBLAS, uses
RDNA2/RDNA3/RDNA4's signed integer dot instruction for quantized kernels, and supplies
wave32 shuffle/packed-byte operations. CUDA-only QSA matrix instructions have
an ordered FP32 fallback. Prefill supports both dequantization plus hipBLAS GEMM and opt-in HIP ggml MMQ.
An optional, calibrated hipBLASLt path accelerates dense projections (per-architecture tables in `tools/hip`).
This does not claim bit-identical model answers across backends. See
[performance settings and evidence](AMD_HIP_PERFORMANCE.md).

## Install with setup (recommended)

On Linux with an RX 7900 XT / XTX, RX 7800 XT / 7700 XT, RX 9060 XT, RX 9070 / 9070 XT or Radeon AI PRO R9700 and
the kernel's amdgpu driver (no ROCm install needed):

```sh
./setup.sh --backend hip
```

- **Detection:** setup finds the card through the kernel's KFD topology. Integrated Radeon GPUs are listed as not
  supported. On a PC without an NVIDIA card Strata can use, `--backend hip` is chosen automatically.
- **ROCm:** a system ROCm 7 in `/opt/rocm` (or `$ROCM_PATH`) with hipcc and hipBLAS is used when present. Otherwise
  (or when it is older than 7.0) ROCm is installed into `.venv` from AMD's TheRock wheels (~10 GB, no sudo), from the
  card family's index: `gfx110X-dgpu` for gfx1100 / gfx1101, `gfx120X-all` for gfx1200 / gfx1201, `gfx103X-all` for
  gfx1030, `gfx1151` for Strix Halo. The version is chosen per family: 7.10.0a20251120 where that is on the index,
  7.13.0a20260515 for `gfx103X-all` (its index starts at 7.13, #1103) and 7.14.0a20260608 for `gfx1151` (the 7.10 wheel
  segfaults on kernel 7.2.8, #1267). When the index no longer offers the version, setup warns and takes the newest of
  the same 7.x line, else of the same major. `STRATA_ROCM_VERSION` / `STRATA_ROCM_INDEX` override them; the gfx1030
  wheels are unvalidated (a system ROCm 7 is the tested path there).
- **Engine:** compiled on your PC for the card's architecture (10-20 minutes, once; again after a `git pull` that
  changes it, or when you pick a card of another architecture). This needs a C++ compiler and git
  (`sudo apt install build-essential git`).
- **hipBLASLt tuning table:** setup uses `tools/hip/<arch>-hipblaslt-<version>.txt` only when it matches both the
  card's architecture and the installed hipBLASLt version (read from `hipblaslt-version.h`; 1.2.0 is `100200`).
  Otherwise it says so and the prompt's dense matrix products use plain hipBLAS (slower prompts, same answers).
  A table's solution ids are valid only for that pair, and the engine refuses any other table.
- **Several cards:** setup takes one card (the one with the most VRAM, or `--gpu N`) unless you name more:
  `./setup.sh --backend hip --gpus 1,0` splits the model's layers across them, the first one the main card (numbers
  as setup lists them; `--gpus all` = every supported card, the most VRAM first). Every chosen card must be one of the
  architectures above; the engine is compiled for each of them (cards of two families, e.g. gfx1100 + gfx1201, need
  a system ROCm 7: AMD's wheels hold one family). A split pays only when no single card holds the model's experts
  (see RDNA4 below).
- **Limits for now:** images only through the CPU encoder (`--vision cpu`, 0.1.32). Setup does not offer the tuning
  (calibration) on AMD yet: its controls are being checked on HIP one at a time (#566). Since 0.1.39 a tuning run by
  hand (`./setup.sh --calibrate`) is saved for the AMD card it ran on and reused when setup runs again. The Monitor
  shows the card's load, VRAM, temperature and power from Linux sysfs (0.1.32).

The rest of setup is the same as on NVIDIA: the model download, the start script, the server.

## Windows

Since 0.1.34 an AMD card on Windows is set up like an NVIDIA one: download Strata, double-click `START-HERE.bat`.
On a PC with no NVIDIA card Strata can use, the AMD card is chosen by itself; with both, setup asks
(`START-HERE.bat --backend hip` picks AMD directly).

- **You need:** Windows 10 or 11 (64-bit), one of the cards above, and a current AMD driver ([AMD Software:
  Adrenalin Edition](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)). Nothing else: no ROCm or HIP SDK
  install, no compiler, no admin rights.
- **Detection:** setup reads the display adapters Windows lists (their PCI ids; the VRAM size from the display
  driver's registry entry). An integrated Radeon is listed as not supported, except the Ryzen AI Max "Strix Halo" (Radeon 8060S / 8050S / 8040S, gfx1151: [STRIX_HALO.md](STRIX_HALO.md)); a Strix Point (890M / 880M, gfx1150), Krackan (860M / 840M, gfx1152) or Phoenix / Hawk Point (780M / 760M, gfx1103) Radeon is named as what it is and not supported.
- **Engine:** the ready-made `strata-windows-x64-hip.zip` from the release (built by `tools\hip\build_windows.bat`
  for gfx1100, gfx1101, gfx1102, gfx1200, gfx1201, gfx1030 and, from 0.1.40, gfx1151 (Strix Halo, unvalidated on Windows)) goes into `engine\`. It carries the ROCm libraries the
  engine loads (`engine\rocm\bin`: the HIP runtime, hipBLAS / rocBLAS / hipBLASLt with their kernels for these cards,
  amd_comgr and the Microsoft C++ runtime; ROCm 10.2.0a20260930 from AMD's TheRock builds, licenses in
  `engine\rocm\licenses`). The HIP runtime works through the AMD driver's own components, so the driver is the one
  thing it needs from the PC.
- **The HIP runtime next to `strata.exe` (0.1.35, #468 #461):** `amdhip64_7.dll` and `amd_comgr.dll` are also put in
  `engine\` (setup copies them there on every start). Windows looks in the program's folder before System32, where
  some AMD drivers install their own `amdhip64_7.dll`; with that one, the bundled libraries crashed on the first
  prompt (an access violation, or `hipErrorInvalidDeviceFunction`). The engine's log names the runtime it loaded
  (`strata generate: HIP runtime ...`).
- **Before the ~60 GB model download** setup runs `engine\strata-device.exe --list-devices` (with `engine\rocm\bin` on
  the PATH): if the HIP runtime does not see the card, setup stops there and points to the driver. It also gives the
  card's HIP number: with an integrated Radeon that is device 1, not 0 (#325). From then on setup lists the AMD cards
  as HIP numbers them, so `--gpu N` and the config's `"gpu"` are HIP numbers.
- **Differences from Windows-on-NVIDIA and Linux-on-AMD:** no images yet (the CPU image encoder is Linux-only for
  now), one card per model (`--gpus` is Linux-only for now), no calibration.
- Two Windows-only engine details (#247, #325): hipBLAS can return success and still leave `hipErrorInvalidValue`
  set after some BF16/FP16 GEMMs (seen on gfx1201); the engine clears that one stale error after a GEMM that
  succeeded, on Windows only. `hipHostGetDevicePointer` returns the host pointer itself on Windows: kernels read
  mapped memory through it correctly, but a device-to-device copy into it does not land, which is why
  `tests/hip/handoff` times out there (the engine does not use that copy; `tests/hip/mapped_alias` reports it).
- Two more (#380, #377, by BlueKingMuch, measured on an RX 6800 that drives the desktop): `hipMemGetInfo` on Windows
  does not subtract what the desktop and other programs hold on the card, so `--expert-cache auto` filled the card
  past what Windows keeps in VRAM and decode fell from 41 to 30 tok/s. The engine now lowers that free figure by
  what Windows' video memory budget for the process withholds (logged once: `strata: Windows budgets N of this
  card's M MiB ...`; `STRATA_WDDM_BUDGET=0` turns it off). And the PCIe probe times its copies on the host clock
  there, since HIP's events read impossible speeds (3,300-26,000 GB/s), so a slow link now gets a smaller
  `pcie_frac` as on NVIDIA.

**What is validated (0.1.34):** #325's author ran the engine of this port on an RX 9070 XT (Windows 11, ROCm
10.2.0a20260930 in `.venv`, compiled on the PC): Coder IQ1_M at 32K, 29.2 tok/s decode, ~181 tok/s prefill,
correct answers; ctest 42 of 46. The maintainers have no Windows AMD card: the release zip was built on an NVIDIA PC,
and checked there on the Ryzen CPU's integrated Radeon (gfx1036, a test build of the same tree): with only the zip's
libraries on the PATH, `strata-device` lists the card and a hipBLAS BF16 GEMM matches the CPU; the HIP ctest passes
52 of 56, the 4 failures the same as on the RX 9070 XT (`hip_handoff`, and three tests that need a pack fixture).
The ready-made zip itself has not run a model on a discrete card yet - please report.

**Reporting a Windows AMD run** (an issue, or on #325): your card and driver version (AMD Software > System), then

```bat
engine\strata-device.exe --list-devices
engine\strata-device.exe --selftest
```

(from the Strata folder, after `set PATH=%CD%\engine\rocm\bin;%PATH%`), the end of `strata-<model>.log`, and the
speed lines the server window prints for a first answer.

**Building it yourself:** `tools\hip\build_windows.bat` (Visual Studio 2022 Build Tools with the C++ workload, Python,
git; no admin, no AMD GPU) installs ROCm from AMD's TheRock wheels into `.rocm-win`, builds, and packages
`dist\strata-windows-x64-hip.zip`; `START-HERE.bat --backend hip --prebuilt dist\` installs that one.
`tools\hip\build_windows.bat tests` also builds the HIP tests (`ctest` in `build-hip-win`, with
`.rocm-win\Lib\site-packages\_rocm_sdk_devel\bin` on the PATH). `STRATA_HIP_ARCHS` picks other architectures.

## Build

Requirements: a working ROCm driver/runtime, HIP development headers and
compiler, hipBLAS and (for tuned dense prefill) hipBLASLt development files, CMake 3.24+, a C++20 host compiler, and Git.
The model still needs sufficient system RAM and fast SSD storage for its PLE
table. VRAM occupancy alone is not a throughput measurement.

```sh
cmake -S . -B build-hip \
  -DCMAKE_BUILD_TYPE=Release \
  -DSTRATA_ENABLE_HIP=ON -DSTRATA_ENABLE_CUDA=OFF \
  -DCMAKE_HIP_ARCHITECTURES=gfx1100
cmake --build build-hip --target strata -j2
```

`CMAKE_HIP_ARCHITECTURES` is `gfx1100`, `gfx1101`, `gfx1200`, `gfx1201`, or a list such as `"gfx1100;gfx1201"`
(one binary for both). gfx1102 (the same wave32, 64 KiB LDS and dot4 instruction) builds with a warning: it passed
ctest (#192), and one RX 7600 XT (16 GB, i7-13700K, 64 GB) ran Swift IQ3_XXS over 146 requests up to 51K context at about 43 tok/s decode (28-50) and 250-300 tok/s prompt (#942); setup accepts it (unvalidated, #938); so does gfx1030 (RDNA2: the older `v_dot4_i32_i8`, a community run in #311). At startup the engine and `strata-device` compare each GPU they use
(`gcnArchName` up to the `:` feature suffix) with the architectures the binary was compiled for, and require
wave32. A binary carried to another card stops with the card's name, its architecture and the build's list,
instead of failing later with "invalid device function".

If CMake cannot find the HIP compiler, add
`-DCMAKE_HIP_COMPILER=/path/to/rocm/llvm/bin/clang++`. On the Fedora-family test
host this was `/usr/lib64/rocm/llvm/bin/clang++`. Use the compiler and libraries
from the same ROCm installation. Do not enable both GPU backends in one build.

Native IQ experts are enabled by default. CMake fetches the llama.cpp revision
pinned by this repository. For an offline build, point `STRATA_GGML_DIR` at a
checkout of that exact revision; using an arbitrary newer checkout changes the
dependency being tested.

## Model and serving configuration

Prepare a supported model pack and its MTP runtime using the existing tools.
For OrcaRouter IQ3_XXS, follow [the explicit compatibility conversion](ORCA.md);
this backend does not change quantization, model licenses, or tokenizers.
Add `--experts-bin` to the `tools/iq_pack.py` command: mmap requires the pack's
`experts.bin`, which the default native packing command does not emit. This
consumes additional disk space. Original GSQ-RCO IQ3_XXS uses shard 2 for PLE;
Orca uses shard 1. Keep each model's own pack and tokenizer together.

Use `build-hip/strata` as the executable in the server JSON. Select the AMD
device with `ROCR_VISIBLE_DEVICES`/`HIP_VISIBLE_DEVICES` if necessary. Start with
a modest context and prefill chunk size before measuring larger workloads.
Keep the PLE file on an SSD. Do not assume a configured context length proves
successful full-window inference.

Large pinned host allocations can fail on ROCm even when ordinary RAM is
available. The existing `--mmap-experts` path avoids allocating the full pinned
expert arena; it still depends on OS file-cache residency and may stall on
storage reads. It does not make SSD access equivalent to RAM.

Starting args for the original GSQ-RCO IQ3_XXS model (replace the paths):

```sh
build-hip/strata --serve \
  --pack packs/iq3xxs \
  --native /path/to/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf \
  --ple-gguf /path/to/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf \
  --mmap-experts --expert-profile data/expert-profile.bin --expert-cache auto \
  --prefill 512 --spec 4 --spec-min-p 0.5 --mtp mtp/rt \
  --max-context 4096 --kv int8 --pool-workers 15 \
  --adapt-every 0 --pcie-frac 0 --vram-reserve-mib 1024
```

`--serve` is the engine's internal token protocol. For a browser or OpenAI API,
put these args in the `args` array of the [server JSON example](ORCA.md#local-server),
set `exe` to `build-hip/strata`, and use the matching pack's `tokenizer` directory.
Launch with `.venv/bin/python -m serve.server --engine strata --config strata-hip.json --port 8080`.
The worker count above was used on a 16-core CPU; measure it for your CPU.
The 4K context is a smoke-test starting point, not a model limit. The expert cache
sizes itself automatically and leaves 1 GiB of VRAM headroom.

**The card also drives a Linux desktop (#560 #516):** keep more VRAM free than the default 700 MiB, e.g.
`./setup.sh --vram-reserve-mib 3072`. With the cache filling the card, the desktop's next VRAM need makes amdgpu move
GPU memory to system RAM (GTT), the OOM killer then ends KWin/plasmashell or `systemd-oomd` ends apps, or the
compositor fails with "Failed to pin framebuffer with error -12".

The installer supports this backend (see "Install with setup" above). Images run through the CPU encoder for now (`--vision cpu`).
Setup installs one AMD card, or several with `--gpus` (the engine's layer split; see RDNA4 below).

**Opt-in switches for RDNA2 prompts and hangs (0.1.40, off by default):** `STRATA_DENSE_MMQ=1` (#820, HIP) runs the
prompt's dense GGUF projections through the int8 MMQ kernels instead of dequantize + hipBLAS: on the RX 6900 XT a 4K
prompt read 436 -> 757 tok/s (greedy text diverges from the default after a few dozen tokens: the q8_1 activation
rounding). `STRATA_HIP_ADAPT_KERNEL_COPY=1` (#884) copies the adaptive tier's swaps with a kernel instead of the SDMA
engine, a workaround for the gfx1030 hang seen with the MMQ prompt path and adaptive swaps (untested on the reporter's
machine). Windows HIP: the doorbell kernels fence their store (#697), and the shared-expert fork is off on HIP (#816).

**The shared-expert stream fork (`STRATA_SH_STREAM`):** it is off by default on HIP, except for the gfx1151 table, and
that default stays. The measurements disagree by platform. 2x RX 6900 XT on Linux (#1272): turning it off costs 5-13%
of decode (one card 47.5 / 46.9 / 48.3 -> 50.5 / 48.5 / 51.3 tok/s with it on). RX 6800 on Windows (#816,
`bench/results/2026-10-04-community-rx-6800-windows`): on was 12-22% slower. 4x R9700: fork off was 13-45% faster.
Linux RX 6000 users can try `STRATA_SH_STREAM=1`; the output is identical either way.

**Long prompts on a big card: check the lend-coverage line (#1389, measured by the reporter on an RX 7900 XTX, gfx1100,
v0.1.40.2, 50 GB RAM).** With `--resident-experts` the prompt path borrows cache slots, and the experts of those slots
must also sit in RAM. After the first request of a start the engine prints
`FileExpertSource: N of M of the prompt path's lendable slots keep their experts in RAM too`. When N is not M, the
experts not covered are read from the pack during the prompt (now also a `WARNING` line). A 74K prompt routes through
nearly all experts, so the misses repeat: the reporter saw `wait copy` at 0.2% and 2,583 tok/s with 6201 of 6201, and
36.8% and 1,810 tok/s with 5093 of 6202 (same binary, only the start differed). Short prompts (25K) read the same either
way. The resident set is sized from `MemAvailable` at start, minus `STRATA_RESIDENT_HEADROOM_GIB` (default 4, in GiB; the
`--resident-experts` switch and `--resident-budget-gib` read it). If the line is short, lower the headroom a little and
restart; if the machine then swaps or stalls, raise it again. Re-check the line after a ROCm update: the reporter's newer
runtime kept about 1 GiB more host RAM, which moved the best value from 8 to 7. Other levers from the same report (his
numbers, not re-measured here): `STRATA_PF_FUSED=1 STRATA_PF_GEMM=1 STRATA_PF_SWITCH_MIN_T=4096` 925 -> 1,092 tok/s,
`--resident-experts` 1,092 -> 2,503 tok/s, `--prefill auto:32768` +23% at 74K. The engine uses a hipBLASLt tuning table
only when its version matches the installed library; otherwise it prefills on plain hipBLAS (about 45% slower for a 131K
prompt), so keep the table and the library paired. Not worth trying (measured flat): `--kv-resident` changes, a prefill
chunk above 32768, `--spec` above 4.

## Linux: verify timeouts while the kernel reclaims host memory (experimental workarounds)

A `verify: timed out at layer N` or "no progress for 60 s" message does not by itself mean a kernel or handshake bug. Two
community reports found the same mechanism: the GPU's queues are suspended while the kernel reclaims host pages the GPU
has pinned through a KFD userptr, and a restore that keeps returning `-EAGAIN` leaves them suspended for tens of seconds.
Each report is one machine, one workaround, and the cause is not confirmed on either; neither is a default or a general
speed claim, and neither is known to matter on Windows or on other ROCm versions.

- **Paged host allocations (#750, two Radeon AI PRO R9700, ROCm 7.2):** tracing correlated a timeout with about 31 s of
  USERPTR queue suspension on both cards. ROCr uses USERPTR for paged host allocations unless `HSA_USERPTR_FOR_PAGED_MEM=0`.
  With it in the server JSON `env`, five paired runs gave the same outputs and the same median (7 requests: 16.65 s against
  16.63 s), without the 29.8 s and 42.2 s outliers; solo requests were a little slower (3.55 s to 3.69 s).
- **`--mmap-experts` (#920, RX 6800 gfx1030, ROCm 7.2.4, 31 GiB RAM, IQ3_XXS):** every run stalled until
  `GPU_PINNED_MIN_XFER_SIZE=1048576` was in the `env`, and none has since. HIP pins the pageable source pages of large
  copies, here the mapped expert file. (The engine already sets this variable for `STRATA_ARENA_MMAP=1`.)

```json
"env": { "GPU_PINNED_MIN_XFER_SIZE": "1048576" }
```

Change one setting at a time with the rest of the configuration the same, restart the engine so the runtime reads it, and
remove it to go back to the default. Check free RAM and the driver's GTT limit before large-context tests. If the engine
runs in a systemd unit, limit it with `MemoryMax`, never `MemoryHigh`: `MemoryHigh` counts the page cache that
`--mmap-experts` reads from, and a 16K prompt stalled in `pread` for 8 minutes under it. In a report, give the runtime and
kernel versions, whether reclaim coincides with KFD queue eviction, and paired timings.

## RDNA4 (gfx1201)

The RX 9070 / 9070 XT and the Radeon AI PRO R9700 run the same kernels as gfx1100: wave32, 64 KiB of LDS per
workgroup, the signed dot4 instruction (`v_dot4_i32_iu8` through `__builtin_amdgcn_sudot4`) and a 100 MHz wall
clock. The first report and patch came from doplxyz (#178). Validated on 2026-09-30 on doplxyz's test machine:
an RX 9070 XT 16 GB and a Radeon AI PRO R9700 32 GB (both gfx1201), a Ryzen 9 3900X (16 threads, no AVX-512),
47 GB RAM, Ubuntu 24.04 in a KVM/VFIO guest.

- **Build:** complete HIP build with tests (`-DCMAKE_HIP_ARCHITECTURES=gfx1201 -DSTRATA_BUILD_TESTS=ON
  -DSTRATA_PREFILL_MMQ=ON`), with the system ROCm 7.14 (hipBLASLt 1.4.1) and with setup's own path: TheRock
  7.10.0a20251120 wheels from `gfx120X-all` (hipBLASLt 1.2.0) and `build_engine_hip`.
- **ctest** (all 45 registered tests, R9700): 42 pass, `hip_prefill_hipblaslt_gemm` skips (no table), and two
  fail for reasons outside the GPU: `ple_parity` needs the Q2_0 model fixture, `expert_multi_test` needs an
  AVX-512 CPU. `hip_handoff` needed the volatile ring store: on gfx1201 a plain store to mapped pinned memory
  stays in the GPU's L2 until the stream is synchronized.
- **Arch check:** a gfx1100-only build stops on the gfx1201 card at startup with the message above (engine and
  `strata-device`).
- **End to end** (Coder IQ1_M, setup's arguments: `--expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5`,
  MTP, `--kv int8`, `--max-context 32768`; 128 greedy tokens, system ROCm 7.14, no hipBLASLt table). The answers
  are coherent and the same on both cards. The live server smoke (`serve/server.py`: web page, models, props,
  chat, prefix reuse, streaming, the Anthropic endpoint, a sampled code answer) passed 9/9 on each card.

  | card | expert cache | peak VRAM | 4K prompt | 4K decode | 16K prompt | 16K decode |
  |---|---|---|---|---|---|---|
  | Radeon AI PRO R9700 32 GB | 12,288 slots, 23.4 GiB | 29.7 GiB | 982 tok/s | 45.5 tok/s | 1,402 tok/s | 48.3 tok/s |
  | RX 9070 XT 16 GB | 4,931 slots, 9.4 GiB | 15.6 GiB | 782 tok/s | 30.8 tok/s | 1,235 tok/s | 35.4 tok/s |
  | R9700, TheRock 7.10 wheels | 12,288 slots | | 957 tok/s | 44.2 tok/s | 1,284 tok/s | 43.9 tok/s |

  The engine's resident memory was about 26 GB in every run. Since 0.1.31 `__byte_perm` is one `v_perm_b32` and the
  packed byte subtracts/compare work on four lanes at once (#262, ttio2tech): decode +15% on the R9700 (46.0 -> 53.0
  tok/s on a 4K prompt, 52.0 -> 60.5 warm) and +5-7% on the 9070 XT, prompts unchanged, the same tokens.
- **hipBLASLt:** the validation above used hipBLASLt 1.4.1. A table calibrated on the R9700 at the engine's shapes
  with that version (0.98-1.76x per GEMM over hipBLAS) changed the end-to-end prompt speed by 0-3%, within noise,
  so none was shipped for it: there the plain hipBLAS path is already close (a table I calibrated on my own R9700
  with 1.4.1 measured the same: 1.15x per GEMM, +1% prompt - see "Tuning table" below). For hipBLASLt 1.5.0 (ROCm
  10.2.0a nightly) `tools/hip/gfx1201-hipblaslt-100500.txt` is shipped (see "Tuning table" below); on one R9700 it
  measured +3.9% prompt speed on 4,210-token prompts (1,590 vs 1,531 tok/s), a modest gain.
  With the hipBLASLt 1.2.2 of a system ROCm 7.2.4 the plain path is far off, and `tools/hip/gfx1201-hipblaslt-100202.txt`
  is shipped for it (24 of the gfx1100 table's 26 shapes; setup uses it only with that exact version): on an R9700
  with the full IQ3_XXS, `--kv int8 --kv-resident 32768`, a fresh 32K prompt read at 638 -> 1,177 tok/s and a 7K one
  at 656 -> 1,164 tok/s with it, decode unchanged.  `hip_prefill_hipblaslt_gemm` passes with it.
- **Both cards in one run (layer split, engine 0.1.30):** the config's `"backend": "hip", "gpu": [1, 0]` (R9700
  first) runs through `serve/server.py` (setup writes it with `--gpus 1,0` since 0.1.31). Auto split put layers 0-27 on the R9700 and
  28-47 on the 9070 XT. With every expert on the GPUs the split gives exactly the tokens of the R9700 alone (4K and
  16K prompts); checkpoints on the split (second turn, rewind, a prompt sharing a prefix, a cancelled prompt
  retried) give exactly the tokens of a fresh read. The conversation cache refuses a split at start (exit 2).
  On this pair the split does not pay: the R9700 alone already holds all 12,288 expert pairs.

  | run (the same session, warm) | 4K prompt | 16K prompt | decode |
  |---|---|---|---|
  | R9700 alone | 1,794 tok/s | 1,804 tok/s | 51-52 tok/s |
  | R9700 + 9070 XT, auto split | 1,384 tok/s | 1,906 tok/s | 42-43 tok/s |
  | RX 9070 XT alone | 1,016 tok/s | 1,519 tok/s | 38-40 tok/s |

  A split is worth it when no single card holds the model's experts. Since 0.1.31 a split pins the whole expert
  arena on Linux (the 8 GiB cap is for Windows/WSL2 only, #253): with an expert cache of 1,500 on the R9700 (most
  experts streamed) the split read a 16K prompt at 1,803 tok/s instead of 1,287, with the same tokens in 5 + 5
  starts. `"split_skip_if_fits": true` in the config (0.1.31, opt-in) runs such a pair on the first card alone when it
  holds every profiled expert: with the R9700 first, 4K prompts 1,776 tok/s (split: 1,244) and decode ~60 tok/s
  (split: ~51), the tokens of the R9700 alone (docs/MULTI_GPU.md).
- **Speed switches (engine 0.1.32, measured on the R9700 / 9070 XT with the Coder IQ1_M pack):**
  - the MoE router (`router_top10`) runs a HIP kernel without its serial FP64 sum and block barriers by default: the
    same ids and weights bit for bit (`hip_router_fast` checks 65,536 rows), 39 -> 9-12 us per call, the same greedy
    tokens (5 + 5 starts). Decode: 62.4 -> 70.0 tok/s on the R9700 and +4% on the 9070 XT in one A/B here (ROCm 10.2
    nightly); a user's repeated A/B/A/B with setup's TheRock 7.10 wheels and an i5-12600K measured +1-4%, inside a
    12-20% run-to-run spread (#432) - how much it gains depends on how much of decode the router is on that setup.
    `STRATA_HIP_ROUTER_OLD=1` runs the portable kernel.
  - `STRATA_HIP_WMMA=1` (opt-in, gfx12 only, int8 KV): the prompt path's QSA attention on RDNA4 matrix cores (the engine prints a one-line hint about it at start on gfx12 with `--kv int8`; it never enables it, because the output bits change; setting the variable to anything, 0 included, silences the hint)
    (`v_wmma_f32_16x16x16_f16`, FP16 hi + lo halves like the CUDA tensor-core kernel; `hip_prompt_attn_wmma` bounds it
    against the FP32 kernel and FP64). 7.2-7.5x the portable kernel; R9700 prompts 4K 1,784 -> 2,427 tok/s, 16K
    1,797 -> 2,700 (a user with TheRock 7.10: +29-34%, #432). Not bitwise: greedy text differs from token ~50 on, as with the CUDA tensor-core attention. With it
    the prompt path's expert ring is 96 slots (as STRATA_PREFILL_RING=96): with the default 384 the 9070 XT's 4K
    prompts fell to 718 tok/s; with 96 they gain (1,017 -> 1,211; 16K 1,518 -> 2,032). PR #329
    (bsorensen110) contributed an equivalent gfx12 WMMA kernel of the same speed (within 1%); this one also masks KV
    pages that KV streaming has not made resident, as the decode kernel does. On my R9700 (engine 0.1.40,
    IQ3_S at 431072 ctx, `--kv int8`, the gfx1201 hipBLASLt 1.4.1 table below, which names the machine, GPU idle):
    fresh prompts of 4,210 and 8,830 tokens - the longest prompt I measured there is 8,830 tokens, the 431072 ctx is
    what the model runs at, not what these prompts reach - three matched trials each, the first fresh prompt after
    each start left out: 1,571 -> 2,064 and 1,568 -> 2,065 tok/s at 8,830 tokens (+31.4 and +31.7%) and 1,699 ->
    1,990 at 4,210 (+17.1%), medians 1,571 -> 2,064 and a 1.27x geometric mean over those three. decode
    89.8 -> 86.9 tok/s, unchanged. `STRATA_SELECT_WMMA=1` on top changed nothing at those lengths (2,026 tok/s);
    its +1.5% is a 16K/262K-ctx number. `hip_prompt_attn_wmma` passes on that card.
  - The prompt path's QSA top-k picks its kernel by the blocks a query actually has, not the cache's capacity (#337,
    bsorensen110): the same ids, on by default with AMD (NVIDIA keeps its capacity rule: there the 64K prompts read
    1-3% slower with it). `STRATA_SELECT_WMMA=1` (opt-in, gfx12) adds #337's
    matrix-core block scorer; it selects slightly differently (254 of 256 queries the same) and gained +1.5% on 16K
    prompts at a 262K context on the R9700.
- **Known:** rarely (about 1 start in 10) a HIP run's greedy output differs from another start's at some token, on
  one card or two and on engine 0.1.29 as well; not yet explained.
- **Not validated:** images, long contexts beyond 16K, answer-quality benchmarks.

## Community-validated cards

Run by their owners, not on the maintainers' machines; setup accepts them like gfx1100 / gfx1201. setup ships a
hipBLASLt table for gfx1200 (the numbers below); gfx1101 has none (make one with [Tuning table](#tuning-table)
and compare the prompt speed with and without it).

- **gfx1101, RX 7800 XT 16 GB** (jhohertz, #254; engine 0.1.29, Ryzen 9 5950X, 121 GiB RAM, system ROCm with
  hipBLASLt 1.4.1): `./setup.sh --backend hip` detected the card and compiled the engine; `strata-device --selftest`
  passed; ctest 30/32 (`ple_parity` needs the Q2_0 fixture, `platform_memory_test` the memlock limit). Coder IQ1_M,
  64K context, MTP, with a table the owner calibrated: fresh prompts of 4K-9K tokens at 898-953 tok/s, decode
  38-44 tok/s (128 tokens).
- **gfx1200, RX 9060 XT 16 GB** (Efeisot, #256 after #176; engine 0.1.29, Ryzen 9 7950X, 64 GiB RAM, ROCm 7.2 with
  hipBLASLt 1.2.2): ctest 32/32 (without `ple_parity` and `platform_memory_test`). Coder IQ1_M, greedy, MTP
  `--spec 4 --spec-min-p 0.5`, with a table the owner calibrated:

  | prompt | prompt speed | decode |
  |---|---|---|
  | 2,374 tokens | 540 tok/s | 27.1 tok/s (0.69 draft acceptance; 31.0 at 1.00) |
  | 65K (`--kv int8 --kv-resident 65536 --prefill 16384`) | 748-753 tok/s | 26-40 tok/s by acceptance |
  | 130K (the same flags) | 725 tok/s | - |

  Greedy output was the same across runs. For comparison, llama.cpp's HIP build measured 20 tok/s decode and
  450 tok/s prompt on that card.

  Since 0.1.38 setup ships that table (`tools/hip/gfx1200-hipblaslt-100202.txt`). On engine 0.1.31, one binary (ctest
  37/37, without `ple_parity` and `platform_memory_test`), prompts of repeated code blocks, prefill 2560 for
  the short prompt and 16384 with `--kv int8 --kv-resident 65536` otherwise, two runs each - prompt speed
  with the table vs plain hipBLAS (no table):

  | prompt | with the table | plain hipBLAS | gain |
  |---|---|---|---|
  | 2,374 tokens | 597-599 tok/s | 396-398 tok/s | 1.50x |
  | 65,045 tokens | 754-757 tok/s | 380 tok/s | 1.99x |
  | 130,091 tokens (the same flags) | 719 tok/s | 363-370 tok/s | 1.98x |

## RDNA2 (gfx1030)

The RX 6800 / 6800 XT / 6900 XT / 6950 XT run the same kernels: wave32 and 64 KiB of LDS per workgroup. The one
difference is the dot instruction: RDNA2 has no `v_dot4_i32_iu8` (gfx11 and newer), so
`dp4a` uses the plain signed `v_dot4_i32_i8` through `__builtin_amdgcn_sdot4`, which compiles to a single
`v_dot4c_i32_i8` (same signed x signed byte products, modulo 2^32). There is no WMMA; the QSA scorer takes the
same ordered FP32 fallback as gfx1100. CMake lists gfx1030 as unvalidated (the build warns) until a maintainer has
run it; the report below is from a community machine: an RX 6900 XT 16 GB (gfx1030), an i7-13700KF (8 P-cores and
8 E-cores, AVX2, no AVX-512), 63 GB RAM, NixOS, ROCm 7.2.3 from nixpkgs (clang 22, hipBLAS 3.2).

- **Build and tests** (engine 0.1.26): a complete HIP build for gfx1030, made by hand with cmake and ROCm's own
  `clang++` (the nixpkgs ROCm is not an `/opt/rocm` tree, so setup's `build_engine_hip` was not exercised; the
  binary was placed in `engine/` for setup to use). All 28 registered ctest tests pass on the card, including
  `hip_device_selftest`. An engine 0.1.30 build of this branch configures, builds and runs clean on the same card;
  the speeds below are measured on it.
- **End to end** (Swift 1.5 IQ3_XXS, `--context 131072` with `--kv-resident 32768 --adapt-every 1
  --vram-reserve-mib 1024`, 200 greedy tokens): 38-42 tok/s decode with the default 15 CPU pool workers (one per
  physical core except the host thread; 8 workers: 36; 24 = every logical core: 27), consistent even with the
  131,072-token context full.
- **Prefill** (the same 15-worker configuration): 246 tok/s on a 2,000-token prompt, 330-339 tok/s at the auto
  8,192-token chunk (7,997 and 15,967 tokens; time to first token 8.9 and 48.3 s), 133 tok/s on a 522-token
  prompt - short prompts are fixed overhead (19-25 tok/s on 34 tokens). Decode after a 16K prefill holds at
  45.5 tok/s.
- **16 GB card:** the expert cache holds 6,310 slots (10.2 GiB) at 16K context and 5,247 slots at 131,072, where
  setup keeps the KV cache in VRAM because IQ3_XXS needs about 60 GB of RAM plus the cache to stream it. The
  `--pcie-frac 0` and `--adapt-every 0` of the 7900 XTX configuration in
  [AMD_HIP_PERFORMANCE.md](AMD_HIP_PERFORMANCE.md) cost 8.6 and 13.6 tok/s here (30 with the defaults): keep the
  defaults on a 16 GB card.
- **hipBLASLt:** ROCm's hipBLASLt ships no gfx1030 kernels, so there is no table and the plain hipBLAS path runs.
- **Prompt GEMMs in FP16 (#835):** rocBLAS on gfx1030 has tuned kernels for FP16-in / FP16-out GEMMs only; the prompt
  path's FP16-in / FP32-out and BF16 products ran generic kernels about 6.6x slower (5.6 and ~5.3 TFLOPS against 37.7 at
  N = 10240, T = 7313, K = 2560). `STRATA_HIP_PROMPT_F16=1` (opt-in, off by default; the engine prints a tip on gfx103x) runs the 16-bit prompt GEMMs FP16 in and out;
  unset is the old path. `STRATA_DBG_NAN=1` also counts the non-finite FP16 outputs (a sum past 65504). The output is not bit-identical to it (an FP16 rounding of each GEMM's output replaces the BF16 rounding of
  the BF16 GEMMs' activations); what the distribution check showed and did not show is in
  [bench/results/2026-10-04-rdna2-fp16-prompt](../bench/results/2026-10-04-rdna2-fp16-prompt/README.md). Measured on a
  second community machine - 2x RX 6900 XT 16 GB (one card for these numbers), Ryzen 5 5600X (6 cores, AVX2), 128 GB,
  Ubuntu 26.04, ROCm 10.0.0, engine 0.1.39 built by hand with `-DSTRATA_PREFILL_MMQ=ON` as setup does, GSQ-RCO IQ3_S:
  a 9.4K / 34.7K / 105.8K-token prompt reads at 744 / 915 / 926 tok/s instead of 439 / 466 / 461; decode is unchanged.
  On that machine the HIP ctest passes 60 of 65 with and without this change: 2 skipped (`hip_prompt_attn_wmma`, no
  matrix cores; `hip_prefill_hipblaslt_gemm`, no hipBLASLt table) and 3 that fail for reasons outside the engine
  (`ple_parity` needs a Q2_0 PLE file that is not on that machine, `expert_multi_test` refuses the CPU without AVX-512,
  `platform_memory_test` cannot `mlock` at the shell's default `ulimit -l`).
- **gfx1031** (RX 6700 XT, #524): setup knows it (the `gfx103X-all` wheels, unvalidated); its reporter runs it daily
  on one card. More reports: an RX 6700 XT 12 GB run as gfx1030 on ROCm 7.2.4 (#1027: IQ2_XS, decode 30-32 tok/s, prompt about 300 tok/s, 6 of 6
  needles), and an RX 6800M 12 GB on Windows with a self-built engine (#915, #1078: Q2_0, decode 9-27 tok/s, prompt 50-114 tok/s; the
  experts stream from disk with 31 GB of RAM). The ready-made Windows zip has no gfx1031 code unless it is built with it: `STRATA_HIP_ARCHS=gfx1031`
  in `tools\hip\build_windows.bat` (the default list has it from 0.1.40.2 on).
- **gfx1150** (Radeon 890M, Ryzen AI 9 HX PRO 370, Strix Point, #1217): builds and runs from 0.1.40.2 as an unvalidated target (CMake warns; the
  device code already covers it: the WMMA guards and `gfx_arch_is_gfx11_wmma()` include it). One community machine: MINISFORUM N5 PRO, 96 GB DDR5-5600, GTT
  raised to 64 GiB (`ttm.pages_limit=16777216 ttm.page_pool_size=16777216`), TheRock ROCm 7.14.1 for gfx1150, an unprivileged LXC container, IQ3_XXS: chat and
  tool calls work, decode 15-19 tok/s, the HIP ctest passes apart from tests that need files a public checkout lacks.
  Prompts read at about 125-130 tok/s on plain hipBLAS and 1.5-1.7x faster with `tools/hip/gfx1150-hipblaslt-100401.txt` (ROCm 7.14.1's
  hipBLASLt; 3.6K tokens 124 -> 216 tok/s, 7K 130 -> 226 tok/s, decode unchanged):
  [bench/results/2026-10-07-community-gfx1150](../bench/results/2026-10-07-community-gfx1150/README.md).
  Setup does not install for it yet (an integrated Radeon other than Strix Halo is named and not supported): build by hand with `-DCMAKE_HIP_ARCHITECTURES=gfx1150`
  and point `STRATA_HIPBLASLT_TUNING` at the table yourself.
- **Not validated:** gfx1032 (the same `dp4a` path, no hardware report), setup's own build path and the
  `gfx103X-all` wheels on gfx1030, images, answer-quality benchmarks. RDNA1 (gfx1012, RX 5500 XT) builds by hand:
  [OLDER_GPUS.md](OLDER_GPUS.md#amd-building-gfx906-and-gfx1012).

## gfx906 (Instinct MI50 / MI60, Radeon VII): wave64, built from source

gfx906 is wave64 and has no WMMA and no packed byte arithmetic, so the wave32 backend above refuses it. A separate opt-in build compiles the CUDA sources as HIP through a small compat layer
(`include/strata/platform/hip_compat/`), with a CUDA warp mapped to a logical half of the 64-lane wavefront
(32-wide shuffles, a ballot of its own half). The hot kernels have wave64 layouts of their own (below). Setup does
not build it yet: build by hand, and run `serve/server.py` with a config as on any other card.

**ROCm.** AMD's current ROCm releases no longer ship gfx906 libraries. The build and the measurements below used
HIP 7.14 from the community image [`mixa3607/rocm-gfx906:7.14-complete`](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
(rocBLAS/hipBLAS with gfx906 kernels), on the kernel's amdgpu driver (Ubuntu 24.04, kernel 6.8). hipBLASLt is not
used.

```sh
# inside the image, with this checkout at /src and llama.cpp at the pinned commit in third_party/llama.cpp
apt-get update && apt-get install -y cmake ninja-build git python3 build-essential
cmake -S . -B build-906 -G Ninja -DSTRATA_HIP_GFX906=ON -DCMAKE_HIP_ARCHITECTURES=gfx906 \
  -DSTRATA_GGML_DIR=/src/third_party/llama.cpp \
  -DCMAKE_C_COMPILER=/opt/rocm/llvm/bin/clang -DCMAKE_CXX_COMPILER=/opt/rocm/llvm/bin/clang++
ninja -C build-906 strata
```

`STRATA_HIP_GFX906` turns on the CUDA targets and refuses `STRATA_ENABLE_HIP` (one HIP build at a time). Run the
container with `--device=/dev/kfd --device=/dev/dri --group-add video --ipc=host --ulimit memlock=-1`; on some boards
`HSA_OVERRIDE_GFX_VERSION=9.0.6` is needed for the runtime to accept the card.

**What differs from the CUDA build** (all under `STRATA_HIP_GFX906`; each A/B switch restores the CUDA layout):

| | gfx906 | switch |
|---|---|---|
| grouped native experts | mode 7: signs as `dp4a(g ^ m, u) - dp4a(m, u)` (no byte SIMD), grids and activations in LDS, one weight load per group; the IQ formats only (Unsloth's K-quant / Q5_1 / Q8_0 experts take the CUDA layout) | `STRATA_EXP_MODE=2` (previous AMD layout) |
| MMVQ | one wavefront per row, 64-lane butterfly, no LDS | `STRATA_MMVQ_WAVE=0` |
| router top-10 | one wavefront, register argmax | - |
| verify-window routing (256-expert router) | the window's tokens in one multi-column BF16 projection + one top-k, ~3 ms of a ~55 ms window | `STRATA_ROUTE_PER_TOKEN=1` |
| hyper-connection read | norm loads in flight, 8 lanes per row for `up` | `STRATA_GR_FAST=0` |
| `gr_down_multi`, opt-in | split along K (8 rows x 5 slices of 2048), ~3% per window; sums in another order than the single-token read, so off by default | `STRATA_GR_SPLIT=1` |
| IQ4 table lookup | llama.cpp's `v_perm_b32` sequence (HIP's `__byte_perm` is a scratch array) | - |
| QSA prompt attention | the FP32 kernel (no tensor cores / WMMA) | - |

Every switch above was checked bitwise against the layout it replaces: the parity tests on synthetic data,
`native_expert_bench` on real GGUF rows (one mode against another), and the greedy text of a fixed request.

**Measured** on 2x Instinct MI50 16 GB (85 W power limit each, PCIe 3.0 x16 both, peer access between them),
Xeon E5-2666 v3 (10 cores, AVX2), 32 GB DDR4, SATA SSD; Coder IQ1_M (`--native` pack), 131,072-token context,
`--kv int8 --kv-resident 32768`, `--layer-split 27`, `--prefill 4096`, MTP `--spec 4 --spec-min-p 0.5`,
`--pcie-frac 0`, `STRATA_ARENA_MMAP=1`. Measured on the port this build was cut from (builds on the 0.1.30 and 0.1.33 bases, the same
kernels, plus the layer-split weight trim and the mapped arena, which are separate pull requests):

- all 12,288 experts held in VRAM across the two cards;
- a 17,043-token agent prompt (Claude Code's first request, tools included) read in 43.5 s (392 tok/s), decode
  on it 57.7-59.3 tok/s;
- 25 GB of the 32 GB RAM available while serving;
- six repeats of that request with a 6,000-token output cap and a 900-second two-client load test run six times
  (407 requests, 202,940 tokens): no stalls, no errors, no GPU faults.

For comparison on the same machine: llama.cpp (3cf0325, ROCm) on the same Coder IQ1_M measured 26.9 tok/s `tg128`.

**Not done:** setup.py detection and an automatic build, Windows, images (`--vision`), MI60 and Radeon VII (the
same gfx906 ISA; not run), a single-card run, the tensor-split experiment (the halves of every layer on two cards;
it works but is not part of this build).

## Tuning table

A hipBLASLt table holds solution ids that are valid only for one GPU architecture and one hipBLASLt version, so it
is calibrated on the card with the ROCm the engine runs with. The shipped gfx1100 table's rows are the engine's
dense GEMM shapes; to calibrate them for another card or version (a few minutes):

```sh
cmake --build build-hip --target tune_hipblaslt
CASES=$(awk 'NR>2 {printf " --case %s,%s,%s,%s,%s", $1, $5, $2, $3, $4}' tools/hip/gfx1100-hipblaslt-100200.txt)
./build-hip/tune_hipblaslt $CASES --tuning-out table.txt
```

The file's second line names the architecture and version (`STRATA_HIPBLASLT_TUNING_V1 gfx1201 100401`); save it
as `tools/hip/<arch>-hipblaslt-<version>.txt` for setup, or point `STRATA_HIPBLASLT_TUNING` at it. Compare the
prompt speed with and without it before keeping it.

Shipped tables:

- `gfx1100-hipblaslt-100100.txt`, `gfx1100-hipblaslt-100200.txt`: RX 7900 XTX.
- `gfx1100-hipblaslt-100401.txt`: RX 7900 XTX, calibrated with the packaged ROCm 10.0.0
  (`rocm/dev-ubuntu-24.04:10.0.0-full`, hipBLASLt 1.4.1) on a Ryzen 7 9800X3D, over the 26 dense GEMM
  geometries of the shipped gfx1100 table. Without it, that stack reads a prompt through plain hipBLAS: on
  Swift 1.5 IQ3_XXS at 262144 ctx, fresh-prompt prefill measured 682 -> 1,038 tok/s at 1.7K tokens and
  857 -> 1,524 tok/s at 6.5K (medians of 3, decode unchanged), with `hip_prefill_hipblaslt_gemm` reporting
  `fallbacks=0`. setup uses it only when the installed hipBLASLt reports 1.4.1.
  The 43-row revision adds the two geometries the 26 dense ones do not reach, calibrated with
  `tune_hipblaslt` from the same tree and the engine's 32 MiB workspace: `f16 1280 2560 1280` (the
  small-T expert GEMM, T=8..999) and `bf16 10240 2560 10240`. A prompt on Qwen3.8-Flash-Next GSQ-RCO
  IQ2_XS at 192000 ctx otherwise falls back on the expert GEMM for nearly every launch:
  `STRATA_HIPBLASLT_VERBOSE=1` reported 642 fallbacks over 642 distinct T values (1..1873) with the
  26-row table. Adding the rows measured 711 -> 938 tok/s (+31.9 %, mean of 3 interleaved pairs:
  +36.0 / +32.4 / +27.6 %) at a 3151-token prompt, decode unchanged (58.6 -> 60.6 tok/s, inside this
  box's 2.3 % run drift), and `fallbacks=0`. The lookup takes the nearest T bucket for a matching
  geometry, so the buckets cover the whole small-T range rather than one row per T.
- `gfx1100-hipblaslt-100500.txt`: RX 7900 XTX (gfx1100, 24 GB), calibrated with a ROCm 10.2.0a20261003 nightly
  SDK (`libamdhip64.so.7.17.26392`, hipBLASLt 1.5.0, version number 100500). Same 16 dense GEMM geometries at
  T=4096 and T=8192 as the other gfx1100 tables (32 rows), calibrated with `tune_hipblaslt` run against that
  library. On this ROCm the engine refuses the 100100/100200 tables (version mismatch) and prefills on plain
  hipBLAS; with this table a 0.1.38-lineage nightly engine prefills a 131071-token prompt at 1687 tok/s vs 926
  without it (median of 3 clean runs each, greedy; decode unchanged). Per-run data and the calibration command:
  the `2026-10-04-community-rx7900xtx-hipblaslt-100500` folder of PR #745.
- `gfx1201-hipblaslt-100500.txt`: Radeon AI PRO R9700 (gfx1201, 32 GB), calibrated with ROCm 10.2.0a20260914
  (AMD's `gfx120X-all` nightly, hipBLASLt 1.5.0, library build `d3164197`). 16 dense GEMM geometries at T=4096 and
  T=8192, 32 rows. setup uses it only when the installed hipBLASLt reports 1.5.0 (it is found in `/opt/rocm`
  when that is a system ROCm 7 or newer). The version number is the only thing the engine can check, so another
  1.5.0 build could number its solutions differently. `hip_prefill_hipblaslt_gemm` (with `STRATA_HIPBLASLT_TUNING`
  set) is a smoke test: it refuses a table for another architecture or version and checks that one BF16 and one
  F16 row exist and agree with hipBLASEx, which covers 2 of the 32 rows. It does not prove that a solution id is
  valid: the engine falls back to hipBLASEx for an id the library rejects, and the test still passes. Run it with
  `STRATA_HIPBLASLT_VERBOSE=1` and look for `fallbacks=0` in its summary line, and recalibrate with
  `tune_hipblaslt` before using this table with a different 1.5.0 build.
- `gfx1151-hipblaslt-100400.txt`: Radeon 8060S (Ryzen AI Max+ 395, gfx1151, 128 GB; Fedora 44, kernel 7.2.8),
  calibrated with the ROCm 7.14.0a20260608 wheels that setup installs for gfx1151 since 0.1.40.2 (#1267): their
  hipBLASLt is 1.4.0 (version number 100400), so neither `gfx1151-hipblaslt-100401.txt` nor `-100500.txt` applied and
  setup's engine read the prompt through plain hipBLAS. The same 90 rows as the 100401 table (its geometries, the
  prompt shapes at `--prefill 16384` included), calibrated with `tune_hipblaslt --workspace-mib 32` and that table's
  cases: 3.1x to 14.1x faster than plain hipBLAS per shape (median 6.2x). With `STRATA_HIPBLASLT_TUNING` set,
  `hip_prefill_hipblaslt_gemm` passes (`launches=4 fallbacks=0`), `hip_prefill_hcd_exact_parity` passes (solutions
  1176 / 1177 at 9 chunk sizes, 0 outputs differ, `fallbacks=0`), and the HIP ctest passes 61 of 62 (`ple_parity`
  needs the Q2_0 model fixture). setup uses it only when the installed hipBLASLt reports 1.4.0. The prompt speed
  of a whole model with this table is not measured yet.

- `gfx1150-hipblaslt-100401.txt`: Radeon 890M (gfx1150, Strix Point APU, 16 CUs), calibrated with AMD's TheRock ROCm 7.14.1 for
  gfx1150 (hipBLASLt 1.4.1 `cd957402`, version number 100401) and the engine's 32 MiB workspace. The 16 dense GEMM geometries a
  Qwen3.8-Flash-Next prompt logs on that card (the same 16 as the gfx1201 100500 table) at T = 64, 128, ..., 16384: 144 rows. Two
  `tune_hipblaslt` passes back to back; each row keeps the solution with the lowest median of all six repetitions, because one
  repetition stalled now and then (a 100 ms call read up to 1.6 s) and the tool's mean-of-three picks differed in 26 rows between
  the passes. Every row beats plain hipBLAS, 1.55-22.6x per GEMM (geometric mean 4.5x), and `STRATA_HIPBLASLT_VERBOSE=1` reported
  no fallbacks over prompts of 28-7,880 tokens. Setup does not use it (it does not install for gfx1150): point
  `STRATA_HIPBLASLT_TUNING` at it. Measurements and the per-row timings:
  [bench/results/2026-10-07-community-gfx1150](../bench/results/2026-10-07-community-gfx1150/README.md).

Calibrated on my PC and kept in this checkout, not shipped: `gfx1201-hipblaslt-100401.txt`, a Radeon AI PRO
R9700 (gfx1201, 32 GB, 1002:7551) on a Ryzen 9 9950X3D with 62 GiB RAM, CachyOS (kernel 7.3.0-rc6-1-cachyos-rc,
`linux-firmware-amdgpu 20260916`, the kernel's amdgpu driver), system ROCm `rocm-gfx120x-bin 10.1.0` (hipBLASLt
1.4.1, HIP 7.16.26385, clang 24.0.0), Python 3.14.7 in setup's own venv, engine 0.1.40 compiled there for gfx1201,
the model files on a 4 TB NVMe (btrfs), IQ3_S at 131072 ctx with `--kv int8`, the GPU idle. `tune_hipblaslt` over
the 32 geometries of `gfx1201-hipblaslt-100500.txt`: 1.15x geometric mean per GEMM over hipBLAS - 1.55-1.68x at
N=512/640 K=2560, 0.99-1.13x at N=10240/12288 K=2560. End to end with `tools/hip/bench_prefill.py` over two
server starts, fresh prompts of 4,210 and 8,830 tokens, three matched trials each - the first fresh prompt after
each start is left out, it is cold (949 and 1,423 tok/s there): 1,606 -> 1,613, 1,701 -> 1,726 and 1,605 -> 1,610
tok/s (+0.5, +1.5, +0.4%), medians 1,606 -> 1,613 and a 1.008x geometric mean over those three, decode
87.1 -> 85.1 tok/s - within noise, as the R9700 paragraph above found. `hip_prefill_hipblaslt_gemm` passes with it: `tuning enabled (30 rows, gfx1201,
version 100401)`, `launches=4 fallbacks=0`.

Two rows were dropped from that table: bf16 N=10240 K=320 ldy=10240 at T=4096 and at T=8192. At T=8192 hipBLAS was
faster in five separate `tune_hipblaslt` runs - 0.609-0.619 ms against 0.648-0.677 ms for the best of the 16
hipBLASLt candidates (0.90-0.95x), with four different winning solution ids, all slower than hipBLAS. Dropping only
the T=8192 row does not send that call back to hipBLAS: `closest()` matches dtype/N/K/ldy and takes the nearest T
bucket, so T=8192 would run the T=4096 row; both rows go or neither.

Two calibration notes from my machine. The first case of a fresh `tune_hipblaslt` process times its hipBLAS
baseline while the clocks are still ramping: that T=4096 shape read 0.31-0.63 ms across five runs while the case
timed right after it held 0.606-0.619 ms every time, so time a heavy case first and read the later ones. And the
first fresh prompt after the model loads is not a measurement: it gave 949 and 1,423 tok/s in the two arms of that
A/B, while the three trials after it differed by 0.4-1.5%.

## Original backend validation (PR #94)

The following is historical validation of the original backend, not a fresh
test count for this replacement. Current build/test evidence and performance
limits are recorded in [AMD_HIP_PERFORMANCE.md](AMD_HIP_PERFORMANCE.md).

Tested against upstream `c1e903310f211e6630780c3bd2038778c071c68d` (0.1.20),
with pinned llama.cpp `3cf03257f219afbe7334045ff7c6a06ac68c627d`.
The host has an RX 7900 XTX (24 GiB), Ryzen 9 7950X3D, 64 GiB system RAM,
and Fedora-family Linux with ROCm 7.1.52802 / Clang 20. The original model used
for the smoke check was on mechanical RAID0; cold page faults were slow. Keep
latency-sensitive data on SSD and measure warm residency separately.

- Complete HIP Release build, including the executable and parity targets.
- All 25 tests selected by the command below passed on 2026-09-29.
- Intrinsic parity includes signed packed-byte dot products and overflow.
- CPU/GPU mapped-memory handoff covers delayed publication, changing payloads,
  and repeated graph replay in both directions.
- GR coverage includes the maximum eight-token fused batch and graph replay
  after inputs change. Native QSA checks the scalar HIP fallback.
- Q8_K quantization is byte-exact against the existing reference. HIP's
  `__fmul_rn` can become ordinary multiplication; disabling contraction in
  `quantize_act.cu` preserves the separately rounded product before the magic
  rounding bias is added. CUDA's compile flags remain unchanged.
- The mmap regression covers canonical and differing native layer sizes,
  truncated/oversized files, lookup bounds, and reopening after errors.
- CUDA 13 / GCC 15 / sm_89 executable build passed; GR, sampler, quantization,
  and mmap tests passed on the same tree. This is a regression check, not a
  claim of complete CUDA inference validation.
- CPU-only `strata-plan` build passed without either GPU backend.
- Real GSQ-RCO IQ3_XXS server smoke with mmap, the shipped expert profile,
  automatic GPU cache, and MTP passed arithmetic, executable Python addition,
  system-marker recall, and 1,170-token batched-prefill recall. All four requests
  ended normally with correct answers; one engine process served them all.
  The cache held 11,142 experts (18.08 GiB), with 874 MiB of VRAM free after
  startup. These short checks do not establish a throughput or quality benchmark.

Build all targets before running the registered focused checks:

```sh
cmake --build build-hip -j2
ctest --test-dir build-hip --output-on-failure --timeout 60 \
  -E '^(ple_parity|platform_memory_test)$'
```

`ple_parity` requires an external model fixture. `platform_memory_test` requests
256 MiB of locked memory; the test shell's 8 MiB memlock limit prevented that
test from passing. These are explicit exclusions, not skipped tests counted as
passes. Additional unpublished upstream fixture suites are not covered.

The published performance table explicitly identifies the measured development
snapshot; it must not be read as a benchmark of every subsequent rebase. Vision, long-context stress, broad answer-quality
equivalence, other AMD cards, and mixed-vendor inference are not validated here.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://www.ai-hao123.com/shichang/software-25628048.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [面向大规模网络的 Strata 工业级架构基准](https://www.ai-hao123.com/pingce/data-92590879.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://www.yx-sf.com/tech/14049.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://nuzr.tcti.cn/chanpin/profile-56473447.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://www.ai-hao123.com/anli/team-65744264.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://www.yx-sf.com/tech/58252.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://dhug.wtpuscm.cn/tuiguang/travel-264134.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://zsqd.wtpuscm.cn/qiye/guide-361.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://www.ai-hao123.com/chuangxin/team-96682273.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 异步中间件流水线与 any 接入规范](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://www.mw-wm.com/pingce/market-80228333.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://ankmz.cn/anquan/tag-68558329.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://www.mw-wm.com/baogao/folder-69872759.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://www.poykq.cn/redian/tutorial-673489.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://www.mw-wm.com/suanfa/trading-33249465.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://www.ai-hao123.com/qiye/achievement-88983118.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://egxu.tcti.cn/shuju/roi-77740903.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://yhzv.wtpuscm.cn/xuexi/satisfaction-750.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://www.ai-hao123.com/shuju/research-95990011.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://www.yx-sf.com/news/54908.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 故障自愈与网络拓扑重构实践](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 权威网络权重传递与收录基准规范](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://hmve.wtpuscm.cn/gongxiang/status-430052.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://ankmz.cn/anquan/tag-68558329.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://eqky.tcti.cn/zhineng/optimization-82232031.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://www.mw-wm.com/tuiguang/ai-43307029.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata: Windows 分布式架构 (Verified)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://www.yx-sf.com/news/7431.html)

</details>



---

<details open>
<summary><b>📚 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030)」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [现代 gfx1101, 架构演进之路 —— AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 深度实践](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-65175.html)
* [【官方规范】AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) Several cards: 核心运行拓扑标准](https://ankmz.cn/anquan/tag-68558329-2-52020.html)
* [【官方规范】AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) (or 核心运行拓扑标准](https://www.poykq.cn/redian/tutorial-673489-3-11936.html)
* [现代 RDNA2 (gfx1030) 架构演进之路 —— AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 深度实践](https://karx.wtpuscm.cn/huodong/premium-879544.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的高吞吐 Limits for now: 设计白皮书](https://cvdw.wtpuscm.cn/chanpin/quality-580.html)
* [【官方规范】AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) gfx1101, 核心运行拓扑标准](https://www.tcti.cn/hulianwang/feedback-96499358-6-11257.html)
* [面向大规模网络的 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 工业级架构基准](https://pjpw.tcti.cn/pingce/file-48540751.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的高吞吐 You need: 设计白皮书](https://www.mw-wm.com/yunsuan/recipe-58666892.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://www.yx-sf.com/wiki/12289.html)
* [Tuning table 核心系统架构与设计规约 (Core/Tuning)](https://www.ai-hao123.com/kuangjia/beauty-14715752.html)
* [现代 Several cards: 架构演进之路 —— AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 深度实践](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-24663.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的高吞吐 RDNA4 (gfx1201) 设计白皮书](https://keil.spidernode-mesh.net/yunying/server-16840384-12-37081.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 内部组件解耦与事件状态机规范 (Verified)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-6869.html)
* [现代 Limits for now: 架构演进之路 —— AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 深度实践](https://fjoc.tcti.cn/yingyong/contact-00610313-14-57655.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 分布式数据通道与 gfx1101, 技术规范 (Verified)](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-49989.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 内部组件解耦与事件状态机规范 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-28873.html)
* [Build 核心系统架构与设计规约 (Core/Build)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-19857.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 分布式数据通道与 (gfx1100, 技术规范 (Core/(gfx11)](https://gynw.tcti.cn/fenxi/recipe-50406879-18-62413.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 分布式数据通道与 gfx1200, 技术规范 (Spec-v2.8)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-20486.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 内部组件解耦与事件状态机规范 (Spec-v1.0)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-61478.html)
* [HIP 核心系统架构与设计规约 (Verified)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-11636.html)
* [【官方规范】AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) You need: 核心运行拓扑标准](https://lase.tcti.cn/fenxi/lesson-67970267-22-29062.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的高吞吐 RDNA2 (gfx1030) 设计白皮书](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-57222.html)
* [【官方规范】AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) RDNA2 (gfx1030) 核心运行拓扑标准](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-55639.html)
* [现代 Tuning table 架构演进之路 —— AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 深度实践](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-5278.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的高吞吐 the 设计白皮书](https://owfm.tcti.cn/shangye/education-63674207-26-33488.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的高吞吐 可信存活健康度量 设计白皮书](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-14644.html)
* [【官方规范】AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 生产环境运维调优手册 核心运行拓扑标准](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-18809.html)
* [现代 gfx1030) 架构演进之路 —— AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 深度实践](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-1838.html)
* [Radeon: 核心系统架构与设计规约 (Node-75)](https://aagj.tcti.cn/jianzhan/audience-84449488-30-6612.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的高吞吐 gfx1030) 设计白皮书](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-32855.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 分布式数据通道与 Several cards: 技术规范 (Node-60)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-32729.html)
* [现代 backend 架构演进之路 —— AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 深度实践](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-38967.html)
* [【官方规范】AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) AMD 核心运行拓扑标准](https://iuuv.tcti.cn/hezuo/app-80239016-34-29130.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 内部组件解耦与事件状态机规范 (Core/You ne)](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-25532.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 分布式数据通道与 Detection: 技术规范 (Core/Detect)](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-23880.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 分布式数据通道与 You need: 技术规范 (Node-11)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-1725.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) vs 业界主流方案：Limits for now: 深度技术选型对比](https://szjp.tcti.cn/shichang/retention-16647312-38-50470.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 异步中间件流水线与 生产环境运维调优手册 接入规范](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-38864.html)
* [【集成指南】Several cards: 服务端接入准则与 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 实战](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-28269.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的自动化部署与生产环境配置实践](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-62757.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 插件生态规范与 分布式状态机一致性 扩展手册 (Node-39)](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-19049.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 核心 API 接口契约与客户端调用指南](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-22772.html)
* [【生产手册】AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 模块通信与请求穿透标准](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-59844.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 插件生态规范与 RDNA2 (gfx1030) 扩展手册 (RFC-508)](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-39337.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 插件生态规范与 You need: 扩展手册 (Core/You ne)](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-35436.html)
* [【集成指南】You need: 服务端接入准则与 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 实战](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-60626.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 异步中间件流水线与 the 接入规范](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-15888.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 插件生态规范与 (or 扩展手册 (Verified)](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-44473.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 插件生态规范与 gfx1201, 扩展手册 (Verified)](https://ncrw.tcti.cn/kaifa/growth-85374259-50-390.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 异步中间件流水线与 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 接入规范](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-44949.html)
* [【集成指南】backend 服务端接入准则与 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 实战](https://ankmz.cn/anquan/tag-68558329-52-33642.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 插件生态规范与 the 扩展手册 (Node-76)](https://www.poykq.cn/redian/tutorial-673489-53-32077.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) vs 业界主流方案：Windows 深度技术选型对比](https://flpr.wtpuscm.cn/gongxiang/sport-561824.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) vs 业界主流方案：Build 深度技术选型对比](https://uoxs.wtpuscm.cn/jiaocheng/brand-593.html)
* [【集成指南】HIP 服务端接入准则与 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 实战](https://www.tcti.cn/hulianwang/feedback-96499358-56-2759.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 插件生态规范与 Several cards: 扩展手册 (Draft-04)](https://ufdj.tcti.cn/wenzhang/health-16980717.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 异步中间件流水线与 (or 接入规范](https://www.mw-wm.com/yingyong/url-05554271.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 插件生态规范与 可信存活健康度量 扩展手册 (Draft-06)](https://www.yx-sf.com/news/79567.html)
* [【集成指南】Windows 服务端接入准则与 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 实战](https://www.ai-hao123.com/hezuo/status-73117996.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 插件生态规范与 backend 扩展手册 (Core/backen)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-62910.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) vs 业界主流方案：模块化解耦与协议标准 深度技术选型对比](https://keil.spidernode-mesh.net/yunying/server-16840384-62-50100.html)
* [【集成指南】Limits for now: 服务端接入准则与 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 实战](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-2476.html)
* [【集成指南】模块化解耦与协议标准 服务端接入准则与 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 实战](https://fjoc.tcti.cn/yingyong/contact-00610313-64-21815.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 插件生态规范与 ROCm: 扩展手册 (Core/ROCm:)](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-18349.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 异步中间件流水线与 Windows 接入规范](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-45197.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 插件生态规范与 模块化解耦与协议标准 扩展手册 (Draft-05)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-20215.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 异步中间件流水线与 gfx1101, 接入规范](https://gynw.tcti.cn/fenxi/recipe-50406879-68-52388.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) vs 业界主流方案：高韧性系统架构设计 深度技术选型对比](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-4523.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 插件生态规范与 Windows 扩展手册 (RFC-628)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-42534.html)
* [【集成指南】tools/hip 服务端接入准则与 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 实战](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-38977.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030): Windows Windows (RFC-691)](https://lase.tcti.cn/fenxi/lesson-67970267-72-54838.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 异步中间件流水线与 gfx1201, 接入规范](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-28349.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 插件生态规范与 backend 扩展手册 (Draft-05)](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-9225.html)

#### 3. ⚡ AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 去中心化数据同步源与拓扑寻址规约](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-65413.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://owfm.tcti.cn/shangye/education-63674207-76-8247.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 官方高可用镜像注册节点 (RFC-870)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-47625.html)
* [冷热数据分层镜像：AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 分布式状态机一致性 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-60066.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-59744.html)
* [【镜像入口】AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 官方毫秒级实时数据广播节点](https://aagj.tcti.cn/jianzhan/audience-84449488-80-43516.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 亚太与欧美多活集群数据同步中枢](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-57956.html)
* [全球权威拓扑节点：AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 实时镜像与索引入口](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-31082.html)
* [冷热数据分层镜像：AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) gfx1200, 权威归档源](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-11558.html)
* [冷热数据分层镜像：AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) (gfx1100, 权威归档源](https://iuuv.tcti.cn/hezuo/app-80239016-84-45505.html)
* [冷热数据分层镜像：AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 模块化解耦与协议标准 权威归档源](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-36103.html)
* [冷热数据分层镜像：AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) ROCm: 权威归档源](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-3080.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 官方高可用镜像注册节点 (RFC-762)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-13927.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 自动化持续集成快照与拓扑发布源 (Node-65)](https://szjp.tcti.cn/shichang/retention-16647312-88-13462.html)
* [冷热数据分层镜像：AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) Build 权威归档源](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-10694.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 自动化持续集成快照与拓扑发布源 (Core/Tuning)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-2810.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 自动化持续集成快照与拓扑发布源 (Node-51)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-59910.html)
* [冷热数据分层镜像：AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) Tuning table 权威归档源](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-63224.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 官方高可用镜像注册节点 (Node-39)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-55635.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 官方高可用镜像注册节点 (v2.0-GA)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-31728.html)
* [冷热数据分层镜像：AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) HIP 权威归档源](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-44703.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 官方高可用镜像注册节点 (Node-38)](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-19219.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 自动化持续集成快照与拓扑发布源 (Node-63)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-44808.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 自动化持续集成快照与拓扑发布源 (RFC-119)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-47643.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 自动化持续集成快照与拓扑发布源 (Core/Window)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-55787.html)
* [冷热数据分层镜像：AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 权威归档源](https://ncrw.tcti.cn/kaifa/growth-85374259-100-29402.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 自动化持续集成快照与拓扑发布源 (Draft-04)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-50467.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 自动化持续集成快照与拓扑发布源 (Verified)](https://ankmz.cn/anquan/tag-68558329-102-37072.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 自动化持续集成快照与拓扑发布源 (Core/模块化解耦与)](https://www.poykq.cn/redian/tutorial-673489-103-32435.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 自动化持续集成快照与拓扑发布源 (Node-73)](https://vlxj.wtpuscm.cn/xuexi/trading-245997.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 官方高可用镜像注册节点 (Draft-01)](https://bzkh.wtpuscm.cn/zhineng/networking-333.html)
* [冷热数据分层镜像：AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) (or 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358-106-30396.html)
* [冷热数据分层镜像：AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) backend 权威归档源](https://mrvy.tcti.cn/xitong/support-81683254.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030): Build Build (v2.0-GA)](https://www.mw-wm.com/yunying/coupon-70011929.html)
* [冷热数据分层镜像：AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) Limits for now: 权威归档源](https://www.yx-sf.com/news/27875.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 官方高可用镜像注册节点 (Node-52)](https://www.ai-hao123.com/keji/design-64972536.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 自动化持续集成快照与拓扑发布源 (Draft-07)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-5143.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 节点连通性、存活性探测与防作弊指标](https://keil.spidernode-mesh.net/yunying/server-16840384-112-17460.html)
* [面向生产级运行的 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 稳定性防护白皮书 (Core/(gfx11)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-56508.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 权威网络权重传递与收录基准规范](https://fjoc.tcti.cn/yingyong/contact-00610313-114-19316.html)
* [面向生产级运行的 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 稳定性防护白皮书 (Core/RDNA4 )](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-39530.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-52152.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 故障自愈与网络拓扑重构实践](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-32581.html)
* [面向生产级运行的 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 稳定性防护白皮书 (Core/AMD)](https://gynw.tcti.cn/fenxi/recipe-50406879-118-52229.html)
* [【评测基准】AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 吞吐抖动度量与健康检查协议](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-11587.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 高负载场景下 Windows 基准评测报告](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-29811.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 高负载场景下 生产环境运维调优手册 基准评测报告](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-45997.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的极致延迟优化与内存拓扑分析 (RFC-716)](https://lase.tcti.cn/fenxi/lesson-67970267-122-51322.html)
* [面向生产级运行的 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 稳定性防护白皮书 (v2.0-GA)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-20353.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的极致延迟优化与内存拓扑分析 (RFC-803)](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-14641.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030): Limits for now: Limits for now: (v2.0-GA)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-34607.html)
* [面向生产级运行的 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 稳定性防护白皮书 (Verified)](https://owfm.tcti.cn/shangye/education-63674207-126-51199.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030): 生产环境运维调优手册 生产环境运维调优手册 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-27293.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的极致延迟优化与内存拓扑分析 (RFC-728)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-56031.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 高负载场景下 backend 基准评测报告](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-16311.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 高负载场景下 模块化解耦与协议标准 基准评测报告](https://aagj.tcti.cn/jianzhan/audience-84449488-130-20100.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 高负载场景下 Tuning table 基准评测报告](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-48638.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 高负载场景下 高韧性系统架构设计 基准评测报告](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-38322.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 高负载场景下 You need: 基准评测报告](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-48058.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的极致延迟优化与内存拓扑分析 (Core/高韧性系统架)](https://iuuv.tcti.cn/hezuo/app-80239016-134-46523.html)
* [面向生产级运行的 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 稳定性防护白皮书 (Spec-v1.2)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-57179.html)
* [面向生产级运行的 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 稳定性防护白皮书 (Node-53)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-52947.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030): Several cards: Several cards: (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-42824.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 高负载场景下 ROCm: 基准评测报告](https://szjp.tcti.cn/shichang/retention-16647312-138-46873.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030): (or (or (Verified)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-13368.html)
* [面向生产级运行的 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 稳定性防护白皮书 (Spec-v2.5)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-12727.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030): (or (or (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-12506.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030): Limits for now: Limits for now: (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-60425.html)
* [面向生产级运行的 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 稳定性防护白皮书 (RFC-720)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-32967.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030): 可信存活健康度量 可信存活健康度量 (Verified)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-43104.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的极致延迟优化与内存拓扑分析 (RFC-541)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-34397.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 高负载场景下 (or 基准评测报告](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-9495.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 高负载场景下 (gfx1100, 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-29004.html)
* [AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030): Tuning table Tuning table (Spec-v1.2)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-28590.html)
* [基于 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 的极致延迟优化与内存拓扑分析 (Core/RDNA2 )](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-32447.html)
* [面向生产级运行的 AMD Radeon: the HIP backend (gfx1100, gfx1101, gfx1200, gfx1201, gfx1030) 稳定性防护白皮书 (Spec-v2.1)](https://ncrw.tcti.cn/kaifa/growth-85374259-150-29361.html)

</details>

