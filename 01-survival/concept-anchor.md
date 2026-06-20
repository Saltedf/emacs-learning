# Concept Anchor: Emacs 编辑器手册的概念锚点 (Module 1)

> 这个文件**完全替代** `emacs-manual-30.2/` 的以下章节:
> - `commands.texi` (Characters, Keys and Commands)
> - `basic.texi` (Basic Editing Commands)
> - `mini.texi` (The Minibuffer)
> - `m-x.texi` (Running Commands by Name)
> - `help.texi` (Help) ★最重要★
> - `mark.texi` (The Mark and the Region)
> - `killing.texi` (Killing and Moving Text)
> - `regs.texi` (Registers)
> - `display.texi` (Controlling the Display)
> - `search.texi` (Searching and Replacement)
> - `fixit.texi` (Commands for Fixing Typos)
> - `kmacro.texi` (Keyboard Macros)
>
> 你不需要翻原始 texi。所有内容在下面,配上 Elisp 视角。

---

## 1. Characters, Keys and Commands (替代 commands.texi)

### 1.1 Emacs 的输入模型

理解 Emacs 第一步是理解它的**输入模型**。这看起来枯燥,但每个 Emacs 用户最终都要面对——尤其是当你想自定义键位、调试"为什么这个键不工作"时。

模型是这样的:

```
键盘事件 → key sequence → keymap lookup → command → 执行
```

你按下键盘上的物理键,操作系统传给 Emacs 一个**事件** (input event)。Emacs 把一个或多个事件组合成**键序列** (key sequence)。Emacs 在**键映射** (keymap) 里查这个序列,找到对应的**命令** (command)。命令是一个标了 `(interactive ...)` 的 Elisp 函数,被 `call-interactively` 调用执行。

这五步每一步都有值得深究的细节。下面分开讲。

**键盘事件** (input event):
- 字符键: `a`、`b`、`1`、`!`...
- 修饰键 + 字符: `C-a`、`M-b`、`C-M-s`...
- 功能键: `<f1>`、`<return>`、`<tab>`、`<backspace>`、`<delete>`...
- 鼠标事件: `<mouse-1>`、`<down-mouse-1>`、`<drag-mouse-1>`...
- 其他: `<wheel-down>`、`<touchscreen>`...

事件是 Emacs 能识别的最小输入单元。一个普通字符是事件,一个 Ctrl+字母 是事件,一个鼠标点击也是事件。Emacs 内部用一个**向量** (vector) 表示一系列事件,例如 `[?\C-x ?\C-f]` 表示 `C-x C-f` 这个两事件序列。

**Key sequence** (键序列):
- 单个 event: `a`、`C-x`、`M-x`
- 多个 event: `C-x C-f`、`C-c C-c`、`C-h k C-g`
- 一个 key sequence 最多通常 3-4 个 event

为什么允许多个 event 组成序列? 因为这是 Emacs 扩展键位空间的核心机制 (见 README 1.3 节)。`C-x` 是 prefix,后面可以接任何字符,所以 `C-x C-f`、`C-x C-s`、`C-x b` 都是有效序列,共占用一个 `C-x` 槽位而不是三个。

**Keymap** (键映射):
- 一个嵌套的 hash table
- 从 key sequence 查到 command (一个 symbol)
- 例子: `(global-set-key (kbd "C-c d") 'my-func)` 把 `C-c d` 绑到 `my-func`

keymap 是数据结构,不是配置文件。你可以在 Elisp 里**编程地**修改它。比如遍历所有 buffer 的 keymap,或临时绑定一个键。`(let ((keymap (make-sparse-keymap))) (define-key keymap (kbd "a") 'foo) ...)` 这种动态操作在 Emacs 包里随处可见。

**Command** (命令):
- 一个 Elisp 函数,标了 `(interactive ...)`
- interactive 让它能被 `M-x` 调用,或被键位触发
- 大多数命令通过 keymap 被触发

不是所有 Elisp 函数都是命令。一个函数要有 `(interactive ...)` 形式,才能被用户直接调用。`(+ 1 2)` 是函数但不是命令,你不能 `M-x +`。而 `delete-char` 有 `(interactive "p")`,所以可以 `M-x delete-char` 或 `C-d`。

### 1.2 Modifier 详解

modifier 是 Emacs 输入模型的另一层抽象。一个事件可以包含多个 modifier:

```
C-    Control       PC: Ctrl 键
M-    Meta          PC: Alt 键,Mac: Option 键,TTY: ESC 然后下一个键
S-    Shift         Shift 键
s-    Super         PC: Win 键,Mac: Cmd 键
H-    Hyper         无默认,自己绑
A-    Alt           老式,等于 Meta
```

为什么有这么多 modifier? 因为 1970s 的 Space-cadet 键盘 (MIT 用的) 物理上有 7 个 modifier 键: Control、Meta、Super、Hyper、Shift、Top、Front。Emacs 设计时假设这些键都存在。现代 PC 键盘只有 Ctrl、Alt、Shift、Win/Cmd,所以大部分 modifier 是"理论存在但实际没用"。

实际上你日常只用 C- 和 M-。其他 modifier 留给自定义,比如 `(setq w32-lwindow-modifier 'super)` 把 Win 键当 Super。

**Multiple modifiers**:
- `C-M-s` = Control + Meta + s
- `C-S-f` = Control + Shift + f
- `M-S-t` = Meta + Shift + t (有时是 `M-T`,大写)

**TTY 的特殊性**:
- TTY 下有些组合键不能传给 Emacs (终端会拦截)
- 解决: 用 ESC 代替 Meta (`ESC x` = `M-x`)
- GUI 没这限制

TTY 限制的根因: 终端协议 (VT100 等) 用字节序列传键,Ctrl+字母 是 1-31 的控制字符 (例如 Ctrl+A 是 0x01)。但终端自己也要用一些控制字符 (比如 Ctrl+S 是 XOFF 流控),这些被终端"截胡"了,Emacs 收不到。所以 TTY 下 `C-s` 可能触发流控而不是 isearch,要 `(stty -ixon)` 关闭流控。GUI 下没有这个问题。

### 1.3 命令的命名约定

Emacs 命令名有强约定。这不是巧合—— Stallman 在 1976 年定下了这些约定,50 年来一直被遵守。学懂这些前缀,你看到一个陌生命令名就能猜出它干啥:

| 模式 | 例子 | 含义 |
|---|---|---|
| `forward-X` | `forward-char`、`forward-word` | 向前 |
| `backward-X` | `backward-char`、`backward-word` | 向后 |
| `next-X` | `next-line` | 下一 |
| `previous-X` | `previous-line` | 上一 |
| `beginning-of-X` | `beginning-of-buffer`、`beginning-of-defun` | 跳到开头 |
| `end-of-X` | `end-of-buffer`、`end-of-line` | 跳到末尾 |
| `kill-X` | `kill-region`、`kill-word` | 杀 (进 kill ring) |
| `delete-X` | `delete-char`、`delete-indentation` | 删 (不进 ring) |
| `copy-X` | `copy-to-register` | 复制 |
| `yank` | `yank` | 粘贴 |
| `mark-X` | `mark-word`、`mark-defun` | 设置 mark |
| `transpose-X` | `transpose-chars`、`transpose-words` | 交换位置 |
| `fill-X` | `fill-paragraph` | 填充 (自动换行) |
| `indent-X` | `indent-region` | 缩进 |
| `comment-X` | `comment-line`、`comment-region` | 注释 |
| `capitalize-X` | `capitalize-word` | 首字母大写 |
| `upcase-X` | `upcase-word` | 全大写 |
| `downcase-X` | `downcase-word` | 全小写 |
| `X-region` | `kill-region`、`comment-region` | 对 region 生效 |
| `my-` / `user-` | `my-insert-date` | 用户自定义前缀 |

**学习窍门**: 看到 `forward-` 开头的命令,猜它"向前移某种单位"。

**为什么这个约定重要?** 因为 Emacs 有几千个命令,你不可能都记住。约定让你**从名字推断行为**:`backward-kill-sexp` 你没见过,但一看就知道是"向前杀一个 sexp"。这种"可猜性"是 Emacs 文化的重要部分。

### 1.4 完整流程: 按键到执行

让我们走一遍 `C-x C-f` 的完整流程,理解每一层发生了什么。这是 Module 3 (Elisp) 的预热,但即使是 Module 1 也值得理解,因为这能帮你 debug "为什么我的键不工作"。

按 `C-x C-f`:

1. **Emacs 收到事件**: `C-x`
   - 操作系统把 Ctrl+X 转成 0x18 (CAN 字符),Emacs 的事件循环 `read-key-sequence` 收到这个字节
2. **`active-keymap` 查找**: `C-x` 是 prefix,返回 `Control-X-prefix` keymap
   - Emacs 看当前 buffer 的 keymap 链 (minor mode keymaps → local map → global map),找到 `C-x` 对应的是另一个 keymap (prefix map),所以它**不执行命令**,而是进入"等待下一个键"状态
3. **等待下一个事件**: `C-f`
   - echo area 显示 `C-x-`,提示你已经在 prefix 状态
4. **在 Control-X-prefix 里查 `C-f`**: 找到 `find-file` 命令
   - 现在 Emacs 在 prefix map 里查 `C-f`,得到 `find-file` 这个 symbol
5. **执行 `find-file`**: 因为它有 `(interactive)`,调用 `(call-interactively 'find-file)`
   - `call-interactively` 是关键: 它读取命令的 `interactive` 形式,生成参数,然后调用命令函数
6. **`find-file` 的 interactive 形式**: `(interactive "FFind file: ")` 提示输入文件名
   - `"F"` 是 interactive code character,表示"读一个文件名,补全"。`"Find file: "` 是 prompt
7. **minibuffer 接管**: 你输入路径,ENTER
   - minibuffer 是一个特殊的 buffer,接管输入。补全用 `*Completions*` buffer 显示候选
8. **打开文件**: 新 buffer 创建,显示
   - `find-file` 调用 `find-file-noselect` (实际干活的函数),后者读文件、创建 buffer、设置 major mode,然后 `switch-to-buffer` 显示

这 8 步在 100ms 内完成,你只感觉到"按了 C-x C-f,输入文件名,文件打开了"。但每一步都是可观察、可干预的: 你可以用 `C-h k C-x C-f` 看绑定,可以用 `M-x find-file` 直接调用,可以用 `(let ((read-file-name-function #'my-func)) (call-interactively 'find-file))` 改文件名读取逻辑。

这就是 Emacs 的透明性。

---

## 2. Basic Editing Commands (替代 basic.texi)

### 2.1 移动 (详细见 Drill 01)

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-f` | `forward-char` | 向前一字符 |
| `C-b` | `backward-char` | 向后一字符 |
| `C-p` | `previous-line` | 上一行 |
| `C-n` | `next-line` | 下一行 |
| `M-f` | `forward-word` | 向前一词 |
| `M-b` | `backward-word` | 向后一词 |
| `C-a` | `move-beginning-of-line` | 行首 |
| `C-e` | `move-end-of-line` | 行尾 |
| `M-a` | `backward-sentence` | 上一句首 |
| `M-e` | `forward-sentence` | 下一句首 |
| `M-{` | `backward-paragraph` | 上一段 |
| `M-}` | `forward-paragraph` | 下一段 |
| `C-v` | `scroll-up-command` | 下一屏 |
| `M-v` | `scroll-down-command` | 上一屏 |
| `M-<` | `beginning-of-buffer` | buffer 头 |
| `M->` | `end-of-buffer` | buffer 尾 |
| `C-l` | `recenter` | 当前行居中 |
| `M-r` | `move-to-window-line-top-bottom` | 在 window 顶/中/底跳 |

**Prefix arg** 应用到移动:
- `C-u 5 C-f` 向前 5 字符
- `M-5 C-n` 向下 5 行
- `C-u C-p` 等同于 `C-p` (前缀数字对单字符移动意义不大)
- `C-u C-a` 切换"跳到第一个非空白字符"和"行首" (取决于 `visual-order-mode`)

### 2.2 删除 (详细见 Drill 02)

| 键位 | 命令 | 作用 |
|---|---|---|
| `DEL` | `delete-backward-char` | 向后删一字符 |
| `C-d` | `delete-char` | 向前删一字符 |
| `M-DEL` | `backward-kill-word` | 向后杀一词 (进 ring) |
| `M-d` | `kill-word` | 向前杀一词 |
| `C-k` | `kill-line` | 杀到行尾 |
| `M-k` | `kill-sentence` | 杀到句尾 |
| `C-S-backspace` | `kill-whole-line` | 杀整行 (含换行) |
| `C-w` | `kill-region` | 杀选中的 region |
| `M-w` | `kill-ring-save` | 复制 region 到 ring |
| `C-y` | `yank` | 粘贴 |
| `M-y` | `yank-pop` | 替换为 ring 中前一项 |

### 2.3 文件操作

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-x C-f` | `find-file` | 打开文件 |
| `C-x C-s` | `save-buffer` | 保存当前 buffer |
| `C-x s` | `save-some-buffers` | 问每个未保存 buffer 是否保存 |
| `C-x C-w` | `write-file` | 另存为 |
| `C-x i` | `insert-file` | 在 point 插入另一个文件内容 |
| `C-x C-v` | `find-alternate-file` | 替换当前 buffer 为另一个文件 |
| `C-x d` | `dired` | 打开 dired (目录编辑) |

### 2.4 Buffer 操作

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-x b` | `switch-to-buffer` | 切换 buffer (默认上一个) |
| `C-x C-b` | `list-buffers` | 列所有 buffer |
| `C-x k` | `kill-buffer` | 关闭 buffer |
| `C-x left` | `previous-buffer` | 上一个 buffer (在 buffer 列表) |
| `C-x right` | `next-buffer` | 下一个 buffer |

### 2.5 退出 / 取消

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-g` | `keyboard-quit` | 取消当前命令/minibuffer |
| `C-x C-c` | `save-buffers-kill-terminal` | 退出 Emacs |
| `C-z` | `suspend-emacs` | 挂起 (TTY: 回 shell; GUI: 最小化) |
| `M-x` | `execute-extended-command` | 用名字跑命令 |

### 2.6 Transpose (交换)

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-t` | `transpose-chars` | 交换相邻两字符 |
| `M-t` | `transpose-words` | 交换相邻两词 |
| `C-M-t` | `transpose-sexps` | 交换相邻两 sexp |
| `C-x C-t` | `transpose-lines` | 交换相邻两行 |

**实际场景**: 你打了 "teh",光标在 h 后,按 `C-t` → "the"。

### 2.7 Case (大小写)

| 键位 | 命令 | 作用 |
|---|---|---|
| `M-c` | `capitalize-word` | 词首字母大写 |
| `M-u` | `upcase-word` | 词全大写 |
| `M-l` | `downcase-word` | 词全小写 |
| `C-x C-u` | `upcase-region` | region 全大写 |
| `C-x C-l` | `downcase-region` | region 全小写 |

**负数前缀**: `M-- M-u` 把前一个词大写。

---

## 3. The Minibuffer (替代 mini.texi)

### 3.1 Minibuffer 是什么

**Minibuffer** 是 Emacs 屏幕最底部的特殊 buffer。
- 名字: ` *Minibuf-0*` (有多个 minibuffer 时会编号)
- major mode: `minibuffer-mode`
- 用途: 接收用户输入 (文件名、命令名、参数)

不是所有 buffer 都是 minibuffer。只有 `*Minibuf-N*` 这些特殊的才是。

### 3.2 进入 minibuffer 的命令

任何需要"输入参数"的命令都会用 minibuffer:

| 命令 | 提示 | 你输入 |
|---|---|---|
| `find-file` | `Find file: ~/` | 文件路径 |
| `switch-to-buffer` | `Switch to buffer: ` | buffer 名 |
| `execute-extended-command` | `M-x ` | 命令名 |
| `kill-buffer` | `Kill buffer: ` | buffer 名 |
| `replace-string` | `Replace string: ` | 字符串 |
| `query-replace` | `Query replace: ` | 字符串 |
| `read-from-minibuffer` | 自定义 | 任意输入 |

### 3.3 在 minibuffer 里

进入 minibuffer 后,你在一个**特殊 buffer** 里编辑。
- `RET` 提交
- `C-g` 取消
- `M-p` 上一条历史
- `M-n` 下一条历史
- `TAB` 补全 (文件名/buffer 名/命令名)
- `SPC` 也补全 (在文件名场景,会插入空格则要 `C-q SPC`)
- `C-M-i` (alternate completion)
- `M-RET` 提交 (recursive minibuffer 模式)
- `?` 显示补全帮助

### 3.4 Minibuffer 历史

每个 prompt 类型有独立历史:
- `find-file` 历史: 你最近输入的文件路径
- `switch-to-buffer` 历史: 最近的 buffer 名
- `M-x` 历史: 最近的命令名

变量 `history-length` 控制保留多少 (默认 100)。

`M-p`/`M-n` 在历史里上下翻。

### 3.5 Recursive Minibuffer

通常你按 `C-x C-f` 进入 minibuffer,在提交/取消前不能再进另一个 minibuffer。
但 `enable-recursive-minibuffers` 设 `t` 后可以:

```
C-x C-f           Find file: ~/...
M-x               (recursive!) M-x find-
...
```

新手不要开 (会迷路),Module 4 配置时可以开。

### 3.6 默认值

许多 minibuffer 提示带默认值:

```
Switch to buffer: (default foo.txt)
```

直接 `RET` 接受默认。或自己输入别的。

### 3.7 Completions buffer

按 `TAB` 后,候选显示在 `*Completions*` buffer (浮窗)。

```
Click <mouse-2> on a completion to select it.
Type M-n to move to next completion, M-p to previous.

Possible completions are:
foo.txt     foo-bar.txt    foo_baz.txt
```

- `M-n`/`M-p` 在 `*Completions*` 里上下移
- `M-v` 切到 `*Completions*` buffer
- 鼠标点击也能选

Emacs 28+ 有更现代的补全 UI (child-frame),Module 5 的 vertico/corfu 是社区主流。

---

## 4. Running Commands by Name (替代 m-x.texi)

### 4.1 M-x 的作用

每个命令有名字 (symbol)。你可以用 `M-x NAME RET` 跑任何命令,不管它绑没绑键。

```
M-x replace-string RET
M-x magit-status RET
M-x compile RET make RET
```

`M-x` 是 `execute-extended-command` 的键位。

### 4.2 命名查找

输入命令名时,可以:
- 完整输入: `replace-string`
- 部分输入 + TAB: `rep TAB` → 补全到 `replace` 开头的命令
- `C-M-i` 显示所有候选

Emacs 27+ 默认开启 fuzzy 补全 (`completion-styles` 含 `flex`),可以 `M-x rsp` 匹配 `replace-string` (r-s-p 各字符顺序)。

### 4.3 命令历史的几个变体

| 键位 | 命令 | 作用 |
|---|---|---|
| `M-x` | `execute-extended-command` | 跑命令 |
| `M-X` | 同上,但只限当前 major mode 的命令 | |
| `<f1> x` | `smarter-M-x` (内置) | 增强版 (Emacs 28+) |

Module 5 会用 vertico + consult 替代默认补全。

### 4.4 命令前缀参数

```
C-u 5 M-x forward-char RET
```

数字前缀可以传给命令。

---

## 5. Help (替代 help.texi) ★最重要★

### 5.1 自文档化的灵魂

让我停下来,认真讲清楚一件事: **Emacs 的 help system 不是"附加功能",它是 Emacs 的世界观核心**。

普通软件把"代码"和"文档"分开。开发者写代码,文档工程师写文档,QA 测试功能。三者可能不同步——代码改了文档没改,文档说有的功能其实被移除了。用户出问题就 Google,搜到的答案可能针对 5 年前的版本,可能作者理解错了,可能是过时的 workaround。这是一场持续的信息损耗游戏。

Emacs 走了完全不同的路: **文档和代码同源**。每个函数的 docstring 写在函数定义上方,每个变量的文档写在 `defvar` 之后。当你 `C-h f` 查询一个函数,Emacs 直接从**当前进程的内存里**读它的 docstring——不是从外部手册,不是从 wiki,是从这个正在运行的 Emacs 进程。

这意味着:
- 你看到的文档与代码**绝对同步** (因为是同一份源码)
- 文档与你的 Emacs 版本**绝对匹配** (因为就是你的 Emacs 自己)
- 不需要 Google,问 Emacs 自己就行
- 文档准确度 100%

### 5.2 历史背景: 自由软件的灵魂

1976 年 Stallman 在 MIT AI Lab 工作,那里有一种"黑客文化": 任何软件都应该**完全可被用户理解和修改**。当时的软件大多是黑盒——你买来,看不到源码,出问题只能找厂商。

Stallman 后来离开 MIT 创立 GNU 项目 (1983 年),核心信念是: **"用户应该能完全理解自己的工具。不理解 = 没有自由。"**

这个信念的具体体现就是 Emacs 的自文档化。当你 `C-h v` 看到一个变量的当前值,你看到的不是文档描述的"应该",是**这个进程内存里真实的值**。这是"透明"的极致——你的工具对你没有任何秘密。

1985 年 GNU Emacs 用 Texinfo 写手册,**手册和源码同步发布**,`C-h` 命令直接读取这些手册和 docstring。这在 1985 年是激进的,WWW 要到 1991 年才出现。Emacs 的 help system 比万维网早 6 年。

这种哲学是为什么 Emacs 50 年来一直活跃: 任何用户都可以深入理解它,任何包都可以被审计,任何行为都可以被查询。VS Code 是闭源 (核心),Sublime 闭源,JetBrains 半闭源。Emacs 是少数"完全对你透明"的编辑器。

### 5.3 C-h 命令大全

**核心 9 个** (你必须肌肉记忆):

| 键位 | 命令 | 用途 |
|---|---|---|
| `C-h f` | `describe-function` | 查函数 (文档 + 源码链接) |
| `C-h v` | `describe-variable` | 查变量 (文档 + 当前值) |
| `C-h k` | `describe-key` | 查键位 (按一个键,显示它跑的命令) |
| `C-h w` | `where-is` | 反向: 命令绑到哪个键 |
| `C-h m` | `describe-mode` | 当前 major mode + minor modes + 键位 |
| `C-h b` | `describe-bindings` | 当前所有键位绑定 |
| `C-h a` | `apropos-command` | 按关键字搜命令 |
| `C-h o` | `describe-symbol` | 查 symbol (函数或变量) |
| `C-h i` | `info` | 进 Info 手册系统 |

这 9 个里有 4 个是"必背中的必背": `C-h f`、`C-h v`、`C-h k`、`C-h a`。每次遇到不懂的 symbol 或键位,你要条件反射地按这 4 个之一。

**辅助 9 个** (查得着就行):

| 键位 | 命令 | 用途 |
|---|---|---|
| `C-h c` | `describe-key-briefly` | 简短版 describe-key (只在 echo area) |
| `C-h F` | `Info-goto-emacs-command-node` | 命令的"教程式"文档 (易读) |
| `C-h S` | `info-lookup-symbol` | 在 Info 里查 symbol (跨手册) |
| `C-h r` | `info-emacs-manual` | 直接打开 Emacs manual |
| `C-h C-f` | `view-emacs-FAQ` | Emacs FAQ |
| `C-h C-n` | `view-emacs-news` | NEWS 文件 (这个版本加了什么) |
| `C-h C-p` | `view-emacs-problems` | 已知问题 |
| `C-h C-t` | `view-emacs-todo` | TODO 列表 |
| `C-h C-c` | `describe-copying` | GPL |

特别值得注意的是 `C-h F` (大写 F) 和 `C-h f` (小写 f) 的区别。`C-h f` 显示 docstring (技术性,简短);`C-h F` 跳到 Info 手册里讲这个命令的章节 (易读,有例子)。新手应该先用 `C-h F`,理解后再用 `C-h f` 查技术细节。

**查找类 6 个**:

| 键位 | 命令 | 用途 |
|---|---|---|
| `C-h e` | `view-echo-area-messages` | 看 `*Messages*` buffer |
| `C-h l` | `view-lossage` | 看你最后按的 300 个键 ★调试神器★ |
| `C-h d` | `apropos-documentation` | 在 docstring 里搜 |
| `C-h .` | `display-local-help` | 显示 point 处的 local help |
| `C-h q` | `help-quit` | 关闭 help window |
| `C-h RET` | `display-help-at-point` | point 处的帮助 |

`C-h l` (view-lossage) 是被低估的神器。你刚做了一串操作,出问题了,但你不记得按了啥。`C-h l` 显示你最后 300 个键——这就是"事后取证"工具。我每周用它至少 5 次,debug 各种"为什么刚才不对"。

**导航类 4 个**:

| 键位 | 命令 | 用途 |
|---|---|---|
| `C-h u` | `help-go-back` | 在 help history 里后退 |
| `C-h TAB` | `help-go-forward` | 前进 |
| `C-h C-h` | `help-for-help` | 显示所有 C-h 子命令 |
| `C-h ?` | 同上 | |

**总结**: 用 `C-h C-h` 不记得时随时查。

### 5.4 describe-function (`C-h f`)

最常用。例子: `C-h f find-file RET`

你会看到:

```
find-file is an interactive compiled Lisp function in ‘files.el’.

(find-file FILENAME &optional WILDCARDS)

Edit file FILENAME.
Switch to a buffer visiting the file FILENAME,
creating one if none already exists.
...

[back] [forward] ...   ← 链接

Click ‘files.el’ to jump to source.
```

**关键功能**:
- 文档 (从 docstring 自动生成)
- 源码位置 (可点击跳转)
- 链接到相关函数 (鼠标点击或 TAB)
- "customize" 链接 (如果是 defcustom)

`C-h f RET` (空名字) 显示所有函数的列表 (info-style)。

注意 "interactive compiled Lisp function" 这个标签——它告诉你三件事: (1) 这个函数是 interactive (能用 M-x 调),(2) 是编译过的 (byte-compiled,比 interpreted 快),(3) 是 Lisp 写的 (不是 C)。如果是 C 写的,会显示 "C source code"。这个标签能帮你判断函数的复杂度和性能特征。

### 5.5 describe-variable (`C-h v`)

例子: `C-h v kill-ring RET`

你会看到:
- 文档
- 默认值
- 当前值 ★★★ (这个超有用)
- 谁设的 (custom? 用户 init?)
- 是否 buffer-local

```
kill-ring is a variable defined in ‘simple.el’.

Its value is
("hello" "world" ...)


Documentation:
The killed text....
```

**第一性原理**: 变量的当前值在内存里,`C-h v` 直接读出来给你看。这就是为什么 Emacs"自文档"——它知道自己的状态。

这个能力的妙处在于 debug。你 init.el 里写了 `(setq make-backup-files nil)`,但保存时还是生成 `~` 备份文件。原因可能很多: 文件加载顺序错了,某行覆盖了你的设置,某包改了它。你怎么诊断? `C-h v make-backup-files RET`,看**当前值**是什么。如果还是 `t`,说明你的 setq 没生效——可能是 init.el 加载顺序问题。如果是 `nil`,那问题在别处。

这种"读运行时状态"的能力是传统 IDE 做不到的。VS Code 的设置文档告诉你"应该是什么",但读不到"现在是什么"。Emacs 的 `C-h v` 读到的是真相。

### 5.6 describe-key (`C-h k`)

按 `C-h k` 然后按**任何键组合**,看它跑什么命令。

例子: `C-h k C-g`

```
C-g runs the command keyboard-quit (found in global-map), which is
an interactive compiled Lisp function in ‘simple.el’.

It is bound to C-g.

(keyboard-quit)

Quit the current command....
```

**用法**:
- 不知道 `C-c C-v` 干啥? `C-h k C-c C-v`
- 别人的 init.el 写了 `(global-set-key (kbd "C-c d") ...)`,你想看绑了啥? `C-h k C-c d`

`C-h k` 最有价值的场景是**学习别人的配置**。你 clone 一个高 star 的 emacs config,看到满屏的 `(global-set-key (kbd "...") ...)`,你不可能记住所有键位。但你可以打开一个 buffer,逐个按 `C-h k` + 那个键,看它实际绑了什么命令,然后判断这个配置是不是你想要的。

`C-h k` 还有一个妙用: 学习 mode。打开一个不熟悉的文件 (比如 yaml),按 `C-h k` 然后试 mode 默认的 prefix (`C-c C-c`? `C-c C-v`?),看它绑了啥。这是"边用边学"的范式。

### 5.7 where-is (`C-h w`)

反向查: 命令绑到哪个键。

例子: `C-h w save-buffer RET`

```
save-buffer is on C-x C-s, <menu-bar> <file> <save-buffer>
```

如果没绑键: `save-buffer is not on any key`。

`C-h w` 的使用场景: 你知道命令名 (从某博客或别人代码里看到),但不知道键位。`C-h w` 告诉你这个命令的**所有**绑定 (global、mode-specific、menu)。比 `C-h k` 反过来,适合"从命令找键位"。

### 5.8 describe-mode (`C-h m`)

显示当前 major mode + 所有 active minor modes,以及它们的键位。

例子: 在 `foo.py` (python-mode) 里 `C-h m`:

```
python-mode defined in ‘python.el’:
Major mode for editing Python files...
...

python-mode key bindings:
key             binding
---             -------
C-c C-p         run-python
C-c C-r         python-shell-send-region
...

Minor modes enabled in this buffer:
abbrev Auto-Revert Display-Line-Numbers...
```

**用法**: 打开任何不熟悉的 mode,先 `C-h m` 看它提供什么。

这是**学习新 mode 的标准动作**。你打开一个 markdown 文件,emacs 自动启用 markdown-mode。你怎么知道这个 mode 提供啥? `C-h m` 看。markdown-mode 有 `C-c C-s b` (加粗)、`C-c C-l` (插入链接)、`C-c C-a L` (插入链接的另一种方式)... 这些键位不在 global map,只在 markdown-mode 里有效。`C-h m` 列出所有这些 mode-specific 键。

更深一层: `C-h m` 也列出**所有 active 的 minor modes**。你装了 20 个 minor mode,有些你忘了装过,有些是某个包自动启用的。`C-h m` 让你看到完整的 active mode 列表,每个 mode 还可以点进去看它提供什么键位。这是"审计你的配置"的工具。

### 5.9 describe-bindings (`C-h b`)

所有键位绑定,包括 mode 特定的。

```
...
key             binding
---             -------
C-a             move-beginning-of-line
C-b             backward-char
C-c TAB         ...
C-c C-c         ...
...
```

`C-h b` 和 `C-h m` 的区别: `C-h m` 重点在"这个 mode 是什么、提供什么命令",`C-h b` 重点在"所有键位一张表"。如果你想找"我装的那个包的键位在哪",`C-h b` 然后 `C-s 包名` 搜更快。

### 5.10 apropos (`C-h a`)

按关键字搜命令。

例子: `C-h a buffer RET`

会搜名字含 "buffer" 的所有命令,列出它们的文档摘要。

变体:
- `C-h d` (apropos-documentation): 在 docstring 里搜
- `M-x apropos-variable`: 只搜变量
- `M-x apropos`: 搜所有 (函数+变量+face)

`C-h a` 和 `C-h d` 的区别: `C-h a` 搜命令**名字**,`C-h d` 搜命令的**文档**。前者适合"我想找名字里有 X 的命令",后者适合"我想找能干 X 事的命令,但不知道叫什么"。

比如: 我想找"删除空行"的命令,但不知道叫啥。`C-h d "remove empty lines"` 在 docstring 里搜 "remove empty lines",可能找到 `delete-blank-lines`。如果用 `C-h a blank`,直接搜名字,也找到。两个工具互补。

### 5.11 describe-symbol (`C-h o`)

`C-h f` 只查函数,`C-h v` 只查变量,`C-h o` 查 symbol (函数或变量,自动判断)。

例子: `C-h o kill-ring RET` → 显示变量
例子: `C-h o kill-region RET` → 显示函数

为什么需要 `C-h o`? 因为 Emacs 的命名空间里,函数和变量是**分开**的——你可以有同名函数和同名变量 (虽然罕见)。`C-h f kill-region` 找函数,`C-h v kill-region` 找变量,但如果你不知道 `kill-region` 是函数还是变量,`C-h o` 自动判断,省去试错。

### 5.12 Info (`C-h i`)

Info 是 Emacs 的手册阅读系统,所有 GNU 手册都在里面。

```
C-h i       进入 Info 目录 (dired 风格)
m NAME RET  跳到某个手册
n           下一节
p           上一节
u           上一级
l           后退 (history)
r           前进
TAB         下一链接
S-TAB       上一链接
RET         跟随链接
q           退出
g NAME RET  跳到节点 NAME
i SYMBOL RET  在索引里查 symbol
s REGEXP RET  全文搜索
```

快捷:
- `C-h r` = 直接进 Emacs manual
- `C-h S` = 跨手册查 symbol

Info 不是 web (1991 年才有的)。它是 1980s GNU 项目自己设计的超文本系统,有链接、有索引、有交叉引用,比 man (Unix 标准手册) 强得多。Emacs manual 有 1000+ 页,全部是 Info 格式,`C-h i` 进去就能读。

**Info 的导航哲学**: 每个手册是一棵树 (tree),有 root、branch、leaf。`n`/`p` 在同一层兄弟节点间移动,`u` 回到父节点,`m` 进入子节点。`l`/`r` 是 history (像浏览器的后退/前进)。`i` 用索引查 symbol,`s` 全文搜索。这套导航在 1985 年是革命性的,比 web 早 6 年。

### 5.13 view-lossage (`C-h l`)

显示你最后按的 300 个键。

```
最近 100 个键盘事件:
C-x C-f ~/foo.txt RET
C-s function
M-y
C-g
...
```

**调试神器**: 你刚做了一串操作,出问题了,`C-h l` 看你到底按了啥。

为什么要专门讲 `C-h l`? 因为它解决一个特别具体的问题: **"我刚才干了什么? "**。你按了一串键,出现意外结果 (光标飞了,文本没了,buffer 切换了),你想还原操作但记不清按了啥。`C-h l` 列出你最后 300 个事件,你可以看到完整序列,找到出错的那一步。

这对 debug kmacro 尤其有用。你录宏时录错了,但不知道哪步错。`C-h l` 看录的过程,精确找到错的那步。

### 5.14 Help System 的核心思维

**第一性原理**: 

```
传统软件:    用户 → 操作 → 软件行为 (黑盒)
              ↓
              Google/Wiki/Manual (外部,可能过时)

Emacs:      用户 → 操作 → 软件行为 (透明)
              ↓
              C-h (软件自己描述自己,与代码同源)
```

这是两种完全不同的世界观。普通软件假设用户是**消费者**,只能用别人设计好的功能,出问题就查文档。Emacs 假设用户是**主人**,可以理解、修改、扩展任何东西,出问题就问 Emacs 自己。

**学 Emacs 的元方法**:
- 看到 `foo-bar-mode` 不知道? `C-h f foo-bar-mode`
- 看到 `(setq ring-bell-function 'ignore)`,想知道 ring-bell-function 干啥? `C-h v ring-bell-function`
- 别人按了某个键,你不知道干啥? 让他再按一次,你按 `C-h k` + 同样的键
- 找命令? `C-h a 关键字`
- 想看手册? `C-h r` 或 `C-h i`

这一套是**肌肉记忆**。训练 1 个月,你的反应速度从"打开浏览器 → Google → 筛选结果" (60 秒) 变成"按 C-h" (3 秒)。这就是为什么 Module 1 把 help system 列为最重要。

---

## 6. The Mark and the Region (替代 mark.texi)

### 6.1 Mark 的概念

**Mark** 是 buffer 里一个"记住的位置"。
- 设置: `C-SPC` (`set-mark-command`)
- 用途 1: 标记 region 起点 (point 和 mark 之间)
- 用途 2: 跳回 (mark ring 历史)

### 6.2 设置 mark

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-SPC` | `set-mark-command` | 在 point 设 mark,激活 region |
| `C-SPC C-SPC` | (双击) | 设 mark 但**不**激活 region (只记位置) |
| `C-x C-x` | `exchange-point-and-mark` | 交换 point 和 mark |
| `C-u C-SPC` | | 跳到上一个 mark (pop mark ring) |
| `M-@` | `mark-word` | 设 mark 在词末 |
| `C-M-@` | `mark-sexp` | 设 mark 在 sexp 末 |
| `C-x h` | `mark-whole-buffer` | 全 buffer |
| `M-h` | `mark-paragraph` | 当前段 |

### 6.3 Region 操作

Region = point 和 mark 之间的文本。

激活 region 后:
- `C-w` kill region
- `M-w` 复制 region
- `C-x C-u` region 大写
- `C-x C-l` region 小写
- `M-x comment-region` / `uncomment-region`
- `C-x TAB` indent region
- `M-x fill-region` fill region
- `M-x replace-string` (只替换 region)

### 6.4 Mark ring

每次设 mark,前一个 mark 推入 mark ring。
`mark-ring` 是 buffer-local 变量。

```
mark-ring:    [old-old-mark, old-mark, current-mark]
                                                ↑
                                          当前 mark
```

`C-u C-SPC` 弹一次,跳到上一个 mark。
`C-u C-u C-SPC` 跳到 2 个之前。
连续 `C-u C-SPC C-u C-SPC C-u C-SPC` 在 history 里循环。

**global-mark-ring** 跨 buffer,用 `C-x C-SPC` 跳。

### 6.5 Region 高亮

`transient-mark-mode` (默认开) 让 region 高亮。
关闭后,region 仍存在但不显示颜色。

`delete-selection-mode` (推荐开): 选中 region 后输入直接替换。

---

## 7. Killing and Yanking (替代 killing.texi)

> 详细见 Drill 02。这里给概念锚点。

### 7.1 Kill vs Delete

这两个术语让新手困惑。看起来都是"删除",但 Emacs 故意分了两套。

第一性原理: 删除有两种**语义**——你可能后悔的 (想召回),和你确定不要的 (不召回)。如果你只有一个"删除"操作,你无法区分这两种意图。Emacs 把它们分开:
- **Kill** = 进入 kill-ring 的删除,可召回
- **Delete** = 真删除,不进 ring

| | Kill | Delete |
|---|---|---|
| 进 kill-ring? | 是 | 否 |
| 可召回? | 是 | 否 |
| 命名约定 | `kill-X` | `delete-X` |
| 例子 | `kill-region`, `kill-word` | `delete-char`, `delete-indentation` |

这个区分的妙处: **意图清晰**。看到 `M-d` 是 `kill-word`,你知道删的内容能召回;看到 `M-\` 是 `delete-horizontal-space`,你知道删的空白不会出现在 kill-ring 里 (避免污染)。

什么时候该用 delete 而不是 kill? 当你**确定不要**的时候。比如删空格、删空行——这些内容没有召回价值,放进 kill-ring 只会挤掉有用的东西。所以 Emacs 把 `M-\` 设计为 delete,不是 kill。

### 7.2 Kill ring 结构

```elisp
kill-ring                  ; 一个 list
kill-ring-yank-pointer     ; 指向"下一个 yank 会取的"
kill-ring-max              ; 最大长度,默认 60
interprogram-cut-function  ; 同步到系统剪贴板 (GUI)
interprogram-paste-function ; 从系统剪贴板读
```

为什么是 ring 而不是 stack? 因为 stack 是 LIFO,弹一次就消失。但用户经常想"召回上一次杀的,再召回上上次的,然后跳回最近的"——需要循环浏览。ring 提供这个能力: `M-y` 在 ring 里转,转一圈回到原点。

为什么默认 60? 这是经验值——大多数用户召回不会超过 60 项之前的。如果你是重度用户,`(setq kill-ring-max 200)` 改大。代价是更多内存 (每项可能有几 KB)。

`interprogram-cut-function` 是 Emacs 和系统剪贴板的桥梁。GUI 下默认设置,所以 `M-w` 复制的文本在浏览器里也能 `Ctrl+V` 粘贴。TTY 下这个变量为 nil (因为 TTY 没有系统剪贴板),所以 TTY Emacs 的 kill 不影响系统剪贴板。

### 7.3 Yank 流程

```
C-y     →  yank
            1. kill-ring-yank-pointer 重置到 head
            2. 插入 (car kill-ring-yank-pointer)
            3. 设 mark 在插入文本末

M-y     →  yank-pop (必须在 C-y 之后)
            1. 检查 last-command 是 yank
            2. kill-ring-yank-pointer 后移一位 (循环)
            3. 删除刚才 yank 的
            4. 插入新的 car
```

为什么 `M-y` 必须紧跟 `C-y`? 因为 `M-y` 的工作是"替换刚才 yank 的内容",它需要知道刚才 yank 了什么、yank 在哪里。`yank` 命令把这些信息存在 `last-command` 和 mark 里,`M-y` 读它们。如果你中间按了别的命令,这些信息被覆盖,`M-y` 就报错 "Previous command was not a yank"。

新手经常踩这个坑: 按 `C-y`,想看历史,先按了 `C-l` (recenter),然后按 `M-y`——报错。解决: `M-y` 必须紧跟 `C-y`,中间不要按别的命令。

### 7.4 Append next kill (`C-M-w`)

```
C-w          →  kill 第一段 (kill-ring: ["a"])
C-M-w        →  宣告: 下次 kill append,不增加新项
C-w          →  kill 第二段 (kill-ring: ["a" "second+first"])
```

实际:`C-M-w` 设变量 `this-command` 为 `kill-append`,下次 `kill-region` 检查它,append 而不是 cons。

为什么需要这个? 想象你在编辑代码,要把 3 段不连续的代码合并粘贴到一处。如果不用 `C-M-w`,你 `C-w` 三次,kill-ring 里有 3 项,`C-y` 只能粘一项,要 `M-y` 切换。用 `C-M-w`,三次 kill 合并成一项,`C-y` 一次粘全部。

### 7.5 Accumulating kills (`M-w` 后接 prefix)

`M-w` 是 copy (不删),进 ring。
`M-w` 跟在 `C-w` 之后会 append。

这是 Emacs 的"连续操作合并"机制: 如果连续两次 `kill-region` 或 `kill-ring-save`,Emacs 把它们合并成一项,而不是新建两项。这避免了"我要连续复制 3 段就污染 kill-ring 3 项"的问题。

---

## 8. Registers (替代 regs.texi)

### 8.1 Register 的概念

Register = 命名 slot (一个字符: a-z, A-Z, 0-9, 一些符号)。
可以存:
- 文本 (string)
- 位置 (integer)
- 矩形 (rectangle)
- 数字 (number)
- 窗口配置 (window-configuration)
- 键盘宏 (kmacro)
- file 名 (file)

### 8.2 文本 register

```
C-x r s a           保存 region 到 register a
C-x r i a           插入 register a 内容到 point
M-x view-register   查看 register 内容
M-x list-registers  列所有 registers
```

### 8.3 位置 register

```
C-x r SPC a         保存当前 point 到 register a
C-x r j a           跳到 register a 的位置
```

非常常用: 你正在写代码,要查另一个文件再回来,`C-x r SPC m` 存位置,查完 `C-x r j m` 回来。

### 8.4 数字 register

```
C-x r n a 5         设 register a 为数字 5
C-x r + a           register a 加 1
C-x r i a           插入数字 (作为文本)
```

### 8.5 窗口配置 register

```
C-x r w a           存当前 window 配置到 register a
C-x r j a           恢复 window 配置
```

用途: 你有一个复杂的 4 窗口布局,存起来,关掉后还能恢复。

### 8.6 Register vs Kill ring

| | Kill ring | Register |
|---|---|---|
| 命名 | 匿名 | 你给名字 |
| 容量 | 60 项环 | 任意 |
| 类型 | 文本 | 文本/位置/数字/窗口配置等 |
| 用途 | 短期 | 长期标记 |

---

## 9. Controlling the Display (替代 display.texi)

### 9.1 Scrolling

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-v` | `scroll-up-command` | 向下翻一屏 |
| `M-v` | `scroll-down-command` | 向上翻一屏 |
| `C-x ]` | `forward-page` | 下一页 (page 是 `\f` 分隔) |
| `C-x [` | `backward-page` | 上一页 |
| `C-l` | `recenter` | 当前行居中 (再按到顶,再按到底) |
| `M-r` | `move-to-window-line-top-bottom` | 在 window 顶/中/底跳 |
| `C-M-l` | `reposition-window` | 智能重定位 |

### 9.2 视觉换行

`(visual-line-mode 1)` 启用视觉换行:
- 长行视觉上换行 (但不插换行符)
- `C-a`/`C-e` 在视觉行首尾
- `M-{`/`M-}` 还是段
- `visual-order-mode` 一起开

或者 `truncate-lines` (默认 nil):
- nil: 长行视觉换行
- t: 长行截断,只能左右滚看

### 9.3 行号

```
(global-display-line-numbers-mode 1)   ; 全局开
(setq display-line-numbers-type 'relative) ; 相对行号
(setq display-line-numbers-type 'visual)   ; 视觉行号 (有 wrap 时)
```

### 9.4 高亮

```
(global-hl-line-mode 1)    ; 高亮当前行
(show-paren-mode 1)        ; 高亮配对括号
(setq show-paren-style 'expression)  ; 高亮整个表达式
(blink-cursor-mode -1)     ; 关闭光标闪烁
```

### 9.5 Fringe

左右两边的灰色区域叫 fringe。显示行 continuation、wrap、git diff 等标记。

```
(fringe-mode '(8 . 8))   ; 左右各 8 像素
(set-fringe-mode 0)      ; 关闭
```

### 9.6 字体和缩放

```
C-x C-+     text-scale-adjust 增大
C-x C--     text-scale-adjust 减小
C-x C-0     text-scale-adjust 重置
```

(只对当前 buffer 生效,不影响 frame 字体)

### 9.7 Window 配置

```
winner-mode  ; window 配置可 undo/redo
C-c <left>   winner-undo (恢复上一个 window 配置)
C-c <right>  winner-redo
```

---

## 10. Searching and Replacement (替代 search.texi)

> 详细见 Drill 03。

### 10.1 Isearch (递增搜索)

```
C-s         向后搜索 (forward)
C-r         向前搜索 (backward)
C-s C-s     重复上次搜索
C-s RET     非递增 (按 ENTER 结束输入)
C-s C-w     把光标处的 word 加入搜索词
C-s C-y     把光标处到行尾加入
C-s C-y C-y 加更多
M-s w       toggle word search
M-s s C-s   symbol search
C-M-s       正则向后搜
C-M-r       正则向前搜
```

### 10.2 Replace

```
M-% FROM RET TO RET   query-replace (问每个匹配)
C-M-%                 正则 query-replace
!                     replace-all (在 query-replace 中)
y / SPC               替换当前
n / DEL               不替换
q                     退出
^                     回到上一个
.                     替换当前,然后退出
u                     撤销上次替换
```

### 10.3 Occur

```
M-s o      occur (列出所有匹配,跳到 occur buffer 编辑)
M-s O      类似但 multi-occur
```

在 occur buffer 里:
- `e` 编辑 (occur-edit-mode),改完 `C-c C-c` 应用到原文件
- `g` 刷新

### 10.4 Imenu

```
M-x imenu         跳到某个定义 (函数/类/章节)
M-s i             imenu (相同)
```

对代码: 跳到任意函数定义
对 org: 跳到任意 headline

### 10.5 Multi-file

```
M-x grep          grep 搜索
M-x rgrep         recursive grep
M-x find-grep     find + grep
M-x ag / M-x ripgrep  (需要安装 ag/rg + 包)
```

结果在 `*grep*` buffer,可点击跳转。

---

## 11. Fixing Typos (替代 fixit.texi)

### 11.1 Undo

Emacs 的 undo 比普通编辑器强:

```
C-/         undo
C-x u       undo (同上,可视化)
C-_         undo (同上,TTY 友好)
```

**关键**: Emacs 默认 undo 是线性的,undo 操作也能被 undo。
如果你 undo 后做了新操作,redo 链断。

```
操作: A B C D
undo:  A B C
undo:  A B
undo:  A
新操作 E: A B E   ← 注意,C D 已经丢失
```

**undo-tree** 包提供树状 undo/redo,可视化看 history。Module 4 装。

### 11.2 Undo ring

`buffer-undo-list` 是一个 list,存所有操作。
默认无上限。

### 11.3 拼写检查

```
M-$             ispell-word (检查当前 word)
M-x ispell      ispell 整个 buffer
M-x ispell-region
M-x flyspell-mode    minor mode, 实时检查
M-x flyspell-prog-mode  只检查注释和字符串 (代码模式)
```

在 ispell 提示下:
- `SPC` 跳过
- 数字 选建议
- `a` 接受这次
- `A` 接受这次 + 加入个人词典
- `r` 自己改
- `q` 退出

---

## 12. Keyboard Macros (替代 kmacro.texi)

> 详细见 Drill 05。

### 12.1 基本流程

```
F3 (或 C-x ()    开始录制
... 操作 ...
F4 (或 C-x ))    结束录制 + 立即运行
C-x e            再运行一次
C-u N C-x e      运行 N 次
C-x C-k r        在 region 内每行运行
```

### 12.2 命名和保存

```
C-x C-k n NAME   给宏命名 (变成一个函数)
M-x insert-kbd-macro  生成宏的 Elisp 代码 (可放到 init.el)
```

### 12.3 编辑宏

```
C-x C-k e        编辑宏
```

打开 `*edit-macro*` buffer,改录下的键,`C-c C-c` 应用。

### 12.4 kmacro ring

宏也存 ring (和 kill ring 类似):
```
C-x C-k C-p    上一宏
C-x C-k C-n    下一宏
C-x C-k b KEY  绑到 KEY
```

### 12.5 kmacro counter

录制时:
```
F3                   开始 + 插入 counter (默认 0)
F3                   继续 + counter +1
F4                   结束
```

`C-x C-k C-c 0 RET` 重置 counter。

### 12.6 实际场景

1. 在一系列相似的行里做同样修改
2. 录一个"找下一行,改成 X,跳到下一行"的宏
3. 用 `C-u 20 C-x e` 跑 20 次

---

## 13. 这个锚点对照到 Elisp

把 editor 概念和 Elisp 对应:

| Editor 概念 | Elisp |
|---|---|
| Point | `(point)`, `point-min`, `point-max` |
| Mark | `(mark)`, `mark-ring`, `set-mark` |
| Region | `(region-beginning)`, `(region-end)`, `(use-region-p)` |
| Buffer | `(current-buffer)`, `get-buffer`, `with-current-buffer` |
| Window | `(selected-window)`, `split-window`, `delete-window` |
| Kill ring | `kill-new`, `kill-append`, `current-kill`, `kill-ring` |
| Register | `set-register`, `get-register`, `register-alist` |
| Search | `search-forward`, `re-search-forward`, `replace-match` |
| Kmacro | `kmacro-exec`, `fset` (宏转函数) |

Module 3+ 会详细学这些函数。Module 1 你只需要会用 (用户视角)。

---

## 14. 自测

完成这个 concept-anchor 后,你应该能:

- [ ] 解释 key sequence、keymap、command 的关系
- [ ] 说出至少 10 个常用 `C-h` 子命令
- [ ] 区分 mark ring 和 kill ring
- [ ] 解释 isearch 的"递增"
- [ ] 知道 undo 在 Emacs 是线性的 (vs undo-tree)
- [ ] 知道 register 可以存哪些类型
- [ ] 知道 kmacro 怎么命名、编辑、保存到 init.el

---

## 15. 下一步

进入 `drills/01-movement.md` 开始 Module 1 的实操训练。
