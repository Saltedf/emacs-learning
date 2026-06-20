# Exercises: Week 8 Processes + OS + Package + Internals

> 12 题

这周练习覆盖 process、网络、file-notify、thread、abbrev、GC——Module 6 W8 的所有 API。**实际跑每题**,因为 process 行为很难从代码预测。

---

### 题 1: shell-command

```elisp
(shell-command "ls -la" "*ls*")
(display-buffer "*ls*")
```

`shell-command` 把输出写到指定 buffer。同步,卡 UI 直到命令完成。

### 题 2: shell-command-to-string

```elisp
(shell-command-to-string "echo $HOME")
;; → "/home/sun\n"
```

返回 string——适合快速拿命令结果。注意 trailing newline (`echo` 加的)。

### 题 3: make-process

```elisp
(make-process :name "test" :buffer "*test*"
              :command '("echo" "hello"))
```

异步 process——立即返回 process 对象,Emacs 后台跑。

### 题 4: 异步下载

```elisp
(defun my-download (url path)
  (make-process
   :name "curl"
   :buffer "*curl*"
   :command (list "curl" "-o" path url)
   :sentinel (lambda (proc event)
               (when (string-match "finished" event)
                 (message "Downloaded to %s" path)))))
```

异步下载——不卡 UI,完成时 message。这是 async process 的经典用例。

### 题 5: 网络 HTTP

```elisp
(defun my-fetch (host path)
  (let ((proc (open-network-stream "http" "*http*" host 80))
        (sent (format "GET %s HTTP/1.0\r\nHost: %s\r\nUser-Agent: Emacs\r\n\r\n"
                      path host)))
    (set-process-filter proc
                        (lambda (proc string)
                          (with-current-buffer (process-buffer proc)
                            (goto-char (point-max))
                            (insert string))))
    (process-send-string proc sent)))
```

filter 把 response 追加到 process buffer——HTTP response 可能分多次到达,filter 处理每次的 chunk。

### 题 6: file-notify

```elisp
(let ((desc (file-notify-add-watch
             "~/log.txt"
             '(change)
             (lambda (event)
               (message "log changed: %S" event)))))
  ;; ...
  (file-notify-rm-watch desc))
```

文件变化触发 callback。注意 desc——用完要 rm。

### 题 7: timer

```elisp
(run-with-timer 60 60 (lambda () (message "tick")))
(run-with-idle-timer 5 nil (lambda () (message "idle 5s")))
(cancel-timer TIMER)
```

第一行: 60 秒后跑,每 60 秒重复。第二行: Emacs idle 5 秒后跑一次。第三行: cancel 已设的 timer。

### 题 8: time

```elisp
(format-time-string "%H:%M:%S")
(float-time)
(time-add (current-time) (seconds-to-time 3600))
```

`time-add` 时间算术——`seconds-to-time 3600` 是 1 小时,加到 current-time 得"1 小时后"。

### 题 9: thread

```elisp
(make-thread
 (lambda ()
   (let ((mutex (make-mutex)))
     (dotimes (i 5)
       (sleep-for 1)
       (with-mutex mutex
         (message "tick %d" i))))))
```

后台 thread 每 1 秒 message。但 Emacs thread 协作式——sleep-for 期间让出,但 message 调用时如果主 thread 在跑,要等切换点。

### 题 10: abbrev

```elisp
(define-abbrev-table 'my-abbrev-table
  '(("btw" "by the way")
    ("ih" "I heard")))

(add-hook 'text-mode-hook
          (lambda ()
            (abbrev-mode 1)
            (setq local-abbrev-table my-abbrev-table)))
```

定义 abbrev table,绑定到 text-mode。输入 "btw SPC" 自动展开。

### 题 11: package descriptor

```elisp
(define-package "my-test" "1.0" "Test." '((emacs "29.1")))
```

package metadata。`'((emacs "29.1"))` 是依赖——要求 Emacs 29.1+。

### 题 12: GC monitor

```elisp
(setq garbage-collection-messages t)
(garbage-collect)
```

开启 GC 消息,手动 GC 一次。`(garbage-collect)` 返回 GC 统计 (各类型对象数量)。

---

## 自测

1. 怎么异步跑 shell 命令?
2. process-sentinel 接收什么?
3. timer 怎么 cancel?
4. `current-time` 返回什么?
5. threads 限制?

**答案**:
> 1. make-process
> 2. process 和 event string
> 3. cancel-timer
> 4. list of high/low/sec/usec
> 5. 协作式,不抢占

---

进入 `../capstone.md`。
