# 三本手册章节对照索引 (Manual Cross-Reference)

> 这个文件是穿插式学习的**总导航**: 告诉你"我现在在学的内容,对应三本手册的哪些章节"。
>
> ★ 重要约定: 配套教程已**完全吸收**这些章节的内容,你不需要再去翻原始 texi 文件。
> 这个索引只是用来: (1) 检查覆盖完整性; (2) 当你想"深入原典"时知道去哪查。

---

## Legend (图例)

- **● 精读 + 实践** (必学): 教程完整转述手册章节,加上第一性原理 + 实战练习
- **○ 实践性浏览**: 教程讲解核心概念 + 操作练习,跳过罕见细节
- **△ 速览/平台附录**: 教程简略提及,知道存在即可
- **M{n}**: 归属模块编号 (M0 到 M8)
- **W{n}**: 模块内的第 n 周

---

## 1. emacs-lispintro (Chassell, 17 章)

> 全部在 Module 3 精读。Chassell 这本书是为非程序员写的 Emacs Lisp 入门,
> 是建立 Lisp 思维的最佳起点。

| # | Chassell 章节 | 归属 | 深度 | 教程文件 |
|---|---|---|---|---|
| 1 | List Processing | M3-W1 | ● | `03-elisp-foundation/week-01-lists-eval/chassell-reading.md` |
| 2 | Practicing Evaluation | M3-W1 | ● | 同上 |
| 3 | How To Write Function Definitions | M3-W2 | ● | `week-02-defun-let-if/chassell-reading.md` |
| 4 | A Few Buffer-Related Functions | M3-W2 | ● | 同上 |
| 5 | A Few More Complex Functions | M3-W3 | ● | `week-03-recursion/chassell-reading.md` |
| 6 | Narrowing and Widening | M3-W3 | ● | 同上 |
| 7 | car, cdr, cons: Fundamental Functions | M3-W1 | ● | `week-01-lists-eval/chassell-reading.md` |
| 8 | Cutting and Storing Text | M3-W4 | ● | `week-04-lambda-mapcar/chassell-reading.md` |
| 9 | How Lists are Implemented | M3-W1 | ● | 同 W1 |
| 10 | Yanking Text Back | M3-W4 | ● | 同 W4 |
| 11 | Loops and Recursion | M3-W3 | ● | `week-03-recursion/chassell-reading.md` |
| 12 | Regular Expression Searches | M3-W4 | ● | `week-04-lambda-mapcar/chassell-reading.md` |
| 13 | Counting via Repetition and Regexps | M3-W4 | ● | 同 W4 |
| 14 | Counting Words in a defun | M3-W5 | ● | `week-05-capstone/chassell-reading.md` |
| 15 | Readying a Graph | M3-W5 | ● | 同 W5 |
| 16 | Your .emacs File | M3-W5 + M4 | ● | 同 W5 + `04-config-system/` |
| 17 | Debugging | M3-W5 + M7 | ● | 同 W5 + `07-package-dev/` |

**额外章节** (Chassell 的 Preface/Why/Lisp History): 在 `00-mindset/intro-reading.md`

---

## 2. emacs-lispref (49 章)

> Module 6 的主线。前面模块会在需要时引用其中部分章节。

### 数据与求值 (M6-W1 到 W2)

| Ref 章节 | 文件 | 归属 | 深度 | 教程文件 |
|---|---|---|---|---|
| 1. Introduction | intro.texi | M0 | △ | `00-mindset/` 速览 |
| 2. Lisp Data Types | objects.texi | M6-W1 | ● | `06-elisp-deep/week-01-data-types/ref-reading.md` |
| 3. Numbers | numbers.texi | M6-W1 | ● | 同上 |
| 4. Strings and Characters | strings.texi | M6-W1 | ● | 同上 |
| 5. Lists | lists.texi | M6-W1 | ● | 同上 |
| 6. Sequences, Arrays, Vectors | sequences.texi | M6-W1 | ● | 同上 |
| 7. Records | records.texi | M6-W1 | ● | 同上 |
| 8. Hash Tables | hash.texi | M6-W1 | ● | 同上 |
| 9. Symbols | symbols.texi | M6-W2 | ● | `week-02-eval-variables/ref-reading.md` |
| 10. Evaluation | eval.texi | M6-W2 | ● | 同上 |
| 11. Control Structures | control.texi | M6-W2 | ● | 同上 |
| 12. Variables | variables.texi | M6-W2 | ● | 同上 |
| 13. Functions | functions.texi | M6-W2 | ● | 同上 |
| 19. Reading and Printing Lisp Objects | streams.texi | M6-W1 | ● | `week-01-data-types/` (streams 部分) |

### 宏、加载、编译、调试 (M6-W3 到 W4)

| Ref 章节 | 归属 | 深度 | 教程文件 |
|---|---|---|---|
| 14. Macros | M6-W3 | ● | `week-03-macros-loading/ref-reading.md` |
| 15. Customization Settings | M6-W3 + M4 | ● | 同 W3 + `04-config-system/` |
| 16. Loading | M6-W3 + M4 | ● | 同 W3 + `04-config-system/` |
| 17. Byte Compilation | M6-W3 + M7 | ● | 同 W3 + `07-package-dev/` |
| 18. Debugging Lisp Programs | M6-W4 + M7 | ● | `week-04-debugging/ref-reading.md` |
| 45. Tips | M7 | ● | `07-package-dev/ref-deep-dive.md` |

### 用户交互: Minibuffer/Commands/Keymaps (M6-W5)

| Ref 章节 | 归属 | 深度 | 教程文件 |
|---|---|---|---|
| 20. Minibuffers | M6-W5 | ● | `week-05-minibuf-commands/ref-reading.md` |
| 21. Command Loop | M6-W5 | ● | 同上 |
| 22. Keymaps | M6-W5 + M4 | ● | 同上 + `04-config-system/keymaps.md` |

### Modes/Files/Buffers/Windows (M6-W6)

| Ref 章节 | 归属 | 深度 | 教程文件 |
|---|---|---|---|
| 23. Major and Minor Modes | M6-W6 + M4 | ● | `week-06-modes-files-buffers/ref-reading.md` + `04-config-system/minor-mode-tutorial.md` |
| 24. Documentation | M6-W6 | ● | 同 W6 |
| 25. Files | M6-W6 | ● | 同 W6 |
| 26. Backups and Auto-Saving | M6-W6 | ● | 同 W6 |
| 27. Buffers | M6-W6 + M2 | ● | 同 W6 + `02-dired/concept-anchor.md` |
| 28. Windows | M6-W6 + M2 | ● | 同上 |
| 29. Frames | M6-W6 + M2 | ● | 同上 |
| 30. Positions | M6-W6 | ● | 同 W6 |
| 31. Markers | M6-W6 | ● | 同 W6 |

### Text/Display/Search/Syntax (M6-W7)

| Ref 章节 | 归属 | 深度 | 教程文件 |
|---|---|---|---|
| 32. Text | M6-W7 | ● | `week-07-text-display/ref-reading.md` |
| 33. Non-ASCII Characters | M6-W7 + M5 | ● | 同 W7 + `05-power-user/` |
| 34. Searching and Matching | M6-W7 | ● | 同 W7 |
| 35. Syntax Tables | M6-W7 | ● | 同 W7 |
| 36. Parsing Program Source | M6-W7 | ● | 同 W7 |
| 37. Parsing Expression Grammars (PEG) | M6-W7 | ● | 同 W7 (含 PEG 深入节) |
| 41. Emacs Display | M6-W7 | ● | 同 W7 |

### Processes/OS/Package/Internals (M6-W8 + M7 + M8)

| Ref 章节 | 归属 | 深度 | 教程文件 |
|---|---|---|---|
| 38. Abbrevs | M6-W8 + M5 | ● | `week-08-processes-package/ref-reading.md` (深入) |
| 39. Threads | M6-W8 | ● | 同 W8 |
| 40. Processes | M6-W8 + M5 | ● | 同 W8 + `05-power-user/` |
| 42. Operating System Interface | M6-W8 | ● | 同 W8 |
| 43. Preparing Lisp code for distribution | M7 | ● | `07-package-dev/ref-deep-dive.md` |
| 44. Antinews | M8 | △ | `08-open-source/ref-deep-dive.md` (趣味性) |
| 46. Emacs Internals | M8 | ● | 同上 |
| 47. Standard Errors | 附录 | ○ | 教程附录 |
| 48. Standard Keymaps | 附录 | ○ | 教程附录 |
| 49. Standard Hooks | 附录 | ○ | 教程附录 |

---

## 3. emacs-manual (34 章 + xtra)

> 分布在 M0-M8。每个章节都对应到某个模块的 `concept-anchor.md`。

### M0: 心法与装备

| Editor 章节 | 文件 | 深度 | 教程文件 |
|---|---|---|---|
| The Organization of the Screen | screen.texi | ● | `00-mindset/concept-anchor.md` |
| Entering and Exiting Emacs | entering.texi | ● | 同上 |
| Emacs History (gnu.texi) | ● | 同上 |
| Glossary (glossary.texi) | ● | 同上 (基础术语,全程查阅) |
| X Resources | xresources.texi | △ | 同上 (设置部分) |
| macOS / Haiku / Android / MS-DOS | macos/haiku/android/msdos.texi | △ | 同上 (平台附录) |

### M1: 编辑生存

| Editor 章节 | 文件 | 深度 | 教程文件 |
|---|---|---|---|
| Characters, Keys and Commands | commands.texi | ● | `01-survival/concept-anchor.md` |
| Basic Editing Commands | basic.texi | ● | 同上 + `drills/01-movement.md` |
| The Minibuffer | mini.texi | ● | 同上 |
| Running Commands by Name | m-x.texi | ● | 同上 |
| Help | help.texi | ● ★最重要★ | `drills/07-help-system.md` |
| The Mark and the Region | mark.texi | ● | `drills/04-mark-rectangle.md` |
| Killing and Moving Text | killing.texi | ● | `drills/02-kill-ring.md` |
| Registers | regs.texi | ● | `drills/04-mark-rectangle.md` |
| Controlling the Display | display.texi | ● | `drills/06-windows-frames.md` |
| Searching and Replacement | search.texi | ● | `drills/03-search.md` |
| Commands for Fixing Typos | fixit.texi | ● | `drills/01-movement.md` (含 undo) |
| Keyboard Macros | kmacro.texi | ● | `drills/05-kmacro.md` |
| Editing Programs (核心) | programs.texi | ● ★M1 主讲★ | `drills/08-code-navigation.md` (sexp 移动/操作、括号匹配、comment 命令、imenu、which-function-mode、xref、hideshow) |

### M2: Dired + Buffer/Window

| Editor 章节 | 文件 | 深度 | 教程文件 |
|---|---|---|---|
| File Handling | files.texi | ● | `02-dired/concept-anchor.md` |
| Using Multiple Buffers | buffers.texi | ● | 同上 |
| Multiple Windows | windows.texi | ● | 同上 |
| Frames and Graphical Displays | frames.texi | ● | 同上 |
| Dired | dired.texi | ● | `02-dired/drills.md` |
| Dired Subdirectory Switches (xtra) | dired-xtra.texi | △ | 同上 (可选) |

### M3: Elisp 入门 (作为 Chassell 的概念锚点)

| Editor 章节 | 锚点用途 | 教程文件 |
|---|---|---|
| Indentation | M3-W2 defun 的对照概念 | `03-elisp-foundation/week-02-defun-let-if/concept-anchor.md` |
| Commands for Human Languages | M3-W4 lambda 的对照 | `week-04-lambda-mapcar/concept-anchor.md` |

### M4: 配置体系

| Editor 章节 | 文件 | 深度 | 教程文件 |
|---|---|---|---|
| Customization | custom.texi | ● | `04-config-system/concept-anchor.md` |
| Major and Minor Modes (用户视角) | modes.texi | ● | 同上 + `minor-mode-tutorial.md` |
| Emacs Lisp Packages | package.texi | ● | 同上 |
| Command Line Arguments | cmdargs.texi | ● | 同上 |

### M5: Power User 工作流

| Editor 章节 | 文件 | 深度 | 教程文件 |
|---|---|---|---|
| International Character Set Support | mule.texi | ● | `05-power-user/mule-international.md` |
| Editing Programs (LSP/eglot 部分) | programs.texi | ● | `05-power-user/lsp-eglot.md` (M1 Drill 08 已主讲 sexp/imenu/which-function/xref/hideshow 基础; M5 深入 LSP backend、项目级跳转、xref-backend 配置) |
| Compiling and Testing Programs (compile/grep + GUD) | building.texi | ● | compile/grep 在 `05-power-user/lsp-eglot.md` + concept-anchor.md;GUD 部分在 `05-power-user/gud-debugging.md` |
| Maintaining Large Programs (VC) | maintaining.texi | ● | `05-power-user/magit.md` |
| Abbrevs | abbrevs.texi | ● | `05-power-user/concept-anchor.md` |
| The Calendar and the Diary | calendar.texi | ● | `05-power-user/calendar-diary.md` |
| Sending Mail | sending.texi | ○ | `05-power-user/` (可选) |
| Reading Mail with Rmail | rmail.texi | △ | 跳过 (用 mu4e/notmuch) |
| Miscellaneous | misc.texi | ● | `05-power-user/misc-amusements.md` + `05-power-user/` |
| Advanced VC (xtra) | vc-xtra.texi | △ | `05-power-user/magit.md` |
| Merging Files with Emerge (xtra) | emerge-xtra.texi | △ | 同上 |
| Editing Pictures (xtra) | picture-xtra.texi | △ | 跳过 |
| Fortran Mode (xtra) | fortran-xtra.texi | △ | 跳过 |
| More advanced Calendar (xtra) | cal-xtra.texi | △ | `05-power-user/calendar-diary.md` |

### M7: 写包

| Editor 章节 | 文件 | 深度 | 教程文件 |
|---|---|---|---|
| Dealing with Trouble | trouble.texi | ● | `07-package-dev/concept-anchor.md` |

### M8: 开源贡献

| Editor 章节 | 文件 | 深度 | 教程文件 |
|---|---|---|---|
| Acknowledgments | ack.texi | ○ | `08-open-source/concept-anchor.md` |
| Glossary (复习) | glossary.texi | ● | 同上 |
| Antinews (editor 版) | anti.texi | △ | 同上 (趣味) |

---

## 覆盖完整性检查

### Chassell (17/17 章)
- ✅ 全部 17 章在 Module 3 精读

### emacs-lispref (49 章)
- ✅ 43 章 ● 精读
- ✅ 3 章 ○ 实践 (Standard Errors/Keymaps/Hooks)
- ✅ 3 章 △ 速览 (Introduction/Antinews)
- 无遗漏

### emacs-manual (34 章 + 7 xtra)
- ✅ 29 章 ● 精读 (新增 misc.texi 全部精读)
- ✅ 3 章 ○ 实践 (sending/ack)
- ✅ 8 章 △ 速览 (rmail/xresources/4 个平台/3 个 xtra 跳过)
- ✅ Glossary 全程查阅
- 无遗漏

---

## 这个索引怎么用

1. **学习时**: 打开某个模块的教程,看它覆盖了哪些手册章节 (concept-anchor 标注)
2. **深入时**: 想看原典,从索引找到对应 texi 文件 (但教程已自包含,通常不需要)
3. **查漏时**: 检查某个手册章节是否真的学了,从矩阵看归属模块
4. **复习时**: 想复习某个主题,从矩阵找到所有相关章节
