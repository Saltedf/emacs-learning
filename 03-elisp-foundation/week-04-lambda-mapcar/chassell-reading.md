# Week 4: Lambda + mapcar + Sequences — 高阶函数

> **目标**: 掌握 lambda、mapcar、funcall、apply、字符串和序列
> **时长**: 1 周 (~20 小时)
> **Chassell 章节**: 8. Cutting and Storing Text, 10. Yanking Text Back, 12. Regular Expression Searches, 13. Counting via Repetition and Regexps

---

## 0. 这周在学什么

上周你学了递归、循环、闭包。这周你学**函数作为值**——Lisp 最强大的特性之一。

"函数作为值"是什么意思?Python 也有 lambda 和高阶函数,但 Lisp 把这做到极致——一切皆函数,函数可以被传、被返回、被存到 list 里、被组合。这让 Lisp 的"函数式编程"风格极其灵活。

完成这周,你能:
- 写匿名函数 (lambda)
- 用 mapcar/mapc 遍历
- 用 funcall/apply 调用函数值
- 操作字符串
- 写正则搜索 + 替换

最后两项让你能"批量处理文本"——日常编辑自动化的核心。

## 1. Lambda: 匿名函数

`lambda` 是 Lisp 表达"匿名函数"的方式。它源自 Alonzo Church 1930 年代的 Lambda Calculus——数学上研究函数的理论。Church 用希腊字母 λ (lambda) 表示"函数抽象",McCarthy 借鉴到 Lisp。今天 Python、JS、Java 都有 lambda——它们的祖先都是 Lisp。

### 1.1 为什么需要匿名函数?

你可能问: "我已经有 defun,为什么还要 lambda?" 想想这些场景:

- `(mapcar (lambda (x) (* x 2)) list)`: 你只想"对每个元素乘 2",你不在乎这个"乘 2 函数"叫什么名字。给它命名 (`(defun double (x) (* 2 x))`) 反而是噪音。
- `(sort list (lambda (a b) (< (length a) (length b))))`: 你只需要这个比较函数一次,何必命名?
- 闭包: `(lambda () (setq count (1+ count)))` 捕获外层 count,这是 defun 做不到的 (defun 创建全局函数,不捕获 let)。

`lambda` 让你"即用即弃",像数学里的 λ 演算——你写 λx.x²,不需要给"平方"命名。

### 1.2 lambda 的语法

`lambda` 创建一个**没有名字的函数**:

```elisp
(lambda (x)
  (* x x))
;; → 一个函数,接受 x,返回 x*x
```

语法和 defun 几乎一样: `(lambda (ARGLIST) BODY...)`。区别是没有名字、没有 docstring (虽然可以加)。

`lambda` 是 special form (不是宏)。

### 1.3 调用 lambda

直接用 `funcall`:

```elisp
(funcall (lambda (x) (* x x)) 5)
;; → 25
```

为什么不能直接 `((lambda (x) (* x x)) 5)`? 因为 Elisp 是 Lisp-2——(W1 学过)函数和变量分开命名空间。`((lambda ...))` 这种"list 第一个元素是函数值"的语法,Lisp-2 不接受 (它要求第一个元素是 symbol,从 symbol 的 function slot 取函数)。

所以你拿到函数值 (lambda 对象),要用 `funcall` 调用——funcall 直接执行函数值,不查 symbol。

或用 `apply`:

```elisp
(apply (lambda (x y) (+ x y)) '(3 4))
;; → 7
```

apply 和 funcall 区别: apply 最后参数是 list,展开。

### 1.4 lambda 作为值

lambda 的核心特性: 它是一个**值**。可以存到变量、传给函数、从函数返回:

```elisp
(setq square (lambda (x) (* x x)))

(funcall square 5)              ;; → 25
```

注意: `square` 是变量,值是 lambda。
调用要 `funcall` (因为 Lisp-2)。

这就是"函数作为一等公民"——函数可以像数字、字符串一样被传递。

### 1.5 #'-syntax

`#'foo` = `(function foo)` = 函数 slot of foo。

```elisp
(symbol-function 'car)          ;; → #<subr car>
(function car)                  ;; → #<subr car>
#'car                           ;; → #<subr car>
```

`#'` 用于传递函数符号:

```elisp
(mapcar #'1+ '(1 2 3))          ;; → (2 3 4)
(mapcar (lambda (x) (* x x)) '(1 2 3))  ;; → (1 4 9)
```

`#'1+` 和 `(lambda (x) (1+ x))` 等价。

为什么需要 `#'`? 因为 `1+` 既是 symbol,又可能是变量。`(mapcar '1+ ...)` 也行 (symbol 作为函数查找),但 `#'1+` 更明确——它是 `(function 1+)`,专门取 function slot。byte-compiler 也更喜欢 `#'` (它能做更好的优化)。

### 1.6 lambda 在 let 里

lambda 可以捕获外层 let 的变量 (lexical binding 下):

```elisp
(let ((multiplier 10))
  (setq scale (lambda (x) (* x multiplier))))

(funcall scale 5)               ;; → 50 (捕获 multiplier)
```

(lexical binding 下)

这就是闭包!W3 详细讲过。lambda 在 let 内定义,捕获 multiplier binding。即使 let 退出,scale 持有 multiplier。

### 1.7 创造性用法

lambda 的几个"意外"用法,展示它的强大:

1. **匿名回调**: `(mapcar (lambda (x) (* x 2)) list)` 不需要命名函数
2. **闭包状态**: `(let ((counter 0)) (defun next! () (cl-incf counter)))` 函数持有私有状态
3. **柯里化**: `(lambda (y) (+ 5 y))` 等于把 + 柯里化成 "+5"
4. **函数作为数据**: list 里可以放函数 `(+ 1 2 3 #'square)` 调用
5. **Y combinator**: 用 lambda 表达递归,不需要 defun (学术兴趣)
6. **CPS (continuation-passing)**: 异步编程的祖先——函数接受 "下一步做什么" 作为参数
7. **宏的基石**: macro 展开成 lambda

第 6 点 (CPS) 特别有意思: 现代 async/await 本质是 CPS 的语法糖。Lisp 在 1970 年代就玩过了。

这些用法 Module 6 详细讲。Module 3 你先掌握 1-3。

---

## 2. funcall vs apply

`funcall` 和 `apply` 都是"调用函数值"的工具。它们的区别是参数传递方式——这个区别新手常混淆,但理解后很简单。

### 2.1 funcall

`(funcall FUNCTION ARGS...)`:
- FUNCTION 是函数值 (或 symbol)
- ARGS 是直接参数

```elisp
(funcall #'+ 1 2 3)             ;; → 6
(funcall '+ 1 2 3)              ;; → 6 (symbol 也行)
(funcall (lambda (x) (* x 2)) 5)  ;; → 10
```

`funcall` 像"普通函数调用",只是第一个参数是函数值。后面参数一个一个传。

### 2.2 apply

`(apply FUNCTION ARGS-LIST)`:
- 最后一个参数是 list,展开为单独参数

```elisp
(apply #'+ '(1 2 3))            ;; → 6 (等同 + 1 2 3)
(apply #'+ 1 2 '(3 4))          ;; → 10 (1, 2 直接,展开 (3 4))
```

`apply` 允许"前面的参数直接,最后一个 list 展开"。这适合"我有一个 list,想把它作为参数列表"的场景。

### 2.3 区别

| | funcall | apply |
|---|---|---|
| 参数形式 | 直接 | 最后是 list,展开 |
| 适用 | 调用时知道参数 | 参数在 list 里 |

例子: 你有一个 list,想 apply `+`:

```elisp
(let ((nums '(1 2 3 4 5)))
  (apply #'+ nums))             ;; → 15
```

用 funcall:

```elisp
(let ((nums '(1 2 3 4 5)))
  (apply #'+ nums))             ;; 必须 apply
;; funcall 不行:
;; (funcall #'+ nums)           → error (加 list 和 nil)
```

为什么需要两套?因为"参数"有两种来源:

- 你写代码时知道有几个参数 → `funcall`
- 参数是动态 list (运行时长度不定) → `apply`

两个工具互补。

### 2.4 函数作为参数

funcall 让你能写"高阶函数"——函数接受函数作为参数:

```elisp
(defun my-apply-twice (fn x)
  "Apply FN to X, then FN to result."
  (funcall fn (funcall fn x)))

(my-apply-twice #'1+ 5)         ;; → 7
(my-apply-twice (lambda (s) (concat s s)) "ab")  ;; → "ababab"
```

这是**高阶函数**: 接受函数作为参数。

高阶函数是函数式编程的核心——它让你能"组合"行为。`mapcar`、`filter`、`reduce` 都是高阶函数。Lisp 的设计让高阶函数自然——因为函数是值,接受它作参数和接受数字作参数没本质区别。

---

## 3. mapcar / mapc

`mapcar` 是 Lisp 最常用的高阶函数——"对每个元素应用函数,收集结果"。Python 的 `map`、JS 的 `Array.map` 都源自它。掌握 mapcar,你就掌握了"批量处理 list"的核心。

### 3.1 mapcar

`(mapcar FN LIST)`:
- 对 LIST 每个元素调用 FN
- 收集结果,返回新 list

```elisp
(mapcar #'1+ '(1 2 3))          ;; → (2 3 4)
(mapcar (lambda (s) (length s)) '("a" "bb" "ccc"))  ;; → (1 2 3)
(mapcar #'upcase '("foo" "bar"))  ;; → ("FOO" "BAR")
```

mapcar 是"循环 + 收集"的语法糖。等价的递归写法:

```elisp
(defun my-mapcar (fn list)
  (if (null list)
      nil
    (cons (funcall fn (car list))
          (my-mapcar fn (cdr list)))))
```

但 mapcar 比递归版快——它内建、用循环、避免栈消耗。所以日常用 mapcar,不用自己递归。

mapcar 的语义: 输入一个 list,输出一个 list (元素 1 对 1 映射)。它不改变原 list (函数式风格,无副作用)。

### 3.2 mapc

`(mapc FN LIST)`:
- 类似 mapcar,但**不收集结果**
- 返回原 LIST
- 用于副作用 (打印、修改)

```elisp
(mapc (lambda (x) (message "%s" x)) '(a b c))
;; 打印 a, b, c
;; 返回 (a b c)
```

比 mapcar 高效 (不分配新 list)。

什么时候用 mapc? 当你不在乎返回值,只想"做点事" (副作用) 时。比如打印、修改 buffer、累加 (通过外部变量)。

mapc 的返回值是原 LIST——这看起来"无用",但允许 chain: `(mapc ... (mapc ... list))`。

### 3.3 多 list 的 mapcar

mapcar 可以同时处理多个 list:

```elisp
(mapcar #'+ '(1 2 3) '(10 20 30))
;; → (11 22 33)

(mapcar (lambda (a b) (list a b))
        '(a b c) '(1 2 3))
;; → ((a 1) (b 2) (c 3))
```

长度按最短的。

这适合"配对处理两个 list"——比如 zip 操作。Python 写 `zip(l1, l2)` 然后 map,Lisp 一步到位。

### 3.4 其他 map 类

`mapconcat` 是字符串版的 mapcar——收集结果拼成字符串:

```elisp
(mapconcat #'identity '("a" "b" "c") "-")
;; → "a-b-c"

(mapconcat (lambda (n) (number-to-string n)) '(1 2 3) "/")
;; → "1/2/3"
```

第三个参数是分隔符。mapconcat 等于 `(mapcar FN LIST)` 后用 sep 拼接——但内建,更快。

还有 `maphash` (hash table 遍历)、`mapatoms` (obarray 遍历) 等。Module 6 详细讲。

---

## 4. 字符串操作

字符串是 Emacs 处理文本的基础——你编辑的每个文件都是字符串。Lisp 的字符串操作和 Python/JS 类似,但有些 quirks 你要记住。

### 4.1 基础

```elisp
(length "hello")                ;; → 5
(stringp "hello")               ;; → t
(substring "hello world" 0 5)   ;; → "hello"
(substring "hello" 1)           ;; → "ello"
(substring-no-properties "hello" 0 3)  ;; → "hel" (无 text props)
(concat "hello" " " "world")    ;; → "hello world"
(format "%s has %d chars" "hi" 2)  ;; → "hi has 2 chars"
```

Elisp 字符串是**不可变**的——substring、concat 都返回新字符串,不改原串。这和 Python 一致,和 C (mutable) 不同。

`format` 用 `%s` (string)、`%d` (integer)、`%S` (sexp,带 quote)、`%f` (float) 占位符。和 C 的 printf 类似。

### 4.2 比较

字符串比较有 `=`、`<`、`>`——但 Elisp 是 Lisp-2,这些算符既用于数字也用于字符串。所以字符串比较用 `string=`、`string<`:

```elisp
(string= "hello" "hello")       ;; → t
(string-equal "hello" "hello")  ;; → t (alias)
(string< "abc" "abd")           ;; → t
(string-lessp "abc" "abd")      ;; → t (alias)
(string> "abd" "abc")           ;; → t
```

为什么不用 `=`?因为 `=` 是数字比较,字符串用会报错。`string=` 是字符串专版。

### 4.3 大小写

```elisp
(upcase "hello")                ;; → "HELLO"
(downcase "HELLO")              ;; → "hello"
(capitalize "hello world")      ;; → "Hello World"
(upcase-initials "hello world") ;; → "Hello World"
```

`capitalize` 把每个 word 首字母大写,其他小写。`upcase-initials` 只首字母大写,其他不变。区别在 "hEllo" → capitalize 是 "Hello",upcase-initials 是 "HEllo"。

### 4.4 查找

```elisp
(string-match "world" "hello world")    ;; → 6 (匹配位置)
(string-match "xyz" "hello")            ;; → nil
(replace-regexp-in-string "o" "0" "hello")  ;; → "hell0"
(replace-match "REPLACEMENT")           ;; 替换最近匹配
```

`string-match` 用正则,返回匹配开始位置 (0-based),没匹配返回 nil。`replace-regexp-in-string` 替换所有匹配。

### 4.5 分割和合并

```elisp
(split-string "a,b,c" ",")              ;; → ("a" "b" "c")
(split-string "  hi  there  " "[ \t]+" t)  ;; → ("hi" "there")
(mapconcat #'identity '("a" "b" "c") ", ")  ;; → "a, b, c"
```

`split-string` 用正则分割,返回 list。第三个参数 `t` 表示"omit empty"——不返回空字符串 (默认会返回)。

### 4.6 转换

```elisp
(string-to-number "42")         ;; → 42
(number-to-string 42)           ;; → "42"
(string-to-char "abc")          ;; → 97 (第一个字符)
(char-to-string ?a)             ;; → "a"
(string-to-list "abc")          ;; → (97 98 99) (字符 list)
```

注意 `string-to-list` 返回**字符 code 的 list** (97, 98, 99),不是字符对象。Elisp 字符就是 integer (ASCII/Unicode code point)。

### 4.7 trim / format

```elisp
(string-trim "  hello  ")       ;; → "hello" (Emacs 25+)
(string-trim-left "  hi")       ;; → "hi"
(string-trim-right "hi  ")      ;; → "hi"
(string-pad "hi" 5)             ;; → "hi   " (Emacs 28+)
```

`string-trim` 是 Emacs 25+ 的便利函数。在更老 Emacs 上,要 `(replace-regexp-in-string "\\`[ \t\n]+\\|[ \t\n]+\\'" "" s)`。

---

## 5. 正则表达式 (Chassell Ch 12)

正则表达式 (regex) 是文本处理的"瑞士军刀"——找模式、提取信息、批量替换都靠它。Module 1 你学了 user 视角 (C-s、M-%),这周学 Lisp 视角——用代码控制正则。

### 5.1 Emacs 正则回顾

(Module 1 学过 user 视角)

```elisp
(string-match "[0-9]+" "abc123def")  ;; → 3 (匹配位置)
(string-match "[0-9]+" "abc")        ;; → nil
(match-string 0 "abc123def")         ;; → "123" (取匹配)
```

`string-match` 在字符串里找模式。返回匹配位置 (integer),没找到返回 nil。`match-string` 取最近一次匹配的字符串。

Elisp 正则语法和其他语言类似,但有几个 quirks:

- `\\(` 和 `\\)` 是 group (不是 `\(` `\)`)
- `\\|` 是 alternation (不是 `\|`)
- `\\{2,\\}` 是 repetition (不是 `{2,}`)

为什么双反斜杠?因为 Elisp 字符串里 `\\` 是一个反斜杠,正则需要一个反斜杠,所以写两个。Module 6 详细讲。

### 5.2 在 buffer 里搜

```elisp
(re-search-forward "[0-9]+")         ;; 找下一个,移动 point
(re-search-backward "[0-9]+")
(re-search-forward "[0-9]+" nil t)   ;; nil bound, t no-error
```

`re-search-forward` 是 buffer 操作的"正则版"——找模式,移 point 到匹配末。配合 while 循环,可以遍历所有匹配。

### 5.3 替换

```elisp
(replace-match "NUM")                ;; 替换最近匹配为 "NUM"
(perform-replace "OLD" "NEW" t nil nil)  ; query-replace 内部
(replace-regexp-in-string "OLD" "NEW" STRING)
```

`replace-match` 替换**最近一次** search 的匹配。`replace-regexp-in-string` 在字符串里替换所有匹配——更常用。

### 5.4 re-search 模式

经典模式 (你 Module 3 W3 学过):

```elisp
(save-excursion
  (goto-char (point-min))
  (while (re-search-forward PATTERN nil t)
    ;; 处理匹配 (point 在匹配末)
    ))
```

这是 Elisp 最常用的模式之一——"遍历 buffer 所有匹配"。`save-excursion` 保证 point 恢复,`goto-char point-min` 起始,`re-search-forward` 找下一个,while 循环。

### 5.5 rx 宏 (可读正则)

字符串正则难读——`"\\([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]\\{2,\\}\\)"` 是什么? 邮箱正则,但你看不出来。

Emacs 提供 `rx` 宏,用 S-表达式写正则:

```elisp
(require 'rx)

(rx (one-or-more digit) "." (one-or-more digit))
;; → "[0-9]+\\.[0-9]+"

(rx bos (? "-") (+ digit) eos)
;; → `\`\\(?-\\)?[0-9]+\'`

(rx-to-string '(seq bos (or "yes" "no") eos))
;; → "\\`\\(?:yes\\|no\\)\\'"
```

`rx` 用 s-expression 写正则,**可读性飞起**。

(Module 6 W7 详细讲 rx)

rx 的优势: 你可以用 Lisp 的组合方式构建正则——`(rx (or (seq "foo" eos) (seq "bar" bos)))` 比等效字符串清晰 10 倍。生产代码推荐用 rx。

### 5.6 实战: 提取所有数字

把正则 + while + 收集 list 组合起来:

```elisp
(defun my-extract-numbers (string)
  "Return list of all numbers in STRING."
  (let ((result nil)
        (start 0))
    (while (string-match "[0-9]+" string start)
      (push (string-to-number (match-string 0 string)) result)
      (setq start (match-end 0)))
    (nreverse result)))

(my-extract-numbers "abc 123 def 45 ghi 6789")
;; → (123 45 6789)
```

这是"提取信息"的典型模式——while 循环 search,每次 push 到 result,最后 nreverse。

`start` 变量是关键——每次 search 从上次的 `match-end` 开始,避免无限循环。新手常忘这个,陷入死循环。

---

## 6. 实战练习

### Ex 4.1: my-square-list
用 mapcar 翻倍 list:

```elisp
(defun my-square-list (list)
  ;; mapcar
  )

(my-square-list '(1 2 3 4))    ;; → (1 4 9 16)
```

<details><summary>答案</summary>

```elisp
(defun my-square-list (list)
  (mapcar (lambda (x) (* x x)) list))
```

</details>

### Ex 4.2: my-apply-sum
```elisp
(defun my-sum-numbers (list)
  ;; 用 apply
  )

(my-sum-numbers '(1 2 3 4 5))    ;; → 15
```

<details><summary>答案</summary>

```elisp
(defun my-sum-numbers (list)
  (apply #'+ list))
```

</details>

### Ex 4.3: my-compose
组合两个函数:

```elisp
(defun my-compose (f g)
  ;; 返回 (lambda (x) = (f (g x)))
  )

(setq add1-square (my-compose #'1+ (lambda (x) (* x x))))
(funcall add1-square 3)    ;; → 10 (1+ (* 3 3))
```

<details><summary>答案</summary>

```elisp
(defun my-compose (f g)
  (lambda (x)
    (funcall f (funcall g x))))
```

</details>

### Ex 4.4: my-repeat-string
```elisp
(defun my-repeat-string (n s)
  ;; "abc" 3 次 → "abcabcabc"
  ;; 用 lambda + mapconcat
  )

(my-repeat-string 3 "abc")    ;; → "abcabcabc"
```

<details><summary>答案</summary>

```elisp
(defun my-repeat-string (n s)
  (mapconcat (lambda (_) s) (make-list n nil) ""))
```

</details>

### Ex 4.5: my-string-join
```elisp
(defun my-string-join (sep list)
  ;; (my-string-join ", " '("a" "b" "c")) → "a, b, c"
  )

(my-string-join ", " '("a" "b" "c"))
```

<details><summary>答案</summary>

```elisp
(defun my-string-join (sep list)
  (mapconcat #'identity list sep))
```

</details>

### Ex 4.6: my-capitalize-words
```elisp
(defun my-capitalize-words (s)
  ;; 每个单词首字母大写
  )

(my-capitalize-words "hello world foo")    ;; → "Hello World Foo"
```

<details><summary>答案</summary>

```elisp
(defun my-capitalize-words (s)
  (mapconcat #'capitalize (split-string s) " "))
```

</details>

### Ex 4.7: my-extract-emails
```elisp
(defun my-extract-emails (string)
  ;; 返回 list of emails in STRING
  )

(my-extract-emails "Contact alice@foo.com or bob@bar.com")
;; → ("alice@foo.com" "bob@bar.com")
```

<details><summary>答案</summary>

```elisp
(defun my-extract-emails (string)
  (let ((result nil)
        (start 0))
    (while (string-match "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]" string start)
      (push (match-string 0 string) result)
      (setq start (match-end 0)))
    (nreverse result)))
```

</details>

### Ex 4.8: my-apply-to-region
```elisp
(defun my-apply-to-region (fn start end)
  ;; 取 region 文本,应用 FN,替换
  )
```

<details><summary>答案</summary>

```elisp
(defun my-apply-to-region (fn start end)
  (let* ((text (buffer-substring start end))
         (new (funcall fn text)))
    (delete-region start end)
    (goto-char start)
    (insert new)))
```

</details>

### Ex 4.9: my-pipe
把多个函数 pipe:

```elisp
(defun my-pipe (&rest fns)
  ;; (my-pipe #'1+ #'1+) 返回 (lambda (x) (1+ (1+ x)))
  )

(setq f (my-pipe #'1+ (lambda (x) (* x 2)) #'1-))
(funcall f 5)    ;; → ((5+1)*2)-1 = 11
```

<details><summary>答案</summary>

```elisp
(defun my-pipe (&rest fns)
  (lambda (x)
    (dolist (fn fns x)
      (setq x (funcall fn x)))))
```

</details>

### Ex 4.10: my-group-by
```elisp
(defun my-group-by (fn list)
  ;; 按返回值分组
  ;; (my-group-by #'evenp '(1 2 3 4 5)) → ((nil 1 3 5) (t 2 4))
  )

(my-group-by #'evenp '(1 2 3 4 5))
```

<details><summary>答案</summary>

```elisp
(defun my-group-by (fn list)
  (let ((groups (make-hash-table)))
    (dolist (x list)
      (let ((key (funcall fn x)))
        (push x (gethash key groups))))
    ;; 返回 alist
    (let (result)
      (maphash (lambda (k v) (push (cons k (nreverse v)) result)) groups)
      result)))
```

</details>

### Ex 4.11: my-trim-and-capitalize
```elisp
(defun my-trim-and-capitalize (s)
  ;; 用 my-compose 组合 trim 和 capitalize
  )
```

<details><summary>答案</summary>

```elisp
(defun my-trim-and-capitalize (s)
  (funcall (my-compose #'capitalize #'string-trim) s))
```

</details>

### Ex 4.12: my-count-vowels
```elisp
(defun my-count-vowels (s)
  ;; 用 mapcar + 累加
  )

(my-count-vowels "hello world")    ;; → 3
```

<details><summary>答案</summary>

```elisp
(defun my-count-vowels (s)
  (let ((count 0))
    (mapc (lambda (c)
            (when (memq c '(?a ?e ?i ?o ?u ?A ?E ?I ?O ?U))
              (setq count (1+ count))))
          s)
    count))
```

</details>

### Ex 4.13: my-interleave
```elisp
(defun my-interleave (l1 l2)
  ;; (my-interleave '(a b c) '(1 2 3)) → (a 1 b 2 c 3)
  )
```

<details><summary>答案</summary>

```elisp
(defun my-interleave (l1 l2)
  (let (result)
    (while (and l1 l2)
      (push (car l1) result)
      (push (car l2) result)
      (setq l1 (cdr l1)
            l2 (cdr l2)))
    (nreverse result)))
```

</details>

### Ex 4.14: my-zip-with
```elisp
(defun my-zip-with (fn l1 l2)
  ;; (my-zip-with #'+ '(1 2 3) '(10 20 30)) → (11 22 33)
  )

(my-zip-with #'+ '(1 2 3) '(10 20 30))    ;; → (11 22 33)
```

<details><summary>答案</summary>

```elisp
(defun my-zip-with (fn l1 l2)
  (mapcar fn l1 l2))
```

(mapcar 支持多 list)

</details>

### Ex 4.15: my-sort-by-length
```elisp
(defun my-sort-by-length (list)
  ;; 按字符串长度排序
  )

(my-sort-by-length '("aaa" "b" "cc"))    ;; → ("b" "cc" "aaa")
```

<details><summary>答案</summary>

```elisp
(defun my-sort-by-length (list)
  (sort list (lambda (a b) (< (length a) (length b)))))
```

</details>

---

## 7. 自测

1. `lambda` 和 `defun` 区别?
2. `funcall` 和 `apply` 区别?
3. `#'foo` 等于什么?
4. `(mapcar #'1+ '(1 2 3))` → ?
5. `(apply #'+ '(1 2 3 4))` → ?
6. 写 `(my-compose F G)` 组合两函数
7. 怎么 trim 字符串?
8. 怎么提取所有数字?

**答案**:
> 1. lambda 无名,defun 命名;lambda 是表达式,defun 是 statement
> 2. funcall 直接参数;apply 最后是 list 展开为参数
> 3. (function foo) = 函数 slot of foo
> 4. (2 3 4)
> 5. 10
> 6. `(defun my-compose (f g) (lambda (x) (funcall f (funcall g x))))`
> 7. `(string-trim s)` 或 `(replace-regexp-in-string "\\`[ \t\n]+\\|[ \t\n]+\\'" "" s)`
> 8. while + string-match

---

## 8. 毕业检查

- [ ] 15 道 exercise 完成
- [ ] 能用 lambda、mapcar、funcall、apply 熟练
- [ ] 能用正则提取信息
- [ ] 能组合函数 (compose, pipe)

完成后进入 `concept-anchor.md` + `exercises.md`,然后 `../week-05-capstone/`。
