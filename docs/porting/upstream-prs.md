# 上游 PR 输出计划（upstream PR campaign）

- 状态：已立项（D22），阶段 0 待启动
- 定稿：2026-11-03（基于当日上游状态快照，见 §1）
- 本文档是 PR 路线图的**唯一权威版本**：队列、纪律、操作手册、
  执行记录都在这里。换机器接着弄：`git pull origin main` 拿到
  本文档，从 §10 执行记录确认进度，按 §4 队列继续。构建命令不
  在本文重复——见本机 `AGENTS.local.md`（§5/§6）与
  `docs/build-windows.md`。

## 1. 背景事实（2026-11-03 快照）

- **作者已同意**：epasveer 于 2026-11-02 在 issue #94 回复
  （<https://github.com/epasveer/seer/issues/94#issuecomment-6591567389>）：
  "Yes, absolutely. PR's are welcomed and your name is attributed
  through the process."，并表示会深入读 console 方案（他的
  "original grief"——他此前不知道 Windows 是弹独立控制台而非
  pty）。issue 仍 open + `wontfix` 标签（PR 全合后再请作者摘）。
- **我们的承诺**（2026-10-26 在 issue 的公告）：PR 严格藏在
  Windows 守卫后；若被接受，fork 退役转向单一上游项目。
- **可上游化资产**：`main` 对 `upstream-main` 的 src/ diff 为
  32 文件 +856/-64，扣除 fork 专用件（`SeerStackTrace.{h,cpp}`
  +257、`docs/`、CI、README 块）后约 **600 行**。D2 的最小 diff
  架构（平台差异进新文件、`Q_OS_WIN` 守卫、调用点零 #ifdef）
  当初就是为此准备的，现在兑现。
- **上游要求**：`CONTRIBUTING.md` 全文一句话——走 PR、遵循
  STYLEGUIDE。无 CLA。
- **上游活跃**：2026-11-03 上游 tip `67abf83`，较本 fork 镜像
  （`8d19e2d`）领先 10 提交（ParallelStacks 主题，与 fork 改动
  零文件交集）。每切新 PR 分支前必须重新 fetch。

## 2. 总体策略：四个阶段

1. **阶段 0 准备**：同步上游进 `main`（maintenance.md §1 全
   流程）、issue #94 回帖告知拆分计划、（本文档即第三项）。
2. **阶段 1 试点**：发 PR#1（队列里接受度最高的那个），目的是
   跑通「切分支 → 提取 → 验证 → PR → review → merge」全流程、
   校准作者的 review 风格。
3. **阶段 2 主体序列**：8 个 PR 串行提交（§4），同时打开的 PR
   不超过 2 个；console 旗舰（#8）压轴，但其**设计讨论**可在
   issue 里先行——作者点名想聊。
4. **阶段 3 收敛与退役判断**：每个上游 merge 后小同步；全部合
   入后做终态决策（fork 退役 / 转型纯发布渠道 / 连 Windows 发布
   工作流一并送上游）。

## 3. 分支纪律（`pr/*`，D22）

- 每个 PR 一个短命 `pr/<topic>` 分支，**一律从 `upstream-main`
  （刚快进的上游 tip）切出**，不从 `main`/`develop` 切——它们
  领先镜像 32 个提交，全是 fork 专属内容（docs、CI、README 块），
  带进 PR 没法 review。
- 推送到 **origin**（`upstream` remote 禁推是 D13 的刻意设计），
  GitHub 上从 `skywind3000:pr/<topic>` → `epasveer:main` 发 PR
  （仓库改名 seer-windows 后 fork 关系仍在，PR 通道可用）。
- `main`/`develop` 的日常流程不动；`pr/*` 与 `fix/*`、`sync/*`、
  `pre-sync-*` 命名空间互不干扰。上游 merge 或放弃即删分支。
- **review 意见的改动以作者为准**：改 pr 分支的同时把等价改动
  带回 fork `main`（小 commit），保持两侧收敛，将来 merge 回来
  零冲突。
- 中途上游又动了：先 `git fetch upstream`，必要时重切/rebase。

## 4. PR 队列（8 个，接受度 × 基础性双降序）

排序满足四条标准：每个 PR 独立可验收、单 PR 最小改动、作者
接受度降序、基础层在前。总规模约 600 行，平均单 PR 不到 150 行。

| # | 分支 | 内容（决策/红线出处） | 规模 | 独立验收 |
|---|---|---|---|---|
| 1 | `pr/mi-wire-escape` | MI 线缆层：GdbMonitor 读侧编码守卫 + `fullname` 记录级重写（`\NNN` 八进制还原）；SeerGdbWidget 写侧 `toLocal8Bit`（D11 + D8 线缆半；I8）+ hellopathnorm 第一段 | ~100 行 + 测试 | **作者 Linux 可亲验**：跑测试（八进制还原即 Linux bug 修复 + identity 不回归）；我们补 Windows GBK 用例 + 中文路径会话 |
| 2 | `pr/path-normalize` | SeerUtl 两函数 + 键控/命令/显示三层 22 处调用点（D8 剩余；I7）+ hellopathnorm 第二段 | ~110 行 + 测试 | Linux identity 用例；Windows：同一文件不同拼写单 tab、混合大小写断点命中 |
| 3 | `pr/win-process-info` | `QProcessInfo_win.cpp` 新文件 + QProcessInfo.cpp 三处守卫 + CMake 源清单（D6；I4） | ~180 行 | Windows 进程列表可用可选中；Linux 列表行为不变 |
| 4 | `pr/win-build-icon` | CMake WIN32 段（RC/gdb 名默认/警告守卫）、seergdb.rc/.manifest/.ico、.gitignore 例外（D21；I2 部分） | ~60 行 | Windows 构建带图标；Linux 构建路径零变化 |
| 5 | `pr/win-ini-settings` | `QSettings::setDefaultFormat(IniFormat)`（D10；I9） | ~4 行 | Windows 落 ini、宏/脚本目录可用；Linux NativeFormat 本就是 INI |
| 6 | `pr/win-default-font` | `resolveMonospaceFamily` + 12 处包裹（D14） | ~40 行 | Windows 默认 Cascadia；Linux 透传 |
| 7 | `pr/win-attach-backfill` | attach 回填 Program name（D9；I6） | ~16 行 | Windows attach 栈帧干净；Linux 保持「非空不覆盖」 |
| 8 | `pr/win-console-split` | console 守卫 + 页隐藏、`CREATE_NO_WINDOW`、`new-console on`、中断映射、菜单隐藏（D1/D3/D4/D5；I3/I5） | ~90 行 | Windows：终端程序弹窗 / MI 流干净 / GUI 程序不弹 / Interrupt 可用；Linux PTY 原文未动 |

各 PR 的文件级清单（行号锚点引自 invariants.md，那是 2026-10-21
版锚点，以内容为准）：

- **PR#1**：`src/GdbMonitor.cpp`（读侧两处编码守卫 + 记录级
  `fullname="..."` 重写块）、`src/SeerGdbWidget.cpp` 写侧
  `toLocal8Bit` 块（:862-867 一带）、`tests/hellopathnorm/`
  第一段（GdbMonitor 用例：行数、`*stopped` 转换、`^done` 双
  字段、`~` 文本不动、`=` 记录不动、八进制还原——Linux 侧的
  八进制用例即修复本体）。
- **PR#2**：`src/SeerUtl.{h,cpp}` 两函数；调用点 22 处
  （SeerBreakpointCreateDialog 1、SeerEditorWidgetSourceAreas 3、
  SeerEditorManagerWidget 6、SeerGdbWidget 10、
  SeerSourceBrowserWidget 2）；显示层 ~10 处
  `QDir::toNativeSeparators` 与表格 UserRole 约定；hellopathnorm
  第二段（函数用例）。
- **PR#3**：`src/QProcessInfo_win.cpp`（新，168 行；`Q_OS_WIN`
  宏守卫必须位于任何 Qt include 之后——I1 的坑，放前面整个
  文件编译成空翻译单元）、`src/QProcessInfo.cpp`（:14/:24/:126
  三处分流）、`src/CMakeLists.txt` 源清单条目。
- **PR#4**：`src/seergdb.rc`、`src/seergdb.exe.manifest`、
  `src/resources/icons/seergdb.ico`、CMake WIN32 段
  （`enable_language(RC)` + `CONFIG_SEER_GDB_NAME` 平台分支 +
  警告守卫；**不含** SeerStackTrace.cpp 条目与
  `-fno-omit-frame-pointer`——那是排除项）、`.gitignore` 的
  ico 定向例外（上游 `icons/` 规则会挡掉它）。
- **PR#5**：`src/seergdb.cpp` 的 `setDefaultFormat` 块。
- **PR#6**：`src/SeerUtl.{h,cpp}` 的 `resolveMonospaceFamily` +
  12 处默认字体调用点包裹。
- **PR#7**：`src/SeerDebugDialog.cpp`（:513 起的 `Q_OS_WIN` 块）。
- **PR#8**：`src/SeerConsoleWidget.cpp`（:22/:240/:278/:318
  守卫）、`src/SeerCommandLogsWidget.cpp`（:179 页隐藏）、
  `src/SeerGdbWidget.cpp` 剩余平台块（:39-44 信号宏占位、
  :99-103 `CREATE_NO_WINDOW`、:2074-2080 `new-console`、:3956
  起中断映射）、`src/SeerMainWindow.cpp`（:126 菜单隐藏）。

**排除项（不上游）**：`SeerStackTrace.{h,cpp}`（R8 崩溃调查
装备，`-fno-omit-frame-pointer` 有性能代价；日后想送另立项）、
`docs/`、`.github/windows-release.yml`、README fork 块、
`AGENTS*`。

**排序的两处可调点**：① #2 与 #3 可对调——纯按接受度
QProcessInfo（纯新增文件）更无摩擦，本表按「基础优先」把
path-normalize 排前（线缆层之后立刻把路径故事讲完、测试连续）；
若作者节奏显示他更爱看纯新增，对调即可，两者无依赖。② #4
（图标）无风险但对作者价值低，放第 4 位是按基础性（构建配置）
而非接受度。

**PR#1 的破例（唯一）**：它改变 Linux 行为——八进制还原本来就
是修 Linux 的 bug（上游对 `\NNN` 转义从不还原，中文源码路径在
Linux 同样打不开，D11 有 Linux 侧复现记录）。「PR 严格藏在
Windows 守卫后」的承诺对它不适用，取而代之是「带回归测试的
Linux 行为改进」。用它打头阵是刻意的：证明这个 fork 的输出
不只是 Windows 平台块。#2 起「Linux 零变化」纪律恢复。

## 5. 每个 PR 的提交纪律

- **范围**：只含该主题的 `src/` 改动 + 必要的 `tests/`。不带
  `docs/`、`.github/`、README 块、AGENTS、fork 决策编号——上游
  不知道也不需要知道我们的 ADR。
- **验证（提交前必须全过）**：① WSL2 Linux 构建（作者唯一能自
  验的平台，绝不能给他留坑）；② Windows MinGW64 构建（命令见
  本机 `AGENTS.local.md` §6）；③ hellopathnorm 相关段落双平台；
  ④ §4 表中该 PR 的验收会话。PR 描述里**主动给出这些验证记录**
  ——作者测不了 Windows，这是我们最大的可信度来源。
- **提交信息**：英文，引用 issue #94，遵循 STYLEGUIDE。不要
  cherry-pick fork 的 phase 状提交（它们混着 docs 与多主题）；
  每个 PR 手工整理成 1-3 个干净提交。
- **提取手法**：新文件直接 `git checkout main -- src/<file>`；
  混合文件（CMakeLists、SeerGdbWidget、GdbMonitor、SeerUtl 都承
  载多个主题）按 hunk 裁剪——`git diff <上游tip> main -- <file>`
  存 `.tmp/`，手工删掉不属于本 PR 的 hunk 后 `git apply`。补丁
  内容**尽量与 `main` 里已有形态逐行一致**——将来上游合入后
  merge 回 fork 时 git 自动对上，冲突最小。
- **PR 描述模板**：motivation（什么坏/缺）→ what（改了什么）→
  design rationale（为什么这样做；console PR 要带 D1/D3/D4 的
  完整论证与 PoC 证据，包括「不发 new-console 会污染 MI 流」的
  证伪记录）→ verification（上述四项的记录）→ 明确说明 Linux
  行为不变（PR#1 写「改进，附回归测试」）。

## 6. 提取操作手册（命令级）

    # 0) 切新分支前先看上游有没有动
    git fetch upstream
    git log --oneline upstream-main..upstream/main   # 有增量则先快进镜像
    git fetch upstream main:upstream-main

    # 1) 从上游 tip 切分支
    git switch -c pr/<topic> upstream-main

    # 2a) 整文件属于本 PR 的，直接取
    git checkout main -- src/QProcessInfo_win.cpp

    # 2b) 混合文件按 hunk 裁剪
    git diff upstream-main main -- src/CMakeLists.txt > .tmp/pr.patch
    #     手工编辑 .tmp/pr.patch 只留本 PR 的 hunk，然后：
    git apply .tmp/pr.patch

    # 3) 双平台验证（命令见 AGENTS.local.md §6 与
    #    docs/build-windows.md；WSL2 用 .tmp/build-wsl）
    #    + hellopathnorm 相关段 + §4 表中该 PR 的验收会话

    # 4) 提交（英文，1-3 个干净提交）并推 origin
    git add <files>
    git commit
    git push origin pr/<topic>

    # 5) GitHub：skywind3000:pr/<topic> → epasveer:main 开 PR，
    #    描述按 §5 模板写

    # 6) review 有意见：改 pr 分支 + 等价改动带回 main（小 commit）

注意：提取 diff 的 base 永远用**最新的上游 tip**（重切后用新
`upstream-main`），避免把上游已合入的内容重复带上。`SeerGdbWidget.cpp`
会被 #1/#2/#8 三个 PR 分别动刀，每次都按 hunk 裁剪。

## 7. 阶段 0 启动清单

- [ ] 同步上游进 `main`：走 maintenance.md §1 全流程（2026-11-03
      时上游领先 10 提交、ParallelStacks 主题、与 fork 改动零文件
      交集，预期零冲突；双平台回归照跑）。
- [ ] issue #94 回帖：确认接受邀请，附 8-PR 拆分计划（从小到大、
      console 压轴），给作者预期；console 设计讨论可顺势先行。
- [x] 本文档 + D22 + AGENTS.md `pr/*` 分支策略（2026-11-03 完成）。

## 8. 阶段 3：收敛与退役

- 每个上游 merge 后：maintenance.md §1 小同步（或攒到上游
  release 批量做）。补丁形态与 `main` 逐行一致，预期 merge 零
  冲突。
- 全部合入后的终态选项（届时另立决策）：
  ① fork 退役、README 指向上游（对应我们在 issue 的承诺）；
  ② 收官 PR 把 Windows 发布工作流也送上游——`windows-release.yml`
  跑在 GitHub 托管的 windows-latest 上，不需要作者有 Windows
  机器，上游即可发官方 Windows 包；
  ③ fork 转型为纯发布渠道。
- 最后请作者摘掉 `wontfix` 标签、以合并告终 issue #94。

## 9. 风险与应对

- **review 意见与 fork 决策冲突**（最大风险）：以作者为准修改，
  等价改动回带 `main`，`decisions.md` 追加记录调整（ADR 只追
  加）。console 方案若被要求换路线（如 ConPTY），D1 预留了
  「日后可选增强」的口子，`docs/porting/qtcreator-terminal-
  research.md` 有承接调研。
- **上游活跃导致基线漂移**：趁作者热度正高尽快启动；每切新分支
  必先 fetch。
- **Linux 回归**：作者测不了 Windows，错误全靠我们的双平台验证
  拦——§5 的四项验证不可裁剪。

## 10. 执行记录（追加，一行一事）

| 日期 | 事件 | 备注 |
| --- | --- | --- |
| 2026-11-03 | 计划定稿并入库（D22、本文档、AGENTS.md 分支策略） | 阶段 0 剩：同步上游、issue 回帖 |
