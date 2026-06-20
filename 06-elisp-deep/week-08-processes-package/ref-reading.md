# Week 8: Processes + OS + Package + Internals

> **Ref 章节**: abbrevs, threads, processes, os, package, internals
> **目标**: 跑外部进程、多线程、发布包、理解 Emacs 内部

这周是 Module 6 的最后一周——把 Emacs 和外部世界连接。processes 让 Emacs 跑外部命令 (git、make、language server);threads 让 Emacs 并发;package 让你发布代码;internals 让你理解 Emacs 怎么工作。

这是从"Elisp 用户"到"Emacs 平台开发者"的跃迁——你能写集成 LSP 的 IDE、写自己的 git client、写后台同步工具。Module 6 学的所有东西 (数据类型、宏、buffer、display) 在这周汇聚成"真实软件"——能和 OS、网络、其他进程交互的应用。

---

## 1. Processes (processes.texi)

### 1.1 什么是 process

process 是 Emacs 异步编程的核心——你启动 process,设 filter (处理输出) 和 sentinel (处理状态变化),然后继续做别的。Emacs 在 process 输出时调 filter,在状态变化时调 sentinel。

这是事件驱动编程——你不需要 busy-wait,Emacs 在 event 到来时通知你。这让 Emacs 能同时跑多个 process,UI 不卡。

Emacs process = 一个外部程序的连接 (subprocess)。
可以是:
- shell command
- 长期程序 (language server, REPL)
- 网络连接

### 1.2 同步 shell

最简单的 process 是 `shell-command`——同步跑,等结果。但同步 shell 会卡 Emacs——如果命令慢 (如 `make`),Emacs 完全 freeze。所以长时间命令用异步 (1.4)。

```elisp
(shell-command COMMAND &optional OUTPUT-BUFFER ERROR-BUFFER)
(shell-command-to-string COMMAND)

(shell-command "ls")
;; 输出到 *Shell Command Output* buffer

(shell-command-to-string "date")
;; → "Mon Jun 19 ..."
```

### 1.3 call-process

`call-process` 是底层同步 process——比 shell-command 灵活,不走 shell (无 glob、无 pipe),直接 exec 程序。

```elisp
(call-process PROGRAM &optional INFILE BUFFER DISPLAY &rest ARGS)
(call-process "ls" nil t nil "-l")
;; 把 ls -l 输出插入当前 buffer
```

`BUFFER` 可以是:
- `nil`: 丢弃输出
- `t`: 当前 buffer
- buffer name
- `(list BUFFER-NAME ERROR-BUFFER-NAME)`

### 1.4 make-process (异步)

`make-process` 是异步 process 的现代 API——比老 `start-process` 灵活。立即返回 process 对象,Emacs 后台跑命令。

```elisp
(make-process
 :name "my-proc"
 :buffer "*my-proc*"
 :command '("ls" "-l")
 :connection-type 'pipe
 :filter (lambda (proc string) ...)
 :sentinel (lambda (proc event) ...)
 :noquery nil)

;; 或老 API:
(start-process NAME BUFFER PROGRAM &rest ARGS)
(start-process "my-proc" "*my-proc*" "ls" "-l")
```

### 1.5 process-filter

filter 函数接收输出——这是你"处理 process 数据"的地方。注意 string 可能不完整——process 可能分多次 flush,要自己 buffer 直到 newline。

```elisp
(set-process-filter
 PROC
 (lambda (proc string)
   (with-current-buffer (process-buffer proc)
     (goto-char (point-max))
     (insert string))))
```

### 1.6 process-sentinel

sentinel 处理状态变化——process 开始、停止、退出。这是"完成时通知"的标准位置——build 完成打开 buffer、失败时报警。

```elisp
(set-process-sentinel
 PROC
 (lambda (proc event)
   (cond
    ((string-match "finished" event)
     (message "Process finished"))
    ((string-match "exited" event)
     (message "Process exited abnormally"))
    ((string-match "killed" event)
     (message "Process killed")))))
```

### 1.7 process-input

你可以向 process 发送输入——控制它的 stdin。这让你和 process 双向通信——发命令,收响应。REPL 集成 (python、shell) 用这个。

```elisp
(process-send-string PROC STRING)
(process-send-region PROC START END)
(process-send-eof PROC)
```

### 1.8 process 状态

```elisp
(process-status PROC)          ; 'run / 'exit / 'signal / ...
(process-exit-status PROC)     ; 退出码
(process-list)                 ; 所有 process
(process-live-p PROC)
(get-process NAME)
```

### 1.9 实战: 跑 git status

```elisp
(defun my-git-status-async ()
  (interactive)
  (let ((buf (get-buffer-create "*git-status*")))
    (with-current-buffer buf
      (erase-buffer))
    (make-process
     :name "git-status"
     :buffer buf
     :command '("git" "status")
     :sentinel (lambda (proc event)
                 (when (string-match "finished" event)
                   (display-buffer buf))))))

(my-git-status-async)
```

### 1.10 网络连接

process 不只是 subprocess——也包括网络连接。`open-network-stream` 创建 TCP 连接,返回 process 对象,和 subprocess 一样用 filter/sentinel。

```elisp
(open-network-stream NAME BUFFER HOST SERVICE &optional TYPE)
(make-network-process ...)
```

例子: 连 HTTP server。这是手写 HTTP client——实战你用 `url-retrieve` 或 `request` 包 (它们处理 redirect、cookie、header parsing)。但理解底层让你能写自定义协议 (如 SMTP、IRC)。

```elisp
(let ((proc (open-network-stream "my-conn" "*my-conn*" "example.com" 80)))
  (set-process-filter proc
                      (lambda (proc string)
                        (message "Got: %s" string)))
  (process-send-string proc "GET / HTTP/1.0\r\n\r\n"))
```

---

## 2. Operating System Interface (os.texi)

Emacs 提供 OS 接口——环境变量、时间、文件 notify、user info。这些让 Emacs 能"融入"系统,不只是文本编辑器。

### 2.1 命令行 args

```elisp
(command-line)                 ; 处理命令行
(command-line-args)            ; list of args
(argv)                         ; raw argv

(argi)                         ; 处理 -arg (旧 API)
```

### 2.2 环境变量

`getenv`/`setenv` 操作环境变量。这些传给 subprocess——你 `make-process` 跑的命令看到这些。

```elisp
(getenv "PATH")
(setenv "PATH" "/new/path")
(environment)                  ; 所有环境变量
```

### 2.3 时间

Emacs 内部用 list 表示时间 (high/low/usec/psec)。`float-time` 返回 epoch 秒 (float,带小数)——适合测量代码时间。

```elisp
(current-time)                 ; 当前时间
(current-time-string)          ; "Mon Jun 19 20:55:00 2026"
(current-time-zone)
(float-time)                   ; epoch 秒
(format-time-string "%Y-%m-%d %H:%M:%S")
(decode-time TIME)
(encode-time SECONDS MINUTES HOUR DAY MONTH YEAR)
(time-add A B)
(time-subtract A B)
(time-less-p A B)
```

### 2.4 文件通知

`file-notify-add-watch` 是内核级文件 watch——比 timer poll 高效。Linux 用 inotify,macOS 用 kqueue,Windows 用 ReadDirectoryChangesW——平台无关 API。

```elisp
(file-notify-add-watch FILE FLAGS CALLBACK)
;; 监听文件变化

(let ((desc (file-notify-add-watch
             "~/important.txt"
             '(change)
             (lambda (event)
               (message "File changed: %S" event)))))
  ;; ...
  (file-notify-rm-watch desc))
```

### 2.5 user info

```elisp
(user-real-login-name)
(user-login-name)
(user-full-name)
(user-real-uid)
(user-uid)
(user-real-gid)
(user-gid)
(system-name)
(system-users)
```

### 2.6 排版

```elisp
(system-type)                  ; 'gnu/linux / 'darwin / 'windows-nt
(window-system)                ; 'x / 'w32 / 'ns / nil (TTY)
(display-pixel-width)
(display-pixel-height)
(framep (selected-frame))
```

---

## 3. Threads (threads.texi)

### 3.1 Emacs 26+ 支持 threads

Emacs 26 (2018) 加入 thread 支持——让 Emacs 能并发跑代码。但 Emacs thread 是**协作式**——不抢占 (见 3.5)。

```elisp
(make-thread (lambda () (message "in thread")))
(current-thread)
(all-threads)
(thread-name THREAD)
(thread-alive-p THREAD)
(thread-join THREAD)
(thread-signal THREAD ERROR-SYMBOL DATA)
```

### 3.2 mutex

```elisp
(make-mutex)
(mutex-lock MUTEX)
(mutex-unlock MUTEX)
(with-mutex MUTEX BODY...)
```

### 3.3 condition-variable

```elisp
(make-condition-variable MUTEX)
(condition-wait CONDVAR)
(condition-notify CONDVAR &optional ALL)
(condition-broadcast CONDVAR)
```

### 3.4 实战

```elisp
(defun my-bg-task ()
  (let ((mutex (make-mutex)))
    (make-thread
     (lambda ()
       (sleep-for 5)
       (with-mutex mutex
         (message "Background done"))))))

(my-bg-task)
```

### 3.5 限制

Emacs threads 是 **cooperative** (协作式):
- 不抢占
- 只在 Lisp 调用点切 (Q)
- 不是真并行

适合 I/O (processes, network),不适合 CPU 密集。

为什么协作式? 因为 Emacs 的内部数据结构 (buffer、obarray) 不是 thread-safe——真并行需要全局锁,代价大。协作式让 Lisp 代码不需要锁访问 buffer (因为不会并发执行)。但代价: CPU 密集任务 (如 sort 大 list) 仍卡 Emacs。

---

## 4. Abbrevs (abbrevs.texi)

### 4.1 第一性原理: 为什么用 abbrev

打字慢是程序员的最大成本——每天敲的代码 80% 是重复模式 (`function`、`return`、`import`、`if (x == nil)` 等)。如果每次只需敲 2 个字符就能展开成 20 字符的完整模板,你每天省下几万次按键。

这就是 abbrev 的核心: **把"常用片段"绑定到"短缩写"**。输入 `def` + SPC,自动变 `def`。这不是模板系统 (那是 yasnippet),它是**最低成本的输入加速**——零配置、buffer-local、按 mode 切换、自动持久化。

为什么 abbrev 比模板 (yasnippet/tempo) 轻? 模板系统需要你定义位置 ($1、$2)、嵌套、字段——复杂。abbrev 就是一个字符串替换——简单,但 90% 的提速场景它都覆盖。VSCode 的 "User Snippets" 类似,但 Emacs abbrev **早在 1985 年就有**,是 Lisp 文化的"小而美"哲学典范。

### 4.2 abbrev table

abbrev 存在 **abbrev table** 里——一个 obarray (符号表),每个 abbrev 名是一个 symbol,值是展开文本。每个 major mode 有自己的 abbrev table (`python-mode-abbrev-table`、`c-mode-abbrev-table`),还有全局 table (`global-abbrev-table`)。

```elisp
(make-abbrev-table &optional PROPS)       ; 创建新 table
(abbrev-table-p OBJECT)                    ; 是 abbrev table?
(clear-abbrev-table TABLE)                 ; 清空
(copy-abbrev-table TABLE)                  ; 复制
(define-abbrev-table TABNAME DEFINITIONS &optional DOCSTRING &rest PROPS)
```

`define-abbrev-table` 是最常用——一次定义多个 abbrev:

```elisp
(define-abbrev-table 'my-mode-abbrev-table
  '(("foo" "function")
    ("bar" "beginning-of-line")
    ("ifn" "if (nil)" my-ifn-hook))
  "Abbrevs for my-mode."
  :regexp "\\(?:^\\|[^a-zA-Z0-9_]\\)\\([a-zA-Z0-9_]+\\)")
```

每个 entry 是 `(NAME EXPANSION &optional HOOK &rest PROPS)`——HOOK 是展开时跑的函数 (比如加 timestamp)。

### 4.3 define-abbrev

单条 abbrev 定义:

```elisp
(define-abbrev TABLE NAME EXPANSION &optional HOOK &rest PROPS)
```

PROPS 控制 abbrev 行为:
- `:count` — 展开 counter
- `:system` — 系统内置 (用户不能改)
- `:enable-function` — 控制何时启用

```elisp
(define-abbrev global-abbrev-table "btw" "by the way")
(define-abbrev global-abbrev-table "ih" "Emacs Lisp Intro")
(define-abbrev python-mode-abbrev-table "df" "def ")
(define-abbrev text-mode-abbrev-table "teh" "the")
```

`global-abbrev-table` 的 abbrev 在所有 mode 生效;`<mode>-abbrev-table` 只在该 mode 生效。这让你 per-mode 定制——Python 的 `df` 不影响 C 的 `df`。

### 4.4 abbrev expansion 触发

默认 abbrev 在输入 word character 后按 SPC、TAB、RET 自动展开。这由 `abbrev-mode` 控制。

```elisp
(setq-default abbrev-mode t)     ; 所有 buffer 默认开
(setq save-abbrevs t)            ; 退出时保存到 ~/.emacs.d/abbrev_defs
```

`abbrev-mode` 是 minor mode——开它,abbrev 自动展开。`save-abbrevs` 设 `t` 让 Emacs 退出时自动保存你交互加的 abbrev (`C-x a i g` 加的)。

手动展开命令:
```elisp
(expand-abbrev)                  ; 在 point 处展开
(abbrev-prefix-mark)             ; 标记前缀 (不展开)
(unexpand-abbrev)                ; 撤销最近展开
```

### 4.5 abbrev-mode

`abbrev-mode` 是 minor mode,buffer-local。开它,abbrev 工作;关它,输入 `btw SPC` 不展开。

`M-x abbrev-mode` 切换。或在 init.el:

```elisp
(setq-default abbrev-mode t)
(add-hook 'text-mode-hook #'abbrev-mode)
(add-hook 'prog-mode-hook #'abbrev-mode)
```

### 4.6 用户命令

```
C-x a i g     add-global-abbrev          加全局 abbrev
C-x a i l     add-mode-abbrev            加当前 mode abbrev
C-x a e       expand-abbrev              手动展开
M-/           dabbrev-expand             动态补全 (见 4.8)
C-M-/         dabbrev-completion         列出所有候选
```

`C-x a i g` 交互式加: 输入缩写,然后输入展开。Emacs 自动存到 abbrev_defs。

### 4.7 动态 abbrev (dabbrev)

静态 abbrev 是你预定义的。**dabbrev** (dynamic abbrev) 是 Emacs 自动从当前 buffer 和其他 buffer 扫描,找匹配前缀的 word——`M-/` 触发。

```
你打 "beg"  按 M-/  → "beginning-of-line"  (从当前 buffer 找到)
再按 M-/    → "beginning-of-buffer"        (下一个候选)
```

dabbrev 不需要预定义——它扫描 buffer 找你之前打过的词。这比静态 abbrev 更智能,因为它**学习你已写的内容**。

```elisp
(dabbrev-expand)                       ; M-/ 默认
(dabbrev-completion)                   ; C-M-/ 列出所有
(setq dabbrev-case-fold-search t)      ; 大小写不敏感
(setq dabbrev-upcase-means-case-search t)  ; 大写开头才 case-sensitive
```

`dabbrev-check-all-buffers` 控制是否扫描所有 buffer。默认扫几个相关 buffer,够用。

### 4.8 持久化 (abbrev_defs)

你交互加的 abbrev 自动存到 `~/.emacs.d/abbrev_defs`。下次启动 Emacs 自动加载。

```elisp
(quietly-read-abbrev-file)             ; 启动时加载
(write-abbrev-file)                    ; 手动保存
(edit-abbrevs)                         ; 编辑所有 abbrev (列表 UI)
```

`M-x edit-abbrevs` 是 UI——列出所有 abbrev table,你能编辑、删除、添加。改完 `C-c C-c` 应用。

### 4.9 创造性用法 (5+)

**1. 公司邮箱 / 签名自动展开**:
```elisp
(define-abbrev global-abbrev-table "mysig"
  (concat "John Doe\njohn@example.com\n"
          (format-time-string "Sent: %Y-%m-%d")))
```
每次输入 `mysig SPC` 自动插入完整签名 + 当前日期。

**2. 代码片段库**:
```elisp
(define-abbrev python-mode-abbrev-table
  "main" "if __name__ == \"__main__\":\n    ")
```
省下重复输入 boilerplate。

**3. 长 URL / 文件路径**:
```elisp
(define-abbrev global-abbrev-table "ghrepo"
  "https://github.com/myname/myrepo/")
```

**4. 时间戳展开 (用 hook)**:
```elisp
(define-abbrev global-abbrev-table "now" ""
  (lambda () (insert (format-time-string "%Y-%m-%d %H:%M"))))
```
HOOK 在展开时跑——插入当前时间。这让 abbrev 不仅是字符串替换,而是**带逻辑的 macro**。

**5. 多语言切换**:
```elisp
(define-abbrev global-abbrev-table "helo" "Hello, world!")
(define-abbrev global-abbrev-table "你好" "你好,世界!")
```
中英文混用。

**6. 自纠正 (typo 库)**:
```elisp
(define-abbrev global-abbrev-table "teh" "the")
(define-abbrev global-abbrev-table "recieve" "receive")
(define-abbrev global-abbrev-table "definately" "definitely")
```
你常拼错的词自动纠正——比 spell-check 主动。

**7. 不同 mode 不同 abbrev table**:
```elisp
(define-abbrev c-mode-abbrev-table "inc" "#include <stdio.h>")
(define-abbrev elisp-mode-abbrev-table "rq" "(require '")
```
Per-mode 上下文化。

**8. abbrev 转 yasnippet**:
abbrev 加 hook 能模拟 yasnippet——展开后定位到 placeholder (用 `(skip-syntax-forward ...)`),用户输入下一个。

### 4.10 陷阱 (3+)

**陷阱 1: abbrev 持久化路径**。默认 `abbrev-file-name` 是 `~/.emacs.d/abbrev_defs`——但如果你用 use-package 管理 init,可能不知道这个文件存在。`save-abbrevs` 设 `silently` 或 `t`,退出时自动写,你看不到 prompt。要明确路径: `(setq abbrev-file-name (expand-file-name "abbrev_defs" user-emacs-directory))`。

**陷阱 2: 局部 vs 全局冲突**。如果全局 abbrev table 有 `df → different`,Python abbrev table 有 `df → def`,**Python mode 内 Python 的 wins** (局部优先)。但如果你在 Python 之外期望全局 `df`,可能困惑。解决: 命名约定——mode-specific abbrev 用 mode-specific 名 (如 `pydf`)。

**陷阱 3: abbrev 在 read-only buffer 不展开**。这是设计——abbrev 展开要修改 buffer,read-only 拒绝。但 minibuffer、echo area 也算 read-only,你在那输入 abbrev 不会展开。如果你期望 abbrev 在 minibuffer 工作,会失望。

**陷阱 4: abbrev-mode 在某些 mode 默认关**。比如 `fundamental-mode` 默认不开 abbrev-mode。如果你的 init 没设 `setq-default abbrev-mode t`,会困惑"为什么有时 abbrev 工作,有时不"。

**陷阱 5: case-fold 让 "I" 不展开**。默认 abbrev 大小写敏感——`I` 和 `i` 不同。如果你定义 `i → interesting`,输入 `I SPC` 不会展开。要 case-insensitive: `(setq abbrev-case-fold-search t)`。

**陷阱 6: hook 多次跑**。abbrev 的 HOOK 在每次展开跑——如果你 hook 改全局状态 (如 counter),展开 N 次 counter 加 N。要小心副作用。

---

## 5. Package (package.texi)

(Module 7 详细)

### 5.1 写包

```elisp
;;; my-package.el -*- lexical-binding: t; -*-

;;; Commentary:
;; My package.

;;; Code:

;;;###autoload
(defun my-package-func ()
  "...")

(provide 'my-package)
;;; my-package.el ends here
```

### 5.2 package descriptor

`my-package-pkg.el`:

```elisp
(define-package "my-package" "1.0" "My package."
  '((emacs "29.1")))
```

### 5.3 发布

- MELPA: 加 recipe 到 melpa/melpa
- NonGNU ELPA: 加到 elpa-admin

---

## 6. Emacs Internals (internals.texi)

### 6.1 C core

Emacs 核心 C 代码:
- alloc.c (内存管理)
- buffer.c (buffer)
- data.c (Lisp types)
- eval.c (eval)
- keyboard.c (输入)
- process.c (process)
- ...

### 6.2 garbage collection

Emacs 的 GC 是 mark-and-sweep,1985 年 Stallman 自己写的。这个 GC 简单但有"stop-the-world"问题——大 GC 时 Emacs 卡顿。这就是为什么 init.el 启动慢——大量 allocation 触发频繁 GC。

解决方案是 early-init.el 里临时调高 `gc-cons-threshold` (Module 4 学过)。这让启动时不 GC,启动完成后恢复阈值。这是 Emacs 27+ 的标准优化——你今天装的 Emacs 都受益于此。

未来 (Emacs 31+) 可能换更先进的 GC (generational),但目前 (Emacs 30.2) 还是 mark-and-sweep。所以"GC 卡顿"在超大 buffer (> 1MB) 仍可能发生——这是 Emacs 用户要注意的。

```elisp
(gc-cons-threshold)            ; 默认 800000
(garbage-collect)              ; 手动 GC
(setq gc-cons-percentage 0.5)
```

启动时调高 (Module 4):
```elisp
;; early-init.el
(setq gc-cons-threshold most-positive-fixnum)
(add-hook 'emacs-startup-hook
          (lambda ()
            (setq gc-cons-threshold 800000)))
```

### 6.3 buffer 实现

- gap buffer (高效插入)
- 多字节 (内部 Unicode)
- point 是 integer

### 6.4 Lisp 对象

每个 Lisp 对象是 tagged pointer:
- 直接 (fixnum, symbol 引用)
- 间接 (cons cell, vector 等在 heap)

### 6.5 pdumper (Emacs 27+)

pdumper (portable dumper) 是 Emacs 27 的优化——把加载后的 Emacs 状态 (load-history、obarray、byte-code cache) 序列化到文件,下次启动直接 mmap,跳过加载。启动时间从 2 秒降到 0.3 秒。

```bash
emacs --dump /path/to.dump
```

预 dump 加载,启动飞快。

---

## 7. 实战

### Ex 8.1: 跑 shell
```elisp
(shell-command-to-string "ls -la")
```

### Ex 8.2: 异步 process
```elisp
(make-process
 :name "ls"
 :buffer "*ls-output*"
 :command '("ls" "-la")
 :sentinel (lambda (proc event)
             (when (string-match "finished" event)
               (display-buffer "*ls-output*"))))
```

### Ex 8.3: 网络
```elisp
(let ((proc (open-network-stream "test" "*test*" "example.com" 80)))
  (process-send-string proc "GET / HTTP/1.0\r\n\r\n"))
```

### Ex 8.4: file notify
```elisp
(let ((desc (file-notify-add-watch
             "~/important.txt"
             '(change)
             (lambda (event) (message "Changed: %S" event)))))
  ;; ...
  (file-notify-rm-watch desc))
```

### Ex 8.5: thread
```elisp
(make-thread
 (lambda ()
   (sleep-for 2)
   (message "From thread")))
```

### Ex 8.6: 包写
写一个完整的包 (Module 7 capstone)。

---

## 8. 自测

1. `make-process` 和 `shell-command` 区别?
2. process-sentinel 干啥?
3. `gc-cons-threshold` 影响?
4. threads 是抢占式还是协作式?
5. file-notify 干啥?
6. abbrev table 是什么? 全局和 mode-specific 区别?
7. `M-/` 干啥? 和静态 abbrev 区别?

**答案**:
> 1. make-process 异步;shell-command 同步
> 2. 监听 process 状态变化 (start/stop/exit)
> 3. GC 阈值,大值减少 GC 但内存大
> 4. 协作式 (cooperative)
> 5. 监听文件系统变化
> 6. abbrev 的符号表;global 在所有 mode 生效,mode-specific 只在该 mode
> 7. dabbrev-expand 从 buffer 动态找词;静态 abbrev 是预定义的

---

## 9. 毕业检查 (Module 6 整体)

### 你应该能:

- [ ] 读任何 elisp 代码
- [ ] 写一个完整的 major mode (font-lock + indent + imenu)
- [ ] 用 Edebug 调试
- [ ] 写宏
- [ ] 用 byte-compile + native comp
- [ ] 跑异步 process
- [ ] 用 text property / overlay
- [ ] 用 syntax table
- [ ] 写 ERT 测试
- [ ] 理解 lexical vs dynamic

### Capstone

进入 `capstone.md` 写一个完整的 major mode。

---

## 10. 下一步

进入 `concept-anchor.md` + `exercises.md`,然后 `capstone.md`。
