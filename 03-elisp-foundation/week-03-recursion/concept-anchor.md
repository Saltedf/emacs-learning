# Concept Anchor: Editor Manual (Week 3)

> 替代 `emacs-manual-30.2/programs.texi` 的核心概念
> 这周你的递归、循环、闭包,对应编辑器的"程序编辑"和"复杂自动化"

---

## 1. programs.texi 概念锚点

### 1.1 什么是 "Editing Programs"

`programs.texi` 讲**编程语言相关的编辑**:
- 缩进 (indentation)
- 注释
- 括号匹配
- 跳定义 (imenu)
- 缩进风格

这些都是 major mode 提供的,通过 Lisp 函数实现。

### 1.2 Lisp 视角

```elisp
(indent-line)                      ;; 当前行缩进
(indent-region START END)          ;; region 缩进
(comment-line N)                   ;; 注释 N 行
(forward-comment N)                ;; 跳过 N 个注释
```

这些是 Elisp 函数,你可以在自己的代码里调用。

### 1.3 Defun 移动

```elisp
(beginning-of-defun)               ;; 上一函数定义
(end-of-defun)                     ;; 下一函数结束
(mark-defun)                       ;; 选中当前 defun
(narrow-to-defun)                  ;; 限制到当前 defun
```

对 Lisp 代码: defun = 顶层 `(defun ...)`
对 Python: def
对 JS: function
对 C: function definition

不同 mode 有不同实现。

### 1.4 Indentation

每种 major mode 有自己的 indent 函数:

```elisp
(indent-for-tab-command)           ;; TAB 键,缩进当前行
(indent-according-to-mode)         ;; 按 mode 规则缩进
(tab-to-tab-stop)                  ;; 跳到下一个 tab stop
```

变量:
- `indent-tabs-mode`: t 用 tab,nil 用空格
- `tab-width`: tab 宽度
- `c-basic-offset` (C 模式)

### 1.5 Comments

```elisp
(comment-region START END)         ;; 注释 region
(uncomment-region START END)
(comment-dwim)                     ;; do what I mean (绑定 M-;)
(comment-kill)                     ;; 杀注释
(comment-or-uncomment-region)
```

变量:
- `comment-start`: 例 `"# "` (Python), `";; "` (Elisp)
- `comment-end`: 例 `""` (Python), `""` (Elisp)
- `comment-multi-line`: 多行注释风格

---

## 2. 你的循环对应编辑器

### 2.1 批量操作

```elisp
(defun my-mark-all-todos ()
  "Add TODO keyword to all 'XXX' markers."
  (interactive)
  (save-excursion
    (goto-char (point-min))
    (while (re-search-forward "XXX" nil t)
      (replace-match "TODO"))))
```

这是 `while + re-search` 的经典模式:
- 找下一个匹配
- 替换
- 循环

### 2.2 操作每个 defun

```elisp
(defun my-process-each-defun (fn)
  "Apply FN to each defun in buffer."
  (save-excursion
    (goto-char (point-min))
    (while (re-search-forward "^(defun" nil t)
      (beginning-of-line)
      (funcall fn))))
```

### 2.3 操作每个文件

```elisp
(defun my-process-files (pattern fn)
  "Apply FN to each file matching PATTERN in current dir."
  (dolist (file (directory-files "." t pattern))
    (with-current-buffer (find-file-noselect file)
      (funcall fn)
      (save-buffer))))
```

---

## 3. 闭包的应用: 模式

闭包不仅是理论——它在 Emacs 实战中无处不在。下面三个 pattern 你会在自己的代码里反复用到。

### 3.1 缓存 (memoization)

memoization 是"把函数调用结果缓存,下次同样参数直接返回"。对慢函数 (如递归 fib) 极其有用。

```elisp
(defun my-memoize (fn)
  "Return a memoized version of FN."
  (let ((cache (make-hash-table :test 'equal)))
    (lambda (arg)
      (or (gethash arg cache)
          (let ((result (funcall fn arg)))
            (puthash arg result cache)
            result)))))

(setq fast-fib (my-memoize #'fib))
(funcall fast-fib 100)             ;; 立即返回 (用缓存)
```

这是闭包的杀手锏——lambda 持有 cache hash table,每次调用查它。直接调用 fib 100 万年算不完,但 memoize 后秒回。

这是"高阶函数 + 闭包"的典型组合: `my-memoize` 接受函数返回函数,返回的函数持有私有 cache。

### 3.2 Counter (Module 1 Kmacro counter 的 Lisp 版)

Module 1 你学了 keyboard macro 的 counter——按一个键,自动插入递增数字。Lisp 版怎么做?闭包:

```elisp
(defvar my-counter-factory
  (let ((counters (make-hash-table)))
    (lambda (op &rest args)
      (pcase op
        ('new (let ((id (gensym)))
                (puthash id 0 counters)
                id))
        ('inc (cl-incf (gethash (car args) counters)))
        ('get (gethash (car args) counters))))))

(funcall my-counter-factory 'new)             ;; → #:g1
;; (假设返回 g1)
(funcall my-counter-factory 'inc 'g1)         ;; → 1
(funcall my-counter-factory 'inc 'g1)         ;; → 2
(funcall my-counter-factory 'get 'g1)         ;; → 2
```

这个例子展示了"用闭包模拟对象"——counters 是私有 state,my-counter-factory 是 dispatcher。从外面看,它像一个小对象,有 new/inc/get 方法。

但没用 defclass——纯 lambda + hash table。这就是 "lambda is the ultimate object" 的实践。

---

## 4. Recursive 编辑

### 4.1 嵌套数据的递归处理

```elisp
(defun my-replace-in-tree (old new tree)
  "Replace all OLD with NEW in TREE (recursive)."
  (cond
   ((null tree) nil)
   ((consp tree)
    (cons (my-replace-in-tree old new (car tree))
          (my-replace-in-tree old new (cdr tree))))
   ((equal old tree) new)
   (t tree)))

(my-replace-in-tree 'foo 'bar '(foo (foo bar) baz))
;; → (bar (bar bar) baz)
```

### 4.2 buffer 文本的递归处理

```elisp
(defun my-process-forms ()
  "Find and process each top-level form."
  (interactive)
  (save-excursion
    (goto-char (point-min))
    (while (not (eobp))
      (let ((form (read (current-buffer))))
        ;; process form
        (message "Form: %S" form)
        (skip-chars-forward " \t\n")))))
```

(`read` 是 Lisp reader,读 sexp)

---

## 5. 编辑器命令对应 Lisp

### 5.1 复杂操作

```elisp
(defun my-format-buffer ()
  "Format current buffer (indent, remove trailing whitespace)."
  (interactive)
  (delete-trailing-whitespace)
  (indent-region (point-min) (point-max))
  (save-buffer))

(defun my-clean-buffer ()
  "Clean up: trailing whitespace, blank lines, etc."
  (interactive)
  (delete-trailing-whitespace)
  (goto-char (point-min))
  (while (re-search-forward "\n\n\n+" nil t)
    (replace-match "\n\n"))
  (goto-char (point-min))
  (while (re-search-forward " +$" nil t)
    (replace-match "")))
```

### 5.2 条件性批量操作

```elisp
(defun my-mark-issues ()
  "Mark lines containing 'FIXME' or 'TODO'."
  (interactive)
  (save-excursion
    (goto-char (point-min))
    (while (re-search-forward "\\(FIXME\\|TODO\\)" nil t)
      (goto-char (line-beginning-position))
      (insert "! ")
      (forward-line 1))))
```

---

## 6. 你的 Lisp 视角看编程编辑

### 6.1 Syntax Table 决定一切

什么是 word? 什么是括号? 什么是注释?
都由 syntax table 决定。

```elisp
(modify-syntax-entry ?_ "w")        ; _ 算 word 字符
(modify-syntax-entry ?\" "\"")      ; " 是 string delimiter
(modify-syntax-entry ?# "_")        ; # 算 symbol
(modify-syntax-entry ?\; "<")       ; 注释开始
(modify-syntax-entry ?\n ">")       ; 注释结束
```

(Module 6 W7 详细讲)

### 6.2 font-lock 决定颜色

```elisp
(font-lock-add-keywords nil
  '((("\\<\\(TODO\\|FIXME\\)\\>" 1 'font-lock-warning-face t))))
```

让 TODO/FIXME 高亮显示。

(Module 6 W7)

### 6.3 Smartparens / Paredit

`paredit` / `smartparens` 包用 Lisp 重定义编辑操作:
- 不允许不平衡括号
- 用结构化命令 (slurp, barf, raise) 编辑

它们的实现都是**很多 defun 和 keymap 修改**。

---

## 7. 自测

1. 写一个函数把当前 buffer 里所有 "XXX" 改成 "TODO"
2. 写一个函数返回 buffer 里 "defun" 出现次数
3. 写一个函数给所有当前行加注释前缀
4. 用闭包写一个"logging" 函数工厂

**答案**:
> 1. 
> ```elisp
> (defun my-xxx-to-todo ()
>   (interactive)
>   (save-excursion
>     (goto-char (point-min))
>     (while (re-search-forward "XXX" nil t)
>       (replace-match "TODO"))))
> ```
> 2. 
> ```elisp
> (defun my-count-defuns ()
>   (let ((count 0))
>     (save-excursion
>       (goto-char (point-min))
>       (while (re-search-forward "^\\s-*(defun" nil t)
>         (setq count (1+ count))))
>     count))
> ```
> 3. 
> ```elisp
> (defun my-comment-all-lines ()
>   (interactive)
>   (save-excursion
>     (goto-char (point-min))
>     (while (not (eobp))
>       (comment-line 1))))
> ```
> 4. 
> ```elisp
> (defun make-logger (prefix)
>   (lambda (msg)
>     (message "[%s] %s" prefix msg)))
> 
> (setq my-log (make-logger "APP"))
> (funcall my-log "started")    ;; → "[APP] started"
> ```

---

## 8. 下一步

进入 `exercises.md`。
