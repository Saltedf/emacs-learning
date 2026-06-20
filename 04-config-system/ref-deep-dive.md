# Ref Deep Dive: Lisp Reference (Module 4)

> 替代 `emacs-lispref-30.2/customize.texi`、`keymaps.texi`、`modes.texi` 的深度内容

---

## 1. customize.texi — 深度

### 1.1 defcustom 的完整语法

`defcustom` 看起来只是 `defvar` 加几个 keyword,但实际上它定义的是一个"有元数据的变量"——类型、setter、getter、初始化器、安全等级、版本……这些元数据让 Emacs 能为这个变量自动生成 UI、做类型检查、响应式触发副作用。这是 setq + defvar 永远做不到的。

下面是 defcustom 的完整语法。STANDARD 是默认值,DOC 是文档,后面跟任意 keyword-value 对:

```elisp
(defcustom SYMBOL STANDARD DOC [KEYWORD VALUE]...)

(defun-custom STANDARD DOC :group GROUP :type TYPE
  :set SETTER :get GETTER
  :initialize INIT
  :require FEATURE
  :version VERSION
  :tag TAG
  :link LINK
  :package-version ASSOC
  :safe PREDICATE
  :risky BOOL
  :set-after VARS)
```

### 1.2 关键 keyword

**`:type`**: 类型 spec

`:type` 是 defcustom 最 powerful 的 keyword——它声明变量的"形状",Emacs 据此生成 UI(字符串显示成输入框,boolean 显示成 checkbox,choice 显示成下拉菜单)。同时它做类型检查——设错类型 customize 会拒绝。下面是从简单到复杂的 type spec 例子:

```elisp
;; 简单
:type 'string
:type 'integer
:type 'boolean

;; 选择
:type '(choice (const :tag "Dark" dark)
               (const :tag "Light" light))

;; List
:type '(repeat string)
:type '(repeat (list (string :tag "Name")
                     (integer :tag "Age")))

;; alist
:type '(alist :key-type string :value-type integer)

;; 文件
:type 'file
:type 'directory

;; 复杂
:type '(list (string :tag "Command")
             (repeat :tag "Args" string)))
```

**`:set`**: 自定义 setter

`:set` 让你能写"响应式"配置——用户改这个变量时(无论通过 UI、customize-set-variable,还是代码),setter 都会被调用。下面这个例子: 用户改 `my-theme` 时,setter 自动应用主题。这是"配置变化驱动代码执行"的模式,Lisp 才能做到:

```elisp
(defcustom my-theme 'dark
  "Theme."
  :type '(choice (const dark) (const light))
  :set (lambda (sym val)
         (set sym val)
         (my-apply-theme val)))
```

每次 customize-set-variable 改值时,跑这个 setter。

**`:init`**: 初始化函数

有时默认值不是一个常量,而是"需要计算的初始状态"(比如一个空 hash table)。`:initialize` 让你指定一个函数来初始化:

```elisp
(defcustom my-cache nil
  "Cache, init lazily."
  :type 'sexp
  :initialize (lambda (sym val)
                (set sym (make-hash-table))))
```

**`:safe`**: 安全 local-variable 函数

Emacs 有"file-local variable"机制——文件末尾可以声明"打开此文件时设这些变量"。但这有安全风险(恶意文件可以设 `eval` 之类)。`:safe` 让你声明"这个变量作为 file-local 是安全的",并给出类型检查函数:

```elisp
(defcustom my-config-value nil
  "..."
  :type 'string
  :safe 'stringp)
```

这样该变量可以安全作为 file-local variable。

**`:risky`**: 标记风险

和 `:safe` 相反——`:ricky t` 标记"这个变量绝对不能作为 file-local"。这用于安全敏感的变量,比如会被 eval 的:

```elisp
(defcustom my-execute-on-load nil
  "..."
  :type 'sexp
  :risky t)
```

risky 变量不能作为 file-local (安全考虑)。

### 1.3 custom-face

face 是 Emacs 对"颜色 / 字体 / 下划线 / 加粗"等外观属性的抽象。defface 定义一个 face,可以让它根据"终端 vs GUI""深色 vs 浅色主题"自动选不同的外观。这是 Emacs 主题系统的基础:

```elisp
(defface my-face
  '((t :foreground "red"))
  "My face."
  :group 'my-app)
```

face 是"颜色/字体"的抽象。

### 1.4 custom variables in code

下面是处理 custom 变量的 Elisp API。写"工具脚本"时常用——比如检查某变量是不是 defcustom、读它的 type spec、强制保存所有改动:

```elisp
(custom-variable-p 'foo)             ; foo 是 defcustom?
(custom-variable-type 'foo)          ; 返回类型 spec
(custom-initialized-p 'foo)
(default-value 'foo)                  ; 默认值
(custom-save-all)                     ; 立即保存到 custom-file
```

---

## 2. keymaps.texi — 深度

### 2.1 keymap 的本质

keymap 看起来像"字典",但底层是嵌套的 vector / char-table 混合结构。Emacs 有两种 keymap 实现: full keymap(预分配所有 128 个 ASCII 槽,lookup 快但占内存)和 sparse keymap(懒分配,只存非 nil 项,内存高效但 lookup 略慢)。

下面是两种 keymap 的创建方式。`global-map` 是 full keymap(启动一次性开销,但每次按键 lookup 都快),minor mode 用 sparse keymap(因为每个 minor mode 通常只绑几个键,full 浪费):

```elisp
(make-keymap)                ; full keymap (128 char + char-table for modifiers)
(make-sparse-keymap)         ; lazy (hash table-like)
```

`global-map` 是 full keymap。
minor mode 用 sparse keymap (高效)。

### 2.2 keymap lookup

理解 keymap lookup 顺序是 debug"键位不工作"的关键。Emacs 按下面顺序查,第一个找到的 binding 生效——这就是为什么 minor mode 的键位能覆盖 major mode 的(它在 major mode 之前被查):

```
按 C-c d
   ↓
查 active keymaps (按优先级):
   1. overriding-local-map (最高,很少用)
   2. keymap text property (光标处文本的)
   3. emulated-mode-keymap
   4. overriding-terminal-local-map
   5. minor-mode-overriding-map-alist
   6. minor-mode-map-alist (所有 active minor modes)
   7. current-local-map (major mode 的)
   8. global-map (最低)
```

minor mode keymap 优先级**高于** major mode。这也是"个人键位 minor mode"模式有效的原因——把你的键位放 minor mode,不会被任何 major mode 覆盖。

### 2.3 current-active-maps

下面是查询当前 active keymap 的 Elisp API。写"键位分析工具"或调试时有用:

```elisp
(current-active-maps t)      ; 返回所有 active keymaps
(active-key-binding KEY)     ; 查 KEY 绑到啥
```

### 2.4 define-key 的细节

`define-key` 的第三个参数 DEF 可以是多种类型——command、keymap(变 prefix)、nil(解绑)、字符串(键盘宏)、menu item。这种多态让一个 API 能绑各种东西:

```elisp
(define-key KEYMAP KEY DEF)

;; DEF 可以是:
;;   command (symbol 或 function)
;;   keymap (变成 prefix)
;;   nil (解绑)
;;   string (键盘宏)
;;   list (menu item)

(define-key global-map (kbd "C-c d") #'my-func)
(define-key global-map (kbd "C-c d") nil)         ; 解绑
(define-key global-map [menu-bar tools] (cons "Tools" (make-sparse-keymap)))  ; menu
```

### 2.5 keymap inheritance

Emacs 27+ 支持让一个 keymap"继承"另一个——子 keymap 自动有父 keymap 的所有 binding,再加自己的。`set-keymap-parent` 和 `make-composed-keymap` 是两种实现方式:

```elisp
(defvar my-base-map
  (let ((m (make-sparse-keymap)))
    (define-key m "a" #'base-a)
    m))

(defvar my-derived-map
  (let ((m (make-composed-keymap nil my-base-map)))
    (define-key m "b" #'derived-b)
    m))
```

`make-composed-keymap` 让 my-derived-map 继承 my-base-map 的所有键。

### 2.6 keymap as text property

Emacs 有一个高级特性: 你可以给 buffer 中的一段文本附加 keymap——光标在这段文本上时,这些键位才生效。这是 org-mode 的链接、按钮、可点击元素背后的机制:

```elisp
(with-current-buffer "*scratch*"
  (put-text-property 1 10 'keymap
                     (let ((m (make-sparse-keymap)))
                       (define-key m [mouse-1] #'my-click)
                       m)))
```

buffer 里第 1-10 字符的鼠标点击绑到 my-click。

(Module 6 W7 详细讲)

---

## 3. modes.texi — 深度

### 3.1 define-derived-mode 完整

`define-derived-mode` 是定义 major mode 的标准宏。它自动做一堆事: 创建 mode 函数、hook、keymap、syntax table,并继承父 mode 的所有特性。这就是为什么写一个新的 major mode 通常只需要几十行代码——宏帮你生成了所有样板:

```elisp
(define-derived-mode NAME PARENT DOC &optional KEYWORD-ARGS... BODY...)

(define-derived-mode python-mode prog-mode "Python"
  "Major mode for Python."
  :group 'python
  :syntax-table python-syntax-table
  :abbrev-table python-mode-abbrev-table
  ;; body
  (setq-local indent-tabs-mode nil)
  (setq-local python-indent-offset 4)
  (font-lock-fontify-buffer))
```

自动:
- 创建 `python-mode` 函数
- 创建 `python-mode-hook`
- 创建 `python-mode-map`
- 创建 `python-mode-syntax-table`
- 继承 PARENT 的所有

继承父 mode 是关键——python-mode 继承 prog-mode,自动得到 prog-mode-hook 的触发、prog-mode 的所有共享行为。

### 3.2 define-minor-mode 完整

`define-minor-mode` 是 minor mode 的标准宏。它支持的 keyword 比 README 里讲的多——下面是完整列表,包括 `:require`(强制 require 某个 feature)、`:variable`(用别的变量名)、`:after-hook`(hook 跑完后跑):

```elisp
(define-minor-mode MODE DOC &optional KEYWORD-ARGS... BODY...)

(define-minor-mode my-mode
  "Toggle My mode."
  :init-value nil          ; 默认 nil
  :lighter " My"            ; mode line 显示
  :keymap my-mode-map
  :global nil               ; nil = buffer-local
  :group 'my-app
  :require 'my-mode         ; require 这个 feature
  :variable my-mode-var     ; 用别的变量名
  :after-hook (message "Started!")
  ;; body (toggle 时跑)
  (if my-mode
      (message "enabled")
    (message "disabled")))
```

### 3.3 define-globalized-minor-mode

普通 minor mode 是 buffer-local 的(只影响当前 buffer)。`define-globalized-minor-mode` 让你创建一个"全局 toggle"——一开就自动在每个(或符合 predicate 的)buffer 启用 local mode。Emacs 28+ 加了 `:predicate` 让你控制"哪些 buffer 启用":

```elisp
(define-globalized-minor-mode my-global-mode
  my-local-mode            ; 局部 mode
  my-local-mode            ; 启用函数 (通常是 same)
  :predicate (lambda () t)) ; 哪些 buffer 启用
```

global mode 自动在每个 buffer 启用 local mode。

### 3.4 hook 机制

进入一个 major mode 时,Emacs 不只跑这个 mode 的 hook,还跑所有父 mode 的 hook——这叫 `run-mode-hooks`。这就是为什么你 add 到 `prog-mode-hook` 的函数会在 python / js / go 等所有编程 mode 里跑:

进入 mode 时:
```elisp
(run-mode-hooks)
;; = run-hooks MODE-hook + parent mode hooks
```

例: python-mode 进入时跑:
- `prog-mode-hook` (parent)
- `python-mode-hook` (self)

### 3.5 minor mode 列表

Emacs 维护一个全局的 minor-mode-list——所有已定义的 minor mode 都在里面。`minor-mode-map-alist` 则记录"哪个 minor mode 启用时,要激活哪个 keymap"。Emacs 按这个 alist 顺序决定 minor mode keymap 优先级:

```elisp
(minor-mode-list)           ; 所有 minor mode
(assq 'my-mode minor-mode-map-alist)
;; → (my-mode . my-mode-map)
```

### 3.6 mode 优先级

active keymaps 顺序:
1. text property keymap
2. minor mode (按 minor-mode-map-alist 顺序)
3. major mode

minor mode 之间按 `minor-mode-map-alist` 顺序,通常**后启用的优先**。这就是为什么"后开的 minor mode 能覆盖先开的"——一个微妙的细节,有时会让你困惑"为什么我的键被抢了"。

---

## 4. 综合: 写一个完整的 minor mode

下面是一个完整可用的 minor mode 例子——`my-tools-mode`,提供三个命令(复制行、上移行、下移行),绑三个键,加一个 defgroup 让用户能 customize。它展示了 minor mode 的"四件套"全貌: defgroup + defcustom + 函数定义 + keymap + define-minor-mode + define-globalized-minor-mode。这是你写自己 minor mode 的模板:

```elisp
;;; my-tools-mode.el -*- lexical-binding: t; -*-

(defgroup my-tools nil
  "Custom tools."
  :group 'convenience
  :prefix "my-tools-")

(defcustom my-tools-enabled-commands
  '(my-duplicate-line
    my-move-line-up
    my-move-line-down)
  "Commands enabled in my-tools-mode."
  :type '(repeat function)
  :group 'my-tools)

(defun my-duplicate-line ()
  "Duplicate current line."
  (interactive)
  (let ((content (buffer-substring
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
    (save-excursion
      (transpose-lines 1))
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

(defvar my-tools-mode-map
  (let ((map (make-sparse-keymap)))
    (define-key map (kbd "C-c d") #'my-duplicate-line)
    (define-key map (kbd "M-<up>") #'my-move-line-up)
    (define-key map (kbd "M-<down>") #'my-move-line-down)
    map))

(define-minor-mode my-tools-mode
  "Toggle My Tools mode."
  :init-value nil
  :lighter " Tools"
  :keymap my-tools-mode-map
  :group 'my-tools
  :global t
  (if my-tools-mode
      (message "My Tools enabled")
    (message "My Tools disabled")))

(define-globalized-minor-mode global-my-tools-mode
  my-tools-mode
  my-tools-mode)

(provide 'my-tools-mode)
;;; my-tools-mode.el ends here
```

---

## 5. 自测

1. `defcustom` 的 `:type` 接受什么?
2. `:set` keyword 干啥?
3. active keymaps 的顺序?
4. `define-derived-mode` 自动创建什么?
5. `define-globalized-minor-mode` 和 `define-minor-mode` `:global t` 区别?

**答案**:
> 1. 类型 spec,如 'string, '(choice ...), '(repeat ...)
> 2. 自定义 setter,改值时跑
> 3. text property > minor mode > major mode > global
> 4. mode 函数、hook、keymap、syntax table (如果指定)
> 5. globalized 创建一个全局 toggle,在每个 buffer 调用 local 的;:global t 直接全局,不每 buffer

---

## 6. 下一步

进入 `use-package.md` 学 use-package 细节。
