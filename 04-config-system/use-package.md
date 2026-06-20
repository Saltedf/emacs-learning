# use-package 详解

> 学完这个文件,你能用 use-package 优雅管理 100+ 个包

---

## 1. 为什么用 use-package

### 1.1 没有 use-package 之前

想象你的 init.el 没有 use-package。装一个 magit 要做四件事: 检查装没装、装、require、绑键、配置。这五件事散落在 init.el 各处,加载顺序敏感,而且 `require` 让 magit 在启动时立即加载——慢。每多一个包,init.el 就更乱一分:

```elisp
;; 老 init.el 风格
(require 'package)
(package-initialize)

(when (not package-installed-p 'magit)
  (package-install 'magit))
(require 'magit)

(add-hook 'magit-mode-hook (lambda () ...))
(global-set-key (kbd "C-x g") 'magit-status)
(setq magit-status- ...)
;; ... 50 行配置一个包
```

混乱——而且这还只是一个包,你装 30 个包就是 1500 行这种代码。性能也差: 每个 require 都在启动时把整个包 load 到内存。

### 1.2 有 use-package

`use-package` 把上面五件事压成一段声明。看起来像配置,实际是宏调用——展开后会生成 autoload、绑键、defer、config body,但写得像 JSON 那样清晰。而且**延迟加载**: magit 只在你真按 C-x g 时才 load 到内存。这是为什么 100 个 use-package 的 init.el 启动还能 < 2 秒。

```elisp
(use-package magit
  :ensure t
  :bind ("C-x g" . magit-status)
  :hook (magit-mode . (lambda () ...))
  :custom (magit-status- ... ...))
```

**清晰、声明式、自包含**。

### 1.3 use-package 的力量

除了清晰,use-package 还有几个工程上的优势:

- **延迟加载**: 默认按需 (autoload)
- **声明式**: 一眼看 package 干啥
- **错误隔离**: 一个包出错不影响其他(use-package 会捕获 config body 的错误)
- **profile**: `use-package-verbose` 看每个包用时
- **统计**: `use-package-statistics` 看哪个包慢

这些特性让"管理 100 个包"变得可行——你能 profile 哪个包慢、能隔离错误、能延迟加载。没有 use-package,这一切都要自己造轮子。

---

## 2. use-package 完整语法

```elisp
(use-package PACKAGE-NAME
  [KEYWORD VALUE]...)
```

### 关键字详解

#### `:ensure`

`:ensure t` 告诉 use-package "如果没装就装"。这是 use-package 最常用的功能——让 init.el 自我修复: 在新机器上 clone 你的 dotfiles 后第一次启动,所有 `:ensure t` 的包会自动安装:

```elisp
(use-package magit
  :ensure t)                   ; 没装就装

(use-package magit
  :ensure nil)                 ; 不自动装 (假设已手动)

(use-package lsp-mode
  :ensure (lsp-mode :pin "melpa"))   ; 指定 archive
```

全局默认:
```elisp
(use-package-always-ensure t)
```

#### `:init`

`:init` 在包**加载之前**跑,因此只能放"不需要包已经加载"的代码——通常是设一些触发 autoload 的变量、绑键。**关键: `:init` 里不要 `require` 包**——那会让延迟加载失效:

加载**之前**跑。**只用于设置触发 autoload 的东西** (键位、hook、mode 触发条件)。

```elisp
(use-package magit
  :init
  (setq magit-refresh-status-buffer nil))   ; magit 加载前设
```

**关键**: `:init` 里不要 `require` 包。

#### `:config`

`:config` 在包**加载之后**跑,可以调用包的函数、改包的状态。`:init` 和 `:config` 的区别是 use-package 的精髓——理解它你就理解了延迟加载:

加载**之后**跑。**配置包状态**。

```elisp
(use-package magit
  :config
  (magit-status-setup-buffer)   ; 这是 magit 函数,需要 magit 已加载
  (setq magit-status-margin ...))
```

#### `:commands`

`:commands` 声明"这些命令需要 autoload"——Emacs 不会立即加载包,但会假装这些命令存在,真调用时才 load。`:bind`、`:hook`、`:mode` 都隐含了 `:commands`,所以一般不用手写,但理解它对调试有用:

声明哪些命令需要 autoload。

```elisp
(use-package magit
  :commands (magit-status magit-dispatch))
;; 这两个命令被调用时才加载 magit
```

`:bind`、`:hook`、`:mode` 自动加 `:commands`,不用显式。

#### `:bind`

`:bind` 绑键的同时自动声明 autoload——绑了键但包没加载,直到你真按这个键时才加载:

```elisp
(use-package magit
  :bind ("C-x g" . magit-status))

;; 多个
(use-package magit
  :bind (("C-x g" . magit-status)
         ("C-x G" . magit-dispatch)
         :map magit-status-mode-map
         ("q" . magit-quit-window)))

;; prefix
(use-package org
  :bind (:map org-mode-map
              ("C-c a" . org-agenda)
              ("C-c c" . org-capture)))
```

#### `:mode`

`:mode` 把文件扩展名映射到 mode——打开匹配的文件时,Emacs 才加载这个包并切到对应 mode:

文件扩展名 → mode:

```elisp
(use-package markdown-mode
  :mode ("\\.md\\'" . markdown-mode)
  ;; 多个
  :mode (("\\.md\\'" . markdown-mode)
         ("\\.markdown\\'" . markdown-mode)))
```

#### `:interpreter`

类似 `:mode`,但根据 shebang 解释器触发:

```elisp
(use-package python
  :interpreter "python")
```

#### `:magic`

根据文件开头的"magic 字符"触发——比如 PNG 文件开头是 `\x89PNG`:

文件 magic 字符:

```elisp
(use-package image-mode
  :magic "\\`\x89PNG\r\n")
```

#### `:hook`

`:hook` 是 `add-hook` 的声明式版本——把"在某 mode 启用时跑某函数"压成一行。注意 `:hook (prog-mode . yas-minor-mode)` 不是绑 prog-mode 到 yas-minor-mode 函数,而是 add-hook prog-mode-hook 启用 yas-minor-mode:

```elisp
(use-package yasnippet
  :hook (prog-mode . yas-minor-mode)
  :hook ((python-mode js-mode) . yas-minor-mode))   ; 多 mode
```

自动生成 `package-mode-hook` 添加。

#### `:custom`

`:custom` 用 customize 机制设变量,比 `:config (setq ...)` 更"正确"——它会触发变量的 `:set` setter,处理 file-local 安全检查等:

```elisp
(use-package magit
  :custom
  (magit-status-margin '(t nil))
  (magit-diff-refine-hunk t))
```

#### `:custom-face`

类似 `:custom`,但设 face(外观):

```elisp
(use-package magit
  :custom-face
  (magit-diff-added ((t (:background "light green")))))
```

#### `:defer`

`:defer t` 显式声明延迟加载——通常 `:bind` / `:commands` / `:mode` 已经隐含 defer,但有时你想明确表达"按需加载":

```elisp
(use-package magit
  :defer t)                    ; 延迟加载 (按 :commands/:bind 触发)

(use-package magit
  :defer 5)                    ; 5 秒后加载

(use-package magit
  :defer t
  :demand t)                   ; 立即加载 (覆盖 :defer)
```

#### `:after`

`:after` 表达"在某 feature 加载之后才加载"——处理包之间的依赖关系:

等某些 feature 加载后再加载:

```elisp
(use-package lsp-mode
  :after (company))            ; company 加载后才加载
```

#### `:demand`

`:demand` 强制立即加载——覆盖默认的延迟行为。少数情况用,比如某包没有 autoload 触发点,你必须强制加载:

立即加载 (不要 defer):

```elisp
(use-package my-package
  :demand t)
```

#### `:load-path`

从本地路径加载(开发自己的包时常用):

```elisp
(use-package my-local
  :load-path "lisp/my-local/")
```

#### `:load`/`:no-require`

`:no-require t` 告诉 use-package "不要真的 require 这个包"——只想配置一些选项,假设别人会加载:

```elisp
(use-package my-package
  :no-require t                ; 不真的 require (只配置,假设别人加载)
  :config
  (message "after loaded"))
```

#### `:defines` / `:functions`

声明变量/函数,告诉 byte-compiler"这些存在,别警告"——开发时常用,避免 byte-compile warning 噪音:

声明变量/函数 (避免 byte-compile warning):

```elisp
(use-package my-package
  :defines my-var
  :functions my-func
  :config
  (setq my-var ...)
  (my-func ...))
```

#### `:diminish` / `:delight`

minor mode 启用后会在 mode line 显示 lighter(如 "Abbrev")。装多了 mode line 会爆。`:diminish` 隐藏指定 minor mode 的 lighter——清理 mode line 的必备:

隐藏 minor mode 的 lighter:

```elisp
(use-package abbrev
  :diminish abbrev-mode)
;; mode line 不显示 "Abbrev"

(use-package flyspell
  :delight)                    ; 用 delight 包
```

---

## 3. 例子: 配置 30 个包

下面是一个完整的 `init-extensions.el`,演示如何用 use-package 管理当代 Emacs 的"标配生态": which-key(按键提示)、vertico/orderless/marginalia/consult(垂直补全 + 模糊匹配 + 注释 + 增强命令)、embark(上下文操作)、corfu/cape(补全框架)、magit/git、org/projectile/flycheck/eglot/yas/lsp。每个包都是独立 use-package,自包含,互不干扰。这是真实生产配置的样子:

```elisp
;;; init-extensions.el -*- lexical-binding: t; -*-

(use-package which-key
  :ensure t
  :diminish which-key-mode
  :config
  (which-key-mode 1)
  (setq which-key-idle-delay 0.3))

(use-package vertico
  :ensure t
  :init
  (vertico-mode 1))

(use-package orderless
  :ensure t
  :custom
  (completion-styles '(orderless basic)))

(use-package consult
  :ensure t
  :bind (("C-s" . consult-line)
         ("C-x b" . consult-buffer)
         ("M-y" . consult-yank-pop)
         ("M-g g" . consult-goto-line)
         ("M-g M-g" . consult-goto-line)))

(use-package marginalia
  :ensure t
  :init
  (marginalia-mode 1))

(use-package embark
  :ensure t
  :bind (("C-." . embark-act)
         ("M-." . embark-dwim)))

(use-package corfu
  :ensure t
  :init
  (global-corfu-mode 1)
  :custom
  (corfu-auto t)
  (corfu-auto-delay 0.2))

(use-package cape
  :ensure t
  :init
  (add-to-list 'completion-at-point-functions #'cape-file)
  (add-to-list 'completion-at-point-functions #'cape-dabbrev))

(use-package magit
  :ensure t
  :bind ("C-x g" . magit-status)
  :custom
  (magit-display-buffer-function #'magit-display-buffer-same-window-except-diff-v1))

(use-package org
  :ensure t
  :mode ("\\.org\\'" . org-mode)
  :bind (("C-c a" . org-agenda)
         ("C-c c" . org-capture)
         ("C-c l" . org-store-link))
  :custom
  (org-directory "~/org/")
  (org-default-notes-file (concat org-directory "inbox.org")))

(use-package projectile
  :ensure t
  :diminish projectile-mode
  :init
  (projectile-mode 1)
  :bind (:map projectile-mode-map
              ("C-c p" . projectile-command-map))
  :custom
  (projectile-completion-system 'default))

(use-package flycheck
  :ensure t
  :diminish flycheck-mode
  :init
  (global-flycheck-mode 1))

(use-package eglot
  :ensure nil              ; Emacs 29+ 内置
  :hook ((python-mode . eglot-ensure)
         (js-mode . eglot-ensure)))

(use-package yasnippet
  :ensure t
  :diminish yas-minor-mode
  :hook (prog-mode . yas-minor-mode)
  :config
  (yas-reload-all))

(use-package lsp-mode
  :ensure t
  :commands (lsp lsp-deferred)
  :hook ((python-mode . lsp-deferred))
  :custom
  (lsp-enable-symbol-highlighting nil)
  (lsp-signature-doc-lines 1))

(provide 'init-extensions)
;;; init-extensions.el ends here
```

---

## 4. use-package 调试

use-package 的强大也带来调试挑战——宏展开后生成了大量代码,出错时不容易看到原始位置。但 use-package 提供了几个调试开关。

### 4.1 verbose

`use-package-verbose t` 让 use-package 在加载每个包时打一条 message,你能看到"哪个包正在加载、用时多少"。这是排查"为什么某包没加载"的第一步:

```elisp
(setq use-package-verbose t)
```

加载时 message 每个包的状态。

### 4.2 statistics

`use-package-statistics t` 收集每个包的 init/config 用时。启动后 `M-x use-package-report` 看报告——哪个包慢一目了然。这是优化启动时间的关键工具:

```elisp
(setq use-package-statistics t)

;; 启动后:
M-x use-package-report
```

显示每个包的 init/config 用时。

### 4.3 检查 expand

如果 use-package 表现和预期不符,最直接的办法是看它展开成什么。`pp-macroexpand-last-sexp` 把光标处的 use-package form 展开,你能看到实际生成的 autoload、绑键、config body——一目了然:

```elisp
M-x pp-macroexpand-last-sexp
;; 光标在 use-package form 上,展开宏
```

看 use-package 实际生成了什么。

### 4.4 debug 加载失败

如果某包加载时抛 error,默认情况下 use-package 会"吞掉"错误继续跑(为了不让一个包挂掉整个 init)。但调试时你想看错误——开 `debug-on-error`:

```elisp
(setq debug-on-error t)
```

加载时 error 进 debugger。

---

## 5. use-package 替代品

use-package 不是唯一选择。社区还有几个类似工具,各有侧重。

### 5.1 leaf.el

`leaf.el` 由日本 Emacs 社区开发,语法和 use-package 很像但更"现代"。在日本社区很流行,但全球范围 use-package 仍是主流:

日本 Emacs 社区流行:

```elisp
(leaf magit
  :ensure t
  :bind (("C-x g" . magit-status)))
```

类似 use-package,语法略不同。

### 5.2 setup.el

`setup.el` 由 systemd 作者 Lennart Poettering 之外的另一位 Lennart—— Lennart Borgman—— 开发,Emacs 29+ 内置。比 use-package 更轻量,语法更"函数式":

内置 (Emacs 29+):

```elisp
(setup magit
  (:package magit)
  (:global "C-x g" magit-status))
```

更轻量,推荐试试。

### 5.3 straight.el + use-package

straight.el 本身不是 use-package 的替代,而是 package.el 的替代——但可以和 use-package 整合,让所有 use-package 用 straight 装(支持 git fork):

```elisp
(straight-use-package 'use-package)
(setq straight-use-package-by-default t)
```

所有 use-package 用 straight 装 (支持 git fork)。

---

## 6. 自测

1. `:init` 和 `:config` 区别?
2. `:ensure` 干啥?
3. `:commands` 干啥?
4. `:bind` 自动做什么?
5. `:defer t` vs `:demand t`?

**答案**:
> 1. :init 加载前跑;:config 加载后跑
> 2. 没装就装
> 3. 声明这些命令需要 autoload (调用时才加载包)
> 4. 绑键 + 自动 :commands
> 5. defer 延迟;demand 立即加载

---

## 7. 下一步

进入 `keymaps.md`。
