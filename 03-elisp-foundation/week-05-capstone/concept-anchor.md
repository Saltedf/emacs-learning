# Concept Anchor: Editor Manual (Week 5)

> 替代 `emacs-manual-30.2/files.texi` 的核心概念
> 这周你的 my-todo 包,对应编辑器的"文件读写"和"持久化"

---

## 1. files.texi 概念锚点

### 1.1 File Handling 用户视角

Module 2 学过:
- `C-x C-f` 打开文件
- `C-x C-s` 保存
- backup 文件
- auto-save

那时你只学"按键"。现在用 Lisp 视角看——这些键位背后都是 Elisp 函数。理解它们,你就能"程序化"操作文件——批量改名、批量格式化、写 cron-like 自动化。

### 1.2 Lisp 视角

```elisp
(find-file PATH)              ;; 打开
(save-buffer)                 ;; 保存当前
(write-file PATH)             ;; 另存
(insert-file-contents PATH)   ;; 读到当前 buffer
(append-to-file START END FILE)  ;; 追加
(copy-file FROM TO)
(rename-file FROM TO)
(delete-file PATH)
(file-exists-p PATH)
(file-directory-p PATH)
(file-readable-p PATH)
(file-writable-p PATH)
(file-regular-p PATH)
(directory-files DIR &optional FULL MATCH NOSORT)
(file-attributes PATH)
(file-truename PATH)
(expand-file-name NAME DIR)
(file-name-directory PATH)
(file-name-nondirectory PATH)
(file-name-extension PATH)
(file-name-sans-extension PATH)
```

这一长串函数覆盖了文件操作的全部——打开、读、写、删、改名、查属性、路径解析。Module 2 你按 `C-x C-f`,现在你写 `(find-file ...)`。

`file-name-*` 系列特别有用——它们操作路径字符串:

- `(file-name-directory "/a/b/c.el")` → `"/a/b/"`
- `(file-name-nondirectory "/a/b/c.el")` → `"c.el"`
- `(file-name-extension "/a/b/c.el")` → `"el"`
- `(file-name-sans-extension "/a/b/c.el")` → `"/a/b/c"`

这些是路径操作的"原语"。你想"遍历某目录所有 .el 文件"——`(directory-files dir t "\\.el\\'")` 拿到文件 list,然后对每个文件 `file-name-nondirectory` 提取文件名。

### 1.3 你的 my-todo 用到的

```elisp
(locate-user-emacs-file "my-todos.el")
;; → "~/.config/emacs/my-todos.el" 或类似

(file-exists-p my-todo-file)
;; → t 或 nil

(with-temp-file PATH BODY...)
;; 在临时 buffer 跑 BODY,写到 PATH

(insert-file-contents PATH)
;; 把 PATH 内容插入当前 buffer

(read (current-buffer))
;; 从当前 buffer 读一个 sexp
```

`locate-user-emacs-file` 是"找用户配置目录下文件"——它在不同 OS 上行为不同 (Linux 是 `~/.config/emacs/`,Windows 是 `%APPDATA%`)。永远用它,不要硬编码路径。

### 1.4 持久化的几种方式

文件持久化有 4 种主流方式,各有适用场景:

**方式 1: Lisp 字面量** (你的 my-todo 用)

```elisp
(with-temp-file PATH
  (pp DATA (current-buffer)))

(with-temp-buffer
  (insert-file-contents PATH)
  (setq data (read (current-buffer))))
```

优点: 简单,Lisp 原生支持。
缺点: 数据必须可 `read` (list, string, number, symbol)。

适合 Emacs 内部数据 (配置、状态)。

**方式 2: JSON** (Emacs 27+)

```elisp
(require 'json)
(json-encode DATA)
(json-read-from-string STRING)
```

优点: 跨语言、人类可读、生态丰富。
缺点: 比 Lisp 字面量啰嗦,不支持 symbol/comment。

适合跨工具共享数据 (比如配置文件给其他程序读)。

**方式 3: 文本格式** (CSV, TSV, custom)

```elisp
;; 写
(with-temp-file PATH
  (dolist (item items)
    (insert (format "%s|%d\n" text priority))))

;; 读
(with-temp-buffer
  (insert-file-contents PATH)
  (goto-char (point-min))
  (while (not (eobp))
    (let ((line (thing-at-point 'line)))
      ;; parse
      (forward-line 1))))
```

优点: 极灵活,人类可读。
缺点: 要自己写 parser。

适合非 Lisp 用户的数据交换。

**方式 4: Emacs customize** (用 defcustom + customize-save-variable)

```elisp
(customize-save-variable 'my-var new-value)
;; 自动写到 custom-file (默认 ~/.config/emacs/custom.el)
```

优点: 集成 Emacs customize UI,用户可以 `M-x customize` 改。
缺点: 不适合大量数据。

适合用户配置 (小量、面向用户)。

---

## 2. save-hooks

### 2.1 before-save-hook

你的代码可以挂 hook,在保存前执行:

```elisp
(add-hook 'before-save-hook
          (lambda ()
            (when (derived-mode-p 'my-todo-mode)
              ;; 保存 todo 数据
              )))
```

### 2.2 after-save-hook

```elisp
(add-hook 'after-save-hook
          (lambda ()
            (when (derived-mode-p 'python-mode)
              (compile "python -m pytest"))))
```

### 2.3 write-file-functions

更细粒度,完全替换 save 行为:

```elisp
(add-hook 'write-file-functions
          (lambda ()
            ;; 返回 non-nil 跳过默认 save
            ))
```

---

## 3. 你 my-todo 的扩展点

### 3.1 加 deadline

数据结构扩展:
```elisp
(text priority tags done deadline)
```

`deadline` 是 `nil` 或 `(encode-time ...)`。

显示:
```elisp
(when deadline
  (let ((days (/
               (- (float-time deadline) (float-time))
               86400)))
    (format "(%d days)" (floor days))))
```

### 3.2 加 recurring

```elisp
(text priority tags done deadline recurring)
```

`recurring` 是 `'daily` / `'weekly` / `'monthly` / `nil`。

done 时,如果 recurring,生成下一个:
```elisp
(defun my-todo--maybe-recur (item)
  (let ((recurring (my-todo--item-recurring item)))
    (when recurring
      (let ((new-item (copy-sequence item)))
        ;; 重置 done
        (my-todo--item-set-done new-item nil)
        ;; 更新 deadline
        (when (my-todo--item-deadline new-item)
          ...)
        (setq my-todo-list
              (append my-todo-list (list new-item)))))))
```

### 3.3 加 export

```elisp
(defun my-todo-export-markdown (path)
  "Export todos to Markdown file."
  (interactive "FExport to: ")
  (with-temp-file path
    (insert "# Todos\n\n")
    (dolist (item my-todo-list)
      (let ((done (my-todo--item-done item))
            (text (my-todo--item-text item)))
        (insert (format "- [%s] %s\n"
                        (if done "x" " ")
                        text)))))
  (message "Exported to %s" path))
```

---

## 4. Backup & Auto-save

生产代码必须考虑"失败模式"——文件写一半断电怎么办? 用户误删数据怎么办? Backup 和 auto-save 是基本保险。

### 4.1 备份策略

你的 my-todo 数据文件应该有 backup:

```elisp
(defcustom my-todo-make-backup t
  "Whether to backup todo file."
  :type 'boolean
  :group 'my-todo)

(defun my-todo-save ()
  (when (and my-todo-make-backup
             (file-exists-p my-todo-file))
    (copy-file my-todo-file
               (concat my-todo-file ".bak")
               t))    ; overwrite
  (with-temp-file my-todo-file
    (pp my-todo-list (current-buffer))))
```

这个 save 在写新内容前,先把旧文件备份成 `.bak`。即使新写失败,`.bak` 还在,数据不丢。

为什么需要 backup? 因为 `with-temp-file` 不是原子的——它先 truncate (清空),再写。如果在 truncate 之后、写之前断电,文件变空,数据丢失。backup 让你"还原到最后已知好状态"。

### 4.2 Auto-save

定期保存:

```elisp
(defvar my-todo-timer nil)

(defun my-todo-start-autosave ()
  (setq my-todo-timer
        (run-with-timer 60 60 #'my-todo-save)))

(defun my-todo-stop-autosave ()
  (when my-todo-timer
    (cancel-timer my-todo-timer)
    (setq my-todo-timer nil)))
```

`run-with-timer` 是 Emacs 定时器——第一个参数是"启动后多少秒触发",第二个是"间隔",第三个是"调用的函数"。这里 60 秒触发一次,自动保存。

为什么 auto-save 重要? 用户可能编辑了 todo 但忘了手动 save。如果 Emacs 崩溃,数据丢。auto-save 让"忘记 save"的成本降到 60 秒内的修改。

---

## 5. 你的 my-todo vs 通用文件操作

### 5.1 你的代码用到的核心概念

| 概念 | 用法 |
|---|---|
| `locate-user-emacs-file` | 找用户配置目录下的文件 |
| `file-exists-p` | 检查文件 |
| `with-temp-file` | 写文件 |
| `insert-file-contents` | 读文件 |
| `read` | 解析 Lisp 字面量 |
| `pp` | pretty-print |
| `expand-file-name` | 展开 `~` 等 |
| `file-name-directory` | 取目录部分 |

### 5.2 安全考虑

- 文件可能不存在 → `(when (file-exists-p ...) ...)`
- 文件可能不可读 → `(when (file-readable-p ...) ...)`
- 文件可能损坏 → 用 `condition-case` 包裹 `read`:

```elisp
(condition-case err
    (read (current-buffer))
  (error
   (message "Failed to read todo file: %s" err)
   nil))
```

### 5.3 性能考虑

- 大文件: 用 `with-temp-buffer` 而不是直接 `find-file`
- 频繁读: 缓存到内存
- 频繁写: 用 timer 批量写

---

## 6. 实战: 加 backup 功能

修改 `my-todo-save`:

```elisp
(defun my-todo-save ()
  "Save todos to file, with backup."
  (interactive)
  ;; Backup existing file
  (when (and my-todo-make-backup
             (file-exists-p my-todo-file))
    (let ((backup-path (concat my-todo-file ".bak")))
      (copy-file my-todo-file backup-path t)))
  ;; Write new content
  (with-temp-file my-todo-file
    (pp my-todo-list (current-buffer)))
  (message "Saved %d todos to %s"
           (length my-todo-list)
           my-todo-file))
```

测试:
- 保存
- 查看 my-todo-file.el.bak 是否生成

---

## 7. 自测

1. 写一个函数 `(save-data-to-file DATA PATH)`
2. 写一个函数 `(load-data-from-file PATH)`
3. 解释 `with-temp-file` 和 `with-temp-buffer` 区别
4. 怎么给 my-todo 加 backup?
5. 怎么让 my-todo 自动保存?

**答案**:
> 1. `(defun save-data-to-file (data path) (with-temp-file path (pp data (current-buffer))))`
> 2. `(defun load-data-from-file (path) (with-temp-buffer (insert-file-contents path) (read (current-buffer))))`
> 3. with-temp-file 写到文件;with-temp-buffer 只 buffer
> 4. 见 6.1
> 5. 用 run-with-timer 定期 save

---

## 8. 下一步

进入 `capstone.md` 做 Module 3 毕业项目。
