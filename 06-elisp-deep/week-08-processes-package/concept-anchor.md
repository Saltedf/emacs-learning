# Concept Anchor + Exercises: Week 8

## Concept Anchor: building.texi + cmdargs.texi

Emacs 的 `M-x compile` 是异步 process 的完美应用——它跑 make (或其他 build 命令),实时显示输出,允许用户继续编辑。理解它如何用 `make-process` 实现让你能写自己的"async runner"。

### building.texi

`M-x compile` 内部用 `make-process`:

```elisp
(compile "make")
;; = (make-process :name "make" :buffer "*compilation*"
;;                  :command '("make"))
```

compile 还做了更多: 设置 `compilation-mode` (让 error line 可点击)、跳转到 error 的 source、gdefine error regex。但底层就是 `make-process`——你已经学过的工具。

### cmdargs.texi

启动参数 → 命令行 → Lisp:

```elisp
(command-line-args)            ; list of strings
(argv)                         ; raw
```

Emacs 启动时,`argv` 是命令行参数 list。你可以用它写"批量处理"工具——`emacs --batch --eval "(my-process)"`。

---

## Exercises: Week 8

这周练习覆盖 process、网络、file-notify、thread、abbrev、GC——Module 6 W8 的所有 API。**实际跑每题**,因为 process 行为很难从代码预测。

### 题 1: shell-command-to-string

最简单的 process——同步跑命令,返回 string。

```elisp
(shell-command-to-string "date +%Y-%m-%d")
```

适合"快速拿命令结果"——但卡 Emacs,不适合长时间命令。

### 题 2: make-process

异步 process——立即返回,不卡 UI。

```elisp
(defun my-bg-cmd (cmd)
  (let ((buf (generate-new-buffer-name "*output*")))
    (make-process :name "bg"
                  :buffer buf
                  :command (list "bash" "-c" cmd)
                  :sentinel (lambda (proc event)
                              (when (string-match "finished" event)
                                (message "Done: %s" buf))))))
```

`:command (list "bash" "-c" cmd)` 跑 bash 执行 cmd——这让你支持 shell 特性 (pipe、glob)。

### 题 3: 网络 HTTP

手写 HTTP request——理解底层。

```elisp
(defun my-http (host path)
  (let ((proc (open-network-stream "http" "*http*" host 80)))
    (process-send-string proc
                         (format "GET %s HTTP/1.0\r\nHost: %s\r\n\r\n"
                                 path host))))
```

实战用 `url-retrieve` 或 `request` 包 (处理 redirect、cookie、HTTPS)。但理解这个让你能写自定义协议。

### 题 4: file-notify

```elisp
(defun my-watch (path fn)
  (file-notify-add-watch path '(change)
    (lambda (event) (funcall fn event))))
```

文件变化自动通知——比 timer poll 高效。可用于: 自动 reload 修改的文件、build on save。

### 题 5: thread

```elisp
(make-thread (lambda () (sleep-for 1) (message "after 1s")))
```

后台 sleep——主 thread 不卡。但 Emacs thread 是协作式,sleep-for 期间让出 CPU。

### 题 6: abbrev

```elisp
(define-abbrev global-abbrev-table "btw" "by the way")
(setq-default abbrev-mode t)
```

输入 "btw SPC" 自动变成 "by the way"。输入加速工具。

### 题 7: time format

```elisp
(format-time-string "%Y-%m-%d %H:%M:%S")
```

`%Y-%m-%d` 是 ISO date 格式——可排序。`%H:%M:%S` 是 24h 时间。

### 题 8: timer

```elisp
(run-with-timer 60 nil (lambda () (message "1 minute passed")))
```

第一参数是 delay (秒),第二是 repeat interval (nil = 不重复)。

### 题 9: env

```elisp
(getenv "HOME")
(setenv "MY_VAR" "value")
```

环境变量传给 subprocess——你 `make-process` 跑的命令看到这些。

### 题 10: system type

```elisp
(if (eq system-type 'darwin)
    (message "macOS")
  (message "Other"))
```

跨平台代码——`(system-type)` 返回 'gnu/linux、'darwin (macOS)、'windows-nt 等。

### 题 11: package descriptor

```elisp
;; my-package-pkg.el
(define-package "my-package" "1.0" "My package." '((emacs "29.1")))
```

package metadata——name、version、description、依赖。MELPA/ELPA 用。

### 题 12: GC tune

```elisp
(setq gc-cons-threshold (* 100 1024 1024))  ; 100MB
(setq gc-cons-percentage 0.5)
```

调高 GC threshold——减少 GC 频率,但单次 GC 时间长。这是性能/内存的权衡。

---

## 自测

1. `make-process` 主要参数?
2. process-sentinel 监听什么?
3. threads 是真并行吗?
4. `file-notify-add-watch` 干啥?
5. 怎么定期跑函数?

**答案**:
> 1. :name :buffer :command :filter :sentinel
> 2. process 状态变化 (start/stop/exit)
> 3. 不是 (协作式)
> 4. 监听文件变化
> 5. run-with-timer 或 run-with-idle-timer

---

进入 `../capstone.md`。
