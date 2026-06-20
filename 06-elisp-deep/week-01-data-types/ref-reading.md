# Week 1: Data Types

> **Ref 章节**: objects, numbers, strings, lists, sequences, records, hash, streams
> **目标**: 理解 Emacs Lisp 所有数据类型的底层

---

## 0. 这周在学什么

Module 3 你学了 list、string、number 的**操作**——怎么 cons、怎么 substring、怎么 +。
这周深入**底层实现**: 这些数据类型在 Emacs 内部是什么对象? 内存怎么分配? 怎么打印 (`print`)? 怎么读 (`read`)?

为什么必须深入? 因为操作层面是"知道怎么做",但底层是"知道为什么这么做"。

例子: 你在 Module 3 学过 `(eq "foo" "foo")` 可能返回 `nil`——这看起来是 bug。但如果你理解底层: 字符串是 mutable 对象,两个不同字面量在堆上是两个独立 cons,`eq` 比较指针,所以不同。这是"行为"层面的理解。如果你只学操作,你会被这个 bug 困扰——你不知道为什么。

另一个例子: `(length "中文")` 返回 2,`(string-bytes "中文")` 返回 6。如果你不深入"char vs byte"的区别,你会困惑: 字符串到底多长?

这周读完,你**永久**摆脱这些困惑。所有 Emacs 数据类型对你透明——你看到任何代码,都能在脑子里画出它的内存图。这是从"用户"到"作者"的跃迁。

---

## 1. Lisp Data Types (objects.texi)

### 1.1 类型大全

Elisp 有几十种内置类型——但可以归为几大类: scalar (number, string)、collection (list, vector, hash)、first-class (symbol, function)、editor-specific (buffer, window, frame, marker, process)。

为什么这么多种? Emacs 30 多年历史,每代维护者加新类型。1985 只有基本的;1990s 加 buffer/window/marker;2000s 加 process/network;2010s 加 thread/mutex (Emacs 26);最近加 record (Emacs 27)、native comp artifacts。每种类型解决一个真实问题。

每个类型有对应的 predicate (`XXXp`),用 `type-of` 可以查任意对象的类型。

```elisp
(integerp 5)           ; → t
(floatp 5.0)           ; → t
(stringp "hi")         ; → t
(symbolp 'foo)         ; → t
(consp '(1 2))         ; → t
(arrayp [1 2])         ; → t
(vectorp [1 2])        ; → t
(char-table-p)         ; 特殊
(bool-vector-p)        ; 特殊
(bufferp (current-buffer))
(markerp (point-marker))
(windowp (selected-window))
(framep (selected-frame))
(processp)
(threadp)
(mutexp)
(condition-variable-p)
(hash-table-p (make-hash-table))
(subrp (symbol-function 'car))   ; C 内置
(byte-code-function-p)
(recordp)
```

`integerp`、`consp` 等都是 C 实现的 predicate (`subrp` 检查)。它们的实现极简: 检查对象的 type tag (Lisp 对象有 low bits 表示类型)。所以 predicate 是 O(1)。

### 1.2 type-of

如果你不知道某对象是什么类型,用 `type-of`。它返回一个 symbol,描述对象的"主类型"。

```elisp
(type-of 5)            ; → integer
(type-of "hi")         ; → string
(type-of 'foo)         ; → symbol
(type-of '(1 2))       ; → cons
(type-of [1 2])        ; → vector
(type-of (current-buffer)) ; → buffer
```

注意 `type-of` 是"粗"分类——`(type-of "hi")` 返回 `string`,但 string 内部还有 unibyte/multibyte 区分。需要细查用 predicate 组合。

### 1.3 类型层次

Elisp 的类型不是严格继承 (像 OOP),而是"互斥类别"。但有些抽象层:

```
integer / float
string
symbol
cons (list)
array (vector / string / char-table / bool-vector)
record (类似 CL struct)
buffer
marker
window
frame
process
thread
hash-table
```

注意 `array` 是抽象——`string`、`vector`、`char-table`、`bool-vector` 都是 array,可以用 `aref`/`aset`。这种"抽象接口"让 sequence 操作 (`length`、`elt`、`mapcar`) 能统一处理 list 和 array——这就是为什么 Module 3 你能用 `mapcar` 作用于 list 也能作用于 string。

record 是 Emacs 27+ 引入的新类型,类似 Common Lisp 的 struct——一个带 type tag 的 vector,可以模拟对象系统 (`cl-defstruct` 底层用 record)。这让 Elisp 能写"OOP-like" 代码,虽然不是真 OOP。

---

## 2. Numbers (numbers.texi)

### 2.1 Integer

Emacs 在 64-bit 平台用 61-bit fixnum (2^61 范围)。这是怎么回事? 不是 64-bit 吗? 因为 Emacs Lisp 对象用 tagged pointer——低 3 bits 是 type tag,所以只有 61 bits 给 integer。这限制你最大整数约 2.3 × 10^18,实际足够。

```elisp
most-positive-fixnum          ; → 2^61 - 1 (64-bit)
most-negative-fixnum          ; → -2^61

(integerp 5)
(integerp 5.0)                ; → nil (float)

(1+ 5)                        ; → 6
(1- 5)                        ; → 4
(abs -5)                      ; → 5
(mod 10 3)                    ; → 1
(% 10 3)                      ; → 1
(/ 10 3)                      ; → 3 (integer)
(/ 10 3.0)                    ; → 3.33...

(max 1 2 3)                   ; → 3
(min 1 2 3)                   ; → 1
```

注意 `mod` vs `%`: `(% -1 3)` 在 Elisp 返回 `-1` (C-like 余数,符号跟随被除数),`(mod -1 3)` 返回 `2` (数学模,符号跟随除数)。这是数学家和程序员都容易搞错的——你写日期计算时用 `mod`,写位操作时用 `%`。

`(/ 10 3)` 整除返回 `3`,但 `(/ 10 3.0)` 返回 `3.333`。这是 type 推断——只要有一个 float,结果就是 float。如果你想强制浮点,加 `0.0`。

### 2.2 Float

Float 是 IEEE 754 double (64-bit)。和 C 的 double 一样,精度约 15-17 位有效数字。Float 用于时间 (`float-time` 返回 epoch 秒,带小数部分)、概率、统计。

```elisp
(floatp 3.14)
(floatp 3)                    ; → nil
(float-time)                  ; 当前时间的秒数
(floatp (float-time))         ; → t

(floor 3.7)                   ; → 3
(ceiling 3.2)                 ; → 4
(round 3.5)                   ; → 4
(truncate 3.7)                ; → 3
```

`floor`/`ceiling`/`round`/`truncate` 是 float → integer 转换,但策略不同。`round` 用"四舍五入",但 Elisp 的 `round` 对 `.5` 是"向偶数舍入" (banker's rounding)——`(round 2.5)` 返回 `2`,`(round 3.5)` 返回 `4`。这是为了减少统计偏差,但容易让人困惑。

### 2.3 转换

```elisp
(number-to-string 42)         ; → "42"
(string-to-number "42")       ; → 42
(string-to-number "abc")      ; → 0
```

`string-to-number` 解析失败返回 `0`,**不报错**。这是设计选择——很多场景下"失败的转换"是合理输入 (如空字符串)。但你要小心: 用户输入 `"abc"` 你以为是数字,你得到 `0` 而不知道。安全做法: `(if (string-match-p "\\`[0-9]+\\'" s) (string-to-number s) ...)`。

### 2.4 Random

```elisp
(random)                      ; 0 to 2^31
(random 100)                  ; 0 to 99
(random t)                    ; seed
```

`(random)` 返回伪随机数。Emacs 默认用 C 的 `rand()`,seed 是启动时的当前时间。如果你要密码学安全随机,用 `secure-hash` 或外部工具——`(random)` 不安全。

`(random t)` 重新 seed (用当前时间)。一般不需要手动 seed。

### 2.5 Bitwise

```elisp
(logand 5 3)                  ; → 1 (and)
(logior 5 3)                  ; → 7 (or)
(logxor 5 3)                  ; → 6 (xor)
(lognot 5)                    ; → -6
(ash 5 1)                     ; → 10 (左移)
(ash 5 -1)                    ; → 2 (右移)
```

位操作是优化常用工具。Elisp 的 `logand`/`logior`/`logxor` 对应 C 的 `&`/`|`/`^`。`ash` (arithmetic shift) 是移位——左移等于乘 2,右移等于除 2 (向下取整)。

实战: 用 bit 表示状态。例如 face 的属性 flags、keymap 的修饰键 (Ctrl=2, Meta=4, Shift=1) 都用 bit 表示。`(logior ?a ?\C-a)` 计算 Ctrl+a 的内部值。

---

## 3. Strings (strings.texi)

### 3.1 字符串是 mutable

Elisp 字符串**可变**——这点和 Python、JS 不同 (它们字符串 immutable)。为什么? 因为 buffer 是 mutable,而 buffer 内部和 string 共享数据结构。如果 string immutable,每次 substring 都要 copy,编辑大文件会极慢。

```elisp
(setq s "hello")
(aset s 0 ?H)                 ; 改第一个字符
s                             ; → "Hello"
```

`aset` 改变字符串内某位置的字符。**注意**: 改字面量字符串 (`"hello"` 在源码里的 literal) 是未定义行为——可能崩溃、可能改了其他地方也用到的 literal。安全做法: 改之前用 `copy-sequence` 复制。Emacs byte-compiler 会警告 "altering literal string"。

### 3.2 多行字符串

```elisp
(setq s "line1
line2
line3")
```

这是字面量多行——直接换行。但风格上**不推荐**,因为:
1. 看代码时不知道哪行是哪行
2. 编辑器可能自动缩进破坏

或用 `concat`:

```elisp
(setq s (concat "line1\n"
                "line2\n"
                "line3"))
```

`concat` 把多个 string 连接成新 string。这里每行后跟 `\n` (换行符)。可读性更好,且你可以一行一行加注释。

### 3.3 字符 vs string

Elisp 区分 **character** 和 **string**:
- character 是 integer (Unicode codepoint),用 `?X` 写
- string 是 character 的 array

这是设计选择——字符作为独立类型让 `char-syntax`、`upcase`、`encode-char` 等 API 简洁。Python 没有 character 类型,只有 1-char string;Java 有 `char`;Elisp 选了"char 是 number"。

```elisp
?A                            ; → 65 (字符)
(char-to-string ?A)           ; → "A"
(string-to-char "abc")        ; → 97 (第一个字符)
(aref "abc" 0)                ; → 97
(aset "abc" 0 ?A)             ; → "Abc"
```

`?A` 是 character literal,值是 65 (Unicode codepoint)。`?中` 是 20013。这个语法来自 Maclisp (1966),`?` 后跟字符,转义用 `?\n`、`?\t` 等。

### 3.4 比较

```elisp
(string= "abc" "abc")         ; → t
(string-equal "abc" "abc")    ; → t (alias)
(string< "abc" "abd")         ; → t
(string-lessp "abc" "abd")    ; → t
(string> "abd" "abc")         ; → t
(string-greaterp)             ; alias
(string-version-lessp "a1" "a10")  ; → t (version compare)
(string-collate-lessp)        ; locale-aware
```

`string=` 是 equality,`string<` 是字典序。**不要用 `eq` 比较字符串**——`(eq "foo" "foo")` 可能 `nil`,因为两个字面量在堆上是不同对象。`equal` 会逐字节比较,安全但慢 (O(n))。

`string-version-lessp` 是 Emacs 26+ 的"版本比较"——`"a1"` < `"a10"` < `"a2"` (自然排序),而普通 `string<` 会 `"a1"` < `"a10"` 但 `"a10"` < `"a2"` (字典序,数字按字符比较)。这在文件名排序时极有用——dired 用它排 v1、v2、v10 这种。

### 3.5 查找

```elisp
(string-match "world" "hello world")    ; → 6
(string-match-p "world" "hello world")  ; → 6 (no match data)
(string-search-forward)                  ; (在 buffer)
(search-forward "world")                 ; buffer 版
```

`string-match` 用正则,返回匹配起始位置或 `nil`。注意它**会修改 match-data** (全局状态),所以如果你的代码用了 `match-data`,后续 `string-match` 会破坏。要么用 `string-match-p` (不修改),要么 `save-match-data` 包起来。

`search-forward` 是 buffer 版本——它移动 point 并设 match-data。

### 3.6 替换

```elisp
(replace-regexp-in-string "o" "0" "foo")    ; → "f00"
(replace-regexp-in-string "[0-9]+" "N" "a1b2")    ; → "aNbN"
(replace-match)                                ; 替换最后匹配
(substitute ?o ?0 "foo")                       ; → "f00"
```

`replace-regexp-in-string` 是字符串替换的瑞士军刀。REP 参数可以是 string (直接替换) 或函数 (动态计算替换)——后者极强: `(replace-regexp-in-string "[0-9]+" (lambda (m) (number-to-string (* 2 (string-to-number m)))) "a1b2")` 把每个数字翻倍。

### 3.7 大小写

```elisp
(upcase "hello")              ; → "HELLO"
(downcase "HELLO")            ; → "hello"
(capitalize "hello world")    ; → "Hello World"
(upcase-initials "hello world") ; → "Hello World"
```

这些是 Unicode-aware——`(upcase "é")` 返回 `"É"` (大写 é)。但土耳其语有特殊 i (`İ`/`ı`),Elisp 默认不处理这种 locale-specific 行为。如果你做土耳其语应用,要特别小心。

### 3.8 分割合并

```elisp
(split-string "a,b,c" ",")    ; → ("a" "b" "c")
(split-string "  hi  there  " "[ \t]+" t)  ; → ("hi" "there")
(mapconcat #'identity '("a" "b" "c") "-")  ; → "a-b-c"
```

`split-string` 第三个参数 `t` 表示"omit nulls"——连续分隔符产生空 string 时跳过。否则 `"a,,b"` 会得到 `("a" "" "b")`。

`mapconcat` 是 split 的逆操作——join list 成 string,中间加分隔符。

### 3.9 Trim

```elisp
(string-trim "  hi  ")        ; → "hi" (Emacs 25+)
(string-trim-left "  hi")     ; → "hi"
(string-trim-right "hi  ")    ; → "hi"
(string-trim-both "  hi  ")   ; → "hi"
```

这些来自 `subr-x` (Emacs 25+)——以前要用 `(replace-regexp-in-string "\\`[ \t\n]+" "" ...)`。这些是 conveniences。

### 3.10 substring

```elisp
(substring "hello" 0 3)       ; → "hel"
(substring "hello" 1)         ; → "ello"
(substring "hello" -2)        ; → "lo" (倒数)
(substring-no-properties "hello" 0 3)  ; → "hel" (无 text props)
```

负数索引从末尾算——`-1` 是最后一个字符,`-2` 是倒数第二个。`substring-no-properties` 用于你不想要 text property 的场景 (如持久化到文件)。

### 3.11 Format

`format` 是 Elisp 的 sprintf——用 `%` 占位符把值转成 string。

```elisp
(format "%d + %d = %d" 1 2 3)   ; → "1 + 2 = 3"
(format "%s is %d" "Alice" 30)  ; → "Alice is 30"
(format "%.2f" 3.14159)        ; → "3.14"
(format "%-10s|" "hi")         ; → "hi        |"
(format "%05d" 42)             ; → "00042"
(format "%x" 255)              ; → "ff" (hex)
(format "%o" 8)                ; → "10" (octal)
(format "%b" 10)               ; → "1010" (binary)
(format "%c" 65)               ; → "A" (char)
(format-seconds "%h:%m:%s" 3661)  ; → "1:1:1"
```

`%s` (string)、`%d` (integer)、`%f` (float)、`%S` (Lisp-readable,加 quote)、`%c` (char)、`%x`/`%o`/`%b` (hex/octal/binary)。`%-10s` 左对齐宽 10,`%05d` 0 填充宽 5。这些和 C printf 一致。

`format-seconds` 是 Emacs 特有——把秒数转成 "1h:2m:3s" 这种人类可读。

---

## 4. Lists (lists.texi)

(Module 3 已深入)

### 4.1 主要操作

list 是 Lisp 的核心数据结构——cons cell 的链。`'(1 2 3)` 实际是 `(cons 1 (cons 2 (cons 3 nil)))`。理解这点,所有 list 操作都自然。

```elisp
(cons 1 '(2 3))
(car '(1 2 3))
(cdr '(1 2 3))
(nth 2 '(a b c))
(nthcdr 2 '(a b c d))
(last '(a b c))
(butlast '(a b c))
(append '(1 2) '(3 4))
(reverse '(1 2 3))
(length '(a b c))
(member 'b '(a b c))
(memq 'b '(a b c))
(assq 'a '((a 1) (b 2)))
(assoc 'a '((a 1) (b 2)) #'equal)
(remq 'b '(a b c b))
(remove 'b '(a b c b))
(delete 'b '(a b c b))
(mapcar #'1+ '(1 2 3))
(mapc #'print '(a b c))
(dolist (x '(a b c)) (print x))
```

每个函数有性能特性: `car`/`cdr` O(1),`nth`/`length` O(n),`append` O(n+m),`member` O(n)。理解这些让你写高效代码——比如累积 list 用 `(push x lst)` (O(1)) + 最后 `(nreverse lst)`,而不是 `(append lst (list x))` (O(n) 每次,总 O(n²))。

### 4.2 破坏性 vs 非破坏性

```elisp
(setq l (list 1 2 3))
(nreverse l)                  ; 破坏性 (返回新 list,但修改原)
(reverse l)                   ; 非破坏性
```

`nreverse` 是 "destructive reverse"——它**修改原 list 的 cons cells**,反转指针方向。比 `reverse` 快 (不需要分配新 cons)。但**危险**: 如果你别处持有原 list 引用,反转后那些引用看到的也是反的。

新手**用非破坏性**,清楚不易错。优化时再考虑破坏性——并且只在确认没有其他引用时用。

Elisp 中 `n` 前缀 (`nconc`、`nreverse`、`ndestructive-...`) 都是破坏性版本。Scheme 用 `!` 后缀 (如 `set!`),命名习惯不同。

### 4.3 Sort

```elisp
(sort '(3 1 4 1 5) #'<)
;; → (1 1 3 4 5)

(sort '("abc" "ab" "abcd") (lambda (a b) (< (length a) (length b))))
;; → ("ab" "abc" "abcd")
```

注意 `sort` 是**破坏性** (改原 list)。它使用 merge sort——稳定排序。如果你不想改原 list,先 `(copy-sequence lst)` 再 sort。

### 4.4 alist (association list)

alist 是 list of cons——`(cons key value)` 的链。常用于"小字典"。

```elisp
(setq al '((a 1) (b 2) (c 3)))
(assoc 'a al)                 ; → (a 1)
(assq 'a al)                  ; → (a 1) (eq 比)
(alist-get 'a al)             ; → 1
(push (cons 'd 4) al)
(setf (alist-get 'd al) 5)    ; 改值
```

`assq` 用 `eq` 比 key——只对 symbol 安全 (string key 不行,因为 string eq 不可靠)。`assoc` 默认用 `equal`,可以 string。

`alist-get` 是 Emacs 25+ 的便捷函数,返回 value 而非整个 cons。`setf` 配合可以"upsert"——存在就改,不存在就加。

### 4.5 plist (property list)

plist 是交替 key-value 的 flat list: `(key1 val1 key2 val2 ...)`。

```elisp
(setq pl '(:name "Alice" :age 30))
(plist-get pl :name)          ; → "Alice"
(plist-put pl :name "Bob")    ; 改值
(plist-member pl :name)       ; → t
```

plist vs alist: plist 更紧凑 (无 cons),但查找 O(n) 必须从头扫;alist 也是 O(n) 但 `assq` 实现略快 (C 内置)。symbol 的 property list (`(symbol-plist 'foo)`) 就是 plist,所以 `put`/`get` 用 plist API。

历史: Maclisp (1960) 用 plist 作为主要数据结构,Common Lisp 后改 alist。Elisp 两者都有,symbol 内部用 plist,通用数据用 alist 或 hash。

---

## 5. Sequences / Arrays (sequences.texi)

### 5.1 Sequence 是抽象

Emacs 设计了一个抽象: "sequence"——可以是 list、vector、string、bool-vector。所有 sequence 都有 `length`、`elt`、`reverse` 等通用操作。这是 Emacs 的"polymorphism"——通过 predicate + 函数重载实现 (而不是 OOP 继承)。

```elisp
(length '(1 2 3))             ; → 3
(length "abc")                ; → 3
(length [1 2 3])              ; → 3

(elt '(a b c) 1)              ; → b
(elt "abc" 1)                 ; → 98 (b 的 codepoint)
(elt [1 2 3] 1)               ; → 2

(reverse '(1 2 3))
(reverse "abc")               ; → "cba"
(reverse [1 2 3])             ; → [3 2 1]

(copy-sequence "abc")         ; → "abc"
(copy-sequence '(1 2 3))
(copy-sequence [1 2 3])
```

`elt` 是 sequence 的 indexing——内部根据 type 调 `nth` (list)、`aref` (array/string)。`copy-sequence` 类似——浅 copy,元素共享。

### 5.2 Array

vector 是 array 的最常见形式——定长、mutable、O(1) 索引。比 list 高效 (无 cons 开销,内存连续),但不能动态增长。

```elisp
(vectorp [1 2 3])
(make-vector 5 nil)           ; → [nil nil nil nil nil]
(vconcat [1 2] [3 4])         ; → [1 2 3 4]
(append [1 2] nil)            ; → (1 2) (vector → list)
```

`make-vector` 创建定长 vector,所有元素初始值相同。`vconcat` 连接多个 vector。`append` 把 vector 转 list——`(append [1 2] nil)` 的 `nil` 表示"最后一个参数是 list 尾",所以 [1 2] 变成 (1 2)。

### 5.3 Vector 操作

```elisp
(aref [1 2 3] 0)              ; → 1
(aset [1 2 3] 0 99)           ; → [99 2 3]
(vector 'a 'b 'c)             ; → [a b c]
```

`aref`/`aset` 是 array 的索引——O(1)。和 list 的 `nth` (O(n)) 不同,vector 索引立即返回。

### 5.4 Char-table

char-table 是特殊 array——按 Unicode codepoint 索引。专门用于"syntax table"、"case table"、"category table" 等"按字符配置"的场景。

```elisp
(make-char-table 'syntax-table nil)
```

(Module 6 W7 详细)——syntax table 就是 char-table,每个字符对应一个 syntax class。

### 5.5 Bool-vector

bool-vector 是 bit array——每个元素 t/nil,但内存只占 1 bit。用于大位图 (如 charset)。

```elisp
(make-bool-vector 10 nil)
```

实战很少直接用——大部分场景 vector 就够。

---

## 6. Records (records.texi)

### 6.1 什么是 record

record 是 Emacs 27+ 引入的新类型——带 type tag 的 vector。设计目的: 让 Elisp 能模拟"对象"——虽然不是真 OOP,但有 type identity。

```elisp
(setq r (record 'person "Alice" 30))
(recordp r)                   ; → t
(aref r 0)                    ; → person (type tag)
(aref r 1)                    ; → "Alice"
(aref r 2)                    ; → 30
```

record 的第一个元素是 type tag (一个 symbol)。`recordp` 检查是否是 record。`aref`/`aset` 可以读写 (record 是 mutable,像 vector)。

### 6.2 cl-defstruct

`cl-defstruct` (来自 `cl-lib`) 是 record 的 syntactic sugar——自动生成 constructor、accessor、predicate。

```elisp
(require 'cl-lib)

(cl-defstruct person
  name age)

(setq p (make-person :name "Alice" :age 30))
(person-name p)               ; → "Alice"
(person-age p)                ; → 30
(setf (person-age p) 31)
(person-p p)                  ; → t
```

底层用 record——`(make-person :name "Alice" :age 30)` 创建 `#s(person "Alice" 30)`。`(person-name p)` 是 `(aref p 1)` 的语法糖。`(person-p p)` 检查 type tag 是否 `person`。

cl-defstruct 让 Elisp 写"OOP-like" 代码——比 defclass (EIEIO) 轻,比 alist 强类型。很多现代 package (如 project.el) 用 cl-defstruct 而非 alist。

---

## 7. Hash Tables (hash.texi)

### 7.1 创建

hash table 是 Elisp 的高效字典——O(1) 平均查找。比 alist (O(n)) 快得多,但内存开销大。

```elisp
(make-hash-table)                       ; 默认
(make-hash-table :test 'equal)          ; 用 equal 比 (string 友好)
(make-hash-table :test 'eq)             ; 用 eq (symbol 友好)
(make-hash-table :size 100)             ; 初始大小
(make-hash-table :weakness 'key)        ; 弱引用 (key 被 GC 时项被删)
```

`:test` 是关键——决定 hash 内部如何比较 key。`eq` 最快 (比较指针),但只对 symbol 可靠;`equal` 慢但通用 (逐字符比较 string)。**几乎总是用 `equal`**,除非你确定所有 key 都是 symbol。

`:weakness` 是高级特性——让 hash 持有"弱引用",key 被 GC 时项自动删除。用于缓存 (cache),避免内存泄漏。`:weakness 'key` 是 key 弱 (value 强);`:weakness 'value` 反之;`:weakness 'key-and-value` 都弱。

### 7.2 操作

```elisp
(setq h (make-hash-table :test 'equal))
(puthash "foo" 1 h)
(gethash "foo" h)             ; → 1
(gethash "bar" h)             ; → nil
(gethash "bar" h 'default)    ; → default
(remhash "foo" h)
(clrhash h)                   ; 清空
(hash-table-count h)          ; 项数
```

`gethash` 第三参数是 default value——key 不存在时返回它。如果你不传且 key 不存在,返回 `nil`——这可能和"key 存在但值是 nil"混淆。区分方式: `(gethash key h 'missing)` 然后 check `eq` 是否 `'missing`。

### 7.3 遍历

```elisp
(maphash (lambda (key value)
           (message "%s = %s" key value))
         h)
```

`maphash` 是 hash 的 `mapc`——遍历但不返回值。注意 hash **没有保证顺序**——遍历顺序和插入顺序无关 (implementation-defined)。如果你要有序,用 alist。

### 7.4 用 hash-table 替代 assoc

如果数据多 (100+ 项),hash 比 assoc 快:
- assoc: O(n) 每次查找
- hash: O(1) 平均

实战: 缓存大量数据 (如 syntax highlight cache、LSP symbol table) 用 hash。少量数据 (配置、metadata) 用 alist——简单、可读、易序列化。

---

## 8. Streams (streams.texi)

"Stream" 在 Emacs 里指 read/print——Lisp 数据和文本之间的转换。这是 Elisp 的序列化机制。

### 8.1 Read

`read` 从输入 (字符串、buffer、minibuffer) 解析 Lisp form。

```elisp
(read)                        ; 从 minibuffer 读 sexp
(read-from-string "(a b c) rest")  ; → ((a b c) . 7)
(read-from-minibuffer "Input: ")
```

`(read)` 不带参数从 minibuffer 读——用户输入 `(+ 1 2)` RET,返回 list `(+ 1 2)`。这是 Elisp 的"eval input"基础。

`read-from-string` 返回 cons——`(form . position)`,position 是解析停止的字符位置。这让你可以连续 read 多个 form。

### 8.2 Print

`print` 和 `princ` 是两个版本:
- `princ`: 人类可读 (不加 quote)
- `print`: Lisp 可读 (加 quote,可被 read 读回)

```elisp
(princ "hello")               ; 不加 quote,人类可读
(print "hello")               ; 加 quote,Lisp 可读
(pp "hello")                  ; pretty-print
(format "%S" "hello")         ; → "\"hello\"" (Lisp-readable)
(format "%s" "hello")         ; → "hello" (human-readable)
```

这个区别极重要——当你持久化数据到文件,用 `print`/`pp` (可读回);当你显示给用户,用 `princ`/`%s`。

`pp` 是 `print` 的 pretty 版本——多行 list 缩进对齐。`(pp '(a (b (c (d)))) (current-buffer))` 输出格式化的多行。

### 8.3 序列化

把 Elisp 数据存到文件,再读回——这是"配置文件"、"cache"、"会话状态"的基础。

```elisp
;; 写
(with-temp-file "data.el"
  (pp my-data (current-buffer)))

;; 读
(with-temp-buffer
  (insert-file-contents "data.el")
  (read (current-buffer)))
```

这个 pattern 极常见——你写的每个 package 几乎都用。`(pp data (current-buffer))` 把 data 写成 Lisp-readable 格式,`(read (current-buffer))` 读回。**注意**: 这只对"自包含"数据 (number、string、symbol、list、vector) 工作。function、buffer、process 等无法序列化。

安全警告: `read` 任意字符串有"代码注入"风险——恶意文件可能包含 `(#.（shell-command "rm -rf ~"))` 这种 reader macro,read 时执行。**永远不要 read 不可信源**。如果必须,用 `read-circle` 限制或自己 parse。

---

## 9. 自测

1. `(type-of "abc")` → ?
2. `(string-to-number "abc")` → ?
3. `(format "%05d" 42)` → ?
4. hash-table 用什么 :test 处理 string key?
5. 怎么遍历 hash-table?
6. `princ` 和 `print` 区别?
7. cl-defstruct 底层用什么?
8. record 的第一个元素是什么?

**答案**:
> 1. string
> 2. 0
> 3. "00042"
> 4. 'equal
> 5. maphash
> 6. princ 人类可读;print Lisp-readable (加 quote)
> 7. record
> 8. type tag (symbol)

---

## 10. Exercises

### Ex 1.1: my-type-name

```elisp
(defun my-type-name (obj)
  "Return type name as string."
  ;; 用 type-of 和 symbol-name
  )

(my-type-name 5)              ; → "integer"
(my-type-name "hi")           ; → "string"
```

<details><summary>答案</summary>

```elisp
(defun my-type-name (obj)
  (symbol-name (type-of obj)))
```

</details>

### Ex 1.2: my-string-capitalize-words

```elisp
(defun my-capitalize-words (s)
  ;; "hello world foo" → "Hello World Foo"
  )
```

<details><summary>答案</summary>

```elisp
(defun my-capitalize-words (s)
  (mapconcat #'capitalize (split-string s) " "))
```

</details>

### Ex 1.3: my-vector-sum

```elisp
(defun my-vector-sum (vec)
  ;; [1 2 3 4] → 10
  )
```

<details><summary>答案</summary>

```elisp
(defun my-vector-sum (vec)
  (let ((sum 0))
    (dotimes (i (length vec) sum)
      (setq sum (+ sum (aref vec i))))))
```

</details>

### Ex 1.4: my-hash-keys

```elisp
(defun my-hash-keys (table)
  ;; 返回所有 key 的 list
  )
```

<details><summary>答案</summary>

```elisp
(defun my-hash-keys (table)
  (let (keys)
    (maphash (lambda (k _v) (push k keys)) table)
    (nreverse keys)))
```

</details>

### Ex 1.5: my-alist-to-hash

```elisp
(defun my-alist-to-hash (alist)
  ;; 把 alist 转 hash-table
  )
```

<details><summary>答案</summary>

```elisp
(defun my-alist-to-hash (alist)
  (let ((h (make-hash-table :test 'equal)))
    (dolist (pair alist h)
      (puthash (car pair) (cdr pair) h))))
```

</details>

---

## 11. 下一步

进入 `concept-anchor.md` + `exercises.md`,然后 `../week-02-eval-variables/`。
