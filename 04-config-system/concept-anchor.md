# Concept Anchor: Editor Manual (Module 4)

> 替代 `emacs-manual-30.2/custom.texi`、`modes.texi`、`package.texi`、`cmdargs.texi`
> 用户视角看 customization / modes / packages / command-line

---

## 1. custom.texi — 用户视角的 Customization

### 1.1 Easy Customization

Emacs 的"配置"有两个层面: 一个是写 init.el 的 Elisp 代码(程序员视角),另一个是通过 GUI 改设置(普通用户视角)。后者叫 **Easy Customization** 系统——Emacs 把所有可配置选项建模成"有类型的变量"(defcustom),并提供一个统一的 UI 去改它们。

这个系统对 Emacs 普及很重要: 不懂 Lisp 的用户也能用 Emacs,通过菜单改设置,自动持久化。VS Code、Sublime、IntelliJ 都有类似的 settings UI——但 Emacs 的独特之处在于: 它的 UI 后面是 Elisp,UI 改值最终调用 `customize-set-variable`,等价于 `setq` + 类型检查 + 持久化。

Emacs 内置 **Easy Customization** 系统:
- 用户通过 GUI 改设置
- 自动持久化到 custom-file
- 类型安全

下面四个命令覆盖了 customize 的入口: 顶层菜单、按 group 改、按单个 option 改、按 face(颜色字体)改。`M-x customize` 会打开一个分类菜单,你按 group 浏览:

```
M-x customize             ; 顶层
M-x customize-group RET NAME RET
M-x customize-option RET VAR-NAME RET
M-x customize-face RET FACE-NAME RET
```

### 1.2 一个例子

下面演示 `M-x customize-option` 改一个具体设置(`line-number-mode`,控制 mode line 是否显示行号)。注意 UI 给出的"状态"信息——它会告诉你这个值是默认值、已修改未保存,还是已保存。这是 setq 给不了的反馈:

```
M-x customize-option RET line-number-mode RET
```

显示:
```
Line Number Mode: [ ] Hide
   Toggle display of line numbers in the mode line.
   State: SAVED.
   [State]  [Apply and Save]  [Apply]  ...

Groups: [Convenience]
```

- 改 State: 点击 toggle
- "Apply and Save": 写到 custom-file,永久生效
- "Apply": 当前 session 生效,不保存

注意"Apply vs Apply and Save"的区别——前者只改当前 session(关闭 Emacs 就丢),后者写到 custom-file(永久)。这种区分是 Emacs 老用户熟悉的。

### 1.3 Lisp 视角

customize UI 后面是 Elisp。每个可配置选项在代码里是 `defcustom`(不是 defvar)——它多了类型信息、group 归属、可选的 setter。下面三行: 第一行是 `line-number-mode` 的源定义,第二行用 Elisp 等价"在 UI 改并保存",第三行设置 custom-file 让 customize 写到固定文件不污染 init.el:

```elisp
(defcustom line-number-mode t
  "..."
  :type 'boolean
  :group 'mode-line)

(customize-save-variable 'line-number-mode nil)
;; 等于用 UI 改,自动保存

(setq custom-file "~/.config/emacs/custom.el")
(load custom-file t)
```

---

## 2. modes.texi — 用户视角的 Modes

### 2.1 Major Modes

Emacs 把"buffer 类型"抽象成 major mode。每个 buffer 同时有且只有一个 major mode,它决定三件事: 语法高亮规则、键位、缩进规则。这是 Emacs 对"如何处理不同文件类型"的核心抽象——比 VS Code 的"language id"深入得多,因为 major mode 还能定义自己的 keymap、hook、缩进函数、font-lock 规则。

每个 buffer 有一个 major mode:
- 决定语法高亮
- 决定键位
- 决定缩进规则

切换 major mode 可以手动,也可以让 Emacs 根据文件扩展名自动选(下面会讲):

切换:
```
M-x python-mode RET
M-x org-mode RET
M-x markdown-mode RET
```

或自动: 文件扩展名映射 (auto-mode-alist)。

### 2.2 auto-mode-alist

`auto-mode-alist` 是一个"文件扩展名 → major mode"的映射表。打开文件时,Emacs 用这个表决定进哪个 mode。下面是手动扩展这个表的传统写法,以及 use-package 的声明式写法:

```elisp
(setq auto-mode-alist
      (append '(("\\.py\\'" . python-mode)
                ("\\.md\\'" . markdown-mode)
                ("\\.org\\'" . org-mode))
              auto-mode-alist))
```

或用 use-package(更简洁,且自带延迟加载):
```elisp
(use-package markdown-mode
  :mode ("\\.md\\'" . markdown-mode))
```

### 2.3 Minor Modes

minor mode 是叠加在 major mode 上的"特性开关"。一个 buffer 只有一个 major mode,但可以有任意多个 minor mode 同时启用——行号、拼写检查、自动换行、snippet、补全,各是一个 minor mode。这种"主从+叠加"的模型让 Emacs 能装 100 个包还不冲突。

附加功能,可以同时多个:

```
M-x linum-mode RET          ; 行号
M-x flyspell-mode RET       ; 拼写检查
M-x auto-fill-mode RET      ; 自动换行
```

查看当前 buffer 有哪些 mode 在跑,用 `C-h m`——它会列出 major mode 和所有 active minor mode,以及它们的 keymap。这是排查"键位不工作"的神器。

查看 active minor modes: `C-h m`。

### 2.4 hook

每个 mode(major 或 minor)都有一个对应的 hook——进入该 mode 时 Emacs 会调用 hook 里的所有函数。这是 mode 提供给用户的"扩展点": mode 加载完毕后,你可以跑自己的初始化代码,改缩进、设变量、启用其他 mode。

每个 mode 有 hook:
- `<mode>-hook` 进入该 mode 时跑

下面这个例子: 每次进入 python-mode,自动设"用空格不用 tab,缩进 4 格"。这种"模式专属配置"是 hook 最常见的用法:

```elisp
(add-hook 'python-mode-hook
          (lambda ()
            (setq indent-tabs-mode nil)
            (setq python-indent-offset 4)))
```

### 2.5 Lisp 视角

在代码层面,major mode 用 `define-derived-mode` 定义(继承自父 mode,如 prog-mode),minor mode 用 `define-minor-mode` 定义。这些宏自动生成 mode 函数、keymap、hook、syntax table 等基础设施:

```elisp
(define-derived-mode python-mode prog-mode "Python"
  "Major mode for Python."
  ...)

(define-minor-mode linum-mode
  "Toggle line numbers."
  ...)
```

(Module 6 W6 详细讲)

---

## 3. package.texi — 用户视角的 Packages

### 3.1 什么是 Package

Emacs 的"扩展单元"叫 package。一个 package 是一组 .el 文件 + 一个 `*-pkg.el` 描述文件(元数据: 名字、版本、依赖、描述)。这种统一格式让 Emacs 能像 apt / npm 一样自动管理依赖——你装包 A 它依赖 B,Emacs 自动也装 B。

Package 是一个 .el 集合,加一个 `*-pkg.el` 描述。
用户可以 `M-x package-install` 装。

### 3.2 Package Archives

包从哪来?从 archive(仓库)。Emacs 社区有几个主流 archive,各有定位:

| Archive | 内容 |
|---|---|
| **ELPA** (elpa.gnu.org) | GNU 官方,严格审核 |
| **NonGNU ELPA** (elpa.nongnu.org) | FSF 维护,非 GNU 包 |
| **MELPA** (melpa.org) | 社区,最大,有些质量参差 |
| **MELPA Stable** | MELPA 稳定版 |
| **Org ELPA** | Org mode 官方 |

ELPA 最严格(必须符合 FSF 标准),MELPA 最大但质量参差。新手推荐同时配三个: ELPA + NonGNU + MELPA。

### 3.3 配置

下面这段配置把三个 archive 都加进去,然后初始化包系统。`package-initialize` 是关键——它扫描已安装包,加到 load-path,activate 它们:

```elisp
(require 'package)
(setq package-archives
      '(("gnu"    . "https://elpa.gnu.org/packages/")
        ("nongnu" . "https://elpa.nongnu.org/nongnu/")
        ("melpa"  . "https://melpa.org/packages/")))
(package-initialize)
```

### 3.4 安装

装包的三步: 刷新索引(从远程拉最新列表)、安装、查看所有可用包。第一次配 Emacs 或换 archive 后必须 refresh:

```
M-x package-refresh-contents RET    ; 刷新索引
M-x package-install RET magit RET   ; 装
M-x package-list-packages RET       ; 列所有
```

### 3.5 use-package

裸用 package-install + require + setq + add-hook 散落各处很快变乱。use-package 把"装包 + 加载 + 配置 + 键位 + hook"压成一个声明:

```elisp
(use-package magit
  :ensure t           ; 自动装
  :bind ("C-x g" . magit-status)
  :config
  (setq magit-display-buffer-function ...))
```

### 3.6 Lisp 视角

下面是 package.el 内部的关键 API——你可以用 Elisp 查"装没装"、activate、查版本。这些是写"管理包的脚本"会用到的:

```elisp
(require 'package)
(package-installed-p 'magit)
(package-activate 'magit)
```

(Module 7 W1 详细讲怎么发布包)

---

## 4. cmdargs.texi — 命令行参数

### 4.1 启动参数

Emacs 不只是 GUI 编辑器——它有强大的命令行接口,可以批处理、daemon、调试 init.el。理解这些参数对调试配置、写自动化脚本至关重要:

| 参数 | 作用 |
|---|---|
| `-q` | 不加载 init |
| `-Q` | 比 -q 更彻底 |
| `-nw` | TTY (no window) |
| `--batch` | 批处理模式 |
| `--daemon` | 后台 daemon |
| `--debug-init` | init 报错进 debugger |
| `-l FILE` | 加载 FILE |
| `--eval EXPR` | eval 表达式 |
| `-f FUNC` | 调用 FUNC |

最常用的: `emacs --debug-init`(init.el 报错时进 debugger),`emacs -Q`(快速验证"是不是我的配置导致的问题"),`emacs --batch --eval ...`(CI 脚本里跑 Elisp)。

### 4.2 环境变量

Emacs 启动时读这些环境变量,你可以用来定制加载路径、数据目录等:

| 变量 | 作用 |
|---|---|
| `EMACSLOADPATH` | 加载路径 (类似 PATH) |
| `EMACSDATA` | 数据目录 |
| `EMACSPATH` | 可执行目录 |
| `EMACSDOC` | 文档目录 |
| `INFOPATH` | info 文件路径 |
| `TMPDIR` | 临时目录 |
| `LC_ALL` 等 | locale |

### 4.3 Lisp 视角

下面是命令行参数和环境变量在 Elisp 里的对应 API。`argv` 给你原始命令行,`getenv/setenv` 读写环境变量:

```elisp
(command-line)              ; 处理命令行
(command-line-args)         ; 当前 args 列表
(argv)                      ; 原始 argv
(getenv "PATH")             ; 环境变量
(setenv "PATH" "...")       ; 设置
```

---

## 5. 实战配置案例

### 5.1 init.el 标准 200 行模板

(详见 `capstone.md`)

### 5.2 use-package 整合

下面是当代 Emacs 配置的"标配四件套": which-key(按 prefix 显示可用命令)、vertico(垂直补全 UI)、orderless(无序匹配)、consult(增强版内置命令)。这四个加起来让默认 Emacs 的 minibuffer 体验脱胎换骨:

```elisp
(use-package which-key
  :ensure t
  :diminish which-key-mode
  :config
  (which-key-mode 1))

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
         ("M-y" . consult-yank-pop)
         ("C-x b" . consult-buffer)))
```

### 5.3 自定义 minor mode (个人键位)

把所有个人键位放一个 minor mode——这是 Emacs 老用户的标配。好处: 键位集中管理,能整体开关,不会污染 global-map,优先级高于 major mode 不会被覆盖:

```elisp
(defvar my-keys-mode-map
  (let ((map (make-sparse-keymap)))
    (define-key map (kbd "M-o") #'other-window)
    (define-key map (kbd "C-c d") #'my-duplicate-line)
    (define-key map (kbd "C-c r") #'my-rename-buffer-file)
    map))

(define-minor-mode my-keys-mode
  "Personal key bindings."
  :lighter " MK"
  :keymap my-keys-mode-map
  :global t)

(my-keys-mode 1)
```

---

## 6. 自测

1. `M-x customize-option` 和 `setq` 区别?
2. major mode 和 minor mode 的关键区别?
3. hook 是什么数据结构?
4. `auto-mode-alist` 干啥?
5. package 和 ELPA 的关系?
6. `emacs -Q` 跳过什么?
7. minor mode 的 lighter 是什么?
8. global minor mode 怎么定义?

**答案**:
> 1. customize 有类型检查、UI、自动持久化;setq 直接改值
> 2. major mode 每 buffer 一个;minor mode 多个可叠加
> 3. 一个变量,值是函数列表
> 4. 文件扩展名 → major mode 的映射
> 5. package 是 .el 集合;ELPA 是 archive,装 package 的源
> 6. 跳过 init.el + site-lisp + 包默认加载
> 7. mode line 上显示 minor mode 的小字符串
> 8. `define-globalized-minor-mode`

---

## 7. 下一步

进入 `ref-deep-dive.md` 看 Lisp Reference 的深度。
