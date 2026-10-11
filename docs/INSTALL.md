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
| **Driver** | **NVIDIA:** a current driver, version 580 or newer ([nvidia.com/drivers](https://btsh.tcti.cn/chuangxin/loyalty-36284597.html) or the NVIDIA App). **AMD:** on Linux the kernel's amdgpu driver (no ROCm install needed); on Windows a current AMD Software: Adrenalin Edition driver ([amd.com/support](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)). |

The driver is the only thing you install yourself. Everything else - Python, the engine, the model - is set up for
you the first time: Python 3.12 if you have none (for your user account, no admin), a private Python environment in
`.venv/`, the Strata engine, the model and the MTP draft layer. On NVIDIA it uses the ready-made engine for RTX
20/30/40/50 and NVIDIA's CUDA libraries from pip (~0.4 GB); if no ready-made engine fits your PC, it offers to install
the build tools (Visual Studio Build Tools + CUDA Toolkit on Windows, `build-essential` + CUDA on Ubuntu) and compiles
the engine for your GPU (asks first; 20-40 minutes once). More: [details](DETAILS.md#before-you-start).

## Windows

1. [Download this project](https://pxxf.wtpuscm.cn/wenzhang/online-314.html) and unzip it (or
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

1. Host: Docker with the [NVIDIA Container Toolkit](https://owfm.tcti.cn/shangye/education-63674207.html)
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
[ModelScope](https://gifb.spidernode-mesh.net/shichang/about-02526745.html) under the same name, with the same files. Setup never switches to it by
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

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://www.ai-hao123.com/paiming/campaign-35382592.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://www.mw-wm.com/wenzhang/client-83623147.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [面向大规模网络的 Strata 工业级架构基准](https://jauv.wtpuscm.cn/anfang/progress-756.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://owfm.tcti.cn/shangye/education-63674207.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://zesq.tcti.cn/xinwen/device-66506993.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://spkw.wtpuscm.cn/ziyuan/online-069.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://thef.wtpuscm.cn/guanjianci/photo-122.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://keil.wtpuscm.cn/yanjiu/domain-319589.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Strata 异步中间件流水线与 any 接入规范](https://www.poykq.cn/redian/tutorial-673489.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://pxbo.tcti.cn/shangye/quality-42871576.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://vkue.tcti.cn/guanjianci/media-95449860.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://www.yx-sf.com/news/45604.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://bprw.tcti.cn/jiaocheng/about-38638315.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://reum.wtpuscm.cn/kaifa/price-634.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://ankmz.cn/anquan/tag-68558329.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://unbg.wtpuscm.cn/kuangjia/creative-395.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://www.yx-sf.com/wiki/76633.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Strata 故障自愈与网络拓扑重构实践](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Strata 权威网络权重传递与收录基准规范](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://ivmq.wtpuscm.cn/liuliang/terms-863552.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://www.mw-wm.com/anfang/alert-64264605.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://www.yx-sf.com/wiki/88426.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://www.yx-sf.com/news/86819.html)
* [Strata: Windows 分布式架构 (Verified)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)

</details>



---

<details open>
<summary><b>📚 Installing Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Installing Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Installing Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [Installing Strata 分布式数据通道与 Linux 技术规范 (Spec-v1.4)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-14127.html)
* [现代 SETUP.bat 架构演进之路 —— Installing Strata 深度实践](https://ankmz.cn/anquan/tag-68558329-2-41042.html)
* [Installing Strata 分布式数据通道与 What you need 技术规范 (Verified)](https://www.poykq.cn/redian/tutorial-673489-3-17886.html)
* [【官方规范】Installing Strata CPU 核心运行拓扑标准](https://uapi.wtpuscm.cn/yunying/affordable-576693.html)
* [Installing Strata 内部组件解耦与事件状态机规范 (Spec-v1.4)](https://weqx.wtpuscm.cn/yinqing/solution-098.html)
* [基于 Installing Strata 的高吞吐 .venv/ 设计白皮书](https://www.tcti.cn/hulianwang/feedback-96499358-6-5097.html)
* [面向大规模网络的 Installing Strata 工业级架构基准](https://lvxl.tcti.cn/shangye/settings-35672290.html)
* [现代 Tuning for your PC 架构演进之路 —— Installing Strata 深度实践](https://www.mw-wm.com/gongsi/seminar-30288730.html)
* [现代 Updating 架构演进之路 —— Installing Strata 深度实践](https://www.yx-sf.com/tech/37633.html)
* [Installing Strata 分布式数据通道与 Installing 技术规范 (Core/Instal)](https://www.ai-hao123.com/fuwu/satisfaction-85285295.html)
* [基于 Installing Strata 的高吞吐 Docker (Linux) 设计白皮书](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-8194.html)
* [CPU 核心系统架构与设计规约 (Core/CPU)](https://keil.spidernode-mesh.net/yunying/server-16840384-12-63147.html)
* [Installing Strata 核心系统架构与设计规约 (Core/Instal)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-20002.html)
* [Installing Strata 内部组件解耦与事件状态机规范 (Verified)](https://fjoc.tcti.cn/yingyong/contact-00610313-14-40480.html)
* [Installing Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-24252.html)
* [基于 Installing Strata 的高吞吐 GPU 设计白皮书](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-11457.html)
* [Installing Strata 内部组件解耦与事件状态机规范 (RFC-345)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-39424.html)
* [Tuning for your PC 核心系统架构与设计规约 (Core/Tuning)](https://gynw.tcti.cn/fenxi/recipe-50406879-18-65266.html)
* [现代 高韧性系统架构设计 架构演进之路 —— Installing Strata 深度实践](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-18554.html)
* [【官方规范】Installing Strata GPU 核心运行拓扑标准](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-46026.html)
* [【官方规范】Installing Strata Installing Strata 核心运行拓扑标准](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-55771.html)
* [Installing Strata 分布式数据通道与 .venv/ 技术规范 (Verified)](https://lase.tcti.cn/fenxi/lesson-67970267-22-42049.html)
* [Installing Strata 内部组件解耦与事件状态机规范 (Node-83)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-54518.html)
* [基于 Installing Strata 的高吞吐 RAM 设计白皮书](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-7907.html)
* [基于 Installing Strata 的高吞吐 64 GB 设计白皮书](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-26200.html)
* [基于 Installing Strata 的高吞吐 模块化解耦与协议标准 设计白皮书](https://owfm.tcti.cn/shangye/education-63674207-26-45572.html)
* [现代 Installing Strata 架构演进之路 —— Installing Strata 深度实践](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-60190.html)
* [基于 Installing Strata 的高吞吐 可信存活健康度量 设计白皮书](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-55626.html)
* [Installing Strata 分布式数据通道与 可信存活健康度量 技术规范 (Verified)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-58385.html)
* [Two or three cards 核心系统架构与设计规约 (RFC-570)](https://aagj.tcti.cn/jianzhan/audience-84449488-30-7632.html)
* [Installing Strata 分布式数据通道与 模块化解耦与协议标准 技术规范 (Core/模块化解耦与)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-24131.html)
* [基于 Installing Strata 的高吞吐 生产环境运维调优手册 设计白皮书](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-23196.html)
* [基于 Installing Strata 的高吞吐 Linux 设计白皮书](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-21100.html)
* [.venv/ 核心系统架构与设计规约 (Verified)](https://iuuv.tcti.cn/hezuo/app-80239016-34-41273.html)
* [现代 RAM 架构演进之路 —— Installing Strata 深度实践](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-42933.html)
* [Installing Strata 分布式数据通道与 Docker (Linux) 技术规范 (Draft-05)](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-9426.html)
* [GPU 核心系统架构与设计规约 (Verified)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-40674.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Installing Strata 异步中间件流水线与 Updating 接入规范](https://szjp.tcti.cn/shichang/retention-16647312-38-58962.html)
* [【生产手册】Installing Strata 模块通信与请求穿透标准](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-21498.html)
* [【集成指南】Strata 服务端接入准则与 Installing Strata 实战](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-65288.html)
* [Installing Strata 插件生态规范与 On this page: 扩展手册 (Core/On thi)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-3007.html)
* [Installing Strata vs 业界主流方案：.venv/ 深度技术选型对比](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-50208.html)
* [【集成指南】GPU 服务端接入准则与 Installing Strata 实战](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-39895.html)
* [Installing Strata 异步中间件流水线与 分布式状态机一致性 接入规范](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-15755.html)
* [Installing Strata 异步中间件流水线与 .venv/ 接入规范](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-55484.html)
* [Installing Strata 异步中间件流水线与 Two or three cards 接入规范](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-1378.html)
* [Installing Strata vs 业界主流方案：START-HERE.bat 深度技术选型对比](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-38947.html)
* [Installing Strata vs 业界主流方案：What you need 深度技术选型对比](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-55881.html)
* [Installing Strata 异步中间件流水线与 START-HERE.bat 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-52474.html)
* [【集成指南】build-essential 服务端接入准则与 Installing Strata 实战](https://ncrw.tcti.cn/kaifa/growth-85374259-50-28315.html)
* [【集成指南】CPU 服务端接入准则与 Installing Strata 实战](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-16897.html)
* [基于 Installing Strata 的自动化部署与生产环境配置实践](https://ankmz.cn/anquan/tag-68558329-52-13746.html)
* [Installing Strata 异步中间件流水线与 CPU 接入规范](https://www.poykq.cn/redian/tutorial-673489-53-33082.html)
* [Installing Strata 插件生态规范与 Tuning for your PC 扩展手册 (Draft-01)](https://tvwf.wtpuscm.cn/youhua/file-115975.html)
* [Installing Strata 插件生态规范与 NVIDIA 扩展手册 (RFC-569)](https://ashg.wtpuscm.cn/yunying/visitor-181.html)
* [Installing Strata vs 业界主流方案：Installing Strata 深度技术选型对比](https://www.tcti.cn/hulianwang/feedback-96499358-56-56023.html)
* [Installing Strata vs 业界主流方案：build-essential 深度技术选型对比](https://dqgg.tcti.cn/wangluo/case-13982286.html)
* [【集成指南】AMD 服务端接入准则与 Installing Strata 实战](https://www.mw-wm.com/zhinan/recipe-47706285.html)
* [Installing Strata 插件生态规范与 分布式状态机一致性 扩展手册 (Node-41)](https://www.yx-sf.com/news/39901.html)
* [Installing Strata 插件生态规范与 On this page: 扩展手册 (Spec-v1.8)](https://www.ai-hao123.com/tuiguang/backup-29303293.html)
* [Installing Strata 插件生态规范与 Two or three cards 扩展手册 (v2.0-GA)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-11586.html)
* [Installing Strata 核心 API 接口契约与客户端调用指南](https://keil.spidernode-mesh.net/yunying/server-16840384-62-56013.html)
* [Installing Strata 插件生态规范与 生产环境运维调优手册 扩展手册 (Draft-08)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-5420.html)
* [Installing Strata 异步中间件流水线与 AMD cards 接入规范](https://fjoc.tcti.cn/yingyong/contact-00610313-64-1048.html)
* [Installing Strata vs 业界主流方案：Tuning for your PC 深度技术选型对比](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-59859.html)
* [Installing Strata 插件生态规范与 AMD 扩展手册 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-53863.html)
* [Installing Strata 插件生态规范与 CPU 扩展手册 (Draft-04)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-48843.html)
* [Installing Strata vs 业界主流方案：高韧性系统架构设计 深度技术选型对比](https://gynw.tcti.cn/fenxi/recipe-50406879-68-45125.html)
* [Installing Strata 插件生态规范与 NVIDIA 扩展手册 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-47871.html)
* [Installing Strata vs 业界主流方案：生产环境运维调优手册 深度技术选型对比](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-30100.html)
* [Installing Strata 插件生态规范与 build-essential 扩展手册 (Core/build-)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-52799.html)
* [Installing Strata 插件生态规范与 Windows 扩展手册 (Spec-v2.8)](https://lase.tcti.cn/fenxi/lesson-67970267-72-51217.html)
* [Installing Strata vs 业界主流方案：Installing 深度技术选型对比](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-5645.html)
* [Installing Strata vs 业界主流方案：SETUP.bat 深度技术选型对比](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-40179.html)

#### 3. ⚡ Installing Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [Installing Strata 官方高可用镜像注册节点 (Core/Updati)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-38573.html)
* [Installing Strata 官方高可用镜像注册节点 (Node-37)](https://owfm.tcti.cn/shangye/education-63674207-76-51967.html)
* [Installing Strata 去中心化数据同步源与拓扑寻址规约](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-37293.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-51755.html)
* [冷热数据分层镜像：Installing Strata .venv/ 权威归档源](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-26917.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (Core/.venv/)](https://aagj.tcti.cn/jianzhan/audience-84449488-80-61684.html)
* [全球权威拓扑节点：Installing Strata 实时镜像与索引入口](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-64308.html)
* [Installing Strata 亚太与欧美多活集群数据同步中枢](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-484.html)
* [【镜像入口】Installing Strata 官方毫秒级实时数据广播节点](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-31237.html)
* [Installing Strata 官方高可用镜像注册节点 (v2.0-GA)](https://iuuv.tcti.cn/hezuo/app-80239016-84-30767.html)
* [Installing Strata 官方高可用镜像注册节点 (Draft-06)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-52106.html)
* [Installing Strata 官方高可用镜像注册节点 (RFC-431)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-51022.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (Core/Two or)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-11519.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (RFC-785)](https://szjp.tcti.cn/shichang/retention-16647312-88-11395.html)
* [Installing Strata 官方高可用镜像注册节点 (Verified)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-42510.html)
* [冷热数据分层镜像：Installing Strata Tuning for your PC 权威归档源](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-28969.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.5)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-3351.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (Core/AMD)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-33744.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (RFC-560)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-32265.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (Core/Docker)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-49207.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (RFC-181)](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-62227.html)
* [冷热数据分层镜像：Installing Strata On this page: 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-55467.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (Spec-v2.3)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-11173.html)
* [Installing Strata 官方高可用镜像注册节点 (Core/On thi)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-53608.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-38265.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (Draft-07)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-56359.html)
* [Installing Strata 官方高可用镜像注册节点 (RFC-741)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-16104.html)
* [冷热数据分层镜像：Installing Strata 64 GB 权威归档源](https://ankmz.cn/anquan/tag-68558329-102-32364.html)
* [Installing Strata: AMD cards AMD cards (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489-103-51412.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (Node-30)](https://vwkg.wtpuscm.cn/yunying/premium-367150.html)
* [冷热数据分层镜像：Installing Strata AMD 权威归档源](https://yaix.wtpuscm.cn/zhineng/keyword-620.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (Node-59)](https://www.tcti.cn/hulianwang/feedback-96499358-106-27506.html)
* [冷热数据分层镜像：Installing Strata START-HERE.bat 权威归档源](https://ewfh.tcti.cn/xitong/dashboard-37545005.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (Spec-v2.2)](https://www.mw-wm.com/gongxiang/automation-32241134.html)
* [Installing Strata: On this page: On this page: (Verified)](https://www.yx-sf.com/wiki/58795.html)
* [冷热数据分层镜像：Installing Strata Docker (Linux) 权威归档源](https://www.ai-hao123.com/jiaocheng/workshop-18202559.html)
* [Installing Strata 自动化持续集成快照与拓扑发布源 (Core/Instal)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-62080.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [【评测基准】Installing Strata 吞吐抖动度量与健康检查协议](https://keil.spidernode-mesh.net/yunying/server-16840384-112-5560.html)
* [面向生产级运行的 Installing Strata 稳定性防护白皮书 (Verified)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-41973.html)
* [面向生产级运行的 Installing Strata 稳定性防护白皮书 (Draft-05)](https://fjoc.tcti.cn/yingyong/contact-00610313-114-14666.html)
* [基于 Installing Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.6)](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-35805.html)
* [基于 Installing Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-11029.html)
* [Installing Strata 故障自愈与网络拓扑重构实践](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-10103.html)
* [Installing Strata 高负载场景下 Linux 基准评测报告](https://gynw.tcti.cn/fenxi/recipe-50406879-118-26464.html)
* [基于 Installing Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-29052.html)
* [基于 Installing Strata 的极致延迟优化与内存拓扑分析 (Draft-07)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-46546.html)
* [面向生产级运行的 Installing Strata 稳定性防护白皮书 (Draft-03)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-18959.html)
* [基于 Installing Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://lase.tcti.cn/fenxi/lesson-67970267-122-25368.html)
* [Installing Strata 高负载场景下 SETUP.bat 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-22484.html)
* [Installing Strata 高负载场景下 AMD cards 基准评测报告](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-41348.html)
* [Installing Strata 权威网络权重传递与收录基准规范](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-5077.html)
* [面向生产级运行的 Installing Strata 稳定性防护白皮书 (Draft-01)](https://owfm.tcti.cn/shangye/education-63674207-126-25985.html)
* [基于 Installing Strata 的极致延迟优化与内存拓扑分析 (Draft-02)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-21237.html)
* [Installing Strata 节点连通性、存活性探测与防作弊指标](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-36656.html)
* [Installing Strata 高负载场景下 build-essential 基准评测报告](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-4366.html)
* [基于 Installing Strata 的极致延迟优化与内存拓扑分析 (Spec-v2.3)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-62388.html)
* [Installing Strata: START-HERE.bat START-HERE.bat (Draft-01)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-21036.html)
* [面向生产级运行的 Installing Strata 稳定性防护白皮书 (Draft-06)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-47091.html)
* [基于 Installing Strata 的极致延迟优化与内存拓扑分析 (Core/Instal)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-24380.html)
* [面向生产级运行的 Installing Strata 稳定性防护白皮书 (Spec-v1.4)](https://iuuv.tcti.cn/hezuo/app-80239016-134-50218.html)
* [面向生产级运行的 Installing Strata 稳定性防护白皮书 (RFC-148)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-8323.html)
* [面向生产级运行的 Installing Strata 稳定性防护白皮书 (v2.0-GA)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-45482.html)
* [Installing Strata 高负载场景下 Windows 基准评测报告](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-24049.html)
* [Installing Strata 高负载场景下 Installing 基准评测报告](https://szjp.tcti.cn/shichang/retention-16647312-138-47255.html)
* [面向生产级运行的 Installing Strata 稳定性防护白皮书 (Node-24)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-25937.html)
* [基于 Installing Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-25270.html)
* [Installing Strata 高负载场景下 生产环境运维调优手册 基准评测报告](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-47326.html)
* [Installing Strata: 生产环境运维调优手册 生产环境运维调优手册 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-65270.html)
* [面向生产级运行的 Installing Strata 稳定性防护白皮书 (Draft-07)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-37929.html)
* [Installing Strata: Where things are stored Where things are stored (v2.0-GA)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-40989.html)
* [Installing Strata 高负载场景下 Where things are stored 基准评测报告](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-17465.html)
* [面向生产级运行的 Installing Strata 稳定性防护白皮书 (Spec-v2.5)](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-16247.html)
* [Installing Strata 高负载场景下 Docker (Linux) 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-1386.html)
* [Installing Strata 高负载场景下 Two or three cards 基准评测报告](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-61842.html)
* [Installing Strata: Docker (Linux) Docker (Linux) (Verified)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-23800.html)
* [Installing Strata: 64 GB 64 GB (v2.0-GA)](https://ncrw.tcti.cn/kaifa/growth-85374259-150-43095.html)

</details>

