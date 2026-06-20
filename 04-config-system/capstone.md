# Capstone: Module 4 毕业项目

> **目标**: 写出 500 行模块化 init.el + 自定义 minor mode
> **时长**: 1 周 (~15 小时)
> **难度**: ★★★★☆

---

## 项目概述

这个 capstone 把 Module 0 的 10 行 init.el 升级成一个真实的、可维护的、可分享的配置。它不只是"加更多配置"——而是把整个 Module 4 学的东西(use-package / keymap / hook / customize / minor mode)整合成一个作品。

把 Module 0 的 10 行 init.el 升级成:
- 模块化 (~10 个文件)
- 500+ 行
- 装管理 30+ 个包
- 自定义 minor mode (个人键位)
- ERT 测试

完成后,你的 init.el 可以放 GitHub 展示你的 Emacs 哲学,也能在新机器上 clone + 启动就跑(use-package 自动装包)。

---

## 任务清单

### Part 1: 文件结构 (2 小时)

第一步是规划目录结构。每个文件对应一个职责——basics(基础)、ui(界面)、editing(编辑)、prog(编程)、org、git、completion(补全)。`my-extensions/` 放你自己的包(Module 3 的 my-todo + 这模块新写的)。这种"按职责拆分"是 Unix 哲学,每个文件做好一件事。

创建:

```
~/.config/emacs/
├── early-init.el
├── init.el
├── custom.el
└── lisp/
    ├── init-package.el
    ├── init-basics.el
    ├── init-ui.el
    ├── init-editing.el
    ├── init-prog.el
    ├── init-org.el
    ├── init-git.el
    ├── init-completion.el
    └── my-extensions/
        ├── my-keys-mode.el
        ├── my-editing-mode.el
        └── my-todo.el   (从 Module 3 来的)
```

### Part 2: early-init.el (30 分钟)

early-init.el 的职责是"在 package.el 和 frame 创建之前必须设的东西"。下面这段做四件事: 关闭启动时包初始化(我们自己管)、开 quickstart(预生成加载代码加速)、关掉默认的菜单/工具/滚动条(避免 GUI 闪一下)、临时调高 GC 阈值让启动快——然后在启动完成后恢复:

```elisp
;;; early-init.el -*- lexical-binding: t; -*-

(setq package-enable-at-startup nil)
(setq package-quickstart t)

(setq frame-inhibit-implied-resize t)
(push '(menu-bar-lines . 0) default-frame-alist)
(push '(tool-bar-lines . 0) default-frame-alist)
(push '(vertical-scroll-bars) default-frame-alist)

(setq gc-cons-threshold most-positive-fixnum)
(add-hook 'emacs-startup-hook
          (lambda ()
            (setq gc-cons-threshold 800000)))
```

### Part 3: init-package.el (1 小时)

init-package.el 配置包管理: 注册三个 archive、初始化 package、自动装 use-package(如果还没装)、设全局 `:ensure t`(每个 use-package 默认自动装包)。这段是整个 init 的"地基"——后面所有 use-package 都依赖它:

```elisp
;;; init-package.el -*- lexical-binding: t; -*-

(require 'package)

(setq package-archives
      '(("gnu"    . "https://elpa.gnu.org/packages/")
        ("nongnu" . "https://elpa.nongnu.org/nongnu/")
        ("melpa"  . "https://melpa.org/packages/")))

(package-initialize)
(unless package-archive-contents
  (package-refresh-contents))

;; 装 use-package
(unless (package-installed-p 'use-package)
  (package-install 'use-package))
(require 'use-package)

(setq use-package-always-ensure t
      use-package-verbose t
      use-package-statistics t
      use-package-compute-statistics t)

(provide 'init-package)
;;; init-package.el ends here
```

### Part 4: init.el (1 小时)

init.el 是入口——它的职责是"协调者",不是"配置者"。它做四件事: 加 load-path、定义平台常量(Mac/Linux)、设 custom-file(避免 customize 污染 init.el)、按顺序 require 各模块。注意 Mac 平台的特殊处理——把 Option 设成 Meta,Cmd 设成 Super:

```elisp
;;; init.el -*- lexical-binding: t; -*-

(add-to-list 'load-path (expand-file-name "lisp" user-emacs-directory))
(add-to-list 'load-path (expand-file-name "lisp/my-extensions" user-emacs-directory))

(defconst *is-mac* (eq system-type 'darwin))
(defconst *is-linux* (eq system-type 'gnu/linux))

(setq custom-file (expand-file-name "custom.el" user-emacs-directory))
(load custom-file t)

(require 'init-package)
(require 'init-basics)
(require 'init-ui)
(require 'init-editing)
(require 'init-completion)
(require 'init-prog)
(require 'init-org)
(require 'init-git)
(require 'my-keys-mode)
(require 'my-editing-mode)
(require 'my-todo)

(when *is-mac*
  (setq mac-option-modifier 'meta
        mac-command-modifier 'super)
  (global-set-key (kbd "s-c") #'ns-copy-including-secondary)
  (global-set-key (kbd "s-v") #'yank))

(global-my-keys-mode 1)
(global-my-editing-mode 1)
(global-set-key (kbd "C-c t") #'my-todo)

(message "Emacs loaded in %s" (emacs-init-time))

(provide 'init)
;;; init.el ends here
```

### Part 5: init-basics.el (1 小时)

init-basics.el 是"基础编辑环境"——身份信息、缩进(用空格不用 tab)、备份(全关,因为烦)、编码(强制 UTF-8)、几个常用 minor mode(recentf 记住最近文件、savehist 记住 minibuffer 历史、winner 记住 window 配置)。这一段配置完,Emacs 就从一个"光秃秃的编辑器"变成"能用的编辑器":

```elisp
;;; init-basics.el -*- lexical-binding: t; -*-

(setq user-full-name "Your Name"
      user-mail-address "you@example.com")

(setq-default
 indent-tabs-mode nil
 tab-width 4
 fill-column 80
 truncate-lines t
 word-wrap t)

(setq
 inhibit-startup-screen t
 initial-scratch-message nil
 ring-bell-function 'ignore
 visible-bell nil
 make-backup-files nil
 auto-save-default nil
 create-lockfiles nil
 delete-by-moving-to-trash t
 load-prefer-newer t
 use-short-answers t
 confirm-kill-processes nil
 enable-recursive-minibuffers t
 read-process-output-max (* 1024 1024))

(set-language-environment "UTF-8")
(set-default-coding-systems 'utf-8-unix)
(set-terminal-coding-system 'utf-8)
(set-keyboard-coding-system 'utf-8)
(prefer-coding-system 'utf-8)

(recentf-mode 1)
(savehist-mode 1)
(save-place-mode 1)
(winner-mode 1)
(global-auto-revert-mode 1)
(pixel-scroll-precision-mode 1)

(setq recentf-max-saved-items 200)

(global-set-key (kbd "C-c r") #'recentf-open-files)
(global-set-key (kbd "C-x k") #'kill-this-buffer)
(global-set-key (kbd "M-o") #'other-window)

(provide 'init-basics)
;;; init-basics.el ends here
```

### Part 6: init-ui.el (1 小时)

init-ui.el 管"外观"——关掉默认的菜单/工具/滚动条/光标闪烁、开行号、高亮当前行、显示匹配括号、自动配对、加载主题(modus-vivendi 是 Emacs 28+ 内置的暗色主题)、设字体(英文 JetBrains Mono,中文 Sarasa Mono SC)。一段好的 UI 配置让你看代码更舒服,也是 Emacs 用户的"个人品味展示":

```elisp
;;; init-ui.el -*- lexical-binding: t; -*-

(menu-bar-mode -1)
(tool-bar-mode -1)
(scroll-bar-mode -1)
(blink-cursor-mode -1)
(global-hl-line-mode 1)
(show-paren-mode 1)
(electric-pair-mode 1)
(delete-selection-mode 1)
(global-display-line-numbers-mode 1)
(column-number-mode 1)
(size-indication-mode 1)
(global-visual-line-mode 1)

(setq display-line-numbers-type 'relative
      show-paren-style 'expression
      indicate-empty-lines t
      indicate-buffer-boundaries 'left)

(load-theme 'modus-vivendi t)

(set-face-attribute 'default nil
                    :family "JetBrains Mono"
                    :height 140)

(dolist (charset '(kana han symbol cjk-misc bopomofo))
  (set-fontset-font (frame-parameter nil 'font)
                    charset
                    (font-spec :family "Sarasa Mono SC")))

(use-package all-the-icons
  :ensure t)

(use-package doom-modeline
  :ensure t
  :init (doom-modeline-mode 1)
  :custom
  (doom-modeline-buffer-file-name-style 'truncate-except-project)
  (doom-modeline-icon t)
  (doom-modeline-modal-icon t))

(provide 'init-ui)
;;; init-ui.el ends here
```

### Part 7: init-completion.el (1 小时)

init-completion.el 配置"补全 / 搜索"——这是当代 Emacs 配置的核心。这套(vertico + orderless + marginalia + consult + embark + corfu + cape + which-key)被称为"现代 Emacs 补全栈",让默认 Emacs 的 minibuffer 体验脱胎换骨。下面这段代码几乎是 Emacs 社区的"标配":

```elisp
;;; init-completion.el -*- lexical-binding: t; -*-

(use-package vertico
  :init
  (vertico-mode 1)
  :custom
  (vertico-cycle t))

(use-package orderless
  :custom
  (completion-styles '(orderless basic))
  (completion-category-overrides
   '((file (styles basic partial-completion)))))

(use-package marginalia
  :init (marginalia-mode 1))

(use-package consult
  :bind (;; Search
         ("C-s" . consult-line)
         ("C-r" . consult-line)
         ("M-s u" . consult-focus-lines)
         ;; Buffer
         ("C-x b" . consult-buffer)
         ("C-x B" . consult-buffer-other-window)
         ("C-x 4 b" . consult-buffer-other-window)
         ;; Yank
         ("M-y" . consult-yank-pop)
         ;; Imenu
         ("M-g i" . consult-imenu)
         ;; Grep
         ("M-s g" . consult-grep)
         ("M-s r" . consult-ripgrep)
         ;; Bookmarks
         ([remap bookmark-jump] . consult-bookmark)))

(use-package embark
  :bind (("C-." . embark-act)
         ("M-." . embark-dwim)
         ("C-h B" . embark-bindings))
  :init
  (setq prefix-help-command #'embark-prefix-help-command))

(use-package embark-consult
  :after (embark consult))

(use-package corfu
  :init
  (global-corfu-mode 1)
  :custom
  (corfu-auto t)
  (corfu-auto-delay 0.2)
  (corfu-auto-prefix 2)
  (corfu-cycle t)
  :bind (:map corfu-map
              ("RET" . nil)
              ("TAB" . corfu-next)
              ([tab] . corfu-next)))

(use-package cape
  :init
  (add-to-list 'completion-at-point-functions #'cape-file)
  (add-to-list 'completion-at-point-functions #'cape-dabbrev)
  (add-to-list 'completion-at-point-functions #'cape-keyword))

(use-package which-key
  :diminish which-key-mode
  :init (which-key-mode 1)
  :custom
  (which-key-idle-delay 0.3)
  (which-key-separator " → "))

(provide 'init-completion)
;;; init-completion.el ends here
```

### Part 8: init-prog.el (1 小时)

init-prog.el 配置"编程辅助"——flycheck(语法检查)、eglot(LSP 客户端,Emacs 29+ 内置)、tree-sitter(更精确的语法高亮)、yasnippet(snippet)、smartparens(智能括号)、rainbow-delimiters(彩虹括号)、aggressive-indent(自动缩进)。这些让 Emacs 变成一个像样的代码编辑器:

```elisp
;;; init-prog.el -*- lexical-binding: t; -*-

(use-package flycheck
  :init (global-flycheck-mode 1)
  :diminish flycheck-mode)

(use-package eglot
  :hook ((python-mode . eglot-ensure)
         (js-mode . eglot-ensure)
         (typescript-mode . eglot-ensure))
  :custom
  (eglot-autoshutdown t))

(use-package tree-sitter
  :diminish tree-sitter-mode
  :hook ((python-mode . tree-sitter-hl-mode)
         (js-mode . tree-sitter-hl-mode))
  :config
  (global-tree-sitter-mode 1))

(use-package tree-sitter-langs
  :after tree-sitter)

(use-package yasnippet
  :diminish yas-minor-mode
  :hook (prog-mode . yas-minor-mode)
  :config
  (yas-reload-all))

(use-package yasnippet-snippets
  :after yasnippet)

(use-package smartparens
  :diminish smartparens-mode
  :hook (prog-mode . smartparens-strict-mode)
  :config
  (require 'smartparens-config))

(use-package rainbow-delimiters
  :hook (prog-mode . rainbow-delimiters-mode))

(use-package aggressive-indent
  :diminish aggressive-indent-mode
  :hook ((emacs-lisp-mode . aggressive-indent-mode)
         (python-mode . aggressive-indent-mode)))

(provide 'init-prog)
;;; init-prog.el ends here
```

### Part 9: init-org.el (30 分钟)

init-org.el 配置 Org mode——Emacs 最强大的工具之一,你会在 Module 5 详细学。这里先做基础: 装 org + org-roam(双向链接笔记)+ org-bullets(美化标题符号)+ 配置 capture 模板(Todo / Note / Journal 三种快速记录)+ 启用 babel(python/shell/elisp 代码块执行):

```elisp
;;; init-org.el -*- lexical-binding: t; -*-

(use-package org
  :ensure t
  :mode ("\\.org\\'" . org-mode)
  :bind (("C-c a" . org-agenda)
         ("C-c c" . org-capture)
         ("C-c l" . org-store-link))
  :custom
  (org-directory "~/org/")
  (org-default-notes-file (concat org-directory "inbox.org"))
  (org-agenda-files (list org-directory))
  (org-capture-templates
   '(("t" "Todo" entry (file+headline org-default-notes-file "Tasks")
      "* TODO %?\n  %i\n  %a")
     ("n" "Note" entry (file+headline org-default-notes-file "Notes")
      "* %?\n  %i")
     ("j" "Journal" entry (file+datetree (concat org-directory "journal.org"))
      "* %?\n  Entered on %U\n  %i"))))

(use-package org-roam
  :ensure t
  :after org
  :custom
  (org-roam-directory (concat org-directory "roam/"))
  (org-roam-completion-everywhere t)
  :bind (("C-c n l" . org-roam-buffer-toggle)
         ("C-c n f" . org-roam-node-find)
         ("C-c n i" . org-roam-node-insert)))

(use-package org-bullets
  :hook (org-mode . org-bullets-mode))

(org-babel-do-load-languages
 'org-babel-load-languages
 '((python . t)
   (shell . t)
   (emacs-lisp . t)
   (sql . t)))

(provide 'init-org)
;;; init-org.el ends here
```

### Part 10: init-git.el (30 分钟)

init-git.el 配置 Git 集成——magit(Emacs 最强 Git 客户端,Module 5 会专门讲)、forge(GitHub/GitLab 集成)、git-gutter(行号旁显示修改标记)、git-timemachine(浏览文件历史版本)。装好这套,你几乎不需要离开 Emacs 处理 Git:

```elisp
;;; init-git.el -*- lexical-binding: t; -*-

(use-package magit
  :ensure t
  :bind (("C-x g" . magit-status)
         ("C-x G" . magit-dispatch))
  :custom
  (magit-display-buffer-function #'magit-display-buffer-same-window-except-diff-v1)
  (magit-repository-directories '(("~/projects" . 2)))
  :config
  ;; 不要 save 时问
  (remove-hook 'server-switch-hook 'magit-process-discard)
  (remove-hook 'magit-status-mode-hook 'magit-load-config-extensions))

(use-package forge
  :after magit)

(use-package git-gutter
  :diminish git-gutter-mode
  :init (global-git-gutter-mode 1)
  :custom
  (git-gutter:modified-sign " ")
  (git-gutter:added-sign " ")
  (git-gutter:deleted-sign " "))

(use-package git-timemachine)

(provide 'init-git)
;;; init-git.el ends here
```

### Part 11: my-keys-mode.el (1 小时)

my-keys-mode.el 是这个 capstone 的"重头戏"——把你所有的个人键位打包成一个 minor mode。这是 Emacs 老用户的标配: 不污染 global-map、能整体开关、优先级高于 major mode。下面的例子涵盖窗口管理(M-o 切窗口、M-1/2/3 分屏)、buffer 管理(M-k 关 buffer)、文件(C-c f 前缀)、编辑(C-c d 复制行、M-上下移动行)、搜索(C-c s)、项目(C-c p 前缀):

参考 `minor-mode-tutorial.md` 的 my-editing-mode,但更全面。

```elisp
;;; my-keys-mode.el -*- lexical-binding: t; -*-

(defvar my-keys-mode-map
  (let ((map (make-sparse-keymap)))
    ;; Window
    (define-key map (kbd "M-o") #'other-window)
    (define-key map (kbd "M-O") (lambda () (interactive) (other-window -1)))
    (define-key map (kbd "M-1") (lambda () (interactive) (delete-other-windows)))
    (define-key map (kbd "M-2") #'split-window-below)
    (define-key map (kbd "M-3") #'split-window-right)
    (define-key map (kbd "M-0") #'delete-window)
    ;; Buffer
    (define-key map (kbd "M-k") #'kill-this-buffer)
    (define-key map (kbd "M-K") #'kill-buffer-and-window)
    (define-key map (kbd "<C-tab>") #'mode-line-other-buffer)
    ;; File
    (define-key map (kbd "C-c f s") #'save-buffer)
    (define-key map (kbd "C-c f r") #'recentf-open-files)
    (define-key map (kbd "C-c f R") #'rename-file-and-buffer)
    (define-key map (kbd "C-c f d") #'my-duplicate-file)
    ;; Editing
    (define-key map (kbd "C-c d") #'my-duplicate-line)
    (define-key map (kbd "M-<up>") #'my-move-line-up)
    (define-key map (kbd "M-<down>") #'my-move-line-down)
    (define-key map (kbd "C-c C-/") #'comment-line)
    ;; Search
    (define-key map (kbd "C-c s") #'consult-line)
    (define-key map (kbd "C-c S") #'consult-ripgrep)
    ;; Project
    (define-key map (kbd "C-c p f") #'project-find-file)
    (define-key map (kbd "C-c p p") #'project-switch-project)
    (define-key map (kbd "C-c p g") #'consult-git-grep)
    map)
  "Personal keymap.")

(defun my-duplicate-line ()
  "Duplicate current line."
  (interactive)
  (let ((content (buffer-substring-no-properties
                  (line-beginning-position)
                  (line-end-position))))
    (save-excursion
      (end-of-line)
      (newline)
      (insert content))))

(defun my-move-line-up ()
  "Move line up."
  (interactive)
  (let ((col (current-column)))
    (save-excursion (transpose-lines 1))
    (forward-line -1)
    (move-to-column col)))

(defun my-move-line-down ()
  "Move line down."
  (interactive)
  (let ((col (current-column)))
    (save-excursion
      (forward-line 1)
      (transpose-lines 1))
    (forward-line 1)
    (move-to-column col)))

(defun my-duplicate-file (new-path)
  "Save a copy of current file as NEW-PATH."
  (interactive "FCopy to: ")
  (write-region (point-min) (point-max) new-path)
  (find-file new-path))

(define-minor-mode my-keys-mode
  "Personal key bindings."
  :lighter " MK"
  :keymap my-keys-mode-map
  :group 'my-extensions)

(define-globalized-minor-mode global-my-keys-mode
  my-keys-mode
  my-keys-mode)

(provide 'my-keys-mode)
;;; my-keys-mode.el ends here
```

### Part 12: 测试 (1 小时)

ERT(Emacs Lisp Regression Test)是 Emacs 内置的测试框架。为 my-keys-mode 写测试——验证"复制行真的复制了"、"上移行真的上移了"、"keymap 存在且绑定了预期键"。测试让你的 init.el 有"质量保证",以后改配置时能立即发现 regression:

为 my-keys-mode 和 minor mode 写测试:

```elisp
;;; my-keys-mode-test.el -*- lexical-binding: t; -*-

(require 'ert)
(require 'my-keys-mode)

(ert-deftest my-duplicate-line-test ()
  (with-temp-buffer
    (insert "hello\n")
    (goto-char 1)
    (my-duplicate-line)
    (should (string= (buffer-string) "hello\nhello\n"))))

(ert-deftest my-move-line-up-test ()
  (with-temp-buffer
    (insert "line1\nline2\n")
    (goto-char 8)              ; line2 的 line 开始
    (my-move-line-up)
    (should (string= (buffer-string) "line2\nline1\n"))))

(ert-deftest my-keys-mode-map-test ()
  (should (keymapp my-keys-mode-map))
  (should (lookup-key my-keys-mode-map (kbd "C-c d"))))

(provide 'my-keys-mode-test)
```

---

## 评分

### 文件结构 (5 分)
- [ ] early-init.el
- [ ] init.el
- [ ] custom.el
- [ ] lisp/ 目录
- [ ] ≥ 5 个 init-*.el 模块

### 内容 (10 分)
- [ ] init-basics
- [ ] init-ui
- [ ] init-completion
- [ ] init-prog
- [ ] init-org
- [ ] init-git
- [ ] my-keys-mode
- [ ] my-editing-mode
- [ ] 加载 my-todo (Module 3)
- [ ] use-package 用得优雅

### 功能 (5 分)
- [ ] `emacs --debug-init` 无错误
- [ ] 启动时间 < 3 秒
- [ ] 30+ 个包安装
- [ ] ERT 测试通过
- [ ] 命令在工作

### 总分

20 分。15+ 合格。

---

## 高级挑战

完成基础 capstone 后,这些挑战把你推向"Emacs 高级用户"的领域。

### 挑战 1: 启动优化

目标: 启动 < 1 秒。这听起来不可能(默认 Emacs 启动 3-5 秒),但通过几个技巧可以达到。

技巧:
- early-init.el 里 `gc-cons-threshold` 临时调高
- use-package `:defer t` 异步加载
- `package-quickstart t`
- `use-package-statistics` 找慢包

### 挑战 2: org-babel literate 配置

把 init.el 改成 `init.org`——用 Org mode 的"文学编程"特性写配置。每个配置在一个 org 章节里,带说明文档,然后 `org-babel-tangle` 自动生成 init.el。这种方式让配置"自我文档化",适合公开分享:

把 init.el 改成 `init.org`:
- 每个配置在 org 章节
- 用 `C-c C-v t` (org-babel-tangle) 生成 init.el
- 文档化更友好

### 挑战 3: 把 init.el 放 GitHub

把你的 dotfiles 放 GitHub,这是 Emacs 社区的传统。你不仅能备份,还能展示你的配置哲学,让别人学习。许多 Emacs 用户的 dotfiles 有几百 star:

- 创建 dotfiles repo
- 用 `org-roam` 或 git 管理
- 写 README

### 挑战 4: chemacs2 (多配置)

`chemacs2` 是一个"配置切换器"——让你在多个 init.el 之间切换(开发 / 工作 / 实验)。适合"工作配置不能动,但想试新东西"的场景:

`chemacs2` 让你切换多个 init.el (开发 / 工作 / 实验)。

### 挑战 5: NixOS / Guix 管理

用 Nix 或 Guix 声明式管理 Emacs 配置——整个系统(包括 Emacs 和包)都用配置文件描述,可重现。这是 DevOps 思维的极致:

用 Nix 或 Guix 声明式管理 Emacs 配置。

---

## 反思

完成 Module 4,你应该:

- 有 500+ 行模块化 init.el
- 30+ 个包用 use-package 管理
- 自定义 minor mode 提供 personal keys
- ERT 测试基础
- 启动优化
- 美观的 UI

你的 init.el 不再是"配置",是**作品**。

后面 Module 5-8 学:
- 用 Org/Magit 工作 (Module 5)
- 攻克 Elisp Reference (Module 6)
- 发布包 (Module 7)
- 贡献开源 (Module 8)

---

## 日志

写 `logs/module-04.md`:

```markdown
# Module 4 学习日志

**日期**: 2026-XX-XX
**用时**: ___ 小时
**完成度**: ___%

## 配置结构

- 文件数:
- 总行数:
- 装的包数:
- 启动时间:

## 关键学习

- use-package:
- keymap:
- hook:
- defcustom:
- minor mode:

## 挑战与解决

- ...
- ...

## 我的 GitHub

[dotfiles 链接]

## 下一步

进入 Module 5: Power User 工作流
```

---

更新 `PROGRESS.md`: Module 4 ✅。

进入 Module 5: `05-power-user/README.md`。
