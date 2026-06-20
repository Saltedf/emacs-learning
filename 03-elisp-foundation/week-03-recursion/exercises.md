# Exercises: Week 3 Recursion + Conditionals

> 20 题练习,递归 + 闭包 + 循环

这周是 Module 3 的"分水岭"——递归思维。前 10 题 (cond/while/dolist) 是热身,后 10 题 (递归 + 闭包) 才是核心。

递归的"诀窍"是先想 base case ("什么时候停?"),再想 recursive case ("怎么把问题变小?")。两个 case 都想清楚,代码就出来了。

**方法**: 每道题先在纸上画递归过程——`(my-length '(a b c))` → `(1+ (my-length '(b c)))` → ... 一旦你能在脑子里"trace"递归,写代码就简单了。

---

## 第一组: cond / while (5 题,30 分钟)

这组训练条件表达式和循环。比 W2 的 if 更"重型"——适合多分支和重复操作。

### 题 1: my-day-name
数字 1-7 → 星期名:

```elisp
(defun my-day-name (n)
  ;; 1=Monday, 7=Sunday
  )

(my-day-name 1)    ;; → "Monday"
(my-day-name 7)    ;; → "Sunday"
(my-day-name 9)    ;; → nil
```

<details><summary>答案</summary>

```elisp
(defun my-day-name (n)
  (cond
   ((= n 1) "Monday")
   ((= n 2) "Tuesday")
   ((= n 3) "Wednesday")
   ((= n 4) "Thursday")
   ((= n 5) "Friday")
   ((= n 6) "Saturday")
   ((= n 7) "Sunday")))
```

</details>

### 题 2: my-categorize
按年龄分组:

```elisp
(defun my-age-group (age)
  ;; < 13 child, 13-19 teen, 20-64 adult, 65+ senior
  )
```

<details><summary>答案</summary>

```elisp
(defun my-age-group (age)
  (cond
   ((< age 13) 'child)
   ((< age 20) 'teen)
   ((< age 65) 'adult)
   (t 'senior)))
```

</details>

### 题 3: my-count-down-while
用 while 打印 5 4 3 2 1:

```elisp
(defun my-count-down ()
  ;; 用 while
  )
```

<details><summary>答案</summary>

```elisp
(defun my-count-down ()
  (let ((n 5))
    (while (> n 0)
      (message "%d" n)
      (setq n (1- n)))))
```

</details>

### 题 4: my-list-with-while
用 while 把 list 元素翻倍:

```elisp
(defun my-double-list (list)
  ;; 用 while
  )

(my-double-list '(1 2 3))    ;; → (2 4 6)
```

<details><summary>答案</summary>

```elisp
(defun my-double-list (list)
  (let ((result nil))
    (while list
      (push (* 2 (car list)) result)
      (setq list (cdr list)))
    (nreverse result)))
```

</details>

### 题 5: my-sum-1-to-n
```elisp
(defun my-sum-to (n)
  ;; 1+2+...+n,用 while
  )

(my-sum-to 100)    ;; → 5050
```

<details><summary>答案</summary>

```elisp
(defun my-sum-to (n)
  (let ((sum 0)
        (i 1))
    (while (<= i n)
      (setq sum (+ sum i)
            i (1+ i)))
    sum))
```

</details>

---

## 第二组: dolist / dotimes (5 题,30 分钟)

### 题 6: my-print-each
```elisp
(defun my-print-each (list)
  ;; 用 dolist,每个元素 message
  )
```

<details><summary>答案</summary>

```elisp
(defun my-print-each (list)
  (dolist (x list)
    (message "%s" x)))
```

</details>

### 题 7: my-list-squared
```elisp
(defun my-list-squared (n)
  ;; 返回 (0 1 4 9 16 ... (n-1)^2)
  ;; 用 dotimes
  )

(my-list-squared 5)    ;; → (0 1 4 9 16)
```

<details><summary>答案</summary>

```elisp
(defun my-list-squared (n)
  (let ((result nil))
    (dotimes (i n)
      (push (* i i) result))
    (nreverse result)))
```

</details>

### 题 8: my-dolist-sum
```elisp
(defun my-sum-list (list)
  ;; 用 dolist
  )

(my-sum-list '(1 2 3 4 5))    ;; → 15
```

<details><summary>答案</summary>

```elisp
(defun my-sum-list (list)
  (let ((sum 0))
    (dolist (x list sum)
      (setq sum (+ sum x)))))
```

</details>

### 题 9: my-count-evens
```elisp
(defun my-count-evens (list)
  ;; 用 dolist
  )

(my-count-evens '(1 2 3 4 5 6))    ;; → 3
```

<details><summary>答案</summary>

```elisp
(defun my-count-evens (list)
  (let ((count 0))
    (dolist (x list count)
      (when (zerop (mod x 2))
        (setq count (1+ count))))))
```

</details>

### 题 10: my-build-string
```elisp
(defun my-build-string (n char)
  ;; 用 dotimes,返回 n 个 char 的字符串
  )

(my-build-string 5 ?a)    ;; → "aaaaa"
```

<details><summary>答案</summary>

```elisp
(defun my-build-string (n char)
  (let ((result ""))
    (dotimes (_ n result)
      (setq result (concat result (string char))))))
```

</details>

---

## 第三组: 递归 (5 题,40 分钟)

这一组训练递归思维。每题要求递归实现 (不用 while/loop)。这是 W3 的核心——做完这组,你就掌握了递归。

模板: `(if (BASE-CASE?) BASE-VALUE (COMBINE (PROCESS (car list)) (RECURSE (cdr list))))`。

### 题 11: my-length-recursive
```elisp
(defun my-length (list)
  ;; 递归
  )
```

**Hint**: Base case: list 是 nil,长度 0。Recursive: 1 + (my-length (cdr list))。

<details><summary>答案</summary>

```elisp
(defun my-length (list)
  (if (null list)
      0
    (1+ (my-length (cdr list)))))
```

</details>

### 题 12: my-member-recursive
```elisp
(defun my-member (elt list)
  ;; 递归
  )

(my-member 'b '(a b c))    ;; → (b c)
```

**Hint**: Base case: list 空 → nil。Recursive: 如果 (car list) 等于 elt,返回整个 list (从 elt 开始);否则递归 (cdr list)。

<details><summary>答案</summary>

```elisp
(defun my-member (elt list)
  (cond
   ((null list) nil)
   ((equal elt (car list)) list)
   (t (my-member elt (cdr list)))))
```

</details>

### 题 13: my-assoc
alist 查找:

```elisp
(defun my-assoc (key alist)
  ;; (my-assoc 'b '((a 1) (b 2) (c 3))) → (b 2)
  )

(my-assoc 'b '((a 1) (b 2) (c 3)))    ;; → (b 2)
```

**Hint**: alist 是 list of pairs,每个 pair 是 `(key value)`。Base: alist 空 → nil。Recursive: 如果 `(car (car alist))` 等于 key (即第一个 pair 的 car),返回整个 pair;否则递归 `(cdr alist)`。

<details><summary>答案</summary>

```elisp
(defun my-assoc (key alist)
  (cond
   ((null alist) nil)
   ((equal key (car (car alist))) (car alist))
   (t (my-assoc key (cdr alist)))))
```

</details>

### 题 14: my-tree-sum
```elisp
(defun my-tree-sum (tree)
  ;; (my-tree-sum '(1 (2 3) (4 (5 6)))) → 21
  )

(my-tree-sum '(1 (2 3) (4 (5 6))))    ;; → 21
```

**Hint**: 嵌套递归。Cond 三种情况: null → 0;numberp → tree 本身;consp → 递归 car + 递归 cdr。这是处理"任意嵌套 list"的标准模板。

<details><summary>答案</summary>

```elisp
(defun my-tree-sum (tree)
  (cond
   ((null tree) 0)
   ((numberp tree) tree)
   ((consp tree)
    (+ (my-tree-sum (car tree))
       (my-tree-sum (cdr tree))))
   (t 0)))
```

</details>

### 题 15: my-deep-find
```elisp
(defun my-deep-find (elt tree)
  ;; 嵌套 list 里找 elt
  )

(my-deep-find 'c '(a (b c) (d (e))))    ;; → t
(my-deep-find 'x '(a (b c) (d (e))))    ;; → nil
```

<details><summary>答案</summary>

```elisp
(defun my-deep-find (elt tree)
  (cond
   ((null tree) nil)
   ((equal elt tree) t)
   ((consp tree)
    (or (my-deep-find elt (car tree))
        (my-deep-find elt (cdr tree))))
   (t nil)))
```

</details>

---

## 第四组: 闭包 + Lexical (5 题,40 分钟)

**前提**: file 头加 `;; -*- lexical-binding: t; -*-`

这组训练闭包——函数式编程的精华。闭包让函数"持有私有状态",这是模拟对象、做 memoization、写状态机的基础。

### 题 16: make-counter
```elisp
(defun make-counter ()
  ;; 返回一个每次调用 +1 的 lambda
  )
```

**Hint**: 用 let 创建局部 count=0,返回 lambda 在内部 `(setq count (1+ count))`。lambda 捕获 count binding,即使 let 退出,count 仍活。

<details><summary>答案</summary>

```elisp
(defun make-counter ()
  (let ((count 0))
    (lambda ()
      (setq count (1+ count))
      count)))
```

</details>

### 题 17: make-adder
```elisp
(defun make-adder (n)
  ;; 返回 (lambda (x) (+ x n))
  )
```

**Hint**: 直接返回 `(lambda (x) (+ x n))`。lambda 捕获 n。这是"部分应用"的简单例子。

<details><summary>答案</summary>

```elisp
(defun make-adder (n)
  (lambda (x)
    (+ x n)))
```

</details>

### 题 18: make-stack
做一个栈 (push/pop):

```elisp
(defun make-stack ()
  ;; 返回 lambda 接受 op 和参数:
  ;;   ('push X): 压入 X
  ;;   ('pop): 弹出 (返回 nil if empty)
  ;;   ('peek): 看栈顶
  ;;   ('size): 返回大小
  )
```

<details><summary>答案</summary>

```elisp
(defun make-stack ()
  (let ((items nil))
    (lambda (op &optional x)
      (pcase op
        ('push (push x items))
        ('pop (pop items))
        ('peek (car items))
        ('size (length items))))))
```

</details>

### 题 19: make-cache
```elisp
(defun make-cache ()
  ;; 返回 ('get KEY) / ('put KEY VAL)
  )
```

<details><summary>答案</summary>

```elisp
(defun make-cache ()
  (let ((table (make-hash-table :test 'equal)))
    (lambda (op &rest args)
      (pcase op
        ('get (gethash (car args) table))
        ('put (puthash (car args) (cadr args) table))
        ('clear (clrhash table))))))
```

</details>

### 题 20: 用 closure 实现私有
做一个"账户":

```elisp
(defun make-account (initial)
  ;; ('deposit X) / ('withdraw X) / ('balance)
  )
```

<details><summary>答案</summary>

```elisp
(defun make-account (initial)
  (let ((balance initial))
    (lambda (op &optional amount)
      (pcase op
        ('deposit (setq balance (+ balance amount)) balance)
        ('withdraw
         (if (>= balance amount)
             (progn (setq balance (- balance amount)) balance)
           (error "Insufficient funds")))
        ('balance balance)))))
```

</details>

---

## 自测

1. 写 `(my-factorial N)` 递归
2. 写 `(my-factorial N)` 用 while
3. 什么是闭包? 例子
4. lexical vs dynamic binding?
5. cond 和 if 区别?
6. 用 dolist 写 `(my-count-evens LIST)`

**答案**:
> 1. 
> ```elisp
> (defun my-factorial (n)
>   (if (zerop n) 1
>     (* n (my-factorial (1- n)))))
> ```
> 2. 
> ```elisp
> (defun my-factorial (n)
>   (let ((result 1))
>     (while (> n 0)
>       (setq result (* result n)
>             n (1- n)))
>     result))
> ```
> 3. 函数 + 定义时环境;如 `make-counter` 闭包持有 count
> 4. lexical 定义时捕获,dynamic 调用时查找
> 5. cond 多分支;if 二选一,then 一个 form
> 6.
> ```elisp
> (defun my-count-evens (list)
>   (let ((count 0))
>     (dolist (x list count)
>       (when (zerop (mod x 2))
>         (setq count (1+ count))))))
> ```

---

## 毕业检查

- [ ] 20 题至少完成 16
- [ ] 自测 6 题至少答对 5
- [ ] 能用递归处理任意嵌套结构
- [ ] 能用闭包做状态封装

完成后进入 `../week-04-lambda-mapcar/`。
