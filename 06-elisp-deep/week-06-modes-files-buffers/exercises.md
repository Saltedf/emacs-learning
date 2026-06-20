# Exercises: Week 6 Modes + Files + Buffers + Windows

> 12 题

这周练习考 mode、文件、buffer、window、marker——Module 6 capstone 的所有 building blocks。每题一个具体技能。

---

### 题 1: 简单 major mode

```elisp
(define-derived-mode my-todo-mode fundamental-mode "MyTodo"
  "Mode for displaying todos."
  (read-only-mode 1))
```

最简单的 major mode——继承 fundamental-mode (最基础,无任何配置),启用 read-only。`*todo*` 这种 display buffer 常用。

### 题 2: auto-mode-alist

```elisp
(add-to-list 'auto-mode-alist '("\\.todo\\'" . my-todo-mode))
```

让 `.todo` 文件自动用 my-todo-mode。

### 题 3: 文件操作

```elisp
(defun my-touch-file (path)
  (with-temp-file path
    (insert "")))
```

`with-temp-file` 创建 temp buffer,跑 body,写内容到文件。空 body 创建空文件——类似 Unix `touch`。

### 题 4: 路径操作

```elisp
(defun my-backup-path (path)
  (let* ((dir (file-name-directory path))
         (base (file-name-nondirectory path))
         (backup-name (concat base ".bak")))
    (expand-file-name backup-name dir)))
```

这个函数生成 backup 路径——`/a/b/c.txt` → `/a/b/c.txt.bak`。用 `file-name-directory` 取目录,`file-name-nondirectory` 取文件名,`expand-file-name` 合并。

### 题 5: 列出大文件

```elisp
(defun my-list-large-files (dir size)
  (let (result)
    (dolist (file (directory-files-recursively dir "."))
      (when (> (nth 5 (file-attributes file)) size)
        (push file result)))
    result))
```

`directory-files-recursively` 递归找所有文件,`file-attributes` 取 size (nth 5 是 size),filter 大于阈值。

### 题 6: 间接 buffer

```elisp
(defun my-new-indirect ()
  (interactive)
  (make-indirect-buffer (current-buffer)
                        (generate-new-buffer-name (buffer-name)))
  (display-buffer (current-buffer)))
```

indirect buffer 是 Emacs 独特设计——同一 buffer 多个视图。Org mode 的 src block 编辑用这个。

### 题 7: marker 用法

```elisp
(defun my-mark-position ()
  (interactive)
  (let ((m (point-marker)))
    (message "Marked at %d" (marker-position m))
    (run-with-timer 5 nil
                    (lambda ()
                      (message "Now at %d" (marker-position m))
                      (set-marker m nil)))))
```

注意最后 `(set-marker m nil)`——释放 marker。如果不释放,marker 持续占用资源,编辑 buffer 时 Emacs 还要更新它。

### 题 8: window 操作

```elisp
(defun my-split-4 ()
  (interactive)
  (delete-other-windows)
  (split-window-right)
  (split-window-below)
  (other-window 1)
  (split-window-below))
```

把当前 window 切成 4 块。`delete-other-windows` 先关其他,然后切。

### 题 9: display-buffer-alist

```elisp
(setq display-buffer-alist
      '(("\\*Help\\*"
         (display-buffer-in-side-window)
         (side . bottom)
         (window-height . 0.25)))))
```

让 `*Help*` 显示在底部 side window,占 25% 高度。这是 Emacs 24+ 的"显示策略"系统。

### 题 10: with-current-buffer

```elisp
(defun my-clean-scratch ()
  (with-current-buffer "*scratch*"
    (erase-buffer)
    (insert ";; This buffer is for Lisp evaluation.\n\n")))
```

`with-current-buffer` 让你在另一个 buffer 跑代码,不切走。

### 题 11: 文件 watch

```elisp
(defvar my-watch-timer nil)

(defun my-start-watch (path fn)
  (setq my-watch-timer
        (run-with-timer
         10 10
         (lambda ()
           (when (file-newer-than-file-p path "~/last-checked")
             (funcall fn))))))

(defun my-stop-watch ()
  (when my-watch-timer
    (cancel-timer my-watch-timer)
    (setq my-watch-timer nil)))
```

这是用 timer 模拟 file watch——每 10 秒检查文件是否新于 marker。Module 6 W8 学真正的 `file-notify-add-watch` (内核级 event)。

### 题 12: directory operation

```elisp
(defun my-cleanup-empty-dirs (dir)
  (dolist (sub (directory-files dir t "^[^.]"))
    (when (file-directory-p sub)
      (my-cleanup-empty-dirs sub)
      (when (null (directory-files sub nil "^[^.]"))
        (delete-directory sub)))))
```

递归删除空目录。先递归子目录,然后如果当前目录空 (无非 . 文件),删除。这是 git clean 的逻辑。

---

## 自测

1. `define-derived-mode` 第一个参数?
2. 怎么让某扩展名用我的 mode?
3. `(file-attributes PATH)` 返回什么?
4. marker 怎么释放?
5. indirect buffer 干啥?

**答案**:
> 1. mode 名
> 2. (add-to-list 'auto-mode-alist '("\\.ext\\'" . my-mode))
> 3. list (mode, links, uid, gid, size, modtime, ...)
> 4. (set-marker m nil)
> 5. 共享 base buffer 内容,独立 point/mode

---

进入 `../week-07-text-display/`。
