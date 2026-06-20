# Concept Anchor: Editor Manual (Week 4)

> 替代 `emacs-manual-30.2/regs.texi` (复习) + `text.texi` 的"字符串处理"
> 这周你的 lambda + mapcar + 字符串操作,对应编辑器的"批量文本变换"

---

## 1. 字符串处理 = buffer 处理的微观

### 1.1 双层视角

Emacs 有两个"文本世界": buffer (可见) 和 string (内存中)。两者操作**对称**——理解一个,另一个就懂。这是 Emacs 设计的优雅之处。

```elisp
;; buffer 级别 (visible):
(buffer-substring START END)        ;; 取文本
(delete-region START END)           ;; 删
(insert "text")                     ;; 插

;; string 级别 (invisible):
(substring STR 0 5)                 ;; 取子串
(replace-regexp-in-string ...)      ;; 替换
(concat STR1 STR2)                  ;; 拼接
```

buffer 操作和 string 操作**对称**。
理解一个,另一个就懂。

为什么这两个对称?因为 buffer 内部就是"有 text property 的 string"。Elisp 给 buffer 提供 buffer-only 操作 (point、mark、narrow),给 string 提供 string-only 操作 (concat、format),但底层都是字符序列。

实战意义: 你写"处理 region 文本"的函数,可以两种写法——直接操作 buffer (用 save-excursion),或者取出文本到 string,处理完再放回。前者高效 (不复制),后者清晰 (适合复杂变换)。

### 1.2 实战: region 操作 = string 操作

最常用的模式: 取 region → 转 string → 处理 → 放回。这种模式让你把 string 操作的便利带到 buffer 操作:

```elisp
(defun my-upper-region (start end)
  (interactive "r")
  (let* ((text (buffer-substring start end))
         (new (upcase text)))
    (delete-region start end)
    (goto-char start)
    (insert new)))
```

模式: buffer → string → 处理 → 回 buffer。

这个模式的优点: 你处理的是 string,所有 string 函数 (upcase, replace-regexp-in-string, split-string) 都能用。如果直接操作 buffer,你要 save-excursion + re-search,更复杂。

### 1.3 更通用

抽象出"把任意 string 函数应用到 region":

```elisp
(defun my-transform-region (fn start end)
  "Apply FN to region text."
  (interactive "*xTransform function: \nr")
  (let* ((text (buffer-substring start end))
         (new (funcall fn text)))
    (delete-region start end)
    (insert new)
    (goto-char start)
    (forward-char (length new))))

(defun my-md5-region (start end)
  (interactive "r")
  (my-transform-region (lambda (s) (md5 s)) start end))
```

这是"高阶函数"的实战——`my-transform-region` 接受任意函数 fn,把它应用到 region。这种"通用工具 + 具体函数"的模式,让你加新功能 (my-md5-region, my-base64-region, ...) 时只需写一行 lambda。

---

## 2. Lambda + 编辑器

### 2.1 hook 是 lambda 的常见场景

hook 是 Emacs 的"扩展点"——在某些事件发生时,Emacs 调用注册的函数。lambda 在 hook 里大显身手:

```elisp
(add-hook 'before-save-hook
          (lambda ()
            (when (derived-mode-p 'python-mode)
              (message "About to save Python file"))))
```

hook 接受 lambda (或函数符号)。

为什么用 lambda 不用 defun?因为这种 hook 是"一次性、特定场景"的——给它命名反而污染命名空间。lambda 是"匿名 + 即用即弃"。

### 2.2 sort-with-key

sort 是日常任务,但 Emacs 内置 sort 只能按元素本身。如果你想"按元素的某个属性"排序,lambda 派上用场:

```elisp
(defun sort-lines-by-length (reverse)
  "Sort lines in region by length."
  (interactive "P")
  (save-excursion
    (save-restriction
      (narrow-to-region (region-beginning) (region-end))
      (goto-char (point-min))
      (sort-subr reverse
                 'forward-line
                 'end-of-line
                 (lambda () (length (buffer-substring
                                     (line-beginning-position)
                                     (line-end-position))))))))
```

`sort-subr` 接受 record 函数和 key 函数,key 函数就是 lambda。

key 函数返回"用来排序的值"——这里返回行的长度。sort-subr 用这个值排序。

---

## 3. mapcar 与编辑器批处理

mapcar 不只是处理 list——它能让 Emacs 批量操作。三个例子:

### 3.1 操作一组 buffer

你想关掉所有 Dired buffer——一个一个找太累。用 dolist + lambda:

```elisp
(defun my-close-all-dired-buffers ()
  "Close all Dired buffers."
  (interactive)
  (let ((count 0))
    (dolist (buf (buffer-list))
      (when (with-current-buffer buf (derived-mode-p 'dired-mode))
        (kill-buffer buf)
        (setq count (1+ count))))
    (message "Closed %d dired buffers" count)))
```

`(buffer-list)` 返回所有 buffer。dolist 遍历,when 过滤 Dired,kill-buffer 关闭。

### 3.2 操作一组文件

```elisp
(defun my-touch-all-el-files (dir)
  "Update modification time of all .el files in DIR."
  (interactive "DDirectory: ")
  (dolist (file (directory-files dir t "\\.el\\'"))
    (set-file-times file)
    (message "Touched %s" file)))
```

`(directory-files dir t PATTERN)` 返回匹配 PATTERN 的文件 list。dolist 遍历,set-file-times 更新时间戳。

### 3.3 操作一组 buffer

```elisp
(defun my-format-all-python-buffers ()
  (interactive)
  (dolist (buf (buffer-list))
    (when (with-current-buffer buf (derived-mode-p 'python-mode))
      (with-current-buffer buf
        (indent-region (point-min) (point-max))
        (save-buffer)))))
```

这种"批量操作"是 Emacs 的杀手锏——你写 5 行 Elisp,就能对几十个文件做任意操作。其他编辑器 (VSCode) 做这种事要写扩展,Emacs 几行就行。

---

## 4. 正则与编辑器

### 4.1 user 视角 vs Lisp 视角

Module 1 学了 user 视角 (`C-s`/`M-%`)。
Lisp 视角:

```elisp
(re-search-forward PATTERN)
(replace-match REPLACEMENT)
(perform-replace ...)         ; query-replace 内部
```

### 4.2 写一个 query-replace

```elisp
(defun my-replace-foo-with-bar ()
  "Replace all 'foo' with 'bar'."
  (interactive)
  (save-excursion
    (goto-char (point-min))
    (while (re-search-forward "\\bfoo\\b" nil t)
      (replace-match "bar"))))
```

注意: 这**不**交互 (不问 y/n)。
要交互用 `(perform-replace OLD NEW t nil nil)`。

### 4.3 高亮匹配

```elisp
(defun my-highlight-todos ()
  "Highlight TODO and FIXME in current buffer."
  (interactive)
  (highlight-regexp "\\b\\(TODO\\|FIXME\\)\\b" 'hi-red-b))

(my-highlight-todos)
```

### 4.4 收集匹配

```elisp
(defun my-collect-matches (pattern)
  "Collect all PATTERN matches in buffer."
  (let (matches)
    (save-excursion
      (goto-char (point-min))
      (while (re-search-forward pattern nil t)
        (push (match-string 0) matches)))
    (nreverse matches)))

(my-collect-matches "defun \\([a-z-]+\\)")
;; → ("defun foo" "defun bar" ...)
```

---

## 5. Sequences 概念

### 5.1 Emacs 的 sequence 类型

```elisp
sequencep                  ;; 通用检查
(listp)                    ;; list
(arrayp)                   ;; array (含 vector, string, char-table, bool-vector)
```

list 和 array 都是 sequence。

### 5.2 通用 sequence 函数

```elisp
(length SEQUENCE)
(elt SEQUENCE INDEX)
(copy-sequence SEQUENCE)
(reverse SEQUENCE)
(nreverse SEQUENCE)        ; destructive
(sort SEQUENCE PREDICATE)
```

### 5.3 mapcar vs mapc 通用版

```elisp
(mapcar FN SEQ)            ;; for list
(mapcar FN VEC)            ;; also works for vector! returns list
```

实际 `mapcar` 接受任何 sequence。

### 5.4 cl-loop 处理 sequence

```elisp
(require 'cl-lib)

(cl-loop for x across "hello" collect x)
;; → (104 101 108 108 111)

(cl-loop for x in "hello" collect x)    ;; ← 这不对,"in" 只 for list
```

`across` for vector/string; `in` for list; `on` for sublist; `for x being the elements of SEQUENCE` 通用。

---

## 6. 实战: 写一个 region 操作库

```elisp
;;; my-region-utils.el -*- lexical-binding: t; -*-

(defun my-region-apply (fn &optional start end)
  "Apply FN to region text."
  (if (use-region-p)
      (let* ((start (or start (region-beginning)))
             (end (or end (region-end)))
             (text (buffer-substring start end))
             (new (funcall fn text)))
        (delete-region start end)
        (goto-char start)
        (insert new))
    (error "No region active")))

(defun my-region-upcase (start end)
  (interactive "r")
  (my-region-apply #'upcase start end))

(defun my-region-downcase (start end)
  (interactive "r")
  (my-region-apply #'downcase start end))

(defun my-region-trim (start end)
  (interactive "r")
  (my-region-apply #'string-trim start end))

(defun my-region-reverse-lines (start end)
  (interactive "r")
  (my-region-apply
   (lambda (text)
     (mapconcat #'identity
                (nreverse (split-string text "\n"))
                "\n"))
   start end))

(defun my-region-base64-encode (start end)
  (interactive "r")
  (my-region-apply #'base64-encode-string start end))

(provide 'my-region-utils)
```

---

## 7. 自测

1. 写 `(my-region-md5)` 把 region 内容替换为 md5
2. 写 `(my-collect-buffers-by-mode MODE)` 返回所有该 mode 的 buffer
3. 写 `(my-format-all-matching REGEXP)` 格式化所有匹配 REGEXP 的 buffer
4. 解释 `(mapcar FN LIST)` 和 `(mapc FN LIST)` 区别

**答案**:
> 1. 
> ```elisp
> (defun my-region-md5 (start end)
>   (interactive "r")
>   (let* ((text (buffer-substring start end))
>          (hash (md5 text)))
>     (delete-region start end)
>     (insert hash)))
> ```
> 2. 
> ```elisp
> (defun my-collect-buffers-by-mode (mode)
>   (cl-remove-if-not
>    (lambda (buf)
>      (with-current-buffer buf
>        (derived-mode-p mode)))
>    (buffer-list)))
> ```
> 3. 
> ```elisp
> (defun my-format-all-matching (regexp)
>   (dolist (buf (buffer-list))
>     (with-current-buffer buf
>       (when (and (buffer-file-name)
>                  (string-match regexp (buffer-file-name)))
>         (indent-region (point-min) (point-max))))))
> ```
> 4. mapcar 收集结果返回新 list;mapc 不收集,返回原 list (副作用用)

---

## 8. 下一步

进入 `exercises.md`。
