# Week 3: Macros + Loading + Compile

> **Ref 章节**: macros, customize, loading, compile
> **目标**: 写自己的宏,理解加载机制,byte-compile

这周是 Elisp 的"高级"部分。宏让你"延伸语言"——你能发明自己的 control structure、DSL、boilerplate-killer。加载机制决定你的代码怎么进入 Emacs 运行时。byte-compile 让代码飞快。

这三者结合让你能写**真正的 package**——不是 init.el 片段,而是可发布、可重用、可优化的代码模块。

---

## 1. Macros (macros.texi)

### 1.1 宏是什么

**宏** = 代码生成器。它接收**未求值的 form**,生成新 form,然后 eval。

普通函数: 接收已 eval 的值,返回值。
宏: 接收未 eval 的 form,返回新 form,然后 eval 这个新 form。

这个区别极重要——宏控制"什么时候 eval 什么"。这让宏能做函数做不到的事: 实现新 control flow (像 `when`、`while`)、改变求值规则 (像 `let` 不 eval 第一个参数)。

```elisp
(defmacro my-when (cond &rest body)
  `(if ,cond (progn ,@body)))

(my-when t (foo) (bar))
;; 展开: (if t (progn (foo) (bar)))
```

`(my-when t (foo) (bar))` 的执行过程:
1. `(foo)` 和 `(bar)` **不** eval (它们是宏的参数,但宏不 eval 参数)
2. 宏的 body 跑,生成 `(if t (progn (foo) (bar)))`
3. 这个生成的 form 被 eval——`if` 条件 t 为 true,执行 progn,foo 和 bar 跑

这就是为什么宏能"控制 eval 时机"——`(foo)` 不在调用 my-when 时 eval,而是在生成的 if 内 eval。

### 1.2 backquote

backquote (反引号) 是宏的核心工具——template + 插值。

```elisp
`(a b c)                      ; → (a b c)
`(a ,b c)                     ; → b 被 eval 插入
`(a ,@list c)                 ; → list 被 splice
```

`` `(a b c) `` 几乎等于 `'(a b c)`——不 eval。但 `` ` `` 内可以用 `,` 和 `,@`:
- `,X`: eval X,把结果插入
- `,@X`: eval X,结果必须是 list,把它 splice 进当前 list

实战: 宏 body 几乎总是 backquote——生成 form 模板,参数用 `,` 插入。

`,` eval 后插入。
`,@` eval 后 splice (必须是 list)。

### 1.3 例子: my-unless

```elisp
(defmacro my-unless (cond &rest body)
  "Like `unless'."
  `(if (not ,cond) (progn ,@body)))

(my-unless nil (foo))
;; → (if (not nil) (progn (foo)))
;; → (foo)
```

`unless` 的实现就是这样——把 `(unless COND BODY)` 转成 `(if (not COND) (progn BODY))`。整个 unless 在 eval 前展开成 if,所以不需要特殊支持。

这是宏的力量——你不需要 C 改 Emacs 源码就能加新 control structure。

### 1.4 macroexpand

`macroexpand` 让你看宏展开成什么——调试宏的关键工具。

```elisp
(macroexpand '(my-when t (foo)))
;; → (if t (progn (foo)))

(macroexpand-all '(my-when t (my-when t2 (foo))))
;; → (if t (if t2 (progn (foo))))
```

`macroexpand` 只展开第一层。`macroexpand-all` 递归展开所有。这是写宏的必备——展开结果对不对,直接决定宏对不对。

### 1.5 gensym (避免变量捕获)

宏的"经典 bug"——变量捕获 (variable capture)。如果宏生成的代码用了 `temp`,而调用上下文也有 `temp`,两者冲突。

```elisp
(defmacro my-swap (a b)
  (let ((temp (gensym)))
    `(let ((,temp ,a))
       (setq ,a ,b)
       (setq ,b ,temp))))

(let ((temp 1) (other 2))
  (my-swap temp other)
  (list temp other))
;; → (2 1) (gensym 防止冲突)
```

如果不用 gensym,直接 ``(let ((temp ,a)) ...)``,生成的代码:
```elisp
(let ((temp 1))           ; temp = 1
  (setq temp other)       ; temp = 2 (改了外层的 temp!)
  (setq other temp))      ; other = 2 (应该是 1)
```

结果错乱——因为宏内部的 `temp` 和外部的 `temp` 冲突。gensym 创建唯一 symbol (如 `temp42`),保证不和任何外部冲突。

这是 Elisp 宏的"unhygienic" 特性——和 Scheme 的 hygienic macro 不同,Scheme 自动避免捕获。Elisp 程序员要手动 gensym。这是历史选择——hygienic 实现复杂,1985 年的 Elisp 选了简单。

### 1.6 pcase-lambda / pcase 宏

`pcase` 不只是 control structure——它有 `pcase-let`、`pcase-lambda` 等衍生宏。

```elisp
(defun my-classify (x)
  (pcase x
    ((pred numberp) "number")
    ((pred stringp) "string")
    (`(,a ,b) (format "two-element: %s %s" a b))
    (_ "other")))
```

这个函数用 pcase 模式匹配——根据 x 的类型/形状返回不同结果。比 cond 简洁。

(Module 6 W3 详细)

### 1.7 cl-lib 宏

`cl-lib` 提供 CL 风格的宏——loop、destructuring、multiple values 等。这些是写复杂代码的工具。

```elisp
(require 'cl-lib)

(cl-loop for x in '(1 2 3 4 5)
         when (> x 2)
         sum x)                ; → 12

(cl-destructuring-bind (a b c) '(1 2 3)
  (list a b c))                ; → (1 2 3)

(cl-multiple-value-bind (a b) (values 1 2)
  (list a b))                  ; → (1 2)

(cl-letf (((my-func)) 5)
  (my-func))                   ; my-func 暂时返回 5
```

`cl-loop` 是强力的 loop DSL——SQL-like 语法:`for x in lst`, `when (> x 5)`, `sum x`。比 dolist + collect 模式简洁。

`cl-destructuring-bind` 是 list 解构——`(a b c) = (1 2 3)`。

`cl-letf` 是"临时绑定"——在 scope 内,某 form 返回固定值。测试时 mock 用。

### 1.8 自定义宏常见模式

下面是几个实用宏模式,你应该内化。

```elisp
(defmacro my-comment (&rest _)
  "Ignore body."
  nil)

(my-comment
  (foo)                       ; 不被执行
  (bar))                      ; 不被执行
;; → nil
```

`my-comment` 让你"注释掉"代码块——比 `;` 注释好用,因为它保持代码结构 (能 indent、能高亮)。Common Lisp 有 `#|...|#` 块注释,但 Elisp 用宏更灵活。

```elisp
(defmacro my-with-temp-buffer (name &rest body)
  `(with-temp-buffer
     (let ((,name (current-buffer)))
       ,@body)))

(my-with-temp-buffer buf
  (insert "hi")
  (buffer-string))
```

`my-with-temp-buffer` 是 with-temp-buffer 的扩展——让你命名 temp buffer,在 body 里引用。这是写"resource manager" 宏的标准模式——open resource,bind 到名字,跑 body,自动 close。

---

## 2. Customization Settings (review)

(Module 4 学过)

```elisp
(defgroup my-app nil "..." :group 'tools)
(defcustom my-var 5 "..." :type 'integer :group 'my-app)
```

`defgroup` 创建 customize 组,`defcustom` 定义用户可配置变量。`:type` 描述变量类型 (integer、string、alist 等),决定 customize UI 怎么显示。

---

## 3. Loading (loading.texi)

### 3.1 load

`load` 是最基础的加载——读文件,eval 内容。

```elisp
(load "foo")                   ; 加载 foo.el 或 foo.elc
(load "foo.el")                ; 指定扩展名
(load-file "~/foo.el")         ; 绝对路径
(load-library "foo")           ; 同 load
```

`load` 在 `load-path` 找文件,优先 `.elc` (byte-compiled),fallback `.el`。如果文件已加载 (在 `load-history`),默认**重新加载**——但 `load` 有可选参数控制。

实战很少直接 load——通常用 require 或 autoload。

### 3.2 require

`require` 是"声明依赖"——加载某 feature,如果还没加载。

```elisp
(require 'foo)                 ; 加载 foo feature (如果还没)
(require 'foo "path")          ; 指定路径
(featurep 'foo)                ; foo 是否已加载
(features)                     ; 所有已加载 features 列表
```

feature 是 symbol——一个 package 在文件末尾 `(provide 'foo)`,表示 "我加载完成了,我是 feature foo"。`(require 'foo)` 检查 `features` 是否含 foo,如果不含,加载 "foo" 文件 (load-path + foo.el)。

文件末尾必须有 `(provide 'foo)`,才算 feature。

require 是模块化的基础——你的 package 顶部写 `(require 'cl-lib)`,表示"我依赖 cl-lib"。Emacs 自动加载它,且只加载一次 (idempotent)。

### 3.3 autoload

autoload 是"延迟加载"——你声明函数,但首次调用时才加载文件。这让 Emacs 启动快 (不需要加载所有 package)。

```elisp
(autoload 'my-func "my-package"
  "Documentation."
  t)                            ; interactive?

(my-func)                       ; 第一次调用时加载 my-package
```

`autoload` 让 `my-func` 在调用前是 "autoload function object"——docstring 显示,但 body 不在。调用时,Emacs 加载 `my-package` (load-path 找),加载后 `my-func` 被真定义,然后调用继续。

或用 `;;;###autoload` 注释:

```elisp
;;;###autoload
(defun my-func ()
  "..."
  ...)
```

跑 `M-x update-directory-loads` 生成 autoloads。

`;;;###autoload` 是 magic comment——告诉 autoload 生成器"下面的 form 是 autoload"。生成器扫描目录,把所有 autoload form 收集到 `<package>-autoloads.el`,用户 require 这个 autoloads 文件后,所有 autoload 都注册。

这是 MELPA/ELPA 包的工作方式——你装包,实际加载的是 autoloads;真正代码在你第一次用某个命令时才加载。

### 3.4 load-path

`load-path` 是 list of dirs——load/require 查找文件的目录列表。

```elisp
load-path                      ; list of dirs
(add-to-list 'load-path "~/my-lisp/")
```

Emacs 默认 load-path 包含 site-lisp、用户 lisp 目录。你加自己的目录,让 Emacs 找到你的 .el 文件。

### 3.5 load-suffixes

```elisp
load-suffixes                  ; 默认 (".elc" ".el")
load-file-rep-suffix           ; ".gz" 等
```

`load-suffixes` 决定 load 时尝试的扩展名顺序——`.elc` 优先 (compiled),fallback `.el`。

### 3.6 after-load

`with-eval-after-load` 是"等某 feature 加载后再跑代码"——常用于"在 org mode 加载后配置它的 keymap"。

```elisp
(with-eval-after-load 'org
  (define-key org-mode-map ...))
;; 等 org 加载后跑
```

为什么需要? 因为 org 可能还没加载 (用户没打开 .org 文件)。如果你直接 `(define-key org-mode-map ...)`,而 org-mode-map 还不存在,报错。

`with-eval-after-load` 注册一个 callback,等 feature 加载后跑。

`eval-after-load` 是旧 API,推荐用 `with-eval-after-load`。

---

## 4. Byte Compilation (compile.texi)

### 4.1 什么是 byte-compile

把 .el (源码) 编译成 .elc (byte-code):
- 加载更快 (10x)——byte-code 比 Lisp form 解析快
- 运行更快 (2-10x)——byte-code interpreter 比 eval 快
- 隐藏源码——.elc 不易读

Emacs byte-code 是栈式虚拟机的指令——比 Lisp form 直接 eval 高效得多,但比 native code 慢。

### 4.2 编译命令

```bash
emacs --batch -f batch-byte-compile foo.el
```

或在 Emacs:
```
M-x byte-compile-file RET foo.el RET
M-x byte-compile-buffer
M-x byte-recompile-directory RET ~/elisp/ RET 0 RET
```

`--batch` 是命令行模式——不显示 UI,跑完退出。CI/CD 通常用这个 compile 所有 .el。

### 4.3 native compilation (Emacs 28+)

进一步编译成**原生码**——比 byte-code 快 2-3x。

```elisp
(native-compile-async "/path/")
```

或启动时 `--with-native-compilation` (Module 0 装过)。

native comp 用 GCC backend 把 byte-code 编译成机器码——比 byte-code interpreter 快。代价:编译慢 (gcc 编译每个文件几秒),磁盘占用大 (.eln 文件)。

文件:
- `.el` 源码
- `.elc` byte-code
- `.eln` native (在 `eln-cache/`)

### 4.4 byte-compile 警告

byte-compiler 会发警告——这些警告是 bug 的早期信号。

```elisp
(setq byte-compile-error-on-warn t)
(byte-compile-file "foo.el")
```

警告:
- `free variable`: 用了未 defvar 的变量——可能 typo,或忘记 require
- `unused lexical variable`: let 绑定但没用——dead code
- `interactive-only`: 某 function 不该在 non-interactive 用 (如 `next-line`)
- `not known to be defined`: 没 autoload / require——可能 typo

修复警告是好习惯——很多 bug 来自未 defvar 的变量 (dynamic binding 时污染全局)。

### 4.5 eval-and-compile

```elisp
(eval-and-compile
  (defconst my-version "1.0"))
;; 编译时和运行时都跑
```

`eval-and-compile` 的 body 在 compile 时和 load 时都跑。常用于定义 macro 需要在 compile 时用的常量。

### 4.6 eval-when-compile

```elisp
(eval-when-compile
  (require 'cl-lib))
;; 只编译时跑 (运行时不跑)
```

`eval-when-compile` 的 body 只在 compile 时跑——生成的 byte-code 不包含这个 form 的运行时效果。常用于"compile-time-only" 的 require (如 cl-lib 用于 macro,但运行时不需要)。

---

## 5. Narrowing and Widening (narrowing 在 buffers.texi + Chassell Ch 6)

### 5.1 第一性原理: 为什么需要 narrowing

考虑一个场景: 你打开一个 10000 行的 `.csv`,只想对其中一段跑 `replace-regexp`。如果不做任何隔离,正则引擎会扫整个 buffer——慢,且容易误改其他区域。你**真正想要的是**: 让 buffer 在这次操作中"暂时只剩这一段"。

这就是 narrowing 解决的问题——**它把 buffer 的可见范围限制到 [start, end)**。所有 point 移动、search、replace 操作都把这个 narrow 范围当作"整个世界"。point-min 返回 start,point-max 返回 end。bob/eob 也指 narrow 边界。从 Elisp 角度看,**narrowing 是"暂时改变 buffer 的逻辑边界"**——一种零拷贝的"虚拟子 buffer"。

为什么"零拷贝"? 因为 narrowing 不删任何字符——它只是设两个 marker (`narrow-begin`、`narrow-end`),所有 buffer 操作都查这两个 marker。`widen` 把它们复位到真实 buffer 边界。所以 narrow 是 O(1) 操作,对 1MB buffer 也瞬间完成。

更深层的设计哲学: **narrowing 让 Emacs 在"全局"和"局部"之间无缝切换**。你不需要复制数据到新 buffer (像 VSCode 那样开 split view),就能"专注"到一段代码。这是 Emacs 文本即数据哲学的延伸——buffer 既是全局,又能局部聚焦,因为编辑器本身就是 Lisp 机器,buffer 操作是函数。

Chassell 第 6 章专门讲 narrowing,因为它是 Emacs 独有的"专注机制"——大部分编辑器没有等价物。学会它让你的批量操作**安全且专注**。

### 5.2 核心 API

```elisp
(narrow-to-region START END)        ; 限制 buffer 到 [START, END)
(widen)                             ; 取消 narrowing
(narrow-to-page &optional MOVE-COUNT)  ; 限制到当前 page (form feed 分隔)
(narrow-to-defun &optional ARG)     ; 限制到当前 defun

(point-min)                         ; narrowed 起点 (widen 时是 1)
(point-max)                         ; narrowed 终点 (widen 时是 buffer-size+1)
(buffer-size)                       ; narrowed 后只算 narrow 范围
(buffer-narrowed-p)                 ; 当前是否 narrowed (Emacs 29+)
```

`(point-min)` 在没有 narrow 时返回 1;**narrowed 后返回 narrow 起点**。这关键——你的代码应该总是用 `(point-min)` 而不是硬编码 `1`,否则在 narrowed buffer 里行为错。

`(buffer-narrowed-p)` 是个布尔检查——常用于 mode-line 提示"当前在 narrowed 状态"。

### 5.3 save-restriction

`save-restriction` 是 narrowing 的"作用域保护"——保存当前 narrowing 状态,跑 body,然后恢复。这是写函数的必备——你的函数不应该**改变 caller 的 narrowing 状态**。

```elisp
(save-restriction
  (narrow-to-region 100 200)
  ;; ... 在 [100, 200) 操作 ...
  )
;; narrowing 自动恢复
```

实战中 `save-restriction` 经常和 `save-excursion` 一起用——`save-excursion` 保护 point,`save-restriction` 保护 narrowing。组合形式:

```elisp
(save-excursion
  (save-restriction
    (widen)
    ;; 在 full buffer 操作,不影响 caller 的 narrow
    ))
```

注意顺序——通常先 `save-excursion`,再 `save-restriction`。这样 widen 不会让 point 跑出 narrow 范围 (因为 `save-excursion` 先保存了)。

### 5.4 用户命令 (键位)

```
C-x n n     narrow-to-region      限制 region
C-x n w     widen                 取消 narrowing
C-x n p     narrow-to-page        限制到 page
C-x n d     narrow-to-defun       限制到当前 defun
```

`C-x n` 是 narrowing 的 prefix。`C-x n n` 最常用——选中一段代码,`C-x n n`,然后整段 focus。

mode-line 在 narrowed 时显示"**Narrow**"提示——这是 Emacs 防你忘记的关键 UI。

### 5.5 创造性用法 (5+)

**1. 安全的批量替换**: 在 narrowed 区域跑 `replace-regexp`,绝对不会动到其他区域。

```elisp
(defun my-replace-in-defun (from to)
  (interactive "sFrom: \nsTo: ")
  (save-excursion
    (save-restriction
      (narrow-to-defun)
      (goto-char (point-min))
      (while (re-search-forward from nil t)
        (replace-match to)))))
```

`narrow-to-defun` 把范围限制到当前函数,然后 replace——安全,不影响其他函数。

**2. 提速 isearch / font-lock**: 在 narrowed buffer,isearch 只搜 narrow 范围。对超大 buffer (> 100MB 日志),narrow 到感兴趣的时段,isearch 瞬间响应。

**3. 用 narrowing 模拟"临时 view"**: 你打开 `init.el`,narrow 到某个 use-package 块,专注编辑,然后 widen 看整体。不需要 split window,不需要复制到新 buffer——同一个 buffer,不同 focus。

**4. 函数式重构**: 把整个函数 `C-x n d` narrow 出来,然后跑 `mark-whole-buffer` (`C-x h` —— 但只 mark narrow 范围),`indent-region`——只重排这一函数,不影响其他。

**5. 测试 / demo 隔离**: 写教程或 demo 时,`narrow-to-region` 让你在一个 buffer 内有多"独立"区段。每段 narrow 后跑演示,widen 看完整。

**6. Org-mode 的 narrowing 子树**: Org 有自己的 `org-narrow-to-subtree` (`C-x n s`) 和 `org-narrow-to-block`——基于 narrowing,但针对 Org 结构。

**7. 编程式 narrowing (lock 用户焦点)**: 写一个 minor mode 在编辑"必须专注"的部分时自动 narrow——比如代码评审工具,只让用户改特定行。

### 5.6 陷阱 (3+)

**陷阱 1: 忘记 widen 导致"buffer 看起来少了内容"**。新手常用 `C-x n n` 后忘了 widen,以为 Emacs 删了内容——其实只是 hidden。mode-line 的 `Narrow` 提示是唯一的救命提示。养成习惯: narrow 完一定 `C-x n w` 恢复。

**陷阱 2: Elisp 函数里 point-min/point-max 的语义**。在 narrowed buffer,`(point-min)` 不是 1。如果你的代码硬编码 `(goto-char 1)` 在 narrowed buffer 里可能跳到 narrow 外 (但默认会 clamp 到 narrow 内)。**永远用 `(point-min)` 和 `(point-max)`**。

**陷阱 3: `buffer-substring` / `buffer-string` 返回的是 narrowed 范围还是 full?** —— `buffer-string` 返回 narrowed 范围 (从 point-min 到 point-max);`buffer-substring-no-properties` 也只在 narrow 范围有效。如果你需要 full buffer 内容,**必须先 widen 或用 `(buffer-substring-no-properties 1 (buffer-size))`**——但注意 `buffer-size` 也只返回 narrow 范围大小。要拿到真实大小,`(1- (position-bytes (buffer-end 1)))` 或先 widen。

**陷阱 4: `print` / `format` 显示的字符位置在 narrowed buffer 是相对 narrow 的**——你 debug 时看到的 `(point) = 5` 可能是 narrowed 后的 5,实际在 full buffer 是 1005。如果你不知道这点,会找错位置。

**陷阱 5: `narrow-to-region` 不会被 `save-excursion` 自动恢复**——它们是两个独立的 save。要保护 narrowing 必须用 `save-restriction`。这是常见 bug——以为 `save-excursion` 也保护 narrow,结果函数返回后 narrow 状态被改了。

### 5.7 内部实现 (高层)

Emacs buffer 内部有 `BEGV`/`ZV` (beg visible / end visible) 两个 C 变量——narrowing 就是修改这两个变量。所有 buffer 操作 (forward-char、search、insert) 都先检查 [BEGV, ZV),不在范围内的报 "Buffer is read-only" 或自动 clamp。`BEG`/`Z` 是真实 buffer 边界 (常量 1 / buffer_size+1)。

widen 操作就是把 BEGV = BEG、ZV = Z。这就是为什么 widen/narrow 都是 O(1)——只是改两个整数。

---

## 6. 自测

1. 宏和函数区别?
2. backquote 的 `,` 和 `,@` 区别?
3. 怎么避免宏变量捕获?
4. `load` 和 `require` 区别?
5. autoload 干啥?
6. byte-compile 好处?
7. `;;;###autoload` 干啥?
8. narrowing 改变 `point-min` 的语义吗?
9. `save-restriction` 和 `save-excursion` 区别?
10. 为什么 `C-x n n` 后 isearch 速度变快?

**答案**:
> 1. 宏接收未 eval 的 form,生成代码;函数接收已 eval 的值
> 2. , 插入;, @ splice
> 3. gensym
> 4. load 加载文件;require 加载 feature (如已加载则 noop)
> 5. 函数被调用时才加载文件
> 6. 加载快 10x,运行快 2-10x
> 7. 标记下一个 form 为 autoload
> 8. 是,narrow 后 point-min 是 narrow 起点
> 9. save-restriction 保护 narrowing;save-excursion 保护 point/mark/window,不保护 narrowing
> 10. 因为 search 范围缩小到 narrow,不扫全 buffer

---

## 7. Exercises

### Ex 3.1: my-unless
```elisp
(defmacro my-unless (cond &rest body)
  `(if (not ,cond) (progn ,@body)))
```

### Ex 3.2: my-while-with-collect
```elisp
(defmacro my-while-collect (cond collect-form &rest body)
  ;; 跑 while,每次收集 COLLECT-FORM 到 list
  )
```

<details><summary>答案</summary>

```elisp
(defmacro my-while-collect (cond collect-form &rest body)
  `(let (result)
     (while ,cond
       ,@body
       (push ,collect-form result))
     (nreverse result)))
```

</details>

### Ex 3.3: my-with-feature
```elisp
(defmacro my-with-feature (feature &rest body)
  ;; require FEATURE,如果成功跑 BODY
  )
```

<details><summary>答案</summary>

```elisp
(defmacro my-with-feature (feature &rest body)
  `(when (require ,feature nil t)
     ,@body))
```

</details>

### Ex 3.4: my-autoload-setup
写 init.el 片段:
```elisp
;; 加 my-todo.el,但延迟加载
```

<details><summary>答案</summary>

```elisp
;;;###autoload
(autoload 'my-todo "my-todo" nil t)
(global-set-key (kbd "C-c t") 'my-todo)
```

</details>

### Ex 3.5: byte-compile 检查
```
M-x byte-compile-file RET ~/.config/emacs/init.el RET
```

修复所有 warning。

### Ex 3.6: 用 narrowing 安全 replace
写一个函数: 在当前 defun 内把所有 `foo` 替换为 `bar`,不影响其他 defun。

<details><summary>答案</summary>

```elisp
(defun my-replace-in-defun (from to)
  (interactive "sFrom: \nsTo: ")
  (save-excursion
    (save-restriction
      (narrow-to-defun)
      (goto-char (point-min))
      (while (re-search-forward (regexp-quote from) nil t)
        (replace-match to nil t)))))
```

</details>

### Ex 3.7: 用 narrowing 验证 buffer-narrowed-p
```
C-x n n  (选 region)
M-: (buffer-narrowed-p)  ; → t
C-x n w
M-: (buffer-narrowed-p)  ; → nil
```

---

## 8. 下一步

进入 `concept-anchor.md` + `exercises.md`,然后 `../week-04-debugging/`。
