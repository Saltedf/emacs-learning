# Module 1: 编辑生存 + 自文档系统

> **目标**: 完全脱离鼠标,熟练使用 help system (Emacs 自我学习的能力来源)
> **时长**: 1 周 (~20 小时)
> **难度**: ★★☆☆☆ (大量肌肉记忆训练)
> **依赖**: Module 0 (装好 Emacs + 写了 init.el)
> **核心产出**: 30 分钟内无鼠标重构 2000 行代码

---

## 0. 这个模块在学什么

你已经知道 Emacs 是 Lisp 机器,知道 Frame/Window/Buffer 的关系,装好了 Emacs 写了 init.el。

现在的问题是: **你打开一个文件,想编辑它,可是鼠标还离不开**。
- "复制粘贴在哪个菜单?"
- "怎么选中一段?"
- "怎么搜索?"
- "怎么撤销?"

这些不是"小麻烦",是**根本性的工作阻力**。每用一次鼠标,你的手离开 home row、移到鼠标、点击、移回键盘,大约花费 2-3 秒。一个工作日 200 次鼠标操作 = 10 分钟纯损耗。一年 250 个工作日 = 40 小时,相当于一整周的工作时间被鼠标吃掉。

更严重的是认知成本: 每次鼠标操作打断你的**心流**。你在写代码时想"删这一行",鼠标移过去、选中、按 Del,这个过程中你的大脑从"思考代码"切换到"操作 GUI"。优秀的程序员追求**单一流**——一个心智状态不被打断。

这个模块解决这些**编辑生存**问题。

完成后你会:
- 在任何 buffer 里像弹钢琴一样移动、选、删、改
- 知道怎么用 `C-h` 系列查**任何**不懂的命令
- 理解 prefix argument、kill ring、mark ring 这些 Emacs 特有的概念
- 用键盘宏 (kmacro) 自动化重复操作

但这些只是表面。**真正的产出**是一种思维方式的转变: 你不再把编辑器看作"工具",而是看作**身体的延伸**。键位不是"快捷键",是肌肉记忆。命令不是"功能",是可以查询和组合的**一等公民**。

这种转变需要时间。Module 1 给你 20 小时,让你形成第一层肌肉记忆。后面 7 个模块继续深化。

---

## 1. 第一性原理: 为什么是 C-/M- 前缀

> 详细版见 `concept-anchor.md`,这里先建立直觉。

### 1.1 一个不那么显然的事实: 大多数现代编辑器"键位不够用"

打开 VS Code,看看它的快捷键: `Ctrl+S` 保存、`Ctrl+C` 复制、`Ctrl+V` 粘贴、`Ctrl+X` 剪切、`Ctrl+Z` 撤销、`Ctrl+Shift+P` 命令面板... 你会发现 VS Code 已经开始用**三键组合** (`Ctrl+K Ctrl+S` 之类的 chord),因为单键空间快满了。

这是因为: 一个 Ctrl 修饰键 + 26 个字母 + 10 个数字 + 几个符号,撑死 80 个槽位。一个像样的编辑器需要的命令远超 80——光 Emacs 的核心命令就有几百个,加上各种 mode 的特有命令,总数上千。

VS Code 的解决: 让用户用鼠标点菜单,或者用 `Ctrl+Shift+P` 打开命令面板慢慢搜。代价是**鼠标依赖**和**速度慢**。

Emacs 在 1976 年就想清楚了这个问题: **不要让一个 modifier 只对应一个动作,让它做"前缀"**。

### 1.2 C-x 是"前缀"而不是"动作"

直觉上,`C-x` 看起来像是一个 modifier+字母,应该绑定一个具体命令。但 Emacs 把它设计成 **prefix key**: 按下 `C-x` 后,Emacs **等待下一个键**,两个键组合才构成完整命令。

```
C-x C-f = find-file (打开)
C-x C-s = save-buffer (保存)
C-x b   = switch-buffer (切 buffer)
C-x k   = kill-buffer (关 buffer)
C-x 2   = split-window-vertically (分屏)
C-x 0   = delete-window (关分屏)
```

这不是"两个快捷键共用一个 modifier",而是 **Emacs 把整个 `C-x` 子树作为一个命名空间**。所有 buffer/window/file 相关的操作都住在 `C-x` 这棵树下,你可以"按主题找键位": "这是 buffer 操作 → 在 C-x 下找"。

### 1.3 键位空间的算术

让我们算算这套设计给 Emacs 多少键位。一个 ASCII 键盘上,不加 modifier 大约有 80 个可输入字符 (字母大小写、数字、标点、空格、回车)。加上一个 Control:

```
无修饰:        80
C- 前缀:       80
M- 前缀:       80
C-M- 组合:     80
S-C-, S-M-:    各 80
单键命令:      26 (a-z) + 26 (A-Z) + 10 (数字) + ~15 (符号) ≈ 80 个
C-x 前缀:      C-x + 单字符 ≈ 80 个,加 C-x C-x ≈ 80 个
C-c 前缀:      同上,留给用户/mode
C-h 前缀:      同上,所有 help
M-x:           命令名补全,无限制
```

把 `C-x`、`C-c`、`C-h`、`C-x C-x`、`C-c C-c` 等 prefix 组合算进来,Emacs 的键位空间 > 1000 个**不用鼠标**的命令。

而 `M-x` (execute-extended-command) 提供了**逃生通道**: 任何命令都可以用 `M-x 命令名 RET` 调用,无需绑定键位。这意味着 Emacs 的命令数**理论上没有上限**——你可以装 100 个包,每个包提供 50 个命令,共 5000 个命令,你用 `M-x` 都能调到。绑键只是为了**常用的命令加速**。

### 1.4 这种设计的代价: 学习曲线

代价是真实的。新手看到 `C-x C-f` 会问"为什么不是 Ctrl+O (像大多数编辑器)? "。答案是: Ctrl+O 是单键命令,占了一个槽位;`C-x C-f` 是 prefix,不占单键槽位。Emacs 把单键槽位留给**最高频的编辑操作** (字符移动、删除),把稍低频的 (文件、buffer、window) 放进 prefix。

**记忆口诀** (Chassell 的建议,来自 Emacs tutorial):
- `C-x` 是"Emacs 级"操作 (buffer/window/file)
- `C-c` 是"模式级"操作 (具体到某个 major mode)
- `C-h` 是"help"
- `C-c <字母>` 是用户自定义保留 (你的 init.el 用这个,不会被 Emacs 占走)

这套语义是**自洽的**。一旦理解,你看到一个新键位 `C-c C-p`,立刻知道"这是某个 mode 提供的命令"。看到 `C-x 4 b`,立刻知道"这是 buffer 相关,且涉及'另一个 window'" (`4` 在 `C-x` 下表示 other-window)。

### 1.5 历史小插曲: 为什么不是 Vim 的模式切换

Vim 走了另一条路解决键位不够的问题: **模式**。在 normal mode 下,所有字母都是命令;在 insert mode 下,所有字母都是输入。这样 80 个键变成 160 个槽位,而且省掉了 Ctrl (Vim 极少用 modifier)。

两条路各有优劣。Emacs 的 prefix 方案: 始终是输入模式,modifier 区分语义。Vim 的 mode 方案: 状态切换更高效但需要"思考当前在哪个模式"。

Emacs 后来有 `evil-mode` (Vim 仿真),证明了这两条路可以融合。Module 4 之后如果你手腕疼,可以考虑装。但现在,我们老老实实学 Emacs 原生方案。

---

## 2. 这个模块的 8 个 Drills

每个 drill 一个主题,30-50 道微练习。

| Drill | 主题 | 题数 | 时长 |
|---|---|---|---|
| 01 | Movement (移动) | 30 | 2h |
| 02 | Kill Ring (杀与召回) | 20 | 1.5h |
| 03 | Search (搜索与替换) | 25 | 2h |
| 04 | Mark & Rectangle (标记与矩形) | 20 | 1.5h |
| 05 | Kmacro (键盘宏) | 15 | 1.5h |
| 06 | Windows & Frames (窗口与框架) | 20 | 1.5h |
| 07 | **Help System (自文档) ★最重要** | 25 | 3h |
| 08 | Code Navigation (代码导航) | 30 | 2.5h |

**总计**: 185 题,~15.5 小时
**加上 capstone (重构 2000 行)**: ~22.5 小时

---

## 3. Drills 的"3 遍过" 学习法

每个 drill 按这个顺序做:

### 第 1 遍: 跟着做 (60%)

- 读题
- 看教程的"内联讲解"
- 跟着例子做一遍
- 不强求记牢,主要熟悉

### 第 2 遍: 独立做 (30%)

- 合上教程
- 把所有题目重新做一遍
- 卡住的地方标记
- 重读相关章节

### 第 3 遍: 速度训练 (10%)

- 用 `M-x kmacro` 录一段你的操作
- 看自己用了多久
- 把不熟练的键位单独练 5 分钟

### 不要做的

- ❌ 不要"看完所有 drill 再开始"——边读边练
- ❌ 不要"做一遍就过"——至少两遍
- ❌ 不要"用鼠标兜底"——卡住用 `C-h`,不用鼠标

---

## 4. 学习路线 (一周计划)

| 天 | 任务 |
|---|---|
| Day 1 | Drill 01 Movement + Drill 02 Kill Ring |
| Day 2 | Drill 03 Search + Drill 04 Mark & Rectangle |
| Day 3 | Drill 05 Kmacro + Drill 06 Windows/Frames |
| Day 4 | Drill 07 Help System (★最重要,给最多时间) |
| Day 5 | Drill 08 Code Navigation + Capstone 准备 |
| Day 6 | Capstone 执行: 30 分钟无鼠标重构 |
| Day 7 | 复盘 + 写 logs/module-01.md |

**最大投入建议** (用户选了"最大强度"):
- 每天一个 drill (4-5 小时)
- 一周内完成所有 drill + capstone
- 第二周回顾,补漏

---

## 5. 关键概念预告

这个模块你会建立这些概念。先扫一眼,不要求记住:

### 5.1 Point / Mark / Region

先建立根本直觉: Emacs 把 buffer 看作一个**字符串**。Point 和 Mark 都是**这个字符串里的整数下标** (从 1 开始)。

```
[Buffer 文本]
Hello, world!
   ↑ point      ↑ mark (用 C-SPC 设置)

[point 和 mark 之间的文本 = region]
```

这意味着 region 不是"选中状态"——它是**两个整数定义的区间**。`transient-mark-mode` (默认开) 只是让这个区间**视觉上高亮**。关掉它,region 仍然存在,只是看不见。

这个设计的妙处: region 可以**程序化操作**。`(buffer-substring (region-beginning) (region-end))` 取 region 内容,`(delete-region (region-beginning) (region-end))` 删 region。任何 Elisp 代码都能查询和操作 region,这是 Emacs 可扩展性的基础。

- `(point)` → 当前光标位置 (integer)
- `(mark)` → 标记位置
- `region-active-p` → region 是否激活
- 大多数编辑命令对 region 生效

### 5.2 Kill ring (杀环)

为什么不是"剪贴板"? 因为剪贴板只能存一份,而**人类经常后悔**。你杀了一段代码,又杀了一段,然后想召回第一段——剪贴板做不到,kill ring 可以。

第一性原理推导:
- 推论 1: 删除是危险的,用户可能后悔 → 删除的东西应该有地方去 → kill ring
- 推论 2: 多次删除应该都保留 → list,不是单值
- 推论 3: 应该能循环浏览 → ring 结构 (不是栈),`M-y` 转一圈回到原点
- 推论 4: 跨 session 不需要 (操作系统有 clipboard) → 进程结束就丢
- 推论 5: 但要和系统 clipboard 同步 → `interprogram-cut-function`

```
kill-ring:    ["最近杀的", "前一次", "再前一次", ...]
                       ↑
                kill-ring-yank-pointer (游标)

C-y → 取游标指向的
M-y → 游标后移一位,替换刚才 yank 的
```

这是 1976 年的设计,比 Windows clipboard history (2018 年 Win10 才有) 早了 42 年。

### 5.3 Prefix argument (前缀参数)

为什么按 `C-u 5 C-f` 会让 `forward-char` 移动 5 个字符? 因为 Emacs 的命令执行函数 `call-interactively` 在调用命令前,会把 prefix argument 作为一个**参数**传给它。

第一性原理: 如果每个"重复 N 次"的操作都给单独的命令,键位空间立刻爆炸 (`forward-1-char`、`forward-2-char`、`forward-3-char`...)。Emacs 的解决: 让一个参数机制统一所有命令。

```
C-u N <command>   数字前缀,重复 N 次
C-u <command>     无数字,默认 4,可连按 (C-u C-u = 16, C-u C-u C-u = 64)

例:
C-u 5 C-f         向前移 5 个字符
C-u 3 M-d         杀 3 个词
C-u C-k           杀一整行 (含换行)
M-5 C-n           向下移 5 行 (M-num 等于 C-u num)
```

为什么 `C-u` 默认是 4 而不是 1? 因为 `(C-u)^0 = 1, (C-u)^1 = 4, (C-u)^2 = 16, (C-u)^3 = 64`——按 C-u 的次数是 4 的幂。这是为了让"按几下 C-u"快速得到 4、16、64 这种常用倍数,而不是线性 1、2、3。**指数思维,不是线性**。

### 5.4 Isearch (递增搜索)

搜索是编辑器的灵魂。1976 年 Emacs 引入 incremental search (递增搜索),这是革命性的——你每按一个字符,光标立刻跳到第一个匹配。反馈循环 < 100ms。这在当时是 radical,因为大多数编辑器要你输完 query 按 ENTER 才搜索。直到 1990s 其他编辑器才学会。

第一性原理: 反馈循环越短,认知负担越低。普通搜索要你在脑子里构建完整 query 再按确认,然后等结果。递增搜索让你"边想边搜",每按一个键都是反馈: "对了,继续",或者"错了,退一格"。这种循环让搜索变成**对话**,不是命令。

```
C-s foo      向后搜索 foo (每输入一个字符,实时跳到匹配)
C-r foo      向前搜索
C-s C-s      重复上次搜索 (用 search-ring)
C-s RET foo  非递增 (用 ENTER 结束,有些场合需要)
C-s C-w      把光标处的 word 加入搜索词
C-s C-y      把光标处到行尾加入搜索词
```

### 5.5 Help system

```
C-h f FUNCTION   查函数
C-h v VARIABLE   查变量
C-h k KEY        查键位绑定的命令
C-h w COMMAND    反向: 命令绑到哪个键
C-h m            查当前 major mode + minor modes
C-h b            查所有键位绑定
C-h i            进 Info (官方手册)
C-h r            直接打开 Emacs manual
C-h a KEYWORD    apropos: 按关键字搜命令
C-h o SYMBOL     查 symbol (变量或函数)
C-h F COMMAND    查命令的"教程式"文档 (比 C-h f 更易读)
C-h S SYMBOL     在 Info 里查 symbol
C-h C-h          显示所有 C-h 子命令
```

**这是整个 Module 1 最重要的一组**。Drill 07 会专门训练。

为什么 help system 是 Module 1 最重要? 因为它教会你**自我学习**。普通软件用户遇到问题就 Google,得到的答案可能过时、可能错。Emacs 用户遇到问题,`C-h` 直接读源码 docstring,与版本完全匹配,准确度 100%。学会这一组,你后面 5 个月的学习速度会翻 5 倍。

### 5.6 Kmacro (键盘宏)

键盘宏是"低配版 Elisp"。当某个操作要做 50 次,你又不想写函数,就录一个宏跑 50 次。

第一性原理: 重复劳动有两种自动化方案——写 Elisp 函数 (通用,可复用,但写起来慢) 和录 kmacro (一次性,特定,但录起来快)。kmacro 不是 Elisp 替代品,而是**快速自动化**。

```
F3            开始录制 (或 C-x ( )
... 操作 ...
F4            结束录制并立即运行 (或 C-x ) )
C-x e         再运行一次 (kmacro-end-and-call-macro)
C-u N C-x e   运行 N 次
C-x C-k r     在 region 内每行运行一次
C-x C-k n     给宏命名
C-x C-k e     编辑宏 (改录下的内容)
```### 5.7 Windows

分屏是 Emacs 的"工作空间"。一个 frame 内可以有任意多个 window,每个 window 独立显示一个 buffer 的某部分。

第一性原理: 人类的工作记忆有限 (大约 4 个 chunk)。改 bug 时,你要同时看代码、测试、日志、文档——如果只能看一个,就要不断切换 buffer,认知负担巨大。Emacs 的解决: 让 4 个 window 同时显示,你用 `C-x o` 在它们之间切换,kill-ring 跨 window 共享。

```
C-x 2         上下分屏
C-x 3         左右分屏
C-x 1         只保留当前
C-x 0         关闭当前
C-x o         切到下一个 window
C-x ^         增高
C-x }         增宽
C-x {         减宽
C-x -         shrink-if-larger-than-buffer
C-x +         balance-windows
windmove:     M-{left,right,up,down}  (需要 (windmove-default-keybindings))
```

### 5.8 代码导航

代码不是普通文本——它有语法结构 (嵌套括号、函数定义、定义-引用关系)。Emacs 把这层结构抽象为 **S-expression** (sexp),提供一组 `C-M-` 前缀命令在 sexp 上移动和操作。

第一性原理: 普通文本是"字符流",代码是"嵌套树"。如果用 word/char 移动 (`M-f`/`C-f`) 跳代码,你要按几十次;用 sexp 移动 (`C-M-f`),一次跳一个语法单元。Stallman 1985 年的洞察是:**大多数代码编辑操作本质上是"在嵌套结构里导航"**,编辑器识别嵌套边界后,90% 操作能秒级完成。

```
C-M-f / C-M-b   跳下/上一个 sexp
C-M-u / C-M-d   上/下穿越嵌套层次
C-M-a / C-M-e   跳 defun 头/尾
C-M-k           杀一个 sexp (结构化删除)
C-M-SPC         选中一个 sexp
M-;             注释/取消注释 (DWIM)
M-x imenu       跳到任意定义
M-.             xref-find-definitions (跨文件)
M-?             xref-find-references
M-,             回上一个位置
hs-minor-mode   代码折叠
```

这是 Emacs 代码操作的**核心肌肉记忆**,Drill 08 专门训练。在 Lisp/Python/C 任何 mode 里都能用,而且**不依赖任何 LSP**——开箱即用。

---

## 6. 这个模块不教什么 (后续模块教)

避免范围蔓延:

- ❌ Org-mode 详细使用 (Module 5)
- ❌ Magit/Git (Module 5)
- ❌ 写 Elisp 包 (Module 3+)
- ❌ LSP/补全 (Module 5)
- ❌ Tree-sitter (Module 5)
- ❌ Dired 高级用法 (Module 2)
- ❌ 配置 use-package (Module 4)

这个模块只教**通用编辑生存技能**,适用于任何 buffer。

---

## 7. 准备开始

打开你的 Emacs,确认:
- [ ] 你的 init.el 加载了 (检查 `C-h v user-init-file`)
- [ ] `show-paren-mode` 开了 (高亮配对括号)
- [ ] `which-key-mode` 没装也没关系 (Module 4 之后装)
- [ ] 一个练习 buffer (新建: `C-x C-f ~/emacs-practice.txt RET`)

进入第一个 drill: `drills/01-movement.md`。

---

## 8. 毕业检查 (这个模块结束前能答出来)

### 概念题

1. 为什么 Emacs 用 `C-x` 这种两步前缀?
2. Kill ring 和 register 的本质区别?
3. Mark ring 的作用?
4. Isearch 的"递增"是什么意思?
5. Prefix argument 怎么用?
6. `C-h f`/`C-h v`/`C-h k`/`C-h a` 各自查什么?
7. Keyboard macro 的录制和运行流程?
8. C-x 2、C-x 3、C-x o、C-x 0、C-x 1 各做什么?
9. `C-M-f`/`C-M-u`/`C-M-a` 各跳什么? `M-;` 在不同情况下做什么?
10. imenu 和 xref-find-definitions 的区别? which-function-mode 显示什么?

### 实操题

1. 不看任何资料,在 5 秒内杀一行 → yank → M-y 替换
2. 用 isearch 跳到下一个 "function" → 替换为 "method"
3. 用矩形操作把一个代码块的每行前面加 "// "
4. 录制一个 kmacro: 在每行末尾加分号,运行 10 次
5. 用 `C-h` 找到 `replace-string` 的文档,理解它的"disastrous" 警告
6. 用 `C-M-f`/`C-M-u`/`C-M-a` 在一个 Lisp buffer 里导航,30 秒内跳 10 个不同位置
7. 用 `M-x imenu` 跳到任意函数定义,用 `M-;` 注释一段 region
8. 30 分钟无鼠标重构 2000 行代码 (capstone)

---

## 9. 下一步

完成这个模块后:

1. 写 `logs/module-01.md` 学习日志
2. 更新 `PROGRESS.md`
3. 进入 Module 2 (`02-dired/README.md`)
