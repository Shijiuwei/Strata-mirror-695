# Community benchmark: Radeon AI PRO R9700 (gfx1201), Windows 11, Ryzen 9 9950X

Measured on 2026-10-03 by [HarukiOtaku](https://oobz.tcti.cn/yunsuan/conversion-15862380.html). Strata **0.1.38**
(prebuilt Windows HIP engine), original Flash-Next **IQ2_XS**, one GPU, context limit
262,144 tokens. Greedy, 256-token cap, reasoning off, three runs per length; prompt and
decode figures are the engine's own log values.

| Prompt tokens | Reused | Generated | Runs | Prompt tok/s (median, range) | Decode tok/s (median, range) | TTFT s (median, range) | Total s |
| ---: | ---: | --- | ---: | --- | --- | --- | ---: |
| 4,198 (4,195–4,199) | 0 | 256 (236–256) | 3 | **855.8** (820.7 – 863.5) | **99.8** (95.3 – 101.3) | 4.95 (4.89 – 5.16) | 7.4 |
| 33,467 | 0 | 256 | 3 | **1,182.1** (1,181.1 – 1,191.2) | **97.8** (95.3 – 101.2) | 28.39 (28.18 – 28.43) | 30.9 |
| 131,388 | 0 | 256 (249–256) | 3 | **1,231.7** (1,225.0 – 1,233.3) | **89.9** (86.2 – 91.0) | 106.90 (106.77 – 107.49) | 109.7 |
| 247,990 | 0 | 256 (250–256) | 3 | **1,130.1** (1,127.2 – 1,132.0) | **87.5** (75.1 – 87.5) | 219.84 (219.49 – 220.42) | 222.7 |

**Read the medians with a band.** The three reps inside one engine session agree to <1 % at
32K/128K/248K and ~5 % at 4K, but the same configuration measured again in a fresh session
moved by up to **11.5 % at 32K** (medians 1,182.1 vs 1,060.6) and **14.2 % at 4K** (855.8 vs
749.3) — 13.4 % and 18.4 % if individual reps are compared — so treat a median as ±10–15 %,
not a fixed figure. Only 4K and 32K have replicate sessions: **131,388 and 247,990 each ran in
one session** (the only other 128K requests are in the excluded void attempt).

Every run in the tables read its whole prompt (`reused = 0`) and no stall line appears in the
log. Six needle checks were found (**6/6**) at ~33.5K and ~131.4K prompt tokens, depths
10/50/90 % — one run each, so an outcome rather than a success rate. One earlier attempt
produced almost no usable answers (13 of its 15 rows empty) and is reported below as a
failure, not as data.

`docs/AMD_HIP.md` marks long contexts beyond 16K as **not validated** on RDNA4 and the
maintainer's R9700 validation is Linux at 4K/16K; the Windows AMD report I could find
([#499](https://vrir.tcti.cn/shangye/metric-21409942.html), RX 9070 XT — still an **open,
unmerged** PR) measures prompts to 31,059 tokens. The novelty here is **Windows + gfx1201 out
to 248K**, not the hardware.

Worth stating plainly what upstream does and does not claim. `docs/AMD_HIP.md` documents one
validated Windows AMD case — an RX 9070 XT on Windows 11 with an engine **compiled on that
PC**, 32K, 29.2 tok/s decode / ~181 tok/s prefill (0.1.34) — and then says: "The ready-made
zip itself has not run a model on a discrete card yet - please report." This run is that
report, on a different card (R9700, gfx1201) and through the ready-made zip. It is **not**
project-validated support: `docs/AMD_HIP_PERFORMANCE.md` excludes "Windows HIP, other AMD
architectures" from its claims, and `docs/AMD_HIP.md` says its RDNA4 table "must not be read
as a benchmark of every subsequent rebase" and that "long-context stress, broad answer-quality
equivalence, other AMD cards, and mixed-vendor inference are not validated here".

**Not a controlled study:** one OS session, one machine, one quant; prompts are synthetic
code; GPU clocks and power limit are not recorded; the desktop was not isolated.

## Hardware and software

- **GPU:** Radeon AI PRO R9700 32 GiB (gfx1201; Windows reports 32 GB, engine sees 31.9 GiB),
  stock clocks, **power limit not recorded**. The display runs on the integrated GPU, so the
  card is compute-only.
- **PCIe:** engine probe `54.7 GB/s host->device (best of 45.5 54.4 54.7 54.6)`;
  **width/generation not measured**.
- **CPU / RAM / storage:** Ryzen 9 9950X 16-Core (AVX-512; the engine ran its default 15
  expert-pool workers + host thread); Windows reports **61.5 GiB total physical memory**
  (the DIMM kit was not recorded, so no kit size is claimed); 2 TB-class NVMe on `D:`
  (**device model not recorded**).
- **OS / driver:** Windows 11 Pro, version **10.0.26300**, AMD driver **32.0.31036.15**
  (iGPU 32.0.11024.2) — quoted from the machine in [PROVENANCE.txt](PROVENANCE.txt).
- **Engine:** repo `main` `99f3dbd0b21d1401b3769e0c0d963913607f380b`; prebuilt Windows HIP
  engine **0.1.38** (each session logs `session is up (engine 0.1.38)`; its `engine/BUILD.json`
  reads prebuilt, hip, 0.1.38, ROCm 10.2.0a20260930, hipBLASLt 100500, gfx1201 — **quoted
  verbatim in [PROVENANCE.txt](PROVENANCE.txt)**, no build hash). Ready-made
  `strata-windows-x64-hip.zip`; HIP runtime `engine/amdhip64_7.dll`.
- **Not captured:** the Windows-AMD bundle `docs/AMD_HIP.md` asks for
  (`strata-device --list-devices`, `--selftest`, engine-log tail), the hipBLASLt
  `STRATA_HIPBLASLT_VERBOSE=1`/`fallbacks=0` check, clocks/power/thermals, per-run peak
  memory, paging/OOM counters, the per-run answer text, and hashes for the IQ2_XS pack and the
  MTP tensors. The tuning table in use is
  upstream's own `tools/hip/gfx1201-hipblaslt-100500.txt` (32 rows, hipBLASLt 1.5.0 /
  ROCm 10.2.0a generation — the table is calibrated on a 2026-09-14 nightly while the engine
  reports a 2026-09-30 one, and upstream warns that a different 1.5.0 build may number its
  solutions differently) and the log shows it enabled, but it was
  not verified with `STRATA_HIPBLASLT_VERBOSE=1`/`fallbacks=0`, and upstream's measured gain
  for it is +3.9 % on 4,210-token prompts.

Engine state of the measured sessions, condensed from `engine-0.1.38.log` (verbatim
excerpts):

```text
expert cache auto: 25.78 GiB free, 700 MiB reserved (+143 MiB for the draft head) -> 17739 slots
expert cache 18575 slots, 24.95 GiB of VRAM
pre-filled 18575 of 18575 slots from the profile; slot 0 verified
FileExpertSource: RAM budget 16.00 GiB: 6001 of the 8.07 GiB of experts the GPU cache does not hold
the file tier reads unbuffered (5 of 16 probe reads from the file cache; 46.0 GiB available, 36.5 GiB of files)
Windows budgets 31704 of this card's 32624 MiB for this process; free VRAM is counted within that
KV streaming: 32768 of 262144 cells per QSA layer in VRAM, the K/V in 3.09 GiB of pinned RAM
the prompt path borrows 3203 CUDA0 cache slots (4.34 GiB)
PLE on, table 320001536 rows
15 expert-pool workers + the host thread
```

The previous engine (pilot, 131,072 context) logged `large pages refused for 35456548864 B
(GetLargePageMinimum=2097152, VirtualAlloc error 1314); using 4 KB pages`, kept in
[pilot-0.1.36-engine-startup.log](pilot-0.1.36-engine-startup.log); "Lock pages in memory" is
not enabled, which #42 documents as the expected desktop outcome for error 1314.

## Model and configuration

- `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF` @
  `ed59f92082b1e93c0e96d60a8b11aab089b52f09` (unchanged since 2026-09-29, before this
  machine's 2026-10-02 download). Shards
  `IQ2_XS/Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf` and `...-00002-of-00002.gguf`.
  **SHA-256 for both shards and for `data/expert-profile.bin` is in
  [PROVENANCE.txt](PROVENANCE.txt)** (the pack and the MTP draft tensors have no hash).
- MTP draft head from `Qwen/Qwen3.8-Flash-Next` (revision not recorded); the log calls it an
  `experimental native Q5_K head, 337715200 bytes` and loads `835 MiB` of draft layer.
  Speculative decoding is on (`--spec 4`, `--spec-min-p 0.5`); draft acceptance was
  62.1–83.8 % on the twelve headline runs, and 57–100 % across all 36 measured rows (the low
  end is a 13-token needle generation) — counts live in the log only.
- Vision off (no `--vision`, no mmproj; Windows AMD has no images), no `--low-ram`, no engine
  calibration, no custom control vectors, no API key. `--kv int8` and greedy 256-token
  sampling come from the launch configuration; the log shows the KV cell count but not the
  dtype.

Engine arguments as written in `strata-iq2_xs.json` (the launchers start `serve/server.py`,
which spawns the engine with these):

```text
--pack  D:\AI\Strata\Strata-main\Strata-data\packs\iq2_xs
--native  D:\AI\Strata\Strata-main\Strata-data\models\IQ2_XS\Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf
--ple-gguf D:\AI\Strata\Strata-main\Strata-data\models\IQ2_XS\Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf
--expert-profile D:\AI\Strata\Strata-main\Strata-main\data\expert-profile.bin
--expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5
--mtp D:\AI\Strata\Strata-main\Strata-data\mtp\rt
--max-context 262144 --kv int8 --kv-resident 32768 --mmap-experts --resident-budget-gib 16
env STRATA_HIPBLASLT_TUNING=...\tools\hip\gfx1201-hipblaslt-100500.txt
```

## Method

- [benchmark.py](benchmark.py) (standard library only) builds each request from synthetic
  Python-like filler, sizes it in **characters** (calibration measured 2.52 chars/token, so
  targets overshoot 2–3.5 %), and puts a **fresh nonce in every prompt** so no run can reuse a
  cached prefix — hence `reused = 0` by construction, not because caching is absent.
- Each series runs a calibration and a 2,082-token warm-up first; both are excluded from the
  tables, as is engine loading. Reasoning is off, so the first token is answer text.
- **Prompt/decode tok/s, token counts and cache states are the engine's log values; TTFT and
  total latency are client-measured** (HTTP + tokenisation + frontend included). No
  "generated tokens ÷ total time" figure is used. As a cross-check, the client timers bracket
  the engine's own clocks and **never come out faster**: TTFT ≥ `read_ms` by 32–414 ms and
  total ≥ `read_ms + gen_ms` by 17–395 ms on every tabled row, so the engine's timings are
  corroborated as lower bounds rather than independently verified (its token counts have no
  external check at all).
- The exact prompts cannot be reconstructed from this packet: the filler is deterministic but
  the per-request seed and nonce are not saved (only `chars` is), and the speed rows keep no
  answer text. [benchmark.py](benchmark.py) is the reproducible instruction the guide asks for.
- The three reps at a length are three **different** prompts of the same nominal size (the
  filler seed increments per rep), so each median mixes prompt-to-prompt variation with
  run-to-run noise. The headline runs also use the **default prompt-kernel path** — the fused
  kernel is only in the A/B leg.
- The switches `docs/DETAILS.md` recommends for byte-identical repeats and clean A/B arms
  (`--prompt-cache 0 --adapt-swaps 0 --pcie-frac 0`, plus `STRATA_IQ_MT_MIN=1`) were **not**
  used, so cache adaptation and PCIe-fraction rounding can move results between requests and
  between arms — one plausible contributor to the 10.8 % default-vs-default spread.
- The log carries **no timestamps or request ids**, so a row is bound to a log line by token
  count; the harness's `pairing_ok` check was added after these runs and is absent from the
  files. Re-run by hand: all 69 rows with a `usage` block (36 measured, 15 void, 18
  calibration/warm-up) match within ±1 and no `read_ms` repeats.
- The `model` string is `qwen3.8-flash-next-iq2_xs` in the standard/240K/void files and
  `strata` (the harness's fallback alias) in the six A/B files.
- Prompt targets are labels, not sizes; the measured `engine_prompt_tokens` are reported
  per row. The needle is placed at 10/50/90 % of the **character** position and the repo's
  `tools/needle_bench.py` was **not** used, so the recall check is this harness's own.

Full method detail, the label/session caveats and every disclosure: see
[METHOD-AND-DISCLOSURES.md](METHOD-AND-DISCLOSURES.md).

## Results

Per-run JSON: [bench-output-standard.json](bench-output-standard.json) (4K/32K/128K +
needles), [bench-output-240k.json](bench-output-240k.json) (248K). "Tabled" = the 36 rows in
a measured series across the nine JSON files (18 in the two headline files, 18 in the six A/B
legs); the tables use the first 18. The log
([engine-0.1.38.log](engine-0.1.38.log)) holds 71 request lines: 36 tabled, 15 rows of the
failed attempt below, 18 calibration/warm-up, and 2 others (a 246,933-token request the
harness did not produce, and a leftover 0.1.36 pilot line).

**The failure:** [bench-output-standard-first-attempt-void.json](bench-output-standard-first-attempt-void.json)
was made before the harness disabled reasoning — 13 of its 15 rows returned no answer text
(7 speed rows at 256 generated tokens and 6 needle rows at the 32-token cap), all six needles
missed, `ttft_s` is null on 7 of 9 speed rows. It is a failure outside the summary, not a
recall or throughput result.

Needle recall:

| Prompt tokens | Depth | Result | Prompt tok/s | Decode tok/s |
| ---: | ---: | --- | ---: | ---: |
| 33,512 | 10 / 50 / 90 % | **FOUND / FOUND / FOUND** | 1,178.6 / 1,193.8 / 1,186.8 | 118.1 / 86.1 / 118.9 |
| 131,433 | 10 / 50 / 90 % | **FOUND / FOUND / FOUND** | 1,230.7 / 1,233.8 / 1,229.7 | 73.7 / 83.5 / 106.2 |

- **Curve shape (one session).** All four lengths ran inside a single engine session; the
  4K→32K and 32K→128K segments each imply ≈1.5 s per-request overhead with ≈1,250 tok/s
  marginal, and 128K→248K is flatter (≈1,030 tok/s). The 32K→128K step is only +4.2 %,
  *smaller* than the 11.5 % session-to-session band, and 248K has no replicate session — so
  this is one session's shape, and a smooth quadratic fits the four medians about as well.
  The ≈1.5 s figure does not describe the calibration requests: the 346.4 tok/s calibration
  follows a 129,816-token needle request in the same session (the one following the
  246,933-token request ran at 308.7 tok/s), while the same prompt as a fresh session's first
  request takes 5.36 s; see [#519](https://uwfo.wtpuscm.cn/liuliang/progress-947513.html).
- **Every engine start performs a ≈26.7 GB one-off read** (26,741 MB and 26,779 MB by the two
  derivations below), which the log's cumulative counter folds into whichever request runs
  first: 31,214.5 MB on a 3,216-token calibration against 4,473 MB for the same prompt
  mid-session, and independently 175,996.2 − 149,217.3 MB in the other session. The 0.1.36
  pilot's note that this was "NOT a one-time cost per engine start" describes that older build;
  on 0.1.38 the excess reproduces in all six later starts. A session's first request line
  therefore overstates that request's own reads, and short-prompt figures from a first request
  are not comparable with warm ones.
- **SSD read volume** (delta of the log's cumulative counter, near-identical across
  same-length runs — within ±0.1 % at 32K and 128K): 1.31 MB per prompt token at 4K, 0.80 at
  32K, 0.63 at 128K, 0.60 at 248K (131,388 → 82,225 MB; 247,990 → 149,217 MB), i.e.
  ≈1.07–1.13 GB/s sustained at 4K falling to ≈0.68 GB/s at 248K, with a ≈3 GB
  per-request component that even the warm-up pays.
- **Definitions follow the guide** (prefill = freshly read tokens ÷ engine read time; decode =
  generated ÷ engine gen time; TTFT = client wall clock, first token is answer text). Two
  defensible alternatives would move the medians slightly: prefill taken from the client's
  TTFT instead of the engine's read time gives ≈1,179 tok/s at 32K (published range
  1,181.1–1,191.2), and decode over the client window (`total − TTFT`) gives ≈88.0 tok/s at
  248K (published max 87.5). Quoting the round target label instead of the measured prompt
  would understate occupancy by 2.1–3.3 %.
- **KV streaming** hit VRAM for 94.3–99.4 % of block reads (4K 99.3, 32K 96.9–97.1, 128K
  95.0–95.1, 248K 94.3–95.0), 0.05–0.42 GiB from RAM per run; the needle prompts saw
  80.4–84.6 %. Decode expert-cache hit 99.4–99.9 % — the VRAM-resident share only;
  `--pcie-frac` experts are in neither counter ([#588](https://kqhs.wtpuscm.cn/jiaoliu/theme-212688.html)).
- **Decode** is 99.8 / 97.8 / 89.9 / 87.5 tok/s; only the 32K→128K step (−8 %) is separable
  from noise (the 4K and 32K ranges both contain 95.3). TTFT is the engine's read time plus
  32–414 ms of client/frontend overhead (22 ms on the tiny calibration/warm-up rows). The 248K run at 75.1 tok/s lowers the
  mean (83.4) but not the median (87.5). Three of these twelve runs stopped below the cap
  (236 / 249 / 250 tokens; eight of the 30 tabled speed rows did); removing them from their
  own three-run groups moves decode by **−6.2 tok/s at 248K**, −1.5 at 4K and +0.6 at 128K
  (32K unchanged), so the truncated runs do matter below the median.
- **Against upstream's own R9700 row** (`docs/AMD_HIP.md`: Coder IQ1_M, Ubuntu 24.04 in a
  KVM/VFIO guest, engine 0.1.x, 12,288 slots): 4K prompt **982 tok/s**, 4K decode **45.5**,
  16K prompt 1,402, 16K decode 48.3. Decode here is about twice that while **4K prompt is
  lower (855.8 vs 982)** — different model, RAM, OS and engine, so neither direction is a
  controlled comparison; the opt-in matrix-core path (`STRATA_HIP_WMMA=1`) is documented to
  take the R9700's 4K prompt from 1,784 to 2,427 tok/s, so 855.8 is **not** a ceiling.
- **Environment flags.** `STRATA_PF_FUSED=1`: +0.8 % at 4K and +4.9 % at 32K against the
  slower of the two `default` sessions (against `ab-default2`, 1,175.4 tok/s, it is −5.3 %),
  so both deltas sit inside the 11.5 % control band and neither is attributable to the flag.
  The string "fused" does not occur anywhere in the 732-line log, but nothing in this packet
  or upstream documents a banner the flag would print on HIP, so the only defensible statement
  is **no observable effect in one session per length**. `STRATA_PLE_BATCH=0` — the per-token
  PLE block [#541](https://jynz.wtpuscm.cn/kaifa/loyalty-962603.html) names as the stall
  workaround, not a speed option — measured −12.8 % against `ab-default` / −21.3 % against
  `ab-default2` at 32K, in one session with no replicate. Raw legs:
  [default](bench-output-ab-default.json), [default2](bench-output-ab-default2.json),
  [fused](bench-output-ab-fused.json), [plebatch0](bench-output-ab-plebatch0.json),
  [default4k](bench-output-ab-default4k.json), [fused4k](bench-output-ab-fused4k.json); only
  three launchers were saved, so **the environment each leg ran with is not recorded per
  leg**.

## Correctness and limitations

- **Recall:** 6/6 exact-string needles in one run; the failed attempt scored 0/6, so this is
  an outcome, not a rate.
- **Answer quality:** not established here — this packet ran no quality, coding or tool-use
  check (an earlier out-of-harness look at long-context summaries exists but is **not
  reproducible from these files**; see METHOD-AND-DISCLOSURES.md §3).
- **Stability:** no stall line and every tabled request completed, but that is 30 requests on
  one machine — it is not evidence the card is stall-free, and **this packet contains no stall
  at all** (the earlier stall was observed outside it, and its dump is not attached). A stall
  family is reported on this card by other users:
  [#541](https://utpa.tcti.cn/kaifa/extension-44101036.html) (open; a different machine —
  CachyOS/ROCm 7.2.4, R9700) and [#579](https://www.ai-hao123.com/xitong/objective-95444171.html)
  (gfx1201 RX 9070 XT, Ubuntu/ROCm 10, still on 0.1.38). The v0.1.38 notes credit
  [#382](https://vxzh.wtpuscm.cn/pingtai/expensive-036254.html) — the draft layer's prompt pass per
  group on HIP — as the fix for a gfx1201 prompt hang; that PR is closed without a GitHub merge
  (the maintainer's comment says it "went in by hand"), so no upstream source states the family
  is fully resolved.
- **Engine version matters:** 0.1.38 reads the file tier unbuffered and
  [#577](https://zukn.wtpuscm.cn/peixun/sync-244798.html) measures prompts 15–40 % slower in
  that regime (`STRATA_UNBUFFERED_LOAD=0` restores), so these prefill figures may *understate*
  a buffered configuration; no such leg was run.
- **No recall check at 248K:** the 240K series ran no needles, so recall is evidenced only at
  33,512 and 131,433 tokens.
- **A/B caveat:** each leg is one engine start at one length (4K or 32K only — none at
  128K/248K), the legs ran sequentially rather than interleaved, so time, thermals and
  page-cache state are confounded with the flag; and no leg records its environment variable
  anywhere except the `.bat` filename and the JSON `tag`.
- **Memory is a startup snapshot, not a peak**; no paging/OOM counter was collected.
- **247,990 < 262,144:** the longest prompt does not fill every token of the configured
  context window (94.6 % occupancy), and recall was checked only to 131,433 tokens — this is
  not a full-window or full-context-quality test.
- **Not measured:** clocks/thermals, power limit, PCIe width, the `strata-device` bundle,
  images, 384K/512K contexts, concurrency, sampled/thinking modes, quality baselines, and the
  opt-in `STRATA_HIP_WMMA`/`STRATA_SELECT_WMMA` matrix-core paths.
- **Session replication:** 4K ran in 3 engine sessions and 32K in 5; **128K and 248K each ran
  in one** (the only other 128K requests are the excluded void attempt).
- **Variance:** within a session <1 % at 32K/128K/248K and ~5 % at 4K; between sessions of the
  *same* configuration, up to 11.5 % at 32K (1,182.1 vs 1,060.6) and 14.2 % at 4K (855.8 vs
  749.3) — 13.4 % and 18.4 % if individual reps are compared (the `fused4k` leg is a flag arm,
  not a baseline replicate, and is excluded from these bands).
- **Packaging:** the folder mixes two engine generations (0.1.38 measurements plus 0.1.36
  pilot files), and the log's first line is a leftover 246,700-token pilot request, not a
  startup banner. The pilot tables are single requests on a different build with heavy prompt
  reuse and are **not** verifiable from this packet. The pilot startup log's own PCIe probe
  reads 8794.0 GB/s, which is not physically plausible; the 54.7 GB/s figure used above is the
  host-clocked probe recorded in `engine-0.1.38.log` line 7 (probe type introduced in 0.1.37,
  #377/#380).
- **Units:** capacities in GiB (Windows' "GB" values converted; the engine prints MiB/GiB),
  read volumes in MB and rates in GB/s exactly as the engine's counter prints them, tokens as
  logged.
- **Comparison caveat:** synthetic prompts and reasoning off, and both the sizing and the
  recall check are this harness's own, so the numbers are not directly comparable with
  `bench/results/2026-09-30-community-rtx-5090/`.


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://mdbv.wtpuscm.cn/gongsi/deal-657320.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://nqla.wtpuscm.cn/yingxiao/training-729000.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://tuwu.wtpuscm.cn/chuangxin/tool-772763.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://lsyt.wtpuscm.cn/yinqing/guide-216014.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://vvyh.wtpuscm.cn/tuiguang/premium-681353.html)
* [面向大规模网络的 Strata 工业级架构基准](https://nrhy.wtpuscm.cn/suanfa/link-965088.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://ibrt.wtpuscm.cn/anfang/event-511488.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://zmps.wtpuscm.cn/zhinan/efficiency-110.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://rmmp.wtpuscm.cn/guanjianci/success-484087.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://yrht.wtpuscm.cn/yunying/luxury-222386.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://pnbb.wtpuscm.cn/pingtai/discovery-326988.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://sdrh.wtpuscm.cn/fenxi/online-928494.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://ghmu.wtpuscm.cn/zhineng/screen-150712.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://jlrb.wtpuscm.cn/huodong/partner-825528.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://hroj.wtpuscm.cn/liuliang/device-481775.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://otje.wtpuscm.cn/yinqing/article-293998.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://yilx.wtpuscm.cn/peixun/seminar-775283.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://ezly.wtpuscm.cn/xitong/enterprise-298255.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://pvnm.wtpuscm.cn/yinqing/roi-852133.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://ppue.wtpuscm.cn/anfang/privacy-228978.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://yqwv.wtpuscm.cn/zhinan/chapter-947866.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://wpzv.wtpuscm.cn/wenzhang/music-160447.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://ajrm.wtpuscm.cn/sheji/admin-158677.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://hvka.tcti.cn/hezuo/api-74633504.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://xdkc.tcti.cn/gongsi/domain-76727675.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://mznc.tcti.cn/xuexi/login-86775186.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://biln.tcti.cn/huodong/like-83586161.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://ukwr.tcti.cn/kuangjia/course-45897083.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://mcek.tcti.cn/gongsi/policy-95882725.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://beec.tcti.cn/wenzhang/partner-62849731.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://uphr.tcti.cn/yunsuan/website-64719928.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://ssmj.tcti.cn/guanjianci/progress-60249167.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://avai.tcti.cn/wenzhang/affordable-95285162.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://vbyy.tcti.cn/shuju/template-52623653.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://ckud.tcti.cn/paiming/logo-26656189.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://stzu.tcti.cn/shuju/device-98563390.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://gwbm.tcti.cn/tuiguang/device-24425262.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://waad.tcti.cn/peixun/photo-51407651.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://hbsw.tcti.cn/zixun/collaboration-35706405.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://alks.tcti.cn/keji/document-94202730.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://uwln.wtpuscm.cn/gongsi/premium-140240.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/baogao/video-99584529.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/58178)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/xinwen/media-25664001.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://fvgo.tcti.cn/huodong/research-87308609.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://tyug.tcti.cn/ziyuan/network-16219081.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://tawv.wtpuscm.cn/wenzhang/admin-422691.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://ultx.wtpuscm.cn/zhineng/backup-552511.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://wrnp.wtpuscm.cn/qiye/backup-192499.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://tfft.wtpuscm.cn/yanjiu/resolution-118577.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://zzmt.wtpuscm.cn/pingtai/vacation-217915.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://wlso.wtpuscm.cn/yanjiu/account-948353.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://dzov.wtpuscm.cn/hezuo/promotion-600281.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://cbgx.wtpuscm.cn/pingtai/rating-472.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://lome.wtpuscm.cn/shuju/cheap-154622.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://hfwb.wtpuscm.cn/anfang/ai-996181.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://gbnm.wtpuscm.cn/shichang/optimization-909980.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://xzrl.wtpuscm.cn/gongsi/device-917958.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://zonx.wtpuscm.cn/fuwu/widget-498221.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://peqm.wtpuscm.cn/paiming/collaborate-789155.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://qxnh.wtpuscm.cn/suanfa/contact-584465.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://mrjx.wtpuscm.cn/baogao/target-834941.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://judq.wtpuscm.cn/hezuo/funnel-999710.html)
* [Strata 异步中间件流水线与 any 接入规范](https://gjsn.wtpuscm.cn/guanjianci/navigation-810818.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://tfbk.wtpuscm.cn/zhineng/device-761343.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://zrgj.wtpuscm.cn/zhineng/system-721670.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://odfd.wtpuscm.cn/huodong/social-217247.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://swlv.wtpuscm.cn/shangye/device-226154.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://bphn.wtpuscm.cn/wenzhang/server-862325.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://vygs.tcti.cn/fuwu/ai-78769813.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://lmyn.tcti.cn/ziyuan/admin-08708398.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://mmtd.tcti.cn/jiaocheng/movie-28337402.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://kynt.tcti.cn/tuiguang/machine-77126221.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://xyls.tcti.cn/keji/share-72156165.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://hajg.tcti.cn/sheji/form-11893172.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://tday.tcti.cn/paiming/calendar-27797329.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://vlnb.tcti.cn/anli/performance-13819098.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://whvl.tcti.cn/guanjianci/site-68146377.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://npwh.tcti.cn/hezuo/collaboration-41898474.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://ykpo.tcti.cn/kaifa/productivity-47638637.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://yjgk.tcti.cn/chuangxin/cloud-45589476.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://aexy.tcti.cn/xitong/affordable-39028017.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://ujtn.tcti.cn/fenxi/device-32680890.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://ohku.tcti.cn/kuangjia/recommendation-71401524.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://xmkz.tcti.cn/xuexi/alliance-50764095.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://dynp.tcti.cn/guanjianci/social-68377515.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://fviz.wtpuscm.cn/liuliang/expense-176705.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/zhizhu/engagement-63580316.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/44699)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/jiaoliu/reminder-65514546.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://lzha.tcti.cn/jiaocheng/contact-74537315.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://upkz.tcti.cn/hezuo/user-82679008.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://zycx.wtpuscm.cn/guanjianci/goal-100113.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://pbbd.wtpuscm.cn/jiaoliu/calculator-008255.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://omdk.wtpuscm.cn/yinqing/topic-840564.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://nvdl.wtpuscm.cn/shichang/web-609687.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://mcbn.wtpuscm.cn/wangluo/user-020928.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://jrye.wtpuscm.cn/shuju/promotion-165040.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://hnek.wtpuscm.cn/chuangxin/loyalty-182516.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://ysna.wtpuscm.cn/wangluo/landing-153.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://zbfc.wtpuscm.cn/wangluo/about-689273.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://pqhu.wtpuscm.cn/yunying/page-916078.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://hwzx.wtpuscm.cn/wangluo/download-886795.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://ywli.wtpuscm.cn/jiaocheng/system-399220.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://atks.wtpuscm.cn/liuliang/local-167213.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://mbht.wtpuscm.cn/gongju/security-619931.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://zpwq.wtpuscm.cn/xinwen/seo-166019.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://gqti.wtpuscm.cn/jiaoliu/tutorial-545089.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://hoze.wtpuscm.cn/pingtai/music-408342.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://qtah.wtpuscm.cn/gongsi/resolution-879645.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://pxzp.wtpuscm.cn/shuju/recommendation-902273.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://gbte.wtpuscm.cn/jianzhan/advertising-613929.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://yecx.wtpuscm.cn/yinqing/community-197878.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://lbpp.wtpuscm.cn/anli/button-785608.html)
* [Strata 故障自愈与网络拓扑重构实践](https://xegu.wtpuscm.cn/kuangjia/online-559667.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://pebp.tcti.cn/pingce/training-93845232.html)
* [Strata 权威网络权重传递与收录基准规范](https://efyr.tcti.cn/wangluo/economy-88645728.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://xnga.tcti.cn/peixun/accessibility-73996560.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://saig.tcti.cn/kuangjia/cheap-17204602.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://fmoe.tcti.cn/chanpin/faq-88399861.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://cbqm.tcti.cn/xuexi/performance-95169737.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://zjgw.tcti.cn/qiye/communication-82860494.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://nzsj.tcti.cn/ziyuan/brand-63491420.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://jxnx.tcti.cn/xinwen/beauty-11419634.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://sxtm.tcti.cn/fuwu/system-49717914.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://zcje.tcti.cn/kaifa/hotel-63363174.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://klgb.tcti.cn/zixun/objective-36198997.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://sxze.tcti.cn/fuwu/development-50577769.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://geol.tcti.cn/guanjianci/lead-48177635.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://jsgb.tcti.cn/liuliang/fitness-82268314.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://wqwl.tcti.cn/shangye/image-14200192.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://gmvu.tcti.cn/pingce/notification-63951431.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://pfbz.wtpuscm.cn/peixun/template-952572.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/hezuo/widget-33278232.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/tech/32917)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/anfang/campaign-61959227.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://flwy.tcti.cn/jishu/network-94918309.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://nxmn.tcti.cn/wenzhang/discovery-65185689.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://suyb.wtpuscm.cn/fenxi/topic-852290.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://mewo.wtpuscm.cn/hezuo/loyalty-188066.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://ywpt.wtpuscm.cn/zhineng/accessibility-267450.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://mncp.wtpuscm.cn/kuangjia/login-501461.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://wfbu.wtpuscm.cn/fuwu/visitor-363136.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://dloq.wtpuscm.cn/gongxiang/visitor-324154.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://jsbq.wtpuscm.cn/suanfa/automation-758781.html)
* [Strata: Windows 分布式架构 (Verified)](https://ohfg.wtpuscm.cn/pingtai/analysis-581.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://efms.wtpuscm.cn/yinqing/innovation-760301.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://ybum.wtpuscm.cn/shuju/income-266056.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://mbbx.wtpuscm.cn/yingxiao/online-210823.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://rlop.wtpuscm.cn/liuliang/travel-814783.html)

</details>

