# cl-lib 完整 + pcase 模式匹配 + EIEIO 对象系统

> **深度教程 / Week 2 补充**
> 这一份文件是整个 Elisp Deep 模块的"硬骨头"。它讲三件事: `cl-lib` 这个从 Common Lisp 借来的工具集、`pcase` 这个 ML 风格的模式匹配、`EIEIO` 这个 Emacs 自己的 CLOS 风格对象系统。三者**没有表面上的关系**——但放在一起讲,因为它们都是 Elisp 社区"如何写不那么痛苦的大程序"这条路上的关键工具。读完之后你会知道为什么 `magit`、`org-roam`、`lsp-mode` 的源码读起来像另一种方言。

这份文件比模块里其它文件长,因为这三个主题任何一个都值得单独一份。我尽量讲慢、讲细、讲真实代码而不是 `(my-fib 5)` 这种玩具。如果某段你读着累,放下、过几天回来——这些概念不是一次读完就懂,是写了几千行 Elisp 之后才慢慢"咔哒"一下。

---

## Part 1: cl-lib 的历史,以及为什么 2026 年我们还在用它

### 1.1 Common Lisp 1984: 一切的起点

1984 年,美国国防部高级研究计划局 (DARPA) 资助的一个委员会发布了 *Common Lisp the Language* (简称 CLtL1),作者 Guy Steele。目标是结束 1970 年代 Lisp 方言爆炸——Maclisp、Interlisp、ZetaLisp、Franz Lisp、Spice Lisp——每个 AI 实验室都自己造一套。Common Lisp 想做一个"所有人共用"的 Lisp。

1988 年,ANSI 委员会开始标准化,1994 年发布 ANSI INCITS 226-1994 (简称 ANSI CL)。这是今天 Common Lisp 的官方定义——像 C++ 一样,几十年靠标准驱动,而不是某个厂商。Common Lisp 体量巨大: 1000+ 页规范,涵盖 CLOS (对象系统)、条件系统 (条件/重启)、loop、format、序列、哈希表、包、reader macro。它是历史上最雄心勃勃的语言标准之一。

Common Lisp 之所以这么"大",因为它的目标是不丢失 Maclisp/ZetaLisp 里任何有用的特性。委员会成员来自 Symbolics (Lisp 机器厂商)、CMU、MIT、Berkeley——每个人都把自己实验室的好东西塞进来。Loop 来自 Lisp Machine 的 ZLISP;CLOS 来自 Xerox LOBB;format 来自 Maclisp 的 format。结果是 Common Lisp 像 Frankenstein——但每个 part 都是好 part。

### 1.2 Emacs Lisp 1985: Stallman 的简化

1985 年 Stallman 写 GNU Emacs 时的 Elisp 是另一个故事。Elisp 不是 Common Lisp 子集,而是直接从 Maclisp 借鉴+一些自己的简化。Stallman 故意不要 Common Lisp 的复杂——他要的是一个能嵌入编辑器的小 Lisp,而不是一个完整的 AI 编程环境。

后果: Elisp 缺少很多 Common Lisp 关键特性:

- 没有 lexical binding 直到 Emacs 24 (2012)——前 27 年 Elisp 是纯 dynamic binding
- 没有 keyword arguments (没有 `&key`)
- 没有 `loop` (没有那个有名的 `LOOP` 宏)
- 没有 multiple values (`values`/`multiple-value-bind`)
- 没有 CLOS 对象系统
- 没有 `format` 的全部 directives (只有部分)
- 没有 conditions/restart 系统 (用 `condition-case` 简化版)
- 没有 `destructuring-bind`
- 没有 `defstruct`

Stallman 的理由很直接: 编辑器脚本不需要这些,加了就慢、就大、就难学。这判断在 1985 完全正确——Emacs 19 在 1994 年发布时,整个发行版 ~2 MB,要在 8MB 内存的 486 上跑。

但 2000 年后,Emacs 用户开始写更大的程序——邮件客户端 (Gnus)、organizer (Org-mode)、Git 客户端 (Magit)。这些不是"编辑器脚本",是中型应用程序。Elisp 的简陋开始成为瓶颈。

### 1.3 cl.el: 1990s 的妥协

1991 年左右,有人 (Dave Gillespie, Caltech) 写了一个叫 `cl.el` 的库,通过宏和函数提供了 Common Lisp 的子集。这个库很快被广泛使用——几乎所有写大 Elisp 程序的人都 `(require 'cl)`。

`cl.el` 提供 `defun*` (支持 `&key`)、`loop` (Common Lisp 风格)、`defstruct*`、`destructuring-bind`、`multiple-value-bind`、`case`、`multiple-value-setq`,等等。它是一个"宏层 Common Lisp"——不是真在 Emacs 内核里实现,而是用 Elisp 自己模拟出来。

`cl.el` 有几个问题:

**第一,命名**。它直接用 Common Lisp 的名字——`defun`、`loop`、`case`、`defstruct`、`sort`、`union`、`intersection`、`remove`。这些名字污染全局 namespace。一个用户的简单程序用 `(require 'cl)` 之后,他自己的 `(defun case ...)` 就被覆盖了。这是个真实的坑——很多包因为这个冲突不能装在一起。

**第二,加载时机**。`cl.el` 太大,加载慢。Emacs 启动早期不能 require,否则拖慢启动。但有些 pre-loaded 文件需要 CL 特性,所以 Emacs 维护者搞了个 `cl-macs.el`、`cl.el`、`cl-seq.el`、`cl-extra.el` 的子集——混乱。

**第三,行为不一致**。`cl.el` 的某些函数和 Common Lisp 略有不同 (因为 Elisp 缺 lexical binding,某些宏不能完美模拟),所以有"看上去是 Common Lisp 实际不是"的坑。

整个 1990s 和 2000s,Elisp 社区一边用 `cl.el` 一边抱怨它。每年邮件列表都有"我们什么时候能修一下 cl.el"的讨论。

### 1.4 cl-lib 2012: Stefan Monnier 和 Leo Liu 的清理

2012 年,Stefan Monnier (蒙特利尔大学,Emacs 维护者,从 2000 年代中期到现在) 决定彻底修这个问题。他和 Leo Liu (在悉尼的 Emacs hacker) 合作,设计了 `cl-lib`。

`cl-lib` 的核心设计决定: **所有符号加 `cl-` 前缀**。`defun*` → `cl-defun`,`loop` → `cl-loop`,`defstruct*` → `cl-defstruct`,`case` → `cl-case`,`sort` (CL 版本) → `cl-sort`,等等。

这个决定看起来"丑",但解决了一切。`cl-lib` 加载之后不污染任何全局 symbol——你只有显式 `(require 'cl-lib)` 然后 `cl-xxx` 才能用。这让你能在生产代码里安全用 CL 特性,不会和别人的代码冲突。

`cl-lib` 2012 年随 Emacs 24.3 发布。当时 Stefan Monnier 同时推 lexical binding 默认 (`lexical-binding: t`),因为很多 CL 特性 (尤其闭包、`cl-labels`) 在 dynamic binding 下行为奇怪。

### 1.5 2026 年为什么还用 cl-lib

12 年后,`cl-lib` 仍然是 Emacs 大型 Elisp 项目的事实基础。原因是:

**Magit** (Git 客户端, Marius Vollmer 起头,现在 Jonas Bernoulli 维护) 整个代码用 `cl-defun`、`cl-defstruct`、`cl-loop`。看 magit 的 `magit-status.el`,你会看到几十处 `cl-loop`、上百处 `cl-defun`。Magit 的核心数据结构 `magit-section` 是 `cl-defstruct`。

**Org-mode** (Carsten Dominik 起头,现在 Bastien Guerry 维护) 大量用 `cl-lib`——`org-element.el` (Org 的 AST 解析) 用 `cl-defun` 关键字参数、`cl-case`、`cl-loop`。Org 9.x 之后所有新代码强制 `cl-lib`。

**lsp-mode** (Vibhav Pant 等) 是 LSP 客户端,数据结构复杂,大量用 `cl-defstruct`、`cl-defgeneric`/`cl-defmethod` (EIEIO 方法)。

**org-roam** (Jethro Kuan) 是笔记管理,用 EIEIO 表示数据库连接和缓存。

**Corfu/Cape/Orderless** (Daniel Mendler) 这一代补全栈,虽然追求简洁,但仍然 `cl-lib` 是基础。

另一方面,新一代包 (Vertico、Marginalia、Embark) 故意少用 `cl-lib`——它们追求"极简 Elisp",能用纯 Elisp 就不用。这是另一种风格选择。

### 1.6 为什么不"纯 Elisp"写

理论上你**完全可以**不用 `cl-lib`。`cl-loop` 能用 `dolist` + accumulator 写出来,`cl-defun &key` 能用 `&rest` + `plist-get` 手工解析,`cl-defstruct` 能用 `(defun make-foo () (vector 'foo nil nil nil))` + `(defun foo-a (f) (aref f 1))` 自己造。

但这种"自己造"的代码:

- **重复劳动**。每个项目都要重新发明一遍,互相不兼容。你 `make-foo` 我 `foo-create` 他 `(foo . make)`——三种风格在社区并存,读代码累。
- **质量低于 cl-lib**。`cl-lib` 经过 30 年迭代,corner case 都被处理。你的手写版本没有。
- **难读**。`cl-loop for x in xs when (pred x) collect (transform x))` 比 `(let (acc) (dolist (x xs) (when (pred x) (push (transform x) acc))) (nreverse acc))` 短 4 倍,而且意图明确——"filter 后 transform"。

**但 cl-lib 也有代价**。某些操作 (尤其 `cl-loop`) 编译后产生的字节码比手写大、慢;某些宏 (尤其 `cl-defmethod` 配 EIEIO) 调试困难。我会在后面具体说哪里。

总原则: **默认用 cl-lib,除非你写的是性能关键的 hot loop**。这是 2026 年 Emacs 社区的共识。

---

## Part 2: cl-loop 逐 token 拆解

`cl-loop` 是 cl-lib 里最有名、最有争议、也最有用的成员。它的语法像迷你语言——你读完这一节,你能读懂任何 `cl-loop` 调用;读完三遍,你能自己写。

为什么这一节最长? 因为 `cl-loop` 的"关键字" (for/in/when/collect/...) 互相组合会产生几十种结构,每一种都解决一个真实的列表处理问题。`loop` 来自 Symbolics Lisp Machine 的 ZLISP (1980s),是 Common Lisp 标准里最复杂的 single form。我慢慢讲。

### 2.1 一个完整例子: 找未 byte-compile 的 .el 文件

```elisp
(require 'cl-lib)

(cl-loop for file in (directory-files-recursively user-emacs-directory "\\.el\\'")
         for compiled = (concat file "c")
         unless (file-exists-p compiled)
         collect file)
```

这个例子返回一个 list——你 `.emacs.d` 下所有 `.el` 文件 (递归),对应的 `.elc` 不存在的那些。如果你改了 init 但忘了重新 byte-compile,这函数一眼看出哪些过期了。

逐行拆:

第一行 `(require 'cl-lib)`。`cl-loop` 在 `cl-macs.el` 里,通过 `cl-lib` 加载。不 require 你直接 `(cl-loop ...)` 会 void-function。

第二行 `(cl-loop ...)` 开始。`cl-loop` 是个宏,它展开成普通 Elisp——通常是 `while` + accumulator。

第三行 `for file in (directory-files-recursively user-emacs-directory "\\.el\\'")`。这是 "**for ... in ...**" 子句: `for VAR in LIST` 让 VAR 依次绑定 LIST 的每个元素。`directory-files-recursively` 是 Emacs 25+ 函数,返回指定目录下所有匹配 regex 的文件 (绝对路径)。这里 regex `"\\.el\\'"` 注意 `\\'`——这是 Emacs regex 的"字符串结束"anchor,避免匹配 `.elc`。

第四行 `for compiled = (concat file "c")`。这是 "**for VAR = EXPR**" 子句,不是迭代——它在每次循环**重新**计算 EXPR 并绑定 VAR。所以 `compiled` 在每次迭代开始等于 `(concat file "c")`——比如 file 是 `/home/me/.emacs.d/init.el`,那 compiled 就是 `/home/me/.emacs.d/init.elc`。注意是 `for ... =`,不是 `for ... = ... then`,所以每次循环都重新算 (没有 memoize)。

第五行 `unless (file-exists-p compiled)`。**unless** 子句是过滤——只有条件**为真时跳过** (注意: unless 是"除非",所以条件 nil 才执行下面)。换句话说: 如果 compiled 不存在,执行下一句。`file-exists-p` 检查文件存在,返回 t/nil。

第六行 `collect file`。**collect** 是 accumulation——把 `file` 加到结果列表。`cl-loop` 自动维护一个 accumulator list,最后返回。

整个 loop 展开 (大致) 成:

```elisp
(let ((--files-- (directory-files-recursively user-emacs-directory "\\.el\\'"))
      (--result-- nil))
  (while --files--
    (let* ((file (car --files--))
           (compiled (concat file "c")))
      (unless (file-exists-p compiled)
        (push file --result--)))
    (setq --files-- (cdr --files--)))
  (nreverse --result--))
```

看到没? `cl-loop` 6 行变普通 Elisp 9 行,而且普通版你需要心里"模拟"——`while`/`push`/`setq` 这些底层操作遮挡了"过滤未编译文件"这个**意图**。这就是 cl-loop 的核心价值: **表达意图**。

### 2.2 for 子句的所有变体

`for` 子句是 cl-loop 里最灵活的部分。它有 6 种主要形式。

#### 2.2.1 `for VAR in LIST` 和 `for VAR in LIST by STEP`

```elisp
(cl-loop for x in '(1 2 3 4 5) collect (* x x))
;; → (1 4 9 16 25)

(cl-loop for x in '(1 2 3 4 5) by #'cddr collect x)
;; → (1 3 5)
```

`by` 指定"前进函数",默认 `cdr`。`#'cddr` 是 "每次跳两个"——这给你奇数位置元素。

#### 2.2.2 `for VAR on LIST` 和 `for VAR on LIST by STEP`

```elisp
(cl-loop for sublist on '(1 2 3 4) collect sublist)
;; → ((1 2 3 4) (2 3 4) (3 4) (4))

(cl-loop for sublist on '(1 2 3 4) by #'cddr collect sublist)
;; → ((1 2 3 4) (3 4))
```

`on` 不绑定元素,而是绑定**子列表** (cons cell)——`sublist` 是当前剩余列表。用于你需要 `(car sublist)` 和 `(cdr sublist)` 一起,或要"chunk" 列表。

#### 2.2.3 `for VAR = EXPR` 和 `for VAR = EXPR then EXPR2`

```elisp
(cl-loop for i from 1 to 5
         for prev = 0 then i
         collect (cons prev i))
;; → ((0 . 1) (1 . 2) (2 . 3) (3 . 4) (4 . 5))
```

`for VAR = EXPR then EXPR2`: 第一次 VAR = EXPR,后续 VAR = EXPR2。这是"延迟"——EXPR2 里可以引用 VAR 自己 (前一帧的值),做 Fibonacci 类序列:

```elisp
(cl-loop repeat 10
         for a = 0 then b
         for b = 1 then (+ a b)
         collect a)
;; → (0 1 1 2 3 5 8 13 21 34)
```

逐 token 看这个 Fibonacci:
- `repeat 10` 跑 10 次
- 第一次: `a = 0` (初始值); `b = 1` (初始值)。collect a → 0
- 第二次: `a = b` 即 `a = 1` (前一帧的 b); `b = (+ a b)`,但这里 a 已经是新值 1,所以 `b = (+ 1 1) = 2`。注意!**cl-loop 的 for 子句按声明顺序更新**,所以 a 先变成 1,b 才计算。

这是个经典坑: 如果你把 `for a` 和 `for b` 顺序反过来,结果完全不同。**cl-loop 的 for 子句是有副作用的赋值,顺序敏感**。

```elisp
(cl-loop repeat 10
         for b = 1 then (+ a b)
         for a = 0 then b
         collect a)
;; 这里 a 是"上一帧的 b"——不同序列
```

记住这条规则: **多 for 子句按声明顺序求值**。这和 functional language 不一样——cl-loop 是命令式的。

#### 2.2.4 `for VAR from START to END` 和 `for VAR from START below END by STEP`

```elisp
(cl-loop for i from 1 to 5 collect i)
;; → (1 2 3 4 5)

(cl-loop for i from 1 below 5 collect i)
;; → (1 2 3 4)

(cl-loop for i from 10 downto 0 by 2 collect i)
;; → (10 8 6 4 2 0)

(cl-loop for i from 0 to 10 by 3 collect i)
;; → (0 3 6 9)
```

`to` 是闭区间,`below`/`above` 是开区间。`downto`/`upward` 指定方向 (默认 upward)。`by` 指定步长。

#### 2.2.5 `for VAR across ARRAY`

```elisp
(cl-loop for c across "hello" collect c)
;; → (104 101 108 108 111)

(cl-loop for x across [10 20 30] sum x)
;; → 60
```

`across` 用于 vector 和 string (任何 array)。比 `for x in (append arr nil)` 快——`in` 要求 list,先转换浪费。`across` 直接用 `aref`,O(1) 索引访问。

#### 2.2.6 `for VAR = EXPR while COND` 组合 (用 while 单独)

实际上"生成 + 边界"通常用 `while`/`until` 子句:

```elisp
(cl-loop for line = (read-line stream)
         while line
         collect (parse-line line))
```

这个是"读流"模式——`for X = (read...)` 每次循环重读,`while X` 检查是不是 EOF。但 Elisp 没有"流"概念,真实代码用 `buffer-substring` + `forward-line` 替代。

### 2.3 累积子句: collect/append/nconc/sum/count/maximize/minimize

```elisp
;; collect - 加到 list
(cl-loop for x in '(1 2 3) collect (* x 10))
;; → (10 20 30)

;; append - 把每个 LIST 结果 append 到结果
(cl-loop for x in '((1 2) (3 4) (5)) append x)
;; → (1 2 3 4 5)

;; nconc - 同 append 但用 nconc (破坏性,快)
(cl-loop for x in (list (list 1 2) (list 3 4)) nconc x)
;; → (1 2 3 4)

;; sum - 累加
(cl-loop for x in '(1 2 3 4) sum x)
;; → 10

;; count - 计数
(cl-loop for x in '(1 nil 3 nil 5) count x)
;; → 3

;; maximize / minimize
(cl-loop for x in '(3 1 4 1 5 9 2 6) maximize x)
;; → 9

(cl-loop for x in '(3 1 4 1 5 9 2 6) minimize x)
;; → 1
```

`collect` 是最常用,返回 list。`append`/`nconc` 把每次迭代的 list "拼"成一个大 list。`sum`/`count`/`maximize`/`minimize` 返回标量。

**陷阱**: `append` 在每次迭代都构造新 list (非破坏),性能差。如果迭代次数多,用 `nconc`——但要求每次返回的是 fresh list (不是引用常量)。`collect` 总是安全。

**累积 INTO**: 你可以让结果累积到指定变量,而不是默认的:

```elisp
(cl-loop for x in '(1 2 3 4 5)
         if (cl-oddp x) collect x into odds
         else collect x into evens
         finally return (list odds evens))
;; → ((1 3 5) (2 4))
```

`into VAR` 让累积分流——你可以在一次 loop 里维护多个 accumulator。`finally return EXPR` 在循环结束时执行,return EXPR 的值。这个例子里我们在 finally 把两个 accumulator 打包成一个 cons 返回。

### 2.4 条件子句: if/when/unless/else

```elisp
(cl-loop for x in '(1 2 3 4 5 6)
         if (cl-oddp x)
           collect x into odds
         else
           collect x into evens
         end
         finally return (list odds evens))
;; → ((1 3 5) (2 4 6))
```

`if` 后面跟一个 long clause——可以是任意 cl-loop 子句 (collect/sum/etc)。`else` 处理假分支。`end` 结束 if 块 (可选,但建议用)。

`when`/`unless` 是 `if` 的简化版本——没有 else。`unless` 是 if 的反义:

```elisp
;; 这两个等价
(cl-loop for x in '(1 2 3) when (> x 1) collect x)
(cl-loop for x in '(1 2 3) unless (<= x 1) collect x)
;; → (2 3)
```

**陷阱**: `if` 的 "then" 和 "else" 只能跟**一个**子句。如果你想多个子句,要用 `and`:

```elisp
(cl-loop for x in '(1 2 3 4)
         if (cl-oddp x)
           collect x into odds
           and count t into odd-count
         else
           collect x into evens
           and count t into even-count
         end
         finally return (list odds evens odd-count even-count))
;; → ((1 3) (2 4) 2 2)
```

`and` 在 cl-loop 里**不是**逻辑 and,是"复合子句"——它让 if 的 then 分支包含多个子句。这是 cl-loop 里最反直觉的设计之一。

### 2.5 with 子句: 局部变量

```elisp
(cl-loop for buf in (buffer-list)
         with mode-count = (make-hash-table)
         for mode = (buffer-local-value 'major-mode buf)
         do (cl-incf (gethash mode mode-count 0))
         finally return (let (result)
                          (maphash (lambda (k v) (push (cons k v) result)) mode-count)
                          result))
```

`with VAR = EXPR` 在循环**开始之前**求值 EXPR 一次,绑定 VAR 在整个循环可见。多个 with 子句顺序求值,可以引用前面的:

```elisp
(cl-loop with a = 10
         with b = (* a 2)
         for x from 1 to b
         sum x)
;; → 210 (1+2+...+20)
```

with 子句可以放任何位置,但 cl-loop 总是先求所有 with,再进循环。

### 2.6 do 子句: 执行任意副作用

```elisp
(cl-loop for file in '("a.txt" "b.txt" "c.txt")
         do (message "Processing %s" file)
         do (sleep-for 0.1))
```

`do EXPR` 在每次迭代执行 EXPR,丢弃返回值。用于 side effect——日志、修改外部状态、调用 process。

### 2.7 finally 子句: 循环结束后的清理

```elisp
(cl-loop for x in '(1 2 3 4 5)
         sum x into total
         finally (message "total: %d" total)
         finally return total)
```

`finally EXPR...` 在循环结束后执行,EXPR 是普通 Elisp (不是 loop 子句)。`finally return EXPR` 让 loop 返回 EXPR 而不是默认 accumulator。

**陷阱**: 如果循环里有 `return` 提前退出,`finally` **仍然会执行**——除非用 `cl-block`/`cl-return-from` 跳过。这是 CL 标准,但容易忘。

### 2.8 while/until/repeat

```elisp
(cl-loop repeat 5 collect (random 100))
;; 5 个随机数

(cl-loop with i = 0
         while (< i 10)
         collect i
         do (cl-incf i))
;; → (0 1 2 3 4 5 6 7 8 9)
```

`repeat N` 跑 N 次。`while COND` 跑到 COND 为 nil。`until COND` 跑到 COND 为 t。

### 2.9 真实例子: Melpa 包依赖统计

假设你想分析你的 `.emacs.d/elpa/` 下所有包的依赖深度。每个包的 `pkg.el` 文件里有 `Package-Requires` header,长这样:

```
;; Package-Requires: ((dash "2.0") (s "1.0") (cl-lib "0.5"))
```

```elisp
(defun my-package-dependencies ()
  "Return alist of (package-name . list-of-dependency-names)."
  (cl-loop for dir in (directory-files-recursively
                       (expand-file-name "elpa" user-emacs-directory)
                       "-pkg\\.el\\'")
           for pkg-file = dir
           for content = (with-temp-buffer
                           (insert-file-contents pkg-file)
                           (buffer-string))
           for matches = (with-temp-buffer
                           (insert content)
                           (goto-char (point-min))
                           (when (re-search-forward "Package-Requires:\\s-*\\(([^)]+)\\)" nil t)
                             (read (match-string 1))))
           for pkg-name = (file-name-base
                           (directory-file-name
                            (file-name-directory pkg-file)))
           when matches
           collect (cons pkg-name (mapcar #'car matches))))

(my-package-dependencies)
;; → (("dash" . ((cl-lib "0.5"))) ("s" . ((cl-lib "0.5"))) ...)
```

逐行拆解:

1. `(directory-files-recursively ... "-pkg\\.el\\'")` 找所有 `-pkg.el` 文件——Emacs 包规范规定每个包有这个文件。
2. `for pkg-file = dir`——别名,提高可读性。
3. `for content = (with-temp-buffer (insert-file-contents pkg-file) (buffer-string))`——读文件内容。用 `with-temp-buffer` 而不是 `find-file` 避免污染 buffer list。
4. `for matches = (...)`——用 regex 提取 `Package-Requires` 后面的 Lisp form,然后 `read` 解析成 list。
5. `for pkg-name = ...`——从路径推断包名。`/home/me/.emacs.d/elpa/dash-2.18.0/dash-pkg.el` → 包名 `dash-2.18.0`。简化版,不处理版本。
6. `when matches` 过滤掉没依赖的包。
7. `collect (cons ...)` 把 `(pkg . deps)` 加到结果。

整个逻辑: scan → read → parse → collect。如果不用 cl-loop,大概要 30+ 行普通 Elisp。cl-loop 让 6 行——而且每行是一个清晰的步骤。

### 2.10 真实例子: 找最长函数名

```elisp
(defun my-longest-function-name ()
  "Find the longest function name symbol in Emacs."
  (cl-loop for sym being the symbols
           when (fboundp sym)
           maximize (length (symbol-name sym))
           into max-len
           finally return
           (cl-loop for sym being the symbols
                    when (and (fboundp sym)
                              (= (length (symbol-name sym)) max-len))
                    return sym)))

(my-longest-function-name)
;; → some-long-symbol-like-widget-button-press ...
```

这里有个新东西: `for VAR being the symbols`。这是 cl-loop 的"迭代符号表"——遍历所有 intern 的 symbol。Common Lisp 风格。

第一个 loop 找最大长度 (maximize into max-len,finally return max-len)。
第二个 loop 找第一个匹配该长度的 symbol。

**陷阱**: `being the symbols` 也可以是 `being the present-symbols` (当前 obarray) 或 `being the hash-keys of HASH` 等——但有些形式 cl-lib 没实现,要查文档。

### 2.11 真实例子: 合并多个 alist

```elisp
(defun my-merge-alists (&rest alists)
  "Merge multiple alists. Later values win."
  (let ((result (make-hash-table :test #'equal)))
    (cl-loop for alist in alists
             do (cl-loop for (key . val) in alist
                         do (puthash key val result)))
    (cl-loop for k being the hash-keys of result
             using (hash-values v)
             collect (cons k v))))

(my-merge-alists '((a . 1) (b . 2)) '((b . 3) (c . 4)))
;; → ((a . 1) (b . 3) (c . 4))
```

nested cl-loop——外层遍历 alists,内层遍历每个 alist 的 entries。最后第三次 cl-loop 把 hash 转回 alist。

`using (hash-values v)` 让你同时拿 key 和 value——`for k being the hash-keys of TABLE using (hash-values v)`。这是 cl-loop 的最 fiddly 语法,但偶尔有用。

### 2.12 cl-loop 的争议

cl-loop 在 Emacs 社区是有争议的。**反对派的论点**:

**它是一种"内部 DSL"**——你是在学一个新语言,不是 Elisp。Symbolics 1980s 设计这个语法时大量参考 COBOL (英语句子风格)。这种"句子式"语法在 ML/Haskell 派看来低俗——"为什么不用 `filter`/`map`/`reduce`?"。

**它的 error message 极差**。一个简单的 typo (`cl-loop for x on xs collect car x)`——漏了 car 前面的 quote 还是括号),报错 "cl-loop: malformed clause" 不告诉你哪里错。新手调试地狱。

**它的 macroexpansion 巨大**。`cl-loop` 一行可能展开成 30 行 Elisp。byte-compile 后产生的 code size 大,性能比手写循环慢 20-30%。

**支持派的论点**:

**它表达力强**。一个 `cl-loop for x in xs when (pred x) collect (transform x))` 完美表达"filter+map",手写 `(delq nil (mapcar (lambda (x) (when (pred x) (transform x))) xs))` 拐弯抹角。

**它可读性高** (一旦学会)。看清式程序员看 cl-loop 觉得"这就是 SQL"。每个子句是一个 step。

**它是标准**。Common Lisp, Emacs Lisp 都有,大量代码用。你必须会读。

**结论**: 学会用,但不滥用。简单情况 (`mapcar`、`dolist`、`seq-filter`) 别上 cl-loop;复杂情况 (多 accumulator、条件分支、嵌套循环) 用。这是社区共识。

### 2.13 cl-loop 5 个真实场景

1. **统计 buffer 里每种 major-mode 的 buffer 数**——`buffer-list` + `do (cl-incf hash) + finally return`
2. ** Org-mode 顶层 heading 列表**——`org-map-entries` + `cl-loop collect`
3. **找未 byte-compile 的 .el** (本节开头例子)
4. **包依赖分析** (Section 2.9)
5. **dedupe list by key**——`cl-loop for x in xs unless (member (key x) seen) collect x and do (push (key x) seen)`

### 2.14 cl-loop 3 个陷阱

**陷阱 1**: 多 for 子句顺序敏感。`for a = ... then ... for b = (+ a 1)` vs `for b = ... for a = ...` 完全不同。

**陷阱 2**: `if`/`when`/`unless` 的 then 分支只能跟一个子句。要多子句用 `and`。

**陷阱 3**: 不要在 cl-loop 里 `cl-return` 不带 block 名——它的 block 名是 `nil`,你要 `(cl-return ...)` 或 `(cl-return-from nil ...)`。如果你嵌套了 `cl-block`,cl-loop 的 return 可能逃逸到外层 block,导致难以调试的行为。

---

## Part 3: cl-defun &key 关键字参数

Elisp 原生 `defun` 只支持 `&optional` 和 `&rest`:

```elisp
(defun foo (a &optional b &rest c) ...)
```

`&optional` 按位置传——调用 `(foo 1 2 3 4)` 时 b=2, c=(3 4)。这种风格在参数少时还好,多了就混乱。一个 5 个可选参数的函数,你想"只设第 5 个"得 `(foo 1 nil nil nil 5)`——丑陋且脆弱。

Common Lisp 1984 给出答案: `&key`。调用时按名字传:

```elisp
(cl-defun my-fetch (url &key (method "GET") headers timeout)
  ...)
(my-fetch "http://example.com" :method "POST" :timeout 30)
```

每个参数都有名字,顺序任意。这是 cl-lib 的核心便利之一。

### 3.1 基本 cl-defun

```elisp
(require 'cl-lib)

(cl-defun my-greet (name &key (greeting "Hello") (punct ?!))
  (format "%s, %s%c" greeting name punct))

(my-greet "World")
;; → "Hello, World!"

(my-greet "World" :greeting "Hi")
;; → "Hi, World!"

(my-greet "World" :punct ?.)
;; → "Hello, World."

(my-greet "World" :greeting "Hi" :punct ?.)
;; → "Hi, World."
```

逐 token:

- `(cl-defun my-greet (name &key (greeting "Hello") (punct ?!)) ...)`
- `name` 是必选参数 (无 `&optional`/`&key`)。
- `&key` 之后所有参数都是关键字参数。
- `(greeting "Hello")` 是一个 list——第一个 `greeting` 是参数名,第二个 `"Hello"` 是默认值。`(punct ?!)` 同理,默认是 `?!` (字符 `!`)。
- 调用时:`(my-greet "World" :greeting "Hi")`——`name = "World"`,`greeting = "Hi"`,`punct` 没传,用默认 `?!`。

**关键 token**: `:greeting`、`:punct` 这些是 Lisp 的 keyword symbol。在 Elisp 里,以 `:` 开头的 symbol 自动 "self-evaluating"——`'foo` 的值是 symbol `foo`,`':foo` 的值是 keyword `:foo` 自己。`cl-defun` 把参数名 `greeting` 转成 keyword `:greeting`。

### 3.2 完整 lambda list

```elisp
(cl-defun my-complex (a b                    ; required
                      &optional (c 10)       ; optional with default
                      &key (d 20) (e (* c d)) ; keyword, e depends on c and d
                      &rest args             ; rest = remaining keywords as plist
                      &allow-other-keys      ; accept unknown keywords
                      &aux scratch)          ; aux variables (not from caller)
  (list a b c d e args scratch))
```

逐段:

- `a b` 必选。
- `&optional (c 10)` 可选,默认 10。也可以省略默认 (`&optional c` → 默认 nil)。
- `&key (d 20) (e (* c d))` 关键字,d 默认 20,e 默认 `(* c d)`——**默认值表达式在调用时求值**,可以引用前面的参数。
- `&rest args` 收集所有未声明的 keyword 参数成一个 plist。
- `&allow-other-keys` 显式允许调用者传未声明的 keyword (默认 cl-defun 会报错)。
- `&aux scratch` 局部变量,等价 `let`——不接收调用者值,默认 nil。

调用示例:

```elisp
(my-complex 1 2)
;; → (1 2 10 20 200 nil nil)
;; e = (* c d) = (* 10 20) = 200

(my-complex 1 2 5 :d 100)
;; → (1 2 5 100 500 nil nil)
;; c=5, d=100, e=(* 5 100)=500

(my-complex 1 2 5 :d 100 :extra 'x)
;; → error! unknown keyword :extra (除非 &allow-other-keys)
```

### 3.3 keyword 参数名 vs keyword symbol

注意: cl-defun 的 `&key (greeting "Hello")` 里,`greeting` 是参数变量名,默认调用用 `:greeting`。这是 cl-lib 的约定: 参数名 `foo` 对应 keyword `:foo`。

但你可以显式指定 keyword:

```elisp
(cl-defun my-foo (&key ((:hi greeting) "Hello"))
  greeting)

(my-foo :hi "World")
;; → "World"
```

这个 `((:hi greeting) "Hello")` 语法: keyword 是 `:hi`,变量名是 `greeting`,默认 `"Hello"`。这种"自定义 keyword 名"很少用,但在接口兼容 (改 keyword 名不破坏代码) 时有用。

### 3.4 真实例子: my-org-task

Org-mode 的 capture 模板是个复杂结构——`("k" "Todo" entry (file+headline "" "Tasks") "* TODO %?")`。这种结构很难记,因为每个元素是位置参数。我们用 cl-defun 包一层让它易用:

```elisp
(cl-defun my-org-task (key description
                           &key (type 'entry)
                           (target 'file)
                           target-file
                           (headline "Tasks")
                           (template "* TODO %?")
                           (immediate-finish nil))
  "Generate an Org capture template."
  (let ((real-target
         (pcase target
           ('file `(file ,target-file))
           ('file+headline `(file+headline ,target-file ,headline))
           ('clock `(clock)))))
    (list key description type real-target
          template
          (when immediate-finish '(:immediate-finish t)))))

(my-org-task "t" "Todo"
             :target 'file+headline
             :target-file "~/org/tasks.org"
             :headline "Reals")
;; → ("t" "Todo" entry (file+headline "~/org/tasks.org" "Reals") "* TODO %?" nil)
```

这个 wrapper 把 Org capture 模板的 6 个位置参数变成 5 个有名字的 keyword,带默认值,易扩展。Org 自己不强制这种风格——但任何写"工具函数集"的人都会做类似封装。

### 3.5 cl-defun 5 个真实场景

1. **HTTP client wrapper**——`(cl-defun my-fetch (url &key method headers timeout body) ...)` 让 curl-like API 友好。
2. **Org capture helper** (上面的例子)。
3. **Logger**——`(cl-defun log (msg &key (level 'info) (time (current-time))) ...)`。
4. **Tree traversal**——`(cl-defun walk-tree (tree &key on-node (test #'identity) (recursion 'dfs)) ...)`。
5. **String formatter**——`(cl-defun fmt (template &key (sep " ") (trim t)) ...)`——任选项多,keyword 必备。

### 3.6 cl-defun 3 个陷阱

**陷阱 1**: 默认值表达式**在调用时**求值,不是函数定义时。这意味着 `(cl-defun f (&key (now (current-time))) ...)` 每次调用都重新算 now。如果你要"定义时求值",用 `&aux` 配合静态值。

**陷阱 2**: keyword 参数**必须**用 keyword symbol (`:foo`),不是普通 symbol (`foo`)。`(my-func 'foo 1)` 不会工作,要 `(my-func :foo 1)`。新手常搞错。

**陷阱 3**: `&allow-other-keys` 默认**不**启用,所以传未声明的 keyword 会报错。这通常是好事 (尽早暴露 bug),但有时你想"接受任何 keyword 然后转发给另一个函数",这时要加 `&allow-other-keys`。

### 3.7 cl-defun 与 cl-function

cl-defun 定义的函数在内部叫 "cl-function"——它**不是**普通 Elisp function。它接收的不是位置参数,而是一个"lambda list",由 cl-lib 的宏在调用前 parse 成 keyword/value。

byte-compile 后,cl-defun 函数其实是一个普通函数——宏已经在编译时把 keyword 解析逻辑内联进去了。所以**运行时没有性能损失**——cl-defun 的开销在编译时,不在调用时。

但有一个**重要后果**: cl-function 不能用 `apply` 直接调用,如果你传了 keyword 但想用 apply。`(apply #'my-fetch '("url" :method "POST"))` 在某些 Emacs 版本可能出问题——因为 apply 把所有参数当位置参数。cl-lib 提供 `cl-function` 来包装,但实际上现代 Emacs (24.4+) 的 apply 处理 cl-function 没问题。

---

## Part 4: cl-letf 临时替换函数

`cl-letf` 是 cl-lib 里最"魔法"的成员——它能在一段时间内**临时改变**任何 place 的值,然后自动恢复。最常见用途是 mock 函数——把 `current-time`、`message`、`network-call` 替换成 stub,测试完恢复。

### 4.1 基本形式

```elisp
(cl-letf (((symbol-function 'current-time) (lambda () (list 0 0 0)))
          ((symbol-function 'message) (lambda (&rest _) nil)))
  ;; 在这个 scope 内:
  ;; (current-time) 返回 (0 0 0)
  ;; (message ...) 不输出
  (my-time-stamped-function))
;; scope 退出后,两个函数自动恢复
```

逐 token:

- `cl-letf` 是 macro,不是 function。
- 第一个参数是 bindings 列表,每个 binding 是 `(PLACE VALUE)`。
- `((symbol-function 'current-time) (lambda () (list 0 0 0)))`——PLACE 是 `(symbol-function 'current-time)`,VALUE 是 lambda。cl-letf 把 `'current-time` 的 function slot 替换成这个 lambda。
- 第二个 binding 同理替换 `message`。
- body 是 `my-time-stamped-function`,它内部调用 `current-time` 和 `message`——拿到的是 stub 版本。
- scope 退出后,cl-letf 把所有 place 恢复原值。

`symbol-function` 是 subr,返回 symbol 的 function slot——`(symbol-function 'car)` 返回 `#<subr car>`。cl-letf 用 `(setf (symbol-function ...) ...)` 临时改它。

### 4.2 PLACE 是什么

cl-letf 的关键是 PLACE——任何能用 `setf` 的地方都是 PLACE:

- `(symbol-function 'foo)`——函数 slot
- `(symbol-value 'foo)`——变量值
- `(buffer-name)`——当前 buffer 名
- `(point)`——point
- `(current-buffer)`——当前 buffer
- `(oref obj slot)`——EIEIO slot
- `(gethash key table)`——hash value
- `(gv-ref place)`——generalized variable

cl-letf 把"setf + unwind-protect"打包——所以即使 body 抛错,值也会恢复。这是 cl-letf 比"手动 set + restore"安全的核心。

### 4.3 cl-letf 与 unwind-protect

cl-letf 内部展开大致:

```elisp
;; (cl-letf (((symbol-function 'foo) my-stub)) body...)
(let ((--old-- (symbol-function 'foo)))
  (unwind-protect
      (progn
        (fset 'foo my-stub)
        body...)
    (fset 'foo --old--)))
```

`unwind-protect` 是 Elisp 的 try/finally——body 抛错或正常退出,finally 都执行,恢复旧值。这是 cl-letf "出错也安全"的底层机制。

### 4.4 真实例子: ERT 测试 mock time

ERT (Emacs Lisp Regression Test) 是 Emacs 内置测试框架。当你测一个"打印时间戳"的函数:

```elisp
(defun my-format-timestamp ()
  (format-time-string "[%Y-%m-%d]" (current-time)))

(ert-deftest my-format-timestamp-test ()
  (cl-letf (((symbol-function 'current-time) (lambda () '(23731 35616 0 0))))
    ;; (current-time) 现在返回 2025-01-15 12:00:00 UTC
    (should (equal (my-format-timestamp) "[2025-01-15]"))))
```

这里 `(23731 35616 0 0)` 是 Emacs 时间戳格式——`(high low usec pid)`。23731 * 65536 + 35616 ≈ 1736900000,是 2025-01-15。

为什么这样 mock? 因为不 mock,`current-time` 永远是"现在",测试结果不稳定。mock 让测试确定。

### 4.5 真实例子: 捕获 message

测试一个函数的 message 输出:

```elisp
(defun my-warn-large-buffer ()
  (when (> (buffer-size) 1000000)
    (message "Warning: buffer has %d bytes" (buffer-size))))

(ert-deftest my-warn-large-buffer-test ()
  (cl-letf (((symbol-function 'message)
             (lambda (fmt &rest args)
               (setq my-captured-message (apply #'format fmt args)))))
    (let ((my-captured-message nil))
      ;; 模拟大 buffer
      (with-temp-buffer
        (insert (make-string 1000001 ?x))
        (my-warn-large-buffer))
      (should (string-match-p "Warning" my-captured-message)))))
```

把 `message` 替换成"存到变量"——捕获输出便于断言。

### 4.6 cl-letf 5 个真实场景

1. **Mock 当前时间**——测时间相关函数。
2. **Capture message** (上面例子)。
3. **Mock 网络**——把 `url-retrieve-synchronously` 替换成返回 fixture 数据。
4. **临时改 default-directory**——测文件操作。
5. **临时禁用 hooks**——`(letf ((my-hook nil)) ...)` 测函数而不触发副作用。

### 4.7 cl-letf 3 个陷阱

**陷阱 1**: cl-letf 恢复**原值**——如果你在 body 里又改了原值,cl-letf 恢复的是 body 进入前的 snapshot。这通常没问题,但在嵌套 cl-letf 或 multi-threaded 代码可能不直观。

**陷阱 2**: `(symbol-function 'foo)` 改的是**全局** function slot。多线程 (Emacs 26+ 有 thread,但 GIL) 下其他线程也能看到改动。这通常没关系,但写并发代码时要注意。

**陷阱 3**: 如果原值是 nil (symbol 没绑定 function),cl-letf 恢复成 nil,而不是"void"。后续 `(symbol-function 'foo)` 返回 nil 而不是 void-function 错误。这种细微差别在反射代码里会咬人。

### 4.8 cl-letf 与 advice

advice (Emacs 25.1+ 的 `advice-add`/`advice-remove`) 是另一种"改函数行为"的方式。区别:

- `advice-add` 永久添加 (除非 remove),`cl-letf` 临时。
- `advice` 是"在函数前后包装",`cl-letf` 是"完全替换"。
- `advice` 可以多个共存 (多个 :before 函数链式调用),`cl-letf` 是 winner-takes-all。

测试用 cl-letf (短作用域,自动清理),生产代码用 advice (持久,可叠加)。

---

## Part 5: cl-labels 递归本地函数

Elisp 的 `let` 不能定义递归函数。为什么? 看:

```elisp
(let ((helper (lambda (x) (if (zerop x) 0 (+ x (helper (1- x)))))))
  (funcall helper 5))
;; → void-function helper
```

为什么 void-function? 因为 `let` 在**右值求值时**绑定变量。`(lambda (x) ...)` 创建 closure 时,`helper` 这个名字**还没绑定**——`let` 还没完成。所以 lambda 里调 `(helper ...)` 找不到 binding。

这是 Lisp-2 的特殊坑。Lisp-1 (Scheme) 里 `(letrec ((f (lambda ...))) ...)` 因为变量和函数同 namespace,lambda 体里直接引用 f 名字就能找到。Elisp 是 Lisp-2,函数和变量分开——`(helper ...)` 在 function 位置找 helper 的 function slot,而 `let` 改的是 value slot,所以找不到。

解决方案有几个:

```elisp
;; 方案 1: 用 defun 全局
(defun my-helper (x) (if (zerop x) 0 (+ x (my-helper (1- x)))))
(my-helper 5)
;; 问题: 污染全局 namespace

;; 方案 2: 用 funcall + 自传
(let ((helper nil))
  (setq helper (lambda (self x) (if (zerop x) 0 (+ x (funcall self self (1- x))))))
  (funcall helper helper 5))
;; 工作,但难看

;; 方案 3: cl-labels
(cl-labels ((helper (x) (if (zerop x) 0 (+ x (helper (1- x))))))
  (helper 5))
;; → 15 (0+1+2+3+4+5)
```

cl-labels 在 lexical binding 下创建递归本地函数。它把函数名**同时**绑定到 value slot 和 (通过 `flet` 机制) function slot。

### 5.1 cl-labels vs cl-flet

cl-lib 有两个相关宏:

- `cl-labels`——递归本地函数,函数体里**可以**调自己 (用名字引用)。
- `cl-flet`——非递归本地函数,函数体里**不能**调自己 (调自己会找全局/外层同名函数)。

```elisp
(cl-flet ((helper (x) (if (zerop x) 0 (+ x (helper (1- x))))))
  (helper 5))
;; helper 调 helper——找全局 helper,可能 void-function!
```

为什么有两个? Common Lisp 历史: `flet` 是 dynamic (函数体里名字查找是动态的——查全局),`labels` 是 lexical (查本地 binding)。在 Emacs 23 之前 (dynamic binding 时代),cl-flet 是动态查找——所以不能递归。cl-labels 用 lexical 名字查找——能递归。

Emacs 24+ lexical-binding 默认后,cl-flet 的"动态查找"行为更不直观——它仍然查全局,而不是 lexically scoped。所以**实践中 cl-labels 几乎总是你想要的**。cl-flet 主要用于"临时覆盖某个全局函数 (类似 advice)"的极少数场景。

### 5.2 cl-labels 实战: 树遍历

```elisp
(defun my-count-nodes (tree)
  "Count leaf nodes in a nested list."
  (cl-labels ((rec (node)
                 (cond
                  ((null node) 0)
                  ((atom node) 1)
                  (t (+ (rec (car node)) (rec (cdr node)))))))
    (rec tree)))

(my-count-nodes '(1 (2 3) (4 (5 6) 7)))
;; → 7 (atoms: 1, 2, 3, 4, 5, 6, 7)
```

`cl-labels` 定义的 `rec` 可以递归——每次 `(rec ...)` 在 body 内查找,找到本地的 rec binding。

注意 `(cl-labels ((rec (node) ...)) (rec tree))`——第一个 `rec` 是名字,`(node)` 是参数列表,body 是 cond。**格式**: `(cl-labels ((NAME ARGLIST BODY...) ...) BODY...)`。

### 5.3 cl-labels 5 个真实场景

1. **递归目录遍历**——`(cl-labels ((walk (dir) ...)) (walk root))`,处理 symlink 等复杂情况。
2. **AST 遍历**——Org/Emacs Lisp 解析后,递归访问每个节点。
3. **Memoized fibonacci**——配合闭包变量记录已计算值。
4. **二分搜索**——递归切分数组。
5. **Reduce-style 累积**——`rec` 每次处理一个元素,带 accumulator。

### 5.4 cl-labels 3 个陷阱

**陷阱 1**: 必须 `lexical-binding: t`。`cl-labels` 在 dynamic binding 下行为不正确——它依赖 lexical capture。

**陷阱 2**: cl-labels 的函数名**只在 body 内可见**,在 cl-labels 之外不可见。这是设计——避免污染。但如果你想"导出"局部函数,要 `defun` 全局。

**陷阱 3**: cl-labels 的函数**捕获**外层 lexical binding——这意味着如果你在 cl-labels 外有 `let` 变量,函数体可以访问。这是 feature 不是 bug,但要小心 closure 持有 reference 导致 GC 不能回收。

---

## Part 6: pcase 模式匹配

pcase 是这一节最重要的主题。**它是现代 Elisp 区分老手和 writes-modern-Elisp 的关键技能**。Stefan Monnier 2010 年设计 pcase (灵感来自 ML/Haskell 的 pattern matching),2011 年随 Emacs 24.1 发布。

为什么 pcase 这么重要? 因为它**统一**了 cond/destructuring/type-dispatch,让代码量减半且意图明确。Org-mode 的 AST 处理 (`org-element.el`) 几乎全是 pcase——一旦学会,你看 org-element 的代码从"天书"变成"清晰"。

### 6.1 基本 pcase

```elisp
(pcase x
  ('foo "it's foo")
  ((pred numberp) "number")
  (`(,a ,b) (format "two: %s %s" a b))
  (`(,a . ,rest) (format "head: %s, tail-length: %d" a (length rest)))
  (_ "default"))
```

`pcase` 接受一个 EXPRESSION 和多个 CLAUSE——每个 CLAUSE 是 `(PATTERN BODY...)`。pcase 求值 EXPRESSION,然后**按顺序**匹配每个 PATTERN。第一个匹配的 PATTERN 的 BODY 被执行,返回值是 pcase 的值。`_` 是 catch-all,匹配任何东西。

每个 PATTERN 形式有不同语义,我逐一讲。

### 6.2 Pattern 1: 字面值

```elisp
(pcase x
  ('foo "it's foo symbol")
  ("bar" "it's bar string")
  (5 "it's number 5")
  (?a "it's char a"))
```

字面值匹配: 如果 `x` `equal` 字面值,匹配成功。注意 `'foo`——单引号是必须的,因为不带 quote 的 `foo` 会被解释成"变量绑定 pattern" (后面讲)。

**陷阱**: `5` 和 `?a` 不需要 quote (它们是 self-evaluating),但 `'foo` 需要。规则: symbol 必须带 quote,其他类型不用。

### 6.3 Pattern 2: 变量绑定 `_` 和 SYMBOL

```elisp
(pcase x
  (a (format "captured: %S" a)))
```

不带 quote 的 symbol 是**变量 pattern**——它匹配任何值,且**绑定**该值到 symbol。所以上面例子,无论 x 是什么,匹配成功,a 绑定 x 的值,body 用 a。

`_` 是特殊变量——它**不**绑定 (名字是下划线,pcase 视为"通配")。

**陷阱**: 这意味着**所有不带 quote 的 symbol 都是 catch-all**。`(pcase x (foo "matched"))` 不会匹配 x 等于 foo symbol——它把 x 绑定到名为 `foo` 的新变量! 这是 pcase 最容易搞错的地方。要匹配 foo symbol,用 `'foo`。

### 6.4 Pattern 3: (pred FN)

```elisp
(pcase x
  ((pred numberp) "it's a number")
  ((pred stringp) "it's a string")
  ((pred (> 5)) "greater than 5"))    ; curried
```

`(pred FN)` 匹配 FN 返回非 nil 时。`FN` 可以是函数名或 lambda。

`(pred (> 5))` 是 curried——`> 5` 是 partial application,返回一个函数,该函数接受 x 返回 `(> 5 x)`。即 x < 5 时匹配。这是 pcase 的"内嵌条件"机制。

### 6.5 Pattern 4: 反引号模板 (backquote)

这是 pcase 最强大的特性。**反引号 pattern** 模仿 `quote`/backquote 的结构,允许"模式 + 解构"。

```elisp
(pcase x
  (`(,a ,b) (format "two-element list: %S %S" a b))
  (`(,a . ,rest) (format "list head: %S, tail: %S" a rest))
  (`(1 ,second 3) (format "list [1, %S, 3]" second)))
```

- `` `(,a ,b) `` 匹配两元素 list,绑定 a 到第一,b 到第二。`,a` 是"unquote pattern"——任何值都匹配,绑定到 a。
- `` `(,a . ,rest) `` 匹配任意 cons,绑定 a 到 car,rest 到 cdr。
- `` `(1 ,second 3) `` 匹配**精确**三元素 list,第一是 1,第三是 3,中间任意 (绑定 second)。

这看起来像 backquote syntax——因为 pcase 重用了 reader 语法。`` ` `` 后面的 form 是 template,`,` 标记 pattern 变量。除 `,variable` 之外的部分都是字面值,必须 match。

```elisp
(pcase '(1 "hello" 3)
  (`(1 ,second 3) (format "second is %S" second)))
;; → "second is \"hello\""
```

**这是 pcase 的核心机制**: 一个表达式 + 一个模板,在模板里用 `,variable` 标"这里抓值"。任何不在 unquote 里的部分都是字面值,必须 match。

### 6.6 Pattern 5: (and PATTERNS...) 和 (or PATTERNS...)

```elisp
(pcase x
  ((and (pred numberp) n) (format "number %d" n))      ; n 绑定 x
  ((or 'foo 'bar) "foo or bar"))
```

`(and P1 P2 ...)` 所有 pattern 都要匹配。`(or P1 P2 ...)` 任一匹配。

`and` 常用于"先 type-check 再绑定": `((and (pred numberp) n) ...)` 检查是 number,然后绑定 n。两个 pattern 共享同一个 EXPRESSION——所以 n 是 x 的值。

`or` 的问题: 不同分支的 binding **不互通**。如果 P1 绑定 a,但 P2 匹配成功,a 不会绑定。所以 or 各分支应独立。

### 6.7 Pattern 6: (guard COND)

```elisp
(pcase x
  ((and n (guard (> n 10))) "big number")
  ((and n (guard (<= n 10))) "small number"))
```

`(guard COND)` 是"运行时条件"——它匹配时如果 COND 求值非 nil,匹配成功;否则失败。COND 可以引用前面 pattern 绑定的变量 (上面 n)。

`and` + `guard` 组合是 pcase 表达复杂条件的方式。

### 6.8 Pattern 7: (app FN PATTERN)

```elisp
(pcase x
  ((app length (pred (> 3))) "list with more than 3 elements")
  ((app car 'header) "cons whose car is 'header"))
```

`(app FN PATTERN)` 把 FN 应用到 EXPRESSION,然后用 PATTERN 匹配结果。`(app length ...)`——取 x 的 length,然后匹配。`(app car 'header)`——取 x 的 car,匹配字面值 header。

`app` 用于"转换后匹配"——常用于结构投影。

### 6.9 真实例子: HTTP 请求解析

```elisp
(defun my-parse-request (req)
  "Parse an HTTP request list into a normalized form.
REQ is like (METHOD PATH ...) or (METHOD PATH BODY ...)."
  (pcase req
    (`(GET ,path . ,headers)
     (list :get path nil headers))
    (`(POST ,path ,body . ,headers)
     (list :post path body headers))
    (`(PUT ,path ,body . ,headers)
     (list :put path body headers))
    (`(DELETE ,path . ,headers)
     (list :delete path nil headers))
    (`(,method ,path . ,_)
     (list :other method path))
    (_
     (error "Malformed request: %S" req))))

(my-parse-request '(GET "/index.html" ("Host" . "example.com")))
;; → (:get "/index.html" nil (("Host" . "example.com")))

(my-parse-request '(POST "/submit" "{\"name\":\"a\"}" ("Content-Type" . "json")))
;; → (:post "/submit" "{\"name\":\"a\"}" (("Content-Type" . "json")))

(my-parse-request '(PATCH "/x"))
;; → (:other PATCH "/x")
```

每个 case 用 backquote pattern 匹配特定 method + path + optional body + headers。这是 pcase 的典型用法——"数据是 list,前几个元素决定结构,后面是 metadata"。

逐 case 拆:

- ``(GET ,path . ,headers)``: list 第一元素是 `GET` symbol,然后 path (任意值,绑 path),然后 `. ,headers` 匹配剩余作为 headers。
- ``(POST ,path ,body . ,headers)``: list 前三是 POST/path/body,剩余 headers。注意 GET 没 body 但 POST 有——pcase 让你精确表达每种 method 的"形状"。
- ``(,method ,path . ,_)``: fallthrough——任意两元素以上 list,method/path 绑定,rest 用 `_` 忽略。
- `_`: 全 catch,报错。

如果不用 pcase,代码会是:

```elisp
(defun my-parse-request (req)
  (cond
   ((and (consp req) (eq (car req) 'GET))
    (list :get (cadr req) nil (cddr req)))
   ((and (consp req) (eq (car req) 'POST))
    (list :post (cadr req) (caddr req) (cdddr req)))
   ...))
```

`car`/`cadr`/`cddr`/`caddr`/`cdddr` 满天飞,可读性差,且容易写错 (cdddr 是 cdr-cdr-cdr-cdr 还是 cdr-cdr-cdr?)。pcase 的 backquote 让结构直接可见。

### 6.10 pcase-let: 多 pattern 解构

```elisp
(pcase-let ((`(,method ,path ,body) req))
  (process method path body))
```

`pcase-let` 像 `let`,但 binding 是 `(PATTERN EXPR)`——EXPR 求值后用 PATTERN 解构。如果 PATTERN 不匹配,抛错。

用于"我知道这数据是这形状,直接抓字段":

```elisp
(pcase-let ((`(,status ,body . ,rest) (my-http-call url)))
  (when (>= status 400)
    (error "HTTP error: %s" body))
  body)
```

### 6.11 pcase-lambda

```elisp
(funcall
 (pcase-lambda (&rest args)
   (pcase args
     (`(,a) (format "one: %S" a))
     (`(,a ,b) (format "two: %S %S" a b))
     (_ "many")))
 1 2)
;; → "two: 1 2"
```

`pcase-lambda` 是 lambda,但它的参数列表是 pattern——根据参数数量/形状不同分支。这在写 overload-style 函数时有用。

实际上 pcase-lambda 不常用——多数情况 cl-defun `&optional`/`&rest` 就够。但偶尔 (像 Clojure 多 arity 函数),pcase-lambda 让意图清晰。

### 6.12 进阶: 复杂树结构

Org-mode 用 pcase 处理 AST。Org element 长这样:

```elisp
(heading (:level 1 :title "Hello")
         (paragraph nil "Some text"))
```

匹配:

```elisp
(pcase element
  (`(heading ,props . ,children)
   (let ((level (plist-get props :level))
         (title (plist-get props :title)))
     (format "Heading L%d: %s (%d children)" level title (length children))))
  (`(paragraph ,props ,text)
   (format "Paragraph: %S" text))
  (`(src-block ,props ,code)
   (format "Code: %S" code)))
```

每种 element 类型有不同的"形状"——heading 有 children (list),paragraph 有 text (string)。pcase 让每种情况独立匹配,清晰分离。

### 6.13 pcase vs cond

什么时候用 pcase vs cond?

**cond** 简单——按顺序求值条件,第一个非 nil 的 win。条件可以是任意表达式。

**pcase** 强大——模式匹配 + 绑定。当你的逻辑是"看数据形状",用 pcase;当你的逻辑是"几个独立条件",用 cond。

```elisp
;; cond 适合
(cond
 ((> x 10) "big")
 ((> x 0) "small")
 (t "negative"))

;; pcase 适合
(pcase x
  (`(,a ,b) "two-element")
  ((pred numberp) "number")
  ('foo "symbol foo"))
```

不要为了用 pcase 而用——简单条件用 cond 更清晰。但凡是"解构 + 检查"的场景,pcase 一定更好。

### 6.14 pcase 5 个真实场景

1. **HTTP request 分发** (本节开头例子)。
2. **AST 遍历** (Org/Emacs Lisp 解析树)。
3. **Error 处理**——`(pcase err ((error . ,msg) ...) ((user-error . ,msg) ...))`。
4. **命令行参数解析**——`(pcase args (("--file" ,f) ...) (("--help") ...))`。
5. **JSON 数据处理**——`(pcase json ((map '("type" . "event")) ...))` (用 pcase 配 `map-elt`)。

### 6.15 pcase 3 个陷阱

**陷阱 1**: 不带 quote 的 symbol 是**绑定 pattern**,不是字面值匹配。`(pcase x (foo ...))` 永远匹配成功,绑定 x 到 foo——不管 x 是什么!要匹配 foo symbol,用 `'foo`。

**陷阱 2**: backquote pattern 必须用 backquote,不是 quote。`(pcase x ((a b) ...))` 是**两个 binding pattern** (a 和 b),不匹配 list!要匹配 list 用 `` `(,a ,b) ``。

**陷阱 3**: pcase 是**按顺序**匹配——第一个 match 的 win。这影响性能: 复杂 pattern (尤其 `pred` 调函数) 放前面浪费,放后面可能匹配不到。原则: 廉价 pattern 在前,具体 case 在前,catch-all (`_`) 最后。

### 6.16 pcase 历史: Stefan Monnier 的设计

Stefan Monnier 是蒙特利尔大学教授,长期 Emacs 维护者。他写了 pcase (`pcase.el`),最早 2010 年提交到 Emacs 仓库,2011 年随 Emacs 24.1 发布。

设计灵感: ML (SML/OCaml) 的 pattern matching,Haskell 的 case。但 Elisp 的 pcase 比 ML 的更松——ML 的 pattern 在编译时类型检查,pcase 是运行时匹配。

为什么 pcase 出现在 2010 而不是更早? 因为 2008 年 Emacs 23 加了 `lexical-binding`,让闭包/模式匹配有意义。pcase 依赖 lexical capture,在 dynamic binding 下行为古怪。

社区接受 pcase 慢——前几年很多包仍然用 `cond`/`car`/`cdr`。但随着 Org 9.x、lsp-mode、magit 新版本大量用 pcase,2020 年后 pcase 成为主流。现在 (2026),不会 pcase 你读不懂大部分现代 Elisp 包。

---

## Part 7: EIEIO 对象系统

EIEIO = "Enhanced Implementation of Emacs Interpreted Objects"。读作 "I-O-I-O" 或 "E-I-E-I-O" (像 Old MacDonald 那首歌)。1995 年 Eric Ludlam 写,2020 年左右 Stefan Monnier 重写整理。

EIEIO 是 Emacs 的 CLOS (Common Lisp Object System) 子集实现——支持 class、继承、method (generic + method)、slot、initarg。它用于 Emacs 内置的 speedbar、cedet (Collection of Emacs Development Environment Tools)、semantic (代码分析层),以及第三方包 org-roam、lsp-mode、company 等。

### 7.1 历史: Eric Ludlam 和 cedet

Eric Ludlam 是 1990s 波士顿地区 Emacs hacker,在 MathWorks 工作。他写了 cedet (一个 Emacs IDE 工具集),需要对象系统。Elisp 当时没有 OOP,所以他写 EIEIO。

为什么不用 cl-struct? 因为 cl-struct 是 POD (plain old data)——没有 method dispatch。Eric 想要 "数据 + 行为" 绑定——即 method 调用根据对象类型分发。这需要 CLOS 风格的多分派。

EIEIO 设计基于 CLOS (1988 ANSI 标准)。CLOS 有几个核心概念:
- **generic function**: 一个抽象函数,有多个 method。
- **method**: 实现 generic 的一个版本,针对特定参数类型。
- **multiple dispatch**: 调用哪个 method 取决于多个参数的类型 (不止 `this`)。
- **class**: 数据结构 + 继承。
- **slot**: class 的字段。

CLOS 是 Lisp-1 的"反叛"——它把 method 从 class 上移到 generic function 上。Java/C++ 是 `obj.method(args)`,CLOS 是 `(method obj args)`——method 不是 class 的"附属品",是独立函数。这让多分派自然——`(method a b)` 的具体实现取决于 a 和 b 的类型,而不是只有 a。

EIEIO 实现了 CLOS 的核心子集——单分派 (eql specializer)、继承、slot、initarg。多分派在 EIEIO 1.0 没实现,后期加入但不完整。

### 7.2 基础 defclass

```elisp
(require 'eieio)

(defclass my-animal ()
  ((name
    :initarg :name
    :initform ""
    :type string
    :documentation "Animal name.")
   (sound
    :initarg :sound
    :initform "???"
    :documentation "Sound the animal makes.")))

(defmethod my-speak ((a my-animal))
  (format "%s says %s" (oref a name) (oref a sound)))

(let ((dog (my-animal :name "Rex" :sound "Woof")))
  (my-speak dog))
;; → "Rex says Woof"
```

逐 token:

- `(require 'eieio)`——加载 EIEIO (实际上 Emacs 25+ 自动预加载,但显式 require 是好习惯)。
- `(defclass my-animal () (...))`——定义 class my-animal,父类列表空 (默认继承 `eieio-default-superclass`,所有 class 的根)。
- 第一个 slot `name`:
  - `:initarg :name`——用户创建对象时用 `:name "Rex"` 设值。
  - `:initform ""`——默认值 (如果用户不传 `:name`)。
  - `:type string`——声明类型 (EIEIO 在 init 时检查,如果传入非 string 报错)。
  - `:documentation "..."`——文档 (用 `describe-class` 查看)。
- `defmethod my-speak ((a my-animal)) ...`——定义 method。`(a my-animal)` 是 specializer——这个 method 在 `a` 是 my-animal 时调用。
- `(oref a name)`——读 slot。`oref` = Object REFerence。`(oset a name "new")` 写。
- `(my-animal :name "Rex" :sound "Woof")`——构造对象。class name 即 constructor,接受 initarg plist。

### 7.3 slot 选项详解

```elisp
(defclass my-example ()
  ((field1
    :initarg :field1            ; 用户构造时用 :field1 设值
    :initform "default"          ; 默认值 (表达式在构造时求值)
    :type string                 ; 类型检查
    :documentation "Field 1 doc"
    :accessor my-example-field1  ; 自动生成 accessor (getter/setter)
    :reader my-example-get-field1 ; 只生成 getter
    :writer my-example-set-field1 ; 只生成 setter
    :allocation :instance        ; :instance (默认) 或 :class
    :custom string               ; customize 接口
    :protection :private)        ; 访问控制 (但 EIEIO 不强制)
   (counter
    :initform 0
    :allocation :class           ; class 变量 (static),所有实例共享
    :documentation "Class-level counter.")))
```

关键选项:

- `:initarg`、`:initform`、`:type`、`:documentation` 已经讲过。
- `:accessor NAME` 自动生成 `(NAME obj)` 和 `(setf (NAME obj) val)`——名字是 class_name-slot_name 默认,但你可以覆盖。
- `:reader`/`:writer` 分别只生成 getter/setter。
- `:allocation :instance` 是默认——每个对象有自己的 slot。`:allocation :class` 是 static——所有对象共享一个 slot (类变量)。
- `:custom` 让 slot 在 `M-x customize` 里可编辑。
- `:protection` 是 hint,EIEIO 不强制——`:public`/`:protected`/`:private` 只影响 `describe-class` 显示。

### 7.4 oref/oset 和 with-slots

```elisp
(let ((dog (my-animal :name "Rex" :sound "Woof")))
  (oref dog name)               ; → "Rex"
  (oref default-obj 'unbound)   ; → error if slot unbound
  (oref-default dog 'name)      ; class default value

  (oset dog name "NewName")     ; set slot
  (oset-default my-animal 'sound "...") ; set class default
  )
```

`oref` 读 instance slot,`oref-default` 读 class 默认值。`oset` 写 instance,`oset-default` 写 class。

频繁 oref 啰嗦——用 `with-slots`:

```elisp
(defmethod my-summary ((a my-animal))
  (with-slots (name sound) a
    (format "%s says %s" name sound)))
```

`with-slots` 把指定 slot 绑定为本地变量——body 里 `name` 直接用 (实际是 `(oref a name)` 的语法糖)。

### 7.5 defmethod 和多分派

```elisp
(defclass my-cat ()
  ((lives :initarg :lives :initform 9)))

(defclass my-dog ()
  ((breed :initarg :breed :initform "Unknown")))

(defmethod my-interact ((a my-cat) (b my-dog))
  (format "Cat hisses at %s dog" (oref b breed)))

(defmethod my-interact ((a my-dog) (b my-cat))
  (format "Dog chases cat with %d lives" (oref b lives)))

(my-interact (my-cat) (my-dog :breed "Poodle"))
;; → "Cat hisses at Poodle dog"

(my-interact (my-dog :breed "Lab") (my-cat :lives 3))
;; → "Dog chases cat with 3 lives"
```

EIEIO 支持多分派——`my-interact` 的 method 选择取决于 a **和** b 的类型。第一个调用 a 是 cat,b 是 dog,选第一个 method。第二个 a 是 dog,b 是 cat,选第二个。

这是 CLOS 风格——比 Java/C++ 的单分派灵活。

但注意: EIEIO 实际只支持 eql specializer (参数必须是该 class 的 instance),不像 CLOS 完整支持 class specializer (参数可以是 subclass)。日常够用。

### 7.6 继承

```elisp
(defclass my-pet (my-animal)
  ((owner
    :initarg :owner
    :initform "Unknown"
    :documentation "Pet owner."))
  "A pet is an animal with an owner.")

(defclass my-dog2 (my-pet)
  ((breed
    :initarg :breed
    :initform "Mixed"))
  "A dog is a specific kind of pet.")
```

`my-pet` 继承 `my-animal`,`my-dog2` 继承 `my-pet`。继承链: my-dog2 → my-pet → my-animal → eieio-default-superclass。

子类继承所有父类 slot。`my-dog2` 实例有 name、sound、owner、breed 四个 slot。

```elisp
(let ((d (my-dog2 :name "Buddy" :sound "Woof" :owner "Alice" :breed "Lab")))
  (oref d name)    ; → "Buddy" (继承自 my-animal)
  (oref d owner)   ; → "Alice" (my-pet)
  (oref d breed))  ; → "Lab" (my-dog2)
```

`defmethod` 在继承层次中也工作:

```elisp
(defmethod my-speak ((d my-dog2))
  (format "%s (a %s) barks" (oref d name) (oref d breed)))

(my-speak (my-dog2 :name "Buddy" :breed "Lab"))
;; → "Buddy (a Lab) barks"
```

method dispatch 找最具体的 method——my-dog2 比 my-animal 具体,选前者。如果子类没定义 method,自动用父类的:

```elisp
(my-speak (my-pet :name "Mittens"))  ; 用 my-animal 的 my-speak (因为 my-pet 没定义)
```

### 7.7 真实例子: 项目管理系统

```elisp
(require 'eieio)
(require 'cl-lib)

(defclass my-project ()
  ((name :initarg :name :type string)
   (tasks :initarg :tasks :initform nil :documentation "List of my-task objects.")
   (deadline :initarg :deadline :initform nil))
  "A project contains a list of tasks.")

(defclass my-task ()
  ((title :initarg :title :type string)
   (done :initarg :done :initform nil :type boolean)
   (assignee :initarg :assignee :initform nil)
   (priority :initarg :priority :initform 0 :type integer))
  "A task belongs to a project.")

(defclass my-software-project (my-project)
  ((language :initarg :language :initform "Unknown")
   (repo-url :initarg :repo-url :initform nil))
  "Software project has code-specific metadata.")

;; Methods

(defmethod my-add-task ((proj my-project) (task my-task))
  "Add TASK to PROJ."
  (object-add-to-list proj 'tasks task)
  proj)

(defmethod my-remove-task ((proj my-project) (task my-task))
  "Remove TASK from PROJ."
  (object-remove-from-list proj 'tasks task)
  proj)

(defmethod my-todo-count ((proj my-project))
  "Number of incomplete tasks."
  (cl-count-if-not (lambda (tk) (oref tk done)) (oref proj tasks)))

(defmethod my-done-count ((proj my-project))
  "Number of completed tasks."
  (cl-count-if (lambda (tk) (oref tk done)) (oref proj tasks)))

(defmethod my-summary ((proj my-project))
  "Human-readable summary."
  (format "Project %s: %d/%d tasks done"
          (oref proj name)
          (my-done-count proj)
          (+ (my-done-count proj) (my-todo-count proj))))

(defmethod my-overdue-p ((proj my-project))
  "Non-nil if deadline passed and tasks remain."
  (and (oref proj deadline)
       (my-todo-count proj)
       (time-less-p (oref proj deadline) (current-time))))

;; 软件项目特有方法
(defmethod my-bug-ratio ((proj my-software-project))
  "Fake metric: ratio of high-priority to total tasks."
  (let ((bugs (cl-count-if (lambda (tk) (>= (oref tk priority) 8))
                           (oref proj tasks))))
    (/ (float bugs) (max 1 (length (oref proj tasks))))))

;; 使用
(let ((proj (my-software-project
             :name "Emacs Plugin"
             :language "Elisp"
             :repo-url "git@github.com:me/plugin.git")))
  (my-add-task proj (my-task :title "Write docs" :priority 5))
  (my-add-task proj (my-task :title "Fix bug #42" :priority 9))
  (my-add-task proj (my-task :title "Release v1" :done t))
  (my-add-task proj (my-task :title "Refactor core" :priority 8))

  (list (my-summary proj)
        (format "Bug ratio: %.2f" (my-bug-ratio proj))
        (format "Language: %s" (oref proj language))))
;; → ("Project Emacs Plugin: 1/4 tasks done" "Bug ratio: 0.50" "Language: Elisp")
```

这个例子展示 EIEIO 的完整使用:

- `my-project`、`my-task` 基础 class。
- `my-software-project` 继承 `my-project`,加 software 特有 slot。
- `my-add-task`、`my-todo-count`、`my-summary` 等 method 对 `my-project` 操作。
- `my-bug-ratio` 只对 `my-software-project` 工作——子类特有 method。
- 主代码构造软件项目,加任务,调各种 method——所有 dispatch 自动。

逐行讲关键 token:

- `(defclass my-project () ((name :initarg :name :type string) ...))`——name slot 必须是 string (EIEIO 检查类型)。
- `tasks` slot 默认 nil (空 list),用 `object-add-to-list` 加元素 (这是 EIEIO 提供的工具,自动处理 list)。
- `my-add-task` 方法:`object-add-to-list proj 'tasks task`——把 task 加到 proj 的 tasks slot。注意 `'tasks` 是 slot 名 (symbol),不是变量。
- `my-todo-count` 用 cl-count-if-not——cl-lib 和 EIEIO 经常一起用。
- `my-overdue-p` 检查 deadline 过期——`time-less-p` 是 Emacs 时间比较。

`my-software-project` 继承意味着所有 `my-project` 的 method 都可用,加自己的 `my-bug-ratio`。继承让代码组织自然——通用 method 在父类,特化 method 在子类。

### 7.8 EIEIO 与 cl-defstruct

cl-lib 也提供 `cl-defstruct`——比 EIEIO 简单:

```elisp
(cl-defstruct my-point-struct (x 0) (y 0))

(let ((p (make-my-point-struct :x 1 :y 2)))
  (my-point-struct-x p))   ; → 1
```

`cl-defstruct` 是 POD——只有 slot accessor,没有 method dispatch。简单数据结构用 cl-defstruct,行为复杂用 EIEIO。

### 7.9 EIEIO 5 个真实场景

1. **Org-roam 数据库**——`org-roam-db` 用 EIEIO 表示连接、缓存。
2. **lsp-mode**——LSP server 抽象、workspace、language client 都用 EIEIO class。
3. **Company/Capf**——completion backend 用 EIEIO。
4. **speedbar/cedet**——Eric Ludlam 自己的代码,大量 EIEIO。
5. **项目管理** (本节例子)——任何需要"实体 + 行为"的应用。

### 7.10 EIEIO 3 个陷阱

**陷阱 1**: EIEIO 慢。method dispatch 比 Emacs Lisp 函数调用慢几倍——因为 dispatch 要查 method table。对于调用频率 < 10000 次/秒的代码没问题;hot loop (像字符渲染) 千万别用。`org-roam`、`lsp-mode` 这些"业务逻辑"用没问题;`font-lock`、`redisplay` 这些用不得。

**陷阱 2**: 调试困难。EIEIO 对象打印为 `#<my-class 0x...>`——你 describe 它要 `(describe-object obj)` 或 `M-x describe-class`。普通 `print` 看不到 slot。这和普通 Elisp 数据 (list/plist) 调试体验差很多——后者用 `message "%S"` 一目了然。

**陷阱 3**: `:type` 检查只在构造时,不持续。`(my-animal :name "Rex")` 构造时检查 name 是 string,但 `(oset a name 42)` 不检查——你可以之后塞错类型进去。要严格的 type safety,你要写 `:before` method 或 cl-defmethod 的 :extra 处理,这又增加复杂度。

**陷阱 (额外)**: EIEIO + dynamic binding 怪行为。EIEIO 严重依赖 lexical-binding——`defclass`/`defmethod` 在 dynamic binding 下某些 corner case (尤其 slot 默认值的闭包) 不工作。**永远** `lexical-binding: t` 用 EIEIO。

### 7.11 何时不用 EIEIO

不是所有问题都需要 OOP。**数据简单用 plist/alist**:

```elisp
;; 不要这样
(defclass my-person ((name ...) (age ...)))
(let ((p (my-person :name "Alice" :age 30)))
  (oref p name))

;; 这样更好
(let ((p '(:name "Alice" :age 30)))
  (plist-get p :name))
```

plist 优点: 打印易读、和 JSON 互通好、序列化简单、Elisp 所有 API 支持。

EIEIO 优点: method dispatch、继承、type check、customize 接口。

选择标准:
- 数据是"被传递" (parsing, API response) → plist
- 数据是"被操作" (CRUD、业务逻辑) → EIEIO
- 数据有"行为" (多种类型,各自不同响应) → EIEIO
- 数据是临时 (临时变量、accumulator) → 普通变量/list

Emacs 内核大量用 plist (overlay、text-property、frame-parameter) 因为这些是"被读取/传递"的。EIEIO 用于"业务对象"。

---

## Part 8: 总结——三者的关系

回过头看 cl-lib、pcase、EIEIO 这三者的关系。它们看起来是独立工具,但放在一起构成现代 Elisp 的"大型程序范式"。

**cl-lib** 提供语言级便利——keyword arguments、loop、destructuring、临时替换。这些是**底层工具**,几乎所有代码都受益。

**pcase** 提供**数据访问**抽象——让你不写 `(car (cdr x))` 而写 `` `(,a ,b . ,rest) ``。它和 cl-lib 配合:`(cl-defun f (&rest args) (pcase args ...))` 是常见模式。

**EIEIO** 提供**结构+行为**抽象——数据有"类型",每种类型有自己的"行为"。这适合复杂领域模型。但 EIEIO 不应该到处用——它最适合"中型应用核心",而不是 Elisp 工具脚本。

### 8.1 Emacs 社区的偏好

Emacs 社区 (相对 Python/Ruby) 偏好**简洁**。一份 200 行的包,可能完全不用 EIEIO,只用 cl-lib 基础 (cl-defun、cl-loop) 和 pcase。这是 vertico/marginalia 这一代包的哲学——"用最少语言特性解决问题"。

但**大型包** (magit、org、lsp-mode、org-roam) 必然用到 EIEIO——领域复杂必须建模。

新手学习路径建议:

1. **先用 cl-lib 基础** (cl-defun, cl-loop, cl-remove-if)——立刻提升代码质量。
2. **学 pcase**——一周内能读懂 Org/magit 源码。
3. **慎用 EIEIO**——直到你写"实体多、关系复杂"的应用才上。早期用 alist/plist 就够。
4. **cl-letf/cl-labels**——按需学,主要在测试和复杂递归。

### 8.2 设计哲学总结

这三者背后的 Emacs 设计哲学是**渐进增强**——核心 Elisp 极简,大特性靠 library。这和 Python ("batteries included")、Ruby ("程序员幸福") 不同——Emacs 选择"小核心 + 丰富 library"。这适合 Emacs 的角色: 编辑器,不是应用平台。

但这也意味着**你,程序员,要选择**——什么用 cl-lib,什么用 pcase,什么用 EIEIO,什么用纯 Elisp。没有"正确"答案,取决于你的代码量、性能需求、维护者偏好。这种"自由"是 Emacs 的味——既迷人又累人。

### 8.3 进一步阅读

- Emacs Lisp Reference: `(info "(elisp)CL Arguments")`、`(info "(elisp)Pattern matching case)")`、`(info "(eieio)Top")`
- Stefan Monnier 的 pcase 论文/邮件列表讨论:-gnu-emacs-dev archive ~2010
- Eric Ludlam 的 cedet tutorial: cedet.sourceforge.net
- Common Lisp the Language (CLtL2): Chapter 7 (objects), Chapter 26 (loop)——EIEIO 和 cl-loop 都参考这里
- 实际代码: org-roam/lsp-mode 的 `*-eieio.el` 文件,看真实项目如何用

### 8.4 最后

这一份文件读完不会让你立刻熟练——这三个工具都需要写真实代码、调试、迭代。但读完之后你看 magit/org/lsp-mode 源码不再"懵",知道每个 form 在做什么、为什么这么写。

下次你在 Elisp 代码里看到 `(cl-loop for x in xs when (pred x) collect (transform x))`,你知道这不只是"魔法"——它是一个 1980s Symbolics 设计的、Stefan Monnier 2012 年清理的、能精确表达 filter+map 意图的工具。你知道它的代价 (字节码大、调试难),也知道它的价值 (可读性)。

这就是"深度"——知道 why, 不只是知道 how。

写代码。
