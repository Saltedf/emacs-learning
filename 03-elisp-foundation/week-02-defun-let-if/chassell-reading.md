# Week 2: defun + let + if — 写真正的函数

> **目标**: 学会 defun、let、if、interactive、buffer 操作函数
> **时长**: 1 周 (~20 小时)
> **Chassell 章节**: 3. How To Write Function Definitions, 4. A Few Buffer-Related Functions

---

## 0. 这周在学什么

上周你学了 list 的结构和操作。你能在数据层面处理 list——`car`、`cdr`、`cons`、`append`。

这周你学**怎么定义函数、怎么用局部变量、怎么做条件**。这是从"数据处理"到"程序构建"的飞跃。完成这周,你能:

- 写自己的命令 (绑定到键位): 把日常重复操作自动化
- 用 let 创建局部变量: 避免污染全局命名空间
- 用 if 做条件判断: 处理"非此即彼"的逻辑
- 用 interactive 让函数变命令: 让函数可被 M-x 触发
- 写操作 buffer 的实用工具: 编辑器的真正"扩展"

这周内容比 W1 实用得多——你写的每个函数都能立刻用到日常编辑。不要光看不练,每学一个概念就写一个解决真实问题的函数。

---

## 1. defun: 定义函数 (Chassell Ch 3)

`defun` 是 Emacs Lisp 最常用到的 special form——你写的每一个函数、每一个命令都用它。掌握 defun,你就有了"扩展 Emacs"的基本功。

### 1.1 defun 的语法

先看模板:

```elisp
(defun NAME (ARGLIST)
  "DOCSTRING"
  BODY...)
```

四个部分,每个都有讲究:

- **NAME**: 函数名,通常全小写,用连字符 `my-func`。Elisp 命名约定: 词之间用 `-` (不是下划线 `_`,那是 C/Python 风格)。
- **ARGLIST**: 参数列表,可以为空 `()`。参数名也用连字符风格,如 `buffer-name`、`file-path`。
- **DOCSTRING**: 函数文档 (强烈推荐),第一行应简洁总结。docstring 是 Elisp 文化的核心——没有 docstring 的函数被认为"未完成"。
- **BODY**: 函数体,最后一个 form 的值是返回值。Lisp 没有 `return` 关键字——返回值由"最后一个 form 的值"决定。

### 1.2 例子

理论讲完,看几个具体例子。从简单到复杂:

```elisp
(defun square (x)
  "Return X squared."
  (* x x))

(square 5)              ;; → 25
(square -3)             ;; → 9

(defun greet (name)
  "Return a greeting string for NAME."
  (format "Hello, %s!" name))

(greet "Alice")         ;; → "Hello, Alice!"

(defun add-three (a b c)
  "Return A + B + C."
  (+ a b c))

(add-three 1 2 3)       ;; → 6
```

注意几个 Elisp 风格:

- docstring 里参数名用大写 (X, NAME, A, B, C)。这是 Elisp 约定,`C-h f square RET` 显示时会高亮。
- docstring 第一行是"summary",简洁,通常一个动词开头 ("Return..."、"Insert..."、"Delete...")。
- 函数体最后一行是返回值。`square` 的最后一行是 `(* x x)`,所以返回它的值。

### 1.3 没有参数

有些函数不需要参数——它们要么永远返回固定值,要么靠副作用工作:

```elisp
(defun always-42 ()
  "Always return 42."
  42)

(always-42)             ;; → 42
```

`always-42` 的参数列表是 `()` (空 list),调用时不传参。这种函数常用于"配置查询"或"动作命令" (后者通过 interactive)。

### 1.4 默认返回值

Lisp 没有 `return` 关键字——这是和 Python/C 最大的不同之一。函数的返回值是**最后一个 form 的值**。

```elisp
(defun foo (x)
  (message "computing...")  ;; 副作用,值丢弃
  (* x x))                   ;; 这是返回值

(foo 5)                  ;; → 25 (minibuffer 显示 "computing...")
```

`foo` 的 body 有两个 form:

- `(message "computing...")` 副作用 (打印),值是字符串 `"computing..."` 但被丢弃
- `(* x x)` 是最后一个 form,它的值是返回值

为什么 Lisp 这么设计?因为 Lisp 的设计哲学是"表达式都有值"。一个函数是一个表达式,它的值就是最后一个子表达式的值。这比 `return` 更"数学化"——函数像数学函数 f(x) = x*x,不是"流程"。

### 1.5 early return (用 if/cond 包裹)

那"提前退出"怎么办? Python 写 `if bad: return None`,Lisp 没 return,怎么做?

答案是: **改写控制流**,用 if/cond 包裹:

```elisp
(defun safe-divide (a b)
  "Divide A by B, return nil if B is zero."
  (if (zerop b)
      nil                          ;; 提前退出 (用 if)
    (/ a b)))                      ;; 否则计算
```

注意结构: if 的 then 分支是"提前退出值",else 分支是"正常计算"。整个 if 是函数最后一个 form,所以返回 if 的值。

或用 `when` / `unless`:

```elisp
(defun safe-divide (a b)
  (when (not (zerop b))
    (/ a b)))
```

`when` 没有 else——条件真就执行 body,否则返回 nil。这写法更简洁,但只适合"早期检查 + 默认 nil"的场景。

### 1.6 docstring 的重要性

docstring 是 Elisp 文化的核心。这不是夸张——Emacs 社区把 docstring 当成"代码的一部分",不是"可选注释"。

```elisp
(defun square (x)
  "Return X squared.
X should be a number."
  (* x x))
```

按 `C-h f square RET` 看到 docstring。你的函数被自文档化——别人不用读源码,只看 docstring 就知道怎么用。

为什么这么重要?因为 Emacs 是"自文档编辑器": 所有内置函数、变量、keymap 都有 docstring。`C-h f`、`C-h v`、`C-h k` 这些命令把 docstring 拉出来给你看。如果用户函数没 docstring,Emacs 的"自文档"特性就断了。

**规则**:
- 第一行: 简洁总结 (一句话,~70 字符)
- 空行后: 详细说明
- 参数名大写 (X, NAME)
- 用 imperative ("Return X squared",不是 "Returns")

不写 docstring 的代码 = 不完整的代码。这是 Elisp 的"绅士准则"。

### 1.7 安装 defun (Chassell)

写完 defun 后,要"安装":

```
;; 把光标放在 defun 内部
C-M-x        ;; eval-defun
```

或:

```
M-x eval-buffer RET    ;; eval 整个 buffer
```

或:

```
;; 把光标在 defun 末尾
C-x C-e        ;; eval-last-sexp
```

### 1.8 永久安装

写到 init.el 或某个 .el 文件,然后 `(require 'my-package)`。

(详见 Module 4)

---

## 2. interactive: 让函数变命令 (Chassell)

`interactive` 是 Emacs 的"魔法咒语"——它让普通函数变成"命令",可以被 M-x 或键位触发。这是 Emacs "自文档编辑器 + 可扩展性"的核心机制。

### 2.1 普通 defun 不能用 M-x

先看反例: 不加 interactive 的函数:

```elisp
(defun hello ()
  (message "Hello!"))

M-x hello RET     ;; → error: commandp not satisfied
```

错误信息 "commandp not satisfied" 看起来神秘,其实是: Emacs 检查 `hello` 是否是 "command" (有 `(interactive ...)` 声明的函数),不是就拒绝 M-x 触发。

为什么这么严格?因为 M-x 是给"用户命令"用的,不是给所有内部函数。如果任何函数都能 M-x,minibuffer 补全会爆炸 (Emacs 有几千个函数),用户也分不清哪些是"命令"哪些是"工具函数"。

### 2.2 加 (interactive)

```elisp
(defun hello ()
  (interactive)
  (message "Hello!"))

M-x hello RET     ;; → "Hello!" 显示在 minibuffer
```

`interactive` 让函数成为 command,可用 M-x 或绑键。

`(interactive)` 不带参数——表示"这个命令不需要任何 minibuffer 输入"。直接调用,不读参数。

### 2.3 interactive codes

`(interactive "STRING")` 的 STRING 编码告诉 Emacs 怎么读参数。这是 Elisp 的"参数读取 DSL"——一个字符串编码复杂的输入逻辑。

| Code | 读什么 | 例子 |
|---|---|---|
| `s` | string | `(interactive "sName: ")` |
| `n` | number | `(interactive "nAge: ")` |
| `f` | existing file | `(interactive "fFile: ")` |
| `F` | file (可能不存在) | `(interactive "FFile: ")` |
| `b` | buffer | `(interactive "bBuffer: ")` |
| `B` | buffer (可能不存在) | `(interactive "BBuffer: ")` |
| `d` | point (位置) | `(interactive "d")` |
| `r` | region 范围 | `(interactive "r")` → (start end) |
| `p` | prefix arg (数字) | `(interactive "p")` |
| `P` | prefix arg (raw) | `(interactive "P")` |
| `M` | sexp | `(interactive "M")` |

每个 code 对应一种 minibuffer 提示。`s` 读字符串,`n` 读数字,`f` 读已存在文件 (tab 补全),等等。

为什么用 codes 而不是函数?这是 1980 年代的"省字节"考虑——单字符 code 比函数调用短。今天看有点 arcane,但 Elisp 几十年兼容承诺让它保留。

### 2.4 例子

```elisp
(defun greet (name)
  "Greet someone by NAME."
  (interactive "sName: ")
  (message "Hello, %s!" name))

M-x greet RET Alice RET   ;; → "Hello, Alice!"
```

`sName: ` 是 prompt——`s` 是 code (string),`Name: ` 是 prompt 文本。Emacs 读到 `s`,知道要 minibuffer 读字符串;读到 `Name: `,用它作 prompt。

```elisp
(defun square-interactive (n)
  "Insert N squared."
  (interactive "nNumber: ")
  (insert (format "%d" (* n n))))

M-x square-interactive RET 5 RET   ;; 插入 25
```

`n` code 读数字。注意函数参数 `n` 接收的是 number,不是 string——Emacs 自动转换。

```elisp
(defun operate-on-region (start end)
  "Count chars in region."
  (interactive "r")
  (message "Region has %d chars" (- end start)))
```

`r` code 特殊——它**两个参数**: region 起点 start,终点 end。所以函数 ARGLIST 要 `(start end)`。`r` 在用户没选 region 时无效 (报错)。

### 2.5 多个参数

```elisp
(defun insert-formatted (string times)
  "Insert STRING TIMES times."
  (interactive "sString: \nnTimes: ")
  (dotimes (_ times)
    (insert string)))

M-x insert-formatted RET foo RET 3 RET   ;; 插入 foofoofoo
```

注意 `\n` 分隔多个 prompt。

`\n` 分隔 interactive 字符串的"段",每段对应一个参数。`sString: \nnTimes: ` 意思: 先 prompt "String: " 读字符串,再 prompt "Times: " 读数字。

### 2.6 复杂 interactive (用 list)

如果 codes 不够用 (比如要做计算、条件判断),用 `(interactive (list ...))`:

```elisp
(defun complex-func (a b c)
  (interactive
   (list (read-from-minibuffer "A: ")
         (read-number "B: ")
         (y-or-n-p "C? ")))
  ;; ...
  )
```

`(interactive (list ...))` 允许任意逻辑。

`list` 的每个元素是一个表达式,Emacs 运行它们收集参数。比如 `(y-or-n-p "C? ")` 是个 yes/no 询问。这种灵活性让你写复杂命令——比如 "if region active, do X; else prompt for input"。

---

## 3. let: 局部变量 (Chassell let)

`let` 创建局部变量。这是 Elisp 避免"全局污染"的基本工具。理解 let,你就理解了 90% 的"局部作用域"。

### 3.1 为什么需要 let?

先看一个不用 let 的"坏例子":

```elisp
(defun circle-area (radius)
  (* 3.14159 radius radius))
```

这里 `3.14159` 是字面常数。如果你想用 pi 多次 (计算周长、面积、体积),每次都写 `3.14159` 又啰嗦又容易出错。能不能"先用 let 缓存"?

```elisp
(defun circle-area (radius)
  (let ((pi 3.14159))
    (* pi radius radius)))
```

`pi` 是**局部变量**,只在 let 内有效。函数外看不到 `pi` (用 `C-h v pi RET` 不会显示它)。

为什么需要局部?想象如果每次需要 pi 就 `(setq pi 3.14159)`——这会创建全局 `pi`,污染命名空间。如果两个函数都用 `pi` 作局部名,可能冲突。`let` 解决这个: 局部绑定,不污染全局。

### 3.2 let 语法

```elisp
(let ((VAR1 VAL1)
      (VAR2 VAL2)
      VAR3              ; 没有初始值,默认 nil
      ...)
  BODY...)
```

`let` 返回 BODY 最后一个 form 的值。

注意第 3 行 `VAR3` 不带括号——这种"裸 symbol"表示"绑定为 nil"。适合"先声明,后面 setq"的情况。

### 3.3 例子

```elisp
(let ((x 5)
      (y 10))
  (+ x y))                  ;; → 15
```

最简单的 let——绑定 x=5, y=10,加起来返回 15。

```elisp
(let ((x 5))
  (setq x (* x 2))          ;; 修改局部 x
  x)                        ;; → 10
```

let 内部可以 setq 修改局部变量——这种修改在 let 外不可见。

```elisp
(let (a b c)                ; 三个 nil
  (list a b c))             ;; → (nil nil nil)
```

三个变量都是 nil (没初始值)。

### 3.4 let vs let*

`let` 和 `let*` 的区别是 Module 3 最经典的"陷阱题"。一定要理解。

`let` 同时绑定 (parallel):

```elisp
(let ((x 1)
      (y 2))
  (let ((x y)         ; y 是外层的 2,这里 x = 2
        (y x))        ; x 是外层的 1 (没被新 x 影响)
    (list x y)))      ;; → (2 1)
```

内层 let 的所有 binding 同时生效——新 `x` 用外层 `y` (因为还没绑新 x),新 `y` 用外层 `x` (因为还没绑新 y)。结果 (2 1)。

`let*` 顺序绑定 (sequential):

```elisp
(let ((x 1)
      (y 2))
  (let* ((x y)        ; y 是外层的 2,这里 x = 2
         (y x))       ; x 是新的 2 (被前面的影响)
    (list x y)))      ;; → (2 2)
```

内层 let* 的 binding 顺序生效——先绑 `x=y` (用外层 y=2,得 x=2),再绑 `y=x` (用新 x=2,得 y=2)。结果 (2 2)。

**经验**: 90% 用 `let`;当某个 var 引用前面的 var 时用 `let*`。

为什么有这两个?`let` (parallel) 的好处是"无依赖"——编译器可以并行计算。`let*` (sequential) 的好处是"后面可见前面"——简化代码。实践中,如果你 binding 之间无依赖,用 let;有依赖 (后面的 var 用前面的 var),用 let*。

### 3.5 使用场景

`let` 有三个常见使用场景:

**场景 1**: 临时计算

```elisp
(defun circle-area (radius)
  (let ((pi 3.14159)
        (r-squared (* radius radius)))
    (* pi r-squared)))
```

把中间计算 (r-squared) 命名,代码更可读。比直接 `(* pi (* radius radius))` 清楚。

**场景 2**: 缓存 (避免重复计算)

```elisp
(defun process-buffer ()
  (let ((content (buffer-string))
        (name (buffer-name)))
    ;; 用 content 和 name 多次,不重复调用
    (message "Processing %s (%d chars)" name (length content))))
```

如果不 let,每次用 `(buffer-string)` 都重新调用——慢。let 缓存一次,多次使用。

**场景 3**: 默认值

```elisp
(defun greet (name &optional greeting)
  (let ((greeting (or greeting "Hello")))
    ;; greeting 要么是传入的,要么默认 "Hello"
    (format "%s, %s!" greeting name)))
```

`or` 返回第一个非 nil 的——`(or greeting "Hello")` 给 greeting 一个默认值。这是 Elisp "默认参数"的常用模式 (Elisp 没真正的 default arg)。

---

## 4. if: 条件 (Chassell if)

`if` 是最基础的条件表达式。Lisp 的 if 和 Python/C 的 if 在概念上类似,但语法不同——它是表达式 (有值),不是语句。

### 4.1 if 语法

```elisp
(if CONDITION
    THEN-FORM
  ELSE-FORM...)
```

- CONDITION 为 non-nil (truthy) → 执行 THEN-FORM
- 否则 → 执行 ELSE-FORM (可以为多个)

注意 Lisp 的 if 是**表达式**——它有值,可以嵌入到其他表达式里。Python 的 `if` 是语句,不能写 `result = if cond: x else: y` (除非用三元 `x if cond else y`)。Lisp 的 if 天然嵌入。

### 4.2 例子

```elisp
(if (> 5 3)
    "yes"
  "no")                  ;; → "yes"

(if (< 5 3)
    "yes"
  "no")                  ;; → "no"

(if nil
    "never"
  "always")              ;; → "always"
```

读法: 第一个 form 是条件,第二个是 then,第三个 (及之后) 是 else。

### 4.3 没有 else

```elisp
(if (boundp 'my-var)
    (message "my-var is bound"))
;; 如果没绑定,返回 nil (没有 else)
```

如果条件 false 且无 else,if 返回 nil。这是合法的——很多时候你只关心 "true 时做某事",不关心 false。

### 4.4 多个 else (隐式 progn)

`if` 有个"不对称"——THEN 只能一个 form,ELSE 可以多个。这是新手困惑点:

```elisp
(if condition
    (progn
      ;; then 多个 form
      form1
      form2)
  ;; else 多个 form (不用 progn)
  form3
  form4)
```

THEN 只能**一个 form**;ELSE 可以多个 (隐式 progn)。
这是 `if` 的不对称。

为什么这么设计?历史原因 + 实用主义。`if` 经常用于"then 是简单值,else 是复杂处理"的场景 (比如错误处理),所以 else 部分给了便利 (多 form 隐式 progn)。如果你 then 部分要多 form,用 `(progn ...)` 包起来。

但这种不对称让代码丑。所以 Elisp 提供 `when` 和 `unless`——它们更"对称"。

### 4.5 when / unless (常用替代)

`when` = if without else:

```elisp
(when CONDITION
  FORM1
  FORM2)                  ;; 多个 form,隐式 progn
```

`when` 允许多个 body form (不像 if 的 then 只一个)。条件真执行所有 form,返回最后一个的值;条件假返回 nil。

```elisp
(when (> x 0)
  (message "positive")
  (sqrt x))
```

`unless` = if not:

```elisp
(unless CONDITION
  FORM...)
```

`unless` 等于 `(when (not CONDITION) ...)`。条件假执行 body。

```elisp
(unless (string-empty-p s)
  (message "non-empty"))
```

**经验**: 多数场景用 `when`/`unless` 更清晰,只有真正需要 then/else 二选一才用 `if`。

读法: 用 `when` 时,你想"如果 A,就做这些";用 `if` 时,你想"如果 A 做 X,否则做 Y"。两者心态不同。

### 4.6 Truth and Falsehood (Chassell)

Lisp 的真假规则是新手最大的"惊讶点":

- `nil` 是 false
- **其他一切** 是 true (包括 0、空字符串、空 list!)

```elisp
(if '()                ;; → nil 是 false
    "yes" "no")        ;; → "no"

(if 0                  ;; 0 是 true!
    "yes" "no")        ;; → "yes"

(if ""                 ;; 空字符串是 true!
    "yes" "no")        ;; → "yes"
```

**注意**: 这和 Python (`if not []`)、C (`if (!0)`) 不同。
新手容易踩坑。

为什么 Lisp 这么设计?历史 + 数学。1958 年 McCarthy 选了"只有 nil 是 false"——简化了语义 (你只需要记住一个 false 值),也契合 Lisp 的"空 list = nil" (空 list 自然是 false)。

但这带来一个陷阱: `(if (length str) ...)`——你以为"非空字符串返回 true",但 `(length "")` 是 0,0 是 true,所以条件永远 true。要写 `(if (> (length str) 0) ...)` 或 `(if (not (string-empty-p str)) ...)`。

### 4.7 常用判断函数

Elisp 用后缀 `-p` 表示 "predicate" (判断函数):

```elisp
(null X)              ;; X 是 nil
(not X)               ;; 同 null
(atom X)              ;; X 不是 cons cell
(consp X)             ;; X 是 cons cell
(listp X)             ;; X 是 list (nil 或 cons)
(integerp X)
(numberp X)
(stringp X)
(symbolp X)
(functionp X)
(arrayp X)
(bufferp X)
(windowp X)
(framep X)
(keymapp X)
```

后缀 `-p` 是 "predicate" 的简写。

为什么这么命名?因为 Elisp 命名空间不分类型——一个名字是 symbol,可以是函数也可以是变量。`-p` 后缀让你一眼看出"这是判断函数"。这是 Elisp 文化: predicate 一律 `-p` 结尾。

---

## 5. Buffer-Related Functions (Chassell Ch 4)

Buffer 是 Emacs 的"工作对象"——你编辑的每个文件都是一个 buffer。掌握 buffer 操作,你就掌握了 Emacs 的核心。

这一节列出常用 buffer 函数。它们都是你日常用的命令的"Lisp 版本"——比如 `forward-word` 就是 `M-f`。

### 5.1 当前 buffer

```elisp
(current-buffer)              ;; → #<buffer foo.txt>
(buffer-name)                 ;; → "foo.txt"
(buffer-file-name)            ;; → "/home/sun/foo.txt"
```

这三个最常用: `(current-buffer)` 返回 buffer 对象本身,`(buffer-name)` 返回名字 (string),`(buffer-file-name)` 返回关联文件路径 (或 nil 如果没文件)。

### 5.2 切换 buffer

```elisp
(set-buffer NAME)             ;; 切换 (不显示)
(switch-to-buffer NAME)       ;; 切换并显示
(pop-to-buffer NAME)          ;; 在另一个 window 显示
(with-current-buffer NAME BODY...)  ;; 临时切换,执行 BODY,切回
```

**`with-current-buffer` 最常用**:

```elisp
(with-current-buffer "*scratch*"
  (buffer-size))              ;; → *scratch* 的大小
;; 即使当前是别的 buffer
```

为什么 with-current-buffer 比 set-buffer 好?因为它是"防御性"的——即使 BODY 报错,buffer 也自动切回。set-buffer 不会,留下不一致状态。这是 Elisp 风格: 任何改变全局状态的操作,都用 with-* 或 save-* 包起来。

### 5.3 创建/获取 buffer

```elisp
(get-buffer NAME)             ;; 找,没有返回 nil
(get-buffer-create NAME)      ;; 找或创建
(generate-new-buffer-name "foo")  ;; → "foo<2>" 等
(make-indirect-buffer BASE NAME)  ;; indirect buffer
```

`get-buffer-create` 常用——保证有 buffer,没有就建。比如显示一个特殊 buffer (`*My Todos*`),用 `get-buffer-create` 拿到它。

### 5.4 杀 buffer

```elisp
(kill-buffer NAME)
(kill-buffer-and-its-windows)
```

`kill-buffer` 关闭 buffer (释放内存)。如果 buffer 有未保存修改,会问用户。

### 5.5 point 和 mark

```elisp
(point)                       ;; 当前 point (integer)
(point-min)                   ;; buffer 最小 (narrowing 后可能 > 1)
(point-max)                   ;; 最大
(mark)                        ;; mark 位置
(mark-active-p)               ;; mark 是否激活
(region-beginning)            ;; region 起点 (min of point/mark)
(region-end)
```

`point` 是光标位置 (整数,buffer 内的字符 index)。`(point-min)` 是 buffer 起点 (通常 1,但 narrow 后可能更大)。`(mark)` 是 mark 位置 (C-SPC 设的)。`(region-beginning)` / `(region-end)` 是 region 的两端。

### 5.6 移动

```elisp
(goto-char POS)
(forward-char N)
(backward-char N)
(forward-word N)
(backward-word N)
(forward-line N)
(beginning-of-line)
(end-of-line)
(beginning-of-buffer)
(end-of-buffer)
```

这些都是你键位背后的函数——`forward-char` = C-f,`forward-word` = M-f。你可以在自己代码里调用它们。

注意: 这些函数改 point。如果不想破坏用户光标,用 `save-excursion` 包起来。

### 5.7 读/写 buffer 内容

```elisp
(char-after POS)              ;; POS 处的字符
(char-before POS)
(buffer-substring START END)  ;; 取 [START, END) 文本
(buffer-substring-no-properties START END)
(buffer-string)               ;; 整个 buffer
(thing-at-point 'word)        ;; 光标处的 word
(thing-at-point 'sentence)
(thing-at-point 'line)
(thing-at-point 'sexp)
(thing-at-point 'url)
(thing-at-point 'email)
```

`thing-at-point` 是"魔法函数"——它能识别光标处的各种"thing" (word, sentence, url, email, ...)。Module 6 详细讲,现在你只要知道它能用。

### 5.8 修改 buffer

```elisp
(insert "text")               ;; 在 point 插入
(insert-char ?a 5)            ;; 插入 5 个 a
(delete-char N)               ;; 删 N 个字符
(delete-backward-char N)
(delete-region START END)
(delete-and-extract-region START END)  ;; 删并返回
(save-excursion BODY...)      ;; 保存 point/mark,执行 BODY,恢复
```

`save-excursion` 是 buffer 操作的标准包装——保存 point/mark/narrowing,执行 body,恢复。任何改 point 的操作都该用它包起来 (除非你**故意**改 point)。

### 5.9 search

```elisp
(search-forward "text")       ;; 找下一个,移 point
(search-backward "text")
(re-search-forward "pattern")
(re-search-backward "pattern")
(search-forward "text" BOUND NOERROR)
```

`search-forward` 字面搜索,`re-search-forward` 正则搜索。第二个参数 BOUND 是搜索范围 (nil = 找到底),第三个 NOERROR 控制找不到时是否报错 (t = 不报错返回 nil)。

### 5.10 例子: 一个实用的 buffer 函数

把上面的函数组合起来,写"数 buffer 里所有 word":

```elisp
(defun my-count-words-buffer ()
  "Count words in current buffer."
  (interactive)
  (save-excursion
    (goto-char (point-min))
    (let ((count 0))
      (while (re-search-forward "\\w+" nil t)
        (setq count (1+ count)))
      (message "Buffer has %d words" count))))
```

读法:
- `save-excursion` 保存 point/mark
- `(goto-char (point-min))` 跳到 buffer 头
- `(re-search-forward "\\w+" nil t)` 找下一个 word (nil = no bound, t = 不报错)
- `while` 循环,每次 count +1

这是 `re-search-forward + while` 的经典模式——"在 buffer 里遍历所有匹配"。你在自己的代码里会反复用到。

### 5.11 例子: 操作 region

```elisp
(defun my-uppercase-region (start end)
  "Uppercase the region."
  (interactive "r")
  (let ((text (buffer-substring start end)))
    (delete-region start end)
    (goto-char start)
    (insert (upcase text))))
```

(实际上 `(upcase-region START END)` 内置就有,这只是练习)

这是"取 region → 处理 → 放回"的标准模式。完成 W4 后,你可以用 lambda 抽象出"任意变换应用 region"的通用工具。

---

## 6. 实战练习

### Ex 2.1: circle-area

```elisp
(defun circle-area (radius)
  "Return area of circle with RADIUS."
  ;; 填写
  )

(circle-area 5)       ;; → ~78.54
```

**思路**: 面积 = π × r²。用 let 绑 pi,然后 `(* pi radius radius)`。注意 Emacs 有内置 `float-pi` (3.14159...)。

<details><summary>答案</summary>

```elisp
(defun circle-area (radius)
  (let ((pi 3.14159))
    (* pi radius radius)))
```

</details>

### Ex 2.2: celsius-to-fahrenheit

```elisp
(defun c-to-f (celsius)
  "Convert CELSIUS to Fahrenheit."
  ;; 填写
  )

(c-to-f 0)            ;; → 32
(c-to-f 100)          ;; → 212
```

<details><summary>答案</summary>

```elisp
(defun c-to-f (celsius)
  (+ (* celsius 9/5) 32))
```

</details>

### Ex 2.3: safe-nth

nth 但 nil-safe:

```elisp
(defun safe-nth (n list)
  "Return Nth element, or nil if out of range."
  ;; 填写
  )

(safe-nth 2 '(a b c))    ;; → c
(safe-nth 10 '(a b c))   ;; → nil
```

**思路**: 先检查 n 是否在 [0, length)。如果越界,返回 nil;否则用 nth。注意 nth 本身越界也返回 nil,这题主要训练 if 检查。

<details><summary>答案</summary>

```elisp
(defun safe-nth (n list)
  (if (or (< n 0) (>= n (length list)))
      nil
    (nth n list)))
```

</details>

### Ex 2.4: buffer-word-count

```elisp
(defun my-buffer-word-count ()
  "Count words in current buffer."
  ;; 填写
  )
```

<details><summary>答案</summary>

```elisp
(defun my-buffer-word-count ()
  (let ((count 0))
    (save-excursion
      (goto-char (point-min))
      (while (re-search-forward "\\w+" nil t)
        (setq count (1+ count))))
    count))
```

</details>

### Ex 2.5: insert-date

```elisp
(defun my-insert-date ()
  "Insert current date at point."
  (interactive)
  ;; 填写
  )

;; 绑到 C-c d:
(global-set-key (kbd "C-c d") 'my-insert-date)
```

<details><summary>答案</summary>

```elisp
(defun my-insert-date ()
  (interactive)
  (insert (format-time-string "[%Y-%m-%d %a]")))
```

</details>

### Ex 2.6: my-reverse-region

```elisp
(defun my-reverse-region (start end)
  "Reverse characters in region."
  (interactive "r")
  ;; 填写
  )
```

<details><summary>答案</summary>

```elisp
(defun my-reverse-region (start end)
  (interactive "r")
  (let* ((text (buffer-substring start end))
         (chars (string-to-list text))
         (reversed (nreverse chars)))
    (delete-region start end)
    (insert (apply #'string reversed))))
```

</details>

### Ex 2.7: copy-line

复制当前行到 kill-ring:

```elisp
(defun my-copy-line ()
  "Copy current line to kill-ring."
  (interactive)
  ;; 填写
  )
```

<details><summary>答案</summary>

```elisp
(defun my-copy-line ()
  (interactive)
  (let ((line (buffer-substring
               (line-beginning-position)
               (line-end-position))))
    (kill-new line)
    (message "Copied: %s" line)))
```

</details>

### Ex 2.8: kill-to-end-of-buffer

```elisp
(defun my-kill-to-end ()
  "Kill from point to end of buffer."
  (interactive)
  ;; 填写
  )
```

<details><summary>答案</summary>

```elisp
(defun my-kill-to-end ()
  (interactive)
  (kill-region (point) (point-max)))
```

</details>

### Ex 2.9: count-matches

```elisp
(defun my-count-matches (pattern)
  "Count occurrences of PATTERN in buffer."
  (interactive "sPattern: ")
  ;; 填写
  )
```

<details><summary>答案</summary>

```elisp
(defun my-count-matches (pattern)
  (interactive "sPattern: ")
  (let ((count 0))
    (save-excursion
      (goto-char (point-min))
      (while (re-search-forward pattern nil t)
        (setq count (1+ count))))
    (message "%d matches" count)))
```

</details>

### Ex 2.10: maybe-insert

如果光标处是空格,不插入,否则插入:

```elisp
(defun my-maybe-insert (text)
  "Insert TEXT if point is not at whitespace."
  (interactive "sText: ")
  ;; 填写
  )
```

<details><summary>答案</summary>

```elisp
(defun my-maybe-insert (text)
  (interactive "sText: ")
  (unless (looking-at "[ \t\n]")
    (insert text)))
```

</details>

### Ex 2.11: trim-string

```elisp
(defun my-trim-string (s)
  "Trim leading and trailing whitespace from S."
  ;; 填写
  )

(my-trim-string "  hello  ")     ;; → "hello"
```

<details><summary>答案</summary>

```elisp
(defun my-trim-string (s)
  (let ((result (replace-regexp-in-string "\\`[ \t\n]+" "" s)))
    (replace-regexp-in-string "[ \t\n]+\\'" "" result)))
```

</details>

### Ex 2.12: split-by-char

```elisp
(defun my-split-by-char (s char)
  "Split S by CHAR into list."
  ;; 填写
  )

(my-split-by-char "a,b,c" ?,)     ;; → ("a" "b" "c")
```

<details><summary>答案</summary>

```elisp
(defun my-split-by-char (s char)
  (let ((parts nil)
        (start 0))
    (while (string-match (regexp-quote (string char)) s start)
      (push (substring s start (match-beginning 0)) parts)
      (setq start (match-end 0)))
    (push (substring s start) parts)
    (nreverse parts)))
```

</details>

### Ex 2.13: my-when-file-exists

```elisp
(defun my-safe-find-file (path)
  "Open PATH if it exists, else error."
  ;; 填写
  )
```

<details><summary>答案</summary>

```elisp
(defun my-safe-find-file (path)
  (if (file-exists-p path)
      (find-file path)
    (error "File not found: %s" path)))
```

</details>

### Ex 2.14: average-of-list

```elisp
(defun my-average (list)
  "Return average of numbers in LIST."
  ;; 填写
  )

(my-average '(1 2 3 4 5))     ;; → 3
```

<details><summary>答案</summary>

```elisp
(defun my-average (list)
  (if (null list)
      0
    (/ (apply #'+ list) (float (length list)))))
```

</details>

### Ex 2.15: my-interactive-greet

提示名字,问候:

```elisp
(defun my-greet (name)
  "Greet NAME."
  (interactive "sWhat's your name? ")
  ;; 填写
  )
```

<details><summary>答案</summary>

```elisp
(defun my-greet (name)
  (interactive "sWhat's your name? ")
  (message "Hello, %s!" name))
```

</details>

---

## 7. 自测

1. `(interactive "sFoo: ")` 干啥?
2. `let` 和 `let*` 区别?
3. `if` 的 THEN 和 ELSE 形式数量有什么不同?
4. `nil` 和 `0` 和 `""` 哪个是 false?
5. `when` 和 `if` 区别?
6. `with-current-buffer` 干啥?
7. `save-excursion` 干啥?
8. `(interactive "r")` 提供什么参数?

**答案**:
> 1. 提示 "Foo: ",读字符串作为函数第一参数
> 2. let 并行绑定;let* 顺序绑定 (后面 var 可见前面的)
> 3. THEN 一个 form;ELSE 多个 (隐式 progn)
> 4. 只有 nil 是 false;0 和 "" 是 true
> 5. when 只有 then 分支,可多 form;if 有 then/else 二选一,then 只能一个 form
> 6. 临时切到指定 buffer,执行 body,切回原 buffer
> 7. 保存 point 和 mark,执行 body,恢复
> 8. 两个数字参数: region 起点 start,终点 end

---

## 8. 毕业检查

- [ ] 15 道 exercise 完成
- [ ] 能用 `defun` 写任意函数
- [ ] 能用 `let` / `let*` 区分
- [ ] 能用 `if` / `when` / `unless` 正确
- [ ] 能用 `interactive` 让函数变命令
- [ ] 写过 5 个实用的 buffer 函数

完成后进入 `concept-anchor.md` + `exercises.md`,然后 `../week-03-recursion/`。
