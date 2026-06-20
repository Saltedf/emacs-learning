# Concept Anchor: Editor Manual (Module 5)

> 替代 `emacs-manual-30.2/` 的以下章节:
> - `maintaining.texi` (VC 内置 Git)
> - `building.texi` (compile/grep)
> - `programs.texi` (xref/imenu)
> - `text.texi` (大纲,fill)
> - `mule.texi` (Unicode)
> - `calendar.texi`

这个文件是"锚点"——把 Emacs 官方手册的章节映射到 Module 5 的内容。Emacs 30.2 手册有几千页,但绝大部分你不需要——只需要知道哪些章节对应你的需求,然后在那里查细节。

学完这个文件,你能:
- 知道 Emacs 内置提供了哪些"power user"功能 (版本控制、编译、跳转、大纲、Unicode、日历)
- 理解为什么社区包 (Magit、Consult、Corfu) 在这些功能之上流行——内置是基础,社区包是增强
- 在出问题时知道去哪查官方文档

---

## 1. maintaining.texi — Maintaining Large Programs

### 1.1 VC (Version Control)

Emacs 早在 1985 年就内置了版本控制接口 (VC, Version Control)——这是 Emacs 的"长寿"设计的体现。Git 是 2005 年才出现的,但 Emacs 已经有 20 年的版本控制集成经验。VC 抽象了一个**通用接口**——不管你用 RCS、CVS、SVN、HG、Bazaar 还是 Git,VC 提供同一套命令。

为什么这重要? 因为版本控制系统的本质是相同的 (commit/log/diff/blame),只是实现不同。VC 把这些**通用操作**抽象出来,让你不需要记每个 VCS 的具体语法。

VC 支持的 VCS: Git、SVN、HG、Bazaar、CVS、RCS...几乎所有主流版本控制系统。下面是核心命令——这些命令在所有 VCS 上都工作:

```
C-x v v    vc-next-action (智能: commit/push/...)
C-x v d    vc-dir (类似 magit-status,但通用)
C-x v +    vc-pull
C-x v P    vc-push
C-x v =    vc-diff
C-x v l    vc-print-log
C-x v i    vc-ignore
C-x v g    vc-annotate (blame)
```

`C-x v v` (vc-next-action) 是 VC 的"魔法"——它会根据当前状态**自动决定**做什么。如果文件改了未提交,它提交;如果已提交未推送,它推送。你不需要记住"我现在该 commit 还是 push",VC 替你决定。

这种"上下文感知"的设计哲学,后来被 Magit 借鉴并发扬光大——Magit 的 popup 系统也是这种"看状态,自动选操作"的思路。

### 1.2 Lisp 视角

VC 在 Lisp 层也是函数——这意味着你可以在脚本里调用,实现自动化:

```elisp
(vc-diff)                    ; 显示 diff
(vc-print-log)               ; log
(vc-annotate)                ; blame
```

这种"所有交互都是函数"是 Emacs 的根本特性——你按 `C-x v =` 实际上就是调用 `(vc-diff)`。这意味着你可以**写一个钩子**,比如"每次保存后自动跑 vc-diff 显示改动"——只需要一行 Elisp。这种可编程性是 GUI 客户端无法提供的。

### 1.3 vs Magit

那么,既然有内置 VC,为什么社区还创造了 Magit? 因为两者目标不同:

- **VC**: 通用,所有 VCS 一个 UI。功能基础,关注"跨 VCS 抽象"。
- **Magit**: 只 Git,但深度整合,功能极强。关注"Git 用户的所有需求"。

类比: VC 像 Python 的 `os.path` (跨平台抽象),Magit 像 `pathlib` (针对现代 Python 的深度工具)。两者各有价值。

对于 2026 年的开发者,99% 用 Git,所以 Magit 是主流。但 VC 仍然有用——它是**fallback**,当你在一个没装 Magit 的机器上 (比如服务器),`C-x v v` 仍然能用。这种"内置保底"是 Emacs 设计的安全网。

Module 5 主要学 Magit (社区主流)。
VC 是 fallback (没装 magit 时用)。

---

## 2. building.texi — Compiling

### 2.1 Compile 命令

Emacs 的 `compile` 命令是一个被低估的功能。它的核心思想: **编译输出是结构化文本,Emacs 能解析**。当你跑 `make`,编译器输出的 `file.c:42: error: ...` 这种行,Emacs 能识别——按 `M-x next-error` 直接跳到出错位置。

这种"把外部工具的输出变成可导航的 buffer"是 Emacs 的核心模式之一。它不是简单的"显示输出",而是**结构化解析**——错误、警告、跳转都自动化。

```
M-x compile RET make RET     ; 跑 make
M-x recompile                ; 重跑上次
```

`compile` 和 `recompile` 的区别: `compile` 每次问命令 (默认上次),`recompile` 直接用上次命令不问。日常用法是: 第一次 `compile`,之后一直 `recompile`。

结果在 `*compilation*` buffer:
- 错误高亮
- `M-x next-error` 跳下一个错误
- 鼠标点击跳

`next-error` 是这个工作流的核心——你按 `M-g M-n` (或 `M-x next-error`),光标就跳到下一个错误对应的源代码位置。在大型 C/C++ 项目里,这比在 IDE 里看错误列表更快,因为编译输出的所有上下文都在 buffer 里。

### 2.2 Grep

同样的模式用于 grep——结果在一个可导航的 buffer:

```
M-x grep
M-x rgrep                    ; 递归
M-x find-grep
```

结果在 `*grep*` buffer。按 RET 跳到匹配行——这就是 Consult-grep 的"前身"。Consult 在这个基础上加了 preview、过滤、跨文件聚合等增强,但底层还是这个 grep buffer 模式。

### 2.3 GUD (Grand Unified Debugger) — 主讲 (见 `gud-debugging.md`)

GUD 是 Emacs 的调试器接口——它的名字说明了设计哲学: "Grand Unified"——把所有调试器统一到一个接口:

```
M-x gdb                      ; GDB
M-x perldb                   ; Perl
M-x pdb                      ; Python pdb
```

可设断点、单步、查看变量。GUD 在 buffer 里显示源代码,断点和当前位置高亮——这在 1985 年是革命性的 (那时主流是命令行 gdb)。

GUD 的核心交互以 `C-c C-` 前缀: `C-c C-s` step into、`C-c C-n` next/over、`C-c C-c` continue、`C-c C-b` 设断点。多 buffer 协作 (GUD + Source + Stack + Breakpoints + Locals) 是它的核心体验,开启 `gdb-many-windows` 一次性打开。

2026 年的对比:
- **C/C++**: GUD + GDB 仍然是黄金组合 (LLDB 的 DAP 不成熟)
- **Python / Rust / Go**: `dap-mode` (基于 DAP 协议) UI 更现代
- **eglot 不包含调试**——调试和 LSP 是两个独立系统

详细内容 (含 conditional breakpoint、watch、多线程、远程调试、反向调试) 见 `05-power-user/gud-debugging.md`。

### 2.4 Lisp 视角

```elisp
(compile "make")
(grep "find . -name '*.el' -exec grep -nH foo {} +")
(next-error)
```

这三个函数可以串起来——比如你可以写一个函数: 跑 `make`,如果有错跳第一个错,没错跑测试。这就是 Emacs 的可编程性在 build 工作流上的体现。

---

## 3. programs.texi (复习)

### 3.1 xref (跳定义)

xref 是 Emacs 25+ 引入的**通用跳转 API**——它抽象了"跳定义"这个操作,不管底层是 etags (本地 TAGS 文件)、LSP server、还是别的什么。这是又一个"通用接口"设计的例子。

```
M-.    xref-find-definitions
M-?    xref-find-references
M-,    xref-pop-marker-stack
M-*    xref-pop-marker-stack (alias)
M-?    xref-find-references
```

`M-.` 和 `M-,` 是你每天会按几十次的键——跳到定义,再跳回来。xref 维护一个"marker stack"——每次跳转压栈,`M-,` 弹栈回到上一个位置。这意味着你可以**连续跳多层** (函数 A 调用 B 调用 C,跳 A→B→C,然后 `M-,` 三次回到 A),不会丢失位置。

### 3.2 imenu

imenu 是"buffer 内导航"——列出当前 buffer 的所有函数/类/变量定义,选一个跳过去。在大文件 (几千行) 里非常有用。

```
M-s i  imenu
```

### 3.3 Lisp 视角

```elisp
(xref-find-definitions SYMBOL)
(etags)                      ; TAGS 文件
```

`etags` 是古老但仍然有用的工具——它扫描代码生成 TAGS 文件,Emacs 用它做跳转。在没有 LSP 的年代 (2010 以前),etags 是唯一方案。现在 LSP 取代了 etags,但 xref API 保留——这意味着你的键位 (`M-.`) 不变,底层从 etags 换到 LSP 是无感的。

---

## 4. text.texi (复习)

### 4.1 Outline

很多 mode (org, markdown) 基于 outline——outline 是 Emacs 的"大纲 minor mode",提供折叠/展开/导航功能。Org-mode 实际上是 outline-mode 的超集——它在 outline 之上加了 TODO、properties、agenda 等。

outline 提供的能力:
- 折叠/展开章节
- 移动章节
- 大纲视图

```elisp
(outline-mode)               ; minor mode
(hide-subtree)
(show-subtree)
```

理解 outline 是理解 Org 的基础——Org 的所有折叠键 (`TAB`, `S-TAB`) 都来自 outline。

### 4.2 Fill

fill 是 Emacs 处理"硬换行"的方式——在 1980 年代,文本文件必须有硬换行 (每行 < 80 字符),因为终端宽度有限。今天我们用"软换行" (视觉换行,文件里没换行符),但 fill 在某些场景仍然有用 (邮件、文档、README):

```elisp
(fill-paragraph)             ; M-q
(fill-region START END)
(auto-fill-mode)             ; 自动 fill
```

`M-q` (fill-paragraph) 是被低估的键——它把当前段落重新换行,让每行长度合适。写文档时非常有用。

---

## 5. mule.texi — International (主讲,见 `mule-international.md`)

MULE (MULtilingual Enhancement) 是 Emacs 处理多语言的能力——这在 1990 年代是革命性的 (那时大多数编辑器只支持 ASCII)。Emacs 内部用 Unicode 表示所有字符,这意味着中文、日文、阿拉伯文、emoji 都是一等公民。

关键洞察: **Emacs 内部不存"UTF-8 字节",它存"Unicode codepoint"**——读文件时解码,保存时编码。这意味着内部永不丢信息,即使文件是 GB2312 你也能无损转 UTF-8。

### 5.1 编码系统 (主讲)

2026 年 UTF-8 一统天下,但**老文件**仍然存在——中国大陆 1990-2010 年的文件大多是 GB2312/GBK。Emacs 用"编码系统"符号 (utf-8-unix, gb2312, big5, shift_jis 等) 抽象这些。

核心命令:
- `C-h C RET` 看当前编码
- `C-x RET f` 改编码保存
- `M-x revert-buffer-with-coding-system` 用指定编码**重新解码** (修乱码的核心命令)
- `C-x RET c` 给下一个命令指定编码 (一次性)
- `prefer-coding-system` / `set-language-environment` 设默认

诊断字符: `C-u C-x =` 显示光标字符的完整信息 (codepoint, charset, font, face)——这是诊断"字符显示不对"的终极工具。

### 5.2 输入法 (Quail)

Emacs 内置一组输入法,基于 **Quail** 框架——和系统 IME (fcitx / ibus / macOS pinyin) 是两套独立系统。`C-\` 切换。常用输入法: `chinese-py` (全拼)、`japanese`、`korean-hangul` 等。

大多数人用系统 IME (更智能),Emacs 内置输入法的价值: (1) SSH 远程无 IME 场景; (2) macOS IME 集成 bug 时回退。

### 5.3 字体配置 (fontset,主讲)

Emacs 用 **fontset** 概念——按 charset 分配字体:

```elisp
(set-fontset-font t 'han      "Source Han Sans CN")
(set-fontset-font t 'kana     "Source Han Sans JP")
(set-fontset-font t 'hangul   "Source Han Sans KR")
(set-fontset-font t '(#x1F300 . #x1F9FF) "Apple Color Emoji")
(setq face-font-rescale-alist '((".*Source Han.*" . 0.95)))
```

这种"按 charset 精确控制字体"是 VSCode 等没有的——它是 Emacs 解决 CJK 字体对齐问题的核心机制。

### 5.4 终端 (TTY) 限制

GUI Emacs 的 Mule 体验完整。TTY Emacs 有几个限制: fontset 无效、tooltip 无效、IME 集成是终端的事。如果可能,**多语言编辑用 GUI Emacs**。

### 5.5 详细内容

编码细节、所有输入法、CJK 字体对齐的调优、emoji、双向文本 (RTL)、Lisp API (charset 查询、编码转换、CJK 字符提取)——见 `05-power_user/mule-international.md` (路径 `05-power-user/mule-international.md`)。

### 5.6 Lisp 视角

```elisp
?中                       ; => 20013 (U+4E2D)
(length "中文")           ; => 2 (字符数)
(string-bytes "中文")     ; => 6 (UTF-8 字节数)
(char-charset ?中)        ; => chinese-gb2312 (或 CJK)
(decode-coding-string bytes 'gb2312)
```

字符是整数 (codepoint),字符串是字符序列——`length` 返回字符数,`string-bytes` 返回字节数。这是 Mule 在 Elisp 层的精细处。

---

## 6. calendar.texi — Calendar (主讲,见 `calendar-diary.md`)

Emacs 的日历看起来朴素,但它是 Org-agenda 的基础——Org 的所有时间相关功能 (schedule、deadline、repeat) 都建立在 calendar 之上。但 Calendar + Diary 本身就是完整系统——理解它们让你在简单场景 (生日、固定事件) 下用比 Org 更轻的方案。

### 6.1 Calendar 模式 (主讲)

```
M-x calendar                 ; 打开日历
.                            ; 今天 (calendar-goto-today)
M-n / M-p                    ; 下/上月
g d                          ; goto date
g w                          ; goto week
g m                          ; goto month
C-f / C-b                    ; 前一天 / 后一天
C-n / C-p                    ; 下一周 / 上一周
```

Calendar 不只是"显示"——它能**算日期**:
- `+` / `-`: 当前光标日期加 / 减 N 天
- `M-=`: 计算两个日期间隔天数 (用 mark + point)
- `p h` / `p j` / `p m`: 打印当前日期的希伯来 / Julian / Mayan 历等价

跨日历支持: Gregorian, Julian, Islamic, Hebrew, Mayan, French, Persian, Coptic, Ethiopic, ISO——`p X` 系列命令打印各种历法下的等价日期。

实用命令: `M-x holidays` (节假日)、`M-x lunar-phases` (月相)、`M-x sunrise-sunset` (日出日落,需配经纬度)、`M-x insert-calendar` (在当前 buffer 插入月历)。

### 6.2 Diary (主讲)

Diary 是 Emacs 内置的"日记"功能——所有事件存在 `~/diary` (纯文本) 里。这是 Org-agenda 的前身,但语法独立:

```
June 20, 2026  Pay rent
&%%(diary-anniversary 6 20 1990) Alice's birthday (%d years)
&%%(diary-cyclic 7 6 1 2026) Weekly team meeting
&%%(diary-float t 3 3) Monthly team meeting (third Wednesday)
```

Diary 支持多种语法: 具体日期、每天 (`&`)、Sexp (`%%(...)`)。Sexp 是 Elisp 调用——`diary-anniversary` (周年)、`diary-cyclic` (周期重复)、`diary-block` (日期范围)、`diary-float` (每月第几个周几)。

Calendar 模式里 `i X` 系列命令基于当前光标日期插入 entry:
```
i d / i w / i m / i y / i a / i c / i b / i f
daily / weekly / monthly / yearly / anniversary / cyclic / block / float
```

`M-x diary` 把所有 entry 收集成 buffer (类似 agenda);`org-agenda-include-diary t` 让 Org-agenda 显示 diary 条目。

### 6.3 Calendar / Diary / Org 的层次

```
calendar        日期算术、跨日历、节假日、月相
   ↓
diary           纯文本事件记录 (anniversary, cyclic, float)
   ↓
org-agenda      TODO + 事件 + 项目 (高层抽象)
```

Org-agenda 是 Org 的"大杀器",但 Diary 在简单场景 (生日、月度账单、固定会议) 更轻——不需要 TODO 状态,不需要 properties,一行 entry 搞定。

### 6.4 Lisp 视角

```elisp
(calendar)
(diary)
(org-agenda)                 ; Org 的日历视图
```

这三个函数展示了"内置 → 增强"的演化: calendar 是基础,diary 是事件管理,org-agenda 是 TODO + 事件 + 项目的统一视图。

详细内容 (含 anniversary 提醒、`%d` 年龄占位、HTML 导出、Org 集成、所有陷阱) 见 `05-power_user/calendar-diary.md` (路径 `05-power-user/calendar-diary.md`)。

---

## 7. 自测

1. `C-x v v` 干啥?
2. `M-x compile` 和 `M-x recompile` 区别?
3. `M-.` 和 `M-?` 分别干啥?
4. 怎么看 char 编码?
5. Emacs VC 和 Magit 区别?
6. `M-x gdb` 时 `-i=mi` 参数是干嘛的?
7. `C-c C-s` 和 `C-c C-n` 在 GUD 里区别?
8. Diary 文件默认叫什么?怎么插入 anniversary?
9. `%%(diary-cyclic 7 6 1 2026)` 什么意思?
10. Emacs 内部用什么表示字符?UTF-8 字节吗?
11. 怎么修乱码 (打开文件编码错了)?

**答案**:
> 1. vc-next-action (智能 commit/push/...)
> 2. compile 重新输命令;recompile 用上次
> 3. M-. find-definitions;M-? find-references
> 4. C-u C-x =
> 5. VC 通用所有 VCS,基础;Magit 只 Git,深度强
> 6. `-i=mi` 让 GDB 用 Machine Interface 输出 (结构化,GUD 能解析)
> 7. C-c C-s = step into (进入函数);C-c C-n = next/step over (不进入)
> 8. `~/diary`;`i a` (insert anniversary)
> 9. 从 2026-06-01 起每 7 天重复一次
> 10. Unicode codepoint (整数),不是字节
> 11. `M-x revert-buffer-with-coding-system RET 编码 RET`

---

## 8. 下一步

进入 `org-mode.md` 学 Org。
