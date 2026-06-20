# LSP (Eglot) 详解

> 学完这个文件,你能用 Emacs 像现代 IDE 一样开发

LSP (Language Server Protocol) 是现代 IDE 的基础——它让编辑器拥有"理解代码"的能力 (跳定义、查引用、自动补全、诊断)。Emacs 通过 eglot 接入 LSP 生态,这意味着 **Emacs 在 LSP 支持上和 VSCode 平起平坐**。这一节深度讲解 LSP 和 eglot。

---

## 1. LSP 是什么

### 1.1 Language Server Protocol

LSP 是微软 2016 年开源的协议——它把"编辑器 × 语言"的集成问题解决了。在 LSP 之前,每个编辑器要为每个语言单独写集成——Emacs 要写 Python 集成、Rust 集成、Go 集成,VSCode 也要写。这是笛卡尔积——M 编辑器 × N 语言 = M×N 个集成。LSP 把它变成加法——M+N。

微软 2016 开源。
协议: editor (client) ↔ language server。

LSP 的核心是"client-server 架构"——编辑器是 client,语言工具是 server。两者用 JSON-RPC 通信:
- Editor 发请求 ("跳定义"、"hover"、"诊断")
- Server 返回结果

这种"远程过程调用"模式让 client 和 server 解耦——server 可以用任何语言写 (pyright 用 TypeScript,rust-analyzer 用 Rust,clangd 用 C++),client 不需要关心。这是 LSP 的核心价值。

LSP 的诞生源于 VSCode 的实践——微软发现 VSCode 每支持一个新语言都要写大量集成代码。他们把这套协议抽出来开源,目的是"任何编辑器都可以用任何 LSP server"。这一招成功——2026 年几乎所有主流编辑器 (VSCode、Emacs、Vim、Sublime、Atom) 都支持 LSP,几乎所有主流语言都有 LSP server。

### 1.2 为什么用 LSP

理解 LSP 的价值,要回到"之前是什么样":

之前: 每个编辑器 × 每个语言 = 一对一写集成。
现在: 编辑器写一个 LSP client,语言写一个 server,通用。

之前的痛苦: Emacs 想支持 Rust——社区要写 `rust-mode` + `racer` (补全) + `flycheck-rust` (诊断) + `cargo.el` (构建)。每个都是独立项目,各自维护,质量参差。VSCode 想支持 Rust——微软写一套完全不同的集成。两套代码,功能重复,各自 bug。

现在: rust-analyzer (官方 Rust LSP server) 由 Rust 社区维护。Emacs 通过 eglot 用它,VSCode 也用它,所有 LSP 兼容编辑器都用它。**语言社区只需要写一次 server,所有编辑器受益**。这是巨大的效率提升。

Emacs 的 LSP clients:
- **eglot** (内置 Emacs 29+): 极简,与内置功能 (xref, flymake, eldoc, completion-at-point) 集成
- **lsp-mode**: 功能多,但重

新人推荐 **eglot**。

eglot 的设计哲学是"不重新发明"——它把 LSP 的功能映射到 Emacs 已有的内置功能:
- LSP "go to definition" → xref-find-definitions (Emacs 25+)
- LSP "diagnostics" → flymake (Emacs 24+)
- LSP "hover" → eldoc (Emacs 24+)
- LSP "completion" → completion-at-point (Emacs 24+)

这意味着 eglot 极简——它只是一个"翻译层",把 LSP 协议翻译成 Emacs API。整个 eglot 包大约 1000 行代码 (lsp-mode 是它的 10 倍以上)。

---

## 2. 安装 LSP servers

LSP server 是独立程序——你需要装它们,Emacs 才能用。下面是常见语言的 server 和装法。

### 2.1 Python

Python 有两个主流 LSP server:
- **pyright**: 微软出品,类型检查强,补全快。推荐。
- **python-lsp-server**: 社区维护,纯 Python,可扩展。

```bash
npm install -g pyright
# 或
pip install python-lsp-server[all]
```

pyright 的优势: 类型检查非常准——它用 TypeScript 团队的类型推导经验。在大型 Python 项目里 (特别是用 type hints 的),pyright 能找到 IDE 级别的 bug。

### 2.2 JS/TS

```bash
npm install -g typescript typescript-language-server
```

TypeScript 本身就是一个"语言服务"——`typescript-language-server` 包装它为 LSP 协议。这是 TypeScript 团队官方维护,质量最高。

### 2.3 Rust

```bash
rustup component add rust-analyzer
```

rust-analyzer 是 Rust 官方 LSP server——Rust 社区的杰作。它是"rust-emoji 之外最大的 Rust 工具投资",功能极强 (类型推导、宏展开、refactor)。Rust 用户对 rust-analyzer 的体验普遍比 RustRover (JetBrains) 好。

### 2.4 C/C++

```bash
sudo apt install clangd
# 或
sudo pacman -S clang
```

clangd 是 LLVM 项目的 LSP server——基于 libclang,质量极高。它替代了老的 cquery/ccls,是 C/C++ LSP 的事实标准。

### 2.5 Go

```bash
go install golang.org/x/tools/gopls@latest
```

gopls 是 Go 官方 LSP server——由 Go 团队维护。它处理 Go 的特殊需求 (go.mod、go.sum、build constraints)。

### 2.6 Java

Java LSP 复杂——Eclipse JDT LS 是巨型项目:

复杂,推荐 `lsp-java` 包:

```elisp
(use-package lsp-java
  :after lsp-mode
  :config
  (add-hook 'java-mode-hook 'lsp))
```

注意这里用 lsp-mode 而不是 eglot——Java 的 LSP 需求复杂 (项目结构、build 系统),lsp-mode 的 lsp-java 包更好。这是 lsp-mode 仍有价值的场景。

### 2.7 Ruby

```bash
gem install solargraph
```

solargraph 是 Ruby 社区的 LSP server——支持 Rails 特殊语法。

### 2.8 其他

几乎每种语言都有 LSP server:

| 语言 | Server |
|---|---|
| Bash | bash-language-server |
| Dockerfile | dockerfile-language-server-nodejs |
| YAML | yaml-language-server |
| JSON | vscode-langservers-extracted |
| HTML/CSS | vscode-langservers-extracted |
| Lua | lua-language-server |
| PHP | intelephense |
| SQL | sqls 或 sql-language-server |

`vscode-langservers-extracted` 是一个宝藏——它从 VSCode 提取了 HTML/CSS/JSON/ESLint 的 LSP server,任何 LSP 客户端都能用。

---

## 3. 配置 eglot

### 3.1 基础

eglot 的基础配置——通过 `:hook` 在进入对应 mode 时自动启动:

```elisp
(use-package eglot
  :ensure nil
  :hook ((python-mode . eglot-ensure)
         (js-mode . eglot-ensure)
         (typescript-mode . eglot-ensure)
         (rust-mode . eglot-ensure)
         (c-mode . eglot-ensure)
         (c++-mode . eglot-ensure)
         (go-mode . eglot-ensure))
  :custom
  (eglot-autoshutdown t)
  (eglot-events-buffer-size 0)
  (eglot-send-changes-idle-time 0.3))
```

`eglot-ensure` 是核心 hook 函数——它在 mode 启动时检查"是否有对应的 LSP server",有就启动。这意味着你打开 .py 文件,pyright 自动启动——你不需要手动 `M-x eglot`。

每个 custom 变量:
- `eglot-autoshutdown t`: 关闭最后一个相关 buffer 时,自动关闭 LSP server 进程。避免"server 一直跑"。
- `eglot-events-buffer-size 0`: 默认 eglot 记录所有 LSP 通信到一个 buffer (用于调试)。设为 0 关掉——平时不需要,占内存。调试时再设大。
- `eglot-send-changes-idle-time 0.3`: 你停止输入 0.3 秒后,eglot 把改动发给 server。这平衡"实时性"和"server 负载"——太频繁发,server 跟不上;太晚发,补全不及时。

### 3.2 Server 配置

eglot 内置了"哪个 mode 用哪个 server"的映射,但你可以覆盖:

```elisp
(setq eglot-server-programs
      '(((python-mode python-ts-mode) "pyright-langserver" "--stdio")
        ((js-mode js-ts-mode) "typescript-language-server" "--stdio")
        ((rust-mode rust-ts-mode) "rust-analyzer")
        ((c-mode c++-mode) "clangd")
        ((go-mode go-ts-mode) "gopls")))
```

这个配置明确指定"每个 mode 用哪个 server 命令"。注意每个条目支持多个 mode——`python-mode` 和 `python-ts-mode` 都用 pyright。这是 tree-sitter 集成的关键——你的 `python-ts-mode` (用 tree-sitter) 和 `python-mode` (用正则) 共享 LSP。

`--stdio` 参数告诉 server "用标准输入输出通信"——这是 LSP 的标准传输方式 (另一种是 WebSocket,但 Emacs 用 stdio)。

### 3.3 键位

eglot 的功能都需要键位——这是社区约定的 `C-c l` 前缀:

```elisp
(use-package eglot
  :bind (:map eglot-mode-map
              ("C-c l d" . eldoc-doc-buffer)        ; 显示 docstring buffer
              ("C-c l a" . eglot-code-actions)      ; code actions
              ("C-c l r" . eglot-rename)            ; 重命名
              ("C-c l f" . eglot-format-buffer)     ; 格式化
              ("C-c l i" . eglot-find-implementation)
              ("C-c l t" . eglot-find-typeDefinition)))
```

`C-c l d` 显示完整 docstring 在一个 buffer——比 echo area 的临时显示更详细。这是"读文档"的工作流。
`C-c l a` 弹 code actions 菜单——LSP 提供的 quick fix (import missing、extract function 等)。
`C-c l r` 重命名符号——LSP 跨文件改所有引用。
`C-c l f` 格式化整个 buffer——用 server 端的 formatter。

---

## 4. 用法

### 4.1 自动启动

进入配置的 mode,eglot 自动启动 (eglot-ensure)。

底部 lighter: `EGLOT[(python-mode)]`

这个 lighter 是 eglot 的状态指示——你在 mode line 看到 `EGLOT[(python-mode)]`,就知道 LSP server 跑起来了。如果没看到,说明启动失败 (server 没装、命令错误等)。

### 4.2 手动启动

有时你想手动启动 (比如 server 崩了重启):

```
M-x eglot RET
```

如果已配置,自动选 server。
否则问 server 命令。

这个交互让你"随时"启动 eglot,即使 mode hook 没触发。

### 4.3 hover

hover 是 LSP 的"看 docstring"功能——光标在符号上,自动显示 docstring:

光标在符号上,eldoc 显示 docstring 在 echo area 或 popup。

```
C-c l d    eldoc-doc-buffer
;; 在 buffer 显示完整 docstring
```

默认情况下,光标停留 0.5 秒,eldoc 在 echo area 显示简短 docstring。但 echo area 太小,长 docstring 看不全。`C-c l d` (eldoc-doc-buffer) 把完整 docstring 显示在一个 popup buffer——这是读复杂 API 文档的方式。

### 4.4 跳定义

跳定义是 LSP 最常用的功能——按 `M-.`,光标跳到符号定义:

```
M-.        xref-find-definitions (eglot 自动填充)
M-?        xref-find-references
M-,        xref-pop-marker-stack (回)
```

注意 `M-.` 是 xref 的标准键位——eglot 启动后,xref 自动用 LSP 数据。这意味着**不管你以前用 etags 还是 LSP,键位都是 `M-.`**。这种"API 一致性"是 Emacs 的设计优势。

`M-?` (find references) 是反向操作——找所有引用这个符号的地方。LSP 返回一个列表,xref 显示。在重构前"看影响范围"必用。

`M-,` (pop marker stack) 是"跳回"——`M-.` 后按 `M-,` 回到原位置。xref 维护一个 marker stack,你可以跳多层 (A → B → C),然后 `M-,` 三次回到 A。

### 4.5 重命名

重命名是 LSP 最让人惊艳的功能——你改一个变量名,LSP 跨所有文件改所有引用:

```
M-x eglot-rename RET
输入新名字 RET
```

LSP server 跨文件改所有引用。

这个操作的危险性在于"影响范围"——一个 rename 可能改几百个文件。LSP 知道 AST,所以只改**真正的引用** (变量、函数调用),不会改注释、字符串里的同名文本。这是 IDE 才有的"语义操作"。

`eglot-rename` 会问"新名字",输入 RET,LSP server 开始工作。改完后,所有受影响的 buffer 自动更新——你只需要保存。

### 4.6 诊断 (Flymake)

诊断是 LSP 的"实时检查"——语法错误、类型错误、风格警告,在你输入时实时显示。

```elisp
(use-package flymake
  :ensure nil
  :bind (:map flymake-mode-map
              ("M-n" . flymake-goto-next-error)
              ("M-p" . flymake-goto-prev-error)))
```

错误在 buffer 高亮 + fringe 显示。
`M-n` / `M-p` 跳下一个 / 上一个错误。

Flymake 在 buffer 里用波浪下划线高亮错误——红色 (error)、黄色 (warning)、蓝色 (info)。fringe (左边缘) 显示对应标记。这让"扫一眼看到错误位置"。

`M-n`/`M-p` 在错误间跳——这是"代码审查"工作流的核心。你写完代码,按 `M-n` 一个个看错误,修复,继续。

```
C-c ! l    flymake-show-buffer-diagnostics
;; 在 buffer 显示所有错误列表
```

如果你想看"所有错误一览",按 `C-c ! l`——一个 buffer 列出所有诊断,你可以跳到任何一个。

### 4.7 Code actions

Code actions 是 LSP 的"快速修复"——server 检测到问题,提供一个或多个修复建议:

```
M-x eglot-code-actions RET
;; 或
C-c l a
```

LSP 提供的 quick fix (例如 "import missing module", "extract function")。

典型场景:
- 你用了 `requests.get` 但没 import——LSP 提供 "Add import 'requests'"。
- 你写了一段长代码——LSP 提供 "Extract function"。
- 你用了一个废弃的 API——LSP 提供 "Replace with new_api"。

这是 IDE 的标志性功能——eglot 通过 LSP 获得了它。按 `C-c l a` 弹菜单,选 action,RET 应用。

### 4.8 格式化

格式化让代码风格一致——LSP server 用语言的官方 formatter:

```
M-x eglot-format-buffer
;; 整个 buffer 格式化

M-x eglot-format-region
;; 选 region 格式化
```

不同语言有不同 formatter:
- Python: black、autopep8、yapf (pyright 不直接 format,通过 LSP 调用)
- Rust: rustfmt (rust-analyzer 内置)
- Go: gofmt (gopls 内置)
- C/C++: clang-format (clangd 内置)

格式化的价值: 团队代码风格统一。每次保存前 format,代码 review 时不再争论"缩进几格"。

### 4.9 Inlay hints (Emacs 29+)

Inlay hints 是 IDE 的新功能——在代码里**内联显示**类型提示、参数名:

```elisp
(add-hook 'eglot-managed-mode-hook #'eglot-inlay-hints-mode)
```

显示类型提示、参数名等 inline。

例如 Rust 代码:
```rust
let x = foo(42);
```

开 inlay hints 后,显示:
```rust
let x: i32 = foo(number: 42);
          //      ^^^^^^^^ inlay hint
```

`x: i32` 是类型推导结果 (LSP 知道),`number:` 是参数名 (LSP 知道函数签名)。这让代码更易读,不需要 hover 看类型。

Inlay hints 在 Emacs 29+ 是实验性功能——开启它会增加一些渲染开销,但对理解代码非常有帮助。

---

## 5. eglot + 补全栈整合

eglot 自动注册 completion-at-point,与 Corfu 配合:

eglot 的核心设计是"用内置 API"——它把 LSP 的补全结果通过 `completion-at-point-functions` 提供。这意味着任何 completion UI (Corfu、Company) 都能用 LSP 补全——eglot 不关心 UI。

```elisp
(use-package corfu
  :init (global-corfu-mode 1))

(use-package eglot
  :hook ((python-mode . eglot-ensure)))
```

进入 python-mode:
- eglot 启动
- corfu 弹补全候选 (来自 LSP)
- TAB 选

这种"UI 和 source 独立"是 Emacs 的优势——VSCode 把 LSP 和它的补全 UI 绑定,你不能换。Emacs 你可以 Corfu 或 Company,都和 eglot 兼容。

---

## 6. Eglot 配置详解

### 6.1 自定义 server 启动

如果你的语言没有默认映射,可以手动加:

```elisp
(add-to-list 'eglot-server-programs
             '(my-mode . ("my-lsp-server" "--stdio")))
```

这让 eglot 知道"my-mode 用 my-lsp-server"。比如你写了一个新语言的 LSP server,这个配置让它接入 eglot。

### 6.2 eglot-stay-out-of (避免 eglot 接管)

有时你不希望 eglot 接管某个功能——比如你想用 flycheck 而不是 flymake:

```elisp
(setq eglot-stay-out-of '(flymake))
;; eglot 不接管 flymake,我用 flycheck
```

`eglot-stay-out-of` 是一个列表,告诉 eglot"不要管这些"。可选: `flymake`、`company`、`xref`、`imenu`、`completion`、`diagnostic-tags` 等。

为什么这样做? 也许你已经用 flycheck 多年,不想换。或者你想用 company 的某些高级功能。eglot 尊重你的选择——它不强行接管。

### 6.3 eglot-workspace-configuration

LSP 协议支持"workspace configuration"——client 告诉 server 一些配置选项。eglot 通过 `eglot-workspace-configuration` 提供:

```elisp
(setq-default eglot-workspace-configuration
              '((pyright . ((disableLanguageServices . nil)
                            (disableOrganizeImports . nil)
                            (autoImportCompletions . t)))))
```

传给 server 的 workspace config。

这个配置告诉 pyright "启用 auto import completions"——你输入 `reque`,pyright 提示 `requests`,选了之后自动加 `import requests` 到顶部。这是 IDE 级的体验。

不同 server 有不同的配置选项——查 server 文档。eglot 只是透传。

### 6.4 Hook 调整

eglot-managed-mode-hook 是 eglot 启动后跑的 hook——你在这里加"eglot 启动后做什么":

```elisp
(add-hook 'eglot-managed-mode-hook
          (lambda ()
            (add-hook 'before-save-hook #'eglot-format-buffer nil t)
            (add-hook 'before-save-hook #'eglot-code-action-organize-imports nil t)))
```

保存前格式化 + organize imports。

这个配置让"保存 = 自动 format + 整理 imports"——你按 `C-x C-s`,eglot 自动:
1. 格式化 buffer (用 server 的 formatter)。
2. 整理 imports (排序、删除未用、添加缺失)。

这是"无摩擦保持代码整洁"的实现——你不需要手动 format,LSP 替你做。

---

## 7. Tree-sitter 整合

Tree-sitter 和 LSP 是互补的——tree-sitter 做语法 (本地解析),LSP 做语义 (server 端):

用 `-ts-mode` 模式 + eglot:

```elisp
(add-to-list 'auto-mode-alist '("\\.py\\'" . python-ts-mode))

(add-hook 'python-ts-mode-hook #'eglot-ensure)
```

ts-mode 用 tree-sitter 解析,语法高亮更准。
eglot 用 LSP 跳定义、补全。

这种组合是 2026 年 Emacs 的"现代开发栈":
- **tree-sitter**: 语法高亮、缩进、text objects (基于 AST)。
- **LSP**: 跳定义、补全、诊断、refactor (基于语义)。

两者分工——tree-sitter 不需要外部进程 (本地解析),快;LSP 需要 server 进程,提供更深的信息。组合起来,Emacs 的代码体验接近或超过 VSCode。

---

## 8. DAP (debugger)

DAP (Debug Adapter Protocol) 是 LSP 的"调试版本"——LSP 协议用于编辑,DAP 协议用于调试。eglot 不直接支持 DAP (它的范围是 LSP),但你可以用 dap-mode 包:

需要 `dap-mode` 包 (eglot 不直接支持 debugger):

```elisp
(use-package dap-mode
  :ensure t
  :after eglot
  :config
  (dap-auto-configure-mode)
  (require 'dap-python)
  (require 'dap-node))
```

```
M-x dap-debug RET
```

dap-mode 和 eglot 协作——eglot 提供编辑能力,dap-mode 提供调试能力 (断点、单步、查看变量)。这让你在 Emacs 里有完整的 IDE 体验。

DAP 的使用场景: 复杂 bug 用 print 不够,需要断点。`M-x dap-debug` 启动调试器,你设断点 (在源代码行号处点击或 `dap-breakpoint-toggle`),按 `M-x dap-next` 单步,看变量值。

---

## 9. 实战

### 任务 1: 配置 Python LSP

```elisp
;; init.el
(use-package python
  :mode ("\\.py\\'" . python-ts-mode)
  :hook (python-ts-mode . eglot-ensure)
  :custom
  (python-indent-offset 4))
```

`npm install -g pyright`。

打开 .py,eglot 自动启动。

这是完整的 Python 配置——`python-ts-mode` 用 tree-sitter (语法高亮准),`eglot-ensure` 启动 pyright (智能)。打开 .py 文件后,你有:
- 准确的语法高亮 (tree-sitter)。
- 跳定义、补全、诊断 (pyright)。
- 类型推导、import 自动 (workspace config)。

这是 IDE 级的 Python 开发环境。

### 任务 2: 跳定义

```
光标在函数名
M-.            ; xref-find-definitions
;; 跳到定义文件
M-,            ; 回原位
```

这是日常高频操作——读代码时,看到一个函数,`M-.` 跳到定义看实现,`M-,` 回来继续读。熟练后这个操作是肌肉记忆,几毫秒完成。

### 任务 3: 重命名

```
光标在符号
M-x eglot-rename RET new-name RET
;; 所有引用被改
```

重命名是"重构"的核心——你改一个变量名,LSP 跨所有文件改。在大型项目里,这节省大量时间——以前要"查找替换",容易漏或多改。

### 任务 4: 修 warning

```
M-n            ; flymake-goto-next-error
C-c l a        ; eglot-code-actions
选 import missing
RET
```

这是"修 warning"的标准工作流——`M-n` 跳到下一个 warning,看是什么问题,`C-c l a` 看 LSP 提供的 quick fix,选一个应用。在"清理代码"时非常有用。

### 任务 5: 格式化保存

```elisp
(add-hook 'python-ts-mode-hook
          (lambda ()
            (add-hook 'before-save-hook #'eglot-format-buffer nil t)))
```

保存时自动 black 格式化 (如果 pyright server 支持)。

加上这个 hook 后,你按 `C-x C-s`,eglot 自动 black 格式化整个 buffer。你不需要手动 format——代码永远整洁。

---

## 10. lsp-mode (替代 eglot)

### 10.1 区别

eglot 和 lsp-mode 是 Emacs LSP 客户端的两大选择——它们的差异是设计哲学的差异:

| | eglot | lsp-mode |
|---|---|---|
| 内置 | 是 (Emacs 29+) | 否 |
| 大小 | 极简 | 大 |
| 集成 | xref/flymake/eldoc | 自己 UI |
| UI | 朴素 | 多 (breadcrumb, sidebar 等) |
| 配置 | 一两行 | 多 |
| 速度 | 快 | 慢 |
| LSP feature | 必需的 | 全 (含 debugger, ...) |

这个表反映了两种哲学:
- **eglot**: 极简,用内置,长期稳定。
- **lsp-mode**: 全功能,自己造 UI,迭代快。

### 10.2 lsp-mode 配置

如果你想用 lsp-mode,这是配置:

```elisp
(use-package lsp-mode
  :ensure t
  :commands (lsp lsp-deferred)
  :hook ((python-mode . lsp-deferred)
         (js-mode . lsp-deferred)
         (rust-mode . lsp-deferred))
  :custom
  (lsp-enable-symbol-highlighting t)
  (lsp-headerline-breadcrumb-enable t)
  (lsp-signature-doc-lines 1)
  (lsp-eldoc-render-all nil)
  (lsp-completion-provider :none)        ; 用 corfu
  :config
  (define-key lsp-mode-map (kbd "C-c l") lsp-command-map))

(use-package lsp-ui
  :ensure t
  :after lsp
  :custom
  (lsp-ui-doc-enable t)
  (lsp-ui-doc-position 'bottom)
  (lsp-ui-sideline-enable t))
```

注意几个 lsp-mode 独有的:
- `lsp-ui`: lsp-mode 的 UI 包——提供 sideline (旁边显示类型)、doc popup、breadcrumb。
- `lsp-headerline-breadcrumb-enable`: 在 header 显示路径面包屑 (像 VSCode)。
- `lsp-enable-symbol-highlighting`: 高亮所有同名符号。
- `lsp-completion-provider :none`: 不用 lsp 自己的补全,用 corfu——这是和 corfu 配合的关键。

### 10.3 选哪个

- **新人**: eglot (轻量,内置)
- **重度用户**: lsp-mode (功能全)

更细的判断:
- 用 Java/Scala (复杂项目结构) → lsp-mode (它的 java/scala 支持更好)。
- 用 dap-mode debug → lsp-mode (集成更好)。
- 想要 breadcrumb、sideline → lsp-mode。
- 极简、长期稳定 → eglot。

Emacs 29 内置 eglot 是信号——eglot 的设计是 Emacs 官方认可的方向。所以新人学 eglot 不会错。

---

## 11. 调试 LSP

LSP 出问题时,几个调试工具:

### 11.1 eglot-events buffer

```
M-x eglot-events-buffer
```

显示 eglot 和 server 的所有通信。

这个 buffer 显示所有 LSP 消息 (请求、响应、通知)——JSON-RPC 格式。如果某个功能不工作 (比如 hover 没反应),看这个 buffer 找原因: server 返回错误了吗? 请求发了吗?

### 11.2 检查 server 是否运行

```
M-x eglot-events-buffer
;; 应该看到 startup messages
```

或:
```
M-x list-processes
```

`list-processes` 显示所有 Emacs 子进程——你应该看到 pyright/typescript-language-server/rust-analyzer 等。如果没看到,说明 server 没启动。

### 11.3 restart

如果 server 卡住:

```
M-x eglot-shutdown
M-x eglot
```

`eglot-shutdown` 关闭当前 buffer 的 server,`eglot` 重启。这解决大多数"server 状态错乱"问题。

### 11.4 server 路径问题

最常见的问题——server 不在 PATH 里:

```bash
which pyright-langserver
```

确认 PATH 里有。

如果 `which` 找不到,说明 server 没装,或者装了但不在 PATH。检查:
- npm 全局装的吗? 检查 `npm config get prefix`。
- pip 装的吗? 检查 `pip show pyright`。
- rustup 装的吗? 检查 `rustup component list | grep rust-analyzer`。

Emacs 启动方式影响 PATH——从 GUI 启动的 Emacs 可能看不到 shell 的 PATH。解决:

```elisp
(when *is-mac*
  (push "/opt/homebrew/bin" exec-path))
```

这把 `/opt/homebrew/bin` 加到 Emacs 的 exec-path——macOS 上 Homebrew 装的 server 通常在那里。Linux 类似,可能需要 `/usr/local/bin` 或 `~/.cargo/bin`。

---

## 12. 自测

1. LSP 是什么协议?
2. eglot 和 lsp-mode 区别?
3. 怎么启动 eglot?
4. 怎么看 docstring?
5. 怎么 rename?
6. 怎么显示诊断?
7. tree-sitter 和 LSP 关系?

**答案**:
> 1. editor 与 language server 的通信协议
> 2. eglot 内置极简;lsp-mode 社区功能全
> 3. M-x eglot 或 hook 自动 eglot-ensure
> 4. C-c l d (eldoc-doc-buffer) 或光标停留自动显示
> 5. M-x eglot-rename
> 6. flymake-mode 自动高亮,M-n/M-p 跳
> 7. tree-sitter 给语法高亮 (本地解析);LSP 给智能 (server 端)

---

## 13. 下一步

Module 5 完成。

进入 Module 6: Elisp 深度 (Reference) ★核心★
