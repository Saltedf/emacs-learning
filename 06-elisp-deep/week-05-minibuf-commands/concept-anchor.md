# Concept Anchor + Exercises: Week 5

## Concept Anchor: mini.texi + m-x.texi + commands.texi

Module 1 已学 user 视角。
Module 6 W5 学 Lisp 视角。

Emacs 的"按键到命令"流程是理解所有交互的基础。一旦你理解它,你能写"智能"命令——监听用户、阻止默认、追踪行为。

### commands.texi 的"按键到命令"流程

```
1. 键盘事件
2. input-decode-map (翻译键)
3. function-key-map
4. keymap lookup (优先级见上)
5. 找到 command (interactive function)
6. prefix arg 处理
7. pre-command-hook
8. interactive 读参数
9. 命令执行
10. post-command-hook
```

理解这流程让你能写更精巧的命令。

每一步都是"插入点"——你可以 hook 到 pre-command-hook 改变 `this-command`,或在 post-command-hook 加副作用。这让 Emacs 极其可编程——任何用户行为都可观察、可改变。

### 例子: 阻止某命令

```elisp
(defun my-block-self-insert ()
  (when (and (eq this-command 'self-insert-command)
             (looking-at "[ \t]"))
    (setq this-command 'ignore)))

(add-hook 'pre-command-hook #'my-block-self-insert)
```

这是 `pre-command-hook` 的经典用例——改变 `this-command` 取消默认。这里: 如果用户输入字符,且 point 后是空白,改 command 为 `ignore` (什么都不做)。

实战用途: 防止 trailing whitespace (输入空格在已有空格后忽略)、阻止某些 key 在某些 mode。

---

## Exercises: Week 5

这周练习覆盖 minibuffer 的所有 `read-*` 函数,以及 interactive 的各种字符代码。

### 题 1: read-string

```elisp
(defun my-greet ()
  (interactive)
  (let ((name (read-string "Name: ")))
    (message "Hello, %s" name)))
```

最简单的 prompt——read-string 读一行,返回 string。

### 题 2: read-number

```elisp
(defun my-square-input ()
  (interactive)
  (let ((n (read-number "Number: ")))
    (message "Square: %d" (* n n)))
```

read-number 自动 parse——用户输入 "42" 得到 integer 42。如果输入 "abc",Emacs 报错让用户重输。

### 题 3: completing-read

```elisp
(defun my-pick-language ()
  (interactive)
  (let ((lang (completing-read "Language: "
                               '("python" "js" "rust" "go"))))
    (message "You picked: %s" lang)))
```

completing-read 让用户从选项中选——TAB 触发补全,*Completions* buffer 显示候选。

### 题 4: read-file-name

```elisp
(defun my-open-other-window ()
  (interactive)
  (let ((file (read-file-name "Open: ")))
    (find-file-other-window file)))
```

read-file-name 加文件补全——TAB 触发 dired-style。`find-file-other-window` 在另一个 window 打开。

### 题 5: y-or-n-p

```elisp
(defun my-delete-current-buffer ()
  (interactive)
  (when (y-or-n-p "Delete this buffer? ")
    (kill-buffer)))
```

`y-or-n-p` 是单字符确认——按 y 接受,按 n 拒绝。比 `yes-or-no-p` 快。

### 题 6: interactive "r"

```elisp
(defun my-uppercase-region (beg end)
  (interactive "r")
  (upcase-region beg end))
```

`(interactive "r")` 提供 region 的 beg 和 end——但只在 region active 时有意义。如果没 region,beg 和 end 都是 point,upcase 一个空 region (no-op)。

### 题 7: interactive "p"

```elisp
(defun my-repeat-hi (n)
  (interactive "p")
  (dotimes (_ n)
    (insert "hi ")))
```

`(interactive "p")` 提供 numeric prefix——`M-3 M-x my-repeat-hi` 插入 "hi hi hi "。

### 题 8: 复杂 interactive

```elisp
(defun my-complex (name color region)
  (interactive
   (list (read-string "Name: ")
         (completing-read "Color: " '("red" "green" "blue"))
         (if (use-region-p)
             (list (region-beginning) (region-end))
           (list (point) (point)))))
  ...)
```

`(interactive (list ...))` 让你写任意逻辑——这里混合 read-string、completing-read、region 检测。返回 list 作为函数参数。

### 题 9: last-command

```elisp
(defun my-double-or-half ()
  (interactive)
  (if (eq last-command 'my-double-or-half)
      (message "called twice in a row")
    (message "called once")))
```

`last-command` 让你检测"上次跑的命令"——常用于"重复命令加倍"模式。例如 `(eq last-command 'my-double-or-half)` 检查"刚才是不是这个命令",如果是,加倍效果。

### 题 10: define-key

```elisp
(defvar my-map
  (let ((map (make-sparse-keymap)))
    (define-key map "a" #'my-action-a)
    (define-key map "b" #'my-action-b)
    map))
(global-set-key (kbd "C-c m") my-map)
```

这是 prefix keymap 的标准模式——`C-c m` 进入 my-map,然后 `a` 或 `b` 触发不同 action。`C-c m a`、`C-c m b` 都是有效绑定。

---

## 自测

1. 怎么读 string? number? file?
2. completing-read 的 REQUIRE-MATCH 含义?
3. interactive "p" vs "P"?
4. 怎么用 region 参数?
5. this-command 和 last-command 区别?

**答案**:
> 1. read-string, read-number, read-file-name
> 2. nil 自由输入;t 必须匹配;确切值更严格
> 3. p 是 numeric prefix (number);P 是 raw prefix
> 4. (interactive "r") 提供 (beg end)
> 5. this-command 当前;last-command 上一个

---

进入 `../week-06-modes-files-buffers/`。
