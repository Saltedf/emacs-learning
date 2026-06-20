# Week 3: Conditionals + Recursion — Lisp 的灵魂

> **目标**: 掌握 cond、递归思维、while/loop、lexical binding
> **时长**: 1 周 (~20 小时)
> **Chassell 章节**: 5. A Few More Complex Functions, 6. Narrowing and Widening, 11. Loops and Recursion

---

## 0. 这周在学什么

上周你学了 if / when / unless。这周深入**条件表达式**和**循环**。

但 W3 真正的核心不是这些语法——是**思维方式**的转变。从"循环思维" (像 Python for loop) 转到"递归思维" (像数学归纳法)。这是 Lisp 区别于 C/Java 的根本之一。

完成这周,你能:
- 用 `cond` 写多分支
- 用 `while` / `dolist` / `dotimes` 写循环
- 写**优雅的递归**处理 list
- 理解**尾递归**和栈溢出
- 理解 lexical vs dynamic binding 的本质

最后一项 (lexical vs dynamic) 是 W3 最难的内容。它是 closure (闭包) 的基础。一旦懂了,你就懂了"函数 + 环境"——所有现代函数式编程的核心。

---

## 1. cond: 多分支 (Chassell)

`if` 只能 "二选一",但实际编程经常"多选一"——比如根据数字返回星期名 (7 个分支)。这时 `cond` 更合适。`cond` 是 Lisp 的 "switch case",但更强大。

### 1.1 cond 语法

```elisp
(cond
 (TEST1 BODY1...)
 (TEST2 BODY2...)
 ...
 (t BODY-DEFAULT))           ; 可选的 else
```

每个 clause 是一个 list:
- 第一个元素是 TEST
- 剩下是 BODY (可多个 form)
- TEST 为 non-nil → 执行 BODY,返回 BODY 最后一个 form 的值
- TEST 为 nil → 试下一个 clause

### 1.2 例子

```elisp
(defun classify (x)
  (cond
   ((numberp x) "number")
   ((stringp x) "string")
   ((listp x) "list")
   ((symbolp x) "symbol")
   (t "other"))))

(classify 5)             ;; → "number"
(classify "hi")          ;; → "string"
(classify 'foo)          ;; → "symbol"
(classify [1 2])         ;; → "other" (vector)
```

### 1.3 没有 t (else)

```elisp
(cond
 ((> x 0) "positive")
 ((< x 0) "negative"))
;; 如果 x = 0,返回 nil (没匹配)
```

### 1.4 没有 BODY 的 clause

```elisp
(cond
 (FOUND-IT)                  ; 如果 FOUND-IT 非 nil,返回 FOUND-IT 的值
 (t (search-it)))
```

如果 TEST 本身就是想要的值,可以省 BODY。

### 1.5 case (cl-case)

如果用 `cl-lib` (Common Lisp extensions):

```elisp
(require 'cl-lib)

(cl-case x
  (1 "one")
  (2 "two")
  (3 "three")
  (otherwise "many"))
```

不像 cond,cl-case **不 eval** TEST (它们是字面值)。

(Module 6 W3 详细讲 cl-lib)

---

## 2. while: 循环 (Chassell Loops)

`while` 是最基础的循环——条件为 true 就反复执行。虽然 Lisp 偏爱递归,但有些场景 while 更合适 (尤其需要 mutable state 累加时)。

### 2.1 while 语法

```elisp
(while CONDITION
  BODY...)
```

CONDITION 为 non-nil 时反复执行 BODY。

while 是 special form——CONDITION 不是函数参数,而是 conditional 求值。每次循环开始前 eval CONDITION,如果 nil 就退出。

### 2.2 例子

```elisp
(let ((i 0))
  (while (< i 5)
    (message "i = %d" i)
    (setq i (1+ i))))
;; 打印 i = 0, 1, 2, 3, 4
```

最简单的 while——一个 mutable counter,每次 +1,直到 5。

### 2.3 用 while 遍历 list

list 遍历用 while 的标准模式——"list 是不是 nil? 不是就处理 car,然后 cdr":

```elisp
(let ((lst '(a b c d))
      (result nil))
  (while lst
    (push (car lst) result)         ; 收集
    (setq lst (cdr lst)))
  (nreverse result))
;; → (a b c d)
```

(`push` 和 `nreverse` 是高效 list 构造模式)

这是 Lisp 风格的 "for loop"——while + car/cdr 模式。等价的递归版本更优雅,但 while 版本更适合深 list (不消耗栈)。

为什么 `push` + `nreverse`? 因为 push 在 list 头加 O(1);如果用 `append` 在尾部加,每次 O(n),总 O(n²)。`nreverse` 是破坏性反转,O(n)。两者配合是 Elisp 高效收集 list 的标准模式。

### 2.4 dolist

更高级的遍历:

```elisp
(dolist (elt '(a b c))
  (message "elt: %s" elt))
;; 打印 elt: a, elt: b, elt: c

(dolist (elt '(1 2 3 4) sum)
  (setq sum (+ sum elt)))
;; → 10 (返回 sum)
```

`dolist (VAR LIST [RESULT])`:
- VAR 每次绑定 LIST 的一个元素
- 可选 RESULT 是最后返回的 form

dolist 比 while 更可读——你不用手动管 cdr 和终止条件。优先用 dolist。

### 2.5 dotimes

```elisp
(dotimes (i 5)
  (message "i = %d" i))
;; 打印 i=0 到 i=4

(dotimes (i 5 result)
  (setq result (cons i result)))
;; → (4 3 2 1 0)
```

dotimes 是"固定次数循环"——i 从 0 到 N-1。适合"重复 N 次"的场景,不用写 counter。

### 2.6 cl-loop (高级,可选)

```elisp
(require 'cl-lib)

(cl-loop for x in '(1 2 3 4 5)
         when (> x 2)
         sum x)
;; → 12 (3+4+5)

(cl-loop for i from 1 to 10
         collect (* i i))
;; → (1 4 9 16 25 36 49 64 81 100)
```

`cl-loop` 是一个 mini-language,可以写复杂循环。
(Module 6 W3 详细学)

cl-loop 来自 Common Lisp,功能极强但语法独特。新手看 cl-loop 代码可能困惑 (它像英语不像 Lisp)。但熟练后,cl-loop 能用一行表达复杂逻辑。Module 6 详细讲。

---

## 3. Recursion: Lisp 的灵魂 (Chassell Loops and Recursion)

递归是 Lisp 的"灵魂操作"——不是因为它优雅,而是因为它**契合 list 的结构**。一旦你理解 list 是递归定义的 (空 list 或 car+cdr),你就理解为什么 Lisp 偏爱递归。

### 3.1 什么是递归?

**递归**: 函数调用自身。听起来神秘,其实就是"用同样的方法处理更小的问题":

```elisp
(defun my-length (list)
  (if (null list)
      0
    (1+ (my-length (cdr list)))))    ; 递归调用
```

读法:
- 如果 list 是空,长度 0 (base case)
- 否则,长度 = 1 + (剩下的长度) (recursive case)

这和数学归纳法一模一样: "基础情况是 0;一般情况 = 1 + 剩下的"。递归只是把"数学定义"直接翻译成代码。

### 3.2 递归三要素

每个递归函数必须有这三要素,缺一不可:

1. **Base case**: 终止条件 (不递归)
2. **Recursive case**: 调用自身,参数更接近 base case
3. **Combine**: 把 recursive 调用的结果组合

例子 `my-length`:
- Base: `(null list)` → 0
- Recursive: `(my-length (cdr list))` (cdr 更短)
- Combine: `(1+ result)`

为什么必须有 base case?否则递归永远不结束——无限循环,栈溢出。base case 是"逃出递归的出口"。

为什么 recursive case 参数要"更接近 base"?每次递归,参数要变化,朝 base case 走。如果参数不变,递归不终止。`my-length` 的参数从 `(cdr list)`——比 `list` 短,朝 `nil` 走。

Combine 是"如何把递归结果和当前情况结合"。my-length 的 combine 是 `(1+ result)`——把当前元素算 1,加上递归结果。

### 3.3 经典例子: factorial

阶乘是递归的经典例子:

```elisp
(defun factorial (n)
  (if (zerop n)
      1
    (* n (factorial (1- n)))))

(factorial 5)              ;; → 120
;; (= 5 * 4 * 3 * 2 * 1 * 1)
```

Base: n=0 → 1
Recursive: `(* n (factorial (1- n)))` —— n × (n-1)!

为什么 factorial 适合递归?因为它的数学定义就是递归的: `0! = 1`, `n! = n × (n-1)!`。递归代码直接对应数学。

### 3.4 经典例子: fibonacci

Fibonacci 数列: 0, 1, 1, 2, 3, 5, 8, 13, 21, ...

```elisp
(defun fib (n)
  (if (< n 2)
      n
    (+ (fib (- n 1))
       (fib (- n 2)))))

(fib 10)                   ;; → 55
```

数学定义: `F(0)=0, F(1)=1, F(n)=F(n-1)+F(n-2)`。直接翻译。

**警告**: 这个实现 O(2^n),慢。
用 memoization 或迭代版本 (Module 6)。

为什么慢?因为 `(fib 10)` 计算 `(fib 9)` 和 `(fib 8)`,`(fib 9)` 又计算 `(fib 8)` 和 `(fib 7)`——`(fib 8)` 被算两次。指数爆炸。

解决: 用闭包做 memoization (W4 讲),或用迭代。但作为教学例子,这个递归版本最清晰。

### 3.5 list 递归

Lisp 里最自然的递归是 list 处理——因为 list 本身递归定义:

```elisp
(defun my-sum (list)
  (if (null list)
      0
    (+ (car list) (my-sum (cdr list)))))

(my-sum '(1 2 3 4))         ;; → 10

(defun my-reverse (list)
  (if (null list)
      nil
    (append (my-reverse (cdr list))
            (list (car list)))))

(my-reverse '(1 2 3))       ;; → (3 2 1)
```

这两个例子是 Lisp 递归的"模板"。几乎所有 list 函数 (filter, map, reduce) 都套这个模板。

模板: `(if (null list) BASE (COMBINE (PROCESS (car list)) (RECURSE (cdr list))))`。

### 3.6 嵌套递归

处理嵌套 list (tree) 时,递归两层——一层处理 car (可能是子 tree),一层处理 cdr:

```elisp
(defun my-flatten (tree)
  (cond
   ((null tree) nil)
   ((atom tree) (list tree))
   (t (append (my-flatten (car tree))
              (my-flatten (cdr tree))))))

(my-flatten '(1 (2 3) (4 (5 6)) 7))
;; → (1 2 3 4 5 6 7)
```

`(my-flatten (car tree))` 处理"第一个元素" (可能是子 tree);`(my-flatten (cdr tree))` 处理"剩下"。append 把两边结果拼起来。

这种"双递归"模式适合所有树形结构: AST、嵌套 JSON、XML。

### 3.7 尾递归 (Tail Recursion)

普通递归和尾递归的区别是"递归调用后还要做事吗"。

普通递归:
```elisp
(defun factorial (n)
  (if (zerop n)
      1
    (* n (factorial (1- n)))))      ; 不是尾递归 (递归调用后还要 *)
```

注意 `(* n (factorial (1- n)))`——递归调用 `(factorial (1- n))` 后,还要乘 `n`。所以"递归调用不是最后操作"。

尾递归版 (用累加器):
```elisp
(defun factorial-tail (n)
  (defun helper (n acc)
    (if (zerop n)
        acc
      (helper (1- n) (* n acc))))
  (helper n 1))
```

注意 `helper` 的递归调用 `(helper (1- n) (* n acc))`——它就是 helper 函数体的最后一个 form,没有"再算什么"。

**特点**: 递归调用是最后操作,不需要"返回再算"。

**Elisp 不优化尾递归** (不像 Scheme),所以深递归仍可能栈溢出。
但尾递归版更**清晰**,推荐用。

为什么 Scheme 优化尾递归?因为 Scheme 标准 (R5RS) 要求实现 TCO (tail-call optimization)——尾递归不消耗栈。Elisp 没这要求,所以即使你写尾递归,栈照样增长。但尾递归版**语义更清晰** (没有"递归后做事"的复杂性),所以推荐。

### 3.8 栈溢出

递归的实际限制是栈深度:

```elisp
(my-length (make-list 100000 0))    ; 通常 OK

(my-length (make-list 1000000 0))   ; 可能 stack overflow
```

Elisp 默认栈深度约 10000-100000 (取决于版本和平台)。超过就 `Lisp nesting exceeds max-lisp-eval-depth`。

深度递归用 `while` 改写:

```elisp
(defun my-length-iter (list)
  (let ((count 0))
    (while list
      (setq count (1+ count)
            list (cdr list)))
    count))
```

经验: 处理"已知浅"的数据 (大多数 list 操作) 用递归,清晰优雅;处理"可能深"的数据 (大 list、嵌套深) 用 while,避免栈溢出。

对比 Python:
```python
def my_length(lst):
    if not lst: return 0
    return 1 + my_length(lst[1:])
```
Python 有问题: 默认递归深度 1000,长 list 会栈溢出。而且 `lst[1:]` 每次 O(n) 复制。

Lisp 写法:
```elisp
(defun my-length (lst)
  (if (null lst) 0
    (1+ (my-length (cdr lst)))))
```
Elisp 也有栈限制,但 `(cdr lst)` 是 O(1) (指针)。语言设计鼓励递归——这是 Lisp 文化。

---

## 4. Lexical vs Dynamic Binding (Chassell Lexical vs Dynamic)

这是 Module 3 最难也最重要的概念之一。理解它,你就理解了 closure、模块化、配置系统的全部基础。

### 4.1 历史

要理解 lexical vs dynamic 的区别,先看历史。

Emacs 24 之前默认 **dynamic binding**。这是 1985 年 Stallman 的选择——当时 dynamic 是主流,lexical 还没普及。
Emacs 24+ 支持 lexical,但默认还是 dynamic (要 file 局部开启)。
Emacs 28+ 默认 lexical (在 init.el 等)。

为什么 Emacs 走得这么慢?因为兼容——几十年积累的 elisp 代码可能依赖 dynamic。突然换默认,会破老代码。所以 Emacs 用 file-local 变量逐步迁移: 老代码继续 dynamic,新代码用 lexical。

### 4.2 Dynamic Binding

先看 dynamic binding 的行为:

```elisp
;; -*- lexical-binding: nil; -*-  ; dynamic

(defvar x 10)

(defun show-x ()
  x)

(show-x)                   ;; → 10

(let ((x 20))              ; 临时改 x
  (show-x))                ;; → 20! (show-x 看到新的 x)

(show-x)                   ;; → 10 (恢复了)
```

**关键**: `x` 在 `show-x` 内**不绑定**,它查**调用栈**找最近的 binding。

dynamic 的语义: 函数内部的自由变量 (像 `show-x` 里的 `x`) 不绑定到定义时的环境,而是查"调用时的动态环境"。所以 `(let ((x 20)) (show-x))` 里 show-x 看到 20。

这听起来 flexible 但危险——你 let 临时改 `x`,所有内部用 `x` 的函数都受影响。

### 4.3 Lexical Binding

对比 lexical:

```elisp
;; -*- lexical-binding: t; -*-  ; lexical

(defvar x 10)

(defun show-x ()
  x)

(show-x)                   ;; → 10

(let ((x 20))
  (show-x))                ;; → 10! (show-x 用自己定义时的 x)

(show-x)                   ;; → 10
```

**关键**: `show-x` 在**定义时**捕获 `x` 的 binding。
`let` 改 x 不影响 show-x (因为 let 是 lexical 局部)。

lexical 的语义: 函数捕获定义时的环境。`show-x` 定义时 `x` 是全局 10,所以 show-x 永远返回 10,不管调用时 let 改了什么。

### 4.4 区别

| | Dynamic | Lexical |
|---|---|---|
| 变量查找 | 调用栈 (运行时) | 词法作用域 (定义时) |
| let 影响 | 影响所有调用 | 只影响 let 内的代码 |
| 闭包 | 不能 (无捕获) | 能 |
| 性能 | 慢 (动态查找) | 快 (编译时确定) |
| 默认 | 旧 Emacs | Emacs 28+ |

### 4.5 为什么 lexical 更好?

三个理由:

**理由 1: 闭包**

```elisp
;; lexical
(defun make-counter ()
  (let ((count 0))
    (lambda ()
      (setq count (1+ count))      ; 捕获外层 count
      count)))

(setq c1 (make-counter))
(setq c2 (make-counter))
(funcall c1)               ;; → 1
(funcall c1)               ;; → 2
(funcall c2)               ;; → 1 (独立的 count)
(funcall c1)               ;; → 3
```

dynamic binding 下**做不到**。

为什么做不到?因为 dynamic 下,lambda 不捕获外层 count。每次调用 lambda,count 查动态环境——它要找全局或调用栈的 count。两个 counter 共享同一个 count——无法独立。

lexical 下,lambda 捕获定义时的 let 局部 count——每次 `make-counter` 调用,let 创建新的 count,lambda 捕获它。所以 c1 和 c2 各自的 count 独立。这是 closure 的核心。

**理由 2: 安全**

```elisp
;; dynamic: 危险
(defun my-func (lst)
  (mapcar (lambda (x) (* 2 x)) lst))

(let ((x 100))
  (my-func '(1 2 3)))      ; x 可能被 lambda "shadow" (在 dynamic 下)
```

lexical 下 lambda 的 x 是**参数**,不受外层影响。

dynamic 下,lambda 里的 `x` 是自由变量,可能被外层 let 影响。这是 "dynamic binding 的 footgun"——你不知道函数内部用了什么变量名,可能不小心 shadow。

**理由 3: 性能**

byte-compiler 可以优化 lexical 代码 (变量索引到栈位置)。
dynamic 必须运行时查找。

lexical 下,编译器看到 `(let ((x 5)) ...)`,知道 x 在栈上某个位置——直接访问。dynamic 下,x 是 symbol,每次访问要查 symbol 的 value slot——慢。

### 4.6 何时用 dynamic?

dynamic 不是没用。少数场景:

- **Hooks**: 你想 hook 替换一个变量
- **buffer-local**: 一些变量希望按 buffer 不同
- **配置变量**: 用户用 let 临时改

例如:

```elisp
(defvar my-temp-var 5)

(defun my-func-uses-temp ()
  (message "temp is %d" my-temp-var))

(let ((my-temp-var 100))   ; dynamic: my-func-uses-temp 看到 100
  (my-func-uses-temp))
```

dynamic 让"临时覆盖配置"变得简单——用户 `(let ((inhibit-message t)) ...)` 就能临时关 message。这是 Elisp 的实用特性。

Module 6 W2 详细讲。

### 4.7 实践建议

**永远用 lexical** (file 头 `-*- lexical-binding: t; -*-`)。

例外: 故意想"动态覆盖"某变量时,用 `defvar` (但不让 lexical 捕获)。

```elisp
;; -*- lexical-binding: t; -*-

(defvar my-debug nil)     ; defvar 是动态特殊变量

(defun my-log (msg)
  (when my-debug
    (message "[LOG] %s" msg)))

(let ((my-debug t))       ; 临时打开 debug
  (my-log "testing"))     ; → "[LOG] testing"
```

惯例: **用 `defvar`/`defcustom` 定义的变量是动态** (即使 lexical binding 模式)。
**用 let 绑定的是 lexical** (除非变量被 defvar 过)。

这个惯例叫 "semantically dynamic special variables"——只有 defvar 过的变量是 dynamic,其他都 lexical。这样既有 lexical 的安全,又允许"配置变量"动态覆盖。

---

## 5. 闭包 (Closure) 深入

闭包是 lexical binding 的"杀手锏"——它让"函数 + 状态"成为可能,从而让函数式编程能做到 OO (面向对象) 能做的事。

### 5.1 闭包是什么

**闭包** = 函数 + 它定义时的环境。

这个定义抽象,看具体:

```elisp
;; lexical
(let ((counter 0))
  (defun inc-counter ()
    (setq counter (1+ counter))))
```

`inc-counter` "捕获" 了 `counter`。
即使 let 退出了,`counter` 仍然活 (因为闭包持有它)。

```elisp
(inc-counter)              ;; → 1
(inc-counter)              ;; → 2
(inc-counter)              ;; → 3
```

`counter` 现在是 "private" 变量,只有 `inc-counter` 能改。

这里有个"反直觉"的事: `(let ((counter 0)) ...)` 退出后,理论上 `counter` 应该消失。但 `inc-counter` 定义在这个 let 里,它"捕获"了 counter 的 binding。所以即使 let 退出,counter 还活着——闭包让它"长生不老"。

这听起来神秘,实现上是: lambda 对象内部持有 counter 的引用。Garbage collector 看到 "有引用,不回收",所以 counter 活着。

### 5.2 工厂函数

闭包最常见的模式是"工厂函数"——返回一个定制的 lambda:

```elisp
(defun make-adder (n)
  (lambda (x)
    (+ x n)))

(setq add5 (make-adder 5))
(setq add10 (make-adder 10))

(funcall add5 3)           ;; → 8
(funcall add10 3)          ;; → 13
```

`make-adder` 返回的 lambda 捕获了 `n`。

`make-adder` 调用两次: `(make-adder 5)` 和 `(make-adder 10)`。每次调用,let 创建新的 `n`,返回的 lambda 捕获各自的 n。所以 add5 的 n=5,add10 的 n=10,独立。

这是"部分应用" (partial application) 的例子——把 `+` 部分应用为 "+5" 或 "+10"。函数式编程的基础模式。

### 5.3 状态封装

闭包可以做"有状态的对象":

```elisp
(defun make-account (initial-balance)
  (let ((balance initial-balance))
    (lambda (action amount)
      (pcase action
        ('deposit (setq balance (+ balance amount)) balance)
        ('withdraw (setq balance (- balance amount)) balance)
        ('balance balance)))))

(setq acc (make-account 100))
(funcall acc 'deposit 50)    ;; → 150
(funcall acc 'withdraw 30)   ;; → 120
(funcall acc 'balance)       ;; → 120
```

`balance` 是 private,只能通过 lambda 操作。

这模拟了"银行账户"对象——有状态 (balance)、有方法 (deposit, withdraw, balance)。但没用 defclass,只用闭包。这就是 "lambda is the ultimate object" (Scheme 文化)。

### 5.4 闭包 vs 全局变量

闭包让"状态"封装,不污染全局命名空间。
比 OO (面向对象) 更轻量。

OO (defclass) 适合大型、复杂状态;闭包适合小型、特定状态。Emacs Lisp 大量用闭包而不是 class——因为大多数状态都是"小型 + 特定"。

闭包的另一个优势是"组合"——你可以 `(my-compose (make-adder 5) (make-multiplier 2))` 把两个工厂组合。class 组合 (继承、mixin) 复杂得多。

---

## 6. Narrowing and Widening (Chassell Ch 6)

Narrowing 是 Emacs 独特的功能——"聚焦"到 buffer 的一部分,其他暂时隐藏。这不是删除,是"视图限制"。Chassell 用一整章讲它,因为 narrowing 是 Emacs 文化的特色。

### 6.1 Narrowing

`narrow-to-region` 把 buffer "缩小"到 region:

```elisp
(narrow-to-region START END)
```

之后:
- `point-min` 返回 START
- `point-max` 返回 END
- 你看不到/不能编辑 buffer 外

实际 buffer 内容没变,只是"视图"缩了。

Narrowing 的本质: 改变 `point-min` 和 `point-max`,让所有"buffer 范围"操作只在 region 内有效。底层 buffer 数据不变,只是访问限制。

### 6.2 Widening

```elisp
(widen)                    ;; 恢复完整 buffer
```

`widen` 取消 narrowing,回到完整 buffer。

### 6.3 save-restriction

```elisp
(save-restriction
  (narrow-to-region ...)
  ...)
;; 退出时恢复 narrowing 状态
```

`save-restriction` 是"防御性 narrowing"——保证退出时恢复。函数内 narrow,函数外无影响。这是 Elisp 风格: 任何改变全局状态的操作,都用 save-* 包起来。

### 6.4 用途

- **聚焦**: 只看一段,其他暂时隐藏。比如写论文,只 narrow 到当前章节,不被其他内容干扰。
- **安全**: 操作只影响 region。比如 replace-regexp 只在 narrow 区域内。
- **效率**: 搜索等只在 region 内。大 buffer 上跑慢操作前 narrow。

### 6.5 默认禁用

新手意外 narrow 会困惑 ("我的内容呢?")。
Emacs 默认 `(put 'narrow-to-region 'disabled nil)`。
加到 init.el 启用:

```elisp
(put 'narrow-to-region 'disabled nil)
```

为什么默认禁用?因为新手不知道 narrowing,突然看到 buffer "少了一大半"会恐慌。Emacs 让你"明确启用"——加这一行,表示"我知道 narrowing 是什么"。

快捷键:
- `C-x n n` narrow-to-region
- `C-x n w` widen
- `C-x n p` narrow-to-page
- `C-x n d` narrow-to-defun

`C-x n d` (narrow-to-defun) 特别有用——只看当前函数,不被其他函数干扰。编程时极便利。

---

## 7. 实战练习

### Ex 3.1: my-factorial
```elisp
(defun my-factorial (n)
  ;; 递归
  )

(my-factorial 5)           ;; → 120
```

<details><summary>答案</summary>

```elisp
(defun my-factorial (n)
  (if (zerop n)
      1
    (* n (my-factorial (1- n)))))
```

</details>

### Ex 3.2: my-fibonacci
```elisp
(defun my-fib (n)
  ;; 递归
  )

(my-fib 10)                ;; → 55
```

<details><summary>答案</summary>

```elisp
(defun my-fib (n)
  (if (< n 2)
      n
    (+ (my-fib (1- n))
       (my-fib (- n 2)))))
```

</details>

### Ex 3.3: my-copy-list
```elisp
(defun my-copy-list (list)
  ;; 递归
  )
```

<details><summary>答案</summary>

```elisp
(defun my-copy-list (list)
  (if (null list)
      nil
    (cons (car list) (my-copy-list (cdr list)))))
```

</details>

### Ex 3.4: my-remove-if
```elisp
(defun my-remove-if (pred list)
  ;; 移除 PRED 返回 non-nil 的元素
  )

(my-remove-if #'numberp '(1 a 2 b 3))    ;; → (a b)
```

<details><summary>答案</summary>

```elisp
(defun my-remove-if (pred list)
  (if (null list)
      nil
    (if (funcall pred (car list))
        (my-remove-if pred (cdr list))
      (cons (car list) (my-remove-if pred (cdr list))))))
```

</details>

### Ex 3.5: my-mapcar (递归版)
```elisp
(defun my-mapcar (fn list)
  ;; 递归
  )

(my-mapcar #'1+ '(1 2 3))    ;; → (2 3 4)
```

<details><summary>答案</summary>

```elisp
(defun my-mapcar (fn list)
  (if (null list)
      nil
    (cons (funcall fn (car list))
          (my-mapcar fn (cdr list)))))
```

</details>

### Ex 3.6: my-filter
```elisp
(defun my-filter (pred list)
  ;; 保留 PRED 返回 non-nil 的
  )

(my-filter #'numberp '(1 a 2 b))    ;; → (1 2)
```

<details><summary>答案</summary>

```elisp
(defun my-filter (pred list)
  (if (null list)
      nil
    (if (funcall pred (car list))
        (cons (car list) (my-filter pred (cdr list)))
      (my-filter pred (cdr list)))))
```

</details>

### Ex 3.7: my-reduce
```elisp
(defun my-reduce (fn list)
  ;; (my-reduce #'+ '(1 2 3 4)) → 10
  )

(my-reduce #'+ '(1 2 3 4))    ;; → 10
(my-reduce #'* '(1 2 3 4))    ;; → 24
```

<details><summary>答案</summary>

```elisp
(defun my-reduce (fn list)
  (if (null (cdr list))
      (car list)
    (funcall fn (car list) (my-reduce fn (cdr list)))))
```

</details>

### Ex 3.8: my-find-if
```elisp
(defun my-find-if (pred list)
  ;; 返回第一个 PRED 非 nil 的元素
  )

(my-find-if #'numberp '(a b 3 c))    ;; → 3
```

<details><summary>答案</summary>

```elisp
(defun my-find-if (pred list)
  (cond
   ((null list) nil)
   ((funcall pred (car list)) (car list))
   (t (my-find-if pred (cdr list)))))
```

</details>

### Ex 3.9: counter (闭包)
```elisp
(defun make-counter ()
  ;; 返回一个 lambda,每次调用返回递增值
  )

(setq c (make-counter))
(funcall c)    ;; → 1
(funcall c)    ;; → 2
(funcall c)    ;; → 3
```

<details><summary>答案</summary>

```elisp
(defvar counter-state 0)    ; 这不能用 (污染全局)

;; lexical 版:
(defun make-counter ()
  (let ((count 0))
    (lambda ()
      (setq count (1+ count))
      count)))
```

注意: 要 `lexical-binding: t` 才能这样写。

</details>

### Ex 3.10: my-merge
合并两个已排序 list:

```elisp
(defun my-merge (l1 l2)
  ;; (my-merge '(1 3 5) '(2 4 6)) → (1 2 3 4 5 6)
  )

(my-merge '(1 3 5) '(2 4 6))    ;; → (1 2 3 4 5 6)
```

<details><summary>答案</summary>

```elisp
(defun my-merge (l1 l2)
  (cond
   ((null l1) l2)
   ((null l2) l1)
   ((< (car l1) (car l2))
    (cons (car l1) (my-merge (cdr l1) l2)))
   (t
    (cons (car l2) (my-merge l1 (cdr l2))))))
```

</details>

### Ex 3.11: my-quick-sort
快速排序:

```elisp
(defun my-qsort (list)
  ;; (my-qsort '(3 1 4 1 5 9 2 6)) → (1 1 2 3 4 5 6 9)
  )

(my-qsort '(3 1 4 1 5 9 2 6))
```

<details><summary>答案</summary>

```elisp
(defun my-qsort (list)
  (if (or (null list) (null (cdr list)))
      list
    (let* ((pivot (car list))
           (rest (cdr list))
           (less nil)
           (more nil))
      (dolist (x rest)
        (if (< x pivot)
            (push x less)
          (push x more)))
      (append (my-qsort (nreverse less))
              (list pivot)
              (my-qsort (nreverse more))))))
```

</details>

### Ex 3.12: my-tree-count
```elisp
(defun my-tree-count (tree)
  ;; 数嵌套 list 里的 atom 数
  )

(my-tree-count '(1 (2 3) ((4) 5)))    ;; → 5
```

<details><summary>答案</summary>

```elisp
(defun my-tree-count (tree)
  (cond
   ((null tree) 0)
   ((atom tree) 1)
   (t (+ (my-tree-count (car tree))
         (my-tree-count (cdr tree))))))
```

</details>

---

## 8. 自测

1. `cond` 和 `if` 区别?
2. 递归三要素?
3. 什么是尾递归?
4. lexical 和 dynamic binding 区别?
5. 闭包是什么?
6. `(setq lexical-binding t)` 怎么用?
7. narrow 是什么?
8. `dotimes` 和 `while` 区别?

**答案**:
> 1. cond 多分支,每 clause TEST+BODY;if 二选一
> 2. base case, recursive case (参数更小), combine
> 3. 递归调用是最后操作,理论上不需要"返回再算"
> 4. lexical 在定义时捕获变量,dynamic 在调用时找
> 5. 函数 + 定义时环境;lexical binding 才有
> 6. file 局部变量 (file-local var): 文件头 `-*- lexical-binding: t; -*-` 或 `(setq-local lexical-binding t)`
> 7. 把 buffer 限制到 region,看不到外面
> 8. dotimes 固定次数;while 条件控制

---

## 9. 毕业检查

- [ ] 12 道 exercise 完成
- [ ] 能用递归写任意 list/tree 操作
- [ ] 能用闭包做状态封装
- [ ] 理解 lexical vs dynamic

完成后进入 `concept-anchor.md` + `exercises.md`,然后 `../week-04-lambda-mapcar/`。
