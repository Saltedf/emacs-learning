# Module 4: 配置体系 + Minor Mode

> **目标**: 写出有自己风格的模块化 init.el,定义一个 minor mode
> **时长**: 2 周 (~40 小时)
> **难度**: ★★★★☆
> **依赖**: Module 3 (能写 defun/let/if,会闭包)
> **核心产出**: 500 行模块化 init.el + 自定义 minor mode

---

## 0. 这个模块在学什么

你已经学了 Module 3 的 Chassell,会写函数。但你的 init.el 可能还是 Module 0 的 10-20 行简单配置——一堆 `setq` 散落在一个文件里,既不优雅,也无法扩展。

这个模块的目标是把你的 init.el 从"配置文件"升级为"模块化作品"。你要掌握四件事: 第一,把配置拆分成多个文件,用 `require` 串起来;第二,用 `use-package` 把每个包的"安装、加载、配置、绑键、hook"封装成一段声明;第三,理解 Emacs 的扩展单元——minor mode——它由变量、函数、keymap、lighter 四件套组成;第四,学会用 hook、keymap、customize 这三套基础设施让代码解耦。

完成后,你的 init.el 不再是"配置",而是**作品**: 别人能读懂,你能维护,可以放在 GitHub 上展示你的 Emacs 哲学。这个模块也是后续 Module 5-8 的地基——没有它,后面学 Org / Magit / 包发布都会卡在"配置乱"这一关。

---

## 1. 第一性原理: init.el 就是 Elisp

### 1.1 没有"配置文件"概念

理解 Emacs 的配置,要从根上扭转一个观念。VS Code 用 `settings.json`,Sublime 用 JSON,Vim 用 `.vimrc` 脚本——这些编辑器把"配置"和"代码"分开,因为它们的底层不是解释器。Emacs 不同: 它本身就是 Lisp 机器,**所有行为都通过 Elisp 函数调用实现**。

那么"配置文件"应该是什么?第一性原理推导: 用户想自定义 Emacs 行为 → 需要一段"启动时执行的逻辑" → 既然 Emacs 是 Lisp 机器,这段逻辑最自然的形式就是 Lisp → 启动时 `load` 它,逐行 eval → 配置即代码。这就是 `init.el` 的本质。

下面这段代码看起来像配置,但每一行都是函数调用:

```elisp
;; init.el
(setq make-backup-files nil)           ; 调 setq 函数
(global-display-line-numbers-mode 1)    ; 调函数
(add-hook 'prog-mode-hook #'linum-mode) ; 调 add-hook
```

启动时,Emacs `load` 你的 init.el,逐行 eval。这三行做的事情是: 调用 `setq` 改一个变量、调用 `global-display-line-numbers-mode` 开一个 mode、调用 `add-hook` 注册一个 hook。没有"配置解析"这一步——Lisp reader 直接读这些 sexp,eval 就执行。

### 1.2 为什么这样设计?

这个设计带来一个巨大的能力跃迁: **你的配置可以做任何 Lisp 能做的事**。这是静态配置永远做不到的。

回忆 Module 0 的第一性原理: **Emacs 是 Lisp 机器**。如果 init 是 JSON,就需要解析、查找、应用——每加一个功能都要写"应用层"代码;如果 init 是 Elisp,直接 eval,**所有 Lisp 能力可用**——Lisp 本身就是应用层。

具体能做什么?下面是 JSON/YAML 配置永远做不到的几件事:

- **条件配置**: 在 Mac 上和 Linux 上做不同事——`(when (eq system-type 'darwin) ...)`
- **计算值**: 根据屏幕宽度算列宽——`(setq my-line-width (* 80 (display-pixel-width)))`
- **定义函数**: 在 init 里直接定义函数,立即调用——比"声明"+"调用"两步走自然
- **包加载**: `(require 'foo)` 加载包——和写普通代码没区别
- **错误处理**: `(condition-case err ... (error (message ...)))` 让某段配置失败不影响其他

这意味着你的 init.el 是**程序**,有控制流、有抽象、有复用。VS Code 的 settings.json 给不了你这些。

### 1.3 init.el 加载流程

启动顺序值得记住,因为很多坑都出在这里(比如"为什么我的 package 在 init.el 里报 not installed"):

```
启动 Emacs
   ↓
读 early-init.el (Emacs 27+,package.el 之前)
   ↓
初始化 package.el (除非 -Q)
   ↓
读 init.el (~/.config/emacs/init.el 或 ~/.emacs.d/init.el)
   ↓
运行 emacs-startup-hook
   ↓
显示 startup screen (除非 inhibit)
```

注意 early-init.el 在 package.el **之前**跑——这意味着你想优化启动时的 GC、frame 默认外观,必须放这里,不能放 init.el(那时 frame 已经建好了)。

### 1.4 early-init.el

early-init.el 是 Emacs 27 (2019) 引入的。它的存在解决一个先有鸡先有蛋的问题: 有些设置必须在"创建第一个 GUI frame 之前"生效(比如 tool-bar 是否显示),但 init.el 加载时 frame 已经建好了。所以 Emacs 加了一个更早的入口。

下面这段配置做四件事: 关闭启动时的包初始化(我们自己手动管)、开 quickstart(预生成加载代码加速)、禁止启动时 frame resize(闪屏)、临时禁用 GC(让启动跑得快):

```elisp
;;; early-init.el -*- lexical-binding: t; -*-

;; 在 package.el 之前跑,优化启动

(setq package-enable-at-startup nil)    ; 我们手动管
(setq package-quickstart t)             ; 生成 quickstart 加速
(setq frame-inhibit-implied-resize t)   ; 不在启动时 resize
(setq gc-cons-threshold most-positive-fixnum)  ; 启动时不 GC

;; GUI 设置 (尽早,避免闪)
(push '(menu-bar-lines . 0) default-frame-alist)
(push '(tool-bar-lines . 0) default-frame-alist)
(push '(vertical-scroll-bars) default-frame-alist)
```

判断规则很简单: **"在创建第一个 frame 前必须设的"放 early-init.el**(frame 外观、GC、包管理),其他放 init.el。一个常见的错误是把所有 setq 都丢进 early-init,这是过度——early-init 应该保持精简,因为它在 Emacs 最脆弱的启动阶段跑。

---

## 2. 这个模块的内容

本模块按"从基础设施到抽象层"的顺序展开: 先学包管理(没有包就没有生态),再学如何把单文件拆成模块(代码组织),然后学三大基础设施 keymap / hook / customize(Emacs 的扩展协议),最后把这些组合起来写自己的 minor mode。

### 2.1 包管理 (Day 1-2)
- package.el
- use-package
- straight.el (可选)

### 2.2 模块化 init (Day 3-5)
- 文件结构
- require / load / autoload
- 字节编译

### 2.3 keymap (Day 6-7)
- global / local / minor mode keymap
- define-key / kbd
- 修改内置 keymap

### 2.4 hook (Day 8-9)
- hook 类型
- add-hook / remove-hook
- 自定义 hook

### 2.5 customize (Day 10)
- defcustom / defgroup
- customize UI

### 2.6 minor mode (Day 11-13)
- define-minor-mode
- lighter / keymap
- 全局 vs 局部

### 2.7 capstone (Day 14)
- 500 行模块化 init.el + 自定义 minor mode

---

## 3. 包管理

### 3.1 package.el (内置)

package.el 是 Emacs 24 (2012) 起内置的包管理器。它的存在改变了 Emacs 生态——在那之前装包要手动下载 .el 放到 load-path,版本冲突、依赖混乱是日常。package.el 引入了"archive(仓库)+ 元数据 + 自动依赖"的模型,类似 apt / npm。

下面这段配置做三件事: 注册三个主流 archive(ELPA 是 GNU 官方严格审核,NonGNU ELPA 是 FSF 维护的非 GNU 包,MELPA 是社区最大的源)、初始化包系统、首次运行时刷新索引(下载包列表缓存到本地):

```elisp
(require 'package)

(setq package-archives
      '(("gnu"    . "https://elpa.gnu.org/packages/")
        ("nongnu" . "https://elpa.nongnu.org/nongnu/")
        ("melpa"  . "https://melpa.org/packages/")))

(package-initialize)
(unless package-archive-contents
  (package-refresh-contents))
```

安装一个包就是把它下载到 `~/.config/emacs/elpa/`,加到 load-path,activate。你可以用 Elisp 调用 `package-install`,也可以走交互式命令:

```elisp
(package-install 'magit)
```

或 M-x:
```
M-x package-install RET magit RET
```

### 3.2 use-package

写多了 package-install + require + add-hook + setq 后,你会发现每个包的"配置段落"都长得差不多,但散落 init.el 各处,可读性差,而且 require 让所有包在启动时立即加载——慢。

`use-package` 是宏,把"加载 + 配置 + keymap + hook"打包成声明式 DSL。它由 John Wiegley 在 2012 年创建——John 后来在 2015-2021 年间担任 Emacs 的 maintainer。use-package 解决了一个真实痛点: 之前的 init.el 是 spaghetti,加载顺序敏感,性能差。它把"装包、配置、绑键、hook"封装成声明式 DSL,让 init.el 从"代码"变成"配置"。2022 年 Emacs 29 把 use-package 内置,承认它是社区标准。

下面这段看起来像配置,但**实际是宏调用**——展开后等价于"autoload magit-status、绑 C-x g、加载后跑 config body":

```elisp
(use-package magit
  :ensure t                           ; 自动安装
  :bind (("C-x g" . magit-status)
         ("C-x G" . magit-dispatch))
  :config
  (setq magit-repository-directories '(("~/projects" . 2)))
  (message "Magit loaded!"))
```

这一段等于:
- 如果没装,装 magit
- 加载 magit (但**只在按 C-x g 时**——这就是 use-package 的延迟加载魔法)
- 绑 C-x g 和 C-x G
- 加载后跑 config body

注意 `:bind` 不是"立即绑",而是"声明这个命令需要 autoload"——Emacs 在你真按 C-x g 时才加载 magit。这是 Lisp 宏的力量: 你定义一个 DSL(Domain Specific Language),用 DSL 写配置,既清晰又高效。

### 3.3 use-package 关键字

| 关键字 | 作用 |
|---|---|
| `:ensure T` | 没装就装 |
| `:init BODY` | 加载前跑 |
| `:config BODY` | 加载后跑 |
| `:bind (PAIRS...)` | 绑键 (会自动 wrap with-config) |
| `:hook (HOOK . FUNC)` | 加 hook |
| `:mode (REGEX . MODE)` | 文件扩展名 → mode |
| `:after (FEATURES...)` | 在某些 feature 之后加载 |
| `:commands (CMDS...)` | autoload 这些命令 |
| `:custom (VAR VAL)` | 设 custom 变量 |
| `:custom-face (FACE SPEC)` | 设 face |
| `:defer N` | 延迟 N 秒加载 |
| `:demand` | 立即加载 (不 defer) |
| `:defines (VARS...)` | 声明这些变量 (避免 byte-warn) |
| `:functions (FUNCS...)` | 声明这些函数 |
| `:load-path PATH` | 加到 load-path |

### 3.4 例子: 完整 use-package

下面这段是配置 org-mode 的"教科书"用法。它一次性涵盖了: 安装、文件关联(.org 自动进 org-mode)、键位(C-c a/c/l 分别绑 agenda / capture / store-link)、自定义变量(org 目录、agenda 文件、capture 模板)、加载后初始化(启用 python 和 shell 代码块执行)。这五件事如果用传统写法,要 5 段散落代码;use-package 把它们压成一个有结构的整体。

```elisp
(use-package org
  :ensure t
  :mode ("\\.org\\'" . org-mode)
  :bind (("C-c l" . org-store-link)
         ("C-c a" . org-agenda)
         ("C-c c" . org-capture))
  :custom
  (org-directory "~/org/")
  (org-agenda-files '("~/org/inbox.org" "~/org/projects.org"))
  (org-default-notes-file "~/org/inbox.org")
  (org-capture-templates
   '(("t" "Todo" entry (file+headline org-directory "Inbox")
      "* TODO %?\n  %i\n")
     ("n" "Note" entry (file+headline org-directory "Notes")
      "* %?\n  %i\n")))
  :config
  (org-babel-do-load-languages
   'org-babel-load-languages
   '((python . t)
     (shell . t))))
```

这一段配置就涵盖了:
- 安装
- 文件关联
- 键位
- 自定义变量
- 加载后初始化

**比手写 require + add-hook + setq 干净 10 倍**——而且**延迟加载**: org 这个庞然大物在你真打开第一个 .org 文件之前不会被加载到内存。这就是为什么一个有 100 个 use-package 的 init.el 启动还能 < 2 秒的原因。

### 3.5 straight.el (可选高级)

package.el 有一个限制: 它只能从 archive 拉,不能从 git fork 拉。如果你需要给某个包打 patch,或跟踪某个未发布的分支,package.el 就力不从心了。

`straight.el` 是另一个包管理器,支持 git fork。它直接 clone git 仓库,可以指定任意 ref / branch / fork。代价是首次启动慢(clone 比 download 慢),磁盘占用大。

```elisp
(defvar bootstrap-version)
(let ((bootstrap-file
       (expand-file-name "straight/repos/straight.el/bootstrap.el" user-emacs-directory))
      (bootstrap-version 6))
  (unless (file-exists-p bootstrap-file)
    (with-current-buffer
        (url-retrieve-synchronously
         "https://raw.githubusercontent.com/radian-software/straight.el/develop/install.el"
         'silent 'inhibit-cookies)
      (goto-char (point-max))
      (eval-print-last-sexp)))
  (load bootstrap-file nil 'nomessage))

(straight-use-package 'use-package)
(setq straight-use-package-by-default t)
```

(MELPA 之外的来源,fork 修改包等)

Module 4 先用 package.el + use-package,5 之后再考虑 straight。

---

## 4. 模块化 init.el

### 4.1 文件结构

当 init.el 超过 200 行,单文件就不可维护了——你找不到某个配置在哪,改一处影响多处。解决方法: 按主题拆分文件。下面是一个典型的目录结构,每个 `init-*.el` 对应一个主题(basics / ui / editing / prog / org / git):

```
~/.config/emacs/
├── early-init.el               ; 启动前
├── init.el                     ; 入口
├── custom.el                   ; customize 自动写
└── lisp/                       ; 模块化配置
    ├── init-basics.el
    ├── init-ui.el
    ├── init-editing.el
    ├── init-prog.el
    ├── init-org.el
    ├── init-git.el
    └── my-extensions.el        ; 你的自定义 (如 my-todo)
```

这种"按主题拆分"的模式来自 Unix 哲学: 每个文件做一件事,做的好。如果哪天你换工作不用 org 了,删 `init-org.el` 就行,其他文件不受影响。

### 4.2 init.el

init.el 的职责变成"协调者"——加 load-path、定义全局常量、按顺序 require 各模块。它本身几乎没有配置,所有具体配置都下放到 lisp/ 下的模块。这种结构的好处: 想加新功能就新建一个 init-xxx.el,init.el 加一行 require,改动局部化。

```elisp
;;; init.el -*- lexical-binding: t; -*-

;; 加载路径
(add-to-list 'load-path (expand-file-name "lisp" user-emacs-directory))

;; 常量
(defconst *is-mac* (eq system-type 'darwin))
(defconst *is-linux* (eq system-type 'gnu/linux))

;; 包管理
(require 'init-package)

;; 各模块
(require 'init-basics)
(require 'init-ui)
(require 'init-editing)
(require 'init-prog)
(require 'init-org)
(require 'init-git)
(require 'my-extensions)

;; 启动消息
(message "Emacs loaded in %s" (emacs-init-time))

(provide 'init)
```

末尾的 `(provide 'init)` 标记"init feature 已加载",这样别人可以 `(require 'init)`(虽然实际很少这么用,但这是 Elisp 模块的协议)。

### 4.3 init-basics.el 例子

每个模块文件遵循同样的模板: 文件头 `lexical-binding: t`(强制词法作用域,Module 3 讲过)、配置代码、文件尾 `provide`。下面这个 init-basics.el 是"基础编辑环境"——身份信息、缩进、备份、编码、几个常用 minor mode、几个键位。结构清晰,每个 setq 都做了什么一目了然。

```elisp
;;; init-basics.el -*- lexical-binding: t; -*-

(setq user-full-name "Your Name"
      user-mail-address "you@example.com")

(setq-default
 indent-tabs-mode nil
 tab-width 4
 fill-column 80
 truncate-lines t)

(setq
 inhibit-startup-screen t
 initial-scratch-message nil
 ring-bell-function 'ignore
 make-backup-files nil
 auto-save-default nil
 create-lockfiles nil)

(set-language-environment "UTF-8")
(set-default-coding-systems 'utf-8-unix)

(recentf-mode 1)
(savehist-mode 1)
(winner-mode 1)

(global-set-key (kbd "C-c r") 'recentf-open-files)

(provide 'init-basics)
;;; init-basics.el ends here
```

注意 `setq-default` vs `setq`: 前者设"buffer-local 变量的全局默认值"(像 indent-tabs-mode 这种变量是 buffer-local 的,setq 只影响当前 buffer,setq-default 才影响所有 buffer 的默认)。这种细节是 Emacs 老用户的肌肉记忆。

### 4.4 require / load / autoload

Emacs 有四种加载代码的方式,各有用场。理解它们的差异是避免"重复加载"和"加载顺序"坑的关键:

```elisp
(require 'foo)                  ; 加载 foo,如果还没加载
(require 'foo "path/to/foo")    ; 指定文件
(load "foo")                    ; 加载 foo.el (扩展名可省)
(load-file "~/path.el")         ; 加载绝对路径
(autoload 'foo-func "foo")      ; 延迟: foo-func 被调用时才加载 foo
```

**规则**(这套规则是 Lisp "feature" 机制的核心):
- `provide` 在文件末尾: 标记该 feature 已加载(把 symbol 加到 `features` 列表)
- `require`: 加载 feature (如果还没在 `features` 里);如果在,跳过——这就是"只加载一次"的保证
- `load`: 直接加载文件,不管 feature 状态(可能重复加载)
- `autoload`: 延迟加载——给一个"未来要加载的承诺",真调用时才 load

`require` 是最常用的——它保证了"加载且只加载一次",这正是模块化的核心需求。

### 4.5 字节编译

Lisp 是解释执行,慢。Emacs 提供 byte-compile,把 .el 编译成 .elc(字节码),加载更快、运行也更快。现代 Emacs(28+)还有 native comp(gccemacs),编译成原生机器码,性能接近 C。

byte-compile 把 .el 编译成 .elc,加载更快。

```bash
emacs --batch -f batch-byte-compile foo.el
```

或在 Emacs:
```
M-x byte-compile-file RET foo.el RET
```

native comp (Emacs 28+) 进一步编译成原生码:
```
M-x native-compile-async RET
```

(Module 6 W3 详细讲)

---

## 5. keymap

### 5.1 keymap 类型

keymap 是 Emacs 把"按键序列"映射到"命令"的数据结构。Emacs 同时维护多个 keymap,按优先级查找。理解 keymap 类型对绑键、调试"键位不工作"至关重要:

```elisp
global-map                   ; 全局
(current-local-map)          ; 当前 major mode 的 keymap
(my-minor-mode-map)          ; 你的 minor mode 的 keymap
 Control-X-prefix            ; C-x 前缀
```

minor mode 的 keymap 优先级**高于** major mode 的 keymap——这就是为什么你能用 minor mode "覆盖" major mode 的默认键位(详见 `keymaps.md`)。

### 5.2 define-key

`define-key` 是绑键的原语。`global-set-key` / `local-set-key` 都是对它的包装:

```elisp
(define-key KEYMAP KEY COMMAND)

;; 例:
(global-set-key (kbd "C-c d") #'my-func)
;; 等于:
(define-key global-map (kbd "C-c d") #'my-func)
```

### 5.3 kbd 函数

`kbd` 把人类可读的字符串("C-c d")解析成 Emacs 内部的事件 vector。直接写 vector 太痛苦(要记得转义、modifier 顺序),kbd 让你用自然语法:

```elisp
(kbd "C-c d")                ; → [?\C-c ?d]
(kbd "M-x")                  ; → [?\M-x]
(kbd "<f5>")                 ; → [<f5>]
(kbd "C-c <return>")         ; → [?\C-c return]
(kbd "C-M-s")                ; → [?\C-\M-s]
```

### 5.4 修改 keymap

```elisp
;; 解绑
(global-unset-key (kbd "C-z"))

;; 改绑
(global-set-key (kbd "C-z") #'undo)

;; 查 keymap
(substitute-key-definition 'undo nil global-map)
```

### 5.5 在 minor mode 里用

minor mode 启用时,它对应的 keymap 会被自动加到 active keymaps 列表里。下面是定义一个 minor mode keymap 的标准模板:

```elisp
(defvar my-mode-map
  (let ((map (make-sparse-keymap)))
    (define-key map "a" #'my-mode-action-a)
    (define-key map "b" #'my-mode-action-b)
    map))
```

minor mode 启用时,这个 keymap 加到 active keymaps。这就是为什么 minor mode 的键位优先级高——它排在 major mode 之前被查找。

---

## 6. hook

### 6.1 什么是 hook

hook 是一个**变量,值是函数列表**。某些事件发生时(保存文件、进入某 mode、执行命令),Emacs 会遍历这个列表调用每个函数。这个简单的数据结构是 Emacs 解耦的基石: 模块 A 定义 hook,模块 B 监听,A 和 B 互不依赖。

```elisp
(defvar before-save-hook nil
  "Normal hook run before each save.")

(add-hook 'before-save-hook #'delete-trailing-whitespace)
;; before-save-hook 现在是 (#'delete-trailing-whitespace)
```

### 6.2 hook 命名

Emacs 约定用 `-hook` 或 `-functions` 结尾,这样用户一眼能认出"这是 hook":
- `<mode>-hook`: 进入该 mode 时运行
  - `prog-mode-hook` (所有编程 mode)
  - `python-mode-hook`
  - `org-mode-hook`
- `before-<event>-hook`、`after-<event>-hook`
  - `before-save-hook`
  - `after-save-hook`
  - `after-init-hook`
  - `emacs-startup-hook`

### 6.3 add-hook

`add-hook` 把函数加到 hook 列表。第四个参数 LOCAL 控制是否只加到当前 buffer——这是一个常被忽略但很重要的特性:

```elisp
(add-hook HOOK FUNCTION &optional DEPTH LOCAL)

(add-hook 'prog-mode-hook #'linum-mode)        ; 全局 prog-mode
(add-hook 'python-mode-hook #'my-python-setup) ; 全局 python
(add-hook 'python-mode-hook #'my-local-setup nil t)  ; buffer-local
```

### 6.4 remove-hook

```elisp
(remove-hook 'prog-mode-hook #'linum-mode)
```

### 6.5 自定义 hook

你也可以定义自己的 hook,让别人扩展你的代码。这是写"可扩展包"的标准模式:

```elisp
(defvar my-todo-saved-hook nil
  "Hook run after todos are saved.")

(defun my-todo-save ()
  ...
  (run-hooks 'my-todo-saved-hook))
```

别人可以:
```elisp
(add-hook 'my-todo-saved-hook #'my-sync-to-cloud))
```

### 6.6 hook 的 LOCAL 选项

buffer-local hook 只影响当前 buffer——这是 Emacs"buffer 是宇宙中心"哲学的体现:

```elisp
(add-hook 'write-file-functions #'my-special-write nil t)
```

第 4 个参数 t = buffer-local。

---

## 7. customize

### 7.1 defgroup / defcustom

customize 是 Emacs 内置的"用户配置系统"——你可以通过 GUI 改设置,也可以通过 defcustom 在代码里声明可配置选项。它和 setq 的区别: customize 有类型检查、有 UI、改值时能触发副作用(下面会看到)。这种设计让"小白用户"也能用 Emacs,而不用学 Lisp:

```elisp
(defgroup my-app nil
  "My application settings."
  :group 'tools
  :prefix "my-app-")

(defcustom my-app-name "default"
  "Application name."
  :type 'string
  :group 'my-app)

(defcustom my-app-size 100
  "Application size in pixels."
  :type 'integer
  :group 'my-app)

(defcustom my-app-theme 'dark
  "Application theme."
  :type '(choice (const dark) (const light))
  :group 'my-app)
```

### 7.2 type specifiers

`:type` 接受:

| 类型 | 含义 |
|---|---|
| `string` | 字符串 |
| `integer` | 整数 |
| `number` | 数字 (含浮点) |
| `boolean` | t / nil |
| `(const VALUE)` | 固定值 |
| `(choice T1 T2...)` | 选择 |
| `(radio T1 T2...)` | 单选 (radio button) |
| `(repeat TYPE)` | list |
| `(alist :key-type K :value-type V)` | alist |
| `(file)` | 文件路径 |
| `(directory)` | 目录 |
| `(color)` | 颜色 |
| `(face)` | face |
| `(hook)` | hook |
| `(variable)` | 变量名 |
| `(function)` | 函数名 |
| `(symbol)` | symbol |
| `(sexp)` | 任意 sexp |

### 7.3 customize UI

customize 不仅能在代码里声明,还能用 GUI 改——这对不熟悉 Lisp 的用户极友好。打开 customize 界面后,你能看到每个选项的类型、当前值、应用按钮,一切可视化:

```
M-x customize-group RET my-app RET
```

打开 customize 界面,可视化改。

保存:
- `(customize-save-variable 'VAR VALUE)` 直接保存
- customize UI 的 "Save for future sessions"

但 customize 默认会把改动写回 init.el 或一个 custom 文件——如果写回 init.el 会污染你的代码。最佳实践: 单独设一个 custom-file,让 customize 只往那里写:

写到 `custom-file`:
```elisp
(setq custom-file (expand-file-name "custom.el" user-emacs-directory))
(load custom-file t)
```

### 7.4 为什么用 customize?

customize 比 setq 强在四点: 用户友好(有 GUI 不需要懂 Lisp)、类型检查(设错类型会警告)、自动持久化(改了就存,不用记着写回文件)、`:set` trigger(改值时能触发副作用代码)。最后一点尤其强大——它让你能写"响应式"配置:

- 用户友好 (不需要懂 Lisp)
- 类型检查
- 自动持久化
- `:set` trigger (改值时跑代码)

下面这个例子: 用户改 `my-app-theme` 时(无论通过 UI 还是 setq),setter 都会被调用,自动应用主题。这就是"配置即代码"的延伸——配置变化能驱动代码执行:

```elisp
(defcustom my-app-theme 'dark
  "Theme."
  :type '(choice (const dark) (const light))
  :set (lambda (var val)
         (set var val)
         (my-app-apply-theme val)))
```

---

## 8. define-minor-mode

### 8.1 minor mode 是 4 个东西

Minor mode 是 Emacs 的"特性开关"。它由 4 个东西组成: 一个变量(t/nil)、一个 toggle 函数、一个 keymap、一个 lighter(mode line 显示)。这看起来简单,但背后是 Emacs 设计的精髓——**所有"特性"都是同构的**。装包、开行号、自动配对括号、flyspell——它们都遵循同样的模式: 一个 minor mode。这就是为什么 Emacs 用户能"装 100 个包还不乱"——每个包都是一个 minor mode,统一接口。

对比 VS Code: 每个扩展有自己的 API、自己的 UI、自己的状态管理。Emacs 强制大家用同一套 minor mode 协议,结果是互操作性极高——你的 minor mode 可以和其他人的 minor mode 共存,不会冲突。这是 40 年沉淀的协议设计。

`define-minor-mode` 是一个宏,你给它一个名字和文档,它自动生成上面这 4 件套:

```elisp
(define-minor-mode my-mode
  "Toggle My mode."
  :init-value nil
  :lighter " My"
  :keymap my-mode-map
  :group 'my-app
  (if my-mode
      (progn
        ;; 启用时
        (message "My mode enabled"))
    ;; 禁用时
    (message "My mode disabled")))
```

`define-minor-mode` 自动:
1. 创建变量 `my-mode` (t/nil)
2. 创建函数 `my-mode` (toggle)
3. 创建 lighter (`:lighter`)
4. 加 keymap
5. 加到 minor-mode-list

这意味着你写一行宏调用,就得到了一整套 mode 基础设施——变量、函数、keymap、lighter、hook、customize group。这种"用宏生成模板代码"是 Lisp 的典型用法。

### 8.2 global minor mode

普通 minor mode 是 buffer-local 的(只影响当前 buffer)。但有些场景需要"全局开关",比如 `linum-mode`、`hl-line-mode`——你想让它们在所有 buffer 都生效。这时用 `:global t`:

```elisp
(define-minor-mode my-mode
  "..."
  :global t
  :lighter " My"
  :keymap my-mode-map
  ...)
```

`:global t` 让它影响所有 buffer,不只当前。注意 `:global t` 和 `define-globalized-minor-mode` 是两个不同的概念(详见 minor-mode-tutorial.md)。

### 8.3 实战: 一个有用的 minor mode

下面是一个真实有用的 minor mode: 把你的个人键位打包成一个 minor mode,这样既不会污染 global-map,又能在所有 buffer 生效。用 `define-globalized-minor-mode` 让它全局启用:

```elisp
(defvar my-keys-minor-mode-map
  (let ((map (make-sparse-keymap)))
    (define-key map (kbd "C-c d") #'my-duplicate-line)
    (define-key map (kbd "C-c r") #'my-rename-file-and-buffer)
    map)
  "Keymap for `my-keys-minor-mode'.")

(define-minor-mode my-keys-minor-mode
  "Toggle my custom keys."
  :init-value t
  :lighter " MK"
  :keymap my-keys-minor-mode-map)

(define-globalized-minor-mode my-keys-global-mode
  my-keys-minor-mode
  my-keys-minor-mode)

(my-keys-global-mode 1)    ; 全局启用
```

这样你的自定义键位不会被 major mode 覆盖 (因为 minor mode 优先级高)。这是几乎所有 Emacs 老用户的标配——把个人键位集中在一个 minor mode 里,而不是污染 global-map。

---

## 9. 毕业检查

### 概念题

1. init.el 和 early-init.el 区别?
2. `require` 和 `load` 区别?
3. `use-package` 的 :init 和 :config 区别?
4. global keymap 和 minor mode keymap 优先级?
5. hook 是什么数据结构?
6. defcustom 和 defvar 区别?
7. minor mode 由哪 4 个东西组成?
8. global minor mode 和普通 minor mode 区别?

### 实操题

1. 把 init.el 模块化到 5 个文件
2. 用 use-package 配置 5 个包
3. 定义自己的 minor mode,绑 5 个键
4. 用 defcustom 加 3 个可配置选项
5. 用 hook 在保存时自动跑 lint

---

## 10. 下一步

打开 `concept-anchor.md` 看 editor manual 对应章节。
然后 `ref-deep-dive.md` 看 Lisp Reference 的深度。
然后 `use-package.md` / `keymaps.md` / `hooks.md` / `minor-mode-tutorial.md` 学各项。
最后 `capstone.md` 做毕业项目。
