# Exercises: Week 1 Data Types

> 15 题练习

这些练习从最简单的 type-of 开始,逐步加难度——直到让你写真实的 utility (format-size、word-wrap、camel-to-snake)。每题考一个具体技能,做完你能感觉自己"知道每个数据类型怎么用"。

---

## 第一组: 类型识别 (5 题)

这 5 题是"基础反射"——你看到值,知道它什么类型。这是 Module 3 教过操作,Module 6 现在要你"知道身份"。类型决定哪些函数可用、哪些不可用——`(aref "abc" 0)` 工作但 `(aref 5 0)` 不工作,因为 integer 不是 array。

### 题 1

`integerp` 是 predicate (返回 t/nil)。`(type-of ...)` 返回类型 symbol。两者的关系: `(integerp X)` 等价 `(eq (type-of X) 'integer)`。

```elisp
(type-of 5)              ; → ?
```

### 题 2

```elisp
(type-of "abc")          ; → ?
```

### 题 3

list 的 type 是 `cons`——因为 list 由 cons cell 构成。空 list `nil` 的 type 是什么? `(type-of nil)` 返回 `symbol`——因为 nil 同时是 symbol 和空 list (Emacs 的"诡计")。

```elisp
(type-of '(1 2 3))       ; → ?
```

### 题 4

```elisp
(type-of [1 2 3])        ; → ?
```

### 题 5

```elisp
(type-of 'foo)           ; → ?
```

**答案**: integer, string, cons, vector, symbol

---

## 第二组: 数字 (5 题)

数字练习考: 浮点转换 (average)、格式化 (format-size)、字符操作 (random-string)、算法 (gcd)、经典 (fizzbuzz)。

### 题 6: my-average

`(/ total count)` 整除返回 integer。如果你要小数,要么转 float。`(float N)` 把 integer 转 float。

```elisp
(defun my-average (numbers)
  ;; 返回 list 平均值
  )

(my-average '(1 2 3 4 5))    ; → 3.0
```

<details><summary>答案</summary>

```elisp
(defun my-average (numbers)
  (/ (apply #'+ numbers) (float (length numbers))))
```

`(float (length numbers))` 强制转 float,这样 `/` 返回 float。如果不加,`(apply #'+ '(1 2 3 4 5))` = 15,`(length ...)` = 5,`(/ 15 5)` = 3 (integer)。

</details>

### 题 7: my-format-size

这道题考 `cond` + `format`。注意阈值是 1024 的幂: KB=1024, MB=1024², GB=1024³。

```elisp
(defun my-format-size (bytes)
  ;; 1024 → "1.00 KB", 1048576 → "1.00 MB"
  )
```

<details><summary>答案</summary>

```elisp
(defun my-format-size (bytes)
  (cond
   ((< bytes 1024) (format "%d B" bytes))
   ((< bytes 1048576) (format "%.2f KB" (/ bytes 1024.0)))
   ((< bytes 1073741824) (format "%.2f MB" (/ bytes 1048576.0)))
   (t (format "%.2f GB" (/ bytes 1073741824.0)))))
```

注意 `(/ bytes 1024.0)`——除以 float 得 float。如果是 `(/ bytes 1024)` 得 integer,format 会显示"1 KB" 而不是 "1.00 KB"。

</details>

### 题 8: my-random-string

这道题考 array 操作——用 `aref` 从 char 池取字符,组合成 string。

```elisp
(defun my-random-string (length)
  ;; 生成长度 N 的随机字符串
  )
```

<details><summary>答案</summary>

```elisp
(defun my-random-string (length)
  (let ((chars "abcdefghijklmnopqrstuvwxyz0123456789"))
    (apply #'string
           (mapcar (lambda (_)
                     (aref chars (random (length chars))))
                   (make-list length nil)))))
```

`make-list` 创建 length 个 nil 的 list (作为计数器),`mapcar` 遍历生成 chars,`apply #'string` 把 list of char 转 string。

实战用途: 生成临时 ID、密码、token。

</details>

### 题 9: my-gcd

经典算法——欧几里得算法。`(mod a b)` 返回余数,gcd(b, mod) 直到 b=0。

```elisp
(defun my-gcd (a b)
  ;; 最大公约数
  )

(my-gcd 12 18)              ; → 6
```

<details><summary>答案</summary>

```elisp
(defun my-gcd (a b)
  (if (zerop b)
      a
    (my-gcd b (mod a b))))
```

递归形式。Emacs 不做 TCO (tail-call optimization),所以大数会爆栈。但 gcd 几步就结束,问题不大。

</details>

### 题 10: my-fizzbuzz

经典入门题——遍历 1 到 N,3 倍数 "Fizz",5 倍数 "Buzz",15 倍数 "FizzBuzz"。

```elisp
(defun my-fizzbuzz (n)
  ;; 1-N 的 list: 3 倍数 "Fizz", 5 倍数 "Buzz", 都倍 "FizzBuzz"
  )

(my-fizzbuzz 15)
;; → (1 2 "Fizz" 4 "Buzz" "Fizz" 7 8 "Fizz" "Buzz" 11 "Fizz" 13 14 "FizzBuzz")
```

<details><summary>答案</summary>

```elisp
(defun my-fizzbuzz (n)
  (mapcar (lambda (i)
            (cond
             ((zerop (mod i 15)) "FizzBuzz")
             ((zerop (mod i 3)) "Fizz")
             ((zerop (mod i 5)) "Buzz")
             (t i)))
          (number-sequence 1 n)))
```

注意 cond 顺序——先 check 15 (最严格),再 check 3 和 5。如果反过来,15 会被 3 匹配,返回 "Fizz" 而不是 "FizzBuzz"。

</details>

---

## 第三组: 字符串 (5 题)

字符串练习从简单 (palindrome) 到复杂 (camel-to-snake)。

### 题 11: my-palindrome-p

```elisp
(defun my-palindrome-p (s)
  ;; "racecar" → t, "hello" → nil
  )

(my-palindrome-p "racecar")
```

<details><summary>答案</summary>

```elisp
(defun my-palindrome-p (s)
  (string= s (reverse s)))
;; 或:
(defun my-palindrome-p (s)
  (let ((len (length s)))
    (cl-loop for i from 0 to (/ len 2)
             always (char-equal (aref s i) (aref s (- len 1 i))))))
```

第一个版本最简洁——`(reverse s)` 返回反转 string,`string=` 比对。O(n) 时间 + O(n) 空间。

第二个版本 O(n) 时间 + O(1) 空间——遍历前半,和后半对应位置比较。`char-equal` 忽略大小写 (`(char-equal ?a ?A)` 返回 t)。

</details>

### 题 12: my-word-wrap

word-wrap 是真实需求——format 注释、生成 commit message 等。最简单用 Emacs 内置 `fill-region`。

```elisp
(defun my-word-wrap (s width)
  ;; 在 width 字符处换行
  )
```

<details><summary>答案</summary>

```elisp
(defun my-word-wrap (s width)
  (with-temp-buffer
    (insert s)
    (fill-region (point-min) (point-max))
    (let ((fill-column width))
      (fill-paragraph))
    (buffer-string)))
```

注意 `let` 必须在 `fill-paragraph` 之前——`(let ((fill-column width)) ...)` 临时设置 `fill-column` buffer-local,影响 `fill-paragraph` 行为。

</details>

### 题 13: my-count-substring

```elisp
(defun my-count-substring (sub str)
  ;; 数 sub 在 str 出现次数
  )

(my-count-substring "ab" "ababab")    ; → 3
```

<details><summary>答案</summary>

```elisp
(defun my-count-substring (sub str)
  (let ((count 0)
        (start 0))
    (while (string-match (regexp-quote sub) str start)
      (setq count (1+ count)
            start (match-end 0)))
    count))
```

`regexp-quote` 转义 sub 中的正则特殊字符 (如 `.` → `\.`),让 search 是字面匹配。`match-end 0` 是上次匹配结束位置,作为下次起点。循环到 `string-match` 返回 nil。

</details>

### 题 14: my-snake-to-kebab

```elisp
(defun my-snake-to-kebab (s)
  ;; "my_var_name" → "my-var-name"
  )
```

<details><summary>答案</summary>

```elisp
(defun my-snake-to-kebab (s)
  (replace-regexp-in-string "_" "-" s))
```

一行搞定。`replace-regexp-in-string` 是 string 替换的瑞士军刀。

</details>

### 题 15: my-camel-to-snake

```elisp
(defun my-camel-to-kebab (s)
  ;; "myVarName" → "my_var_name"
  )
```

<details><summary>答案</summary>

```elisp
(defun my-camel-to-snake (s)
  (let ((result "")
        (prev-lower nil))
    (dotimes (i (length s))
      (let ((c (aref s i)))
        (when (and prev-lower
                   (upper-case-p c))
          (setq result (concat result "_")))
        (setq result (concat result (downcase (string c)))
              prev-lower (lower-case-p c))))
    result))
```

算法: 遍历每个字符。如果前一个是小写,当前是大写——插入 `_` (词边界)。否则直接 downcase 加。

`prev-lower` 是状态机——记住前一个字符是小写还是大写,决定当前字符是否是"驼峰边界"。

</details>

---

## 自测

1. `(type-of (current-buffer))` → ?
2. `(format "%.2f" 3.14159)` → ?
3. `(string-to-number "abc")` → ?
4. 怎么生成 0-99 的随机数?
5. 怎么遍历 hash-table?

**答案**:
> 1. buffer
> 2. "3.14"
> 3. 0
> 4. `(random 100)`
> 5. maphash

---

## 毕业检查

- [ ] 15 题至少完成 12
- [ ] 能用 format 各种格式化
- [ ] 能操作 string 和 list
- [ ] 能用 hash-table

完成后进入 `../week-02-eval-variables/`。
