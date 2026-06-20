# Week 6: Modes + Files + Buffers + Windows + Markers

> **Ref 章节**: modes, help, files, buffers, windows, positions, markers
> **目标**: 写完整的 major mode,精通 buffer 系统

这周是 Module 6 的"重头戏"——buffer、mode、marker 是 Emacs 的核心抽象。理解它们让你能写真实的 major mode (Module 6 capstone) 和高级配置。

这周内容多,但都围绕一个核心: **buffer 是有状态的、有附属对象 (marker、overlay、point) 的可编辑文本容器**。mode 是 buffer 的"行为配置"。window 是 buffer 的"视图"。理解这三者关系,你就理解了 Emacs 的编辑器核心。

---

## 1. Modes (modes.texi)

### 1.1 Major Mode

(Module 4 学过基础)

major mode 决定 buffer 的"语言"——是 Python、Org、Elisp 还是 Markdown。每个 buffer 有**恰好一个** major mode。完整定义用 `define-derived-mode`:

```elisp
(define-derived-mode NAME PARENT DOC &rest BODY)
```

或更完整:

```elisp
(define-derived-mode mylang-mode prog-mode "Mylang"
  "Major mode for Mylang code."
  :group 'mylang
  :syntax-table mylang-syntax-table
  :abbrev-table mylang-abbrev-table

  ;; 局部变量
  (setq-local comment-start "// ")
  (setq-local comment-end "")
  (setq-local indent-line-function #'mylang-indent-line)
  (setq-local font-lock-defaults '(mylang-font-lock-keywords))

  ;; keymap
  (setq-local imenu-generic-expression mylang-imenu)

  ;; 跑 hook (自动)
  )
```

`define-derived-mode` 是宏——它做很多事:
- 创建 `mylang-mode` 函数 (调用 PARENT mode 后跑 BODY)
- 创建 `mylang-mode-hook` 变量
- 创建 `mylang-mode-map` 变量 (除非你自定义)
- 设置 syntax table 和 abbrev table

`:group` 让 customize 知道这个 mode 属于哪个组。`:syntax-table` 和 `:abbrev-table` 让 mode 用你的 table。

`setq-local` 让变量 buffer-local——每个 buffer 有独立 `comment-start`、`font-lock-defaults` 等。

### 1.2 自动设置

```elisp
;;;###autoload
(define-derived-mode mylang-mode prog-mode "Mylang" ...)

;;;###autoload
(add-to-list 'auto-mode-alist '("\\.mylang\\'" . mylang-mode))
```

`;;;###autoload` 让 mode 在用户打开 `.mylang` 文件时自动加载。

`auto-mode-alist` 决定扩展名到 mode 的映射。Emacs 打开文件时,看扩展名,匹配 auto-mode-alist,启用对应 mode。

### 1.3 hook

```elisp
(defvar mylang-mode-hook nil)

(add-hook 'mylang-mode-hook #'flymake-mode)
(add-hook 'mylang-mode-hook #'eglot-ensure)
```

`define-derived-mode` 自动创建 `<mode-name>-hook`。mode 启用后,这个 hook 跑——让用户加自己的配置。

实战: 用户在 init.el 写 `(add-hook 'python-mode-hook #'eglot-ensure)`,这样所有 Python buffer 自动启 eglot。这是 Emacs 的"用户定制"机制——package 提供 hook,用户决定 hook 跑什么。

### 1.4 mode 优先级

一个 buffer 同时有:
- 1 个 major mode
- 0+ 个 minor modes

keymap lookup 顺序:
- minor mode keymaps (in `minor-mode-map-alist`)
- major mode keymap (`mode-map`)
- global-map

这意味 minor mode 可以 override major mode——这是为什么 `company-mode` (补全) 能在任何 major mode 工作。但 minor mode 之间也有优先级——`minor-mode-overriding-map-alist` 让某 minor mode 优先。

### 1.5 special-mode

`special-mode` 是用于"只读,特殊 buffer"的基类:

```elisp
(define-derived-mode my-special-mode special-mode "My"
  "Special mode."
  ...)
```

继承:
- buffer 只读
- `q` 退出 (quit-window)
- `g` revert
- 标准 special-mode-map

例子: `*Help*`、`*Messages*`、`*Compile-Log*` 都是 special-mode 衍生。

special-mode 是 Emacs 的"非编辑 buffer"基础——只读、特殊导航键 (q 退、g 刷)。如果你写一个"日志查看器"或"配置查看器",继承 special-mode。

### 1.6 prog-mode vs text-mode

```elisp
(prog-mode)             ; 编程模式基类
(text-mode)             ; 文本模式基类
(special-mode)          ; 特殊 buffer
(emacs-lisp-mode)       ; Emacs Lisp
```

继承 `prog-mode` 的好处:
- `prog-mode-hook` 自动跑 (你的所有编程 mode 都受影响)
- 默认 `font-lock`、`electric-indent` 等

如果你写编程语言的 mode,继承 `prog-mode`。如果是文本 (如 markdown),继承 `text-mode`。如果是特殊 buffer (如 dired),继承 `special-mode`。

---

## 2. Help (help.texi)

### 2.1 documentation 函数

Elisp 代码可以通过 function 访问 docstring——让你写自定义 help。

```elisp
(documentation 'foo)               ; 函数文档
(documentation-property 'foo 'variable-documentation)  ; 变量
(documentation-property 'foo 'group-documentation)     ; group
```

`documentation` 返回函数的 docstring (string)。`documentation-property` 取任意 property——变量文档存在 `variable-documentation` property。

### 2.2 describe 函数 (Lisp 视角)

`C-h f` 等命令的 Lisp 后端。

```elisp
(describe-function 'foo)           ; 等同 C-h f
(describe-variable 'foo)           ; C-h v
(describe-symbol 'foo)
(describe-key [?\C-c ?d])
(describe-mode)
```

这些函数显示 help buffer——你可以从代码调用,触发 help UI。

### 2.3 添加自己的 help

```elisp
(defun my-help ()
  "Show my custom help."
  (interactive)
  (with-help-window (help-buffer)
    (princ "My Help\n\n")
    (princ "Useful commands:\n")
    (princ "  C-c a - Action A\n")
    (princ "  C-c b - Action B\n")))

(with-help-window (current-buffer) ...)
```

`with-help-window` 在 `*Help*` buffer 显示。

这是写"自定义 help" 的标准模式——package 可以提供 `M-x my-package-help` 显示使用说明。

### 2.4 add-help-button

可以在 `*Help*` 加按钮:

```elisp
(help-insert-xref-button "[Source]" 'help-function-def 'foo)
```

`help-insert-xref-button` 插入可点击按钮——点击跳到 source。这是 Emacs help 的 hyperlink 系统。

---

## 3. Files (files.texi)

(Module 2 学过基础)

### 3.1 find-file 流程

`find-file` 是 Emacs 的"打开文件"——内部经过多个步骤。

```elisp
(find-file PATH)
;; 内部:
;; 1. 检查 buffer 已开 (get-file-buffer)
;; 2. 否则创建 buffer
;; 3. 调 insert-file-contents
;; 4. 跑 find-file-hook
;; 5. 设 buffer-file-name
;; 6. after-find-file (检查文件存在、可读)
```

理解这个流程让你能 hook 任何环节——`find-file-hook` 让你在文件打开后跑代码 (如自动 format、加载 LSP)。

`find-file-noselect` 是不 select window 的版本——适合"代码打开文件但不切走"。

### 3.2 find-file-not-found

```elisp
(find-file-noselect PATH &optional NOWARN RAWFILE WILDCARDS)
```

不 select window。

### 3.3 save-buffer 流程

`save-buffer` (C-x C-s) 内部:

```elisp
(save-buffer)
;; 1. run-before-save-hook
;; 2. 写文件 (basic-save-buffer)
;; 3. set-visited-file-modtime
;; 4. set-buffer-modified-p nil
;; 5. run-after-save-hook
```

`before-save-hook` 和 `after-save-hook` 让你在保存前后跑代码——典型的用例是 "保存前 format" (`(add-hook 'before-save-hook #'lsp-format-buffer)`)。

### 3.4 文件信息

```elisp
(file-attributes PATH)
;; → (mode num-links uid gid size mod-time status-change access-time)
(file-attributes "/etc/passwd" 'string)
;; uid/gid 用 string
```

`file-attributes` 返回文件的所有元数据——mode、size、modtime 等。比单独调用 `file-exists-p` 等高效 (一次 syscall)。

```elisp
(file-exists-p PATH)
(file-readable-p PATH)
(file-writable-p PATH)
(file-executable-p PATH)
(file-directory-p PATH)
(file-regular-p PATH)
(file-symlink-p PATH)
(file-truename PATH)              ; 解析 symlink
```

`file-truename` 解析 symlink——`(file-truename "~/link")` 给真实路径。用于避免重复打开 symlinked 文件。

### 3.5 路径操作

Elisp 提供完整的路径 API——比手动 concat 安全。

```elisp
(expand-file-name "foo" "~/bar")  ; → "/home/user/bar/foo"
(file-name-directory "/a/b/c.txt") ; → "/a/b/"
(file-name-nondirectory "/a/b/c.txt") ; → "c.txt"
(file-name-base "/a/b/c.txt")     ; → "c"
(file-name-extension "c.txt")     ; → "txt"
(file-name-sans-extension "c.txt") ; → "c"
(file-name-as-directory "/a/b")   ; → "/a/b/"
(directory-file-name "/a/b/")     ; → "/a/b"
(concat (file-name-as-directory "/a") "b.txt")  ; → "/a/b.txt"
```

这些函数处理跨平台路径分隔符 (Windows 是 `\`,Unix 是 `/`)、相对路径、`~` 展开等。**永远用它们,不要手动 concat**——手动 concat 容易错 (忘 trailing slash、忘 expand ~)。

### 3.6 目录操作

```elisp
(directory-files DIR &optional FULL MATCH NOSORT)
;; → list of file names

(directory-files-recursively DIR REGEXP &optional INCLUDE-DIRS PRED FOLLOW-SYMLINKS)
;; → 所有匹配的文件 (递归)

(file-name-all-completions "foo" "/path/")
;; 补全
```

`directory-files` 列出目录文件——`FULL` t 给绝对路径,`MATCH` regex 过滤。

`directory-files-recursively` 是 Emacs 25+ 的强力工具——递归找所有匹配文件。`project.el`、`xref` 等大量用。

### 3.7 修改文件

```elisp
(copy-file FROM TO &optional OKAY-IF-ALREADY-EXISTS)
(rename-file FROM TO &optional OKAY-IF-ALREADY-EXISTS)
(delete-file PATH &optional TRASH)
(make-directory DIR &optional PARENTS)
(delete-directory DIR &optional RECURSIVE TRASH)
(set-file-modes PATH MODE)
(set-file-times PATH)
```

`make-directory` 的 `PARENTS` 类似 `mkdir -p`——创建中间目录。`delete-file` 的 `TRASH` 让文件进回收站而非直接删 (跨平台)。

---

## 4. Buffers (buffers.texi)

### 4.1 获取

```elisp
(current-buffer)
(get-buffer NAME)
(get-buffer-create NAME)
(get-file-buffer PATH)
(generate-new-buffer-name BASE)
```

`get-buffer` 找已存在 buffer (不存在返回 nil)。`get-buffer-create` 找或创建。`get-file-buffer` 找某文件对应的 buffer。`generate-new-buffer-name` 生成唯一名 (如 `*foo*<2>`)。

### 4.2 切换

```elisp
(set-buffer BUFFER)
(switch-to-buffer BUFFER)
(pop-to-buffer BUFFER)
(display-buffer BUFFER)
(with-current-buffer BUFFER BODY...)
(save-current-buffer BODY...)
```

- `set-buffer`: 设当前 buffer (Lisp 视角),不更新 window
- `switch-to-buffer`: 切 window 显示的 buffer
- `pop-to-buffer`: 弹出 buffer (可能新 window)
- `display-buffer`: 显示 buffer (按 display-buffer-alist)
- `with-current-buffer`: 临时切,跑 body,恢复
- `save-current-buffer`: 保存当前 buffer,跑 body,恢复

`with-current-buffer` 最常用——你想在另一个 buffer 跑代码,但不想切走。`(with-current-buffer "*scratch*" (insert "hi"))` 在 *scratch* 插入,但当前 buffer 不变。

### 4.3 信息

```elisp
(buffer-name)
(buffer-file-name)
(buffer-size)
(buffer-modified-p)
(buffer-base-buffer)              ; indirect buffer 的 base
(buffer-local-value VAR BUFFER)
(buffer-local-variables)
```

`buffer-base-buffer` 用于 indirect buffer——返回它共享内容的 base buffer。

### 4.4 修改

```elisp
(rename-buffer NAME)
(set-buffer-modified-p BOOL)
(set-visited-file-name PATH)      ; 改关联文件
(kill-buffer BUFFER)
(kill-buffer-and-its-windows BUFFER)
```

`set-visited-file-name` 改 buffer 关联的文件——常用于"另存为"。

### 4.5 间接 buffer

```elisp
(make-indirect-buffer BASE-BUFFER NAME &optional CLONE)
```

indirect buffer 共享 base 的内容,但有独立 point、narrowing、mode 等。

这是 Emacs 独特的设计——同一个 buffer 内容可以多个"视图",各自有独立 point 和 narrowing。Org mode 用它实现"代码块编辑"——每个 src block 可以开 indirect buffer,用对应语言 mode 编辑,但内容还是 Org 的一部分。

### 4.6 buffer list

```elisp
(buffer-list)
(buffer-list 'visible)            ; 显示的
(buffer-list 'frame)              ; 当前 frame 的
(mapcar #'buffer-name (buffer-list))
```

`buffer-list` 返回所有 buffer——按 recent 顺序 (最近用的在前)。这让你写"切到最近 buffer" 等功能。

### 4.7 关闭 buffer

```elisp
(kill-buffer)                     ; 当前
(kill-buffer NAME)
(kill-some-buffers (list BUF1 BUF2))
(kill-matching-buffers REGEXP &optional INTERNAL TOO)
```

`kill-matching-buffers` 按名字 regex 批量关——`(kill-matching-buffers "^\\*temp")` 关所有 *temp* 开头 buffer。

### 4.8 buffer 名字约定

| 名字 | 用途 |
|---|---|
| `*scratch*` | Lisp 交互 |
| `*Messages*` | 系统消息 |
| `*Help*` | help 输出 |
| `*Buffer List*` | buffer 列表 |
| `*Completions*` | 补全候选 |
| `*shell*` / `*terminal*` | shell |
| `*Compile-Log*` | 编译日志 |
| `*Backtrace*` | 错误调试 |

`*...*` 约定: 特殊 buffer,不和文件关联。

Emacs 用 `*NAME*` 约定区分"内部 buffer" 和"文件 buffer"。这让 ibuffer、bs 等工具可以分组显示。

---

## 5. Windows (windows.texi)

(Module 1 + Module 4 学过)

window 是 buffer 的"视图"——同一个 buffer 可以在多个 window 显示 (各自有独立 point)。

### 5.1 获取

```elisp
(selected-window)
(window-list)
(window-at X Y)
(next-window)
(previous-window)
(get-buffer-window BUFFER)
(minibuffer-window)
```

### 5.2 操作

```elisp
(select-window WINDOW)
(split-window &optional WINDOW SIZE SIDE)
(delete-window &optional WINDOW)
(delete-other-windows &optional WINDOW)
(delete-windows-on BUFFER)
(set-window-buffer WINDOW BUFFER)
(display-buffer BUFFER &optional ACTION)
(switch-to-buffer-other-window BUFFER)
```

### 5.3 大小

```elisp
(window-total-height WINDOW)
(window-total-width WINDOW)
(window-body-height WINDOW)
(window-body-width WINDOW)
(window-edges WINDOW)
(window-pixel-edges WINDOW)
(window-resize WINDOW DELTA &optional HORIZONTAL IGNORE)
(window-resize-no-error WINDOW DELTA &optional HORIZONTAL)
```

### 5.4 point 关联

```elisp
(window-point WINDOW)
(set-window-point WINDOW POS)
(window-start WINDOW)
(set-window-start WINDOW POS &optional NOFORCE)
(window-end WINDOW &optional UPDATE)
```

每个 window 有独立的 point (在该 window 的 buffer 里)。

这是 Emacs 独特设计——window 是 buffer 的视图,但视图有自己的 point。这意味着同一个 buffer 在两个 window,可以看不同部分,各自有 point。

### 5.5 display-buffer-alist

控制 buffer 显示行为:

```elisp
(setq display-buffer-alist
      '(
        ;; *Help* 在底部
        ("\\*Help\\*"
         (display-buffer-in-side-window)
         (side . bottom)
         (window-height . 0.3))
        ;; *shell* 在右侧
        ("\\*shell\\*"
         (display-buffer-in-side-window)
         (side . right)
         (window-width . 80))
        ))
```

这是 Emacs 24+ 的"显示策略"系统——`display-buffer` 调用时,根据 buffer name 匹配 alist,应用对应 action。让你精确控制每个 buffer 怎么显示。

---

## 6. Positions (positions.texi)

### 6.1 Point

```elisp
(point)
(point-min)
(point-max)
(goto-char POS)
```

`point` 是 1-based integer——当前位置。`point-min`/`point-max` 是当前 buffer (或 narrowed 范围) 的边界。

### 6.2 行列

```elisp
(line-beginning-position &optional N)
(line-end-position &optional N)
(pos-bol)                       ; = line-beginning-position
(pos-eol)                       ; = line-end-position
(current-column)
(line-number-at-pos &optional POS ABSOLUTE)
(column-at-pos POS)
(move-to-column COLUMN &optional FORCE)
```

`line-beginning-position` 是当前行起点。Emacs 29+ 加了 `pos-bol`/`pos-eol` (短名)。

### 6.3 region

```elisp
(region-beginning)
(region-end)
(use-region-p)
(region-active-p)
(deactivate-mark)
(activate-mark)
```

`use-region-p` 检查 region 是否 active 且非空——`(interactive "r")` 命令的标准检查。

### 6.4 移动

```elisp
(forward-char N)
(backward-char N)
(forward-word N)
 backward-word N)
(forward-line N)
(forward-sentence N)
(forward-paragraph N)
(beginning-of-buffer)
(end-of-buffer)
```

注意原文 typo——`backward-word` 前面漏了 `(`。

### 6.5 skip

```elisp
(skip-chars-forward CHARSET &optional LIM)
(skip-chars-backward CHARSET &optional LIM)
(skip-syntax-forward SYNTAX &optional LIM)
(skip-syntax-backward SYNTAX &optional LIM)
```

例子:
```elisp
(skip-chars-forward " \t")    ; 跳过空格和 tab
(skip-syntax-forward "w")     ; 跳过 word 字符
```

`skip-chars-forward` 跳过指定字符集合——常用于跳过空白、标点。`skip-syntax-forward` 跳过指定 syntax class——按 syntax table 分类。

这两个是写 mode 的核心——indent 时跳过前导空白,parse 时跳过标识符。

---

## 7. Markers (markers.texi)

### 7.1 什么是 marker

第一性原理: buffer 编辑时,point 位置需要保持有意义。

- 推论 1: 静态 integer 不行——字符插入后位置错位
- 推论 2: 需要一个"自动跟踪"的对象 → marker
- 推论 3: marker 是 (buffer . position) 对,buffer 改时自动更新
- 推论 4: marker 有开销,大量时用 integer
- 推论 5: point 本身是 special marker (always updated)

**Marker** 是一个**对象**,指向 buffer 中的位置。
和 integer 的区别:
- integer: 静态
- marker: 在 buffer 编辑时**自动更新**

```elisp
(setq m (point-marker))        ; 当前位置的 marker
(set-marker m 100)             ; 改位置
(marker-position m)            ; 位置
(marker-buffer m)              ; buffer
(move-marker m 200)            ; 移动
(set-marker m nil)             ; 释放 (重要)
```

### 7.2 用途

- 记住位置,即使 buffer 改了
- buffer 内的"书签"

实战: 编译错误列表——`compile-mode` 存每个 error 的 marker,即使你编辑了源码,marker 自动更新到正确位置。

### 7.3 性能

marker 有开销——每次编辑 buffer,所有相关 marker 都要更新。大量 marker 时,编辑变慢。

解决方案: 
- 不再需要的 marker 用 `(set-marker m nil)` 释放
- 大量位置用 integer——只在需要"自动更新"时用 marker

---

## 8. 实战: 写一个完整的 major mode

(详见 `capstone.md`,Module 6 毕业项目)

```elisp
;;; mylang-mode.el -*- lexical-binding: t; -*-

(defgroup mylang nil "..." :group 'languages)

(defcustom mylang-indent-offset 4 "..." :type 'integer :group 'mylang)

(defvar mylang-mode-map
  (let ((map (make-sparse-keymap)))
    (define-key map (kbd "C-c C-c") #'mylang-eval-buffer)
    (define-key map (kbd "C-c C-r") #'mylang-eval-region)
    map))

(defvar mylang-mode-syntax-table
  (let ((st (make-syntax-table)))
    (modify-syntax-entry ?_ "w" st)
    (modify-syntax-entry ?# "_" st)
    (modify-syntax-entry ?/ ". 124b" st)
    (modify-syntax-entry ?* ". 23" st)
    (modify-syntax-entry ?\n "> b" st)
    st))

(defvar mylang-font-lock-keywords
  '(("//.*" . font-lock-comment-face)
    ("\"[^\"]*\"" . font-lock-string-face)
    ("\\<\\(if\\|else\\|while\\|for\\|return\\)\\>" . font-lock-keyword-face)
    ("\\<\\(int\\|str\\|bool\\)\\>" . font-lock-type-face)
    ("\\<\\([A-Z][a-zA-Z0-9_]*\\)\\>" . font-lock-type-face)
    ("\\<\\([a-z_][a-zA-Z0-9_]*\\)\\s-*(" . (1 font-lock-function-name-face))))
```

(完整版在 capstone)

---

## 9. 自测

1. major mode 和 minor mode 的根本区别?
2. `define-derived-mode` 自动创建什么?
3. `(file-truename PATH)` 干啥?
4. indirect buffer 是什么?
5. marker 和 integer position 区别?
6. display-buffer-alist 干啥?

**答案**:
> 1. major 每 buffer 一个;minor 多个可叠加
> 2. mode 函数、mode 变量、mode hook、mode-map
> 3. 解析 symlink 到真实路径
> 4. 共享 base buffer 内容,独立 point/mode/narrowing
> 5. marker 自动更新 (buffer 改了);integer 静态
> 6. 控制 display-buffer 行为

---

## 10. 下一步

进入 `concept-anchor.md` + `exercises.md`,然后 `../week-07-text-display/`。
