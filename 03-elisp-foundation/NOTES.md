# Module 3 Notes

这份 NOTES 是 Module 3 全部内容的**压缩参考**。读完 5 周后,你应该能合上书,凭这份 notes 复述所有概念。如果某个 pattern 你说不清楚,回去看对应 week 的 chassell-reading。

每条结论都有"为什么"。如果只记结论不记为什么,3 个月后你会忘。

## 第一性原理总结

### 1. Lisp 是 List Processing

所有代码、所有数据都是 list。代码即数据 (homoiconicity)。

为什么这是第一原理?因为 Lisp 1958 年的设计目标就是"用一种结构表示一切"。如果代码和数据用同样结构,你就可以用一个工具操作两者——所以 `read`、`eval`、`mapcar` 这些函数既能处理代码也能处理数据,代码变换 (宏) 自然诞生。

### 2. Eval 规则

- atom (number, string): eval 为自身
- symbol: eval 为 variable value
- list (a b c): a 是函数,eval 参数,调用

这三条规则**完备**——任何 Elisp 表达式的求值都能用这三条推。复杂表达式靠"递归应用"展开: `(+ 1 (* 2 3))` 先 eval `(* 2 3)` (list 规则) → 6,再 eval `(+ 1 6)` → 7。

记住: eval 是"输入 form,输出 value"的纯函数 (除了副作用如 message)。

### 3. Quote 阻止 eval

`'foo` = `(quote foo)` = foo (the symbol)

为什么需要 quote?因为 Lisp 默认会 eval 所有 symbol——查它的变量值。如果你想要 symbol 本身 (比如当 key 用),必须阻止 eval。`'` 就是这个"阻止"信号。

引申: `'(+ 1 2)` 给你 list 本身,不 eval。这是宏的基础——宏操作的是"未 eval 的 list"。

### 4. Cons Cell 是底层

list = 嵌套 cons cell

- car: 第一个值
- cdr: 剩下的 (是 list 或 nil)
- (a . b): dotted pair,不是 list

为什么 cons cell 这么基本?因为它就是"一对指针"。任何复杂数据都能用"指针对"构建——这是 1958 年 McCarthy 的洞察。list 是 cons cell 的特例 (cdr 永远是 nil 或另一个 cons)。

list 的内存模型 `(a b c)` 是 `(a . (b . (c . nil)))`。理解了这个,所有 list 操作 (car, cdr, append, reverse) 都是 cons cell 操作。

### 5. Lisp-2

- 变量和函数有独立命名空间
- `#'foo` = function slot
- `funcall` 用于调用函数值

为什么 Lisp-2 设计成两个 namespace?这有历史原因: 1960 年代函数是一等公民,Lisp 想允许 `(list list)` (函数 list 和变量 list 同名) 不冲突。代价是当函数存到变量后,要 `(funcall var)` 而不是 `(var)`。

Scheme 是 Lisp-1 (一个 namespace),`(list list)` 会冲突,所以 Scheme 程序员要避免这种命名。Clojure 也是 Lisp-1。Elisp 和 Common Lisp 是 Lisp-2。

### 6. Lexical vs Dynamic

- lexical: 定义时捕获 (默认 in Emacs 28+)
- dynamic: 调用时查找 (用 defvar 声明的变量)
- lexical 才有闭包

为什么有两个?dynamic 是 1960 年代 Lisp 的默认,简单但容易出 bug (函数依赖调用栈的变量)。lexical 是 Scheme 引入,后来 Common Lisp 和 Elisp 跟进。lexical 更"安全" (函数行为可预测) 且能做闭包。

但 dynamic 不是没用: 配置变量 (用户想临时覆盖) 用 dynamic 才合理。所以现代 Elisp: 默认 lexical,需要"可被 let 覆盖"的变量用 `defvar` (声明成 dynamic)。

### 7. 闭包 = 函数 + 环境

lexical binding 下,lambda 捕获外层变量。

闭包不是"魔法",是 lexical binding 的直接推论: 函数定义时,它看到的变量 binding 被打包进去。即使外层 let 退出,这些变量仍活 (因为闭包持有引用)。

闭包用途极广: 计数器、缓存 (memoization)、状态机、对象系统 (用 closure 模拟 OO)。Module 3 W3-W4 大量讲。

---

## 常用模式

下面 6 个 pattern 覆盖了 90% 的日常 Elisp 代码。每个 pattern 都"反复出现",所以记住它们就能快速读懂别人的代码。Module 3 各 week 会展开讲。

### Pattern 1: 递归遍历 list

```elisp
(defun my-process (list)
  (if (null list)
      nil                                  ; base case
    (cons (process-one (car list))         ; 处理首
          (my-process (cdr list)))))       ; 递归
```

这是 list 处理的"模板"。任何对 list 元素逐一处理 + 收集的函数,都可以套这个模板 (改 `process-one` 和 base case)。它体现了 Lisp 的"递归天然"——和 list 的递归定义同构。

但注意栈深度: list 长度 > 1000 时,改用 Pattern 2 (while)。

### Pattern 2: 用 while 收集

```elisp
(let (result)
  (while CONDITION
    (push ITEM result)
    ...)
  (nreverse result))
```

为什么 `push` + `nreverse`?因为 push 在 list 头加 O(1);如果用 `append` 在尾部加,每次 O(n),总 O(n²)。`nreverse` 是破坏性反转,O(n)。两者配合是 Elisp 高效收集 list 的标准模式。

`push` 加到头,但我们要的顺序是"先处理的在前"——所以最后 reverse。

### Pattern 3: save-excursion + re-search

```elisp
(save-excursion
  (goto-char (point-min))
  (while (re-search-forward PATTERN nil t)
    ;; 处理匹配
    ))
```

这是"在 buffer 里找所有匹配并处理"的标准模式。`save-excursion` 保证 point/mark 不被破坏 (函数退出后恢复)。`re-search-forward` 找下一个,找到后 point 在匹配末。

`nil t` 参数: nil = 无 bound (找到底),t = 找不到不报错 (返回 nil)。

### Pattern 4: dolist + 累加

```elisp
(let ((sum 0))
  (dolist (x list sum)
    (setq sum (+ sum x))))
```

dolist 比手动 while 更可读。注意 `sum` 在 dolist 第三参数——这是"返回值 form",dolist 退出时返回它。这是 Elisp 风格: 把返回值和循环绑在一起,不用额外一行 return。

### Pattern 5: with-current-buffer

```elisp
(with-current-buffer NAME
  ;; 临时切换,执行,切回
  )
```

为什么不用 `(set-buffer NAME) ... (set-buffer original)`?因为万一中间 error 了,buffer 切不回来,留下不一致状态。`with-current-buffer` 用 `unwind-protect` 保证切回——即使出错。

这是 Elisp "防御性编程"的常见模式: 任何改变 Emacs 状态的操作,都用 `with-*` 或 `save-*` 包起来。

### Pattern 6: 闭包状态封装

```elisp
(defun make-counter ()
  (let ((count 0))
    (lambda ()
      (setq count (1+ count))
      count)))
```

闭包把 `count` 封装——只有返回的 lambda 能改。全局命名空间不污染。这是"对象"的轻量替代: 不要 defclass、不要 method,只要一个 lambda 持有 state。

Module 3 W3 详细讲闭包,W4 在高阶函数里大量用。

---

## 命令清单 (你必须会的)

### List 操作
- `(car LIST)`、`(cdr LIST)`、`(cons X LIST)`
- `(list A B C)`、`(append L1 L2)`、`(reverse L)`
- `(length L)`、`(nth N L)`、`(member X L)`
- `(mapcar FN L)`、`(mapc FN L)`
- `(remove X L)`、`(sort L PRED)`

### 条件
- `(if COND THEN ELSE)`
- `(when COND BODY...)`、`(unless COND BODY...)`
- `(cond (TEST BODY...)...)`

### 循环
- `(while COND BODY...)`
- `(dolist (X LIST [RESULT]) BODY...)`
- `(dotimes (I N [RESULT]) BODY...)`

### 函数
- `(defun NAME (ARGS) DOC BODY...)`
- `(lambda (ARGS) BODY...)`
- `(funcall FUNCTION ARGS...)`
- `(apply FUNCTION ARGS-LIST)`

### 变量
- `(setq VAR VAL)`
- `(let ((VAR VAL)...) BODY...)`
- `(let* ((VAR VAL)...) BODY...)`
- `(defvar NAME VAL DOC)`
- `(defcustom NAME VAL DOC :type ...)`

### Buffer
- `(current-buffer)`、`(set-buffer X)`、`(with-current-buffer X BODY...)`
- `(point)`、`(mark)`、`(region-beginning)`、`(region-end)`
- `(insert ...)`、`(delete-region A B)`、`(buffer-substring A B)`
- `(save-excursion BODY...)`

### 字符串
- `(concat ...)`、`(substring STR N [M])`、`(length STR)`
- `(string= A B)`、`(string< A B)`
- `(upcase STR)`、`(downcase STR)`、`(capitalize STR)`
- `(split-string STR SEP)`、`(mapconcat FN LIST SEP)`
- `(string-match REGEX STR)`、`(replace-regexp-in-string ...)`

### 文件
- `(find-file PATH)`、`(save-buffer)`
- `(file-exists-p PATH)`、`(file-readable-p PATH)`
- `(with-temp-file PATH BODY...)`
- `(insert-file-contents PATH)`
- `(read BUFFER)`、`(pp OBJ BUFFER)`

---

## 易错点

这 8 个易错点几乎每个 Elisp 新手都踩过。读别人的代码看到这些,要条件反射地警惕。

### 1. 'foo 和 foo

```elisp
foo              ; ← 这是变量,eval 为值
'foo             ; ← 这是 symbol,不 eval
```

新手常忘 `'`,导致 void-variable error。

为什么这是陷阱?因为大多数语言没有"symbol 作为值"的概念——Python 写 `foo` 永远是变量。Lisp 不同: symbol 既是"变量名"也是"值本身"。`'` 决定你要哪个。

经验: 看到"void-variable foo"错误,99% 是忘了 quote。

### 2. (eq A B) vs (equal A B)

```elisp
(eq "abc" "abc")          ; → nil (可能不同对象)
(equal "abc" "abc")       ; → t (内容相同)
(eq 'foo 'foo)            ; → t (symbol 唯一)
(eq 5 5)                  ; → t (数字)
(eq (list 1 2) (list 1 2)) ; → nil (不同 cons)
(equal (list 1 2) (list 1 2)) ; → t
```

为什么两套?`eq` 是"指针相等" (像 Python 的 `is`),最快但严格。`equal` 是"内容相等" (像 Python 的 `==`),慢但符合直觉。

经验: 比较 symbol/number 用 `eq`,比较 string/list 用 `equal`。

### 3. nil 是 false,但 0 和 "" 是 true

```elisp
(if 0 'yes 'no)       ; → yes
(if "" 'yes 'no)      ; → yes
(if '() 'yes 'no)     ; → no ('() = nil)
```

为什么 Lisp 这么设计?历史原因 + 数学理由。Lisp 只有 nil 是 false,因为它"既是空 list 又是 false symbol"——这是 1958 年的简化设计。其他值都是 true,包括 0 和空字符串。

这和 Python (`if not []`)、C (`if (!0)`) 不同。新手容易踩坑: 写 `(if (length str) ...)` 以为"非空字符串返回 true",但 length 0 时也是 0 (true)。要用 `(if (string-empty-p str) ...)` 或 `(if (> (length str) 0) ...)`。

### 4. nth 越界

```elisp
(nth 10 '(a b c))     ; → nil (不报错)
```

为什么 silent return nil?Elisp 哲学: "宽容对待,返回 nil"。这有好处 (代码不频繁报错),有坏处 (bug 隐藏)。所以写 nth 时要心里有数,可能拿到 nil。

如果需要严格,用 `(aref ARRAY IDX)` (array 越界报错) 或自己加 check。

### 5. cons 不一定是 list

```elisp
(cons 1 2)            ; → (1 . 2) (dotted pair, 不是 list)
(cons 1 nil)          ; → (1) (list)
(cons 1 '(2 3))       ; → (1 2 3) (list)
```

为什么 cons 这么"乱"?因为 cons 就是"造一个 cons cell",不管 cdr 是不是 list。list 是 cons 的特例——cdr 是 nil 或另一个 cons。

所以 `cons` 既能造 list (cdr 是 list) 也能造 dotted pair (cdr 是其他)。要看上下文。

### 6. setq 和 let

```elisp
(setq x 5)            ; 全局设
(let ((x 5)) ...)     ; 局部绑定
```

混用要小心。

为什么这是陷阱?新手有时在函数里写 `(setq x 5)` 想造局部变量,结果污染全局。永远用 `let` 造局部变量,除非你**真的**想设全局 (那时考虑用 `defvar` 加文档)。

### 7. interactive 参数

```elisp
(interactive "sName: ")    ; string
(interactive "nAge: ")     ; number
(interactive "r")          ; region
(interactive "P")          ; prefix
(interactive "f")          ; file
(interactive "d")          ; point
```

为什么这么多 code?因为 Emacs 命令的参数来源五花八门 (字符串、数字、文件、region),interactive 用单字符 code 简化常见情况。复杂情况用 `(interactive (list ...))` 写任意逻辑。

经验: 单一参数用 code,多个参数或需要变换用 list。

### 8. file-local variables

文件第一行 `-*- lexical-binding: t -*-` 让该文件用 lexical。

为什么用 file-local?因为 Emacs 允许不同文件用不同 binding 模式 (向后兼容)。文件头声明清楚,byte-compiler 知道怎么编译。

这是 Elisp 的"工程化"考虑: 老代码可能仍 dynamic,新代码默认 lexical,文件头区分。

---

## 推荐阅读 (Module 3 之后)

- **Common Lisp the Language** (Steele) - 深入 Lisp 思维
- **ANSI Common Lisp** (Graham) - Graham 的书
- **SICP** (Abelson & Sussman) - 用 Scheme 教计算机科学
- **On Lisp** (Graham) - 高级 Lisp 技术 (宏)
- **Let Over Lambda** (Hoyte) - 极高级宏

Module 6 会重新精读 Emacs Lisp Reference,这次用更深的视角。
