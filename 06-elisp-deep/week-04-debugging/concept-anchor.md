# Concept Anchor + Exercises: Week 4

## Concept Anchor: trouble.texi

Emacs 区别于其他编辑器的一个文化: **bug 报告是第一公民**。Emacs 有专门的 bug tracker (`debbugs.gnu.org`)、邮件列表 (`bug-gnu-emacs@gnu.org`)、内置工具 (`M-x report-emacs-bug`)。这是 1985 年传承下来的 GNU 文化——软件透明、社区驱动。

### 1. 报告 Bug

Emacs 的 bug 走邮件列表 `bug-gnu-emacs@gnu.org`。

或用 `M-x report-emacs-bug`:

```
M-x report-emacs-bug RET
```

自动填:
- Emacs 版本
- 系统信息
- recent input (`C-h l` 的内容)
- recent messages (`C-h e` 的内容)

你写 bug 描述,发送 (`C-c C-c`)。

`report-emacs-bug` 自动收集"重现 bug 所需的一切"——版本、配置、最近按键、最近消息。这极大提升 bug 报告质量——维护者拿到报告能立即重现,而不需要"我的 Emacs 坏了"这种无用报告。

这是好的工程实践——任何软件项目都应该有"标准 bug 报告流程"。Emacs 30 年前就有了。

### 2. 常见问题排查

| 症状 | 原因 |
|---|---|
| 启动慢 | init.el 大,package 多;用 use-package :defer |
| 卡顿 | GC 频繁,调高 gc-cons-threshold |
| 不响应 | 后台 process 卡;`M-x list-processes` |
| 配置不生效 | 拼写错;load-path 不对;require 失败 |
| 中文乱码 | 编码不对;`(prefer-coding-system 'utf-8)` |
| 字体不对 | set-fontset-font 没设 |
| 颜色不对 | terminal 不支持 256 色,或主题问题 |

这个表是"诊断决策树"——看到症状,立即知道往哪个方向查。这是经验积累,你遇到一次就记住。

### 3. 启动调试

```bash
emacs --debug-init       # init 报错进 debugger
emacs -Q                 # 干净启动
emacs -q                 # 只不加载 init
```

`--debug-init` 是 init 出错时第一选择——直接进 backtrace,看到底哪里错。
`-Q` 是"全新 Emacs"——完全不加载任何配置、任何 site-lisp、任何 package。用于"我的配置是否引起这个问题"——如果 `-Q` 下问题消失,是配置问题;否则是 Emacs 本身问题。
`-q` 比 `-Q` 弱——只跳过 init.el,但仍加载 site-lisp。

这是 debug 的"层次化"——从最简单 (`-q`) 到最彻底 (`-Q`) 逐步缩小范围。

---

## Exercises: Week 4

这周练习让你**实际用**调试工具——不只是看 API。Edebug、condition-case、profile、ERT 都是"必须自己跑过"才能掌握。

### 题 1: 用 Edebug

调试一个有 bug 的函数:

```elisp
(defun my-buggy (lst)
  "Sum list, but buggy."
  (let ((sum 0))
    (while lst
      (setq sum (+ sum (car lst)))
      (setq lst (cdr lst)))
    sum))

(my-buggy '(1 2 3))
```

发现啥 bug?

(答: 没问题,但用 `dolist` 更清晰)

这个函数实际没 bug——但用 Edebug 你能看到它怎么跑 (sum 累积过程)。如果改成 `(setq sum (+ sum lst))` (忘 car),就有 bug——Edebug 让你立即看到 sum 变成 `(1 2 3)` 而不是 1。

### 题 2: condition-case

```elisp
(defun my-divide-safe (a b)
  (condition-case nil
      (/ a b)
    (arith-error 0)))

(my-divide-safe 10 2)          ; → 5
(my-divide-safe 10 0)          ; → 0
```

`(/ 10 0)` 抛 arith-error,handler 返回 0。注意 `(condition-case nil ...)` 第一个参数 nil——不绑定错误信息,因为我们不关心 detail。

### 题 3: 自定义 error

```elisp
(put 'my-error 'error-conditions '(my-error error))
(put 'my-error 'error-message "My error")

(defun my-func ()
  (signal 'my-error '("something wrong")))

(condition-case err
    (my-func)
  (my-error (message "Got: %s" err)))
```

`error-conditions` 列 my-error 的所有祖先——`(my-error error)` 表示它继承 error。这让 `(error ...)` handler 也能捕获它。

### 题 4: profiler

```elisp
(profiler-start 'cpu)
(dotimes (_ 100000) (+ 1 2 3))
(profiler-report)
(profiler-stop)
```

看哪个函数慢。

profile 报告显示 `+` 占用 CPU——但具体到 dotimes 调用、`+` 内部 subr 等。profiler 采样调用栈,显示 hot spot。

### 题 5: ERT

```elisp
(require 'ert)

(ert-deftest my-sum-test ()
  (should (= (+ 1 2 3) 6)))

(ert-deftest my-error-test ()
  (should-error (error "foo")))

M-x ert-run-tests-interactively RET
```

ERT 测试是"未来你"的礼物——半年后改代码,跑测试知道没破坏现有功能。

### 题 6: unwind-protect

```elisp
(defun my-safe-modify ()
  (let ((modified (buffer-modified-p)))
    (unwind-protect
        (progn
          (insert "temp")
          ;; do stuff
          )
      (unless modified
        (set-buffer-modified-p nil)))))
```

这是"不污染 buffer modified 状态"的标准 pattern——你内部 insert (临时),但 user 视角 buffer 没改。Emacs 内部 font-lock、imenu 等大量用这个。

### 题 7: cl-assert

```elisp
(require 'cl-lib)

(defun my-only-positive (n)
  (cl-assert (> n 0) nil "Expected positive, got %s" n)
  (* n 2))

(my-only-positive 5)           ; → 10
(my-only-positive -1)          ; → error
```

cl-assert 是 design by contract——声明 precondition。调用方违反,立即报错,而不是 silently 错误结果。

### 题 8: message 调试

```elisp
(defun my-debug-vars ()
  (let ((x 1) (y "hello"))
    (message "x=%S y=%S" x y)))     ; → "x=1 y=\"hello\""
```

`%S` 让 y 显示带 quote—— `"hello"` 而不是 hello。这告诉你 y 是 string (而非 symbol)。

### 题 9: read environmental

```elisp
(getenv "HOME")                ; → "/home/sun"
(setenv "MY_VAR" "value")
```

`getenv`/`setenv` 操作环境变量——这些传给 subprocess (你 `make-process` 跑的命令能看到)。

### 题 10: report-emacs-bug

```
M-x report-emacs-bug RET
```

学习如何报告 bug。

即使你不发,跑一次看 Emacs 自动收集什么信息——版本、系统、最近按键、最近消息。这是好 bug 报告的标准。

---

## 自测

1. Edebug 怎么启动?
2. condition-case 语法?
3. unwind-protect 干啥?
4. 怎么 profile?
5. `M-x report-emacs-bug` 自动填什么?

**答案**:
> 1. M-x edebug-defun 在 defun 内
> 2. (condition-case VAR FORM (ERROR-TYPE BODY...)...)
> 3. 跑 BODY,无论错与否跑 CLEANUP
> 4. M-x profiler-start + report
> 5. 版本、系统信息、recent input、recent messages

---

进入 `../week-05-minibuf-commands/`。
