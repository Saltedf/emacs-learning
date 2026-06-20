# Reading Source Code

> 学完这个文件,你能高效读任何 Elisp 源码

读源码是 Module 8 的**核心技能**。所有开源贡献都从读别人的源码开始——你不读懂就改不出有用的东西。但读源码和读小说不一样,它需要一套方法论,从入口往里挖,而不是从头到尾扫一遍。

这个文件给你完整的方法论 + 推荐阅读清单。

---

## 1. 为什么要读源码

### 1.1 第一性原理

Emacs 是**自文档化**的——这是 1980 年代 Lisp Machine 时代的传统,Emacs 把它发挥到极致。`C-h f` 显示任何函数的 docstring,点击 `file.el` 链接直接跳源码。其他编辑器做不到这点——VS Code 给你看 docstring,但不给你看源码 (除非安装额外插件)。

但 docstring **有限**——它告诉你"做什么",不告诉你"怎么做"。复杂的逻辑、性能优化、边界情况,只能读源码看到。

读源码让你:

- **深入理解**——docstring 只讲 API,源码讲实现。一个函数的 docstring 可能 50 字,实现可能 200 行。读 200 行才能真正懂。
- **学大师写法**——你装的包都是顶级 Lisp 黑客写的。读 Carsten Dominik (Org)、Jonas Bernoulli (Magit)、Daniel Mendler (Vertico) 的代码,等于在上大师课。
- **找灵感**——你的包可以怎么写?别人的代码给你答案。
- **修 bug**——只有读源码才能定位 bug 在哪。
- **提 PR**——你必须读懂才能贡献。

### 1.2 比阅读书强

书讲概念,源码讲实践。两者的差距巨大:

- 书里讲 `defmacro`,可能给 3 个简单例子。源码里的 `defmacro` (比如 `use-package`) 是几千行的真实工程,处理边界、组合、性能。
- 书里讲 hook,简单示例。源码里的 hook (比如 Org 的 `org-capture-before-finalize-hook`) 配合 capture 的整个流程,你能看到 hook 在真实系统的位置。

你装的所有包的源码都在 `~/.config/emacs/elpa/`。**免费**的代码库——你不需要 clone GitHub,直接打开就能读。这是 Emacs 生态的独特优势。

---

## 2. 找源码

### 2.1 内置函数

> ```
> C-h f find-file RET
> ```

这个命令显示函数的 docstring + 文件位置。最关键的是文件位置——它会显示 "in files.el" 或 "in C source code",点链接直接跳。

对于 C source,Emacs 默认可能不显示——你需要 `find-function-C-source-directory` 指向 Emacs 源码目录。但 Emacs 30+ 默认会自动下载 source (通过 package 系统)。

### 2.2 装的包

> ```
> M-x find-library RET foo RET
> ```

`find-library` 找名为 foo 的 .el 文件 (按 `load-path` 搜索),打开它。这是读装包的标准入口。

或者直接看目录:

> ```bash
> ls ~/.config/emacs/elpa/foo-*/
> ```

每个包有版本号后缀 (foo-1.2.3/),所以用 `*/` 通配。

### 2.3 包未装

去 GitHub:
- 直接搜
- 用 `M-x package-install-file RET URL RET` (有些包支持从 URL 安装)

或 MELPA:
- https://melpa.org/#/PACKAGE

MELPA 页面有"Source"链接指向 GitHub/GitLab。clone 下来读。

---

## 3. 跳定义

### 3.1 xref

`xref` 是 Emacs 27+ 的标准"代码导航"系统。它统一了以前各种 jump-to-definition 的包 (gtags、cscope、ede)。

> ```
> M-.    xref-find-definitions
> M-?    xref-find-references
> M-,    xref-pop-marker-stack
> ```

光标在符号上 `M-.`——跳到定义。`M-?`——找所有引用。`M-,`——回到跳之前的位置 (marker stack)。

对 Elisp,`xref` 默认就能用——它知道怎么读 Emacs 的 symbol table。对其他语言 (Python、JS),你需要 LSP server。

### 3.2 dumb-jump (备用)

如果 xref 不工作 (没 LSP),`dumb-jump` 用 grep 做 fallback——速度慢但不挑语言,几乎所有语言都能用。

> ```elisp
> (use-package dumb-jump
>   :ensure t
>   :init (add-hook 'xref-backend-functions #'dumb-jump-xref-activate))
> ```

这个配置把 dumb-jump 注册为 xref 的一个 backend。当其他 backend (LSP) 不工作时,xref 自动 fallback 到 dumb-jump。

### 3.3 etags (老式)

etags 是 1980 年代的工具,生成 TAGS 索引文件,Emacs 用它做跳转。老程序员都熟悉。

> ```bash
> cd ~/project
> find . -name "*.el" | etags -
> ```

这个命令找所有 .el 文件,生成 TAGS 文件。Emacs 自动用它 (`visit-tags-table`)。`M-.` 在 etags 启用时也能用。

etags 现在用得少 (xref 替代了),但在没有 LSP/grep 的情况下 (比如读老代码),还是有用。

---

## 4. 读源码的策略

### 4.1 从入口开始

**不要从头读 .el 文件**——那是写作顺序,不是阅读顺序。从**你常用**的命令开始:

> ```
> C-h f magit-status RET
> 点击 magit.el RET
> 读 magit-status 函数
> ```

`magit-status` 是 Magit 的核心入口,你每次开 Magit 都用它。从这里读起,你能立刻理解"开 Magit 时发生了什么",而不是从头看 5000 行代码不知道哪部分重要。

### 4.2 跟踪调用链

> ```
> magit-status
>   ↓
> magit-status-internal
>   ↓
> magit-generate-buffer-name
>   ↓
> ...
> ```

跟几层,理解整体。用 Edebug (`C-u C-M-x` instrument 一个函数) 单步执行,看真实调用顺序——比静态读 10 倍快。

跟踪到一定深度,你会发现"够用了"——再深就是细节。学的是架构,不是每行代码。

### 4.3 看 tests

测试是**用法文档**——比 docstring 更具体。

> ```
> test/magit-tests.el
> ```

每个 test 显示函数**怎么用**——参数、返回值、副作用。如果你看不懂 docstring,看 test 就懂了。

Test 还告诉你**正确行为**——当行为变时,test 必须改。所以 test 是"事实来源"。

### 4.4 看 Commentary

`.el` 文件开头的 `;;; Commentary:` 是高层概述——作者自己写"这个包是干啥的"。这是包的"README 在源码里"的形式。

Commentary 通常比 docstring 更高层——它讲包的设计理念,不讲具体函数。读 Commentary 是开始读包的第一步。

---

## 5. 读源码工具

### 5.1 imenu

> ```
> M-g i RET NAME RET
> ```

imenu 跳到当前 buffer 的某个函数 / 变量 / 类定义。它根据当前 major mode 的规则识别"什么是定义"——elisp-mode 识别 `defun`、`defvar`,python-mode 识别 `def`、`class`。

imenu 是读长文件最快的方式——`M-g i` 输入函数名,一秒跳过去。

### 5.2 occur

> ```
> M-s o defun RET
> ```

occur 列所有匹配正则的行,在一个新 buffer 显示。`M-s o defun RET` 列所有 `defun`,你能浏览整个文件的结构。

比 imenu 更灵活——imenu 只识别"定义",occur 能匹配任何东西 (注释、字符串、特定模式)。

### 5.3 consult-imenu

(Module 5 学过)

> ```
> M-g i
> ```

`consult-imenu` 是 imenu 的增强——它用 consult 的 completion UI,可以模糊搜索 + 预览。读大文件必备。

### 5.4 elisp-slime-nav / helpful

`helpful` 包增强 `C-h f`:

> ```elisp
> (use-package helpful
>   :bind (("C-h f" . helpful-callable)
>          ("C-h v" . helpful-variable)
>          ("C-h k" . helpful-key)
>          ("C-h ." . helpful-at-point)))
> ```

这个配置把 `C-h f` 替换为 `helpful-callable`。helpful 显示更丰富:
- **完整源码** (不只位置)
- **调用历史** (函数最近被谁调用)
- **引用列表** (函数在哪里被引用)
- **文档的源码位置** (docstring 在 .el 的第几行)

helpful 是读源码的瑞士军刀——装上就再也不用 `C-h f` 原版了。

### 5.5 srefactor / lispy

重构 / 结构化编辑。 (Module 5+ 可选)

---

## 6. 推荐读的源码

### 6.1 入门级 (小,清晰)

从小的开始,建立信心。下面这些包都是几千行,代码清晰,适合学习:

- **use-package**: 宏,~ 1000 行——学宏的最佳教材
- **which-key**: minor mode,~ 1500 行——学 minor mode 和 post-command-hook
- **try**: 极简,~ 200 行——一个晚上读完,最小包的样子
- **crux**: 实用工具集,~ 500 行——简单的辅助函数集
- **diff-hl**: fringe 显示 git diff,~ 700 行——学 fringe 和 VC 集成

### 6.2 中级

库类包,代码密集但单一职责:

- **dash.el**: 函数式 list 库,~ 1500 行——学 anaphoric macro (it, acc)
- **s.el**: 字符串库,~ 1000 行——学字符串处理的 idiom
- **f.el**: 文件操作库,~ 800 行——文件 API 的 wrapper
- **ht.el**: hash table 库,~ 600 行——学 hash table 的 functional 接口

这些库都是"基础设施"——很多包依赖它们,读懂它们等于读懂半个 Emacs 生态。

### 6.3 高级 (大,复杂)

大项目,复杂但价值高:

- **magit**: Git 客户端, ~ 30000 行——Emacs 最大的包之一,Jonas Bernoulli 的 magnum opus
- **org-mode**: 大型, ~ 50000 行——Carsten Dominik 设计的文档系统
- **lsp-mode**: LSP, ~ 30000 行——LSP 协议的 Emacs 实现
- **vertico** + **corfu** + **consult**: 现代补全——Daniel Mendler 的极简设计,代码紧凑

不要一开始就读 magit——你会迷路。先读完入门级和中级,再来挑战。

### 6.4 Emacs core

Emacs 自带的 Lisp 代码,在 `lisp/` 目录:

- `simple.el`: 基础命令 (kill-region, yank, ...)——最常用的命令的实现
- `files.el`: find-file, save-buffer——文件操作核心
- `buffer.el`: buffer 操作
- `window.el`: window
- `ibuffer.el`: ibuffer
- `dired.el`: dired
- `isearch.el`: isearch
- `font-lock.el`: font-lock

读这些等于读"教科书级别的 Elisp"——50 年的迭代让代码质量极高。

### 6.5 C core

`src/`:

- `eval.c`: eval——Lisp 解释器本身
- `alloc.c`: GC——内存管理

C core 比 Lisp 难读——C 风格、性能优化、宏魔法。但读 eval.c 能让你真正理解 Elisp 怎么工作。

---

## 7. 读源码方法

### 7.1 主动读

不要被动浏览——边读边问问题。比如"Magit 怎么 stash?",用 grep / consult-ripgrep 找 `stash`,读相关函数。这种"问题驱动"的阅读比"线性扫"高效 10 倍。

### 7.2 实验读

边读边改:
- 复制函数到 `*scratch*`
- 改一点
- eval,看行为

实验是验证理解的最佳方式——你以为函数怎么工作,改一下看实际行为,差距就是你的盲区。

### 7.3 复述

读完一个函数,关掉源码,用自己的话复述,写下逻辑。复述不出来 = 没真懂。这是费曼学习法——能教别人才是真懂。

### 7.4 模仿

读到一个好的实现,模仿到自己代码。注明来源 ("inspired by magit's `magit-with-blob`")——这是开源礼仪,也是法律要求 (GPL 要求 attribution)。

### 7.5 改进

发现 bug 或可改进,fork,改,提 PR。读源码的终极产出就是 PR——你读懂了,你就能贡献。

---

## 8. 创造性读法

下面这些"意外用法"是高级 Emacs 用户的读源码技巧,每个都能打开一扇新门:

### 8.1 从 bug 找入口

看到报错 `void-variable foo`,grep 源码找 `foo` 的 `defvar`。Bug 是入口——它告诉你"这里有代码",顺着 bug 能挖到深层。

### 8.2 从 feature 学实现

Org 的 capture 怎么实现的?读 `org-capture.el`。从你用的 feature 反向追到实现——这是最高效的学习路径,因为你已经懂"它做什么",只剩"怎么做"。

### 8.3 从 test 学 API

test 文件是最好的用法文档——比 docstring 具体,比 README 准确。`test/org-test.el` 显示 org 函数的所有用法。

### 8.4 用 Edebug 跑 magit

instrument `magit-status`,看它内部干啥。Edebug 单步执行,你能看到每一步的变量值——比静态读 100 倍清楚。

### 8.5 看 git blame

谁加的这行?为啥? `magit-blame` (或 `vc-annotate`) 显示每行的作者和 commit。读 commit message 能理解"为什么这样写"——很多时候代码看着怪,blame 一看就懂 (原来是为了修某个 bug)。

### 8.6 看 commit log

项目演化历史。`git log --oneline | head -50` 看最近 commits,理解项目在做什么。`git log --stat` 看每个 commit 改了哪些文件,理解代码结构演化。

### 8.7 fork + 实验

改一下,看效果,学最快。比如"如果我把这个 hook 改了会怎样?"——直接改,跑,看结果。改坏了 reset 就行,git 保护你。

### 8.8 抄代码

好的实现直接借鉴 (注意 license)。GPL 兼容 GPL,BSD 兼容 BSD——抄之前看 license。如果是 MIT/BSD,可以抄到任何项目;如果是 GPL,只能抄到 GPL 项目。

### 8.9 看 etc/NEWS

每个 Emacs release 的 NEWS 文件列出所有变化——读 NEWS 能学到"Emacs 最近加了什么",这些是新特性,文档少,源码是唯一参考。

### 8.10 看 README 的 "Alternatives" 段

很多包的 README 列"alternatives"——比如 vertico 列 ivy、helm。读这些对比能理解设计取舍——为什么作者写新包,而不是改老的。

---

## 9. 实战

### Ex 1: 读 use-package

> ```bash
> ls ~/.config/emacs/elpa/use-package-*/
> ```

读 `use-package.el`:
- 看 `;;; Commentary:`——作者讲 use-package 是干啥的
- 看 `defmacro use-package`——核心宏,几千行
- 跟踪 `:ensure` 处理——`:ensure foo` 怎么变成 `package-install 'foo`

读完你会理解 use-package 不只是"语法糖",而是一个完整的 DSL (Domain Specific Language)。

### Ex 2: 读 vertico

读 `vertico.el`:
- minor mode 定义——`define-minor-mode` 的标准用法
- post-command-hook——每次命令后更新候选
- 候选显示——怎么在 minibuffer 下面画候选列表

Vertico 代码紧凑 (1500 行),是学 modern Emacs 包设计的最佳样本。

### Ex 3: 读 simple.el

> ```elisp
> (find-library "simple.el")
> ```

读 `kill-region`:
- 参数处理——`(beg end &optional region)` 的各种组合
- 调用 `kill-new`——把文本加入 kill-ring
- 设 `interprogram-cut-function`——同步到系统 clipboard

读完你会理解 kill/yank 的完整流程,不只是"复制粘贴"。

### Ex 4: 跟踪 find-file

> ```
> C-h f find-file RET
> 点击 files.el RET
> M-. find-file
> ```

跟几层,看完整流程。`find-file` 调用 `find-file-noselect`,后者调用 `find-file-literally`,层层往下。跟踪到底,你会理解 Emacs 的文件加载机制。

### Ex 5: grep Emacs 源码

> ```bash
> git clone https://git.savannah.gnu.org/git/emacs.git
> cd emacs
> rg "defun kill-region" lisp/
> ```

`rg` (ripgrep) 比 grep 快 10 倍,且默认递归。clone 整个 Emacs 源码后,你可以 grep 任何东西——这是研究 Emacs 内部的标准方法。

---

## 10. 自测

1. 怎么跳定义? (答: `M-.` xref-find-definitions)
2. 怎么找反向引用? (答: `M-?` xref-find-references)
3. 推荐先读哪 3 个包? (答: use-package, which-key, try —— 小的开始)
4. 内置 simple.el 在哪? (答: C source 或 lisp/simple.el,`M-x find-library`)
5. 怎么读 magit 源码? (答: 装后看 ~/.config/emacs/elpa/magit-*/ 或 git clone magit/magit)

---

## 11. 下一步

进入 `first-pr.md` 提第一个 PR。
