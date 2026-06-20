# Module 0: 心法与装备

> **目标**: 建立 Emacs 心智模型,从源码编译安装,写出最小可用的 `init.el`
> **时长**: 2 天 (~6 小时)
> **难度**: ★☆☆☆☆ (但概念密度高)
> **依赖**: 无 (这是起点)

---

## 0. 这个模块在学什么

在你写一行配置、敲一行 Elisp 之前,你必须先回答两个问题:

1. **Emacs 到底是什么?** (不是"一个编辑器")
2. **为什么它的设计与众不同?** (为什么用 Lisp? 为什么所有 buffer?)

如果你跳过这一步,直接去学键位,你会:
- 觉得键位"反人类" (因为你不知道 C-/M- 前缀的来历)
- 觉得"配置即代码"很奇怪 (因为你不知道 Emacs 是 Lisp 机器)
- 半途放弃 (因为你不知道自己在学什么)

这个模块不讲"怎么用 Emacs",讲"**Emacs 是什么以及为什么**"。
然后带你装好一个真·Geek 版的 Emacs,写出第一行自己的配置。

---

## 0.5 在开始之前: 为什么"心法"放在 Module 0

你可能在其他地方见过这样的教程: 第 1 天讲安装,第 2 天讲键位,第 3 天讲插件。那种教程的隐含假设是"你只要会用了,自然就懂了"。**这个假设在 Emacs 上是错的。**

Emacs 不是"会按键位"的工具。它是一种**思考方式**: 把文本当作数据,把编辑当作函数调用,把配置当作程序。如果你按"模仿 VS Code"的方式学,你会觉得它"反人类",因为它的设计目的就不是模仿 VS Code。

把心法放 Module 0 是**倒过来学**: 先建立"Emacs 的设计哲学是什么",再去看键位。键位是哲学的**投影**,看懂哲学后键位就不再是死记硬背的咒语,而是有逻辑的推理结果。

举个具体例子: 你看到 `C-x C-f` 打开文件时,如果只把它当成"快捷键",你会觉得"为什么不是 Ctrl+O? 为什么这么麻烦?"。但如果你知道:

- Emacs 是 Lisp 机器,每个键位调用一个函数
- `C-x` 是"全局前缀",`C-f` 是 `find-file` 的助记 (f = find)
- `find-file` 是一个 Lisp 函数,可以重定义、可以 hook

那么 `C-x C-f` 不再是"古怪快捷键",而是一个**透明的、可解释的、可改造的函数调用**。这就是"心法"的力量。

---

## 1. 第一性原理: Emacs 的五条核心心智模型

这五条会贯穿你后面 6 个月的学习。每一条都先建立直觉,不要求记定义。

### 1.1 Emacs 是 Lisp 机器伪装成编辑器

**直觉建立**: 打开一个普通编辑器 (VS Code、Sublime、Notepad++),你看到的是一个"文本编辑程序",
它**有**一些插件能力。但本质上,它是一个编辑器。

打开 Emacs,你看到的好像是编辑器。但真相是:

```
   你看到的:        Emacs 实际是:
   ┌──────────┐     ┌──────────────────────┐
   │ 文本编辑器 │     │  Lisp 解释器          │
   │           │     │  + 一堆文本原语函数    │
   │           │     │  + 一堆 buffer/window │
   │           │     │  + 一个事件循环        │
   └──────────┘     └──────────────────────┘
```

**关键证据** (你现在就可以验证):

打开 Emacs,按 `C-x b RET` (切换到 `*scratch*` buffer,默认就是它)。
然后在某个空行输入:

```elisp
(+ 1 2 3 4)
```

把光标放在这一行末尾,按 `C-x C-e` (eval-last-sexp)。

你会在 minibuffer 看到 `7`。

你刚才做了什么? **在编辑器里运行了一段代码,代码的结果立即显示出来**。
这不是编辑器的"插件能力",这就是 Emacs 的本质——一个**交互式的 Lisp 环境**。

VS Code 也能跑 JS,但那是"编辑器+插件"。Emacs 是反过来: **Lisp 解释器是底座,编辑器是它的一个应用**。

**为什么这重要**:
- 你学的不只是"按键做什么",而是"这些按键背后是什么函数"
- 任何你看到的行为,都可以用 Elisp 重定义
- 配置文件不是"配置",是**代码**

**推论**: 学 Emacs 等于学 Elisp。键位是表层,Elisp 是底层。

#### 深入: 为什么 Stallman 不直接用 C 写所有命令?

这是 1985 年 Stallman 必须回答的设计问题。考虑两条路:

**路 A (全 C)**: 每个编辑命令都写成 C 函数,编译进二进制。
- 优点: 性能极致
- 缺点: 用户改任何行为都要重新编译整个 Emacs,无法 live 调试,无法让用户自己扩展

**路 B (C 内核 + Lisp 上层)**: C 只写最底层原语 (buffer 操作、字符处理、GC),所有"编辑命令"是 Lisp。
- 优点: 用户能改一切,无需重新编译,可以热重载,可以写宏
- 缺点: Lisp 解释器要嵌入,启动稍慢 (1985 年的"稍慢"是几秒,2026 年的"稍慢"是 0.1 秒,基本无感)

Stallman 选了路 B,这是 Emacs 50 年生命力的根本原因。**如果选 A,Emacs 早就像其他编辑器一样被时代淘汰了**——因为它没法适应用户的每一个独特工作流。

#### 反事实: 如果 Emacs 现在被重写成 JS/TS 会怎样?

VS Code 就是这条路。后果:
- 扩展用 JS 写,需要打包、编译、加载到 extension host 进程
- 扩展不能直接访问主进程,要经过 IPC (慢、API 受限)
- 改 VS Code 核心行为要 fork 整个项目,重新打包发布
- 你不能在编辑器里 `eval` 一段 JS 立即看到结果 (没有这样的入口)

Emacs 的设计**让每一个用户都是开发者**——这是 1985 年的"操作系统哲学"放到编辑器上。

---

### 1.2 一切皆 Buffer

**直觉建立**: 在普通编辑器里,"文件"、"终端输出"、"搜索结果"、"设置面板"是不同的东西。

在 Emacs 里,**所有这些都是 buffer**。

打开 Emacs 默认就有这几个 buffer:
- `*scratch*` — 一个 Lisp 交互 buffer (没有关联文件)
- `*Messages*` — Emacs 给你的所有消息 (是一个 buffer!)
- `*Buffer List*` — buffer 列表 (本身也是 buffer)
- `*Completions*` — 补全候选 (也是 buffer)

然后:
- 你打开 `foo.txt` → 一个名叫 `foo.txt` 的 buffer
- 你 `M-x dired` → 一个名叫 `~/` 的 buffer (展示目录内容)
- 你 `M-x shell` → 一个名叫 `*shell*` 的 buffer (展示 shell 输出)
- 你 `M-x help` → 名叫 `*Help*` 的 buffer

**Buffer 的定义** (技术层面):

```
一个 buffer = {
  文本: mutable string,
  point: 当前光标位置 (integer),
  mark: 标记位置 (integer),
  major-mode: 主模式 (symbol),
  minor-modes: 次模式列表,
  text-properties: 文本的属性层,
  buffer-local-variables: 这个 buffer 独有的变量值,
  ...更多元数据
}
```

**为什么这重要**:
- 你学一套操作 (movement, kill, search),可以应用到**所有 buffer**
- 不需要为"文件"学一套,为"shell"学一套,为"目录"学一套
- Org 文件、邮件、聊天、IRC...都用同一套编辑原语

**推论**: 学 Emacs = 学 buffer 操作。后面所有的 drills 都在教你 buffer 的不同操作。

#### 第一性原理推导: 为什么是 buffer,而不是"打开的文件"

想象你在 1976 年要设计一个文本编辑器。你面对的根本问题是:

- 用户要在屏幕上看文本 (这是**显示**)
- 用户的文本存在某处 (这是**数据**)
- 用户做过的剪切复制,临时存放 (这是**剪贴板**)
- 用户在命令行输入命令 (这是**输入**)

普通编辑器的做法: 每种东西单独建模。"文件"有文件对象,"剪贴板"是一个全局变量,"终端"是另一个子系统。结果是: 这四种东西操作各不相同,代码不能复用,扩展要写四套。

Emacs 的做法: **统一抽象**——把上面这四种东西都当作"文本容器",叫 buffer。
- 显示问题 → window 显示 buffer
- 文件问题 → buffer 关联一个 file
- 剪贴板问题 → `*scratch*` 之类的临时 buffer
- 输入问题 → minibuffer (本质也是 buffer)

**推论 1**: 所有"编辑命令" (移动、复制、删除、搜索) 只实现一次,所有 buffer 都能用。这就是为什么你学 `C-s` (搜索) 时,它能在文件 buffer、shell buffer、dired buffer、Info buffer 里都工作。

**推论 2**: 一个 buffer 可以被多个 window 同时显示。改一处,所有 window 同步更新——因为它们看的是**同一份数据**。

**推论 3**: buffer 之间可以共享状态 (比如 kill-ring 全局共享),这是 buffer 抽象的"副作用福利"。

#### 创造性应用: buffer 的"啊哈时刻"

下面这些用法,在你只把 buffer 当"打开的文件"时是想不到的。**每一个都是真实的 Emacs 功能,你现在就能试。**

**啊哈 1: `M-x occur` 把搜索结果变成新 buffer**

在某个文件里 `M-x occur RET TODO RET`,Emacs 会创建一个新 buffer,列出所有匹配 "TODO" 的行,每行带行号。在 occur buffer 里按 `e` 进入 `occur-edit-mode`,你**修改 occur buffer 中的某一行**,改动会自动**写回到原文件对应位置**。

这是 buffer 抽象的威力: 搜索结果不是"列表 UI",是**可编辑的文本视图**。

**啊哈 2: 间接 buffer (indirect buffer),同一文件两个视图**

`M-x clone-indirect-buffer` 创建当前 buffer 的"双胞胎"。两个 buffer 共享同一份文本数据,但可以有**不同的 narrow** (一个只显示函数 1,一个显示函数 2),可以有**不同的 major mode** (一段在 org-mode,一段在 markdown-mode)。

这在普通编辑器里**做不到**——一个文件只能有一个视图状态。

**啊哈 3: `M-x wdired` 把目录变成 buffer 来改文件名**

`M-x dired` 打开目录后,`M-x wdired-change-to-wdired-mode` 进入 wdired 模式。现在你可以**像改文本一样批量改文件名**: 选中一段文件名用 `query-replace`,改完 `C-c C-c` 提交。所有改动应用为 rename 操作。

**啊哈 4: 任何进程输出都可以是 buffer**

`M-x shell` 创建一个 buffer,shell 的 stdout 持续追加进去。你可以在 shell buffer 里 `C-s` 搜索过去的输出 (即使已经滚出屏幕)。`M-x compile` 创建 `*Compilation*` buffer,你可以在错误行上按回车跳到对应代码。

对比 VS Code: 终端输出是一个 UI 组件,不是 buffer,搜索需要专门的"终端搜索"功能。

---

### 1.3 自文档化 (Self-Documenting)

**直觉建立**: 在普通软件里,如果你想查"这个按钮干什么",你要去查文档、Google、StackOverflow。
文档与软件是**分离的**。

Emacs 不同: 文档与软件**同源**。Emacs 自己描述自己。

**实际操作** (你一定要现在试):

```
按 C-h k   然后按 C-g
```

(minibuffer 会显示 `C-g` 跑的函数 `keyboard-quit`,以及它的文档)

```
按 C-h f   然后输入 kill-line RET
```

(会显示 `kill-line` 函数的完整文档,以及它的源码链接)

```
按 C-h v   然后输入 kill-ring RET
```

(会显示 `kill-ring` 变量的文档和**当前值**!)

```
按 C-h m
```

(会显示当前 major mode 和所有 active minor modes,以及它们的 keymap)

**关键**: 这些文档不是某个作家写的,是从**源码里的 docstring**自动生成的。
源码变了,文档立即变。绝不会脱节。

**为什么这重要**:
- 你不需要 Google,Emacs 自己知道
- 你看任何人的配置,不懂的 `C-h v` 一查就有
- 这是你后续 6 个月的"超能力"

**推论**: 学 Emacs 第一步是**学 help system**。Module 1 的 Drill 07 是最重要的 drill。

#### 深入: 为什么"自文档化"是 Emacs 的独特设计

考虑普通软件的文档问题:
- 文档在网站 (ReadTheDocs、wiki),由人工维护
- 软件版本 A.0 写了文档 A.0
- 几个月后代码改成 A.1,文档忘记同步
- 用户看到 A.1 的行为但查 A.0 的文档,产生困惑

VS Code 的 GitHub wiki 经常有"过时"的页面。Sublime Text 文档说"按 Ctrl+Shift+P",但实际可能是 Cmd+Shift+P。文档和代码之间的"漂移"是所有软件的通病。

Emacs 的解法是**让代码和文档同源**。每个函数、变量都有 docstring,写在源码里紧贴定义。`C-h f` 等命令是从源码读取 docstring 并展示,不经过任何中间层。

**结果**:
- 代码改了,docstring 改了,文档立即变 (因为本来就是同一个东西)
- 你装的包,help 也能查 (包自己带 docstring)
- 没有"文档落后于代码"的问题

#### 反事实: 如果 Emacs 像 VS Code 那样做文档会怎样?

如果 Emacs 把所有文档放 emacs.readthedocs.io:
- 你按 `C-h f find-file RET` 看到的不再是"实际行为",而是"某次写下的描述"
- 你装的某个包升级了,文档没同步,help 看到的还是旧描述
- 你无法离线查文档 (除非下载整个 docs)

这种"软件"和"文档"的解耦,看似分离关注点,实际制造了**信息不一致的恒久问题**。Emacs 的选择是把它们焊死在一起,用"代码即文档"换取一致性。

#### 历史: 这是 Lisp 文化的延伸

Lisp 1.5 (1960 年)就有 docstring 概念。Smalltalk、Python 后续也学了。但 Emacs 把它推到极致——**所有**用户可见的符号 (函数、变量、面、键位) 都必须有 docstring,否则 `checkdoc` 会警告。这种"文档纪律"是 Emacs 文化的核心。

---

### 1.4 Live System (热重载)

**直觉建立**: 在普通程序里,改代码要"重新编译、重启"。
在 Emacs 里,**改代码立即生效**。

**实际操作**:

在 `*scratch*` 输入:

```elisp
(defun my-hello ()
  (message "Hello, world!"))
```

光标放末尾,`C-x C-e` (定义函数)。

然后 `M-x my-hello RET`。minibuffer 显示 `Hello, world!`。

现在改它:

```elisp
(defun my-hello ()
  (message "Goodbye, world!"))
```

再 `C-x C-e`,再 `M-x my-hello RET`。显示 `Goodbye, world!`。

**没有重启,没有编译,立即生效**。

**为什么这重要**:
- 配置 init.el 后,你可以 `M-x eval-buffer` 整个重新加载,不用重启 Emacs
- 你调试一个函数,可以一边改一边试,迭代飞快
- 这是为什么 Emacs 用户"住在 Emacs 里"——他们的整个工作环境是 live 的

**推论**: 不存在"配置改完要重启"。一切是动态的。

#### 第一性原理: 为什么能做到 Live?

考虑"重启"为什么存在:
- 编译型语言 (C/Java): 代码改动要重新编译成二进制,旧的进程加载的是旧二进制,必须杀掉重启
- 解释型但有 GC 隔离 (Python): 可以热改一部分,但 module 加载后有缓存,有些改动要 reload module
- 完全动态 (Lisp Smalltalk Erlang): 程序本身是一个**可修改的数据结构**,改了立即生效

Emacs 是第三种。Emacs 进程的"状态"是一棵 Lisp 对象的树 (符号表、函数定义、变量值、buffer 内容)。**这棵树本身可以被 Lisp 代码读写**——`defun` 就是在树上"替换某个节点的函数定义",替换后下次调用就执行新定义。

这是 Lisp homoiconicity (代码即数据) 的工程后果。如果你熟悉 Smalltalk 的 image-based 系统,Emacs 是同一类东西。

#### 对比: 配置文件改完要重启的痛苦

考虑 VS Code 用户的工作流:
1. 改 `settings.json`
2. 命令面板 `Reload Window`
3. 等 3-5 秒,所有状态丢失 (打开的文件、未保存编辑、终端历史)
4. 改 `keybindings.json`,又 reload
5. 调试一个插件,每次改要重新 launch extension host

Emacs 用户的工作流:
1. 改 init.el 的某一行
2. `C-x C-e` 在那行末尾,立即生效
3. 不重启,所有 buffer 状态保留
4. 调试一个函数,边写边 eval,迭代飞快

这就是为什么 Emacs 用户"住在 Emacs 里"——他们的环境**永远热着**,从开机到关机,所有上下文连续。

#### 创造性应用: Live 系统的"啊哈时刻"

**啊哈 1: 在编辑过程中"试一个想法"**

你写代码时突然想到"如果函数 foo 也接受 nil 呢?",你不必切到 REPL、不必重启、不必写测试。直接在 `*scratch*` 里写 `(defun foo (x) ...)`,eval,然后回到代码 buffer 试。

**啊哈 2: 修复一个生产中的 Emacs bug 而不退出**

Emacs 卡住了某个操作 (比如某个 hook 死循环)。你不必杀进程——`C-g` 中断,然后用 `M-x edebug` 或直接重定义那个 hook 函数。**整个编辑环境在你修复 bug 时一直可用**。

**啊哈 3: 配置演化是连续的,不是离散的**

你的 init.el 不是"一锤定音"的配置。它是一个**活着**的 Elisp 程序,你在每天的编辑过程中持续小修小补,`eval-buffer` 或 `eval-defun` 立即应用。一个月后回看,你的 Emacs 已经演化成了一个完全为你定制的工作环境,但你从未"重启过它"。

---

### 1.5 可组合性 (Composability)

**直觉建立**: 在 VS Code 里,一个插件是一个隔离的盒子,插件之间一般不能互相调用。

在 Emacs 里,**所有 Elisp 函数都在同一个命名空间**,任何函数都能调用任何函数。

Magit (Git 客户端) 用了 `magit-status`,它内部调 `magit-refresh`,它内部调 `magit-insert-status-headers`...这些函数你**都能调用**。

你可以写自己的 Elisp 函数,组合 Magit 的函数,做出 Magit 作者没想到的功能。

**实际操作** (现在不要做,只是看):

```elisp
(defun my-magit-commit-with-co-author (co-author)
  "Commit with a Co-authored-by trailer."
  (interactive "sCo-author: ")
  (let ((magit-commit-arguments
         (list (format "--trailer=Co-authored-by: %s" co-author))))
    (call-interactively #'magit-commit)))
```

这个函数"扩展"了 Magit,但是用 Magit 自己的变量 (`magit-commit-arguments`)。
没有 API 边界,因为根本没有"API",所有东西都是函数和变量。

**为什么这重要**:
- 你"精通 Emacs"意味着你能组合任何包做任何事
- 不存在"等作者加这个功能"——你自己加
- 这是 Emacs 50 年生态的根本设计

**推论**: 不要等"插件支持 X",自己写。

#### 深入: 为什么"无 API 边界"是 Emacs 设计选择

VS Code 的扩展运行在 extension host 进程,有严格的 API 边界: `vscode.window.showInformationMessage(...)` 这种"白名单"调用。后果:
- 扩展作者想做的事,API 不支持,就**做不到** (比如读 VS Code 内部状态)
- 扩展之间不能直接调用对方的函数,只能通过 vscode.commands 注册的命令
- 任何"超出 API 范围"的需求,要等 VS Code 团队加 API

Emacs 反过来: **所有函数都在同一个全局命名空间**。Magit 的 `magit-status` 函数,你直接 `(call-interactively #'magit-status)` 就能调。Magit 内部的私有函数 `magit-insert-status-headers` 你也能直接调 (虽然名字带 magit- 前缀是"约定私有",但**技术上没人能阻止你**)。

这种"无边界"听起来危险,实际很安全——因为用户都是成年人,且改坏了用 `C-h f` 就能查清楚。

#### 历史: Unix 哲学的延伸

Emacs 可组合性是 Unix 哲学的极致表现:
- Unix: 小工具 + 管道
- Emacs: 小函数 + 嵌套调用

但 Emacs 比 Unix 更进一步: Unix 管道只能传字节流,Emacs 函数能传任何 Lisp 对象 (list、hash table、buffer、window)。这是为什么 Emacs 比 shell 脚本更强大。

#### 创造性应用: 组合的"啊哈时刻"

**啊哈 1: 用 Org + Lisp 写一个轻量 GTD**

不需要装任何包。org-mode 已内置,你可以写 20 行 Elisp:
- `C-c a` 调 agenda (org-mode 内置)
- 自定义 agenda 命令: `(setq org-agenda-custom-commands ...)` 过滤你的 todo
- 加 hook: `(add-hook 'org-after-todo-state-change-hook 'my-notify)` 在 todo 完成时弹通知

这是组合的威力: org 是文本编辑,magit 是 git 客户端,projectile 是项目管理——它们都是函数,你随意编排。

**啊哈 2: "事实编程"——一个需求 5 分钟搞定**

假设你想"保存任何文件时,如果文件名以 .log 结尾,自动 git commit"。整个实现:

```elisp
(defun my-auto-commit-logs ()
  (when (string-suffix-p ".log" (buffer-file-name))
    (let ((default-directory (file-name-directory (buffer-file-name))))
      (shell-command "git add . && git commit -m 'auto: log update'"))))

(add-hook 'after-save-hook 'my-auto-commit-logs)
```

5 行 Elisp,工作。在 VS Code 里同样功能要写完整扩展 (package.json、activationEvents、main entry、API call),发布到市场,用户安装。**Emacs 让"小需求"也有低成本的解**。

---

## 2. Emacs 的历史 (背景故事)

> 这一节替代 `emacs-manual-30.2/gnu.texi` 的内容。看完不需要去翻原文。

### 2.1 起源: TECO 和 ITS (1970s)

时间回到 1975 年的 MIT AI Lab。程序员们用一台 PDP-10 主机,跑 ITS 操作系统。
他们用的编辑器叫 **TECO** (Text Editor and Corrector)。

TECO 是个奇葩: 它实际上是一个**可编程的语言**,默认用来编辑文本。
程序员们写了大量 TECO 宏 (macro),让 TECO 做各种自动化的事。

其中两个程序员——**Richard Stallman** 和 **Guy Steele**——收集了大家最常用的 TECO 宏,
统一成一套,叫做 "EMACS" (Editor MACroS)。这是 1976 年。

**关键认知**: Emacs 一开始就是"一堆宏的集合",不是一个"编辑器"。
它的精神是: **用户自己改造工具**。

### 2.2 GNU Emacs (1985)

1981 年,Stallman 离开 MIT 创立 GNU 项目。他意识到 TECO 的限制 (语法怪、性能差),
决定用 **Lisp** 重写 Emacs,这就是 1985 年的 GNU Emacs 1.0。

为什么选 Lisp?

- Lisp 是"代码即数据"的语言 (homoiconicity),非常适合做"宏"
- Lisp 的动态特性允许热重载
- Lisp 在 MIT AI Lab 是"母语"

GNU Emacs 用一种叫 **Emacs Lisp** 的方言。它是 **Lisp-2** (变量和函数有独立命名空间),
默认动态作用域 (Emacs 25+ 支持 lexical binding)。

### 2.3 XEmacs 分裂 (1991)

1991 年,Lucid 公司的 Jamie Zawinski 等人 fork 了 Emacs 19,叫 **XEmacs**。
原因主要是开发节奏和功能差异。这导致长达 20 年的 "Emacs vs XEmacs" 之争。

**结局**: 2015 年左右 XEmacs 维护停滞,GNU Emacs 胜出。
现在说 "Emacs" 默认指 GNU Emacs。但这段历史影响了大量 API 设计 (你看老包会看到 `(featurep 'xemacs)` 这种兼容代码)。

### 2.4 现代 Emacs (2020s)

- 2020: Emacs 27,引入 `early-init.el`、内置 `use-package` (部分)
- 2022: Emacs 28,引入 **native compilation** (libgccjit 把 Elisp 编译成原生码,性能飞起)
- 2023: Emacs 29,引入 `tree-sitter` 支持、`eglot` 内置
- 2024: Emacs 30,重大改进 (`use-package` 完全内置、pgtk 主线等)

你现在装的是 **30.2**,这是当前最新稳定版。

### 2.5 关键人物 (你要知道的名字)

- **Richard Stallman (RMS)**: 创始人,守门人。现在还在参与 emacs-devel 讨论
- **Guy Steele**: 共同作者,后来搞 Scheme 和 Java
- **Stefan Monnier**: 长期维护者,current co-maintainer
- **Eli Zaretskii: 现任主 maintainer,Windows 平台大佬
- **John Wiegley**: 前 maintainer,写了 `use-package`
- **Bozhidar Batsov`: Emacs Prelude 作者,社区布道者

---

## 3. Emacs 的架构 (三层 + 两个特殊 buffer)

> 这一节替代 `emacs-manual-30.2/screen.texi` 的内容。

### 3.1 五个核心对象

打开一个 Emacs (图形界面),你看到这些东西。每一个都对应一个 Elisp 概念:

```
┌─────────────────────────────────────────────────────────┐  ← Frame (物理窗口)
│ ┌───────────────────────────────────────────┐ ┌───────┐ │
│ │ Menu bar                                  │ │       │ │
│ ├───────────────────────────────────────────┤ │       │ │
│ │ Tool bar                                  │ │       │ │
│ ├───────────────────────────────────────────┤ │       │ │
│ │                                            │ │       │ │
│ │  Window 1                                  │ │ Wind. │ │  ← Windows (分屏)
│ │  (显示 Buffer A)                           │ │ 2     │ │
│ │                                            │ │(Buf   │ │
│ │                                            │ │  B)   │ │
│ │                                            │ │       │ │
│ │ -uu-:**- foo.txt   Top L1    (Text) ----  │ │       │ │  ← Mode line
│ ├───────────────────────────────────────────┤ │       │ │
│ │ M-x                                        │ │       │ │  ← Minibuffer
│ └───────────────────────────────────────────┘ └───────┘ │
│                                                           │
│                                              Echo area →  │
└─────────────────────────────────────────────────────────┘
```

**Frame** (`(selected-frame)`):
- 物理窗口 (GUI 下) 或整个终端屏幕 (TTY 下)
- 一个 Emacs 可以有多个 frame (`C-x 5 2` 新建)

**Window** (`(selected-window)`):
- Frame 内的分屏区域
- 每个 window 显示一个 buffer (一个 buffer 可以被多个 window 显示)
- `C-x 2` 上下分屏,`C-x 3` 左右分屏,`C-x 1` 只保留当前,`C-x 0` 关闭当前

**Buffer** (`(current-buffer)`):
- 文本的容器,与 window 解耦
- 一个 buffer 可以不被任何 window 显示 (后台 buffer)
- `C-x b` 切换 buffer,`C-x C-b` 列出所有 buffer

**Minibuffer**:
- 一个特殊的 buffer,在底部,承担"命令交互"
- 当你按 `M-x` 输入命令名,或按 `C-x C-f` 输入文件名,你在和 minibuffer 交互
- 它本质上就是一个 buffer,有 history、有补全

**Mode line**:
- 每个 window 底部的一行,显示当前 buffer 的状态
- 格式由 `mode-line-format` 变量控制
- 例: ` -UU-:**-F foo.txt   Top L1  (Fundamental) ---`

**Echo area**:
- 最底部的一行,显示 Emacs 给你的消息 (`message` 函数的输出)
- 也在你按 `M-x` 时变成 minibuffer

### 3.2 一个心理模型: "Buffer 是世界,Window 是窗"

想象你在一个大楼 (Frame) 里。
楼里有很多房间 (Buffer),每个房间里有文本。
你不能同时看到所有房间,所以你在墙上开窗 (Window)。
窗里能看到一个房间 (Buffer) 的某一部分。
你可以同时开多个窗 (split-window)。
房间一直存在,不管你开不开窗。
有些房间是"特殊房间"——比如前台 (Minibuffer),墙上贴的通告。

```
Buffer (房间):    [Buffer A] [Buffer B] [Buffer C] [Buffer D]...
                       ↓         ↓
Window (窗):     [窗1: A]    [窗2: B]
                  ↑          ↑
Frame (大楼):    ┌──────────────────┐
                 │                  │
                 │   [窗1] [窗2]    │
                 │                  │
                 │  Minibuffer___   │
                 └──────────────────┘
```

### 3.3 Mode 的概念

每个 buffer 有:

- **一个 major mode**: 决定这个 buffer 是"什么" (代码? 文本? 邮件? 图片?)
  - 例: `python-mode`、`org-mode`、`dired-mode`、`fundamental-mode` (默认)
  - 决定语法高亮、缩进规则、特殊键位
- **零到多个 minor modes**: 附加功能,可以叠加
  - 例: `flyspell-mode` (拼写检查)、`auto-fill-mode` (自动换行)、`linum-mode` (行号)
  - 不冲突,可以同时开多个

### 3.4 Point 和 Mark

每个 buffer 有两个特殊位置:

- **Point**: 当前光标位置。技术上是一个 integer (从 1 开始的字符偏移)
- **Mark**: 一个"记住"的位置。可以用 `C-SPC` 设置

`Point` 和 `Mark` 之间的文本叫 **region** (选区)。

**Mark ring**: 每次 `C-SPC C-SPC` (双击)会把当前 point 推入 mark ring。
`C-u C-SPC` 弹出,跳回上一个 mark。这是"光标历史栈"。

---

## 4. 键位系统的来历 (为什么是 C-/M-)

> 这一节替代 `emacs-manual-30.2/commands.texi` 的开头。

### 4.1 Space-cadet 键盘

1970s 的 MIT 用 **Space-cadet keyboard** (太空学员键盘)。它有 7 个修饰键:

```
[Control] [Meta] [Super] [Hyper] [Top] [Front] [Shift]
```

Emacs 设计时,Control 和 Meta 是真实存在的按键。
所以 `C-x` 是"按住 Control 再按 x",`M-x` 是"按住 Meta 再按 x"。

现代键盘没有 Meta 键。Emacs 默认映射:
- **Meta → Alt** (或 Option,在 macOS)
- **Super → Win** (或 Command,在 macOS)
- **Hyper → 没默认**

#### 深入: Space-cadet 键盘的物理事实

1970s 的 MIT AI Lab 用 Lisp Machine,键盘叫 **Space-cadet** (Tom Knight 设计)。它有 7 个修饰键 (Control、Meta、Super、Hyper、Top、Front、Greek),还有大量专用键 (Resume、Abort、Call、Clear Screen)。

这套键盘的"修饰键哲学"是: **不浪费一个修饰键给单个功能**。意思是——一个修饰键 + 一个普通键 = 一个 namespace,namespace 下可以有 26+ 个命令。这是把"键位空间"最大化利用的设计。

现代键盘 (101 键 PC) 退化到只有 Ctrl、Alt、Shift、Win 四个。Emacs 的 `M-x` 是历史遗产: 现代键盘没有 Meta,Emacs 把 Alt 当 Meta 用,但**保留了名字** (你 `C-h k` 看到 `M-x` 而不是 `A-x`)。

#### 反事实: 如果当年用现代键盘设计 Emacs 会怎样?

现代键盘只有 4 个修饰键,Emacs 会:
- `C-x` 还是前缀,因为 Control 在
- `M-x` 可能改成 `Win-x` (因为现代键盘 Win 等价 Meta)
- 但 Win 键在 Windows 上被系统劫持,在 macOS 是 Cmd——跨平台不一致

这是为什么大多数现代编辑器用 `Ctrl+Shift+P` 这种"双修饰键 + 字母"组合 (如 VS Code 的命令面板),而不是"修饰键 + 字母 + 字母"的 Emacs 风格。Emacs 的键位设计**确实是 1970s 的孩子**——这既是它的负担也是它的特色。

### 4.2 为什么是 C-x / C-c 这种"两步"前缀?

普通编辑器:`Ctrl+S` 是一个动作 (保存)。
Emacs:`C-x` 是**前缀**,后面要接一个字符才算完整命令。

为什么这么设计? **键位空间扩展**。

`C-x` 前缀下有 ~30 个命令 (`C-x C-f` find-file, `C-x C-s` save, `C-x b` switch-buffer...)
`C-c` 前缀下留给用户/模式自己定义
`C-h` 前缀下是所有 help 命令
`C-x r` 前缀下是 register/rectangle 命令
`C-x 4` 前缀下是"在其他 window 操作"
`C-x 5` 前缀下是"在其他 frame 操作"

```
单键命令:    a-z, A-Z, 数字, 符号 (~50 个)
C- 前缀:     C-a 到 C-z (~26 个)
M- 前缀:     M-a 到 M-z (~26 个)
C-x 前缀:    C-x a 到 C-x z + C-x C-a 到 C-x C-z (~50 个)
C-c 前缀:    同上 (~50 个)
C-h 前缀:    同上 (~50 个)
...
```

**总键位空间**: 远超 1000 个不用鼠标的命令。

#### 第一性原理推导: 键位空间是有限的资源

想象你是 1976 年的 Stallman,要给 Emacs 设计键位。约束是:
- 修饰键有 2 个真正能用的 (Control、Meta,Space-cadet 还有 Super/Hyper 但不方便)
- 普通键有 ~50 个 (字母、数字、标点、空格、回车等)
- 用户每天可能用的命令有 1000+ 个 (各种 buffer、window、文件、编辑、搜索、模式相关操作)

**单层键位** (直接按普通键): 50 个。这是 self-inserting 字符 (a, b, c...),已被占用。

**C- + 单键**: 50 个。但 `C-i` 在终端是 TAB,`C-m` 是 RET,`C-[` 是 ESC——所以真正能自由用的是 ~30 个。

**M- + 单键**: 50 个。但 M- 字母大多用于 Meta 字符输入 (M-a 在某些环境下是 ä)。

到这一步,你有 50 + 30 + 50 = **130 个命令空间**。对于"专业编辑器"远远不够 (Magit 一个包就要几十个键)。

**前缀键**是出路: `C-x` 后面再接一个键,等于"扩展了 50 个槽位"。`C-x` 加 `C-x C-` 双前缀,再加 `C-c`、`C-h`、`C-x 4`、`C-x r` 等多级前缀,总空间轻松突破 1000。

这是为什么 Emacs 用"两步前缀": **物理键有限,前缀是逻辑放大器**。

#### 对比: 其他编辑器怎么解决这个问题

**VS Code 的做法**: 加 Shift。`Ctrl+Shift+P`、`Ctrl+K Ctrl+S` (chord)。Chord 和 Emacs 前缀本质相同,但 VS Code 只有少数命令用 chord。

**Vim 的做法**: 模式。Normal 模式按 `d` 是删除,按 `y` 是 yank,按 `p` 是粘贴——通过模式切换,同一物理键在不同模式有不同含义。Vim 用"模式"扩展键位空间,Emacs 用"前缀"扩展。这是两个伟大编辑器的根本分野。

**Emacs 的取舍**: 拒绝模式 (默认不装 evil)。代价是要更多前缀 (`C-x`、`C-c`)。好处是: 任何时候所有键的语义一致,没有"忘记在哪个模式"的混乱。

#### 为什么是 C-x 而不是 M-x 当主前缀?

历史原因: 在 Space-cadet 键盘上 Control 和 Meta 都好按 (大拇指),但 Meta 在现代键盘映射到 Alt,小拇指够起来很别扭。Control (现代 Ctrl) 相对好按 (虽然在角落)。所以**最常用的扩展命令** (`C-x C-f`、`C-x C-s`) 放在 `C-x` 前缀,而**最不常用的"调用任意命令"** 放在 `M-x`。这是历史 + 人机工学的妥协。

### 4.3 著名的"Emacs 小拇指"

`Ctrl` 用得太频繁,小拇指容易疼。Geek 解决方案:

- **方案 A**: 把 CapsLock 改成 Ctrl (推荐,系统级改)
- **方案 B**: 用 HHKB 等键盘,Ctrl 在 QWERTY 的 CapsLock 位置
- **方案 C**: 双手轮流按 Control
- **方案 D**: evil-mode (Vim 键位模拟,大幅减少 Ctrl 使用)

本教程**先教你默认 Emacs 键位**,因为:
1. 不会"依赖 evil"才能用
2. 看任何 Emacs 教程都默认这套
3. Module 5 之后你可以自己决定要不要装 evil

### 4.4 Modifier 的符号

| 符号 | 名字 | 键盘映射 |
|---|---|---|
| `C-` | Control | Ctrl |
| `M-` | Meta | Alt (PC) / Option (Mac) / ESC (然后松开,再按下一个键) |
| `S-` | Shift | Shift |
| `s-` | Super | Win (PC) / Cmd (Mac) |
| `H-` | Hyper | 无默认,自己绑定 |
| `A-` | Alt | 历史遗留,等于 Meta |

**终端的特别说明**: TTY 下 `C-m` `C-i` 等会被终端解释成 TAB/RET,所以有些键在终端下用不了。
GUI Emacs 没这限制。

---

## 5. 修炼装备: 从源码编译 Emacs

> 这一节替代 `emacs-manual-30.2/cmdargs.texi` 的安装部分。

### 5.1 为什么要从源码编译

大多数 Linux 发行版有 `emacs` 包,直接 `apt install emacs` 或 `pacman -S emacs` 就行。
但 Geek 应该编译,原因:

1. **最新版本**: 发行版的 Emacs 通常落后 1-2 个版本
2. **编译选项**: 开启 `--with-native-compilation` 性能飞起
3. **理解系统**: 知道 Emacs 依赖什么,怎么构建
4. **调试能力**: 后面如果要给 Emacs core 提 patch,你需要源码

#### 深入: native compilation 改变了什么

Emacs 28 之前的 Elisp 是**解释执行**——每次调用函数,Lisp 解释器遍历 AST,逐个求值。对 IO 密集任务 (编辑文本、读文件) 这够用,但对 CPU 密集任务 (大文件搜索、复杂 regex、JSON 解析) 就慢。

Andrea Corallo 在 Emacs 27 引入的 **native compilation** (libgccjit) 把 Elisp 编译成机器码。原理:
1. Emacs 在后台把 .el 编译成 .eln (LLVM-like 字节码 + JIT)
2. 第一次调用某函数时,JIT 把它编成原生机器码
3. 后续调用直接执行机器码

实际效果: Magit 启动从 200ms 降到 50ms,LSP 响应更快,大文件编辑不卡。

**为什么不默认开?** 因为编译需要 libgccjit,这要求系统装了正确版本的 gcc。发行版为了不强制依赖,默认不开。所以**自己编译才能享受这个红利**。

#### 反事实: 用 apt 装的 Emacs 会怎样?

Debian/Ubuntu 的 emacs 包通常:
- 版本落后 (Ubuntu 24.04 装 Emacs 29,但 30 已发布)
- 没 native comp (避免依赖 gcc)
- 没 tree-sitter (Emacs 29+ 需要,但发行版可能没编进去)
- 字体渲染用 GTK 默认,不如 pgtk 流畅

后果: 你装完发现 Magit 启动慢、大文件卡——以为"Emacs 慢",其实是发行版打包的问题。**编译版 Emacs 性能完全不同**。

### 5.2 准备依赖 (Arch Linux 例子)

```bash
sudo pacman -S base-devel
sudo pacman -S --needed \
    git gtk3 giflib libxpm libjpeg-turbo libtiff jansson \
    gnutls harfbuzz tree-sitter
# native compilation 需要 gcc 和 libgccjit
sudo pacman -S libgccjit
```

(Ubuntu/Debian 的包名不同,但大致相同;macOS 用 `brew install ...`)

### 5.3 拉源码 + 配置 + 编译

```bash
# 假设你放在 ~/src
cd ~/src
git clone https://git.savannah.gnu.org/git/emacs.git
cd emacs
git checkout emacs-30.2  # 或 master

# 配置 (推荐选项,Wayland 用户用 --with-pgtk)
./autogen.sh
./configure \
    --with-native-compilation=aot \
    --with-tree-sitter \
    --with-x-toolkit=gtk3 \
    --with-pgtk \
    --with-mailutils \
    --with-imagemagick \
    --prefix=$HOME/.local/emacs-30 \
    CFLAGS="-O2 -pipe"

# 编译 (用全部 CPU 核心)
make -j$(nproc)

# 测试 (可选,~5 分钟)
make check

# 安装
make install
```

把 `$HOME/.local/emacs-30/bin` 加到 `PATH`。

### 5.4 验证

```bash
emacs --version
# 应该输出 GNU Emacs 30.2 等

emacs --batch --eval='(message "%d" (+ 1 2))'
# 应该输出 3,证明 native comp 在工作 (启动飞快)
```

### 5.5 第一次启动

```bash
emacs
# 或在 GUI 启动:
emacs &
```

你会看到欢迎屏 (splash screen)。

**立即做的第一件事**: 按 `C-h t` (help-with-tutorial)。
这是 Emacs **内置的官方教程**,~30 分钟读完,教你最基本的移动和编辑。

**不要跳过这个内置教程**。它是 Emacs 团队精心设计的入门,比任何博客都权威。

#### 深入: 为什么从 splash 进 tutorial

考虑普通软件的 onboarding:
- VS Code: 启动后显示"欢迎"页,有几张互动卡片
- Sublime: 显示一个示例文件让你随便按按
- IntelliJ: 弹一堆对话框让你配置

这些设计假设: 用户不知道从哪开始,需要"引导"。

Emacs 不一样: `C-h t` 直接带你进入 **tutorial buffer**——一个**真实可编辑的 buffer**,内容就是教程。教程一边讲键位,你一边按。按错了 `C-h k` 立即查。

这是 Emacs 自文档化哲学的体现: **不需要看视频,不需要看博客,软件自己教你**。

#### 反事实: 如果 Emacs 像普通软件那样做 onboarding

可能弹窗:"欢迎使用 Emacs! 按下一步继续。" 用户点 5 次 Next,然后什么都没学到。普通软件的"引导"通常是为不知道自己要做什么的用户设计的——而 Emacs 用户的目标是**深度使用**,引导必须是"动手做"。所以 `C-h t` 是 Emacs 50 年来不变的开机仪式。

---

## 6. 写出最小可用的 init.el

### 6.1 找到你的 init.el 位置

Emacs 启动时会自动找配置文件,优先级:

1. `~/.config/emacs/init.el` (XDG 标准,推荐)
2. `~/.emacs.d/init.el` (经典路径)
3. `~/.emacs` (老式,已不推荐)

确认你的:

```bash
ls -la ~/.config/emacs/init.el 2>/dev/null
ls -la ~/.emacs.d/init.el 2>/dev/null
```

如果都没有,创建 `~/.config/emacs/init.el`:

```bash
mkdir -p ~/.config/emacs
touch ~/.config/emacs/init.el
```

### 6.2 写入最小配置

打开 Emacs,`C-x C-f ~/.config/emacs/init.el RET`,输入:

```elisp
;;; init.el --- 最小可用配置 -*- lexical-binding: t; -*-

;;; 基础界面
(setq inhibit-startup-screen t)         ; 不显示欢迎屏
(setq initial-scratch-message nil)      ; scratch 不显示提示
(setq ring-bell-function 'ignore)        ; 关闭烦人的蜂鸣
(tool-bar-mode -1)                       ; 关闭工具栏 (GUI)
(scroll-bar-mode -1)                     ; 关闭滚动条
(blink-cursor-mode -1)                   ; 光标不闪烁

;;; 编辑体验
(setq make-backup-files nil)             ; 不要 ~ 备份文件 (后面 Module 2 改回)
(setq auto-save-default nil)             ; 不要 #auto-save# 文件
(setq create-lockfiles nil)              ; 不要 .#lock 文件
(delete-selection-mode 1)                ; 选中后输入直接替换
(electric-pair-mode 1)                   ; 自动配对括号

;;; 编码
(set-language-environment "UTF-8")
(set-default-coding-systems 'utf-8-unix)

;;; 显示
(global-display-line-numbers-mode 1)     ; 行号
(column-number-mode 1)                   ; mode line 显示列号
(size-indication-mode 1)                 ; mode line 显示文件大小

;;; 主题 (Emacs 28+ 内置 modus 主题,高质量)
(load-theme 'modus-vivendi t)            ; 深色
;; 或 (load-theme 'modus-operandi t)     ; 浅色

;;; 字体 (GUI 才有用)
(set-face-attribute 'default nil
                    :family "JetBrains Mono"
                    :height 140)

;;; 包管理 (Emacs 27+ 内置 package.el + use-package 部分功能)
(require 'package)
(setq package-archives
      '(("gnu"    . "https://elpa.gnu.org/packages/")
        ("nongnu" . "https://elpa.nongnu.org/nongnu/")
        ("melpa"  . "https://melpa.org/packages/")))
(package-initialize)
(unless package-archive-contents
  (package-refresh-contents))

;;; 绑定一些键
(global-set-key (kbd "<escape>") 'keyboard-escape-quit)  ; ESC 关闭提示
(global-set-key (kbd "C-c r") 'recentf-open-files)       ; 最近文件

;;; 最近文件
(recentf-mode 1)
(setq recentf-max-saved-items 100)

(provide 'init)
;;; init.el ends here
```

### 6.3 应用配置

`C-x C-s` (保存),然后:

- **方式 A** (推荐理解): `M-x eval-buffer RET` — 立即应用整个 buffer
- **方式 B**: 重启 Emacs

### 6.4 验证

启动后你应该看到:
- 没有 splash screen
- 没有工具栏、滚动条
- 行号显示
- modus-vivendi 深色主题
- 光标不闪烁

### 6.5 每一行都要理解

对每一行配置,用 `C-h v` 查变量文档:

```
C-h v inhibit-startup-screen RET
```

读它的文档,理解它在干什么。

如果某个变量名你不懂 (比如 `ring-bell-function`),`C-h v` 查。
然后 `(setq ring-bell-function 'ignore)` 改变它的值。

**第一性原理**: 你不是在"写配置",你在**调用 Elisp 函数**改变 Emacs 状态。
`setq` 是设置变量值的函数。`tool-bar-mode` 是一个函数,-1 表示关闭。

#### 深入: init.el 不是"配置文件",是"程序"

考虑 JSON 配置 (VS Code 的 settings.json):
- 静态数据,描述"我想要的状态"
- VS Code 读它,内部代码解释这些数据
- 用户只能改"允许的字段",不能加逻辑

Emacs init.el 是**真正的程序**:
- 加载时,Emacs 解释器**逐行执行**这些 Elisp
- 你可以写 `if`、`when`、`let`、`loop`——任何 Elisp
- 你可以调用任何函数,包括定义新函数

具体例子: 假设你想"只在 GUI 模式下关闭工具栏,TTY 下保留"——VS Code 做不到 (没有条件),Emacs 只需一行:

```elisp
(when (display-graphic-p)
  (tool-bar-mode -1))
```

这就是"配置即代码"的真正含义。init.el 是**程序**,可以表达任何逻辑。

#### 创造性应用: init.el 能做的"啊哈时刻"

**啊哈 1: 根据操作系统不同配置**

```elisp
(pcase system-type
  ('gnu/linux   (message "Linux 配置"))
  ('darwin      (message "macOS 配置"))
  ('windows-nt  (message "Windows 配置")))
```

同一份 init.el 在多台机器上工作,自动适配。

**啊哈 2: 根据时间/状态切换**

```elisp
;; 早上 6 点到 18 点用浅色主题,晚上用深色
(let ((hour (string-to-number (format-time-string "%H"))))
  (if (< 6 hour 18)
      (load-theme 'modus-operandi t)
    (load-theme 'modus-vivendi t)))
```

VS Code 做不到这种"运行时计算"的配置。

**啊哈 3: 读取环境变量决定行为**

```elisp
(when (getenv "EMACS_DEBUG")
  (setq debug-on-error t)
  (message "调试模式开启"))
```

启动时 `EMACS_DEBUG=1 emacs` 就能开 debug。普通 JSON 配置完全没有这种能力。

**啊哈 4: 在 init.el 里定义自己的"配置 DSL"**

```elisp
(defmacro my/setq-many (&rest pairs)
  "一次性设多个变量。"
  (let (forms)
    (while pairs
      (push `(setq ,(pop pairs) ,(pop pairs)) forms))
    `(progn ,@(nreverse forms))))

(my/setq-many
 inhibit-startup-screen t
 ring-bell-function 'ignore
 make-backup-files nil)
```

**你在 init.el 里写宏**——这种灵活性是 JSON 永远做不到的。

---

## 7. 毕业检查 (这个模块结束前能答出来)

### 概念题

1. **解释**: Emacs 为什么说"是 Lisp 机器伪装成编辑器"?
2. **架构**: Frame、Window、Buffer 的关系,画图说明。
3. **历史**: RMS 在哪一年做了第一版 Emacs? 用什么语言? 为什么选 Lisp?
4. **键位**: 为什么是 `C-x` 这种"两步前缀"?
5. **自文档**: `C-h f`、`C-h v`、`C-h k` 分别查什么?
6. **Live system**: 你刚改了一个函数定义,如何立即应用?
7. **Buffer**: `*scratch*`、`*Messages*`、`*Help*` 这些 buffer 的作用是什么?

### 实操题

1. 在 `*scratch*` 里 eval `(+ 1 2 3)`,确认看到 `6`
2. 在 `*scratch*` 里 eval `(message "Hello")`,看 minibuffer 显示什么
3. 用 `C-h k` 查 `C-g` 的功能
4. 用 `C-h f` 查 `find-file` 的文档
5. 用 `C-h v` 查 `kill-ring` 的当前值
6. 用 `C-h m` 查 `*scratch*` 的 major mode
7. 写一个 init.el 至少 10 行,逐行 `C-h v` 理解

### 反向题 (不要做什么)

1. ❌ 不要装 evil-mode (Module 5 之后再考虑)
2. ❌ 不要装一堆 melpa 包 (先用裸 Emacs 一周)
3. ❌ 不要抄别人的 init.el (一行一行自己写)
4. ❌ 不要用鼠标 (强迫自己用键盘)

---

## 8. 下一步

完成这个模块的所有检查后:

1. 在 `logs/module-00.md` 写一篇学习日志 (用.Module 0 多久,学到的最重要的概念,踩的坑)
2. 打开 `PROGRESS.md`,把 Module 0 状态改为 ✅
3. 进入 `01-survival/README.md`

---

## 附录: 推荐阅读 (可选,不要求)

> 这些是延伸材料,不是必需的。教程已 self-contained。

- **《Just For Fun》** Linus Torvalds 自传 — 理解 hacker 文化
- **《Free as in Freedom》** Sam Williams 写的 RMS 传记 — 理解 GNU 哲学
- **Emacs Wiki**: https://www.emacswiki.org/ — 社区知识库 (但有些过时)
- **r/emacs**: https://reddit.com/r/emacs — 最活跃的 Emacs 社区
- **System Crafters** (YouTube): https://systemcrafters.net/ — 高质量视频教程

---

完成 Module 0 的标志: **你能用 30 秒说清楚 Emacs 是什么,以及它和你用过的编辑器本质区别在哪**。
