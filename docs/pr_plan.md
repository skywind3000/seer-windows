# 上游 PR 工作阶段进度跟踪（pr_plan）

- 用途：**进度跟踪**。把「向上游输出 Windows 移植」这件事切成 6 个
  阶段，每阶段有任务清单与出口条件，状态随进展更新。
- 权威划分：队列内容、提交纪律、提取操作手册、风险应对的**唯一
  权威版本**仍是 `docs/porting/upstream-prs.md`；本文只跟踪「做到
  哪一步」，不复制那边的细则，冲突时以那边为准。
- 换机器接着弄：`git pull origin main` → 看本文「阶段总览」找当前
  阶段 → 按 `upstream-prs.md` §4/§5/§6 提取改动 → 按本文
  「验证装备（V1-V7）」与该阶段任务里的验证子清单**跑完本地验证**
  → 验证全过才建 PR → 回来更新本文状态与执行记录。构建命令的占
  位符（`<MINGW64>` / `<repo>` / 两份 gdb）本机取值见
  `AGENTS.local.md` §1/§2/§4/§6，通用形态见 `docs/build-windows.md`。
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
- 上游现有分支 `94-windows-support`，与上游 `main` **完全相同**
  （compare: 0 ahead / 0 behind，同为 `6ec2019`），无任何独有提交，
  历史上也从未有 PR 以它为 base。它是作者在我们回帖之前自己开的
  工作分支（其命名惯例 `<issue号>-<主题>`）。
- **PR base 已定：`epasveer:main`**（主人 2026-11-10 决定，不再向
  作者问询，理由：让 PR 更快落地；`main` 也是代码最终必须落的地
  方与「fork 退役」承诺的前提）。若作者 review 时驳回、要求改投
  `94-windows-support`，用 `gh pr edit --base` 改，并按 §2 阶段 2
  前置的说明把后续 PR 的切分支基线换成该分支的 tip。
- 本 fork 镜像 `upstream-main` 停在 `8d19e2d`，落后上游；阶段 0 要
  先快进镜像并把上游合进 `main`，PR 才切在当前 tip 上。

## 阶段总览

| 阶段 | 名称 | 分支 | 状态 |
|---|---|---|---|
| 0 | 准备与基线（同步上游、确认 base 分支） | `sync/<date>` | 已完成 |
| 1 | PR#1 路径归一化（试点，跑通全流程） | `pr/path-normalize` | 进行中（PR #572 已开，待 review） |
| 2 | PR#2 Windows 平台块（五合一） | `pr/win-platform` | 未开始 |
| 3 | PR#3 console 分流（旗舰，压轴） | `pr/win-console-split` | 未开始 |
| 4 | Windows 发布 workflow 上游化 + 自动构建 | `pr/win-release-ci` | 未开始 |
| 5 | 收敛与退役判断 | — | 未开始 |

阶段 1→2→3 串行（同文件动刀，见各阶段「前置」）；阶段 3 的**设计
讨论**可与阶段 1/2 并行先行（作者点名想聊 console）。阶段 4 前置是
阶段 1-3 全部合入。阶段 5 在阶段 4 之后。

---

## 验证装备（V1-V7）：PR 发出去之前必须在本地跑完

`upstream-prs.md` §5 只说了「验证四件套」；本节把它落成可执行命令，
每个阶段的任务清单直接引用 V 编号。**验证不过就不建 PR**——作者测不
了 Windows，我们的验证记录是唯一的可信度来源。

### 占位符与通则

- `<MINGW64>`：MinGW64 根目录；`<repo>`：仓库根；`<GDB64>` /
  `<GDB32>`：两份 gdb。本机具体路径全在 `AGENTS.local.md`
  §1/§2/§4/§6（不进版本库，换机器先看那份）。
- **每个 pr 分支用独立构建目录**（如 `.tmp/build-win64-pr1`、
  `.tmp/build-wsl-pr1`），不要复用 `main` 的 `.tmp/build-win64`：
  切分支后旧 CMake cache 会指向别的源码状态，结果不可信。想同时保
  留 `main` 的工作树，用 `git worktree add .tmp/wt-pr1 pr/<topic>`。
- 验证输出（构建日志、测试退出码、会话截图）统一存
  `.tmp/pr<N>-verify/`（先 `mkdir -p`）；需要长期引用的摘录进
  `docs/porting/evidence/`（`.tmp/` 不进版本库，裸引用会失效）。
- 手工 GUI 会话会真弹窗抢焦点：主人在用机器时先打招呼
  （`AGENTS.local.md` §9）。

### V1 — Linux 构建（WSL2）

作者唯一能自验的平台，绝不能给他留坑。

    wsl.exe -d Ubuntu-24.04 -- bash -lc "cd /mnt/<repo路径> && \
      cmake -S src -B .tmp/build-wsl-pr<N> \
            -DCMAKE_BUILD_TYPE=Debug -DQTVERSION=QT6 && \
      cmake --build .tmp/build-wsl-pr<N> -j\$(nproc) seergdb 2>&1 \
      | tee .tmp/pr<N>-verify/build-wsl.log"

判定：configure 与构建均成功，产出 `seergdb`。

### V2 — Windows 构建（MinGW64）

    export PATH="<MINGW64>/bin:$PATH"
    <MINGW64>/bin/cmake.exe -S src -B .tmp/build-win64-pr<N> -G Ninja \
        -DCMAKE_MAKE_PROGRAM=<MINGW64>/bin/ninja.exe \
        -DCMAKE_BUILD_TYPE=Debug \
        -DCMAKE_C_COMPILER=<MINGW64>/bin/gcc.exe \
        -DCMAKE_CXX_COMPILER=<MINGW64>/bin/g++.exe \
        -DQTVERSION=QT6 -DCMAKE_PREFIX_PATH=<MINGW64>
    <MINGW64>/bin/cmake.exe --build .tmp/build-win64-pr<N> -j \
        2>&1 | tee .tmp/pr<N>-verify/build-win64.log

`-G` / `CMAKE_MAKE_PROGRAM` / 两个 `COMPILER` 一个都不能省（本机同
时存在 VS、scoop cmake、32 位 mingw，见 `AGENTS.local.md` §2）。
判定：产出 `.tmp/build-win64-pr<N>/seergdb.exe`。

### V3 — 零新增编译警告（V1/V2 的附加判定）

上游只在 `UNIX AND NOT APPLE` 下加警告选项，所以 Linux 构建日志是
有效信号；Windows 侧默认无警告选项，不作为判据。

    # 基线日志怎么来：在上游 tip 上跑一遍 V1 的同款命令，构建目录
    # 换 .tmp/build-wsl-base、日志名换 build-wsl-base.log。免切分支：
    #   git worktree add .tmp/wt-base upstream-main
    # 基线一次即可，三个 PR 共用（除非期间又同步了上游）。
    grep -E "warning:" .tmp/pr<N>-verify/build-wsl-base.log | sort -u \
        > .tmp/pr<N>-verify/warn-base.txt
    grep -E "warning:" .tmp/pr<N>-verify/build-wsl.log | sort -u \
        > .tmp/pr<N>-verify/warn-pr.txt
    diff .tmp/pr<N>-verify/warn-base.txt .tmp/pr<N>-verify/warn-pr.txt

判定：`warn-pr.txt` 相对基线**无新增行**（本 PR 触及的文件必须干
净；既有警告不属本 PR 责任，但新增的必须修掉再发）。

### V4 — `tests/hellopathnorm` 双平台

自检式回归测试，全过退出码 0（覆盖 `normalizeSourcePath` /
`normalizeFileKey` / GdbMonitor 记录级 `fullname` 重写 + 八进制还
原；期望值按平台条件化：Windows 变换、其他平台 identity）。

    # Linux / WSL2
    cd tests/hellopathnorm && make && ./hellopathnorm; echo "exit=$?"

    # Windows：配方在 tests/hellopathnorm/README.md（moc + g++ 链接
    # 真实 SeerUtl/GdbMonitor 源码），跑完 echo 退出码同样要是 0

判定：两侧 `exit=0`。PR#1 必跑；PR#2/#3 也跑一次作回归（它们同样
动 `SeerUtl` / `SeerGdbWidget`）。

### V5 — diff 卫生（发 PR 前的最后一道闸）

    git diff --stat upstream-main..HEAD        # 文件清单 = 本 PR 范围
    git diff --check upstream-main..HEAD       # 空白 / tab 错误
    git diff upstream-main..HEAD | grep -nE \
      "docs/porting|decisions\.md|invariants|AGENTS|SeerStackTrace|\bD[0-9]+\b"

判定：① 文件清单与 `upstream-prs.md` §4 该 PR 的清单一致，**不含**
`docs/`、`.github/`、`AGENTS*`、README fork 块、
`SeerStackTrace.{h,cpp}`（阶段 4 的 yml PR 是唯一例外）；② 无空白
错误（STYLEGUIDE：4 空格缩进、源码里不许有真 tab）；③ 第三条 grep
**零命中**——上游不知道也不需要知道我们的 ADR 编号。

已知要手工清理的点（PR#1）：`tests/hellopathnorm/README.md` 与
`hellopathnorm.cpp` 头部注释里引了 `docs/porting/decisions.md`、
`docs/porting/specs/gdb-glue.md`、`.tmp/regression/run.sh` 和
"(fork addition; not upstream)" 字样——上游没有这些文件，进 PR 前
改写成自洽描述（只讲测什么、怎么构建怎么跑）。

### V6 — 平台隔离边界 grep

    git diff upstream-main..HEAD -- src/ | grep -E "^\+" \
      | grep -nE "posix_openpt|fork\(|vfork|execl|dlfcn|termios|QSocketNotifier"

判定：命中项必须都在 `#ifdef Q_OS_WIN` 之外**未被改动**的上下文，
或已被正确分流（`maintenance.md` §3 的清单）。另两条同节检查按需：
`QSettings` 派生目录仍成立（D10 前提）、路径进 gdb 命令的新入口都
包了归一化（D8 三层）。

### V7 — 手工 GUI 会话

被测程序先构建（Makefile 都是朴素 `g++ -g -o x x.c`）：

    # Linux
    cd tests/<name> && make
    # Windows（make 的名字是 mingw32-make）
    cd tests/<name> && PATH="<MINGW64>/bin:$PATH" mingw32-make

启动会话（CLI 组合：自动载程序 + 断 main + 运行）：

    <构建目录>/seergdb.exe --gdb-program <GDB64> --bf main -r <程序>

判定（每条会话四项全中，同 `verification.md` §2.1）：停在 main
（消息面板 "Program reached breakpoint" + 栈帧面板 + 源码打开）∧
程序按该场景预期行为 ∧ seergdb 存活 ∧ 无残留进程
（`tasklist | grep <程序名>` / `ps`）。截图存 `.tmp/pr<N>-verify/`。

个别程序在 Windows 下构建要小调（链接库置于源文件之后、helloglut
需 `-DGLUT_DISABLE_ATEXIT_HACK` 等），清单见 `verification.md`
§1.2 的构建侧注记；只改构建命令，不动上游测试源码。

批量构建/自动化脚本（`.tmp/phase6/build-programs.sh`、
`session.ps1`）**不进版本库**，换机器可能没有；缺失时按上面手工步
骤逐个来，别因为脚本不在就跳过验证。

### 涉及 gdb/mi 解析的 PR 追加：双 gdb 交叉

`docs/plan.md` 使用规则 7：改 gdb/mi 解析、寄存器/内存视图、地址
格式化的，64 位与 32 位两份 gdb 各验一遍（位置见
`AGENTS.local.md` §4）——不能假设地址宽度、寄存器名集合、gdb 版本
固定。做法：同一会话用 `--gdb-program <GDB32>` 重跑，配 32 位被测
程序。PR#1（MI 记录重写）与 PR#3（中断/信号映射）必做。

---

## 阶段 0：准备与基线

状态：已完成（2026-11-10）

前置：无。

### 任务

- [x] 0.1 `git fetch upstream`，把镜像快进到上游当前 tip
      （`git fetch upstream main:upstream-main`、
      `git push origin upstream-main`）。已完成（2026-11-10）：
      `8d19e2d` → `6ec2019`，镜像本地与 origin 同步。
- [x] 0.2 按 `maintenance.md` §1 全流程把上游合进 `main`：打
      `pre-sync-<date>` 锚点 → `sync/<date>` 分支 merge → 逐条核对
      `invariants.md` 红线 → 双平台回归（= 本文 V1 + V2 + V4，加
      `maintenance.md` §1 第 4 步的 Windows 矩阵抽样与 V6 grep 复
      查）→ 快进 `main` 并 push → `docs/CHANGELOG.md` 记一条。
      已完成（2026-11-10，提交 `c942eb4` + `da21069`，锚点
      `pre-sync-20261110`）：merge 零冲突（16 文件 +629/-50）；
      invariants I1-I9 结构核对全过；上游新增命令点
      （`-thread-select` / `frame %1`）只带整数参数，I7 无需补包裹；
      Windows 全量构建 126 步零 error；WSL2 clean 全量重建 exit=0
      （增量构建曾报 clock skew，已 clean 重跑排除假绿）；
      hellopathnorm 双平台 13/13；x64（gdb 18.1 + hellofibonacci）
      与 x86（gdb 17.1 + t-hello32）会话冒烟均 PASS（停 main 命中、
      优雅关闭、无残留）。脚本存 `.tmp/pr0-verify/smoke-x64.ps1`
      （ConstrainedLanguage 限 cmdlet/`taskkill`，不可用
      `.CloseMainWindow()`）。
- [x] 0.3 同步后复核可上游化 diff 规模（原估约 600 行 / 32 文件），
      确认上游新增提交（ParallelStacks 等）与三个 PR 的文件没有新
      交集；有交集则更新 `upstream-prs.md` §4 的文件级清单。
      已完成（2026-11-10）：合并后 diff 仍为 32 文件 +856/-64
      （规模不变）；上游本轮 14 提交与 fork diff 的交集仅
      `SeerGdbWidget.cpp` 一个文件，且 hunk 区域不重叠（上游动
      :245 connect 块 / :2411 `handleGdbThreadSelectFrame` / :3091
      ParallelStacks 接线；fork 动 :39-44 / :99-103 / :868 /
      :2081 / :3972 一带），§4 文件级清单无需更新。
- [x] 0.4 PR base 分支：**定为 `epasveer:main`**（主人 2026-11-10
      决定，不向作者问询；详见 §0）。
- [x] 0.5 把两项新事实补进 `upstream-prs.md`：§1 背景事实加作者
      2026-11-09「smaller PR's are better」（大 PR 口子关闭）与
      base=main 的决定及理由，§3 的 `epasveer:main` 处加一句
      「已确认，若作者驳回改投 `94-windows-support`」；同时追加
      D24 与 `docs/plan.md` 决策记录表索引行。已完成（2026-11-10，
      随 `da21069` 入库）。
- [x] 0.6 检查 `origin` 的 fork 关系仍可用于跨仓 PR（仓库已改名
      `seer-windows`）：`gh api` 实测 `parent` / `source` 均为
      `epasveer/seer`，`fork=true`，通道完好（2026-11-10）。真正
      发起 PR 的动作在任务 1.5 完成。

### 出口条件

1. `main` 含上游当前 tip，双平台回归通过（WSL2 全量构建 +
   `tests/hellopathnorm` + Windows 构建与抽样会话），`origin/main`
   已推送。
2. `upstream-main` 镜像与上游 tip 一致，本地与 origin 同步。
3. base 分支已定为 `main` 并落文档（不再等作者答复）。

---

## 阶段 1：PR#1 路径归一化（试点）

状态：进行中——PR **#572** 已开（2026-11-10，base=`epasveer:main`），
待作者 review；任务剩 1.6（review 往返）/1.7（合入后小同步）/1.8
（试点复盘）。

前置：阶段 0 出口条件全部满足。

内容见 `upstream-prs.md` §4 表第 1 行与文件级清单（GdbMonitor 读侧
编码守卫 + `fullname` 记录级重写、SeerGdbWidget 写侧 `toLocal8Bit`、
SeerUtl 两函数 + 22 处调用点、显示层 `toNativeSeparators`、
`tests/hellopathnorm` 两段）。规模约 210 行 + 测试。**这是唯一改变
Linux 行为的 PR**（八进制还原本身修 Linux bug），PR 描述按「带回归
测试的行为改进」写。

### 任务

- [x] 1.1 fetch 上游，从最新 tip 切 `pr/path-normalize`（§6 步骤
      0-1）。
- [x] 1.2 提取改动：新文件整取、混合文件（`SeerGdbWidget.cpp`、
      `GdbMonitor.cpp`、`SeerUtl.{h,cpp}`）按 hunk 裁剪，补丁形态
      与 `main` 逐行一致（§5 提取手法）。
- [x] 1.3 **本地验证（全过才进 1.4）**——记录存 `.tmp/pr1-verify/`：
      - [ ] 1.3a V1 Linux 构建 + V2 Windows 构建（各用
            `build-*-pr1` 目录）。
      - [ ] 1.3b V3 零新增警告：先在 `upstream-main` 上原样构建一
            次拿基线日志，再与 pr1 的日志 diff。
      - [ ] 1.3c V4 `hellopathnorm` 双平台 `exit=0`。**Linux 侧的
            八进制还原用例就是本 PR 的修复本体**（上游从不还原
            `\NNN`，中文源码路径在 Linux 同样打不开），必须亲眼
            看到它在 pr1 上过、在 `upstream-main` 基线上不过。
      - [ ] 1.3d V5 diff 卫生：含清理 `tests/hellopathnorm/` 里引
            `docs/porting/*`、`.tmp/regression/run.sh`、"not
            upstream" 的字样（V5 末尾已列）。
      - [ ] 1.3e V6 隔离边界 grep（本 PR 不新增平台 API，预期零
            命中）。
      - [ ] 1.3f V7 Windows 会话三例：
            ① **中文路径**——被测程序放中文目录（exe 路径中文 +
            源码目录中文，即 `verification.md` B3/B4 场景）：停
            main、编辑器打开源码、**不得出现** "Can't find source
            file" 模态框、标题栏中文正确；
            ② **同一文件不同拼写只开一个 tab**——分别用正斜杠 /
            反斜杠 / 盘符大小写不同 / 相对与绝对四种拼写触发打开
            同一源文件（断点对话框、栈帧双击、源码浏览器各走一
            遍），编辑器 tab 数始终为 1；
            ③ **混合大小写断点命中**——用与 DWARF 记录大小写不同
            的路径下断点，运行后确实停下。
      - [ ] 1.3g V7 Linux 会话对照：普通 ASCII 路径行为与上游一致
            （identity 不回归）+ 中文/非 ASCII 源码路径能打开（改
            进生效）。WSL2 侧 GUI 用 Xvfb 或直接人工。
      - [ ] 1.3h 双 gdb 交叉（`<GDB64>` 18.1 / `<GDB32>` 17.1）：
            MI 记录重写涉及 `fullname` 解析，两份 gdb 各跑一遍
            1.3f 的 ①，注意 17.1 输出的反斜杠拼写差异。
- [x] 1.4 整理成 1-3 个干净英文提交（引用 issue #94，遵循
      STYLEGUIDE），push `origin pr/path-normalize`。
- [x] 1.5 开 PR（base = `epasveer:main`），描述按 §5 模板五段
      （motivation / what / design rationale / verification / Linux
      行为说明）。verification 段直接贴 1.3 的记录摘要（命令 + 结
      果 + 日期 + 双平台构建版本号），**尤其要写清 1.3c 的对照**：
      同一条八进制还原用例在 `upstream-main` 基线上失败、在本 PR
      上通过——这是 PR#1 最有说服力的一行证据。Linux 段按「行为改
      进 + 回归测试」写，不含糊。
- [ ] 1.6 review 往返：改动以作者为准；每轮把等价改动以小 commit
      带回 fork `main`；若与既有决策冲突，`decisions.md` 追加记录。
- [ ] 1.7 作者合入后：按 `maintenance.md` §1 小同步（或攒到下一
      release），确认 merge 回 `main` 零冲突；删 `pr/path-normalize`。
- [ ] 1.8 复盘试点：把作者的 review 风格、粒度偏好、验证记录的反
      馈写进 `upstream-prs.md` §10，作为阶段 2/3 的校准输入。

### 出口条件

1. 1.3 各项全过，记录在 `.tmp/pr1-verify/`，摘要已进 PR 描述；需
   长期引用的佐证已摘录进 `docs/porting/evidence/`。
2. PR 被作者合入上游 `main`（若他改投 `94-windows-support`，以他
   指定的分支为准并回写本文 §0）。
3. fork `main` 已同步该合入，双平台回归通过。
4. `pr/path-normalize` 已删除，试点复盘已记录。

---

## 阶段 2：PR#2 Windows 平台块（五合一）

状态：未开始

前置：阶段 1 出口条件满足（`SeerUtl.{h,cpp}` 由 #1/#2 共同动刀，
必须等 #1 合入后再从新 tip 切分支，否则 hunk 上下文错位）。若作者
要求并行提交，改为在 #1 分支之上切出并说明依赖。若 PR base 被作者
改投 `94-windows-support`，则这里的「新 tip」指该分支的 tip——需
额外 `git fetch upstream 94-windows-support`（`upstream-main` 镜像
只跟 `main`）。

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
- [ ] 2.3 **本地验证（全过才进 2.4）**——记录存 `.tmp/pr2-verify/`：
      - [ ] 2.3a V1 + V2 构建（`build-*-pr2` 目录）。**Linux 侧重点
            看 configure 输出与构建路径零变化**：不得出现 RC 语言
            启用、`.rc` / `.manifest` 编译、`SEER_GDB_NAME` 默认值
            改动等 WIN32 段动作（拿 `upstream-main` 的 configure
            日志对照 diff）。
      - [ ] 2.3b V3 零新增警告（基线同 1.3b 手法）。
      - [ ] 2.3c V4 `hellopathnorm` 双平台作回归（本 PR 动
            `SeerUtl`，别把 PR#1 的成果碰坏）。
      - [ ] 2.3d V5 diff 卫生：**特别核对排除项没混进来**——
            `SeerStackTrace.{h,cpp}` 的 CMake 条目与
            `-fno-omit-frame-pointer` 不得出现在补丁里；
            `.gitignore` 只加 ico 的定向例外。
      - [ ] 2.3e V6 隔离边界 grep：新增的 Win32 调用
            （Toolhelp32 / `QueryFullProcessImageNameW` / SID 查
            询）必须全在 `QProcessInfo_win.cpp` 或 `Q_OS_WIN` 守卫
            内；`QProcessInfo_win.cpp` 的守卫要在任何 Qt include
            **之后**（`invariants.md` I1，放前面整个文件会编成空翻
            译单元——构建过了也要人工看一眼这个顺序）。
      - [ ] 2.3f V7 Windows 会话五例：
            ① **进程列表**——attach 对话框列表非空、有 pid / 程序
            名 / 用户，可选中（对照任务管理器核几行真值）；
            ② **attach 回填**——选一个进程后 Program name 自动填
            成该进程的真实路径（先手填一个错的目标再选进程，确认
            被覆盖），attach 后栈帧干净（B7 场景：附加 → 自动停
            止 → 帧面板出栈 → Detach 后 inferior 仍存活）；
            ③ **图标**——构建产物 exe 在资源管理器 / 任务栏 / 窗口
            左上角显示 Seer 图标（多分辨率，缩放到 16/32/48/256
            看是否清晰）；
            ④ **设置落 ini**——改一项设置后
            `%APPDATA%\seergdb\` 下出现 ini 文件、注册表无新增
            项；重启后设置读回；**宏目录与 user MI scripts 目录可
            用**（它们从 `settings.fileName()` 派生，registry 形态
            下是坏的，这条是 D10 的验收点）；
            ⑤ **默认字体**——首次启动（无设置文件）时源码编辑器与
            console 字体为 Cascadia Code；把该字体临时改名/卸载后
            依次降级 Cascadia Mono → Consolas → Courier New。
      - [ ] 2.3g V7 Linux 会话对照（**零行为变化**是本 PR 的纪律）：
            进程列表照常从 `/proc` 读、内容不变；设置文件位置与
            格式不变（NativeFormat 在 Linux 本就是 ini）；字体请
            求原样透传（设成什么就是什么，不被偏好列表改写）；
            attach 保持「非空不覆盖」的原有语义。
      - [ ] 2.3h 双 gdb 交叉：本 PR 不改 MI 解析，可免；但若 review
            中动了 attach / 进程信息链路，补一次。
- [ ] 2.4 push + 开 PR（base = `epasveer:main`；描述分节对应五个子
      主题）。verification 段贴 2.3 记录，**并把 2.3a 的 Linux
      configure 日志对照与 2.3g 的零变化清单单独列一节**——作者只
      能验 Linux，这一节就是给他看的「你那边什么都不会变」的证明。
- [ ] 2.5 review 往返（同 1.6）。
- [ ] 2.6 合入后小同步、删分支。

### 出口条件

1. 2.3 各项全过，记录在 `.tmp/pr2-verify/`，摘要已进 PR 描述。
2. PR 被合入；3. fork `main` 同步且双平台回归通过；4. 分支已删。

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
- [ ] 3.3 **本地验证（全过才进 3.4）**——记录存 `.tmp/pr3-verify/`：
      - [ ] 3.3a V1 + V2 构建（`build-*-pr3` 目录）。**另外再配一
            份带 gdb 通信日志的 Windows 构建**（configure 加
            `-DCONFIG_SEER_GDB_LOGOUT=1`，见 `build-windows.md`
            §2）——3.3f 的「MI 流干净」只能靠日志证。
      - [ ] 3.3b V3 零新增警告。
      - [ ] 3.3c V4 `hellopathnorm` 双平台作回归（本 PR 动
            `SeerGdbWidget`）。
      - [ ] 3.3d V5 diff 卫生：确认**没有**夹带 `SeerStackTrace`
            与 PR#1/#2 已合入的内容（基线是含前两个 PR 的上游
            tip，diff 应当只剩 console 主题）。
      - [ ] 3.3e V6 隔离边界 grep：`posix_openpt` / `termios` /
            `QSocketNotifier` / 裸 fd 读写这些命中项必须**全部位
            于 Linux 分支且原文未改**——这是「Linux PTY 一字不动」
            承诺的机械核对；配合
            `git diff upstream-main..HEAD -- src/SeerConsoleWidget.cpp`
            逐行确认改动只有守卫与页隐藏。
      - [ ] 3.3f V7 Windows 会话六例：
            ① **终端程序弹独立控制台 + ANSI 颜色**——
            `helloansicodes` / `helloansicodes2`：弹出独立控制台
            窗口、颜色正确渲染（前置：`HKCU\Console\
            VirtualTerminalLevel=1`，见 `AGENTS.local.md` §8；不
            设的表现是转义序列按字面打印）；
            ② **stdin 交互**——`hellostack`（两个 `gets`）：向弹窗
            控制台键入两行，程序消费后正常退出；
            ③ **MI 流干净**——用 3.3a 那份带日志的构建跑 ①②，
            日志里不得混入 inferior 的 stdout；同时确认
            `new-console on` 确实发出。已有证伪/证实记录可直接引
            用：`docs/porting/evidence/poc-q1-log1-default-polluted.txt`
            （不发 `new-console` 时 MI 流被污染）、
            `poc-q1-log3-newconsole-console.txt`、
            `poc-q1-log4-newconsole-gui.txt`；
            ④ **GUI 程序不弹控制台**——`helloglut`（`never` 流
            程）：不出现多余控制台窗口，与 Visual Studio 行为一
            致（对照 `poc-q1-log4`）；
            ⑤ **Interrupt 可用**——`hellofft` 长输出运行中发
            Ctrl+I（B6 场景）：inferior 转挂起（进程仍存活）、消
            息面板与帧面板回到停止态、seergdb 存活；`kill` 走 gdb
            的 `kill`；
            ⑥ **UI 收敛**——Command Logs 里 console 页已隐藏；信号
            菜单里无 Windows 等价物的信号已隐藏（菜单项数对照
            Linux 侧）。
      - [ ] 3.3g 双 gdb 交叉：中断/信号映射与 gdb 版本相关，
            `<GDB64>` 18.1 与 `<GDB32>` 17.1 各跑一遍 3.3f 的
            ⑤（32 位配 32 位被测程序）。既有记录
            `poc-q2-x64-gdb18.txt` / `poc-q2-x86-gdb17.txt` 可引
            用，但**发 PR 前要在当前 tip 上重跑一次**，不能只引旧
            证据。
      - [ ] 3.3h V7 Linux 会话对照：内嵌 console 页照常存在、PTY
            链路未坏——跑一个终端程序（如 `hellostack`），输出进
            GUI 内嵌终端、可在其中键入 stdin、Ctrl-C 行为与上游一
            致。WSL2 侧 GUI 用 Xvfb 或人工。
- [ ] 3.4 push + 开 PR（base = `epasveer:main`）。design rationale
      段带 3.0 的完整论证与 PoC 证据（`docs/porting/evidence/` 的
      poc-q1 系列：默认形态污染 MI 流 / `new-console on` 下控制台
      与 GUI 两例的干净日志）；verification 段贴 3.3 记录，六例会
      话逐条给结果，Linux 段明确写「PTY 代码零改动，附 V6 grep 与
      `SeerConsoleWidget.cpp` 的 diff 行号范围」。
- [ ] 3.5 review 往返（同 1.6）。若作者要求换路线（如 ConPTY），按
      `upstream-prs.md` §9 处理：D1 已预留增强口子，另立决策。
- [ ] 3.6 合入后小同步、删分支。

### 出口条件

1. 3.3 各项全过，记录在 `.tmp/pr3-verify/`，摘要与 PoC 引用已进 PR
   描述。
2. PR 被合入；3. fork `main` 同步且双平台回归通过；4. 分支已删；
5. 至此约 600 行可上游化资产全部进上游。

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
      机械核对：

          grep -nE "skywind3000|seer-windows|E:/|D:/|/mnt/" \
              .github/windows-release.yml

      判定：零命中（或命中项均已参数化）。
- [ ] 4.2 与作者确认落地形态：直接进上游 `.github/`，还是先在他
      的分支上跑通再合 `main`；确认触发条件（release 发布 +
      `workflow_dispatch` 指定 tag 演练）他接受。
- [ ] 4.3 切 `pr/win-release-ci` 提取 yml（这是唯一带 `.github/`
      的 PR，属 §5「只含 src/ 与 tests/」纪律的**明确例外**，PR
      描述里说明理由）。V5 diff 卫生照跑，但把「不含 `.github/`」
      这条判定反过来用：本 PR **只**含 yml，不含任何 src/ 改动。
- [ ] 4.4 **验证（在 fork 上演练，不动上游）**——记录存
      `.tmp/pr4-verify/`：
      - [ ] 4.4a 推分支后触发 `workflow_dispatch` 指定一个 tag：

                gh workflow run windows-release.yml \
                    -R skywind3000/seer-windows --ref pr/win-release-ci \
                    -f tag=<演练tag>
                gh run watch <run-id> -R skywind3000/seer-windows

        判定：run 结论 success；失败则读
        `gh run view <run-id> --log-failed` 定位。`tag` 不必预先存
        在——yml 内已处理：找不到对应 release 时自建 draft（演练
        完把该 draft 删掉，别留在 fork 的 release 列表里）。
      - [ ] 4.4b 产物自检——下载 release 资产解压，在 **PATH 只含
            System32** 的环境里跑一次完整会话（`build-windows.md`
            §4 末尾的验证法）：

                <D>\seergdb.exe --gdb-program <D>\gdb.exe --bf main \
                    -r <测试程序>

        判定：起得来（无缺 DLL 弹窗 / 无 0xC0000135）、停 main、
        gdb 随包可用、DLL 闭包完整（导入表迭代收敛，配方见
        `build-windows.md` §4）。
      - [ ] 4.4c 记录 run 链接、产物体积、自检会话结果，写进 PR
            描述——作者没有 Windows 机器，这段是他判断 yml 可用
            的唯一依据。
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
| 2026-11-10 | 0 | 任务 0.4 完成：PR base 定为 `epasveer:main`（主人决定，不问作者，驳回再改） | 查证：`94-windows-support` 与 `main` 完全相同、无独有提交、从无 PR 以其为 base；剩余任务 0.1/0.2/0.3/0.5 |
| 2026-11-10 | 0 | 补「验证装备（V1-V7）」一节，各阶段任务里的验证项展开成可执行子清单（含具体命令、被测程序、判定标准） | 依据：`docs/build-windows.md` §2/§4、`docs/porting/verification.md` §2.1/§3/§4、`tests/hellopathnorm/README.md`、`AGENTS.local.md`；`.tmp/phase6` 自动化脚本本机不存在，验证步骤一律写成手工可复现形态 |
| 2026-11-10 | 0 | 任务 0.1/0.2/0.3/0.5 完成：镜像快进 `8d19e2d`→`6ec2019`、merge 零冲突、invariants I1-I9 全过、双平台构建+hellopathnorm 13/13+x64/x86 冒烟 PASS、diff 交集复核（仅 SeerGdbWidget.cpp 且 hunk 不重叠）、D24 入库；`main` 快进并 push（`c942eb4`+`da21069`），sync 分支已删 | 阶段 0 出口条件满足，置「已完成」；下一步：阶段 1（PR#1 路径归一化） |
| 2026-11-10 | 1 | 任务 1.1-1.3 完成：切分支 + 提取（12 src 文件 210+/41- + 测试三文件）+ 本地验证全过（V1-V7、双平台 hellopathnorm 13/13、octal PoC 基线 broken/PR resolved 对照、V5 卫生与 V6 隔离零命中、Linux 中文路径 headless 会话） | 规模与 §4 估算一致 |
| 2026-11-10 | 1 | 深度回归抓到两个真缺陷并修复（Finding A：octal 字节编码随 inferior 构建方式而异，UTF-8/ANSI 双态，原解码只支持 GBK → 严格 UTF-8 先验+回退；Finding B：`file="..."` 字段同样带转义未重写 → 扩为 `(?:fullname\|file)`）；修复 commit `54da161`（pr 分支）已镜像到 main（`b2a0687`+`50a2e07`），hellopathnorm 补 2 用例至 14 项双平台全过 | Windows 会话复验：x64 18.1 与 x86 17.1 中文路径会话截图确认编辑器打开中文源码；--bs 反斜杠/小写拼写断点哨兵测试双双命中 |
| 2026-11-10 | 1 | PR #572 已开（+473/-41，15 文件，3 提交）；描述含 motivation/what/rationale/verification/Linux 行为五段，验证记录含基线对照 | 剩 1.6 review 往返、1.7 合入后小同步、1.8 试点复盘 |
