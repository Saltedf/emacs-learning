# Exercises: Week 5 Minibuffer + Commands + Keymaps

> 12 题

这周练习考 minibuffer 的所有 `read-*`、interactive 各种 char code、keymap 操作。每题一个具体技能,做完你能写任意交互命令。

---

### 题 1: 简单 prompt

```elisp
(defun my-greet ()
  (interactive)
  (let ((name (read-string "Your name: ")))
    (message "Hello, %s!" name)))
```

最简 prompt——`read-string` 读一行 string。`message` 用 `%s` 显示。

### 题 2: 数字 + 默认

```elisp
(defun my-pick-num ()
  (interactive)
  (let ((n (read-number "Pick (default 42): " 42)))
    (message "You picked: %d" n)))
```

`read-number` 第二参数是默认值——用户空 RET 接受 42。这让 prompt 友好。

### 题 3: completing-read 多选

```elisp
(defun my-pick-colors ()
  (interactive)
  (let ((color (completing-read "Color: "
                                '("red" "green" "blue" "yellow")
                                nil t)))
    (message "Picked: %s" color)))
```

`completing-read` + `t` (REQUIRE-MATCH)——用户必须选列表中的一个,不能自由输入。防止 typo。

### 题 4: 文件 + 检查

```elisp
(defun my-open-checked ()
  (interactive)
  (let ((file (read-file-name "Open: " nil nil t)))
    (if (file-exists-p file)
        (find-file file)
      (message "Not exists: %s" file))))
```

`read-file-name` 第四参数 `t` 是 MUSTMATCH——用户必须选已存在的文件。但我们额外 check (因为可能取消),给明确错误。

### 题 5: region 操作

```elisp
(defun my-md5-region (beg end)
  (interactive "r")
  (let* ((text (buffer-substring beg end))
         (hash (md5 text)))
    (message "MD5: %s" hash)))
```

`(interactive "r")` 自动提供 beg end——只有 region active 时有意义。但即使没 region,beg=end=point,md5 空 string 也不报错。

### 题 6: prefix 数字

```elisp
(defun my-repeat-line (n)
  (interactive "p")
  (dotimes (_ (or n 1))
    (save-excursion
      (end-of-line)
      (newline)
      (insert (buffer-substring
               (line-beginning-position)
               (line-end-position))))))
```

`(interactive "p")` 提供 numeric prefix——`M-3 M-x my-repeat-line` 复制当前行 3 次。`(or n 1)` 防御 n=nil。

### 题 7: prefix raw

```elisp
(defun my-debug-prefix (arg)
  (interactive "P")
  (message "Raw prefix: %S" arg))
;; 按 C-u C-u M-x my-debug-prefix RET → "Raw prefix: (16)"
```

`P` (大写) 是 raw prefix——`C-u` 是 `'(4)`,`C-u C-u` 是 `'(16)`,`M-5` 是 `5`。复杂,但保留所有信息。

### 题 8: this/last-command

```elisp
(defun my-double-on-repeat (n)
  (interactive "p")
  (let ((factor (if (eq last-command 'my-double-on-repeat) 2 1)))
    (message "n=%d, factor=%d, result=%d" n factor (* n factor))))
```

连续按两次同样的命令——`last-command` 检测,加倍 effect。这是 kmacro "repeat" 的核心模式。

### 题 9: read-key

```elisp
(defun my-pick-key ()
  (interactive)
  (message "Press a key...")
  (let ((key (read-key)))
    (message "You pressed: %S" key)))
```

`read-key` 读单个 key event——比 read-key-sequence 简单,适合"按任意键继续"。

### 题 10: read-char

```elisp
(defun my-confirm (prompt)
  (let ((char (read-char (format "%s (y/n): " prompt))))
    (memq char '(?y ?Y))))
```

`read-char` 读单字符 codepoint——`?y` 是 121。返回 boolean (是否 y/Y)。

### 题 11: 自定义 keymap

```elisp
(defvar my-mode-map
  (let ((map (make-sparse-keymap)))
    (define-key map "a" (lambda () (interactive) (message "A")))
    (define-key map "b" (lambda () (interactive) (message "B")))
    map))
```

这是 prefix keymap 的基础——`a` 和 `b` 绑到不同 lambda。要让 `C-c m a` 工作,还要 `(global-set-key (kbd "C-c m") my-mode-map)`。

### 题 12: lookup-key

```elisp
(lookup-key global-map (kbd "C-x C-f"))
;; → find-file

(lookup-key global-map (kbd "C-c C-v"))
;; → nil 或某命令
```

`lookup-key` 让你查"某键绑了什么"。返回 command symbol 或 nil。`C-h w` (where-is) 是反向 lookup。

---

## 自测

1. 写一个命令,问"项目名",然后 `cd ~/projects/NAME`
2. 写一个命令,有 region 就 upcase,没 region 就 insert "hi"
3. 怎么实现"按两次同键"的 toggle?
4. `(interactive "sPrompt: ")` 怎么传多个参数?
5. 怎么查某命令绑到哪些键?

**答案**:
> 1. 
> ```elisp
> (defun my-cd-project ()
>   (interactive)
>   (let ((name (read-string "Project: ")))
>     (cd (expand-file-name name "~/projects/"))))
> ```
> 2. 
> ```elisp
> (defun my-smart (beg end)
>   (interactive "r")
>   (if (use-region-p)
>       (upcase-region beg end)
>     (insert "hi")))
> ```
> 3. 用 last-command 比较
> 4. `(interactive "sFoo: \nnBar: ")` (\n 分隔)
> 5. C-h w 或 where-is-internal

---

进入 `../week-06-modes-files-buffers/`。
