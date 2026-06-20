# Hooks 详解

> 学完这个文件,你能熟练用 hook 解耦代码

---

## 1. Hook 是什么

### 1.1 数据结构

hook 的本质极其简单: 一个变量,值是函数列表。某些事件发生时,Emacs 遍历这个列表调用每个函数。这个简单的数据结构是 Emacs 解耦的基石——它让模块之间能"订阅事件"而不互相依赖。

下面演示 hook 的完整生命周期: 定义一个 hook 变量(初始 nil)、加一个函数(列表变成有一个元素)、触发 hook(调用所有函数)。这就是 hook 的全部机制——没有魔法,就是函数列表:

```elisp
(defvar my-hook nil
  "List of functions to run when X happens.")

;; 加:
(add-hook 'my-hook #'my-func)
;; → my-hook 现在是 (#'my-func)

;; 跑:
(run-hooks 'my-hook)
;; → 调每个函数
```

### 1.2 设计模式

Hook 体现了"观察者模式"(Observer Pattern): 模块 A 提供事件,模块 B 监听。A 不需要知道 B 存在,B 也不需要修改 A。这种解耦让 Emacs 生态可以无限扩展——你写一个 minor mode,别人能用 hook 扩展它,而不需要改你的代码。

Hook 让你**解耦**:
- 模块 A 提供事件 (定义 hook)
- 模块 B 监听事件 (add-hook)
- A 和 B 不直接依赖

### 1.3 历史

Hook 是 Emacs 最古老的设计之一——1976 年的原始 Emacs 就有 hook。现代编辑器后来才学到这个概念(VS Code 的 "events"、Sublime 的 "listeners" 都是类似设计)。但 Emacs 的 hook 有一个独特优势: hook 函数本身是 Elisp,可以做任何事,不只是"响应"——可以修改参数、阻断流程、注入副作用。

Emacs 1976 年就有 hook。
现代编辑器后来才学 (VS Code 的 "events" 是类似概念)。

---

## 2. 标准 hook 类型

Emacs 有四种 hook 运行方式,区别在于"传不传参数"和"何时停止"。

### 2.1 Normal hook

最常见。值是函数列表,运行时无参数。99% 的 hook(如 `prog-mode-hook`、`before-save-hook`)都是这种:

最常见。值是函数列表,运行时无参数。

```elisp
;; 定义:
(defvar my-event-hook nil "...")

;; 运行 (调用每个函数):
(run-hooks 'my-event-hook)
```

### 2.2 Function hook (run-hook-with-args)

有时你需要给 hook 函数传参数——比如"保存了什么内容"。用 `run-hook-with-args`:

```elisp
(defvar my-filter-hook nil "...")

(run-hook-with-args 'my-filter-hook ARG1 ARG2)
;; 每个 hook 函数接收 ARG1 ARG2
```

### 2.3 Run until success

有些场景你想"找第一个能处理这个事件的函数"——比如格式化代码,你有多个 formatter,hook 里每个 formatter 试一下,第一个成功的就停。用 `run-hook-with-args-until-success`:

```elisp
(run-hook-with-args-until-success 'my-hook ARG)
;; 跑每个,第一个返回 non-nil 就停
```

### 2.4 Run until failure

相反方向: 跑每个函数,第一个返回 nil 就停。用于"验证链"——所有验证都通过才继续:

```elisp
(run-hook-with-args-until-failure 'my-hook ARG)
;; 跑每个,第一个返回 nil 就停
```

---

## 3. 内置 hook 大全

Emacs 内置几百个 hook,覆盖几乎所有事件。下面分类列出最常用的。

### 3.1 启动相关

启动相关 hook 让你能在 Emacs 生命周期的不同阶段跑代码——`before-init-hook` 在 init.el 加载前(极少用),`after-init-hook` 在加载后(常用),`emacs-startup-hook` 在完全启动后(最常用,因为这时 frame 已经准备好):

| Hook | 何时跑 |
|---|---|
| `before-init-hook` | init 加载前 |
| `after-init-hook` | init 加载后 |
| `emacs-startup-hook` | 启动完成 |
| `window-setup-hook` | window 初始化 |
| `term-setup-hook` | terminal 初始化 |
| `server-visit-hook` | emacsclient 访问 |
| `server-switch-hook` | emacsclient 切换 |

### 3.2 文件相关

文件 hook 是最常用的——保存、打开、关闭文件时跑代码。`before-save-hook` 和 `after-save-hook` 让"保存时自动格式化""保存时跑 lint"成为几行代码的事:

| Hook | 何时跑 |
|---|---|
| `find-file-hook` | find-file 之后 |
| `find-file-not-found-functions` | 文件不存在时 |
| `before-save-hook` | save 之前 |
| `after-save-hook` | save 之后 |
| `write-file-functions` | 替换 save 逻辑 |
| `write-contents-functions` | 替换 contents 写入 |
| `after-insert-file-functions` | 文件插入后 |
| `kill-buffer-hook` | kill buffer 时 |
| `kill-buffer-query-functions` | kill 前询问 |

### 3.3 Mode 相关

每个 mode 都有对应 hook——进入该 mode 时跑。`prog-mode-hook` 特别有用: 所有编程 mode(python/js/go/...)进入时都会跑,所以你 add 一个函数到 prog-mode-hook,所有编程语言都受益:

| Hook | 何时跑 |
|---|---|
| `prog-mode-hook` | 进入 prog-mode |
| `text-mode-hook` | 进入 text-mode |
| `<mode>-hook` | 进入某 mode (例 python-mode-hook) |
| `change-major-mode-hook` | 离开 major mode |
| `after-change-major-mode-hook` | 切换后 |
| `minor-mode-<mode>-hook` | minor mode 切换 |

### 3.4 编辑相关

编辑 hook 让你能对"每次按键""每次修改"做反应。`post-command-hook` 极其常用但也极危险——它在每次命令后跑,如果你的函数慢,整个 Emacs 会卡:

| Hook | 何时跑 |
|---|---|
| `pre-command-hook` | 命令执行前 |
| `post-command-hook` | 命令执行后 |
| `first-change-hook` | 第一次修改 buffer |
| `before-change-functions` | 修改前 |
| `after-change-functions` | 修改后 |
| `point-motion-hook` | 光标移动 |

### 3.5 buffer / window

| Hook | 何时跑 |
|---|---|
| `buffer-list-update-hook` | buffer 列表变 |
| `window-configuration-change-hook` | window 配置变 |
| `window-size-change-functions` | window 大小变 |
| `buffer-switch-hook` (有些包) | 切 buffer |

### 3.6 Other

| Hook | 何时跑 |
|---|---|
| `focus-out-hook` | Emacs 失焦 |
| `focus-in-hook` | Emacs 得焦 |
| `suspend-hook` | 挂起 |
| `kill-emacs-hook` | 退出 Emacs |
| `kbd-macro-termination-hook` | 宏结束 |

---

## 4. add-hook 完整

`add-hook` 的完整签名有四个参数,后两个常被忽略但很重要。

```elisp
(add-hook HOOK FUNCTION &optional DEPTH LOCAL APPEND)
```

### 4.1 参数

- HOOK: hook 变量名 (symbol)
- FUNCTION: 函数 (symbol 或 lambda)
- DEPTH (Emacs 28+): 排序优先级 (整数,负数早跑,正数晚跑)
- LOCAL: if non-nil,buffer-local 加
- APPEND (deprecated): if non-nil,加到末尾 (用 DEPTH 替代)

### 4.2 DEPTH 例子

DEPTH 是 Emacs 28 加的,解决"hook 执行顺序不可控"的老问题。之前 hook 按添加顺序跑,你想让某函数"最早跑"或"最晚跑"很麻烦。DEPTH 让你显式控制——负数早跑,正数晚跑,0 是默认:

```elisp
(add-hook 'python-mode-hook #'setup-1)            ; 默认 depth 0
(add-hook 'python-mode-hook #'setup-2 -50)        ; 早跑
(add-hook 'python-mode-hook #'setup-3 100)        ; 晚跑

;; 执行顺序: setup-2 → setup-1 → setup-3
```

### 4.3 LOCAL

LOCAL 参数让 hook 只加到当前 buffer——这是 Emacs "buffer 是宇宙中心"哲学的体现。同一个 hook 变量在不同 buffer 可以有不同的函数列表:

```elisp
(add-hook 'write-file-functions #'my-write nil t)
;; t = buffer-local,只影响当前 buffer
```

### 4.4 lambda

可以直接用 lambda 加 hook——但有个坑: lambda 没有名字,以后无法 remove。所以推荐用命名函数:

```elisp
(add-hook 'python-mode-hook
          (lambda ()
            (setq indent-tabs-mode nil)
            (setq python-indent-offset 4)))
```

注意: lambda 加 hook 后**不能 remove-hook** (没有名字)。
推荐用命名函数,方便 remove:

```elisp
(defun my-python-setup ()
  (setq indent-tabs-mode nil)
  (setq python-indent-offset 4))

(add-hook 'python-mode-hook #'my-python-setup)
(remove-hook 'python-mode-hook #'my-python-setup)
```

---

## 5. remove-hook

```elisp
(remove-hook HOOK FUNCTION &optional LOCAL)
```

---

## 6. run-hooks

触发 hook 的几种方式——选哪种取决于"要不要传参""何时停":

```elisp
(run-hooks 'HOOK1 'HOOK2 ...)        ; 跑多个 normal hook
(run-hook 'HOOK)                     ; 旧 API
(run-hook-with-args 'HOOK ARGS...)   ; 调每个,传 args
(run-hook-with-args-until-success 'HOOK ARGS...)
(run-hook-with-args-until-failure 'HOOK ARGS...)
(run-hook-wrapped 'HOOK WRAP-FN ARGS...)   ; 用 wrapper 跑
```

---

## 7. 自定义 hook

写自己的包时,定义 hook 让别人能扩展你的代码——这是"可扩展包"的标志。

### 7.1 定义

```elisp
(defvar my-todo-saved-hook nil
  "Hook run after saving todos.
Each function receives the saved list as argument.")
```

### 7.2 跑

在"事件发生"的地方调 `run-hooks` 或 `run-hook-with-args`:

```elisp
(defun my-todo-save ()
  ...
  (run-hook-with-args 'my-todo-saved-hook my-todo-list))
```

### 7.3 别人用

其他用户(或未来的你)可以 add-hook 来扩展——这就是 Emacs 包生态的精髓。你不需要改原代码就能加新行为:

```elisp
(add-hook 'my-todo-saved-hook
          (lambda (todos)
            (message "Saved %d todos" (length todos))))
```

---

## 8. hook vs advice

hook 和 advice 都能"在某函数周围做事",但本质不同。理解区别能让你选对工具。

### 8.1 Hook

Hook 是**事件通知**: "X 发生了"。多个监听者都能收到通知,但它们不能改变 X 本身的行为:

事件通知: "X 发生了"
- 多个监听者
- 不能改变原行为

### 8.2 Advice

Advice 是**函数包装**: 在某函数调用前后插入代码,可以改参数、改返回值、甚至完全替代原函数。这是更强大的"侵入式"机制:

函数包装: "X 调用时,先做 Y"
- 单个 wrapper
- 可以改变参数 / 返回值

```elisp
(define-advice my-func (:around (orig-func arg) my-log)
  (message "Calling my-func with %s" arg)
  (let ((result (funcall orig-func arg)))
    (message "Result: %s" result)
    result))
```

(Module 6 W3 详细讲 advice)

### 8.3 选择

简单规则: 通知用 hook(不改行为,只监听),改变行为用 advice(包装、修改):

- 通知用 hook
- 改变行为用 advice

---

## 9. 实战例子

下面是几个真实有用的 hook 用法——每一个都是 Emacs 老用户的标配。

### 9.1 自动格式化保存

`before-save-hook` 让"保存前自动清理"成为一行代码。下面两行: 保存时自动删除行尾空格、emacs-lisp mode 下自动 indent 整个 buffer:

```elisp
(add-hook 'before-save-hook #'delete-trailing-whitespace)
(add-hook 'before-save-hook
          (lambda ()
            (when (derived-mode-p 'emacs-lisp-mode)
              (indent-region (point-min) (point-max)))))
```

### 9.2 进入 prog mode 设置

进入任何编程 mode 时自动启用行号、flyspell-prog(注释拼写检查)、show-paren(高亮匹配括号)。这三件事一次配置,所有编程语言受益:

```elisp
(add-hook 'prog-mode-hook
          (lambda ()
            (display-line-numbers-mode 1)
            (flyspell-prog-mode 1)
            (show-paren-mode 1)))
```

### 9.3 监控命令

`post-command-hook` 在每次命令后跑。下面这段记录你用过的每个命令(带时间戳),用于"分析自己的 Emacs 使用习惯"。注意 post-command-hook 必须快——它跑得极其频繁:

```elisp
(defvar my-command-log nil)

(add-hook 'pre-command-hook
          (lambda ()
            (push (list (current-time) this-command)
                  my-command-log)))
```

### 9.4 自动 revert

打开文件后如果外部修改了(比如 git pull),Emacs 默认不会自动重读。`global-auto-revert-mode` 解决这个问题——它内部用 file-notify 或定时检查 hook:

```elisp
(global-auto-revert-mode 1)
;; 实际上这个 mode 内部用 after-save-hook / file-notify
```

### 9.5 公司: 在 shell 里禁用 flymake

有些 mode 默认开了某些功能,但你在特定 buffer 不想要。下面这段在 shell-mode 里关掉 flymake——用 buffer-local 思路:

```elisp
(add-hook 'shell-mode-hook
          (lambda ()
            (flymake-mode -1)))
```

---

## 10. Hook 的常见坑

hook 看起来简单,但有几个经典坑。

### 10.1 hook 跑多次

某些 hook(如 after-save-hook、post-command-hook)在每次事件都跑。如果你的 hook 函数慢,整个 Emacs 会卡。原则: hook 函数要快,避免长操作(尤其 post-command-hook):

某些 hook (如 after-save-hook) 在每次事件都跑。
hook 函数要快,避免长操作。

### 10.2 buffer-local vs global

LOCAL 参数的坑: 忘了加 `t`,你以为只影响当前 buffer,其实影响了所有 buffer:

```elisp
(add-hook 'write-file-functions #'my-func nil t)
;; buffer-local,只当前 buffer

(add-hook 'write-file-functions #'my-func)
;; global,所有 buffer
```

新手常搞混。

### 10.3 lambda 不能 remove

lambda 加 hook 后无法 remove——因为 remove-hook 需要函数对象相等,而每次 `(lambda ...)` 求值都创建新对象:

```elisp
(add-hook 'foo-hook (lambda () ...))
;; 之后不能 remove,因为 lambda 没名字
```

要 remove 用命名函数。

### 10.4 hook 顺序

历史 hook 按添加顺序跑,但依赖顺序不可控。Emacs 28+ 用 DEPTH 显式控制:

历史 hook 按添加顺序跑。
Emacs 28+ 用 DEPTH 显式控制。

---

## 11. 自测

1. hook 是什么数据结构?
2. add-hook 的 LOCAL 参数干啥?
3. normal hook vs run-hook-with-args 区别?
4. `before-save-hook` 何时跑?
5. 怎么自定义 hook?

**答案**:
> 1. 变量,值是函数列表
> 2. t = buffer-local 加,只影响当前 buffer
> 3. normal 无参数;run-hook-with-args 传参数
> 4. 每次保存前
> 5. defvar 定义,run-hooks 触发

---

## 12. 下一步

进入 `minor-mode-tutorial.md`。
