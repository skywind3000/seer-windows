# 上游 PR 工作阶段进度跟踪（pr_plan）

- 用途：**进度跟踪**。把「向上游输出 Windows 移植」这件事切成 6 个
  阶段，每阶段有任务清单与出口条件，状态随进展更新。
- 权威划分：队列内容、提交纪律、提取操作手册、风险应对的**唯一
  权威版本**仍是 `docs/porting/upstream-prs.md`；本文只跟踪「做到
  哪一步」，不复制那边的细则，冲突时以那边为准。
- 换机器接着弄：`git pull origin main` → 看本文「阶段总览」找当前
  阶段 → 按 `upstream-prs.md` §4/§5/§6 执行 → 回来更新本文状态与
  执行记录。构建命令见本机 `AGENTS.local.md` §5/§6 与
  `docs/build-windows.md`。
- 状态取值沿用 `docs/plan.md`：`未开始` / `进行中` / `待复核` /
  `已完成` / `受阻`。

## 0. 基线事实（2026-11-10 核对）

- 作者 epasveer 2026-11-09 回复（comment
  <https://github.com/epasveer/seer/issues/94#issuecomment-6747992508>）：
  "Smaller PR's are better, I think." —— **3-PR 拆分定案**，
  `upstream-prs.md` §4 留的「并成单个大 PR」口子关闭。
- 作者 2026-11-08 表态（comment 6724066182）：src/ 改动已通读、
  "They look good to me"；他会把 PR **并进 #94 的分支**再在 Linux
  构建验证；`docs/` 与 LLM 计划不需要；workflow yml 愿意用我们的。
- 上游现有分支 `94-windows-support`，当前 tip 与上游 `main` 同 sha
  （`6ec2019`）。→ **PR 的 base 分支待与作者确认**（见阶段 0 任务
  0.4）：`upstream-prs.md` §3 目前写的是 `epasveer:main`。
- 本 fork 镜像 `upstream-main` 停在 `8d19e2d`，落后上游；阶段 0 要
  先快进镜像并把上游合进 `main`，PR 才切在当前 tip 上。

## 阶段总览

| 阶段 | 名称 | 分支 | 状态 |
|---|---|---|---|
| 0 | 准备与基线（同步上游、确认 base 分支） | `sync/<date>` | 进行中 |
| 1 | PR#1 路径归一化（试点，跑通全流程） | `pr/path-normalize` | 未开始 |
| 2 | PR#2 Windows 平台块（五合一） | `pr/win-platform` | 未开始 |
| 3 | PR#3 console 分流（旗舰，压轴） | `pr/win-console-split` | 未开始 |
| 4 | Windows 发布 workflow 上游化 + 自动构建 | `pr/win-release-ci` | 未开始 |
| 5 | 收敛与退役判断 | — | 未开始 |

阶段 1→2→3 串行（同文件动刀，见各阶段「前置」）；阶段 3 的**设计
讨论**可与阶段 1/2 并行先行（作者点名想聊 console）。阶段 4 前置是
阶段 1-3 全部合入。阶段 5 在阶段 4 之后。

---

## 阶段 0：准备与基线

状态：进行中

前置：无。

### 任务

- [ ] 0.1 `git fetch upstream`，把镜像快进到上游当前 tip
      （`git fetch upstream main:upstream-main`、
      `git push origin upstream-main`）。
- [ ] 0.2 按 `maintenance.md` §1 全流程把上游合进 `main`：打
      `pre-sync-<date>` 锚点 → `sync/<date>` 分支 merge → 逐条核对
      `invariants.md` 红线 → 双平台回归 → 快进 `main` 并 push →
      `docs/CHANGELOG.md` 记一条。
- [ ] 0.3 同步后复核可上游化 diff 规模（原估约 600 行 / 32 文件），
      确认上游新增提交（ParallelStacks 等）与三个 PR 的文件没有新
      交集；有交集则更新 `upstream-prs.md` §4 的文件级清单。
- [ ] 0.4 与作者确认 PR 的 base 分支：`epasveer:main` 还是
      `epasveer:94-windows-support`。结论写回 `upstream-prs.md` §3
      并追加一条决策（D24）。
- [ ] 0.5 把 2026-11-09 作者「smaller PR's are better」与 base 分支
      结论补进 `upstream-prs.md` §1 背景事实与 §10 执行记录（含
      D24 与 `docs/plan.md` 决策记录表索引行）。
- [ ] 0.6 检查 `origin` 的 fork 关系仍可用于跨仓 PR（仓库已改名
      `seer-windows`）：在 GitHub 上确认能从 `skywind3000:pr/*` 向
      `epasveer` 发起 PR。

### 出口条件

1. `main` 含上游当前 tip，双平台回归通过（WSL2 全量构建 +
   `tests/hellopathnorm` + Windows 构建与抽样会话），`origin/main`
   已推送。
2. `upstream-main` 镜像与上游 tip 一致，本地与 origin 同步。
3. base 分支问题有作者明确答复并落文档。

---

## 阶段 1：PR#1 路径归一化（试点）

状态：未开始

前置：阶段 0 出口条件全部满足。

内容见 `upstream-prs.md` §4 表第 1 行与文件级清单（GdbMonitor 读侧
编码守卫 + `fullname` 记录级重写、SeerGdbWidget 写侧 `toLocal8Bit`、
SeerUtl 两函数 + 22 处调用点、显示层 `toNativeSeparators`、
`tests/hellopathnorm` 两段）。规模约 210 行 + 测试。**这是唯一改变
Linux 行为的 PR**（八进制还原本身修 Linux bug），PR 描述按「带回归
测试的行为改进」写。

### 任务

- [ ] 1.1 fetch 上游，从最新 tip 切 `pr/path-normalize`（§6 步骤
      0-1）。
- [ ] 1.2 提取改动：新文件整取、混合文件（`SeerGdbWidget.cpp`、
      `GdbMonitor.cpp`、`SeerUtl.{h,cpp}`）按 hunk 裁剪，补丁形态
      与 `main` 逐行一致（§5 提取手法）。
- [ ] 1.3 验证四件套（§5）：① WSL2 Linux 构建；② Windows MinGW64
      构建；③ `hellopathnorm` 双平台；④ §4 表验收会话（中文路径
      会话、同一文件不同拼写单 tab、混合大小写断点命中）。记录留
      `.tmp/`，摘要进 PR 描述。
- [ ] 1.4 整理成 1-3 个干净英文提交（引用 issue #94，遵循
      STYLEGUIDE），push `origin pr/path-normalize`。
- [ ] 1.5 开 PR，描述按 §5 模板五段（motivation / what / design
      rationale / verification / Linux 行为说明）。
- [ ] 1.6 review 往返：改动以作者为准；每轮把等价改动以小 commit
      带回 fork `main`；若与既有决策冲突，`decisions.md` 追加记录。
- [ ] 1.7 作者合入后：按 `maintenance.md` §1 小同步（或攒到下一
      release），确认 merge 回 `main` 零冲突；删 `pr/path-normalize`。
- [ ] 1.8 复盘试点：把作者的 review 风格、粒度偏好、验证记录的反
      馈写进 `upstream-prs.md` §10，作为阶段 2/3 的校准输入。

### 出口条件

1. PR 被作者合入上游目标分支。
2. fork `main` 已同步该合入，双平台回归通过。
3. `pr/path-normalize` 已删除，试点复盘已记录。

---

## 阶段 2：PR#2 Windows 平台块（五合一）

状态：未开始

前置：阶段 1 出口条件满足（`SeerUtl.{h,cpp}` 由 #1/#2 共同动刀，
必须等 #1 合入后再从新 tip 切分支，否则 hunk 上下文错位）。若作者
要求并行提交，改为在 #1 分支之上切出并说明依赖。

内容见 `upstream-prs.md` §4 表第 2 行与文件级清单：
`QProcessInfo_win.cpp` 新文件 + `QProcessInfo.cpp` 三处守卫 + CMake
源清单；CMake WIN32 段（RC / gdb 名默认 / 警告守卫）+
`seergdb.rc` / `.exe.manifest` / `.ico` + `.gitignore` 例外；
`QSettings::setDefaultFormat(IniFormat)`；`resolveMonospaceFamily` +
12 处包裹；attach 回填 Program name。规模约 300 行。**排除项**：
`SeerStackTrace.{h,cpp}` 与 `-fno-omit-frame-pointer`（不上游）。

### 任务

- [ ] 2.1 fetch 上游，从最新 tip 切 `pr/win-platform`。
- [ ] 2.2 提取改动，按五个子主题各一提交（进程列表 / 构建与图标 /
      INI 设置 / 默认字体 / attach 回填），便于作者逐块 review。
      注意 `QProcessInfo_win.cpp` 的 `Q_OS_WIN` 守卫必须在任何 Qt
      include 之后（`invariants.md` I1）。
- [ ] 2.3 验证四件套：Linux 构建路径零变化、进程列表行为不变、
      NativeFormat 本就是 INI、字体透传、attach 保持「非空不覆盖」；
      Windows 侧进程列表可用可选中、构建带图标、落 ini、默认
      Cascadia、attach 栈帧干净。
- [ ] 2.4 push + 开 PR（描述分节对应五个子主题，明确 Linux 零行为
      变化）。
- [ ] 2.5 review 往返（同 1.6）。
- [ ] 2.6 合入后小同步、删分支。

### 出口条件

1. PR 被合入；2. fork `main` 同步且双平台回归通过；3. 分支已删。

---

## 阶段 3：PR#3 console 分流（压轴）

状态：未开始

前置：阶段 2 出口条件满足。设计讨论（3.0）不受此限，可提前。

内容见 `upstream-prs.md` §4 表第 3 行：`SeerConsoleWidget.cpp` 四处
守卫、`SeerCommandLogsWidget.cpp` 页隐藏、`SeerGdbWidget.cpp` 剩余
平台块（信号宏占位 / `CREATE_NO_WINDOW` / `new-console on` / 中断映
射）、`SeerMainWindow.cpp` 菜单隐藏。规模约 90 行。Linux PTY 原文
一字不动。

### 任务

- [ ] 3.0 （可提前并行）在 issue #94 与作者过 console 设计：D1/D3/D4
      的论证、「不发 `new-console` 会污染 MI 流」的证伪记录、与
      VS / WinDbg / VS Code 的类比、ConPTY+libvterm 作为日后增强
      （参考 `docs/porting/qtcreator-terminal-research.md`）。结论
      记进 `upstream-prs.md` §10。
- [ ] 3.1 fetch 上游，从最新 tip 切 `pr/win-console-split`。
- [ ] 3.2 提取改动（`SeerGdbWidget.cpp` 剩余 hunk 归本 PR）。
- [ ] 3.3 验证四件套：Windows 终端程序弹窗 / MI 流干净 / GUI 程序
      不弹 / Interrupt 可用；Linux PTY 行为不变（WSL2 构建 +
      console 会话实测）。
- [ ] 3.4 push + 开 PR，design rationale 段带 3.0 的完整论证与 PoC
      证据。
- [ ] 3.5 review 往返（同 1.6）。若作者要求换路线（如 ConPTY），按
      `upstream-prs.md` §9 处理：D1 已预留增强口子，另立决策。
- [ ] 3.6 合入后小同步、删分支。

### 出口条件

1. PR 被合入；2. fork `main` 同步且双平台回归通过；3. 分支已删；
4. 至此约 600 行可上游化资产全部进上游。

---

## 阶段 4：Windows 发布 workflow 上游化 + 自动构建

状态：未开始

前置：阶段 1-3 全部合入（yml 构建的代码得先在上游）。作者
2026-11-08 已表态愿意用我们的 yml 替代他早年试的那份。

内容：把 `.github/windows-release.yml` 送上游（配方见
`docs/build-windows.md` §4：windeployqt + DLL 闭包 + gdb 随包），
让上游能在 GitHub 托管的 `windows-latest` 上自行发官方 Windows 包
——作者没有 Windows 机器，这条正是他缺的能力。

### 任务

- [ ] 4.1 审 yml 的上游适配度：去掉 fork 专属假设（仓库名、release
      命名、tag 约定、`SEER_GDB_NAME` 路径、任何指向
      `skywind3000` 的硬编码），确认在上游仓库语境下可直接跑。
- [ ] 4.2 与作者确认落地形态：直接进上游 `.github/`，还是先在他
      的分支上跑通再合 `main`；确认触发条件（release 发布 +
      `workflow_dispatch` 指定 tag 演练）他接受。
- [ ] 4.3 切 `pr/win-release-ci` 提取 yml（这是唯一带 `.github/`
      的 PR，属 §5「只含 src/ 与 tests/」纪律的**明确例外**，PR
      描述里说明理由）。
- [ ] 4.4 在 fork 上用 `workflow_dispatch` 指向一个 tag 演练一次，
      产物自检（解压可跑、gdb 随包、DLL 闭包完整），记录进 PR
      描述。
- [ ] 4.5 push + 开 PR + review 往返。
- [ ] 4.6 合入后请作者在上游做一次演练（他侧 `workflow_dispatch`），
      确认托管 runner 上绿灯；失败则由我们改 yml 再发小 PR。

### 出口条件

1. yml 进上游并在 GitHub 托管 runner 上成功产出 Windows 包（作者
   侧或我们侧演练均可，但要有成功记录）。
2. 上游具备独立发布 Windows 包的能力，不依赖本 fork。

---

## 阶段 5：收敛与退役判断

状态：未开始

前置：阶段 4 完成。

对应 `upstream-prs.md` §8。终态选项届时另立决策：① fork 退役、
README 指向上游（兑现我们在 issue 的承诺）；② fork 转型为纯发布
渠道；③ 其他。

### 任务

- [ ] 5.1 请作者摘掉 issue #94 的 `wontfix` 标签，以「已合并」告终
      该 issue。
- [ ] 5.2 终态决策：评估三个选项，追加 `decisions.md`（D2x）并在
      `docs/plan.md` 决策记录表加索引行。
- [ ] 5.3 按终态执行：退役则改 README fork 说明块（D20 的块，只改
      该块）指向上游、归档 fork 专属文档；转型发布渠道则保留
      release 流水线、停止 src/ 分叉。
- [ ] 5.4 更新 `AGENTS.md` 硬性约束与 `maintenance.md`：若 fork
      退役，上游同步流程与 `pr/*` 纪律相应作废或改写。
- [ ] 5.5 收尾：`upstream-prs.md` §10 与本文执行记录各记一行终态。

### 出口条件

1. issue #94 关闭且标签清理。
2. 终态决策入库，相关文档（AGENTS.md / maintenance.md /
   upstream-prs.md / README 块）与实际状态一致。

---

## 执行记录（追加，一行一事）

| 日期 | 阶段 | 事件 | 备注 |
| --- | --- | --- | --- |
| 2026-11-10 | — | 本文建立；6 阶段切分定稿 | 依据作者 2026-11-09「smaller PR's are better」确认 3-PR 拆分；上游 tip `6ec2019`，`94-windows-support` 分支存在，base 分支待确认 |
| 2026-11-10 | 0 | 阶段 0 置「进行中」 | 剩：0.1-0.6（镜像快进、上游合入 main、base 分支确认） |
