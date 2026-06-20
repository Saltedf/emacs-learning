# Concept Anchor: Editor Manual (Week 1)

> 这个文件**完全替代** `emacs-manual-30.2/commands.texi` (复习) + `basic.texi` 的"命令 = Elisp 函数"概念锚点
> 这个 anchor 把你这周学的 Lisp 与你每天用的编辑器概念连起来

---

## 1. 命令的本质: 命令 = Elisp 函数

### 1.1 你的认知升级

Module 1 你学的是"按键 → 行为"——比如按 `C-s` 触发搜索。这是 user 视角。

现在升级到 Lisp 视角——按 `C-s` 实际上是: Emacs 调用 `isearch-forward` 这个 Elisp 函数。**所有键位背后都是 Elisp 函数**。

这个升级的意义: 它让你看到 Emacs 的"内部"。从此你不再是"按键的用户",而是"能修改键位背后函数的开发者"。比如你觉得 `isearch-forward` 不够好,你可以 redefine 它;或者你写自己的 `my-search`,绑到一个新键。这是 Module 1 → Module 3 的认知跳跃。

### 1.2 验证

```
C-h k C-s
```

显示:

```
C-s runs the command isearch-forward (found in global-map), which is
an interactive compiled Lisp function in ‘isearch.el’.
```

看到了吗? **Lisp function**。

`C-h k C-x C-s` → `save-buffer`,也是 Lisp function。

`C-h k` (describe-key) 是你"看见 Emacs 内部"的窗口。任何键位,你都能 C-h k 查它背后的 Lisp 函数。这个习惯建立后,你学 Emacs 的速度会飞跃——不再记键位,而是记函数 (函数有名字、有 docstring、能 grep)。

### 1.3 什么是"interactive"?

不是所有 Lisp 函数都能用键位/M-x 触发。
只有 **interactive function** (有 `(interactive ...)` 声明) 才能。

```elisp
(defun my-func ()
  (message "hello"))          ;; 不是 interactive

(defun my-func-interactive ()
  (interactive)
  (message "hello"))          ;; interactive
```

- `my-func` 只能在 Elisp 代码里调用
- `my-func-interactive` 可以 `M-x my-func-interactive RET` 或绑键

为什么这么分? 因为 Emacs 有几千个内部函数 (工具函数、辅助函数),不希望它们出现在 M-x 补全里。`interactive` 是"用户可调用"的标记——只有用户面向的命令才有。

### 1.4 interactive 的形式

```elisp
(interactive)                    ;; 不读 minibuffer
(interactive "P")                ;; 读 prefix arg
(interactive "sName: ")          ;; 读 string
(interactive "nAge: ")           ;; 读 number
(interactive "fFile: ")          ;; 读 existing file
(interactive "FFile: ")          ;; 读 file (可能不存在)
(interactive "bBuffer: ")        ;; 读 buffer
(interactive "r")                ;; 用 region 范围
(interactive "d")                ;; 用 point
(interactive "M")                ;; read sexp
(interactive (read-from-minibuffer "Foo: "))  ;; 复杂逻辑
```

(Module 3 W2 详细学)

每个 code 对应一种"用户输入"。比如 `"sName: "` 会 prompt "Name: " 让用户输入字符串,作为函数第一参数。这是 Elisp 把"minibuffer 提示 + 参数传递"合一的设计。

### 1.5 命令运行流程

按 `C-s` 不是"魔法"——是 Emacs 内部的一连串步骤:

1. Emacs 收到键盘事件 `C-s`
2. `active-keymap` 查找 → 找到 `isearch-forward` symbol
3. 检查是不是 interactive → 是
4. 调用 `(call-interactively 'isearch-forward)`
5. `isearch-forward` 的 interactive form 决定参数 (这里无)
6. 调用 `(isearch-forward)` 实际函数
7. 函数体执行

整个流程**纯 Elisp**。你可以重定义任何一步。

理解这个流程的意义: 它让你看到"键位 → 命令 → 函数"的链条是可解构的。你想改键位,改 keymap;你想改函数行为,改函数;你想加 hook,加 hook。Emacs 完全透明,没有黑箱。

---

## 2. 你这周学的 Lisp,对应编辑器的什么

### 2.1 car/cdr 对应 buffer 操作

你已经学了 `(car '(a b c))` → `a`。
现在想想: buffer 的"第一行"是什么?

```elisp
(buffer-substring (point-min) (line-end-position))
;; 第一行的文本
```

或:

```elisp
(with-current-buffer "*scratch*"
  (goto-char (point-min))
  ;; 现在 point 在 buffer 头
  )
```

list 的 car 是"第一个元素",buffer 的"第一个字符"是 `(buffer-substring 1 2)` 或 `(char-after 1)`。

### 2.2 cons 对应 buffer 修改

`(cons 'a '(b c))` 在 list 头加 `a`。
buffer 的"在 point 处插入文本":

```elisp
(insert "hello")
```

或:

```elisp
(save-excursion
  (goto-char (point-min))
  (insert "header\n"))
```

注意: list 是不可变的 (`cons` 创建新 list),buffer 是可变的 (`insert` 改变 buffer)。

### 2.3 引用对应 quoting

`'foo` 阻止 eval,返回 symbol。

编辑器里类似: `C-q` (quoted-insert) 阻止键盘事件被解释,直接插入字符。

例如: `C-q C-m` 插入一个字面的 `^M` (Control-M 字符),而不是触发回车。

### 2.4 eval 对应 commands

Lisp 的 eval 把 form 转成值。
编辑器的"按 C-x C-s"把"保存意图"转成"实际保存"。

```elisp
(save-buffer)              ;; 直接调用
(call-interactively 'save-buffer)  ;; 像 C-x C-s 那样调用
(M-x save-buffer)          ;; 等同
```

---

## 3. Editor Manual 中相关章节 (锚点)

### 3.1 commands.texi (你已学过的)

Module 1 你学了 user 视角:
- C-/M-/S-/s-/H- 修饰键
- Key sequence 和 keymap
- 命令命名约定

Module 3 W1 你学 Lisp 视角:
- 命令 = `(interactive)` 标注的函数
- `(call-interactively ...)` 把 symbol 当命令调用
- 命令运行流程的 Elisp 视角

### 3.2 basic.texi 的移动命令

Module 1 学的:
- `C-f` = forward-char
- `C-b` = backward-char

Module 3 W1 学的 Lisp:

```elisp
(forward-char 1)            ;; 移动 1 字符
(forward-char 5)            ;; 移动 5 字符
(backward-char 3)           ;; 后退 3
(forward-word 1)            ;; 前一词
(backward-word 1)           ;; 后一词
(forward-line 1)            ;; 下一行
(forward-line -1)           ;; 上一行
(goto-char (point-min))     ;; buffer 头
(goto-char (point-max))     ;; buffer 尾
(goto-line 100)             ;; 第 100 行 (慢,大 buffer 慎用)
```

这些都是**你可以调用的 Elisp 函数**。

### 3.3 你的代码里用移动

```elisp
(defun my-goto-end-of-next-defun ()
  "Go to end of next defun."
  (interactive)
  (end-of-defun)              ;; 当前 defun 结束
  (beginning-of-defun)        ;; 下一个开始 (其实是同一个,因为 end 移到下一个)
  (end-of-defun))             ;; 下一个结束
```

bind to `C-c e`,试。

### 3.4 buffer 的元数据

每个 buffer 有:
- `(buffer-name)` — 名字
- `(buffer-file-name)` — 关联的文件
- `(buffer-size)` — 字符数
- `(point-min)` — buffer 最小位置 (1 或 narrowing 后不同)
- `(point-max)` — 最大位置
- `(buffer-modified-p)` — 是否修改了

```elisp
(buffer-name)            ;; → "*scratch*"
(buffer-size)            ;; → 1234
(point-max)              ;; → 1235 (size+1)
```

---

## 4. Lisp 视角看 Module 1 学的命令

### 4.1 Kill ring

Module 1 学的:
- `C-w` = kill-region
- `M-w` = kill-ring-save
- `C-y` = yank

Lisp 视角:

```elisp
(kill-region START END)        ;; 杀 region (函数版)
(kill-new "text")              ;; 加 "text" 到 kill-ring
(current-kill 0)               ;; 取最近一个
(setq kill-ring '())           ;; 清空 kill-ring (慎)
(length kill-ring)             ;; 当前 kill-ring 长度
```

### 4.2 Search

```elisp
(search-forward "foo")         ;; 找下一个 "foo"
(search-forward "foo" nil t)   ;; 找不到返回 nil 而不是报错
(re-search-forward "[0-9]+")   ;; 正则
(replace-match "NUM")          ;; 替换刚才匹配的
```

### 4.3 Mark / Region

```elisp
(set-mark (point))             ;; 在 point 设 mark
(mark)                         ;; 返回 mark 位置
(region-beginning)             ;; region 起点
(region-end)                   ;; region 终点
(use-region-p)                 ;; region 是否激活
(deactivate-mark)              ;; 取消 region
```

### 4.4 实战: 用 Elisp 写自动化

```elisp
(defun my-kill-line-and-yank-below ()
  "Kill current line and yank it below."
  (interactive)
  (let ((line (buffer-substring (line-beginning-position)
                                (line-end-position))))
    (kill-whole-line)
    (forward-line 1)
    (insert line)
    (forward-line -1)
    (move-to-column 0)))

(global-set-key (kbd "C-c k") 'my-kill-line-and-yank-below)
```

这就是把"杀行 + 下一行 + 粘贴"自动化。
你 Module 1 学的命令,在 Lisp 里**都可以组合**。

---

## 5. 你这周建立的关键认知

### 5.1 编辑器即 Lisp

你的整个 Emacs,本质是:
- 一堆 Lisp 函数
- 一堆 buffer (有数据)
- 一个事件循环
- 一个 keymap

键位 → 命令 → Lisp 函数 → 改变 buffer/数据。

**没有"魔法"**,所有行为都是 Elisp。

### 5.2 你的力量

Lisp 给你能力:
- 重定义任何函数 (甚至 C-x C-s 都能改)
- 组合函数做新事
- 写自动化

这是为什么 Emacs 用户"住在 Emacs 里":
- 写代码: 用 Emacs
- 写邮件: 用 Emacs (mu4e)
- 看网页: 用 Emacs (eww)
- 聊天: 用 Emacs (erc/circo)
- shell: 用 Emacs (eshell)
- 文件管理: 用 Dired

所有都在**同一个 Lisp 环境**里。

### 5.3 学习曲线的拐点

学完这周,你应该感到**拐点**:
- 之前: 学键位、学命令 (记忆为主)
- 之后: 学 Lisp、学组合 (创造为主)

这是从"用户"到"Geek"的转折。

---

## 6. 实战任务: 重定义一个内置命令

试试这个 (但要小心):

```elisp
(defalias 'yes-or-no-p 'y-or-n-p)
```

把 `yes-or-no-p` (问 "yes" 或 "no") 改成 `y-or-n-p` (问 "y" 或 "n")。
现在 Emacs 问你要不要保存时,只输入 y/n。

这是**一行 Elisp 改变 Emacs 行为**。

更激进:

```elisp
(defalias 'save-buffer 'my-save-with-backup)
(defun my-save-with-backup (&optional args)
  (interactive "P")
  (let ((make-backup-files t))
    (save-buffer args)))
```

重定义 `save-buffer`。但**不要这么做**,会破坏太多东西。

正确做法: 用 hook 或 advice。

---

## 7. 自测

1. 为什么不是所有 Elisp 函数都能用 M-x 调用?
2. `(interactive "sName: ")` 干啥?
3. 写一个函数,提示输入名字,在 point 插入 "Hello, NAME"。
4. `(call-interactively 'foo)` 和 `(foo)` 区别?
5. 解释 "命令 = Elisp 函数" 的意义。

**答案**:
> 1. 只有带 (interactive) 声明的才能 (interactive function 或 command)
> 2. 提示输入字符串,绑到名为 "Name:" 的 prompt,作为函数第一参数
> 3.
> ```elisp
> (defun my-insert-hello (name)
>   (interactive "sName: ")
>   (insert (format "Hello, %s" name)))
> ```
> 4. 前者像 M-x 调用,处理 interactive 参数;后者直接调用,忽略 interactive
> 5. 意味着所有编辑器行为都是 Elisp,你可以重定义、组合、自动化

---

## 8. 下一步

进入 `exercises.md` 做本周的练习 (Chassell 之外的)。
