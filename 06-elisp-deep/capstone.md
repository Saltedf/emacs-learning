# Capstone: Module 6 毕业项目 (Major Mode)

> **目标**: 写一个完整的 major mode,500+ 行 + ERT
> **时长**: 1 周 (~15 小时)
> **难度**: ★★★★★

---

## 项目概述

写一个名为 `mylang-mode` 的 major mode,功能:

1. **语法高亮** (font-lock): keyword、type、function、string、comment
2. **缩进** (indent): 智能缩进规则
3. **imenu**: 函数列表
4. **注释**: `// ...`
5. **code navigation**: `beginning-of-defun`、`end-of-defun`
6. **键位**: `C-c C-c` eval buffer (如果是 Lisp-like)
7. **ERT 测试**: 至少 10 个
8. **额外 (可选,加分)**:
   - 用 `peg` 解析 buffer 结构 (W7 学的),做更准确的 indent / navigation
   - 用 `narrow-to-defun` 加 `C-c n` 命令 (W3 学的 narrowing)
   - 加 mode-specific abbrev table (`define-abbrev-table 'mylang-mode-abbrev-table`) 给常用代码片段

### 为什么是 major mode 作为毕业项目

major mode 是 Elisp 综合性的考验——它需要你同时用上 Module 6 学的所有东西:
- **数据类型** (week 1): list 表示 syntax table, hash 表示 symbol table
- **求值/变量** (week 2): buffer-local variables, closures for advice
- **宏** (week 3): `define-derived-mode` 本身是宏,你写自己的 `defmylang-...`
- **调试** (week 4): Edebug 调 font-lock (font-lock 极其难调,需要 Edebug)
- **minibuf/commands** (week 5): interactive commands, keymaps
- **modes/files/buffers** (week 6): `define-derived-mode`, `auto-mode-alist`
- **text/display** (week 7): text properties, overlays, font-lock, syntax table
- **processes** (week 8): eval-buffer 跑外部解释器

如果你能独立写出一个 500 行的 major mode,你就证明了你对 Emacs Lisp 真正的掌握——不是"知道 API",而是"能用 API 组合出真实软件"。这就是毕业项目的设计意图。

### 为什么 500 行

500 行是一个**真实** package 的最小规模。少于这个,你的 mode 只是玩具 (toy);多于这个,你已经在做产品级 (org-mode 是 4 万行)。500 行能让你:
- 有足够 font-lock 规则体现语法多样性
- 有 indent 处理嵌套 (`if` 内 `for` 内 `try`)
- 有 ERT 测试覆盖各种边界
- 有 keymap 和 menu,体现"用户友好"

MELPA 上很多小包就 500-1000 行,完全够用。

---

## 完整代码

### `mylang-mode.el`

下面是完整源码。**不要复制粘贴就完事**——每一段都对照 Module 6 哪周学的,确认你理解。代码后面有逐段解释。

```elisp
;;; mylang-mode.el --- Major mode for Mylang -*- lexical-binding: t; -*-

;;; Copyright (C) 2026 Your Name

;;; Author: Your Name <you@example.com>
;;; Version: 1.0
;;; Package-Requires: ((emacs "29.1"))
;;; Keywords: languages
;;; URL: https://github.com/you/mylang-mode

;;; Commentary:
;; This package provides a major mode for editing Mylang files.
;;
;; Features:
;; - Syntax highlighting
;; - Indentation
;; - Imenu
;; - Comment support

;;; Code:

(require 'imenu)
(require 'cl-lib)

;;;; Custom variables

(defgroup mylang nil
  "Major mode for Mylang code."
  :group 'languages
  :prefix "mylang-")

(defcustom mylang-indent-offset 4
  "Indentation offset for Mylang."
  :type 'integer
  :group 'mylang)

(defcustom mylang-command "mylang"
  "Mylang interpreter."
  :type 'string
  :group 'mylang)

;;;; Syntax table

(defvar mylang-mode-syntax-table
  (let ((st (make-syntax-table)))
    ;; Word chars
    (modify-syntax-entry ?_ "w" st)
    (modify-syntax-entry ?# "_" st)
    ;; String chars
    (modify-syntax-entry ?\" "\"" st)
    (modify-syntax-entry ?\\ "\\" st)
    ;; Comments (single line: //)
    (modify-syntax-entry ?/ ". 124b" st)
    (modify-syntax-entry ?* ". 23" st)
    (modify-syntax-entry ?\n "> b" st)
    ;; Parens
    (modify-syntax-entry ?\( "()" st)
    (modify-syntax-entry ?\) ")(" st)
    (modify-syntax-entry ?\[ "(]" st)
    (modify-syntax-entry ?\] ")[" st)
    (modify-syntax-entry ?\{ "(}" st)
    (modify-syntax-entry ?\} "){" st)
    ;; Punctuation
    (modify-syntax-entry ?. "." st)
    (modify-syntax-entry ?, "." st)
    (modify-syntax-entry ?\; "." st)
    (modify-syntax-entry ?: "." st)
    (modify-syntax-entry ?+ "." st)
    (modify-syntax-entry ?- "." st)
    (modify-syntax-entry ?= "." st)
    (modify-syntax-entry ?< "." st)
    (modify-syntax-entry ?> "." st)
    st)
  "Syntax table for `mylang-mode'.")

;;;; Font-lock

(defconst mylang-keywords
  '("if" "else" "elif" "while" "for" "in" "do" "while"
    "return" "function" "fun" "def" "class" "struct"
    "break" "continue" "pass" "import" "from" "as"
    "true" "false" "null" "nil" "None" "True" "False"
    "and" "or" "not" "is" "in")
  "Mylang keywords.")

(defconst mylang-types
  '("int" "str" "bool" "float" "void" "list" "dict"
    "tuple" "set" "any" "Optional" "Union")
  "Mylang types.")

(defvar mylang-font-lock-keywords
  `(
    ;; Comments
    ("//.*" . font-lock-comment-face)
    ("\\(#\\).*\\(\n\\|$\\)" . (1 font-lock-comment-face))
    ;; Strings
    ("\"[^\"]*\"" . font-lock-string-face)
    ("'[^']*'" . font-lock-string-face)
    ;; Keywords
    (,(regexp-opt mylang-keywords 'symbols) . font-lock-keyword-face)
    ;; Types
    (,(regexp-opt mylang-types 'symbols) . font-lock-type-face)
    ;; Capitalized words are types
    ("\\<\\([A-Z][a-zA-Z0-9_]*\\)\\>" . font-lock-type-face)
    ;; Function definitions
    ("^\\s-*\\(function\\|fun\\|def\\)\\s-+\\([a-z_][a-zA-Z0-9_]*\\)"
     . (2 font-lock-function-name-face))
    ;; Class definitions
    ("^\\s-*\\(class\\)\\s-+\\([A-Z][a-zA-Z0-9_]*\\)"
     . (2 font-lock-type-face))
    ;; Constants (UPPER_CASE)
    ("\\<\\([A-Z][A-Z0-9_]+\\)\\>" . font-lock-constant-face)
    ;; Decorators
    ("^\\s-*\\(@[a-zA-Z_][a-zA-Z0-9_.]*\\)" . font-lock-preprocessor-face)
    ;; Numbers
    ("\\<\\([0-9]+\\.?[0-9]*\\)\\>" . font-lock-constant-face)
    ;; Operators
    ("\\(->\\|=>\\|==\\|!=\\|<=\\|>=\\)" . font-lock-operator-face))
  "Font lock keywords for `mylang-mode'.")

;;;; Indent

(defun mylang-indent-line ()
  "Indent current Mylang line."
  (interactive)
  (let* ((current-indent (current-indentation))
         (previous-indent (mylang--previous-nonblank-indent))
         (base-indent (max 0 previous-indent))
         (new-indent base-indent))
    (save-excursion
      (beginning-of-line)
      ;; 如果上一行以 { ( [ 结尾, +1 indent
      (when (mylang--previous-line-ends-with-open)
        (setq new-indent (+ base-indent mylang-indent-offset))))
    ;; 如果当前行以 } ) ] 开头, -1 indent
    (when (looking-at "\\s-*[})\\]]")
      (setq new-indent (max 0 (- new-indent mylang-indent-offset))))
    ;; Apply
    (if (<= (current-column) current-indent)
        (save-excursion
          (indent-line-to new-indent))
      (save-excursion
        (indent-line-to new-indent)))))

(defun mylang--previous-nonblank-indent ()
  "Return indent of previous non-blank line."
  (save-excursion
    (forward-line -1)
    (while (and (not (bobp))
                (looking-at "\\s-*$"))
      (forward-line -1))
    (if (bobp)
        0
      (current-indentation))))

(defun mylang--previous-line-ends-with-open ()
  "Check if previous non-blank line ends with open bracket."
  (save-excursion
    (forward-line -1)
    (while (and (not (bobp))
                (looking-at "\\s-*$"))
      (forward-line -1))
    (end-of-line)
    (skip-chars-backward " \t")
    (memq (char-before) '(?\{ ?\( ?\[ ?\,))))

;;;; Imenu

(defvar mylang-imenu-generic-expression
  '(("Function" "^\\s-*\\(?:function\\|fun\\|def\\)\\s-+\\([a-zA-Z_][a-zA-Z0-9_]*\\)" 1)
    ("Class" "^\\s-*class\\s-+\\([A-Z][a-zA-Z0-9_]*\\)" 1)
    ("Constant" "^\\s-*\\([A-Z][A-Z0-9_]+\\)\\s-*=" 1))
  "Imenu generic expression for `mylang-mode'.")

;;;; Defun navigation

(defun mylang-beginning-of-defun (&optional arg)
  "Move to beginning of current defun."
  (interactive "p")
  (or arg (setq arg 1))
  (when (> arg 0)
    (forward-line 0)
    (while (and (> arg 0)
                (re-search-backward "^\\s-*\\(function\\|fun\\|def\\|class\\)\\s-+" nil t))
      (setq arg (1- arg)))))

(defun mylang-end-of-defun (&optional arg)
  "Move to end of current defun."
  (interactive "p")
  (or arg (setq arg 1))
  (when (> arg 0)
    (mylang-beginning-of-defun)
    (forward-line 1)
    (while (and (not (eobp))
                (looking-at "\\s-*$\\|\\s-+"))
      (forward-line 1))
    (while (and (not (eobp))
                (looking-at "^\\s-+"))
      (forward-line 1))))

;;;; Commands

(defun mylang-eval-buffer ()
  "Eval current buffer in Mylang interpreter."
  (interactive)
  (when (buffer-modified-p)
    (when (y-or-n-p "Buffer modified. Save? ")
      (save-buffer)))
  (let ((file (buffer-file-name)))
    (if file
        (make-process :name "mylang"
                      :buffer "*mylang-output*"
                      :command (list mylang-command file)
                      :sentinel (lambda (proc event)
                                  (when (string-match "finished" event)
                                    (display-buffer "*mylang-output*"))))
      (message "Buffer has no file"))))

(defun mylang-format-buffer ()
  "Format current buffer."
  (interactive)
  (indent-region (point-min) (point-max))
  (delete-trailing-whitespace))

;;;; Keymap

(defvar mylang-mode-map
  (let ((map (make-sparse-keymap)))
    (define-key map (kbd "C-c C-c") #'mylang-eval-buffer)
    (define-key map (kbd "C-c C-f") #'mylang-format-buffer)
    (define-key map (kbd "C-M-a") #'mylang-beginning-of-defun)
    (define-key map (kbd "C-M-e") #'mylang-end-of-defun)
    map)
  "Keymap for `mylang-mode'.")

;;;; Mode definition

;;;###autoload
(define-derived-mode mylang-mode prog-mode "Mylang"
  "Major mode for editing Mylang code.

\\{mylang-mode-map}"
  :group 'mylang
  :syntax-table mylang-mode-syntax-table

  (setq-local comment-start "// ")
  (setq-local comment-end "")
  (setq-local comment-start-skip "//+\\s-*")

  (setq-local font-lock-defaults
              '((mylang-font-lock-keywords)
                nil nil nil nil
                (font-lock-syntactic-face-function
                 . mylang--syntactic-face-function)))

  (setq-local indent-line-function #'mylang-indent-line)
  (setq-local indent-region-function nil)

  (setq-local beginning-of-defun-function #'mylang-beginning-of-defun)
  (setq-local end-of-defun-function #'mylang-end-of-defun)

  (setq-local imenu-generic-expression mylang-imenu-generic-expression))

(defun mylang--syntactic-face-function (state)
  "Return face for syntactic context STATE."
  (if (nth 3 state)
      'font-lock-string-face
    'font-lock-comment-face))

;;;; Auto-mode

;;;###autoload
(add-to-list 'auto-mode-alist '("\\.myl\\'" . mylang-mode))
;;;###autoload
(add-to-list 'interpreter-mode-alist '("mylang" . mylang-mode))

(provide 'mylang-mode)
;;; mylang-mode.el ends here
```

### 代码逐段讲解

**File header** (`;;; mylang-mode.el --- ...`): 这是 Emacs 包的标准 header。`lexical-binding: t` 必须在第一行——告诉 Emacs 启用 lexical binding (Module 3 W3 学过,Module 6 W2 深入)。没有这个,所有 let 都是 dynamic,闭包不工作。

**`;;;###autoload`** 注释是给 `update-directory-loads` 用的——它在生成 `mylang-mode-autoloads.el` 时,把这个注释下面的 form 抽出来,放到 autoloads 文件。这样用户装包后不需要立刻 `require`,首次调用 `mylang-mode` 时才加载 (Module 6 W3)。

**`defgroup` + `defcustom`** (lines 54-67): `defgroup` 创建一个 customize 组,`defcustom` 定义用户可配置变量。它们的关系: `defcustom` 的 `:group` 指向 `defgroup`,这样在 `M-x customize` 里能找到。这是 Emacs 的"配置 UI 系统"——所有包都用,避免每个包自己发明 UI。

**Syntax table** (lines 71-101): 这是 week 7 的核心。`modify-syntax-entry ?/ ". 124b" st` 这个魔法字符串表示: `/` 是 punctuation (`.`),且和 `\n`、`*` 一起构成 C 风格注释 (`124b`)。具体: `1` = comment start first char, `2` = comment start second char, `4` = comment end second char, `b` = "b" style comment (C-like)。这让 Emacs 知道 `/* ... */` 和 `// ...` 都是注释,font-lock、indent、movement 都依赖这个。

**Font-lock keywords** (lines 105-146): 每个 entry 是 `(REGEXP . FACE)` 或 `(REGEXP . (SUBEXP FACE))`。第二种形式用 `SUBEXP` 从 REGEXP 提取子组——比如 `^\\s-*\\(function\\|fun\\|def\\)\\s-+\\([a-z_]+\\)` 提取第 2 组 (函数名),高亮成 `font-lock-function-name-face`。`regexp-opt` 是优化函数,把 `("if" "else" ...)` 编译成高效的 `(?:if|else|...)` 形式。

**Indent** (lines 150-192): 缩进算法是 Emacs major mode 最难写的部分。这里用简单规则: 上一行以 `{` `(` `[` `,` 结尾 → 缩进 +offset;当前行以 `}` `)` `]` 开头 → 缩进 -offset。真实 major mode (如 `python.el`) 用 SMIE 或更复杂状态机,但这个简化版足以学原理。

**Imenu generic expression** (lines 196-200): imenu 是 Emacs 的"函数/类列表"——按 `C-c C-l` 在 mode line 看到。每个 entry 是 `(TITLE REGEXP SUBEXP)`,在 buffer 找匹配的 REGEXP,取 SUBEXP 组作为名字。

**Defun navigation** (lines 204-226): `beginning-of-defun` / `end-of-defun` 让 `C-M-a` / `C-M-e` 在函数间跳。原理是 `re-search-backward` 找 defun 开头 (`function` `class` 等)。

**Mode definition** (lines 265-289): `define-derived-mode` 是宏,它做 4 件事: 1) 创建 `mylang-mode` 函数 (调用父 mode 后跑 body); 2) 创建 `mylang-mode-hook` 变量; 3) 设置 keymap 继承; 4) 处理 syntax table。`setq-local` 让变量 buffer-local——每个 buffer 有独立 `comment-start`、`font-lock-defaults` 等。

**`font-lock-defaults`** 的 6 元素 list 是 font-lock 引擎的配置——第一个元素是 keywords list,后面的 nil 表示"不要 case-fold"、"不要 syntax strings"等。最后一对 `(font-lock-syntactic-face-function . mylang--syntactic-face-function)` 让我们自定义"什么时候用 string face,什么时候用 comment face"。

### `mylang-mode-test.el`

ERT 测试是"软件工程"和"业余脚本"的分水岭。业余的 mode 改一行代码就重启 Emacs 看效果,有 bug 不知道哪坏。专业 mode 每个功能有 ERT 测试,改代码后跑测试立刻知道有没有破坏。

下面是 10 个 ERT 测试,覆盖 major mode 的各个部分。

```elisp
;;; mylang-mode-test.el -*- lexical-binding: t; -*-

(require 'ert)
(require 'mylang-mode)

(ert-deftest mylang-keywords-test ()
  (should (member "if" mylang-keywords))
  (should (member "function" mylang-keywords)))

(ert-deftest mylang-syntax-table-test ()
  (with-temp-buffer
    (mylang-mode)
    (should (eq (char-syntax ?_) ?w))
    (should (eq (char-syntax ?a) ?w))))

(ert-deftest mylang-indent-test ()
  (with-temp-buffer
    (mylang-mode)
    (insert "function foo() {\n")
    (insert "    if (x) {\n")
    (insert "        y\n")
    (insert "    }\n")
    (insert "}\n")
    (goto-char (point-min))
    (forward-line 1)
    (mylang-indent-line)
    (should (= (current-indentation) mylang-indent-offset))))

(ert-deftest mylang-font-lock-test ()
  (with-temp-buffer
    (mylang-mode)
    (insert "function foo() { return 1; }")
    (font-lock-fontify-buffer)
    (goto-char (point-min))
    (re-search-forward "function")
    (should (eq (get-text-property (match-beginning 0) 'face)
                'font-lock-keyword-face))))

(ert-deftest mylang-imenu-test ()
  (with-temp-buffer
    (mylang-mode)
    (insert "function foo() {}\n")
    (insert "function bar() {}\n")
    (insert "class Baz {}\n")
    (let ((imenu-index (imenu--make-index-alist)))
      (should (assoc "Function" imenu-index)))))

(ert-deftest mylang-comment-test ()
  (with-temp-buffer
    (mylang-mode)
    (insert "// comment")
    (should (eq (get-text-property 1 'face)
                'font-lock-comment-face))))

(ert-deftest mylang-string-test ()
  (with-temp-buffer
    (mylang-mode)
    (insert "\"hello\"")
    (font-lock-fontify-buffer)
    (should (eq (get-text-property 2 'face)
                'font-lock-string-face))))

(ert-deftest mylang-defun-navigation ()
  (with-temp-buffer
    (mylang-mode)
    (insert "function foo() {\n  x\n}\n")
    (insert "function bar() {\n  y\n}\n")
    (goto-char (point-max))
    (mylang-beginning-of-defun)
    (should (looking-at "function bar"))))

(ert-deftest mylang-mode-map-test ()
  (should (keymapp mylang-mode-map))
  (should (lookup-key mylang-mode-map (kbd "C-c C-c"))))

(ert-deftest mylang-auto-mode-test ()
  (with-temp-buffer
    (set-visited-file-name "test.myl" t t)
    (normal-mode)
    (should (eq major-mode 'mylang-mode))))

(provide 'mylang-mode-test)
;;; mylang-mode-test.el ends here
```

### 测试设计原则

这些测试覆盖 major mode 的核心维度:

- **`mylang-keywords-test`**: 数据测试——确认你的 keyword list 包含期望值。简单但重要,防止 typo。
- **`mylang-syntax-table-test`**: 用 `with-temp-buffer` 创建临时 buffer,启用 mode,检查 `char-syntax`。这测的是 syntax table 的 `modify-syntax-entry` 是否正确。
- **`mylang-indent-test`**: 在 buffer 插入代码,调 `mylang-indent-line`,检查 `current-indentation`。这是 indent 函数的"行为测试"——不关心实现,只关心结果。
- **`mylang-font-lock-test`**: 调 `font-lock-fontify-buffer`,然后查 text property `'face`。这是 week 7 学过的——font-lock 通过 text property 实现高亮。
- **`mylang-imenu-test`**: 调 `imenu--make-index-alist`,检查是否包含期望类别。
- **`mylang-defun-navigation`**: 插入多个 defun,跳到末尾,调 `beginning-of-defun`,检查光标位置。

注意每个测试都用 `with-temp-buffer` ——它创建临时 buffer,scope 退出时自动删除。这是 Elisp 测试的标准 idiom: 不污染用户当前 buffer,测试可重复。

ERT 测试通过条件: `M-x ert-run-tests-interactively RET mylang-` 跑所有 `mylang-` 开头测试,应该全绿。

---

## 实施步骤

### Day 1: 骨架
创建文件,加 Commentary、Code、provide、define-derived-mode。

### Day 2: Syntax table
完整 syntax table,试注释 `// ...`。

### Day 3: Font-lock
关键词、类型、字符串、注释、函数名。

### Day 4: Indent
`mylang-indent-line`,处理 `{` `}`。

### Day 5: Imenu + defun navigation
imenu-generic-expression、beginning-of-defun。

### Day 6: Commands
eval-buffer、format-buffer。

### Day 7: ERT 测试 + 整合
测试,绑到 init.el,真实试用。

---

## 评分

| 项 | 满分 |
|---|---|
| 文件 ≥ 500 行 | 5 |
| Syntax table | 10 |
| Font-lock (≥ 8 patterns) | 15 |
| Indent | 15 |
| Imenu | 10 |
| Defun navigation | 10 |
| Keymap | 5 |
| auto-mode-alist | 5 |
| ERT 测试 (≥ 10) | 15 |
| 文档 (Commentary + docstring) | 10 |
| **加分: PEG 解析某结构** | +5 |
| **加分: narrowing 命令 (C-c n)** | +3 |
| **加分: mode abbrev table** | +2 |

满分 100。70+ 合格。加分项超过 100 不计入,但作为"超出预期"的标识。

---

## 日志

写 `logs/module-06.md`:

```markdown
# Module 6 学习日志

**用时**: ___ 小时
**Capstone 用时**: ___ 小时
**Capstone 评分**: ___/100

## 8 周心得

### Week 1: 数据类型
### Week 2: Eval + Variables
### Week 3: Macros + Loading
### Week 4: Debugging
### Week 5: Minibuf + Commands + Keymaps
### Week 6: Modes + Files + Buffers
### Week 7: Text + Display
### Week 8: Processes + OS

## Capstone (Major Mode)

- 行数:
- 测试数:
- 用了什么特性:

## 我学到的最重要 5 件事

## 下一步

进入 Module 7: 写一个真实的包
```

---

更新 `PROGRESS.md`: Module 6 ✅。

进入 Module 7: `07-package-dev/README.md`。

**恭喜**。

你已经能写完整的 major mode。
你能读任何 elisp。
你接近"Elisp 极客"的境界。

Module 7 把你的能力**发布到 MELPA**。
Module 8 让你**贡献开源**。
