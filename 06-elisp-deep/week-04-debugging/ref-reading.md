# Week 4: Debugging — 当代码不工作时

> **Ref 章节**: debugging, edebug, errors, tips
> **目标**: 你能 debug 任何 Elisp 代码,无论自己的还是别人的

---

## 0. 当代码出错时

你已经写了 Module 3-5 的代码,经历过错误。但每次遇到错误,你可能:
- 按 `C-g` 跳过
- 用 `message` 打印调试
- 不知道从哪查

这周系统学**调试技术**。完成后你能:
- 读 backtrace,理解错误类型
- 用 Edebug 单步执行
- 用 `condition-case` 优雅处理错误
- 写 ERT 测试预防 bug
- profile 找性能瓶颈

为什么调试这么重要? 因为代码不是"写完就完"——它要持续维护。bug 总会出现,你需要工具快速诊断。业余程序员用 print 调试 (慢、不可重复),专业程序员用 Edebug + ERT (快、自动化)。

---

## 1. 第一性原理: 错误是朋友,不是敌人

Emacs 的设计哲学: 错误是反馈,不是失败。当 Emacs 报错,它在说"这里有问题,我停下来,你来诊断"。这个态度让你**欢迎**错误——它告诉你 bug 在哪,而不是默默破坏数据。

### 1.1 错误信号 (Error Condition)

Elisp 的错误分两步:

1. **signal** (raise): `(signal ERROR-SYMBOL DATA)`
2. **handler** (catch): `condition-case`

如果没有 handler,Emacs 默认:
- 显示错误信息到 echo area
- 进入 `*Backtrace*` buffer (如果 `debug-on-error` 开)
- 或终止命令

这是经典 try/catch 模式——但 Elisp 用 `signal`/`condition-case` 而非 throw/try。设计上 signal 是 non-local exit——它跳出当前 form,找到最近的 handler。

### 1.2 错误家族

Elisp 内置多种 error type——每种对应一类问题。理解 type 让你 condition-case 精确捕获。

```elisp
(arith-error)                  ; 算术 (除 0 等)
(wrong-type-argument PRED OBJECT)  ; 类型错 (期望 list,给 number)
(void-variable SYMBOL)         ; 引用未定义变量
(void-function SYMBOL)         ; 调用未定义函数
(args-out-of-range LIST/FN)    ; 参数越界
(file-error MESSAGE PATH)      ; 文件错
(search-failed PATTERN)        ; 搜索没找到
(end-of-file)                  ; read 时 EOF
(circular-list)                ; 循环 list
(invalid-read-syntax)          ; read 语法错
(wrong-number-of-arguments FN EXPECTED ACTUAL)
(error MESSAGE)                ; 通用
(quit)                         ; 用户按 C-g
```

每种 error 是一个 symbol——`arith-error`、`wrong-type-argument` 等。`(signal 'arith-error '("Division by zero"))` 抛 arith-error。`(condition-case err FORM (arith-error ...))` 捕获它。

注意层次: 大多数 error 是 `error` 的子类——`(error ...)` 捕获所有。`quit` 不属于 error family (它是 special signal),所以 condition-case 用 `quit` 单独处理。

### 1.3 错误信息读法

错误信息有固定格式——读懂它你就知道 bug 在哪。

```
(byte-code-function-p): Wrong type argument: char-or-string-p, nil
```

读法:
- 命令: `byte-code-function-p` (或宏)
- 错误类型: `wrong-type-argument`
- 期望: `char-or-string-p` (predicate,期望 char 或 string)
- 实际: `nil`

意思: 某函数把 `nil` 传给了 `byte-code-function-p` 的某个参数,但那个参数期望 char 或 string。

读懂这个格式让你瞬间知道: 是哪个函数、什么期望、实际给了什么。这比"我的代码不工作了"精确得多。

---

## 2. Backtrace Buffer

### 2.1 启用

`debug-on-error` 让任何 error 进入 backtrace——开发时必开。

```elisp
(setq debug-on-error t)        ; 任何 error 进 backtrace
(setq debug-ignored-errors '(error))  ; 反向,忽略某些
```

`debug-ignored-errors` 是反向——某些 error 你不想进 debugger (如 user-error)。`(setq debug-ignored-errors '(user-error))` 让 user-error 不进 backtrace。

或在 init 启动时:
```bash
emacs --debug-init
```

`--debug-init` 是临时开 debug-on-error——专门用于调试 init.el 错误。

### 2.2 看 backtrace

错误时显示 `*Backtrace*` buffer:

```
Debugger entered--Lisp error: (wrong-type-argument numberp nil)
  +(1 nil)
  (let ((x 1) (y nil)) (+ x y))
  my-func()
  funcall(my-func)
  command-execute(my-func)
```

读法:
- 错误类型: wrong-type-argument
- 期望 numberp (数字),实际 nil
- 最里层: `+(1 nil)` 加 1 和 nil → 错
- 调用栈: my-func → funcall → command-execute

backtrace 从最里层 (error 发生处) 往外读——`+(1 nil)` 是直接的出错点,外层是调用栈。读 backtrace 是 debug 的核心技能——你必须能从这堆 form 找到 bug。

### 2.3 在 backtrace 里操作

```
q          退出 backtrace
RET        在 source 打开
e          eval expression (在 backtrace 的 frame 上下文)
d          step in (Edebug 风格)
b          backtrace frame (show locals)
```

backtrace 不只是看——你可以交互。`e` eval expression 在某 frame 的 context——让你检查变量值 (在那个调用栈层)。`d` 进入 Edebug-style step——单步从错误点继续。

---

## 3. condition-case (异常处理)

### 3.1 基础

`condition-case` 是 Elisp 的 try/catch——捕获特定 error type,跑 handler。

```elisp
(condition-case VAR
    PROTECTED-FORM
  (ERROR-CONDITION-1
   HANDLER-BODY-1...)
  (ERROR-CONDITION-2
   HANDLER-BODY-2...)
  ...)
```

- VAR: 绑定到错误信息 (condition)
- PROTECTED-FORM: 可能出错的代码
- ERROR-CONDITION: error type symbol
- HANDLER-BODY: 匹配时跑

这是 CL 的 handler-case 等价物。VAR 是 nil 表示"不绑定错误信息"——只 catch 不看 detail。

### 3.2 例子

```elisp
(condition-case err
    (/ 1 0)
  (arith-error
   (message "Math error: %s" (cadr err))
   0))
;; → 0, 显示 "Math error: Division by zero"
```

`err` 绑定到 `(arith-error Division by zero)`。
`car` 取 error-type,`cadr` 取 message。

注意 `cadr err` 是 `(car (cdr err))`——`(arith-error "Division by zero")` 的第二个元素 (message)。这是 condition 信息结构——`(error-type . data)`,data 通常是 message。

### 3.3 多个 handler

```elisp
(condition-case err
    (do-something)
  (file-error (message "File problem: %s" err))
  (wrong-type-argument (message "Type problem: %s" err))
  (error (message "Other error: %s" err)))    ; 通用
```

`error` 匹配所有 error (它是 root)。

handler 顺序匹配——先 specific 再 generic。把 `error` 放最后,作为"兜底"。

### 3.4 没有 VAR

```elisp
(condition-case nil
    (/ 1 0)
  (arith-error 'failed))
;; → 'failed
```

第一个参数 nil 表示"不绑定错误信息"——只关心捕获,不需要看 detail。

### 3.5 实战: 安全读文件

这是 condition-case 的经典用例——读文件,失败时优雅降级。

```elisp
(defun my-safe-read (path)
  "Read sexp from PATH, return nil on error."
  (condition-case nil
      (with-temp-buffer
        (insert-file-contents path)
        (read (current-buffer)))
    (error nil)))
```

文件可能不存在 (file-error)、内容可能损坏 (end-of-file、invalid-read-syntax)、可能没权限。`condition-case nil ... (error nil)` 一律捕获,失败返回 nil。调用方检查 nil 就知道失败。

### 3.6 unwind-protect (cleanup)

`unwind-protect` 是 try/finally——保证 cleanup 跑,无论是否出错。

```elisp
(unwind-protect
    (do-something-may-error)
  (cleanup-no-matter-what))
```

无论出错与否,cleanup 都跑。

例: 临时改 buffer,确保恢复:

```elisp
(unwind-protect
    (progn
      (read-only-mode -1)
      ;; 编辑
      )
  (read-only-mode 1))
```

或 `with-silent-modifications`:

```elisp
(let ((modified (buffer-modified-p)))
  (unwind-protect
      (progn
        (set-buffer-modified-p nil)
        ...)
    (set-buffer-modified-p modified)))
```

这个 pattern——保存状态、修改、跑 body、恢复——是 `with-X` 宏的基础。`save-excursion`、`save-current-buffer`、`save-restriction` 都是 unwind-protect 的糖。

---

## 4. Edebug: 单步调试器

### 4.1 启动

Edebug 是 Emacs 自带的源码级调试器——你 instrument 一个 defun,下次调用时进入单步模式。

光标在 defun 内部:
```
M-x edebug-defun RET
;; 或 C-u M-C-x (eval-defun with prefix)
```

函数被 instrumented——Emacs 把 break point 插到每个 form。下次调用,进入 Edebug mode。

### 4.2 在 Edebug 里

```
SPC        下一步 (step)
n          next (skip 调用)
g          go (跑到 breakpoint)
G          go 不停 (到结束)
b          设 breakpoint
u          移除 breakpoint
B          next breakpoint
d          step in
o          step out (跳出当前)
t          trace (跑,每步停)
T          fast trace
c          continue (不停)
e          eval expression
i          instrument (调试中的函数)
h          here (跑到 point)
f          forward-sexp
q          退出 Edebug
?          help
```

Edebug 的核心是 SPC (下一步) 和 e (eval expression)。SPC 让你逐 form 执行——看到代码顺序、变量值。e 让你在调试时检查任意表达式。

### 4.3 看变量

```
i X        inspect X (in *edebug* buffer)
e EXPR RET eval EXPR,显示在 echo area
```

`e EXPR RET` 让你输入任意 form,在当前调试上下文 eval。例如 `(e x)` 看变量 x 的值,`(e (buffer-size))` 看 buffer 大小。

### 4.4 看 backtrace

```
d          step into stack
```

`d` 让你 step into——进入子函数的 Edebug (如果它也被 instrumented)。

### 4.5 条件 breakpoint

```
B          设条件 breakpoint
;; 输入条件表达式
;; 当条件为 t 时停
```

条件 breakpoint 极强——你可以设"只在 i=5 时停",避免逐次 step。例如循环里调 bug,你知道是某次迭代出错——设条件 breakpoint `i = N`,跑到那里自动停。

### 4.6 实战: 调试 my-func

```elisp
(defun my-func (lst)
  (let ((sum 0))
    (dolist (x lst)
      (setq sum (+ sum x)))
    sum))

(my-func '(1 2 3))
```

光标在 my-func 内,`M-x edebug-defun`。
然后 `(my-func '(1 2 3))`,进入 Edebug。
按 SPC 单步。

你会看到 sum 从 0 变成 1、3、6。如果 lst 含 nil (bug),`(+ sum nil)` 报错,你在 Edebug 看到具体哪步出错——比 backtrace 详细得多。

---

## 5. Emacs Lisp Error Handling

### 5.1 signal (raise)

`signal` 是底层抛错——`(signal ERROR-SYMBOL DATA)`。

```elisp
(signal 'wrong-type-argument (list 'stringp 5))
;; → error: (wrong-type-argument stringp 5)
```

`DATA` 是 list——错误信息。`(list 'stringp 5)` 因为 wrong-type-argument 的 data 格式是 `(predicate value)`。

通常你不用 `signal` 直接——用 `error` 或 `user-error` 更方便。

### 5.2 error / user-error

`error` 是通用抛错,`user-error` 是"用户错误" (不严重,不被 debug-on-error 捕获)。

```elisp
(error "Something went wrong: %s" detail)
;; → error

(user-error "You pressed the wrong key")
;; → user-error (不被 debug-on-error 捕获)
```

`user-error` 用于用户错误 (不严重),`error` 用于真正错误。

区分的意图: `debug-on-error` 是开发工具——它捕获 error 进 backtrace。但 `user-error` (如"你按了无效键") 不应该进 backtrace——那是正常用户交互,不是 bug。所以 Emacs 把 user-error 排除在 debug-on-error 外。

### 5.3 cl-assert

`cl-assert` 是断言——如果条件假,抛错。

```elisp
(require 'cl-lib)

(cl-assert (stringp s) nil "Expected string, got %s" (type-of s))
;; 如果 (stringp s) 为 nil,signal error
```

`nil` 是"不显示 backtrace" (t 会进 debugger),后面是 message format。

cl-assert 是设计 by contract 的工具——你声明"我假设 s 是 string",如果违反,立即报错。这比"运行到深处莫名出错"好。

### 5.4 自定义 error type

你可以定义自己的 error type——让你的库的 error 有 identity,调用方可以精确捕获。

```elisp
(put 'my-error 'error-conditions '(my-error error))
(put 'my-error 'error-message "My custom error")

(signal 'my-error '("details"))
;; 在 condition-case:
(condition-case err
    (something)
  (my-error (message "Got my-error: %s" err)))
```

`error-conditions` 是 list——`(my-error error)` 表示 my-error 是 error 的子类。这样 `(condition-case ... (error ...))` 也能捕获 my-error (因为它是 error 的子)。

---

## 6. message 调试

最简单也最常用——printf debugging。Elisp 的 printf 是 `message`。

### 6.1 简单 message

```elisp
(message "x = %s" x)
(message "list = %S" lst)     ; %S 用 Lisp 语法 (字符串带 quote)
```

`%s` 显示值 (string 不带 quote),`%S` 用 Lisp 语法 (string 带 quote)。调试复杂结构用 `%S`——`(message "lst = %S" '(a "b" c))` → `"lst = (a \"b\" c)"`。

### 6.2 message 到 *Messages*

```elisp
(let ((message-log-max t))    ; 不限制 log size
  (message "..."))
```

`*Messages*` buffer 默认只保留最近 100 条 (message-log-max)。设 t 让它无限——长循环调试时不丢失早期 message。

### 6.3 print 到 buffer

如果 message 太多 (循环里),echo area 滚动看不清——直接写 buffer。

```elisp
(with-current-buffer (get-buffer-create "*my-debug*")
  (goto-char (point-max))
  (insert (format "x = %S\n" x)))
```

这样你能 `M-x switch-to-buffer *my-debug*` 看完整 log。

### 6.4 toggle debug-on-message

```elisp
(setq debug-on-message "my-")    ; message 含 "my-" 时进 debugger
```

适合"奇怪的地方打印了什么?"的调试。

如果你不知道哪个函数打印了某 message,设 `debug-on-message` 为该 message 的 regex——下次打印时进 backtrace,看调用栈。

---

## 7. Tips (tips.texi)

### 7.1 编码风格

```elisp
(defun my-func (arg)
  "Documentation.
ARG should be a string."
  (check-type arg string)
  ...)

;; 命名:
my-foo              ; 你的 (前缀)
foo-bar             ; 通用
foo-internal        ; 内部 (--)
*my-var*            ; 特殊变量 (动态)

;; indent:
(let ((x 1)
      (y 2))
  (+ x y))

;; 函数 indent:
(defun my-func ()
  (foo)
  (bar))
```

命名约定让你的代码融入 Emacs 生态——所有内置包用 `my-` 前缀 (你自己的),`--` 后缀 (内部)。这让用户知道哪些是 public API、哪些是内部 (可能改)。

### 7.2 性能

性能优化是细节——但一些 pattern 立即让你快 10x。

```elisp
;; 慢:
(setq lst (append lst (list new)))     ; O(n) 每次 append
;; 快:
(push new lst)                          ; O(1) cons
;; 最后 nreverse

;; 慢:
(while ... (setq result (cons x result)))
;; 快:
(mapcar FN LIST)

;; 慢:
(re-search-forward "a" nil t)
(re-search-forward "b" nil t)         ; 两次搜索
;; 快:
(re-search-forward "[ab]" nil t)      ; 一次

;; 慢:
(eq "abc" "abc")                       ; string eq 不可靠
;; 正确:
(equal "abc" "abc")
```

append 在末尾加是 O(n)——每次都遍历整个 list。push 在头部是 O(1)——只 cons 一次。如果你要末尾加,用 push + 最后 nreverse。

`(re-search-forward "[ab]")` 用一次 regex 找 a 或 b,比两次 search 快——regexp 是高度优化的。

### 7.3 常见陷阱

```elisp
(setq x 5)           ; 全局
(let ((x 10)) ...)   ; 局部

(setq my-list nil)
(add-to-list 'my-list 1)    ; add-to-list 第一个参数是引用

;; 弱 eq for string:
(eq "foo" "foo")             ; → 不一定 t
(equal "foo" "foo")          ; → t

;; nil 检查:
(if list ...)                ; nil 是 false
;; 但:
(if "empty" ...)             ; "" 是 true!
(if 0 ...)                   ; 0 是 true!
```

最隐蔽的陷阱: Elisp 里**只有 nil 是 false**。空 string `""`、零 `0`、空 vector `[]` 都是 true。这和 Python、JS 不同 (它们空 collection 是 false)。`(if (string-empty-p s) ...)` 用 predicate,不要 `(if s ...)`。

---

## 8. Profiling

性能问题先 profile——不要瞎猜。

### 8.1 Edebug profile

```
M-x edebug-eval-top-level-form
;; 显示每个 form 用时
```

Edebug 不只调试,也能 profile——显示每个 form 的执行时间。简单但有用。

### 8.2 elp (Emacs Lisp Profiler)

```elisp
(require 'elp)
(elp-instrument-package "my-")
M-x my-func
M-x elp-results
```

显示每个函数调用次数 + 总时间。

elp instrument 一个 prefix 的所有函数——加 wrapper 记录调用。跑后 `elp-results` 显示表格。

### 8.3 profiler (内置 Emacs 27+)

```elisp
(profiler-start 'cpu)
;; 跑你的代码
(profiler-report)
(profiler-stop)
```

显示调用图。

内置 profiler 比 elp 强——它采样 CPU 时间,显示 call graph。`profiler-start 'cpu` 启动 CPU profile,`'memory` 启动内存 profile。

---

## 9. ERT (测试)

(Module 3 用过,Module 7 详细)

ERT 是 Emacs 的测试框架——类似 Python 的 unittest、JS 的 jest。

```elisp
(require 'ert)

(ert-deftest my-test ()
  (should (= (+ 1 2) 3))
  (should (null nil))
  (should-error (error "foo")))

M-x ert-run-tests-interactively RET my-test RET
```

`should` 是 assert——失败时 ERT 报告哪行、什么期望、什么实际。`should-error` 期望 form 抛错——如果没抛,失败。

ERT 是预防 bug 的工具——你写测试,每次改代码跑测试,确保没破坏。

---

## 10. 实战练习

### Ex 4.1: 用 Edebug 调 my-todo

```
1. 打开 my-todo.el
2. 光标在 my-todo-add-internal 内
3. M-x edebug-defun
4. M-x my-todo RET
5. a 添加,触发函数
6. SPC 单步,看变量
```

这是 Edebug 的标准 workflow——instrument defun,触发,step。

### Ex 4.2: 用 condition-case 处理 file-error

```elisp
(defun my-safe-load (path)
  (condition-case err
      (with-temp-buffer
        (insert-file-contents path)
        (read (current-buffer)))
    (file-error
     (message "Cannot load %s: %s" path (cadr err))
     nil)))

(my-safe-load "/nonexistent/path")    ; → nil, message
```

`(cadr err)` 取错误 message——file-error 的 data 是 `(message path)`。

### Ex 4.3: 用 profiler

```elisp
(profiler-start 'cpu)
(dotimes (_ 1000)
  (my-slow-function))
(profiler-report)
(profiler-stop)
```

profile 后看 report——找到 hot spot (CPU 时间最多的函数),优化它。

### Ex 4.4: 用 elp

```elisp
(require 'elp)
(elp-instrument-package "my-")
M-x my-todo RET
M-x elp-results
```

elp 比 profiler 简单——只统计函数级,不显示 call graph。适合"我的 package 哪个函数慢"。

### Ex 4.5: 写 ERT

```elisp
(ert-deftest my-todo-add-test ()
  (let ((my-todo-list nil))
    (my-todo-add-internal "test" 1 nil)
    (should (= 1 (length my-todo-list)))
    (should (equal "test" (my-todo--item-text (car my-todo-list))))))
```

测试 setup (let my-todo-list nil)、action (add)、assert (length 和 text)——这是 AAA pattern (Arrange-Act-Assert)。

---

## 11. 自测

1. `(signal 'wrong-type-argument ...)` 干啥?
2. `condition-case` 怎么用?
3. Edebug 怎么启动?
4. `user-error` 和 `error` 区别?
5. 怎么 profile?

**答案**:
> 1. 抛 wrong-type-argument 错误
> 2. (condition-case VAR FORM (ERROR-TYPE BODY...)...)
> 3. M-x edebug-defun 在 defun 内
> 4. user-error 是用户错误,不被 debug-on-error;error 是真正错误
> 5. M-x profiler-start + M-x profiler-report

---

## 12. 下一步

进入 `concept-anchor.md` + `exercises.md`,然后 `../week-05-minibuf-commands/`。
