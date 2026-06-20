# Exercises: Week 2 Eval + Variables + Functions

> 15 题

这周练习考"理解 Elisp 怎么思考"——eval、变量作用域、函数对象。前三题纯概念 (eval 返回什么),中间变量作用域 (buffer-local),最后高阶函数 (compose、curry、flip)。

---

## 第一组: Eval (5 题)

### 题 1

`eval` 是 meta-evaluator。`(+ 1 2 3)` 是 list,eval 把它当作 form 求。

```elisp
(eval '(+ 1 2 3))              ; → ?
```

### 题 2

`(list 'a 'b)` 求 list——但它求的是 "(list 'a 'b)" 这个 form,不是 "(a b)"。`list` 是函数,所以先 eval `'a` (得 a),`'b` (得 b),再调 `(list a b)`。

```elisp
(eval '(list 'a 'b))           ; → ?
```

### 题 3

注意 dynamic vs lexical。在 dynamic binding 下,`x` 的值查找是"调用栈上最近的 binding"。这里 `(setq x 10)` 全局设 x=10,`(+ x 5)` 看到 10。

```elisp
(setq x 10)
(eval '(+ x 5))                ; → ? (dynamic)
```

### 题 4

`read` 把 string 转 form,`eval` 求。这是 Elisp 实现 REPL 的核心——用户输入 string,read 成 form,eval。

```elisp
(my-eval-string "(+ 1 2)")     ; → 3
```

<details><summary>答案</summary>

```elisp
(defun my-eval-string (s)
  (eval (read s)))
```

注意安全: `(read "(shell-command \"rm -rf ~\")")` 会创建一个 dangerous form,`eval` 会执行它。**永远不要 eval-string 不可信源**。

</details>

### 题 5

`eval` 的第二个参数控制 lexical (t) 还是 dynamic (nil)。在 lexical mode,form 内的 let 是 lexical binding;在 dynamic mode,是 dynamic。

**答案**:
> lexical 和 dynamic eval

---

## 第二组: Variables (5 题)

### 题 6

`boundp` 检查 symbol 是否有 value slot。

```elisp
(my-bound-p 'kill-ring)        ; → t
(my-bound-p 'foo-bar-baz)      ; → nil
```

<details><summary>答案</summary>

```elisp
(defun my-bound-p (sym)
  (boundp sym))
```

这就是 `boundp` 的别名。但理解 `boundp` 的语义——它检查 value slot 是否非 void。`makunbound` 让 slot 变 void。

</details>

### 题 7

`unless` + `boundp` 是"如果没定义则定义"——和 defvar 的"如果已设则不覆盖"类似,但更可控。

```elisp
(my-define-if-not 'foo 5)      ; 如果 foo 没 bound,设为 5
```

<details><summary>答案</summary>

```elisp
(defun my-define-if-not (sym val)
  (unless (boundp sym)
    (set sym val)))
```

</details>

### 题 8

`setq-local` 是糖——在当前 buffer 创建 buffer-local binding。

```elisp
;; 怎么让变量 indent-tabs-mode 在某 buffer 局部 nil?
```

<details><summary>答案</summary>

```elisp
(setq-local indent-tabs-mode nil)
```

注意 `indent-tabs-mode` 默认是 buffer-local (`make-variable-buffer-local` 在加载时调用),所以 `setq-local` 在任何 buffer 都创建/覆盖 buffer-local。如果不是 buffer-local 变量,`setq-local` 第一次会 `make-local-variable`。

</details>

### 题 9

`default-value` 返回变量的"全局默认" (非 buffer-local 部分)。

```elisp
;; 怎么看某变量的全局默认值 (非 buffer-local)?
```

<details><summary>答案</summary>

```elisp
(default-value 'fill-column)
```

例如 `(setq-local fill-column 100)` 后,`(fill-column)` (变量引用) 在该 buffer 是 100,但 `(default-value 'fill-column)` 仍是 70 (默认)。

</details>

### 题 10

```elisp
;; 怎么让文件 .el 头自动启用 lexical-binding?
```

**答案**: 第一行 `;; -*- lexical-binding: t; -*-`

这是 file-local variable——`C-x C-v` reload 文件时,Emacs 读这行,设 `lexical-binding` 为 t (buffer-local)。这影响整个文件的 byte-compile 和 eval。

**新代码必须加这行**——dynamic binding 是历史遗留,不要用。

---

## 第三组: Functions (5 题)

### 题 11: my-compose

函数组合——`(my-compose f g)` 返回 `(lambda (x) (f (g x)))`。这是函数式编程的基本操作。

```elisp
(defun my-compose (f g)
  (lambda (x) (funcall f (funcall g x))))
```

实战: `(funcall (my-compose #'1+ #'square) 5)` = `(1+ (square 5))` = 26。组合让你"管道化"操作。

### 题 12: my-curry

curry 是"预先填充参数"——`(my-curry f x)` 返回 `(lambda (y) (f x y))`。

```elisp
(defun my-curry (fn x)
  (lambda (y) (funcall fn x y)))
```

实战: `(funcall (my-curry #'+ 5) 3)` = `(+ 5 3)` = 8。预先填充某些参数,得到部分应用函数。

注意: 这个 curry 只支持 binary function。真正 curry 支持任意参数数量——可以用 `apply-partially` (built-in)。

### 题 13: my-flip

flip 交换参数顺序——`(my-flip -)` 让 `(- a b)` 变成 `(- b a)`。

```elisp
(defun my-flip (fn)
  ;; (funcall (my-flip #'-) 1 2) → 1 (调换参数)
  )
```

<details><summary>答案</summary>

```elisp
(defun my-flip (fn)
  (lambda (a b) (funcall fn b a)))
```

实战: `(funcall (my-flip #'/) 2 10)` = `(/ 10 2)` = 5。Haskell 的 `flip` 一样。

</details>

### 题 14: my-identity

identity 是最简单的函数——返回参数本身。看似无用,但作为高阶函数的"占位符"有用。

```elisp
(defun my-identity (x) x)
```

实战: `(mapcar #'identity lst)` 复制 list (浅 copy)。`(funcall #'identity x)` 在需要"函数但不改值"时用。

### 题 15: my-constantly

constantly 返回"总是返回 V"的函数——忽略参数,总是返回 V。

```elisp
(defun my-constantly (val)
  (lambda (&rest _) val))
```

实战: `(mapcar (my-constantly 0) '(1 2 3))` → `(0 0 0)`。在需要"函数但忽略输入"时用——例如 default value 函数。

---

## 自测

1. 怎么设 buffer-local?
2. 怎么看默认值?
3. `&optional` 和 `&rest` 怎么用?
4. defvar 和 defcustom 区别?
5. condition-case 怎么用?

**答案**:
> 1. setq-local
> 2. default-value
> 3. (defun foo (a &optional b &rest args) ...)
> 4. defvar 简单;defcustom 加 customize
> 5. (condition-case err FORM (ERROR-TYPE BODY...))

---

## 毕业检查

- [ ] 15 题至少完成 12
- [ ] 能用 condition-case
- [ ] 能写高阶函数
- [ ] 理解 buffer-local

完成后进入 `../week-03-macros-loading/`。
