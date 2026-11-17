# SnipView Linux 截图软件技术选型调研

- **项目名称**：SnipView
- **目标平台**：Linux 桌面
- **参考产品**：Snipaste、Pixpin
- **调研日期**：2026-11-17
- **文档目的**：确定首版功能边界、核心技术栈、显示系统架构、分发方式及实施顺序。

## 1. 结论摘要

SnipView 推荐采用以下技术基线：

- **语言与 UI**：C++20、Qt 6 Widgets、QGraphicsView、QPainter。
- **构建与测试**：CMake、Ninja、Qt Test、CTest。
- **X11 后端**：XCB、XShm、XRandR、XFixes、EWMH。
- **Wayland 后端**：Qt D-Bus、XDG Desktop Portal、PipeWire；在支持的合成器上增加 layer-shell 适配。
- **首版分发**：AppImage；后续补充 Flatpak。
- **后期可选模块**：OCR、二维码、长截图、录屏，不进入首版核心依赖。

最重要的架构约束是：截图、标注、复制和保存可以形成较统一的跨桌面体验，但置顶贴图、鼠标穿透、全局快捷键和窗口自动识别在 Wayland 下无法保证所有桌面环境行为一致。平台相关代码必须从一开始就与 UI 和图像编辑核心分离。

Snipaste 官方已经提供 Linux AppImage，并在 2.9 加入 Wayland 截屏支持。因此，SnipView 不宜仅以“支持 Linux”为差异化，应重点考虑开源、离线能力、原生 Wayland 适配、CLI/D-Bus 自动化和清晰的桌面能力降级策略。[cite:1]

## 2. 产品范围与功能优先级

Snipaste 的核心能力包括截图、贴图、标注、历史记录、多屏和高分屏；Pixpin 进一步整合了长截图、OCR、二维码以及 MP4/GIF 录制。[cite:1][cite:2]

| 阶段 | 功能 | 边界与目的 |
|---|---|---|
| P0：核心闭环 | 区域/全屏截图、放大镜、取色、矩形、箭头、画笔、文字、马赛克、撤销重做、复制、保存 | 先保证交互流畅、导出正确和图像坐标一致 |
| P0：差异化核心 | 置顶贴图、缩放、透明度、基础历史记录、多屏和混合 DPI | 形成区别于普通截图工具的核心体验 |
| P1：增强功能 | 文字/GIF 贴图、贴图分组、序号、聚光灯、OCR、二维码 | OCR 和二维码保持独立，不侵入截图核心 |
| P1：高难度功能 | 滚动长截图 | 处理滚动速度、重复区域、动态内容和拼接失败 |
| P2：后期功能 | MP4/GIF 录屏、控件级自动识别、动态窗口贴图 | 平台差异和依赖规模最大，不进入首版 |

窗口和控件自动检测应放在后期。X11 可以遍历窗口树，但 Wayland 没有统一的客户端窗口枚举机制，不能承诺达到 Windows 产品相同的识别能力。

## 3. X11 与 Wayland 能力边界

| 能力 | X11 方案 | Wayland 方案与限制 |
|---|---|---|
| 一次性截图 | XCB `GetImage` 或 XShm | `Screenshot` Portal 最容易接入，但目标与交互由 Portal 后端决定，不能假设完全静默。[cite:3] |
| 自定义选区和放大镜 | 读取桌面后显示应用自己的全屏选区层 | 为实现 Snipaste 式自定义选区，通常需要 `ScreenCast` Portal + PipeWire 获取画面；`Start` 阶段可能出现系统来源选择界面。[cite:4] |
| 全局快捷键 | `XGrabKey`，并处理快捷键冲突 | 使用 `GlobalShortcuts` Portal 并运行时探测能力；建议快捷键不一定被接受，也可能出现配置 UI。需要保留桌面快捷键调用应用 CLI/D-Bus 的回退方案。[cite:5] |
| 置顶贴图 | EWMH `_NET_WM_STATE_ABOVE`、`STICKY` 等；穿透可使用 XFixes 输入区域 | 在支持的合成器上使用 layer-shell；其他环境回退为普通窗口。不能跨 GNOME、KDE、Sway、Hyprland 保证一致的置顶、跨工作区和穿透行为。[cite:6][cite:7] |
| 多屏与缩放 | RandR 获取输出几何，自行处理 DPI | 保存每个输出的逻辑几何、缩放、旋转和 PipeWire 流像素尺寸，不能假设存在统一坐标、统一缩放的虚拟桌面位图。[cite:4][cite:8] |

Portal 解决的是授权和屏幕内容获取，不等于提供 Windows 式桌面控制能力。

应用应在运行时维护能力表，并据此启用、降级或隐藏功能：

```text
supportsSilentCapture
supportsCustomRegionSelection
supportsGlobalHotkey
supportsAlwaysOnTop
supportsClickThrough
supportsWindowEnumeration
supportsPerOutputCapture
```

## 4. 语言与 GUI 技术栈比较

| 方案 | 优势 | 主要代价 | 结论 |
|---|---|---|---|
| C++20 + Qt 6 Widgets | `QImage/QPainter` 适合图像处理；`QGraphicsScene/QGraphicsView` 提供图元、缩放、旋转和事件体系；剪贴板、托盘、多屏和高分屏 API 完整。[cite:8] | Qt 运行时和插件需要随应用分发；需要提前确定许可证路线 | 首选，适合完整产品 |
| Rust + GTK4 | Rust 内存安全；GTK 在 Linux 桌面集成自然；已有 Rust bindings。[cite:10] | 标注画布、贴图和各显示后端仍需大量自行实现 | 适合明确偏好 Rust、偏 GNOME/GTK 的项目 |
| Rust + Slint | UI 轻量，支持 Rust/C++；Winit 后端可接入 X11/Wayland。[cite:11] | 复杂图像编辑器、平台窗口行为和 Portal/PipeWire 集成工作量更大 | 更适合轻量截图工具 |
| Tauri + Web 前端 | 设置页、历史记录等普通 UI 开发方便 | Linux 依赖 WebKitGTK；底层截图、快捷键和贴图仍需原生实现，并需维护 JS/Rust 两层。[cite:12] | 不建议作为主架构 |
| Electron | Web 生态完整 | 运行时体积较大，也不能回避底层桌面集成 | 不推荐 |

### 4.1 Qt Widgets 与 QML 的选择

SnipView 的核心界面是图像选区和标注画布，因此首版优先 Qt Widgets：

- **选区覆盖层**：QWidget + QPainter。
- **标注编辑器**：QGraphicsScene/QGraphicsView，每个矩形、箭头、文字都是可编辑图元。
- **撤销重做**：QUndoStack。
- **最终导出**：将矢量标注和底图渲染到 QImage。
- **设置页**：继续使用 Widgets，首版不混用 QML。

标注文档保存为矢量对象和图像像素坐标，不直接保存窗口逻辑坐标；显示时再映射到 Qt 的设备无关像素。Qt 官方明确提醒，多屏逻辑几何不能简单视为连续物理像素桌面。[cite:8]

## 5. 推荐模块架构

| 模块 | 推荐技术 | 设计要点 |
|---|---|---|
| 构建 | CMake、Ninja、C++20 | 保持较老但仍受支持的发行版可构建 |
| 截屏抽象 | `CaptureBackend` | 统一返回图像、输出几何、缩放和光标信息 |
| X11 后端 | XCB、XShm、XRandR、XFixes、EWMH | 主要功能不依赖 Xlib |
| Wayland 后端 | Qt D-Bus、XDG Portal、PipeWire | 处理拒绝授权、会话失效、token 失效和多流 |
| 快捷键抽象 | `HotkeyBackend` | 分离 X11 原生快捷键、Portal 和桌面快捷键回退 |
| 贴图抽象 | `PinWindowBackend` | 分离 X11、layer-shell 和普通窗口降级实现 |
| 选区界面 | Qt Widgets、QPainter | 每个输出一个覆盖层，避免假设全局窗口坐标 |
| 标注 | QGraphicsView、QUndoStack、JSON 文档 | 非破坏性编辑，导出时再扁平化 |
| 配置与历史 | QSettings、QStandardPaths、JSON + 图像文件 | 首版无需引入数据库服务 |
| OCR | 独立 worker 进程 | 中文/英文可选 RapidOCR 模型 + ONNX Runtime；轻量方案可选 Tesseract。[cite:13] |
| 二维码 | ZXing-C++，可选模块 | 不进入截图核心依赖 |
| 长截图 | 独立拼接模块，按需引入 OpenCV | 避免核心应用依赖整个 OpenCV |
| 录屏 | 后期验证 Qt Multimedia `QScreenCapture/QMediaRecorder` | 当前文档仍把 Wayland 支持标为 experimental，且依赖 FFmpeg 后端和系统来源选择界面。[cite:9] |
| 测试 | Qt Test、CTest + 真实桌面矩阵 | 单元测试不能替代 GNOME、KDE、wlroots 实测 |

核心版本只保留 Qt、XCB、D-Bus/Portal 和 PipeWire 等必要依赖，不在首版同时引入 ONNX、OpenCV、FFmpeg 或 Python 运行时。

### 5.1 核心数据流

```text
快捷键 / 托盘 / CLI
        ↓
CaptureBackend
        ↓
每输出选区覆盖层
        ↓
矢量标注文档 + 原始图像
        ↓
复制 / 保存 / 贴图 / 历史记录
```

### 5.2 多屏坐标原则

- 原始截图和标注统一使用图像物理像素坐标。
- Qt UI 使用设备无关像素，并在窗口边界完成坐标转换。
- 每个输出分别记录逻辑几何、缩放、旋转和流像素尺寸。
- 优先使用 QWindow 的 devicePixelRatio，而不是假设所有屏幕共享一个缩放值。
- 显示器热插拔、旋转和缩放变化后重新建立映射，不缓存固定虚拟桌面位图。

## 6. 打包、分发与许可证

### 6.1 首发 AppImage

AppImage 符合单文件、免安装目标。Qt 库和 Qt 插件需要随包部署；官方建议捆绑不能假定为系统基础组件的依赖，并在较老的受支持系统上构建。[cite:14]

首发建议包含：

- 优先提供 x86_64 包；
- `--capture`、`--paste`、`--pin <file>` 等 CLI 参数；
- `.desktop`、图标和用户可选择的登录自启动；
- 无需安装即可运行的版本。

### 6.2 后续 Flatpak

Flatpak 适合进入 Flathub，但沙箱默认限制宿主资源访问，截图等能力通过 Portal 获取。全局快捷键、贴图、后台运行和 X11 回退需要单独验证，不能假设 AppImage 行为可以完全平移。[cite:14]

### 6.3 许可证决策

Qt 提供商业许可和 LGPL/GPL 开源许可，但并非所有模块都采用 LGPL。若计划闭源收费，应在立项时确定使用商业 Qt，还是严格满足相应 LGPL 义务。参考或直接复用其他开源项目代码前，也必须分别检查其许可证。[cite:15]

## 7. 开源项目参考

| 项目 | 主要价值 | 注意事项 |
|---|---|---|
| KSnip | Qt 截图、标注、贴图及 OCR 插件，可重点研究 kImageAnnotator 和平台抽象。[cite:17] | GPL，闭源项目不能直接复制实现 |
| Flameshot | 选区交互、贴图、CLI/D-Bus 和 Wayland 适配经验。[cite:16] | GPL；Wayland issue 可作为兼容性测试清单 |
| KDE Spectacle | KDE 截图后端、PipeWire、Portal 与桌面集成。[cite:18] | KDE Frameworks 依赖较多，适合参考架构而非整体引入 |
| Satty | Rust/GTK4 标注编辑器、可再次编辑的标注对象、layer-shell 多屏经验。[cite:19] | 适合 Rust/GTK 路线参考 |
| Swappy | 轻量编辑器及 stdin/stdout 管道式解耦。[cite:19] | 适合参考捕获工具与编辑器分离 |
| grim/slurp/wayshot | wlroots 截屏、选区和低层 DMA-BUF 路径。[cite:20] | 主要适用于 wlroots，不是 GNOME/KDE 通用替代方案；各组件许可证分别核对 |

## 8. 分阶段实施路线

### 阶段 0：平台验证原型

在正式开发大量 UI 前完成以下验证：

1. Wayland 下通过 ScreenCast + PipeWire 取得帧并转换为 QImage。
2. GNOME、KDE、Sway/Hyprland 上分别显示选区层和贴图窗口。
3. 验证 GlobalShortcuts Portal，以及桌面快捷键调用应用 CLI 的回退方案。
4. 验证双屏、不同缩放、竖屏和屏幕热插拔时的选区坐标。

这些验证决定核心交互能否成立，优先级高于 OCR 和长截图。

### 阶段 1：核心闭环

实现截图 → 选区 → 标注 → 复制/保存 → 贴图 → 历史记录，同时提供 X11 和 Wayland 后端，但允许 Wayland 按运行时能力降级。

### 阶段 2：增强贴图与离线识别

加入 GIF/文字贴图、分组、OCR 和二维码。OCR 模型可单独下载，避免扩大核心安装包。

### 阶段 3：长截图与录屏

长截图和录屏分别作为独立模块开发，不反向改变已经稳定的截图核心。

## 9. 测试与发布门槛

建议将桌面环境划分为明确的支持等级：

- **一级支持**：GNOME、KDE 的 Wayland 环境，以及确定支持的主流 X11 桌面。
- **二级支持**：Sway、Hyprland 等 wlroots 环境。
- **其他环境**：尽力兼容，不作一致体验承诺。

每次发布至少覆盖：

- 区域截图、复制、保存和贴图主流程；
- Portal 授权接受、拒绝和会话失效；
- 全局快捷键冲突及回退方案；
- 多屏、混合缩放、竖屏、热插拔；
- 剪贴板图片在其他应用中的粘贴；
- AppImage 在目标发行版上的干净环境启动；
- 不支持某项能力时，UI 是否正确降级而非静默失败。

## 10. 最终技术决策

SnipView 首版技术清单确定为：

> C++20、Qt 6 Widgets、QGraphicsView、QPainter、CMake、XCB、Qt D-Bus、XDG Desktop Portal、PipeWire、Qt Test、CTest、AppImage。

首版核心不引入 Electron、Python、OpenCV、ONNX 和录屏框架。OCR、二维码、长截图和录屏采用后续模块化路线。架构设计必须优先解决 Wayland 能力差异和多屏坐标问题，而不是仅完成一个能在单一桌面运行的截图界面。

## 11. 全 AI 编码的开发环境准备

项目完全由 AI 编码并不意味着可以省略 Linux 开发环境。Portal、Wayland、全局快捷键和贴图行为必须在真实桌面中运行验证。推荐准备一套真实 Linux 桌面、Qt/C++ 工具链和可自动执行的测试体系。

### 11.1 开发机与测试机

推荐组合：

- **主开发环境**：Ubuntu 24.04 LTS，GNOME Wayland。
- **第二测试环境**：KDE Plasma 6 Wayland，可使用另一台机器、双系统或完整桌面虚拟机。
- **后续兼容性环境**：Sway 或 Hyprland。
- **版本控制**：Git，可选 GitHub/GitLab 用于备份和 CI。

环境边界：

- WSL2/WSLg 不能作为唯一测试环境，它不能代表真实 GNOME/KDE 的 Portal、快捷键和窗口行为。
- Docker 适合编译和单元测试，不适合验证截图、贴图和桌面快捷键。
- 源码放在 Linux 原生文件系统，例如 `~/src/SnipView`，不要放在 `/mnt/c` 等 Windows 挂载目录中编译。
- Windows 中的项目目录可用于保存文档，并通过 Git 同步到 Linux 开发环境。

### 11.2 基础编译工具链

建议准备：

- GCC 或 Clang；
- CMake、Ninja、pkg-config；
- Git、GDB；
- Qt 6 Widgets、D-Bus、Test；
- XCB 开发库；
- PipeWire 开发库；
- clang-format、clang-tidy。

Ubuntu 可安装以下基础包：

```bash
sudo apt update

sudo apt install \
  git build-essential cmake ninja-build pkg-config \
  clang clang-format clang-tidy gdb \
  qt6-base-dev qt6-base-dev-tools \
  qt6-tools-dev qt6-tools-dev-tools \
  libxcb1-dev libxcb-image0-dev libxcb-shm0-dev \
  libxcb-randr0-dev libxcb-xfixes0-dev libxcb-keysyms1-dev \
  libpipewire-0.3-dev libspa-0.2-dev
```

Portal、PipeWire 和相应的 GNOME/KDE Portal 后端通常随桌面环境安装，不建议首版自行替换系统组件。Qt Multimedia、OpenCV、ONNX Runtime 等暂时不用安装。

### 11.3 AI 编码工具环境

AI 工具需要具备以下能力：

1. 能读写 Linux 中的 SnipView 源码目录。
2. 能执行 Linux 终端命令，完成配置、编译、测试和查看日志。

编辑器可选择 Trae 或 VS Code，配合 clangd、CMake Tools；Qt Creator 可作为可选的 Qt 调试工具，不必作为主力 AI 编辑器。

建议由 AI 在项目初期建立：

- `CMakePresets.json`：统一 Debug/Release 配置。
- `compile_commands.json`：供 clangd 和静态分析使用。
- `.clang-format`：统一代码格式。
- `.clang-tidy`：基础静态检查。
- 编译警告配置：例如 `-Wall -Wextra -Wpedantic`。
- AddressSanitizer/UndefinedBehaviorSanitizer 测试构建。
- 一键构建和测试命令，避免每次猜测构建方式。

### 11.4 面向 AI 的开发约束

相比 IDE，以下材料对全 AI 编码更重要：

- 产品功能和首版不做事项；
- 技术选型及模块边界；
- 每个任务的验收条件；
- 统一构建、测试和格式化命令；
- 小步提交、失败不得跳过测试等开发规则；
- 支持桌面和降级能力矩阵。

本文档可作为 AI 的技术约束输入，后续再补一份面向 AI 的项目开发规范。

### 11.5 三层测试环境

| 层级 | 用途 | 推荐方式 |
|---|---|---|
| 自动测试 | 标注模型、坐标转换、配置和图像处理 | CTest + Qt Test，可无界面运行 |
| CI 构建 | GCC/Clang、Debug/Release、格式和静态检查 | GitHub Actions 或其他 Linux CI |
| 真实桌面测试 | Portal 授权、截图、贴图、快捷键、多屏和缩放 | GNOME、KDE、Sway/Hyprland 实际会话 |

Xvfb、offscreen 可以辅助测试 QWidget 和纯逻辑，但不能替代真实 Wayland 测试。AI 可以自动执行大部分构建和测试，用户仍需参与系统授权弹窗和最终视觉验收。

最低可用组合为：一台 Ubuntu Linux 桌面机、Qt 6、GCC、CMake、Ninja、Git，以及一个能操作该 Linux 项目目录和终端的 AI 工具。准备好后即可从阶段 0 的 Portal/PipeWire 和多屏验证原型开始。

## Sources

- [cite:1] [Snipaste 官方功能与 Linux 下载](https://zh.snipaste.com/)；[Snipaste 更新日志](https://zh.snipaste.com/download.html)
- [cite:2] [Pixpin 功能说明](https://pixpin.cn/docs/getting-started/about-pixpin)
- [cite:3] [XDG Desktop Portal：Screenshot](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.Screenshot.html)
- [cite:4] [XDG Desktop Portal：ScreenCast](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.ScreenCast.html)
- [cite:5] [XDG Desktop Portal：GlobalShortcuts](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.GlobalShortcuts.html)
- [cite:6] [Wayland Protocols：wlr-layer-shell](https://wayland.app/protocols/wlr-layer-shell-unstable-v1)
- [cite:7] [X.Org X11 Protocol](https://www.x.org/releases/current/doc/xproto/x11protocol.html)；[Extended Window Manager Hints](https://specifications.freedesktop.org/wm/latest-single/)
- [cite:8] [Qt Graphics View Framework](https://doc.qt.io/qt-6/graphicsview.html)；[Qt High DPI](https://doc.qt.io/qt-6/highdpi.html)
- [cite:9] [Qt QScreenCapture 及平台限制](https://doc.qt.io/qt-6/qscreencapture.html)
- [cite:10] [GTK4 Rust bindings](https://gtk-rs.org/gtk4-rs/stable/0.7/docs/gtk4/index.html)
- [cite:11] [Slint Backends & Renderers](https://slint.dev/latest/docs/slint/guide/backends-and-renderers/backends_and_renderers/)；[Slint 许可说明](https://slint.dev/faqs)
- [cite:12] [Tauri Linux 系统依赖](https://v2.tauri.app/start/prerequisites/)
- [cite:13] [RapidOCR 文档](https://rapidai.github.io/RapidOCRDocs/latest/en/)；[Tesseract 文档](https://tesseract-ocr.github.io/tessdoc/Home.html)
- [cite:14] [AppImage 设计原则](https://docs.appimage.org/introduction/concepts.html)；[Flatpak 沙箱与 Portal](https://docs.flatpak.org/en/latest/sandbox-permissions.html)
- [cite:15] [Qt Licensing](https://doc.qt.io/qt-6/licensing.html)
- [cite:16] [Flameshot](https://github.com/flameshot-org/flameshot)；[Flameshot Wayland Help](https://flameshot.org/docs/guide/wayland-help/)
- [cite:17] [KSnip](https://github.com/ksnip/ksnip)；[KSnip OCR 插件](https://github.com/ksnip/ksnip-plugin-ocr)
- [cite:18] [KDE Spectacle](https://apps.kde.org/spectacle/)
- [cite:19] [Satty](https://github.com/Satty-org/Satty)；[Swappy](https://github.com/jtheoof/swappy)
- [cite:20] [wayshot](https://github.com/waycrate/wayshot)；[grim](https://sr.ht/~emersion/grim/)；[slurp](https://github.com/emersion/slurp)
