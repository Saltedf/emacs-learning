# Ref Deep Dive: Emacs Internals (Module 8)

> 替代 `emacs-lispref-30.2/internals.texi` + `anti.texi`

这个文件深入 Emacs 内部——给你贡献 Emacs core 之前的"地图"。前面模块里你用了 Emacs,但没问过"它内部怎么工作的"。读源码、提 patch 到 core,你必须懂这些 internals,否则读 `src/alloc.c` 的几千行 C 代码完全懵。

Internals 不是黑魔法——是工程决策。每个决策都有历史原因,理解原因能让你知道"为什么这里这样写",而不是觉得莫名其妙。

---

## 1. internals.texi

### 1.1 Emacs 架构

Emacs 是个**分层系统**。最上面是用户写的 Elisp,中间是 Emacs 自己的 byte-code 或 native code,下面是 C core 提供基本原语,最底是 libc 和 OS。这种分层让用户代码不需要直接面对 C 的复杂性——你写 Elisp,Emacs runtime 把它翻译成可执行的东西。

> ```
> ┌─────────────────────────┐
> │   Elisp (运行时)        │  ←── 你写的代码
> ├─────────────────────────┤
> │   byte-code / native    │
> ├─────────────────────────┤
> │   C core (alloc, etc.)  │
> ├─────────────────────────┤
> │   libc / OS calls       │
> └─────────────────────────┘
> ```

这种架构的精妙之处:**Elisp 是动态的**——你可以 `eval` 任意表达式、运行时改函数定义,C core 不需要重新编译。这是为什么 Emacs 能"自文档化"和"热重载"。代价是性能——纯 Elisp 比 C 慢一个数量级,所以性能敏感的部分 (GC、display engine) 用 C 写。

### 1.2 C core 文件

(Emacs source 的 `src/`)

这些文件名告诉你 Emacs core 的结构。每个文件负责一个领域——这是 1970 年代的 Unix 风格,一个文件做一件事。读懂这个表,你就知道改什么去哪个文件找。

| 文件 | 内容 |
|---|---|
| alloc.c | GC, 内存分配 |
| buffer.c | buffer 数据结构 |
| bytecode.c | byte-code interpreter |
| callint.c | interactive 调用 |
| cmds.c | 基础命令 |
| data.c | Lisp types (cons, vector, symbol) |
| dired.c | Dired C 部分 |
| doc.c | 文档加载 |
| editfns.c | 编辑函数 (point, marker, etc.) |
| eval.c | eval, funcall, apply |
| fileio.c | 文件 I/O |
| filelock.c | 文件锁 |
| fns.c | 通用函数 (length, equal, ...) |
| font.c | 字体 |
| frame.c | frame |
| fringe.c | fringe |
| image.c | 图片显示 |
| indent.c | 缩进 |
| insdel.c | 插入删除 |
| keyboard.c | 键盘事件 |
| keymap.c | keymap |
| lread.c | Lisp reader (read) |
| marker.c | marker |
| minibuf.c | minibuffer |
| print.c | Lisp printer (print) |
| process.c | 进程 |
| search.c | 搜索 |
| syntax.c | syntax table |
| term.c | terminal |
| textprop.c | text properties |
| undo.c | undo |
| window.c | window |
| xdisp.c | display engine |
| xfaces.c | face |
| xterm.c | X11 |

文件名都是简短缩写——这是 1980 年代的习惯,文件系统 inode 限制文件名 14 字符,所以短名。这个限制早就没了,但传统保留。`xdisp.c` (display engine) 是 Emacs 最复杂的文件之一——你想理解 Emacs 怎么把 buffer 渲染到屏幕,读 xdisp.c。

### 1.3 内存管理

Emacs 用 GC (garbage collection) 管内存——你创建 Lisp object,GC 自动回收不用的。这避免手动 free 内存的 bug (C 的常见崩溃源)。

GC 算法是 **mark-and-sweep**:GC 从 root (栈、变量) 出发,标记所有可达的 object;然后扫描整个堆,释放没标记的。这个算法简单但暂停时间长——大 GC 时 Emacs 会卡一下。

GC 触发时机:
- **分配量超过 `gc-cons-threshold`** (默认 800KB)
- **分配量超过 `gc-cons-percentage` × 已用堆** (默认 0.1)
- **手动 `(garbage-collect)`**

> ```elisp
> (gc-cons-threshold)           ; 当前阈值
> (garbage-collect)             ; 跑 GC
> (memory-use-counts)           ; 内存统计
> (memory-limit)                ; 上限
> ```

这些函数在调试内存问题时有用——比如你的包吃内存,你想知道是不是 GC 不够频繁,可以 `M-: (gc-cons-threshold)` 看当前值。

### 1.4 GC 优化

启动慢因为 GC 频繁——Emacs 加载大量 Lisp 文件,中间产生大量临时 object,GC 一次一次触发。优化技巧:**启动时把阈值设到无限大**,启动完恢复正常。

> ```elisp
> ;; early-init.el
> (setq gc-cons-threshold most-positive-fixnum)
> (setq gc-cons-percentage 0.5)
> (add-hook 'emacs-startup-hook
>           (lambda ()
>             (setq gc-cons-threshold 800000)
>             (setq gc-cons-percentage 0.1)
>             (message "Emacs ready")))
> ```

`most-positive-fixnum` 是个 Elisp 能表示的最大整数——把 GC 阈值设到这个,等于"启动时永不 GC"。startup hook 在 Emacs 完全启动后跑,把阈值恢复正常。这个 trick 能让启动时间从 2 秒降到 0.5 秒。

但**不要永久把 GC 关掉**——内存会爆。GC 是健康的,只是启动时频繁触发是浪费。

### 1.5 buffer 内部

Emacs buffer 用 **Gap buffer** 实现——一个连续的字符数组,中间有个 gap。这个数据结构在 1970 年代就是文本编辑器的标准,因为编辑操作有"局部性"——你大部分时间在光标附近编辑。

> ```
> [abc...def_______ghi]
>        ^   ^
>        gap 大小
> ```

Gap buffer 的精妙:插入字符是 O(1)——把字符填进 gap,缩小 gap。删除也是 O(1)——扩大 gap。只有移动光标时才需要 O(n) 移动 gap——但移动光标的"成本"分摊在多次操作里,实际很快。

如果用普通数组,每次插入都是 O(n) (后面所有字符都要移动)。Gap buffer 把这个 O(n) 推迟到"光标大幅跳跃"时,而频繁的局部编辑是 O(1)。

> ```elisp
> (gap-size)        ; 当前 gap 大小
> (gap-position)    ; 当前 gap 位置
> ```

这两个函数是 C 内部用的,但 Elisp 也能调——调试 buffer 操作性能时有用。

### 1.6 pdumper

Pre-dump (Emacs 27+): 启动时加载一个 dump,跳过 load 时间。

Emacs 启动时,要加载几百个 .el 文件 (简单.el、files.el、...),即使有 .elc 缓存,read + eval 还是慢。pdumper 把这个状态**预序列化**成一个二进制文件,启动时直接 mmap,跳过 read/eval。

> ```bash
> emacs --dump-file ~/.config/emacs/emacs.pdump
> ```

或编译时 `--with-pdumper` (默认开)。pdumper 让启动从 1-2 秒降到 100ms 以下。这是 Emacs 27 的重大改进——之前用户都是用各种 trick (lazy load、defer) 来缩短启动,pdumper 一次性解决。

pdumper 生成的 dump 包含所有预加载的 Lisp 状态——你 `M-x find-library` 加载的 simple.el 等核心文件都已经在 dump 里,所以不需要重新读盘。

### 1.7 native comp

Module 0 装过。Native compilation 把 Elisp 编译成机器码,绕过 byte-code interpreter,性能 2-10x。

技术细节:用 libgccjit (GCC 的 JIT 库) 把 Elisp AST → C → 机器码。生成的 .eln 文件可以分发——别的同架构 Emacs 加载时直接用。

这是 Andrea Corallo (Emacs 维护者之一) 在 2020 年引入的,是 Emacs 近年来最大的性能改进。之前 byte-code 已经比纯 Elisp 快很多,native comp 又快一倍——很多计算密集的包 (lsp-mode、magit) 启动和操作都更流畅。

但 native comp 不解决所有问题——大量"等待 I/O"的操作 (网络、文件) 还是慢,因为瓶颈不在 CPU。I/O bound 的工作需要并发 (Emacs 28+ 的 thread,或 async 包) 才能加速。

---

## 2. anti.texi (Antinews)

Emacs manual 每版都有一个**玩笑章节**,叫 "Antinews"——讲"如果这是上一版本会怎样"。这是 Emacs 文化的一个 tradition,从 1980 年代就有了。

Antinews 把上一版本的特点**反向叙述**——好像它们是新功能一样。例: Emacs 30 的 antinews 讲 "Emacs 29":

> "In Emacs 29, we removed native compilation because nobody used it..."

Antinews 是 Emacs 幽默感的体现——一群极客用反讽来回顾自己项目的演化。读 antinews 是 Emacs 文化的一部分,也是新手和老手的"暗号"——能笑 antinews 的人懂 Emacs 的历史。

---

## 3. Standard Errors / Keymaps / Hooks (附录)

(Module 6 提过)

### 3.1 Standard Errors

Emacs 预定义的错误类型,每个有特定含义。写代码 throw 错误时用正确的类型,让 caller 能 `condition-case` 精确捕获。

主要类型:
- `error`——通用错误
- `arith-error`——除零等
- `args-out-of-range`——索引越界
- `wrong-type-argument`——类型错 (e.g., 给 string 函数传了 number)
- `void-variable`——变量未定义
- `void-function`——函数未定义
- `file-error`——文件 I/O 失败
- `search-failed`——re-search 找不到

### 3.2 Standard Keymaps

Emacs 的全局键位组织成嵌套 keymap——而不是一个巨大的 flat table。这让 minor mode 可以"挂"自己的键到 prefix (如 `C-c`) 而不冲突。

主要 keymap:
- `global-map`——所有 buffer 共享
- `ctl-x-map`——`C-x` 后的子 keymap
- `ctl-x-4-map`——`C-x 4` (其他 window)
- `mode-specific-map`——`C-c` 后 (用户/mode 自定义)
- `esc-map`——`ESC` 后 (= Meta 在某些终端)

### 3.3 Standard Hooks

Hook 是函数列表,特定事件触发时调用所有函数。Emacs 有几百个 hook,主要分几类:

- **init hooks**: `before-init-hook`, `after-init-hook`, `emacs-startup-hook`, `window-setup-hook`
- **file hooks**: `find-file-hook`, `find-file-not-found-functions`, `write-file-functions`, `after-save-hook`
- **buffer hooks**: `kill-buffer-hook`, `kill-buffer-query-functions`
- **window hooks**: `window-configuration-change-hook`, `window-size-change-functions`

每个 hook 有"什么时候触发"和"能否 abort"的语义——读 docstring 知道。

---

## 4. 给 Emacs core 贡献

### 4.1 找任务

Emacs 有一个 TODO 系统——maintainer 把没时间做的任务记下来,等待贡献者来做。这些是"准备好被做"的任务,不会引发争议。

> ```elisp
> M-x view-emacs-todo
> ```

这个命令显示 Emacs 源码树里的 TODO 文件——包括 `etc/TODO` (用户可见) 和 `admin/notes/TODO` (内部)。

也可以直接读源码树里的文件:

> ```bash
> cat admin/notes/TODO
> cat etc/TODO
> ```

`etc/TODO` 是新人友好的——里面有 "wishlist" 性质的特性,标了难度。`admin/notes/` 是 maintainer 内部笔记,有时有具体的设计思路。

### 4.2 setup dev 环境

要贡献 Emacs core,你必须能**本地编译**——否则你改的代码没法测试。Emacs core 是 C + Lisp 混合,需要完整的 build chain。

Emacs core 的源码托管在 GNU Savannah (不是 GitHub!)。这是因为 GNU 项目坚持用自己的服务器,避免依赖商业平台。Savannah 基于 Savane,是 FSF 自己开发的代码托管系统,功能简单但够用。

> ```bash
> git clone https://git.savannah.gnu.org/git/emacs.git
> cd emacs
> ./autogen.sh
> ./configure --prefix=/tmp/emacs-dev
> make -j$(nproc)
> ./src/emacs -Q            # 跑你的编译
> ```

这个流程和普通 C 项目类似,但 Emacs 有一些特点:第一,build 时间长 (~10 分钟,因为 1.5M 行代码);第二,需要 autotools (autoconf, automake);第三,可配置选项多 (`--with-native-compilation`, `--with-tree-sitter` 等,Module 0 学过)。Build 完后 `./src/emacs` 是你编译的版本,可以 `-Q` 测试。

**关键**: 你不能 push 到 Savannah,除非你有 commit 权限 (只有 maintainer 有)。贡献流程是: patch 通过邮件发到 `bug-gnu-emacs@gnu.org`,maintainer review 后如果接受,他们 commit。这和 GitHub PR 模式完全不同。

### 4.3 改代码

- 改 .c (C core) 或 .el (Lisp)
- 加测试 (in `test/`)
- 跑测试

**Emacs 代码风格**非常严格——docstring 第一行必须以句号结尾、参数全部大写、commit message 格式固定。读 `CONTRIBUTE` (Emacs 源码根目录的文件) 看全部规则。Maintainer 不会合**风格不对**的 patch,即使功能正确。

### 4.4 提 patch

最简单的方式是用 Emacs 内置的命令:

> ```
> M-x report-emacs-bug
> ```

它会问你 bug 描述,然后帮你格式化邮件 (包括 Emacs 版本、loaded packages、系统信息),自动发到 bug-gnu-emacs@gnu.org 并 cc emacs-devel。如果你 attach patch,它会附上。

或者用 git 命令行 (更灵活):

> ```bash
> git format-patch -1 HEAD
> git send-email --to=bug-gnu-emacs@gnu.org 0001-*.patch
> ```

`git format-patch` 把最新 commit 转成 mbox 格式的 patch 文件 (0001-...patch)。`git send-email` 把这个文件作为邮件正文发送——maintainer 用 `git am` 一键应用。这是 git 的原生 workflow,比 GitHub PR 更古老也更纯粹。

### 4.5 copyright assignment

贡献量大时,FSF 会要求签 paper。一次性,后续贡献不再需要。详见 README.md 的解释——把版权转给 FSF 让其能法律执行 GPL。

---

## 5. 自测

1. GC 是什么算法?
2. gap buffer 干啥?
3. native comp 强在哪?
4. 怎么改 C core?
5. 怎么提 patch?

**答案**:
> 1. mark-and-sweep
> 2. buffer 内的 gap,允许 O(1) 插入
> 3. 编译到原生机器码
> 4. clone emacs.git,改 src/*.c,make
> 5. git format-patch + git send-email 到 bug-gnu-emacs

---

## 6. 下一步

进入 `reading-source.md`。
