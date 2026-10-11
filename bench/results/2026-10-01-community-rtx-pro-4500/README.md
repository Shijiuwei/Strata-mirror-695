# Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX

Measured on 2026-10-01 by [mlfather](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html), on a single-GPU Linux workstation. This tests
Strata engine 0.1.31 (source build, same binary and commit for every
quantization below) with three packs on the same GPU: the Unsloth UD-Q4_K_XL
pack, the Unsloth UD-IQ4_XS pack, and ISTA-DASLab's GSQ-RCO IQ3_S pack, each at
three separate context limits (8,192 / 81,920 / 139,264 tokens, one server
process per limit per pack). Greedy decoding, synthetic prompts, no correctness
suite run beyond what is noted in Correctness and limitations.

UD-Q4_K_XL median decode throughput was **61.5 tok/s at a 2,700-token prompt,
72.4 tok/s on a 370-token code-generation prompt, 72.1 tok/s at 64K prompt
tokens, and 70.2 tok/s at 128K**. UD-IQ4_XS reached **82.0 / 98.0 / 95.8 / 91.4
tok/s** on the same four prompts; ISTA IQ3_S reached **106.3 / 123.8 / 120.3 /
116.5 tok/s**. All three packs import and run on pristine, unpatched engine
0.1.31 (see Model and configuration: UD-IQ4_XS previously needed a local,
unpublished patch on engine 0.1.18 and no longer does). These are synthetic
code-review/code-generation requests with a 256-1,024-token output cap. They do
not establish general answer quality or performance on other workloads or
hardware.

## Hardware and software

- NVIDIA RTX PRO 4500 Blackwell; 32,623 MiB reported VRAM (405 MiB reserved by
  the driver); power limit 200 W (min 150 W, max 200 W, fixed at default for
  this test). PCIe: slot/host maximum Gen 4 x16, device capable of Gen 5; the
  link was sampled at Gen 1 x16 while the GPU was idle between runs (NVIDIA's
  power-saving link state), current width 16x. The engine's own PCIe probe
  measured **28.2-28.3 GB/s host-to-device** at each server start. GPU clocks
  were not fixed for this test.
- AMD Ryzen Threadripper PRO 3975WX; 32 cores / 64 threads (no AVX-512; AVX2
  only). The engine selected AVX2 and **31 expert-pool workers plus its host
  thread**.
- 512 GB installed RAM (8-channel DDR4); `free -g` reported 503 GiB total, with
  86 GiB free and 407 GiB in buff/cache at measurement time. Storage for the
  model and pack: a Sabrent SB-RKT4P-4TB NVMe SSD mounted at `/mnt/fast`
  (3.6 TB, 23% used). 32 GiB zram swap (not relevant: no paging under test).
- Omarchy (Arch-based) Linux, kernel `7.2.5-3-omarchy`, NVIDIA driver
  `615.71.09`, CUDA UMD version 13.4 (nvidia-smi reports both `Driver Version`
  and `CUDA Version` as deprecated header fields on this driver; CUDA UMD
  Version is the successor field).
- Source commit `9259cad4cfa3543cd3b8decab5962672b968c649` (tag `v0.1.31`);
  engine 0.1.31. Self-built with CMake + Ninja, `CMAKE_CUDA_ARCHITECTURES=120`
  (sm_120, this GPU's native architecture), `CMAKE_CUDA_COMPILER=/opt/cuda/bin/nvcc`,
  `CMAKE_CUDA_HOST_COMPILER=/usr/bin/g++-15`. See [BUILD.json](BUILD.json).
- NVCC release 13.3, V13.3.73 (`cuda_13.3.r13.3/compiler.38244171_0`); GCC
  (g++-15) 15.3.0; Python 3.13.15 in the project's `.venv`.
- Background workloads: a lightweight hardware/job-monitoring pair of daemons
  using about 2% of one CPU core throughout, and the Hyprland desktop compositor
  running on a second, separate GPU (an RTX PRO 4000) that did not share the
  benchmarked card. Nothing else used the benchmarked GPU during the runs.

## Model and configuration

Model: `unsloth/Qwen3.8-Flash-Next-GGUF`, subfolder `UD-Q4_K_XL`, revision
`38bb39e`:

- `Qwen3.8-Flash-Next-UD-Q4_K_XL-00001-of-00004.gguf`
- `Qwen3.8-Flash-Next-UD-Q4_K_XL-00002-of-00004.gguf`
- `Qwen3.8-Flash-Next-UD-Q4_K_XL-00003-of-00004.gguf`
- `Qwen3.8-Flash-Next-UD-Q4_K_XL-00004-of-00004.gguf`

All four local GGUF SHA-256 hashes matched the values published in the
project's own `docs/UNSLOTH_Q4.md` for this revision (recorded in
[BUILD.json](BUILD.json)). "UD-Q4_K_XL" is a mixed-format quantization: routed
experts are Q4_K (Q5_K in layer 2) for gate/up and Q5_1 (Q8_0 in layers 2, 4,
30, 46, 47) for down; embedding, output head, attention, shared-expert
projections and the PLE key are Q8_0; the PLE table is IQ4_NL.

No vision encoder was used. No custom expert profile or draft vocabulary was
built; the pack used the model's bundled expert profile
(`data/expert-profile.bin`, 24,576 ranked pairs for 24,576 total experts: 48
layers x 512 experts/layer). The pack itself was built with
`tools/iq_pack.py --compat-bf16` (required for this mode: 195 small
hyper-connection/PLE-value tensors are rounded Q8_0-to-BF16, listed with exact
error bounds in the pack's own `conversions.json`). `--experts-bin` was not
used: the four GGUF shards are read in place via `--mmap-experts`, with no
separate 77 GB experts file. The MTP draft layer is the base model's own,
built from the published `mtp.*` tensors the same way the project's other
packs build it.

- Context: three separate server runs at 8,192 / 81,920 / 139,264 tokens
  (`--max-context`), each its own process with its own config file.
- KV: `--kv int8`. `--vram-reserve-mib 1536`. `--expert-cache auto`, no
  eviction policy change. `--prefill auto` (selected 8,192-token chunks).
- `--resident-budget-gib 100`: with this budget, all 24,576 experts were
  resident across GPU VRAM + pinned host RAM at every context length tested,
  with no experts left to the SSD-backed tier:

  | Context | GPU expert-cache slots | GPU VRAM for experts | RAM-resident slots | Total (of 24,576) |
  |---|---:|---:|---:|---:|
  | 8,192 | 7,982 | 23.30 GiB | 16,594 | 24,576 |
  | 81,920 | 7,614 | 22.22 GiB | 16,962 | 24,576 |
  | 139,264 | 7,329 | 21.39 GiB | 17,247 | 24,576 |

- MTP: `--spec 4 --spec-min-p 0.5` (draft depth 4, min-p 0.5); MTP is mandatory
  to start this engine's serve mode.
- Low-RAM mode off (not applicable: `--resident-budget-gib` is this engine's
  own RAM-budgeted mode, distinct from the low-RAM mode used on smaller GPUs
  in the project's own `docs/UNSLOTH_Q4.md`). No GPU vision. No experimental
  speed projection, no control vectors or calibration. Reasoning disabled per
  request (`chat_template_kwargs.enable_thinking=false`), temperature 0,
  top_k 1 (greedy).
- One additional comparison toggled `STRATA_KQ256=1` (multi-token AVX2 kernels
  for the Q4_K/Q5_1/Q8_0 expert formats) against the same 8,192-context config,
  on the short and codegen prompts only. Everything else unchanged.

The three complete measured configurations are
[configs/strata-ud-q4_k_xl.json](configs/strata-ud-q4_k_xl.json) (8,192 ctx),
[configs/strata-ud-q4_k_xl-64k.json](configs/strata-ud-q4_k_xl-64k.json)
(81,920 ctx) and
[configs/strata-ud-q4_k_xl-128k.json](configs/strata-ud-q4_k_xl-128k.json)
(139,264 ctx). Each config's `--native` path and `--pack` directory are local
to the machine that produced them; set them to your own checkout and model
location.

```text
.venv/bin/python serve/server.py --engine strata --config strata-ud-q4_k_xl.json --host 127.0.0.1 --port 8080
```

(`strata-ud-q4_k_xl-64k.json` / `-128k.json` were run the same way as separate
server processes, one context limit per run.)

Engine startup excerpts (AVX selection, worker count, PCIe probe, expert-cache
fill) are in [logs/strata-ud-q4_k_xl.log](logs/strata-ud-q4_k_xl.log),
[logs/strata-ud-q4_k_xl-64k.log](logs/strata-ud-q4_k_xl-64k.log) and
[logs/strata-ud-q4_k_xl-128k.log](logs/strata-ud-q4_k_xl-128k.log). The KQ256
comparison's console transcript is
[logs/strata-ud-q4_k_xl-kq256.log](logs/strata-ud-q4_k_xl-kq256.log).

### Unsloth UD-IQ4_XS (same engine build, 0.1.31)

Model: `unsloth/Qwen3.8-Flash-Next-GGUF`, subfolder `UD-IQ4_XS`, three shards
(`Qwen3.8-Flash-Next-UD-IQ4_XS-0000{1,2,3}-of-00003.gguf`), already present
locally; this submission did not re-verify the local files' hashes against a
published manifest for this quant (unlike UD-Q4_K_XL above).

**This import no longer needs a patch.** A same-machine 2026-09-28 study
(engine 0.1.18) found that Unsloth UD-IQ4_XS "needed the local patch on branch
`p620-q8_0-down` ... to run at all", because the engine's native PLE-key path
took Q2_0 only and this quant's PLE key is Q8_0. On this 0.1.31 checkout
(commit `9259cad`), `tools/iq_pack.py --gguf <shard 1> --out packs/ud-iq4_xs
--compat-bf16` built the pack directly (one layer's gate/up/down split across
shards, handled by `native_experts.txt` v4's per-role shard column; 460
tensors converted, 195 of them the same "round Q8_0 small projections to
BF16" `--compat-bf16` path UD-Q4_K_XL uses), and the server started and served
all four prompts with no code changes, using the model's own
`data/expert-profile.bin` and the same MTP draft layer (`mtp/rt`) the other
packs use. `docs/UNSLOTH_Q4.md`, current as of this commit, still says "Only
UD-Q4_K_XL ... is targeted" and warns other Unsloth quantizations "may" lack
kernel support; empirically, for UD-IQ4_XS on this engine version and GPU,
that caveat no longer applies. Quality was not compared against the patched
0.1.18 build or against llama.cpp in this submission.

Settings matched the UD-Q4_K_XL runs exactly except `--pack packs/ud-iq4_xs`
and `--native` pointing at the UD-IQ4_XS shard 1:
`--resident-budget-gib 100`, `--kv int8`, `--vram-reserve-mib 1536`,
`--expert-cache auto`, `--prefill auto`, `--spec 4 --spec-min-p 0.5`, one
server per context limit (8,192 / 81,920 / 139,264). All 24,576 experts were
resident (GPU cache + RAM budget, no file reads) at every context length:

| Context | GPU expert-cache slots | GPU VRAM for experts | RAM-resident slots | Total |
|---|---:|---:|---:|---:|
| 8,192 | 10,434 | 23.52 GiB | 14,142 | 24,576 |
| 81,920 | 9,957 | 22.43 GiB | 14,619 | 24,576 |
| 139,264 | 9,585 | 21.60 GiB | 14,991 | 24,576 |

Configs: [configs/strata-ud-iq4_xs.json](configs/strata-ud-iq4_xs.json),
[configs/strata-ud-iq4_xs-64k.json](configs/strata-ud-iq4_xs-64k.json),
[configs/strata-ud-iq4_xs-128k.json](configs/strata-ud-iq4_xs-128k.json).
Logs: [logs/strata-ud-iq4_xs.log](logs/strata-ud-iq4_xs.log),
[logs/strata-ud-iq4_xs-64k.log](logs/strata-ud-iq4_xs-64k.log),
[logs/strata-ud-iq4_xs-128k.log](logs/strata-ud-iq4_xs-128k.log).

### ISTA-DASLab GSQ-RCO IQ3_S (same engine build, 0.1.31)

Model: `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`, subfolder `IQ3_S`, two
shards, re-downloaded for this submission (the local copy had been deleted):

| File | Bytes | SHA-256 (as downloaded here; not checked against a published manifest) |
| --- | --- | --- |
| `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf` | 54,817,524,224 | `4c1eb2ceb4915e1192f4f386021897bde56a97f40a0bb78bb86465e0f7d2aca3` |
| `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf` | 28,800,138,432 | `316b46f3a2dbd68c900f43136ab9449f9dcc3725dfd8c794847c204bc161e113` |

Download measured at **104.9 MB/s** in the first two minutes (12.58 GB
transferred); at that rate the full ~83.6 GB download projected to about
13.7 minutes total, well under this submission's 60-minute download budget,
so it ran to completion (actual total: both shards, no incomplete files).

The pack was built with upstream's normal, undocumented-flag path (the same
one `setup.py` uses for this family when a GGUF is already on disk): `tools/
iq_pack.py --gguf <shard 1> --out packs/iq3_s`, no `--compat-bf16` and no
`--experts-bin` (this is not the Q2_0 AVX-512 special case, and `--mmap-experts`
mode needs no `experts.bin`). The pack built in seconds with no tensor
conversions (`conversions.json` is empty: this quant's small tensors are
already a type the engine reads natively, unlike the Unsloth packs' BF16
hyper-connection tensors). Same MTP draft layer (`mtp/rt`) and the model's own
`data/expert-profile.bin`; same per-context settings as the two packs above.

| Context | GPU expert-cache slots | GPU VRAM for experts | RAM-resident slots | Total |
|---|---:|---:|---:|---:|
| 8,192 | 12,812 | 24.31 GiB | 11,764 | 24,576 |
| 81,920 | 12,240 | 23.23 GiB | 12,336 | 24,576 |
| 139,264 | 11,800 | 22.39 GiB | 12,776 | 24,576 |

Configs: [configs/strata-iq3_s.json](configs/strata-iq3_s.json),
[configs/strata-iq3_s-64k.json](configs/strata-iq3_s-64k.json),
[configs/strata-iq3_s-128k.json](configs/strata-iq3_s-128k.json).
Logs: [logs/strata-iq3_s.log](logs/strata-iq3_s.log),
[logs/strata-iq3_s-64k.log](logs/strata-iq3_s-64k.log),
[logs/strata-iq3_s-128k.log](logs/strata-iq3_s-128k.log).

## Method

The benchmark script, prompt builder and their README are not published in
this folder (a private local harness, `bench.py` / `make_prompts.py`). They are
described here so the method is reproducible against an equivalent harness:

- Four fixed prompts, built once by the prompt builder and reused byte-identical
  across every engine/config tested on this machine: **short** (a ~2,700-token
  code-review request over one Python module), **codegen** (a ~370-token
  request to write a self-contained ~400-line Python job-queue module, 1,024-
  token output cap), **long64k** and **long128k** (the same code-review task
  with a large multi-file Python corpus concatenated ahead of it to reach
  roughly 64K and 128K prompt tokens). The corpus concatenated into the long
  prompts is the operator's own local pipeline code, which is why the prompt
  text itself is not included in this submission; the token counts and method
  are otherwise exactly as in the prompt manifest. `short` and `codegen` use a
  512-token output cap; the long prompts use 512; codegen uses 1,024.
- Every prompt is prefixed with a unique per-run tag so no measured run can be
  served from another run's prefix cache; run N's prompt text is otherwise
  identical across configs.
- 1 warm-up request + 3 measured requests for `short` and `codegen`; 1 warm-up
  + 2 measured for `long64k` and `long128k` (a trimmed long-prompt plan, noted
  explicitly here rather than silently reported as 3). Every measured request
  in every configuration hit its output cap (`finish_reason: length`); no
  request failed or was cancelled.
- Requests streamed over the OpenAI-compatible `/v1/chat/completions` endpoint
  with `temperature=0`, `top_k=1` (greedy) and
  `chat_template_kwargs.enable_thinking=false`; `reasoning_chars` was 0 on
  every run, confirming thinking stayed off.
- **TTFT**: wall-clock seconds from request start to the first streamed content
  token (client-side, includes HTTP and local tokenization over loopback).
  **Prefill tok/s**: the server's reported prompt-token count divided by TTFT.
  **Decode tok/s**: (locally re-counted completion tokens - 1) / (time of last
  token - time of first token); completion tokens were re-counted with the
  model's own tokenizer rather than trusting server-reported usage, to keep one
  ruler across configurations.
- The GPU and system RAM were sampled once per second for the duration of each
  request (`nvidia-smi` + `/proc/meminfo`); VRAM and power figures below are the
  per-run **maximum** (VRAM) and **mean** (power) of those samples, not a single
  snapshot.
- The server process was freshly started once per context-limit configuration
  (three server starts total for the primary run, one more for the KQ256
  comparison at the 8,192-context config). Within one server's lifetime, all
  prompts for that config were measured in increasing-length order; model
  loading and expert-cache fill are excluded from every timing (they happen
  once at server start, before the warm-up request).
- `long64k` and `long128k` server starts logged the resident-RAM expert-cache
  fill event-by-event ("copied cache complement through layer N/48...").
  Decode-time per-request logs separately reported a GPU **expert-cache hit
  rate** of 80.7-93.6% across all requests, and a small, request-by-request
  growing count of "blob reads from the file" even though the full 24,576-expert
  pool fit in VRAM + the RAM budget at every context length tested (see
  Correctness and limitations: this was not investigated further).

## Results

Each cell is the median **[minimum-maximum]** of the measured runs (3 for
`short`/`codegen`, 2 for `long64k`/`long128k`). Every request hit its output
cap; no failures.

| Prompt | Prompt tokens (server) | Runs | Prefill tok/s | Decode tok/s | TTFT s |
|---|---:|---:|---|---|---|
| short | 2,700 | 3 | 1,149.1 [1,129.1-1,149.6] | 61.5 [61.0-64.59] | 2.35 [2.35-2.39] |
| codegen | 370 | 3 | 330.4 [321.2-332.1] | 72.43 [72.33-73.77] | 1.12 [1.11-1.15] |
| long64k | 63,997 | 2 | 2,454.25 [2,451.2-2,457.3] | 72.11 [71.17-73.05] | 26.08 [26.04-26.11] |
| long128k | 127,999 | 2 | 2,351.35 [2,349.3-2,353.4] | 70.15 [69.9-70.39] | 54.44 [54.39-54.48] |

| Prompt | VRAM, MiB (peak) | RAM used, GiB (peak) | GPU power, W (mean, median [min-max]) |
|---|---:|---:|---|
| short | 31,420 | 101.1 | 140.4 [139.0-144.8] |
| codegen | 31,422 | 101.4 | 141.9 [139.7-142.9] |
| long64k | 31,256 | 102.8 | 177.2 [176.5-177.9] |
| long128k | 31,260 | 104.3 | 184.15 [183.7-184.6] |

Raw per-run records: [runs/strata-ud-q4_k_xl/](runs/strata-ud-q4_k_xl/) (each
prompt's `warmup.json`/`run1.json`/`run2.json`/`run3.json`). Aggregates:
[runs/strata-ud-q4_k_xl/summary.json](runs/strata-ud-q4_k_xl/summary.json).
RAM figures are `MemTotal - MemAvailable` (process + page cache + everything
else resident), not a single process's RSS; the model/pack/expert budget
dominates it. Decode expert-cache hit rate per request ranged 80.7% (long128k,
first/coldest request) to 93.6% (short, warmest request).

### Unsloth UD-IQ4_XS (engine 0.1.31, same engine build as UD-Q4_K_XL above)

| Prompt | Prompt tokens (server) | Runs | Prefill tok/s | Decode tok/s | TTFT s |
|---|---:|---:|---|---|---|
| short | 2,700 | 3 | 1,532.9 [1,520.1-1,538.1] | 82.02 [80.62-83.01] | 1.76 [1.75-1.78] |
| codegen | 370 | 3 | 484.0 [462.0-488.2] | 98.04 [98.01-99.4] | 0.76 [0.76-0.80] |
| long64k | 63,997 | 2 | 3,052.55 [3,049.8-3,055.3] | 95.78 [92.89-98.66] | 20.97 [20.95-20.98] |
| long128k | 127,999 | 2 | 2,893.25 [2,890.7-2,895.8] | 91.36 [88.96-93.76] | 44.24 [44.20-44.28] |

| Prompt | VRAM, MiB (peak) | RAM used, GiB (peak) | GPU power, W (mean, median [min-max]) |
|---|---:|---:|---|
| short | 31,224 | 84.7 | 154.2 [152.5-159.2] |
| codegen | 31,224 | 84.9 | 163.4 [158.8-163.5] |
| long64k | 31,228 | 86.6 | 187.85 [187.0-188.7] |
| long128k | 31,228 | 88.1 | 191.9 [191.5-192.3] |

Decode expert-cache hit rate per request ranged 85.1% (long128k, first/coldest
request) to 94.9% (long64k, warmest request); 91.7-94.6% on `short`/`codegen`.
Raw per-run records: [runs/strata-0131-ud-iq4_xs/](runs/strata-0131-ud-iq4_xs/).
Aggregates: [runs/strata-0131-ud-iq4_xs/summary.json](runs/strata-0131-ud-iq4_xs/summary.json).

### ISTA-DASLab GSQ-RCO IQ3_S (engine 0.1.31, same engine build as the two packs above)

| Prompt | Prompt tokens (server) | Runs | Prefill tok/s | Decode tok/s | TTFT s |
|---|---:|---:|---|---|---|
| short | 2,700 | 3 | 1,855.1 [1,834.3-1,867.4] | 106.29 [105.78-111.21] | 1.46 [1.45-1.47] |
| codegen | 370 | 3 | 667.4 [666.4-685.5] | 123.81 [121.74-125.45] | 0.55 [0.54-0.56] |
| long64k | 63,997 | 2 | 3,281.15 [3,279.8-3,282.5] | 120.31 [116.14-124.47] | 19.50 [19.50-19.51] |
| long128k | 127,999 | 2 | 3,101.85 [3,100.5-3,103.2] | 116.5 [111.82-121.19] | 41.27 [41.25-41.28] |

| Prompt | VRAM, MiB (peak) | RAM used, GiB (peak) | GPU power, W (mean, median [min-max]) |
|---|---:|---:|---|
| short | 31,192 | 74.4 | 181.4 [171.7-182.1] |
| codegen | 31,192 | 74.8 | 185.4 [183.9-186.3] |
| long64k | 31,182 | 76.7 | 192.9 [191.6-194.2] |
| long128k | 31,190 | 78.3 | 194.6 [194.1-195.1] |

Decode expert-cache hit rate per request ranged 85.5-89.4% (first/coldest
request on `short`/`long64k`/`long128k`) to 97.9% (warmest `short` request).
Raw per-run records: [runs/strata-0131-iq3_s/](runs/strata-0131-iq3_s/).
Aggregates: [runs/strata-0131-iq3_s/summary.json](runs/strata-0131-iq3_s/summary.json).

At every context length and for every pack above (UD-Q4_K_XL, UD-IQ4_XS,
IQ3_S), all 24,576 experts fit resident across the GPU cache and the 100 GiB
RAM budget (see Model and configuration), so none of these numbers reflect
SSD-backed expert reads. On the same engine build, same prompts, same
settings, ISTA IQ3_S decodes 1.19-1.29x faster than UD-IQ4_XS and 1.66-1.73x
faster than UD-Q4_K_XL, consistent with it having the smallest/lowest-bitwidth
expert set of the three and therefore keeping more of it resident in VRAM
rather than on the CPU's AVX2 path.

### STRATA_KQ256 comparison

`STRATA_KQ256=1` turns on multi-token AVX2 kernels for the Q4_K/Q5_1/Q8_0
expert formats, against the same 8,192-context configuration. Tested on
`short` and `codegen` only (3 measured runs each).

| Prompt | Decode tok/s, default | Decode tok/s, KQ256=1 | Prefill tok/s, default | Prefill tok/s, KQ256=1 |
|---|---|---|---|---|
| short | 61.5 [61.0-64.59] | 67.61 [61.58-68.13] | 1,149.1 [1,129.1-1,149.6] | 1,146.3 [1,140.5-1,150.1] |
| codegen | 72.43 [72.33-73.77] | 71.74 [70.73-75.48] | 330.4 [321.2-332.1] | 329.8 [314.4-337.7] |

On `short`, KQ256's median is higher but its range overlaps the default
range almost entirely (61.58 min vs. 61.0 min without it); on `codegen` the two
are statistically indistinguishable (medians within 1%). With only three runs
per configuration, this is not a confident win either way; it is consistent
with the project's own published guidance that this flag is "bit-exact, but
measured no faster" on different hardware. Raw records:
[runs/strata-ud-q4_k_xl-kq256/](runs/strata-ud-q4_k_xl-kq256/).

### Same-machine comparison: other engines/builds on this card

ISTA IQ3_S and Unsloth UD-IQ4_XS were previously measured on this card on
engine 0.1.18 with a local, unpublished Q8_0-down patch (2026-09-28); those
numbers are superseded by the same-engine (0.1.31), same-prompt, no-patch
results in the Results section above and are not repeated here. For reference,
the 0.1.18-patched decode medians were 109.5/129.3/127.6/125.2 tok/s (ISTA
IQ3_S) and 97.2/110.5/107.1/109.3 tok/s (Unsloth UD-IQ4_XS) across
short/codegen/long64k/long128k - both engine versions show the same ranking
(IQ3_S fastest, then UD-IQ4_XS, then UD-Q4_K_XL), but 0.1.31's unpatched
numbers are 10-25% lower than 0.1.18's patched ones on these two packs; this
submission did not investigate why (candidates include the patch's own
changes, prefill-chunk or expert-cache default differences between versions,
or normal run-to-run variance) and that comparison is confounded by version,
patch and day, same caveat as before.

The row below (EXL3, production) **is** still a same-day/same-prompt but
different-engine comparison, not superseded by anything above, and is kept
for context:

**Production baseline: TabbyAPI/exllamav3, EXL3 4.05 bpw, 2026-09-28** (same
card, same prompts, this is the engine normally used in production on this
machine, MTP and n-gram-in-RAM both off):

| Prompt | Decode tok/s (median [min-max]) | Prefill tok/s | TTFT s |
|---|---|---|---|
| short | 34.0 [33.9-34.2] | 533.5 | 5.06 |
| codegen | 34.0 [33.8-34.1] | 248.0 | 1.49 |
| long64k | 33.6 [33.4-33.7] | 559.9 | 114.3 |
| long128k | 33.5 [33.3-33.5] | 545.5 | 234.6 |

A same-day, same-prompt variant of this engine with MTP (4 draft tokens) and
the n-gram table pinned in RAM reached 36.8-50.9 tok/s decode and 1.6-4.4x
faster prefill than the plain baseline, still well under every Strata
configuration above on these prompts. Full detail for both engines on
2026-09-28, including MTP acceptance rates, quality checks and exact
configuration diffs, is kept in the project's internal results file and is
summarized, not reproduced in full, here.

## Correctness and limitations

- **Needle-in-a-haystack recall, UD-Q4_K_XL only, 139,264-ctx server
  (`configs/strata-ud-q4_k_xl-128k.json`)**: `tools/needle_bench.py --lengths
  32k,128k --depths 10,50,90` found the planted code word in all 6
  length/depth combinations (3 depths x 2 lengths), 14-52 s per check. Full
  per-check data (prompt tokens, seconds, the exact word and the model's
  answer) is in [needles.json](needles.json). This is recall only (one short
  exact-match phrase retrieved from a long context), not a general quality or
  reasoning check, and it was run on UD-Q4_K_XL alone; UD-IQ4_XS and IQ3_S
  (above) have no needle check in this submission.
- **Beyond the needle check, no broader correctness/quality suite was run as
  part of this submission** for any of the three packs. The only other quality
  evidence available is from the 2026-09-28 same-machine study, on a different
  engine version (0.1.18, patched) and different quantizations (ISTA IQ3_S and
  Unsloth UD-IQ4_XS): a 10-check coding-quality suite scored equal to or better
  than the production EXL3 baseline under matched sampling, and MTP acceptance
  stayed far higher inside long reasoning on Strata than on the EXL3/TabbyAPI
  stack. That evidence does not cover engine 0.1.31 or the UD-Q4_K_XL pack, so
  answer quality beyond needle recall is otherwise unmeasured here.
- **Single GPU, single machine.** No multi-GPU, no concurrency, no sustained
  thermal run.
- **Greedy decoding only** (temperature 0, top_k 1). No sampled-decoding or
  reasoning/thinking-mode speed was measured in this submission.
- **Long prompts (64K/128K) were measured with only 2 runs each**, not 3, as
  stated in Method; `short` and `codegen` have 3.
- **Every expert format in every pack here runs on this CPU's AVX2 path**
  (the Threadripper PRO 3975WX has no AVX-512); the optional multi-token AVX2
  kernel (`STRATA_KQ256=1`) was evaluated on the UD-Q4_K_XL pack above and made
  no clear difference there. Decode throughput on the UD-Q4_K_XL pack is
  accordingly the lowest of the three same-engine packs measured above (see
  Results): its larger, higher-bitwidth expert set keeps less of it resident
  in VRAM relative to IQ3_S and UD-IQ4_XS.
- **An unexplained small file-read count.** Per-request engine telemetry logged
  a nonzero and request-by-request growing "blobs read from the file" count
  even though startup logs show the full 24,576-expert pool fit entirely in
  GPU VRAM plus the RAM budget (no experts left over for the SSD-backed tier).
  This submission did not investigate whether that counter reflects genuine
  recurring disk I/O, one-time fill accounting carried forward, or something
  else; see the attached engine logs for the raw lines.
- No vision, no tool use, no multi-turn conversation reuse was exercised; every
  measured request started from an empty/unique prefix by design (to defeat
  prefix caching for a clean per-run measurement), so prompt-reuse behavior is
  untested here.

## Two GPUs (layer split)

Added 2026-10-01, same machine, same engine build (0.1.31, commit `9259cad`),
same three packs and the same four prompts as above. Both GPUs are the
identical card: a second **NVIDIA RTX PRO 4500 Blackwell, 32,623 MiB**, on the
same Threadripper PRO 3975WX host (the desktop's RTX PRO 4000 was never
visible to the engine: `CUDA_VISIBLE_DEVICES` was pinned to the two RTX PRO
4500 UUIDs only). `--layer-split auto --split-device 1` was added to each
single-GPU config; everything else (quant, context limit, `--kv int8`,
`--vram-reserve-mib 1536`, `--expert-cache auto`, `--prefill auto`, `--spec 4
--spec-min-p 0.5`, prompts, run counts, sampling) is unchanged from the
single-GPU section above. Configs are
[configs-2gpu/](configs-2gpu/), engine logs [logs-2gpu/](logs-2gpu/), raw runs
[runs-2gpu/](runs-2gpu/).

**`--resident-budget-gib 100` could not be reused.** That flag silently
implies `--resident-cpu-experts` in this engine build, and the engine refuses
to start with `--resident-cpu-experts does not support layer splits or remote
expert caches` (`strata generate`, exit code 2) the moment both are present
(confirmed by a failed server start before the working configs below; the
failed attempt's config was not kept since it never produced a server or
log, only that one-line engine error). The two-GPU configs therefore drop
`--resident-budget-gib` and fall
back to the engine's plain `--expert-cache auto` mode (GPU cache per stage,
the CPU AVX2 pool computes whatever a stage's cache does not hold, same as
`docs/MULTI_GPU.md`'s own worked example) - this is the only configuration
that starts, not a choice made for speed. One consequence: peak host RAM
(`MemTotal - MemAvailable`) reads **higher** on two GPUs than on one for
UD-Q4_K_XL (126-127 GiB vs. 101-104 GiB single-GPU) despite dropping the
RAM-resident expert tier entirely; this is most likely page cache warmed by
the CPU pool reading the mmapped GGUF for whichever half of the model each
stage's cache misses (the file was also still warm in cache from the
single-GPU runs minutes earlier on the same prompts), not a genuine
per-process RSS increase - this was not isolated further (UNCERTAIN; flagged
rather than asserted).

### Split decision and expert-cache occupancy

Startup log line (`layer split auto: K=... predicted ... the caches hold N of
24576 profiled pairs`) is the auto-search's own prediction; the actual final
allocation (`expert cache N slots` for CUDA0, `layer split: CUDA1 runs layers
K-47, expert cache N slots` for CUDA1) comes a few lines later, after VRAM is
also reserved for the prompt path's per-stage borrow and (on CUDA1) the draft
head. The two numbers are close but not identical - actual is consistently
0.5-1.9% below the search's prediction, presumably the reserve taken after the
search picks K:

| Pack | Context | K (CUDA0 layers) | Predicted combined | Actual: CUDA0 slots (VRAM) | Actual: CUDA1 slots (VRAM) | Actual combined |
|---|---:|---:|---:|---:|---:|---:|
| UD-Q4_K_XL | 8,192 | 24 | 16,843 (98.9%) | 8,550 (24.97 GiB) | 8,043 (23.47 GiB) | 16,593 (67.5%) |
| UD-Q4_K_XL | 81,920 | 23 | 16,500 (98.9%) | 8,405 (24.56 GiB) | 7,811 (22.78 GiB) | 16,216 (66.0%) |
| UD-Q4_K_XL | 139,264 | 24 | 16,219 (98.8%) | 8,237 (24.08 GiB) | 7,680 (22.41 GiB) | 15,917 (64.8%) |
| UD-IQ4_XS | 8,192 | 25 | 21,797 (99.7%) | 11,139 (25.00 GiB) | 10,428 (23.65 GiB) | 21,567 (87.8%) |
| UD-IQ4_XS | 81,920 | 25 | 21,349 (99.6%) | 10,914 (24.50 GiB) | 10,164 (23.05 GiB) | 21,078 (85.8%) |
| UD-IQ4_XS | 139,264 | 25 | 21,002 (99.6%) | 10,740 (24.10 GiB) | 9,965 (22.59 GiB) | 20,705 (84.2%) |
| IQ3_S | 8,192 | 24 | **24,576 (100.0%)** | 12,288 (22.14 GiB) | 12,170 (24.45 GiB) | 24,458 (99.5%) |
| IQ3_S | 81,920 | 25 | **24,576 (100.0%)** | 12,800 (23.08 GiB) | 11,776 (23.75 GiB) | **24,576 (100.0%)** |
| IQ3_S | 139,264 | 25 | **24,576 (100.0%)** | 12,800 (23.08 GiB) | 11,605 (23.40 GiB) | 24,405 (99.3%) |

Combined-% is of the model's 24,576 total profiled pairs, not of the routed
mass the predicted-% figures use, so the two percentages in a row are not
directly comparable; both are given as the engine itself reports them. Two
32 GB cards hold roughly **twice** the single-card slot count at matching
context (UD-Q4_K_XL 8K: 7,982 slots on one card in the single-GPU section
above vs. 16,593 actual combined here). IQ3_S's combined cache holds
essentially the entire 24,576-pair profile at every context tested (99.3-
100.0% actual) - the CPU AVX2 pool is barely used for this pack on two cards,
where on one card it carried roughly half the routed experts (11,800-12,812
of 24,576 single-GPU slots, from the Results section above).

**`--split-skip-if-fits` decision** (extra run, UD-Q4_K_XL, 139,264 context,
`configs-2gpu/strata-2gpu-ud-q4_k_xl-128k-skipfits.json`): the engine logged
`--split-skip-if-fits: CUDA0 (NVIDIA RTX PRO 4500 Blackwell) would hold only
22.17 of the profile's 71.73 GiB (26.74 GiB free, 4.57 GiB for the session,
drafter and reserve): the split stays` and ran the split exactly as the
non-skip config did. A single confirmatory measurement (1 warm-up + 1 run,
`runs-2gpu/strata-0131-2gpu-ud-q4_k_xl-skipfits/`) landed within noise of the
non-skip 128K run: 5,130.2 vs. 5,123.85 tok/s prefill, 127.15 vs. 130.15 tok/s
decode. For this pack/context the main card alone is nowhere near large
enough to hold the full 71.73 GiB profile, so the flag is a no-op here; it
would only matter for a pack/context small enough for one RTX PRO 4500 to
hold on its own (none of the three packs at any tested context qualify - even
IQ3_S's 46.84 GiB experts exceed one card's ~27-28 GiB free after reserving
VRAM for context/session/drafter).

### Results: two GPUs vs. one (median of measured runs)

Decode tokens/s, two GPUs vs. the single-GPU figure (Results section above),
and the speedup:

| Prompt | UD-Q4_K_XL 2 GPU | 1 GPU | x | UD-IQ4_XS 2 GPU | 1 GPU | x | IQ3_S 2 GPU | 1 GPU | x |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| short | 111.54 | 61.5 | 1.81x | 121.69 | 82.02 | 1.48x | 130.18 | 106.29 | 1.22x |
| codegen | 142.4 | 72.43 | 1.97x | 151.92 | 98.04 | 1.55x | 157.31 | 123.81 | 1.27x |
| long64k | 134.77 | 72.11 | 1.87x | 149.19 | 95.78 | 1.56x | 159.12 | 120.31 | 1.32x |
| long128k | 130.15 | 70.15 | 1.86x | 147.81 | 91.36 | 1.62x | 153.51 | 116.5 | 1.32x |

Prefill tokens/s (TTFT in parentheses), two GPUs vs. one:

| Prompt | UD-Q4_K_XL 2 GPU | 1 GPU | x | UD-IQ4_XS 2 GPU | 1 GPU | x | IQ3_S 2 GPU | 1 GPU | x |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| short | 1,791.0 (1.51s) | 1,149.1 (2.35s) | 1.56x | 2,158.5 (1.25s) | 1,532.9 (1.76s) | 1.41x | 2,229.4 (1.21s) | 1,855.1 (1.46s) | 1.20x |
| codegen | 607.5 (0.61s) | 330.4 (1.12s) | 1.84x | 952.2 (0.39s) | 484.0 (0.76s) | 1.97x | 996.7 (0.37s) | 667.4 (0.55s) | 1.49x |
| long64k | 4,920.0 (13.01s) | 2,454.25 (26.08s) | 2.00x | 5,649.75 (11.33s) | 3,052.55 (20.97s) | 1.85x | 5,642.85 (11.34s) | 3,281.15 (19.50s) | 1.72x |
| long128k | 5,123.85 (24.98s) | 2,351.35 (54.44s) | 2.18x | 5,769.9 (22.18s) | 2,893.25 (44.24s) | 1.99x | 5,757.25 (22.23s) | 3,101.85 (41.27s) | 1.86x |

**The pattern is consistent across all three packs: the bigger/lower-bitwidth
a pack's expert set relative to one card, the more two cards help.**
UD-Q4_K_XL, the largest pack (7,982-8,550 of 24,576 pairs fit one card, the
rest ride the CPU AVX2 pool), nearly doubles decode (1.81-1.97x) and more than
doubles long-prompt prefill (2.00-2.18x). IQ3_S, which already fit 11,800-
12,812 of 24,576 pairs on one card and now fits all 24,576 on two, gains the
least on decode (1.22-1.32x) because the second card is removing a smaller
CPU-pool fraction, but still gains meaningfully on prefill (1.20-1.86x) from
the pipelined chunk reads docs/MULTI_GPU.md describes. UD-IQ4_XS sits between
the two, as its single-card cache occupancy (9,585-10,434 of 24,576) does too.

VRAM (peak, MiB) and power (mean W) per card, and peak host RAM:

| Prompt | Pack | CUDA0 (main) VRAM | CUDA1 VRAM | CUDA0 power (active, per-request) | CUDA1 power (session mean, incl. idle gaps)\*\* | Host RAM peak |
|---|---|---:|---:|---:|---:|---:|
| short/codegen | UD-Q4_K_XL | 31,130 | 31,324 | 114.1 / 120.2 W | 123.7 W | 125.9-126.0 GiB |
| long64k | UD-Q4_K_XL | 31,136 | 31,330 | 150.6 W | 159.7 W | 126.5 GiB |
| long128k | UD-Q4_K_XL | 31,134 | 31,322 | 174.55 W | 171.4 W | 127.2 GiB |
| short/codegen | IQ3_S | 27,362 | 31,248 | 121.4 / 128.0 W | 137.6 W | 100.6-100.7 GiB |
| long64k | IQ3_S | 28,840 | 31,138 | 166.35 W | 157.2 W | 101.2 GiB |
| long128k | IQ3_S | 29,244 | 31,242 | 175.15 W | 168.5 W | 99.8 GiB |
| short/codegen | UD-IQ4_XS | 31,138 | 31,258 | 122.0 / 129.1 W | 131.4 W | 108.4-108.6 GiB |
| long64k | UD-IQ4_XS | 31,140 | 31,258 | 159.9 W | 156.9 W | 109.3 GiB |
| long128k | UD-IQ4_XS | 31,138 | 31,260 | 175.2 W | 167.7 W | 109.7 GiB |

\*\* CUDA1 was sampled by a second, independent `nvidia-smi` poll at 1 Hz for
the whole server lifetime (not per-request like `bench.py`'s own `--gpu`
sampling of CUDA0), so its power figure is a session mean including the idle
seconds between requests and is a few watts to ~15 W lower than a true
per-request average would read; VRAM peaks are unaffected by this (a peak is
a peak). Raw samples: `logs-2gpu/sec-*.csv`.

### Limitations added by this section

- **`--resident-budget-gib`/`--resident-cpu-experts` and `--layer-split` are
  mutually exclusive on engine 0.1.31** (see above) - this is a hard engine
  check, not a configuration choice, and it means the two-GPU numbers above
  are not on the same expert-residency mode as the single-GPU numbers earlier
  in this report; they use the engine's other, non-experimental caching mode
  instead, the one `docs/MULTI_GPU.md` itself demonstrates.
- Same caveats as the single-GPU section apply here too: greedy decoding
  only, long prompts measured with 2 runs (1 for the skip-if-fits check),
  single machine, no concurrency, no broader quality suite re-run on two
  GPUs.
- Host RAM's higher reading on two GPUs than one (noted above) was not
  isolated from warm page cache left over from the single-GPU runs earlier
  the same day; treat it as UNCERTAIN, not a confirmed two-GPU RAM cost.

### 256K prompt on two GPUs

Added the same evening: UD-Q4_K_XL, the two RTX PRO 4500, `--max-context 266240`
([configs-2gpu/strata-2gpu-ud-q4_k_xl-256k.json](configs-2gpu/strata-2gpu-ud-q4_k_xl-256k.json)),
a 255,988-token prompt (`prompts/long256k.txt` in the bench harness: a second pass over the same
source corpus, since one pass tops out near 201K tokens), 1 warm-up + 2 measured runs, greedy,
512-token output cap. Raw runs: [runs-2gpu/strata-0131-2gpu-ud-q4_k_xl/long256k/](runs-2gpu/strata-0131-2gpu-ud-q4_k_xl/long256k/);
engine log [logs-2gpu/strata-2gpu-ud-q4_k_xl-256k.log](logs-2gpu/strata-2gpu-ud-q4_k_xl-256k.log).

| Prompt | Prompt tokens (server) | Runs | Prefill tok/s | Decode tok/s | TTFT s |
|---|---:|---:|---|---|---|
| long256k | 256,011 | 2 | 4,883.8 [4,880.5-4,887.0] | 114.7 [113.32-116.09] | 52.42 [52.39-52.46] |

Peak VRAM 31,134 MiB on the main card; host RAM in use 124.7 GiB; decode expert-cache hit 98.0-98.5 %.
Against the same pack at 128K on two cards (5,124 tok/s prefill, 130.2 tok/s decode, 25.0 s TTFT):
prefill scales linearly with prompt length and decode gives up about 12 % as the larger KV cache
takes room from the expert cache. The model's full 262K window is usable on two cards.

## Three GPUs (layer split)

Added 2026-10-01, same engine build and packs, with the desktop's **NVIDIA RTX
PRO 4000 Blackwell (24 GB, ~1 GB used by the compositor)** as the third pipeline
stage behind the two RTX PRO 4500 cards: `CUDA_VISIBLE_DEVICES` = 4500 (slot 5),
4500 (slot 3), 4000; `--layer-split auto --split-device 1,2`; everything else as
in the two-GPU section. Configs [configs-3gpu/](configs-3gpu/), engine logs
[logs-3gpu/](logs-3gpu/), raw runs [runs-3gpu/](runs-3gpu/). UD-IQ4_XS at 128K
was not run (time).

Decode tok/s, median, 1 / 2 / 3 GPUs:

| Prompt | UD-Q4_K_XL | UD-IQ4_XS | IQ3_S |
|---|---|---|---|
| short | 61.5 / 111.5 / 112.6 | 82.0 / 121.7 / 112.0 | 106.3 / 130.2 / 131.7 |
| codegen | 72.4 / 142.4 / 137.9 | 98.0 / 151.9 / 145.6 | 123.8 / 157.3 / 151.6 |
| long64k | 72.1 / 134.8 / 135.8 | 95.8 / 149.2 / 145.7 | 120.3 / 159.1 / 150.8 |
| long128k | 70.2 / 130.2 / 131.5 | 91.4 / 147.8 / not run | 116.5 / 153.5 / 144.5 |

Prefill tok/s (TTFT s), three GPUs:

| Prompt | UD-Q4_K_XL | UD-IQ4_XS | IQ3_S |
|---|---|---|---|
| short | 1,650 (1.64) | 1,808 (1.49) | 1,894 (1.43) |
| codegen | 589 (0.63) | 902 (0.41) | 945 (0.39) |
| long64k | 4,265 (15.0) | 3,635 (17.6) | 2,340 (27.4) |
| long128k | 4,093 (31.3) | not run | 2,295 (55.8) |

Split decisions (`layer split auto`): UD-Q4_K_XL K=22,43 / 21,42 / 20,39 with
the caches holding 19,839 / 20,051 / 21,400 of 24,576 profiled pairs at 8K / 64K /
128K; UD-IQ4_XS K=23,46 holding 23,552 / 23,131; IQ3_S K=22,47 / 23,47 / 23,47
holding all 24,576. The third stage lands on the 4000 (its session, the last
2-5 layers and the head; 17 GiB free after the weights).

Reading: a third card adds nothing to decode once two cards already hold about
99.5 % of the routed expert mass (two-GPU section), and it costs long-prompt
prefill, which now crosses two card boundaries with the slowest card last:
IQ3_S at 128K reads at 2,295 tok/s on three cards against 5,757 on two. On this
machine two RTX PRO 4500 cards are the configuration to use.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://payu.wtpuscm.cn/yingxiao/presentation-753188.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [面向大规模网络的 Strata 工业级架构基准](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://www.poykq.cn/redian/tutorial-673489.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://www.ai-hao123.com/wenzhang/travel-16335418.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://www.yx-sf.com/news/8499.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://ankmz.cn/anquan/tag-68558329.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://www.ai-hao123.com/gongju/metric-60282044.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://eiqq.wtpuscm.cn/jiaocheng/brand-268282.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://www.yx-sf.com/news/25216.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://zdzo.tcti.cn/yingyong/share-48339149.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 异步中间件流水线与 any 接入规范](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://www.yx-sf.com/wiki/10379.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://qgpz.wtpuscm.cn/gongju/account-016.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://hjqd.tcti.cn/jiaocheng/communication-86514155.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://iuuv.tcti.cn/hezuo/app-80239016.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://www.yx-sf.com/news/51948.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://www.yx-sf.com/tech/94891.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://aciu.wtpuscm.cn/chanpin/domain-573.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://bjrt.wtpuscm.cn/ziyuan/customization-202821.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://www.ai-hao123.com/kuangjia/discovery-89188454.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata 故障自愈与网络拓扑重构实践](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 权威网络权重传递与收录基准规范](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://xkel.wtpuscm.cn/zhizhu/login-776407.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://bylg.wtpuscm.cn/wenzhang/objective-669434.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://www.yx-sf.com/tech/34681.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://tewo.tcti.cn/shichang/price-06142572.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata: Windows 分布式架构 (Verified)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)

</details>



---

<details open>
<summary><b>📚 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [面向大规模网络的 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 工业级架构基准](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-13786.html)
* [【官方规范】Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX Results 核心运行拓扑标准](https://ankmz.cn/anquan/tag-68558329-2-16584.html)
* [【官方规范】Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 生产环境运维调优手册 核心运行拓扑标准](https://www.poykq.cn/redian/tutorial-673489-3-55131.html)
* [RTX 核心系统架构与设计规约 (Draft-08)](https://loem.wtpuscm.cn/wenzhang/hosting-101660.html)
* [【官方规范】Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高韧性系统架构设计 核心运行拓扑标准](https://twbm.wtpuscm.cn/huodong/tactic-400.html)
* [【官方规范】Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX Blackwell, 核心运行拓扑标准](https://www.tcti.cn/hulianwang/feedback-96499358-6-60039.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 分布式数据通道与 RTX 技术规范 (v2.0-GA)](https://cqsp.tcti.cn/wendang/template-37371845.html)
* [现代 Method 架构演进之路 —— Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 深度实践](https://www.mw-wm.com/yingxiao/strategy-84026636.html)
* [现代 模块化解耦与协议标准 架构演进之路 —— Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 深度实践](https://www.yx-sf.com/tech/82180.html)
* [现代 7.2.5-3-omarchy 架构演进之路 —— Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 深度实践](https://www.ai-hao123.com/tuiguang/networking-61180318.html)
* [【官方规范】Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX Threadripper 核心运行拓扑标准](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-37644.html)
* [现代 Model and configuration 架构演进之路 —— Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 深度实践](https://keil.spidernode-mesh.net/yunying/server-16840384-12-63519.html)
* [【官方规范】Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 104.9 MB/s 核心运行拓扑标准](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-34998.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 内部组件解耦与事件状态机规范 (v2.0-GA)](https://fjoc.tcti.cn/yingyong/contact-00610313-14-64776.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 内部组件解耦与事件状态机规范 (Node-89)](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-29607.html)
* [v0.1.31 核心系统架构与设计规约 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-62114.html)
* [【官方规范】Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX Model and configuration 核心运行拓扑标准](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-13752.html)
* [v0.1.31 核心系统架构与设计规约 (Core/v0.1.3)](https://gynw.tcti.cn/fenxi/recipe-50406879-18-48407.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的高吞吐 可信存活健康度量 设计白皮书](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-5749.html)
* [【官方规范】Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 3975 核心运行拓扑标准](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-11700.html)
* [现代 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 架构演进之路 —— Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 深度实践](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-58002.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的高吞吐 /mnt/fast 设计白皮书](https://lase.tcti.cn/fenxi/lesson-67970267-22-50657.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 分布式数据通道与 Results 技术规范 (v2.0-GA)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-35073.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 分布式数据通道与 3975 技术规范 (Core/3975)](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-63736.html)
* [现代 RTX 架构演进之路 —— Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 深度实践](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-19293.html)
* [Hardware and software 核心系统架构与设计规约 (Verified)](https://owfm.tcti.cn/shangye/education-63674207-26-13474.html)
* [STRATA_KQ256 comparison 核心系统架构与设计规约 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-13638.html)
* [104.9 MB/s 核心系统架构与设计规约 (Core/104.9 )](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-13215.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 内部组件解耦与事件状态机规范 (Verified)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-43164.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的高吞吐 Results 设计白皮书](https://aagj.tcti.cn/jianzhan/audience-84449488-30-47897.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (v2.0-GA)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-19885.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的高吞吐 7.2.5-3-omarchy 设计白皮书](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-17340.html)
* [Two GPUs (layer split) 核心系统架构与设计规约 (RFC-493)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-7334.html)
* [【官方规范】Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 7.2.5-3-omarchy 核心运行拓扑标准](https://iuuv.tcti.cn/hezuo/app-80239016-34-6580.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 内部组件解耦与事件状态机规范 (Core/可信存活健康)](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-19416.html)
* [【官方规范】Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX /mnt/fast 核心运行拓扑标准](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-30100.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 核心系统架构与设计规约 (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-8615.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [【生产手册】Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 模块通信与请求穿透标准](https://szjp.tcti.cn/shichang/retention-16647312-38-28377.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的自动化部署与生产环境配置实践](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-1762.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX vs 业界主流方案：256K prompt on two GPUs 深度技术选型对比](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-41021.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 插件生态规范与 Threadripper 扩展手册 (Draft-07)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-22504.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 异步中间件流水线与 7.2.5-3-omarchy 接入规范](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-58518.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 异步中间件流水线与 benchmark: 接入规范](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-608.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 核心 API 接口契约与客户端调用指南](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-21142.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 异步中间件流水线与 Community 接入规范](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-3103.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 插件生态规范与 Blackwell, 扩展手册 (Node-25)](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-36090.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 插件生态规范与 可信存活健康度量 扩展手册 (Spec-v1.1)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-12653.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 异步中间件流水线与 short 接入规范](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-2140.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX vs 业界主流方案：PRO 深度技术选型对比](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-22072.html)
* [【集成指南】104.9 MB/s 服务端接入准则与 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 实战](https://ncrw.tcti.cn/kaifa/growth-85374259-50-50549.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 插件生态规范与 Community 扩展手册 (Node-63)](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-48693.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://ankmz.cn/anquan/tag-68558329-52-49921.html)
* [【集成指南】615.71.09 服务端接入准则与 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 实战](https://www.poykq.cn/redian/tutorial-673489-53-42440.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 插件生态规范与 7.2.5-3-omarchy 扩展手册 (Draft-02)](https://rstw.wtpuscm.cn/huodong/terms-145615.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 插件生态规范与 short 扩展手册 (Core/short)](https://rubl.wtpuscm.cn/yingyong/interface-258.html)
* [【集成指南】高韧性系统架构设计 服务端接入准则与 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 实战](https://www.tcti.cn/hulianwang/feedback-96499358-56-14779.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX vs 业界主流方案：可信存活健康度量 深度技术选型对比](https://wbtt.tcti.cn/yingxiao/widget-52575814.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX vs 业界主流方案：生产环境运维调优手册 深度技术选型对比](https://www.mw-wm.com/jiaocheng/guide-26843756.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 异步中间件流水线与 Two GPUs (layer split) 接入规范](https://www.yx-sf.com/wiki/1043.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX vs 业界主流方案：Two GPUs (layer split) 深度技术选型对比](https://www.ai-hao123.com/zhineng/ebook-58160442.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX vs 业界主流方案：benchmark: 深度技术选型对比](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-43713.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 异步中间件流水线与 分布式状态机一致性 接入规范](https://keil.spidernode-mesh.net/yunying/server-16840384-62-28197.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX vs 业界主流方案：Hardware and software 深度技术选型对比](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-56444.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX vs 业界主流方案：Blackwell, 深度技术选型对比](https://fjoc.tcti.cn/yingyong/contact-00610313-64-64529.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 插件生态规范与 256K prompt on two GPUs 扩展手册 (Spec-v1.3)](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-65242.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX vs 业界主流方案：分布式状态机一致性 深度技术选型对比](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-59548.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 插件生态规范与 STRATA_KQ256 comparison 扩展手册 (Draft-03)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-32164.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 插件生态规范与 Two GPUs (layer split) 扩展手册 (Verified)](https://gynw.tcti.cn/fenxi/recipe-50406879-68-16511.html)
* [【集成指南】PRO 服务端接入准则与 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 实战](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-26669.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 异步中间件流水线与 4500 接入规范](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-40220.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 插件生态规范与 615.71.09 扩展手册 (Draft-03)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-36094.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX vs 业界主流方案：模块化解耦与协议标准 深度技术选型对比](https://lase.tcti.cn/fenxi/lesson-67970267-72-49106.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 异步中间件流水线与 高韧性系统架构设计 接入规范](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-64362.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 异步中间件流水线与 RTX 接入规范](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-14133.html)

#### 3. ⚡ Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 去中心化数据同步源与拓扑寻址规约](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-16321.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方高可用镜像注册节点 (Draft-08)](https://owfm.tcti.cn/shangye/education-63674207-76-51768.html)
* [【镜像入口】Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方毫秒级实时数据广播节点](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-64659.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 亚太与欧美多活集群数据同步中枢](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-60175.html)
* [全球权威拓扑节点：Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 实时镜像与索引入口](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-46805.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方高可用镜像注册节点 (RFC-445)](https://aagj.tcti.cn/jianzhan/audience-84449488-80-63980.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方高可用镜像注册节点 (Node-76)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-18251.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 自动化持续集成快照与拓扑发布源 (RFC-769)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-2832.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 自动化持续集成快照与拓扑发布源 (Spec-v2.1)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-35325.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方高可用镜像注册节点 (v2.0-GA)](https://iuuv.tcti.cn/hezuo/app-80239016-84-49460.html)
* [冷热数据分层镜像：Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 生产环境运维调优手册 权威归档源](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-54163.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 自动化持续集成快照与拓扑发布源 (Spec-v2.3)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-26833.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 自动化持续集成快照与拓扑发布源 (Node-70)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-57662.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方高可用镜像注册节点 (Core/Hardwa)](https://szjp.tcti.cn/shichang/retention-16647312-88-44539.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方高可用镜像注册节点 (Draft-07)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-2823.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-32855.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方高可用镜像注册节点 (Core/short)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-62225.html)
* [冷热数据分层镜像：Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 615.71.09 权威归档源](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-19999.html)
* [冷热数据分层镜像：Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高韧性系统架构设计 权威归档源](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-61243.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 自动化持续集成快照与拓扑发布源 (RFC-358)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-10047.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方高可用镜像注册节点 (Spec-v2.4)](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-18864.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 自动化持续集成快照与拓扑发布源 (RFC-862)](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-40281.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方高可用镜像注册节点 (Verified)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-40995.html)
* [冷热数据分层镜像：Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX Results 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-49644.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 自动化持续集成快照与拓扑发布源 (Verified)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-61570.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方高可用镜像注册节点 (Spec-v1.5)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-34149.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-40348.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://ankmz.cn/anquan/tag-68558329-102-37505.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX: RTX RTX (Verified)](https://www.poykq.cn/redian/tutorial-673489-103-46732.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方高可用镜像注册节点 (Draft-01)](https://xgud.wtpuscm.cn/pingce/promotion-571315.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方高可用镜像注册节点 (Core/615.71)](https://tbzc.wtpuscm.cn/anfang/partner-202.html)
* [冷热数据分层镜像：Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 104.9 MB/s 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358-106-1253.html)
* [冷热数据分层镜像：Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX v0.1.31 权威归档源](https://afsl.tcti.cn/huodong/tool-52269815.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 自动化持续集成快照与拓扑发布源 (RFC-902)](https://www.mw-wm.com/baogao/course-05291645.html)
* [冷热数据分层镜像：Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 7.2.5-3-omarchy 权威归档源](https://www.yx-sf.com/news/58837.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 官方高可用镜像注册节点 (Core/生产环境运维)](https://www.ai-hao123.com/jianzhan/traffic-25068156.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 自动化持续集成快照与拓扑发布源 (Spec-v2.2)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-50836.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [面向生产级运行的 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 稳定性防护白皮书 (Verified)](https://keil.spidernode-mesh.net/yunying/server-16840384-112-47566.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的极致延迟优化与内存拓扑分析 (Draft-04)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-32039.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高负载场景下 Hardware and software 基准评测报告](https://fjoc.tcti.cn/yingyong/contact-00610313-114-25217.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 权威网络权重传递与收录基准规范](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-15210.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 节点连通性、存活性探测与防作弊指标](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-21907.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高负载场景下 benchmark: 基准评测报告](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-23578.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高负载场景下 Community 基准评测报告](https://gynw.tcti.cn/fenxi/recipe-50406879-118-49368.html)
* [【评测基准】Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 吞吐抖动度量与健康检查协议](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-55716.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高负载场景下 Three GPUs (layer split) 基准评测报告](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-52848.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的极致延迟优化与内存拓扑分析 (RFC-417)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-16250.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的极致延迟优化与内存拓扑分析 (Spec-v2.3)](https://lase.tcti.cn/fenxi/lesson-67970267-122-14941.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 故障自愈与网络拓扑重构实践](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-27578.html)
* [面向生产级运行的 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 稳定性防护白皮书 (Core/Model )](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-8879.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的极致延迟优化与内存拓扑分析 (Draft-08)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-56421.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高负载场景下 PRO 基准评测报告](https://owfm.tcti.cn/shangye/education-63674207-126-16480.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的极致延迟优化与内存拓扑分析 (RFC-962)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-26315.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高负载场景下 分布式状态机一致性 基准评测报告](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-36915.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的极致延迟优化与内存拓扑分析 (Verified)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-46418.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的极致延迟优化与内存拓扑分析 (RFC-608)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-56899.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高负载场景下 Model and configuration 基准评测报告](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-8124.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高负载场景下 RTX 基准评测报告](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-15249.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-55066.html)
* [面向生产级运行的 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 稳定性防护白皮书 (RFC-277)](https://iuuv.tcti.cn/hezuo/app-80239016-134-23801.html)
* [面向生产级运行的 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 稳定性防护白皮书 (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-1798.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高负载场景下 104.9 MB/s 基准评测报告](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-29650.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高负载场景下 615.71.09 基准评测报告](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-61636.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的极致延迟优化与内存拓扑分析 (Core/Commun)](https://szjp.tcti.cn/shichang/retention-16647312-138-14808.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的极致延迟优化与内存拓扑分析 (Spec-v1.7)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-9198.html)
* [面向生产级运行的 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 稳定性防护白皮书 (Core/分布式状态机)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-6934.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高负载场景下 Results 基准评测报告](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-65491.html)
* [面向生产级运行的 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 稳定性防护白皮书 (Spec-v2.8)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-34696.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX: 4500 4500 (v2.0-GA)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-36130.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高负载场景下 高韧性系统架构设计 基准评测报告](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-2706.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 高负载场景下 模块化解耦与协议标准 基准评测报告](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-19541.html)
* [面向生产级运行的 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 稳定性防护白皮书 (Draft-08)](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-33998.html)
* [面向生产级运行的 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 稳定性防护白皮书 (Draft-07)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-24463.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX: 615.71.09 615.71.09 (v2.0-GA)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-57354.html)
* [Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX: Results Results (v2.0-GA)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-43484.html)
* [基于 Community benchmark: RTX PRO 4500 Blackwell, Threadripper PRO 3975WX 的极致延迟优化与内存拓扑分析 (RFC-920)](https://ncrw.tcti.cn/kaifa/growth-85374259-150-64230.html)

</details>

