# Minor Mode 教程

> 手把手教你写一个完整的 minor mode

---

## 1. Minor Mode 是什么

### 1.1 回顾

Minor mode 是 Emacs 的"特性开关"。它由 4 个东西组成: 一个变量(t/nil)、一个 toggle 函数、一个 keymap、一个 lighter(mode line 显示)。这看起来简单,但背后是 Emacs 设计的精髓——**所有"特性"都是同构的**。装包、开行号、自动配对括号、flyspell——它们都遵循同样的模式: 一个 minor mode。这就是为什么 Emacs 用户能"装 100 个包还不乱"——每个包都是一个 minor mode,统一接口。

对比 VS Code: 每个扩展有自己的 API、自己的 UI、自己的状态管理。Emacs 强制大家用同一套 minor mode 协议,结果是互操作性极高——你的 minor mode 可以和其他人的 minor mode 共存,不会冲突。这是 40 年沉淀的协议设计。

Module 4 README 学过:
- major mode: 每 buffer 一个,定义 buffer 类型
- minor mode: 多个可叠加,提供附加功能

例子:
- `linum-mode` (行号)
- `flyspell-mode` (拼写)
- `auto-fill-mode` (自动换行)
- `yas-minor-mode` (snippets)
- `corfu-mode` (补全)

### 1.2 minor mode 的 4 个组件

`define-minor-mode` 是一个宏,你给它名字和文档,它自动生成一整套基础设施——变量、函数、keymap、lighter、hook、customize group。这意味着你写一行宏调用,就得到了一整套 mode 基础设施。这种"用宏生成模板代码"是 Lisp 的典型用法:

```elisp
(define-minor-mode MODE
  DOC
  :init-value DEFAULT
  :lighter LIGHTER
  :keymap KEYMAP
  :group GROUP
  BODY...)
```

自动创建:
1. **变量** `MODE` (t/nil)
2. **函数** `MODE` (toggle)
3. **keymap** `MODE-map` (如果你提供)
4. **hook** `MODE-hook`

---

## 2. 完整例子: my-editing-mode

让我从零写一个 minor mode,功能:
- 自动配对括号
- 自动缩进
- 选中后输入替换
- 一些个人键位

这个例子涵盖 minor mode 开发的所有典型元素: defgroup(组织 customize)、defcustom(可配置选项)、辅助函数(实际逻辑)、keymap(键位)、define-minor-mode(主定义)、define-globalized-minor-mode(全局版本)。学完它你就有了写自己 minor mode 的完整模板:

```elisp
;;; my-editing-mode.el -*- lexical-binding: t; -*-

;;; Commentary:
;; 个人编辑模式,提供配对括号、自动缩进等。

;;; Code:

;;;; Custom variables

(defgroup my-editing nil
  "Personal editing enhancements."
  :group 'convenience
  :prefix "my-editing-")

(defcustom my-editing-auto-pair t
  "If non-nil, auto-pair brackets."
  :type 'boolean
  :group 'my-editing)

(defcustom my-editing-pairs
  '((?( . ?))
    (?[ . ?])
    (?{ . ?})
    (?\" . ?\")
    (?\' . ?\')
    (?` . ?`))
  "Pairs to auto-complete."
  :type '(repeat (cons character character))
  :group 'my-editing)

;;;; Helper functions

(defun my-editing--insert-pair (open)
  "Insert OPEN and its matching close, position point between."
  (let ((close (cdr (assoc open my-editing-pairs))))
    (if (and close
             (or (eolp)
                 (looking-at "[ \t\n]")))
        (progn
          (insert open close)
          (backward-char 1))
      (insert open))))

(defun my-editing--maybe-pair (char)
  "If CHAR is an opening bracket, auto-pair. Otherwise insert."
  (interactive "p")
  (let ((last-command-event (if (integerp char) char last-command-event)))
    (if (and my-editing-auto-pair
             (assoc last-command-event my-editing-pairs))
        (my-editing--insert-pair last-command-event)
      (self-insert-command char))))

;;;; Keymap

(defvar my-editing-mode-map
  (let ((map (make-sparse-keymap)))
    ;; 用转义绑定,覆盖默认插入
    (define-key map (kbd "(") #'my-editing--open-paren)
    (define-key map (kbd "[") #'my-editing--open-bracket)
    (define-key map (kbd "{") #'my-editing--open-brace)
    ;; 个人键位
    (define-key map (kbd "C-c d") #'my-duplicate-line)
    (define-key map (kbd "C-c r") #'my-reverse-region)
    map)
  "Keymap for `my-editing-mode'.")

(defun my-editing--open-paren (n)
  (interactive "p")
  (my-editing--maybe-pair ?\())

(defun my-editing--open-bracket (n)
  (interactive "p")
  (my-editing--maybe-pair ?\[))

(defun my-editing--open-brace (n)
  (interactive "p")
  (my-editing--maybe-pair ?{))

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

(defun my-reverse-region (start end)
  "Reverse characters in region."
  (interactive "r")
  (let* ((text (buffer-substring start end))
         (reversed (apply #'string (nreverse (string-to-list text)))))
    (delete-region start end)
    (insert reversed)))

;;;; The mode

(define-minor-mode my-editing-mode
  "Toggle My Editing mode.

When enabled, provides:
- Auto-pair brackets
- Personal editing shortcuts"
  :init-value nil
  :lighter " Edit"
  :keymap my-editing-mode-map
  :group 'my-editing
  :global nil
  (if my-editing-mode
      (progn
        (electric-pair-mode 1)
        (when my-editing-auto-pair
          (message "Auto-pair enabled")))
    (message "My Editing mode disabled")))

;;;; Global mode

(define-globalized-minor-mode global-my-editing-mode
  my-editing-mode
  my-editing-mode
  :predicate (lambda () (derived-mode-p 'text-mode 'prog-mode)))

(provide 'my-editing-mode)
;;; my-editing-mode.el ends here
```

---

## 3. 逐步讲解

### 3.1 defgroup + defcustom

`defgroup` 创建一个 customize 组——它让你的 minor mode 的所有选项在 `M-x customize-group` 里聚在一起。`:prefix` 让组里的变量自动加前缀显示,组结构清晰:

```elisp
(defgroup my-editing nil
  "..."
  :group 'convenience
  :prefix "my-editing-")
```

`:prefix "my-editing-"` 让 customize 自动加前缀显示。

```elisp
(defcustom my-editing-auto-pair t
  "..."
  :type 'boolean
  :group 'my-editing)
```

可配置选项,类型是 boolean。用户可以在 customize UI 改,也可以 Elisp 改。

### 3.2 keymap

`make-sparse-keymap` 创建一个空的 sparse keymap,然后用 `define-key` 一个个绑键。注意 `let` 包装——这是一种常见的模式: 在 let 里构造 map,let 返回 map 本身:

```elisp
(defvar my-editing-mode-map
  (let ((map (make-sparse-keymap)))
    (define-key map (kbd "(") #'my-editing--open-paren)
    ...
    map))
```

`make-sparse-keymap` 创建空 map。
绑定用 `define-key` + `kbd`。

### 3.3 函数

内部辅助函数用 `--` 双横线前缀(如 `my-editing--insert-pair`)。这是 Elisp 社区约定——告诉读者"这是内部函数,别从外面调用"。`defun` 是普通函数,加 `interactive` 才能被用户调用(M-x 或绑键):

```elisp
(defun my-editing--insert-pair (open)
  "Insert OPEN and matching close."
  ...)
```

内部函数用 `--` 双横线前缀 (Elisp 约定)。

### 3.4 define-minor-mode

`define-minor-mode` 是核心宏——它生成一整套 mode 基础设施。下面这些 keyword 控制 mode 的行为:

```elisp
(define-minor-mode my-editing-mode
  "Toggle My Editing mode."
  :init-value nil
  :lighter " Edit"
  :keymap my-editing-mode-map
  :group 'my-editing
  :global nil
  BODY...)
```

- `:init-value nil`: 默认不启用
- `:lighter`: mode line 显示 `" Edit"`
- `:keymap`: 上面定义的
- `:global nil`: buffer-local (普通 minor mode)
- BODY: toggle 时跑的代码

宏展开后等价于定义了变量、函数、hook、autoload 等一整套东西。`if my-mode` 在 BODY 里判断"现在是开还是关"——因为 BODY 在 toggle 两种情况都会跑:
- 定义变量 `my-editing-mode`
- 定义函数 `my-editing-mode` (toggle)
- 定义 hook `my-editing-mode-hook`
- 自动绑定 `M-x my-editing-mode`

### 3.5 global mode

`define-globalized-minor-mode` 创建一个"全局 toggle"——一开就自动在每个(符合 predicate 的)buffer 启用 local 版本。`:predicate` 是 Emacs 28+ 加的,让你精确控制"哪些 buffer 启用":

```elisp
(define-globalized-minor-mode global-my-editing-mode
  my-editing-mode
  my-editing-mode
  :predicate (lambda () (derived-mode-p 'text-mode 'prog-mode)))
```

`:predicate` (Emacs 28+) 控制**哪些 buffer** 启用。

`global-my-editing-mode` 启用后:
- 每个进入 `text-mode` 或 `prog-mode` 的 buffer 自动启用 `my-editing-mode`

---

## 4. Minor Mode 进阶

### 4.1 :after-hook

`:after-hook` 让你指定"mode 的 hook 跑完之后跑的代码"——通常用于"用户 hook 加完后再做收尾"。注意它和 BODY 的区别: BODY 在 hook 之前跑,`:after-hook` 在 hook 之后跑:

```elisp
(define-minor-mode my-mode
  "..."
  :after-hook (message "My mode hook done!"))
```

hook 跑完后跑这个。

### 4.2 :variable

`:variable` 让你用一个不同的变量名——通常用于"模式名和变量名要分开"的场景(比如 mode 叫 `my-mode`,变量叫 `my-mode-state`):

```elisp
(define-minor-mode my-mode
  "..."
  :variable my-mode-state)
```

用 `my-mode-state` 作为变量 (而不是默认的 `my-mode`)。
模式函数名还是 `my-mode`。

### 4.3 :interactive

`:interactive` 控制 mode 函数的 interactive 形式——默认是 `(lambda (arg) (list arg))`,你可以自定义。极少用,但有时需要根据 prefix arg 做不同行为:

控制 interactive 形式:

```elisp
(define-minor-mode my-mode
  "..."
  :interactive (lambda (arg) (message "arg: %s" arg) (list arg)))
```

### 4.4 :required

`:require` 声明"这个 mode 依赖某个 feature"——启用 mode 时如果 feature 没加载,会自动加载。用于"我的 mode 需要另一个包"的场景:

```elisp
(define-minor-mode my-mode
  "..."
  :require 'foo)
```

`foo` 必须 require 才能 enable。

---

## 5. Minor Mode 的 lighter

`lighter` 是 mode line 上显示的字符串——让用户看到"这个 mode 现在开着"。装很多 minor mode 会让 mode line 挤满,所以 `diminish` / `delight` 包被发明用来隐藏常用的 lighter:

`lighter` 是 mode line 上显示的字符串。

```elisp
:lighter " Edit"             ; 显示 "Edit"
:lighter ""                  ; 不显示
:lighter (:propertize " Edit" face 'bold)
```

如果用 `diminish` 或 `delight` 包,可以隐藏。

---

## 6. 启用条件

minor mode 可以通过多种方式启用——选哪种取决于"何时何地要启用"。

### 6.1 全局启用

最简单: 在 init.el 里调一次 `(global-my-editing-mode 1)`,所有 buffer(或符合 predicate 的)都启用:

```elisp
(global-my-editing-mode 1)
```

### 6.2 hook 启用

只在特定 mode 里启用——比如"只在 python 里启用"。用 hook 加函数:

```elisp
(add-hook 'python-mode-hook #'my-editing-mode)
```

### 6.3 按需启用

条件启用——比如"只在文件大于 1000 行时启用"。把逻辑放函数里,加 hook:

```elisp
(defun my-editing-enable-maybe ()
  (when (some-condition)
    (my-editing-mode 1)))

(add-hook 'python-mode-hook #'my-editing-enable-maybe)
```

---

## 7. 测试 minor mode

minor mode 也是 Elisp 代码,可以用 ERT(Emacs Lisp Regression Test)测试。测试 minor mode 的关键: 用 `with-temp-buffer` 隔离环境,启用 mode 后检查变量、keymap、实际行为。

### 7.1 基本

测试"toggle 是否工作"——启用后变量为 t,关闭后为 nil:

```elisp
(ert-deftest my-editing-mode-test ()
  (with-temp-buffer
    (my-editing-mode 1)
    (should my-editing-mode)
    (my-editing-mode -1)
    (should-not my-editing-mode)))
```

### 7.2 测试键位

测试"keymap 绑定是否正确"——查 `my-editing-mode-map` 里 `(` 绑到预期函数:

```elisp
(ert-deftest my-editing-mode-bindings ()
  (with-temp-buffer
    (my-editing-mode 1)
    (should (eq (lookup-key my-editing-mode-map (kbd "("))
                #'my-editing--open-paren))))
```

### 7.3 测试行为

测试"实际行为"——插入 `(` 后,buffer 内容变成 `()`,光标在中间。这种行为测试最有价值,因为它验证"用户体验":

```elisp
(ert-deftest my-editing-auto-pair ()
  (with-temp-buffer
    (my-editing-mode 1)
    (let ((my-editing-auto-pair t))
      (insert "(")
      (should (string= (buffer-string) "()"))
      (should (= (point) 2)))))
```

---

## 8. 调试

### 8.1 mode 没生效

```
C-h m
```

看 active minor modes,确认你的在里面。

### 8.2 键位不工作

```
C-h k KEY
```

看绑到啥。可能被其他 minor mode 抢了。

### 8.3 mode 不显示

lighter 是空字符串?
mode-line-format 配置覆盖了?

---

## 9. 实战: 一个完整的 minor mode

下面再写一个不同的 minor mode——`my-spell-check-mode`,启用时检查拼写、高亮错误、按 M-$ 修复。这个例子展示 minor mode 的另一种用法: 作为"完整功能模块"的容器。注意它的结构: defgroup + defcustom(可配置程序名)+ keymap + 命令函数 + define-minor-mode:

写一个 `my-spell-check-mode`,启用时:
- 检查拼写
- 高亮错误
- 按 `M-$` 修复

```elisp
;;; my-spell-check-mode.el -*- lexical-binding: t; -*-

(defgroup my-spell-check nil
  "Custom spell checking."
  :group 'tools)

(defcustom my-spell-check-program "aspell"
  "Spell checker to use."
  :type 'string
  :group 'my-spell-check)

(defvar my-spell-check-mode-map
  (let ((map (make-sparse-keymap)))
    (define-key map (kbd "M-$") #'my-spell-check-word)
    (define-key map (kbd "C-c s") #'my-spell-check-buffer)
    map))

(defun my-spell-check-word ()
  "Check word at point."
  (interactive)
  (if-let ((word (thing-at-point 'word)))
      (message "Checking: %s" word)
    (message "No word at point")))

(defun my-spell-check-buffer ()
  "Check whole buffer."
  (interactive)
  (message "Checking buffer..."))

(define-minor-mode my-spell-check-mode
  "Toggle My Spell Check mode."
  :init-value nil
  :lighter " Spell"
  :keymap my-spell-check-mode-map
  :group 'my-spell-check
  (if my-spell-check-mode
      (message "Spell check enabled")
    (message "Spell check disabled")))

(provide 'my-spell-check-mode)
;;; my-spell-check-mode.el ends here
```

---

## 10. 自测

1. minor mode 由哪 4 个组件组成?
2. `define-minor-mode` 自动创建什么?
3. `:lighter` 干啥?
4. `:global t` 和 `define-globalized-minor-mode` 区别?
5. `:predicate` 干啥?

**答案**:
> 1. 变量、函数、keymap、hook
> 2. mode 函数、mode 变量、mode hook、(如果指定) lighter
> 3. mode line 显示的小字符串
> 4. :global t 让 mode 函数直接全局影响 (但每 buffer 还是单独 toggle);globalized 创建一个全局 toggle,自动在每个 buffer 启用 local 版
> 5. 控制 globalized mode 在哪些 buffer 启用

---

## 11. 下一步

进入 `capstone.md` 做 Module 4 毕业项目。
