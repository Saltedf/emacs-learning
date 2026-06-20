# Concept Anchor: Editor Manual (Week 2)

> 这周你的 eval/variables/functions 对应编辑器的"命令"和"hook"

Emacs 的核心是 **command loop**——一个无限循环: 读按键 → 找命令 → 执行 → 更新显示。这是 Emacs 区别于其他编辑器的本质: 几乎所有用户行为都是 command,所有 command 都是 Lisp function。这让 Emacs 极其可编程——你可以"插入"到任何环节 (hook、advice、keymap) 改变行为。

理解 command 循环让你能写"智能"功能: 监听用户行为 (`post-command-hook`)、阻止默认行为 (`pre-command-hook` 改 `this-command`)、追踪最近命令 (`last-command`)。

---

## 1. 命令 = interactive function

### 1.1 回顾

Module 3 W1 学过: 命令是有 `(interactive)` 的函数。

普通函数和命令的区别只在 `(interactive)`——它告诉 command loop "这个函数可以由按键触发"。`(+ 1 2)` 是普通函数,不能 M-x 调用;`(defun my-cmd () (interactive) ...)` 是命令,可以。

### 1.2 深入

```elisp
(commandp #'find-file)        ; → t
(commandp #'+)                ; → nil
(commandp (lambda () (interactive) (message "hi")))  ; → t
```

`commandp` 检查对象是否是 command——predicate。`(commandp #'+)` 返回 nil 因为 `+` 没有 interactive。但你可以直接 `(funcall #'+ 1 2)`——它是函数,只是不是命令。

### 1.3 call-interactively

```elisp
(call-interactively #'find-file)
;; 等同按 C-x C-f
```

`call-interactively` 让 Lisp 代码"模拟"用户按键——它会触发 interactive 读取参数 (minibuffer prompt),然后调函数。这是 Elisp 触发交互命令的标准方式。

实战: 你写一个"快捷宏",例如 `(defun my-open-init () (interactive) (find-file user-init-file))`。但有时你想从代码触发——`(call-interactively #'find-file)` 就用这个。

### 1.4 this-command

Emacs 维护"最近命令"状态——当前和上一个 command 的 symbol。

```elisp
this-command                  ; 当前命令 symbol
last-command                  ; 上一个
real-this-command             ; 真实命令 (不被 advice 影响)
```

`this-command` 是当前正在执行的 command。`last-command` 是上一个。这些让"重复命令"模式可能——例如 `(eq last-command 'my-toggle)` 检查"刚才是不是 toggle 命令",决定当前行为。

`real-this-command` 是"真实"命令——`this-command` 可能被 advice 或 hook 修改,`real-this-command` 总是用户实际按的。

```elisp
(defun my-log-commands ()
  (message "ran: %s" this-command))

(add-hook 'post-command-hook #'my-log-commands)
```

`post-command-hook` 在每个命令后跑——加这个 hook 让你看到所有用户行为。可以用于审计、统计、调试。

---

## 2. Hooks (review)

Module 4 学过,Module 6 W6 深入。

hook 是"事件系统"——变量存函数列表,事件发生时 Emacs 顺序调用。这是 Emacs 解耦的核心机制。

---

## 3. pre-command-hook / post-command-hook

```elisp
(add-hook 'pre-command-hook
          (lambda ()
            (when (eq this-command 'self-insert-command)
              ;; 用户输入字符
              )))
```

`pre-command-hook` 在每个 command 执行前跑——你可以检查 `this-command`,甚至改它 (`(setq this-command 'ignore)` 阻止默认)。

`post-command-hook` 在 command 执行后跑——用于 update display、log、cleanup。

每个命令前/后都跑——所以 hook 函数要快 (< 1ms),否则卡。

实战: 
- `pre-command-hook` 阻止某操作 (`(setq this-command 'ignore)` 取消命令)
- `post-command-hook` 自动保存 (`(when (buffer-modified-p) (my-save-if-idle))`)
- `post-command-hook` 更新 mode line (`force-mode-line-update`)

---

## 4. 自测

1. `(commandp #'foo)` 检查什么?
2. `this-command` 和 `last-command` 区别?
3. 怎么从代码"按"C-x C-f?

**答案**:
> 1. 是不是 interactive function
> 2. 当前正在跑的;上一个
> 3. (call-interactively #'find-file)

---

## 5. 下一步

进入 `exercises.md`。
