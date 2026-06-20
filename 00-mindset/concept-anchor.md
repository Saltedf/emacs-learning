# Concept Anchor: Emacs 编辑器手册的概念锚点 (Module 0)

> 这个文件**完全替代** `emacs-manual-30.2/` 中的以下章节:
> - `screen.texi` (The Organization of the Screen)
> - `entering.texi` (Entering and Exiting Emacs)
> - `intro.texi` (Distribution, terms)
> - `gnu.texi` (Emacs History)
> - `glossary.texi` (Glossary,节选)
> - `xresources.texi` (X Resources) (速览)
> - `macos.texi` / `haiku.texi` / `android.texi` / `msdos.texi` (平台附录)
>
> 你不需要去翻原始 texi 文件。所有内容已内联在下面。

---

## 1. The Organization of the Screen (替代 screen.texi)

### 1.1 启动后的屏幕

打开 Emacs (GUI),你看到:

```
┌──────────────────────────────────────────────────────────────┐
│ File  Edit  Options  Buffers  Tools  ...                     │  ← Menu bar (GUI)
├──────────────────────────────────────────────────────────────┤
│ [图标图标图标图标图标图标图标图标图标图标图标图标图标图标]      │  ← Tool bar (GUI)
├──────────────────────────────────────────────────────────────┤
│                                                              │
│                                                              │
│                     编辑区域                                  │
│                                                              │
│                                                              │
│                                                              │
│                                                              │
│                                                              │
│ -uu-:**-F  foo.txt   Top L1    (Fundamental) ------          │  ← Mode line
├──────────────────────────────────────────────────────────────┤
│ For information about GNU Emacs...                           │  ← Echo area
└──────────────────────────────────────────────────────────────┘
```

TTY (终端) 下没有 Menu bar 和 Tool bar,其余一样。

### 1.2 各部分详解

**Menu bar** (`menu-bar-mode`):
- 顶部下拉菜单 (File/Edit/Options/...)
- 大多数 Emacs 用户关掉它 (`(menu-bar-mode -1)`),因为所有命令都有键位
- TTY 下用 `F10` 或 `M-`` 访问

**Tool bar** (`tool-bar-mode`):
- 图标工具栏
- 几乎所有人都关掉 (`(tool-bar-mode -1)`),占空间且不灵活

**编辑区域** (Window):
- 实际显示 buffer 内容的地方
- 一个 window 显示一个 buffer
- 可以分屏 (`C-x 2` 上下,`C-x 3` 左右)

**Mode line** (`mode-line-format`):
- 每个 window 底部的状态行
- 显示当前 buffer 的元数据
- 默认格式解读 (从左到右):
  ```
  -UU-:**-F  foo.txt   Top L1    (Fundamental) ----
  ```
  - `-UU-`: 文件编码 (UTF-8 Unix)
  - `:**-`: buffer 状态 (`**` 已修改,`--` 未修改,`%%` 只读已修改,`%%` 只读)
  - `F`: Frame 的 focus 状态
  - `foo.txt`: buffer 名
  - `Top`: buffer 在顶部 (还可显示 `Bot` 底部,或百分比)
  - `L1`: 当前光标行号
  - `(Fundamental)`: major mode 名
  - `----`: window 边界

**Echo area** (echo area):
- 最底部一行,默认显示提示信息
- 当你按 `M-x` 或 `C-x C-f` 时,变成 **minibuffer**,接收你的输入

#### 深入: 为什么是"一个 Echo area 兼任 Minibuffer"

这是 Emacs 设计的一个反直觉选择。考虑普通软件:
- 状态栏只显示信息 (静态)
- 输入框单独有 UI 组件 (弹窗、对话框)

Emacs 把这两个角色**复用同一行**——空闲时它显示消息 (`message` 函数输出),需要输入时它变成 minibuffer。

**为什么这样设计?**
1. 终端字符位置宝贵 (1970s 终端 24 行 × 80 列,不能浪费一行给"输入框")
2. 一个区域两个角色 = 代码复用 (minibuffer 本质就是 buffer)
3. 你看到的 "M-x" 不是弹出窗口,是 echo area **切换了用途**

**反事实**: 如果 minibuffer 是独立窗口 (像 VS Code 的命令面板)
- 屏幕少一行
- 用户视线要在主编辑区和弹窗间跳
- minibuffer 内容不能 scroll-back (Emacs 里 minibuffer 历史 `M-p`/`M-n` 是真实 buffer 内容)

#### 创造性应用: Minibuffer 的"啊哈时刻"

**啊哈 1: minibuffer 历史**

按 `M-x`,然后按 `M-p`——看到你上次 `M-x` 的命令。`M-p`/`M-n` 翻 minibuffer 历史,像 shell 的上下箭头。这不是"扩展功能",是 minibuffer 作为 buffer 的自然属性。

**啊哈 2: `enable-recursive-minibuffers`**

默认 minibuffer 里不能再用 minibuffer。但开启 `(setq enable-recursive-minibuffers t)` 后,你可以在 `M-x` 输入一半时再按 `C-x C-f` 找文件,然后回到原 `M-x`。这是 buffer 抽象的极限——minibuffer 是 buffer,minibuffer 里的 minibuffer 也是 buffer。

### 1.3 特殊 buffer

| Buffer 名 | 用途 |
|---|---|
| `*scratch*` | Lisp 交互 buffer,默认无文件关联 |
| `*Messages*` | 所有 Emacs 输出的消息,完整 log |
| `*Completions*` | 补全候选列表 |
| `*Help*` | `C-h` 系列命令的输出 |
| `*Buffer List*` | buffer 列表 (`C-x C-b`) |
| `*shell*` / `*terminal*` | shell 输出 |
| `*Compile-Log*` | 编译日志 |

切换到这些 buffer: `C-x b *Messages* RET`。

### 1.4 Point 和 Cursor

- **Point**: 抽象的"当前位置",是一个 integer (buffer 中字符的偏移,从 1 开始)
- **Cursor**: 你看到的闪烁方块,显示 point 的位置
- `(point)` 函数返回当前 point 值
- 在 `*scratch*` 试: `(point)` → 返回一个数字

### 1.5 Mode line 自定义

`mode-line-format` 是一个复杂变量,可以自定义。常见增强:

```elisp
(setq mode-line-format
      '("%e"
        mode-line-front-space
        mode-line-mule-info
        mode-line-client
        mode-line-modified
        mode-line-remote
        mode-line-frame-identification
        mode-line-buffer-identification
        "   "
        mode-line-position
        (vc-mode vc-mode)
        "  "
        mode-line-modes
        mode-line-misc-info
        mode-line-end-spaces))
```

或者用 `doom-modeline` / `smart-mode-line` 包 (Module 4 之后)。

#### 深入: 为什么 mode line 是一个 Elisp 表达式而非模板字符串

考虑普通软件的状态栏: 通常是模板字符串 `"Line: {line}, Col: {col}"`,模板引擎渲染。简单但不灵活——比如"如果文件是只读则显示红色 lock 图标"这种条件需要专门加 if 语法。

Emacs 的 mode-line-format 是**列表形式的 Elisp 表达式**,可以在任意位置嵌套 `(:eval (my-fn))` 动态计算:

```elisp
(setq mode-line-format
      '(:eval (if (buffer-modified-p)
                  "** modified **"
                "clean")))
```

这是"配置即代码"在 mode line 上的体现。mode line 是**实时计算的程序**,不是字符串。

#### 创造性应用: mode line 的"啊哈时刻"

**啊哈 1: 把 mode line 当状态仪表盘**

```elisp
(setq mode-line-format
      '("%e"
        mode-line-front-space
        "%*"                              ; 修改标记
        "  "
        mode-line-buffer-identification   ; buffer 名
        "  "
        (:eval (format "WC:%d" (count-words (point-min) (point-max))))  ; 词数
        "  "
        (:eval (when (boundp 'eglot--managed-mode)
                 "LSP:✓"))                ; LSP 状态
        "  "
        mode-line-position))
```

你的 mode line 实时显示**任意计算的状态**——词数、git 分支、电池、天气、CPU 使用率——任何能用 Elisp 计算的东西。

**啊哈 2: 不同 buffer 的 mode line 不同**

每个 buffer 的 `mode-line-format` 是 buffer-local 变量。你可以让 `prog-mode` buffer 显示编译错误数,让 `org-mode` buffer 显示 todo 数。这是 buffer 抽象的延伸——**buffer 不仅有自己的文本,还有自己的 UI 元数据**。

---

## 2. Entering and Exiting Emacs (替代 entering.texi)

### 2.1 启动方式

```bash
emacs                  # 启动 GUI (如果编译了 GUI)
emacs -nw              # 强制 TTY (no window)
emacs file.txt         # 启动并打开 file.txt
emacs -q               # 不加载 init.el (干净启动)
emacs -Q               # 比 -q 更彻底,也不加载 site-lisp
emacs --batch          # 批处理模式,不进入交互 (适合脚本)
emacs --daemon         # 后台 daemon 模式,然后用 emacsclient 连接
```

**Daemon 模式推荐**:

```bash
emacs --daemon         # 启动一次,后台运行
emacsclient -c         # 打开新 frame 连接到 daemon (秒开)
emacsclient -t         # 终端版本
```

这样你的 Emacs 进程一直运行,所有 buffer 状态保持,开关 frame 都很快。

把 `(server-start)` 加到 init.el 也可以让普通 Emacs 接受 emacsclient 连接。

#### 深入: 为什么 Daemon 模式如此强大

考虑普通编辑器的"启动-关闭"周期:
- 打开 VS Code → 等 5 秒加载扩展 → 工作 → 关闭 → 所有上下文丢失
- 下次再开,又是 5 秒,所有"打开的文件"列表还要重新加载

Emacs daemon 是**完全不同的模型**:
- `emacs --daemon` 启动一次,持续在后台运行 (永远不退出)
- 你的 init.el 在 daemon 启动时跑一次,所有配置加载好
- `emacsclient -c` 创建一个 frame (UI 窗口),连接到这个 daemon,**毫秒级**
- 关 frame 只关 UI,不关 daemon——buffer、window 布局、变量状态全部保留
- 下次 `emacsclient -c`,瞬间回到上次状态

**为什么这能工作?** 因为 Emacs 的状态全在 Lisp 机器的"内存"里。frame 只是一个 view,关 view 不杀 Lisp 机器。这又是 buffer/window 解耦的福利。

#### 创造性应用: daemon 模式的"啊哈时刻"

**啊哈 1: "永远不关 Emacs"**

很多 Emacs 用户开机自动 `systemctl --user start emacs`,然后整天用 `emacsclient`。**从不真正退出 Emacs**——所有 buffer 状态、minibuffer 历史、recent files、window 配置全保留。一周不重启很正常。

**啊哈 2: 多设备共享一个 Emacs**

把 Emacs daemon 跑在服务器上,在本地用 `emacsclient -c` 远程连接 (通过 SSH 转发)。所有编辑都在服务器上做,本地只是 UI。这是普通编辑器做不到的。

**啊哈 3: `emacsclient --eval` 远程操作**

```bash
emacsclient --eval '(with-current-buffer "todo.org" (org-agenda-list))'
```

从 shell 调 Emacs 内的函数。你可以写脚本,让 cron 在每天早上让 Emacs 把 org-agenda 输出成 HTML。

### 2.2 退出

| 操作 | 命令 | 说明 |
|---|---|---|
| `C-x C-c` | `save-buffers-kill-terminal` | 保存所有未保存 buffer,退出 |
| `C-x C-z` | `suspend-emacs` | 挂起 (TTY 下回到 shell,用 `fg` 唤回;GUI 下最小化) |

**Daemon 模式下不要用 `C-x C-c`** (会杀掉 daemon),用 `C-x 5 0` 关闭当前 frame。

### 2.3 启动选项速查

| 选项 | 作用 |
|---|---|
| `-q` | 不加载 `~/.emacs` / `~/.config/emacs/init.el` |
| `-Q` | `-q` + 不加载 site-lisp + 不加载 package.el 默认包 |
| `-nw` / `--no-window-system` | 强制 TTY |
| `--debug-init` | init.el 报错时进入 debugger |
| `--batch` | 跑完 `--eval` 后退出,适合脚本 |
| `--daemon` | 后台 daemon |
| `-l FILE` | 加载指定文件 |
| `--eval EXPR` | 启动时 eval 表达式 |

例子:

```bash
# 用某个特定配置启动
emacs -q -l ~/special-config.el

# 跑一个 elisp 脚本
emacs --batch --eval '(message "hello")'

# 调试 init.el
emacs --debug-init
```

---

## 3. Distribution & GNU Project (替代 intro.texi 的 Distribution 节)

### 3.1 GNU Emacs 是什么

GNU Emacs 是 **GNU Project** 的旗舰软件之一,由 **Free Software Foundation (FSF)** 发布。
它是 **自由软件** (free software),不只是开源。

**自由** 指 4 项自由:
- 自由 0: 为任何目的运行程序
- 自由 1: 研究源码并修改
- 自由 2: 重新分发
- 自由 3: 改进并发布改进

许可证: **GPL v3+** (GNU General Public License version 3 or later)。

### 3.2 怎么获取

- **源码**: `git clone https://git.savannah.gnu.org/git/emacs.git`
- **官方下载**: https://ftp.gnu.org/gnu/emacs/
- **Linux 包管理**: 各发行版都有 `emacs` 包 (但通常落后)
- **macOS**: `brew install emacs` 或下载 https://emacsformacosx.com/
- **Windows**: 官方有 .zip,或用 MSYS2

### 3.3 与其他编辑器的关系

| 编辑器 | 关系 |
|---|---|
| **XEmacs** | 1991 fork,2020s 维护停滞,基本死 |
| **Lucid Emacs** | XEmacs 的前身 |
| **µEmacs** (MicroEMACS) | 简化版,Stallman 的早期合作者 Dan Murphy 写的 |
| **mg** | 轻量 Emacs 克隆,OpenBSD 默认 |
| **VISUAL EDITOR / vim** | 完全不同的家族 (vi 派系) |
| **Spacemacs / Doom Emacs** | 基于 GNU Emacs 的"配置发行版",不是 fork |

注意: Spacemacs、Doom、Prelude、Centaur 这些都是 **配置发行版**——它们是别人写好的 init.el。
本教程教你**自己写**,所以不要装这些。

#### 深入: 为什么"配置发行版"是双刃剑

Doom Emacs、Spacemacs 这类"开箱即用"配置看起来很诱人——一条命令装好,效果惊艳。但代价:

1. **你不理解**: Doom 改了几百个变量,你不知道为什么某行为是这样
2. **难调试**: 出问题时不知道是哪个包/配置冲突
3. **难扩展**: 想加自己的功能,要懂 Doom 的内部架构
4. **依赖作者**: Doom 维护停滞,你的配置就过时

**反事实**: 如果 Emacs 像 VS Code 那样默认就有"漂亮"配置
- 用户门槛低,启动即用
- 但失去"完全掌控"
- Emacs 的极客精神 (自己改造) 被弱化

Emacs 选择"开箱是空白页"——这是哲学选择,不是技术缺陷。本教程带你从空白页开始,**让你真正理解每一行配置**。理解之后,你可以读 Doom 的源码,挑你喜欢的部分抄过来——这才是真正的掌控。

---

## 4. Emacs History (替代 gnu.texi)

### 4.1 时间线

```
1976   EMACS (TECO macros) 诞生于 MIT AI Lab
       Stallman + Steele 收集整理

1981   Stallman 离开 MIT,创立 GNU Project

1985   GNU Emacs 1.0 发布 (用 C 写核心 + Emacs Lisp)

1991   Lucid Emacs 19 fork,后改名 XEmacs
       "Emacs vs XEmacs" 之争开始

1993   GNU Emacs 19 发布,引入 X11 支持

1996   GNU Emacs 20,引入 MULE (多语言)

2001   GNU Emacs 21,引入 fringe, image support

2007   GNU Emacs 22,集成 CEDET,UCS 支持

2009   GNU Emacs 23,更好的 Unicode,Newsticker

2012   GNU Emacs 24,引入 package.el (革命性)

2014   GNU Emacs 24.4 内置 webkit (xwidget)

2017   GNU Emacs 25,引入 lexical binding 默认

2020   GNU Emacs 27,early-init.el, native comp 实验性

2022   GNU Emacs 28,native comp 正式

2023   GNU Emacs 29,tree-sitter 集成,eglot 内置

2024   GNU Emacs 30,use-package 完全内置,pgtk 主线
```

### 4.2 关键事件详细

**TECO Emacs (1976)**:
- Carl Mikkelsen 写了实时显示更新 (real-time display),这是革命性的
- Stallman 把它和 TECO macros 收集整合,叫 EMACS
- 名字 "Editor MACroS" 是 Steele 起的

**GNU Emacs (1985)**:
- Stallman 决定用 C 写核心,而不是 TECO
- 嵌入一个 Lisp 解释器 (Emacs Lisp),所有编辑命令都是 Lisp 函数
- 这个架构 ("C core + Lisp 上层") 至今未变

**XEmacs 分裂 (1991)**:
- Lucid 公司的 Jamie Zawinski 等人 fork 了 Emacs 19
- 起因是开发节奏冲突 (Lucid 想加功能,Stallman 想保持简洁)
- XEmacs 长期在某些方面领先 (GUI、内嵌图片)
- 2015 年后基本停滞

**package.el (2012)**:
- 之前装包要手动下载 .el 文件
- package.el 引入 ELPA (Emacs Lisp Package Archive)
- 后来社区搞了 MELPA,生态爆发

**native compilation (2022)**:
- Andrea Corallo 实现,把 Elisp 用 libgccjit 编译成原生机器码
- 性能提升 2-5 倍
- 让 Elisp 不再"慢"

#### 深入: package.el 如何改变了 Emacs 生态

考虑 2010 年前的 Emacs 用户:
- 想装一个 mode,要去作者主页下 .el
- 手动放到 `~/.emacs.d/lisp/`
- 在 init.el 写 `(require 'mode-name)`
- 更新要重新下载,版本兼容自己处理
- 没有依赖管理 (装 A 包,A 依赖 B,你要自己装 B)

package.el (2012, Emacs 24) 解决了所有这些问题:
- `M-x package-install RET magit RET` 自动下载、放到 load-path、记录依赖
- `M-x list-packages` 浏览所有 ELPA 包
- 版本管理: `package.el` 检查依赖版本是否兼容
- MELPA 出现后,大量包快速涌现

**这是 Emacs 第二次革命** (第一次是 1985 Lisp 内核)。生态从"几百个独立脚本"变成"统一包管理的现代生态"。

**反事实**: 如果没有 package.el
- Emacs 可能早在 2015 年就被 Sublime/Atom 替代——因为装包太痛苦
- VS Code 的 extension marketplace 学的就是 package.el 模式
- Doom Emacs 等配置发行版根本不可能存在 (它们依赖 package.el 装所有包)

#### 创造性应用: package.el 的"啊哈时刻"

**啊哈 1: 一个 use-package 表达式 = 一个完整配置**

```elisp
(use-package magit
  :ensure t                        ; 自动装
  :bind ("C-x g" . magit-status)   ; 绑定键
  :config                          ; 加载后跑
  (setq magit-push-always-verify nil))
```

这一行做了: 检查是否装 → 没装就装 → 加载 → 设变量 → 绑键。比 VS Code 的 settings.json + keybindings.json + extensions.json 三个文件分散管理优雅得多。

**啊哈 2: 自己的包发到 MELPA**

你写一个好用的 elisp,推到 GitHub,在 MELPA recipe 里加一行。**几小时后全世界 Emacs 用户 `M-x package-install RET your-package RET` 就能用**。这是开源生态的最短路径。

### 4.3 文化: "Eight Megabytes And Constantly Swapping"

Emacs 的玩笑缩写:
- "Eight Megabytes And Constantly Swapping" (1980s,觉得 Emacs 太大)
- "Emacs Makes A Computer Slow"
- "Escape Meta Alt Control Shift"

现在看 8MB 笑话很可爱——Chrome 一个 tab 100MB。

### 4.4 "Operating System" 笑话

> "Emacs 是一个优秀的操作系统,只缺一个好编辑器。"

这是说 Emacs 太大而全——能发邮件 (Rmail/mu4e)、看 PDF (doc-view)、浏览网页 (eww、xwidget)、玩俄罗斯方块 (`M-x tetris`)、当计算器 (`M-x calc`)、画图 (artist-mode)、终端 (eshell)、IRC (erc)...

Kernel 不在 Emacs 内 (但 EXWM 把 X server 当 Emacs 的输入法,几乎就是 OS 了)。

#### 深入: "Emacs 是 OS" 笑话背后的真理

这个笑话半真半假。**Emacs 不是 OS,但它有 OS 的特征**:
- 它有进程管理 (`make-process`、`start-process`)
- 它有文件系统抽象 (`expand-file-name`、`directory-files`)
- 它有窗口管理 (frame、window 切换)
- 它有事件循环 (key strokes、timer、process filter)
- 它有"应用商店" (package.el)
- 它有大量"应用" (邮件、IRC、shell、git、PDF、网页)

唯一缺的是**硬件驱动**和**用户隔离**——这些是 Linux kernel 干的。

EXWM (Emacs X Window Manager) 把这个笑话推到极致: **X server 的窗口都变成 Emacs buffer**。你的 Chrome、终端、PDF reader 全是 Emacs 的 window。这就是"Emacs 当 OS"的真实实现。

#### 反事实: 如果 Stallman 当年真把 Emacs 做成 OS

1985 年 Stallman 选择了"GNU 是 OS,Emacs 是 OS 上的应用"这条路。如果反过来——**Emacs 是 OS,Linux 是 Emacs 上的应用**:
- 我们不会有现在的 Linux 服务器市场
- 但个人电脑用户可能更早有 Lisp Machine 体验
- 信息安全的整个模型会不同 (Lisp 机器的"对象级安全"vs Unix 的"用户级安全")
- 我们今天讨论的可能是"Emacs 内核 vs Hurd"

这是历史 fork 点。Stallman 没走那条路,但 Lisp Machine 文化在 Emacs 里延续下来了。

---

## 5. Glossary 节选 (替代 glossary.texi 的常用术语)

> 完整 glossary 用 `C-h i d m emacs RET m glossary RET` 查 (但这不要求,你的教程已涵盖)

| 术语 | 含义 |
|---|---|
| **Buffer** | 文本容器,与文件解耦 |
| **Window** | Frame 内的显示区域,显示一个 buffer |
| **Frame** | 物理窗口 (GUI) 或整个终端屏幕 (TTY) |
| **Point** | 当前光标位置 (integer) |
| **Mark** | 标记的位置 |
| **Region** | Point 和 Mark 之间的文本 |
| **Kill** | 剪切 (进 kill-ring) |
| **Yank** | 粘贴 (从 kill-ring) |
| **Kill ring** | kill 的历史 list,有循环指针 |
| **Minibuffer** | 底部特殊 buffer,承担命令交互 |
| **Echo area** | 最底部一行,显示消息 |
| **Mode line** | window 底部的状态行 |
| **Major mode** | buffer 的主模式 (如 `python-mode`) |
| **Minor mode** | 附加功能 (如 `linum-mode`) |
| **Hook** | 一组函数,在某事件发生时运行 |
| **Keymap** | 键位到 command 的映射 |
| **Prefix arg** | `C-u` 或数字前缀,影响命令行为 |
| **Sexp** | S-expression,即一个 Lisp 表达式 |
| **Defun** | 定义函数 (Lisp) |
| **Form** | 一个 Lisp 表达式 (可以是 atom 或 list) |
| **Eval** | 求值 |
| **Symbol** | Lisp 的标识符,可绑定 variable 和 function |
| **Load** | 加载一个 .el 文件 |
| **Require** | 加载某个 feature (如果还没加载) |
| **Feature** | 一个被 require 的命名功能单元 |
| **Auto-load** | 函数被调用时才加载其定义 |
| **File-local variable** | 某个文件专属的变量值 (用 `-*- var: value -*-` 或 file footer) |
| **Dir-local variable** | 某个目录下所有文件共享的变量值 |
| **Echo** | 在 echo area 显示消息 |
| **Sit for** | 等待 N 秒或用户输入 |

后面遇到不懂的术语,先来这里查;查不到,`C-h i d m emacs RET m glossary RET` 查官方。

---

## 6. X Resources (替代 xresources.texi)

### 6.1 什么是 X Resources

X Window 系统下,程序可以从 `~/.Xresources` 或 `~/.Xdefaults` 读取配置。
Emacs 也支持,但**现代用法已经很少用了**,因为 init.el 可以做一切。

### 6.2 速览

```Xresources
! 字体和大小
Emacs.font: JetBrains Mono-14

! 几何位置 (列x行)
Emacs.geometry: 80x24

! 光标
Emacs.cursorColor: white
```

然后 `xrdb -merge ~/.Xresources`。

但你可以完全忽略 X Resources,用 init.el 设置一切:

```elisp
(set-face-attribute 'default nil :family "JetBrains Mono" :height 140)
(setq default-frame-alist '((width . 100) (height . 50)))
```

---

## 7. 平台附录 (macOS / Haiku / Android / MS-DOS)

> 这些章节是平台特定的备注。大多数用户只关心自己用的那个平台。

### 7.1 macOS (`macos.texi`)

- 推荐用 `brew install --cask emacs` 装的版本 (有 native comp + tree-sitter)
- 或下载 https://emacsformacosx.com/
- Cmd 键默认是 `s-` (super)
- Option 键默认是 `M-` (meta) — 需要在 Emacs 设置 `(setq mac-option-modifier 'meta)`
- 触控板手势不会传递给 Emacs (除了基本滚动)
- 推荐装 `exec-path-from-shell` 包,让 Emacs 知道 shell 的 PATH

### 7.2 Haiku (`haiku.texi`)

Haiku OS 是 BeOS 的开源继承者。Emacs 30+ 支持。99% 用户不需要关心。

### 7.3 Android (`android.texi`)

Emacs 29+ 有 Android 原生移植。可以在手机/平板上跑。
对学习有用: 在手机上写代码、看文档。
Termux 也可以跑 TTY 版本。

### 7.4 MS-DOS (`msdos.texi`)

历史遗留。没人真的在 DOS 上用 Emacs 了。可以跳过。

---

## 8. 这个锚点对照到 Elisp 的什么

本模块的"概念锚点"已经讲完。把它们和 Elisp 对应:

| Editor 概念 | Elisp 函数/变量 |
|---|---|
| Frame | `(selected-frame)`、`(make-frame)`、`modify-frame-parameters` |
| Window | `(selected-window)`、`split-window`、`delete-window` |
| Buffer | `(current-buffer)`、`get-buffer-create`、`with-current-buffer` |
| Point | `(point)`、`(point-min)`、`(point-max)`、`goto-char` |
| Mark | `(mark)`、`(set-mark-command)`、`mark-ring` |
| Region | `(region-beginning)`、`(region-end)`、`buffer-substring` |
| Mode line | `mode-line-format` |
| Minibuffer | `read-from-minibuffer`、`completing-read`、`read-file-name` |
| Echo area | `message`、`princ`、`format` |

这些函数在 Module 3 (Chassell) 和 Module 6 (Reference) 会系统学。
**Module 0 你只需要知道概念**,具体函数后面会用到。

---

## 9. 自测

完成这个 concept-anchor 后,你应该能:

- [ ] 画一个 Emacs 屏幕结构图,标出 Frame/Window/Buffer/Mode line/Echo area
- [ ] 解释 point 和 mark 的区别,以及 region 是什么
- [ ] 说出至少 5 个特殊 buffer 的名字和用途
- [ ] 知道 `emacs -Q` 和 `emacs --daemon` 各自的用途
- [ ] 复述 Emacs 的简史 (1976 TECO → 1985 GNU Emacs → 1991 XEmacs → ...)
- [ ] 知道 GPL v3+ 和"4 项自由"
- [ ] 看到术语 "Defun" / "Kill ring" / "Sexp" 能立刻反应过来
- [ ] 知道 X Resources 是过时的、macOS 上 Option 怎么映射、Android 有原生版

如果一半以上答不出来,重读相关章节。

---

## 10. 下一步

回 `00-mindset/README.md` 继续看完 Module 0 的剩余部分 (编译安装 + init.el),
或直接进 `00-mindset/intro-reading.md` 看 Chassell 的 Lisp History。
