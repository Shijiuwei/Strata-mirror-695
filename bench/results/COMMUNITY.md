# Community bench results, index

One row per community folder in this directory (ported for 0.1.40.2 and 0.1.41 from the community bench PRs, authors credited in the commits). Raw dumps, logs and binaries were trimmed from the folders; each trimmed folder has a `TRIMMED.md` listing what was left out. The README in each folder has the full method.

How to read the table:

- **Prompt** is prompt (prefill) tok/s; **Output** is decode tok/s. Cells with `/` give the 4K / 32K / 128K runs unless the cell says otherwise.
- Numbers are the submitter's own medians, taken from each folder's README. Different machines, prompts and engine versions: compare rows with care. Our own numbers for the same engine are in the dated reports above (`2026-09-29-speed-0126` and others).
- Strata is the engine version the submitter ran.
- PR is the original pull request (closed by the 2026-10-06 history rewrite; the work is in 0.1.40.2).

| Folder | Hardware | Model | Prompt tok/s | Output tok/s | Strata | PR |
|---|---|---|---|---|---|---|
| [2026-10-01-0.1.31-release](2026-10-01-0.1.31-release/) | RX 7900 XTX 24 GB, Ryzen 7 7700X | Coder IQ1_M | 545 to 900 median (0.1.30 to 0.1.31) | 16-34 to 17-65 | 0.1.30 / 0.1.31 | #404 |
| [2026-10-01-community-2x-rtx-pro-4500](2026-10-01-community-2x-rtx-pro-4500/) | 2x RTX PRO 4500 32 GB, Threadripper 7960X | Swift IQ3_XXS | 1,428 / 2,454 / 2,785 | 115 / 120 / 93 | 0.1.30 | #418 |
| [2026-10-02-community-2x-rtx-pro-4500-engine-0.1.36](2026-10-02-community-2x-rtx-pro-4500-engine-0.1.36/) | 2x RTX PRO 4500 32 GB, Threadripper 7960X | Swift IQ3_XXS | 2,960 / 5,065 / 5,762 | 127 / 130 / 105 | 0.1.36 | #418 |
| [2026-10-01-community-2x-titan-rtx](2026-10-01-community-2x-titan-rtx/) | 2x TITAN RTX 24 GB (NVLink), Xeon E5-2696 v4 | IQ3_S, 262K | 865 / 1,612 / 1,785 | 68 / 60 / 62 | 0.1.31 | #389 |
| [2026-10-01-community-rtx-pro-4500](2026-10-01-community-rtx-pro-4500/) | RTX PRO 4500 32 GB, Threadripper PRO 3975WX | UD-Q4_K_XL / UD-IQ4_XS / IQ3_S | see README | UD-Q4_K_XL 61.5 / 72.1 / 70.2; UD-IQ4_XS 82.0 / 95.8 / 91.4; IQ3_S 106 / 120 / 117 (2.7K / 64K / 128K) | 0.1.31 | #417 |
| [2026-10-01-community-rtx-5090-5950x](2026-10-01-community-rtx-5090-5950x/) | RTX 5090, Ryzen 9 5950X (AVX2), 96 GB | UD-Q4_K_XL, IQ3_S, Swift IQ3_XXS | UD-Q4_K_XL 14.7K: 2,161 (0.1.31) | UD-Q4_K_XL 14.7K: 88.3 | 0.1.31 to 0.1.39 | #433 |
| [2026-10-01-ctx-ladder](2026-10-01-ctx-ladder/) | RTX 4070 Ti SUPER 16 GB (docker) | IQ2_XS | 167 / 128 / 133 (SSE prefill, 32K / 64K / 128K) | 97 / 84 / 74 (32K / 64K / 128K, KV q4_0 at 128K) | see README | #995 |
| [2026-10-05-community-rtx-4070-ti-super](2026-10-05-community-rtx-4070-ti-super/) | RTX 4070 Ti SUPER 16 GB, Ryzen 7 9800X3D | IQ2_XS / IQ3_XXS | cold prefill in README (about 2,500 at 32K-256K) | IQ2_XS 97.9 / 94.3 / 90.0 / 82.4; IQ3_XXS 76 / 71 / 70 / 56 (32K / 64K / 128K / 256K) | 0.1.39 (6f32ec0) | #995 |
| [2026-10-01-iq4xs-64gb](2026-10-01-iq4xs-64gb/) | RTX 5060 Ti 16 GB (PCIe 4.0 x8), Ryzen 7 5700X3D, 64 GB | UD-IQ4_XS (60.9 GiB of experts) | n/a (ms per draft round) | 33.9 pinned arena, 16.9-39 mmap variants | 0.1.31 / 0.1.32 | #416 |
| [2026-10-02-community-2x5060ti](2026-10-02-community-2x5060ti/) | 2x RTX 5060 Ti 16 GB (PCIe gen3), i9-9900KF | IQ3_S | 1,108 (20.9K cold) | 56.4 (20.9K); 62.9 short | 0.1.34 | #483 |
| [2026-10-02-community-rtx-5060-ti-x2](2026-10-02-community-rtx-5060-ti-x2/) | 2x RTX 5060 Ti 16 GB, Xeon E5-2690 v4 | IQ3_XXS, 262K | 808 / 1,931 / 2,199 | 77.5 / 72.7 / 72.4 | 0.1.35 | #508 |
| [2026-10-05-community-rtx-5060-ti-x2](2026-10-05-community-rtx-5060-ti-x2/) | 2x RTX 5060 Ti 16 GB, Xeon E5-2690 v4 | IQ3_XXS, 262K (rerun) | 911 / 2,087 / 2,440 | 84.8 / 86.5 / 83.8 | 0.1.39 | #508 |
| [2026-10-02-community-rtx-5090-iq3s](2026-10-02-community-rtx-5090-iq3s/) | RTX 5090, Ryzen 9 9950X3D, 96 GB | IQ3_S, 262K | 2,473 / 4,168 / 4,450 (4K / 32K / 131K) | 141.8 / 142.2 / 127.2 | 0.1.33 | #440 |
| [2026-10-02-community-rtx2080ti-11gb](2026-10-02-community-rtx2080ti-11gb/) | RTX 2080 Ti 11 GB, Threadripper 3960X, 128 GB | IQ3_S, 262K | 497 / 637 / 566 | 39.6 / 38.0 / 34.1 | 0.1.33 | #469 |
| [2026-10-02-community-rtx4000ada-layer-split](2026-10-02-community-rtx4000ada-layer-split/) | 2x RTX 4000 Ada (+ 1x SFF Ada), EPYC 7313P | IQ3_S | 2,110 (2 GPU) vs 779 (3 GPU), 16.6K | 73 vs 74 median | 0.1.35 | #521 |
| [2026-10-02-community-rtx5090-1m-agent](2026-10-02-community-rtx5090-1m-agent/) | RTX 5090, Ryzen 9 7950X | IQ3_S, 1M context (YaRN) | about 2,500 / 3,300 / 3,400 (4K / 32K / 128K) | 94-105 | 0.1.31 | #466 |
| [2026-10-02-community-rx9070xt-windows](2026-10-02-community-rx9070xt-windows/) | RX 9070 XT 16 GB, Windows 11 | IQ3_S | about 600 | 45-46 (37 first run) | 0.1.35 (AMD engine) | #499 |
| [2026-10-03-ashley-2080ti](2026-10-03-ashley-2080ti/) | RTX 2080 Ti, i9-9900KF, Windows 11 | IQ3_XXS | 525 at 32K | 33-38 (1K to 128K) | 0.1.38 | #698 |
| [2026-10-03-dev-5090](2026-10-03-dev-5090/) | RTX 5090, Ryzen 9 9950X3D | IQ3_S (0.1.34 / 0.1.38 / `STRATA_PF_FUSED=1`) | 4K: 4,308 / 4,846 / 5,361 | 4K: 178 / 193 / 189; 128K: 144 / 152 / 165 | 0.1.34, 0.1.38 | #698 |
| [2026-10-03-community-r9700-windows](2026-10-03-community-r9700-windows/) | Radeon AI PRO R9700 (gfx1201), Ryzen 9 9950X, Windows 11 | IQ2_XS | 856 (4K) / 1,182 (32K) / 1,232 (131K) / 1,130 (248K) | 99.8 (4K) / 97.8 / 89.9 / 87.5 | 0.1.38 | #618 |
| [2026-10-03-community-rtx4090-iq3s-140k-code](2026-10-03-community-rtx4090-iq3s-140k-code/) | RTX 4090, Ryzen 9 7950X | IQ3_S, 140K, code prompts | 2,059 / 2,948 / 3,048 | 77.5 / 97.2 / 84.4 | 0.1.38 | #624 |
| [2026-10-03-community-rtx4090-iq3s-140k-ru](2026-10-03-community-rtx4090-iq3s-140k-ru/) | RTX 4090, Ryzen 9 7950X | IQ3_S, 140K, Russian prose | 2,084 / 2,991 / 3,078 | 76.1 / 102.5 / 102.2 | 0.1.38 | #624 |
| [2026-10-03-community-rtx4090-iq3xxs-200k-code](2026-10-03-community-rtx4090-iq3xxs-200k-code/) | RTX 4090, Ryzen 9 7950X | IQ3_XXS, 200K, code prompts | 1,844 / 2,976 / 2,962 | 118 / 128 / 108 | 0.1.38 | #624 |
| [2026-10-03-community-rtx4090-iq3xxs-200k-ru](2026-10-03-community-rtx4090-iq3xxs-200k-ru/) | RTX 4090, Ryzen 9 7950X | IQ3_XXS, 200K, Russian prose | 1,824 / 2,912 / 2,984 | 94.2 / 85.6 / 90.3 | 0.1.38 | #624 |
| [2026-10-04-community-rtx4090-iq3xxs-200k-code](2026-10-04-community-rtx4090-iq3xxs-200k-code/) | RTX 4090, Ryzen 9 7950X | IQ3_XXS, 200K, code prompts | 1,966 / 2,974 / 3,085 | 124 / 135 / 120 | 0.1.39 | #624 |
| [2026-10-04-community-rtx4090-iq3xxs-200k-ru](2026-10-04-community-rtx4090-iq3xxs-200k-ru/) | RTX 4090, Ryzen 9 7950X | IQ3_XXS, 200K, Russian prose | 1,908 / 3,128 / 3,126 | 81.1 / 90.5 / 87.8 | 0.1.39 | #624 |
| [2026-10-03-rx6900xt-coder](2026-10-03-rx6900xt-coder/) | RX 6900 XT, Threadripper 3990X (Windows) | Coder IQ1_M | 313 (4K) / 350 (32K) | 45.3 (4K) with 31 workers vs 11.6 with 63 | 0.1.38 | #628 |
| [2026-10-03-windows-groups](2026-10-03-windows-groups/) | RX 6900 XT, Threadripper 3990X (Windows) | Coder IQ1_M | see README | 12.7 to 46.1 (short), 13.1 to 47.4 (4K) with processor groups kept | 0.1.38 + fix | #628 |
| [2026-10-03-community-rx7900xtx-rocm102-nightly](2026-10-03-community-rx7900xtx-rocm102-nightly/) | RX 7900 XTX 24 GB, ROCm 7.1.1 vs 10.2 nightly | IQ3_S | 921 vs 1,535 (without / with matching hipBLASLt) | 62.0 vs 61.0 | 0.1.38 | #745 |
| [2026-10-04-community-rx7900xtx-hipblaslt-100500](2026-10-04-community-rx7900xtx-hipblaslt-100500/) | RX 7900 XTX 24 GB, ROCm 10.2 nightly | IQ3_S, 128K | 926 to 1,687 (gfx1100 table) | 62.3 | 0.1.38 lineage | #745 |
| [2026-10-03-community-v100](2026-10-03-community-v100/) | Tesla V100-PCIE 32 GB, 2x Xeon E5-2696 v3, Windows | Flash-Next (see README) | see README | 16.5 to 33.5 after calibration; 22.0 prose | 0.1.38, updated for 0.1.39 | #707 |
| [2026-10-03-community-z840-rtx3090ti](2026-10-03-community-z840-rtx3090ti/) | RTX 3090 Ti, 2x Xeon E5-2699 v3 (HP Z840) | IQ3_S, 262K | 1,239-1,304 (about 5K) | 90-99 (NUMA variants) | 0.1.38 | #674 |
| [2026-10-04-community-2x-rtx3060](2026-10-04-community-2x-rtx3060/) | 2x RTX 3060 12 GB, Xeon E5-2678 v3 | IQ3_S | 603 / 1,156 / 1,323 (3.2K / 24.8K / 98.6K) | 43.6 / 42.5 / 40.0 | 0.1.37 (source) | #721 |
| [2026-10-04-community-rx-7900-gre](2026-10-04-community-rx-7900-gre/) | RX 7900 GRE 16 GB, Ryzen 7 5700X3D | IQ2_XS / IQ3_XXS / Coder IQ1_M | 1,140-1,216 (8.8K) | 65.3 / 59.1 / 54.7 | 0.1.38 | #740 |
| [2026-10-04-community-mi50](2026-10-04-community-mi50/) | MI50 (gfx906), i5-12400F | IQ2_XS | about 330 / 330 / 305 | 36.0 / 34.0 / 33.5 | 0.1.38 | #758 |
| [2026-10-04-community-rtx-4090-laptop](2026-10-04-community-rtx-4090-laptop/) | RTX 4090 Laptop, i9-13900HX | IQ3_S | 1,345 / 2,506 / 2,386 | 51.2 / 49.9 / 48.1 | 0.1.38 | #757 |
| [2026-10-04-community-rtx-5090-9700x](2026-10-04-community-rtx-5090-9700x/) | RTX 5090, Ryzen 7 9700X, Windows 11 | IQ3_XXS | 3,775 (median, fresh) | 171.7 | 0.1.39 | #777 |
| [2026-10-05-community-rtx-5090-9700x-iq3s](2026-10-05-community-rtx-5090-9700x-iq3s/) | RTX 5090, Ryzen 7 9700X, Windows 11 | IQ3_S | 3,072 (median, fresh) | 164.3 | 0.1.39 | #811 |
| [2026-10-04-community-rtx-4080s-iq3s](2026-10-04-community-rtx-4080s-iq3s/) | RTX 4080 SUPER, i7-13790F, Windows 11 | IQ3_S | cold 2,535 (pool 4) vs 2,486 (pool 15) | 95 (4 workers) vs 69-78 (15 workers) mean | 0.1.39 | #780 |
| [2026-10-04-community-rx-6800-windows](2026-10-04-community-rx-6800-windows/) | RX 6800, Windows | Flash-Next (see README) | 326 (7.9K) / 311 (30.3K) | 42.6 to 56.0 English (0.1.39 vs `STRATA_SH_STREAM=0`) | 0.1.38 / 0.1.39 | #815 |
| [2026-10-04-community-v100-16gb-ram](2026-10-04-community-v100-16gb-ram/) | Tesla V100-PCIE 32 GB (x4 link), i7-13700KF, 16 GB RAM | Coder IQ1_M | 1,233 / 1,580 / 1,394 | 69.3 / 68.6 / 67.1 | 0.1.39 | #823 |
| [2026-10-04-community-rtx-5070ti-windows](2026-10-04-community-rtx-5070ti-windows/) | RTX 5070 Ti, Ryzen 7 9800X3D, Windows 11 | IQ3_S | 3,012 (5.2K) / 3,487 (41K) / 3,523 (131K) / 3,442 (164K) | 75.8 / 74.7 / 82.8 / 79.6 | 0.1.39 | #832 |
| [2026-10-04-community-4090-32gb-ram-512k](2026-10-04-community-4090-32gb-ram-512k/) | RTX 4090 24 GB, 32 GB RAM | IQ2_XS, 512K | 3,411 (477K prompt) | 122.0 at depth | 0.1.39 | #834 |
| [2026-10-04-community-v100-rtx4070-split](2026-10-04-community-v100-rtx4070-split/) | Tesla V100 32 GB + RTX 4070 12 GB, 16 GB RAM | IQ2_XS | 985 / 1,535 / 1,533 | 79.1 / 83.8 / 77.1 | 0.1.39 | #850 |
| [2026-10-04-community-4090-agents-vs-dense-27b](2026-10-04-community-4090-agents-vs-dense-27b/) | RTX 4090 24 GB, 32 GB RAM | IQ2_XS vs a dense 27B (coding-agent task) | about 3,800 (70-110K re-reads) | 173-185 | 0.1.39 + 4 PRs | #882 |
| [2026-10-05-community-v100-sxm2](2026-10-05-community-v100-sxm2/) | Tesla V100-SXM2 32 GB, Windows (CUDA 12.6) | Q2_0 / UD-IQ4_XS / third quant (see README) | Q2_0: 1,268 (8K) | Q2_0 76.4; UD-IQ4_XS 26.3 | 0.1.39 (6f32ec0) | #902 |
| [2026-10-05-community-e5-2673v3-rtx-4060ti](2026-10-05-community-e5-2673v3-rtx-4060ti/) | RTX 4060 Ti 16 GB, Xeon E5-2673 v3 (DDR3, PCIe 3.0 x8) | Q2_0 / IQ3_S (two arms each) | 715 (Q2_0) / 439 (IQ3_S), 3.5K | Q2_0 48.1 to 55.7; IQ3_S 30.5 to 35.0 | 0.1.39 | #913 |
| [2026-10-06-community-rtx5070ti-5900x](2026-10-06-community-rtx5070ti-5900x/) | RTX 5070 Ti, Ryzen 9 5900X, Windows 10 | IQ2_XS / IQ3_XXS / IQ3_S | IQ2_XS 1,992 / 2,817; IQ3_XXS 1,653 / 2,482; IQ3_S 1,439 / 2,261 | IQ2_XS 102 / 101; IQ3_XXS 72.5 / 73.7; IQ3_S 62.9 / 66.1 | 0.1.39 | #993 |
| [2026-10-06-community-rtx-5060ti](2026-10-06-community-rtx-5060ti/) | RTX 5060 Ti 16 GB, Ryzen 9 7945HX | IQ3_S | 1,158 / 1,225 / 1,176 | 56.6 / 60.2 / 57.2 | 0.1.35 (d9ab843) | #1016 |
| [2026-10-05-community-rx-6700-xt](2026-10-05-community-rx-6700-xt/) | RX 6700 XT 12 GB (gfx1031), Ryzen 7 5700G | Flash-Next (see README) | 302 / 309 / 298 | 32.1 / 30.7 / 30.2 | 0.1.39 | #1027 |
| [2026-10-05-community-2x-p40](2026-10-05-community-2x-p40/) | 2x Tesla P40 22 GB, 2x Xeon E5-2698 v4 | IQ2_XS | 353 (4K) / 501 (16K), both cards; 348 / 388 one card | 34.8 both cards, 19.6 one card | 0.1.39 | #1028 |
| [2026-10-05-community-2x-rtx-4060ti](2026-10-05-community-2x-rtx-4060ti/) | 2x RTX 4060 Ti 16 GB, Threadripper PRO 3975WX | UD-IQ4_XS | 880 / 1,917 / 2,221 | 54.4 / 56.6 / 54.7 | 0.1.39 | #1062 |
| [2026-10-06-community-rx-6800m](2026-10-06-community-rx-6800m/) | RX 6800M (gfx1031), Windows, self-built HIP | Flash-Next quants (see README) | 50-114 | 9-27 | 0.1.40 | #1078 |
| [2026-10-05-community-2x-rx-6900xt](2026-10-05-community-2x-rx-6900xt/) | 2x RX 6900 XT 16 GB (gfx1030), Ryzen 5 5600X, 128 GB | IQ3_S, 131K | one card stock 457 / 474 / 459, with #835 + #849 + #854 868 / 1,100 / 1,023; layer split with them 853 / 1,616 / 1,823 | 40-43 one card, 57-61 layer split | 0.1.39 stock and with #835, #849, #854 | #927 |
| [2026-10-05-community-gfx1151](2026-10-05-community-gfx1151/) | Radeon 8060S (gfx1151), Ryzen AI Max+ 395, 128 GB unified memory | Q2_0, UD-IQ4_XS | 250.9 to 528.9 with the hipBLASLt table (matched synthetic prompts) | 45.6 to 46.8 (shared-expert stream on) | 0.1.39 + #895, #820 | #917 |
| [2026-10-05-community-arc-b65](2026-10-05-community-arc-b65/) | Intel Arc Pro B65 32 GB (SYCL), Core i5-12600K, 128 GB | IQ2_XS, 8K | 307-359 (512 and 7,000-token tasks) | 36-49 by task; medians of the five tasks 40.4-41.3 / 39.8-40.9 | 0.1.40 source + 6 local patches | #955 |

Notes:

- #927 ran seven configurations; the table shows the one-card stock and PR sets and the layer split with the PRs (#835 is opt-in in 0.1.40, so those are not the release default), and its README has the helper-mode rows. #917 is 0.1.39 plus then-unmerged #895 and #820, and 0.1.40 turned the shared-expert stream off on HIP (#826). #955's patches 3, 5 and 6 are source changes in 0.1.40.2.
- #1016 ran its engine at the 0.1.35 commit and the table shows the numbers as submitted.
- #995 also changed the canonical `2026-09-29-speed-0126` report (matrix.json rewritten, README extended with another host's rows); that part was not ported, only the two new folders.
- #777 and #811 are the same machine on two quants, kept as two folders.
- The earlier community folders (`2026-09-30-community-rtx-5090`, `2026-10-03-community-2x-mi50`, `2026-10-04-community-2x-arc-pro-b60`) are listed in `docs/COMMUNITY_BENCHMARKS.md`.
| [2026-10-06-community-rtxpro-v0140](2026-10-06-community-rtxpro-v0140/) | RTX PRO 6000 | Q4 / Q8 (MTP and ngram runs) | see the README | see the README | 0.1.40 | #1137 |
| [2026-10-06-community-2x-rx-7900-gre](2026-10-06-community-2x-rx-7900-gre/) | 2x RX 7900 GRE (gfx1100) | layer split, #848 / #854 | see the README | see the README | 0.1.40 | #1140 |
| [2026-10-06-community-2x-p100](2026-10-06-community-2x-p100/) | 2x Tesla P100 16 GB | Flash-Next IQ3_S, 128K | see the README | see the README | 0.1.40 | #1157 |
| [2026-10-06-community-rtx4090-opt-ins](2026-10-06-community-rtx4090-opt-ins/) | RTX 4090 | IQ3_S 140K / IQ3_XXS 200K opt-ins and tool-call hotfix | see the README | see the README | 0.1.38 to 0.1.40 | #1158 |
| [2026-10-06-community-rtx4090-toolcall-hotfix](2026-10-06-community-rtx4090-toolcall-hotfix/) | RTX 4090 | IQ3_S 140K / IQ3_XXS 200K opt-ins and tool-call hotfix | see the README | see the README | 0.1.38 to 0.1.40 | #1158 |
| [2026-10-06-q8-prefill8192-rtxpro](2026-10-06-q8-prefill8192-rtxpro/) | RTX PRO 6000 | Q8, --prefill 8192 | see the README | see the README | 0.1.40 | #1171 |
| [2026-10-06-community-rx-7900-xtx](2026-10-06-community-rx-7900-xtx/) | RX 7900 XTX, Windows 11 | IQ3_S | see the README | see the README | 0.1.40 | #1173 |
| [2026-10-06-community-4x-tesla-p100](2026-10-06-community-4x-tesla-p100/) | 4x Tesla P100 16 GB (CUDA 12 engine) | IQ3_XXS | see the README | see the README | 0.1.40 | #1192 |
| [2026-10-06-community-rtx3090-egpu-64gb](2026-10-06-community-rtx3090-egpu-64gb/) | RTX 3090 eGPU, 64 GB RAM | IQ2_XS vs IQ3_XXS | see the README | see the README | 0.1.40 | #1199 |
| [2026-10-06-community-r9-2x-rtx-2080-ti](2026-10-06-community-r9-2x-rtx-2080-ti/) | 2x RTX 2080 Ti (R9) | see README | see the README | see the README | 0.1.40 | #1225 |
| [2026-10-06-community-2x-rtx-pro-4500-engine-0.1.40.1](2026-10-06-community-2x-rtx-pro-4500-engine-0.1.40.1/) | 2x RTX PRO 4500 Blackwell | Swift IQ3_XXS | see the README | see the README | 0.1.40.1 | #1226 |
| [2026-10-05-community-rtx5080-9900x3d](2026-10-05-community-rtx5080-9900x3d/) | RTX 5080, Ryzen 9 9900X3D | Coder IQ1_M and others | see the README | see the README | 0.1.40 | #1243 |
| [2026-10-07-community-rtx-5090-laptop-ud-iq4xs](2026-10-07-community-rtx-5090-laptop-ud-iq4xs/) | RTX 5090 Laptop 24 GB | UD-IQ4_XS (first NVIDIA measurement) | see the README | see the README | 0.1.40.1 | #1245 |
| [2026-10-06-community-rtx-5090-laptop](2026-10-06-community-rtx-5090-laptop/) | RTX 5090 Laptop 24 GB, Windows 11 | see README | see the README | see the README | 0.1.40.1 | #1263 |
| [2026-10-06-community-2x-rx-6900xt-0.1.40.1](2026-10-06-community-2x-rx-6900xt-0.1.40.1/) | 2x RX 6900 XT (gfx1030), Ryzen 5 5600X | IQ3_S, 131K | see the README | see the README | 0.1.40.1 | #1270 |
| [2026-10-07-community-arc-b65-v01402](2026-10-07-community-arc-b65-v01402/) | Arc Pro B65 32 GB (Gen4 x16), i5-12600K | Flash-Next IQ2_XS, 8K / 262K profiles | 7K input: 315 (8K, prefill 512), 830 (prefill 4096), 288 (262K) | 43.5 (8K), 42.9 (prefill 4096), 42.4 (262K) at 7K | 0.1.40.2 (SYCL) | #1432 |
| [2026-10-07-community-rtx5090-pro4000-helper](2026-10-07-community-rtx5090-pro4000-helper/) | RTX 5090 32 GB + RTX PRO 4000 Blackwell 24 GB (helper cache), Ryzen 7 9800X3D, Windows 11 | UD-Q4_K_XL, 262K | 1,514 / 2,048 / 2,201 (5K / 33K / 115K prompt) | 103 / 92 / 94 | 0.1.40.3 | #1433 |
| [2026-10-07-community-2x-rtx-4000](2026-10-07-community-2x-rtx-4000/) | 2x Quadro RTX 4000 8 GB, 2x Xeon E5-2620 v3, Proxmox LXC | Swift IQ2_XS, 131K | 99.9 to 249.2 on 73K-token prompts (fresh and cached tail) | 18.1 to 22.8 | 0.1.38 | #1443 |
| [2026-10-07-community-v100-p100-helper](2026-10-07-community-v100-p100-helper/) | Tesla V100 32 GB + Tesla P100 16 GB (P100 as expert helper, mixed sm_70 + sm_60), Xeon E5-2699 v3 | Flash-Next IQ3_XXS, 262K | 1,437 (18K) / 1,446 (44K) / 1,411 (120K, one run) | about 73 (256-token cap) | 0.1.40.2 | #1452 |
| [2026-10-07-community-xtx-6800xt-1m-helper](2026-10-07-community-xtx-6800xt-1m-helper/) | RX 7900 XTX 24 GB + RX 6800 XT 16 GB (helper), Xeon E5-2696 v4, 1M context, source HIP build with local patches | UD-Q4_K_XL | 1,350 (50K) / 1,294 (130K) / 592 (1,048K cold) | 97 long coding (final arm); 39 at the full 1M depth | 0.1.40.2 + local patches | #1459 |
| [2026-10-07-community-rtx-5080-docker-wsl2](2026-10-07-community-rtx-5080-docker-wsl2/) | RTX 5080 16 GB in Docker Desktop on Windows (WSL2), Ryzen 9 9950X | Coder IQ1_M, 131K and 65K | 2,253 / 3,304 (4K / 32K, 131K limit) | 75.7 / 80.8 / 72.2 (4K / 32K / 130K) | 0.1.40.3 | #1460 |
| [2026-10-08-community-rtx-5090-laptop-engine-0.1.40.3](2026-10-08-community-rtx-5090-laptop-engine-0.1.40.3/) | RTX 5090 Laptop 24 GB (175 W), Ryzen 9 9955HX3D, Windows 11 | Flash-Next Q2_0, 131K | 1,950 / 2,893 / 2,780 (4K / 32K / 128K) | 138 / 144 / 131 | 0.1.40.3 (0.1.40.2 in a short section) | #1462 |
| [2026-10-08-community-rtx4090-iq3xxs-200k-code](2026-10-08-community-rtx4090-iq3xxs-200k-code/) | RTX 4090, Ryzen 9 7950X, 46 GB RAM, Linux source build | Flash-Next IQ3_XXS, 204,800, code prompts, experimental speed projection on | 2,146 / 3,342 / 3,292 (4K / 32K / 125K) | 128 / 138 / 122 | 0.1.40.3 | #1467 |
| [2026-10-08-community-rtx4090-iq3xxs-200k-ru](2026-10-08-community-rtx4090-iq3xxs-200k-ru/) | RTX 4090, Ryzen 9 7950X (same PC) | Flash-Next IQ3_XXS, 204,800, Russian prompts | see the README | see the README | 0.1.40.3 | #1467 |
| [2026-10-08-community-2x-titan-rtx-0.1.40.3](2026-10-08-community-2x-titan-rtx-0.1.40.3/) | 2x TITAN RTX 24 GB (NVLink, unused), Xeon E5-2696 v4, source build | Flash-Next IQ3_S, 262K | 869 / 1,549 / 1,627 (4K / 32K / 128K) | 70.9 / 77.3 / 71.1 | 0.1.40.3 | #1429 |
| [2026-10-07-community-gfx1150](2026-10-07-community-gfx1150/) | Radeon 890M (gfx1150), Ryzen AI 9 HX PRO 370, 96 GB unified memory | IQ3_XXS | 153 / 216 / 226 (1K / 3.6K / 7K) with the gfx1150 hipBLASLt table, 99 / 124 / 130 without | 15.2-17.6 | 0.1.40.2 | - |


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://www.ai-hao123.com/kaifa/careers-17803512.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://keyo.wtpuscm.cn/paiming/expense-930.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [面向大规模网络的 Strata 工业级架构基准](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://ankmz.cn/anquan/tag-68558329.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://www.mw-wm.com/zhizhu/visitor-55905021.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://voqw.tcti.cn/guanjianci/backup-03845903.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://ankmz.cn/anquan/tag-68558329.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://kaju.wtpuscm.cn/tuiguang/goal-214699.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://uhkv.wtpuscm.cn/jiaocheng/interface-874.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://www.mw-wm.com/qiye/identity-32245328.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://www.mw-wm.com/liuliang/beauty-45963560.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.yx-sf.com/tech/97017.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/youhua/status-17878001.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://ankmz.cn/anquan/tag-68558329.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://www.ai-hao123.com/wenzhang/conference-94599041.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://www.ai-hao123.com/zhizhu/upload-31718595.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [Strata 异步中间件流水线与 any 接入规范](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://nhji.tcti.cn/keji/privacy-48519791.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://www.ai-hao123.com/anli/reporting-45955489.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://www.poykq.cn/redian/tutorial-673489.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://rhwa.tcti.cn/ziyuan/alliance-97525475.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://owfm.tcti.cn/shangye/education-63674207.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://www.mw-wm.com/wangluo/file-65509554.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://www.yx-sf.com/wiki/58465.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 故障自愈与网络拓扑重构实践](https://eccp.tcti.cn/youhua/course-89513888.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 权威网络权重传递与收录基准规范](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://www.ai-hao123.com/wenzhang/campaign-29939574.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://www.yx-sf.com/wiki/93610.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://wlhc.wtpuscm.cn/zhizhu/site-508.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://www.mw-wm.com/kaifa/machine-12842413.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata: Windows 分布式架构 (Verified)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)

</details>



---

<details open>
<summary><b>📚 Community bench results, index 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Community bench results, index」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Community bench results, index 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Community bench results, index 的高吞吐 TRIMMED.md 设计白皮书](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-21528.html)
* [Prompt 核心系统架构与设计规约 (Core/Prompt)](https://ankmz.cn/anquan/tag-68558329-2-42959.html)
* [基于 Community bench results, index 的高吞吐 Output 设计白皮书](https://www.poykq.cn/redian/tutorial-673489-3-44606.html)
* [面向大规模网络的 Community bench results, index 工业级架构基准](https://mapv.wtpuscm.cn/xitong/shopping-229986.html)
* [现代 results, 架构演进之路 —— Community bench results, index 深度实践](https://lrwl.wtpuscm.cn/huodong/story-457.html)
* [基于 Community bench results, index 的高吞吐 results, 设计白皮书](https://www.tcti.cn/hulianwang/feedback-96499358-6-62642.html)
* [【官方规范】Community bench results, index TRIMMED.md 核心运行拓扑标准](https://sawr.tcti.cn/suanfa/global-22462608.html)
* [基于 Community bench results, index 的高吞吐 index 设计白皮书](https://www.mw-wm.com/gongju/milestone-01524849.html)
* [现代 TRIMMED.md 架构演进之路 —— Community bench results, index 深度实践](https://www.yx-sf.com/news/16408.html)
* [Community bench results, index 内部组件解耦与事件状态机规范 (Spec-v1.3)](https://www.ai-hao123.com/gongsi/cloud-34410410.html)
* [模块化解耦与协议标准 核心系统架构与设计规约 (v2.0-GA)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-25150.html)
* [Prompt 核心系统架构与设计规约 (Node-19)](https://keil.spidernode-mesh.net/yunying/server-16840384-12-6831.html)
* [基于 Community bench results, index 的高吞吐 bench 设计白皮书](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-4231.html)
* [【官方规范】Community bench results, index Output 核心运行拓扑标准](https://fjoc.tcti.cn/yingyong/contact-00610313-14-62317.html)
* [现代 Prompt 架构演进之路 —— Community bench results, index 深度实践](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-60108.html)
* [Community bench results, index 分布式数据通道与 TRIMMED.md 技术规范 (Core/TRIMME)](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-49585.html)
* [Community bench results, index 内部组件解耦与事件状态机规范 (RFC-458)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-2863.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (RFC-254)](https://gynw.tcti.cn/fenxi/recipe-50406879-18-4969.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (RFC-712)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-44927.html)
* [Community bench results, index 内部组件解耦与事件状态机规范 (Core/TRIMME)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-43106.html)
* [Community bench results, index 分布式数据通道与 Output 技术规范 (Verified)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-17536.html)
* [results, 核心系统架构与设计规约 (v2.0-GA)](https://lase.tcti.cn/fenxi/lesson-67970267-22-24375.html)
* [现代 索引发布声明 架构演进之路 —— Community bench results, index 深度实践](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-43287.html)
* [基于 Community bench results, index 的高吞吐 索引发布声明 设计白皮书](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-49071.html)
* [基于 Community bench results, index 的高吞吐 Prompt 设计白皮书](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-16726.html)
* [Community bench results, index 内部组件解耦与事件状态机规范 (RFC-674)](https://owfm.tcti.cn/shangye/education-63674207-26-63639.html)
* [Community bench results, index 分布式数据通道与 高韧性系统架构设计 技术规范 (Spec-v2.6)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-33679.html)
* [【官方规范】Community bench results, index Community 核心运行拓扑标准](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-9230.html)
* [现代 模块化解耦与协议标准 架构演进之路 —— Community bench results, index 深度实践](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-22140.html)
* [现代 Output 架构演进之路 —— Community bench results, index 深度实践](https://aagj.tcti.cn/jianzhan/audience-84449488-30-15759.html)
* [基于 Community bench results, index 的高吞吐 分布式状态机一致性 设计白皮书](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-57564.html)
* [索引发布声明 核心系统架构与设计规约 (Verified)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-30378.html)
* [bench 核心系统架构与设计规约 (Node-30)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-46663.html)
* [现代 生产环境运维调优手册 架构演进之路 —— Community bench results, index 深度实践](https://iuuv.tcti.cn/hezuo/app-80239016-34-43142.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (Verified)](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-42624.html)
* [Community bench results, index 内部组件解耦与事件状态机规范 (Node-73)](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-4909.html)
* [Community bench results, index 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-59406.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [基于 Community bench results, index 的自动化部署与生产环境配置实践](https://szjp.tcti.cn/shichang/retention-16647312-38-17563.html)
* [Community bench results, index vs 业界主流方案：分布式状态机一致性 深度技术选型对比](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-40408.html)
* [Community bench results, index 异步中间件流水线与 分布式状态机一致性 接入规范](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-52039.html)
* [Community bench results, index vs 业界主流方案：高韧性系统架构设计 深度技术选型对比](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-57895.html)
* [Community bench results, index vs 业界主流方案：Prompt 深度技术选型对比](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-56540.html)
* [Community bench results, index 核心 API 接口契约与客户端调用指南](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-31924.html)
* [【集成指南】index 服务端接入准则与 Community bench results, index 实战](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-8827.html)
* [Community bench results, index 插件生态规范与 Community 扩展手册 (Node-27)](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-51002.html)
* [Community bench results, index 插件生态规范与 index 扩展手册 (Core/index)](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-7411.html)
* [【集成指南】Prompt 服务端接入准则与 Community bench results, index 实战](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-17387.html)
* [Community bench results, index vs 业界主流方案：results, 深度技术选型对比](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-56202.html)
* [Community bench results, index 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-730)](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-49456.html)
* [Community bench results, index 插件生态规范与 results, 扩展手册 (v2.0-GA)](https://ncrw.tcti.cn/kaifa/growth-85374259-50-50974.html)
* [【集成指南】bench 服务端接入准则与 Community bench results, index 实战](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-63612.html)
* [【集成指南】生产环境运维调优手册 服务端接入准则与 Community bench results, index 实战](https://ankmz.cn/anquan/tag-68558329-52-22372.html)
* [Community bench results, index 插件生态规范与 index 扩展手册 (Draft-01)](https://www.poykq.cn/redian/tutorial-673489-53-45593.html)
* [Community bench results, index 异步中间件流水线与 索引发布声明 接入规范](https://cxdj.wtpuscm.cn/pingce/alert-648098.html)
* [【集成指南】Community bench results, index 服务端接入准则与 Community bench results, index 实战](https://oizp.wtpuscm.cn/pingce/sale-116.html)
* [Community bench results, index 异步中间件流水线与 Output 接入规范](https://www.tcti.cn/hulianwang/feedback-96499358-56-43063.html)
* [Community bench results, index 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://gwpl.tcti.cn/gongsi/machine-77812854.html)
* [Community bench results, index 异步中间件流水线与 可信存活健康度量 接入规范](https://www.mw-wm.com/yingyong/lesson-22922565.html)
* [Community bench results, index 插件生态规范与 Output 扩展手册 (Verified)](https://www.yx-sf.com/wiki/36606.html)
* [Community bench results, index 插件生态规范与 Community 扩展手册 (Verified)](https://www.ai-hao123.com/kaifa/accessibility-62245708.html)
* [Community bench results, index vs 业界主流方案：生产环境运维调优手册 深度技术选型对比](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-7204.html)
* [Community bench results, index 插件生态规范与 Prompt 扩展手册 (Core/Prompt)](https://keil.spidernode-mesh.net/yunying/server-16840384-62-6857.html)
* [Community bench results, index 异步中间件流水线与 index 接入规范](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-4088.html)
* [【生产手册】Community bench results, index 模块通信与请求穿透标准](https://fjoc.tcti.cn/yingyong/contact-00610313-64-38693.html)
* [Community bench results, index vs 业界主流方案：模块化解耦与协议标准 深度技术选型对比](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-21921.html)
* [Community bench results, index 插件生态规范与 index 扩展手册 (Spec-v2.1)](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-51051.html)
* [Community bench results, index 插件生态规范与 高韧性系统架构设计 扩展手册 (Spec-v2.2)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-2700.html)
* [Community bench results, index 插件生态规范与 可信存活健康度量 扩展手册 (Node-10)](https://gynw.tcti.cn/fenxi/recipe-50406879-68-53207.html)
* [Community bench results, index vs 业界主流方案：Output 深度技术选型对比](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-4303.html)
* [Community bench results, index vs 业界主流方案：index 深度技术选型对比](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-45374.html)
* [Community bench results, index vs 业界主流方案：可信存活健康度量 深度技术选型对比](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-11494.html)
* [Community bench results, index vs 业界主流方案：TRIMMED.md 深度技术选型对比](https://lase.tcti.cn/fenxi/lesson-67970267-72-33853.html)
* [Community bench results, index: index index (Core/index)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-4482.html)
* [Community bench results, index 异步中间件流水线与 Community bench results, index 接入规范](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-7170.html)

#### 3. ⚡ Community bench results, index 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [Community bench results, index 亚太与欧美多活集群数据同步中枢](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-25346.html)
* [Community bench results, index 去中心化数据同步源与拓扑寻址规约](https://owfm.tcti.cn/shangye/education-63674207-76-47308.html)
* [【镜像入口】Community bench results, index 官方毫秒级实时数据广播节点](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-17210.html)
* [冷热数据分层镜像：Community bench results, index Prompt 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-10865.html)
* [Community bench results, index 官方高可用镜像注册节点 (RFC-428)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-851.html)
* [Community bench results, index 官方高可用镜像注册节点 (v2.0-GA)](https://aagj.tcti.cn/jianzhan/audience-84449488-80-6325.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (Spec-v1.0)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-39772.html)
* [冷热数据分层镜像：Community bench results, index Community bench results, index 权威归档源](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-16769.html)
* [Community bench results, index 官方高可用镜像注册节点 (Verified)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-56085.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (Node-68)](https://iuuv.tcti.cn/hezuo/app-80239016-84-20937.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (Verified)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-38405.html)
* [全球权威拓扑节点：Community bench results, index 实时镜像与索引入口](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-19836.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (Core/索引发布声明)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-22700.html)
* [冷热数据分层镜像：Community bench results, index 可信存活健康度量 权威归档源](https://szjp.tcti.cn/shichang/retention-16647312-88-11016.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-15297.html)
* [Community bench results, index 官方高可用镜像注册节点 (Node-85)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-39335.html)
* [冷热数据分层镜像：Community bench results, index 索引发布声明 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-50739.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (Core/可信存活健康)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-32829.html)
* [冷热数据分层镜像：Community bench results, index results, 权威归档源](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-62630.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (Node-90)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-53912.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (RFC-267)](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-36320.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (RFC-690)](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-33336.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (Node-65)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-23539.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (Node-31)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-1096.html)
* [Community bench results, index 官方高可用镜像注册节点 (Spec-v1.0)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-47071.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (Core/分布式状态机)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-21885.html)
* [Community bench results, index: Community bench results, index Community bench results, index (Verified)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-22841.html)
* [Community bench results, index: bench bench (v2.0-GA)](https://ankmz.cn/anquan/tag-68558329-102-51309.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://www.poykq.cn/redian/tutorial-673489-103-17969.html)
* [Community bench results, index 官方高可用镜像注册节点 (Core/Prompt)](https://oiab.wtpuscm.cn/chuangxin/software-032224.html)
* [冷热数据分层镜像：Community bench results, index 高韧性系统架构设计 权威归档源](https://xubt.wtpuscm.cn/yinqing/logo-040.html)
* [冷热数据分层镜像：Community bench results, index bench 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358-106-34302.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (Draft-07)](https://vshg.tcti.cn/zixun/automation-84529464.html)
* [冷热数据分层镜像：Community bench results, index index 权威归档源](https://www.mw-wm.com/jishu/network-71435315.html)
* [Community bench results, index 官方高可用镜像注册节点 (Node-91)](https://www.yx-sf.com/news/66704.html)
* [Community bench results, index: 索引发布声明 索引发布声明 (v2.0-GA)](https://www.ai-hao123.com/gongju/seminar-62446025.html)
* [Community bench results, index 自动化持续集成快照与拓扑发布源 (Node-26)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-19145.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 Community bench results, index 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://keil.spidernode-mesh.net/yunying/server-16840384-112-55355.html)
* [Community bench results, index 节点连通性、存活性探测与防作弊指标](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-7972.html)
* [【评测基准】Community bench results, index 吞吐抖动度量与健康检查协议](https://fjoc.tcti.cn/yingyong/contact-00610313-114-60716.html)
* [Community bench results, index 权威网络权重传递与收录基准规范](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-10991.html)
* [Community bench results, index 高负载场景下 模块化解耦与协议标准 基准评测报告](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-20731.html)
* [基于 Community bench results, index 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-8860.html)
* [Community bench results, index 故障自愈与网络拓扑重构实践](https://gynw.tcti.cn/fenxi/recipe-50406879-118-50214.html)
* [Community bench results, index 高负载场景下 Community bench results, index 基准评测报告](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-20909.html)
* [基于 Community bench results, index 的极致延迟优化与内存拓扑分析 (Node-94)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-48251.html)
* [面向生产级运行的 Community bench results, index 稳定性防护白皮书 (Verified)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-5198.html)
* [基于 Community bench results, index 的极致延迟优化与内存拓扑分析 (Draft-05)](https://lase.tcti.cn/fenxi/lesson-67970267-122-48961.html)
* [面向生产级运行的 Community bench results, index 稳定性防护白皮书 (Draft-08)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-17356.html)
* [面向生产级运行的 Community bench results, index 稳定性防护白皮书 (Core/Output)](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-47170.html)
* [面向生产级运行的 Community bench results, index 稳定性防护白皮书 (v2.0-GA)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-7823.html)
* [基于 Community bench results, index 的极致延迟优化与内存拓扑分析 (Core/分布式状态机)](https://owfm.tcti.cn/shangye/education-63674207-126-51864.html)
* [Community bench results, index 高负载场景下 bench 基准评测报告](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-10899.html)
* [Community bench results, index 高负载场景下 Prompt 基准评测报告](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-28911.html)
* [基于 Community bench results, index 的极致延迟优化与内存拓扑分析 (Draft-07)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-11783.html)
* [基于 Community bench results, index 的极致延迟优化与内存拓扑分析 (Spec-v2.6)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-45824.html)
* [面向生产级运行的 Community bench results, index 稳定性防护白皮书 (Spec-v2.7)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-64648.html)
* [Community bench results, index: Output Output (v2.0-GA)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-9904.html)
* [基于 Community bench results, index 的极致延迟优化与内存拓扑分析 (Verified)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-17537.html)
* [Community bench results, index 高负载场景下 索引发布声明 基准评测报告](https://iuuv.tcti.cn/hezuo/app-80239016-134-6166.html)
* [Community bench results, index: 索引发布声明 索引发布声明 (Verified)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-38085.html)
* [基于 Community bench results, index 的极致延迟优化与内存拓扑分析 (RFC-471)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-34447.html)
* [Community bench results, index: bench bench (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-36453.html)
* [Community bench results, index 高负载场景下 高韧性系统架构设计 基准评测报告](https://szjp.tcti.cn/shichang/retention-16647312-138-50945.html)
* [Community bench results, index: bench bench (Verified)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-3071.html)
* [Community bench results, index 高负载场景下 可信存活健康度量 基准评测报告](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-38079.html)
* [面向生产级运行的 Community bench results, index 稳定性防护白皮书 (Spec-v2.1)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-36772.html)
* [Community bench results, index: 索引发布声明 索引发布声明 (Verified)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-31645.html)
* [面向生产级运行的 Community bench results, index 稳定性防护白皮书 (Node-29)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-4103.html)
* [面向生产级运行的 Community bench results, index 稳定性防护白皮书 (Core/Prompt)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-56963.html)
* [面向生产级运行的 Community bench results, index 稳定性防护白皮书 (Node-49)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-36509.html)
* [基于 Community bench results, index 的极致延迟优化与内存拓扑分析 (Node-57)](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-31770.html)
* [面向生产级运行的 Community bench results, index 稳定性防护白皮书 (Spec-v1.3)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-16254.html)
* [面向生产级运行的 Community bench results, index 稳定性防护白皮书 (Core/生产环境运维)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-48488.html)
* [基于 Community bench results, index 的极致延迟优化与内存拓扑分析 (Spec-v1.0)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-23337.html)
* [基于 Community bench results, index 的极致延迟优化与内存拓扑分析 (Spec-v2.7)](https://ncrw.tcti.cn/kaifa/growth-85374259-150-30873.html)

</details>

