# Exercises: Week 4 Lambda + mapcar + Sequences

> 20 题练习,高阶函数 + 字符串 + 正则

这周是 Module 3 最"实用"的一周——你学的 lambda、mapcar、正则,日常编辑每天都用。

**方法**: 每道题做完后,**问自己"这个函数能解决什么真实问题?"**。比如 `my-find-numbers` 提取字符串中所有数字——可能用于"分析 log"、"提取代码中的数字"。让练习变成"工具集"。

---

## 第一组: Lambda + funcall (5 题,30 分钟)

这组训练 lambda 的基本用法——定义、调用、传给函数。

### 题 1: 简单 lambda 调用
```elisp
(funcall (lambda (x y) (- x y)) 10 3)    ;; → ?
```

**Hint**: lambda 接受 x y,返回 x - y。funcall 调用它,参数 10 3。

### 题 2: my-call-twice
```elisp
(defun my-call-twice (fn x)
  ;; (funcall fn (funcall fn x))
  )

(my-call-twice #'1+ 5)            ;; → 7
(my-call-twice (lambda (s) (concat s s)) "ab")    ;; → "ababab"
```

**Hint**: 调用 fn 两次——`(funcall fn (funcall fn x))`。第二次的输入是第一次的输出。

<details><summary>答案</summary>

```elisp
(defun my-call-twice (fn x)
  (funcall fn (funcall fn x)))
```

</details>

### 题 3: my-identity
```elisp
(defun my-identity (x)
  ;; 返回 x 不变
  )
```

### 题 4: my-constantly
```elisp
(defun my-constantly (val)
  ;; 返回一个 lambda,永远返回 val
  )

(setq always-5 (my-constantly 5))
(funcall always-5)         ;; → 5
(funcall always-5 'foo)    ;; → 5
```

<details><summary>答案</summary>

```elisp
(defun my-constantly (val)
  (lambda (&rest _)
    val))
```

</details>

### 题 5: my-partial
部分应用:

```elisp
(defun my-partial (fn &rest args)
  ;; (my-partial #'+ 5) 返回 (lambda (&rest more) (apply + 5 more))
  )

(setq add5 (my-partial #'+ 5))
(funcall add5 3)            ;; → 8
(funcall add5 10 20)        ;; → 35
```

**Hint**: 返回一个 lambda,它接受 `&rest more`,内部用 `(apply fn (append args more))` 把预设参数 args 和新参数 more 合并。这就是"柯里化" (currying) 的部分实现——把多参数函数变成"先填一部分,再填剩下"。

<details><summary>答案</summary>

```elisp
(defun my-partial (fn &rest args)
  (lambda (&rest more)
    (apply fn (append args more))))
```

</details>

---

## 第二组: mapcar / mapc (5 题,30 分钟)

这一组训练批量处理——mapcar 是 Lisp 最常用的高阶函数。做完这组,你就掌握了"对每个元素做同样操作"的标准模式。

### 题 6: 双倍
```elisp
(defun my-double-list (list)
  ;; mapcar
  )

(my-double-list '(1 2 3))    ;; → (2 4 6)
```

**Hint**: `(mapcar (lambda (x) (* 2 x)) list)`。lambda 接受 x,返回 2x。

<details><summary>答案</summary>

```elisp
(defun my-double-list (list)
  (mapcar (lambda (x) (* 2 x)) list))
```

</details>

### 题 7: 长度 list
```elisp
(defun my-lengths (list-of-strings)
  ;; (my-lengths '("a" "bb" "ccc")) → (1 2 3)
  )

(my-lengths '("a" "bb" "ccc"))    ;; → (1 2 3)
```

<details><summary>答案</summary>

```elisp
(defun my-lengths (list-of-strings)
  (mapcar #'length list-of-strings))
```

</details>

### 题 8: 双 list
```elisp
(defun my-add-pairs (l1 l2)
  ;; (my-add-pairs '(1 2 3) '(10 20 30)) → (11 22 33)
  )

(my-add-pairs '(1 2 3) '(10 20 30))
```

<details><summary>答案</summary>

```elisp
(defun my-add-pairs (l1 l2)
  (mapcar #'+ l1 l2))
```

</details>

### 题 9: 大写所有
```elisp
(defun my-upcase-words (list)
  ;; (my-upcase-words '("foo" "bar")) → ("FOO" "BAR")
  )

(my-upcase-words '("foo" "bar"))
```

<details><summary>答案</summary>

```elisp
(defun my-upcase-words (list)
  (mapcar #'upcase list))
```

</details>

### 题 10: mapc 打印
```elisp
(defun my-print-all (list)
  ;; 用 mapc,每个 message,不收集
  )
```

<details><summary>答案</summary>

```elisp
(defun my-print-all (list)
  (mapc (lambda (x) (message "%s" x)) list))
```

</details>

---

## 第三组: 字符串 (5 题,30 分钟)

这一组训练字符串操作。字符串处理是 Emacs 的日常——你编辑的所有内容都是字符串。

### 题 11: my-trim
```elisp
(defun my-trim (s)
  ;; 去前后空白
  )

(my-trim "  hello  ")    ;; → "hello"
```

**Hint**: 用 replace-regexp-in-string 两次——先去开头空白 `\\`[ \t\n]+`,再去结尾空白 `[ \t\n]+\\'`。或者直接 `(string-trim s)` (Emacs 25+)。

<details><summary>答案</summary>

```elisp
(defun my-trim (s)
  (let ((r (replace-regexp-in-string "\\`[ \t\n]+" "" s)))
    (replace-regexp-in-string "[ \t\n]+\\'" "" r)))
```

</details>

### 题 12: my-split-by-space
```elisp
(defun my-split-by-space (s)
  ;; "a b c" → ("a" "b" "c")
  )

(my-split-by-space "a b c")    ;; → ("a" "b" "c")
```

<details><summary>答案</summary>

```elisp
(defun my-split-by-space (s)
  (split-string s "[ \t]+" t))
```

</details>

### 题 13: my-capitalize-first
```elisp
(defun my-capitalize-first (s)
  ;; 首字母大写,其他不变
  )

(my-capitalize-first "hello")    ;; → "Hello"
(my-capitalize-first "HELLO")    ;; → "HELLO"
```

<details><summary>答案</summary>

```elisp
(defun my-capitalize-first (s)
  (if (string-empty-p s)
      s
    (concat (upcase (substring s 0 1))
            (substring s 1))))
```

</details>

### 题 14: my-reverse-words
```elisp
(defun my-reverse-words (s)
  ;; "hello world foo" → "foo world hello"
  )

(my-reverse-words "hello world foo")
```

<details><summary>答案</summary>

```elisp
(defun my-reverse-words (s)
  (mapconcat #'identity (nreverse (split-string s)) " "))
```

</details>

### 题 15: my-snake-to-camel
```elisp
(defun my-snake-to-camel (s)
  ;; "my_var_name" → "myVarName"
  )

(my-snake-to-camel "my_var_name")    ;; → "myVarName"
```

<details><summary>答案</summary>

```elisp
(defun my-snake-to-camel (s)
  (let ((parts (split-string s "_")))
    (concat (car parts)
            (mapconcat #'capitalize (cdr parts) ""))))
```

</details>

---

## 第四组: 正则 (5 题,40 分钟)

这一组训练正则——文本提取的"杀手锏"。难度比前三组高,因为正则本身就是一门"小语言"。

**方法**: 每题先想清楚"我要匹配什么模式",再写正则。用 `M-x re-builder` 交互式测试正则,验证正确再写到代码里。

### 题 16: my-find-numbers
```elisp
(defun my-find-numbers (s)
  ;; 提取所有数字
  )

(my-find-numbers "abc 12 def 34 ghi 56")    ;; → (12 34 56)
```

**Hint**: `[0-9]+` 匹配一个或多个数字。用 while + string-match + match-string 收集。每次更新 start 为 `(match-end 0)` 避免死循环。

<details><summary>答案</summary>

```elisp
(defun my-find-numbers (s)
  (let (result
        (start 0))
    (while (string-match "[0-9]+" s start)
      (push (string-to-number (match-string 0 s)) result)
      (setq start (match-end 0)))
    (nreverse result)))
```

</details>

### 题 17: my-find-urls
```elisp
(defun my-find-urls (s)
  ;; 提取 URLs (简化)
  )

(my-find-urls "visit https://foo.com and http://bar.com/baz")
;; → ("https://foo.com" "http://bar.com/baz")
```

<details><summary>答案</summary>

```elisp
(defun my-find-urls (s)
  (let (result
        (start 0))
    (while (string-match "https?://[a-zA-Z0-9./?=&-]+" s start)
      (push (match-string 0 s) result)
      (setq start (match-end 0)))
    (nreverse result)))
```

</details>

### 题 18: my-replace-tags
```elisp
(defun my-replace-tags (s)
  ;; 把 [b]...[/b] 改成 **...**
  )

(my-replace-tags "[b]hello[/b] world [b]foo[/b]")
;; → "**hello** world **foo**"
```

<details><summary>答案</summary>

```elisp
(defun my-replace-tags (s)
  (let* ((r (replace-regexp-in-string "\\[b\\]" "**" s)))
    (replace-regexp-in-string "\\[/b\\]" "**" r)))
```

</details>

### 题 19: my-validate-email
```elisp
(defun my-valid-email-p (s)
  ;; 简单邮箱验证
  )

(my-valid-email-p "alice@foo.com")    ;; → t
(my-valid-email-p "invalid")          ;; → nil
```

<details><summary>答案</summary>

```elisp
(defun my-valid-email-p (s)
  (string-match "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]\\{2,\\}$" s))
```

</details>

### 题 20: my-extract-markdown-links
```elisp
(defun my-extract-md-links (s)
  ;; 从 [text](url) 提取 (text . url) 对
  )

(my-extract-md-links "[foo](http://foo.com) and [bar](http://bar.com)")
;; → (("foo" . "http://foo.com") ("bar" . "http://bar.com"))
```

<details><summary>答案</summary>

```elisp
(defun my-extract-md-links (s)
  (let (result
        (start 0))
    (while (string-match "\\[\\([^]]+\\)\\](\\([^)]+\\))" s start)
      (push (cons (match-string 1 s) (match-string 2 s)) result)
      (setq start (match-end 0)))
    (nreverse result)))
```

</details>

---

## 自测

1. `(funcall (lambda (x) (* x 2)) 5)` → ?
2. `(mapcar #'1+ '(1 2 3))` → ?
3. `(apply #'+ '(1 2 3))` → ?
4. 写 `(my-compose F G)` 组合两函数
5. 写 `(my-reverse-words S)` 反转词序
6. 写 `(my-find-emails S)` 找所有 email
7. `(split-string "a,b,c" ",")` → ?
8. `(replace-regexp-in-string "[0-9]" "X" "a1b2c3")` → ?

**答案**:
> 1. 10
> 2. (2 3 4)
> 3. 6
> 4. `(defun my-compose (f g) (lambda (x) (funcall f (funcall g x))))`
> 5. `(mapconcat #'identity (nreverse (split-string s)) " ")`
> 6. 见题 17
> 7. ("a" "b" "c")
> 8. "aXbXcX"

---

## 毕业检查

- [ ] 20 题至少完成 16
- [ ] 能用 lambda + mapcar + funcall 熟练
- [ ] 能用正则提取信息
- [ ] 能写组合函数 (compose, pipe, partial)

完成后进入 `../week-05-capstone/`。
