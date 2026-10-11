# Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D

Measured on 2026-10-06 by [jase100k](https://keil.spidernode-mesh.net/yunying/server-16840384.html) on a NixOS desktop. This tests the
Coder model (IQ1_M) with the layer split across two RX 7900 GRE cards on engine 0.1.40, and the four
things the 0.1.40 notes asked a two-card rig to check: `--pipeline-windows` (#859), resident RAM mode
on a split (#848), `--batch` (#776) and the stage-weights trim (#880).

The main limitation is that the two cards are not on the same link. One is wired straight to a
16.0 GT/s x16 root port (the engine probes **28.2 GB/s** host to device, `pcie_frac 0.55`); the other
sits behind a chain of bridges that settles at 8.0 GT/s x4 (**3.1 GB/s**, `pcie_frac 0.08`). Layers
0-26 run on the fast card, layers 27-47 on the slow one, so every number below is for that lopsided
pair. Memory is reported in GiB, the engine's link and transfer rates in GB/s as it prints them.

Median decode throughput was **69.3 tok/s with `--pipeline-windows 2` and 63.8 tok/s with
`--pipeline-windows 1`** over 10 interleaved pairs of 500-token greedy generations (1,757-4,238 fresh
prompt tokens each): the one-window arm was slower in 9 of 10 pairs, the paired change median
**-7.2%** (range -17.8% to +0.7%). With the stage trim off (`STRATA_STAGE_TRIM=0`) the fresh-prompt
decode median fell from 75.7 to 67.6 tok/s and fresh prefill from 1,582.9 to 1,103.4 tok/s, with
7,624 resident experts instead of 9,438. Resident RAM mode (`--resident-budget-gib 30`) landed
inside the baseline's own run-to-run spread: 72.1 tok/s fresh decode against 71.2-79.1 tok/s for the
same config measured as arm A of the other four arms. `--batch 2` and `--batch 4` load and run
without hanging, but filling the slots does not raise throughput: one request at a time on the batch
config decoded at 68.3 tok/s, two at once totalled 56.6 and four at once 64.6.

## Hardware and software

- **GPUs:** 2x AMD Radeon RX 7900 GRE, 16 GiB each (17,163,091,968 B of VRAM per card; PCI device
  `1002:744c`, gfx1100, wave32, HIP names the agents gfx1100). Subsystem IDs `1DA2:475E` (slot
  `0000:2d:00.0`) and `1EAE:790A` (slot `0000:06:00.0`). rocm-smi saw no maximum power limit without
  root; average package power at collection time (idle) was 29.0 W and 46.0 W.
- **PCIe:** `0000:2d:00.0` is on a clean path - root port `00:03.1` through `2b:00.0` and `2c:00.0`,
  every hop 16.0 GT/s x16, and the engine probes **28.2 GB/s** host to device (`pcie_frac 0.55`). The
  other card, `0000:06:00.0`, goes `00:01.2` (8.0 GT/s now, max 16.0 GT/s x8) then `02:00.2`,
  `03:00.0`, `04:00.0`, `05:00.0`, and the hops from `03:00.0` down run **8.0 GT/s x4**, which the
  engine measures as **3.1 GB/s** (`pcie_frac 0.08`). The endpoint's own link registers still report
  x16; the limit is the root-port hop. Extended control/status registers and `lspci` need root, so
  negotiated widths beyond what sysfs shows were not read.
- **Order:** the config's `"gpu": [1, 0]` makes the server set `HIP_VISIBLE_DEVICES=1,0`, so the
  engine's CUDA0 is HIP device 1 (`0000:2d:00.0`, the fast card) and CUDA1 is HIP device 0
  (`0000:06:00.0`, the slow card). Layers 0-26 on the fast card, 27-47 on the slow one.
- **CPU and RAM:** AMD Ryzen 7 5700X3D, 8 cores / 16 threads, AVX2 only (no AVX-512). 65,760,268 kB
  installed RAM = **62.7 GiB**. Swap: `/dev/zram0` 32,880,124 kB (31.4 GiB, priority 5) plus an
  unused NVMe partition of 72,347,404 kB (69.0 GiB, priority -1), 100.3 GiB in total.
- **Storage:** KINGSTON SKC3000D2048G, 1.9 TB NVMe (no rotating media). Model, pack and MTP files
  live on it.
- **OS and runtime:** NixOS 26.11 (Zokor), kernel `7.2.8-cachyos`, amdgpu. ROCm at
  `<local opt>/rocm` (a tree built by this machine's `make-rocm-tree.py` from nixpkgs ROCm packages,
  ROCm 7.2.3, HIP 7.2.53211, rocm-smi 4.0.0); `ROCM_PATH` and the loader paths come from
  `<local opt>/strata-env.sh`.
- **Source:** `main` at `1735d64` (pulled 2026-10-06 19:11), engine version 0.1.40, llama.cpp pinned
  at `3cf0325`.
- **Build:** a local HIP source build, not a release binary: `engine/BUILD.json` says
  `"source": "local-hip", "backend": "hip", "version": "0.1.40", "archs": ["gfx1100"]`, built with
  `cmake -DSTRATA_ENABLE_HIP=ON -DSTRATA_ENABLE_CUDA=OFF -DSTRATA_BUILD_TESTS=OFF
  -DSTRATA_PREFILL_MMQ=ON -DCMAKE_HIP_ARCHITECTURES=gfx1100 -DSTRATA_PORTABLE=OFF` (Release, ROCm
  clang); `engine/strata` is 11,419,856 B, mtime 2026-10-06 19:13, and `--version` reports 0.1.40.
  One bookkeeping wrinkle: the `src` fingerprint in `BUILD.json` is `d63ddbc6a262e39e`, which is the
  fingerprint `setup.py` computes for the tree at `6f32ec0` (the commit before the 19:11 pull), while
  the tree at run time (`1735d64`) fingerprints as `6f298dff754d187f`. `setup.py` hashes the source
  when the compile starts and stamps the version when it ends, so this build started before the pull
  and finished after it. The binary reports 0.1.40 and behaves like it: the `--pipeline-windows 2`
  start-up line carries the 0.1.40 clause and the two window settings print different lines and
  measure different speeds (Results below).
- **Background workloads:** a normal desktop session, not a dedicated bench box. No other GPU client
  was known to be running (amdgpu reported 1-2% busy at collection time); CPU-side services and the
  desktop kept running. Nothing was clock-locked and no power limit was set. GPU temperatures at
  collection: edge 32 / 44 C, junction 39 / 52 C, memory 54 / 66 C.

## Model and configuration

- **Model:** [`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-Coder-GGUF`](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html),
  IQ1_M, 48 layers / 5,376 routed experts (the profile counts 12,288 ranked pairs). The repository
  revision was not recorded; the files were downloaded 2026-09-30 20:29-20:36:
  - `Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00001-of-00002.gguf` 29,608,446,496 B (29.6 GB / 27.6 GiB),
    passed as `--native`;
  - `Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00002-of-00002.gguf` 28,800,138,432 B (28.8 GB / 26.8 GiB),
    passed as `--ple-gguf`.
- **Pack:** `<data root>/packs/coder-iq1_m`, built with the repository's own tool as `index.txt` says:
  `# strata pack index v3 -- generated by tools/iq_pack.py (native experts) from
  Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00001-of-00002.gguf` (sha256 `2cb3352786bcae81...`; `dense.bin`
  1,475,120,640 B, `experts.bin` 25,146,163,200 B, `native_experts.txt` sha256
  `9ace1b92e1fe1259...`).
- **Expert profile:** `expert-profile-learned-coder.bin` (98,328 B, sha256
  `be2c27b6d2267470162d9ebadeac8f3e2d36c4f70399cea04d51db6fef78d636` as it stood after the last run).
  It is a *learned* profile: passed as `--expert-profile` at start and rewritten by
  `--expert-profile-save` while the server runs, so it is both an input to and an output of these
  measurements. The profile ranks 12,288 pairs for 12,288 slots.
- **MTP draft:** `<data root>/mtp/rt` (`dense.bin` sha256 `c724dc0b0822ada5...`, `experts.bin`
  sha256 `09398406be61f1f5...`, `draft_vocab.bin` sha256 `369151522226a5ed...`), draft vocabulary
  `en`.
- **Vision:** `--vision` with `strata-vision` and `mmproj-Qwen3.8-Flash-Next-BF16.gguf`, encoder on
  the CPU (4 threads, `gpu: false`). No image was sent in this report, so the vision path is enabled
  but unmeasured.
- **Context and KV:** 262,144 tokens (`--max-context 262144`), `--kv int8 --kv-resident 32768`
  (32,768 KV cells per attention layer resident in VRAM, the rest streamed from RAM).
- **Split and caches:** `--layer-split 27` (layers 0-26 on CUDA0, 27-47 on CUDA1),
  `--expert-cache auto`, `--prefill auto`, `--vram-reserve-mib 3072 --vram-reserve-later-mib 800`.
- **Pipeline:** `--pipeline-windows 2` in the install config; one arm uses `--pipeline-windows 1`.
- **MTP and sampling:** `--spec 5 --spec-min-p 0.5`, suffix drafting on (the default). Every request
  in this report is greedy: `temperature 0, top_k 1, top_p 1, min_p 0, seed 42, reasoning_effort
  none` (`reasoning_budget_tokens` is 2048 in the config but no request asks for reasoning).
- **Environment:** `STRATA_HIPBLASLT_TUNING=<strata checkout>/tools/hip/gfx1100-hipblaslt-100202.txt`
  (sha256 `a594869223200c62...`), `STRATA_ADAPT_NOWAIT=1`, `STRATA_STAGE_TRIM=1`,
  `STRATA_PREFILL_HELP=1`, `STRATA_RING_BYTES=1`, `STRATA_SPLIT_OWN=1`, `STRATA_EXCHANGE_ROTATE=1`,
  `STRATA_PF_FUSED=1`.
- **Not used:** no calibration, no experimental speed projection, no control vectors, no API key,
  `fit_max_tokens` on.

```text
# the server (run-coder-iq1_m.sh); the config holds the model paths and the environment
. "<local opt>/strata-env.sh"
"<strata checkout>/.venv/bin/python" "<strata checkout>/serve/server.py" --engine strata \
  --config "<strata checkout>/strata-coder-iq1_m.json" --port 8080

# what the server starts (the "engine started" line in data/engine.log)
<strata checkout>/engine/strata --serve --pack <data root>/packs/coder-iq1_m \
  --native <data root>/models/coder-IQ1_M/Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00001-of-00002.gguf \
  --ple-gguf <data root>/models/coder-IQ1_M/Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00002-of-00002.gguf \
  --expert-profile expert-profile-learned-coder.bin --expert-cache auto --prefill auto \
  --spec 5 --spec-min-p 0.5 --mtp <data root>/mtp/rt --max-context 262144 --kv int8 \
  --kv-resident 32768 --vram-reserve-mib 3072 --vram-reserve-later-mib 800 \
  --pipeline-windows 2 --vision --layer-split 27 \
  --expert-profile-save expert-profile-learned-coder.bin
# with HIP_VISIBLE_DEVICES=1,0 (from "gpu": [1, 0]) and the eight environment variables above
```

## Method

Four kinds of measurement, run one after another on the same machine, each starting its own
server on port 8080 and killing the previous one (`scripts/run-all.sh` runs them in order:
[interleave500](scripts/interleave500.py), then the `pipeline-off`, `stage-trim-off`
and `resident` arms, then the soak, then the `batch2` and `batch4` arms). The A/B harness is
vendored into this folder as [scripts/ab-run.sh](scripts/ab-run.sh) - arm A is the install config as
it stands, arm B the same config plus the extra engine args/env - and is called through
[scripts/run-arm.sh](scripts/run-arm.sh), which strips the `--arg` marker the runner spells (the
harness takes bare flag/value pairs and would otherwise raise `IndexError` before writing arm B's
config) and archives each arm's JSON, server output and engine-log slice into `data/`. Because
`ab-run.sh` ends by starting a server on the base config, the next arm's log slice can begin
with one extra start-up block from that leftover server; the blocks are told apart by their expert
counts, and only the blocks that then serve requests belong to the arms.

- **`--pipeline-windows 2` vs 1, interleaved** ([`scripts/interleave500.py`](scripts/interleave500.py),
  results under "Per-pair detail"): 10 pairs, each pair posting the same prompt to arm A (the install
  config, window 2) and arm B (`--pipeline-windows 1`) back to back, alternating which arm goes first
  by pair, one server restart per request so neither arm keeps a warm cache the other does not get.
  `max_tokens=500`, greedy (`temperature 0, top_k 1, top_p 1, min_p 0, seed 42`), `reasoning_effort:
  none`, no reused tokens (every request is a fresh conversation). Prompt tokens 1,757, 2,334, 2,913,
  3,584 and 4,238, twice each per arm.
- **Short arms** ([`scripts/ab-run.sh`](scripts/ab-run.sh) + `tools/hip/bench_prefill.py`): one
  warm-up request
  (`Reply READY.`, excluded), then 4 fresh prompts (synthetic JavaScript of 140 and 280 lines,
  4,210 and 8,830 prompt tokens, `reused 0`) and 4 follow-ups on those same conversations
  (4,451 and 9,071 prompt tokens, of which 4,203-8,958 were reused), 128 generated tokens per
  request, same greedy settings. Each arm restarts the server for side A and again for side B, with
  a 30 s pause before the pair. `STRATA_STAGE_TRIM` is already `1` in the install config, so the
  stage-trim arm's *B* side sets it to `0`: A is "trim on", the default, and B is "trim off".
- **Soak:** one server for 30 minutes with `--resident-budget-gib 30`, one fresh conversation after
  another at 128 tokens, `scripts/sample_mem.py` writing a row every 60 s.
- **`--batch`:** the same short arm with `--batch 2` and `--batch 4` (single requests, so it shows
  that the config loads and that nothing hangs), then
  [`scripts/batch_concurrent.py`](scripts/batch_concurrent.py) posting N requests at once, which is
  what actually fills the batch slots (`tools/hip/bench_prefill.py` is single request).
- **Timing boundaries:** prompt tok/s and decode tok/s always come from the engine's own completion
  line (`prompt N tokens = X reused + Y read in Z ms (P tok/s), G generated in D ms (D tok/s)`), never
  from wall time divided by token count. TTFT is measured only in `interleave500.py`, client side over
  loopback, from just before the POST to the first non-empty streaming delta (empty deltas and
  keep-alives ignored); it is not measured in the short arms. Wall time is the whole request at the
  client, over loopback. Model loading is excluded: every side waits for
  `{"loaded": true}` on `/health` before its first request.
- **State between runs:** each side of each arm is a fresh server, so the expert cache, KV cache and
  CUDA context are cold at the start of every side; within a side, the expert cache is pre-filled from
  the profile at start and then keeps adapting (the engine prints its hit rate at the end of each
  request). Fresh prompts and follow-ups that reuse a prefix are always reported separately.
- **Memory:** `scripts/sample_mem.py` samples every 60 s: whole-machine RAM as
  `MemTotal - MemAvailable` (this counts page cache that is in use, not "anonymous memory only"), each
  card's `mem_info_vram_used` from amdgpu's sysfs counters, and the engine process's RSS. Values in
  the soak section are first sample and peak over those 60 s samples; a peak between samples can be
  missed.
- **Browser:** `ab-run.sh` passes `--open`, so `BROWSER=true` was exported to stop it opening a
  tab; no other desktop effect is known.

### Files

| File | What it holds |
| --- | --- |
| [data/hardware.txt](data/hardware.txt) | os-release, uname, rocminfo, rocm-smi, sysfs PCIe links, CPU/RAM/storage, engine version, PCI topology of both cards |
| [data/interleave500.json](data/interleave500.json) | the 20 interleave requests: arm, pair, prompt id, TTFT, wall, engine metrics, finish reason, output head |
| [data/interleave500-server.out](data/interleave500-server.out), [data/engine-interleave.log](data/engine-interleave.log) | the server output and the engine log slice for those 20 requests |
| [data/ab-\<label\>-A.json](data), [data/ab-\<label\>-B.json](data) | the short arms' per-request records (`pipeline-off`, `stage-trim-off`, `resident`, `batch2`, `batch4`) |
| [data/cfg-\<label\>-B.json](data) | the config each arm's B side actually ran (`cfg-soak.json` and `cfg-interleave-B.json` are the soak's and the interleave's), with private paths rewritten |
| [data/ab-\<label\>-run.out](data) | the harness's own printed A/B summary for each arm |
| [data/ab-\<label\>-server-A.out](data), [data/ab-\<label\>-server-B.out](data) | each side's server output (start-up decisions, `done` lines) |
| [data/engine-ab-\<label\>.log](data) | each arm's engine log slice (start-up decisions plus request lines) |
| [data/soak-mem.csv](data/soak-mem.csv), [data/soak.json](data/soak.json) | the 60 s memory samples and the soak's 239 per-prompt records |
| [data/soak-summary.json](data/soak-summary.json), [data/soak-engine.log](data/soak-engine.log), [data/soak-server.out](data/soak-server.out) | the rebuilt soak summary (survived, peaks, request counts), the engine slice it was rebuilt from, and the soak's server output |
| [data/run-all.out](data/run-all.out) | the console transcript of the whole serial run, including the soak's per-prompt lines |
| [data/batch-concurrent-batch\<n\>.json](data) | the N-concurrent batch measurements: per-slot TTFT, wall, finish, the server's `done` lines and the engine slice |
| [data/batch-concurrent-batch\<n\>.server.out](data), [data/batch-concurrent-batch\<n\>.engine.log](data), [data/run-batch-concurrent.out](data/run-batch-concurrent.out) | the server output, engine slice and console transcript of each concurrency run |
| [data/engine.log](data/engine.log) | start-up lines plus request lines for every step of this report |
| [data/summary.md](data/summary.md) | the machine-built median/range tables this README's numbers are taken from |
| [scripts/](scripts) | every script that produced the above, including the A/B harness (`ab-run.sh`) |

## Results

### All runs

Every row is a **median (minimum-maximum)** over the runs in that row; every run is in the linked
JSON. No request failed and no arm needed a retry. Prompt throughput is freshly read tokens per
second from the engine's own completion line; decode is generated tokens per that line's decode
time; TTFT is client side over loopback and is `not measured` outside the interleave. Warm-up
requests are excluded everywhere.

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median and range | Decode tok/s median and range | TTFT seconds median and range |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| `--pipeline-windows 2` (A), 500-token, interleaved | 1,757-4,238 | 0 | 256-500 (cap 500) | 10 | 961.0 (841.5-1,270.1) | **69.3 (66.4-74.3)** | 2.9 (1.9-3.6) |
| `--pipeline-windows 1` (B), 500-token, interleaved | 1,757-4,238 | 0 | 256-500 (cap 500) | 10 | 959.9 (839.6-1,271.9) | **63.8 (59.7-69.7)** | 2.9 (1.9-3.7) |
| baseline (A) fresh, `pipeline-off` arm | 4,210-8,830 | 0 | 128 | 4 | 1,582.8 (1,302.1-1,756.8) | 77.1 (70.9-83.9) | not measured |
| baseline (A) follow-up, `pipeline-off` arm | 4,451-9,071 | 4,203-8,958 | 128 | 4 | 185.8 (182.3-315.1) | 66.2 (60.0-73.2) | not measured |
| `--pipeline-windows 1` (B) fresh | 4,210-8,830 | 0 | 128 | 4 | 1,568.0 (1,174.4-1,738.6) | 70.0 (63.1-73.6) | not measured |
| `--pipeline-windows 1` (B) follow-up | 4,451-9,071 | 4,337-8,823 | 118-128 | 4 | 246.8 (182.7-313.1) | 62.8 (56.9-64.6) | not measured |
| baseline (A) fresh, `stage-trim-off` arm | 4,210-8,830 | 0 | 128 | 4 | 1,582.9 (1,229.7-1,751.3) | 75.7 (67.6-84.7) | not measured |
| baseline (A) follow-up, `stage-trim-off` arm | 4,451-9,071 | 4,337-8,957 | 128 | 4 | 186.1 (180.4-193.3) | 73.5 (68.7-79.1) | not measured |
| `STRATA_STAGE_TRIM=0` (B) fresh | 4,210-8,830 | 0 | 128 | 4 | 1,103.4 (883.2-1,213.6) | 67.6 (57.7-77.1) | not measured |
| `STRATA_STAGE_TRIM=0` (B) follow-up | 4,451-9,071 | 4,203-8,957 | 128 | 4 | 193.4 (132.3-254.3) | 70.9 (65.1-76.7) | not measured |
| baseline (A) fresh, `resident` arm | 4,210-8,830 | 0 | 76-128 | 4 | 1,558.5 (1,235.2-1,736.0) | 72.1 (63.9-78.5) | not measured |
| baseline (A) follow-up, `resident` arm | 4,451-9,071 | 4,203-8,958 | 128 | 4 | 186.6 (181.1-301.3) | 63.4 (61.3-70.1) | not measured |
| `--resident-budget-gib 30` (B) fresh | 4,210-8,830 | 0 | 128 | 4 | 1,550.0 (1,313.5-1,718.9) | 69.0 (58.0-72.1) | not measured |
| `--resident-budget-gib 30` (B) follow-up | 4,451-9,071 | 4,203-8,958 | 128 | 4 | 243.9 (179.6-312.5) | 64.6 (60.5-66.6) | not measured |
| baseline (A) fresh, `batch2` arm | 4,210-8,830 | 0 | 76-128 | 4 | 1,542.7 (1,301.4-1,712.5) | 79.1 (74.1-82.0) | not measured |
| baseline (A) follow-up, `batch2` arm | 4,398-9,071 | 4,203-8,958 | 128 | 4 | 227.3 (188.4-311.4) | 70.1 (62.3-71.3) | not measured |
| `--batch 2` (B) fresh, one request at a time | 4,210-8,830 | 0 | 128 | 4 | 1,404.0 (1,155.0-1,552.9) | 68.3 (66.2-69.4) | not measured |
| `--batch 2` (B) follow-up, one request at a time | 4,451-9,071 | 4,203-8,957 | 128 | 4 | 288.7 (174.1-300.7) | 62.8 (59.6-69.1) | not measured |
| baseline (A) fresh, `batch4` arm | 4,210-8,830 | 0 | 128 | 4 | 1,452.3 (956.0-1,798.1) | 71.2 (56.7-72.3) | not measured |
| baseline (A) follow-up, `batch4` arm | 4,451-9,071 | 4,338-8,823 | 127-128 | 4 | 213.9 (141.3-298.2) | 61.0 (57.8-68.0) | not measured |
| `--batch 4` (B) fresh, one request at a time | 4,210-8,830 | 0 | 128 | 4 | 1,117.7 (854.2-1,238.9) | 68.5 (66.0-74.9) | not measured |
| `--batch 4` (B) follow-up, one request at a time | 4,451-9,071 | 4,337-8,957 | 128 | 4 | 147.1 (144.7-256.3) | 60.9 (59.0-67.6) | not measured |

Four runs stopped before the 128-token cap (76, 76, 118 and 127 generated tokens: two in arm A
follow-ups of the `resident` and `batch2` arms, one in the pipeline arm's B follow-up and one in the
`batch4` arm's A follow-up) and are published as they are; their prompts are the reason those rows'
token counts differ slightly. The same baseline config was measured as the A side of all five arms:
fresh decode medians 77.1, 75.7, 72.1, 79.1 and 71.2 tok/s, a **71.2-79.1 tok/s spread on one
configuration**, which is the noise floor for any arm-to-arm difference discussed below.

### Soak: 30 minutes with `--resident-budget-gib 30`

[data/soak-summary.json](data/soak-summary.json), [data/soak-mem.csv](data/soak-mem.csv) and
[data/soak-engine.log](data/soak-engine.log):

- **survived: yes.** 239 prompts sent over 30 minutes (one fresh conversation each, 128 tokens,
  greedy), **239 completed, 0 errors**, all `finish_reason: length`. Request wall time 2.02 / 2.55 /
  3.23 s (min / median / max). No `no progress`, NaN, crash, `never rang`, doorbell, abort, OOM or
  "killed" line appears in the soak's engine-log slice.
- **Samples:** 31 rows spanning 1,800.2 s, one every 60 s.
- **RAM** (`MemTotal - MemAvailable`, whole machine, includes page cache in use): first sample
  **22,713.6 MiB**, peak **24,009.0 MiB** (23.4 GiB) of 62.7 GiB installed.
- **Engine RSS:** first sample **10,810.3 MiB**, peak **11,180.1 MiB** (10.9 GiB).
- **VRAM** (amdgpu `mem_info_vram_used`, whole card): `0000:06:00.0` first 15.004 GiB, peak
  **15.054 GiB** of 16.0; `0000:2d:00.0` first 14.314 GiB, peak **14.440 GiB** of 16.0. These are
  60 s samples; a peak between samples can be missed.
- Resident RAM mode was active for the whole soak: `resident RAM mode: 5.38 GiB of experts in RAM
  (page-locked), 4648 in the GPU cache; adaptive swaps exchange them with the GPU cache (no file
  reads)`, 9,413 resident experts on the token-graph hit path, 77% of the experts resident, and the
  decode expert-cache hit rate went from 95.9% on the first completed request to 97.2% on the last.
- **The harness's teardown failed, not the soak:** `sample_mem.py` looks for its stop file at the top
  of its loop, so it only noticed one full 60 s interval later, and `soak.py` waited 30 s - its
  `subprocess.TimeoutExpired` is the one traceback in [data/run-all.out](data/run-all.out). The
  samples, the per-prompt records and the engine slice were all written before that point;
  [scripts/soak_summary.py](scripts/soak_summary.py) rebuilt `soak-summary.json` from them, and
  [scripts/soak.py](scripts/soak.py) now waits out one whole interval so a repeat finishes cleanly.

### Per-pair detail: `--pipeline-windows 2` vs `--pipeline-windows 1`

The same prompt to both arms, one server restart per request, A always first in the pair:

| Pair | Prompt tokens | Generated A / B | Decode A tok/s | Decode B tok/s | B vs A |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 1 | 1,757 | 409 / 414 | 66.4 | 60.5 | -8.9% |
| 2 | 2,334 | 500 / 500 | 72.6 | 59.7 | -17.8% |
| 3 | 2,913 | 500 / 500 | 67.8 | 68.3 | +0.7% |
| 4 | 3,584 | 500 / 500 | 68.8 | 68.1 | -1.0% |
| 5 | 4,238 | 500 / 256 | 73.2 | 63.3 | -13.5% |
| 6 | 1,757 | 336 / 258 | 67.4 | 63.5 | -5.8% |
| 7 | 2,334 | 500 / 500 | 66.9 | 63.8 | -4.6% |
| 8 | 2,913 | 500 / 500 | 71.0 | 69.7 | -1.8% |
| 9 | 3,584 | 500 / 500 | 74.3 | 64.6 | -13.1% |
| 10 | 4,238 | 500 / 500 | 69.9 | 63.9 | -8.6% |

Prompt tokens are identical within a pair, which is what makes the pairing useful. Decode is greedy
but not bit-reproducible across server restarts: prompt 0 generated 409 tokens in pair 1 and 336 in
pair 6 under arm A, and in 3 of the 10 pairs the two arms stopped at different token counts (pairs 1,
5 and 6). Those short runs are included, not thrown away.

### What the engine printed

- `--pipeline-windows 2: two verifiers per stage, the stages overlap across windows (decode too; the
  GDN snapshots beside the window)`
- `--pipeline-windows 1: two verifiers per stage, the stages overlap across windows`

The two settings print different start-up lines, so 0.1.40 does act on the flag. The install config
already carried `--pipeline-windows 2` under 0.1.39, where it was ignored; this report does not
compare versions, only this setting inside 0.1.40.

### Stage weights (#880)

| | `STRATA_STAGE_TRIM=1` (default, A) | `STRATA_STAGE_TRIM=0` (B) |
| --- | --- | --- |
| CUDA0 expert cache | 4,673 slots, 8.46 GiB | 3,832 slots, 6.93 GiB |
| CUDA1 expert cache | 4,765 slots, 9.62 GiB | 3,792 slots, 7.67 GiB |
| resident experts at start | **9,438** | **7,624** |
| engine's own share line | `77% of the experts resident` | not printed |
| CUDA1 free before the cache fill | 12.38 GiB of 15.98 | 10.43 GiB of 15.98 |
| free with everything loaded | 3,192 MiB | 3,170 MiB |
| decode expert-cache hit rate | 93.6-99.2% | 87.9-97.6% |
| fresh prefill median | 1,582.9 tok/s | 1,103.4 tok/s (-30.3%) |
| fresh decode median | 75.7 tok/s | 67.6 tok/s (-10.7%) |

Both sides print `CUDA0 loads the dense weights of layers 0-26 only` and
`CUDA1 loads the dense weights of layers 27-47 only`; what the trim removes is 1,814 resident experts
(a little over 2 GiB of slots across both cards), and the slow card pays for every expert it then
has to fetch.

### `--batch` (#776): slot sessions, real concurrency, doorbells

Arm B adds `--batch 2` or `--batch 4`, and the engine switches the pipeline window off by itself:

- `--pipeline-windows 2 is off: not with --batch slots`
- `--batch: 4 slot sessions on CUDA0 (0.52 GiB each); 11.16 GiB free` and the same on CUDA1
  (0.50 GiB each), `--batch 4: the slot sessions take 2.08 GiB of VRAM on CUDA0 that the expert
  cache would otherwise hold` (2 slots: 1.04 GiB)
- `strata verify: batch windows of up to 4 sequences (layers [0, 27))` and `(layers [27, 48))`

So the `--batch` rows in the table are the batch config *as a whole* - slot sessions taking VRAM
away from the expert cache **and** the window overlap switched off - not batch alone. Against the
baseline, fresh prefill fell from 1,582.8 to 1,404.0 (`--batch 2`) and 1,117.7 tok/s (`--batch 4`),
and fresh decode from 77.1 to 68.3 and 68.5 tok/s. All 36 runs of the two batch arms completed (9
requests per side, two sides, two arms) with no error and no stage waiting.

**Concurrency** ([`scripts/batch_concurrent.py`](scripts/batch_concurrent.py)): N fresh 500-token
greedy conversations posted at the same instant, per-request TTFT and wall time measured at the
client, decode taken from the server's own `[strata] done: N tokens in T s (X tok/s)` line (the
engine's completion line is one per prompt *chunk* under `--batch`, so it cannot be summed). Files:
[data/batch-concurrent-batch2.json](data/batch-concurrent-batch2.json),
[data/batch-concurrent-batch4.json](data/batch-concurrent-batch4.json), each with its
`.server.out` and `.engine.log` slice.

| Run | Slots filled | TTFT seconds, by slot | Wall seconds, by slot | Decode tok/s, by slot | Aggregate decode tok/s | Hung slots |
| --- | ---: | --- | --- | --- | ---: | ---: |
| `--batch 2`, one request at a time (the arm above, 128-token runs) | 1 | not measured | not measured | 68.3 (66.2-69.4) | 68.3 | 0 |
| `--batch 2`, 2 at once | 2 | 2.33, 6.26 | 19.13, 21.69 | 24.2, 32.4 | **56.6** | 0 |
| `--batch 4`, 4 at once | 4 | 3.22, 8.15, 13.36, 18.68 | 37.85, 42.03, 43.15, 43.85 | 13.1, 14.8, 16.8, 19.9 | **64.6** | 0 |

- **No doorbell was left ringing.** Every slot finished inside the 300 s per-slot timeout (worst
  43.9 s), `hung_slots` is empty in both JSONs, and neither engine slice contains a `no progress`,
  `never rang`, stall, abort, NaN or OOM line. The windows visibly form: `strata batch: slot 3 takes
  3584 tokens (copied in 111.7 ms)`, `strata verify: captured the batch window over slots 0,1,2,3`.
- **The slots did not buy throughput here.** One request at a time on the same config decoded at
  68.3 tok/s; two at once totalled 56.6 and four at once 64.6 (the sum of the per-slot rates), so
  the batch kept the machine near a single request's output rate while each request fell to
  24.2-32.4 tok/s with 2 slots (2.4x slower) and 13.1-19.9 tok/s with 4 (4.2x slower). TTFT also
  stacks, because prompts are admitted in turn: the fourth slot waited 18.7 s for its first token
  (`strata batch: the prompt was read in 2 parts, the slots decoding 1034 ms between them`).
- Caveats: the single-request reference row is from the arm's 128-token runs while the concurrent
  rows are 500-token runs, so it is a rate comparison and not a latency one, and it is one machine
  and one quantization, as the limitations below say.

## Correctness and limitations

- **No answer-quality check was run.** This report is speed and stability only: no `needle_bench.py`,
  no coding task with tests, no vision input. Nothing here says anything about answer quality.
- **Greedy is not bit-reproducible across restarts.** Same config, same prompt, same seed gave 409
  then 336 generated tokens for prompt 0 under arm A, and 3 of the 10 window pairs stopped at
  different points. Decode tok/s still comes from the engine's own decode timing, so the short runs
  are comparable, but they are noisier than the runs that hit the 500-token cap.
- **One machine, one quantization, one layer split, small synthetic prompts.** The longest prompt
  measured fresh is 4,238 tokens and the longest context used is that; the 262,144-token context,
  128,000-token prompts, vision, tool use, concurrency beyond the batch runs, thinking and sampled
  decoding were not tested. The two cards differ by nearly a factor of nine in measured link
  bandwidth, so a machine with two identical links would show different numbers.
- **The baseline was measured five times** (once as the A side of each arm) and its fresh-decode
  medians were 77.1, 75.7, 72.1, 79.1 and 71.2 tok/s (71.2-79.1), plus 69.3 in the interleaved
  500-token runs, which are a different workload. Differences smaller than that spread are not
  treated as effects here.
- **The expert profile changes while the server runs** (`--expert-profile-save`), so the arms do not
  see byte-identical profiles; each side starts from the file the previous side left behind.
- **Model revision unknown.** The GGUF files' repository revision was not recorded, only their names,
  sizes and download time, so a later revision of the same repository could differ.
- **Suggested comparison on this machine:** none. There is no second version, second card or second
  quantization measured here to compare against, and the report rules forbid a version-to-version
  claim from a single configuration.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://www.mw-wm.com/peixun/system-91878500.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://www.ai-hao123.com/zixun/trading-73950548.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [面向大规模网络的 Strata 工业级架构基准](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://zbdx.tcti.cn/peixun/login-94319677.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://bncd.wtpuscm.cn/gongsi/status-086701.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://cqea.tcti.cn/shuju/consulting-59426808.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://inee.wtpuscm.cn/jiaoliu/seo-932.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://www.mw-wm.com/pingtai/privacy-55592921.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://ankmz.cn/anquan/tag-68558329.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.ai-hao123.com/shangye/course-80533553.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 异步中间件流水线与 any 接入规范](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://ankmz.cn/anquan/tag-68558329.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://sxwc.wtpuscm.cn/baogao/recommendation-879.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://www.yx-sf.com/wiki/99048.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://qcba.wtpuscm.cn/sheji/backup-629.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://www.ai-hao123.com/gongxiang/website-48100160.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://www.ai-hao123.com/fenxi/metric-01470285.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://wyow.wtpuscm.cn/gongxiang/internet-612235.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://www.mw-wm.com/pingce/support-24906057.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://ipod.wtpuscm.cn/kuangjia/interface-117879.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 故障自愈与网络拓扑重构实践](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 权威网络权重传递与收录基准规范](https://owfm.tcti.cn/shangye/education-63674207.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://www.ai-hao123.com/youhua/plugin-82119310.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://www.poykq.cn/redian/tutorial-673489.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://www.mw-wm.com/yingyong/screen-28857819.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata: Windows 分布式架构 (Verified)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://kuvc.wtpuscm.cn/gongju/workshop-982.html)

</details>



---

<details open>
<summary><b>📚 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [Hardware and software 核心系统架构与设计规约 (Core/Hardwa)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-62800.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的高吞吐 What the engine printed 设计白皮书](https://ankmz.cn/anquan/tag-68558329-2-18225.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 内部组件解耦与事件状态机规范 (RFC-478)](https://www.poykq.cn/redian/tutorial-673489-3-38198.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的高吞吐 Method 设计白皮书](https://dgon.wtpuscm.cn/ziyuan/dashboard-298044.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 分布式数据通道与 PCIe: 技术规范 (Node-42)](https://kkwi.wtpuscm.cn/jiaocheng/partner-538.html)
* [现代 What the engine printed 架构演进之路 —— Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 深度实践](https://www.tcti.cn/hulianwang/feedback-96499358-6-1373.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 内部组件解耦与事件状态机规范 (Spec-v2.3)](https://vpew.tcti.cn/paiming/dashboard-32836464.html)
* [GRE 核心系统架构与设计规约 (Draft-06)](https://www.mw-wm.com/paiming/workshop-61336693.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 内部组件解耦与事件状态机规范 (RFC-684)](https://www.yx-sf.com/news/88217.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 内部组件解耦与事件状态机规范 (RFC-649)](https://www.ai-hao123.com/huodong/rating-48314763.html)
* [面向大规模网络的 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 工业级架构基准](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-54585.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Node-86)](https://keil.spidernode-mesh.net/yunying/server-16840384-12-13215.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的高吞吐 高韧性系统架构设计 设计白皮书](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-19725.html)
* [【官方规范】Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D Community 核心运行拓扑标准](https://fjoc.tcti.cn/yingyong/contact-00610313-14-33691.html)
* [【官方规范】Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D Method 核心运行拓扑标准](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-56873.html)
* [【官方规范】Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 3.1 GB/s 核心运行拓扑标准](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-45732.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 核心系统架构与设计规约 (v2.0-GA)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-23332.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 分布式数据通道与 PCIe: 技术规范 (Draft-07)](https://gynw.tcti.cn/fenxi/recipe-50406879-18-13696.html)
* [What the engine printed 核心系统架构与设计规约 (RFC-637)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-2820.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的高吞吐 分布式状态机一致性 设计白皮书](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-26999.html)
* [benchmark: 核心系统架构与设计规约 (Spec-v2.7)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-10040.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的高吞吐 All runs 设计白皮书](https://lase.tcti.cn/fenxi/lesson-67970267-22-24668.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 内部组件解耦与事件状态机规范 (Verified)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-13262.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 内部组件解耦与事件状态机规范 (Spec-v1.1)](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-53567.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 内部组件解耦与事件状态机规范 (Node-37)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-46500.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 分布式数据通道与 7900 技术规范 (Draft-03)](https://owfm.tcti.cn/shangye/education-63674207-26-57851.html)
* [GPUs: 核心系统架构与设计规约 (Draft-07)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-53951.html)
* [现代 7900 架构演进之路 —— Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 深度实践](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-18247.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 内部组件解耦与事件状态机规范 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-53984.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Spec-v1.7)](https://aagj.tcti.cn/jianzhan/audience-84449488-30-39396.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 分布式数据通道与 Results 技术规范 (Spec-v1.3)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-19915.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 内部组件解耦与事件状态机规范 (Spec-v1.7)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-22113.html)
* [【官方规范】Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D All runs 核心运行拓扑标准](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-45409.html)
* [【官方规范】Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D PCIe: 核心运行拓扑标准](https://iuuv.tcti.cn/hezuo/app-80239016-34-26007.html)
* [All runs 核心系统架构与设计规约 (Core/All ru)](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-8601.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 分布式数据通道与 Results 技术规范 (Spec-v2.7)](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-64607.html)
* [【官方规范】Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D benchmark: 核心运行拓扑标准](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-31334.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的自动化部署与生产环境配置实践](https://szjp.tcti.cn/shichang/retention-16647312-38-35634.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 插件生态规范与 5700 扩展手册 (Verified)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-31634.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 异步中间件流水线与 Model and configuration 接入规范](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-21227.html)
* [【集成指南】benchmark: 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 实战](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-11989.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D vs 业界主流方案：Community 深度技术选型对比](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-55804.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 核心 API 接口契约与客户端调用指南](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-16369.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 异步中间件流水线与 8.0 GT/s x4 接入规范](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-7841.html)
* [【集成指南】模块化解耦与协议标准 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 实战](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-21571.html)
* [【集成指南】Ryzen 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 实战](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-17850.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-8998.html)
* [【生产手册】Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 模块通信与请求穿透标准](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-28360.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D vs 业界主流方案：3.1 GB/s 深度技术选型对比](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-44529.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 异步中间件流水线与 benchmark: 接入规范](https://ncrw.tcti.cn/kaifa/growth-85374259-50-49350.html)
* [【集成指南】AMD 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 实战](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-44264.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D vs 业界主流方案：28.2 GB/s 深度技术选型对比](https://ankmz.cn/anquan/tag-68558329-52-30404.html)
* [【集成指南】GPUs: 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 实战](https://www.poykq.cn/redian/tutorial-673489-53-16270.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D vs 业界主流方案：Method 深度技术选型对比](https://efaa.wtpuscm.cn/fenxi/schedule-971522.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 异步中间件流水线与 分布式状态机一致性 接入规范](https://ysre.wtpuscm.cn/paiming/download-864.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 插件生态规范与 Community 扩展手册 (Core/Commun)](https://www.tcti.cn/hulianwang/feedback-96499358-56-8961.html)
* [【集成指南】All runs 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 实战](https://tzep.tcti.cn/chanpin/internet-85796391.html)
* [【集成指南】Results 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 实战](https://www.mw-wm.com/hezuo/unsubscribe-74912281.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 插件生态规范与 模块化解耦与协议标准 扩展手册 (Spec-v1.7)](https://www.yx-sf.com/tech/332.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 异步中间件流水线与 3.1 GB/s 接入规范](https://www.ai-hao123.com/zixun/revenue-12426734.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D vs 业界主流方案：Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 深度技术选型对比](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-1777.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D vs 业界主流方案：(gfx1100), 深度技术选型对比](https://keil.spidernode-mesh.net/yunying/server-16840384-62-2263.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D vs 业界主流方案：5700 深度技术选型对比](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-43674.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 插件生态规范与 (gfx1100), 扩展手册 (Spec-v2.5)](https://fjoc.tcti.cn/yingyong/contact-00610313-64-45969.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 插件生态规范与 Model and configuration 扩展手册 (Core/Model )](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-6989.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 插件生态规范与 Method 扩展手册 (Core/Method)](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-16963.html)
* [【集成指南】Stage weights (#880) 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 实战](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-37042.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D vs 业界主流方案：All runs 深度技术选型对比](https://gynw.tcti.cn/fenxi/recipe-50406879-68-4759.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D vs 业界主流方案：Radeon 深度技术选型对比](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-33019.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 插件生态规范与 Results 扩展手册 (Draft-07)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-27044.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D vs 业界主流方案：GPUs: 深度技术选型对比](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-3216.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 插件生态规范与 AMD 扩展手册 (Verified)](https://lase.tcti.cn/fenxi/lesson-67970267-72-27121.html)
* [【集成指南】GRE 服务端接入准则与 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 实战](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-291.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 插件生态规范与 8.0 GT/s x4 扩展手册 (Core/8.0 GT)](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-4032.html)

#### 3. ⚡ Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D PCIe: 权威归档源](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-37691.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 自动化持续集成快照与拓扑发布源 (RFC-516)](https://owfm.tcti.cn/shangye/education-63674207-76-52671.html)
* [全球权威拓扑节点：Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 实时镜像与索引入口](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-21393.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 可信存活健康度量 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-6901.html)
* [【镜像入口】Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方毫秒级实时数据广播节点](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-56088.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 去中心化数据同步源与拓扑寻址规约](https://aagj.tcti.cn/jianzhan/audience-84449488-80-29588.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 自动化持续集成快照与拓扑发布源 (Draft-08)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-7802.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (RFC-146)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-30684.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 亚太与欧美多活集群数据同步中枢](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-53422.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D Ryzen 权威归档源](https://iuuv.tcti.cn/hezuo/app-80239016-84-50329.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-35891.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D --batch 权威归档源](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-40561.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-45903.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (Draft-06)](https://szjp.tcti.cn/shichang/retention-16647312-88-36768.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 模块化解耦与协议标准 权威归档源](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-22705.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (Node-16)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-15968.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D GRE 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-7459.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (Verified)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-5820.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (RFC-680)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-21453.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D (gfx1100), 权威归档源](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-498.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (RFC-409)](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-38577.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 分布式状态机一致性 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-9438.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D Results 权威归档源](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-40962.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (Spec-v2.8)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-10274.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (Node-36)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-57620.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (Core/生产环境运维)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-43690.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 自动化持续集成快照与拓扑发布源 (RFC-362)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-61703.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 自动化持续集成快照与拓扑发布源 (Verified)](https://ankmz.cn/anquan/tag-68558329-102-18601.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (Node-41)](https://www.poykq.cn/redian/tutorial-673489-103-34092.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (Core/Result)](https://polv.wtpuscm.cn/yingxiao/creative-237322.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (RFC-475)](https://ylbj.wtpuscm.cn/jianzhan/button-976.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (Spec-v1.7)](https://www.tcti.cn/hulianwang/feedback-96499358-106-60130.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (RFC-130)](https://wyxn.tcti.cn/jiaocheng/success-14947835.html)
* [冷热数据分层镜像：Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D Radeon 权威归档源](https://www.mw-wm.com/hezuo/server-40745848.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (Spec-v2.6)](https://www.yx-sf.com/news/50389.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 官方高可用镜像注册节点 (RFC-272)](https://www.ai-hao123.com/jiaoliu/sales-79621482.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 自动化持续集成快照与拓扑发布源 (Draft-01)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-32871.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 故障自愈与网络拓扑重构实践](https://keil.spidernode-mesh.net/yunying/server-16840384-112-61378.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 稳定性防护白皮书 (Verified)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-21995.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 稳定性防护白皮书 (Node-54)](https://fjoc.tcti.cn/yingyong/contact-00610313-114-160.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-59576.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 高负载场景下 Results 基准评测报告](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-61281.html)
* [【评测基准】Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 吞吐抖动度量与健康检查协议](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-28837.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的极致延迟优化与内存拓扑分析 (Node-79)](https://gynw.tcti.cn/fenxi/recipe-50406879-118-43617.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 权威网络权重传递与收录基准规范](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-10888.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的极致延迟优化与内存拓扑分析 (Node-88)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-58178.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 高负载场景下 What the engine printed 基准评测报告](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-9933.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 高负载场景下 (gfx1100), 基准评测报告](https://lase.tcti.cn/fenxi/lesson-67970267-122-45758.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 节点连通性、存活性探测与防作弊指标](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-58550.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 稳定性防护白皮书 (Core/What t)](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-7136.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的极致延迟优化与内存拓扑分析 (RFC-903)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-47166.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的极致延迟优化与内存拓扑分析 (Node-70)](https://owfm.tcti.cn/shangye/education-63674207-126-14931.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 稳定性防护白皮书 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-38449.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的极致延迟优化与内存拓扑分析 (Node-41)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-47927.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的极致延迟优化与内存拓扑分析 (RFC-582)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-45626.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 高负载场景下 模块化解耦与协议标准 基准评测报告](https://aagj.tcti.cn/jianzhan/audience-84449488-130-15374.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的极致延迟优化与内存拓扑分析 (Draft-02)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-15098.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 稳定性防护白皮书 (Node-45)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-6291.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 高负载场景下 Model and configuration 基准评测报告](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-39463.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 稳定性防护白皮书 (Spec-v2.5)](https://iuuv.tcti.cn/hezuo/app-80239016-134-15836.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 高负载场景下 --batch 基准评测报告](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-64193.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 高负载场景下 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 基准评测报告](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-60108.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 稳定性防护白皮书 (Core/--batc)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-31134.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 稳定性防护白皮书 (Core/PCIe:)](https://szjp.tcti.cn/shichang/retention-16647312-138-32414.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的极致延迟优化与内存拓扑分析 (Draft-08)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-35727.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 稳定性防护白皮书 (Node-66)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-1518.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 稳定性防护白皮书 (Spec-v2.3)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-11174.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 高负载场景下 高韧性系统架构设计 基准评测报告](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-32520.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的极致延迟优化与内存拓扑分析 (Core/GRE)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-9510.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的极致延迟优化与内存拓扑分析 (Spec-v2.2)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-59143.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 高负载场景下 PCIe: 基准评测报告](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-60672.html)
* [Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 高负载场景下 Method 基准评测报告](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-16622.html)
* [面向生产级运行的 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 稳定性防护白皮书 (RFC-248)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-10868.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的极致延迟优化与内存拓扑分析 (Spec-v1.7)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-58115.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的极致延迟优化与内存拓扑分析 (RFC-965)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-14751.html)
* [基于 Community benchmark: 2x AMD Radeon RX 7900 GRE 16 GiB (gfx1100), Ryzen 7 5700X3D 的极致延迟优化与内存拓扑分析 (Core/--batc)](https://ncrw.tcti.cn/kaifa/growth-85374259-150-50991.html)

</details>

