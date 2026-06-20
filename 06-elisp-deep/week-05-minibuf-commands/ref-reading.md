# Week 5: Minibuffer + Commands + Keymaps

> **Ref 章节**: minibuf, commands, keymaps
> **目标**: 写自定义的 minibuffer 交互、命令、keymap

这周是"用户交互"的深度——你如何让用户输入、如何响应按键、如何设计 keymap。理解这三者让你能写**真正的 Emacs 命令**——不只是函数,而是有 prompt、补全、history、prefix arg 的完整交互。

Module 4 你学过用户视角 (怎么用 minibuffer)。这周学 Lisp 视角——怎么**编程** minibuffer。

---

## 1. Minibuffer (minibuf.texi)

### 1.1 Minibuffer 是什么

(Module 1 学过用户视角)
Module 6 学 Lisp 视角:

minibuffer 是 Emacs 的"输入对话框"——一个特殊的 buffer,出现在屏幕底部,用于 prompt 用户输入。它和普通 buffer 区别: 只一行、自动 focus、有特殊 keymap (history 导航、补全)。

Emacs 提供 `read-*` 函数族,每个对应一种输入类型。

```elisp
(read-from-minibuffer PROMPT &optional INITIAL KEYMAP READ HIST DEFAULT INHERIT-INPUT-METHOD)
(read-string PROMPT &optional INITIAL HIST DEFAULT INHERIT-INPUT-METHOD)
(read-number PROMPT &optional DEFAULT)
(read-char)                    ; 读一个字符
(read-event)                   ; 读一个 event
(read-key-sequence PROMPT)
(read-command PROMPT)
(read-variable PROMPT)
(read-function PROMPT)
(read-buffer PROMPT)
(read-file-name PROMPT &optional DIR DEFAULT MUSTMATCH INITIAL PREDICATE)
(read-directory-name PROMPT &optional DIR DEFAULT MUSTMATCH INITIAL)
(read-no-blanks-input PROMPT &optional INITIAL INHERIT-INPUT-METHOD)
(completing-read PROMPT COLLECTION &optional PREDICATE REQUIRE-MATCH INITIAL-INPUT HIST DEF INHERIT-INPUT-METHOD)
```

这一堆 `read-*` 函数每个针对一种输入——string、number、char、file、buffer 等。它们共享 minibuffer 基础设施 (history、补全),但行为特化。

`read-string` 最简单——读 string。`read-number` 解析成 number。`read-file-name` 加文件补全。`completing-read` 是最灵活——支持任意 COLLECTION。

### 1.2 基础用法

```elisp
(read-from-minibuffer "Enter: ")    ; 简单读 string

(read-string "Name: ")               ; alias

(read-number "Age: ")                ; 读数字

(read-file-name "Open: " "~/")       ; 文件名,带补全

(completing-read "Color: "
                 '("red" "green" "blue")
                 nil t)              ; 选一个
```

每个调用都会显示 prompt,等待用户输入 RET。`read-file-name` 和 `completing-read` 自动加补全 (TAB 触发)。

### 1.3 输入历史

每个 `read-*` 可以指定 history 变量——让用户用 M-p/M-n 浏览历史输入。

```elisp
(read-from-minibuffer "Cmd: " nil nil nil 'my-cmd-history)

(defvar my-cmd-history nil)
;; 每次 read 用这个变量,历史自动累积
```

`my-cmd-history` 是 list,初始 nil。每次 read,如果用户输入非空,自动 push 到这个 list。下次 read,按 M-p 看上次输入。

这是 UX 的重要部分——用户不需要每次重新输入。Emacs 内置 `read-expression-history`、`minibuffer-history` 等通用 history 变量。

### 1.4 默认值

```elisp
(read-from-minibuffer "Name (default Alice): "
                       nil nil nil nil "Alice")
;; 按 RET 接受 "Alice"
```

DEFAULT 参数是"按 RET 时的默认值"——用户不输入直接 RET,返回这个。这让 prompt 友好——用户不需要每次输入,空 RET 接受默认。

### 1.5 completing-read

`completing-read` 是最灵活的——支持任意 collection,带补全。

```elisp
(completing-read PROMPT COLLECTION &optional PRED REQUIRE-MATCH INITIAL-INPUT HIST DEF)
```

- COLLECTION: list, obarray, function, hash-table, 或 alist
- PRED: 过滤 predicate
- REQUIRE-MATCH:
  - `nil`: 不要求匹配
  - `t`: 必须匹配
  - 确切: 必须 match (confirm)
- INITIAL-INPUT: 初始输入
- HIST: 历史变量
- DEF: 默认值

```elisp
(completing-read "Pick: "
                 '(("apple" . 1) ("banana" . 2))
                 nil t)
;; 用户选 "apple" 或 "banana"
```

COLLECTION 可以是 alist——用户看到 key,代码可以通过 assoc 查 value。这是"下拉菜单"的标准实现。

REQUIRE-MATCH 是 UX 关键——`nil` 让用户自由输入 (可能错),`t` 强制选择 (防止错)。`(confirm)` 中间——用户输入不匹配的值时 Emacs 问"你确定吗?"

### 1.6 read-key-sequence

```elisp
(read-key-sequence "Press key: ")
;; 读 key sequence (如 C-c C-v)
```

`read-key-sequence` 让你"录"用户按键——返回 key sequence vector。用于"绑定命令到键"功能 (如 `M-x global-set-key` 交互模式)。

### 1.7 yes-or-no-p / y-or-n-p

确认对话框。

```elisp
(yes-or-no-p "Are you sure? ")     ; 输入 yes / no
(y-or-n-p "OK? ")                   ; 输入 y / n

;; 推荐:
(setq use-short-answers t)         ; yes-or-no-p 变 y-or-n-p
```

`yes-or-no-p` 要求完整输入 "yes" 或 "no"——避免误操作 (如删除文件)。`y-or-n-p` 单字符——快速。

`use-short-answers` (Emacs 28+) 让 `yes-or-no-p` 自动用 `y-or-n-p` 行为——大多数场景更友好。

### 1.8 read-passwd

```elisp
(require 'passwd)
(read-passwd "Password: ")
```

`read-passwd` 读密码——输入不显示 (用 * 替代)。安全考虑——密码不入 history,不写到 *Messages*。

### 1.9 minibuffer 里的特殊键

minibuffer 有专属 keymap,这些键绑定:

```
TAB          补全
SPC          补全 (在某些场景)
RET          提交
M-p / M-n    history prev/next
M-r          regexp search history
C-g          取消
```

理解这些让你 prompt 时知道用户能做什么。例如你可以告诉用户"按 TAB 补全"。

---

## 2. Commands (commands.texi)

### 2.1 命令循环

Emacs 的核心是 command loop——一个无限循环,把按键变成命令执行。

```
按键事件
   ↓
input-decode-map / function-key-map 转换
   ↓
keymap lookup 找到 command
   ↓
prefix-arg 处理
   ↓
(call-interactively COMMAND)
   ↓
interactive 读参数
   ↓
pre-command-hook
   ↓
COMMAND 执行
   ↓
post-command-hook
   ↓
更新 mode line
```

理解这个流程让你写"智能"命令——你可以在任何环节插入逻辑:
- pre-command-hook: 改变 `this-command` (取消默认)
- interactive: 读参数,可能为空 (用默认)
- 命令 body: 实际逻辑
- post-command-hook: 后续动作 (update UI、log)

### 2.2 interactive 完整

(Module 3 学过基础)

`interactive` 是 special form——告诉 command loop 怎么读参数。

```elisp
(interactive)
(interactive "P")                ; prefix arg (raw)
(interactive "p")                ; prefix arg (number)
(interactive "sPrompt: ")        ; string
(interactive "nPrompt: ")        ; number
(interactive "fPrompt: ")        ; existing file
(interactive "FPrompt: ")        ; file
(interactive "bPrompt: ")        ; buffer
(interactive "BPrompt: ")        ; buffer (可能不存在)
(interactive "r")                ; region
(interactive "d")                ; point (integer)
(interactive "D")                ; directory
(interactive "M")                ; sexp (Lisp expression)
(interactive "c")                ; char
(interactive "k")                ; key sequence
(interactive "K")                ; key (no down events)
(interactive "e")                ; last event
(interactive "x")                ; Lisp form
(interactive "X")                ; Lisp form (must be valid)
(interactive "a")                ; function name
(interactive "v")                ; variable name
(interactive "U")                ; unit prefix (mnemonic)
(interactive "m")                ; mouse position
(interactive "S")                ; symbol

;; 组合:
(interactive "sName: \nnAge: ")    ; 两参数
(interactive (list ...))            ; 任意逻辑
```

每个字符代码对应一种输入:
- `s` = string (minibuffer)
- `n` = number (minibuffer,parse)
- `f`/`F` = file (补全)
- `b`/`B` = buffer (补全)
- `r` = region (beg end)
- `P`/`p` = prefix arg
- `c` = char (单键)
- `k`/`K` = key sequence
- `D` = directory

组合用 `\n` 分隔——`(interactive "sName: \nnAge: ")` 提示两次,得两个参数。

### 2.3 interactive 例

```elisp
(defun my-func (name age region)
  (interactive
   (list (read-string "Name: ")
         (read-number "Age: ")
         (if (use-region-p)
             (list (region-beginning) (region-end))
           (list (point) (point)))))
  ...)
```

`(interactive (list ...))` 让你写任意逻辑——条件读取、组合参数。这里如果用户有 region,提供 region 范围;否则 point point (空 region)。

```elisp
;; 让用户选颜色
(defun my-pick-color (color)
  (interactive
   (list (completing-read "Color: "
                          '("red" "green" "blue"))))
  ...)
```

这是 `completing-read` + interactive 的组合——让用户从选项中选。

### 2.4 prefix-arg

prefix arg 是 Emacs 的"修饰"——`C-u` 或 `M-N` 前缀,让命令改变行为。

```elisp
(defun my-func (arg)
  (interactive "P")            ; raw prefix (C-u 后 4)
  (message "arg: %S" arg))

(my-func nil)                  ; 无 C-u
(my-func '(4))                 ; C-u (单)
(my-func '(16))                ; C-u C-u
(my-func 5)                    ; M-5

(defun my-func-num (arg)
  (interactive "p")            ; numeric prefix
  (message "got %d" arg))

;; (my-func-num 5)  ←→  M-5 M-x my-func-num RET
```

`P` (大写) 是 raw prefix——`C-u` 是 `'(4)`,`C-u C-u` 是 `'(16)`,`M-5` 是 `5`。复杂,但保留所有信息。

`p` (小写) 是 numeric——`C-u` 是 `4`,`C-u C-u` 是 `16`,`M-5` 是 `5`。简单,适合数字。

实战: 用 `p` 当"重复次数"——`(defun my-insert-x (n) (interactive "p") (dotimes (_ n) (insert "x")))`,`M-5 M-x my-insert-x` 插入 5 个 x。

### 2.5 this-command / last-command

```elisp
(defun my-func ()
  (interactive)
  (if (eq last-command 'my-other)
      (message "called after my-other")
    (message "called standalone")))
```

`last-command` 让你检测"上次跑的命令"——常用于"重复命令加倍"模式 (如 kmacro 的"按两次 e 加倍")。

---

## 3. Keymaps (review)

(Module 4 学过)

### 3.1 active maps

按键到命令的查找经过多个 keymap,有优先级。理解优先级让你知道为什么某键在某 mode 不工作。

```
1. overriding-local-map        (rare)
2. text property 'keymap
3. overriding-terminal-local-map
4. minor-mode-overriding-map-alist
5. minor-mode-map-alist
6. current-local-map           (major mode)
7. global-map
```

minor mode 在 major mode 之前——所以 minor mode 可以 override major mode。global 最后——只在没其他 map 处理时用。

### 3.2 keymap 数据结构

```elisp
(make-sparse-keymap)            ; sparse
(make-keymap)                   ; full
(keymapp OBJ)
```

`sparse-keymap` 是 alist——只存已定义的键。`full keymap` 是 char-table——每个字符都有 entry (默认 nil)。

`sparse` 节省内存 (大部分键未定义),`full` 查找稍快。Major mode 几乎都用 sparse。

内部是 vector of vectors。

### 3.3 lookup-key

```elisp
(lookup-key KEYMAP KEY)
;; 返回 command 或 nil

(lookup-key global-map (kbd "C-x C-f"))
;; → find-file

(lookup-key global-map (kbd "C-c C-v"))
;; → nil (如果没绑)
```

`lookup-key` 让你查"某键绑了什么"。返回 command symbol 或 nil。

### 3.4 where-is-internal

```elisp
(where-is-internal 'find-file)
;; → ([24 6] [(control ?x) (control ?f)])
;; C-x C-f 的几种表示
```

反向 lookup——给定 command,找它绑到哪些键。`C-h w` (where-is) 用这个。

### 3.5 define-key 复杂形式

```elisp
(define-key map [?\C-c ?\C-v] #'my-func)
(define-key map [menu-bar my-menu] (cons "My" (make-sparse-keymap)))
(define-key map [menu-bar my-menu my-action]
  '(menu-item "Action" my-action :help "Run"))
```

define-key 不只绑键——也能定义 menu bar item。`[menu-bar ...]` 是 menu 的 key 表示。

### 3.6 keymap inheritance

```elisp
(set-keymap-parent CHILD-PARENT)
(make-composed-keymap MAP-LIST)
```

keymap 可以继承——child 自动有 parent 的所有 binding。这让 major mode 继承 prog-mode 时,自动有 prog-mode 的 binding。`make-composed-keymap` 合并多个 keymap。

---

## 4. 实战

### Ex 5.1: 写一个 query 用户

```elisp
(defun my-ask ()
  (interactive)
  (let ((name (read-string "Your name: "))
        (age (read-number "Age: ")))
    (message "Hello, %s (%d)!" name age)))
```

最简单的 prompt——read-string 和 read-number 组合。

### Ex 5.2: completing-read

```elisp
(defun my-pick ()
  (interactive)
  (let ((choice (completing-read
                 "Pick: "
                 '(("apple" . fruit)
                   ("carrot" . vegetable)
                   ("beef" . meat))
                 nil t)))
    (message "You picked: %s" choice)))
```

alist 作为 collection——用户看到 key (apple/carrot/beef),通过 assoc 查 value (类别)。

### Ex 5.3: read-file-name

```elisp
(defun my-open ()
  (interactive)
  (let ((file (read-file-name "Open: ")))
    (find-file file)))
```

read-file-name 自动加文件补全——TAB 触发 dired-style 补全。

### Ex 5.4: 用 region 自动

```elisp
(defun my-uppercase-region-or-word (beg end)
  (interactive "r")
  (if (use-region-p)
      (upcase-region beg end)
    (save-excursion
      (let ((bounds (bounds-of-thing-at-point 'word)))
        (when bounds
          (upcase-region (car bounds) (cdr bounds)))))))
```

`(interactive "r")` 给 beg/end——但只在 region active 时有意义。`use-region-p` 检查——如果没 region,转而 uppercase 当前 word。这是"region-aware"命令的标准模式。

### Ex 5.5: prefix arg

```elisp
(defun my-repeat (n)
  (interactive "p")
  (dotimes (_ (or n 1))
    (insert "X")))
```

`(interactive "p")` 提供 numeric prefix——`M-5 M-x my-repeat` 插入 5 个 X。无 prefix 时 n 是 1 (或 nil,所以 `(or n 1)`)。

---

## 5. 自测

1. `(interactive "sFoo: ")` 干啥?
2. `completing-read` 怎么传 COLLECTION?
3. prefix arg raw 和 numeric 区别?
4. keymap 优先级?
5. `(interactive (list ...))` 干啥?

**答案**:
> 1. 提示 "Foo: ",读字符串
> 2. list / obarray / function / hash-table / alist
> 3. raw 是 raw prefix (C-u → '(4));numeric 是数字 (C-u → 4)
> 4. overriding > text prop > minor > local > global
> 5. 允许任意 interactive 逻辑

---

## 6. 下一步

进入 `concept-anchor.md` + `exercises.md`,然后 `../week-06-modes-files-buffers/`。
