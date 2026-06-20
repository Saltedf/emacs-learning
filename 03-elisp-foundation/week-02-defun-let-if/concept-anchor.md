# Concept Anchor: Editor Manual (Week 2)

> 替代 `emacs-manual-30.2/text.texi` 的核心概念
> 这周你的 defun + buffer 函数,对应编辑器的"文本操作"

---

## 1. text.texi 的概念锚点

### 1.1 Words / Sentences / Paragraphs

Module 1 你学了用 `M-f`/`M-b` 移动 word。那是 user 视角——按键 → 移动。

这周你学 Lisp: 每个键位背后都是函数,你可以调用它们:

```elisp
(forward-word 1)              ;; 等价 M-f
(backward-word 1)             ;; 等价 M-b
(forward-sentence 1)          ;; 等价 M-e
(backward-sentence 1)
(forward-paragraph 1)
(backward-paragraph 1)
```

**关键**: 编辑器的"移动命令"就是 Lisp 函数。

理解这点的意义: 你可以在自己代码里调用这些函数,组合成新命令。比如 `(defun my-jump-to-next-function () (re-search-forward "^(defun") (forward-line -1) (back-to-indentation))` ——把几个移动函数串起来,做"跳到下一个 defun"。

### 1.2 Words 的定义

什么是 "word"? 由 syntax table 决定。这是 Emacs 的精妙设计——"word" 不是固定的字符集,而是 buffer-local 的语法表定义。

```elisp
(modify-syntax-entry ?_ "w")  ; _ 算 word 字符
```

syntax table 是 buffer-local,不同 mode 不同:
- `prog-mode`: word = 字母数字下划线 (因为编程标识符有 `_`)
- `text-mode`: word = 字母数字 (因为英语单词不含 `_`)

(Module 6 W7 详细讲)

为什么 buffer-local?因为不同 mode 的"word"概念不同。Python 的 `my_var` 是一个 word (含 `_`),英语 `it's` 是两个 word (`'` 分隔)。Syntax table 让 Emacs 灵活适应各种语言。

### 1.3 Sentences

默认 sentence 由 `.?!` 后跟 2 个空格或换行定义。
你可以改:

```elisp
(setq sentence-end "[.!?][]\"')]*\\($\\| \\| \\)[
\n]*")
```

注意 `sentence-end-double-space` 变量控制是否需要双空格。

为什么默认双空格?这是打字机时代的传统——句号后两个空格更易读。现代英文写作用单空格,所以很多人改这个变量。但 Elisp 默认保留双空格——向后兼容。

### 1.4 Paragraphs

段落由空行分隔。
`(forward-paragraph)` 移到下一段。

### 1.5 Pages

`C-x ]` 和 `C-x [` 移动 page。
page 由 `\f` (form feed, `C-q C-l`) 分隔。

某些格式 (nroff, texinfo) 用 page 分文档。现代编辑器很少用 page 概念,但 Emacs 保留——它对长文档 (man pages、texinfo) 仍有用。

---

## 2. 你的 defun 对应编辑器

### 2.1 自定义命令 = 编辑器扩展

```elisp
(defun my-foo ()
  (interactive)
  ...)
(global-set-key (kbd "C-c f") 'my-foo)
```

这等于"给编辑器加一个新命令 + 绑键"。
没有"插件"概念,这就是 Lisp 函数。

### 2.2 一些实用自定义

```elisp
(defun my-duplicate-line ()
  "Duplicate current line below."
  (interactive)
  (let ((content (buffer-substring
                  (line-beginning-position)
                  (line-end-position))))
    (end-of-line)
    (newline)
    (insert content)
    (beginning-of-line)))
(global-set-key (kbd "C-c d") 'my-duplicate-line)
```

```elisp
(defun my-move-line-up ()
  "Move current line up."
  (interactive)
  (let ((col (current-column)))
    (save-excursion
      (transpose-lines 1))
    (forward-line -1)
    (move-to-column col)))
(global-set-key (kbd "M-<up>") 'my-move-line-up)
```

```elisp
(defun my-move-line-down ()
  "Move current line down."
  (interactive)
  (let ((col (current-column)))
    (save-excursion
      (forward-line 1)
      (transpose-lines 1))
    (forward-line 1)
    (move-to-column col)))
(global-set-key (kbd "M-<down>") 'my-move-line-down)
```

### 2.3 转换 region

```elisp
(defun my-toggle-letter-case (start end)
  "Toggle letter case in region."
  (interactive "r")
  (let ((region (buffer-substring start end)))
    (delete-region start end)
    (goto-char start)
    (insert
     (cond
      ((string= region (upcase region)) (downcase region))
      ((string= region (downcase region)) (capitalize region))
      (t (upcase region))))))
```

---

## 3. editor 的 Auto-Fill / Fill

### 3.1 Fill 概念

"fill" = 自动换行,让每行不超过 `fill-column`:

```elisp
(fill-paragraph)            ;; 当前段
(fill-region START END)
(fill-region-as-paragraph START END)
```

`M-q` 是 `fill-paragraph`。

### 3.2 Auto-fill mode

```elisp
(add-hook 'text-mode-hook #'turn-on-auto-fill)
```

minor mode,你输入时自动 fill。

### 3.3 Lisp 实现

```elisp
(defun my-fill-long-lines (max)
  "Fill lines longer than MAX chars in buffer."
  (interactive "nMax line length: ")
  (save-excursion
    (goto-char (point-min))
    (while (re-search-forward (format "^.\{%d,\\}$" max) nil t)
      (fill-paragraph))))
```

---

## 4. editor 的 Undo

### 4.1 Undo 数据结构

```elisp
buffer-undo-list              ;; 一个 list,每个元素是一个 undo record
```

每个 record 是:
- `(POS . OLD)` 修改了 POS 处的字符 (old → new)
- `(t POS . NEW)` 插入
- `(nil PROPERTY VALUE POS . OLD)` text property 改变
- `nil` 边界 (一个 undo 单位)

### 4.2 用 Lisp 撤销

```elisp
(undo)              ;; 撤销一次
(undo-only)         ;; 只撤销,不 redo
(undo-redo)         ;; redo (Emacs 28+)
```

### 4.3 自定义 undo 行为

```elisp
;; 关闭某个 buffer 的 undo
(buffer-disable-undo)

;; 限制 undo 大小
(setq undo-limit 80000)        ; 80KB
(setq undo-strong-limit 120000) ; 强 undo 限制
```

---

## 5. 你的代码里的实用工具

### 5.1 在文件开头插入时间戳

```elisp
(defun my-insert-timestamp-header ()
  "Insert timestamp at top of file."
  (interactive)
  (save-excursion
    (goto-char (point-min))
    (insert (format ";; Last modified: %s\n"
                    (format-time-string "%Y-%m-%d %H:%M")))))

(add-hook 'before-save-hook #'my-insert-timestamp-header)
```

### 5.2 自动格式化保存

```elisp
(defun my-format-on-save ()
  "Format buffer before saving (Elisp only)."
  (when (derived-mode-p 'emacs-lisp-mode)
    (indent-region (point-min) (point-max))))

(add-hook 'before-save-hook #'my-format-on-save)
```

### 5.3 计算 region 字符

```elisp
(defun my-region-stats (start end)
  "Show region stats."
  (interactive "r")
  (let* ((text (buffer-substring start end))
         (chars (length text))
         (words (length (split-string text)))
         (lines (count-lines start end)))
    (message "%d chars, %d words, %d lines" chars words lines)))
```

### 5.4 buffer 字符统计

```elisp
(defun my-buffer-info ()
  "Display buffer info."
  (interactive)
  (message "%s: %d chars, %d lines, modified=%s"
           (buffer-name)
           (buffer-size)
           (line-number-at-pos (point-max))
           (buffer-modified-p)))
```

---

## 6. 编辑器概念对应 Lisp 函数

| Editor | Lisp |
|---|---|
| 选中 region | `(mark)`、`(region-beginning)`、`(region-end)` |
| 杀 ring | `kill-ring`、`kill-new`、`current-kill` |
| 移动 | `forward-char`、`goto-char` |
| 插入 | `insert` |
| 删除 | `delete-char`、`delete-region` |
| 替换 | `replace-match`、`subst-char-in-region` |
| 查找 | `search-forward`、`re-search-forward` |
| Undo | `undo`、`buffer-undo-list` |
| Fill | `fill-paragraph`、`fill-region` |
| 大小写 | `upcase`、`downcase`、`capitalize` |
| 排序 | `sort-lines`、`sort-fields` |

---

## 7. 自测

1. 写一个函数 `(insert-date)` 插入日期
2. 写一个函数 `(count-words)` 数当前 buffer 词数
3. `(interactive "r")` 提供什么?
4. `let` 和 `let*` 区别?
5. `if` 的 then/else 形式不对称,具体是?

**答案**:
> 1. `(defun insert-date () (interactive) (insert (format-time-string "%Y-%m-%d")))`
> 2. 
> ```elisp
> (defun count-words ()
>   (let ((count 0))
>     (save-excursion
>       (goto-char (point-min))
>       (while (re-search-forward "\\w+" nil t)
>         (setq count (1+ count))))
>     count))
> ```
> 3. 两个数字: region 起点 start,终点 end
> 4. let 并行,let* 顺序
> 5. then 一个 form;else 多个 (隐式 progn)

---

## 8. 下一步

进入 `exercises.md` 做 Week 2 的练习。
