# Exercises: Week 4 Debugging

> 12 题练习

---

### 题 1: 用 Edebug 调试
找一个有 bug 的函数,用 Edebug 单步:

```
1. 光标在 defun 内
2. C-u C-M-x (eval-defun with prefix) 或 M-x edebug-defun
3. 调用函数
4. SPC 单步
5. e EXPR RET 看变量
6. q 退出
```

记录你的发现。

Edebug 是 Elisp 最强的调试工具。一旦你习惯它,你不会回到 print debugging——因为 Edebug 让你**交互**地探索变量值,比固定 print 灵活得多。

### 题 2: condition-case

```elisp
(defun my-safe-divide (a b)
  (condition-case nil
      (/ a b)
    (arith-error 0)))
```

`(condition-case nil ...)` 不绑定错误——只捕获。如果 b=0,`(/ a 0)` 抛 arith-error,handler 返回 0。

### 题 3: 自定义 error type

自定义 error type 让你的库的 error 有 identity。`error-conditions` 设 `(my-parse-error error)` 让它继承 error——所以 generic `(error ...)` handler 也能捕获。

```elisp
(put 'my-parse-error 'error-conditions '(my-parse-error error))
(put 'my-parse-error 'error-message "Parse error")

(defun my-parse (s)
  (unless (string-match "^[0-9]+$" s)
    (signal 'my-parse-error (list s)))
  (string-to-number s))

(condition-case err
    (my-parse "abc")
  (my-parse-error
   (message "Parse failed: %s" (cadr err))))
```

### 题 4: unwind-protect

无论 do-something 是否出错,最后 "Done or errored" 都打印。这是资源管理的基础——`save-excursion` 等内置宏都是 unwind-protect 的糖。

```elisp
(defun my-with-focus-message (msg)
  (unwind-protect
      (progn
        (message msg)
        (do-something))
    (message "Done or errored")))
```

### 题 5: cl-assert

cl-assert 是 design by contract——前置条件检查。如果调用方传非 list,立即报错,而不是运行到深处莫名出错。

```elisp
(require 'cl-lib)

(defun my-only-lists (lst)
  (cl-assert (listp lst) nil "Expected list, got %s" (type-of lst))
  (length lst))
```

### 题 6: ERT

`should` 期望真,`should-not` 期望假。失败时 ERT 报告哪行、期望、实际——比手动测试快得多。

```elisp
(require 'ert)

(ert-deftest my-test ()
  (should (eq (+ 1 2) 3))
  (should-not (eq "a" "b")))

M-x ert RET my-test RET
```

### 题 7: profiler

profile 报告显示 mapcar 和 number-sequence 是 hot spot——这是预期的 (它们在循环里)。实战你会找意外 hot spot。

```elisp
(profiler-start 'cpu)
(dotimes (_ 100000)
  (mapcar #'identity (number-sequence 1 100)))
(profiler-report)
(profiler-stop)
```

### 题 8: report-emacs-bug

`C-h l` (`view-lossage`) 显示最近 300 个按键——这是 report-emacs-bug 收集的"recent input"。`C-h e` (`view-echo-area-messages`) 显示 *Messages* buffer——"recent messages"。

```
M-x report-emacs-bug RET
```

看自动填的信息,但**不发送** (除非真有 bug)。

### 题 9: 修复 byte-compile warning

很多 warning 是 bug 的早期信号:
- `free variable`: typo 或忘 require
- `unused lexical variable`: dead code
- `not known to be defined`: 忘 autoload

```
M-x byte-compile-file RET ~/.config/emacs/init.el RET
```

修复所有 warning。

### 题 10: use-package verbose

`use-package-verbose` 让 use-package 打印加载时间——你看到 "Loading org took 1.5s" 这种。这是优化启动时间的起点。

```elisp
(setq use-package-verbose t)
```

重启,看哪些包加载慢。

### 题 11: gc 监控

开启后,每次 GC 在 echo area 显示 "Garbage collector has recovered X bytes"。如果你的代码频繁 GC,这里会刷屏——是 GC 频繁的信号。

```elisp
(setq garbage-collection-messages t)
```

看 GC 何时跑。

### 题 12: 用 message 调试

`message` 是最简单的 debug——但很有效。`%s` 显示值,`%S` 显示 Lisp 语法 (string 带 quote)。

给一个函数加 message 调试,看变量值。

```elisp
(defun my-func (x)
  (message "entering my-func, x=%s" x)
  (let ((result (* x x)))
    (message "result=%s" result)
    result))
```

---

## 自测

1. Edebug 怎么进入?
2. condition-case 怎么用?
3. unwind-protect 干啥?
4. `M-x report-emacs-bug` 自动填什么?
5. profiler 怎么用?

**答案**:
> 1. M-x edebug-defun 或 C-u C-M-x
> 2. (condition-case VAR FORM (ERR-TYPE BODY...)...)
> 3. 跑 FORM,无论出错与否跑 cleanup
> 4. 版本、系统、recent input、recent messages
> 5. M-x profiler-start,跑代码,M-x profiler-report

---

进入 `../week-05-minibuf-commands/`。
