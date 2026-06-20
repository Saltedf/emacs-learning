# Emacs 极客学习路线 — 实施计划

## Context (背景与动机)

**用户现状**:
- Emacs 经验: 只会基本移动 + 简单编辑,基本把 Emacs 当记事本
- 编程经验: 有基础但不熟练,几乎无 Lisp 经验
- 投入强度: 最大强度 (3-4+ 小时/天)
- 已有材料: 官方手册三件套 (`emacs-lispintro-30.2` / `emacs-lispref-30.2` / `emacs-manual-30.2`)

**用户目标**:
1. 日常生产力极致
2. 精通 Elisp 写自己的包
3. 贡献开源 Emacs 包

**用户偏好**: 以两本 Elisp 手册为主线 (intro + lispref),editor manual **穿插在每个模块中作为概念锚点**——讲解"用户看到的表面"对应"Elisp 实现的底层"。三本手册形成"概念-入门-深度"的三层对照。

**关键约束** ★顶级要求★: 教程**完全替代三本手册**——读者不需要再翻开原始 texi 文件。所有概念、原理、代码示例、练习题都内联在教程中,原手册的每一节内容都被吸收、转述、扩展并配上实践。教程是 self-contained 的教科书,不是阅读指南。

**预期产出**: 一份螺旋上升的学习路线 + 详尽的分步骤教程 (讲解 + 实践,完全自包含),6-7 个月可走完。

---

## 第一性原理: Emacs 的核心心智模型

学习 Emacs 的本质不是学键位,而是建立以下五条心智模型。每一条都会在 Module 0 详细展开,并贯穿整个学习过程:

1. **Emacs 是 Lisp 机器伪装成编辑器** — 不是"编辑器 + 脚本能力",而是"Lisp 解释器 + 一些文本原语"。学 Emacs 等于学 Elisp。
2. **一切皆 Buffer** — 你看到的文件、目录列表、shell 输出、邮件、聊天、错误日志都是 buffer。Buffer 是统一的数据载体。
3. **自文档化** — Emacs 自己描述自己。`C-h f`、`C-h v`、`C-h k` 直接从源码生成文档。文档与代码同源,绝不脱节。
4. **Live system** — 函数可以热重载。`eval-buffer` 立即生效。这是为什么 Elisp 比"配置文件"更强大。
5. **可组合性** — 任何行为都是函数调用,任何函数都可重定义。没有"插件边界",整个 Emacs 是一个统一对象系统。

**推论**: 既然 Emacs 本质是 Lisp,那么学 Elisp 优先级 > 学键位 > 学特定模式 (org/magit)。键位可以查、模式可以换,只有 Elisp 是不变的核心。

---

## 学习方法论

#### 三本手册的角色定位 (穿插对照法)

```
┌──────────────────────────────────────────────────────────────┐
│  emacs-manual      → 概念锚点 (What + Why)                   │
│  (editor surface)    "用户看到什么,为什么这样设计"            │
│                          ↓                                    │
│  emacs-lispintro   → 入门实现 (How, simple)                  │
│  (Chassell)          "用最少的 Elisp 把概念表达出来"          │
│                          ↓                                    │
│  emacs-lispref     → 深度机制 (How, complete)                │
│  (reference)         "源码层面到底怎么运作"                    │
└──────────────────────────────────────────────────────────────┘
```

**穿插原则**: 每个模块明确标注三本手册的对应章节:
- **概念锚点** (editor manual): 读这一章理解"用户视角看到什么"
- **入门实现** (lisp intro): 读 Chassell 学"最简单的 Elisp 表达"
- **深度机制** (lisp ref): 精读对应章节"源码层机制"

这样学习时,每个概念都从"用户视角 → 简单 Elisp → 深度实现"三个层次打通。

### 螺旋上升 (Spiral Curriculum)
不是"读完一本再读下一本",而是**多次穿越材料,每次更深一层**:
- **第 1 圈** (Module 0-2): 装备 + 编辑生存 + 最小化 Elisp (能让 Emacs 听话)
- **第 2 圈** (Module 3-4): 系统读完 Chassell 入门书 + 把 Emacs 配成自己的样子
- **第 3 圈** (Module 5-6): 攻克 Elisp Reference 的核心章节 + 写真实可用的包
- **第 4 圈** (Module 7-8): 进入开源生态,贡献代码

### 做中学 (Learning by Doing)
每个概念必须落到一个**可触摸的产物**:
- 读到 `defun` → 立刻写 5 个自己的函数
- 读到 `mapcar` → 立刻重构一段重复代码
- 读到 `mode` → 立刻定义一个自己的 minor mode
- 不产出 = 没学会

### 三段式 (Read → Drill → Build)
每个 Module 都包含:
1. **Read (教程内联教学)**: 不再是"去读手册某章",而是教程本身**完整转述**手册内容,加上第一性原理讲解、图示、对比例子、常见误区。读者不需要再翻原手册。
2. **Drill**: 30-50 个微练习,形成肌肉记忆
3. **Build**: 一个毕业项目,综合应用本模块所学

### 教程自包含原则 (Self-Contained Principle) ★最高优先级★

教程不依赖原始三本手册。对每一节手册内容,教程要做四件事:

| 手册原文 | 教程替代 |
|---|---|
| "See section X of manual" | **直接内联讲解**,把手册章节的核心论点、例子、表格搬过来并扩展 |
| 单一示例 | **多个对比例子** + 反例 + 边界情况 |
| 静态描述 | **可执行代码块** + "在你的 Emacs 里试一遍"指引 |
| 章末无练习 | **每节末尾 3-5 道微练习** + 章末综合练习 |

**对比示意**:

❌ 不合格 (依赖原文):
> "Kill ring 的运作机制详见 `killing.texi` 第 9.3 节"

✅ 合格 (自包含):
> "## 9.3 Kill Ring 的运作机制
>
> Kill ring 是一个 list + 一个游标 (`kill-ring-yank-pointer`)。`C-y` 取游标指向的元素;`M-y` 把游标后移一位,并用 `replace-region` 替换刚才 yank 的内容...
>
> ```elisp
> (defvar kill-ring ...)
> (defvar kill-ring-yank-pointer ...)
> ```
> (实际执行: 在 *scratch* 里 eval `kill-ring` 看你的 ring)
>
> **练习 9.3.1**: ..."
>
> **图示** (ASCII 画一遍 list + 游标移动)"

每节都要满足这四点: **内联讲解 + 多例 + 可执行 + 配练习**。

---

## 总体路线图 (宏观,9 个模块,~6-7 个月)

```
┌─────────────────────────────────────────────────────────────┐
│ Module 0: 心法与装备              (2 天,   ~6h)              │
│   建立 Emacs 心智模型 + 编译安装 + 最小 init.el              │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Module 1: 编辑生存 + 自文档系统   (1 周,  ~20h)              │
│   键位、kill-ring、isearch、help system 全打通               │
│   → 脱离鼠标做日常编辑                                        │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Module 2: Dired + Buffer/Window 管理 (4 天, ~12h)           │
│   像操作文本一样操作文件                                      │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Module 3: Elisp 入门 (Chassell)   (5 周, ~100h) ★核心★     │
│   精读 emacs-lispintro-30.2 全文 + 全部练习                  │
│   建立 Lisp 思维: list, eval, defun, recursion, lambda       │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Module 4: 配置体系 + 第一个 Minor Mode (2 周, ~40h)         │
│   use-package、keymap、hook、define-minor-mode              │
│   → 写出 500 行的自己风格 init.el                            │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Module 5: Power User 工作流       (3 周, ~60h)              │
│   Org-mode (capture/agenda/babel) + Magit + Projectile      │
│   + Corfu/Vertico/Eglot                                       │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Module 6: Elisp 深度 (Reference)  (8 周, ~160h) ★核心★     │
│   精读 emacs-lispref-30.2 核心章节:                          │
│   - data types / strings / lists / sequences / hash          │
│   - eval / variables / functions / macros                    │
│   - keymaps / modes / commands / minibuffer                  │
│   - text properties / overlays / display                     │
│   - files / buffers / windows / processes                    │
│   - byte-compile / debugging (Edebug) / package              │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Module 7: 写一个真实的包          (3 周, ~60h)              │
│   选题 → 设计 → 实现 → 测试 (ERT) → 文档 → MELPA 提交        │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Module 8: 开源贡献                (持续)                    │
│   读源码 → 找 issue → 提 PR → 跟 emacs-devel 邮件列表        │
└─────────────────────────────────────────────────────────────┘
```

---

---

## 手册覆盖矩阵 (Manual Coverage Matrix) ★硬性指标★

**目标**: 三本手册的**全部主要章节**都必须在某个模块里被讲解和实践,不能只是"参考"。每个章节在矩阵中标明归属模块与教学深度 (●精读 / ○实践 / △速览)。

### emacs-lispintro (Chassell, 17 章) — Module 3 主线,全部精读

| Chassell 章节 | 归属 | 深度 |
|---|---|---|
| 1. List Processing | M3-W1 | ● |
| 2. Practicing Evaluation | M3-W1 | ● |
| 3. How To Write Function Definitions | M3-W2 | ● |
| 4. A Few Buffer-Related Functions | M3-W2 | ● |
| 5. A Few More Complex Functions | M3-W3 | ● |
| 6. Narrowing and Widening | M3-W3 | ● |
| 7. car, cdr, cons: Fundamental Functions | M3-W1 | ● |
| 8. Cutting and Storing Text | M3-W4 | ● |
| 9. How Lists are Implemented | M3-W1 | ● |
| 10. Yanking Text Back | M3-W4 | ● |
| 11. Loops and Recursion | M3-W3 | ● |
| 12. Regular Expression Searches | M3-W4 | ● |
| 13. Counting via Repetition and Regexps | M3-W4 | ● |
| 14. Counting Words in a defun | M3-W5 (capstone prep) | ● |
| 15. Readying a Graph | M3-W5 (capstone prep) | ● |
| 16. Your .emacs File | M3-W5 + M4 复习 | ● |
| 17. Debugging | M3-W5 + M7 复习 | ● |

### emacs-lispref (~44 章) — Module 6 主线 + 其他模块按需精读

| Ref 章节 | 归属 | 深度 |
|---|---|---|
| 1. Introduction | M0 | △ |
| 2. Lisp Data Types | M6-W1 | ● |
| 3. Numbers | M6-W1 | ● |
| 4. Strings and Characters | M6-W1 | ● |
| 5. Lists | M6-W1 + M3 复习 | ● |
| 6. Sequences, Arrays, Vectors | M6-W1 | ● |
| 7. Records | M6-W1 | ● |
| 8. Hash Tables | M6-W1 | ● |
| 9. Symbols | M6-W2 | ● |
| 10. Evaluation | M6-W2 | ● |
| 11. Control Structures | M6-W2 | ● |
| 12. Variables | M6-W2 | ● |
| 13. Functions | M6-W2 | ● |
| 14. Macros | M6-W3 | ● |
| 15. Customization Settings | M6-W3 + M4 | ● |
| 16. Loading | M6-W3 + M4 | ● |
| 17. Byte Compilation | M6-W3 + M7 | ● |
| 18. Debugging Lisp Programs | M6-W4 + M7 | ● |
| 19. Reading and Printing Lisp Objects | M6-W1 | ● (streams) |
| 20. Minibuffers | M6-W5 | ● |
| 21. Command Loop | M6-W5 | ● |
| 22. Keymaps | M6-W5 + M4 | ● |
| 23. Major and Minor Modes | M6-W6 + M4 | ● |
| 24. Documentation | M6-W6 | ● (help) |
| 25. Files | M6-W6 | ● |
| 26. Backups and Auto-Saving | M6-W6 | ● |
| 27. Buffers | M6-W6 + M2 | ● |
| 28. Windows | M6-W6 + M2 | ● |
| 29. Frames | M6-W6 + M2 | ● |
| 30. Positions | M6-W6 | ● |
| 31. Markers | M6-W6 | ● |
| 32. Text | M6-W7 | ● |
| 33. Non-ASCII Characters | M6-W7 + M5 | ● |
| 34. Searching and Matching | M6-W7 | ● |
| 35. Syntax Tables | M6-W7 | ● |
| 36. Parsing Program Source | M6-W7 | ● |
| 37. Parsing Expression Grammars (PEG) | M6-W7 | ● |
| 38. Abbrevs | M6-W8 + M5 | ● |
| 39. Threads | M6-W8 | ● |
| 40. Processes | M6-W8 + M5 | ● |
| 41. Emacs Display | M6-W7 | ● |
| 42. Operating System Interface | M6-W8 | ● |
| 43. Preparing Lisp code for distribution | M7 | ● (package) |
| 44. Antinews | M8 | △ (趣味) |
| 45. Tips | M7 | ● |
| 46. Emacs Internals | M8 | ● |
| 47. Standard Errors | M8 / 附录 | ○ |
| 48. Standard Keymaps | M8 / 附录 | ○ |
| 49. Standard Hooks | M8 / 附录 | ○ |

### emacs-manual (34 章 + xtra) — 分布在 M0-M8

| Editor 章节 | 归属 | 深度 |
|---|---|---|
| 1. The Organization of the Screen (screen) | M0 | ● |
| 2. Entering and Exiting Emacs (entering) | M0 | ● |
| 3. Characters, Keys and Commands (commands) | M1 | ● |
| 4. Basic Editing Commands (basic) | M1 | ● |
| 5. The Minibuffer (mini) | M1 | ● |
| 6. Running Commands by Name (m-x) | M1 | ● |
| 7. Help (help) | M1 | ● ★最重要★ |
| 8. The Mark and the Region (mark) | M1 | ● |
| 9. Killing and Moving Text (killing) | M1 | ● |
| 10. Registers (regs) | M1 | ● |
| 11. Controlling the Display (display) | M1 | ● |
| 12. Searching and Replacement (search) | M1 | ● |
| 13. Commands for Fixing Typos (fixit) | M1 | ● |
| 14. Keyboard Macros (kmacro) | M1 | ● |
| 15. File Handling (files) | M2 | ● |
| 16. Using Multiple Buffers (buffers) | M2 | ● |
| 17. Multiple Windows (windows) | M2 | ● |
| 18. Frames and Graphical Displays (frames) | M2 | ● |
| 19. International Character Set Support (mule) | M5 | ● |
| 20. Major and Minor Modes (modes) | M4 | ● |
| 21. Indentation (indent) | M3 (anchor) | ● |
| 22. Commands for Human Languages (text) | M3 (anchor) | ● |
| 23. Editing Programs (programs) | M5 | ● |
| 24. Compiling and Testing Programs (building) | M5 | ● |
| 25. Maintaining Large Programs (maintaining, VC) | M5 | ● |
| 26. Abbrevs (abbrevs) | M5 | ● |
| 27. Dired (dired) | M2 | ● |
| 28. The Calendar and the Diary (calendar) | M5 | ● |
| 29. Sending Mail (sending) | M5 | ○ |
| 30. Reading Mail with Rmail (rmail) | M5 | △ (可选) |
| 31. Miscellaneous (misc) | M5 + M7 | ○ |
| 32. Emacs Lisp Packages (package) | M4 + M7 | ● |
| 33. Customization (custom) | M4 | ● |
| 34. Dealing with Trouble (trouble) | M7 | ● |
| 35. Command Line Arguments (cmdargs) | M4 | ● |
| 36. X Resources (xresources) | M0 (setup) | △ |
| 37. macOS-specific (macos) | M0 附录 | △ |
| 38. Haiku (haiku) | M0 附录 | △ |
| 39. Android (android) | M0 附录 | △ |
| 40. MS-DOS (msdos) | M0 附录 | △ |
| 41. Emacs History (gnu) | M0 | ● |
| 42. Glossary (glossary) | M0 + 全程查阅 | ● |
| 43. Acknowledgments (ack) | M8 | ○ |
| Extras: dired-xtra / emerge-xtra / vc-xtra / cal-xtra / picture-xtra / fortran-xtra / arevert-xtra | M2/M5 | △ |

### 覆盖完整性检查

- ✅ Chassell 17/17 章全部在 Module 3 精读
- ✅ Lispref 49 章全部被分配 (43 章●精读 + 3 章○实践 + 3 章△速览)
- ✅ Editor manual 34 章 + xtra 全部被分配 (28 章●精读 + 4 章○实践 + 8 章△速览 + 平台附录)
- ✅ 每章都有对应的 practice (drill 或 capstone)

**Legend**: ● = 精读 + 实践 (必学), ○ = 实践性浏览 (会操作), △ = 速览/平台附录 (了解即可)

---

## 文件结构 (实施时创建在 `/home/sun/src/learning/emacs-learning/`)

```
emacs-learning/
├── ROADMAP.md                       # 路线图精简版 (本计划文件的对外版)
├── PROGRESS.md                      # 学习进度跟踪表
├── MANUAL-CROSSREF.md               # ★三本手册的章节对照表 (穿插式锚点索引)
├── 00-mindset/
│   ├── README.md                    # 第一性原理 + Emacs 历史 + 架构
│   ├── concept-anchor.md            # editor: screen/entering/intro 章节导读
│   ├── intro-reading.md             # Chassell Preface/Lisp History
│   ├── exercises.md                 # 心智模型自测
│   └── capstone.md                  # 编译安装 + 5 行 init.el
├── 01-survival/
│   ├── README.md
│   ├── concept-anchor.md            # editor: basic/commands/mini/help/mark/killing/regs/search/fixit/kmacro
│   ├── drills/
│   │   ├── 01-movement.md
│   │   ├── 02-kill-ring.md
│   │   ├── 03-search.md
│   │   ├── 04-mark-rectangle.md
│   │   ├── 05-kmacro.md
│   │   ├── 06-windows-frames.md
│   │   └── 07-help-system.md        # ★最重要★
│   └── capstone.md                  # 重构 2000 行代码,全程无鼠标
├── 02-dired/
│   ├── README.md
│   ├── concept-anchor.md            # editor: dired/buffers/windows/files
│   ├── drills.md
│   └── capstone.md
├── 03-elisp-foundation/
│   ├── README.md                    # 5 周计划
│   ├── week-01-lists-eval/
│   │   ├── chassell-reading.md      # intro: List Processing + Practicing Evaluation
│   │   ├── concept-anchor.md        # editor: commands.texi
│   │   ├── exercises.md
│   │   └── artifacts/
│   ├── week-02-defun-let-if/
│   │   ├── chassell-reading.md      # intro: Writing Defuns
│   │   ├── concept-anchor.md        # editor: text.texi
│   │   └── exercises.md
│   ├── week-03-recursion/
│   │   ├── chassell-reading.md      # intro: Conditionals + Recursion
│   │   ├── concept-anchor.md        # editor: programs.texi
│   │   └── exercises.md
│   ├── week-04-lambda-mapcar/
│   │   ├── chassell-reading.md      # intro: Lambda + mapcar
│   │   ├── concept-anchor.md        # editor: regs.texi + text.texi
│   │   └── exercises.md
│   ├── week-05-capstone/
│   │   ├── chassell-reading.md      # intro: 复习全书
│   │   ├── concept-anchor.md        # editor: files.texi
│   │   └── capstone.md              # TODO 管理器
│   └── NOTES.md
├── 04-config-system/
│   ├── README.md
│   ├── concept-anchor.md            # editor: custom/modes/package/cmdargs
│   ├── ref-deep-dive.md             # ref: customize/keymaps/modes
│   ├── use-package.md
│   ├── keymaps.md
│   ├── hooks.md
│   ├── minor-mode-tutorial.md
│   └── capstone.md
├── 05-power-user/
│   ├── README.md
│   ├── concept-anchor.md            # editor: maintaining/building/text/calendar
│   ├── org-mode.md
│   ├── magit.md
│   ├── project-el.md
│   ├── completion.md
│   └── lsp-eglot.md
├── 06-elisp-deep/
│   ├── README.md                    # 8 周计划
│   ├── week-01-data-types/
│   │   ├── ref-reading.md           # intro/objects/numbers/strings/lists/sequences
│   │   ├── concept-anchor.md        # editor: text.texi
│   │   └── exercises.md
│   ├── week-02-eval-variables/
│   │   ├── ref-reading.md           # eval/control/variables/functions
│   │   ├── concept-anchor.md        # editor: commands.texi
│   │   └── exercises.md
│   ├── week-03-macros-loading/
│   │   ├── ref-reading.md           # macros/customize/loading/compile
│   │   ├── concept-anchor.md        # editor: custom.texi + package.texi
│   │   └── exercises.md
│   ├── week-04-debugging/
│   │   ├── ref-reading.md           # debugging/edebug/errors/tips
│   │   ├── concept-anchor.md        # editor: trouble.texi
│   │   └── exercises.md
│   ├── week-05-minibuf-commands/
│   │   ├── ref-reading.md           # minibuf/commands/keymaps
│   │   ├── concept-anchor.md        # editor: mini.texi + m-x.texi
│   │   └── exercises.md
│   ├── week-06-modes-files-buffers/
│   │   ├── ref-reading.md           # modes/help/files/buffers/windows
│   │   ├── concept-anchor.md        # editor: modes.texi + files.texi + buffers.texi
│   │   └── exercises.md
│   ├── week-07-text-display/
│   │   ├── ref-reading.md           # text properties/overlays/display/searching/syntax/parsing
│   │   ├── concept-anchor.md        # editor: display.texi
│   │   └── exercises.md
│   ├── week-08-processes-package/
│   │   ├── ref-reading.md           # processes/os/threads/package/internals
│   │   ├── concept-anchor.md        # editor: building.texi + cmdargs.texi
│   │   └── exercises.md
│   └── capstone.md                  # 写一个完整 major mode
├── 07-package-dev/
│   ├── README.md
│   ├── concept-anchor.md            # editor: package/trouble/misc
│   ├── ref-deep-dive.md             # ref: package/tips/compile/debugging/edebug
│   ├── package-design.md
│   ├── ert-testing.md
│   ├── docs-texinfo.md
│   ├── melpa-submission.md
│   └── capstone.md
├── 08-open-source/
│   ├── README.md
│   ├── concept-anchor.md            # editor: glossary/ack
│   ├── ref-deep-dive.md             # ref: internals/anti
│   ├── reading-source.md
│   ├── first-pr.md
│   ├── emacs-devel.md
│   └── capstone.md
└── logs/
    └── module-XX.md                 # 每个模块一份学习日志
```

**关键设计**: 每个模块都有 `concept-anchor.md` 作为 editor manual 的"穿插锚点",把用户视角的编辑器概念与 Elisp 底层实现一一对照。

---

## 各模块详细规划

### Module 0: 心法与装备 (2 天, ~6h)

**目标**: 建立 Emacs 心智模型;从源码编译安装;写出最小可用的 `init.el`。

**三本手册对照** (本模块锚点):
- **概念锚点** (editor): `emacs.texi` (总论)、`screen.texi` (Frame/Window/Buffer/Mode-line/Minibuffer 的视觉组织)、`entering.texi` (进入与退出)
- **入门实现** (intro): Chassell 的 `Preface` + `Lisp History` 章节 (Emacs 的 Lisp 渊源)
- **深度机制** (ref): `intro.texi` (Lisp ref 的导论,只是浏览,不深读)

**第一性原理讲解** (`00-mindset/README.md`):
- Emacs 起源: Stallman 1976 年在 MIT,从 TECO macros 演化而来
- GNU Emacs vs XEmacs 分裂 (1985/1991)
- 架构三层: Frame (物理窗口) → Window (Emacs 分屏) → Buffer (文本)
- 第四个关键对象: Minibuffer (一个特殊 buffer,承担命令交互)
- Mode line: 一个 buffer 的"状态卡片"
- "Living inside Emacs" 哲学 (Stallman 的极端版)

**阅读材料** (详细对照):
- 概念锚点: `emacs-manual-30.2/entering.texi`、`screen.texi`、`emacs-manual-30.2/intro.texi`
- 入门实现: `emacs-lispintro-30.2/emacs-lisp-intro.texi` 的 Preface + Why + Lisp History
- 深度机制 (略读): `emacs-lispref-30.2/intro.texi` 的开头 (了解 Elisp 与其他 Lisp 的差异)

**实践**:
1. 从源码编译 Emacs 30.2 (理解 `--with-*` 编译选项,如 `--with-native-compilation`、`--with-tree-sitter`、`--with-x-toolkit=gtk3`)
2. 启动 Emacs,完成 `M-x help-with-tutorial` (内置教程)
3. 写出最小 `~/.config/emacs/init.el` 或 `~/.emacs.d/init.el`:
   ```elisp
   (setq inhibit-startup-screen t)
   (setq make-backup-files nil)
   (global-display-line-numbers-mode 1)
   (load-theme 'modus-vivendi t)
   ```
4. 用 `C-h v` 查每个变量的文档,确认理解

**毕业检查**:
- 能口述 Emacs 的 Frame/Window/Buffer/Minibuffer/Mode-line 五层关系
- 能解释为什么 `(require 'foo)` 比 `(load "foo")` 更常用 (符号追踪 vs 文件加载)
- init.el 至少 10 行,且每一行都能用 `C-h v` 解释清楚

---

### Module 1: 编辑生存 + 自文档系统 (1 周, ~20h) ★最重要基础★

**目标**: 完全脱离鼠标;熟练使用 help system (Emacs 自我学习的能力来源)。

**三本手册对照** (本模块锚点 — editor 是主角):
- **概念锚点** (editor, 本模块主要阅读):
  - `basic.texi` (基本编辑命令)
  - `commands.texi` (命令与键位系统)
  - `mini.texi` + `m-x.texi` (minibuffer 与 M-x)
  - `help.texi` ★最重要★ (help system 全套)
  - `mark.texi` (mark/region)
  - `killing.texi` (kill ring)
  - `regs.texi` (registers)
  - `search.texi` (isearch/occur/replace)
  - `fixit.texi` (undo/拼写检查)
  - `kmacro.texi` (键盘宏)
  - `display.texi` (可选,显示选项)
- **入门实现** (intro): Chassell 的 `Practicing Evaluation` 章节 — 用 `C-x C-e` eval 表达式
- **深度机制** (ref, 只在遇到疑问时查阅): `keymaps.texi`、`commands.texi` (ref 版)

**第一性原理讲解** (每个概念用三层结构讲):
- 例: Kill ring
  - **概念** (editor/`killing.texi`): "杀死" 文本进 kill ring,`C-y` 召回,`M-y` 循环前一项
  - **入门** (intro): 用 Elisp 写 `(kill-new "hello")` 然后看 `kill-ring` 变量
  - **深度** (ref/`buffers.texi` 相关): `kill-new`、`kill-append`、`current-kill` 的源码与 `interprogram-cut-function`

**第一性原理讲解**:
- **为什么是 C-/M- 前缀?** Space-cadet 键盘设计,Control 和 Meta 是真实按键
- **Kill ring 的本质**: 一个有循环指针的环状栈。比 clipboard 多了"历史"和"循环"
- **Mark ring**: 类似 cursor 历史的栈。`C-SPC C-SPC` 推入,`C-u C-SPC` 弹出
- **Prefix argument 的设计哲学**: 一个统一的原语,影响几乎所有命令的行为 (数字 = 重复,C-u = 4 倍递进)
- **Why Isearch 是递增的?** 1970s 的革命性想法:反馈循环越短,认知负担越低

**Drills** (`01-survival/drills/` 共 7 个文件, ~155 题):

`01-movement.md` (30 题): 字符 → 词 → 句 → 段 → defun → buffer 头尾 → 行首尾 → 配对的括号

`02-kill-ring.md` (20 题): `C-w`/`M-w`/`C-y`/`M-y`/`M-d`/`M-k`/`C-k`;kill-ring 操作;append next kill

`03-search.md` (25 题): `C-s`/`C-r` (isearch) → `M-s o` (occur) → `M-%` (replace) → `C-M-%` (regexp replace) → `M-s i` (imenu) → `M-s h` (highlight)

`04-mark-rectangle.md` (20 题): `C-SPC` mark → `C-x C-x` 交换 → `C-x r k`/`C-x r y`/`C-x r o`/`C-x r t` 矩形操作 → `C-M-@` mark sexp → `C-x h` mark whole buffer

`05-kmacro.md` (15 题): `F3`/`F4` 录制运行 → `C-x C-k r` 应用到 region → `C-x C-k e` 编辑宏 → `C-x C-k n` 命名 → `kmacro-bind-to-key`

`06-windows-frames.md` (20 题): `C-x 2/3/1/0/o` 窗口 → `C-x 5` 框架 → `C-x ^`/`C-x }`/`C-x {` 调整大小 → `C-x 4`/`C-x 5` 在新窗口/框架打开 buffer → windmove 包

`07-help-system.md` (25 题) ★最重要★: `C-h f` (function) → `C-h v` (variable) → `C-h k` (key) → `C-h w` (where-is) → `C-h m` (mode) → `C-h b` (bindings) → `C-h i` (info) → `C-h r` (Emacs manual) → `C-h F` (command 历史来源) → `C-h S` (info symbol)

**毕业项目** (`capstone.md`):
- 拿一份 2000 行杂乱代码 (任意语言)
- 完成: 函数重排、变量改名、注释批量添加、4 处搜索替换、5 处矩形编辑
- 全程不离开键盘,录屏自检 (用 kmacro 录下来自己看)
- 时间预算: 30 分钟内完成

---

### Module 2: Dired + Buffer/Window 管理 (4 天, ~12h)

**目标**: 把文件管理从 IDE/文件管理器迁到 Emacs 内。

**三本手册对照**:
- **概念锚点** (editor):
  - `dired.texi` (Dired 完整章节)
  - `buffers.texi` (buffer 概念)
  - `windows.texi` (window 管理)
  - `files.texi` (访问文件、自动保存、备份)
- **入门实现** (intro): Chassell 中 `Buffer Names`、`Getting Buffers`、`Switching Buffers`、`Buffer Size & Locations` 章节 (已经读过,这里复习并对应到 editor 概念)
- **深度机制** (ref): `buffers.texi`、`windows.texi`、`files.texi` (ref 版,精读 buffer-local 变量机制)

**第一性原理讲解**:
- Dired 是 buffer — 一个目录被渲染成文本,你可以像编辑代码一样编辑它
- WDired (`C-x C-q`): 直接在 dired buffer 里 rename 文件,改完 `C-c C-c` 提交
- Register: 命名 slot,可以存文本、位置、矩形、窗口配置
- Buffer 命名约定: `*Buffer List*`、`*Help*`、`*Messages*`、`*Scratch*`

**Drills** (~40 题):
- Dired 基础: 标记 (`m`/`u`/`U`/`%m`)、操作 (`D`/`R`/`C`/`+`)
- 子目录递归: `i` (insert 子目录) vs `^` (上一级)
- WDired: `C-x C-q` → 改文件名 → `C-c C-c`
- 正则标记: `% m` (按名标记)、`* /` (标记目录)、`* s` (标记所有)
- Dired+ 操作: `Q` (query-replace 文件内容)、`A` (search 内容)、`=` (diff)
- ibuffer (`C-x C-b`): 比 `buffer-menu` 更强,按组分类

**毕业项目**:
- 拿一个混乱的下载目录,用 dired 在 10 分钟内整理成分类子目录
- 全程在 Emacs 内完成

---

### Module 3: Elisp 入门 (Chassell) (5 周, ~100h) ★核心★

**目标**: 系统读完 `emacs-lispintro-30.2`,建立 Lisp 思维。

**核心材料**: `emacs-lispintro-30.2/emacs-lisp-intro.texi` 全文 (~22k 行)。

**穿插的 editor 概念锚点** (每周一章对照):

| 周 | Chassell 章节 | editor 锚点 | 对照讲解 |
|---|---|---|---|
| 1 | List Processing, Practicing Evaluation | `commands.texi` 复习 | "命令的本质是 Elisp 函数" |
| 2 | Writing Defuns (defun/let/if) | `text.texi` (基本文本概念) | defun 出来的函数可以挂在键位上 |
| 3 | Conditionals, Recursion | `programs.texi` (程序编辑) | 写一个自动缩进/括号匹配的小工具 |
| 4 | Lambda, mapcar, funcall | `regs.texi` (registers) + `text.texi` | 用 lambda 实现 register 内容的批量处理 |
| 5 | Capstone: TODO 管理器 | `files.texi` (find-file/save-buffer) | todo 数据持久化到文件 |

**为什么这本书**:
- 为非程序员写,从零开始
- 每个概念都有真实可跑的例子
- 包含练习题和答案
- 教 Lisp 哲学 (递归、cons、引用、闭包) 而非语法

**5 周拆解**:

| 周 | 主题 | 章节 | 产出 |
|---|---|---|---|
| 1 | Lists + Evaluation | List Processing + Practicing Evaluation | 写 10 个 list 操作函数 |
| 2 | defun + let + if | Writing Defuns + Buffer-related | 5 个实用 buffer 函数 |
| 3 | Conditionals + Recursion | Conditionals + Recursion | 实现 length/recursion-based utilities |
| 4 | Lambda + mapcar + sequences | Lambda Functions + funcall + mapcar | 重构日常重复代码 |
| 5 | Capstone: 写一个 TODO 管理器 | 综合 | 完整的 todo 包原型 |

**每周模板**:
```
week-XX/
├── README.md          # 本周目标 + 章节范围
├── notes.md           # 重点摘录
├── exercises.md       # 我自己写的练习 (不是手册里的)
└── artifacts/         # 本周写的代码
```

**毕业项目** (`capstone.md`):
- 写一个简化版 `todo-mode`
- 要求: `M-x todo-add`、`M-x todo-list`、`M-x todo-done`、`M-x todo-save`、`M-x todo-load`
- 数据持久化到 `~/.emacs.d/todos.el`
- 用 `define-derived-mode` 实现 major mode
- 至少 200 行带注释的代码

**第一性原理讲解**:
- **为什么 list 是核心?** Lisp = "LISt Processing";所有代码就是 list;代码即数据 (homoiconicity)
- **`'foo` 是什么?** `(quote foo)` 的语法糖,意思是"不要 eval 这个,给我符号本身"
- **为什么 defun 用 list?** 一个函数 = 名字 + 参数列表 + 文档 + body = 一个嵌套 list
- **Lexical vs Dynamic binding**: Module 3 会讲核心区别,这是 Emacs 25+ 默认 Lexical

---

### Module 4: 配置体系 + 第一个 Minor Mode (2 周, ~40h)

**目标**: 写出有自己风格的 init.el;定义一个 minor mode。

**三本手册对照**:
- **概念锚点** (editor):
  - `custom.texi` (用户级定制,custom 系统)
  - `modes.texi` (major/minor mode 用户视角)
  - `package.texi` (包安装与使用)
  - `cmdargs.texi` (命令行参数与环境变量)
- **入门实现** (intro): Chassell 中 `defun`、`defvar`、`setq` 章节已读过,这里学 `defcustom`、`define-key`
- **深度机制** (ref):
  - `customize.texi` (defcustom 完整机制)
  - `keymaps.texi` (keymap 完整: current-active-maps、key-binding 查找)
  - `modes.texi` (ref 版,define-derived-mode、define-minor-mode 内部)

**第一性原理讲解**:
- `init.el` 就是普通 Elisp 代码,启动时被 `load` 一次。没有"配置文件"这个特殊概念。
- `use-package` 是宏,把"加载 + 配置 + keymap + hook"打包成一个声明式 DSL
- keymap 是嵌套的 hash table,从 event 序列查到 command 符号
- minor mode 是 4 个东西的打包: 一个变量、一个 toggle 函数、一个 keymap、一个 lighter

**实践**:
1. 从零写 init.el,模块化:
   ```
   ~/.config/emacs/
   ├── init.el              # 入口
   ├── early-init.el        # 包管理
   └── lisp/
       ├── init-basics.el
       ├── init-ui.el
       ├── init-editing.el
       ├── init-prog.el
       └── init-org.el
   ```
2. 用 `use-package` 安装管理 20+ 个包
3. 写一个自己的 minor mode `my-editing-mode`,绑定一组键 + 一个 lighter

**毕业项目**:
- 500 行模块化 init.el
- 至少 1 个自定义 minor mode
- 用 `M-x report-emacs-bug` 风格文档化自己的配置

---

### Module 5: Power User 工作流 (3 周, ~60h)

**目标**: 把日常编辑器工作流完全迁到 Emacs。

**三本手册对照**:
- **概念锚点** (editor, 本模块的"包生态"都有对应手册章节):
  - Org-mode 概念锚点: `text.texi` (大纲、fill、auto-fill) + `mule.texi` (Unicode)
  - Magit/版本控制: `maintaining.texi` (VC 体系,Emacs 内置 Git/ SVN 接口)
  - 编译: `building.texi` (compile、grep、flymake 用户视角)
  - 日历/计划: `calendar.texi`、`text.texi`
- **入门实现** (intro): 复习 Chassell 中有关 buffer/region 操作的章节 (这些是 org 表格、magit diff 的底层)
- **深度机制** (ref): `processes.texi` (eglot/magit 都基于 process)、`text.texi` (ref 版,org 表格依赖 text property)

**重点学习** (每个 4-5 天):

**Org-mode** (`05-power-user/org-mode.md`):
- capture (模板化快速记录)
- agenda (跨文件 agenda 视图)
- org-babel (literate programming,在 org 里跑代码块)
- org-table (强大的表格 + 公式)
- org-roam (可选,Zettelkasten)

**Magit** (`05-power-user/magit.md`):
- `magit-status` (`C-x g`) 主入口
- Stage/commit/push 全流程
- Branch/rebase/cherry-pick
- Ediff 处理合并冲突
- Forge (集成 GitHub PR/issue)

**项目与补全**:
- `project.el` (内置) 或 `projectile`
- `vertico` + `corfu` + `orderless` + `consult` 现代补全栈
- `eglot` (内置 LSP client) + `flymake`
- `which-key` (按键提示)

**毕业项目**:
- 用 org-mode 写一个 literate 配置文档 (`config.org` → tangling 到 `init.el`)
- 用 org-babel 跑 Python/Bash 代码块解决一个真实任务
- 用 Magit 完成一个 feature 分支: branch → commit → push → PR

---

### Module 6: Elisp 深度 (Reference) (8 周, ~160h) ★核心★

**目标**: 精读 `emacs-lispref-30.2` 核心章节,达到能读懂任何 elisp 代码的水平。

**核心材料**: `emacs-lispref-30.2` (~111k 行,不全读,选核心章节精读)。

**8 周拆解** (editor manual 穿插锚点):

| 周 | Ref 章节 | Editor 锚点 | 对照讲解 |
|---|---|---|---|
| 1 | intro, objects, numbers, strings, lists, sequences | `text.texi` (字符/字/句子定义依赖 syntax table) | 数据类型 vs 编辑单位 |
| 2 | eval, control, variables, functions | `commands.texi` (interactive 让函数变命令) | eval 流程 + interactive |
| 3 | macros, customize, loading, compile | `custom.texi` + `package.texi` (用户视角的 defcustom/package) | defmacro 如何让 use-package 工作 |
| 4 | debugging, edebug, errors, tips | `trouble.texi` (报 bug 流程) | Edebug + report-emacs-bug |
| 5 | minibuf, commands, keymaps | `mini.texi` + `m-x.texi` + `commands.texi` | completing-read 与 Vertico 的关系 |
| 6 | modes, help, files, buffers, windows | `modes.texi` + `files.texi` + `buffers.texi` | mode 的 hook、find-file-not-found-function |
| 7 | text properties, overlays, display, searching, syntax, parsing | `display.texi` (用户视角的字体/颜色) | font-lock 用 text property 实现 |
| 8 | processes, os, threads, package, internals | `building.texi` (compile/grep 用 process) + `cmdargs.texi` | make-process 跑 git/ripgrep |

**每周作业** (`06-elisp-deep/week-XX/exercises.md`):
- 每章 10-15 题,要求读懂指定函数源码 (用 `C-h f RET click source`)
- 至少 3 题"实现一个简化版 X" (如简化版 `mapconcat`)

**第一性原理讲解** (每周 README):
- **数据即代码** (homoiconicity): `(eval '(+ 1 2))` 为什么返回 3
- **Symbol vs Variable vs Function**: 一个符号可以同时绑定变量值和函数值 (Lisp-2)
- **Buffer 是 mutable string + point + mark + text properties**
- **Text property vs overlay**: text property 持久写入 buffer,overlay 是临时叠加 (类似图层)
- **Hook 的本质**: 一个 buffer-local 或 global 的变量,里面是一个函数列表

**毕业项目** (`capstone.md`):
- 写一个完整 major mode `mylang-mode`
- 要求: 语法高亮 (font-lock)、缩进 (indentation function)、imenu 支持、company/corfu 补全后端
- 至少 500 行带文档的代码
- 用 ERT 写至少 10 个测试

---

### Module 7: 写一个真实的包 (3 周, ~60h)

**目标**: 设计、实现、测试、文档化、发布一个真实可用的包到 MELPA。

**三本手册对照**:
- **概念锚点** (editor): `package.texi` (用户视角的包管理) + `trouble.texi` (如何 report bug) + `misc.texi` (各种小特性)
- **入门实现** (intro): 复习 Chassell 中 `defun`/`let`/`if`/`save-excursion` 章节
- **深度机制** (ref):
  - `package.texi` (ref 版,package-archive 接口、目录约定)
  - `tips.texi` (代码风格 ★必读★)
  - `compile.texi` (byte-compile 注意事项)
  - `debugging.texi` + `edebug.texi` (ERT 与 Edebug)

**步骤** (`07-package-dev/`):

1. **选题** (`package-design.md`):
   - 找一个自己真实有的痛点 (不是想象的需求)
   - 调研现有方案,确认没解决你的场景
   - 写一个 1 页 README 描述"为什么"

2. **设计** (`package-design.md`):
   - 列出所有 interactive commands
   - 列出所有 defcustom (用户可配置)
   - 列出 hooks (其他包/用户可挂载)
   - 列出 minor mode / major mode
   - 画一个状态机或时序图

3. **实现**:
   - TDD: 先写 ERT 测试,再实现
   - 每个函数都有 `;;; Commentary` 和 docstring
   - 遵循 `emacs-lispref-30.2/tips.texi` 的代码风格

4. **测试** (`ert-testing.md`):
   - ERT 单元测试 (ert-deftest)
   - 在干净的 Emacs (emacs -Q) 里手动测试
   - 测试多个 Emacs 版本 (26/27/28/29/30)

5. **文档** (`docs-texinfo.md`):
   - 至少 README.md (Markdown)
   - 推荐 Texinfo 手册 (像官方包那样)
   - 网站可以读 `package-design.md` 中的说明

6. **发布** (`melpa-submission.md`):
   - 提交到 GitHub
   - Fork `melpa/melpa`,加 recipe 到 `recipes/`
   - 提 PR,等 review

**毕业项目**: 你的包被 MELPA 合并,可以 `M-x package-install RET your-package RET`。

---

### Module 8: 开源贡献 (持续)

**目标**: 给已有的开源 Emacs 包或 Emacs core 贡献代码。

**三本手册对照**:
- **概念锚点** (editor): `glossary.texi` (Emacs 术语,跟社区沟通必备) + `ack.texi` (主要贡献者名单,了解历史)
- **入门实现** (intro): Chassell 的 `Thank You` 章节 (致谢社区,理解文化)
- **深度机制** (ref): `internals.texi` (Emacs 内部实现,要碰 core 必读) + `anti.texi` (反特性,为什么 Emacs 不做某些事)

**入门** (`08-open-source/README.md`):
- 订阅 `emacs-devel@gnu.org` 邮件列表 (只读 1 个月,先了解文化)
- 关注 r/emacs、Hacker News 的 Emacs 标签
- 在 GitHub star 10 个你常用的包

**第一次贡献流程** (`first-pr.md`):
1. 选一个你重度使用的包 (如 `magit`、`org`、`vertico`、`use-package`)
2. 看它的 issue tracker,找标注 "good first issue" 的
3. Fork、branch、fix、test
4. 用 `M-x report-emacs-bug` 风格写 commit message
5. 提 PR,接受 review

**读源码方法论** (`reading-source.md`):
- 从 `package.el` 找包安装路径 (`M-x package-describe-file` 或 `(package-user-dir)`)
- 用 `g d` (xref-find-definitions) 跳转定义
- 用 `g r` (xref-find-references) 找所有调用点
- 用 `M-xSpeedbar` 或 `imenu` 在大文件中导航
- 推荐先读的好包: `use-package` (宏)、`dash.el` (list 库)、`s.el` (string 库)、`f.el` (file 库)

**Emacs core 贡献** (`emacs-devel.md`):
- `git clone https://git.savannah.gnu.org/git/emacs.git`
- 阅读 `CONTRIBUTE` 和 `admin/notes/`
- 走 `M-x report-emacs-bug` 流程,patch 走邮件 (Emacs 不接受 GitHub PR)
- copyright assignment: FSF 要求大贡献者签 paper (一次性,后续可继续贡献)

**毕业项目**: 一个被合并到非自己包的 PR。

---

## PROGRESS.md (学习进度跟踪表)

实施时创建,模板:
```markdown
# 学习进度

| 模块 | 状态 | 开始日期 | 完成日期 | 实际用时 | 备注 |
|---|---|---|---|---|---|
| 0. 心法 | ⬜ 未开始 | | | | |
| 1. 生存 | ⬜ | | | | |
| 2. Dired | ⬜ | | | | |
| 3. Elisp 入门 | ⬜ | | | | |
| 4. 配置体系 | ⬜ | | | | |
| 5. Power User | ⬜ | | | | |
| 6. Elisp 深度 | ⬜ | | | | |
| 7. 写包 | ⬜ | | | | |
| 8. 开源贡献 | ⬜ | | | | |
```

---

## 验证 (How to Verify)

### 每模块的"毕业检查"
- Module 0: init.el + 编译的 Emacs
- Module 1: 30 分钟无鼠标重构 2000 行代码,录屏自检
- Module 2: 10 分钟内用 dired 整理混乱目录
- Module 3: 写出 200+ 行的 todo-mode 包原型
- Module 4: 500 行模块化 init.el + 1 个自定义 minor mode
- Module 5: org literate 配置 + 用 Magit 提 PR
- Module 6: 写出 500+ 行的 major mode with font-lock + indent + imenu + ERT tests
- Module 7: 包被 MELPA 合并
- Module 8: 一个被合并到他人包的 PR

### 整体里程碑
- **3 个月** (Module 0-4 完成): 完全脱离鼠标,有自己的 init.el,能读懂大部分 elisp 代码
- **6 个月** (Module 0-7 完成): 写过完整的 major mode,发布过自己的包
- **9 个月+** (持续贡献): 给 melpa 包贡献过 PR,读 emacs-devel

### 真实能力测试 (Module 6 完成后能答出来)
1. 解释 `lexical-binding: t` 这个 file-local variable 干了什么
2. 描述 `(defun foo () (let ((x 1)) (lambda () x)))` 返回的闭包行为
3. 区分 `(eq 'a 'a)` `(eq "a" "a")` `(equal "a" "a")` `(eql 1 1.0)`
4. 写一个 `make-process` 跑 `git status` 并把输出插入当前 buffer
5. 解释 `font-lock-defaults` 的结构
6. 写一个 `define-derived-mode` 继承 `prog-mode`
7. 解释 `with-current-buffer` 与 `save-excursion` 的区别
8. 用 Edebug 单步执行一个函数

---

## 实施顺序 (退出 Plan Mode 后)

1. **创建目录结构** — `mkdir -p` 所有 Module 目录
2. **写 `ROADMAP.md`** — 路线图精简版,本计划文件的对外版本
3. **写 `MANUAL-CROSSREF.md`** ★关键★ — 三本手册的完整章节对照表 (哪些章节对应哪个模块),作为穿插式学习的总索引
4. **写 `PROGRESS.md`** — 进度跟踪表
5. **写 `00-mindset/` 全套** — README + concept-anchor + intro-reading + exercises + capstone
6. **写 `01-survival/` 全套** — README + concept-anchor + 7 个 drill 文件 + capstone
7. **写 `02-dired/` 全套**
8. **写 `03-elisp-foundation/` 全套** — 5 周,每周一组文件 ★最大工作量★
9. **写 `04-config-system/` 全套**
10. **写 `05-power-user/` 全套**
11. **写 `06-elisp-deep/` 全套** — 8 周,每周一组文件 ★次大工作量★
12. **写 `07-package-dev/` 全套**
13. **写 `08-open-source/` 全套**

**预计写作工作量**: 写完所有教程内容约需 40-60 个大文档,每个文档 800-2000 行 (因为要 self-contained,每个文件都包含完整教学而不只是大纲)。可以分批写,先写 Module 0-3 + MANUAL-CROSSREF,后续按用户进度推进。

**写作纪律**:
- 每个 drill/exercise 文件必须通过"自包含校验清单"才能算完成
- 写完后用 `wc -l` 检查,文件应 ≥ 500 行 (高密度内容)
- 包含至少 1 张 ASCII 图示、3 个代码块、10 道练习题
- 不允许出现"详见原手册第 X 节"这类偷懒表述,必须内联讲完

---

## 备注

- 本计划严格遵循"Elisp 优先"原则 (用户偏好),但通过 **editor manual 穿插式锚点** 让每个 Elisp 概念都有"用户视角"对照
- 三本手册形成三层结构: editor = What/Why,intro = How simple,ref = How complete
- editor manual 不再是"参考资料",而是**每个模块的必修概念锚点** (concept-anchor.md)
- 两个 Lisp 手册是核心,必须深度精读
- ★**教程完全自包含**★: 读者不需要翻开原始 texi 文件,所有内容都被吸收到教程中 (参考"自包含校验清单")
- 每个模块的"毕业项目"是硬性检验,必须完成才进入下一模块
- 用户可以随时根据实际进度调整时间预算,但**不要跳模块**

---

## 样章示范 (Tutorial Sample) — 展示"讲解+实践皆备"的真实深度

下面是 Module 1 / Drill 02 (`02-kill-ring.md`) 的完整内容样例,作为后续所有教程单元的深度参考。每个 drill/exercise 都按这种风格写。

---

### 样例: `01-survival/drills/02-kill-ring.md` (节选)

```markdown
# Drill 02: Kill Ring — 死亡与重生的栈

> 对应: emacs-manual/killing.texi 第 9-12 节
> 预计用时: 90 分钟
> 难度: ★★☆☆☆

## 0. 学习目标

读完这个 drill,你应该能够:

- [ ] 解释 kill 与 delete 的本质区别
- [ ] 描述 kill-ring 的数据结构 (它是一个 list,有 `kill-ring-yank-pointer`)
- [ ] 在 30 秒内完成: 杀一行 → 召回 → 替换为前一次杀的内容
- [ ] 用键盘宏批量处理 50 个相似的"杀-改-粘"
- [ ] 用 Elisp 写一个把当前行 kill 到 register 的命令

## 1. 第一性原理: 为什么是 "Kill" 不是 "Cut"?

### 1.1 历史背景

1970s 的 TECO 和 Emacs 都运行在 paper terminal 上。删除一段文字,物理上就是"消失"。
Stallman 的设计直觉是:**被删除的文字应该有家可归**。
于是有了 kill ring: 一个**有循环指针的栈**。

### 1.2 Kill vs Delete 的核心区别

| 操作 | 进入 kill-ring? | 行为 |
|---|---|---|
| `C-w` (kill-region) | ✅ | 杀掉的文本进 kill-ring |
| `M-d` (kill-word) | ✅ | 同上 |
| `M-k` (kill-sentence) | ✅ | 同上 |
| `C-k` (kill-line) | ✅ | 同上 |
| `DEL` (delete-backward-char) | ❌ | 真删除,无召回 |
| `M-DEL` (backward-kill-word) | ✅ | 杀掉进 ring |
| `M-\` (delete-horizontal-space) | ❌ | 真删除 |

**第一性原理**: 进入 kill-ring 的命令都带 "kill" 字样;真删除的命令带 "delete"。
记住这条规则,你就不需要背每个键。

### 1.3 Kill-ring 的数据结构

打开 `*scratch*`,输入并 eval (`C-x C-e`):

```elisp
kill-ring
;; => ("最近杀的" "前一次" "再前一次" ...)
```

```elisp
kill-ring-yank-pointer
;; => ("最近杀的" "前一次" ...) — 指向当前 yank 会取的那个
```

`C-y` 召回 `car` of `kill-ring-yank-pointer`。
`M-y` (after `C-y`) 把指针向后移一位,并替换刚才 yank 的内容。

**画图理解** (在纸上画一遍):

```
kill-ring:        [A] -> [B] -> [C] -> nil
                   ^
                   |
yank-pointer ------+
```

`M-y` 后:

```
kill-ring:        [A] -> [B] -> [C] -> nil
                           ^
                           |
yank-pointer --------------+
```

到尾部后**循环回 A** (这是 kill-ring 与普通栈的本质区别)。

## 2. 内联讲解 (30 分钟) — 替代 killing.texi 第 9-12 节

### 2.1 Kill 命令家族对照表

| 命令 | 键位 | 杀什么 | 进 kill-ring? |
|---|---|---|---|
| `kill-region` | `C-w` | 选中的 region | ✅ |
| `kill-word` | `M-d` | 光标后一个 word | ✅ |
| `backward-kill-word` | `M-DEL` / `C-M-w`* | 光标前一个 word | ✅ |
| `kill-line` | `C-k` | 光标到行尾 | ✅ |
| `kill-sentence` | `M-k` | 光标到句尾 | ✅ |
| `kill-paragraph` | 无默认键 | 当前段 | ✅ |
| `kill-sexp` | `C-M-k` | 一个 sexp | ✅ |
| `zap-to-char` | `M-z CHAR` | 到下一个 CHAR (含) | ✅ |
| `delete-backward-char` | `DEL` | 前一字符 | ❌ |
| `delete-char` | `C-d` | 后一字符 | ❌ |
| `delete-horizontal-space` | `M-\` | 周围空白 | ❌ |

\* `backward-kill-word` 默认是 `C-M-backspace` 或 `M-DEL`,有些终端会拦截。

**记忆口诀**: "kill" = 入环,"delete" = 真删。

### 2.2 `C-k` 的细节 (新手最容易踩的坑)

打开一个空 buffer,输入 `abcdef`,光标在 `a`,按 `C-k`:

```
abcd|ef        ← 之前
abcd|          ← 之后 (光标没动,kill 了 "ef" 但没动换行)
```

如果 `abcdef` 是 buffer 的最后一行 (无换行),按 `C-k` 后:

```
abcdef|        ← 之前
|             ← 之后 (空 buffer,光标在行首)
```

按 `C-u C-k` 或 `C-k C-k` (连按两次):

```
|ef           ← 之后 (kill 了 "abcd" + 换行,光标在新行行首)
```

**底层机制** (内联源码讲解,不是引用):

`kill-line` 调用 `kill-region`,后者调用 `kill-new` 把文本推入 `kill-ring`。
但如果区域为空 (光标就在行尾),`kill-line` 会先调用 `(forward-line 1)`,然后 kill 该位置。
这就是为什么 `C-k` 在行尾会"跨行"。

### 2.3 Append Next Kill (`C-M-w`)

通常每次 `kill-*` 都会在 kill-ring 里增加一项。但有时你想合并多次 kill:

- `C-w` (kill 第一段)
- `C-M-w` (宣告下次 kill append,不增加新项)
- `C-w` (kill 第二段,这次与前一次合并)

**底层**: `interprogram-cut-function` 和 `kill-new` 检查 `last-command`:
如果是 `(kill-region kill-append ...)`,就把新文本 append 到 `car kill-ring`,而不是 cons 新项。

### 2.4 Yanking (`C-y` 和 `M-y`)

`C-y` 调用 `yank`,做两件事:
1. 把 `kill-ring-yank-pointer` 重置到 `kill-ring` 头部
2. 在 point 插入 `(car kill-ring-yank-pointer)`,并把 point 设到插入文本的起始

紧接着按 `M-y` 调用 `yank-pop`:
1. 检查 `last-command` 是不是 `yank`,否则报错 "Previous command was not a yank"
2. 把 `kill-ring-yank-pointer` 向后移动一位 (循环到尾后回头部)
3. 删除刚才 yank 的文本 (`(car (last kill-ring-yank-pointer))` 算偏移)
4. 插入新的 car

**循环机制**:

```elisp
;; 简化版 yank-pop 内部逻辑
(setq kill-ring-yank-pointer
      (or (cdr kill-ring-yank-pointer)  ; 向后移
          kill-ring))                    ; 到尾了,循环回头
```

### 2.5 Kill-ring 的容量与配置

```elisp
kill-ring-max        ;; 默认 60,可以改大
kill-ring            ;; 实际的 list
kill-ring-yank-pointer ;; 游标
interprogram-cut-function  ;; 同步到系统剪贴板的函数 (GUI 下)
interprogram-paste-function ;; 从系统剪贴板读
```

**练习 2.5.1** (5 分钟): 在 `*scratch*` 里 eval `(setq kill-ring-max 200)`,然后连续 kill 100 段,用 `(length kill-ring)` 验证。

**练习 2.5.2** (10 分钟): 写一个函数 `(my-kill-ring-size)`,返回当前 kill-ring 长度,绑到 `C-c k s`。

### 2.6 与 Register 的对比

| 维度 | Kill ring | Register |
|---|---|---|
| 数据结构 | list (FIFO 环) | 命名 slot (a-z 0-9) |
| 命名 | 匿名 | 你给名字 |
| 容量 | `kill-ring-max` (60 默认) | 任意多 |
| 持久化 | 进程结束就没 | 同 (除非 `register-alist` 持久化) |
| 用途 | 短期多次 yank | 长期持有特定文本/位置 |

Kill ring 像"近期剪贴板历史",register 像"我标记的固定位置"。

## 3. 微练习 (40 分钟,共 25 题)

每题先在 `*scratch*` 或一个临时 buffer 里准备文本,然后操作,最后用 `C-/` 撤销。

### 基础 (★)

1. 输入 "hello world",光标在行首,杀 "hello " (用 `M-d`)
2. 召回 (用 `C-y`)
3. 召回前一次 (用 `C-y M-y`)
4. 连续按 `M-y` 5 次,观察指针循环
5. 杀一整行 (`C-k` 一次,观察是否包含换行)
6. 杀一整行含换行 (`C-k` 两次,或 `C-S-backspace`)
7. 杀到下一个 "world" (`M-z world RET`)

### 中级 (★★)

8. 杀一个 sentence (`M-k`),注意 sentence 的定义 (用 `C-h v sentence-end-double-space`)
9. 杀一段 paragraph (`M-{` 移到段头,`M-h` mark-paragraph,`C-w`)
10. 杀矩形区域 (`C-SPC` 起始,移动到结束,`C-x r k`)
11. **append next kill**: 杀第一段 (`C-w`),按 `C-M-w`,杀第二段,观察 kill-ring 只增加一项
12. 用 numeric prefix: `C-u 3 M-d` 一次杀 3 个 word

### 高级 (★★★)

13. 写一个函数: 把当前行 (不含换行) 追加到 register `a`:

    ```elisp
    (defun my-kill-line-to-register-a ()
      "Kill current line (without newline) into register a."
      (interactive)
      (let ((beg (line-beginning-position))
            (end (line-end-position)))
        (copy-to-register ?a beg end t))) ; t = delete source
    ```
    绑定到 `C-c k a`,试用。

14. 用 `M-x list-kill-ring-queue` 看完整 kill-ring (这是 browse-kill-ring 包,如果你装了)
    或者直接 eval `kill-ring` 在 `*scratch*` 里看
15. 写一个 minor mode 让 `M-y` 总是弹出 menu (研究 `consult-yank-from-kill-ring` 怎么做的)
16. 你杀的内容会自动同步到系统剪贴板 (在 X/Wayland GUI 下)。研究变量 `interprogram-cut-function`,理解机制。
17. **Capstone 微版**: 写一个命令 `my-kill-ring-rotate`,把 `kill-ring-yank-pointer` 强制重置到 head。

## 4. 实战练习 (20 分钟)

打开 `/home/sun/src/learning/emacs-learning/emacs-manual-30.2/killing.texi`,完成:

1. 把所有 `@section` 标题改成大写 (用 `C-x r t` string-rectangle)
2. 复制所有 `@code{...}` 内的命令名到 buffer 末尾
3. 用 kmacro 录一个"找到下一个 `@code`,复制其中的文本"的宏,运行 20 次

全程不要用鼠标。

## 5. 自测 (5 分钟)

回答以下问题 (合上手册):

1. `C-k` 在行首按一次,会杀掉什么? 按两次呢?
2. `kill-ring` 默认长度是多少? 用哪个变量改?
3. `M-y` 不在 `C-y` 之后直接按,会怎么样?
4. 为什么 Emacs 把"删除"分成 kill/delete 两套?
5. 解释 `kill-ring-yank-pointer` 的作用,以及它如何实现 `M-y` 的循环。

**答案** (反白查看):
> 1. 按一次杀到行尾不含换行 (除非行尾就是 buffer 末尾);按两次杀到下一行行首 (含换行)
> 2. `kill-ring-max` 默认 60
> 3. 报错 "Previous command was not a yank"
> 4. 因为有些操作你确定不要 (如 `DEL` 退格),有些可能要召回。两套语义清晰。
> 5. 它是 kill-ring 的一个"游标",指向当前 yank 会取的位置。`M-y` 把它向后移动 (到末尾后回环),并 replace-region 替换刚才 yank 的内容。

## 6. 毕业检查

- [ ] 能在 30 秒内: 杀一段 → yank → M-y 替换为前两次杀的内容
- [ ] 能解释 kill-ring 是 list + yank-pointer 的组合,不是栈
- [ ] 能写出第 13 题的函数,并理解 `copy-to-register` 的第 4 个参数 `delete-flag`
- [ ] 完成第 4 节的实战练习

完成后在 `logs/module-01.md` 记录用时与心得,进入下一个 drill `03-search.md`。

## 7. 延伸阅读 (可选)

- `emacs-lispref/text.texi` 中关于 kill 相关函数 (`kill-new`, `kill-append`, `current-kill`) 的源码
- `browse-kill-ring` 包的源码 (~200 行,很好的 elisp 阅读材料)
- Emacs Wiki: KillRing (社区历史与讨论)
```

---

**说明**: 这只是一个 drill 文件的样例。所有 ~80 个 drill/exercise 文件都按这个深度写,确保每个概念都有:
1. 第一性原理讲解 (Why) — **内联**,不是"去读手册"
2. 内联教学 (Read) — 把手册内容直接转述、扩展、配上源码与图示
3. 循序渐进的微练习 (Drill) — 每节有练习
4. 实战综合练习 (Build)
5. 自测题 (Check)
6. 延伸阅读 (Go deeper)

### 自包含校验清单 (每个教程文件发布前自查)

- [ ] 读者**不打开**原始 texi 文件,也能完全理解本节内容
- [ ] 所有手册中的表格、图示、关键代码示例都被**内联**到教程中
- [ ] 每个关键概念有 ≥ 2 个对比例子 (正例 + 反例 / 边界)
- [ ] 每节末尾有 ≥ 3 个可执行练习 (在 `*scratch*` 或临时 buffer 里能跑)
- [ ] 所有命令、变量、函数都有 `C-h X` 的查询提示 (授人以渔)
- [ ] 第一性原理讲解 (Why) 占内容 ≥ 30%,不是只讲 How


