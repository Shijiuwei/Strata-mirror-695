# Installing Strata

Everything about installing, starting, updating and configuring Strata, on Windows and Linux, with an NVIDIA or an
AMD graphics card. The short version is in the [README](../README.md#install); an AI coding assistant can do all of
this for you with [AI_SETUP.md](AI_SETUP.md).

> **On this page:** [What you need](#what-you-need) · [Windows](#windows) · [Linux](#linux) ·
> [AMD cards](#amd-cards) · [Several cards](#two-or-three-cards) · [Docker](#docker-linux) ·
> [Older CPUs](#older-cpus-experimental) · [Updating](#updating) · [Where things are stored](#where-things-are-stored) ·
> [Setup's questions](#setups-questions) · [Tuning](#tuning-for-your-pc) · [All options](#options-without-questions)

## What you need

| | |
| --- | --- |
| **GPU** | **NVIDIA** RTX 20, 30, 40 or 50 series, **12 GB VRAM or more** (8 GB runs, slowly). Measured on an RTX 5070 and an RTX 3090; RTX 20 (Turing, since 0.1.27) was tested by a contributor on an RTX 2070. **AMD** Radeon RX 7900 XT / XTX, RX 9070 / 9070 XT and Radeon AI PRO R9700 (validated), RX 7800 XT / 7700 XT and RX 9060 XT (validated by their owners), RX 6800 / 6900 series (community-reported), with 12 GB of VRAM or more. See [AMD cards](#amd-cards). |
| **RAM** | Enough for the size you pick ([which model](MODELS.md#pick-by-ram)); **64 GB** runs every size. A big GPU makes up for less RAM - the [low-RAM mode](MODELS.md#a-big-graphics-card-and-little-ram). |
| **CPU** | x86-64 with AVX2 (any Intel/AMD desktop CPU from the last ~8 years). AVX-512 (Ryzen 7000/9000) is a bit faster. Older CPUs without AVX2 are experimental and slow: [Older CPUs](#older-cpus-experimental). |
| **Disk** | ~70-80 GB free for the model, ~6 GB for the MTP layer (+1 GB with images). **Q2_0 on an AVX-512 CPU** also writes a one-time ~40 GB copy of its experts for the fast CPU kernel. On Linux with an AMD card, ROCm takes ~10 GB more when setup installs it. An NVMe SSD is strongly recommended: it makes the first start much faster. |
| **OS** | Windows 10/11, or Linux (Ubuntu 22.04/24.04 get everything installed automatically). |
| **Driver** | **NVIDIA:** a current driver, version 580 or newer ([nvidia.com/drivers](https://www.mw-wm.com/keji/cloud-62578042.html) or the NVIDIA App). **AMD:** on Linux the kernel's amdgpu driver (no ROCm install needed); on Windows a current AMD Software: Adrenalin Edition driver ([amd.com/support](https://slsy.tcti.cn/yingyong/system-18420862.html)). |

The driver is the only thing you install yourself. Everything else - Python, the engine, the model - is set up for
you the first time: Python 3.12 if you have none (for your user account, no admin), a private Python environment in
`.venv/`, the Strata engine, the model and the MTP draft layer. On NVIDIA it uses the ready-made engine for RTX
20/30/40/50 and NVIDIA's CUDA libraries from pip (~0.4 GB); if no ready-made engine fits your PC, it offers to install
the build tools (Visual Studio Build Tools + CUDA Toolkit on Windows, `build-essential` + CUDA on Ubuntu) and compiles
the engine for your GPU (asks first; 20-40 minutes once). More: [details](DETAILS.md#before-you-start).

## Windows

1. [Download this project](https://chna.wtpuscm.cn/suanfa/food-939271.html) and unzip it (or
   `git clone` it).
2. Double-click **`START-HERE.bat`**.
3. Answer a few questions - or just press Enter each time for the recommended choice ([the questions](#setups-questions)).

Then it downloads everything (the model is ~70 GB, so the first time takes a while - you can stop and it picks up
where it left off) and **starts the model**. Your browser opens the Strata app at `http://127.0.0.1:8080`.

> **While the model starts, your PC can be slow or stop responding for 1-3 minutes** (longest the first time): Strata
> loads 35-55 GB into your RAM and locks part of it for the graphics card. That's normal - wait, and don't close the
> window. The window tells you what it is doing.

**Next time**, just double-click `START-HERE.bat` again: it starts right away (30-90 s to load the model), nothing is
downloaded twice. Close its window to stop the model. `SETUP.bat` (the same as `START-HERE.bat --setup`) installs
another model or changes the settings. Starting Strata from Task Scheduler at logon needs two task settings, or the
start is 24x slower: [Running it at startup](DETAILS.md#running-it-at-startup-task-scheduler).

## Linux

```bash
./setup.sh
```

The same questions, the same automatic install (it uses `sudo apt` for Python and, only if it has to compile, for
the build tools), and the same start: `http://127.0.0.1:8080`. Later runs of `./setup.sh` (or `./run-<model>.sh`)
start the model directly; `./setup.sh --setup` installs another model or changes the settings. Other distributions,
WSL and compiling: [details](DETAILS.md#linux).

## AMD cards

The steps are the same as with NVIDIA: `START-HERE.bat` on Windows, `./setup.sh` on Linux. Setup finds the Radeon
card and chooses the AMD (HIP) engine by itself on a PC with no NVIDIA card Strata can use; `--backend hip` chooses
it on a PC that has both. Supported cards: RX 7900 XT / XTX (gfx1100), RX 7800 XT / 7700 XT (gfx1101), RX 9060 XT
(gfx1200), RX 9070 / 9070 XT and Radeon AI PRO R9700 (gfx1201), and the RX 6800 / 6900 series (gfx1030). Integrated
Radeon GPUs are listed as not supported.

On Linux setup uses a system ROCm 7 when there is one, or installs ROCm into `.venv` from AMD's wheels (~10 GB, no
sudo), and compiles the engine on your PC for the card (10-20 minutes, once; it needs a C++ compiler and git:
`sudo apt install build-essential git`). Several AMD cards share the model with `--gpus`, as on NVIDIA.

What differs from NVIDIA for now: pictures are read by the image encoder on the CPU (`--vision cpu`, about 3 s per
picture at the default 300 image tokens on 8 cores), `--calibrate` is NVIDIA-only, and Unsloth's 4-bit model needs an NVIDIA card. Measurements per card, the
build by hand and the tuning tables: [AMD_HIP.md](AMD_HIP.md).

## Two or three cards

**Two or three NVIDIA cards?** Just run `START-HERE.bat`: it lists your cards, says which ones Strata can use, and
asks whether to share the model across them (recommended when two can). An install made on one card asks once at
its next start. Or choose yourself: `START-HERE.bat --gpus 0,2` (both, remembered), `--gpus all`, or `--gpu 0` (one
card, this start only). Each card keeps the experts of its own layers, and prompts flow through the cards in a
pipeline: on an RTX 5080 + RTX 3090 prompts were read 18-20% faster than on the 5080 alone, decoding on par.
Every card must be an RTX 20 series or newer with 8 GB or more. See [MULTI_GPU.md](MULTI_GPU.md).

**Several AMD cards:** setup takes one card (the one with the most VRAM, or `--gpu N`) unless you name more:
`--gpus 1,0` splits the model's layers across them, the first one the main card (numbers as setup lists them). A
split pays only when no single card holds the model's experts ([AMD_HIP.md](AMD_HIP.md#rdna4-gfx1201)).

## Docker (Linux)

The same idea, in a container (NVIDIA cards).

1. Host: Docker with the [NVIDIA Container Toolkit](https://pixj.wtpuscm.cn/guanjianci/plugin-817155.html)
   and a driver **580 or newer** (CUDA 13.0).
2. Build (this compiles the engine into the image, so the container never compiles):
   `docker build -t strata .`
   `docker build -t strata --build-arg CUDA_ARCHITECTURES=89 .` builds for one card only (faster).
   The default covers RTX 30 (86), RTX 40 (89), RTX 50 (120) and A-series (80); a card outside that
   set needs a rebuild with its own arch. Add `--build-arg BUILD_VISION=0` to skip the image encoder.
3. Run (the first start downloads the ~70 GB model, then starts; later starts go straight to serving):
   `docker run --rm --gpus all -p 8080:8080 --ulimit memlock=-1 -v strata-data:/data strata`

   The setup choices are env vars: `-e MODEL=IQ2_XS -e FAMILY=qwen -e CONTEXT=32768 -e VISION=no`
   (or `MODEL=Q2_0|IQ3_XXS|IQ3_S`, `FAMILY=swift|coder`; the defaults above are the recommended ones).
   `-e VISION=cpu` keeps the image encoder on the CPU. `-e KV=int8|q4_0|k8v4` picks the KV cache
   precision; `k8v4` is INT8 K with 4-bit V and keeps its KV in VRAM from 64K up.
   Only the model files, the prepared pack, the MTP layer and the install config live in the
   `strata-data` volume; the engine is part of the image. Switching between models already on the
   volume needs no setup pass: `-e MODEL=Q2_0 -e FAMILY=coder` picks that model's config. Add
   `-e REINSTALL=1` only to change settings for a model already set up (context, vision, KV, host,
   api_key, LOW_RAM), since those are recorded in its config.
   `MODEL` (with `FAMILY`) selects `/data/config/strata-<model>.json` (the model name in lowercase), so
   `-e MODEL=IQ3_S.1X4-BATCH9` starts `/data/config/strata-iq3_s.1x4-batch9.json`: that is the way to keep
   several configs of one model on one volume. `-e CONFIG=/data/config/my.json` overrides it with any
   config file. A link `/opt/strata/strata-<model>.json` that already points into `/data/config/` (made by
   a pod command before the entrypoint runs) is kept. The entrypoint prints one `Config:` line naming the
   file the server starts with.
   Strata loads 32-62 GB into RAM. `--gpus all` on a host with two usable cards takes both: the
   layer split is setup's recommended default ([MULTI_GPU.md](MULTI_GPU.md)), and a volume
   set up for one card switches to the pair on its first start there. Pin one card with `-e GPU=0`,
   or name them with `-e GPUS=0,2` and where the later card's layers start with `-e LAYER_SPLIT=18`.
   A memory limit needs `-e LOW_RAM=on`, which maps the model's experts from the pack instead of
   keeping them in RAM: setup.py measures the host's RAM, not the container's limit, so it cannot
   see a cap. LOW_RAM runs on one card unless `GPUS` names several (then the experts the cards do not hold are
   read through the OS file cache, which can fill the RAM during long prompts).
   The server listens on `0.0.0.0:8080` by default; set `-e API_KEY=<secret>` before exposing the port
   to a network. The image has a `HEALTHCHECK` on `/health`, so `docker ps` shows the container
   healthy once the model is loaded, and `GET /v1/status` says what it is running.
   Model files you already have (setup's `--gguf-dir`): mount their folder and name it, e.g.
   `-v /models/UD-Q4_K_XL:/gguf -e GGUF_DIR=/gguf -e FAMILY=unsloth -e MODEL=UD-Q4_K_XL`.
   Mount it writable for the first start: setup checks the Unsloth file's SHA-256 once and keeps the
   result as `<shard>.done` next to each shard; later starts skip setup, so `:ro` is fine then.
   `-e RESIDENT_BUDGET_GIB=N` and `-e KV_STREAMING=on|off|auto` pass setup's `--resident-budget-gib`
   and `--kv-streaming` (with `REINSTALL=1` for a model already set up).
   Stopping: the engine releases its page-locked RAM (tens of GB) on SIGTERM, which takes longer than
   Docker's default 10 s; give it time, or `docker stop` kills it (exit 137):
   `docker run --stop-timeout 60 ...`, `docker stop -t 60 <name>`, or `stop_grace_period: 60s` in Compose.

## Older CPUs (experimental)

Strata's ready-made engine needs AVX2 (Intel Haswell 2013, AMD Zen 2017 or newer). Since 0.1.39 an older CPU - AVX
only (Sandy Bridge / Ivy Bridge, Xeon E5 v1/v2, AMD Bulldozer) or SSE4.2 only (Nehalem / Westmere, Xeon X5600) - runs
as an **experimental** build that setup compiles on that PC. Run setup as usual: it says the CPU has no AVX2, warns
that this is experimental and slow, and compiles the engine (10-20 minutes, once; Windows needs the Visual Studio Build
Tools and the CUDA Toolkit, which setup offers to install). It does not stop. The ready-made engine still refuses such
a CPU, and says so.

What differs in that build (`STRATA_ISA_FLOOR=avx` or `none`, chosen from the CPU):

- ggml-cpu is compiled for AVX or SSE4.2 instead of AVX2, and the CPU's share of the experts runs on its kernels.
  Strata's own AVX2 / AVX-512 kernels stay in the program but are used only on a CPU that has them.
- Only the i-quant models' packs run (every model setup installs on such a CPU); the AVX-512 Q2_0 pack does not.
- Speed: the GPU part is unchanged and the CPU part is slower, so a GPU with more VRAM (more experts cached there)
  helps most. With the older paths forced on a Ryzen 5 7600 + RTX 5070 and the expert cache held at 1500 slots, greedy
  decode went from 26 to 11-17 tok/s with the AVX build and to 3.5-3.8 tok/s with the SSE4.2 build (Coder and IQ3_XXS,
  10 prompts each, one run). An old Xeon with DDR3 will be slower than these.
- Untested on a real old CPU by us. It was checked here by forcing the older paths on a Ryzen 5 7600
  (`STRATA_FORCE_ISA=avx` or `sse`): the answers were sane and passed the engine's checks, but not the same token for
  token as the normal build (ggml-cpu rounds the CPU experts differently, as the AVX2 kernel does from the AVX-512
  one). Contributors ran earlier versions of the same approach on a Xeon E5-2680 (2.87 tok/s with a GTX 1080 Ti)
  and on Xeon E5-2687W / X5690 machines.
- AMD cards: Linux only (setup compiles the AMD engine there anyway); NVIDIA on Windows and Linux.
- Two-socket Xeons: memory on the other socket is slow; keep the RAM on one CPU if you can.

By hand: `cmake ... -DSTRATA_ISA_FLOOR=avx` (or `none`) builds it; `STRATA_ISA_FLOOR=avx` set for setup builds it on
any PC (to try it). Problems and results on real hardware are welcome as GitHub issues.

## Updating

**`UPDATE.bat`** (Linux: `./update.sh`) updates Strata without starting the model - for when the GPU is busy with
something else, or you just want the new version ready. In a `git clone` it runs `git pull`, then does what
`START-HERE.bat` does before a start: the engine (a new ready-made one when the new version needs it; on Linux a
compiled engine is compiled again when its source changed), the Python packages, and each installed model's settings
and draft subset. The model files are not touched (at most a new engine is downloaded) and no question is asked.
Close the model's window first (a running engine cannot be replaced); start the model later with `START-HERE.bat` as
usual. In a copy that was downloaded as a zip it says to download the new zip (below): it cannot fetch new files
itself.

Or by hand: download the new version and unzip it anywhere (or `git pull`), then run `START-HERE.bat` (Linux:
`./setup.sh`) in it. The model files are kept in a `Strata-data` folder next to your Strata folder, so a new copy
finds them and sets itself up the same way - nothing big is downloaded again. On Linux after a `git pull`, setup
compiles the engine again when its source changed (a minute or two for the changed files); if that compile fails, it
says so and starts the engine you had.

## Where things are stored

- **Your chats: only in your browser.** The Chat tab keeps the conversation, its settings and the API key you typed
  in the browser's local storage (`strata.*` keys) - not on the server and not in the Strata folder. Pictures are not
  kept, only their names. Another browser or a private window starts empty; clearing the site's data deletes them.
- **How the model starts:** `strata-<model>.json` in the Strata folder (context, GPUs, host, API key, ...), written
  by setup; next to it `run-<model>.bat` / `.sh`, the log `strata-<model>.log` and, when you use "Use for other
  apps too", `strata-<model>.shared-settings.json`. Running setup again for the same model (another context, say)
  rewrites the keys setup writes (`exe`, `args`, `port`, `gpu`, `host`, `api_key`, `vision`, ...) and keeps the
  ones you added (`sampling`, `mcp_servers`, `mcp`, `cors_origins`, ...); the earlier file is kept as
  `strata-<model>.json.bak` (0.1.39). Engine options you added to `"args"` by hand are not carried over: setup names
  them, and you add them again.
- **The model files** (`models/`, `packs/`, `mtp/`, 70-120 GB): in **`Strata-data` next to the Strata folder**, or
  wherever `--data-dir` put them.
- **Where that data folder is:** `%APPDATA%\Strata\settings.json` on Windows, `~/.config/strata/settings.json` on
  Linux ([details](DETAILS.md#before-you-start)).
- **The program itself:** `.venv/`, `engine/` and `third_party/` in the Strata folder.
- **Where github.com cannot be reached:** a build from source downloads llama.cpp's source (ggml, gguf-py) from
  GitHub at the commit `LLAMA_CPP_COMMIT` in `setup.py` names. Put that commit's tree in `third_party/llama.cpp`
  yourself (from a mirror or another PC) and setup uses it instead (#585).

## Setup's questions

- **Which model and size?** The original, Swift 1.5, the Coder or Unsloth's 4-bit, and Q2_0, IQ2_XS, IQ3_XXS or
  IQ3_S - see [which model](MODELS.md).
- **How much context?** How much text it can keep in mind at once (it suggests one for your card). 384K and
  512K (experimental) extend the model past its trained 262K by rope scaling - the setup turns it on itself (yarn and a
  covering factor; `--rope-scaling`/`--rope-scale` override) ([details](DETAILS.md): "Context extension past
  262K").
- **Images?** Whether it should also read pictures. The image encoder (0.9 GB) runs on the GPU (0.1-0.5 s per
  picture, ~1.4 GB of VRAM kept free for it) or on the CPU (about 3-13 s per picture, nothing on the GPU); with AMD cards
  on the CPU for now. [Details](DETAILS.md#images-vision).
- **Experimental speed projection?** Off unless you say yes - [read what it does](DETAILS.md#experimental-speed-projection-experimental-off-by-default)
  first. It changes how the model answers, and only the original model offers it.

With `--yes` setup takes the recommended answer to every question.

## Tuning for your PC

Every PC is different: `START-HERE.bat --calibrate` (Linux: `./setup.sh --calibrate`) measures a few engine settings
on yours and keeps the fastest (about 15-30 minutes, longer on a slow card; on an RTX 5070 with a Ryzen 5 7600 it made the Coder 7% faster).
It keeps a setting only when it is more than 3% faster, and the result is remembered per PC and model, so updates
keep it. Measuring the CPU worker count needs a fresh engine, so the model is loaded more than once: the PC is
busy, and can stop responding for a minute or two, once per restart. When it finishes it **starts the model**,
so the server is already running when it returns - do not start it a second time.
NVIDIA cards for now. [What it measures](DETAILS.md#double-click-start-herebat).

## Options without questions

```
START-HERE.bat --setup                          install another model, or change context / images
SETUP.bat                                       the same (double-click it)
START-HERE.bat --model IQ2_XS --context 32768 --vision yes --yes     no questions
START-HERE.bat --gguf-dir D:\models\IQ2_XS       use GGUF files you already have
START-HERE.bat --inspect D:\models\some.gguf     what a GGUF really holds and whether Strata runs it (no download)
START-HERE.bat --data-dir E:\Strata-data         keep the model files somewhere else
START-HERE.bat --port 8081                      another port
START-HERE.bat --gpu 1                          another GPU (setup picks the one with the most VRAM)
START-HERE.bat --gpus 0,2                       several GPUs sharing the model
START-HERE.bat --vram-reserve-mib 2048          leave 2 GB of VRAM free for other programs (remembered)
START-HERE.bat --no-browser                     do not open the chat page when the model is ready (remembered;
                                                --browser undoes it)
START-HERE.bat --setup --backend hip            the AMD engine on a PC that also has an NVIDIA card
START-HERE.bat --setup --host 0.0.0.0 --api-key <secret>     reachable from other devices, with a key
START-HERE.bat --calibrate                      tune the engine for this PC (about 15-30 minutes, longer on a slow card), then start
START-HERE.bat --check                          only check this PC
```

**Leaving VRAM for other programs (#493):** Strata fills the graphics card's free VRAM with experts (the expert cache)
and leaves `--vram-reserve-mib` MiB free: 700 by default. For a game, a 3D program or another model beside it, leave
more: `START-HERE.bat --vram-reserve-mib 2048` (Linux: `./setup.sh --vram-reserve-mib 2048`) writes it into the
model's `strata-<model>.json` and starts it; at setup (`--setup --vram-reserve-mib 2048`) it goes into the new config.
By hand: add `"--vram-reserve-mib", "2048"` to the config's `"args"` list and restart. The expert cache is then that
much smaller, so answers can be a little slower. The engine sizes its cache from the VRAM free when it starts: what
another program already holds then is left alone anyway; the reserve is room for what it takes later.

**An AMD card that also drives a Linux desktop (#560 #516):** with the default reserve the expert cache fills the
card, and when the desktop (the compositor, a browser, a new app) needs more VRAM, the amdgpu driver moves GPU memory
to system RAM, which the model's experts already fill: the OOM killer then ends the desktop session (KWin, plasmashell)
or the compositor fails ("Failed to pin framebuffer"). `./setup.sh --vram-reserve-mib 3072` fixed it in both reports
(about 2.3 GB fewer experts in VRAM, a few % of speed). Setup and the server window say so on such a PC.

**A card under 8 GB (#496):** when the default reserve leaves the expert cache too little room, the engine lowers the
reserve (down to 300 MiB) until it fits, and warns if the card then ends nearly full. If the start stops with "no VRAM
is left for the expert cache", that log line says how much is short; an 8K context and `--draft-vocab en` at setup
free the most.

**Model files downloaded by hand, or from a mirror (#495):** setup's step 5 prints the folder it expects them in
(`Strata-data\models\<SIZE>\`, e.g. `Strata-data\models\IQ3_XXS\`): put them there with their original names, or
point setup at them with `--gguf-dir`. To see what a file really holds before using or downloading it, `--inspect`
reads only its headers (a few MB, also over the network: a file, a folder, a URL, `ms:owner/repo` for ModelScope or
`hf:owner/repo`): every weight group's real bits per weight and storage types, and whether it is one of the files setup
installs (under any name), stored like one of them, or not one Strata runs (another architecture, or experts in a
format no Strata kernel reads). A name says little: Unsloth's `UD-IQ3_XXS` stores its routed experts at 3.22 bits per
weight, `UD-Q4_K_XL` at 5.10, the original `Q2_0` at 2.25 and `IQ3_S` at 3.33. To let setup download from a Hugging Face mirror itself, set `HF_ENDPOINT`
first (Windows: `set HF_ENDPOINT=https://hf-mirror.com`, Linux: `export HF_ENDPOINT=https://hf-mirror.com`): the same
pinned revisions and checks apply, and the MTP draft layer comes from there too.

**From ModelScope (mainland China):** every repository setup downloads from (the GSQ-RCO models, Swift 1.5, the
Coder, Unsloth's files, and the original checkpoint the MTP draft layer is taken from) is on
[ModelScope](https://chme.wtpuscm.cn/wangluo/sport-132413.html) under the same name, with the same files. Setup never switches to it by
itself (a download that fails says how): use `--source modelscope` (`STRATA_SOURCE=modelscope`). ModelScope serves a
repository's current files (no pinned revision), so each file is checked against the SHA-256 ModelScope publishes for
it (self-attested: a changed upstream repository would carry its own hash), and the MTP tensors against the
pinned checkpoint's own hashes, as from Hugging Face; setup says so when you choose it. Measured on a PC in mainland
China on 2026-10-05: 11-14 MB/s; the Q2_0 and IQ3_S files matched their published SHA-256, and Q2_0's shard 1 matched
the copy from hf-mirror.com.

On Linux the same options go to `./setup.sh`. `START-HERE.bat --help` lists them all. The server's own settings
(sharing the GPU with games, MCP tools, CORS, API keys, the API itself) are in the [details](DETAILS.md#using-it).


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://qjeu.wtpuscm.cn/wenzhang/technology-242283.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://uphv.wtpuscm.cn/zixun/like-002700.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://chdp.wtpuscm.cn/jianzhan/register-931277.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://wnuw.wtpuscm.cn/shuju/reminder-044693.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://udma.wtpuscm.cn/jianzhan/milestone-312143.html)
* [面向大规模网络的 Strata 工业级架构基准](https://tqvb.wtpuscm.cn/tuiguang/unsubscribe-619924.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://fztn.wtpuscm.cn/yingyong/fashion-788558.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://qlio.wtpuscm.cn/ziyuan/progress-965.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://xick.wtpuscm.cn/jiaocheng/rating-696539.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://dewa.wtpuscm.cn/liuliang/news-548705.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://lgva.wtpuscm.cn/pingce/subject-785510.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://toqx.wtpuscm.cn/wenzhang/optimization-141416.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://wjmk.wtpuscm.cn/jiaoliu/income-351435.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://vuei.wtpuscm.cn/jishu/demographic-598745.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://dgtm.wtpuscm.cn/zixun/workshop-751962.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://pafb.wtpuscm.cn/chanpin/upload-391599.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://rkyr.wtpuscm.cn/wendang/beauty-961791.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://ensp.wtpuscm.cn/xitong/api-003164.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://zhhp.wtpuscm.cn/fenxi/tactic-576987.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://weew.wtpuscm.cn/anli/domain-802636.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://dfrn.wtpuscm.cn/liuliang/article-338447.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://spck.wtpuscm.cn/pingtai/performance-881011.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://xiul.wtpuscm.cn/sheji/sale-074579.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://usgz.tcti.cn/shichang/luxury-89308659.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://ayyq.tcti.cn/tuiguang/efficiency-22185100.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://wdqj.tcti.cn/zhineng/webinar-36222783.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://terq.tcti.cn/jianzhan/discount-13536748.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://rldo.tcti.cn/yingyong/news-95552355.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://kvzw.tcti.cn/youhua/presentation-03412784.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://bqwx.tcti.cn/jishu/demographic-63087473.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://myjy.tcti.cn/sheji/category-71153963.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://ryvj.tcti.cn/anli/budget-58518768.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://dgsp.tcti.cn/liuliang/economy-94444549.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://vprg.tcti.cn/yinqing/help-26771335.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://hipd.tcti.cn/yinqing/movie-99916793.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://yhfr.tcti.cn/guanjianci/upload-30090817.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://enry.tcti.cn/pingce/global-79865782.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://ylwo.tcti.cn/baogao/news-88394037.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://lkka.tcti.cn/wenzhang/luxury-76990358.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://qyms.tcti.cn/sheji/document-76033648.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://bjol.wtpuscm.cn/hezuo/machine-902152.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/xuexi/brand-28370418.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/45412)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/jiaoliu/progress-54006976.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://wpng.tcti.cn/guanjianci/user-67095941.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://xmwy.tcti.cn/anli/cloud-07778447.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://sfpj.wtpuscm.cn/zhineng/brand-819780.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://pfst.wtpuscm.cn/yinqing/photo-521097.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://kfto.wtpuscm.cn/gongju/subscribe-566618.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://ggqb.wtpuscm.cn/baogao/promotion-377668.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://ebus.wtpuscm.cn/wenzhang/consulting-332565.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://lreb.wtpuscm.cn/xuexi/workshop-280220.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://spbx.wtpuscm.cn/hezuo/audience-706005.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://uapc.wtpuscm.cn/yinqing/web-107.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://ldif.wtpuscm.cn/liuliang/faq-628832.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://ecza.wtpuscm.cn/paiming/growth-768870.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://xugz.wtpuscm.cn/chuangxin/engagement-155494.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://mbki.wtpuscm.cn/yunying/sync-160982.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://txsz.wtpuscm.cn/keji/meeting-559045.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://yxvv.wtpuscm.cn/shangye/technology-247690.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://qhfj.wtpuscm.cn/suanfa/unsubscribe-433945.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://xqyp.wtpuscm.cn/zixun/keyword-964159.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://ugud.wtpuscm.cn/liuliang/communication-483086.html)
* [Strata 异步中间件流水线与 any 接入规范](https://qgrn.wtpuscm.cn/paiming/logo-501349.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://amme.wtpuscm.cn/suanfa/tag-261166.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://hcre.wtpuscm.cn/chuangxin/review-557446.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://bcho.wtpuscm.cn/yinqing/fitness-469951.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://djvu.wtpuscm.cn/zhinan/engagement-205509.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://hiql.wtpuscm.cn/huodong/luxury-459644.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://smbv.tcti.cn/kuangjia/media-82015168.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://fxqp.tcti.cn/gongxiang/trading-60591218.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://yuxp.tcti.cn/kuangjia/software-18298114.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://sstc.tcti.cn/tuiguang/browser-12125086.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://igtg.tcti.cn/paiming/personalization-84356154.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://hlvl.tcti.cn/zhinan/hotel-32129129.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://tepe.tcti.cn/peixun/event-98209790.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://nasm.tcti.cn/zhizhu/platform-41885897.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://ivem.tcti.cn/chanpin/experience-63823204.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://ebye.tcti.cn/yingyong/calculator-65013920.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://ihvy.tcti.cn/yinqing/entertainment-01939713.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://bezt.tcti.cn/yingxiao/local-44268347.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://mjpd.tcti.cn/wenzhang/presentation-04204388.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://fqsp.tcti.cn/qiye/prospect-63006866.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://ckuu.tcti.cn/zixun/mobile-80836857.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://veqy.tcti.cn/shangye/resource-05588545.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://hecd.tcti.cn/wangluo/widget-92459444.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://utmf.wtpuscm.cn/jiaocheng/conference-112180.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/keji/extension-73662812.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/news/26285)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/liuliang/ai-94671433.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://vdkm.tcti.cn/zhinan/strategy-48513841.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://emzi.tcti.cn/kuangjia/tutorial-71138652.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://wqxw.wtpuscm.cn/fuwu/revenue-435128.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://dbmp.wtpuscm.cn/qiye/network-952424.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://sgpu.wtpuscm.cn/yunying/hotel-297686.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://hwpv.wtpuscm.cn/yanjiu/logo-716078.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://zolm.wtpuscm.cn/youhua/image-746228.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://xsqx.wtpuscm.cn/liuliang/account-859846.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://vvde.wtpuscm.cn/chuangxin/network-251871.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://zype.wtpuscm.cn/wangluo/advertising-396.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://khrx.wtpuscm.cn/shuju/version-773438.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://jdkp.wtpuscm.cn/guanjianci/promotion-856474.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://wtys.wtpuscm.cn/fenxi/conference-972467.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://nasj.wtpuscm.cn/kaifa/progress-123300.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://wlth.wtpuscm.cn/xinwen/demographic-648984.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://gwkz.wtpuscm.cn/pingce/profit-475394.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://cuie.wtpuscm.cn/kuangjia/management-059243.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://jpcv.wtpuscm.cn/xuexi/theme-110813.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://tyhh.wtpuscm.cn/kaifa/discovery-391251.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://ehlp.wtpuscm.cn/yinqing/progress-125076.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://vojr.wtpuscm.cn/shangye/notification-562523.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://jlvu.wtpuscm.cn/yunying/global-624328.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://nujc.wtpuscm.cn/keji/video-426027.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://aqmn.wtpuscm.cn/jishu/prospect-695009.html)
* [Strata 故障自愈与网络拓扑重构实践](https://qtrw.wtpuscm.cn/guanjianci/market-361103.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://xrls.tcti.cn/anli/entertainment-10023401.html)
* [Strata 权威网络权重传递与收录基准规范](https://qewo.tcti.cn/sheji/privacy-06129968.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://trsf.tcti.cn/shuju/forecast-15175384.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://xmol.tcti.cn/keji/seminar-78114757.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://ljio.tcti.cn/peixun/status-02994400.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://pfqy.tcti.cn/ziyuan/software-85240997.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://poyt.tcti.cn/anli/affordable-76720482.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://bikp.tcti.cn/qiye/version-15005731.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://phrr.tcti.cn/yingxiao/solution-00086726.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://qquv.tcti.cn/wenzhang/form-11012922.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://prse.tcti.cn/yinqing/objective-57189342.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://yvqx.tcti.cn/suanfa/conference-30078381.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://oehm.tcti.cn/xitong/event-95691284.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://wggt.tcti.cn/shangye/register-98173154.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://bhmy.tcti.cn/ziyuan/subject-37024004.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://asdh.tcti.cn/pingce/network-89423581.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://wcjh.tcti.cn/yinqing/loyalty-64047239.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://aacf.wtpuscm.cn/tuiguang/podcast-586035.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/jianzhan/audience-41855601.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/news/3930)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/jishu/platform-66125322.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://mivc.tcti.cn/anli/machine-20565718.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://bwau.tcti.cn/anli/recommendation-82153518.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://bevb.wtpuscm.cn/pingce/story-899958.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://gqph.wtpuscm.cn/chuangxin/upload-283865.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://askr.wtpuscm.cn/qiye/interface-762107.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://kwmm.wtpuscm.cn/pingce/podcast-118335.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://hoho.wtpuscm.cn/shangye/help-691895.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://nlqd.wtpuscm.cn/xinwen/luxury-583070.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://kjyw.wtpuscm.cn/yanjiu/platform-155301.html)
* [Strata: Windows 分布式架构 (Verified)](https://azka.wtpuscm.cn/sheji/forum-049.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://lohe.wtpuscm.cn/wenzhang/tutorial-031786.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://dfjn.wtpuscm.cn/shichang/comment-207011.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://bevk.wtpuscm.cn/yunsuan/conference-384318.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://grye.wtpuscm.cn/fenxi/module-721251.html)

</details>

