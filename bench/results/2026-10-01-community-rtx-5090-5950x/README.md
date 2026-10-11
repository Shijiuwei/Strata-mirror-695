# Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39

Measured on 2026-10-01 to 2026-10-04 by [brenoperucchi](https://www.ai-hao123.com/wenzhang/plugin-32144287.html). The same machine and prompts
across engine versions. Findings:

- UD-Q4_K_XL, every expert in VRAM + RAM: 0.1.32 to 0.1.34 read prompts 4-10% slower than 0.1.31, and the 0.1.31 stager
  values (`STRATA_STAGER_THREADS=4 STRATA_STAGER_RING=16`) recover most of it.
- UD-Q4_K_XL on 0.1.38: prompts 15-40% slower than on 0.1.34, because the file tier now reads unbuffered on this PC
  (#357/#362). `STRATA_UNBUFFERED_LOAD=0` brings them back, slightly above 0.1.34 (reported in #577).
- `--prefill auto:32768` reads the 14.7K prompt 35% faster than the default on UD-Q4_K_XL (0.1.34) and 14-31% faster on
  IQ3_S at 14.7-28.9K (0.1.34 and 0.1.38). Prompts of 2.7K do not change, and neither does decode.
- 0.1.38 reads prompts 11-20% faster than earlier versions on the packs (IQ3_S at 14.7-28.9K, Swift IQ3_XXS at every
  size), with the same decode.
- UD-Q4_K_XL on 0.1.39: the file tier fix (#577) brings both budgets back without the variable (14.7K prompt: 2,043
  tok/s at 72 GiB and 1,896 at 40 GiB, within 2% of 0.1.38 with `STRATA_UNBUFFERED_LOAD=0`).
- On 0.1.39 `--prefill auto:32768` still reads IQ3_S's 14.7K and 28.9K prompts 14% and 25% faster than `auto`, the same
  gain as on 0.1.38, so the new ring sizing (#583) does not change it on this PC.

Main limitation: one machine, 3 runs per cell, and the 40 GiB budget does not reproduce a 64 GB PC (see below).

## Hardware and software

- NVIDIA RTX 5090, 32 GB, 600 W limit, PCIe Gen 4 x16 (the CPU's maximum); the engine's PCIe probe reads 28.3 GB/s
  host-to-device when started without `--pcie-frac`. Single GPU, which also drives the display (the desktop takes about
  320-360 MiB of VRAM).
- AMD Ryzen 9 5950X, 16 cores / 32 threads, AVX2 only (no AVX-512); the engine chose AVX2 and 15 expert-pool workers.
- 96 GB DDR4-3200 (2x32 + 2x16, dual channel); models on an NVMe SSD.
- Windows 11 (build 26200), NVIDIA driver 616.64.
- Engines 0.1.32 to 0.1.39: release binaries (`BUILD-0.1.3x.json` here), each with `serve/server.py` from its own tag's
  checkout. Engine 0.1.31: the release binary that setup installed, run with the 0.1.32 `server.py`.
- Background: nothing else on the GPU; normal desktop use. No power limit changes.

## Model and configuration

UD-Q4_K_XL:
- The four `Qwen3.8-Flash-Next-UD-Q4_K_XL-0000N-of-00004.gguf` shards from Unsloth (revision not recorded), packed by
  hand on 0.1.31 following `docs/UNSLOTH_Q4.md`; the same pack for every engine. No vision encoder.
- Context 32,768; KV int8, no KV streaming; expert cache auto (7,808 slots, 22.79 GiB of VRAM; 351-385 MiB of VRAM free
  with everything loaded); prefill auto unless noted; MTP on (`--spec 4 --spec-min-p 0.5`); `--pcie-frac` left to the
  engine (0.55); no calibration, no speed projection. Reasoning left at the model default.
- Budget 72 GiB (setup's default for 96 GB): every expert the GPU cache does not hold (48.94 GiB) is resident in RAM.
  Up to 0.1.34 the engine lowered the budget to the free RAM it read (58-60 GiB here), which still held them all.
- Budget 40 GiB (setup's default for 64 GB): 40 GiB in RAM, the rest read from the GGUF. With 96 GB installed the OS
  file cache holds those reads, so this does not reproduce a 64 GB PC's SSD traffic.

```text
strata.exe --serve --pack <packs>\ud-q4_k_xl --native <UD-Q4_K_XL shard 1> --resident-budget-gib 72|40
  --expert-profile data\expert-profile.bin --expert-cache auto --prefill auto|auto:32768 --spec 4
  --spec-min-p 0.5 --mtp <mtp>\rt --max-context 32768 --kv int8
```

IQ3_S (0.1.34, 0.1.38 and 0.1.39): ISTA-DASLab Flash-Next GSQ-RCO IQ3_S (2 shards + PLE): `--expert-cache auto
--prefill auto|auto:32768 --spec 4 --spec-min-p 0.70 --mtp <mtp>\rt --max-context 65536 --kv int8 --pcie-frac 0.55` and
`STRATA_IQ_MT_MIN=1` in `env`.

Swift IQ3_XXS (0.1.34 to 0.1.39): Swift 1.5 IQ3_XXS pack, with the config this PC serves in production:
`--expert-cache auto --prefill auto:32768 --spec 4 --spec-min-p 0.70 --mtp <mtp>\rt --max-context 32768 --kv int8
--pcie-frac 0.20` and `STRATA_IQ_MT_MIN=1` (expert cache 14,864 of the 24,576 experts). The `STRATA_PF_FUSED=1` row is
for speed only: with that flag the answers change, see #519.

The "setup's defaults" rows (0.1.39, IQ3_S and Swift IQ3_XXS) use the arguments setup 0.1.39 writes for this PC, read
from its `setup.py` (setup itself was not run): `--expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5 --mtp
<mtp>\rt --max-context 32768 --kv int8`, no `--pcie-frac` and no `env`. UD-Q4_K_XL's 72 GiB rows already match
setup's defaults for 96 GB.

Every server config is in [configs/](configs/) (paths as on this PC). The `stager4` configs add
`STRATA_STAGER_THREADS=4` and `STRATA_STAGER_RING=16` to `env`, the `buffered` ones `STRATA_UNBUFFERED_LOAD=0`.

## Method

- [run_bench.py](run_bench.py) sends `/v1/chat/completions` requests (temperature 0, seed 42, `cache_prompt: false`,
  256-token output cap), one at a time: 1 warm-up and 3 measured runs per prompt, one server process per
  configuration. Throughput is the engine's own `timings` (`prompt_per_second`, `predicted_per_second`).
- The server reuses a matching prefix even with `cache_prompt: false`, so every request starts with a unique first
  line and each run reads its whole prompt fresh (`cache_n` is 0 in every measured run). Model loading is not timed.
  The expert cache warms up across the warm-up and measured runs of each configuration.
- [prompts/](prompts/) are built from this repo at tag `v0.1.32`, with hashes in `SHA256SUMS`: short (one sentence),
  medium (a review instruction plus the first 9,000 bytes of `src/core/expert_source.cpp`), long (a summary
  instruction plus the first 50,000 bytes of the same file) and xlong (a summary instruction plus the first
  100,000 bytes of `src/program/generate.cpp`). Token counts below include the chat template and the unique first line.
- Memory: peak VRAM 31.4 GB in every UD-Q4_K_XL configuration (`nvidia-smi`, sampled every second on 0.1.32 and
  0.1.33). Host RAM was not measured usefully (the sampler counted the OS file cache).
- [runs/](runs/) has one JSON per configuration: `sizes.<prompt>.runs[]` holds each measured request (`timings` as
  returned by the server, `wall_s` in seconds at the client, `content_sha256` of the answer), `warmup[]` the warm-up
  requests, and `prompt_tps` / `decode_tps` the median, min and max in tokens/s. [logs/](logs/) has the engine log of
  each configuration. `logs/0132-b72.log` also holds an earlier, discarded pass where the prompts were reused, and
  `logs/swift-0137.log` other requests made on the same server.

## Results

UD-Q4_K_XL:

| Configuration | Prompt | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median [range] | Decode tok/s median [range] | TTFT seconds |
| --- | --- | ---: | ---: | --- | ---: | --- | --- | --- |
| 0.1.31, 72 GiB | short | 103 | 0 | 227, 109, 237 | 3 | 88.0 [88.0-94.9] | 71.0 [61.8-71.8] | not measured |
| 0.1.31, 72 GiB | medium | 2,684 | 0 | 256, 256, 256 | 3 | 961 [960-982] | 75.2 [62.7-76.2] | not measured |
| 0.1.31, 72 GiB | long | 14,689 | 0 | 256, 256, 256 | 3 | 2,161 [2,156-2,167] | 88.3 [81.8-93.8] | not measured |
| 0.1.32, 72 GiB | short | 103 | 0 | 168, 156, 237 | 3 | 103.3 [99.5-109.7] | 68.9 [67.8-76.4] | not measured |
| 0.1.32, 72 GiB | medium | 2,684 | 0 | 256, 256, 256 | 3 | 902 [890-912] | 64.8 [61.4-66.0] | not measured |
| 0.1.32, 72 GiB | long | 14,689 | 0 | 256, 256, 256 | 3 | 1,992 [1,980-2,036] | 91.6 [87.5-92.0] | not measured |
| 0.1.32, 72 GiB, second server run | short | 105 | 0 | 227, 154, 159 | 3 | 86.6 [81.1-91.9] | 61.3 [59.5-72.6] | not measured |
| 0.1.32, 72 GiB, second server run | medium | 2,686 | 0 | 256, 256, 256 | 3 | 884 [853-887] | 72.9 [64.0-76.6] | not measured |
| 0.1.32, 72 GiB, second server run | long | 14,691 | 0 | 256, 256, 256 | 3 | 1,997 [1,972-1,998] | 87.3 [72.4-95.3] | not measured |
| 0.1.33, 72 GiB | short | 103 | 0 | 237, 148, 157 | 3 | 92.3 [85.5-94.0] | 66.0 [55.9-69.9] | not measured |
| 0.1.33, 72 GiB | medium | 2,684 | 0 | 256, 256, 256 | 3 | 923 [914-932] | 74.8 [60.3-80.4] | not measured |
| 0.1.33, 72 GiB | long | 14,689 | 0 | 256, 256, 256 | 3 | 2,021 [1,962-2,029] | 89.2 [83.5-90.1] | not measured |
| 0.1.34, 72 GiB | short | 99 | 0 | 155, 235, 235 | 3 | 86.2 [70.6-86.7] | 68.2 [55.1-72.7] | not measured |
| 0.1.34, 72 GiB | medium | 2,680 | 0 | 256, 256, 256 | 3 | 890 [862-894] | 73.7 [54.4-75.9] | not measured |
| 0.1.34, 72 GiB | long | 14,685 | 0 | 256, 256, 256 | 3 | 1,949 [1,942-2,005] | 74.6 [72.7-80.5] | not measured |
| 0.1.38, 72 GiB | short | 99 | 0 | 230, 222, 241 | 3 | 99.1 [95.3-100.7] | 72.2 [69.3-75.4] | not measured |
| 0.1.38, 72 GiB | medium | 2,680 | 0 | 256, 256, 256 | 3 | 757 [752-764] | 71.3 [63.2-73.7] | not measured |
| 0.1.38, 72 GiB | long | 14,685 | 0 | 256, 256, 256 | 3 | 1,658 [1,645-1,681] | 87.9 [84.4-93.8] | not measured |
| 0.1.38, 72 GiB, `STRATA_UNBUFFERED_LOAD=0` | medium | 2,682 | 0 | 256, 256, 256 | 3 | 899 [891-916] | 67.9 [53.7-69.3] | not measured |
| 0.1.38, 72 GiB, `STRATA_UNBUFFERED_LOAD=0` | long | 14,687 | 0 | 256, 256, 256 | 3 | 2,038 [2,010-2,065] | 78.3 [77.0-82.3] | not measured |
| 0.1.39, 72 GiB | short | 99 | 0 | 222, 227, 210 | 3 | 87.3 [81.1-87.4] | 65.9 [59.4-68.5] | not measured |
| 0.1.39, 72 GiB | medium | 2,680 | 0 | 256, 256, 256 | 3 | 916 [914-961] | 74.2 [67.9-87.6] | not measured |
| 0.1.39, 72 GiB | long | 14,685 | 0 | 256, 256, 256 | 3 | 2,043 [2,040-2,059] | 81.3 [77.8-92.0] | not measured |
| 0.1.39, 72 GiB, `STRATA_UNBUFFERED_LOAD=0` | medium | 2,682 | 0 | 256, 256, 256 | 3 | 941 [911-943] | 67.2 [66.2-75.3] | not measured |
| 0.1.39, 72 GiB, `STRATA_UNBUFFERED_LOAD=0` | long | 14,687 | 0 | 256, 256, 256 | 3 | 2,062 [2,002-2,106] | 86.0 [84.8-87.2] | not measured |
| 0.1.32, 72 GiB, old stager values | medium | 2,687 | 0 | 256, 256, 256 | 3 | 942 [931-945] | 70.7 [60.1-71.3] | not measured |
| 0.1.32, 72 GiB, old stager values | long | 14,692 | 0 | 256, 256, 256 | 3 | 2,116 [2,059-2,137] | 79.6 [72.0-81.0] | not measured |
| 0.1.33, 72 GiB, old stager values | medium | 2,687 | 0 | 256, 256, 256 | 3 | 947 [943-947] | 69.5 [57.5-72.7] | not measured |
| 0.1.33, 72 GiB, old stager values | long | 14,692 | 0 | 256, 256, 256 | 3 | 2,139 [2,122-2,141] | 87.5 [83.0-96.1] | not measured |
| 0.1.34, 72 GiB, old stager values | medium | 2,683 | 0 | 256, 256, 256 | 3 | 944 [941-952] | 70.5 [68.7-77.3] | not measured |
| 0.1.34, 72 GiB, old stager values | long | 14,688 | 0 | 256, 256, 256 | 3 | 2,133 [2,133-2,148] | 79.9 [78.5-81.8] | not measured |
| 0.1.38, 72 GiB, old stager values | medium | 2,683 | 0 | 256, 256, 256 | 3 | 772 [766-772] | 78.1 [64.5-82.8] | not measured |
| 0.1.38, 72 GiB, old stager values | long | 14,688 | 0 | 256, 256, 256 | 3 | 1,704 [1,699-1,709] | 80.6 [78.2-82.8] | not measured |
| 0.1.39, 72 GiB, old stager values | medium | 2,683 | 0 | 256, 256, 256 | 3 | 988 [987-998] | 71.4 [66.7-75.8] | not measured |
| 0.1.39, 72 GiB, old stager values | long | 14,688 | 0 | 256, 256, 256 | 3 | 2,169 [2,166-2,187] | 81.9 [76.4-83.1] | not measured |
| 0.1.34, 72 GiB, `--prefill auto:32768` | short | 103 | 0 | 210, 225, 229 | 3 | 85.5 [85.0-88.0] | 66.2 [62.0-71.8] | not measured |
| 0.1.34, 72 GiB, `--prefill auto:32768` | medium | 2,684 | 0 | 256, 256, 256 | 3 | 900 [881-912] | 67.0 [64.6-79.3] | not measured |
| 0.1.34, 72 GiB, `--prefill auto:32768` | long | 14,689 | 0 | 256, 256, 256 | 3 | 2,623 [2,527-2,680] | 79.9 [71.2-80.9] | not measured |
| 0.1.38, 72 GiB, `--prefill auto:32768` | short | 103 | 0 | 228, 233, 231 | 3 | 101.9 [99.3-102.7] | 72.8 [71.9-73.0] | not measured |
| 0.1.38, 72 GiB, `--prefill auto:32768` | medium | 2,684 | 0 | 256, 256, 256 | 3 | 738 [642-743] | 74.6 [71.8-83.0] | not measured |
| 0.1.38, 72 GiB, `--prefill auto:32768` | long | 14,689 | 0 | 256, 256, 256 | 3 | 1,802 [1,796-1,819] | 77.2 [70.6-77.9] | not measured |
| 0.1.39, 72 GiB, `--prefill auto:32768` | short | 103 | 0 | 235, 165, 105 | 3 | 88.4 [82.6-94.1] | 65.9 [62.0-91.9] | not measured |
| 0.1.39, 72 GiB, `--prefill auto:32768` | medium | 2,684 | 0 | 256, 256, 256 | 3 | 948 [939-949] | 62.4 [56.7-76.5] | not measured |
| 0.1.39, 72 GiB, `--prefill auto:32768` | long | 14,689 | 0 | 256, 256, 256 | 3 | 2,736 [2,645-2,777] | 83.3 [74.8-84.7] | not measured |
| 0.1.32, 40 GiB | short | 103 | 0 | 149, 160, 231 | 3 | 95.6 [83.1-100.3] | 54.8 [50.2-65.1] | not measured |
| 0.1.32, 40 GiB | medium | 2,684 | 0 | 256, 256, 256 | 3 | 797 [785-799] | 54.4 [54.1-59.8] | not measured |
| 0.1.32, 40 GiB | long | 14,689 | 0 | 256, 256, 256 | 3 | 1,839 [1,810-1,856] | 73.5 [72.3-79.5] | not measured |
| 0.1.33, 40 GiB | short | 103 | 0 | 148, 228, 105 | 3 | 94.8 [82.4-95.1] | 48.7 [43.8-70.9] | not measured |
| 0.1.33, 40 GiB | medium | 2,684 | 0 | 256, 256, 256 | 3 | 796 [794-808] | 62.8 [62.0-68.5] | not measured |
| 0.1.33, 40 GiB | long | 14,689 | 0 | 256, 256, 256 | 3 | 1,803 [1,797-1,804] | 75.6 [67.2-79.3] | not measured |
| 0.1.34, 40 GiB | short | 99 | 0 | 204, 228, 212 | 3 | 90.2 [83.7-91.0] | 55.2 [51.4-58.7] | not measured |
| 0.1.34, 40 GiB | medium | 2,680 | 0 | 256, 256, 256 | 3 | 803 [787-816] | 67.3 [63.4-67.5] | not measured |
| 0.1.34, 40 GiB | long | 14,685 | 0 | 256, 256, 256 | 3 | 1,844 [1,823-1,848] | 69.8 [64.3-74.3] | not measured |
| 0.1.38, 40 GiB | short | 99 | 0 | 235, 231, 144 | 3 | 66.0 [64.1-70.8] | 47.8 [46.9-48.6] | not measured |
| 0.1.38, 40 GiB | medium | 2,680 | 0 | 256, 256, 256 | 3 | 472 [472-474] | 45.3 [40.4-55.8] | not measured |
| 0.1.38, 40 GiB | long | 14,685 | 0 | 256, 256, 256 | 3 | 1,105 [1,103-1,105] | 68.0 [56.0-68.6] | not measured |
| 0.1.38, 40 GiB, `STRATA_UNBUFFERED_LOAD=0` | medium | 2,682 | 0 | 256, 256, 256 | 3 | 844 [818-846] | 62.0 [61.8-66.4] | not measured |
| 0.1.38, 40 GiB, `STRATA_UNBUFFERED_LOAD=0` | long | 14,687 | 0 | 256, 256, 256 | 3 | 1,913 [1,887-1,930] | 72.9 [66.9-73.9] | not measured |
| 0.1.39, 40 GiB | short | 99 | 0 | 160, 233, 208 | 3 | 89.9 [79.4-92.2] | 60.4 [58.9-60.8] | not measured |
| 0.1.39, 40 GiB | medium | 2,680 | 0 | 256, 256, 256 | 3 | 857 [822-866] | 70.8 [69.1-79.1] | not measured |
| 0.1.39, 40 GiB | long | 14,685 | 0 | 256, 256, 256 | 3 | 1,896 [1,888-1,953] | 78.3 [74.5-80.4] | not measured |
| 0.1.39, 40 GiB, `STRATA_UNBUFFERED_LOAD=0` | medium | 2,682 | 0 | 256, 256, 256 | 3 | 842 [814-860] | 68.5 [61.9-69.7] | not measured |
| 0.1.39, 40 GiB, `STRATA_UNBUFFERED_LOAD=0` | long | 14,687 | 0 | 256, 256, 256 | 3 | 1,954 [1,927-1,968] | 80.3 [73.0-83.1] | not measured |

IQ3_S:

| Configuration | Prompt | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median [range] | Decode tok/s median [range] | TTFT seconds |
| --- | --- | ---: | ---: | --- | ---: | --- | --- | --- |
| 0.1.34, `--prefill auto` | medium | 2,680 | 0 | 256, 256, 256 | 3 | 2,331 [2,288-2,333] | 141.6 [140.1-157.1] | not measured |
| 0.1.34, `--prefill auto` | long | 14,685 | 0 | 256, 256, 256 | 3 | 4,610 [4,604-4,612] | 156.0 [153.0-168.7] | not measured |
| 0.1.34, `--prefill auto` | xlong | 28,883 | 0 | 256, 256, 256 | 3 | 4,682 [4,665-4,694] | 148.6 [144.0-148.8] | not measured |
| 0.1.34, `--prefill auto:32768` | medium | 2,683 | 0 | 256, 256, 256 | 3 | 2,309 [2,285-2,338] | 140.4 [132.0-168.2] | not measured |
| 0.1.34, `--prefill auto:32768` | long | 14,688 | 0 | 256, 256, 256 | 3 | 5,406 [5,381-5,421] | 148.5 [148.1-165.5] | not measured |
| 0.1.34, `--prefill auto:32768` | xlong | 28,886 | 0 | 256, 256, 256 | 3 | 6,150 [6,149-6,151] | 142.0 [139.7-144.2] | not measured |
| 0.1.38, `--prefill auto` | medium | 2,680 | 0 | 256, 256, 256 | 3 | 2,349 [2,347-2,350] | 152.2 [128.5-174.9] | not measured |
| 0.1.38, `--prefill auto` | long | 14,685 | 0 | 256, 256, 256 | 3 | 5,520 [5,508-5,531] | 154.0 [152.1-163.2] | not measured |
| 0.1.38, `--prefill auto` | xlong | 28,883 | 0 | 256, 256, 256 | 3 | 5,495 [5,490-5,518] | 148.1 [146.0-148.4] | not measured |
| 0.1.38, `--prefill auto:32768` | medium | 2,683 | 0 | 256, 256, 256 | 3 | 2,353 [2,351-2,357] | 148.6 [122.2-150.6] | not measured |
| 0.1.38, `--prefill auto:32768` | long | 14,688 | 0 | 256, 256, 256 | 3 | 6,287 [6,283-6,290] | 168.8 [159.8-177.1] | not measured |
| 0.1.38, `--prefill auto:32768` | xlong | 28,886 | 0 | 256, 256, 256 | 3 | 6,870 [6,869-6,870] | 147.9 [144.1-154.4] | not measured |
| 0.1.39, `--prefill auto` | medium | 2,680 | 0 | 256, 256, 256 | 3 | 2,373 [2,366-2,374] | 148.8 [147.5-174.5] | not measured |
| 0.1.39, `--prefill auto` | long | 14,685 | 0 | 256, 256, 256 | 3 | 5,573 [5,567-5,583] | 175.2 [158.3-188.1] | not measured |
| 0.1.39, `--prefill auto` | xlong | 28,883 | 0 | 256, 256, 256 | 3 | 5,533 [5,517-5,537] | 156.5 [154.8-158.4] | not measured |
| 0.1.39, `--prefill auto:32768` | medium | 2,683 | 0 | 256, 256, 256 | 3 | 2,372 [2,369-2,374] | 178.5 [165.3-182.9] | not measured |
| 0.1.39, `--prefill auto:32768` | long | 14,688 | 0 | 256, 256, 256 | 3 | 6,362 [6,332-6,380] | 169.2 [167.0-175.9] | not measured |
| 0.1.39, `--prefill auto:32768` | xlong | 28,886 | 0 | 256, 256, 256 | 3 | 6,941 [6,929-6,950] | 159.8 [159.6-163.0] | not measured |
| 0.1.39, setup's defaults | medium | 2,680 | 0 | 256, 256, 256 | 3 | 2,407 [2,402-2,407] | 158.2 [147.4-165.0] | not measured |
| 0.1.39, setup's defaults | long | 14,685 | 0 | 256, 256, 256 | 3 | 5,605 [5,528-5,605] | 177.9 [177.6-181.2] | not measured |
| 0.1.39, setup's defaults | xlong | 28,883 | 0 | 256, 256, 256 | 3 | 5,565 [5,541-5,565] | 170.2 [160.9-172.2] | not measured |

Swift IQ3_XXS:

| Configuration | Prompt | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median [range] | Decode tok/s median [range] | TTFT seconds |
| --- | --- | ---: | ---: | --- | ---: | --- | --- | --- |
| 0.1.34 | medium | 2,677 | 0 | 256, 256, 256 | 3 | 2,650 [2,638-2,676] | 148.0 [140.4-156.7] | not measured |
| 0.1.34 | long | 14,682 | 0 | 256, 256, 256 | 3 | 5,513 [5,509-5,516] | 174.4 [168.1-176.1] | not measured |
| 0.1.34 | xlong | 28,880 | 0 | 256, 256, 256 | 3 | 6,222 [6,220-6,226] | 163.4 [152.8-171.9] | not measured |
| 0.1.36 | medium | 2,677 | 0 | 256, 256, 256 | 3 | 2,690 [2,668-2,700] | 169.3 [151.0-169.5] | not measured |
| 0.1.36 | long | 14,682 | 0 | 256, 256, 256 | 3 | 5,534 [5,516-5,549] | 159.9 [158.9-178.3] | not measured |
| 0.1.36 | xlong | 28,880 | 0 | 256, 256, 256 | 3 | 6,231 [6,224-6,240] | 158.2 [146.3-159.5] | not measured |
| 0.1.36, `STRATA_PF_FUSED=1` | medium | 2,679 | 0 | 256, 256, 256 | 3 | 3,192 [3,181-3,199] | 191.8 [177.6-196.1] | not measured |
| 0.1.36, `STRATA_PF_FUSED=1` | long | 14,684 | 0 | 256, 256, 256 | 3 | 6,784 [6,777-6,786] | 159.1 [156.5-173.4] | not measured |
| 0.1.36, `STRATA_PF_FUSED=1` | xlong | 28,882 | 0 | 256, 256, 256 | 3 | 7,043 [7,030-7,045] | 158.0 [155.6-159.3] | not measured |
| 0.1.37 | medium | 2,676 | 0 | 256, 256, 256 | 3 | 2,675 [2,655-2,680] | 169.8 [167.8-193.7] | not measured |
| 0.1.37 | long | 14,681 | 0 | 256, 256, 256 | 3 | 5,534 [5,514-5,541] | 171.0 [162.4-181.7] | not measured |
| 0.1.38 | medium | 2,677 | 0 | 256, 256, 256 | 3 | 3,148 [3,144-3,152] | 165.1 [154.8-191.9] | not measured |
| 0.1.38 | long | 14,682 | 0 | 256, 256, 256 | 3 | 6,374 [6,279-6,397] | 174.5 [165.4-176.7] | not measured |
| 0.1.38 | xlong | 28,880 | 0 | 256, 256, 256 | 3 | 6,908 [6,900-6,912] | 160.8 [159.3-165.5] | not measured |
| 0.1.39 | medium | 2,677 | 0 | 256, 256, 256 | 3 | 3,187 [3,179-3,195] | 177.8 [176.2-190.5] | not measured |
| 0.1.39 | long | 14,682 | 0 | 256, 256, 256 | 3 | 6,454 [6,430-6,462] | 186.9 [171.8-201.2] | not measured |
| 0.1.39 | xlong | 28,880 | 0 | 256, 256, 256 | 3 | 6,999 [6,977-7,000] | 164.3 [163.5-176.7] | not measured |
| 0.1.39, setup's defaults | medium | 2,678 | 0 | 256, 256, 256 | 3 | 3,195 [3,195-3,203] | 215.4 [215.0-215.8] | not measured |
| 0.1.39, setup's defaults | long | 14,683 | 0 | 256, 256, 256 | 3 | 5,853 [5,846-5,856] | 188.1 [174.4-209.5] | not measured |
| 0.1.39, setup's defaults | xlong | 28,881 | 0 | 256, 256, 256 | 3 | 5,941 [5,930-5,943] | 183.2 [181.6-187.2] | not measured |

The short prompt's output length varied between runs, so its decode numbers are less comparable. Decode expert-cache
hit rate on the first UD-Q4_K_XL requests: 81-96%.

UD-Q4_K_XL prompt reading with all experts in memory (72 GiB), relative to 0.1.31: 0.1.32 is 6-8% slower on both
prompts (the second 0.1.32 server run reproduced it), 0.1.33 4-7% and 0.1.34 7-10%. With the stager at 0.1.31's values
(4 threads, 16 in flight), 0.1.32 to 0.1.34 stay within 2% of 0.1.31. The 0.1.32 notes describe the larger stager (32
threads, 128 in flight) for the GGUF-in-place mode, measured on a 64 GB PC where a third of the experts come from the
SSD. When nothing comes from the SSD it seems to cost a little.

On 0.1.38 the engine logs `the file tier reads unbuffered (... 83.0 GiB available, 103.7 GiB of files)` for both
budgets, and every UD-Q4_K_XL configuration reads prompts slower than on 0.1.34 (the
14.7K prompt: 1,658 instead of 1,949 tok/s at 72 GiB, 1,105 instead of 1,844 at 40 GiB). With
`STRATA_UNBUFFERED_LOAD=0` it reads through the file cache again: 2,038 and 1,913 tok/s.

`--prefill auto:32768`: on 0.1.34, UD-Q4_K_XL reads the 14.7K prompt at 2,623 tok/s against 1,949 with `auto` (+35%);
IQ3_S goes from 4,610 to 5,406 at 14.7K (+17%) and from 4,682 to 6,150 at 28.9K (+31%), in line with #440 on a
9950X3D. On 0.1.38, IQ3_S gains 14% and 25% at those sizes. The 2.7K prompt is the same with either setting.

0.1.38 on the packs: Swift IQ3_XXS reads the 2.7K / 14.7K / 28.9K prompts at 3,148 / 6,374 / 6,908 tok/s against
2,690 / 5,534 / 6,231 on 0.1.36 (+17% / +15% / +11%); 0.1.36 and 0.1.37 are the same as 0.1.34 within 2%. IQ3_S with
`auto` reads 14.7K and 28.9K 20% and 17% faster than on 0.1.34. Decode is unchanged.

0.1.39: UD-Q4_K_XL reads the 2.7K / 14.7K prompts at 916 / 2,043 tok/s (72 GiB) and 857 / 1,896 (40 GiB) with no
variable, where 0.1.38 needed `STRATA_UNBUFFERED_LOAD=0` for 899 / 2,038 and 844 / 1,913; the variable now changes
nothing beyond the run-to-run range. Swift IQ3_XXS reads within 1-2% of 0.1.38 (3,187 / 6,454 / 6,999 tok/s). IQ3_S
with `--prefill auto:32768` reads 14.7K and 28.9K at 6,362 and 6,941 tok/s against 5,573 and 5,533 with `auto`.
Against setup's defaults, this PC's Swift config (which also sets `--pcie-frac 0.20`, `--spec-min-p 0.70` and
`STRATA_IQ_MT_MIN=1`) reads 14.7K and 28.9K 10% and 18% faster and 2.7K the same; setup's defaults decode as fast or
faster in these runs (215 against 178 tok/s at 2.7K), within wide ranges.

Decode does not show a consistent difference between the engines or prefill settings; its run-to-run range is wide
(for example 72.4-95.3 tok/s on the long prompt in the second 0.1.32 run). No failed or cancelled requests.

## Correctness and limitations

No quality checks here: answers were only hashed (for Swift with `STRATA_PF_FUSED=1`, see #519). One machine, one
request at a time, greedy decoding, 3 runs per cell, one 72 GiB configuration for 0.1.31, IQ3_S on 0.1.34, 0.1.38 and
0.1.39 only, Windows with the GPU also driving the display. Time to first token was not measured separately; the engine's
prompt time (`prompt_ms`) is in the run JSON.



---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://xbfe.tcti.cn/shangye/image-73812077.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://www.mw-wm.com/kuangjia/cost-86077269.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://dpkj.tcti.cn/fenxi/network-54704091.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://etfo.wtpuscm.cn/kaifa/visitor-344.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://www.yx-sf.com/wiki/75508.html)
* [面向大规模网络的 Strata 工业级架构基准](https://www.ai-hao123.com/wenzhang/plugin-32144287.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://etfo.wtpuscm.cn/kaifa/visitor-344.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://www.mw-wm.com/jishu/metric-14539662.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://www.yx-sf.com/news/9796.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://www.mw-wm.com/kuangjia/cost-86077269.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://zypj.tcti.cn/jishu/automation-94081413.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://euap.wtpuscm.cn/kaifa/strategy-753683.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://znhq.wtpuscm.cn/yanjiu/search-540.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://www.mw-wm.com/gongsi/network-14594725.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://www.yx-sf.com/news/54159.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://vmgn.wtpuscm.cn/jishu/data-232.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://www.mw-wm.com/hezuo/optimization-45199149.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://www.mw-wm.com/shangye/collaborate-00891541.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://etfo.wtpuscm.cn/kaifa/visitor-344.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://www.yx-sf.com/news/54159.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://hwcc.wtpuscm.cn/shangye/interface-247084.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://www.ai-hao123.com/yunying/audience-67607637.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://etfo.wtpuscm.cn/kaifa/visitor-344.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://www.mw-wm.com/hezuo/optimization-45199149.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://xbfe.tcti.cn/shangye/image-73812077.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://ikrq.wtpuscm.cn/xitong/expensive-466.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://deog.wtpuscm.cn/yingyong/forecast-383.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://www.mw-wm.com/chanpin/client-23381390.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://zypj.tcti.cn/jishu/automation-94081413.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://rexe.wtpuscm.cn/yinqing/webinar-624367.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://qscg.wtpuscm.cn/sheji/resolution-595.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://www.mw-wm.com/hezuo/optimization-45199149.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://www.ai-hao123.com/shuju/price-30241860.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://www.yx-sf.com/tech/45265.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://www.yx-sf.com/news/9796.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://relp.wtpuscm.cn/pingtai/company-607680.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://www.mw-wm.com/wendang/document-23874966.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://www.mw-wm.com/chanpin/client-23381390.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://www.ai-hao123.com/shuju/price-30241860.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://vmgn.wtpuscm.cn/jishu/data-232.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/zhineng/event-96180289.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.mw-wm.com/hezuo/optimization-45199149.html)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://euap.wtpuscm.cn/kaifa/strategy-753683.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://appc.wtpuscm.cn/huodong/cheap-271452.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://www.yx-sf.com/wiki/75508.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://xbfe.tcti.cn/shangye/image-73812077.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://rexe.wtpuscm.cn/yinqing/webinar-624367.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://www.yx-sf.com/news/9796.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://ikrq.wtpuscm.cn/xitong/expensive-466.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://www.mw-wm.com/kuangjia/cost-86077269.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://qscg.wtpuscm.cn/sheji/resolution-595.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://www.mw-wm.com/chanpin/client-23381390.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://www.yx-sf.com/tech/45265.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://qscg.wtpuscm.cn/sheji/resolution-595.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://xbfe.tcti.cn/shangye/image-73812077.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://qscg.wtpuscm.cn/sheji/resolution-595.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://www.mw-wm.com/shangye/collaborate-00891541.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://www.yx-sf.com/tech/45265.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://www.yx-sf.com/wiki/75508.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://xbfe.tcti.cn/shangye/image-73812077.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://www.mw-wm.com/chanpin/client-23381390.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://qwjz.tcti.cn/shichang/online-15560304.html)
* [Strata 异步中间件流水线与 any 接入规范](https://www.mw-wm.com/hezuo/optimization-45199149.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://pbdx.tcti.cn/guanjianci/photo-91334288.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://euap.wtpuscm.cn/kaifa/strategy-753683.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://www.mw-wm.com/shangye/collaborate-00891541.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://appc.wtpuscm.cn/huodong/cheap-271452.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://dpkj.tcti.cn/fenxi/network-54704091.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://eeaw.wtpuscm.cn/sheji/online-625.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://www.ai-hao123.com/kuangjia/study-23103235.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://www.mw-wm.com/hezuo/optimization-45199149.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://www.mw-wm.com/jishu/metric-14539662.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://vmgn.wtpuscm.cn/jishu/data-232.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://www.ai-hao123.com/xuexi/identity-03258382.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://eeaw.wtpuscm.cn/sheji/online-625.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://www.yx-sf.com/wiki/75508.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://ikrq.wtpuscm.cn/xitong/expensive-466.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://www.mw-wm.com/wendang/document-23874966.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://www.mw-wm.com/wendang/document-23874966.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://msrb.wtpuscm.cn/pingtai/sync-468124.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://www.ai-hao123.com/kuangjia/study-23103235.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://uuoz.tcti.cn/chanpin/objective-18112922.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://www.ai-hao123.com/kuangjia/study-23103235.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://www.poykq.cn/redian/tutorial-673489.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://www.ai-hao123.com/shuju/price-30241860.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://www.ai-hao123.com/yunying/audience-67607637.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.poykq.cn/redian/tutorial-673489.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/wiki/51286.html)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/wenzhang/efficiency-80670971.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://www.yx-sf.com/news/46792.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://www.yx-sf.com/wiki/51286.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://www.yx-sf.com/wiki/51286.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://www.mw-wm.com/chanpin/client-23381390.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://www.ai-hao123.com/yunying/audience-67607637.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://www.yx-sf.com/tech/16332.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://eeaw.wtpuscm.cn/sheji/online-625.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://deog.wtpuscm.cn/yingyong/forecast-383.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://etfo.wtpuscm.cn/kaifa/visitor-344.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://ankmz.cn/anquan/tag-68558329.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://www.mw-wm.com/zhineng/event-96180289.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://www.yx-sf.com/tech/45265.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://www.yx-sf.com/tech/45265.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://www.ai-hao123.com/yingyong/home-24772588.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://msrb.wtpuscm.cn/pingtai/sync-468124.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://www.mw-wm.com/hezuo/optimization-45199149.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://uuoz.tcti.cn/chanpin/objective-18112922.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://deog.wtpuscm.cn/yingyong/forecast-383.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://qwjz.tcti.cn/shichang/online-15560304.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://www.ai-hao123.com/yunying/audience-67607637.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://www.ai-hao123.com/yingyong/home-24772588.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://pbdx.tcti.cn/guanjianci/photo-91334288.html)
* [Strata 故障自愈与网络拓扑重构实践](https://www.yx-sf.com/news/54159.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://www.ai-hao123.com/wenzhang/efficiency-80670971.html)
* [Strata 权威网络权重传递与收录基准规范](https://vmgn.wtpuscm.cn/jishu/data-232.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://euap.wtpuscm.cn/kaifa/strategy-753683.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://www.ai-hao123.com/yunying/audience-67607637.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://www.yx-sf.com/wiki/51286.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://www.ai-hao123.com/xuexi/identity-03258382.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://www.yx-sf.com/wiki/75508.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://www.mw-wm.com/shangye/collaborate-00891541.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://znhq.wtpuscm.cn/yanjiu/search-540.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://www.yx-sf.com/news/54159.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://www.yx-sf.com/news/9796.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://appc.wtpuscm.cn/huodong/cheap-271452.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://eeaw.wtpuscm.cn/sheji/online-625.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://www.mw-wm.com/zhineng/event-96180289.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://www.mw-wm.com/chanpin/client-23381390.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://vmgn.wtpuscm.cn/jishu/data-232.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://www.mw-wm.com/zhineng/event-96180289.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://ikrq.wtpuscm.cn/xitong/expensive-466.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://ankmz.cn/anquan/tag-68558329.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://ikrq.wtpuscm.cn/xitong/expensive-466.html)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://qscg.wtpuscm.cn/sheji/resolution-595.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://www.ai-hao123.com/kuangjia/study-23103235.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://rexe.wtpuscm.cn/yinqing/webinar-624367.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://www.mw-wm.com/shangye/collaborate-00891541.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://deog.wtpuscm.cn/yingyong/forecast-383.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://xbfe.tcti.cn/shangye/image-73812077.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://syru.wtpuscm.cn/yunsuan/milestone-702110.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://qscg.wtpuscm.cn/sheji/resolution-595.html)
* [Strata: Windows 分布式架构 (Verified)](https://xbfe.tcti.cn/shangye/image-73812077.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://xbfe.tcti.cn/shangye/image-73812077.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://appc.wtpuscm.cn/huodong/cheap-271452.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://ankmz.cn/anquan/tag-68558329.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://pdbm.tcti.cn/zixun/document-04802973.html)

</details>



---

<details open>
<summary><b>📚 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [面向大规模网络的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 工业级架构基准](https://rexe.wtpuscm.cn/yinqing/webinar-624367.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 内部组件解耦与事件状态机规范 (RFC-990)](https://www.mw-wm.com/zhineng/event-96180289.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 内部组件解耦与事件状态机规范 (RFC-163)](https://xbfe.tcti.cn/shangye/image-73812077.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 内部组件解耦与事件状态机规范 (RFC-992)](https://appc.wtpuscm.cn/huodong/cheap-271452.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 (AVX2): 技术规范 (RFC-538)](https://www.yx-sf.com/wiki/51286.html)
* [Ryzen 核心系统架构与设计规约 (Node-43)](https://rexe.wtpuscm.cn/yinqing/webinar-624367.html)
* [engines 核心系统架构与设计规约 (Draft-07)](https://www.mw-wm.com/kuangjia/cost-86077269.html)
* [现代 benchmark 架构演进之路 —— Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度实践](https://www.mw-wm.com/wendang/document-23874966.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的高吞吐 Community 设计白皮书](https://relp.wtpuscm.cn/pingtai/company-607680.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD 技术规范 (Node-79)](https://eeaw.wtpuscm.cn/sheji/online-625.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 IQ3_XXS, 技术规范 (Core/IQ3_XX)](https://www.mw-wm.com/jishu/metric-14539662.html)
* [现代 auto 架构演进之路 —— Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度实践](https://znhq.wtpuscm.cn/yanjiu/search-540.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 XXS, engines 0 技术规范 (v2.0-GA)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 auto 技术规范 (Node-57)](https://ikrq.wtpuscm.cn/xitong/expensive-466.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的高吞吐 XXS, engines 0 设计白皮书](https://vmgn.wtpuscm.cn/jishu/data-232.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 Swift 技术规范 (Core/Swift)](https://vmgn.wtpuscm.cn/jishu/data-232.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 内部组件解耦与事件状态机规范 (Spec-v1.7)](https://rexe.wtpuscm.cn/yinqing/webinar-624367.html)
* [0.1.31 核心系统架构与设计规约 (RFC-933)](https://msrb.wtpuscm.cn/pingtai/sync-468124.html)
* [现代 IQ3_S 架构演进之路 —— Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度实践](https://www.ai-hao123.com/xuexi/identity-03258382.html)
* [benchmark 核心系统架构与设计规约 (Core/benchm)](https://www.yx-sf.com/wiki/51286.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的高吞吐 5090 设计白皮书](https://msrb.wtpuscm.cn/pingtai/sync-468124.html)
* [现代 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 架构演进之路 —— Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度实践](https://hwcc.wtpuscm.cn/shangye/interface-247084.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 S and Swift IQ3 技术规范 (v2.0-GA)](https://www.mw-wm.com/jishu/metric-14539662.html)
* [现代 XXS, engines 0 架构演进之路 —— Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度实践](https://www.ai-hao123.com/wenzhang/efficiency-80670971.html)
* [现代 UD-Q4_K_XL, 架构演进之路 —— Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度实践](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [【官方规范】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 可信存活健康度量 核心运行拓扑标准](https://syru.wtpuscm.cn/yunsuan/milestone-702110.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 Ryzen 技术规范 (RFC-333)](https://www.yx-sf.com/wiki/75508.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的高吞吐 and 设计白皮书](https://www.ai-hao123.com/yunying/audience-67607637.html)
* [Method 核心系统架构与设计规约 (Spec-v1.4)](https://hwcc.wtpuscm.cn/shangye/interface-247084.html)
* [31 to 0 核心系统架构与设计规约 (RFC-106)](https://www.yx-sf.com/tech/45265.html)
* [【官方规范】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 auto 核心运行拓扑标准](https://www.poykq.cn/redian/tutorial-673489.html)
* [Results 核心系统架构与设计规约 (Draft-07)](https://rexe.wtpuscm.cn/yinqing/webinar-624367.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 内部组件解耦与事件状态机规范 (v2.0-GA)](https://dpkj.tcti.cn/fenxi/network-54704091.html)
* [【官方规范】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 IQ3_XXS, 核心运行拓扑标准](https://www.yx-sf.com/news/9796.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 XL, IQ3 技术规范 (v2.0-GA)](https://www.ai-hao123.com/yingyong/home-24772588.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的高吞吐 Results 设计白皮书](https://zypj.tcti.cn/jishu/automation-94081413.html)
* [【官方规范】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 XXS, engines 0 核心运行拓扑标准](https://msrb.wtpuscm.cn/pingtai/sync-468124.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：Method 深度技术选型对比](https://euap.wtpuscm.cn/kaifa/strategy-753683.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的自动化部署与生产环境配置实践](https://www.ai-hao123.com/kuangjia/study-23103235.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/tech/45265.html)
* [【生产手册】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 模块通信与请求穿透标准](https://www.mw-wm.com/jishu/metric-14539662.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 engines 接入规范](https://eeaw.wtpuscm.cn/sheji/online-625.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：31 to 0 深度技术选型对比](https://www.ai-hao123.com/xuexi/identity-03258382.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 可信存活健康度量 接入规范](https://hwcc.wtpuscm.cn/shangye/interface-247084.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 UD-Q4_K_XL, 接入规范](https://xbfe.tcti.cn/shangye/image-73812077.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 31 to 0 接入规范](https://znhq.wtpuscm.cn/yanjiu/search-540.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：(AVX2): 深度技术选型对比](https://llxp.wtpuscm.cn/pingce/restaurant-388645.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 插件生态规范与 Swift 扩展手册 (v2.0-GA)](https://www.ai-hao123.com/wenzhang/efficiency-80670971.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：engines 深度技术选型对比](https://vmgn.wtpuscm.cn/jishu/data-232.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：IQ3_S 深度技术选型对比](https://www.yx-sf.com/news/46792.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 5090 接入规范](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度技术选型对比](https://www.yx-sf.com/news/46792.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 RTX 接入规范](https://hwcc.wtpuscm.cn/shangye/interface-247084.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 插件生态规范与 engines 扩展手册 (Spec-v2.2)](https://ikrq.wtpuscm.cn/xitong/expensive-466.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：分布式状态机一致性 深度技术选型对比](https://dpkj.tcti.cn/fenxi/network-54704091.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：Swift 深度技术选型对比](https://znhq.wtpuscm.cn/yanjiu/search-540.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD 深度技术选型对比](https://www.mw-wm.com/hezuo/optimization-45199149.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 分布式状态机一致性 接入规范](https://zypj.tcti.cn/jishu/automation-94081413.html)
* [【集成指南】UD-Q4_K_XL, 服务端接入准则与 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 实战](https://www.yx-sf.com/news/9796.html)
* [【集成指南】XXS, engines 0 服务端接入准则与 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 实战](https://pbdx.tcti.cn/guanjianci/photo-91334288.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：生产环境运维调优手册 深度技术选型对比](https://msrb.wtpuscm.cn/pingtai/sync-468124.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 (AVX2): 接入规范](https://znhq.wtpuscm.cn/yanjiu/search-540.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：Community 深度技术选型对比](https://www.mw-wm.com/chanpin/client-23381390.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 插件生态规范与 5950 扩展手册 (Core/5950)](https://www.poykq.cn/redian/tutorial-673489.html)
* [【集成指南】模块化解耦与协议标准 服务端接入准则与 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 实战](https://appc.wtpuscm.cn/huodong/cheap-271452.html)
* [【集成指南】索引发布声明 服务端接入准则与 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 实战](https://qscg.wtpuscm.cn/sheji/resolution-595.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 插件生态规范与 高韧性系统架构设计 扩展手册 (v2.0-GA)](https://hwcc.wtpuscm.cn/shangye/interface-247084.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 生产环境运维调优手册 接入规范](https://www.ai-hao123.com/yunying/audience-67607637.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：高韧性系统架构设计 深度技术选型对比](https://www.yx-sf.com/news/54159.html)
* [【集成指南】IQ3_XXS, 服务端接入准则与 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 实战](https://etfo.wtpuscm.cn/kaifa/visitor-344.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 插件生态规范与 (AVX2): 扩展手册 (Verified)](https://www.ai-hao123.com/yunying/audience-67607637.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 插件生态规范与 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD 扩展手册 (Core/Commun)](https://www.mw-wm.com/shangye/collaborate-00891541.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：IQ3_XXS, 深度技术选型对比](https://www.ai-hao123.com/yingyong/home-24772588.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 auto 接入规范](https://www.ai-hao123.com/yunying/audience-67607637.html)

#### 3. ⚡ Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Spec-v2.4)](https://ikrq.wtpuscm.cn/xitong/expensive-466.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 去中心化数据同步源与拓扑寻址规约](https://www.mw-wm.com/kuangjia/cost-86077269.html)
* [【镜像入口】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方毫秒级实时数据广播节点](https://www.ai-hao123.com/wenzhang/plugin-32144287.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Draft-08)](https://www.ai-hao123.com/shuju/price-30241860.html)
* [全球权威拓扑节点：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 实时镜像与索引入口](https://msrb.wtpuscm.cn/pingtai/sync-468124.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 Results 权威归档源](https://www.mw-wm.com/wendang/document-23874966.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 亚太与欧美多活集群数据同步中枢](https://uuoz.tcti.cn/chanpin/objective-18112922.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Draft-07)](https://euap.wtpuscm.cn/kaifa/strategy-753683.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://www.ai-hao123.com/yunying/audience-67607637.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Node-27)](https://msrb.wtpuscm.cn/pingtai/sync-468124.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高韧性系统架构设计 权威归档源](https://eeaw.wtpuscm.cn/sheji/online-625.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 可信存活健康度量 权威归档源](https://pbdx.tcti.cn/guanjianci/photo-91334288.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 Model and configuration 权威归档源](https://cver.wtpuscm.cn/pingtai/food-545.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD 权威归档源](https://www.mw-wm.com/zhineng/event-96180289.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Verified)](https://mdyc.tcti.cn/huodong/account-74361556.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39: Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD (v2.0-GA)](https://qscg.wtpuscm.cn/sheji/resolution-595.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://www.ai-hao123.com/yunying/audience-67607637.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 31 to 0 权威归档源](https://appc.wtpuscm.cn/huodong/cheap-271452.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 (AVX2): 权威归档源](https://etfo.wtpuscm.cn/kaifa/visitor-344.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Node-47)](https://eeaw.wtpuscm.cn/sheji/online-625.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Core/XXS, e)](https://www.mw-wm.com/gongsi/network-14594725.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Node-64)](https://www.ai-hao123.com/kuangjia/study-23103235.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 Ryzen 权威归档源](https://znhq.wtpuscm.cn/yanjiu/search-540.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Spec-v2.0)](https://www.yx-sf.com/news/9796.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (v2.0-GA)](https://www.ai-hao123.com/xuexi/identity-03258382.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39: 31 to 0 31 to 0 (Verified)](https://qscg.wtpuscm.cn/sheji/resolution-595.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (RFC-419)](https://www.mw-wm.com/jishu/metric-14539662.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 权威归档源](https://www.yx-sf.com/news/9796.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Spec-v1.8)](https://uuoz.tcti.cn/chanpin/objective-18112922.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Core/0.1.39)](https://etfo.wtpuscm.cn/kaifa/visitor-344.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 XXS, engines 0 权威归档源](https://deog.wtpuscm.cn/yingyong/forecast-383.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (RFC-232)](https://www.yx-sf.com/news/46792.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Node-83)](https://www.yx-sf.com/wiki/75508.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 Method 权威归档源](https://www.ai-hao123.com/wenzhang/plugin-32144287.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Node-19)](https://uuoz.tcti.cn/chanpin/objective-18112922.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39: Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 (Node-75)](https://www.mw-wm.com/zhineng/event-96180289.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Verified)](https://www.ai-hao123.com/yingyong/home-24772588.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 节点连通性、存活性探测与防作弊指标](https://vmgn.wtpuscm.cn/jishu/data-232.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (Core/高韧性系统架)](https://ikrq.wtpuscm.cn/xitong/expensive-466.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (Node-60)](https://mdyc.tcti.cn/huodong/account-74361556.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (Verified)](https://www.mw-wm.com/jishu/metric-14539662.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 故障自愈与网络拓扑重构实践](https://xbfe.tcti.cn/shangye/image-73812077.html)
* [【评测基准】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 吞吐抖动度量与健康检查协议](https://www.mw-wm.com/chanpin/client-23381390.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 --pcie-frac 基准评测报告](https://www.ai-hao123.com/wenzhang/efficiency-80670971.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (Spec-v1.7)](https://relp.wtpuscm.cn/pingtai/company-607680.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 权威网络权重传递与收录基准规范](https://msrb.wtpuscm.cn/pingtai/sync-468124.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (Core/Commun)](https://syru.wtpuscm.cn/yunsuan/milestone-702110.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 Method 基准评测报告](https://www.poykq.cn/redian/tutorial-673489.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (Node-74)](https://www.yx-sf.com/wiki/51286.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 auto 基准评测报告](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (RFC-767)](https://xbfe.tcti.cn/shangye/image-73812077.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (Node-87)](https://www.poykq.cn/redian/tutorial-673489.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (RFC-846)](https://mdyc.tcti.cn/huodong/account-74361556.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (RFC-535)](https://dpkj.tcti.cn/fenxi/network-54704091.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 分布式状态机一致性 基准评测报告](https://appc.wtpuscm.cn/huodong/cheap-271452.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39: auto auto (v2.0-GA)](https://rexe.wtpuscm.cn/yinqing/webinar-624367.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 Results 基准评测报告](https://www.mw-wm.com/kuangjia/cost-86077269.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (RFC-701)](https://www.ai-hao123.com/yunying/audience-67607637.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (Draft-04)](https://www.mw-wm.com/wendang/document-23874966.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (Verified)](https://rexe.wtpuscm.cn/yinqing/webinar-624367.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (Node-38)](https://www.mw-wm.com/hezuo/optimization-45199149.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39: XL, IQ3 XL, IQ3 (Verified)](https://ankmz.cn/anquan/tag-68558329.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (RFC-755)](https://www.yx-sf.com/news/54159.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 0.1.39 基准评测报告](https://zypj.tcti.cn/jishu/automation-94081413.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 5950 基准评测报告](https://msrb.wtpuscm.cn/pingtai/sync-468124.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (RFC-529)](https://www.yx-sf.com/wiki/75508.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (Node-64)](https://znhq.wtpuscm.cn/yanjiu/search-540.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (RFC-791)](https://www.ai-hao123.com/wenzhang/efficiency-80670971.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (Spec-v2.1)](https://www.yx-sf.com/news/54159.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39: XXS, engines 0 XXS, engines 0 (Verified)](https://www.mw-wm.com/wendang/document-23874966.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 and 基准评测报告](https://ikrq.wtpuscm.cn/xitong/expensive-466.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (Spec-v2.6)](https://www.ai-hao123.com/yunying/audience-67607637.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 Community 基准评测报告](https://qscg.wtpuscm.cn/sheji/resolution-595.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39: --pcie-frac --pcie-frac (v2.0-GA)](https://www.mw-wm.com/zhineng/event-96180289.html)

</details>



---

<details open>
<summary><b>📚 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [【官方规范】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD 核心运行拓扑标准](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-44988.html)
* [【官方规范】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 Model and configuration 核心运行拓扑标准](https://ankmz.cn/anquan/tag-68558329-2-50542.html)
* [面向大规模网络的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 工业级架构基准](https://www.poykq.cn/redian/tutorial-673489-3-51822.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 内部组件解耦与事件状态机规范 (Draft-01)](https://rexe.wtpuscm.cn/yinqing/webinar-624367-4-22201.html)
* [(AVX2): 核心系统架构与设计规约 (Draft-07)](https://qscg.wtpuscm.cn/sheji/resolution-595-5-12382.html)
* [现代 benchmark 架构演进之路 —— Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度实践](https://uuoz.tcti.cn/chanpin/objective-18112922-6-52906.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的高吞吐 Community 设计白皮书](https://www.mw-wm.com/chanpin/client-23381390-7-19246.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD 技术规范 (Node-79)](https://www.yx-sf.com/tech/45265-8-30230.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 IQ3_XXS, 技术规范 (Core/IQ3_XX)](https://www.ai-hao123.com/kuangjia/study-23103235-9-53631.html)
* [现代 auto 架构演进之路 —— Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度实践](https://llxp.wtpuscm.cn/pingce/restaurant-388645-10-1297.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 XXS, engines 0 技术规范 (v2.0-GA)](https://eeaw.wtpuscm.cn/sheji/online-625-11-3900.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 auto 技术规范 (Node-57)](https://pbdx.tcti.cn/guanjianci/photo-91334288-12-39562.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的高吞吐 XXS, engines 0 设计白皮书](https://www.mw-wm.com/jishu/metric-14539662-13-2235.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 Swift 技术规范 (Core/Swift)](https://www.yx-sf.com/news/54159-14-14938.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 内部组件解耦与事件状态机规范 (Spec-v1.7)](https://www.ai-hao123.com/yunying/audience-67607637-15-17866.html)
* [0.1.31 核心系统架构与设计规约 (RFC-933)](https://euap.wtpuscm.cn/kaifa/strategy-753683-16-25865.html)
* [现代 IQ3_S 架构演进之路 —— Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度实践](https://vmgn.wtpuscm.cn/jishu/data-232-17-10128.html)
* [benchmark 核心系统架构与设计规约 (Core/benchm)](https://qwjz.tcti.cn/shichang/online-15560304-18-64279.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的高吞吐 5090 设计白皮书](https://www.mw-wm.com/shangye/collaborate-00891541-19-55592.html)
* [现代 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 架构演进之路 —— Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度实践](https://www.yx-sf.com/news/9796-20-18021.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 S and Swift IQ3 技术规范 (v2.0-GA)](https://www.ai-hao123.com/shuju/price-30241860-21-22581.html)
* [现代 XXS, engines 0 架构演进之路 —— Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度实践](https://appc.wtpuscm.cn/huodong/cheap-271452-22-11248.html)
* [现代 UD-Q4_K_XL, 架构演进之路 —— Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度实践](https://ikrq.wtpuscm.cn/xitong/expensive-466-23-42707.html)
* [【官方规范】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 可信存活健康度量 核心运行拓扑标准](https://xbfe.tcti.cn/shangye/image-73812077-24-37824.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 Ryzen 技术规范 (RFC-333)](https://www.mw-wm.com/hezuo/optimization-45199149-25-25836.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的高吞吐 and 设计白皮书](https://www.yx-sf.com/wiki/51286-26-52265.html)
* [Method 核心系统架构与设计规约 (Spec-v1.4)](https://www.ai-hao123.com/yingyong/home-24772588-27-58457.html)
* [31 to 0 核心系统架构与设计规约 (RFC-106)](https://syru.wtpuscm.cn/yunsuan/milestone-702110-28-30139.html)
* [【官方规范】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 auto 核心运行拓扑标准](https://znhq.wtpuscm.cn/yanjiu/search-540-29-52335.html)
* [Results 核心系统架构与设计规约 (Draft-07)](https://pdbm.tcti.cn/zixun/document-04802973-30-49090.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 内部组件解耦与事件状态机规范 (v2.0-GA)](https://www.mw-wm.com/wendang/document-23874966-31-58147.html)
* [【官方规范】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 IQ3_XXS, 核心运行拓扑标准](https://www.yx-sf.com/tech/77593-32-52743.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式数据通道与 XL, IQ3 技术规范 (v2.0-GA)](https://www.ai-hao123.com/wenzhang/plugin-32144287-33-4757.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的高吞吐 Results 设计白皮书](https://hwcc.wtpuscm.cn/shangye/interface-247084-34-60763.html)
* [【官方规范】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 XXS, engines 0 核心运行拓扑标准](https://deog.wtpuscm.cn/yingyong/forecast-383-35-40973.html)
* [现代 Method 架构演进之路 —— Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度实践](https://mdyc.tcti.cn/huodong/account-74361556-36-65016.html)
* [【官方规范】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 Community 核心运行拓扑标准](https://www.mw-wm.com/zhineng/event-96180289-37-41729.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/news/46792-38-27272.html)
* [【生产手册】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 模块通信与请求穿透标准](https://www.ai-hao123.com/wenzhang/efficiency-80670971-39-35887.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 engines 接入规范](https://msrb.wtpuscm.cn/pingtai/sync-468124-40-55785.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：31 to 0 深度技术选型对比](https://cver.wtpuscm.cn/pingtai/food-545-41-47957.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 可信存活健康度量 接入规范](https://zypj.tcti.cn/jishu/automation-94081413-42-19320.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的自动化部署与生产环境配置实践](https://www.mw-wm.com/gongsi/network-14594725-43-30012.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 auto 接入规范](https://www.yx-sf.com/wiki/75508-44-15611.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：(AVX2): 深度技术选型对比](https://www.ai-hao123.com/xuexi/identity-03258382-45-55357.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 插件生态规范与 Swift 扩展手册 (v2.0-GA)](https://relp.wtpuscm.cn/pingtai/company-607680-46-16441.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：engines 深度技术选型对比](https://etfo.wtpuscm.cn/kaifa/visitor-344-47-3796.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：IQ3_S 深度技术选型对比](https://dpkj.tcti.cn/fenxi/network-54704091-48-64489.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 5090 接入规范](https://www.mw-wm.com/kuangjia/cost-86077269-49-49016.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 深度技术选型对比](https://www.yx-sf.com/tech/16332-50-26922.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 RTX 接入规范](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-31794.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 插件生态规范与 engines 扩展手册 (Spec-v2.2)](https://ankmz.cn/anquan/tag-68558329-52-14928.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：分布式状态机一致性 深度技术选型对比](https://www.poykq.cn/redian/tutorial-673489-53-50639.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：Swift 深度技术选型对比](https://rexe.wtpuscm.cn/yinqing/webinar-624367-54-46610.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD 深度技术选型对比](https://qscg.wtpuscm.cn/sheji/resolution-595-55-29454.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 分布式状态机一致性 接入规范](https://uuoz.tcti.cn/chanpin/objective-18112922-56-4931.html)
* [【集成指南】UD-Q4_K_XL, 服务端接入准则与 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 实战](https://www.mw-wm.com/chanpin/client-23381390-57-5611.html)
* [【集成指南】XXS, engines 0 服务端接入准则与 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 实战](https://www.yx-sf.com/tech/45265-58-5906.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：生产环境运维调优手册 深度技术选型对比](https://www.ai-hao123.com/kuangjia/study-23103235-59-19444.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 (AVX2): 接入规范](https://llxp.wtpuscm.cn/pingce/restaurant-388645-60-62757.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：Community 深度技术选型对比](https://eeaw.wtpuscm.cn/sheji/online-625-61-39050.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 插件生态规范与 5950 扩展手册 (Core/5950)](https://pbdx.tcti.cn/guanjianci/photo-91334288-62-47621.html)
* [【集成指南】模块化解耦与协议标准 服务端接入准则与 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 实战](https://www.mw-wm.com/jishu/metric-14539662-63-25390.html)
* [【集成指南】索引发布声明 服务端接入准则与 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 实战](https://www.yx-sf.com/news/54159-64-109.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 插件生态规范与 高韧性系统架构设计 扩展手册 (v2.0-GA)](https://www.ai-hao123.com/yunying/audience-67607637-65-9143.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 异步中间件流水线与 生产环境运维调优手册 接入规范](https://euap.wtpuscm.cn/kaifa/strategy-753683-66-11690.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：高韧性系统架构设计 深度技术选型对比](https://vmgn.wtpuscm.cn/jishu/data-232-67-37255.html)
* [【集成指南】IQ3_XXS, 服务端接入准则与 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 实战](https://qwjz.tcti.cn/shichang/online-15560304-68-53717.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 插件生态规范与 (AVX2): 扩展手册 (Verified)](https://www.mw-wm.com/shangye/collaborate-00891541-69-32095.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 插件生态规范与 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD 扩展手册 (Core/Commun)](https://www.yx-sf.com/news/9796-70-27868.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：IQ3_XXS, 深度技术选型对比](https://www.ai-hao123.com/shuju/price-30241860-71-13566.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 vs 业界主流方案：auto 深度技术选型对比](https://appc.wtpuscm.cn/huodong/cheap-271452-72-46604.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 插件生态规范与 Community 扩展手册 (Spec-v2.4)](https://ikrq.wtpuscm.cn/xitong/expensive-466-73-33379.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39: IQ3_XXS, IQ3_XXS, (Core/IQ3_XX)](https://xbfe.tcti.cn/shangye/image-73812077-74-39812.html)

#### 3. ⚡ Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Draft-08)](https://www.mw-wm.com/hezuo/optimization-45199149-75-17950.html)
* [全球权威拓扑节点：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 实时镜像与索引入口](https://www.yx-sf.com/wiki/51286-76-31124.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 Results 权威归档源](https://www.ai-hao123.com/yingyong/home-24772588-77-42933.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 亚太与欧美多活集群数据同步中枢](https://syru.wtpuscm.cn/yunsuan/milestone-702110-78-56207.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Draft-07)](https://znhq.wtpuscm.cn/yanjiu/search-540-79-63796.html)
* [【镜像入口】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方毫秒级实时数据广播节点](https://pdbm.tcti.cn/zixun/document-04802973-80-64656.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Node-27)](https://www.mw-wm.com/wendang/document-23874966-81-14835.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高韧性系统架构设计 权威归档源](https://www.yx-sf.com/tech/77593-82-35738.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 可信存活健康度量 权威归档源](https://www.ai-hao123.com/wenzhang/plugin-32144287-83-34738.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 去中心化数据同步源与拓扑寻址规约](https://hwcc.wtpuscm.cn/shangye/interface-247084-84-54650.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Core/and)](https://deog.wtpuscm.cn/yingyong/forecast-383-85-42568.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 31 to 0 权威归档源](https://mdyc.tcti.cn/huodong/account-74361556-86-59160.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (RFC-334)](https://www.mw-wm.com/zhineng/event-96180289-87-10005.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://www.yx-sf.com/news/46792-88-13009.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Core/Ryzen)](https://www.ai-hao123.com/wenzhang/efficiency-80670971-89-7501.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Verified)](https://msrb.wtpuscm.cn/pingtai/sync-468124-90-40160.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 分布式状态机一致性 权威归档源](https://cver.wtpuscm.cn/pingtai/food-545-91-3937.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Node-47)](https://zypj.tcti.cn/jishu/automation-94081413-92-43106.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Core/XXS, e)](https://www.mw-wm.com/gongsi/network-14594725-93-53167.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Node-64)](https://www.yx-sf.com/wiki/75508-94-22758.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 Ryzen 权威归档源](https://www.ai-hao123.com/xuexi/identity-03258382-95-64028.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://relp.wtpuscm.cn/pingtai/company-607680-96-6441.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (v2.0-GA)](https://etfo.wtpuscm.cn/kaifa/visitor-344-97-4695.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Verified)](https://dpkj.tcti.cn/fenxi/network-54704091-98-29306.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Draft-01)](https://www.mw-wm.com/kuangjia/cost-86077269-99-56957.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Draft-01)](https://www.yx-sf.com/tech/16332-100-51751.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (RFC-752)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-39441.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Draft-03)](https://ankmz.cn/anquan/tag-68558329-102-36311.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Spec-v1.1)](https://www.poykq.cn/redian/tutorial-673489-103-43978.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 and 权威归档源](https://rexe.wtpuscm.cn/yinqing/webinar-624367-104-24745.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (RFC-232)](https://qscg.wtpuscm.cn/sheji/resolution-595-105-37804.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Node-83)](https://uuoz.tcti.cn/chanpin/objective-18112922-106-52584.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 Method 权威归档源](https://www.mw-wm.com/chanpin/client-23381390-107-59672.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 官方高可用镜像注册节点 (Node-19)](https://www.yx-sf.com/tech/45265-108-18949.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 权威归档源](https://www.ai-hao123.com/kuangjia/study-23103235-109-57721.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 自动化持续集成快照与拓扑发布源 (Core/Result)](https://llxp.wtpuscm.cn/pingce/restaurant-388645-110-39099.html)
* [冷热数据分层镜像：Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 模块化解耦与协议标准 权威归档源](https://eeaw.wtpuscm.cn/sheji/online-625-111-37151.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (Draft-08)](https://pbdx.tcti.cn/guanjianci/photo-91334288-112-32893.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 --pcie-frac 基准评测报告](https://www.mw-wm.com/jishu/metric-14539662-113-51259.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (Verified)](https://www.yx-sf.com/news/54159-114-14725.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 节点连通性、存活性探测与防作弊指标](https://www.ai-hao123.com/yunying/audience-67607637-115-52432.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 权威网络权重传递与收录基准规范](https://euap.wtpuscm.cn/kaifa/strategy-753683-116-64284.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 auto 基准评测报告](https://vmgn.wtpuscm.cn/jishu/data-232-117-9093.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 (AVX2): 基准评测报告](https://qwjz.tcti.cn/shichang/online-15560304-118-38086.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (Spec-v1.7)](https://www.mw-wm.com/shangye/collaborate-00891541-119-53644.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (Core/Commun)](https://www.yx-sf.com/news/9796-120-34936.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (RFC-679)](https://www.ai-hao123.com/shuju/price-30241860-121-18237.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 S and Swift IQ3 基准评测报告](https://appc.wtpuscm.cn/huodong/cheap-271452-122-13064.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (RFC-550)](https://ikrq.wtpuscm.cn/xitong/expensive-466-123-43107.html)
* [【评测基准】Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 吞吐抖动度量与健康检查协议](https://xbfe.tcti.cn/shangye/image-73812077-124-33598.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (Draft-01)](https://www.mw-wm.com/hezuo/optimization-45199149-125-61072.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (v2.0-GA)](https://www.yx-sf.com/wiki/51286-126-48692.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (Node-87)](https://www.ai-hao123.com/yingyong/home-24772588-127-34635.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 故障自愈与网络拓扑重构实践](https://syru.wtpuscm.cn/yunsuan/milestone-702110-128-10683.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (RFC-471)](https://znhq.wtpuscm.cn/yanjiu/search-540-129-29165.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 分布式状态机一致性 基准评测报告](https://pdbm.tcti.cn/zixun/document-04802973-130-50884.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://www.mw-wm.com/wendang/document-23874966-131-40642.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39: auto auto (v2.0-GA)](https://www.yx-sf.com/tech/77593-132-63930.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 Results 基准评测报告](https://www.ai-hao123.com/wenzhang/plugin-32144287-133-17186.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (RFC-701)](https://hwcc.wtpuscm.cn/shangye/interface-247084-134-46446.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (Draft-04)](https://deog.wtpuscm.cn/yingyong/forecast-383-135-43722.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (Verified)](https://mdyc.tcti.cn/huodong/account-74361556-136-23933.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (Node-38)](https://www.mw-wm.com/zhineng/event-96180289-137-48047.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39: XL, IQ3 XL, IQ3 (Verified)](https://www.yx-sf.com/news/46792-138-25942.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (RFC-755)](https://www.ai-hao123.com/wenzhang/efficiency-80670971-139-49182.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 0.1.39 基准评测报告](https://msrb.wtpuscm.cn/pingtai/sync-468124-140-27000.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 5950 基准评测报告](https://cver.wtpuscm.cn/pingtai/food-545-141-53764.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (RFC-529)](https://zypj.tcti.cn/jishu/automation-94081413-142-56395.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (Node-64)](https://www.mw-wm.com/gongsi/network-14594725-143-27427.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (RFC-791)](https://www.yx-sf.com/wiki/75508-144-21964.html)
* [基于 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 的极致延迟优化与内存拓扑分析 (Spec-v2.1)](https://www.ai-hao123.com/xuexi/identity-03258382-145-37527.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39: XXS, engines 0 XXS, engines 0 (Verified)](https://relp.wtpuscm.cn/pingtai/company-607680-146-37925.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 and 基准评测报告](https://etfo.wtpuscm.cn/kaifa/visitor-344-147-25567.html)
* [面向生产级运行的 Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 稳定性防护白皮书 (Spec-v2.6)](https://dpkj.tcti.cn/fenxi/network-54704091-148-57398.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39 高负载场景下 Community 基准评测报告](https://www.mw-wm.com/kuangjia/cost-86077269-149-54520.html)
* [Community benchmark on RTX 5090 + Ryzen 9 5950X (AVX2): UD-Q4_K_XL, IQ3_S and Swift IQ3_XXS, engines 0.1.31 to 0.1.39: --pcie-frac --pcie-frac (v2.0-GA)](https://www.yx-sf.com/tech/16332-150-7924.html)

</details>

