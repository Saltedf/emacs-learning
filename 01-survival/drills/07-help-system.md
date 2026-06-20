# Drill 07: Help System — Emacs 自我学习的能力来源 ★最重要★

> **预计用时**: 3 小时 (这是 Module 1 的核心)
> **难度**: ★★☆☆☆
> **替代**: `emacs-manual/help.texi` 全文
> **目标**: 任何 Emacs 问题,30 秒内从 Emacs 自己找到答案

---

## 0. 为什么这是 Module 1 最重要的 Drill?

让我们停下来想一个问题: **你刚学 Emacs,你不会的东西有 1000 个。你怎么学?**

这是 Module 1 的根本问题。你可以花 20 小时学完 155 道微练习,但 Emacs 有几千个命令、几百个变量、几十个 mode。你不可能全学完。**真正的生存技能不是"知道多少",而是"不会时怎么找答案"**。

普通人的方法:
- Google "emacs 怎么 X"
- 看 Stack Overflow
- 看博客 (但博客可能过时)
- 看视频教程 (但很慢)

每种方法都有问题:
- Google 结果可能针对老版本 (Emacs 24 时代的博客,你用 Emacs 30,API 已变)
- 博客作者的理解可能错 (博客是"某人认为",不是真相)
- 视频效率低 (10 分钟视频里只有 30 秒是你要的)
- 你不知道答案对不对,只能信 (缺乏验证手段)

**Emacs 的设计哲学: 不依赖外部信息。**

Emacs 的源码就是文档。任何函数、变量、键位的文档**写在源码里**,通过 `C-h` 系列命令直接读取。

这意味着:
- 你看到的文档与代码同源,**绝对不会过时**
- 不需要 Google,问 Emacs 自己就行
- 文档准确度 100% (因为是源码本身)

**Module 1 教你编辑生存,但 help system 教你"自我学习"。**
学会 help system,你后面 5 个月的学习速度会**翻 5 倍**。

这不是夸张。假设你遇到一个陌生命令,普通流程是 Google + 读博客 + 验证,大约 5 分钟。用 `C-h`,大约 10 秒。一天遇到 20 个陌生命令,省 1.5 小时。一个月省 30 小时。5 个月省 150 小时——相当于多出 4 周的学习时间。

更重要: `C-h` 给的是**权威答案**,不是博客的二手解读。你不会学到错误信息,不会浪费时间在过时的 workaround 上。你的 Emacs 知识结构从一开始就是正确的。

### 0.1 这个 Drill 的特殊之处

其他 drill 教你"做",这个 drill 教你"找"。其他 drill 的产出是肌肉记忆 (按 C-s 会搜),这个 drill 的产出是**反射** (看到陌生 symbol 就条件反射地按 C-h)。

反射比肌肉记忆更深层。肌肉记忆是"知道怎么做",反射是"本能地做"。一个优秀的 Emacs 用户,看到 `(setq foo-bar 'baz)` 这种代码,手会自动按 `C-h v foo-bar RET`——不需要思考,不需要决定"要不要查",反射触发。

这种反射需要训练。Module 1 给你 3 小时专门练这个,目标就是建立反射。后面 7 个模块继续强化。

### 0.2 一个具体例子

让我演示 help system 怎么改变你的工作流。

假设你看到别人的 init.el 里有这一行:

```elisp
(setq ring-bell-function 'ignore)
```

你想知道 `ring-bell-function` 干啥的。

**普通软件路径** (5 分钟):
- 打开浏览器
- Google "emacs ring-bell-function"
- 看 5 个博客,说法不一 (有的说"关声音",有的说"自定义铃声",有的过时)
- 你不确定哪个对,选一个看起来合理的试
- 试错几次,最后理解了

**Emacs 路径** (10 秒):
- `C-h v ring-bell-function RET`
- 立刻看到:
  ```
  ring-bell-function is a variable defined in ‘C source code’.

  Its value is 'ignore
  Original value was 'default-ring-bell-function

  Documentation:
  Non-nil means call this function to ring the terminal bell.
  The function should accept no arguments.
  ...
  ```

**10 秒之内**,你得到了:
- 它的定义位置 (C source code,在 src/term.c 或类似)
- 当前值 (`ignore` — 被设为 ignore)
- 默认值 (`default-ring-bell-function` — 默认会响铃)
- 文档 (function to ring bell)

而且这是**从你电脑上 Emacs 进程的内存里直接读的**,与你的 Emacs 版本完全匹配。不可能过时,不可能错。

这就是 help system 的力量。

---

## 1. 第一性原理: 自文档化 (Self-Documentation)

### 1.1 一个根本不同的设计

让我对比一下:

**普通软件**:
```
代码 (开发者看到)
   ↓
   编译
   ↓
可执行文件 (用户看到)
   ↓
   使用
   ↓
   出问题? → 查 Wiki / 论坛 / Google
```

代码和用户**隔离**。文档是另一拨人写的,可能和代码不一致。

**Emacs**:
```
代码 (含 docstring)
   ↓
   加载到运行时
   ↓
   你看到的 Emacs
   ↓
   出问题? → C-h 直接读 docstring
```

代码和用户**无隔**。docstring 是源码的一部分,与代码同生同死。

### 1.2 历史背景

1976 年 Stallman 设计 Emacs 时,有一个核心信念:

> "用户应该能完全理解自己的工具。"
> 
> "不理解 = 没有自由。"

这就是 GNU 自由软件运动的灵魂。Emacs 的"自文档化"是这个信念的体现。

1985 年 GNU Emacs 用 Texinfo (一种文档格式) 写手册,**手册和源码同步发布**。
然后 `C-h` 命令直接读取这些手册和 docstring。

### 1.3 一个具体例子

让我演示"自文档化"的力量。

假设你看到别人的 init.el 里有这一行:

```elisp
(setq ring-bell-function 'ignore)
```

你想知道 `ring-bell-function` 干啥的。

**普通软件路径**:
- Google "emacs ring-bell-function"
- 看 5 个博客,说法不一
- 不确定答案对不对

**Emacs 路径**:
- `C-h v ring-bell-function RET`
- 立刻看到:
  ```
  ring-bell-function is a variable defined in ‘C source code’.
  
  Its value is 'ignore
  Original value was 'default-ring-bell-function
  
  Documentation:
  Non-nil means call this function to ring the terminal bell.
  The function should accept no arguments.
  ...
  ```

**3 秒之内**,你得到了:
- 它的定义位置 (C source code)
- 当前值 (`ignore`)
- 默认值 (`default-ring-bell-function`)
- 文档 (function to ring bell)

而且这是**从你电脑上 Emacs 进程的内存里直接读的**,与你的 Emacs 版本完全匹配。

这就是自文档化。

---

## 2. 核心命令大全 (慢一点,这是核心)

我会一个一个讲,每个都举例。**不要跳读,每个命令都要在 Emacs 里实操**。

### 2.1 `C-h f` (describe-function) — 查函数

**用法**: `C-h f FUNCTION-NAME RET`

**返回**:
- 函数文档
- 是 interactive 吗
- 接受什么参数
- 源码位置 (可点击)
- 相关链接

**例子**: `C-h f find-file RET`

你会看到:

```
find-file is an interactive compiled Lisp function in ‘files.el’.

(find-file FILENAME &optional WILDCARDS)

Edit file FILENAME.
Switch to a buffer visiting the file FILENAME,
creating one if none already exists.

... (more docs)

[back] [forward] [backward] [forward] [tags]
[custom] [help]

Click ‘files.el’ to jump to the source code.
```

**关键交互**:
- 鼠标点击 `files.el` (或 `TAB` 到链接 + `RET`),跳到源码
- 按 `n` / `p` 在函数间浏览 (按字母序)
- 按 `q` 退出 help window (它会被特殊关闭)

**场景**: 你看到 `(global-set-key (kbd "C-c d") 'my-func)`,想知道 `my-func` 的功能? `C-h f my-func RET`。

### 2.2 `C-h v` (describe-variable) — 查变量

**用法**: `C-h v VARIABLE-NAME RET`

**返回**:
- 文档
- 默认值
- 当前值 ★★★ (这个超级有用)
- 是否 buffer-local
- 谁设的

**例子**: `C-h v kill-ring RET`

```
kill-ring is a variable defined in ‘simple.el’.

Its value is
("hello" "world")

Original value was nil

Documentation:
The killed text...
```

**关键认知**: 你看到的是**当前 Emacs 进程内存里**的 `kill-ring` 值。
不是文档说的"应该是",是**实际是**。

**场景**: 你 init.el 写了 `(setq make-backup-files nil)`,但保存时还生成 `~` 文件。`C-h v make-backup-files RET`,看它**当前值**是什么。如果还是 `t`,说明 init.el 没生效。

### 2.3 `C-h k` (describe-key) — 查键位

**用法**: `C-h k` 然后按**任何键组合**

**返回**: 这个键位跑什么命令 + 命令文档

**例子**: `C-h k C-g`

```
C-g runs the command keyboard-quit (found in global-map), which is
an interactive compiled Lisp function in ‘simple.el’.

It is bound to C-g.

(keyboard-quit)

Quit the current command....
```

**场景**: 你看到别人按 `C-c C-v`,不知道干啥。让他再按一次,你也按 `C-h k C-c C-v`,看绑了啥。

**变体**: `C-h K` (大写 K) 显示更详细的 Info 版本 (如果有)。

### 2.4 `C-h w` (where-is) — 反向查

**用法**: `C-h w COMMAND-NAME RET`

**返回**: 这个命令绑到哪些键

**例子**: `C-h w save-buffer RET`

```
save-buffer is on C-x C-s, <menu-bar> <file> <save-buffer>
```

(如果没绑键: `save-buffer is not on any key`)

**场景**: 你想保存 buffer,知道命令叫 `save-buffer` 但忘了键位。`C-h w save-buffer RET`,看到 `C-x C-s`。

### 2.5 `C-h m` (describe-mode) — 查当前 mode

**用法**: `C-h m`

**返回**:
- 当前 major mode 名 + 文档
- 所有 active minor modes 名 + 文档
- 当前 buffer 的所有键位 (按 mode 区分)

**例子**: 在 `foo.py` (python-mode) 里 `C-h m`:

```
python-mode defined in ‘python.el’:
Major mode for editing Python files...

python-mode key bindings:
key             binding
---             -------
C-c C-p         run-python
C-c C-r         python-shell-send-region
C-c C-s         python-shell-send-string
...

Minor modes enabled in this buffer:
Display-Line-Numbers Show-Paren...
```

**场景**: 你打开一个新文件类型 (org/markdown/yaml...),`C-h m` 看它提供什么键位。**这是学新 mode 的标准动作**。

### 2.6 `C-h b` (describe-bindings) — 查所有键位

**用法**: `C-h b`

**返回**: 当前 buffer 的所有键位绑定 (列表)

```
key             binding
---             -------
C-a             move-beginning-of-line
C-b             backward-char
C-c TAB         ...
C-c C-c         my-org-publish
C-c d           my-func
...
```

**场景**: 你忘了某键位,但知道它大概在 `C-c` 前缀下。`C-h b` 然后 `C-s C-c` 搜。

### 2.7 `C-h a` (apropos-command) — 按关键字搜命令

**用法**: `C-h a KEYWORD RET`

**返回**: 所有名字含 KEYWORD 的命令,带文档摘要

**例子**: `C-h a buffer RET`

```
buffer (display-line-numbers-turn-on) ...
buffer-menu        Command: ...
buffer-substring    ...
list-buffers       ...
switch-to-buffer   ...
... (几十个)
```

**场景**: 你想做"buffer 相关的事",但不知道命令名。`C-h a buffer RET`,浏览。

**变体**:
- `M-x apropos` — 搜所有 (函数 + 变量 + face)
- `M-x apropos-variable` — 只搜变量
- `M-x apropos-function` — 只搜函数
- `C-h d` (apropos-documentation) — 在 docstring 里搜 (更深)

### 2.8 `C-h o` (describe-symbol) — 查 symbol (函数或变量)

**用法**: `C-h o SYMBOL RET`

**返回**: 这个 symbol 是函数还是变量 (或都是),分别的文档

**例子**: `C-h o kill-ring RET` → 显示变量
**例子**: `C-h o kill-region RET` → 显示函数

**与 C-h f / C-h v 的区别**:
- `C-h f` 只查函数,如果你给它变量名会"找不到"
- `C-h v` 只查变量,如果你给它函数名会"找不到"
- `C-h o` 自动判断

**场景**: 看到一个 symbol,不确定它是函数还是变量。`C-h o`。

### 2.9 `C-h i` (info) — 进入 Info 手册系统

**用法**: `C-h i`

**返回**: Info 目录 (所有已安装的手册)

```
* Emacs: (emacs).     The extensible self-documenting text editor.
* Elisp: (elisp).     The Emacs Lisp Reference Manual.
* Eintr: (eintr).     Programming in Emacs Lisp: An Introduction.
* Info: (info).       Info documentation viewer.
... (几十个,包括所有装的包的手册)
```

**Info 导航**:

| 键 | 作用 |
|---|---|
| `n` | 下一节 |
| `p` | 上一节 |
| `u` | 上一级 |
| `l` | history 后退 |
| `r` | history 前进 |
| `TAB` | 下一链接 |
| `S-TAB` | 上一链接 |
| `RET` | 跟随链接 |
| `m NAME RET` | 跳到指定菜单 |
| `g NODE RET` | 跳到指定节点 |
| `i SYMBOL RET` | 在索引里查 symbol |
| `s REGEXP RET` | 全文搜索 |
| `q` | 退出 |
| `?` | 所有命令帮助 |

**场景 1**: 想读 Emacs 官方手册的"Search" 章节。`C-h i m emacs RET s Search`。

**场景 2**: 想知道 Elisp 的 `format` 函数。`C-h i m elisp RET i format`。

**场景 3**: 想看 org-mode 手册。`C-h i m org RET` (如果装了 org 手册)。

### 2.10 `C-h r` (info-emacs-manual) — 直接进 Emacs manual

**用法**: `C-h r`

跳过 Info 目录,直接进 Emacs manual。

**场景**: 想快速查某个编辑命令的详细解释,`C-h r i SOMETHING RET` 在索引查。

### 2.11 `C-h S` (info-lookup-symbol) — 跨手册查 symbol

**用法**: `C-h s SYMBOL RET`

在**所有 Info 手册**里搜 symbol。

**例子**: `C-h S save-buffer RET`,跳到 Elisp manual 里讲 `save-buffer` 的部分。

**与 C-h f 区别**: 
- `C-h f` 显示 docstring
- `C-h S` 跳到 Info 手册里讲这个 symbol 的部分 (更详细的解释)

### 2.12 `C-h F` (Info-goto-emacs-command-node) — 命令的"教程式"文档

**用法**: `C-h F COMMAND-NAME RET`

跳到 Emacs manual 里讲这个命令的部分。

**与 C-h f 区别**:
- `C-h f` 显示 docstring (技术性,简短)
- `C-h F` 显示手册教程 (易读,有例子)

**例子**: `C-h F isearch-forward RET` → 跳到 Emacs manual 的 isearch 章节。

### 2.13 `C-h l` (view-lossage) — 看你最后按了啥

**用法**: `C-h l`

显示你最后按的 100 (默认) 个键。

```
Recent input:
7 C-n C-s foo C-g
C-x C-f ~/test RET
...
```

**场景**: 你刚按了一串操作,出问题了,但你不记得按了啥。`C-h l` 看。

**调试神器**。

### 2.14 `C-h e` (view-echo-area-messages) — 看 *Messages* buffer

**用法**: `C-h e`

跳到 `*Messages*` buffer,看所有 Emacs 给你的消息历史。

**场景**: 你刚才 minibuffer 闪过一个消息,没看清。`C-h e` 看。

### 2.15 `C-h C-h` (help-for-help) — 显示所有 C-h 子命令

**用法**: `C-h C-h`

忘记时随时查。

```
Type C-h followed by one of these letters for more help:

a  apropos-command
b  describe-bindings
c  describe-key-briefly
...
```

---

## 3. Help Buffer 的导航

当你按 `C-h f`/`C-h v`/`C-h m` 等,会打开一个 `*Help*` buffer。

### 3.1 导航键

| 键 | 作用 |
|---|---|
| `q` | 关闭 help window (不杀 buffer) |
| `n` | 下一个 symbol (按字母序) |
| `p` | 上一个 symbol |
| `TAB` | 下一链接 |
| `S-TAB` | 上一链接 |
| `RET` | 跟随链接 |
| `mouse-1` / `mouse-2` | 鼠标点击链接 |
| `C-c C-b` | history 后退 (回到上一个 help) |
| `C-c C-f` | history 前进 |
| `r` | (某些 help) Related references |
| `i` | (某些 help) Info link |

### 3.2 跨 help 的"历史栈"

每次你点链接 (例如从 `C-h f foo` 点 `bar`),进入新的 help。
`C-c C-b` 回到 foo。

这种"超文本浏览"在 1985 年是革命性的,比 web 早。

---

## 4. 工作流程: 一个真实场景

让我用一个真实场景展示 help system 怎么用。

### 场景: 你想"自动保存文件之前跑一个钩子"

#### 错误的方法

Google "emacs before save hook"。
看到一些博客,有的说 `before-save-hook`,有的说 `write-contents-functions`,混乱。

#### 正确的方法

**Step 1**: `C-h a save RET`

浏览结果,看到 `before-save-hook`。

**Step 2**: `C-h v before-save-hook RET`

```
before-save-hook is a variable defined in ‘files.el’.
Its value is nil
This variable is safe as a file local variable.
Documentation:
Normal hook run before saving a buffer.
...
```

读完,你知道:
- 它是一个 hook (函数列表)
- 在 save buffer 之前运行
- 当前是 nil (空)

**Step 3**: 想知道"hook 是啥",`C-h f run-hooks RET`,或 `C-h i m elisp RET s Hooks RET`。

**Step 4**: 想加一个函数,`M-x add-hook RET before-save-hook RET my-func RET`。

**整个过程 1 分钟,完全在 Emacs 内**,不需要 Google。

---

## 5. Info: Emacs 的"维基百科"

Info 是 GNU 的文档系统,所有 GNU 项目用它写手册。

### 5.1 Info 不是 man

Linux 用户熟悉 `man`: 终端里 `man ls` 看 ls 的手册。
Info 是更高级的版本: 超文本,有链接,有索引,跨手册。

```bash
info coreutils   # GNU coreutils 手册
info emacs       # Emacs 手册
info libc        # GNU C 库
```

在 Emacs 里 `C-h i` 直接进。

### 5.2 Emacs 里的 Info 比 bash 里的好

bash info 是 TTY,不能点链接。
Emacs info 可以:
- 鼠标点击链接
- TAB 导航
- 全文搜索 (`s REGEXP`)
- 索引查 (`i SYMBOL`)
- 跨手册搜 (`C-h S`)

### 5.3 实战: 读 Emacs manual 的 Dired 章节

`C-h r` 进 Emacs manual。
`m Dired RET` 跳 Dired 章节。
`n` `n` `n` 翻几节。
`i wdired RET` 在索引查 wdired。
`l` 后退。
`q` 退出。

读完后,你应该能回答:
- Dired 是什么?
- 怎么标记文件?
- WDired 怎么用?

### 5.4 跨手册搜: `C-h S`

`C-h S save-buffer RET`: 在**所有**手册里搜 `save-buffer`,跳到最相关的。

通常跳到 Elisp manual 里讲 save-buffer 的部分。

---

## 6. 培养习惯: "Always Ask Emacs"

学会 help system 命令不难,难的是**养成习惯**。

### 6.1 反模式: 先 Google

新手遇到问题,本能反应:
- 打开浏览器
- Google
- 看几个结果

这很慢,而且结果可能不准。

### 6.2 正模式: 先问 Emacs

训练自己:
- 看到任何 symbol → `C-h f` 或 `C-h v` 或 `C-h o`
- 看到任何键 → `C-h k`
- 想找命令 → `C-h a`
- 想看手册 → `C-h r` 或 `C-h i`

只有当 help system **找不到**时,才去 Google。

### 6.3 实际比例

我自己的统计:
- 90% 的问题用 `C-h` 解决 (变量、函数、键位)
- 5% 用 Info 解决 (概念的详细解释)
- 5% 才需要 Google (其他人的经验,新包的使用)

训练 1 个月,你的比例会接近这个。

### 6.4 帮助机制: 当 Emacs 报错

新手经常看到 error 就 `C-g` 跳过。**不要这样**。

错误信息有信息:
- `void-variable foo` → 你引用了不存在的变量,可能拼错
- `void-function bar` → 调用不存在的函数
- `wrong-number-of-arguments` → 参数数不对
- `wrong-type-argument listp 5` → 期望 list,给了 integer

读这些信息,理解,再 `C-h` 查相关函数。

Emacs 还提供 backtrace: 出错时按 `M-x toggle-debug-on-error RET`,下次出错会显示完整调用栈。

### 6.5 Help System 的创造性应用

学完基本命令,试试这些高级工作流:

1. **解构别人的 init.el**: clone 一个高 star 的 emacs config,逐行 `C-h f` / `C-h v` 验证你理解。这是学习最佳实践的捷径。
2. **`C-h l` 取证**: 你做了一串操作出问题,`C-h l` 看最后 300 个键,定位错误。
3. **`C-h e` 看消息历史**: minibuffer 闪过的消息没看清? `C-h e` 看 `*Messages*` buffer 的完整历史。
4. **`C-h F` vs `C-h f`**: `C-h f` 给技术性 docstring,`C-h F` 给教程式解释。新手先用 `C-h F`,理解后再用 `C-h f` 查细节。
5. **`C-h S` 跨手册搜**: 想知道某 symbol 在哪个手册里被讨论,`C-h S` 一次搜所有 Info 手册。
6. **`shortdoc` 学函数族**: `M-x shortdoc-display-group RET strings RET` 显示所有字符串函数的概览,带例子。这是学 Elisp 函数的好工具。
7. **`apropos-documentation` 找功能**: 你想做某事但不知道命令名,`C-h d 关键词` 在所有 docstring 里搜,找到能干这事的命令。
8. **`describe-char` 看字符详情**: 光标在一个字符上,`C-u C-x =` 显示这个字符的详细信息 (编码、字体、syntax class)。debug 中文显示问题时有用。
9. **`info-index` 精确查**: 在 Info 里 `i symbol RET` 用索引查 symbol,比全文搜索精确。
10. **`view-emacs-news` 跟版本**: `C-h C-n` 看你 Emacs 版本的 NEWS,知道新功能。每次升级 Emacs 都该读。

---

## 7. 微练习 (25 题,80 分钟)

每个题先自己想答案,再用 `C-h` 验证。

### describe-function (5 题)

1. `C-h f save-buffer RET`,看它有几个参数
2. `C-h f find-file RET`,看它是不是 interactive
3. `C-h f + RET`,看 `+` 是函数吗 (是的)
4. `C-h f lambda RET`,看 docstring,理解 lambda 特殊形式
5. `C-h f my-non-existent-func RET`,看找不到的提示

### describe-variable (5 题)

6. `C-h v kill-ring RET`,看当前 kill-ring 内容
7. `C-h v init-file-user RET`,看你的 init 文件路径
8. `C-h v load-path RET`,看 load path
9. `C-h v fill-column RET`,看默认 fill-column
10. `C-h v mode-line-format RET`,看 mode-line 结构 (复杂!)

### describe-key (5 题)

11. `C-h k C-g`,看 C-g 跑什么
12. `C-h k C-x C-s`,看保存命令
13. `C-h k M-x`,看 execute-extended-command
14. `C-h k C-h k`,看 C-h k 自己
15. `C-h k <f1>`,看 f1 干啥 (通常是 help prefix)

### where-is (3 题)

16. `C-h w save-buffer RET`,看 save-buffer 绑哪些键
17. `C-h w keyboard-quit RET`,看 C-g 绑在哪
18. `C-h w nonexistent-command RET`,看 "not on any key"

### describe-mode (3 题)

19. 在 `*scratch*` 里 `C-h m`,看 lisp-interaction-mode
20. 在任意代码文件 `C-h m`,看 major mode 的键位
21. 看 minor mode 列表,确认有哪些是 active

### apropos (2 题)

22. `C-h a buffer RET`,浏览所有 buffer 命令
23. `C-h d file RET`,在 docstring 里搜 file

### Info (2 题)

24. `C-h i m emacs RET m Dired RET`,读 Dired 章节
25. `C-h i m elisp RET i format RET`,查 Elisp 的 format 函数

---

## 8. 实战练习 (30 分钟)

### 任务 1: 解构别人的 init.el

找一个别人的 init.el (GitHub 上搜 "emacs config",随便点一个)。

对每一行用 `C-h` 查:

```elisp
(tool-bar-mode -1)
;; 用 C-h f tool-bar-mode RET,看它的作用

(setq inhibit-startup-screen t)
;; 用 C-h v inhibit-startup-screen RET

(global-set-key (kbd "C-c d") 'my-func)
;; 用 C-h f global-set-key RET
;; 用 C-h k C-c d RET (如果 eval 过)
```

挑 10 行,每行都用 `C-h` 验证你理解。

### 任务 2: 解决一个真实问题

问题: 你想让 Emacs 在保存 `.py` 文件时自动跑 `black` (Python formatter)。

不要 Google。用 help system:

1. `C-h a hook RET` 搜 hook 相关
2. 看到 `before-save-hook`? 查 `C-h v before-save-hook RET`
3. 看 docstring,知道怎么用
4. 想知道怎么调用 shell 命令? `C-h a shell RET`
5. 看到 `shell-command`? 查 `C-h f shell-command RET`
6. 综合写出:
   ```elisp
   (add-hook 'before-save-hook
             (lambda ()
               (when (eq major-mode 'python-mode)
                 (shell-command (format "black %s" buffer-file-name)))))
               ;; 注意: 实际用 black 包更好,这只是 help-driven learning
   ```

### 任务 3: 读 Emacs News

`C-h C-n` (view-emacs-news)。

显示你装的 Emacs 版本的 NEWS。这是"这个版本加了什么"。

读完 Module 1 的你需要能看懂大部分条目。如果有看不懂的,用 `C-h` 查。

### 任务 4: 探索一个 mode

打开一个 markdown 文件 (如果没有,创建一个 `test.md`,输入一些 markdown)。

```
M-x find-file RET ~/test.md RET
```

输入:

```
# Title

Some **bold** and *italic* text.

- item 1
- item 2
```

`markdown-mode` 应该自动启用 (Emacs 30 内置或装 markdown-mode 包)。

`C-h m` 看 markdown-mode 的所有键位。

试 5 个键 (例如 `C-c C-s b` 加粗,`C-c C-l` 插入链接,等等)。

---

## 9. 高级技巧

### 9.1 shortdoc

Emacs 28+ 有 `shortdoc` 集合: 把相关函数的快速参考放在一个文档里。

```
M-x shortdoc-display-group RET strings RET
```

显示 string 相关函数的概览 (with 例子)。

groups: `strings`, `numbers`, `sequences`, `arrays`, `lists`, `files`, `buffer`, `processes`, ...

**这是学 Elisp 函数的好工具**。

### 9.2 elisp-demos

装 `elisp-demos` 包后,`C-h f` 会显示**用法示例**。

```bash
M-x package-install RET elisp-demos RET
```

然后 `C-h f mapcar RET`,会看到示例:

```elisp
(mapcar #'1+ '(1 2 3))   ;; => (2 3 4)
```

### 9.3 helpful 包

`helpful` 是 `C-h f`/`C-h v` 的增强版,显示:
- 源码 (完整,不只位置)
- 调用历史
- 引用列表

```
M-x package-install RET helpful RET
```

绑定:
```elisp
(global-set-key (kbd "C-h f") #'helpful-callable)
(global-set-key (kbd "C-h v") #'helpful-variable)
(global-set-key (kbd "C-h k") #'helpful-key)
```

### 9.4 devhelp / dash (外部)

Linux 下 `devhelp` 是 GNOME 的 API 文档浏览器。
macOS 下 `Dash` 是付费但很好的工具。

但这些是外部,Emacs 自己的 help 已经够用 90%。

### 9.5 self-help 包

`M-x self-help-...` 系列,教程式问答。

---

## 10. 反模式 (要避免的)

### 10.1 不读 error message

新手看到 Lisp error 就按 `C-g`。
**慢一点,读一下**。

```
Error: (void-variable foo)
```

意思: 你引用了一个不存在的变量 `foo`。
检查拼写,或检查 `(boundp 'foo)`。

### 10.2 不看 *Messages*

很多动作的反馈在 `*Messages*`。
`C-h e` 看历史。

### 10.3 Google 先

记住: **先问 Emacs**。
help system 90% 能解决。

### 10.4 不更新知识

Emacs 每个版本加新功能。
`C-h C-n` (NEWS) 定期读,保持更新。

---

## 11. 自测 (10 题)

1. `C-h f` 和 `C-h F` 区别?
2. `C-h k C-h k` 会发生什么?
3. `C-h o` 和 `C-h f`/`C-h v` 的关系?
4. 怎么看你最后按的 100 个键?
5. 怎么看 Emacs manual 的某一节?
6. 怎么在所有手册里搜 symbol?
7. 怎么看当前 mode 的键位?
8. `*Messages*` buffer 怎么查看?
9. `before-save-hook` 是什么类型? 当前值是? (用 C-h v 查)
10. 怎么用 help system 学一个新 mode?

**答案** (反白):
> 1. C-h f 显示 docstring;C-h F 跳到 Info 手册里讲这个命令的部分
> 2. 第一次 C-h k 进入 describe-key 等你按一个键;你按 C-h k,所以它显示 describe-key 自己的文档
> 3. C-h o 自动判断 symbol 是函数还是变量;C-h f 只查函数,C-h v 只查变量
> 4. C-h l (view-lossage)
> 5. C-h r 然后 m 章节名 RET,或 g 节点名 RET,或 i 关键字 RET
> 6. C-h S (info-lookup-symbol)
> 7. C-h m (describe-mode)
> 8. C-h e (view-echo-area-messages)
> 9. Hook (一个变量,值是函数列表);当前值用 C-h v before-save-hook RET 看
> 10. 打开新 mode 的 buffer,C-h m,看它提供什么键位和命令

---

## 12. 毕业检查

- [ ] 25 题至少完成 20
- [ ] 能用 `C-h` 系列 90% 的时候不 Google
- [ ] 能熟练用 Info 浏览手册
- [ ] 看到任何 symbol 都能 5 秒内查到文档

完成后进入 `capstone.md`,做 Module 1 毕业项目。

---

## 13. 终极反思

学完这个 drill,你应该意识到:

**Emacs 的 help system 不是一个工具,是一种世界观**。

它传达的信息是: **你的工具不应该有秘密**。

普通软件是黑盒,你不知道它在干嘛,出问题就 Google。
Emacs 是透明盒,所有逻辑、所有变量、所有键位,你都能查到。

这种透明性,是自由软件运动的核心。

每当你 `C-h v` 看到变量当前值,你看到的是**你电脑上 Emacs 进程的真实状态**。
不是文档,不是截图,是 live 的真相。

掌握 help system = 掌握 Emacs 的"自我学习"能力。
这是为什么 Module 1 把它列为最重要。

后面 5 个月,你每天都会用几十次 `C-h`。形成肌肉记忆,你就独立了。

不再依赖教程,不再依赖博客,不再依赖视频。

**你成为 Emacs 的主人**。
