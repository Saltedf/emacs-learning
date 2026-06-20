# Project.el + Completion Stack 详解

> 学完这个文件,你能用 Emacs 高效处理大型项目

这一节讲两件事: project.el (识别和管理项目) 和补全栈 (高效查找/切换)。它们是 Emacs "power user" 的基础设施——配置好后,你的每一次找文件、切 buffer、跨项目搜索都比以前快几秒。这种"看不见的提速"累积起来,是 Emacs 用户高生产力的来源。

---

## 1. project.el (内置)

### 1.1 什么是 project

现代 IDE 都有"项目"概念——打开一个项目,IDE 知道所有文件、提供跳转、补全、搜索的"项目级"能力。Emacs 27+ 内置了 `project.el` 来提供这种能力。

Emacs 27+ 内置 `project.el`。
识别"项目"的方法:
- 找 `.git` / `.hg` 等版本控制目录
- `project-vc-extra-root-markers` (Emacs 30+) 自定义标记

project.el 的核心思想是"项目边界"——一个目录树如果包含 `.git` (或其他版本控制标记),它就是一个项目。project.el 把这个目录树作为"项目根",所有 project 命令都在这个范围内工作。

为什么需要"项目边界"? 因为 Emacs 默认是"buffer 级"的——每个 buffer 独立。在大型项目里,你想"找项目内的文件"、"项目内 grep"——这需要知道项目范围。project.el 提供这个范围。

### 1.2 基础命令

project.el 的命令都在 `C-x p` 前缀下——这是 Emacs 30+ 的官方约定:

```
C-x p f    project-find-file        ; 找文件 (含 fuzzy)
C-x p F    project-or-external-find-file
C-x p g    project-find-regexp      ; 项目内 grep
C-x p b    project-switch-to-buffer ; 项目内 buffer
C-x p k    project-kill-buffers     ; 关项目所有 buffer
C-x p D    project-dired            ; 项目根 dired
C-x p v    project-vc-dir           ; magit 或 vc-dir
C-x p !    project-shell-command
C-x p &    project-async-shell-command
C-x p e    project-eshell
C-x p s    project-shell
```

最常用的是 `C-x p f` (find-file) 和 `C-x p g` (grep)。`C-x p f` 是"项目内 fuzzy 找文件"——你输入文件名一部分,实时过滤。`C-x p g` 是"项目内 grep"——找包含某字符串的文件。

这些命令和"非项目"的对应命令 (`C-x C-f`, `M-x grep`) 区别在于**范围**——非项目命令在整个文件系统工作,项目命令限定在当前项目。这让结果更相关 (不会列出系统文件),速度更快 (扫描更少文件)。

### 1.3 切换项目

当你有多个项目时 (比如同时维护 5 个仓库),需要快速切换:

```
C-x p p    project-switch-project
```

显示最近项目,选一个切。

`C-x p p` 弹出最近的项目列表——你选一个,project.el 记住"当前项目"。之后 `C-x p f` 等命令在新项目里工作。

project.el 自动维护项目列表 (在 `project-list-file` 里),所以你不需要手动注册——只要用 `C-x p f` 打开过一次的项目,下次切换时就在列表里。

### 1.4 配置

这是一个完整的 project.el 配置:

```elisp
(use-package project
  :ensure nil                      ; 内置
  :bind (("C-c p f" . project-find-file)
         ("C-c p g" . project-find-regexp)
         ("C-c p b" . project-switch-to-buffer)
         ("C-c p p" . project-switch-project))
  :custom
  (project-switch-commands 'project-find-file)
  (project-vc-extra-root-markers '(".project" "package.json" "Cargo.toml"))
  (project-list-file "~/.config/emacs/projects"))
```

每个设置都值得解释:
- `:ensure nil`: project.el 是内置,不需要从 ELPA 装。
- `C-c p` 前缀: 这是社区约定的"项目命令"前缀,和官方 `C-x p` 互补——很多人习惯 `C-c p` 因为更容易按。
- `project-switch-commands`: 切换项目后默认做什么——`project-find-file` 意思是切换后立刻弹"找文件"。其他选项: `project-dired` (打开 dired)、`magit-status` (打开 magit)。
- `project-vc-extra-root-markers`: 除了 `.git`,这些文件也作为项目标记——`package.json` (Node)、`Cargo.toml` (Rust)、`.project` (空文件,自定义)。这让 project.el 识别非 Git 项目。
- `project-list-file`: 项目列表存储位置——和你的 Emacs 配置一起,便于备份。

### 1.5 projectile (替代,更老但流行)

projectile 是 project.el 的"前辈"——2011 年诞生,比 project.el 早 8 年。它功能更多,生态更成熟 (几乎所有 Emacs 包都集成 projectile),但 project.el 是内置且越来越好。

```elisp
(use-package projectile
  :ensure t
  :diminish projectile-mode
  :init (projectile-mode 1)
  :bind (:map projectile-mode-map
              ("C-c p" . projectile-command-map))
  :custom
  (projectile-completion-system 'default)
  (projectile-project-search-path '("~/projects")))
```

projectile 比 project.el 功能多,但 project.el 内置且越来越好。
新人推荐 project.el。

为什么推荐 project.el? 几个原因:
1. **内置**——不依赖第三方包,Emacs 升级时跟着升级。
2. **简单**——API 更小,更易理解。
3. **官方支持**——Emacs 团队维护,长期稳定。
4. **未来**——所有新 Emacs 功能都基于 project.el。

只有当你需要 projectile 的特殊功能 (比如缓存策略、特定项目类型识别) 时才用 projectile。

---

## 2. Vertico + Orderless + Consult + Marginalia + Embark

### 2.1 这套组合的角色

现代 Emacs 补全栈是一个被精心设计的组合——五个包,每个做一件事:

| 包 | 角色 |
|---|---|
| **Vertico** | Minibuffer 垂直显示候选 |
| **Orderless** | 匹配算法 (空格分隔,任意顺序) |
| **Consult** | 高级命令 (consult-line, consult-buffer 等) |
| **Marginalia** | 候选旁边显示元信息 |
| **Embark** | 从候选拉出菜单 (act on) |

这五个包加起来 < 3000 行 Elisp,但替代了 Helm (一万行) / Ivy (几千行) 这种"大而全"包。

为什么这种设计更好? 因为每个包可以独立升级,独立维护。这五个包的作者有重叠——Daniel Mendler (vertico/corfu/orderless/embark 作者) 是核心贡献者,他的哲学是"小而美"。每个包 < 500 行,纯 Elisp,无依赖。这反映了 Emacs 社区的"工具哲学": **工具应该透明、可组合、可替换**。

对比 Helm——Helm 是一个大包,UI/命令/source 都打包。优点是开箱即用,缺点是难定制。如果你不喜欢 Helm 的某个细节,改它意味着读大量代码。而 Vertico 的 UI 简单到你可以读完——整个 vertico.el 大约 500 行。

### 2.2 Vertico

Vertico 是这套栈的"显示层"——它把 minibuffer 的候选从"水平 + 补全"改为"垂直 + 实时过滤":

```elisp
(use-package vertico
  :ensure t
  :init
  (vertico-mode 1)
  :custom
  (vertico-cycle t)
  (vertico-resize 'grow)
  :bind (:map vertico-map
              ("RET" . vertico-exit)
              ("TAB" . vertico-insert)))
```

配置非常简单——只需 `(vertico-mode 1)`。一旦启用,所有 minibuffer 补全 (`M-x`、`find-file`、`switch-to-buffer`) 都变成垂直列表。

效果: `M-x` 输入命令时,候选**垂直列表**显示。你输入字符,列表实时过滤。你用 `C-n`/`C-p` 或方向键选,RET 执行。这是 1990 年代就有的"垂直补全"概念,但 Vertico 把它做得很优雅——极简、无干扰、可配置。

`vertico-cycle t`: 在列表首尾循环——按 `C-p` 在第一个候选时跳到最后,反之亦然。这避免"按多了到顶/底"的烦恼。

`vertico-resize 'grow`: minibuffer 高度根据候选数量增长——一开始只显示几行,候选多了自动扩大。

### 2.3 Orderless

Orderless 是"匹配算法"——它替代了 Emacs 默认的"前缀匹配",改为"空格分隔,任意顺序":

```elisp
(use-package orderless
  :ensure t
  :custom
  (completion-styles '(orderless basic))
  (completion-category-defaults nil)
  (completion-category-overrides
   '((file (styles partial-completion)))))
```

输入 `ins set` 匹配 `insert-setting` (空格分隔,任意顺序)。

这个能力很难用语言描述,但用一次就回不去。在默认前缀匹配下,你输入 "is" 找 `insert-setting`,候选包括所有 "is..." 开头的——可能有几百个。Orderless 让你输入 "is set" (空格分隔),立刻只匹配 `insert-setting` (因为同时含 "is" 和 "set")。

`completion-styles '(orderless basic)`: 先用 orderless 匹配,如果没结果用 basic。basic 是前缀匹配,作为 fallback。

`completion-category-overrides`: 特定类别用不同 style——`file` 用 `partial-completion`,允许 `~/.c/em/in.el` 缩写匹配 `~/.config/emacs/init.el`。这是另一个提速点——长路径用首字母缩写。

### 2.4 Consult

Consult 是"命令包"——它提供"现代化"版本的内置命令,带 preview、过滤、跨源聚合:

```elisp
(use-package consult
  :ensure t
  :bind (;; 搜索
         ("C-s" . consult-line)                ; 替代 isearch
         ("C-r" . consult-history)
         ;; Buffer / 文件
         ("C-x b" . consult-buffer)            ; 智能切 buffer
         ("C-x B" . consult-buffer-other-window)
         ;; Yank
         ("M-y" . consult-yank-pop)            ; kill-ring 选择
         ;; Goto
         ("M-g g" . consult-goto-line)
         ("M-g M-g" . consult-goto-line)
         ("M-g i" . consult-imenu)
         ("M-g r" . consult-ripgrep)
         ;; Search
         ("M-s g" . consult-grep)
         ("M-s r" . consult-ripgrep)
         ("M-s f" . consult-find)
         ("M-s l" . consult-locate)
         ;; 项目
         ([remap project-find-regexp] . consult-ripgrep)))
```

Consult 把内置命令包成"现代化"版本 (带 preview)。

preview 是 Consult 的杀手锏——当你在候选列表上下移动时,目标 buffer 实时显示对应内容。比如 `consult-line` (搜当前 buffer 的行),你按 `C-s` 输入 pattern,候选显示所有匹配的行——光标移到候选,buffer 自动跳到那一行,你能看到上下文。这是"isearch 不带预览"做不到的。

`consult-buffer` 的另一个特性是"多源"——一个列表同时显示 buffer、recent files、bookmarks。你不需要分别 `switch-to-buffer`、`recentf`、`bookmark-jump`——一个 `C-x b` 全搞定。

### 2.5 Marginalia

Marginalia 是"注释层"——它在候选旁边显示元信息:

```elisp
(use-package marginalia
  :ensure t
  :init
  (marginalia-mode 1)
  :custom
  (marginalia-align 'right))
```

配置后,所有补全候选旁边都有注释。注释的内容根据候选类型不同:

```
find-file: init.el
           500 lines, Lisp file
find-file: README.md
           148 lines, Markdown file
```

这看起来简单,但极大提升了"决策速度"。找文件时,你立刻知道每个文件多大、什么类型——不需要打开看。`switch-to-buffer` 时,知道每个 buffer 的 mode——立刻找到那个 Python buffer。`M-x` 时,知道每个命令的 keybinding——找到没绑定的命令。

Marginalia 体现了"信息密度"的价值——一行注释可以省一次"打开看"的操作。

### 2.6 Embark

Embark 是"上下文菜单"——它让你对候选**做更多事**,不只是选:

```elisp
(use-package embark
  :ensure t
  :bind (("C-." . embark-act)               ; 在候选上拉菜单
         ("M-." . embark-dwim)
         ("C-h B" . embark-bindings)))

(use-package embark-consult
  :ensure t
  :after (embark consult))
```

在候选上 `C-.`,弹出菜单 (open / copy / delete / ...)。

Embark 的核心是 `embark-act` (默认 `C-.`)——它在当前 candidate 上弹一个菜单,菜单选项根据 candidate 类型不同:
- **buffer 候选**: switch / kill / rename / display
- **file 候选**: open / copy / delete / magit-status / dired
- **command 候选**: execute / describe / bind to key

比如 `consult-buffer` 列出 buffer,你在某个 buffer 候选上按 `C-.`——弹出菜单让你 kill/rename/dired。这就是"批量操作"的工作流: 一次 consult,多个操作。

`embark-dwim` (M-.): "do what I mean"——执行最可能的操作 (file = open, buffer = switch)。这是快速路径。

---

## 3. Corfu (代码补全)

### 3.1 安装

Corfu 是"代码补全 UI"——它和 Vertico 是兄弟项目 (同一作者 Daniel Mendler),设计哲学一致: 极简、与内置功能集成、不重新发明轮子。

```elisp
(use-package corfu
  :ensure t
  :init
  (global-corfu-mode 1)
  :custom
  (corfu-auto t)                           ; 自动弹出
  (corfu-auto-delay 0.2)
  (corfu-auto-prefix 2)
  (corfu-cycle t)
  (corfu-quit-no-match 'separator)
  :bind (:map corfu-map
              ("TAB" . corfu-next)
              ([tab] . corfu-next)
              ("S-TAB" . corfu-previous)
              ([backtab] . corfu-previous)
              ("RET" . nil)))              ; RET 不选,正常换行
```

关键设置:
- `corfu-auto t`: 自动弹出——输入字符时自动显示候选。如果设为 nil,需要手动触发 (`M-TAB` 或 `C-M-i`)。
- `corfu-auto-delay 0.2`: 输入后等 0.2 秒才弹——避免每按一键就弹,影响体验。
- `corfu-auto-prefix 2`: 至少 2 个字符才弹——单字符补全太多候选,意义不大。
- `corfu-cycle t`: 候选循环——TAB 在最后一个时跳到第一个。
- `corfu-quit-no-match 'separator`: 没匹配时按 separator (空格等) 关闭。
- `RET` 设为 nil: **这是关键设计**——RET 应该换行,不应该选补全。如果让 RET 选,你在写代码时按 RET 想换行,但补全弹出来了,变成选了候选——非常烦人。让 TAB 选,RET 换行,这是社区约定。

### 3.2 Cape (补全 source)

Cape 是 "Completion At Point Extensions"——它提供各种补全 source:

```elisp
(use-package cape
  :ensure t
  :init
  (add-to-list 'completion-at-point-functions #'cape-dabbrev)  ; buffer 词
  (add-to-list 'completion-at-point-functions #'cape-file)     ; 文件路径
  (add-to-list 'completion-at-point-functions #'cape-keyword)  ; 语言关键字
  (add-to-list 'completion-at-point-functions #'cape-abbrev))
```

Cape 的设计哲学是"补全 source 应该可叠加"——`completion-at-point-functions` 是一个列表,Corfu 按顺序尝试每个 source。这意味着你可以**同时**用:
- LSP (eglot 自动加)——提供函数、变量、类型
- `cape-dabbrev`——当前 buffer 的词 (适合写文档)
- `cape-file`——文件路径 (在字符串里)
- `cape-keyword`——语言关键字

一个键位 (TAB),多个 source 协作。这是和 Company 的核心差异——Company 有自己的 backend 系统,Corfu 用内置的 completion-at-point。这让 Corfu 更轻、更与 LSP 集成。

### 3.3 Company (替代)

如果习惯 company,类似:

```elisp
(use-package company
  :ensure t
  :init (global-company-mode 1)
  :custom
  (company-idle-delay 0.2)
  (company-minimum-prefix-length 2))
```

现代人**推荐 Corfu**,更轻量,与内置 completion-at-point 配合好。

为什么推荐 Corfu 而不是 Company? 几个原因:
1. **与内置集成**: Corfu 用 completion-at-point,这是 Emacs 24+ 的标准接口。LSP (eglot)、cape 都通过这个接口工作。
2. **极简**: Corfu 代码 < 500 行,Company 是几千行。简单意味着好维护。
3. **性能**: Corfu 更快——它不重新发明 UI,用 child frame / overlay 显示。
4. **未来**: Daniel Mendler 的设计哲学是 Emacs 的方向。

Company 仍然是优秀的包——如果你已经在用,没必要换。但新人推荐 Corfu。

---

## 4. Eglot (LSP)

### 4.1 安装

Eglot 是 Emacs 29+ 内置的 LSP client。它的设计哲学和 Magit 相反——Magit 是大而专,Eglot 是极简而集成。Eglot 不发明新概念,只是把 LSP 协议接到 Emacs 内置功能上。

Emacs 29+ 内置。

```elisp
(use-package eglot
  :ensure nil
  :hook ((python-mode . eglot-ensure)
         (js-mode . eglot-ensure)
         (typescript-mode . eglot-ensure)
         (rust-mode . eglot-ensure)
         (c-mode . eglot-ensure)
         (c++-mode . eglot-ensure))
  :custom
  (eglot-autoshutdown t)
  (eglot-events-buffer-size 0)
  :bind (:map eglot-mode-map
              ("C-c l r" . eglot-rename)
              ("C-c l a" . eglot-code-actions)
              ("C-c l f" . eglot-format-buffer)
              ("C-c l d" . eldoc)))
```

`eglot-ensure` 是核心——它在进入对应 mode 时自动启动 LSP server。你**不需要手动** `M-x eglot`——打开 .py 文件,pyright 自动启动。

`eglot-autoshutdown t`: 关闭最后一个 buffer 时,自动关闭 LSP server——避免进程泄漏。

`eglot-events-buffer-size 0`: 关闭 events buffer——默认 eglot 会记录所有 LSP 通信,占内存。设为 0 关掉 (调试时再开)。

### 4.2 用法

进入匹配 mode,自动启动 LSP server。
- hover 显示 docstring (`eldoc`)
- `M-.` 跳定义 (xref)
- `M-?` find references
- `M-x eglot-rename` 重命名
- `M-x eglot-format-buffer` 格式化
- `M-x eglot-code-actions` code actions

这些功能不是 eglot 自己实现的——它们都是 Emacs 内置功能 (xref、eldoc、flymake),eglot 只是把 LSP 的数据接到这些功能上。这是 eglot 的核心设计——**不重复造轮子**。

比如 `M-.` (xref-find-definitions) 是 Emacs 25+ 的标准跳转键。eglot 启动后,xref 自动用 LSP 的数据——你按 `M-.`,eglot 问 LSP server "定义在哪",server 返回位置,xref 跳过去。整个体验和无 LSP 时的 `M-.` 一模一样——这是"无缝集成"。

### 4.3 LSP servers

eglot 是 client,你还需要 server。下面是常见语言的 server 和装法:

| 语言 | Server | 装法 |
|---|---|---|
| Python | pyright | `npm install -g pyright` |
| JS/TS | typescript-language-server | `npm install -g typescript-language-server typescript` |
| Rust | rust-analyzer | `rustup component add rust-analyzer` |
| C/C++ | clangd | `apt install clangd` |
| Go | gopls | `go install golang.org/x/tools/gopls@latest` |
| Java | Eclipse JDT LS | (复杂,用 lsp-java 包) |
| Ruby | solargraph | `gem install solargraph` |

装好后,eglot 自动检测——它内置了"哪个 mode 用哪个 server"的映射。如果 server 在 PATH 里,打开对应文件,server 自动启动。

### 4.4 Flymake (诊断)

Eglot 用 flymake 显示 diagnostic——这是 Emacs 内置的"实时诊断"框架:

```elisp
(use-package flymake
  :ensure nil
  :bind (:map flymake-mode-map
              ("M-n" . flymake-goto-next-error)
              ("M-p" . flymake-goto-prev-error)
              ("C-c ! l" . flymake-show-buffer-diagnostics)))
```

Flymake 在 buffer 里高亮错误 (默认红色波浪线),fringe 显示标记。`M-n`/`M-p` 在错误间跳——这是"代码审查"工作流的核心。

注意 flymake 和 flycheck 的区别——flycheck 是第三方包,历史更久。但 flymake 是内置,且 Emacs 28+ 后功能追上了。eglot 用 flymake (设计选择),所以推荐 flymake。如果你用 lsp-mode,它可能默认用 flycheck。

---

## 5. lsp-mode (替代,功能更全)

如果需要更多 LSP 功能 (javaproject, debugger, ...):

```elisp
(use-package lsp-mode
  :ensure t
  :commands (lsp lsp-deferred)
  :hook ((python-mode . lsp-deferred)
         (js-mode . lsp-deferred)
         (rust-mode . lsp-deferred))
  :custom
  (lsp-enable-symbol-highlighting t)
  (lsp-signature-doc-lines 1)
  (lsp-eldoc-render-all nil)
  (lsp-headerline-breadcrumb-enable t))
```

lsp-mode 重一些,功能多。
新人推荐 eglot。

lsp-mode 和 eglot 的本质区别:
- **lsp-mode** 重写了一切——自己的 UI (lsp-ui)、自己的诊断显示、自己的 breadcrumb、自己的 sideline。
- **eglot** 用内置——xref、flymake、eldoc、completion-at-point。

哪种更好? 看你的偏好:
- 想要"开箱即用、功能多、像 VSCode"——lsp-mode。
- 想要"极简、和内置一致、长期稳定"——eglot。

Emacs 29+ 把 eglot 内置,这是官方信号——eglot 的设计哲学是 Emacs 的方向。

---

## 6. 实战

### 任务 1: 项目内查找

```
C-c p p         切到项目
C-c p f         找文件
C-c p g         grep 项目内
```

这是日常工作的核心三步。`C-c p p` 切项目 (一次,持续工作),然后 `C-c p f` 找文件 (重复),`C-c p g` 跨文件搜 (找用法)。

### 任务 2: 跳定义

```
光标在函数名
M-.             eglot xref find definitions
                自动跳到定义文件
M-,             跳回
```

这是 LSP 最常用的功能。`M-.` 跳定义,`M-,` 跳回——xref 维护一个 marker stack,你可以跳多层再回。

### 任务 3: 重命名

```
光标在符号
M-x eglot-rename RET
输入新名字 RET
LSP server 跨文件改所有引用
```

LSP rename 是"语义重命名"——它不是简单的查找替换 (那会改注释、字符串里的同名)。LSP 知道 AST,只改真正的引用。

### 任务 4: 格式化

```
M-x eglot-format-buffer
;; 或绑定到 C-c l f
```

格式化让代码风格一致——LSP server 用语言的官方 formatter (Python 用 black/autopep8,Rust 用 rustfmt)。

### 任务 5: Code actions

```
M-x eglot-code-actions
;; 显示 LSP 提供的 quick fix
```

Code actions 是 LSP 的"快速修复"——比如你 import 缺失,LSP 提供 "Add import" action; 你写了一个 unused var,LSP 提供 "Remove unused" action。这是 IDE 的标配功能,eglot 通过 LSP 获得了它。

---

## 7. Tree-sitter (高级语法高亮)

Tree-sitter 是一个增量解析库——它把代码解析为 AST (抽象语法树),而不是用正则匹配。这让语法高亮更准确、更快、可扩展。

### 7.1 安装

Emacs 29+ 内置 tree-sitter 支持——你只需要装 grammar (每种语言的解析器):

```elisp
(use-package treesit
  :ensure nil                       ; 内置
  :custom
  (treesit-font-lock-level 4))

(use-package treesit-auto
  :ensure t
  :init
  (global-treesit-auto-mode))
```

`treesit-auto` 自动装 grammar——你打开一个 .py 文件,它自动下载 Python 的 grammar (如果有网络)。这让 tree-sitter 设置变成"零配置"。

`treesit-font-lock-level 4`: 最高级别语法高亮——区分更多 token 类型 (函数名 vs 变量、字段 vs 局部变量等)。

### 7.2 用 -tsi 模式

```elisp
(add-to-list 'auto-mode-alist '("\\.py\\'" . python-ts-mode))
(add-to-list 'auto-mode-alist '("\\.rs\\'" . rust-ts-mode))
```

`-ts-mode` 用 tree-sitter 解析,语法高亮更准确。

`python-ts-mode` vs `python-mode` 区别:
- `python-mode`: 用正则匹配 (传统方式)
- `python-ts-mode`: 用 tree-sitter AST

后者更准确——正则容易出错 (比如字符串里的 `def` 被误认为函数定义),tree-sitter 不会。后者也更快——tree-sitter 是增量的,只解析改动的部分。

Tree-sitter 的另一个能力是**结构化编辑**——你可以基于 AST 做"选中函数"、"移动语句"、"重命名局部变量"等操作。这是未来的方向,Emacs 社区在积极开发。

---

## 8. 自测

1. project.el 怎么识别项目?
2. Vertico/Orderless/Consult/Marginalia/Embark 各自角色?
3. Corfu 和 Cape 关系?
4. eglot 和 lsp-mode 区别?
5. Tree-sitter 比正则语法高亮强在哪?

**答案**:
> 1. 找 .git / .hg 等版本控制目录,或 project-vc-extra-root-markers
> 2. Vertico UI,Orderless 算法,Consult 命令,Marginalia 注释,Embark 菜单
> 3. Corfu 是 UI;Cape 提供 source (dabbrev/file/keyword)
> 4. eglot 轻量内置;lsp-mode 功能全社区
> 5. Tree-sitter 是 AST 解析,准确;正则是模式匹配,易错

---

## 9. 下一步

进入 `lsp-eglot.md` 深入学 LSP。
