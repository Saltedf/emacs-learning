# Concept Anchor + Exercises: Week 7

## Concept Anchor: display.texi + mule.texi

Emacs 的 display 系统是分层抽象——从字符 (codepoint) 到 face (样式) 到 display property (任意显示)。理解每层让你能从"改颜色"到"嵌图片"全栈操作。

### display.texi 用户视角

```
C-x C-+    text-scale-adjust
C-x C--    text-scale-decrease
C-x C-0    text-scale-reset
M-x face-remap-reset-face
```

这些是用户视角的 display 调整。`text-scale-adjust` 改 buffer-local 的 face height——不影响其他 buffer。

### mule.texi 用户视角

```
C-\        toggle-input-method
C-x RET f  set-buffer-file-coding-system
C-x RET c  universal-coding-system-argument
C-x =      describe-char (point 处的字符)
```

`C-x =` (`describe-char`) 是强大的工具——显示 point 处字符的所有信息: codepoint、encoding、charset、face、text properties。debug 显示问题时第一选择。

---

## Exercises: Week 7

这周练习覆盖 text property、overlay、face、search、syntax table——Module 6 capstone font-lock 部分的所有 building blocks。

### 题 1: text property

```elisp
(with-temp-buffer
  (insert "hello world")
  (put-text-property 1 6 'face 'bold)
  (buffer-string))
```

`put-text-property` 给字符 [1, 6) 加 'face 'bold 属性。buffer-string 返回时 property 跟着——`(get-text-property 3 'face)` 返回 `bold`。

### 题 2: overlay 临时

```elisp
(defun my-flash-line ()
  (let ((ov (make-overlay (line-beginning-position) (line-end-position))))
    (overlay-put ov 'face 'highlight)
    (run-with-timer 0.5 nil (lambda () (delete-overlay ov)))))
```

闪烁高亮——overlay 显示 0.5 秒,然后 timer 删除。注意 `(delete-overlay ov)`——overlay 必须删除,否则泄漏。

### 题 3: face 定义

```elisp
(defface my-warning
  '((t :foreground "red" :weight bold))
  "Warning face.")
```

`defface` 定义 face。`((t ...))` 的 `t` 是"所有 display"——你可以加条件 `(((class color) (background dark)) (:foreground "orange"))` 让深色主题用不同色。

### 题 4: 搜索 + 替换

```elisp
(defun my-replace-foo-bar ()
  (save-excursion
    (goto-char (point-min))
    (while (re-search-forward "\\bfoo\\b" nil t)
      (replace-match "bar"))))
```

`(save-excursion ...)` 保存 point——函数跑完 point 复原。`\\b` 是 word boundary——确保只替换完整 word foo,不影响 food。

### 题 5: match data

```elisp
(defun my-extract-numbers (s)
  (let (nums)
    (with-temp-buffer
      (insert s)
      (goto-char (point-min))
      (while (re-search-forward "[0-9]+" nil t)
        (push (string-to-number (match-string 0)) nums)))
    (nreverse nums)))
```

`(match-string 0)` 取最后匹配的文本。循环 search,每次 push,最后 nreverse。

### 题 6: thing-at-point

```elisp
(defun my-copy-word ()
  (interactive)
  (let ((word (thing-at-point 'word)))
    (when word
      (kill-new word)
      (message "Copied: %s" word))))
```

`thing-at-point 'word` 智能取 point 处的 word——你不需要自己写"找 word 边界"。

### 题 7: syntax table

```elisp
(defvar my-syntax-table
  (let ((st (make-syntax-table)))
    (modify-syntax-entry ?_ "w" st)
    st))
```

`_` 设为 word class——让 `my_var` 被识别为一个 word (而非 my、_、var 三个)。

### 题 8: font-lock

```elisp
(font-lock-add-keywords nil
  '((("\\<\\(TODO\\|FIXME\\)\\>" 1 'font-lock-warning-face t))))
```

让所有 buffer 的 TODO/FIXME 高亮成 warning 色。第一参数 nil 是"当前 mode",你可以 `'python-mode` 等。

### 题 9: image display

```elisp
(insert-image (create-image "~/pic.png" 'png nil))
```

`create-image` 创建 image 对象,`insert-image` 插入 buffer (字符显示为图片)。这是 Emacs 嵌图的基础。

### 题 10: 显示属性

```elisp
(put-text-property 1 2 'display '(height 3.0))
```

`'display '(height 3.0)` 让字符显示 3 倍高度——用于标题、强调。

### 题 11: char info

```elisp
(describe-char)                ; point 处的字符详情
```

显示 char 的所有 info——codepoint、charset、encoding、face、properties。

### 题 12: input method

```elisp
(toggle-input-method)          ; C-\
```

切换输入法——`set-input-method` 设具体的。

---

## 自测

1. text property 和 overlay 哪个持久?
2. 怎么定义 face?
3. `re-search-forward` 找不到默认?
4. thing-at-point 怎么用?
5. syntax table 控制什么?

**答案**:
> 1. text property 持久;overlay 临时
> 2. defface
> 3. 报错 (除非 NOERROR)
> 4. (thing-at-point 'word)
> 5. 字符类别、bracket、comment

---

进入 `../week-08-processes-package/`。
