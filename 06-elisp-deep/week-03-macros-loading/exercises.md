# Exercises: Week 3 Macros + Loading

> 10 题

这周练习考宏的各个角度——从最简单 (my-when) 到复杂 (with-gensyms)。最后两题考 increment/decrement 宏,让你"重发明" `incf`/`decf`。

---

### 题 1: my-when

`when` 的最简宏——展开成 `(if cond (progn body))`。

```elisp
(defmacro my-when (cond &rest body)
  `(if ,cond (progn ,@body)))
```

`,@body` 是 splice——body 是 `(&rest)` 收集的 list,splice 进 progn。如果 body 是 `(foo bar)`,展开 `(progn foo bar)`;如果 body 是 `(foo)`,展开 `(progn foo)`。

### 题 2: my-unless

`unless` = if not。

```elisp
(defmacro my-unless (cond &rest body)
  `(if (not ,cond) (progn ,@body)))
```

和 my-when 对称——条件取反。

### 题 3: my-with-gensym

`with-gensyms` 是写复杂宏的标准 helper——一次性创建多个 gensym。

```elisp
(defmacro my-with-gensyms (vars &rest body)
  ;; 创建 gensyms 给 VARS
  )
```

<details><summary>答案</summary>

```elisp
(defmacro my-with-gensyms (vars &rest body)
  `(let ,(mapcar (lambda (v) (list v '(gensym))) vars)
     ,@body))
```

`(mapcar (lambda (v) (list v '(gensym))) vars)` 把 `(a b c)` 转成 `((a (gensym)) (b (gensym)) (c (gensym)))`。然后 `(let ,bindings ...)` 展开成 `(let ((a (gensym)) (b (gensym)) (c (gensym))) ...)`。

用法: `(my-with-gensyms (temp1 temp2) `(let ((,temp1 ...) (,temp2 ...)) ...))`——一次性创建 temp1、temp2 两个 gensym,在宏 body 用。

</details>

### 题 4: my-with-temp-file

`with-temp-file` 是 Emacs 内置宏——创建 temp buffer,跑 body,写到文件。这题让你重写它。

```elisp
(defmacro my-with-temp-file (path &rest body)
  ;; 创建 temp buffer,跑 BODY,写到 PATH
  )
```

<details><summary>答案</summary>

```elisp
(defmacro my-with-temp-file (path &rest body)
  `(let ((path ,path))
     (with-temp-buffer
       ,@body
       (write-region (point-min) (point-max) path))))
```

注意 `(let ((path ,path)) ...)`——这里 `,path` 是用户传入的 form (可能是变量引用或字面量 string),let 让 body 内的 `path` 指向它。但 body 内可能也用 `path`,所以这是潜在冲突——正确做法用 gensym。

</details>

### 题 5: autoload

给你的包加 `;;;###autoload`。

这是让你的 package 可发布的第一步——`;;;###autoload` 让包安装后,你的命令自动注册到 Emacs,首次调用时才加载完整代码。

### 题 6: require vs load

解释区别。

**答案**: require 加载 feature (有 provide);load 加载文件。

`require` 是声明式——"我需要 foo feature,请确保它加载"。Emacs 检查 features list,如果没,加载文件。`load` 是命令式——"加载这个文件",无论是否已加载。

实战: package 用 require (idempotent、声明依赖);用户配置用 load (强制加载,即使重复)。

### 题 7: with-eval-after-load

```elisp
(with-eval-after-load 'org
  (define-key org-mode-map (kbd "C-c C-x j") #'my-org-jump))
```

这是配置 package 的标准模式——等 package 加载后再配置它的 keymap。如果你直接 `(define-key org-mode-map ...)`,org-mode-map 可能还不存在 (org 没加载),报错。

### 题 8: byte-compile

```
M-x byte-compile-file RET ~/.config/emacs/lisp/my-todo.el RET
```

修复所有 warning。

修复 warning 是关键工程实践:
- `free variable`: 可能是 typo (打错变量名)
- `unused lexical variable`: dead code,可以删
- `not known to be defined`: 可能忘 require 或 autoload

### 题 9: 写宏 my-incf

`incf` 是 CL 的"自增" 宏——`(incf x)` 等价 `(setq x (1+ x))`。

```elisp
(defmacro my-incf (var)
  `(setq ,var (1+ ,var)))

(setq x 5)
(my-incf x)                    ; → 6
```

注意为什么是宏而非函数: `(incf x)` 中 `x` 不应该 eval——它是要被赋值的 symbol,不是值。如果用函数 `(defun incf (var) ...)`,调用 `(incf x)` 时 x 已 eval (得 5),传给函数的是 5 不是 symbol——无法 setq。

### 题 10: 写宏 my-decf

```elisp
(defmacro my-decf (var)
  `(setq ,var (1- ,var)))
```

和 my-incf 对称——减 1。

---

## 自测

1. 宏接收什么? (eval'd or uneval'd forms)
2. backquote 的 `,` 和 `,@`?
3. gensym 解决什么问题?
4. autoload 怎么实现?
5. native comp 比 byte comp 强在哪?

**答案**:
> 1. uneval'd forms
> 2. , 插入值;,@ splice list
> 3. 宏变量捕获——gensym 保证内部 symbol 不和外部冲突
> 4. 把函数标记为延迟加载,调用时才 require (通过 autoloads.el)
> 5. 编译到原生机器码 (用 GCC backend),比 byte-code 快 2-3x

---

进入 `../week-04-debugging/`。
