# 调研：Qt Creator 内嵌终端的实现（libvterm + ConPTY）

- 日期：2026-10-26
- 性质：调研记录，**非决策**——console 路线仍按 D1/D4 的弹窗方案执行，本文不改变任何现行决策与红线。
- 用途：D1 范围说明预留的「GUI 内 ConPTY 虚拟终端」可选增强，日后若评估立项，本文是技术输入（issue #94 的发帖里也已向作者表态「ConPTY 内嵌终端留作日后增强」）。另对上游讨论有参考意义：作者在找「mimic PTY」的代码，而 Qt Creator 的答案是**不 mimic**——换接线方式（前端 spawn + 调试器 attach），见 §5。
- 信息源：qt-creator master（2026-10-26，经 code.qt.io cgit 与 GitHub contents API 逐文件核读；Terminal 模式自哪个版本引入未考）；kafeg/ptyqt README；vim `runtime/doc/terminal.txt`。核读文件清单见 §7。

## 1. 结论

现代 Windows 上「GUI 内嵌终端」的通用配方是两层：**VT 仿真器**
（解析转义序列、维护屏幕 cell 模型）**+ ConPTY**（Windows 10+ 的
伪终端 API）；winpty 只是 ConPTY 不可用的老系统上的 legacy 兜底。
Qt Creator 与 vim 同构，连仿真库都是同一个 libvterm：

| | 仿真器 | Windows pty 层 | 渲染 |
| --- | --- | --- | --- |
| vim | libvterm（内嵌源码） | ConPTY（Win10 1809 起可用，vim 文档注明 Win11 起才算稳定）；老系统 winpty.dll + winpty-agent.exe | vim 自身 grid |
| Qt Creator | libvterm（vendored） | ConPTY / winpty 双后端（ConPTY 需 Win10 ≥1903；可换微软新版 conpty.dll + OpenConsole.exe） | QPainter/QTextLayout 自绘控件 |
| VS Code | xterm.js（自研） | ConPTY | DOM/Canvas |

对本项目最重要的发现是 §5 的**调试接线控制反转**：Qt Creator 里
被调试进程由前端自己拉起（挂起态）、调试器 attach 后放行，gdb
从不 spawn inferior——`-inferior-tty-set` / `new-console` 的两难
在该架构里根本不存在。

## 2. 仿真层：libvterm

- `src/libs/solutions/terminal/terminalsurface.cpp` 直接调用 VTerm
  API：`vterm_new`、`vterm_set_utf8`、`vterm_obtain_screen`、
  `vterm_screen_reset`、`vterm_screen_enable_altscreen`、
  `vterm_screen_get_cell`、`vterm_input_write`、
  `VTermScreenCallbacks`、`vterm_keyboard_*`、`vterm_mouse_*` 等，
  include 即 `vterm.h`。
- libvterm 本体 vendored 于 `src/libs/3rdparty/libvterm`（拷贝入库，
  非 submodule——qt-creator 的 `.gitmodules` 无此项）。
- libvterm 之上自建配套（同目录）：`scrollback`（回滚缓冲）、
  `sixel`（图像解码）、`boxdrawing`、`glyphcache`、`celliterator`、
  `keys`。`TerminalSurface` 只做屏模型（cell 矩阵 + 损伤/光标
  信号），不含绘制。
- 终端插件还实现了 VS Code 的 shell integration 协议（OSC
  633/133），为 bash/zsh/fish/PowerShell/clink 打包集成脚本
  （`src/plugins/terminal/shellintegrations/`）。

## 3. 渲染层：自绘控件

- `TerminalView`（`src/libs/solutions/terminal/terminalview.cpp`）与
  `Core::SearchableTerminal`（`src/plugins/coreplugin/terminal/`
  searchableterminal.{h,cpp}，继承 `TerminalSolution::TerminalView`）：
  QPainter / QTextLayout / QRawFont 自绘，不是 QML，也不是
  QTextEdit。
- 终端插件本体在 `src/plugins/terminal/`：`TerminalWidget`（继承
  SearchableTerminal）持有 `Utils::Process` + `Pty::Data`，实现
  `writeToPty` / `resizePty`；`TerminalPane` 管多 tab；
  `TerminalProcessImpl` 实现 `ProcessInterface`。

## 4. pty 层：libptyqt

- vendored `src/libs/3rdparty/libptyqt`，上游
  <https://github.com/kafeg/ptyqt>（MIT，"Qt/C++ binding for WinPty,
  ConPty and standard Unix Pty"）。
- `Utils::Process`（`src/libs/utils/qtcprocess.cpp`）经
  `PtyQt::createPtyProcess(IPtyProcess::AutoPty)` 创建 pty。`AutoPty`
  模式上游 README 未列（疑为 vendored 时加的自动选择补丁）。
  Windows 后端：ConPTY（README 要求 Win10 ≥1903 / build >18309）、
  winpty 兜底（vendored `3rdparty/winpty`，CMake `if(WIN32)` 才链
  接，需随包 winpty.dll + winpty-agent.exe）；Unix 走标准 POSIX
  pty；Emscripten 构建跳过。
- `src/plugins/terminal/consolehost.cpp`：微软 ConPTY NuGet 包
  （microsoft.windows.console.conpty）下载安装器——拉新版
  `conpty.dll` + `OpenConsole.exe`，经 `Pty::setConsoleHostDirectory()`
  （声明于 `src/libs/utils/processinterface.h`）交给 pty 后端。动机：
  系统自带 conhost「只透传文本」，新版可透传图像（与 §2 的 sixel
  配套）。

## 5. 调试接线：控制反转（对本项目最关键）

`src/plugins/debugger/debuggerruncontrol.cpp` 的 terminalRecipe
（run-in-terminal 的调试会话）：

1. 前端（不是 gdb）用 `Utils::Process` 以 `TerminalMode::Debug` 把
   inferior 以挂起态拉起；
2. 启动后把 `processId()` 与 `applicationMainThreadId()` 写入
   runParameters，调试引擎以 attach 模式接上 pid；
3. `EnginesDriver::kickoffTerminalProcessRequested` →
   `Process::kickoffProcess()` 放行运行；
4. 打断经 `EnginesDriver::interruptTerminalRequested` →
   `Process::interrupt()`（不走 POSIX kill 信号）。

控制信号统一走 `ProcessInterface::sendControlSignal(ControlSignal::...)`
（枚举含 `KickOff` / `Interrupt`，`src/libs/utils/processinterface.h`）。
`Utils::Process::setTerminalMode()` 经 `Terminal::Hooks`
（`src/libs/utils/terminalhooks.cpp`）分发到 terminal 插件的
`TerminalProcessImpl`（内嵌终端）或回退 `ExternalTerminalProcessImpl`
（外部终端窗口）。

未逐行钉死的两点：

- 挂起的具体原语（Windows 侧应为 `CREATE_SUSPENDED`、Unix 侧为
  pre-exec 停止）藏在 `TerminalInterface` / `PtyProcessImpl` 的多层
  委托里，本次核读到 run-control 层为止；
- Windows 下调试会给 inferior 设置 `QT_WIN_DEBUG_CONSOLE=attach`
  环境变量（Qt 运行时相关，语义未深究）。

## 6. 对本 fork 的启示（若日后立项 D1 增强）

1. **不必自研仿真器**：libvterm 是小体量纯 C 库（vim 同款），MIT
   许可与 GPL-3 兼容；ConPTY 是系统 API，直连即可；PtyQt（MIT）
   可选；winpty（MIT）只有要支持 pre-1903 老系统才需要。渲染层
   需要自绘（Qt Creator 的 TerminalView 即参照）。
2. **真正的工作量在启动路径重构**。按 D1「日后若证明想要且成本
   低」的标准衡量——仿真层便宜，接线不便宜。现状（D4）是 Seer
   让 gdb 自己 spawn inferior（`new-console on`）；内嵌终端要求
   控制反转：Seer 自己以挂起态把 inferior 拉到 ConPTY 上 → gdb
   `-target-attach` → 放行。`--exec-and-symbols` / `-exec-run` /
   断在 main / `-exec-arguments` 的时序全部重排，中断路径（D5 的
   `-exec-interrupt` 映射）也要换到 ConPTY 侧。
3. **规模参照**：Qt Creator 为此维护 3rdparty 三个库 +
   solutions/terminal 一整个渲染库 + terminal 插件。Seer 若做，
   合理范围是 libvterm + ConPTY 直连 + 一个自绘 widget，不必引入
   PtyQt 全套。
4. 与 D2 兼容性良好：可全部落在新增文件 + `SeerConsoleWidget` 的
   Windows 分支，Linux 侧零改动。

## 7. 附：本次核读的关键文件

qt-creator master，2026-10-26：

- `src/libs/3rdparty/`：libvterm、libptyqt、winpty（均 vendored 拷
  贝，非 submodule）
- `src/libs/solutions/terminal/`：terminalsurface.cpp（VTerm API 调
  用处）、terminalview.cpp、scrollback / sixel / boxdrawing /
  glyphcache / keys / celliterator
- `src/plugins/terminal/`：terminalwidget.cpp、
  terminalprocessimpl.cpp、consolehost.cpp（ConPTY NuGet 下载器）、
  shellintegration.cpp、terminalpane.cpp
- `src/plugins/coreplugin/terminal/`：searchableterminal.cpp
- `src/libs/utils/`：qtcprocess.cpp（PtyQt 接入、TerminalMode 分
  发）、processinterface.h（`Pty::Data`、ControlSignal 枚举）、
  terminalhooks.cpp、terminalinterface.h
- `src/plugins/debugger/debuggerruncontrol.cpp`：terminalRecipe
  （`TerminalMode::Debug` + pid 交接 + kickoff）
- 外部：kafeg/ptyqt README（Windows 后端版本要求、winpty 部署
  物）；vim `runtime/doc/terminal.txt`（ConPTY 自 Win10 1809 起、
  Win11 起稳定、老系统用 winpty）
