# Cyber-Epoch

**中**：一份按时间线整理的工程记录，覆盖 2026-09-22 至 2026-09-29 期间的七个工作日。
**EN** : A chronological engineering log covering seven working days, 2026-09-22 through 2026-09-29.

---

## 0. 阅读约定 · Conventions

**中**
- 本文只记录**可复现**的事实：每一项数值都来自当次运行落盘的日志或采样文件，并标注了测量条件。
- 凡未经实机验证的内容，一律在正文中显式标注为「未验证」，并在第 4 节汇总，不与已验证结论混排。
- 为遵守发布规范，本文不含任何个人标识、账号、联系方式、主机名、内网地址或绝对路径。所有路径以 `<project_root>` 表示；第三方上游项目以「上游项目」「某 Electron 架构应用」等中性称谓指代。
- 工作涉及的子项目在本记录中以其**功能**命名（如「原生宿主」「媒体管线」「自建发行版」），不以品牌名出现。

**EN**
- Only **reproducible** facts are recorded. Every figure originates from a log or sample file persisted during the run in question, with its measurement conditions stated.
- Anything not verified on real hardware is explicitly marked *unverified* inline, and collected in Section 4. Unverified items are never mixed into the verified figures.
- No personal identifier, account, contact detail, hostname, private address or absolute path appears in this document. Paths are written as `<project_root>`; third-party upstream projects are referred to by neutral terms such as "the upstream project" or "an Electron-architecture application".
- Subprojects are named by **function** (e.g. "native host", "media pipeline", "self-built distribution") rather than by brand.

---

## 1. 摘要 · Abstract

**中**

本周期的工作集中在三条相对独立的工程线上，并在第七天汇合为一次统一的仓库化与测试收口。

第一条线是**媒体管线的吞吐优化**：在不牺牲主观画质的前提下，通过定位真正的耗时热点（编码阶段的速率控制参数），把一段 640×360 / 241 帧 / 带 Alpha 的测试素材插帧到 48 fps 的整条流水线墙钟时间从 25.1 秒压到 18.8 秒，并同步降低了峰值 CPU 占用。

第二条线是**桌面运行时的资源与正确性收敛**：第三方桌面宠物项目在本机被优化到素材编码规格不变、CPU 占用下降，随后又定位并修复了一个「窗口长期占用整块矩形输入区」的输入掩码缺陷；更晚些时候，一个基于 Electron 架构的桌面应用被以**零 Electron** 的方式迁移到 Linux，整机常驻内存从 1447 MB 收敛到 201 MB。

第三条线是**系统与协议层的自主实现**：使用 `mmdebstrap` 从 Debian 13.7 手工构建出一个可开机、可上网、可编译的自有发行版；以及在本机原本没有任何合成器的前提下，用纯 `ctypes` 直连 `libwayland-client`，实现了一个不依赖编译器、不依赖工具包的 Wayland 原生客户端，并给出了「全屏透明层」与「子表面」两种降级路径的实测代价。

贯穿全周期的一条隐性主线是**测量可信度**：本周期内至少有三项初步结论被后续复核推翻或修正，相关记录集中在 §3.6。这些修正没有被删除，而是保留在版本历史中作为方法学参考。

**EN**

Work in this period proceeded along three largely independent lines, converging on day seven into a unified repository consolidation and test closure.

The first line was **media-pipeline throughput**. Without compromising subjective quality, the actual hot spot was located (rate-control parameters in the encoding stage). End-to-end wall-clock time for interpolating a 640×360, 241-frame, alpha-bearing test asset to 48 fps dropped from 25.1 s to 18.8 s, with peak CPU utilisation reduced at the same time.

The second line was **resource and correctness convergence of a desktop runtime**. A third-party desktop-pet project was tuned on the local machine so that CPU utilisation fell without changing asset encoding specifications. A separate input-mask defect — a window permanently occupying a full rectangular input region — was root-caused and fixed. Later, an Electron-architecture desktop application was migrated to Linux with **zero Electron**, reducing whole-machine resident memory from 1447 MB to 201 MB.

The third line was **independent implementation at the system and protocol layers**: a bootable, network-capable, self-compiling distribution built from Debian 13.7 with `mmdebstrap`; and — on a host that originally shipped no compositor at all — a raw Wayland client built purely on `ctypes` against `libwayland-client`, requiring neither a compiler nor a toolkit, together with measured costs for two fallback architectures (full-screen transparent layer, and sub-surface).

An implicit thread runs through the whole period: **measurement integrity**. At least three preliminary conclusions were overturned or corrected by later review; the records are consolidated in §3.6. They are retained rather than deleted, as methodology references.

---

## 2. 阶段性工程日志 · Phase Log

### Phase I — 媒体管线吞吐优化 · Media-Pipeline Throughput
**2026-09-22 → 2026-09-24**

**中**

**问题陈述。** 该管线负责将源视频插帧至目标帧率，产物必须保留 Alpha 通道，因此输出容器限定为 WebM。早期版本的瓶颈被直觉归因于插帧推理，但分段计时显示这个判断是错的。

**分段计时（640×360 / 241 帧 / 带 Alpha → 48 fps，同机）：**

| 阶段 | 墙钟 | 峰值 CPU |
|---|---|---|
| 拆帧 | 0.9 s | ~785% |
| 插帧推理 | 10.8 s | ~336% |
| 编码 | 5.4 s | ~264% |
| **合计** | **18.8 s** | — |

**根因。** 编码阶段的速率控制参数从未被显式设置，编码器运行在其默认档位（`cpu-used=0`）。逐档对照：

| 配置 | 编码耗时 | 峰值 CPU | 产物大小 |
|---|---|---|---|
| `cpu-used=0`（原默认） | 13.09 s | 364% | 1257 KB |
| `cpu-used=4` | 7.40 s | — | — |
| `cpu-used=4` + `realtime` | **5.35 s** | **264%** | 1435 KB |

编码耗时下降 59%，峰值 CPU 从 364% 降至 264%，结构相似性指标 SSIM 0.9964，产物体积增加 14%。**整条流水线因此从 25.1 秒降至 18.8 秒。**

**调度策略。** 除绝对速度外，另定义了四种性能档位，用于区分「真实能耗」与「用户可感知的卡顿」：

| 档位 | 墙钟 | 总 CPU 工作量 | 平均占用 |
|---|---|---|---|
| 旧版 | 27.6 s | 68.3 CPU·s | 247% |
| `auto`（默认） | 19.5 s | 40.8 CPU·s | 209% |
| 省电 | 26.8 s | 40.8 CPU·s | 152% |
| 全速 | 19.9 s | 42.1 CPU·s | 212% |

值得说明的是「省电」档并不降低总能耗（40.8 CPU·s，与 `auto` 相同），它降低的是**瞬时并发度**：把平均占用从 209% 压到 152%（约 1.5 个核心），使同一台机器上的前台交互更少受影响。

**硬件适配。** 设备的自动选择对吞吐影响显著：自动绑定独立 GPU 用时 5 秒，而强行指定软件光栅化后端为 29 秒，相差 5.8 倍。核显路径的对照结论是：同一素材下集成显卡比独立显卡慢 3.3~3.5 倍（18.8 s → 约 43 s），但输出画质逐像素平均差仅 0.0132/255，部分帧逐字节相同 —— 即**换用集成显卡只影响耗时，不影响正确性**。

**格式自适应。** 检测到输入不含有效 Alpha（整段所有像素的 Alpha 均为 255）时，跳过 Alpha 插帧路径并改用不支持 Alpha 的输出像素格式。实测该分支使一段 GIF 的处理时间从 7.46 秒降至 5.51 秒（-26%）。

**EN**

**Problem statement.** The pipeline interpolates source video to a target frame rate. Alpha must survive, so the output container is restricted to WebM. Early versions attributed the bottleneck to interpolation inference; staged timing showed that attribution to be wrong.

**Staged timing (640×360 / 241 frames / with alpha → 48 fps, same host):**

| Stage | Wall clock | Peak CPU |
|---|---|---|
| Demux / frame extraction | 0.9 s | ~785% |
| Interpolation inference | 10.8 s | ~336% |
| Encoding | 5.4 s | ~264% |
| **Total** | **18.8 s** | — |

**Root cause.** The encoder's rate-control parameters had never been set explicitly, so it ran at its default setting (`cpu-used=0`). Per-step comparison:

| Configuration | Encode time | Peak CPU | Output size |
|---|---|---|---|
| `cpu-used=0` (prior default) | 13.09 s | 364% | 1257 KB |
| `cpu-used=4` | 7.40 s | — | — |
| `cpu-used=4` + `realtime` | **5.35 s** | **264%** | 1435 KB |

Encode time fell 59%, peak CPU fell from 364% to 264%, SSIM held at 0.9964, and output size grew 14%. **The whole pipeline consequently dropped from 25.1 s to 18.8 s.**

**Scheduling policy.** Beyond raw speed, four performance profiles were defined to distinguish *actual energy cost* from *user-perceived contention*:

| Profile | Wall clock | Total CPU work | Mean occupancy |
|---|---|---|---|
| legacy | 27.6 s | 68.3 CPU·s | 247% |
| `auto` (default) | 19.5 s | 40.8 CPU·s | 209% |
| power-saver | 26.8 s | 40.8 CPU·s | 152% |
| full-speed | 19.9 s | 42.1 CPU·s | 212% |

Notably the power-saver profile does not reduce total energy (40.8 CPU·s, identical to `auto`); it reduces *instantaneous concurrency*, lowering mean occupancy from 209% to 152% (about 1.5 cores) so that foreground interaction on the same machine is disturbed less.

**Hardware adaptation.** Automatic device selection materially affects throughput: automatic binding to the discrete GPU completes in 5 s, whereas forcing the software rasteriser takes 29 s — a 5.8× difference. The integrated-GPU comparison concludes: the iGPU path is 3.3–3.5× slower on the same asset (18.8 s → ~43 s), yet the mean per-pixel output difference is 0.0132/255 with some frames byte-identical — i.e. **switching to integrated graphics affects latency, not correctness**.

**Format adaptation.** When the input is detected to carry no effective alpha (every pixel across the whole clip has alpha = 255), the alpha interpolation path is skipped and a non-alpha output pixel format is used. Measured on one GIF, this branch cut processing time from 7.46 s to 5.51 s (-26%).

---

### Phase II — 运行时缺陷根因分析 · Runtime Defect Root-Cause Analysis
**2026-09-24 → 2026-09-27**

**中**

**缺陷一：窗口吞掉整块屏幕的鼠标事件。**

*现象。* 桌面宠物窗口在视觉上是透明的不规则轮廓，但其下方的其他窗口无法接收点击。

*定位。* 该运行时依靠一个掩码同步函数（调用点 3 处）把透明区域从窗口输入区中挖掉，只保留角色轮廓。逐调用点核查后发现：三个调用点的触发条件分别是「帧交付」「缩放变更」与另一条同类路径，而在窗口首次显示、尚无任何帧可交付的时刻，三者**均不满足**。结果是该函数从进程启动到退出**一次都未执行**，窗口始终停留在工具包默认的「整块矩形输入区」状态。

*修复。* 两处改动，共 +52 / −3 行：
1. 在窗口显示事件的末尾主动调用一次掩码同步，使输入区在显示时即收敛，而不依赖首帧到达；
2. 在掩码同步函数内增加「尚无可用帧」的兜底分支。

*修复中发现的次生约束。* 兜底分支需要一个退化掩码。**不能传入空区域** —— 在所用 GUI 工具包中，空 `QRegion` / 空 `QBitmap` 的语义是「清除掩码」，其效果等价于恢复为**全窗口**输入区，与意图完全相反。必须使用一个 1×1 的非空退化区域。

**缺陷二：素材命名约定不一致导致动画表几乎为空。**

*现象。* 发行包中的素材文件名使用 `#Uxxxx` 形式的转义名，而源码仓库中使用中文名；动画表以文件主名（`stem`）作为动画标识。对同一批 51 个动画做匹配统计：转义名命中 **0/51**，中文名命中 **46/51**。

*处置。* 将 106 个素材文件由转义名解码回中文名，原稿另行备份。该改动**不是兼容性补丁，而是命名约定的统一**。

**扩展机制：外部角色包。**

在此基础上增加了一套外部角色包机制，使角色可以以数据包形式安装，而不必修改发行版本体。补丁规模为 2 个补丁、合计 +210 / −9 行，全部为新增逻辑，未改写既有分支：
- `0001-本地扩展包与画布自适应.patch`（+158 / −6）：扩展外部角色目录的扫描范围；新增包级画布声明（`canvas{width,height,feet_y,fps}`）与缩放因子，并在库创建阶段将其应用到模块级画布常量上。
- `0002-无帧时不占输入区.patch`（+52 / −3）：即上文缺陷一的修复。

**EN**

**Defect 1: the window consumed mouse input across the entire screen.**

*Symptom.* The desktop-pet window presents a transparent, non-rectangular silhouette, yet other windows beneath it never received clicks.

*Diagnosis.* The runtime relies on a mask-synchronisation function (3 call sites) to carve transparent areas out of the window input region, leaving only the character outline. Auditing each call site showed their trigger conditions were "frame delivered", "scale changed" and one equivalent path. At the moment the window is first shown, none of those conditions holds, because no frame has yet been delivered. Consequently the function **never executed** between process start and exit, and the window remained in the toolkit default state of a full rectangular input region.

*Fix.* Two changes, +52 / −3 lines in total:
1. invoke the mask synchronisation explicitly at the end of the window's show event, so the input region converges on display rather than waiting for a first frame;
2. add a "no usable frame yet" fallback branch inside the mask-synchronisation function.

*Secondary constraint discovered during the fix.* The fallback branch requires a degenerate mask. An **empty region must not be used** — in the GUI toolkit concerned, an empty `QRegion` / `QBitmap` is interpreted as *clearing* the mask, which is equivalent to restoring a **full-window** input region, precisely the opposite of intent. A degenerate non-empty 1×1 region is required instead.

**Defect 2: inconsistent asset naming left the animation table nearly empty.**

*Symptom.* Distribution assets use escaped names of the form `#Uxxxx`, whereas the source repository uses native names. The animation table keys on the file stem. Across the same 51 animations: escaped names matched **0/51**, native names matched **46/51**.

*Remediation.* 106 asset files were decoded from escaped names back to native names, with the originals backed up separately. This was **not a compatibility patch but a unification of naming convention**.

**Extension mechanism: external character packs.**

On top of this, an external character-pack mechanism was added so that characters can be installed as data packages without modifying the distribution itself. It consists of two patches totalling +210 / −9 lines, all additive, with no existing branch rewritten:
- `0001-local-extension-packs-and-canvas-adaptation.patch` (+158 / −6): widens the scan path for external character directories; adds a package-level canvas declaration (`canvas{width,height,feet_y,fps}`) plus a scale factor, applied to module-level canvas constants during library creation.
- `0002-no-input-region-while-frameless.patch` (+52 / −3): the fix for Defect 1 above.

---

### Phase III — 独立发行版构建 · Independent Distribution Build
**2026-09-25 → 2026-09-26**

**中**

**目标。** 以现成组件从 Debian 13.7 (`trixie`) 手工组装一个完整的桌面系统，产出可复刻的构建脚本与实测对比数据，并预留 PCI 直通能力。

**构建路径与规模演进。**

| 阶段 | 包数 | 说明 |
|---|---|---|
| 底座 | 248 | 含内核，可在虚拟机中引导至 `login:` |
| 加桌面环境 | 964 | 整套 XFCE 4.20 + 显示管理器 + 文件管理器 + 面板插件 |
| 终版 | **1349** | 加上办公/CJK/音视频/开发工具链 |

**终版产物。** 根文件系统 4.1 GB；可直接引导的 8 GB 稀疏镜像（实际占用 4.2 GB）。

**实测对比（同机、同一块磁盘，1 GB 内存 / 2 vCPU / KVM）：**

| 指标 | 自建发行版 | 宿主发行版 |
|---|---|---|
| 包数量 | 1349 | 2311（少 42%） |
| 进程数 | 149 | 330（少 55%） |
| 空载内存 | 474 MB | 未测（宿主当时有前台负载，不为此中断） |
| 开机到图形界面 | 约 11 秒 | — |
| 失败系统单元 | 0 | — |
| 网络 | DHCP 获取成功 | — |
| 中文字形 | 83 个 | — |
| 工具链 | gcc 14.2.0 / Python 3.13.5 / ffmpeg 7.1.5 / Vulkan(llvmpipe) | — |

空载内存与开机耗时由客机内部的自检服务在图形界面就绪后 50 秒写入串口，宿主侧直接读取，无需登录。

**事故与工程护栏。**

2026-09-25 23:15 发生一次宿主级联故障：构建过程未受任何资源上限约束，且构建脚本把宿主的 `/dev` 与 `/run` 以 `rbind` 方式暴露进 `chroot`，最终导致宿主内存耗尽、桌面进程崩溃、需要强制重启。事后复盘确认：内核日志中无 OOM 记录（日志服务当时已无法落盘），因此**不能仅凭「日志里没有 OOM」判断未发生内存耗尽**。

此后所有构建脚本统一加入三条护栏：
1. **隔离**：在 `chroot` 内构造私有的最小 `/dev`（按需 `mknod`），并提供私有且限额的 `/dev/shm`（512 MB）与 `/run`（32 MB）。**绝不 `rbind` 宿主机的 `/dev`、`/run`。**另以 `policy-rc.d`（`exit 101`）阻止包安装脚本启动系统服务。
2. **限额**：整个构建被包裹进临时 cgroup —— `MemoryMax=4G`、`MemorySwapMax=2G`、`CPUQuota=250%`、`IOWeight=40`；即使失控，受影响范围也仅限该构建单元。
3. **准入**：启动前检查 `MemAvailable`，低于 2500 MB 直接拒绝执行。

另有内层硬超时与 `trap` 挂载清理，保证异常退出后不留残余挂载。复测结果：一轮构建后宿主可用内存保持在约 10 GB，残余挂载 0 项。

**桌面外观渠道排查。**

发行版改名后壁纸未生效。定位过程：将客机截图与根文件系统中 104 张候选图像逐一做像素比对（缩放到同一分辨率后取同区域计算平均绝对差），目标图像差值 **0.15**，第二名 **18.3** —— 客机实际渲染的仍是桌面包**编译内置**的默认图，而非任何一份配置指定的图。

进一步核查确认：该桌面环境的壁纸存储在一个名为 `xfce4-desktop` 的配置渠道中，其**配置文件名必须严格为 `xfce4-desktop.xml`**；此前写入的文件名不匹配，因此从未被读取。修正后采用三重保险：正确文件名（系统级与用户预置各一份，并覆盖两个可能的显示接口名）、主题壁纸目录内全部格式替换、以及默认兜底图本身替换。

**EN**

**Objective.** Assemble a complete desktop system from Debian 13.7 (`trixie`) using existing components, producing reproducible build scripts, measured comparison data, and reserved PCI-passthrough capability.

**Build path and scale progression.**

| Stage | Package count | Note |
|---|---|---|
| Base | 248 | Includes kernel; boots to `login:` in a VM |
| + desktop environment | 964 | Full XFCE 4.20 + display manager + file manager + panel plugins |
| Final | **1349** | Plus office / CJK / A-V / development toolchain |

**Final artefacts.** 4.1 GB root filesystem; an 8 GB sparse, directly bootable image occupying 4.2 GB on disk.

**Measured comparison (same host, same disk, 1 GB RAM / 2 vCPU / KVM):**

| Metric | Self-built distribution | Host distribution |
|---|---|---|
| Packages | 1349 | 2311 (−42%) |
| Processes | 149 | 330 (−55%) |
| Idle memory | 474 MB | not measured (host had foreground load; not interrupted for this) |
| Boot to graphical session | ~11 s | — |
| Failed system units | 0 | — |
| Network | DHCP obtained | — |
| CJK glyph coverage | 83 | — |
| Toolchain | gcc 14.2.0 / Python 3.13.5 / ffmpeg 7.1.5 / Vulkan (llvmpipe) | — |

Idle memory and boot time were reported by an in-guest self-check service that writes to the serial port 50 s after the graphical session is ready; the host reads it directly, requiring no login.

**Incident and engineering guards.**

On 2026-09-25 at 23:15 a host-level cascading failure occurred: the build ran without any resource ceiling, and the scripts exposed the host's `/dev` and `/run` into the `chroot` via `rbind`, ultimately exhausting host memory, crashing desktop processes, and requiring a forced reboot. Post-mortem confirmed the kernel log contained **no** OOM record — the logging service was already unable to persist at that point. The lesson: **absence of an OOM record is not evidence that memory exhaustion did not occur.**

All build scripts subsequently gained three guards:
1. **Isolation**: construct a private minimal `/dev` inside the `chroot` (with explicit `mknod` for required nodes), plus private, size-capped `/dev/shm` (512 MB) and `/run` (32 MB). **Never `rbind` the host's `/dev` or `/run`.** A `policy-rc.d` stub (`exit 101`) additionally prevents package install scripts from starting services.
2. **Limits**: wrap the entire build in a transient cgroup — `MemoryMax=4G`, `MemorySwapMax=2G`, `CPUQuota=250%`, `IOWeight=40`. Even under runaway behaviour, blast radius is confined to that unit.
3. **Admission control**: check `MemAvailable` before starting; refuse to run below 2500 MB.

An inner hard timeout and `trap`-based mount cleanup guarantee no residual mounts after abnormal exit. Re-measurement: after one full build the host's available memory stayed near 10 GB, with zero residual mounts.

**Desktop appearance channel investigation.**

After the distribution was renamed, the wallpaper did not change. Diagnosis: the guest screenshot was compared pixel-wise against all 104 candidate images inside the root filesystem (scaled to a common resolution, mean absolute difference over the same region). The intended image scored **0.15**; the runner-up scored **18.3** — the guest was in fact rendering the desktop package's **compiled-in** default image rather than any configured one.

Further inspection confirmed: this desktop environment stores the wallpaper in a configuration channel named `xfce4-desktop`, and the **config file name must be exactly `xfce4-desktop.xml`**. The previously written file had a non-matching name and was therefore never read. The corrected approach uses three layers of redundancy: the correct file name (both system-level and user-preset, covering both possible display interface names), replacement of every format in the theme wallpaper directory, and replacement of the default fallback image itself.

---

### Phase IV — 桌面运行时的去浏览器化 · De-browserization of the Desktop Runtime
**2026-09-28**

**中**

**背景。** 一个基于 Electron 架构的桌面宠物应用官方仅提供 Windows / macOS 构建。目标是在 Linux 上运行它，同时把常驻资源压到前台可用水平。

**第一阶段：保留架构，收敛资源配置。**

在完全不改动上游业务代码的前提下（仅使用其自身提供的扩展点），逐档测量整机资源。同口径结果：

| 档位 | 内存 | 进程数 | CPU | 画面是否正常 |
|---|---|---|---|---|
| ① 原样 | 1447 MB | 7 | 134% | ✔ |
| ② + 低端设备模式 + V8 堆上限 128 MB | 944 MB | 7 | 124% | ✔ |
| ③ ② 再关 GPU | 789 MB | 7 | 14% | ✘ 空白画布 |
| ④ ③ 再单进程 | 574 MB | 3 | 14% | ✘ 空白画布 |
| ⑦ 保留 GPU + 单进程 | 701 MB | 3 | 133% | ✔ |
| ⑧ ⑦ 再改软件 GL（SwiftShader） | 701 MB | 3 | 110% | ✔ |
| ⑨ ⑧ 再撤去中间启动进程 + 堆上限 32 MB | 631 MB | 2 | — | ✔ ← 定稿 |
| 正式实例（⑨，稳态跑一分钟） | **629 MB** | **2** | — | ✔ |

**关于档位 ③ / ④ 的说明（本周期最重要的一个陷阱）。** 关闭 GPU 后端后内存显著下降、且 CPU 占用极低，看起来是首选配置。但该应用的骨骼合成依赖 WebGL2。禁用 GPU 后合成器立即抛出 `webgl2 unavailable`，画布**静默变为空白**。危险之处在于：**动画帧循环仍在运行、帧率读数仍为 48 fps、内存也确实下降了** —— 仅凭帧率与内存两个指标无法发现画面已经消失。必须读取画布像素或直接观察屏幕才能验收。

**定稿配置的构成：**
- 直接以脚本形式启动核心进程，撤去仅负责派生子进程的中间启动器。依据是核心模块中对父进程只使用了 `process.send?.()` 与 `disconnect` 兜底，**不存在 IPC 握手**，因此可独立运行。该项节省约 47 MB / 1 个进程。
- 单进程模式（渲染进程并入主进程），为内存收益最大的一项，约 −240 MB。
- V8 堆上限 32 MB / 半空间 2 MB（实测 JS 堆实际占用仅 9.5 MB，32 MB 上限余量充足）。
- 软件 GL 改用 SwiftShader，相比默认的软件回退路径节省约 25% CPU。
- 关闭缓存与后台服务；终端与另一可选子系统在配置中禁用。
- 帧率上限 24 fps（详见 §3.6 的订正记录）。

**该阶段的性能下限。** 即使采用最激进的进程模型，空白页 + 单进程的绝对下限仍为 **383 MB** —— 最终 629 MB 中有约 61% 属于浏览器引擎自身的基础开销。

**第二阶段：重写运行时，彻底移除浏览器引擎。**

在明确「资源下限由浏览器引擎决定」之后，改为用 Python + Qt 重写整个宠物运行时，**完全不使用 Web 引擎**。上游核心逻辑保持独立进程、不做改动。

| 指标 | 浏览器引擎方案 | 原生重写方案 |
|---|---|---|
| 整机内存 | 583 ~ 629 MB | **201 MB**（核心 117 + 身体 84） |
| 进程数 | 2 | 2 |
| 身体侧常驻内存 | 约 466 MB | **84 MB** |
| 每帧渲染 | — | 10 ~ 25 ms |

**内存收益明确（约降至三分之一），但 CPU 未获得同等收益。** 瓶颈从「浏览器合成」转移到了「每帧以 CPU 合成网格」。已通过「空闲时自动降帧」与「网格抽稀」两轮压制，结论是这一路径的 CPU 优势有限，真正的大头在渲染算法本身。

**端到端复测（真实核心进程 + 真实开窗）：**

```
窗口出现时间    4.8 s
进程数          4
宿主侧 CPU     43.0%（整段平均）
核心进程 CPU    0.2%
内存           RSS 217 MB（PSS 190 MB）
CPU 四段采样    42 / 44 / 44 / 43 %  →  最低 42.4% / 平均 43.4% / 最高 44.4%
温度           core 34 °C / PCH 45 °C
```

这是本机第一次量到「CPU 稳定在 42~44%」的平负载 —— 此前记录的 19~82% 区间源于「角色会走会停」造成的单点采样失真，不具备代表性。

**交付物自检（AppImage 形态，运行 12 秒）：**

```
稳态帧率        24.0 fps（跑满设定上限）
每帧耗时        1.8 ms
每次重绘        18.5 ~ 19.8 ms（其中骨骼贴合 18.3 ms、网格出图 20.9 ms）
体积            70.4 MB
```

**EN**

**Background.** An Electron-architecture desktop-pet application ships officially only for Windows and macOS. The goal was to run it on Linux while bringing resident resource usage down to a level acceptable for interactive use.

**Stage 1: keep the architecture, converge the configuration.**

With upstream business logic entirely untouched (only extension points it already exposes), whole-machine resources were measured tier by tier. Same-scope results:

| Tier | Memory | Processes | CPU | Rendering correct |
|---|---|---|---|---|
| ① as-is | 1447 MB | 7 | 134% | ✔ |
| ② + low-end device mode + V8 heap cap 128 MB | 944 MB | 7 | 124% | ✔ |
| ③ ② plus GPU disabled | 789 MB | 7 | 14% | ✘ blank canvas |
| ④ ③ plus single-process | 574 MB | 3 | 14% | ✘ blank canvas |
| ⑦ GPU retained + single-process | 701 MB | 3 | 133% | ✔ |
| ⑧ ⑦ plus software GL (SwiftShader) | 701 MB | 3 | 110% | ✔ |
| ⑨ ⑧ minus the intermediate launcher + heap cap 32 MB | 631 MB | 2 | — | ✔ ← final |
| Production instance (⑨, one-minute steady state) | **629 MB** | **2** | — | ✔ |

**On tiers ③ / ④ — the single most important trap in this period.** Disabling the GPU backend lowered memory markedly and left CPU extremely low, so it appeared to be the preferred configuration. But the application composites its skeleton via WebGL2. With the GPU disabled the compositor immediately throws `webgl2 unavailable` and the canvas **silently goes blank**. The danger is that **the animation frame loop keeps running, the frame-rate readout still shows 48 fps, and memory genuinely did decrease** — neither frame rate nor memory can reveal that the picture has disappeared. Acceptance requires reading canvas pixels or observing the screen directly.

**Composition of the final configuration:**
- Launch the core process directly by script, removing the intermediate launcher that merely forks a child. Justification: the core module references the parent only through `process.send?.()` and a `disconnect` fallback, with **no IPC handshake**, so it runs standalone. Saving: about 47 MB / 1 process.
- Single-process mode (renderer folded into the main process), the largest memory win at roughly −240 MB.
- V8 heap cap 32 MB / semi-space 2 MB (JS heap actually used only 9.5 MB, leaving ample headroom).
- Software GL switched to SwiftShader, saving roughly 25% CPU versus the default software fallback.
- Caches and background services disabled; the terminal and one optional subsystem disabled in configuration.
- Frame-rate ceiling of 24 fps (see the correction in §3.6).

**The floor of this stage.** Even with the most aggressive process model, a blank page in single-process mode still costs **383 MB** — about 61% of the final 629 MB is the browser engine's own baseline.

**Stage 2: rewrite the runtime, removing the browser engine entirely.**

Once it was established that the resource floor is set by the browser engine, the entire pet runtime was rewritten in Python + Qt with **no web engine at all**. The upstream core logic remains a separate, unmodified process.

| Metric | Browser-engine approach | Native rewrite |
|---|---|---|
| Whole-machine memory | 583 ~ 629 MB | **201 MB** (core 117 + body 84) |
| Processes | 2 | 2 |
| Body-side resident memory | ~466 MB | **84 MB** |
| Per-frame rendering | — | 10 ~ 25 ms |

**The memory win is unambiguous (roughly one third), but CPU did not gain equivalently.** The bottleneck moved from *browser compositing* to *per-frame CPU grid compositing*. Two rounds of mitigation ("auto frame-rate halving when idle", "grid decimation") were applied; the conclusion is that this path offers limited CPU advantage, and the real cost lies in the rendering algorithm itself.

**End-to-end re-measurement (real core process + real window):**

```
Window appearance    4.8 s
Processes            4
Host-side CPU        43.0% (segment mean)
Core process CPU     0.2%
Memory               RSS 217 MB (PSS 190 MB)
CPU, four segments   42 / 44 / 44 / 43 %  →  min 42.4% / mean 43.4% / max 44.4%
Temperature          core 34 °C / PCH 45 °C
```

This was the first flat 42–44% CPU load measured on this host. The previously recorded 19–82% range was sampling distortion caused by the character alternately walking and resting, and is not representative.

**Deliverable self-check (AppImage form, 12 s run):**

```
Steady-state rate   24.0 fps (saturating the configured ceiling)
Per-frame cost      1.8 ms
Per repaint         18.5 ~ 19.8 ms (skeleton binding 18.3 ms, grid raster 20.9 ms)
Size                70.4 MB
```

---

### Phase V — 音频处理子系统 · Audio Processing Subsystem
**2026-09-27 → 2026-09-28**

**中**

**目标。** 为既有媒体管线增加音频修复能力，且不引入任何第三方 Python 依赖（全部基于通用媒体工具自带滤镜）。

**处理链（默认）：** 低频衰减均衡 → 响度标准化（默认 −16 LUFS）→ 静音裁剪（阈值 >0.5 s）→ 限制器兜底。

**一次完整的真实运行（58 分钟素材）：**

| 指标 | 输入 | 输出 |
|---|---|---|
| 响度 | −13.29 LUFS | **−17.75 LUFS** |
| 真峰值 | +1.35 dBTP | **−1.91 dBTP** |
| 动态范围 | 5.10 LU | 3.90 LU |
| 时长 | 3468.12 s | 2008.06 s（57:48 → 33:28） |
| 裁掉的静音 | — | **1460.07 s（24 分 20 秒）** |

端到端耗时 3 分 15 秒，其中渲染 26.3 秒，其余为两遍响度分析。

**增益决策（「保真」策略）。** 均衡后实测为 −14.32 LUFS / +1.44 dBTP。要达到目标 −16 LUFS 只需施加 −1.59 dB，但真峰值不允许超过 −1.0 dBTP，这要求施加 −3.44 dB。取两者中更严的一个，即 **−3.44 dB**，因此最终落在 −17.75 LUFS 而非 −16。**预测值 −17.76，实测 −17.75，误差 0.01 LU。**

这一取舍是本子系统最需要向使用者交代的一点：**在「保真」策略下，输出响度会刻意低于目标值**，因为真峰值约束优先。要精确命中目标需改用「保响度」策略，代价是约 1.9 dB 的峰值限幅。

**外挂插件架构。** 需要更高处理能力时，通过子进程 + 单行 JSON 的作业/结果协议调用外部模型，主程序零第三方依赖。实测其中一个语音增强模型（30 秒片段）：80–300 Hz −0.1 dB、1.5k–4k −1.2 dB、**>8 kHz −5.9 dB**，处理速度约 2.7 倍实时（58 分钟素材约需 20 分钟）。结论是**该类模型主要抑制高频噪声，不负责改善低频沉闷感** —— 后者由均衡阶段承担。

**工具链经验（可复用）：**
- 响度分析工具的 JSON 输出只写入标准错误流，不写入标准输出。
- 音频统计滤镜在多路输出时，输出顺序与分支顺序**不一致**，须按带内部标识的输出名认领，不可按顺序对齐。
- 限制器滤镜的默认电平补偿会凭空引入约 +1 LU 的增益，必须显式关闭。
- 静音裁剪必须置于增益调整**之前**，且与测量链同序。
- 将 Opus 音轨封装进 MP4 容器会导致无声，跨容器时必须转码为 AAC。

**EN**

**Objective.** Add audio restoration to the existing media pipeline without introducing any third-party Python dependency — everything is built on filters shipped with the general-purpose media tool.

**Default chain:** low-frequency shelving EQ → loudness normalisation (default −16 LUFS) → silence trimming (threshold > 0.5 s) → limiter as the final stage.

**One complete real run (58-minute asset):**

| Metric | Input | Output |
|---|---|---|
| Loudness | −13.29 LUFS | **−17.75 LUFS** |
| True peak | +1.35 dBTP | **−1.91 dBTP** |
| Dynamic range | 5.10 LU | 3.90 LU |
| Duration | 3468.12 s | 2008.06 s (57:48 → 33:28) |
| Silence removed | — | **1460.07 s (24 min 20 s)** |

End-to-end took 3 min 15 s, of which 26.3 s was rendering; the remainder was two passes of loudness analysis.

**Gain decision (the "fidelity-first" strategy).** After EQ the measurement was −14.32 LUFS / +1.44 dBTP. Reaching the −16 LUFS target would require only −1.59 dB, but the true peak may not exceed −1.0 dBTP, which demands −3.44 dB. The stricter of the two was taken — **−3.44 dB** — so the result lands at −17.75 LUFS rather than −16. **Predicted −17.76 versus measured −17.75: an error of 0.01 LU.**

This trade-off is the single most important thing to communicate to users of this subsystem: **under the fidelity-first strategy the output loudness is deliberately below target**, because the true-peak constraint takes precedence. Hitting the target exactly requires the loudness-first strategy, at the cost of roughly 1.9 dB of peak limiting.

**Plugin architecture.** When greater processing power is needed, external models are invoked through a subprocess protocol carrying a one-line JSON job and a one-line JSON result, keeping the host application free of third-party dependencies. Measured on one speech-enhancement model (30-second excerpt): 80–300 Hz −0.1 dB, 1.5k–4k −1.2 dB, **>8 kHz −5.9 dB**, running at roughly 2.7× real time (about 20 minutes for a 58-minute asset). The conclusion: **this class of model primarily suppresses high-frequency noise and does not address low-frequency dullness** — that remains the EQ stage's responsibility.

**Toolchain findings (reusable):**
- The loudness analysis tool writes its JSON report to standard error only, never to standard output.
- When an audio statistics filter has multiple outputs, their order **does not** match branch order; they must be claimed by their internally identified output names rather than by position.
- The limiter filter's default level compensation injects roughly +1 LU out of nowhere and must be explicitly disabled.
- Silence trimming must be applied **before** gain adjustment and in the same order as the measurement chain.
- Wrapping an Opus track in an MP4 container yields silence; cross-container work requires transcoding to AAC.

---

### Phase VI — 仓库化、测试归因与上游同步 · Repository Consolidation, Test Attribution, Upstream Sync
**2026-09-29**

**中**

**一、仓库化与脱敏。**

将散落在开发机上的工作划分为 8 个仓库，其中 6 个为本周期产出的项目代码与文档，2 个为归档性质。全部先创建为私有仓库。

自动化脱敏分三道：
1. **文件名黑名单**（密码 / 密钥 / token / `.env` / 私钥等）—— 命中文件不进入仓库；
2. **凭证模式匹配**（API key 样式、平台令牌、私钥头、密钥赋值语句、网络认证凭据等）—— 命中即替换为占位符；
3. **个人与机器信息**（用户名、绝对路径、主机名、内网地址、设备名、订单号、昵称）—— 替换为 `<user>` / `<intranet-ip>` / `<hostname>` 一类占位符。

最终复核：8 个仓库全量扫描，**凭证类命中 0 项**；共替换 62 处用户标识、14 处内网地址、3 处主机名。每个仓库另附一份脱敏说明。

**二、全量测试与失败归因。**

对改动规模最大的那一个项目，运行其完整测试套件（受控条件下，降低 CPU 与 IO 优先级）：

| 套件 | 结果 | 用时 |
|---|---|---|
| Python 单元测试 | **2887 通过 / 17 失败 / 21 跳过**（共 2925） | 263 s |
| JS 桥接测试 | **7 / 7 通过** | 每个 12~38 ms |

跑测期间整机负载 1.5~2.1，CPU 封装温度峰值 52 °C，风扇未升档 —— **全程未影响桌面交互**。

**17 个失败全部由本项目的改动引起。** 为分清责任，做了四层对照，基线用 `git worktree add --detach` 从仓库 HEAD 拉出的一份**未改动工作树**：

| 层 | 改动 | 失败数 |
|---|---|---|
| ① 纯上游 | 无 | **0** |
| ② + 重编码的 106 个素材 | 640×360/24fps → 320×180/72fps | **3** |
| ③ + 画布常量 | 画布尺寸与帧间隔常量同步调整 | **13** |
| ④ + 补丁中其余代码 | 新增函数与挂载点 | **17** |

其中**只有补丁代码自身引起的失败为 4 个**：

- 架构行数预算测试：主窗口文件增长至 4667 行，超出预算 4648 行（本次补丁 +19 行）；
- 双实例偏移避让测试 1 个；
- 性能统计测试 1 个（缺少 `rebuild.mask` 时段记录）—— 这是「无帧守卫」引入的**行为变更**，需选择补回记账或修改测试口径；
- 会话结束守卫测试 1 个。

**结论：该补丁不能直接向上游提交。** 17 个失败中，13 个的断言把旧画布尺寸与帧间隔写死，改动它们等于改写测试意图而非修复缺陷；余下 4 个需要单独处理。

**三、上游同步（v0.1.6 → v0.1.7）。**

上游在该日发布了新版本，其中一项与本地运行时直接相关（角色美术重绘）。逐文件对照结果：

| 文件 | 变化 |
|---|---|
| 骨骼/网格渲染脚本 | **完全一致** —— 网格渲染器无需改动 |
| 角色几何定义 | 29,281 B → 32,199 B，差异 55 行 |
| 角色模型清单 | 8,830 B → 8,289 B |
| 部件数 | **16 → 25** |
| 贴图数 | 16 → 25 |

新增 9 个部件（眉毛、双眼皮褶线、后脑回卷、前后垂发、鬓发、两种姿态的腰部蝴蝶结、远端手臂）。本地移植新增了对应的形变器与开关，并保持向后兼容：旧版素材缺少相关部件时自动跳过。

**移植正确性以数值而非目视验证**（直接打印形变量并与上游定义表逐项比对）：

```
眉毛外端 / 内端
  neutral     外端  0.00   内端  0.00   内−外  0.00   平行
  surprised   外端 -1.14   内端 -1.14   内−外  0.00   整条抬起
  angry       外端  0.19   内端  1.03   内−外 +0.83   皱眉 ✓
  sad         外端 -0.19   内端 -1.03   内−外 -0.83   挑眉 ✓

眼皮褶线（仅眯眼时下移）
  neutral     → 褶线 0.00 / 0.00   透明度 1.00
  heart-eye   → 褶线 0.00 / 0.00   透明度 0.00   ← 非睁眼型，整条淡出
  sad         → 褶线 4.97 / 4.61   透明度 1.00   ← 眯眼，褶线下移
  surprised   → 褶线 0.00 / 0.00   透明度 1.00   ← 睁大，不下移
```

**渲染成本影响（同机、同脚本，仅替换素材版本，离屏渲染，8 次 `thread_time` 均值）：**

| 网格步进 | 网格格数 | 旧版 ms/帧 | 新版 ms/帧 | 变化 |
|---|---|---|---|---|
| 1 | 547 → 764 | 35.85 | 46.29 | **+29%** |
| 2 | 148 → 204 | 12.52 | 16.31 | **+30%** |
| 3 | 81 → 122 | 8.47 | 12.62 | **+49%** |

上涨的原因是**要画的部件更多**（16→25，网格格数上涨 40~50%），而非单格变慢。要回到原有开销，调高网格抽稀参数即可（例如该版本使用 4 而非 3）。

**端到端复测（上游新内核 + 本地原生宿主 + 新素材）：**

```
进程数        3
内存          RSS 283 MB（PSS 206 MB）
CPU 四段      53 / 49 / 45 / 46 %   → 平均 48.5%
温度          CPU 35 °C / PCH 46 °C
每次重绘       13 ~ 28 ms
```

高于本周期早先记录的 43.4%，原因是**素材变重（+49% @网格步进 3）且该轮角色持续走动**，不是同步过程引入的开销。

**四、渲染加速的复核（详见 §3.6）。** 本阶段对一项加速方案做了真程序验证，推翻了其先前声称的「像素等价」，并给出了一个真正逐位一致的保守版本。

**EN**

**1. Repository consolidation and redaction.**

Work scattered across the development machine was partitioned into 8 repositories — 6 holding project code and documents produced in this period, 2 archival. All were created private initially.

Automated redaction runs in three passes:
1. **File-name blacklist** (passwords, keys, tokens, `.env`, private keys, …) — matching files never enter the repository;
2. **Credential pattern matching** (API-key shapes, platform tokens, private-key headers, secret assignment statements, network authentication credentials, …) — matches are replaced by placeholders;
3. **Personal and machine information** (user name, absolute paths, hostname, intranet address, device names, order numbers, nicknames) — replaced by placeholders such as `<user>` / `<intranet-ip>` / `<hostname>`.

Final review: a full scan across all 8 repositories found **0 credential hits**; 62 user identifiers, 14 intranet addresses and 3 hostnames were replaced. Each repository carries its own redaction note.

**2. Full test run and failure attribution.**

For the project with the largest diff, the complete test suite was run under controlled conditions (reduced CPU and I/O priority):

| Suite | Result | Time |
|---|---|---|
| Python unit tests | **2887 passed / 17 failed / 21 skipped** (2925 total) | 263 s |
| JS bridge tests | **7 / 7 passed** | 12–38 ms each |

During the run, machine load stayed between 1.5 and 2.1, CPU package temperature peaked at 52 °C, and the fan never stepped up — **desktop interaction was unaffected throughout**.

**All 17 failures were caused by this project's changes.** To attribute responsibility, a four-layer comparison was performed against a baseline produced by `git worktree add --detach` from the repository HEAD — an **untouched working tree**:

| Layer | Change | Failures |
|---|---|---|
| ① pristine upstream | none | **0** |
| ② + the 106 re-encoded assets | 640×360/24fps → 320×180/72fps | **3** |
| ③ + canvas constants | canvas size and frame-interval constants adjusted to match | **13** |
| ④ + the rest of the patch | new functions and mount points | **17** |

Of these, **only 4 are attributable to the patch code itself**:

- an architectural line-budget test: the main window file grew to 4667 lines against a budget of 4648 (this patch added 19 lines);
- one dual-instance offset-avoidance test;
- one performance-instrumentation test (a missing `rebuild.mask` segment record) — a genuine **behaviour change** introduced by the frameless guard; it requires either restoring the accounting or amending the test's expectations;
- one session-end guard test.

**Conclusion: this patch cannot be submitted upstream as-is.** Of the 17 failures, 13 assert against the old canvas size and frame interval, so amending them would rewrite the tests' intent rather than fix a defect; the remaining 4 need individual treatment.

**3. Upstream synchronisation (v0.1.6 → v0.1.7).**

Upstream published a new version that day, one change of which bears directly on the local runtime (a character art redraw). File-by-file comparison:

| File | Change |
|---|---|
| Skeleton / grid rendering script | **identical** — the grid renderer required no change |
| Character geometry definition | 29,281 B → 32,199 B, 55 lines different |
| Character model manifest | 8,830 B → 8,289 B |
| Part count | **16 → 25** |
| Texture count | 16 → 25 |

Nine new parts were added (eyebrows, eyelid creases, a rear hair curl, front left/right hair strands, sidelocks, waist bows in two poses, and a far-arm segment). The local port gained corresponding deformers and switches, and remains backward compatible: when older assets lack the relevant parts, they are skipped automatically.

**Port correctness was verified numerically rather than visually** (deformation values printed directly and compared item by item against the upstream definition table):

```
Eyebrow, outer end / inner end
  neutral     outer  0.00   inner  0.00   inner−outer  0.00   parallel
  surprised   outer -1.14   inner -1.14   inner−outer  0.00   whole brow raised
  angry       outer  0.19   inner  1.03   inner−outer +0.83   frown ✓
  sad         outer -0.19   inner -1.03   inner−outer -0.83   raised ✓

Eyelid creases (drop only when the eye narrows)
  neutral     → crease 0.00 / 0.00   opacity 1.00
  heart-eye   → crease 0.00 / 0.00   opacity 0.00   ← not an open-eye type; fades entirely
  sad         → crease 4.97 / 4.61   opacity 1.00   ← narrowed eye; creases drop
  surprised   → crease 0.00 / 0.00   opacity 1.00   ← widened eye; no drop
```

**Rendering cost impact (same host, same script, only the asset version changed; offscreen; mean of 8 `thread_time` samples):**

| Grid step | Grid cells | Old ms/frame | New ms/frame | Change |
|---|---|---|---|---|
| 1 | 547 → 764 | 35.85 | 46.29 | **+29%** |
| 2 | 148 → 204 | 12.52 | 16.31 | **+30%** |
| 3 | 81 → 122 | 8.47 | 12.62 | **+49%** |

The increase is caused by **drawing more parts** (16 → 25, grid cell count up 40–50%), not by any individual cell becoming slower. Restoring the previous cost is a matter of raising the grid decimation parameter (e.g. 4 instead of 3 for this version).

**End-to-end re-measurement (new upstream core + local native host + new assets):**

```
Processes      3
Memory         RSS 283 MB (PSS 206 MB)
CPU segments   53 / 49 / 45 / 46 %   → mean 48.5%
Temperature    CPU 35 °C / PCH 46 °C
Per repaint    13 ~ 28 ms
```

Higher than the 43.4% recorded earlier in this period because **the assets became heavier (+49% at grid step 3) and the character walked continuously during this run** — not because the synchronisation introduced overhead.

**4. Review of a rendering acceleration (see §3.6).** In this phase an acceleration scheme was validated against the real program, overturning its earlier claim of "pixel equivalence", and a genuinely bit-identical conservative variant was produced.

---

### Phase VII — 合成器原生协议适配 · Native Compositor Protocol Adaptation
**2026-09-29**

**中**

**问题。** 该机器原本运行在 X11 会话下，而目标运行时依赖三项 X11 专有机制：窗口自主定位、窗口置顶、以及用一个 X 扩展把透明区域从输入区中挖掉（点击穿透）。需要评估迁移到 Wayland 合成器的可行性，且明确要求**从协议底层入手**。

**关键判断。** 该运行时的渲染器本来就产出「CPU 光栅化完成的 ARGB 位图」，而 Wayland 的共享内存机制本质上就是「共享一块像素缓冲」。因此**渲染层无需任何改动**，需要替换的只有窗口与输入层。

**测试环境。** 本机原本未安装任何 Wayland 合成器。为验证该路径，在**不需要 root、不安装任何系统包**的前提下搭建了测试床：通过包管理器下载合成器及其依赖的安装包，解压到用户目录直接运行。启动时需覆盖若干项在编译期写死的模块搜索路径。

为此对该合成器库中的一个路径解析函数做了反汇编，确认其环境变量映射机制为：**「模块名=路径」形式的列表，以分号分隔**，且**值过长会被静默丢弃**（内部存在长度上限），因此必须使用短路径（以符号链接缩短）。模块搜索路径本身亦为编译期常量，无法通过常规动态库路径覆盖。此外，默认桌面包会尝试拉起若干同样路径被写死的辅助客户端，故测试床改用不需要辅助客户端的最小外壳。

**协议实现的验证方式。** 不使用任何现成的 Wayland 绑定库，而是以纯 `ctypes` 直接调用 `libwayland-client`；协议接口表在运行时从协议描述文件生成。这类表最容易出现「看着对、实际错位一格」的问题，因此验证方式是：调用协议描述文件的**权威生成工具**产出参考实现，再与本实现逐字段比对 —— 接口名、版本号、请求/事件条数、每条消息的名称与签名、以及每个参数位引用的接口。

**结果：28 个接口逐字段全等，0 处不一致。**

比对过程中确认了三条会实际影响正确性的规则：
1. 消息签名存在**版本前缀**：`since > 1` 的消息在签名最前面附加版本数字；
2. 消息签名存在**可空前缀**：以 `?` 标注，且**不限于对象类型**（可空字符串同样使用）；
3. **协议描述文件不等于线上协议**：某一条注册表绑定请求在描述文件中仅声明 2 个参数，而线上真实签名为 4 个参数（生成工具对此有硬编码特例）。因此核心协议一律直接使用客户端库内建的接口表，只有第三方协议才自行生成 —— 这样也顺带回避了「内建表比可获取的描述文件版本更新」造成的版本代差。

**性能实测（同机、同合成器、同内容，仅改变提交策略）：**

| 模式 | 画面 | 帧率 | 客户端 CPU | 合成器 CPU | 绘制 ms | 提交 ms |
|---|---|---|---|---|---|---|
| 小窗 | 360×360 | 38.6 | 6.6% | 4.0% | 1.04 | 0.39 |
| 全屏层，整屏提交 | 1920×1080 | 31.8 | 10.5% | **21.0%** | 2.65 | 0.41 |
| 全屏层，局部提交 | 1920×1080 | 38.7 | 11.4% | **3.9%** | 2.19 | 0.43 |
| **子表面** | 父面 1920×1080 | 39.4 | **1.2%** | **1.9%** | **0.06** | 0.14 |

三条结论：

1. **Wayland 合成器只为被报告为「已损坏」的区域付出代价。** 同样送出全屏缓冲，整屏提交使合成器占用升至 21.0%，而只提交受影响的小矩形时降至 3.9% —— 与 360×360 小窗模式的 4.0% 基本持平。
2. **在 Wayland 上，输入区裁剪是核心协议而非扩展。** X11 路径依赖一个 X 扩展实现点击穿透，Wayland 的对应请求属于基础协议，性质上更规范。
3. **子表面方案把降级路径的代价压低了约一个数量级。** 做法是：父面全屏、全透明、**提交一次之后不再更新**；宠物挂在子表面上，每帧只上传 360×360，位置由子表面的定位请求变更。每帧实际产出的像素量从 8.3 MB 降至 0.5 MB（16 倍），客户端 CPU 从 11.4% 降至 1.2%，折合每帧 2.95 ms → 0.30 ms。

**该方案的三条实现约束（均会直接导致功能失效）：**
1. 父面必须设置一个**空区域**作为输入区；**不可传空指针** —— 空指针的语义是「恢复为整个表面接收输入」，与本意完全相反；
2. 子表面必须显式切换为**非同步模式**。默认的同步模式要求子表面的状态变更随父面提交生效，而父面在本方案中提交一次后不再更新，因此不同步即永不刷新；
3. 子表面需显式声明置于父面内容之上。

**端到端验证。**

- **画面确实呈现。** 以嵌套方式在真实桌面内启动合成器，让原生协议客户端向其绘制，然后对合成器窗口截图：非黑像素 95132 个（占 18.9%），包围盒 348×348，与生成的图形一致；同时帧率跑满 **60.0 fps**。此项验证是必要的 —— 声明「代价很低」之前必须先排除「根本没有绘制」这一可能。
- **点击穿透确实成立。** 在全屏面上以核心协议请求挖出一个 240×240 的输入区域，其余区域全部穿透；随后向合成器窗口注入真实鼠标点击。结果：区域内 2 次点击全部收到（含指针进入与按键事件），区域外 2 次点击**一个事件都未泄漏进来**。

**平台能力差异（关键约束）。**

| 桌面环境 | 层壳协议（可自主定位 + 置顶） | 结论 |
|---|---|---|
| 桌面环境 A | 支持（源码中存在对应实现） | 与 wlroots 系合成器同等 |
| 桌面环境 B | **明确不支持** | 见下 |

桌面环境 B 的维护侧立场是明确的：窗口堆叠顺序归合成器这一特权组件管理，不提供给应用控制，并建议「如需该能力请使用平台专属接口」—— 而该平台专属接口在桌面环境 B 上并不存在。生态侧的两项佐证：存在专门为桌面环境 B 的合成器打补丁以加入该协议的分支；以及某知名应用直接实现了「无该协议时降级到基础窗口协议」的分支。

因此**不存在一条所有桌面环境都支持的原生协议**。可移植的形态是**按能力分层、运行时探测**，而非按发行版名称分支：

1. **第 1 层** —— 存在层壳协议：原生方案，可自主定位、可常驻置顶、输入区机制不变。覆盖多数 wlroots 系合成器与桌面环境 A。
2. **第 2 层** —— 无层壳协议，但 X11 连接可用（兼容层）：**直接复用现有 X11 实现，无需改动**。自主定位、置顶（EWMH 属性）、输入区裁剪（X 扩展）均可用。覆盖桌面环境 B 及其他提供兼容层的会话。**值得注意的是：在桌面环境 B 上，X11 路径反而优于原生路径。**
3. **第 3 层** —— 两者皆不可用：原生全屏窗口 + 子表面。可显示、可穿透、代价极低，但会占用一个全屏窗口位。

**一条尚未解决的问题域约束。** Wayland 协议中**不存在**「我的窗口位于屏幕何处」这一类事件：窗口配置事件只提供尺寸，表面进入事件只提供输出信息。因此在第 3 层中，若要把宠物放置在屏幕绝对坐标上，父窗口**必须请求全屏**，以获得「原点即 (0,0)、尺寸即屏幕尺寸」这一前提；否则只能得知宠物相对于窗口的位置，而无法得知窗口在屏幕上的位置。

**EN**

**Problem.** The machine originally ran an X11 session, and the target runtime depends on three X11-specific mechanisms: client-side window positioning, always-on-top stacking, and carving transparent areas out of the input region via an X extension (click-through). The task was to assess feasibility of moving to a Wayland compositor, explicitly **starting from the protocol layer**.

**Key observation.** The runtime's renderer already produces a fully CPU-rasterised ARGB bitmap, and Wayland's shared-memory mechanism is fundamentally "share a pixel buffer". Therefore **the rendering layer needs no change at all** — only the window and input layers must be replaced.

**Test environment.** The host originally had no Wayland compositor installed. To validate the path, a testbed was built **without root and without installing any system package**: the compositor and its dependencies were downloaded as packages and extracted into the user's home directory. At startup, several module search paths hard-coded at compile time must be overridden.

To that end a path-resolution function inside the compositor library was disassembled, confirming that its environment-variable mapping is a **semicolon-separated list of `module-name=path` entries**, and that **over-long values are silently discarded** (there is an internal length limit) — hence short paths are mandatory (achieved via symlinks). The module search path itself is a compile-time constant and cannot be overridden through ordinary dynamic-linker paths. Furthermore, the default desktop package attempts to launch several helper clients whose paths are likewise hard-coded, so the testbed uses a minimal shell that requires no helper clients.

**How the protocol implementation was verified.** No existing Wayland binding library is used; `libwayland-client` is called directly through pure `ctypes`, and interface tables are generated at runtime from the protocol description files. Such tables are exactly where "looks right, actually off by one slot" defects hide, so verification consisted of invoking the **authoritative generator** for the protocol description files to produce a reference implementation, then comparing field by field: interface name, version, request/event counts, each message's name and signature, and the interface referenced at each argument position.

**Result: 28 interfaces matched field for field, with 0 discrepancies.**

Three rules with direct correctness impact were established during the comparison:
1. Message signatures carry a **version prefix**: messages with `since > 1` prepend the version digit to the signature.
2. Message signatures carry a **nullable prefix**: marked with `?`, and **not limited to object types** (nullable strings use it too).
3. **The protocol description file is not the wire protocol.** One registry bind request declares only 2 arguments in the description file while its true wire signature has 4 (the generator hard-codes a special case for it). Core protocol therefore always uses the client library's built-in tables; only third-party protocols are generated locally. This also sidesteps version skew, since the built-in tables are newer than the available description files.

**Performance measurements (same host, same compositor, same content; only the commit strategy varies):**

| Mode | Surface | FPS | Client CPU | Compositor CPU | Draw ms | Commit ms |
|---|---|---|---|---|---|---|
| Small window | 360×360 | 38.6 | 6.6% | 4.0% | 1.04 | 0.39 |
| Full-screen layer, full damage | 1920×1080 | 31.8 | 10.5% | **21.0%** | 2.65 | 0.41 |
| Full-screen layer, partial damage | 1920×1080 | 38.7 | 11.4% | **3.9%** | 2.19 | 0.43 |
| **Sub-surface** | parent 1920×1080 | 39.4 | **1.2%** | **1.9%** | **0.06** | 0.14 |

Three conclusions:

1. **A Wayland compositor pays only for regions reported as damaged.** Shipping an identically sized full-screen buffer, full-screen damage raises compositor occupancy to 21.0%, whereas committing only the affected small rectangles drops it to 3.9% — essentially level with the 4.0% of the 360×360 small-window mode.
2. **On Wayland, input-region clipping is core protocol rather than an extension.** The X11 path depends on an X extension for click-through; the corresponding Wayland request is part of the base protocol and is therefore more canonical.
3. **The sub-surface approach cuts the cost of the fallback path by about an order of magnitude.** Method: the parent surface is full-screen and fully transparent and is **committed once and never updated again**; the pet lives on a sub-surface, uploading only 360×360 per frame, with position changed through the sub-surface's positioning request. Per-frame pixel production drops from 8.3 MB to 0.5 MB (16×), client CPU from 11.4% to 1.2%, i.e. 2.95 ms → 0.30 ms per frame.

**Three implementation constraints of this approach (each of which breaks the feature outright):**
1. The parent surface must be given an **empty region** as its input region; a **null pointer must not be used** — the null semantics are "restore input over the whole surface", exactly the opposite of intent;
2. The sub-surface must be explicitly switched to **desynchronised mode**. The default synchronised mode applies the sub-surface's state changes on the parent's commit, and the parent in this design is committed once and never updated, so without desynchronisation it would never refresh;
3. The sub-surface must be explicitly declared above the parent's content.

**End-to-end validation.**

- **The image is genuinely presented.** The compositor was started nested inside the real desktop; the raw-protocol client drew into it; the compositor window was then captured. Non-black pixels: 95,132 (18.9%), bounding box 348×348, matching the generated graphic; the frame rate saturated at **60.0 fps**. This check was necessary — before claiming "the cost is low", the possibility that "nothing is drawn at all" must be excluded.
- **Click-through genuinely holds.** A 240×240 input region was carved out of a full-screen surface using the core protocol request, with all remaining area passing through; real mouse clicks were then injected into the compositor window. All 2 clicks inside the region were received (pointer-enter and button events included); the 2 clicks outside it **leaked not a single event**.

**Platform capability differences (the key constraint).**

| Desktop environment | Layer-shell protocol (self-positioning + always-on-top) | Conclusion |
|---|---|---|
| Desktop environment A | Supported (a corresponding implementation exists in its source) | Equivalent to wlroots-family compositors |
| Desktop environment B | **Explicitly unsupported** | See below |

The maintainers of desktop environment B hold an explicit position: window stacking order is governed by the compositor as a privileged component and is not exposed to applications for control; the stated recommendation is to use platform-specific interfaces — and that platform-specific interface does not exist on desktop environment B. Two ecosystem corroborations: a fork exists that patches B's compositor to add the protocol, and a well-known application ships an explicit "fall back to the base window protocol when it is unavailable" path.

**There is therefore no single native protocol supported by every desktop environment.** The portable shape is **capability tiering with runtime probing**, not branching on distribution name:

1. **Tier 1** — layer-shell protocol present: the native solution, with self-positioning, persistent always-on-top and unchanged input-region behaviour. Covers most wlroots-family compositors and desktop environment A.
2. **Tier 2** — no layer-shell, but an X11 connection is available (compatibility layer): **reuse the existing X11 implementation unchanged**. Self-positioning, always-on-top (EWMH property) and input-region clipping (X extension) are all available. Covers desktop environment B and any session offering a compatibility layer. **Notably, on desktop environment B the X11 path is the better one.**
3. **Tier 3** — neither available: native full-screen window plus sub-surface. It displays, it passes clicks through, it costs very little, but it occupies a full-screen window slot.

**An unresolved domain constraint.** The Wayland protocol has **no** event of the form "here is where your window sits on screen": a window configure event supplies size only, and a surface enter event supplies output information only. Hence in Tier 3, to place the pet at absolute screen coordinates, the parent window **must request full-screen**, obtaining the premise "origin is (0,0), size is screen size". Otherwise only the pet's position relative to the window is known, never the window's position on screen.

---

## 3. 技术指标汇总表 · Performance Metrics Summary

### 3.1 媒体管线 · Media Pipeline

| 指标 / Metric | 数值 / Value | 条件 / Condition |
|---|---|---|
| 流水线墙钟 / Pipeline wall clock | 25.1 s → **18.8 s** | 640×360 / 241 帧 / 带 Alpha → 48 fps |
| 编码耗时 / Encode time | 13.09 s → **5.35 s**（−59%） | `cpu-used=0` → `4` + `realtime` |
| 编码峰值 CPU / Encode peak CPU | 364% → **264%** | 同上 |
| 结构相似性 / SSIM | **0.9964** | 编码参数变更后 |
| 产物体积变化 / Output size delta | **+14%** | 同上 |
| 总 CPU 工作量 / Total CPU work | 68.3 → **40.8 CPU·s**（−40%） | `auto` 档 vs 旧版 |
| 核显相对耗时 / iGPU relative cost | **3.3 ~ 3.5×** | 18.8 s → 约 43 s |
| 核显画质差 / iGPU visual delta | **0.0132 / 255** | 逐像素平均绝对差 |

### 3.2 桌面运行时资源 · Desktop Runtime Resources

| 指标 / Metric | 数值 / Value | 条件 / Condition |
|---|---|---|
| 整机内存（浏览器引擎方案） / Whole-machine memory (browser engine) | 1447 MB → **629 MB**（−57%） | 同口径逐档收敛 |
| 进程数 / Processes | 7 → **2**（−71%） | 同上 |
| 浏览器引擎空白页下限 / Browser-engine blank-page floor | **383 MB** | 单进程 + 全部缓存关闭 |
| 整机内存（原生重写） / Whole-machine memory (native rewrite) | 629 MB → **201 MB** | 核心 117 + 身体 84 |
| 身体侧常驻内存 / Body-side resident memory | ~466 MB → **84 MB** | 同上 |
| 端到端 CPU（原生重写） / End-to-end CPU (native rewrite) | **43.4%**（四段 42/44/44/43） | 真实核心 + 真实开窗 |
| 端到端内存（原生重写） / End-to-end memory | RSS **217 MB** / PSS **190 MB** | 同上 |
| 每帧渲染成本 / Per-frame render cost | **10 ~ 25 ms** | 原生重写 |
| 稳态帧率（交付物） / Steady-state rate (artefact) | **24.0 fps** | 跑满设定上限 |
| 每次重绘 / Per repaint | **18.5 ~ 19.8 ms** | 骨骼贴合 18.3 / 网格出图 20.9 |
| 素材重编码 / Asset re-encode | 640×360@24fps → **320×180@72fps** | 106 个文件 |
| 四格对照 CPU / Four-cell CPU comparison | 36.4% / 65.3% / 22.8% / **14.3%** | 官方配置×原素材 / 官方配置×新素材 / 自有配置×原素材 / **自有配置×新素材** |

### 3.3 系统构建 · System Build

| 指标 / Metric | 数值 / Value |
|---|---|
| 包数量演进 / Package count progression | 248 → 964 → **1349** |
| 根文件系统大小 / Root FS size | **4.1 GB** |
| 镜像 / Image | 8 GB 稀疏，实占 **4.2 GB** |
| 空载内存 / Idle memory | **474 MB**（1 GB 内存 / 2 vCPU / KVM） |
| 开机到图形 / Boot to GUI | **约 11 s** |
| 失败系统单元 / Failed units | **0** |
| 进程数对比 / Process count vs host | 149 vs 330（−55%） |
| 包数对比 / Package count vs host | 1349 vs 2311（−42%） |
| 中文字形 / CJK glyphs | **83** |
| 工具链 / Toolchain | gcc 14.2.0 / Python 3.13.5 / ffmpeg 7.1.5 / Vulkan (llvmpipe) |

### 3.4 音频子系统 · Audio Subsystem

| 指标 / Metric | 数值 / Value |
|---|---|
| 响度 / Loudness | −13.29 → **−17.75 LUFS** |
| 真峰值 / True peak | +1.35 → **−1.91 dBTP** |
| 动态范围 / Dynamic range | 5.10 → 3.90 LU |
| 时长 / Duration | 3468.12 s → 2008.06 s |
| 裁掉静音 / Silence removed | **1460.07 s**（24 分 20 秒） |
| 增益预测误差 / Gain prediction error | **0.01 LU**（预测 −17.76 / 实测 −17.75） |
| 端到端耗时 / End-to-end time | **3 分 15 秒**（渲染 26.3 s） |
| 外挂模型高频抑制 / Plugin HF suppression | **−5.9 dB**（>8 kHz） |
| 外挂模型实时倍率 / Plugin real-time factor | **约 2.7×** |

### 3.5 合成器原生协议 · Native Compositor Protocol

| 指标 / Metric | 数值 / Value |
|---|---|
| 接口表比对 / Interface table comparison | **28 / 28 全等，0 不一致** |
| 小窗模式 / Small window | 客户端 6.6% / 合成器 4.0% / 38.6 fps |
| 全屏整屏提交 / Full damage | 客户端 10.5% / 合成器 **21.0%** / 31.8 fps |
| 全屏局部提交 / Partial damage | 客户端 11.4% / 合成器 **3.9%** / 38.7 fps |
| **子表面 / Sub-surface** | 客户端 **1.2%** / 合成器 **1.9%** / 39.4 fps |
| 子表面每帧像素量 / Per-frame pixels | 8.3 MB → **0.5 MB**（16×） |
| 子表面每帧耗时 / Per-frame cost | 2.95 ms → **0.30 ms** |
| 真实垂直同步下帧率 / Rate under real vsync | **60.0 fps**（帧间隔 17 ms） |
| 画面验证 / Visual verification | 非黑像素 **95132**（18.9%），包围盒 **348×348** |
| 点击穿透验证 / Click-through verification | 区域内 **2** 次全部收到 / 区域外 **0** 次泄漏 |

### 3.6 被复核推翻或修正的结论 · Conclusions Overturned or Corrected by Review

**中**

本节单独列出，因为这些修正比结论本身更有方法学价值。

| # | 原结论 | 复核结果 | 根因 |
|---|---|---|---|
| 1 | 渲染加速方案「像素等价」 | **不成立** | 原比对只覆盖**一帧固定姿势**。补做真实动画（32 帧）验证后，该方案并非逐位一致（累计 43 个像素单通道差 ≥65/255，最大通道差 99）。拆解后确认差异全部来自「用外接矩形裁切代替路径裁切」这一项；保留路径裁切即得逐位一致的保守版本，加速比相应从 1.41× 降至 1.11×。**该版本默认不启用。** |
| 2 | 帧率上限将 CPU 从 110% 降至 52% | **不成立** | 原限帧写法使动画帧循环**完全停止**（实测帧率 0，画面冻结），CPU 因而下降。改为定时器 + 真实帧回调后，实测 21.5 fps（目标 24），画面正常。 |
| 3 | 基准脚本报告「30.897% 像素不同、最大差 255」 | **脚本缺陷** | 该脚本把「现状」与一个**已知不等价**的变体作比较。修正为同时对两个变体分别输出并各自下结论后，数字自洽。**教训：基准脚本的输出必须能追溯到「比的是哪两个对象」，否则比不测更糟。** |
| 4 | 内存下降即代表优化生效 | **反例（见 Phase IV）** | 关闭图形后端后内存与 CPU 双双下降，但画布静默变空。帧率读数与内存指标均无法发现问题。**结论：涉及渲染的改动，验收必须读取像素。** |
| 5 | 日志中没有内存耗尽记录 ⇒ 未发生内存耗尽 | **不成立** | 2026-09-25 事故中内核日志确实无 OOM 记录，原因是日志服务当时已无法落盘。 |

**EN**

This subsection stands alone because these corrections carry more methodological value than the conclusions themselves.

| # | Original conclusion | Review outcome | Root cause |
|---|---|---|---|
| 1 | The rendering acceleration is "pixel-equivalent" | **Does not hold** | The original comparison covered only a **single fixed pose**. Re-validated against real animation (32 frames), the scheme is not bit-identical (43 pixels accumulate a single-channel difference ≥65/255; maximum channel difference 99). Decomposition showed the entire difference stems from "bounding-rectangle clipping replacing path clipping"; retaining path clipping yields a bit-identical conservative variant, with speed-up falling from 1.41× to 1.11×. **That variant is disabled by default.** |
| 2 | A frame-rate ceiling cut CPU from 110% to 52% | **Does not hold** | The original limiter caused the animation frame loop to **stop entirely** (measured 0 fps, frozen image), which is why CPU fell. After switching to timer-driven, genuine frame callbacks, the measured rate is 21.5 fps against a 24 fps target, with correct rendering. |
| 3 | The benchmark script reported "30.897% pixels differ, maximum difference 255" | **Script defect** | The script compared the current state against a variant it **already knew to be non-equivalent**. Corrected to report both variants separately with per-variant conclusions, the figures became self-consistent. **Lesson: a benchmark script's output must be traceable to *which two objects* are being compared; otherwise it is worse than no benchmark.** |
| 4 | Falling memory implies the optimisation worked | **Counter-example (see Phase IV)** | Disabling the graphics backend lowered both memory and CPU, but the canvas went silently blank. Neither the frame-rate readout nor the memory figure could reveal it. **Conclusion: for anything touching rendering, acceptance requires reading pixels.** |
| 5 | No OOM record in the log ⇒ memory exhaustion did not occur | **Does not hold** | In the 2026-09-25 incident the kernel log indeed had no OOM record, because the logging service could no longer persist at that point. |

---

## 4. 备注与免责声明 · Notes & Disclaimer

**中**

**4.1 数据来源。** 本文全部数值来自本周期内实际运行所落盘的日志、采样输出与截图，未做任何事后估算或外推。每张表的测量条件均随表标注。凡涉及「同口径对比」的结论，均指两次测量在同一台机器、同一份脚本、仅改变被测变量的条件下得到。

**4.2 未验证事项（明确列出，不与已验证结论混排）。**
- **macOS 路径**：媒体管线的 macOS 分支从未在真实设备上运行过。
- **非 NVIDIA 图形设备**：独立显卡与集成显卡的对照结论来自同一台机器的两种设备，其他厂商设备未测。
- **层壳协议（Tier 1）从未运行过**：本机不存在支持该协议的合成器，该路径仅有能力探测代码，无实机数据。
- **真实桌面环境下的表现**：第 VII 阶段的验证在嵌入式合成器中完成，未在完整桌面环境下运行；桌面环境 A 与 B 的行为均未在实机确认。Tier 2（X11 兼容层）中关于窗口定位与置顶属性的可用性，是按惯例推断，**未经实机确认**。
- **多显示器与分数缩放**：未测。
- **缓冲管理**：协议原型使用单缓冲，未处理缓冲释放事件；正式实现需要双缓冲。
- **音效、麦克风、菜单、键盘输入、缩放偏好**：桌面宠物运行时的对应功能尚未实现。

**4.3 测量环境。** 开发机为 4 核 8 线程 x86-64 处理器、16 GB 内存、独立 GPU 6 GB 显存 + 集成 GPU、1920×1080@144 Hz 显示。宿主桌面会话为 X11。所有测试均在降低 CPU 与 IO 优先级的条件下进行，结束后确认无残留进程、无残留挂载，且测试过程中未影响前台交互。温度读数均取自运行期间的传感器输出。

**4.4 上游与本文件的边界。** 本记录中提及的第三方项目均以中性称谓指代，其名称、作者与许可证**不在本文件中出现**，以避免任何形式的归属混淆。所有派生作品的原始许可证与版权声明，随对应子项目仓库分别保存并逐字保留，不因本文件而变更。涉及上游协议描述文件的使用，仅限其公开许可范围内。

**4.5 本文件的保密审查。** 本文件在发布前经过自动化扫描，扫描规则覆盖个人标识、凭证、绝对路径、主机名、内网地址与机器指纹（磁盘标识、分区标识、裸设备节点、网络硬件地址、PCI 设备标识等）。与本记录相关的扫描结果为零命中。若读者发现任何应当脱敏而未脱敏的内容，请通过仓库的 issue 渠道反馈，将优先处理。

**4.6 本文档的性质。** 这是一份工程记录，不是规范、不是路线图，也不构成任何形式的承诺。文中的性能数字高度依赖硬件、驱动、素材与配置，**不应被视为通用基准**，仅用于说明本周期内各项改动的相对影响。

**EN**

**4.1 Data provenance.** Every figure in this document originates from logs, sampled output and screenshots persisted during actual runs in this period. No post-hoc estimation or extrapolation was performed. Measurement conditions accompany every table. Any statement of "same-scope comparison" means two measurements taken on the same host, with the same script, varying only the quantity under test.

**4.2 Unverified items (listed explicitly; never mixed into verified conclusions).**
- **macOS path**: the macOS branch of the media pipeline has never been run on real hardware.
- **Non-NVIDIA graphics**: the discrete-versus-integrated comparison uses two devices in the same machine; other vendors' hardware was not tested.
- **The layer-shell protocol (Tier 1) has never been executed**: no compositor on this host supports it. Only capability-probing code exists for that path; there is no hardware data.
- **Behaviour under real desktop environments**: Phase VII validation was performed against a nested compositor, not a full desktop session. The behaviour of desktop environments A and B was not confirmed on real systems. The availability of window positioning and always-on-top properties in Tier 2 (X11 compatibility layer) is inferred from convention and is **not confirmed on hardware**.
- **Multi-monitor and fractional scaling**: not tested.
- **Buffer management**: the protocol prototype uses a single buffer and does not handle buffer-release events; a production implementation requires double buffering.
- **Sound effects, microphone, menus, keyboard input, scale preference**: not yet implemented in the desktop-pet runtime.

**4.3 Measurement environment.** The development machine is a 4-core/8-thread x86-64 processor with 16 GB RAM, a discrete GPU with 6 GB VRAM plus an integrated GPU, and a 1920×1080 @ 144 Hz display. The host desktop session is X11. All tests ran at reduced CPU and I/O priority; afterwards, no residual processes or mounts were present, and foreground interaction was unaffected throughout. Temperature readings are taken from sensors during the runs.

**4.4 Upstream scope and the boundary of this document.** Third-party projects referenced here are named by neutral terms only; their names, authors and licences **do not appear in this file**, in order to avoid any attribution confusion. The original licences and copyright notices of all derivative works are preserved verbatim and stored with their respective subproject repositories, unaffected by this document. Use of upstream protocol description files stays within their public licensing terms.

**4.5 Confidentiality review of this document.** Before publication this file was scanned by automated rules covering personal identifiers, credentials, absolute paths, hostnames, intranet addresses and machine fingerprints (disk identifiers, partition identifiers, raw device nodes, network hardware addresses, PCI device identifiers, and so on). The scan for this document returned zero hits. If a reader finds anything that should have been redacted, please report it through the repository's issue channel; it will be prioritised.

**4.6 Nature of this document.** This is an engineering record — not a specification, not a roadmap, and not a commitment of any kind. The performance figures are highly dependent on hardware, drivers, assets and configuration and **must not be treated as general benchmarks**; they exist solely to characterise the relative impact of the changes made in this period.

---

## 5. 许可协议 · License

**中**

本仓库采用 **MIT 许可证**（详见发布时随附的 `LICENSE` 文件）。

```
MIT License

Copyright (c) 2026 BZYS17Mintstar

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

**范围说明。** 本许可证仅覆盖本仓库自身的内容。本文档所描述的各子项目为**独立仓库**，各自适用其自身许可证；其中派生自第三方上游的部分，其原始许可证与版权声明逐字保留，未被本许可证取代。本文件的描述性内容不构成对任何第三方作品的再许可。

**EN**

This repository is released under the **MIT License** (see the accompanying `LICENSE` file at publication time).

```
MIT License

Copyright (c) 2026 BZYS17Mintstar

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

**Scope note.** This licence covers only the contents of this repository. Each subproject described here lives in its **own repository** under its own licence; portions derived from third-party upstream projects retain their original licences and copyright notices verbatim and are not superseded by this licence. The descriptive content of this document does not relicense any third-party work.
