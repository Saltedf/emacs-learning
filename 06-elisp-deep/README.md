# Module 6: Elisp 深度 (Reference)

> **目标**: 精读 `emacs-lispref-30.2` 核心章节,达到能读懂任何 elisp 代码的水平
> **时长**: 8 周 (~160 小时) ★核心模块★
> **难度**: ★★★★★
> **依赖**: Module 3 (Chassell 基础) + Module 4-5 (实用经验)
> **核心产出**: 写一个完整的 major mode (500+ 行 + ERT)

---

## 0. 这个模块在学什么

Module 3 的 Chassell 教你**入门**: list、lambda、buffer 怎么用。
Module 4-5 教你**实用**: 配置 init.el、写自己的 todo-list。
Module 6 教你**精通**: 精读 Emacs Lisp Reference Manual (lispref),达到能读懂任何 elisp 代码 (包括 Org、Magit、LSP mode 这种万行项目) 的水平。

为什么必须读 lispref? 因为前面那些教程都是"归纳"——告诉你"这么做"。但 Emacs 内部 API 有上千个 function,你不可能归纳完。**lispref 是规范**,告诉你"为什么这么做"和"边界在哪"。读完 lispref,你不再需要教程: 任何 function,你看 docstring + lispref 就能搞懂。

这周开始,你将精读 Emacs Lisp Reference Manual 的核心章节:
- 数据类型底层 (objects, numbers, strings, lists, sequences, hash) —— Module 3 的"用什么"变成"为什么这么设计"
- 求值规则 (eval, control, variables, functions, macros) —— 理解 lexical/dynamic 这 30 年争议
- 用户交互 (commands, keymaps, minibuffer) —— Emacs 的"命令循环"是它所有交互的核心
- Buffer 系统 (modes, files, buffers, windows, markers) —— Emacs 区别于其他编辑器的根本
- 文本操作 (text properties, overlays, display, search, syntax) —— "富文本"在 Emacs 里的实现
- 进程 (processes, os, threads, package) —— Emacs 怎么和 OS、网络、并发交互

完成后你能:
- 读任何 elisp 代码 (即使没 docstring)
- 写一个完整的 major mode (font-lock + indent + imenu)
- 用 Edebug 单步调试别人写的代码
- 写宏 (defmacro) 改变语言本身
- 用 byte-compile / native comp 让代码飞起来

最重要的是: 你能**像 Emacs 设计时的人那样思考**。当你看到某个 API,你会想"他们为什么这么设计,这背后的权衡是什么"。这是从"用户"到"作者"的跃迁。

---

## 1. 8 周计划

| 周 | Ref 章节 | 主题 | 产出 |
|---|---|---|---|
| W1 | objects, numbers, strings, lists, sequences, records, hash, streams | 数据类型 | 实现 hash/list 操作 |
| W2 | symbols, eval, control, variables, functions | 求值/变量/函数 | 写闭包/高阶 |
| W3 | macros, customize, loading, compile | 宏 + 加载 | 写自己的宏 |
| W4 | debugging, edebug, errors, tips | 调试 | 用 Edebug 单步 |
| W5 | minibuf, commands, keymaps | 用户交互 | 写自定义命令 |
| W6 | modes, help, files, buffers, windows, positions, markers | Modes + Buffer | 写 minor mode |
| W7 | text, nonascii, searching, syntax, parsing, peg, display | Text + Display | 写 font-lock |
| W8 | abbrevs, threads, processes, os, package, internals | 进程 + OS | 写异步 process |

**每周模板**:
- `ref-reading.md`: 内联讲解 ref 章节
- `concept-anchor.md`: editor manual 对照
- `exercises.md`: 10-15 题

---

## 2. 第一性原理深化

### 2.1 Lisp-2 的本质

Emacs Lisp 是 **Lisp-2**: 变量和函数有独立命名空间。这是它和 Scheme、Clojure (Lisp-1) 的根本区别之一。

为什么有这个区别? 历史原因: 1960 年代的 Lisp 机器内存紧张,把函数和变量分开 namespace 让 `(list 1 2 3)` 和 `(setq list '(1 2 3))` 能共存——你可以同时有一个叫 `list` 的变量和一个叫 `list` 的函数,互不冲突。这在 Common Lisp 和 Emacs Lisp 中保留至今。

Lisp-1 (Scheme, Clojure) 的好处是简洁——一个名字一个意思,函数也是值,直接传。坏处是你不能起一个叫 `list` 的变量,会遮蔽函数。Lisp-2 的代价是: 当你想把函数作为值传时,要用 `#'foo` 而不是 `'foo`,因为 `'foo` 默认是"变量 foo 的 symbol",而 symbol 不直接是函数。

```elisp
(list +)            ; 错: + 是函数,但作为变量查,没值
(symbol-function '+) ; → #<subr +>
(symbol-value '+)   ; → void-variable

(setq + 5)          ; 把 + 设为变量 5
(+ 1 2)             ; → 3 (+ 函数还在)
+                   ; → 5 (变量)
```

这段代码揭示了双 namespace 的本质: 同一个 symbol `+`,它的 value slot (变量) 和 function slot 是两个独立的"格子"。`(setq + 5)` 写到 value slot,`(+ 1 2)` 读 function slot。两者互不影响。

Module 3 提过这个概念,Module 6 W2 会从 symbol 内部结构 (4 个 slot) 角度再深入。

### 2.2 Buffer 数据模型

buffer 不是一个 string——它是一个**可变字符序列**附带大量状态。Emacs 的设计天才之处在于: buffer 不只装文本,它还装 point、mark、text-properties、overlays、markers、本地变量、语法表、键映射……所有这些构成一个"可编辑的富文档"对象。

```elisp
(buffer-string)              ; 全部文本
(buffer-substring A B)       ; 子串
(point)                      ; 当前位置 (integer)
(buffer-size)                ; 字符数
(gap-size)                   ; gap buffer 的 gap (内部优化)
(buffer-contents)            ; 同 buffer-string
```

底层是 **gap buffer**。为什么不用 rope (树的字符串) 或 piece table (VS Code 用的)? 因为 gap buffer 在 1985 年的硬件上足够快,且实现极简: 维护一段连续内存,中间留个 gap。插入: 把 gap 移到 point,在 gap 头写新字符,gap 缩小。删除: gap 扩大。所有编辑操作都是 O(1) 移动 gap + O(n) 写——而 gap 移动只在 jump 时发生,连续输入时 gap 已经在 point,几乎是 free 的。

VS Code 后期也加了一个 gap buffer 的优化,但 Emacs 是从一开始就用 gap——这就是为什么 Emacs 编辑超大文件 (> 100MB) 也不卡 (在合理范围内)。

### 2.3 Hook 机制深入

hook 是变量,值是函数列表。但 Emacs 的 hook 比这复杂:
- **normal hook**: 函数无参,顺序调用 (最常见)
- **abnormal hook**: 函数接收参数,可能影响返回值 (如 `find-file-hook` 接收文件名)
- **depth** (Emacs 28+): 函数有 depth,可以控制执行顺序 (`add-hook` 的 `:depth`)
- **buffer-local hook**: hook 变量本身是 buffer-local
- **file-local hook**: 通过 `dir-locals.el` 设置

hook 是 Emacs 的"事件系统"。它让 package 之间能解耦——Org mode 不需要知道你的 init.el 在哪,只需在 `org-mode-hook` 跑时通知。这是 Unix 哲学的"小工具 + 管道"在 Lisp 里的对应。

(Module 6 W6 详细讨论 hook 的设计)

### 2.4 Edebug: 强大的单步调试

Edebug 是 Emacs 自带的源码级调试器——你不需要装额外包。它和 Python 的 pdb、JS 的 debugger 一样,可以单步、看变量、设断点。但 Edebug 的特别之处: 它直接 instrument 你的 defun,不需要 recompile、不需要重启。

```elisp
(defun my-func (x)
  (interactive "nX: ")
  (edebug)                    ; 进入 edebug
  (let ((y (* x x)))
    (+ x y)))

(my-func 5)
;; 进入 edebug mode,可以单步、看变量
```

或在 defun 内任意位置 `M-x edebug-defun`。下次调用该函数,自动进入 Edebug mode。

Edebug 的核心理念: 调试器是源码的一部分,不是事后工具。这反映 Emacs 的整体设计——所有工具都应该"内置 + 可编程"。当你写完一个函数,立刻可以单步,不用切到外部工具。

(Module 6 W4 详细)

### 2.5 宏 (defmacro)

```elisp
(defmacro my-when (cond &rest body)
  `(if ,cond (progn ,@body)))

(my-when t (message "hi") (message "bye"))
;; 展开为: (if t (progn (message "hi") (message "bye")))
```

宏是**代码生成器**——它接收**未求值的 form**,生成新 form,然后 eval。这是 Lisp 区别于其他语言的杀手特性: 你可以让语言"自我延伸"。

对比其他语言:
- **Common Lisp**: 宏 + 反引号 (和 Elisp 几乎一样,但有 hygiene 系统 `&environment`)
- **Scheme**: `syntax-rules` + `syntax-case`,**hygienic** (自动避免变量捕获)
- **Clojure**: 宏 + reader macros,有 hygiene 工具但不强制
- **Elisp**: `defmacro` + 反引号,和 CL 类似,**无 hygiene**——你要自己用 `gensym` 防止捕获
- **Rust**: macro_rules! + 过程宏 (proc macro),完全不同的范式
- **C**: `#define`,纯文本替换 (危险,没类型)

Elisp 选择"不 hygiene"是有原因的: 1985 年实现简单,且很多 Emacs 内置宏依赖"非 hygiene"做巧妙事 (例如 `setf`)。代价是用户写宏时要小心,但实战中 `gensym` 就够了。

(Module 6 W3 详细讨论宏的细节、陷阱、和 `pcase-defmacro` 等高级用法)

---

## 3. 毕业检查

### 概念题 (8 周后能答)

1. 解释 `lexical-binding: t`
2. 描述闭包行为
3. 区分 `eq` / `equal` / `eql`
4. 写 `make-process` 跑 git
5. 解释 `font-lock-defaults`
6. 写 `define-derived-mode`
7. 区分 `with-current-buffer` 和 `save-excursion`
8. 用 Edebug 单步
9. 写一个 macro
10. 解释 text property 和 overlay 区别

### 实操

1. 写一个完整的 major mode (500+ 行 + ERT)
2. 写一个 macro 替代某内置函数
3. 写一个 process 跑 ripgrep
4. 写一个 font-lock 规则
5. 用 Edebug 调试别人的代码

---

## 4. 下一步

进入 `week-01-data-types/ref-reading.md`。
