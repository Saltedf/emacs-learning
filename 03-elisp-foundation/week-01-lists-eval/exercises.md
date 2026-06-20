# Exercises: Week 1 Lists & Evaluation

> 30 题练习,超越 Chassell 原书的题
> **用时**: 5 小时

练习不是"考你",是"巩固"。每道题都是 chassell-reading 里某个概念的"实战"。做不出来不要怕——回去看对应章节,再来。Lisp 学习就是"反复"——同一个概念从不同角度练 5 次,你才真正懂。

每道题的"答案"是参考实现,不是唯一答案。你的实现如果对就是对的——但要看参考答案的风格 (是否优雅、是否充分利用 Lisp 特性)。

---

## 第一组: List 基础 (10 题,30 分钟)

这一组是"心算训练"——不写代码,只用脑子算 list 操作的结果。这训练你的"list 直觉": 看到 `(car '(a b c))`,立刻反应 `a`,不需要打开 Emacs 验证。

**方法**: 拿张纸盖住答案,自己算,然后对答案。错的地方,想想为什么错 (是忘了 quote? 是把 car/cdr 弄反? 是 improper list?)。

写出每个表达式的值,**先想,再用 `C-x C-e` 验证**。

### 题 1
```elisp
(car '(1 2 3))
```

### 题 2
```elisp
(cdr '(1 2 3))
```

### 题 3
```elisp
(car (cdr '(1 2 3)))
```

### 题 4
```elisp
(cons 1 '(2 3))
```

### 题 5
```elisp
(cons 1 (cons 2 nil))
```

### 题 6
```elisp
(list 1 2 3)
```

### 题 7
```elisp
'(+ 1 2)
```

### 题 8
```elisp
(+ 1 2)
```

### 题 9
```elisp
(length '(a b c d e))
```

### 题 10
```elisp
(nth 3 '(a b c d e))
```

**答案**:
> 1. 1, 2. (2 3), 3. 2, 4. (1 2 3), 5. (1 2), 6. (1 2 3), 7. (+ 1 2), 8. 3, 9. 5, 10. d

---

## 第二组: Car/Cdr 组合 (5 题,20 分钟)

这一组训练 caar/cadr/caddr 等"组合操作"。这些缩写在别人代码里频繁出现,你要能立刻看懂。

记忆法: 从右往左读,a 是 car,d 是 cdr。`cadr` = 先 cdr 再 car = 取第二个元素。

### 题 11
```elisp
(caar '((a b) c))
```

### 题 12
```elisp
(cadr '(a b c))
```

### 题 13
```elisp
(caddr '(a b c d))
```

### 题 14
```elisp
(cdddr '(a b c d e))
```

### 题 15
```elisp
(cdar '((x) b c))
```

**答案**:
> 11. a, 12. b, 13. c, 14. (e), 15. nil (car 是 (x),cdr of (x) 是 nil)

---

## 第三组: Quote 详解 (5 题,20 分钟)

这一组是 Module 3 W1 最关键的练习——理解 quote。Quote 是 Lisp 与所有其他语言最大的不同,做不出来这组说明你还没"内化"代码即数据。

**关键认知**: `'X` 等于 `(quote X)`,返回 X 本身不 eval。`''foo` 是 `(quote (quote foo))` = `(quote foo)` 这个 list。

### 题 16
```elisp
(quote foo)
```

### 题 17
```elisp
'foo
```

### 题 18
```elisp
foo
```
(假设 `foo` 没定义)

### 题 19
```elisp
''foo
```

### 题 20
```elisp
(list 'quote 'foo)
```

**答案**:
> 16. foo (symbol), 17. foo, 18. void-variable error, 19. (quote foo), 20. (quote foo)

---

## 第四组: Cons Cell 画图 (5 题,30 分钟)

这一组训练"看见底层"。`(a b c)` 看起来是平的,但底层是嵌套 cons cell。能画图,说明你真正理解了 list 的结构。

为每个 list 画 cons cell 图。

### 题 21
```elisp
(a)
```

### 题 22
```elisp
(a b c)
```

### 题 23
```elisp
(a . b)   ; dotted pair
```

### 题 24
```elisp
(a b . c) ; improper list
```

### 题 25
```elisp
((a b) c) ; 嵌套
```

**答案**:

21.
```
┌───┬───┐
│ • │nil│
└─┬─┴───┘
  ↓
  a
```

22.
```
┌───┬───┐   ┌───┬───┐   ┌───┬───┐
│ • │ •─┼──►│ • │ •─┼──►│ • │nil│
└─┬─┴───┘   └─┬─┴───┘   └─┬─┴───┘
  ↓           ↓           ↓
  a           b           c
```

23.
```
┌───┬───┐
│ • │ • │
└─┬─┴─┬─┘
  ↓   ↓
  a   b
```

24.
```
┌───┬───┐   ┌───┬───┐
│ • │ •─┼──►│ • │ • │
└─┬─┴───┘   └─┬─┴─┬─┘
  ↓           ↓   ↓
  a           b   c
```

25.
```
┌───┬───┐   ┌───┬───┐
│ • │ •─┼──►│ • │nil│
└─┬─┴───┘   └─┬─┴───┘
  ↓           ↓
  ┌───┬───┐   c
  │ • │ •─┼──►┌───┬───┐
  └─┬─┴───┘   │ • │nil│
    ↓         └─┬─┴───┘
    a           ↓
                b
```

---

## 第五组: 写函数 (15 题,90 分钟)

这一组是 Module 3 W1 的核心练习——用递归实现经典 list 函数。每道题有要求:

- **不用** `mapcar`/`length`/`reverse`/`member` 等内置 (这是"练习"的目的)
- **用** `car`/`cdr`/`cons`/`if`/`null`/`equal` 这些原语
- **递归** (不用 while,这周学的是递归思维)

为什么限制?因为这些内置函数的实现就是这 15 道题。写一遍 `my-length`,你就真正理解了 `length` 怎么工作。下次用 `length`,你不是"调用黑箱",而是"知道它内部干什么"。

每道题先尝试 5-10 分钟,卡住了看 hint,实在不行看答案。看完答案**关上答案重写一遍**——这是关键,否则你只"看懂了",没"学会了"。

每题用递归 (不用 `mapcar`/`length`/`reverse` 等内置)。

### 题 26: my-last
返回 list 的最后一个元素:

```elisp
(defun my-last (list)
  ;; 你的代码
  )

(my-last '(a b c))    ;; → c
(my-last '(a))        ;; → a
(my-last '())         ;; → nil
```

**Hint**: 思考 base case——什么时候是"最后一个"?当 `(cdr list)` 是 `nil` 时,`(car list)` 就是最后。否则递归 `(cdr list)`。

<details><summary>答案</summary>

```elisp
(defun my-last (list)
  (if (null (cdr list))
      (car list)
    (my-last (cdr list))))
```

</details>

### 题 27: my-butlast
返回除最后元素的 list:

```elisp
(defun my-butlast (list)
  ;; 你的代码
  )

(my-butlast '(a b c))    ;; → (a b)
(my-butlast '(a))        ;; → nil
```

**Hint**: 跟 my-last 类似的 base case——当 `(cdr list)` 是 nil 时返回 nil。否则 cons 第一个元素和递归结果。

<details><summary>答案</summary>

```elisp
(defun my-butlast (list)
  (if (null (cdr list))
      nil
    (cons (car list) (my-butlast (cdr list)))))
```

</details>

### 题 28: my-append
拼接两个 list:

```elisp
(defun my-append (l1 l2)
  ;; 你的代码
  )

(my-append '(a b) '(c d))    ;; → (a b c d)
(my-append nil '(1 2))        ;; → (1 2)
```

**Hint**: 递归 l1——如果 l1 空,直接返回 l2。否则 cons `(car l1)` 到递归 `(cdr l1)` 和 l2 的结果。这是"复制 l1 然后接 l2"的标准模式。

<details><summary>答案</summary>

```elisp
(defun my-append (l1 l2)
  (if (null l1)
      l2
    (cons (car l1) (my-append (cdr l1) l2))))
```

</details>

### 题 29: my-reverse
反转 list:

```elisp
(defun my-reverse (list)
  ;; 你的代码
  )

(my-reverse '(1 2 3))    ;; → (3 2 1)
```

**Hint**: 反转 = 拿第一个,放到"反转剩下的"末尾。用 append 把第一个元素 (包成 list) 接在递归结果后面。注意这实现是 O(n²),高效版用累加器 (W3 学)。

<details><summary>答案</summary>

```elisp
(defun my-reverse (list)
  (if (null list)
      nil
    (append (my-reverse (cdr list))
            (list (car list)))))
```

</details>

### 题 30: my-intersection
两个 list 的交集:

```elisp
(defun my-intersection (l1 l2)
  ;; 你的代码
  )

(my-intersection '(a b c) '(b c d))    ;; → (b c)
```

**Hint**: 对 l1 递归——如果 `(car l1)` 在 l2 中 (`(member ...)`),保留它 (cons 进递归结果);否则跳过 (直接返回递归结果)。

<details><summary>答案</summary>

```elisp
(defun my-intersection (l1 l2)
  (if (null l1)
      nil
    (if (member (car l1) l2)
        (cons (car l1) (my-intersection (cdr l1) l2))
      (my-intersection (cdr l1) l2))))
```

</details>

### 题 31: my-union
两个 list 的并集 (去重):

```elisp
(defun my-union (l1 l2)
  ;; 你的代码
  )

(my-union '(a b) '(b c))    ;; → (a b c)
```

<details><summary>答案</summary>

```elisp
(defun my-union (l1 l2)
  (if (null l1)
      l2
    (if (member (car l1) l2)
        (my-union (cdr l1) l2)
      (cons (car l1) (my-union (cdr l1) l2)))))
```

</details>

### 题 32: my-flatten
展平嵌套 list:

```elisp
(defun my-flatten (tree)
  ;; 你的代码
  )

(my-flatten '(1 (2 3) (4 (5))))    ;; → (1 2 3 4 5)
```

**Hint**: 这题考"嵌套递归"——`(car tree)` 可能是 atom 也可能是子 list。如果是 list,递归展平它;如果是 atom,包成 list。append 拼接两边结果。

<details><summary>答案</summary>

```elisp
(defun my-flatten (tree)
  (if (null tree)
      nil
    (if (consp (car tree))
        (append (my-flatten (car tree))
                (my-flatten (cdr tree)))
      (cons (car tree) (my-flatten (cdr tree))))))
```

</details>

### 题 33: my-subst
替换所有等于 OLD 的为 NEW (嵌套):

```elisp
(defun my-subst (old new tree)
  ;; 你的代码
  )

(my-subst 'a 'x '(a b (a c)))    ;; → (x b (x c))
```

<details><summary>答案</summary>

```elisp
(defun my-subst (old new tree)
  (cond
   ((null tree) nil)
   ((consp (car tree))
    (cons (my-subst old new (car tree))
          (my-subst old new (cdr tree))))
   ((equal old (car tree))
    (cons new (my-subst old new (cdr tree))))
   (t
    (cons (car tree) (my-subst old new (cdr tree))))))
```

</details>

### 题 34: my-max
返回 list 中最大值:

```elisp
(defun my-max (list)
  ;; 你的代码
  )

(my-max '(3 1 4 1 5 9 2 6))    ;; → 9
```

<details><summary>答案</summary>

```elisp
(defun my-max (list)
  (if (null (cdr list))
      (car list)
    (let ((rest-max (my-max (cdr list))))
      (if (> (car list) rest-max)
          (car list)
        rest-max))))
```

</details>

### 题 35: my-remove
移除所有等于 ELT 的:

```elisp
(defun my-remove (elt list)
  ;; 你的代码
  )

(my-remove 'a '(a b a c a))    ;; → (b c)
```

<details><summary>答案</summary>

```elisp
(defun my-remove (elt list)
  (if (null list)
      nil
    (if (equal elt (car list))
        (my-remove elt (cdr list))
      (cons (car list) (my-remove elt (cdr list))))))
```

</details>

### 题 36: my-position
返回 ELT 在 LIST 的 index (0-based),找不到返回 nil:

```elisp
(defun my-position (elt list)
  ;; 你的代码
  )

(my-position 'b '(a b c))    ;; → 1
(my-position 'x '(a b c))    ;; → nil
```

<details><summary>答案</summary>

```elisp
(defun my-position (elt list)
  (cl-labels ((rec (lst idx)
                   (if (null lst)
                       nil
                     (if (equal elt (car lst))
                         idx
                       (rec (cdr lst) (1+ idx))))))
    (rec list 0)))
```

(或更简单,用 helper)

```elisp
(defun my-position (elt list)
  (defun helper (lst idx)
    (if (null lst)
        nil
      (if (equal elt (car lst))
          idx
        (helper (cdr lst) (1+ idx)))))
  (helper list 0))
```

</details>

### 题 37: my-every
检查 PRED 对所有元素都返回 t:

```elisp
(defun my-every (pred list)
  ;; 你的代码
  )

(my-every #'numberp '(1 2 3))    ;; → t
(my-every #'numberp '(1 a 3))    ;; → nil
```

<details><summary>答案</summary>

```elisp
(defun my-every (pred list)
  (if (null list)
      t
    (and (funcall pred (car list))
         (my-every pred (cdr list)))))
```

</details>

### 题 38: my-some
检查 PRED 至少对一个元素返回非 nil:

```elisp
(defun my-some (pred list)
  ;; 你的代码
  )

(my-some #'numberp '(a b 3))    ;; → t
(my-some #'numberp '(a b c))    ;; → nil
```

<details><summary>答案</summary>

```elisp
(defun my-some (pred list)
  (if (null list)
      nil
    (or (funcall pred (car list))
        (my-some pred (cdr list)))))
```

</details>

### 题 39: deep-count
数嵌套 list 里所有 atom 数量:

```elisp
(defun deep-count (tree)
  ;; 你的代码
  )

(deep-count '(1 (2 3) ((4))))    ;; → 4
(deep-count nil)                  ;; → 0
```

<details><summary>答案</summary>

```elisp
(defun deep-count (tree)
  (cond
   ((null tree) 0)
   ((consp (car tree))
    (+ (deep-count (car tree))
       (deep-count (cdr tree))))
   (t
    (1+ (deep-count (cdr tree))))))
```

</details>

### 题 40: my-copy-tree
深复制 tree:

```elisp
(defun my-copy-tree (tree)
  ;; 你的代码
  )
```

<details><summary>答案</summary>

```elisp
(defun my-copy-tree (tree)
  (if (consp tree)
      (cons (my-copy-tree (car tree))
            (my-copy-tree (cdr tree)))
    tree))
```

</details>

---

## 第六组: Buffer 操作 (5 题,30 分钟)

### 题 41: 数 buffer 字符数

```elisp
(defun my-buffer-char-count ()
  "Return total characters in current buffer."
  ;; 你的代码
  )

(my-buffer-char-count)
```

<details><summary>答案</summary>

```elisp
(defun my-buffer-char-count ()
  (buffer-size))
;; 或
(defun my-buffer-char-count ()
  (1- (point-max)))
```

</details>

### 题 42: 数 buffer 行数

```elisp
(defun my-buffer-line-count ()
  "Return total lines in current buffer."
  ;; 你的代码
  )
```

<details><summary>答案</summary>

```elisp
(defun my-buffer-line-count ()
  (line-number-at-pos (point-max)))
;; 或
(defun my-buffer-line-count ()
  (count-lines (point-min) (point-max)))
```

</details>

### 题 43: 在 point 插入当前时间

```elisp
(defun my-insert-time ()
  "Insert current time at point."
  (interactive)
  ;; 你的代码
  )
```

<details><summary>答案</summary>

```elisp
(defun my-insert-time ()
  (interactive)
  (insert (format-time-string "%H:%M:%S")))
```

</details>

### 题 44: 数 region 内的词数

```elisp
(defun my-count-words-region (start end)
  "Count words in region."
  (interactive "r")
  ;; 你的代码
  )
```

<details><summary>答案</summary>

```elisp
(defun my-count-words-region (start end)
  (interactive "r")
  (let ((count 0))
    (save-excursion
      (goto-char start)
      (while (and (< (point) end)
                  (re-search-forward "\\w+" end t))
        (setq count (1+ count))))
    (message "%d words" count)))
```

</details>

### 题 45: 颠倒 region 内字符顺序

```elisp
(defun my-reverse-region-chars (start end)
  "Reverse the order of characters in region."
  (interactive "r")
  ;; 你的代码
  )
```

<details><summary>答案</summary>

```elisp
(defun my-reverse-region-chars (start end)
  (interactive "r")
  (let ((text (buffer-substring start end)))
    (delete-region start end)
    (insert (reverse (string-to-list text)))
    ;; 注意: insert 接受 string 或 char,这里需要把 list 转 string
    ))
;; 正确版:
(defun my-reverse-region-chars (start end)
  (interactive "r")
  (let* ((text (buffer-substring start end))
         (reversed (concat (reverse (string-to-list text)))))
    (delete-region start end)
    (insert reversed)))
```

</details>

---

## 第七组: 综合题 (5 题,60 分钟)

### 题 46: 检测 list 是否为 proper

proper list: 最后 CDR 是 nil。
improper: 最后是 dotted pair 或循环。

```elisp
(defun my-proper-list-p (list)
  ;; 你的代码
  )

(my-proper-list-p '(a b c))    ;; → t
(my-proper-list-p '(a . b))    ;; → nil
(my-proper-list-p nil)         ;; → t
```

<details><summary>答案</summary>

```elisp
(defun my-proper-list-p (list)
  (cond
   ((null list) t)
   ((consp list) (my-proper-list-p (cdr list)))
   (t nil)))
```

</details>

### 题 47: 列出 list 所有 atom

```elisp
(defun my-atoms (tree)
  ;; 你的代码
  )

(my-atoms '(a (b c) (d (e) f)))    ;; → (a b c d e f)
```

<details><summary>答案</summary>

```elisp
(defun my-atoms (tree)
  (cond
   ((null tree) nil)
   ((atom tree) (list tree))
   (t (append (my-atoms (car tree))
              (my-atoms (cdr tree))))))
```

</details>

### 题 48: 替换 list 中第一个匹配

```elisp
(defun my-subst-first (old new list)
  ;; 你的代码
  )

(my-subst-first 'a 'x '(a b a c))    ;; → (x b a c)
```

<details><summary>答案</summary>

```elisp
(defun my-subst-first (old new list)
  (if (null list)
      nil
    (if (equal old (car list))
        (cons new (cdr list))
      (cons (car list) (my-subst-first old new (cdr list))))))
```

</details>

### 题 49: list 的最长前缀

返回两个 list 共同的最长前缀:

```elisp
(defun my-common-prefix (l1 l2)
  ;; 你的代码
  )

(my-common-prefix '(a b c d) '(a b x))    ;; → (a b)
```

<details><summary>答案</summary>

```elisp
(defun my-common-prefix (l1 l2)
  (if (or (null l1) (null l2)
          (not (equal (car l1) (car l2))))
      nil
    (cons (car l1)
          (my-common-prefix (cdr l1) (cdr l2)))))
```

</details>

### 题 50: 把 list 分成两半

```elisp
(defun my-split-list (list)
  ;; 返回 ((前半) (后半))
  )

(my-split-list '(a b c d))    ;; → ((a b) (c d))
(my-split-list '(a b c))      ;; → ((a b) (c))  或 ((a) (b c))
```

<details><summary>答案</summary>

```elisp
(defun my-split-list (list)
  (let ((len (length list)))
    (if (<= len 1)
        (list list nil)
      (let ((mid (/ len 2)))
        (list (butlast list (- len mid))
              (nthcdr mid list))))))
```

</details>

---

## 自测 (10 题,5 分钟)

1. `(car (cdr (cdr '(a b c d))))` → ?
2. `(cons 1 (cons 2 3))` → ?
3. `(eq 'foo 'foo)` → ?
4. `(equal "abc" "abc")` → ?
5. `(eq "abc" "abc")` → ? (string,可能不同对象)
6. 写一个函数返回 list 的第二个元素
7. 写一个函数把两个 list 交错合并
8. `(length '(1 2 . 3))` → ? (improper,会 error)
9. 解释 `(list 'a 'b)` 和 `'(a b)` 的区别
10. 解释 `'('a 'b)` 的求值结果

**答案**:
> 1. c
> 2. (1 2 . 3) (improper)
> 3. t (同一 symbol)
> 4. t (内容相同)
> 5. 通常 nil (不同 string 对象,虽然内容一样)
> 6. `(car (cdr list))` 或 `(cadr list)`
> 7.
> ```elisp
> (defun interleave (l1 l2)
>   (if (or (null l1) (null l2))
>       nil
>     (cons (car l1)
>           (cons (car l2)
>                 (interleave (cdr l1) (cdr l2))))))
> ```
> 8. error (length 要求 proper list)
> 9. (list 'a 'b) 每次创建新 list;'(a b) 是字面量,可能共享
> 10. ((quote a) (quote b)) — 因为 'a 展开为 (quote a),作为 list 元素

---

## 毕业检查

- [ ] 50 题至少完成 40
- [ ] 自测 10 题至少答对 8
- [ ] 能用递归写任意 list 操作
- [ ] 能画 cons cell 图

完成后进入 `../week-02-defun-let-if/chassell-reading.md`。
