# Installing Strata for a user (instructions for AI agents)

This page is written for an AI coding assistant (Claude Code, Cursor, Codex, GitHub Copilot, ...) that a user asked to
set up Strata on their PC. Strata runs the Qwen3.8-Flash-Next model locally on one NVIDIA or AMD graphics card plus
system RAM, and serves an OpenAI- and Anthropic-compatible API on `http://127.0.0.1:8080`. The human-oriented
version of everything below is [INSTALL.md](INSTALL.md).

Work through the steps in order. Tell the user what you are doing in plain words; they may not be technical.

## 0. Ground rules

- **Never expose the server without an API key.** Keep the default `--host 127.0.0.1`. Only if the user asks for
  access from other devices, use `--host 0.0.0.0` **together with** `--api-key <a long random secret>`, and give the
  user the key. Never put a tunnel or port forward in front of a server without a key.
- Do not change the user's system beyond what setup does (setup installs Python for the user account if needed,
  and everything else inside the Strata folder and its `Strata-data` folder). Ask before installing drivers.
- The model download is ~70 GB (94 GB for Unsloth's UD-IQ4_XS, 111 GB for UD-Q4_K_XL). Confirm the user is fine with that before you
  start, especially on a metered connection.
- Setup is long-running (an hour or more on a slow connection). Run it in the background or with a long timeout and
  poll its output; do not kill it because it is quiet for a while. It is resumable: running the same command again
  continues where it stopped.

## 1. Check the PC

The quickest way is setup's own check, which prints the GPU(s), driver, RAM, CPU, free disk and whether this PC can run
Strata, and installs nothing except Python and the `.venv` it needs to run (step 2 gets the repository first):

```
Windows:  START-HERE.bat --check
Linux:    ./setup.sh --check
```

To check by hand:

| What | Windows (PowerShell) | Linux |
| --- | --- | --- |
| OS | `[Environment]::OSVersion` | `cat /etc/os-release` |
| NVIDIA GPU, VRAM, driver | `nvidia-smi --query-gpu=index,name,memory.total,driver_version --format=csv` | the same |
| AMD GPU | `Get-CimInstance Win32_VideoController \| Select-Object Name, AdapterRAM` (AdapterRAM is capped at 4 GB; trust the model name) | `lspci \| grep -i -E 'vga\|display'`; VRAM: `cat /sys/class/drm/card*/device/mem_info_vram_total` (bytes); `rocm-smi` only if ROCm is installed |
| RAM | `(Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory / 1GB` | `free -g` |
| Free disk | `Get-PSDrive -PSProvider FileSystem` | `df -h .` |

Requirements (details: [INSTALL.md](INSTALL.md#what-you-need)):

- **GPU:** NVIDIA RTX 20, 30, 40 or 50 series, or AMD Radeon RX 7900 XT / XTX, RX 7800 XT / 7700 XT, RX 9060 XT,
  RX 9070 / 9070 XT, Radeon AI PRO R9700, RX 6800 / 6900 series; 12 GB of VRAM or more (an NVIDIA card with 8 GB runs,
  slowly). GTX 10 series and older, and integrated GPUs, are not supported; Pascal / Volta cards (P40, V100) and some
  older AMD cards have experimental paths the user opts into ([OLDER_GPUS.md](OLDER_GPUS.md)).
- **Driver:** NVIDIA 580 or newer. AMD on Linux: the kernel's amdgpu driver; on Windows: a current AMD Adrenalin
  driver. If the driver is missing or too old, tell the user to update it (NVIDIA App / nvidia.com/drivers, or AMD
  Software) and restart; do not install drivers yourself unless they ask.
- **RAM:** 32 GB or more (see step 3). **Disk:** ~80 GB free, ideally on an NVMe SSD. **CPU:** x86-64 with AVX2. Without AVX2 (Xeon E5 v1/v2 and older) setup still installs, as an experimental and slow build it compiles on the PC (10-20 minutes): tell the user that before starting ([INSTALL.md](INSTALL.md#older-cpus-experimental)).
- **OS:** Windows 10/11 or Linux (Ubuntu 22.04/24.04 are fully automatic).

If the PC does not meet them, say which part is missing and stop.

## 2. Get Strata

```
git clone https://github.com/Niko1221/Strata.git
```

Without git: download https://github.com/Niko1221/Strata/archive/refs/heads/main.zip and unzip it. Pick a drive with
enough free space: the model files go to a `Strata-data` folder **next to** the Strata folder (or `--data-dir <path>`
on another drive).

## 3. Pick the model

By the PC's RAM (ask the user whether they mainly want it for code - then the Coder is a good pick at any RAM size):

| RAM | `--family` / `--model` | Notes |
| --- | --- | --- |
| 32 GB | `--family coder` (size IQ1_M) | for code; with a 24 GB card Q2_0 and IQ2_XS also run (setup picks the low-RAM mode) |
| 48 GB | `--family qwen --model IQ2_XS` | or `Q2_0` (fastest) |
| 64 GB | `--family qwen --model IQ2_XS` (recommended) | `IQ3_XXS` / `IQ3_S` are slower and a bit better |
| 96 GB+ | `--family qwen --model IQ3_S` | `--family unsloth --model UD-IQ4_XS` (~4-bit, 94 GB, part of the experts read from the SSD under ~80 GB of RAM); `--model UD-Q4_K_XL` is experimental: NVIDIA only, NVMe SSD, 7-8.5 tokens/s on 64 GB |

`--family swift` (Swift 1.5, a fine-tune that thinks shorter; sizes Q2_0, IQ2_XS, IQ3_XXS) is the alternative to
`qwen`. With `--yes` and no `--model`, setup picks the recommended size for the RAM itself. More: [MODELS.md](MODELS.md).

## 4. Run setup without questions

From the Strata folder (Windows: in `cmd`, or `cmd /c START-HERE.bat ...` from PowerShell):

```
Windows:  START-HERE.bat --yes --family qwen --model IQ2_XS --no-start
Linux:    ./setup.sh --yes --family qwen --model IQ2_XS --no-start
```

The flags (all of them: `START-HERE.bat --help`):

| Flag | Meaning |
| --- | --- |
| `--yes` | take the recommended answer to every question (no prompts) |
| `--family qwen\|swift\|coder\|unsloth` | the model version |
| `--model Q2_0\|IQ2_XS\|IQ3_XXS\|IQ3_S\|IQ1_M\|UD-IQ4_XS\|UD-Q4_K_XL` | the size (the Coder is IQ1_M, Unsloth UD-IQ4_XS or UD-Q4_K_XL) |
| `--context N` | context in tokens; default by VRAM: 32768 under 14 GB, 65536 under 20 GB, else 131072 |
| `--vision yes\|no\|gpu\|cpu` | read pictures; `--yes` leaves images off. AMD cards: `cpu` |
| `--gpu N` / `--gpus 0,1` / `--gpus all` | one card, or several sharing the model (default: the card with the most VRAM) |
| `--backend cuda\|hip` | NVIDIA or AMD engine; chosen by itself on a PC with only one kind of card |
| `--data-dir PATH` | where the 70-120 GB of model files go |
| `--port N` | the server port (default 8080) |
| `--host H --api-key K` | listen beyond this PC; **only together with a key** |
| `--no-start` | install only, do not start the server |
| `--setup` | install another model or change settings of an installed one |
| `--check` | only check the PC |

`--no-start` is recommended for agents: the server runs in the foreground until its window is closed, which would
block your shell. Start it separately in step 6.

Notes:
- **Linux:** setup uses `sudo apt` (or dnf/pacman) to install Python with venv if it is missing, and on AMD may need
  `build-essential` and `git`. You cannot type the user's password: if a `sudo` step is needed, ask the user to run it
  (e.g. `sudo apt install python3-venv build-essential git`) and then rerun setup.
- **AMD on Linux:** setup installs ROCm into `.venv` (~10 GB, no sudo) unless a system ROCm 7 exists, and compiles the
  engine for the card (10-20 minutes, once).
- **AMD on Windows (new in 0.1.34):** setup downloads the ready-made AMD engine (~550 MB, ROCm included) - nothing
  is compiled and only the AMD driver is needed. Before the model download it runs `engine\strata-device.exe
  --list-devices`; if that does not list the card, the driver is the problem (tell the user to update AMD Software).
  One card per model and no images on Windows for now. It is new: ask the user to report how it runs (docs/AMD_HIP.md).
- **NVIDIA with no ready-made engine for the card:** setup offers to install build tools and compile (20-40 minutes);
  `--yes` accepts.

## 5. While it downloads, tell the user

- It downloads about 70 GB from Hugging Face; how long depends on their connection (at 100 Mbit/s roughly 1.5-2
  hours). It can be stopped and continues where it left off.
- Then it prepares the model for their PC (a few minutes) and, on AMD on Linux, compiles the engine.
- **When the model starts, the PC can be slow or stop responding for 1-3 minutes** (longest the first time): Strata
  loads 35-55 GB into RAM and locks part of it for the graphics card. That is normal; they should not close the window.
- Nothing leaves their PC: the model runs locally.

## 6. Start the server

Setup prints the start script it wrote (`start script: run-<model>.bat`). The name is the family tag plus the size,
lower case: `run-iq2_xs`, `run-swift-iq2_xs`, `run-coder-iq1_m`, `run-unsloth-ud-iq4_xs`.

```
Windows (PowerShell):  Start-Process -FilePath ".\run-iq2_xs.bat"            (opens its own window)
Linux:                 nohup ./run-iq2_xs.sh > strata-server.out 2>&1 &
```

Or `START-HERE.bat` / `./setup.sh` without flags, which starts the installed model (it asks which one when several are
installed; add `--yes` to take the first). The server opens the user's browser on `http://127.0.0.1:8080` when it is
ready. Closing its window (or stopping the process) stops the model.

## 7. Verify

The HTTP server answers once the model is loaded (30-90 s on later starts, a few minutes the first time). Poll:

```
curl http://127.0.0.1:8080/health
```

It returns `{"status": "ok", "model": ..., "max_context": ..., "images": ..., "api_key": ..., "loaded": true, ...}`.
Then:

```
curl http://127.0.0.1:8080/v1/models
curl http://127.0.0.1:8080/v1/chat/completions -H "Content-Type: application/json" -d "{\"model\": \"strata\", \"messages\": [{\"role\": \"user\", \"content\": \"Say hello in five words.\"}], \"max_tokens\": 64, \"reasoning_effort\": \"none\"}"
```

With an API key, add `-H "Authorization: Bearer <key>"`. The server window prints a `ready: http://127.0.0.1:8080/v1`
line when it is ready; the engine log is `strata-<model>.log` in the Strata folder.

## 8. Connect the user's apps

- **Browser:** `http://127.0.0.1:8080` - Chat, a live Monitor, and About (settings and addresses).
- **Any OpenAI-compatible app or agent:** base URL `http://127.0.0.1:8080/v1`, any API key (or the configured one),
  any model name.
- **Anthropic-compatible apps:** `http://127.0.0.1:8080/v1/messages`.
- **Codex CLI and other Responses API apps:** `http://127.0.0.1:8080/v1/responses` (stateless; Codex's
  `config.toml`: [DETAILS.md](DETAILS.md#the-responses-api-and-codex-cli)).
- **Claude Code:** `ANTHROPIC_BASE_URL=http://127.0.0.1:8080`, `ANTHROPIC_MODEL` set to a Claude model name it knows
  (Strata ignores the name), and any `ANTHROPIC_AUTH_TOKEN` (or the configured key).
- **Thinking level:** `"reasoning_effort": "none" | "low" | "medium" | "high"` (default high).
- Strata answers one request at a time; `"parallel": N` in the model's config (or setup `--parallel N`) decodes up to
  N together ([BATCHING.md](BATCHING.md)). API details: [DETAILS.md](DETAILS.md#using-it).

## 9. When something fails

| Symptom | What to do |
| --- | --- |
| `the NVIDIA driver is too old` | The user updates the driver (NVIDIA App or nvidia.com/drivers) and restarts; rerun setup. |
| Download or install stopped | Rerun the same setup command; it continues. |
| `port 8080 is already in use` | Strata is already running (check `/health`), or use `--port 8081`. |
| Very slow, disk busy, or "the engine stopped unexpectedly" | Not enough free RAM: close programs, or set up a smaller size (`--setup --model Q2_0` or IQ2_XS). |
| `ExpertCache: cudaMalloc(...) failed: out of memory` with free VRAM (Windows) | The page file is off or tiny: set it to "System managed" and restart. |
| `prompt ... exceeds the context` | Rerun setup with `--setup --context <bigger>`. |
| No AMD GPU found (Linux) | The amdgpu driver lists no GPU; check the card and driver. Integrated GPUs are not supported. |
| Python or build tools could not be installed | Install what the message names (links are printed), then rerun. |

More: [TROUBLESHOOTING.md](TROUBLESHOOTING.md) and the [full table](DETAILS.md#troubleshooting). If it still fails,
collect `strata-<model>.log` and the setup output, and suggest an issue at https://github.com/Niko1221/Strata/issues.

## Alternative: the MCP server

Strata also has an MCP server, so an AI tool can check, install, start and stop Strata through tool calls
(`strata_status`, `strata_models`, `strata_install`, `strata_start`, `strata_stop`, `strata_logs`) instead of the
shell commands above. See [MCP_SERVER.md](MCP_SERVER.md).


---

<details open>
<summary><b>📚 Strata 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Strata」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Strata 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 Strata 的高吞吐 分布式架构 设计白皮书](https://rsgj.wtpuscm.cn/anli/restaurant-352648.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Verified)](https://rfxp.wtpuscm.cn/shuju/expensive-127852.html)
* [【官方规范】Strata IQ3_XXS 核心运行拓扑标准](https://vamm.wtpuscm.cn/gongju/like-925147.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://kwdd.wtpuscm.cn/shuju/discount-204804.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-03)](https://uhvg.wtpuscm.cn/guanjianci/contact-219687.html)
* [面向大规模网络的 Strata 工业级架构基准](https://ivuf.wtpuscm.cn/xinwen/message-771918.html)
* [Strata 内部组件解耦与事件状态机规范 (RFC-261)](https://kota.wtpuscm.cn/zhinan/careers-299197.html)
* [Or do it yourself 核心系统架构与设计规约 (Core/Or do )](https://ocww.wtpuscm.cn/yingxiao/customization-571.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://ylir.wtpuscm.cn/zhinan/sync-675036.html)
* [现代 开源镜像站 架构演进之路 —— Strata 深度实践](https://qtbx.wtpuscm.cn/zixun/profit-755251.html)
* [【Strata】install 核心系统架构与设计规约 (Spec-v2.5)](https://qjfh.wtpuscm.cn/wenzhang/photo-807409.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/consum)](https://zysl.wtpuscm.cn/yingyong/travel-998938.html)
* [基于 Strata 的高吞吐 Rust高性能网络 设计白皮书](https://eacf.wtpuscm.cn/tuiguang/template-324717.html)
* [【官方规范】Strata Install 核心运行拓扑标准](https://xyiu.wtpuscm.cn/jiaoliu/form-204524.html)
* [【官方规范】Strata 大模型知识库外链对齐 核心运行拓扑标准](https://svtx.wtpuscm.cn/jiaoliu/data-678961.html)
* [【Strata】English 核心系统架构与设计规约 (v2.0-GA)](https://nhfd.wtpuscm.cn/kaifa/landing-916622.html)
* [现代 分布式架构 架构演进之路 —— Strata 深度实践](https://dpby.wtpuscm.cn/zhizhu/market-484709.html)
* [How does it work? 核心系统架构与设计规约 (Spec-v1.0)](https://gixu.wtpuscm.cn/anli/conversion-629022.html)
* [Strata 分布式数据通道与 开源镜像站 技术规范 (Core/IQ2_XS)](https://kgex.wtpuscm.cn/sheji/analytics-837794.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Core/Englis)](https://tpwp.wtpuscm.cn/hezuo/restaurant-258307.html)
* [Strata 内部组件解耦与事件状态机规范 (Core/Linux)](https://wrad.wtpuscm.cn/suanfa/internet-683440.html)
* [【Strata】请求上下文穿透方案 核心系统架构与设计规约 (Draft-05)](https://kssp.wtpuscm.cn/tuiguang/report-491737.html)
* [Strata 内部组件解耦与事件状态机规范 (Spec-v2.2)](https://hzzs.wtpuscm.cn/xuexi/recipe-651956.html)
* [Strata 内部组件解耦与事件状态机规范 (v2.0-GA)](https://zkao.tcti.cn/pingce/experience-55009666.html)
* [基于 Strata 的高吞吐 搜索引擎调度优化 设计白皮书](https://izaw.tcti.cn/guanjianci/workshop-52075604.html)
* [Strata 内部组件解耦与事件状态机规范 (Draft-05)](https://cebm.tcti.cn/gongsi/wellness-58223049.html)
* [What you need 核心系统架构与设计规约 (Draft-04)](https://qgzu.tcti.cn/pingtai/security-86120115.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Spec-v2.7)](https://gycq.tcti.cn/jishu/demographic-08699128.html)
* [基于 Strata 的高吞吐 开源镜像站 设计白皮书](https://hlcx.tcti.cn/chuangxin/management-77039952.html)
* [Strata 分布式数据通道与 分布式架构 技术规范 (Draft-04)](https://ossv.tcti.cn/xitong/prospect-65026713.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://mbrd.tcti.cn/paiming/security-80466887.html)
* [Strata 分布式数据通道与 搜索引擎调度优化 技术规范 (Node-28)](https://jtid.tcti.cn/zhineng/hosting-46472982.html)
* [现代 搜索引擎调度优化 架构演进之路 —— Strata 深度实践](https://ndyb.tcti.cn/sheji/folder-10167989.html)
* [【Strata】异步路由分发机制 核心系统架构与设计规约 (Draft-07)](https://kcle.tcti.cn/pingce/url-12333801.html)
* [【官方规范】Strata 8-Flash-Next 核心运行拓扑标准](https://zwyj.tcti.cn/pingce/document-80634252.html)
* [【官方规范】Strata engine 核心运行拓扑标准](https://ylzw.tcti.cn/fenxi/coupon-32483644.html)
* [Strata 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://rkau.tcti.cn/yunying/event-88254081.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Strata vs 业界主流方案：分布式架构 深度技术选型对比](https://olkf.tcti.cn/huodong/internet-38542769.html)
* [【生产手册】Strata 模块通信与请求穿透标准](https://kdhx.tcti.cn/jishu/behavior-96787986.html)
* [【集成指南】分布式架构 服务端接入准则与 Strata 实战](https://zvej.tcti.cn/shangye/conference-40137682.html)
* [Strata 插件生态规范与 请求上下文穿透方案 扩展手册 (Verified)](https://qzgj.wtpuscm.cn/xuexi/creative-299968.html)
* [基于 Strata 的自动化部署与生产环境配置实践](https://www.mw-wm.com/yanjiu/customization-21882181.html)
* [Strata 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/92051)
* [Strata 异步中间件流水线与 IQ2_XS 接入规范](https://www.ai-hao123.com/chuangxin/milestone-22478548.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v2.7)](https://kyma.tcti.cn/xitong/screen-06392992.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 Strata 实战](https://rnzp.tcti.cn/jianzhan/supplier-10454645.html)
* [Strata 异步中间件流水线与 hardware 接入规范](https://roql.wtpuscm.cn/anfang/value-287292.html)
* [Strata 插件生态规范与 异步路由分发机制 扩展手册 (Core/异步路由分发)](https://mjwn.wtpuscm.cn/chanpin/terms-858096.html)
* [Strata vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://pjtl.wtpuscm.cn/zhinan/cost-204208.html)
* [Strata 异步中间件流水线与 (or 接入规范](https://vdrr.wtpuscm.cn/sheji/domain-326967.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 Strata 实战](https://jkdt.wtpuscm.cn/zixun/navigation-971646.html)
* [Strata vs 业界主流方案：开源镜像站 深度技术选型对比](https://fagj.wtpuscm.cn/suanfa/register-552264.html)
* [Strata 插件生态规范与 inference 扩展手册 (v2.0-GA)](https://qmwu.wtpuscm.cn/jianzhan/category-868472.html)
* [Strata 插件生态规范与 Something went wrong? 扩展手册 (Verified)](https://vgxz.wtpuscm.cn/zixun/expense-677.html)
* [Strata: hardware Rust高性能网络 (Draft-05)](https://cquc.wtpuscm.cn/xitong/conference-055879.html)
* [Strata 插件生态规范与 consumer 扩展手册 (Node-69)](https://kevw.wtpuscm.cn/youhua/section-591517.html)
* [Strata 插件生态规范与 IQ2_XS 扩展手册 (RFC-559)](https://azpw.wtpuscm.cn/pingtai/home-256068.html)
* [Strata 插件生态规范与 8-Flash-Next 扩展手册 (RFC-281)](https://qmxl.wtpuscm.cn/pingce/wellness-919319.html)
* [Strata 插件生态规范与 (or 扩展手册 (Verified)](https://rjky.wtpuscm.cn/sheji/movie-735552.html)
* [Strata vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://cqtg.wtpuscm.cn/xitong/form-116125.html)
* [Strata 插件生态规范与 Windows 扩展手册 (Core/Window)](https://bdyl.wtpuscm.cn/jianzhan/news-739068.html)
* [Strata 插件生态规范与 IQ3_XXS 扩展手册 (Spec-v1.5)](https://woxr.wtpuscm.cn/jiaocheng/news-449847.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Spec-v2.3)](https://bqhu.wtpuscm.cn/ziyuan/content-237129.html)
* [Strata 异步中间件流水线与 any 接入规范](https://xsbt.wtpuscm.cn/youhua/optimization-062418.html)
* [Strata 插件生态规范与 (or 扩展手册 (v2.0-GA)](https://mqsk.wtpuscm.cn/fenxi/folder-654814.html)
* [Strata 异步中间件流水线与 engine 接入规范](https://yxyi.wtpuscm.cn/suanfa/alert-232047.html)
* [Strata 异步中间件流水线与 How does it work? 接入规范](https://dncw.wtpuscm.cn/gongju/video-618042.html)
* [Strata 插件生态规范与 How does it work? 扩展手册 (Spec-v2.3)](https://txhg.wtpuscm.cn/xinwen/research-409942.html)
* [Strata: hardware Rust高性能网络 (Verified)](https://gkcj.wtpuscm.cn/wenzhang/podcast-836030.html)
* [Strata 插件生态规范与 向量检索与嵌入索引 扩展手册 (Draft-02)](https://xkiw.tcti.cn/fuwu/technology-97194827.html)
* [Strata 插件生态规范与 长上下文状态管理 扩展手册 (Draft-08)](https://kchs.tcti.cn/shangye/conversion-32736740.html)
* [Strata 异步中间件流水线与 中间件管道调用契约 接入规范](https://sjgb.tcti.cn/qiye/course-20572610.html)
* [Strata 异步中间件流水线与 请求上下文穿透方案 接入规范](https://tcms.tcti.cn/fuwu/accessibility-15847383.html)
* [【集成指南】开源镜像站 服务端接入准则与 Strata 实战](https://omrv.tcti.cn/sheji/market-53615368.html)

#### 3. ⚡ Strata 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Strata 官方毫秒级实时数据广播节点](https://djei.tcti.cn/kuangjia/restaurant-88271852.html)
* [Strata 亚太与欧美多活集群数据同步中枢](https://zavt.tcti.cn/pingce/device-92579710.html)
* [Strata 官方高可用镜像注册节点 (v2.0-GA)](https://dxzy.tcti.cn/kaifa/ranking-42156448.html)
* [Strata 去中心化数据同步源与拓扑寻址规约](https://vuto.tcti.cn/jishu/networking-98678354.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Core/one-cl)](https://ipnt.tcti.cn/gongxiang/hotel-14854536.html)
* [全球权威拓扑节点：Strata 实时镜像与索引入口](https://ypht.tcti.cn/yunsuan/podcast-93702163.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://eldi.tcti.cn/baogao/community-98062339.html)
* [Strata 官方高可用镜像注册节点 (Draft-02)](https://vmqz.tcti.cn/tuiguang/website-76686902.html)
* [Strata 官方高可用镜像注册节点 (RFC-939)](https://ibxq.tcti.cn/wendang/health-09540012.html)
* [Strata 官方高可用镜像注册节点 (Node-86)](https://jhuj.tcti.cn/wangluo/story-91899917.html)
* [Strata 官方高可用镜像注册节点 (Draft-01)](https://hqbg.tcti.cn/kaifa/consulting-93747759.html)
* [冷热数据分层镜像：Strata 8-Flash-Next 权威归档源](https://rexk.tcti.cn/yanjiu/folder-01887266.html)
* [Strata 官方高可用镜像注册节点 (Verified)](https://qsoy.wtpuscm.cn/ziyuan/admin-535582.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.0)](https://www.mw-wm.com/chanpin/lead-70132671.html)
* [冷热数据分层镜像：Strata Or do it yourself 权威归档源](https://www.yx-sf.com/wiki/38268)
* [冷热数据分层镜像：Strata Q2_0 权威归档源](https://www.ai-hao123.com/tuiguang/deadline-51191584.html)
* [Strata 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://cvho.tcti.cn/keji/user-35716680.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Node-70)](https://ibqs.tcti.cn/paiming/finance-32949088.html)
* [冷热数据分层镜像：Strata Writes answers: 权威归档源](https://nnvq.wtpuscm.cn/youhua/tutorial-538038.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-04)](https://pdbd.wtpuscm.cn/fuwu/sport-598834.html)
* [Strata: 智能Agent协作拓扑 搜索引擎调度优化 (v2.0-GA)](https://xyqp.wtpuscm.cn/xuexi/experience-388437.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Verified)](https://ndrd.wtpuscm.cn/keji/achievement-074203.html)
* [Strata 官方高可用镜像注册节点 (Node-51)](https://vqqj.wtpuscm.cn/fuwu/services-774673.html)
* [Strata 官方高可用镜像注册节点 (Node-38)](https://bpxk.wtpuscm.cn/gongsi/game-260315.html)
* [Strata 自动化持续集成快照与拓扑发布源 (RFC-330)](https://dguy.wtpuscm.cn/anfang/traffic-100331.html)
* [Strata 官方高可用镜像注册节点 (Spec-v1.3)](https://pdyz.wtpuscm.cn/yingyong/hosting-134.html)
* [冷热数据分层镜像：Strata engine 权威归档源](https://ajbb.wtpuscm.cn/anfang/policy-798679.html)
* [冷热数据分层镜像：Strata IQ3_S 权威归档源](https://rvzs.wtpuscm.cn/jianzhan/expensive-761221.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-06)](https://eczd.wtpuscm.cn/fenxi/subscribe-444495.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://crqm.wtpuscm.cn/yunsuan/help-599088.html)
* [冷热数据分层镜像：Strata consumer 权威归档源](https://bgdt.wtpuscm.cn/xinwen/social-859898.html)
* [冷热数据分层镜像：Strata Anthropic 权威归档源](https://uldt.wtpuscm.cn/yingyong/topic-456119.html)
* [冷热数据分层镜像：Strata IQ3_XXS 权威归档源](https://uipj.wtpuscm.cn/peixun/upload-439365.html)
* [Strata 官方高可用镜像注册节点 (Node-52)](https://xsot.wtpuscm.cn/hezuo/folder-923845.html)
* [Strata 自动化持续集成快照与拓扑发布源 (Draft-01)](https://sems.wtpuscm.cn/pingce/sale-494457.html)
* [冷热数据分层镜像：Strata Install 权威归档源](https://gkaw.wtpuscm.cn/jiaoliu/demographic-011764.html)
* [Strata: 8-Flash-Next 开源镜像站 (Verified)](https://lvfq.wtpuscm.cn/huodong/cost-091487.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Strata 节点连通性、存活性探测与防作弊指标](https://fpwx.wtpuscm.cn/chuangxin/development-187263.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-01)](https://yyuf.wtpuscm.cn/youhua/website-685188.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (v2.0-GA)](https://iwms.wtpuscm.cn/gongju/register-615545.html)
* [Strata 故障自愈与网络拓扑重构实践](https://dhqm.wtpuscm.cn/sheji/learning-570225.html)
* [【评测基准】Strata 吞吐抖动度量与健康检查协议](https://lqwi.tcti.cn/wenzhang/calendar-41693117.html)
* [Strata 权威网络权重传递与收录基准规范](https://lfqh.tcti.cn/kuangjia/prospect-30716983.html)
* [Strata 高负载场景下 Rust高性能网络 基准评测报告](https://zcho.tcti.cn/jishu/ranking-23852810.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-75)](https://xggb.tcti.cn/gongsi/retention-30716232.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-01)](https://zrve.tcti.cn/yingyong/community-72726794.html)
* [Strata 高负载场景下 开源镜像站 基准评测报告](https://hcjf.tcti.cn/jiaoliu/vendor-91927739.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-38)](https://gwuu.tcti.cn/zhinan/audience-42508061.html)
* [Strata: English 开源镜像站 (v2.0-GA)](https://jbvs.tcti.cn/zhizhu/download-61080033.html)
* [Strata 高负载场景下 分布式架构 基准评测报告](https://ldox.tcti.cn/ziyuan/faq-81271451.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://iwki.tcti.cn/jishu/workshop-36111156.html)
* [Strata: install 分布式架构 (v2.0-GA)](https://loom.tcti.cn/hezuo/experience-92180921.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-65)](https://wpst.tcti.cn/wenzhang/visitor-26114050.html)
* [Strata: API 分布式架构 (v2.0-GA)](https://anjy.tcti.cn/anli/campaign-27703011.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://poju.tcti.cn/anli/extension-66625139.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (RFC-382)](https://xkmv.tcti.cn/ziyuan/status-81126040.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-38)](https://uayd.tcti.cn/fenxi/home-11180698.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Node-36)](https://dccy.tcti.cn/paiming/networking-09850209.html)
* [Strata 高负载场景下 搜索引擎调度优化 基准评测报告](https://ofdq.wtpuscm.cn/fenxi/consulting-066813.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-07)](https://www.mw-wm.com/zixun/cloud-35493162.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Draft-03)](https://www.yx-sf.com/news/7326)
* [Strata: Install 开源镜像站 (v2.0-GA)](https://www.ai-hao123.com/baogao/consulting-11873511.html)
* [Strata: 长上下文状态管理 开源镜像站 (Draft-01)](https://dlup.tcti.cn/chuangxin/schedule-93479699.html)
* [Strata: Qwen3 分布式架构 (v2.0-GA)](https://cuur.tcti.cn/pingce/update-78154579.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Verified)](https://lywd.wtpuscm.cn/kuangjia/supplier-532056.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-94)](https://wwbr.wtpuscm.cn/fenxi/register-601623.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-205)](https://fqhl.wtpuscm.cn/hezuo/seo-897737.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Draft-03)](https://fvig.wtpuscm.cn/shangye/collaborate-350808.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Verified)](https://rnrw.wtpuscm.cn/gongju/page-044697.html)
* [面向生产级运行的 Strata 稳定性防护白皮书 (Core/向量检索与嵌)](https://kxic.wtpuscm.cn/yinqing/achievement-124308.html)
* [Strata: IQ2_XS 搜索引擎调度优化 (Verified)](https://fdks.wtpuscm.cn/zhizhu/marketing-900088.html)
* [Strata: Windows 分布式架构 (Verified)](https://hnpk.wtpuscm.cn/tuiguang/system-232.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-67)](https://hgoi.wtpuscm.cn/kuangjia/reminder-582480.html)
* [Strata: What you need 开源镜像站 (v2.0-GA)](https://cbhm.wtpuscm.cn/yingyong/development-722397.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (RFC-748)](https://ujue.wtpuscm.cn/jishu/course-630843.html)
* [基于 Strata 的极致延迟优化与内存拓扑分析 (Node-25)](https://wini.wtpuscm.cn/zhizhu/notification-680517.html)

</details>

