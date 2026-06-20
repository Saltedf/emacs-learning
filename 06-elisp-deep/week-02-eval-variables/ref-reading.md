# Week 2: Eval + Variables + Functions

> **Ref 章节**: symbols, eval, control, variables, functions
> **目标**: 深入求值规则、变量作用域、函数对象

这周是 Lisp 哲学的核心——理解 Elisp 怎么"思考"。Module 3 你学了"怎么写 defun、let、lambda"。Module 6 这周告诉你: 这些 form 背后的求值规则是什么,为什么 `(defvar x 5)` 后再 `(setq x 10)`,然后 reload 文件,x 还是 10 (不是 5)?为什么 Emacs 25 之前没有真正的闭包?为什么 dynamic binding 让人困惑?

这些问题不是 curiosity——它们影响你每天写的代码。不理解 eval,你写的 macro 会出诡异 bug;不理解 scope,你的 hook 会污染全局状态;不理解 closure,你的 counter 函数会"莫名其妙"共享状态。

---

## 1. Symbols (symbols.texi)

### 1.1 Symbol 的 4 个 slot

symbol 是 Elisp 的核心抽象——一个有名字的对象,内部有 4 个独立"格子":
1. **name** (string): symbol 的名字,如 `"foo"`
2. **value** (任意): 变量值 (setq 写到这里)
3. **function** (function): 函数 (defun 写到这里)
4. **plist** (plist): 任意元数据

这个设计是 Lisp-2 的实现基础——同一个 symbol 同时是变量和函数,因为它们在 symbol 上是不同的"槽"。

```elisp
(symbol-name 'foo)             ; → "foo"
(symbol-value 'foo)            ; → 值 (或 void)
(symbol-function 'foo)         ; → 函数 (或 void)
(symbol-plist 'foo)            ; → property list
```

理解这 4 个 slot 让所有 Elisp "诡异行为"变得清晰。例如:
- `(setq foo 5)` 写到 value slot
- `(defun foo () ...)` 写到 function slot
- `(put 'foo 'category 'test)` 写到 plist
- `(fmakunbound 'foo)` 清空 function slot (但 value 还在)
- `(makunbound 'foo)` 清空 value slot (但 function 还在)

### 1.2 intern (创建/查找)

symbol 存在 obarray (一个特殊的 hash table)。当你写 `'foo` 或 `(setq foo 5)`,Emacs 在 obarray 里查找 "foo"——存在则返回,不存在则创建。

```elisp
(intern "foo")                 ; → foo (找到或创建)
(intern-soft "foo")            ; → foo (只找,不创建)
(unintern 'foo)                ; 删除
(mapatoms (lambda (s) ...))    ; 遍历所有 symbol
```

`intern` 是 idempotent——多次调用返回同一个 symbol。`(intern "foo")` 和 `(intern "foo")` 是 `eq` 的 (同一个对象)。这让 symbol 比较 O(1)——`eq` 比指针,而 string `equal` 比内容。

`intern-soft` 是"只找不创建"——如果 symbol 不存在返回 nil。用于 check symbol 是否已 interned。

### 1.3 obarray

`obarray` = symbol 表 (全局命名空间)。它是一个特殊的 vector,内部用 hash 算法存 symbol。

```elisp
(setq my-obarray (make-vector 1511 0))
(intern "foo" my-obarray)
```

Module 6 大多数场景用默认 obarray——全局 `obarray` 变量。但你可以创建 private obarray,实现"私有命名空间"——一些宏和 DSL (domain-specific language) 用这个隔离 symbol。

`1511` 是素数——hash table size 用素数减少冲突。Emacs 内部 obarray 默认 size 是 1511,但可以 grow。

### 1.4 Property list

symbol 的第 4 个 slot 是 plist——可以存任意元数据。

```elisp
(put 'foo 'count 5)
(get 'foo 'count)              ; → 5
(symbol-plist 'foo)            ; → (count 5)
(remprop 'foo 'count)
```

某些 symbol 自带 properties (如 `face-documentation`)。Emacs 内部大量用 plist 存 metadata——例如 error type 的 `error-conditions` (Module 6 W4)、symbol 的 `risk-level` (file-local variable 安全等级)。

实战: 你可以用 symbol plist 实现"对象"——`(put 'my-obj 'field val)` 模拟 slot。但更推荐 cl-defstruct (record-based)。

---

## 2. Evaluation (eval.texi)

### 2.1 Eval 规则回顾

(Module 3 W1 学过)

Elisp 的 eval 规则极简——只有几条:

```
form → 
  - atom (number, string, vector, record): 自身
  - symbol: variable value
  - list (a b c):
    - a 是 special form: 特殊求值
    - a 是 macro: 展开后求值
    - 否则: eval a 得函数,eval 参数,调用
```

这是 Lisp "code is data" 的体现——每个 form 都按这几条规则 eval。简单到可以用 100 行 C 实现 (Emacs 的 eval.c 实际更长,因为优化和 special cases)。

对比其他语言:
- **Python**: AST 解释 + bytecode,复杂
- **JS**: AST → bytecode → JIT,极复杂
- **C**: 编译到机器码
- **Lisp**: 直接 eval (or compile to bytecode)

Lisp 的简单让 macro 可能——因为代码就是 list,evaluator 自然能"操作"它。

### 2.2 eval 函数

`eval` 是 meta-evaluator——给一个 form,求值它。

```elisp
(eval '(+ 1 2))               ; → 3
(eval '(+ 1 2) t)             ; → 3 (lexical)
(eval '(+ 1 2) nil)           ; → 3 (dynamic)
```

第二个参数是 lexical/dynamic 选择。`t` = lexical,`nil` = dynamic。这影响 form 内的 let 和 symbol-value 解析。

不推荐日常用 eval (慢,难分析)。但它有用——例如 `M-:` (`eval-expression`) 就用 eval;配置文件用 `eval-after-load` 等也是 eval。**只在确实需要"运行时生成代码"时用**——比如写 REPL、DSL 解释器。

### 2.3 apply-partially

`apply-partially` 是 partial application——预先填充部分参数。

```elisp
(setq add5 (apply-partially #'+ 5))
(funcall add5 3)              ; → 8
```

`(apply-partially #'+ 5)` 返回一个新函数,这个函数被调用时等价 `(+ 5 ARGS...)`。所以 `(funcall add5 3)` = `(+ 5 3)` = 8。

这是 currying 的简化——Lisp 通常用 lambda 实现相同效果: `(lambda (x) (+ 5 x))`。`apply-partially` 是 C 实现,稍快,且能用任意函数。

### 2.4 special forms

special form 是 " evaluator 自己处理的 form"——不按"eval 参数后调用"规则。例如 `if` 的 then/else 分支不都 eval (短路),`setq` 不 eval 第一个参数 (它就是要赋值的 symbol)。

| Form | 描述 |
|---|---|
| `quote` | 不 eval |
| `function` | 函数 slot |
| `if` | 条件 |
| `cond` | 多分支 |
| `when` / `unless` | 条件 |
| `and` / `or` | 短路 |
| `let` / `let*` | 局部 binding |
| `setq` / `set` | 设置 |
| `defvar` / `defcustom` | 定义变量 |
| `defun` | 定义函数 |
| `defmacro` | 定义宏 |
| `lambda` | 匿名函数 |
| `condition-case` | 异常 |
| `unwind-protect` | cleanup |
| `save-excursion` | 保存 point |
| `save-restriction` | 保存 narrowing |
| `save-current-buffer` | 保存 buffer |
| `track-mouse` | 鼠标 |
| `with-output-to-temp-buffer` | 临时输出 |
| `while` | 循环 |
| `catch` / `throw` | 非局部退出 |
| `block` / `return-from` (cl) | CL 风格 |

注意 `and`/`or` 是 special form——它们短路,只在需要时 eval 后续参数。`(and t (foo))` 如果 `(foo)` 副作用重要,你会期望它执行——确实执行,因为 t 为 true,and 继续。`(and nil (foo))` 不会执行 foo,因为 nil 已让 and 返回 nil。

### 2.5 catch / throw

`catch`/`throw` 是非局部退出——跳出多层嵌套。

```elisp
(catch 'done
  (dotimes (i 10)
    (when (= i 5)
      (throw 'done i)))
  'never)
;; → 5
```

类似 goto,但更安全——`throw` 必须有匹配的 `catch`,否则报错。`(throw 'done i)` 找最近的 `(catch 'done ...)`,把 i 作为返回值。

实战: 深度递归找到答案后跳出。例如遍历文件树找特定文件,找到后停止——`throw` 比显式 propagate "found" 状态简洁。

但要注意: `catch`/`throw` 跨越 `unwind-protect` 时,cleanup 仍跑——这是好的,确保资源释放。

### 2.6 condition-case (异常)

`condition-case` 是 Elisp 的 try/catch。

```elisp
(condition-case err
    (/ 1 0)
  (arith-error
   (message "Math error: %s" err)
   nil))
;; → nil, 打印 "Math error: (arith-error)"
```

`condition-case VAR FORM HANDLERS...`:
- VAR: 错误信息绑定
- FORM: 可能出错的代码
- HANDLERS: (ERROR-CONDITION BODY...) 对

`err` 绑定到错误对象——通常是 `(error-symbol . data)`。`(car err)` 是 error type,`(cdr err)` 是 data。

这是异常处理的基础。Module 6 W4 详细讨论各种 error type 和 debug 流程。

---

## 3. Control (control.texi)

### 3.1 sequencing (progn)

`progn` 顺序执行 body,返回最后一个 form 的值。

```elisp
(progn
  (foo)
  (bar)
  (baz))                       ; 返回 baz 的值
```

很多 form 内部用 progn——`let` body、`if` then 分支等都是隐式 progn。

### 3.2 inline (prog1, prog2)

`prog1`/`prog2` 是 progn 的变体——返回第 1/第 2 个 form 的值。

```elisp
(prog1 (foo)
  (bar))                       ; 返回 foo 的值

(prog2 (foo)
    (bar)
  (baz))                       ; 返回 bar 的值
```

实战: `(prog1 (do-setup) (do-cleanup))`——执行 setup 和 cleanup,但返回 setup 的结果。

### 3.3 条件 (review)

```elisp
(if COND THEN ELSE)
(when COND BODY...)
(unless COND BODY...)
(cond (TEST BODY...)...)
(pcase KEY PAT1-BODY PAT2-BODY...)   ; 模式匹配 (Emacs 25+)
(cl-case KEY (VAL BODY...) ...)       ; CL 风格
```

`if` 是基础,`when`/`unless` 是糖 (单分支 if)。`cond` 是 multi-branch,`pcase` 是模式匹配 (强,但语法怪)。

### 3.4 pcase

`pcase` 是 Emacs 25+ 的模式匹配——比 cond 强大得多。

```elisp
(pcase x
  ((pred numberp) "number")
  ((pred stringp) "string")
  ((and (pred consp) (app car 'foo)) "starts with foo")
  (`(a ,b ,c) (format "a, %s, %s" b c))
  (_ "other"))
```

`pred` 是 predicate 模式,`and` 是组合,`app` 是"应用函数到值再 match",`` `(a ,b ,c) `` 是 backquote 模式 (list 解构)。

`pcase` 让你写"看数据形状决定行为"的代码——这是函数式语言的核心 (Haskell、OCaml、Scala 都有)。Emacs 25 之后大量内置代码用 pcase。

(Module 6 W3 详细讨论 pcase-defmacro 和高级用法)

### 3.5 循环 (review)

```elisp
(while COND BODY...)
(dotimes (I N [RESULT]) BODY...)
(dolist (X LIST [RESULT]) BODY...)
(cl-loop ...)                  ; CL 风格,复杂
(cl-do ...)                    ; CL 风格
(mapc FN LIST)
(mapcar FN LIST)
```

`while` 是基础循环,`dotimes`/`dolist` 是糖 (数字/列表)。`cl-loop` 是 CL 的强力 loop DSL——可以写 SQL-like 查询:`(cl-loop for x in lst when (> x 5) sum x)`。

`mapc`/`mapcar` 是函数式——前者副作用 (不收集返回),后者收集。新手倾向用 mapcar (返回 list),但很多时候你只是想遍历,用 mapc 更高效 (不分配新 list)。

### 3.6 catch / throw / unwind-protect

```elisp
(unwind-protect
    (do-something-dangerous)
  (cleanup))                   ; 无论是否出错都跑
```

`unwind-protect` 是 try/finally——BODY 跑完后 (无论成功或 throw/error),CLEANUP 都跑。这是资源管理的基础——close file、unlock mutex、restore state 都用 unwind-protect。

Emacs 内置很多 `with-X` 宏都是 unwind-protect 的糖:
- `with-current-buffer`: 切 buffer,跑 body,切回
- `with-temp-file`: 创建 temp file,跑 body (写到 file),删 file
- `with-output-to-string`: 收集 stdout 到 string
- `save-excursion`: 保存 point/mark,跑 body,恢复

这些宏让你不用手动写 cleanup,代码更清晰。

---

## 4. Variables (variables.texi)

### 4.1 defvar vs defcustom vs setq

(Module 4 学过)

```elisp
(defvar my-var 5 "doc")        ; 默认值,docstring,标记 defvar
(defcustom my-opt 5 "doc" :type 'integer)  ; + customize
(setq my-var 10)               ; 直接设
(set 'my-var 10)               ; 同上
```

三者的设计意图:
- `defvar`: 模块级变量定义,带 docstring,有"如果已设则不覆盖"的语义
- `defcustom`: 用户可配置变量 (通过 customize UI)
- `setq`: 运行时设值 (用于改变 defvar'd 变量,或临时设)

### 4.2 defvar 行为

`defvar` 有"如果已设则不覆盖"的语义——这是为了支持用户的 init.el 自定义。

- 如果变量已设,defvar **不覆盖**
- 只设默认值 (`default-value`)

```elisp
(setq my-var 100)
(defvar my-var 5 "doc")        ; my-var 还是 100
```

为什么这么设计? 想象你的 package 有 `(defvar my-mode-fancy t)`。用户在 init.el 写 `(setq my-mode-fancy nil)`。如果 defvar 总是覆盖,每次 reload package 都会重置用户的 nil 成 t——体验糟糕。"如果已设则不覆盖"让用户设置优先。

代价: 你不能在 defvar 后用 defvar 改默认值。如果真要改,用 `(setq-default my-var new-default)`。

### 4.3 defconst

```elisp
(defconst +pi+ 3.14159 "Pi")
```

类似 defvar,但表示"常量"。byte-compiler 会**inline** 引用 (更高效)——byte-compile 后,引用 `+pi+` 的地方直接是 `3.14159`,不需要 lookup。

不要 setq 改 const——byte-compile 后那个引用已经是字面量,setq 不影响它。所以 `(setq +pi+ 3)` 后,有些地方还是 3.14159,有些是 3,行为不一致。

命名约定: 全大写 `+CONST+` 或 `*const*` (老式) 表示 const。

### 4.4 let-scan

`let` 创建局部 binding——只在这个 let 范围内,变量有值。

```elisp
(let ((x 1) (y 2))
  (+ x y))
;; → 3

(let* ((x 1) (y (1+ x)))       ; y 看到 x
  (* x y))
;; → 2
```

`let` 和 `let*` 的区别: `let` 所有 binding 并行 (用 let 外的值),`let*` 顺序 (后面的 binding 看到前面的)。

实战 99% 用 `let*`——更直觉。但 `let` 在性能上略好 (并行 binding 一次设置)。

### 4.5 buffer-local

buffer-local 是 Emacs 独特的设计——变量在每个 buffer 有独立值。这让 major mode、minor mode 能"per-buffer 配置"。

```elisp
(setq-local indent-tabs-mode nil)
;; 当前 buffer 局部

(make-local-variable 'my-var)
(setq my-var 100)              ; 这个 buffer 100

(make-variable-buffer-local 'my-var)
;; 所有 buffer 都自动 buffer-local

(default-value 'my-var)        ; 默认值 (非 buffer-local)
(buffer-local-value 'my-var (current-buffer))

(kill-local-variable 'my-var)  ; 移除 buffer-local binding
```

`setq-local` 是糖——`(setq-local x v)` = `(make-local-variable 'x) (setq x v)` (但只在第一次 make)。这让 buffer 第一次 setq-local 时创建 buffer-local binding。

`make-variable-buffer-local` 是"全局 buffer-local"——所有 buffer (即使将来创建的) 都自动 buffer-local。用于 indent-tabs-mode、tab-width 等所有 buffer 都需要的变量。

### 4.6 file-local / dir-local

文件级 / 目录级变量——通过文件头或 `.dir-locals.el` 设置。

```elisp
;; 文件头:
;; -*- mode: python; fill-column: 80; my-var: 100 -*-

;; 文件尾:
;; Local Variables:
;; my-var: 100
;; End:

;; dir-local (.dir-locals.el):
((python-mode . ((my-var . 100)))
 (js-mode . ((my-var . 200))))
```

这让你"per-file"、"per-project" 配置——比如某个项目用 tab,另一个用 space。

**安全警告**: file-local 变量可以是任意 Lisp——恶意文件可以执行代码。Emacs 有 `enable-local-variables` 控制是否自动应用,以及 risk-level 系统 (变量名含特殊字符或值是 form 时需用户确认)。

### 4.7 lexical binding (review)

lexical binding 是 Emacs 25+ 的现代模式——开启后,let 创建 lexical scope (静态),闭包工作正确。

```elisp
;; -*- lexical-binding: t; -*-

(defun make-counter ()
  (let ((count 0))
    (lambda ()
      (setq count (1+ count))
      count)))
```

`(make-counter)` 返回一个 lambda,这个 lambda 持有 count。即使外层 let 退出,count 仍活——因为 lambda 闭包持有它。每次调用 lambda,count 增加。

(Module 3 W3 学过)

### 4.8 变量陷阱

dynamic binding 是 Elisp 默认 (除非 lexical-binding: t)。dynamic 让 defvar'd 变量"穿透"函数边界——任何函数都能看到外层 let 的 binding。

```elisp
(defvar x 5)
(let ((x 10))                  ; dynamic (x 是 defvar'd)
  (my-func-using-x))           ; 看到 10

;; 但:
(let* ((y 5))
  (my-func-using-y))           ; lexical (y 不是 defvar'd)
                              ; my-func 看不到 y
```

经验: **defvar'd 变量是动态**,let 绑定时也动态。这意味着你的 `(let ((my-temp ...)) ...)` 如果 `my-temp` 是 defvar'd,会污染所有在该 let 范围内调用的函数的 `my-temp` 视图。

这是老 Elisp 代码常见的 bug 来源——动态作用域让"局部变量"不真正局部。lexical binding (Emacs 25+) 解决了这个问题。新代码**必须**用 lexical-binding: t。

---

## 5. Functions (functions.texi)

### 5.1 defun (review)

```elisp
(defun NAME (ARGLIST)
  "DOC"
  BODY...)
```

`defun` 是 special form——它把 lambda 存到 symbol 的 function slot。

### 5.2 lambda (review)

```elisp
(lambda (ARGLIST) BODY...)
```

`lambda` 创建匿名函数。它返回 function 对象——一个 closure (如果 lexical) 或 lambda list (如果 dynamic)。

### 5.3 interactive (review)

(Module 4 学过)

`interactive` 让函数成为 command——可以从 M-x 调用、绑 key。

### 5.4 argument types

```elisp
(defun my-func (a b &optional c &rest args)
  ...)
```

- `a`, `b`: required
- `&optional`: 可选
- `&rest`: 收集剩余
- `&key` (cl-defun): 关键字

```elisp
(cl-defun my-func (&key name age)
  (list name age))

(my-func :name "Alice" :age 30)
```

`cl-defun` (来自 cl-lib) 提供 CL 风格参数——支持 `&key`、`&aux`、default values、类型声明。比 `defun` 强但稍微慢 (参数解析)。

### 5.5 function description

```elisp
(defun foo (a b)
  "Add A and B.
A and B should be numbers."
  (+ a b))

(documentation 'foo)          ; → "Add A and B.\nA and B should be numbers."
```

docstring 第一行是 summary (`C-h f` 显示在第一行),后续是 detail。Emacs 风格: 大写参数名 (如 `A`、`B`),让 help mode 高亮。

`(documentation 'foo)` 返回 docstring——可以用于自定义 help 系统。

### 5.6 function 引用

```elisp
(symbol-function 'foo)        ; → function object
(function foo)                ; → function
#'foo                         ; → function (糖)
(funcall #'foo 1 2)
(apply #'foo '(1 2))
```

`#'foo` 是 `(function foo)` 的糖——返回 foo 的 function slot。`funcall` 调用函数对象 (函数作为值传时),`apply` 类似但参数是 list。

为什么要 `#'`? 在 Lisp-2,`'foo` 是 symbol,不是 function。如果你传 `'foo` 给 `mapcar`:
- `(mapcar 'foo '(1 2 3))` ——symbol 'foo 被解释为 function (因为 mapcar 内部调 symbol-function)。但 byte-compiler 警告。
- `(mapcar #'foo '(1 2 3))` ——直接传 function 对象,byte-compiler 优化更好。

**总是用 `#'foo` 传函数**。

### 5.7 functionp

```elisp
(functionp #'foo)             ; → t
(functionp 'foo)              ; → t (Emacs 25+)
(functionp (lambda ()))       ; → t
(functionp 5)                 ; → nil
```

`functionp` 检查对象是否可以作为函数——返回 t 如果是 subr (C 函数)、byte-code function、lambda、closure,或者 (Emacs 25+) 是 symbol with function。

### 5.8 closure

闭包是 lexical binding 的直接推论——不是魔法。当 lambda 在某个 let 内定义,它"看到"那些局部变量。lexical binding 下,这些变量 binding 被打包进 lambda 对象,即使外层 let 退出,变量仍活 (因为闭包持有引用)。

```elisp
(let ((counter 0))
  (defun my-counter ()
    (setq counter (1+ counter))
    counter))
;; my-counter 闭包持有 counter
```

这听起来抽象,但实战上极强。你可以做"私有状态"——一个 lambda 持有 counter,外部代码只能通过调用它来改 counter,没有全局命名空间污染。这是"对象系统"的轻量替代——不要 defclass、不要 method,只要一个 lambda 持有 state。Module 3 W3 的 `make-counter` 是最简例子,但同样的模式可以做出"账户" (存取款)、"缓存" (memoization)、"事件发射器" (订阅/通知)。

JavaScript 程序员对这个不陌生——JS 的闭包和 Elisp 的一样 (都是 lexical)。但 Elisp 在 1985-2017 间默认 dynamic,没有闭包。Emacs 25 (2017) 引入 lexical binding,才让 Elisp 进入"现代"时代。这是为什么很多老 Elisp 代码"看起来奇怪"——它在 dynamic binding 下写的,行为和 lexical 不同。

lexical binding 才有闭包。dynamic binding 下,"闭包"实际上捕获的是动态环境——每次调用时查找,所以同一个 lambda 在不同调用看到不同值。

### 5.9 advice

advice 是"包裹函数"——在原函数前后加 hook 代码。

```elisp
(define-advice my-func (:around (orig-func arg) my-logger)
  (message "calling with %s" arg)
  (let ((result (funcall orig-func arg)))
    (message "result: %s" result)
    result))

(advice-add 'my-func :around
            (lambda (orig arg)
              (message "before")
              (funcall orig arg)
              (message "after")))

(advice-remove 'my-func my-logger)
```

`:around` 是最灵活的 advice——你能控制何时调原函数、看返回值。`:before`/`:after` 是糖——只在前/后跑,不能改返回。

实战: 加日志、改默认行为、修复 bug 而不改源码。例如 advice `find-file` 让它自动转码。

(Module 6 W3 详细)

### 5.10 function alias

```elisp
(defalias 'my-foo #'foo "alias")
(my-foo ...)                  ; 调用 foo
```

`defalias` 让一个 symbol 指向另一个 symbol 的 function——`(my-foo ...)` 实际调 foo。这是 rename 或提供短名的标准方式。

---

## 6. 自测

1. `(symbol-plist 'foo)` 返回什么?
2. 怎么捕获异常?
3. `&optional` 和 `&rest` 区别?
4. `#'foo` 等于什么?
5. buffer-local 变量怎么设?

**答案**:
> 1. property list
> 2. condition-case
> 3. &optional 单个可选;&rest 收集剩余
> 4. (function foo)
> 5. setq-local 或 make-local-variable

---

## 7. Exercises

### Ex 2.1: my-retry
```elisp
(defun my-retry (fn &optional times)
  ;; 重试 N 次,失败返回 nil
  )
```

<details><summary>答案</summary>

```elisp
(defun my-retry (fn &optional times)
  (setq times (or times 3))
  (let ((result nil)
        (success nil))
    (while (and (not success) (> times 0))
      (condition-case nil
          (progn
            (setq result (funcall fn))
            (setq success t))
        (error (setq times (1- times)))))
    (if success result nil)))
```

</details>

### Ex 2.2: my-memoize
```elisp
(defun my-memoize (fn)
  ;; 返回 memoized 版
  )
```

<details><summary>答案</summary>

```elisp
(defun my-memoize (fn)
  (let ((cache (make-hash-table :test 'equal)))
    (lambda (arg)
      (or (gethash arg cache)
          (let ((result (funcall fn arg)))
            (puthash arg result cache)
            result)))))
```

</details>

### Ex 2.3: my-with-timer
```elisp
(defmacro my-with-timer (&rest body)
  ;; 跑 BODY,打印用时
  )
```

<details><summary>答案</summary>

```elisp
(defmacro my-with-timer (&rest body)
  `(let ((start (float-time)))
     ,@body
     (message "took %.3fs" (- (float-time) start))))
```

</details>

### Ex 2.4: my-curry
```elisp
(defun my-curry (fn x)
  ;; 返回 (lambda (y) (funcall fn x y))
  )
```

<details><summary>答案</summary>

```elisp
(defun my-curry (fn x)
  (lambda (y) (funcall fn x y)))
```

</details>

### Ex 2.5: my-pipeline
```elisp
(defun my-pipeline (&rest fns)
  ;; (my-pipeline #'1+ #'1+) → (lambda (x) (1+ (1+ x)))
  )
```

<details><summary>答案</summary>

```elisp
(defun my-pipeline (&rest fns)
  (lambda (x)
    (dolist (fn fns x)
      (setq x (funcall fn x)))))
```

</details>

---

## 8. 下一步

进入 `concept-anchor.md` + `exercises.md`,然后 `../week-03-macros-loading/`。
