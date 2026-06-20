# Concept Anchor + Exercises: Week 6

## Concept Anchor: modes.texi + files.texi + buffers.texi

Emacs 的"打开文件自动选 mode"是一个集成系统——多个 alist 协作决定 buffer 用什么 mode。理解这个系统让你能"hook 进 Emacs 的文件打开流程"。

### modes.texi 用户视角

```
M-x MODE-NAME         切到某 mode
C-h m                 describe-mode (看当前 mode)
```

mode 自动选:
- `auto-mode-alist`: 文件扩展名 → mode
- `interpreter-mode-alist`: shebang → mode
- `magic-mode-alist`: 文件 magic → mode
- `magic-fallback-mode-alist`: 后备

查找顺序: auto-mode-alist 先 (扩展名),如果没匹配,看 shebang (interpreter-mode-alist),再不行用 magic (file 内容头部)。这让你 `.py` 文件、`#!/usr/bin/env python` 文件、包含 `import` 的文件都能进 python-mode。

### files.texi 用户视角

```
C-x C-f              find-file
C-x C-s              save
C-x C-w              write-file (另存)
C-x d                dired
M-x recover-file     恢复 autosave
```

这些是用户视角的文件操作。Module 6 学 Lisp 视角——`find-file` 内部经过多个步骤和 hook (见 ref-reading)。

### buffers.texi 用户视角

```
C-x b                switch-buffer
C-x C-b              ibuffer
C-x k                kill-buffer
```

buffer 是 Emacs 的"工作单元"——所有文件、shell、help、message 都在某个 buffer。理解 buffer list 和 navigation 让你能高效管理多个工作。

---

## Exercises: Week 6

这周练习覆盖 major mode 创建、文件操作、buffer/window 操作、marker——Module 6 capstone 的所有 building blocks。

### 题 1: define-derived-mode

```elisp
(define-derived-mode my-text-mode text-mode "MyText"
  "My text mode."
  (visual-line-mode 1)
  (flyspell-mode 1))
```

最简单的 major mode——继承 text-mode,启用 visual-line-mode 和 flyspell-mode。`(visual-line-mode 1)` 调用 minor mode 启用函数,`1` 是 enable 参数。

### 题 2: auto-mode-alist

```elisp
(add-to-list 'auto-mode-alist '("\\.mylang\\'" . mylang-mode))
```

让 `.mylang` 文件自动用 mylang-mode。`'\\.mylang\\'` 是 regex——`\\.` 是字面点,`\\'` 是 string end (区别于 `$`)。

### 题 3: file-attributes

```elisp
(nth 5 (file-attributes "/etc/passwd"))    ; → size (bytes)
(nth 7 (file-attributes "/etc/passwd"))    ; → modtime
```

`file-attributes` 返回 list,每个元素是文件的不同属性。`(nth N ...)` 取第 N 个——0-based,所以 size 是 nth 7 (实际是 `(nth 7 ...)`, size 是 `(nth 7 ...)` 取决于 Emacs 版本,要查 docstring)。

### 题 4: directory-files

```elisp
(directory-files "~/projects" t "\\.el\\'")  ; → 所有 .el 绝对路径
```

第一参数 dir,第二 `t` 是 FULL (绝对路径),第三是 MATCH regex。

### 题 5: marker

```elisp
(setq m (point-marker))
;; 编辑 buffer
(marker-position m)             ; 自动更新
```

marker 是"记住位置" 的工具——你 set 一次,后续 buffer 编辑,marker 自动跟着。这就是 compile-mode 的 error 跳转为什么能工作的基础。

### 题 6: with-current-buffer

```elisp
(with-current-buffer "*scratch*"
  (buffer-size))
```

`with-current-buffer` 临时切换,跑 body,恢复。让你"在另一个 buffer 跑代码"而不污染当前 buffer。

### 题 7: window-list

```elisp
(window-list)                   ; 当前 frame 所有 window
```

返回当前 frame 的所有 window。你可以 mapcar 操作每个。

### 题 8: display-buffer

```elisp
(with-current-buffer (get-buffer-create "*my-output*")
  (erase-buffer)
  (insert "result"))
(display-buffer "*my-output*")
```

先准备 buffer 内容,再 display。`display-buffer` 根据 `display-buffer-alist` 决定怎么显示。

### 题 9: skip-chars-forward

```elisp
(skip-chars-forward " \t\n")    ; 跳过空白
```

跳过空白的标准操作——indent、parse 时常用。返回跳过的字符数。

### 题 10: line operations

```elisp
(line-beginning-position)
(line-end-position)
(forward-line 1)
```

行操作的基础——`line-beginning-position` 是行起点,`forward-line 1` 移到下一行。

### 题 11: indirect buffer

```elisp
(make-indirect-buffer (current-buffer) "*indirect*")
;; 间接 buffer,共享内容
```

indirect buffer 共享 base 内容,但有独立 point/mode/narrowing。Org mode 用它实现 src block 编辑。

### 题 12: write region

```elisp
(write-region START END FILENAME)
(write-region "hello\n" nil "~/test.txt" 'append)
```

`write-region` 写一段到文件。`'append` 模式追加而非覆盖——日志文件常用。

---

## 自测

1. 怎么写一个 major mode?
2. auto-mode-alist 怎么加?
3. file-attributes 返回什么?
4. marker 怎么用?
5. indirect buffer 干啥?

**答案**:
> 1. define-derived-mode NAME PARENT DOC BODY...
> 2. (add-to-list 'auto-mode-alist '("\\.ext\\'" . my-mode))
> 3. list (mode, links, uid, gid, size, ...)
> 4. (point-marker),自动更新
> 5. 共享 base 内容,独立 point/mode

---

进入 `../week-07-text-display/`。
