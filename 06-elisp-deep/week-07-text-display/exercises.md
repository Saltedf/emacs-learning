# Exercises: Week 7 Text + Display + Search + Syntax

> 12 题

这周练习考 text property、overlay、face、search、syntax table——Module 6 capstone font-lock 部分的所有 building blocks。

---

### 题 1: text property

```elisp
(with-temp-buffer
  (insert "hello world")
  (put-text-property 1 6 'face 'bold)
  (buffer-string))
```

`put-text-property` 给 [1, 6) 范围字符加 face 属性。`(get-text-property 3 'face)` 在那个 buffer 返回 `bold`。

### 题 2: overlay 临时高亮

```elisp
(defun my-highlight-line ()
  (let ((ov (make-overlay (line-beginning-position) (line-end-position))))
    (overlay-put ov 'face 'highlight)
    (run-with-timer 0.5 nil (lambda () (delete-overlay ov)))))
```

闪烁高亮——overlay 创建、显示、0.5 秒后删除。**必须 delete-overlay**,否则泄漏 (overlay 持续占内存)。

### 题 3: defface

```elisp
(defface my-todo-face
  '((t :foreground "yellow" :weight bold))
  "Face for TODO items.")
```

`defface` 定义新 face。`((t ...))` 是"所有 display"——可以加条件 (如 dark theme 用不同色)。

### 题 4: 搜索

```elisp
(defun my-find-all-todos ()
  (let (todos)
    (save-excursion
      (goto-char (point-min))
      (while (re-search-forward "TODO: \\(.*\\)" nil t)
        (push (match-string 1) todos)))
    (nreverse todos)))
```

`(match-string 1)` 取 regex 第 1 个 capture group——`\\(.*\\)` 捕获 TODO 后的内容。这是"提取所有匹配" 的标准模式。

### 题 5: 替换

```elisp
(defun my-replace-py-js ()
  (save-excursion
    (goto-char (point-min))
    (while (re-search-forward "\\.py\\b" nil t)
      (replace-match ".js"))))
```

`(save-excursion ...)` 保存 point——函数跑完 point 复原。

### 题 6: thing-at-point URL

```elisp
(defun my-open-url-at-point ()
  (interactive)
  (let ((url (thing-at-point 'url)))
    (when url
      (browse-url url))))
```

`thing-at-point 'url` 智能识别 URL——你不需要自己写 URL regex。

### 题 7: bounds-of-thing

```elisp
(defun my-kill-word-at-point ()
  (interactive)
  (let ((bounds (bounds-of-thing-at-point 'word)))
    (when bounds
      (kill-region (car bounds) (cdr bounds)))))
```

`bounds-of-thing-at-point` 返回 (beg . end) cons。`kill-region` 删那一段到 kill-ring。

### 题 8: syntax table

```elisp
(defvar my-syntax-table
  (let ((st (make-syntax-table)))
    (modify-syntax-entry ?_ "w" st)
    (modify-syntax-entry ?# "_" st)
    (modify-syntax-entry ?/ ". 124b" st)
    (modify-syntax-entry ?* ". 23" st)
    (modify-syntax-entry ?\n "> b" st)
    st))
```

这是 C 风格注释的 syntax table。`?/ ". 124b"` 表示 `/` 是 punctuation + comment start first char (1) + comment second char (2) + comment end second char (4) + b style。

### 题 9: font-lock

```elisp
(font-lock-add-keywords
 nil
 '(("DONE" . font-lock-string-face)
   ("TODO" . font-lock-warning-face)
   ("FIXME" . font-lock-warning-face)))
```

让 DONE/TODO/FIXME 高亮。第一参数 nil 是"当前 mode"。

### 题 10: char

```elisp
(char-syntax ?a)               ; → 'w'
(char-syntax ?.)               ; → '.'
```

`(char-syntax ?a)` 返回 `'w'` (字符代码 119,不是 symbol)。比较时用 character literal: `(eq (char-syntax ?a) ?w)`。

### 题 11: image

```elisp
(insert-image (create-image "/path/to.png" 'png nil :width 200))
```

`:width 200` 限制显示宽度。Emacs 嵌图的基础。

### 题 12: display property

```elisp
(put-text-property 1 3 'display '(height 2.0))
```

让 [1, 3) 字符显示 2 倍高度。这是"特殊显示"的基础——你可以做标题、隐藏内容、嵌图。

---

## 自测

1. text property 怎么 put?
2. overlay 怎么创建?
3. 怎么查找 buffer 中所有 "TODO"?
4. thing-at-point 'url 干啥?
5. font-lock-add-keywords 怎么用?

**答案**:
> 1. (put-text-property START END PROP VALUE)
> 2. (make-overlay START END)
> 3. while + re-search-forward
> 4. 返回光标处的 URL
> 5. (font-lock-add-keywords nil '((PATTERN . FACE)))

---

进入 `../week-08-processes-package/`。
