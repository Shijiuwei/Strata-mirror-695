# Community benchmark on RTX 5080 + Ryzen 9 9900X3D

Measured on 2026-10-05 by kalabaddon.

* **Machine:** RTX 5080 16 GB at **PCIe 5.0 x8**, Ryzen 9 9900X3D, 96 GiB DDR5-6000, Windows 11 25H2, Strata engine 0.1.39 (commit `6f32ec0`).
* **Tested:** four models in five configurations: IQ3_S, Unsloth UD-IQ4_XS (twice, before and after `START-HERE.bat --calibrate`), Unsloth UD-Q4_K_XL, and Coder IQ1_M.
* **Speed test:** 3 runs at each of 1k, 4k, 32k, 64k, 128k and 262k prompt tokens, 256-token output cap, greedy, reasoning off.
* **Needle test:** 9 cases (32k / 128k / 256k at 10 / 50 / 90% depth) on IQ3_S, calibrated UD-IQ4_XS, UD-Q4_K_XL and Coder IQ1_M.
* **Main caveat:** many answers stopped before the 256-token cap, so decode figures are less reliable than prompt figures.

## Summary

All four models found 9 of 9 needles. Speed, engine-reported medians (prompt range across 4k-262k, 1k shown separately because it is much lower for all of them; decode range across all six sizes):

| Configuration | Prompt tok/s, 4k-262k | Prompt tok/s, 1k | Decode tok/s | TTFT at 262k |
|---|---|---|---|---|
| IQ3_S | 2,344-3,603 | 674 | 80-98 | 80.6 s |
| UD-IQ4_XS, before calibration | 1,498-2,424 | 483 | 39-51 | 174.9 s |
| UD-IQ4_XS, calibrated | 1,480-2,410 | 479 | 52-63 | 174.8 s |
| UD-Q4_K_XL | 1,238-1,861 | 382 | 44-51 | 179.8 s |
| Coder IQ1_M (reduced quant, see below) | 3,175-4,001 | 1,431 | 86-117 | 73.6 s |

* **IQ3_S** was the fastest of the full models at every size, in prompt speed, decode speed and TTFT: from 4k up, about 1.5-2.2x the prompt speed and 1.4-2.5x the decode speed of the UD-IQ4_XS passes, and less than half their TTFT at 262k.
* **UD-Q4_K_XL** had the slowest prompt speed of the five at every size and the longest TTFT. Its decode speed was close to the uncalibrated UD-IQ4_XS and below the calibrated pass. It also had the largest expert arena (66,482 MiB) and the least RAM headroom (85.5 of 93.6 GiB in use, engine reading).
* **Calibration (UD-IQ4_XS)** changed only `--pcie-frac` (0.55 to 0.35). Decode medians rose 13-44%; prompt speed moved -2% to +8%.
* **Coder IQ1_M** is the Coder variant at the much smaller IQ1_M quant (expert arena 23,981 MiB, against 47,962-66,482 MiB for the others), aimed at more limited systems. It was tested in the hope of a large speed gain, but its lead over IQ3_S was modest: prompt speed about 9-11% higher from 32k up (35% at 4k, 112% at 1k), decode mixed (higher at most sizes, lower at 4k and 128k). It also differs from IQ3_S in `--spec-min-p` (0.50 vs 0.70) and expert profile, so even that gap is not down to the quant alone.

**Runs that wrote the full 256-token answer** (instead of stopping early, usually with a line like "I'll start by exploring the repository"). This is only a proxy for following the task: some full-length IQ3_S answers began by saying the file was truncated.

| Context | IQ3_S | UD-IQ4_XS, before calibration | UD-IQ4_XS, calibrated | UD-Q4_K_XL | Coder IQ1_M |
|---|---|---|---|---|---|
| 1k-64k (of 12) | 4 | 12 | 12 | 11 | 3 |
| 128k (of 3) | 1 | 2 | 1 of 2 \* | 1 | 2 |
| 262k (of 3) | 0 | 0 | 0 | 1 | 3 |
| Total (of 18) | 5 | 14 | 13 of 17 \* | 13 | 8 |

\* One calibrated 128k run failed with an engine error and is excluded (see Results).

**Conditions that limit the comparison:** PCIe 5.0 x8 rather than x16; only 249 MiB of VRAM free with IQ3_S loaded at 262,144 context; RAM nearly full with UD-IQ4_XS and UD-Q4_K_XL; one prompt per size; one failed request (calibrated UD-IQ4_XS, 128k); short answers make decode figures noisier. Details under Limitations.

## Hardware and software

* **GPU:** NVIDIA GeForce RTX 5080, 16 GB (16,303 MiB reported), the only GPU. The display is attached to it (`nvidia-smi` shows Disp.A On), so the desktop uses some VRAM.
* **PCIe link:** Gen 5 at **x8** (the card supports x16). `nvidia-smi --query-gpu=pcie.link.gen.current,pcie.link.gen.max,pcie.link.width.current,pcie.link.width.max`, sampled once a second for 15 seconds during a request, read 5, 5, 8, 16 every time. Cause, per the owner: lane sharing with the NVMe drives (not independently verified).
* **CPU:** AMD Ryzen 9 9900X3D (AM5).
* **RAM:** 96 GiB DDR5 (2x48 GiB Corsair Vengeance, 2 of 4 slots) at 6000 MT/s per Task Manager; 93.6 GiB usable per the engine.
* **Storage:** Samsung 970 EVO Plus 2 TB NVMe, the boot drive and the only drive used; negotiated link speed not checked.
* **Software:** Windows 11 25H2 (build 26200.9457); NVIDIA driver 610.47 (WDDM), CUDA 13.3 per `nvidia-smi`; Python 3.14. Strata engine 0.1.39 (startup log "session is up (engine 0.1.39)", matching the checkout's `CMakeLists.txt`), commit `6f32ec0`; release binary or source build: not recorded; no changed build options.
* **Background:** Firefox and a few other apps open; Steam and other gaming services stopped; memory use not otherwise minimized.
* **Memory snapshot with UD-IQ4_XS loaded** (Task Manager, one snapshot, not a peak during inference): 88.6 GiB in use (1.2 GiB compressed), 5.1 GiB available, 5.0 GiB cached, 97 of 157 GiB committed, 2.4 GiB hardware reserved. `nvidia-smi` at about the same time: 15,692 of 16,303 MiB VRAM in use, 51 W of a 360 W cap. No Task Manager snapshot for the other configurations; engine RAM readings for all five are in the table under Model and configuration.
* **Also available:** the machine dual-boots Linux with a software RAID (believed RAID 0) of two Crucial T705 1 TB PCIe Gen5 NVMe SSDs (rated up to 13,600 MB/s) and lower OS overhead. Not used here; I can rerun there if anyone wants a faster-storage comparison.

## Model and configuration

Models (all Qwen3.8-Flash-Next):

* **IQ3_S:** the original model (not the Coder or Swift variant), 2 shards, the first entry in setup's model menu, as provided by Strata's setup. Served as `qwen3.8-flash-next-iq3_s`.
* **UD-IQ4_XS:** Unsloth family, 3 shards, the experimental entry in setup's menu, as provided by Strata's setup. `docs/MODELS.md` on main describes this family as UD-Q4_K_XL, so UD-IQ4_XS may be newer than that page. Served as `qwen3.8-flash-next-unsloth-ud-iq4_xs`.
* **UD-Q4_K_XL:** Unsloth family, 4 shards, the model `docs/MODELS.md` names for this family. Served as `qwen3.8-flash-next-unsloth-ud-q4_k_xl`.
* **Coder IQ1_M:** Coder variant, 2 shards, from Strata's official install list. Served as `qwen3.8-flash-next-coder-iq1_m`.

Model repository and revision: not recorded; all GGUFs were downloaded by Strata's setup. Packs under `<Strata-data>\packs\` were generated by setup (no hashes recorded); expert profiles are the ones shipped in `data\`. No custom draft vocabulary, vision encoder off.

Engine launch flags. Paths are relative to `<Strata-data>` (the data folder next to the Strata folder) or to the Strata folder.

| Flag | IQ3_S | UD-IQ4_XS, before calibration | UD-IQ4_XS, calibrated | UD-Q4_K_XL | Coder IQ1_M |
|---|---|---|---|---|---|
| `--pack` | `<Strata-data>\packs\iq3_s` | `<Strata-data>\packs\unsloth-ud-iq4_xs` | same | `<Strata-data>\packs\unsloth-ud-q4_k_xl` | `<Strata-data>\packs\coder-iq1_m` |
| `--native` | `<Strata-data>\models\IQ3_S\Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf` | `<Strata-data>\models\unsloth-UD-IQ4_XS\Qwen3.8-Flash-Next-UD-IQ4_XS-00001-of-00003.gguf` | same | `<Strata-data>\models\unsloth-UD-Q4_K_XL\Qwen3.8-Flash-Next-UD-Q4_K_XL-00001-of-00004.gguf` | `<Strata-data>\models\coder-IQ1_M\Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00001-of-00002.gguf` |
| `--ple-gguf` | second shard of the model | not used | not used | not used | `<Strata-data>\models\coder-IQ1_M\Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00002-of-00002.gguf` |
| `--expert-profile` | `data\expert-profile.bin` (shipped) | `data\expert-profile.bin` (shipped) | same | `data\expert-profile.bin` (shipped) | `data\expert-profile-coder.bin` (shipped, Coder-specific) |
| `--expert-cache` | auto | auto | auto | auto | auto |
| `--prefill` | auto | auto | auto | auto | auto |
| `--spec` | 4 | 4 | 4 | 4 | 4 |
| `--mtp` | `<Strata-data>\mtp\rt` | `<Strata-data>\mtp\rt` | same | `<Strata-data>\mtp\rt` | `<Strata-data>\mtp\rt` |
| `--max-context` | 262144 | 262144 | 262144 | 262144 | 262144 |
| `--kv` | int8 | int8 | int8 | int8 | int8 |
| `--kv-resident` | 32768 | 32768 | 32768 | 32768 | 32768 |
| `--resident-budget-gib` | not set | 55 | 55 | 66 | not set |
| `--pcie-frac` | 0.20 | 0.55 † | 0.35 | 0.20 | 0.20 |
| `--spec-min-p` | 0.70 | 0.50 † | 0.50 | 0.70 | 0.50 |

Captured engine arguments for UD-Q4_K_XL, with local paths shortened (the other launches differ only as shown in the table):

```text
--pack <Strata-data>\packs\unsloth-ud-q4_k_xl
--native <Strata-data>\models\unsloth-UD-Q4_K_XL\Qwen3.8-Flash-Next-UD-Q4_K_XL-00001-of-00004.gguf
--expert-profile data\expert-profile.bin
--expert-cache auto --prefill auto --spec 4
--mtp <Strata-data>\mtp\rt
--max-context 262144 --kv int8 --kv-resident 32768
--resident-budget-gib 66 --pcie-frac 0.20 --spec-min-p 0.70
```

† Reported as in effect for that run (the flag was absent from its config); only the calibrated pass's flag list was captured directly. All other columns are the flag lists as launched.

Engine-reported state at the first 1k run of each configuration (from the metrics saved in each run's JSON):

| Engine-reported | IQ3_S | UD-IQ4_XS, before calibration | UD-IQ4_XS, calibrated | UD-Q4_K_XL | Coder IQ1_M |
|---|---|---|---|---|---|
| Expert cache slots (VRAM) | 4,010 (7,835 MiB) | 3,040 (7,009 MiB) | 2,941 (6,781 MiB) | 2,335 (6,967 MiB) | 3,973 (7,750 MiB) |
| Expert arena (RAM) | 47,962 MiB | 49,753 MiB | 49,981 MiB | 66,482 MiB | 23,981 MiB |
| VRAM free after load | 249 MiB | 99 MiB | 464 MiB | 176 MiB | 383 MiB |
| RAM in use (of 93.6 GiB) | 63.1 GiB | 68.8 GiB | 67.7 GiB | 85.5 GiB | 45.0 GiB |

Other settings: reasoning off (`reasoning_effort: none`), greedy (temperature 0), vision off, CPU workers 11 (engine-reported `pool_workers`; not set on the command line). Low-RAM mode and the experimental speed projection were not checked; the IQ3_S startup log has no control-vector line, which the Strata docs say appears when the projection is on.

## Method

* **Script:** `strata-bench.py` from `bench/results/2026-09-29-rtx3090-epyc-milan/data/` (stdlib only), run from the Strata repo root against the running server, once per configuration with its own `--label` and `--log`. The UD-Q4_K_XL command is shown; the others changed only `--label` and `--log` (labels `iq3_s`, `unsloth-ud-iq4_xs`, `unsloth-ud-iq4_xs_calibrated`, `coder-iq1_m`). Key removed:

```text
python bench\results\2026-09-29-rtx3090-epyc-milan\data\strata-bench.py --url http://127.0.0.1:8080 --api-key <API_KEY> --label unsloth-ud-q4_k_xl --contexts 1k,4k,32k,64k,128k,262k --log strata-unsloth-ud-q4_k_xl.log --out bench\results\2026-10-05-community-rtx5080-9900x3d
```

* **Prompt:** text from the Strata repository's own source and docs, cut by character count, followed by "explain what the last file above does, then propose one concrete improvement as a unified diff". Each run has a unique prefix, so the engine log shows 0 reused tokens in every run.
* **Output:** 256-token cap, streaming API, greedy.
* **Repetitions:** 3 per size; tables show median (range). No separate warm-up; the 1k runs came first. The expert cache adapts during a run and its state carried over between runs. Model loading is not timed.
* **Timing:** TTFT is client-side, from sending the request to the first content or reasoning token (with reasoning off, the first answer token). Total request time is client-side from send to the end of the stream, excluding model loading. Prompt and decode tok/s are the engine's own log-line rates (freshly read prompt tokens / ms, generated tokens / ms), not total request time divided by tokens. API-side figures are in each `matrix.json` as a cross-check.
* **Memory:** not logged during runs; only the snapshots above (Task Manager snapshot with UD-IQ4_XS loaded; engine RAM and VRAM readings at each configuration's first 1k run). No paging or out-of-memory failures were seen.
* **Units:** RAM in GiB (Task Manager's "GB" are binary; engine byte counts are converted to GiB), VRAM and expert memory in MiB.

## Results

Prompt tokens and generated tokens are the engine log counts for each run; reused tokens were 0 in every run, and a generated count of 256 means the run hit the output cap. Prompt and decode tok/s are engine-reported; TTFT and total request time are client-measured. Format: median (range). Every request completed except the one marked failed.

### IQ3_S

| Context | Prompt tokens | Generated tokens, runs 1 / 2 / 3 | Prompt tok/s | Decode tok/s | TTFT s | Total s |
|---|---|---|---|---|---|---|
| 1k | 1,042-1,143 | 174 / 8 / 256 | 673.8 (618.7-676.0) | 80.3 (59.9-94.7) | 1.71 (1.70-1.72) | 3.86 (1.83-4.41) |
| 4k | 4,184-4,450 | 256 / 256 / 96 | 2,344.3 (2,337.5-2,422.0) | 93.3 (87.9-97.2) | 1.84 (1.83-1.87) | 4.45 (2.91-4.59) |
| 32k | 31,513-31,601 | 112 / 83 / 49 | 3,603.3 (3,590.9-3,607.6) | 98.1 (96.9-105.0) | 8.86 (8.85-8.87) | 9.63 (9.34-10.01) |
| 64k | 64,376-67,076 | 256 / 87 / 94 | 3,596.1 (3,382.4-3,601.1) | 85.5 (83.5-104.3) | 18.08 (18.04-20.01) | 19.11 (18.93-22.99) |
| 128k | 129,580-132,302 | 108 / 57 / 256 | 3,488.3 (3,377.3-3,496.0) | 90.2 (78.6-98.8) | 37.43 (37.42-39.46) | 40.54 (38.05-40.67) |
| 262k | 257,832-261,112 | 183 / 53 / 77 | 3,255.5 (3,242.7-3,259.9) | 91.0 (86.3-98.1) | 80.63 (80.08-80.78) | 81.46 (81.30-82.18) |

### UD-IQ4_XS, before calibration

| Context | Prompt tokens | Generated tokens, runs 1 / 2 / 3 | Prompt tok/s | Decode tok/s | TTFT s | Total s |
|---|---|---|---|---|---|---|
| 1k | 1,042-1,144 | 256 / 256 / 256 | 482.9 (198.4-508.1) | 48.3 (46.8-49.2) | 2.42 (2.27-5.33) | 7.59 (7.55-10.76) |
| 4k | 4,185-4,451 | 256 / 256 / 256 | 1,503.5 (702.2-1,518.1) | 51.3 (49.9-51.3) | 2.84 (2.83-6.40) | 7.94 (7.79-11.36) |
| 32k | 31,487-31,600 | 256 / 256 / 256 | 2,214.5 (1,474.8-2,239.1) | 39.0 (38.6-47.6) | 14.37 (14.21-21.48) | 20.75 (19.72-28.08) |
| 64k | 64,377-67,074 | 256 / 256 / 256 | 2,423.8 (1,563.4-2,451.0) | 47.0 (46.3-48.8) | 26.75 (26.51-43.10) | 32.25 (31.73-48.53) |
| 128k | 129,581-132,301 | 87 / 256 / 256 | 2,285.0 (1,707.0-2,318.0) | 39.6 (36.1-43.0) | 57.00 (56.36-77.82) | 64.08 (62.80-79.81) |
| 262k | 257,822-261,110 | 94 / 125 / 67 | 1,497.5 (1,497.1-1,813.4) | 50.6 (44.1-53.2) | 174.89 (142.77-175.01) | 176.13 (144.87-177.46) |

### UD-IQ4_XS, calibrated

| Context | Prompt tokens | Generated tokens, runs 1 / 2 / 3 | Prompt tok/s | Decode tok/s | TTFT s | Total s |
|---|---|---|---|---|---|---|
| 1k | 1,043-1,142 | 256 / 256 / 256 | 479.4 (248.9-500.6) | 58.7 (47.9-60.8) | 2.42 (2.32-4.25) | 6.65 (6.60-9.56) |
| 4k | 4,184-4,449 | 256 / 256 / 256 | 1,480.1 (680.3-1,499.1) | 59.2 (50.9-59.8) | 2.87 (2.87-6.60) | 7.16 (7.12-11.61) |
| 32k | 31,507-31,604 | 256 / 256 / 256 | 2,385.0 (1,466.1-2,395.0) | 56.3 (54.9-57.0) | 13.39 (13.35-21.64) | 17.87 (17.85-26.27) |
| 64k | 64,374-67,074 | 256 / 256 / 256 | 2,410.4 (1,508.3-2,417.0) | 53.2 (50.6-57.6) | 26.98 (26.83-44.67) | 31.62 (31.40-49.71) |
| 128k \* | 129,577-132,307 | failed / 83 / 256 | 2,264.1 (2,238.0-2,290.1) | 52.1 (49.9-54.2) | 58.25 (56.97-59.54) | 61.57 (61.05-62.08) |
| 262k | 257,730-261,118 | 96 / 54 / 113 | 1,499.1 (1,494.9-1,763.3) | 63.4 (58.5-65.3) | 174.78 (146.94-175.15) | 175.59 (148.55-176.91) |

\* Run 1 failed: the engine recorded the request as finishing with an error after reading 132,308 prompt tokens (1,674.4 tok/s) and generating 51 tokens (49.6 tok/s) in 80.4 s, and the API reply carried no usage block (`matrix.json` shows `prompt_tokens: null`, 17 completion tokens, 16.0 tok/s API decode). It is excluded from this row's medians and from every summary figure; its values are listed here and in `data/speed-128k-1.json`.

### UD-Q4_K_XL

| Context | Prompt tokens | Generated tokens, runs 1 / 2 / 3 | Prompt tok/s | Decode tok/s | TTFT s | Total s |
|---|---|---|---|---|---|---|
| 1k | 1,041-1,143 | 256 / 256 / 256 | 382.0 (225.7-403.4) | 49.2 (35.3-52.5) | 3.02 (2.86-4.69) | 8.20 (7.71-11.90) |
| 4k | 4,185-4,441 | 256 / 256 / 256 | 1,237.9 (714.3-1,283.6) | 48.1 (47.6-52.7) | 3.43 (3.33-6.28) | 8.73 (8.68-11.11) |
| 32k | 31,511-31,602 | 256 / 256 / 256 | 1,619.2 (1,584.5-1,728.5) | 51.0 (46.2-51.9) | 19.63 (18.40-20.01) | 24.54 (23.90-25.00) |
| 64k | 64,378-67,073 | 256 / 95 / 256 | 1,680.6 (1,426.7-1,998.7) | 45.2 (43.7-48.3) | 38.47 (32.42-47.23) | 43.75 (34.57-52.86) |
| 128k | 129,581-132,301 | 53 / 57 / 256 | 1,860.6 (1,565.0-1,883.0) | 46.8 (37.2-54.9) | 69.93 (69.26-84.89) | 76.12 (70.95-85.99) |
| 262k | 257,822-261,112 | 107 / 256 / 113 | 1,455.8 (1,446.0-1,527.7) | 44.3 (37.8-44.7) | 179.84 (169.47-181.17) | 182.37 (172.25-186.87) |

### Coder IQ1_M

| Context | Prompt tokens | Generated tokens, runs 1 / 2 / 3 | Prompt tok/s | Decode tok/s | TTFT s | Total s |
|---|---|---|---|---|---|---|
| 1k | 1,041-1,145 | 41 / 50 / 256 | 1,430.8 (1,318.8-1,447.5) | 104.9 (92.5-134.5) | 0.82 (0.81-0.83) | 1.29 (1.12-3.56) |
| 4k | 4,183-4,447 | 256 / 256 / 52 | 3,175.1 (3,140.9-3,180.2) | 89.2 (76.6-100.9) | 1.36 (1.35-1.46) | 3.88 (2.02-4.31) |
| 32k | 31,507-31,602 | 52 / 55 / 49 | 3,939.5 (3,931.3-3,947.4) | 108.1 (98.3-112.2) | 8.11 (8.09-8.13) | 8.61 (8.55-8.61) |
| 64k | 64,373-67,071 | 24 / 48 / 48 | 4,000.9 (3,829.8-4,002.2) | 116.8 (99.8-129.1) | 16.26 (16.25-17.67) | 16.66 (16.61-17.90) |
| 128k | 129,581-132,308 | 100 / 256 / 256 | 3,805.2 (3,789.7-3,847.9) | 86.2 (81.9-118.2) | 34.56 (33.96-35.05) | 37.07 (35.89-37.51) |
| 262k | 257,824-261,113 | 256 / 256 / 256 | 3,574.8 (3,555.9-3,577.7) | 94.1 (90.6-98.6) | 73.63 (72.62-73.96) | 76.33 (75.43-76.53) |

### What calibration changed (UD-IQ4_XS, medians)

| Context | Decode tok/s, before -> after | Prompt tok/s, before -> after |
|---|---|---|
| 1k | 48.3 -> 58.7 (+22%) | 482.9 -> 479.4 (-1%) |
| 4k | 51.3 -> 59.2 (+15%) | 1,503.5 -> 1,480.1 (-2%) |
| 32k | 39.0 -> 56.3 (+44%) | 2,214.5 -> 2,385.0 (+8%) |
| 64k | 47.0 -> 53.2 (+13%) | 2,423.8 -> 2,410.4 (-1%) |
| 128k \* | 39.6 -> 52.1 (+32%) | 2,285.0 -> 2,264.1 (-1%) |
| 262k | 50.6 -> 63.4 (+25%) | 1,497.5 -> 1,499.1 (0%) |

\* Calibrated 128k uses the two successful runs.

### Expert streaming

From the engine log lines across each configuration's successful runs (18, or 17 for the calibrated pass) ("N more read by the GPU over PCIe", and the decode expert cache hit rate):

| | IQ3_S | UD-IQ4_XS, before calibration | UD-IQ4_XS, calibrated | UD-Q4_K_XL | Coder IQ1_M |
|---|---|---|---|---|---|
| Routed experts read over PCIe | 2-6% (median 3%) | 14-25% (median 18%) | 10-14% (median 11%) | 5.5-8.0% (median 6.2%) | 0.3-3.6% (median 2.1%) |
| Expert cache hit rate | ~50-80% | ~50-80% | ~50-80% | 39-59% | 67-92% |

Per-run data and engine log lines are in `iq3_s/`, `unsloth-ud-iq4_xs/`, `unsloth-ud-iq4_xs_calibrated/`, `unsloth-ud-q4_k_xl/` and `coder-iq1_m/` (each has `matrix.json`, `matrix.md` and `data/`, with each run's actual prompt and generated token counts).

## Needle recall

`tools/needle_bench.py`, thinking off, greedy. All four found 9 of 9, with no errors or skipped cases. Prompts were identical across models: 32,171-32,172 tokens ("32k"), 125,449-125,451 ("128k") and 250,574-250,575 ("256k"). Results: `needles-iq3_s.json`, `needles-unsloth-ud-iq4_xs_calibrated.json`, `needles-unsloth-ud-q4_k_xl.json`, `needles-coder-iq1_m.json`. There is no needle run for the uncalibrated UD-IQ4_XS pass.

Command (UD-Q4_K_XL shown; the others changed only the `--out` file name; key removed):

```text
python tools\needle_bench.py --url http://127.0.0.1:8080 --api-key <API_KEY> --lengths 32k,128k,256k --depths 10,50,90 --out bench\results\2026-10-05-community-rtx5080-9900x3d\needles-unsloth-ud-q4_k_xl.json
```

Request time in seconds at depth 10 / 50 / 90%:

| Context | IQ3_S | UD-IQ4_XS, calibrated | UD-Q4_K_XL | Coder IQ1_M |
|---|---|---|---|---|
| 32k | 9.2 / 9.1 / 4.8 | 29.0 / 13.5 / 7.4 | 17.1 / 16.5 / 9.0 | 8.4 / 8.6 / 4.4 |
| 128k | 37.1 / 37.4 / 33.4 | 79.2 / 55.8 / 47.9 | 89.3 / 71.3 / 66.0 | 33.6 / 33.7 / 29.5 |
| 256k | 73.4 / 73.9 / 74.5 | 106.0 / 130.0 / 131.9 | 130.4 / 157.2 / 152.9 | 67.3 / 67.7 / 67.6 |

These are whole-request client times, and later depths probably reuse part of an earlier prompt's cached prefix, so use the speed tables for speed. Each needle run used the same server launch as that configuration's speed run (assumed for IQ3_S).

## Limitations

* **Short answers.** The benchmark prompt cuts the last file mid-function, and models often commented on that or stopped with a one-liner instead of writing 256 tokens (counts in the Summary). Examples: IQ3_S 1k run 2 answered "I can't help with that."; one Coder 64k run emitted a raw tool-call fragment. Decode tok/s from short answers is less reliable, so comparisons across configurations mix answer lengths. The UD-IQ4_XS before/after comparison is the cleanest, since all 12 runs at 1k-64k are full-length in both passes.
* **Task-following is not a quality measure.** At 1k-64k the UD-IQ4_XS passes and UD-Q4_K_XL nearly always wrote full answers while IQ3_S and Coder IQ1_M mostly did not; at 128k and 262k every configuration except Coder IQ1_M mostly stopped early, and calibration did not change the pattern. This is one prompt per size with greedy decoding, so it shows behaviour on this prompt only. All four models found every needle, so recall did not separate them.
* **Configurations differ in more than the model.** UD-IQ4_XS vs IQ3_S also differs in `--pcie-frac` (0.35 vs 0.20), `--spec-min-p` (0.50 vs 0.70) and `--resident-budget-gib 55`. UD-Q4_K_XL matches IQ3_S on `--pcie-frac` (0.20) and `--spec-min-p` (0.70) but sets `--resident-budget-gib 66`. Coder IQ1_M differs from IQ3_S in quant, `--spec-min-p` and expert profile, and was not re-run back to back with IQ3_S.
* **Warm-up.** In every UD-IQ4_XS run set, the first run at each size read its prompt more slowly than runs 2 and 3 (1k before calibration: 198 vs 483 and 508 tok/s), which looks like expert-cache warm-up; medians hide it, ranges show it. At 262k the first run was the faster one in both passes (about 1,800 vs 1,500 tok/s); I do not know why.
* **PCIe x8.** Strata streams experts over PCIe, and existing reports (for example engine 0.1.26 on an RTX 5070) list PCIe 5.0 x16, so absolute speeds here may be lower than the same card at x16. The calibrated `--pcie-frac` of 0.35 (0.55 before) is consistent with a narrower link, but untested.
* **Tight VRAM.** With IQ3_S at 262,144 context the engine reported 249 MiB of VRAM free and warned requests may stall; none did. A smaller `--max-context` or larger `--vram-reserve-mib` was not tested. Windows refused large pages for the expert arena, so it used 4 KB pages; whether granting them changes speed was not tested.
* **Tight RAM.** UD-IQ4_XS ran with 5.1 GiB available (Task Manager) and UD-Q4_K_XL with 85.5 of 93.6 GiB in use (engine), with Firefox and other apps open. The Strata docs list low free RAM as a cause of slow output, so those speeds may be lower than on a cleaner system.
* **Prompt lengths** vary by a few thousand tokens between runs of the same size, since the script builds prompts by character count.
* **Scope:** one set of 3 runs per size per configuration on one machine; no continuous memory, power or thermal logging.



---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [面向大规模网络的 Strata 工业级架构基准](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://ticg.tcti.cn/chanpin/deadline-99730252.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://www.ai-hao123.com/youhua/vacation-90853203.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://fpih.wtpuscm.cn/shangye/segment-538461.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://jkcz.wtpuscm.cn/zhizhu/update-432831.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://hfyj.wtpuscm.cn/jiaoliu/quality-992413.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://ajbb.wtpuscm.cn/fenxi/innovation-601.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://lmhz.wtpuscm.cn/yunsuan/home-301.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.yx-sf.com/tech/42195.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://ztit.tcti.cn/gongxiang/technology-56467044.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://ankmz.cn/anquan/tag-68558329.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://www.yx-sf.com/tech/56522.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://ankmz.cn/anquan/tag-68558329.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 异步中间件流水线与 any 接入规范](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://www.mw-wm.com/jiaoliu/affordable-54544556.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://ankmz.cn/anquan/tag-68558329.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://dxwz.wtpuscm.cn/gongsi/sales-095094.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://owfm.tcti.cn/shangye/education-63674207.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://wkzs.tcti.cn/yingyong/value-46386350.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.mw-wm.com/fuwu/support-49506961.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://egmh.wtpuscm.cn/chuangxin/objective-597.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://www.mw-wm.com/xitong/unsubscribe-87712344.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://www.ai-hao123.com/jishu/analysis-29199686.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://www.mw-wm.com/wenzhang/discovery-17709673.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://www.poykq.cn/redian/tutorial-673489.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata 故障自愈与网络拓扑重构实践](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata 权威网络权重传递与收录基准规范](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://www.yx-sf.com/tech/66510.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://www.yx-sf.com/news/20716.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://www.yx-sf.com/news/85938.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://www.mw-wm.com/zixun/podcast-00142724.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://www.ai-hao123.com/zhizhu/image-10705374.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://phpb.wtpuscm.cn/xinwen/event-087977.html)
* [Strata: Windows 分布式架构 (Verified)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)

</details>



---

<details open>
<summary><b>📚 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Community benchmark on RTX 5080 + Ryzen 9 9900X3D」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 分布式数据通道与 PCIe 5.0 x8 技术规范 (Core/PCIe 5)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-56714.html)
* [CMakeLists.txt 核心系统架构与设计规约 (v2.0-GA)](https://ankmz.cn/anquan/tag-68558329-2-55468.html)
* [【官方规范】Community benchmark on RTX 5080 + Ryzen 9 9900X3D Method 核心运行拓扑标准](https://www.poykq.cn/redian/tutorial-673489-3-55115.html)
* [面向大规模网络的 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 工业级架构基准](https://feih.wtpuscm.cn/baogao/identity-976530.html)
* [现代 RTX 架构演进之路 —— Community benchmark on RTX 5080 + Ryzen 9 9900X3D 深度实践](https://jzuv.wtpuscm.cn/sheji/meeting-172.html)
* [现代 Coder IQ1_M 架构演进之路 —— Community benchmark on RTX 5080 + Ryzen 9 9900X3D 深度实践](https://www.tcti.cn/hulianwang/feedback-96499358-6-9271.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 分布式数据通道与 --pcie-frac 技术规范 (Verified)](https://mxxs.tcti.cn/gongsi/analytics-60257814.html)
* [现代 Tested: 架构演进之路 —— Community benchmark on RTX 5080 + Ryzen 9 9900X3D 深度实践](https://www.mw-wm.com/paiming/upload-88775032.html)
* [Method 核心系统架构与设计规约 (RFC-652)](https://www.yx-sf.com/news/82200.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 分布式数据通道与 Coder IQ1_M 技术规范 (v2.0-GA)](https://www.ai-hao123.com/kaifa/income-07006782.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的高吞吐 Model and configuration 设计白皮书](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-29200.html)
* [现代 6f32ec0 架构演进之路 —— Community benchmark on RTX 5080 + Ryzen 9 9900X3D 深度实践](https://keil.spidernode-mesh.net/yunying/server-16840384-12-14837.html)
* [现代 Needle test: 架构演进之路 —— Community benchmark on RTX 5080 + Ryzen 9 9900X3D 深度实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-62500.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的高吞吐 Machine: 设计白皮书](https://fjoc.tcti.cn/yingyong/contact-00610313-14-46805.html)
* [Hardware and software 核心系统架构与设计规约 (v2.0-GA)](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-16737.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 内部组件解耦与事件状态机规范 (Verified)](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-5819.html)
* [现代 5080 架构演进之路 —— Community benchmark on RTX 5080 + Ryzen 9 9900X3D 深度实践](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-27949.html)
* [现代 高韧性系统架构设计 架构演进之路 —— Community benchmark on RTX 5080 + Ryzen 9 9900X3D 深度实践](https://gynw.tcti.cn/fenxi/recipe-50406879-18-3422.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 内部组件解耦与事件状态机规范 (Core/Hardwa)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-1872.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 分布式数据通道与 Needle recall 技术规范 (Spec-v2.1)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-47366.html)
* [现代 Results 架构演进之路 —— Community benchmark on RTX 5080 + Ryzen 9 9900X3D 深度实践](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-27258.html)
* [RTX 核心系统架构与设计规约 (RFC-814)](https://lase.tcti.cn/fenxi/lesson-67970267-22-6180.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 分布式数据通道与 Results 技术规范 (Core/Result)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-50172.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 分布式数据通道与 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 技术规范 (Node-46)](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-34899.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 分布式数据通道与 Needle test: 技术规范 (Node-38)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-59934.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的高吞吐 Summary 设计白皮书](https://owfm.tcti.cn/shangye/education-63674207-26-43770.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Core/生产环境运维)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-43948.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的高吞吐 Tested: 设计白皮书](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-53310.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的高吞吐 9900 设计白皮书](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-61199.html)
* [现代 nvidia-smi 架构演进之路 —— Community benchmark on RTX 5080 + Ryzen 9 9900X3D 深度实践](https://aagj.tcti.cn/jianzhan/audience-84449488-30-12993.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的高吞吐 Community 设计白皮书](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-37502.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 内部组件解耦与事件状态机规范 (v2.0-GA)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-7592.html)
* [现代 分布式状态机一致性 架构演进之路 —— Community benchmark on RTX 5080 + Ryzen 9 9900X3D 深度实践](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-26109.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的高吞吐 IQ3_S 设计白皮书](https://iuuv.tcti.cn/hezuo/app-80239016-34-3529.html)
* [模块化解耦与协议标准 核心系统架构与设计规约 (Core/模块化解耦与)](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-57925.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 内部组件解耦与事件状态机规范 (Spec-v1.3)](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-36510.html)
* [--spec-min-p 核心系统架构与设计规约 (Verified)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-38379.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D vs 业界主流方案：分布式状态机一致性 深度技术选型对比](https://szjp.tcti.cn/shichang/retention-16647312-38-55754.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 插件生态规范与 9900 扩展手册 (Core/9900)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-1025.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 插件生态规范与 Ryzen 扩展手册 (Draft-05)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-7010.html)
* [【集成指南】Needle test: 服务端接入准则与 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 实战](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-57481.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的自动化部署与生产环境配置实践](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-3438.html)
* [【集成指南】Community benchmark on RTX 5080 + Ryzen 9 9900X3D 服务端接入准则与 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 实战](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-62418.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 核心 API 接口契约与客户端调用指南](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-21657.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 异步中间件流水线与 Needle test: 接入规范](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-13741.html)
* [【生产手册】Community benchmark on RTX 5080 + Ryzen 9 9900X3D 模块通信与请求穿透标准](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-25212.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D vs 业界主流方案：Community benchmark on RTX 5080 + Ryzen 9 9900X3D 深度技术选型对比](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-63886.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D vs 业界主流方案：Needle recall 深度技术选型对比](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-1545.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 异步中间件流水线与 benchmark 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-65248.html)
* [【集成指南】Tested: 服务端接入准则与 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 实战](https://ncrw.tcti.cn/kaifa/growth-85374259-50-59239.html)
* [【集成指南】Ryzen 服务端接入准则与 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 实战](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-35054.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 异步中间件流水线与 --pcie-frac 接入规范](https://ankmz.cn/anquan/tag-68558329-52-48859.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D vs 业界主流方案：CMakeLists.txt 深度技术选型对比](https://www.poykq.cn/redian/tutorial-673489-53-22288.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 异步中间件流水线与 UD-IQ4_XS, calibrated 接入规范](https://ciky.wtpuscm.cn/fenxi/backup-755845.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D vs 业界主流方案：Summary 深度技术选型对比](https://ylmx.wtpuscm.cn/paiming/excellence-755.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 异步中间件流水线与 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 接入规范](https://www.tcti.cn/hulianwang/feedback-96499358-56-12982.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://twrk.tcti.cn/yanjiu/course-69597529.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 插件生态规范与 Method 扩展手册 (v2.0-GA)](https://www.mw-wm.com/xinwen/audience-27829584.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 插件生态规范与 benchmark 扩展手册 (Draft-07)](https://www.yx-sf.com/wiki/71552.html)
* [【集成指南】Community 服务端接入准则与 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 实战](https://www.ai-hao123.com/zhizhu/discount-76911901.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 插件生态规范与 RTX 扩展手册 (RFC-440)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-58558.html)
* [【集成指南】benchmark 服务端接入准则与 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 实战](https://keil.spidernode-mesh.net/yunying/server-16840384-62-57593.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 插件生态规范与 RTX 扩展手册 (v2.0-GA)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-47237.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 异步中间件流水线与 Expert streaming 接入规范](https://fjoc.tcti.cn/yingyong/contact-00610313-64-53304.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 异步中间件流水线与 Main caveat: 接入规范](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-34432.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 插件生态规范与 模块化解耦与协议标准 扩展手册 (Verified)](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-31480.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 插件生态规范与 Summary 扩展手册 (Draft-06)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-41790.html)
* [【集成指南】高韧性系统架构设计 服务端接入准则与 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 实战](https://gynw.tcti.cn/fenxi/recipe-50406879-68-26332.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 插件生态规范与 UD-IQ4_XS, calibrated 扩展手册 (Spec-v2.4)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-45409.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 插件生态规范与 benchmark 扩展手册 (Verified)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-55046.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 插件生态规范与 5080 扩展手册 (RFC-249)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-43216.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 插件生态规范与 Coder IQ1_M 扩展手册 (Core/Coder )](https://lase.tcti.cn/fenxi/lesson-67970267-72-56626.html)
* [【集成指南】Main caveat: 服务端接入准则与 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 实战](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-9589.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D vs 业界主流方案：模块化解耦与协议标准 深度技术选型对比](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-14401.html)

#### 3. ⚡ Community benchmark on RTX 5080 + Ryzen 9 9900X3D 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 官方高可用镜像注册节点 (Verified)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-645.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 亚太与欧美多活集群数据同步中枢](https://owfm.tcti.cn/shangye/education-63674207-76-15847.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D Needle test: 权威归档源](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-2916.html)
* [全球权威拓扑节点：Community benchmark on RTX 5080 + Ryzen 9 9900X3D 实时镜像与索引入口](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-5593.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D PCIe 5.0 x8 权威归档源](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-32468.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 官方高可用镜像注册节点 (RFC-355)](https://aagj.tcti.cn/jianzhan/audience-84449488-80-56455.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 自动化持续集成快照与拓扑发布源 (Core/PCIe 5)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-30757.html)
* [【镜像入口】Community benchmark on RTX 5080 + Ryzen 9 9900X3D 官方毫秒级实时数据广播节点](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-33224.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D IQ3_S 权威归档源](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-62626.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 去中心化数据同步源与拓扑寻址规约](https://iuuv.tcti.cn/hezuo/app-80239016-84-49634.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D --spec-min-p 权威归档源](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-52133.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D Summary 权威归档源](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-18711.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 自动化持续集成快照与拓扑发布源 (Core/--pcie)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-60966.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D 5080 权威归档源](https://szjp.tcti.cn/shichang/retention-16647312-88-7921.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 自动化持续集成快照与拓扑发布源 (Core/Result)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-59404.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 自动化持续集成快照与拓扑发布源 (Node-95)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-38172.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D Needle recall 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-3289.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 官方高可用镜像注册节点 (Node-85)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-25306.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D Tested: 权威归档源](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-23690.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D Machine: 权威归档源](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-27179.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D UD-Q4_K_XL 权威归档源](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-39719.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D UD-IQ4_XS, calibrated 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-33791.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 官方高可用镜像注册节点 (v2.0-GA)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-5148.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 官方高可用镜像注册节点 (RFC-218)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-38498.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 自动化持续集成快照与拓扑发布源 (Node-11)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-48840.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D Community benchmark on RTX 5080 + Ryzen 9 9900X3D 权威归档源](https://ncrw.tcti.cn/kaifa/growth-85374259-100-47257.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D: Summary Summary (v2.0-GA)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-56593.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 自动化持续集成快照与拓扑发布源 (Verified)](https://ankmz.cn/anquan/tag-68558329-102-16353.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D benchmark 权威归档源](https://www.poykq.cn/redian/tutorial-673489-103-8699.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D 9900 权威归档源](https://ojcq.wtpuscm.cn/jianzhan/cost-978777.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 官方高可用镜像注册节点 (Core/UD-Q4_)](https://olmb.wtpuscm.cn/jishu/platform-424.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 官方高可用镜像注册节点 (Node-58)](https://www.tcti.cn/hulianwang/feedback-96499358-106-27985.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 官方高可用镜像注册节点 (Node-75)](https://rbsh.tcti.cn/shuju/innovation-87944310.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D: 5080 5080 (Node-50)](https://www.mw-wm.com/xuexi/lead-50968495.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 自动化持续集成快照与拓扑发布源 (Core/Needle)](https://www.yx-sf.com/tech/9989.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://www.ai-hao123.com/shuju/blog-03216349.html)
* [冷热数据分层镜像：Community benchmark on RTX 5080 + Ryzen 9 9900X3D Expert streaming 权威归档源](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-48049.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 故障自愈与网络拓扑重构实践](https://keil.spidernode-mesh.net/yunying/server-16840384-112-9274.html)
* [面向生产级运行的 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 稳定性防护白皮书 (Verified)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-6451.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 权威网络权重传递与收录基准规范](https://fjoc.tcti.cn/yingyong/contact-00610313-114-63852.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 高负载场景下 Hardware and software 基准评测报告](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-47958.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 节点连通性、存活性探测与防作弊指标](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-61479.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的极致延迟优化与内存拓扑分析 (Core/IQ3_S)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-24351.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 高负载场景下 分布式状态机一致性 基准评测报告](https://gynw.tcti.cn/fenxi/recipe-50406879-118-36728.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 高负载场景下 Method 基准评测报告](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-21144.html)
* [【评测基准】Community benchmark on RTX 5080 + Ryzen 9 9900X3D 吞吐抖动度量与健康检查协议](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-16594.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的极致延迟优化与内存拓扑分析 (Spec-v2.0)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-43813.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 高负载场景下 高韧性系统架构设计 基准评测报告](https://lase.tcti.cn/fenxi/lesson-67970267-122-29479.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 高负载场景下 UD-Q4_K_XL 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-57302.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 高负载场景下 Machine: 基准评测报告](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-45566.html)
* [面向生产级运行的 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 稳定性防护白皮书 (Core/Coder )](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-60987.html)
* [面向生产级运行的 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 稳定性防护白皮书 (RFC-275)](https://owfm.tcti.cn/shangye/education-63674207-126-7971.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的极致延迟优化与内存拓扑分析 (Verified)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-25490.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 高负载场景下 9900 基准评测报告](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-61081.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的极致延迟优化与内存拓扑分析 (Node-29)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-42503.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 高负载场景下 Results 基准评测报告](https://aagj.tcti.cn/jianzhan/audience-84449488-130-1037.html)
* [面向生产级运行的 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 稳定性防护白皮书 (Draft-07)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-11458.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的极致延迟优化与内存拓扑分析 (Core/Needle)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-2061.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 高负载场景下 Expert streaming 基准评测报告](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-14371.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D: 高韧性系统架构设计 高韧性系统架构设计 (Verified)](https://iuuv.tcti.cn/hezuo/app-80239016-134-31924.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D: IQ3_S IQ3_S (Verified)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-2787.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D: Results Results (Verified)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-1357.html)
* [面向生产级运行的 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 稳定性防护白皮书 (Draft-05)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-12951.html)
* [面向生产级运行的 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 稳定性防护白皮书 (RFC-960)](https://szjp.tcti.cn/shichang/retention-16647312-138-22134.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的极致延迟优化与内存拓扑分析 (Draft-06)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-20245.html)
* [面向生产级运行的 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 稳定性防护白皮书 (Node-92)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-46415.html)
* [面向生产级运行的 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 稳定性防护白皮书 (Draft-01)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-5979.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D: Community benchmark on RTX 5080 + Ryzen 9 9900X3D Community benchmark on RTX 5080 + Ryzen 9 9900X3D (Verified)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-64937.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的极致延迟优化与内存拓扑分析 (Core/5080)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-6194.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的极致延迟优化与内存拓扑分析 (Spec-v1.0)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-21928.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的极致延迟优化与内存拓扑分析 (RFC-890)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-16328.html)
* [面向生产级运行的 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 稳定性防护白皮书 (RFC-653)](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-55372.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的极致延迟优化与内存拓扑分析 (Spec-v1.6)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-20516.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 高负载场景下 Model and configuration 基准评测报告](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-38103.html)
* [基于 Community benchmark on RTX 5080 + Ryzen 9 9900X3D 的极致延迟优化与内存拓扑分析 (Node-95)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-37780.html)
* [Community benchmark on RTX 5080 + Ryzen 9 9900X3D 高负载场景下 Needle test: 基准评测报告](https://ncrw.tcti.cn/kaifa/growth-85374259-150-41314.html)

</details>

