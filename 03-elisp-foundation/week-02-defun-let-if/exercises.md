# Exercises: Week 2 defun + let + if

> 20 题练习,巩固 defun/let/if/interactive/buffer 操作

这一周练习重点不是"算法",而是"工具"——你写的每个函数都能用到日常编辑。所以每题都问自己: "这个函数能解决什么真实问题?"。

**方法**: 每题先尝试 5-10 分钟。卡住看 hint。完成后**把函数加到自己的 init.el**——这样它成了你的"日常工具",而不只是练习答案。

---

## 第一组: defun 基础 (5 题,30 分钟)

这组训练 defun 的基本语法——参数、返回值、简单算术。简单但重要: 它们是所有后续 defun 的模板。

### 题 1: my-square
写 `(my-square N)`,返回 N 的平方:

```elisp
(defun my-square (n)
  ;; 你的代码
  )

(my-square 5)      ;; → 25
(my-square -3)     ;; → 9
```

<details><summary>答案</summary>

```elisp
(defun my-square (n)
  (* n n))
```

</details>

### 题 2: my-max-of-two
返回两个数中较大的:

```elisp
(defun my-max-of-two (a b)
  ;; 你的代码
  )

(my-max-of-two 3 5)    ;; → 5
(my-max-of-two 10 2)   ;; → 10
```

<details><summary>答案</summary>

```elisp
(defun my-max-of-two (a b)
  (if (> a b) a b))
```

</details>

### 题 3: my-fahrenheit-to-celsius
```elisp
(defun my-f-to-c (f)
  ;; 你的代码
  )

(my-f-to-c 32)     ;; → 0
(my-f-to-c 212)    ;; → 100
```

<details><summary>答案</summary>

```elisp
(defun my-f-to-c (f)
  (* (- f 32) 5/9))
```

</details>

### 题 4: my-evenp
判断 N 是否偶数:

```elisp
(defun my-evenp (n)
  ;; 你的代码
  )

(my-evenp 4)       ;; → t
(my-evenp 3)       ;; → nil
```

<details><summary>答案</summary>

```elisp
(defun my-evenp (n)
  (zerop (mod n 2)))
```

</details>

### 题 5: my-leap-year
判断是否闰年:

```elisp
(defun my-leap-year (year)
  ;; 你的代码
  )

(my-leap-year 2000)    ;; → t
(my-leap-year 1900)    ;; → nil
(my-leap-year 2024)    ;; → t
```

**Hint**: 闰年规则: 能被 4 整除且不能被 100 整除,**或者**能被 400 整除。用 or 和 and 组合两个条件。`zerop (mod year N)` 判断"能被 N 整除"。

<details><summary>答案</summary>

```elisp
(defun my-leap-year (year)
  (or (and (zerop (mod year 4))
           (not (zerop (mod year 100))))
      (zerop (mod year 400))))
```

</details>

---

## 第二组: let + let* (5 题,30 分钟)

这组训练局部变量。关键是理解 let vs let*——它们在"后面的 var 引用前面的 var"时行为不同。

### 题 6: my-distance
两点距离:

```elisp
(defun my-distance (x1 y1 x2 y2)
  ;; 你的代码 (用 let)
  )

(my-distance 0 0 3 4)    ;; → 5.0
```

**Hint**: 用 let 缓存 dx 和 dy,然后 `sqrt (+ (* dx dx) (* dy dy))`。

<details><summary>答案</summary>

```elisp
(defun my-distance (x1 y1 x2 y2)
  (let ((dx (- x2 x1))
        (dy (- y2 y1)))
    (sqrt (+ (* dx dx) (* dy dy)))))
```

</details>

### 题 7: 用 let*
```elisp
(defun my-complicated (x)
  ;; 用 let* 计算 ((x+1)*2)+3
  )
```

**Hint**: 这是 let* 的典型用例——`plus1` 依赖 x,`times2` 依赖 plus1。用 let* 让"后面可见前面"。

<details><summary>答案</summary>

```elisp
(defun my-complicated (x)
  (let* ((plus1 (+ x 1))
         (times2 (* plus1 2)))
    (+ times2 3)))
```

</details>

### 题 8: my-sum-of-squares
```elisp
(defun my-sum-of-squares (a b)
  ;; 返回 a^2 + b^2
  )
```

<details><summary>答案</summary>

```elisp
(defun my-sum-of-squares (a b)
  (let ((asq (* a a))
        (bsq (* b b)))
    (+ asq bsq)))
```

</details>

### 题 9: my-bind-test
解释这个 let 和 let* 的输出:

```elisp
(let ((x 1) (y 2))
  (let ((x y) (y x))     ; let
    (list x y)))
;; → ?

(let ((x 1) (y 2))
  (let* ((x y) (y x))    ; let*
    (list x y)))
;; → ?
```

<details><summary>答案</summary>

> let: → (2 1) (外层 x=1, y=2;let 内 x=外y=2, y=外x=1)
> let*: → (2 2) (内 x=外y=2;内 y=新x=2)

</details>

### 题 10: my-swap
交换两个变量的值 (用 let):

```elisp
(defun my-swap (a b)
  ;; 返回 (B . A)
  )

(my-swap 1 2)    ;; → (2 . 1)
```

<details><summary>答案</summary>

```elisp
(defun my-swap (a b)
  (let ((temp a))
    (cons b temp)))
```

</details>

---

## 第三组: if / when / unless (5 题,30 分钟)

### 题 11: my-grade
分数 → 等级:

```elisp
(defun my-grade (score)
  ;; 90+ → A, 80+ → B, 70+ → C, 60+ → D, else → F
  )

(my-grade 95)    ;; → "A"
(my-grade 85)    ;; → "B"
(my-grade 55)    ;; → "F"
```

<details><summary>答案</summary>

```elisp
(defun my-grade (score)
  (cond
   ((>= score 90) "A")
   ((>= score 80) "B")
   ((>= score 70) "C")
   ((>= score 60) "D")
   (t "F")))
```

</details>

### 题 12: my-sign
返回数字符号:

```elisp
(defun my-sign (n)
  ;; n>0 → 1, n<0 → -1, else → 0
  )

(my-sign 5)      ;; → 1
(my-sign -3)     ;; → -1
(my-sign 0)      ;; → 0
```

<details><summary>答案</summary>

```elisp
(defun my-sign (n)
  (cond
   ((> n 0) 1)
   ((< n 0) -1)
   (t 0)))
```

</details>

### 题 13: my-safe-divide
```elisp
(defun my-safe-divide (a b)
  ;; 返回 a/b,如果 b=0 返回 nil
  )

(my-safe-divide 10 2)    ;; → 5
(my-safe-divide 10 0)    ;; → nil
```

<details><summary>答案</summary>

```elisp
(defun my-safe-divide (a b)
  (if (zerop b)
      nil
    (/ a b)))
```

</details>

### 题 14: my-empty-or
```elisp
(defun my-empty-or-default (str default)
  ;; 如果 str 是空字符串,返回 default;否则返回 str
  )

(my-empty-or-default "" "fallback")    ;; → "fallback"
(my-empty-or-default "hi" "fallback")  ;; → "hi"
```

<details><summary>答案</summary>

```elisp
(defun my-empty-or-default (str default)
  (if (string-empty-p str)
      default
    str))
```

</details>

### 题 15: my-classify
```elisp
(defun my-classify (x)
  ;; 返回 symbol: number / string / list / symbol / other
  )

(my-classify 5)        ;; → number
(my-classify "hi")     ;; → string
(my-classify '(1 2))   ;; → list
(my-classify 'foo)     ;; → symbol
```

<details><summary>答案</summary>

```elisp
(defun my-classify (x)
  (cond
   ((numberp x) 'number)
   ((stringp x) 'string)
   ((listp x) 'list)
   ((symbolp x) 'symbol)
   (t 'other)))
```

</details>

---

## 第四组: interactive + buffer (5 题,40 分钟)

### 题 16: my-greet
提示名字,问好:

```elisp
(defun my-greet (name)
  ;; 用 interactive "s..."
  )

;; M-x my-greet RET Alice RET → "Hello, Alice!"
```

<details><summary>答案</summary>

```elisp
(defun my-greet (name)
  (interactive "sName: ")
  (message "Hello, %s!" name))
```

</details>

### 题 17: my-insert-with-prefix
插入 prefix arg 次 "X":

```elisp
(defun my-insert-x (times)
  ;; interactive "p"
  )
```

<details><summary>答案</summary>

```elisp
(defun my-insert-x (times)
  (interactive "p")
  (dotimes (_ times)
    (insert "X")))
```

</details>

### 题 18: my-count-region
```elisp
(defun my-count-region (start end)
  ;; interactive "r"
  ;; 显示 region 内字符数和词数
  )
```

<details><summary>答案</summary>

```elisp
(defun my-count-region (start end)
  (interactive "r")
  (let* ((text (buffer-substring start end))
         (chars (length text))
         (words (length (split-string text))))
    (message "Region: %d chars, %d words" chars words)))
```

</details>

### 题 19: my-capitalize-word-at-point
```elisp
(defun my-capitalize-word-at-point ()
  ;; 把光标处的 word 首字母大写
  )
```

<details><summary>答案</summary>

```elisp
(defun my-capitalize-word-at-point ()
  (interactive)
  (let ((bounds (bounds-of-thing-at-point 'word)))
    (when bounds
      (let* ((start (car bounds))
             (end (cdr bounds))
             (word (buffer-substring start end)))
        (delete-region start end)
        (goto-char start)
        (insert (capitalize word))))))
```

</details>

### 题 20: my-comment-toggle
```elisp
(defun my-comment-toggle-line ()
  ;; 切换当前行的注释状态
  )
```

<details><summary>答案</summary>

```elisp
(defun my-comment-toggle-line ()
  (interactive)
  (save-excursion
    (comment-line 1)))   ; 内置
```

实际上 `(comment-line 1)` 已经能做这事,这题是教你 save-excursion + 调用内置。

</details>

---

## 自测

1. `(interactive "sFoo: ")` 干啥?
2. `(interactive "nFoo: ")` 和 `"sFoo: "` 区别?
3. `let` vs `let*`?
4. `if` 和 `when` 区别?
5. `nil` 和 `0` 哪个是 false?
6. `(if "hello" 'yes 'no)` → ?
7. `(when '() 'foo)` → ?
8. 写一个函数 `(my-trim S)`,去掉字符串两端空白。

**答案**:
> 1. 提示 "Foo: ",读字符串作第一参数
> 2. n 读数字 (read-number),s 读字符串
> 3. let 并行绑定,let* 顺序
> 4. when 只有 then (可多 form);if 有 then/else
> 5. nil 是 false;0 是 true
> 6. yes (非 nil 字符串是 true)
> 7. nil ('() = nil,when 不执行)
> 8. 
> ```elisp
> (defun my-trim (s)
>   (let ((result (replace-regexp-in-string "\\`[ \t\n]+" "" s)))
>     (replace-regexp-in-string "[ \t\n]+\\'" "" result)))
> ```

---

## 毕业检查

- [ ] 20 题至少完成 16
- [ ] 自测 8 题至少答对 6
- [ ] 能用 defun/let/if 写任意函数
- [ ] 能用 interactive 让函数变命令
- [ ] 能用 save-excursion 操作 buffer

完成后进入 `../week-03-recursion/chassell-reading.md`。
