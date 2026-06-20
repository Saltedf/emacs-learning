# Drill 08: Code Navigation — 在代码结构里飞

> **预计用时**: 2.5 小时
> **难度**: ★★★★☆ (sexp 移动是 Emacs 最重要的肌肉记忆之一)
> **替代**: `emacs-manual/programs.texi` 全文 (核心部分)
> **目标**: 5 秒内跳到任意函数定义;10 秒内列出所有定义;不靠鼠标浏览任何代码

---

## 0. 学习目标

- [ ] 用 `C-M-f` / `C-M-b` 在 sexp 之间飞
- [ ] 用 `C-M-u` / `C-M-d` 上下穿越括号层次
- [ ] 用 `C-M-a` / `C-M-e` 跳到函数头尾
- [ ] 用 `C-M-n` / `C-M-p` 在同级括号之间跳
- [ ] 用 `M-;` 注释/取消注释 region 或行
- [ ] 用 `imenu` / `consult-imenu` 跳到任意定义
- [ ] 用 `which-function-mode` 在 mode line 看当前函数
- [ ] 用 `M-.` / `M-?` / `M-,` 完成一次定义-引用往返
- [ ] 用 `hs-minor-mode` 折叠代码块

---

## 1. 第一性原理: 为什么代码 ≠ 普通文本

### 1.1 普通文本是"字符流",代码是"树"

Module 1 的前 7 个 drill 教你处理**字符流**: 移动、搜索、选区、矩形、宏、窗口、help。这些工具对任何文本都有效——README、邮件、小说、log 文件。

但代码不一样。代码不是字符流,代码是**结构化的语法树**。

```
普通文本:     "今天天气不错,适合出去走走。"
              ↑ 字符 ←→ 字符 ←→ 字符 ...

代码:          (defun factorial (n)
                 (if (<= n 1)
                     1
                   (* n (factorial (- n 1)))))
              ↑ 是嵌套的树,括号是树的边
```

一段代码同时有**三层结构**:

1. **字符层**: `d`, `e`, `f`, `u`, `n` (前 7 drill 处理的)
2. **token 层**: `defun`, `factorial`, `n`, `if` (word 移动处理)
3. **语法层**: `(defun ...)`, `(if ...)`, `(* n ...)`, `(factorial (- n 1))` (**本 drill 处理**)

第 3 层是代码独有的。它定义了"哪个括号匹配哪个括号"、"哪个表达式包含哪个表达式"、"哪个 `defun` 是顶级、哪个是嵌套"。理解第 3 层,你就能像读语法树一样读代码——不是逐字读,而是**按结构跳**。

### 1.2 S-expression 是 Emacs 代码操作的原子单位

Emacs 把代码的第 3 层结构抽象为一个统一概念: **S-expression** (sexp)。

一个 sexp 是:
- 一个 token (word、number、string、symbol)
- 或者一对配对的括号 + 括号里的所有东西

```
factorial             ← 一个 sexp (symbol)
"hello"               ← 一个 sexp (string)
(if (<= n 1) 1 (* n (factorial (- n 1))))
   ↑                   ← 整个 if 表达式是一个 sexp
   (<= n 1)            ← 嵌套的 sexp
   (* n (factorial (- n 1)))
   ↑                   ← 另一个嵌套 sexp
```

Emacs 的代码操作全部建立在 sexp 上。`C-M-f` 跳到下一个 sexp,`C-M-k` 杀一个 sexp,`C-M-SPC` 选一个 sexp——所有这些命令共享同一个底层模型: **找当前的语法边界**。

**与 Vim/VSC 的区别**:
- VS Code 默认的"jump to definition" 靠 LSP,没有 LSP 就不行
- Vim 的 `%` 跳匹配括号,但不区分"哪个括号是 sexp 边界"
- Emacs 的 sexp 命令**不需要任何 LSP**,纯靠 syntax table 算。开箱即用,对任何 mode 都有效

这是 Emacs 1985 年就有的设计。Stallman 当时的洞察是: **大多数代码编辑操作本质上是"在一个嵌套结构里导航"**,如果编辑器能识别嵌套边界,90% 的代码操作能秒级完成。

### 1.3 三种代码导航模型

Emacs 提供三种代码导航,各有适用场景:

| 模型 | 代表命令 | 速度 | 准确度 |
|---|---|---|---|
| **Sexp 移动** (语法层) | `C-M-f`/`C-M-b`/`C-M-u` | 极快 | 高 (受 syntax table 限制) |
| **Imenu / which-function** (定义层) | `M-x imenu`/`which-function-mode` | 快 | 中 (受 mode indexer 限制) |
| **xref** (语义层) | `M-.`/`M-?`/`M-,` | 慢 (需要 backend) | 极高 (LSP/tag 精确) |

理解这三层是关键。日常编辑用 sexp 移动,跳定义用 imenu,跨文件重构用 xref——选错工具会浪费时间。本 drill 训练你**在合适场景选合适工具**。

---

## 2. Sexp 移动命令对照

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-M-f` | `forward-sexp` | 跳到下一个 sexp 末尾 |
| `C-M-b` | `backward-sexp` | 跳到上一个 sexp 开头 |
| `C-M-u` | `backward-up-list` | 跳到当前 sexp 的外层开括号 |
| `C-M-d` | `down-list` | 进入下一个嵌套括号 |
| `C-M-a` | `beginning-of-defun` | 跳到当前 defun 开头 |
| `C-M-e` | `end-of-defun` | 跳到当前 defun 末尾 |
| `C-M-n` | `forward-list` | 跳到下一个同级闭括号 (跨整个 list) |
| `C-M-p` | `backward-list` | 跳到上一个同级开括号 |

**记忆法** (Chassell 的 mnemonic):
- `f`/`b` = **f**orward / **b**ackward (和 `C-f`/`C-b` 一致,只是加 `M-`)
- `u`/`d` = **u**p / **d**own (在嵌套树上上下)
- `a`/`e` = **a**head / **e**nd (和 `C-a`/`C-e` 一致,只是 defun 级别)
- `n`/`p` = **n**ext / **p**revious (跨整个 list)

注意所有这些键都是 `C-M-` 前缀。**`C-M-` 是 Emacs 用来标记"代码级"操作的标准前缀**,就像 `C-x` 是"buffer/window 级"、`C-c` 是"mode 级"。

---

## 3. 内联讲解: Sexp 移动

### 3.1 forward-sexp (`C-M-f`)

`C-M-f` 把光标移到下一个 sexp 的**末尾**。这听起来抽象,看例子:

```elisp
(defun factorial (n)
  (if (<= n 1)
      1
    (* n (factorial (- n 1)))))
```

光标在 `defun` 的 `d` 上。按 `C-M-f`:

```
位置: |(defun factorial (n) ...)
      ↑ 起点

按 C-M-f:
      (defun| factorial (n) ...)
            ↑ 跳过 "defun" sexp,停在它末尾的空格之前
```

再按 `C-M-f`:

```
      (defun factorial| (n) ...)
                     ↑ 跳过 "factorial" sexp
```

再按 `C-M-f`:

```
      (defun factorial (n)| ...)
                         ↑ 跳过 "(n)" 这个 sexp
```

再按 `C-M-f`:

```
      (defun factorial (n) (if (<= n 1) 1 (* n (factorial (- n 1))))|)
                                                                    ↑ 跳过整个 (if ...) sexp
```

最后一个 `C-M-f` 一次跳过了整个 `if` 表达式——包括它内部所有的嵌套。这就是 sexp 移动的威力: **无视内部嵌套,按"语法边界"跳**。

如果你用普通 word 移动 (`M-f`) 跳这个表达式,要按 20+ 次。用 sexp,4 次到位。在大代码文件里,这种差距是数量级的。

**陷阱 1**: sexp 移动依赖 syntax table。不同 major mode 的 syntax table 不一样。在 Python mode 里,`C-M-f` 不会按 sexp 跳 (Python 没有括号 sexp),它用 indentation 作为边界。在 C/Java mode 里,sexp 是 `{...}` 块和 `(...)`。这意味着 `C-M-f` 在不同语言里行为不一样——理解你的 mode 的语法规则才能正确用 sexp。

**陷阱 2**: sexp 移动在**括号不匹配**时会卡住。如果某处有 unbalanced 括号,`C-M-f` 会报错 "Scan error: containing expression ends prematurely"。这是好事——它告诉你代码有语法错误。但如果你不知道这一点,会以为 Emacs 出 bug 了。

### 3.2 backward-sexp (`C-M-b`)

`C-M-b` 是 `C-M-f` 的镜像,跳到上一个 sexp 的**开头**:

```elisp
(defun factorial (n) (if (<= n 1) 1 (* n (factorial (- n 1)))))
                                              ↑ 光标在这里
```

按 `C-M-b`:

```
                                              (- n 1)
                                            ↑ 跳到 "(- n 1)" 开头
```

再按 `C-M-b`:

```
                                              (factorial (- n 1))
                                       ↑ 跳到 "(factorial ...)" 开头
```

`C-M-b` 一次跳一个 sexp,无视嵌套深度。

**对比**: `M-b` (backward-word) 一次跳一个 word。`(factorial (- n 1))` 用 `M-b` 跳要 5 次 (`factorial`、`(`、`-`、`n`、`1`)。用 `C-M-b` 一次。**代码场景永远优先用 `C-M-b`**,除非你确实要 word 级精度。

### 3.3 backward-up-list (`C-M-u`) — 穿越嵌套

`C-M-u` 是 sexp 移动里最重要的一个: **从当前 sexp 跳到它的外层开括号**。让你"向上"穿越嵌套层次。

```elisp
(defun factorial (n)
  (if (<= n 1)
      1
    (* n (factorial |- n 1)))))   ; 光标在 - 上
```

按 `C-M-u`:

```
    (* n (factorial |- n 1)))
        ↑ 跳到 "(factorial ...)" 的开括号
```

再按 `C-M-u`:

```
  (* n (factorial (- n 1)))
  ↑ 跳到 "(* ...)" 的开括号
```

再按 `C-M-u`:

```
(if (<= n 1) 1 (* n (factorial (- n 1))))
↑ 跳到 "(if ...)" 的开括号
```

再按 `C-M-u`:

```
(defun factorial (n) ...)
↑ 跳到 "(defun ...)" 的开括号
```

`C-M-u` 让你**像爬树一样往上爬**,每按一次上一级。这在深嵌套代码里救命——你不需要精确知道光标在哪一层,只要按几下 `C-M-u`,你就在最外层。

**实战场景**: 你在改一个 6 层嵌套的 JSON 处理代码,改完想看看这个表达式的最外层 context 是什么。按 `C-M-u` 几下,立刻跳到最外层 `let` 或 `if`。这比 page-up 或 `C-a` (去行首) 高效得多——前者带结构,后者瞎移。

**陷阱**: `C-M-u` 总是跳到**开**括号,不是闭括号。如果你想"跳到当前 list 的末尾",要用 `C-M-f` (forward-sexp) 在 list 末尾按,或者先 `C-M-u` 到开头,再 `C-M-f` 跳过整个 list。

### 3.4 down-list (`C-M-d`) — 进入嵌套

`C-M-d` 是 `C-M-u` 的反向: 从外层进入**下一个嵌套括号**。

```elisp
(defun factorial (n)
  (if (<= n 1)
      1
    (* n (factorial (- n 1)))))
↑ 光标在 "(defun ..." 的 `(` 上
```

按 `C-M-d`:

```
(defun factorial (n) ...)
       ↑ 进入 "(n)" 这个嵌套
```

按 `C-M-d`:

```
(if (<= n 1) 1 (* n (factorial (- n 1))))
    ↑ 进入 "(<= n 1)" 这个嵌套
```

`C-M-d` 让你**往下钻**。配合 `C-M-u`,你可以在嵌套树里上下穿梭——这是 Emacs 处理深嵌套代码的核心肌肉记忆。

**陷阱**: `C-M-d` 要求光标在**某个 sexp 内部或之前**。如果光标后面没有嵌套括号 (比如在 `(a b c)` 的 `a` 之后),`C-M-d` 会报 "No following paren". 这个命令需要预见性——它跳到下一个开括号,如果看不到就报错。

### 3.5 beginning-of-defun / end-of-defun (`C-M-a` / `C-M-e`)

`defun` 是 Emacs 对"顶层定义"的术语——通常是函数定义,在 Lisp 里就是 `(defun ...)`。

`C-M-a` 跳到**当前 defun 的开头** (左括号),`C-M-e` 跳到**末尾** (右括号)。

```elisp
(defun foo ()
  (bar)
  (baz))            ; 光标在这

(defun qux ()
  ...)
```

按 `C-M-a`:

```
(defun foo ()        ; 跳到当前 defun (foo) 的开头
  ...
```

按 `C-M-e`:

```
  (baz))             ; 跳到当前 defun 的末尾
```

按 `C-u 2 C-M-a`:

```
(defun qux ()        ; 跳到上面第 2 个 defun 的开头
```

数字前缀让 `C-M-a`/`C-M-e` 跳 N 个 defun。这在浏览代码时极有用: `C-u 5 C-M-e` 跳到下 5 个函数末尾。

**对比 isearch 跳 `defun`**: 你可以用 `C-s defun RET` 跳所有 defun 关键字,但这跳不到非 Lisp 语言的函数定义。在 Python mode 里,`C-M-a` 跳到上一个 `def` 顶部;在 C mode 里,跳到上一个函数签名。**`C-M-a` 是 mode-aware 的**,它知道当前语言的"defun"是什么。

### 3.6 forward-list / backward-list (`C-M-n` / `C-M-p`)

`C-M-n` 跳到下一个**同级闭括号**,`C-M-p` 跳到上一个**同级开括号**。这不是 sexp 移动,是**括号移动**——它跳整对括号,不跳括号内的内容。

```elisp
(let ((a 1) (b 2) (c 3))
  (foo)
  (bar)
  (baz))
↑ 光标在 "(let ...)" 的 "(" 上
```

按 `C-M-n`:

```
  (foo)
  (bar)
  (baz))         ; 跳到匹配的 ")"
```

按 `C-M-n` (假设下面还有别的 list):

跳到下一个 list 末尾。

`C-M-n`/`C-M-p` 适合**在同级 list 之间跳**,比如跳过一系列函数调用。配合 `C-M-f`/`C-M-b` (sexp 跳),你能精确控制移动粒度。

**记忆**: sexp 命令 (`C-M-f`/`C-M-b`) 跳的是"语法单位",list 命令 (`C-M-n`/`C-M-p`) 跳的是"括号对"。两者粒度不同,组合使用最灵活。

---

## 4. Sexp 操作命令

移动是基础,操作是目的。以下命令对 sexp 做操作:

### 4.1 kill-sexp (`C-M-k`)

`C-M-k` 杀掉**光标后的一个 sexp**,把它放进 kill ring。

```elisp
(defun foo (bar baz) ...)
          ↑ 光标在 bar 之前
```

按 `C-M-k`:

```elisp
(defun foo ( baz) ...)
          ↑ "bar" 被杀,可以 C-y 召回
```

如果光标在括号上:

```elisp
(defun foo (bar baz) ...)
           ↑ 光标在 "(bar baz)" 的 "(" 上
```

按 `C-M-k`:

```elisp
(defun foo )
       ↑ 整个 "(bar baz)" 被杀
```

`C-M-k` 是**结构化删除**——它删除一个语法完整的单元,不会留下半截括号。比 `C-k` (删到行尾) 安全得多。

**实战场景**: 你想删一个函数参数。光标移到参数名前,`C-M-k`,完成。如果你用 `M-d` (kill-word),只能删一个 word,删不干净 `:type`、`->int` 这些复合 token。`C-M-k` 一次到位。

### 4.2 mark-sexp (`C-M-@` 或 `C-M-SPC`)

`C-M-@` (或等价的 `C-M-SPC`) 把 region 设到**当前 sexp**:

```elisp
(foo bar baz)
     ↑ 光标在 bar 之前
```

按 `C-M-SPC`:

```
(foo [bar] baz)
     ↑ 选中 bar (region 高亮)
```

按 `C-u C-M-SPC` 选中更多 sexp (向前扩展)。

**对比 mark-word (`M-@`)**: `M-@` 选一个 word,`C-M-SPC` 选一个 sexp。代码场景永远优先 `C-M-SPC`,除非你要选的就是 word。

**创造性用法**: `C-M-SPC` 配合 `M-w` 复制 sexp,或者 `C-M-k` 替代——先 `C-M-SPC` 选中,看清楚再 `M-w` (复制) 或 `C-w` (剪切)。先看再删比直接 kill 安全。

### 4.3 transpose-sexps (`C-M-t`)

`C-M-t` 交换**前后两个 sexp**:

```elisp
(foo bar)
 ↑ 光标在 foo 前
```

按 `C-M-t`:

```elisp
(bar foo)
```

**实战场景**: 重命名时把两个参数顺序对调。比如 `(let ((a 1) (b 2)) ...)` 想改成 `(let ((b 2) (a 1)) ...)`,把光标放到 `(a 1)` 前,按 `C-M-t` 一次。

`C-M-t` 比 `C-t` (transpose-chars) 粒度大,比手动剪切粘贴快。

### 4.4 regex query-replace (`C-M-%`)

`C-M-%` (注意: 不是 `M-%` 普通替换) 是**正则表达式替换**:

```
C-M-% \(defun\)[ ]+\([a-z]+\)
       ↓
       defun \2
```

这个例子把 `(defun foo` 替换为 `defun foo` (移除开括号)。`\(...\)` 是捕获组,`\1`/`\2` 引用。

正则在 Module 3 Drill 03 已经覆盖,这里不重复。但**代码场景**的正则替换需要特别小心:
- 大多数时候应该用 `M-%` 普通替换,正则容易过度复杂
- 正则替换可能改变括号配对,导致 sexp 移动失效
- **永远在替换前 `C-x C-s` 保存**,这样替换错了能 `C-/` 回退

---

## 5. 括号匹配辅助

### 5.1 show-paren-mode

`(show-paren-mode 1)` 在 init.el 里开后,光标在括号上时**高亮匹配的另一半**:

```
(defun foo (bar baz))
          ↑ 光标
          ↓ 匹配高亮
                  ↑ 高亮 )
```

这是 Module 0 就该开的 minor mode。如果你没开,现在开:

```elisp
(show-paren-mode 1)
```

**为什么必须开**: 没有 show-paren-mode,你不知道哪个 `(` 对应哪个 `)`。手动数括号是反人类的,Show-paren-mode 让你视觉上立刻看到。Emacs 30+ 默认开。

### 5.2 electric-pair-mode

`(electric-pair-mode 1)` 让你输入 `(` 时**自动插入配对的 `)`**,光标在中间:

```
按 (
结果: ()
      ↑ 光标
```

这是"自动补全括号"模式,大多数现代编辑器都有。Emacs 的版本可以高度定制 (electric-pair-pairs、electric-pair-skip-whitespace)。

**陷阱**: electric-pair-mode 经常和 lisp-mode 或 third-party 包 (paredit) 冲突。如果你装了 paredit/smartparens,**不要**开 electric-pair-mode——会双重补全。

### 5.3 paredit / smartparens 简介

`paredit` 是 Lisp 社区的"括号守护神"。它强制你**永远不能破坏括号配对**——你不能删单个 `(`,只能删完整的 sexp。

```
你按:  M-d
paredit 拒绝: "Sigh."
你按:  C-M-k (kill-sexp)
paredit 允许: 删一个完整 sexp
```

paredit 让 Lisp 代码永远结构正确,代价是**学习曲线陡峭**。新手经常被"为啥我删不掉这个字符"搞疯。

`smartparens` 是 paredit 的通用版,支持非 Lisp 语言 (Python、JavaScript、HTML 等)。更复杂但更通用。

**建议**: Module 1 不要装 paredit/smartparens。先用原生的 sexp 命令建立基础肌肉记忆。Module 4 配置阶段再决定要不要装。

### 5.4 括号匹配跳转

| 键 | 命令 | 作用 |
|---|---|---|
| `C-M-n` | forward-list | 跳到匹配的闭括号 |
| `C-M-p` | backward-list | 跳到匹配的开括号 |
| `M-x check-parens` | | 检查 buffer 里 unbalanced 括号 |

`M-x check-parens` 是检查代码语法的工具。如果你 `C-M-f` 报 "scan error",运行 `M-x check-parens` 找到出问题的位置。

---

## 6. Comment Commands

### 6.1 comment-dwim (`M-;`)

`M-;` (comment-dwim, DWIM = Do What I Mean) 是 Emacs 注释命令的瑞士军刀:

| 情况 | 行为 |
|---|---|
| 当前行无注释 | 在行尾加注释 (用 mode 的 comment character) |
| 当前行有注释 | 把注释延伸到下一行 |
| region 激活 | 注释整个 region (或取消,如果已注释) |
| 空行 | 在行首加注释 |

例子 (Lisp):

```elisp
(foo bar)
```

光标在行尾按 `M-;`:

```elisp
(foo bar) ;; 
```

(光标在注释空格里)

例子 (region):

```elisp
(foo)
(bar)
(baz)
```

选中三行,按 `M-;`:

```elisp
;; (foo)
;; (bar)
;; (baz)
```

再按 `M-;` 取消注释。

`M-;` 是**最常用的注释命令**,因为它根据上下文做不同的事。比 comment-region / uncomment-region 更智能——你不需要决定"加还是删",它自己看。

### 6.2 comment-region / uncomment-region

如果你想强制注释或取消注释 (不走 dwim 逻辑):

| 命令 | 键位 | 作用 |
|---|---|---|
| `comment-region` | `C-c C-c` (有些 mode) 或 `M-x comment-region` | 强制注释 region |
| `uncomment-region` | `C-u M-;` 或 `M-x uncomment-region` | 强制取消注释 |
| `comment-line` | `M-x comment-line` | 注释/取消当前行 |
| `comment-or-uncomment-region` | `M-;` (在 region 激活时) | 等同于 dwim |
| `comment-kill` | `M-x comment-kill` | 杀掉注释 |

**`M-;` 已经够用 90% 场景**。只在需要精确控制时才用 comment-region / uncomment-region。

### 6.3 comment-line

`M-x comment-line` (无默认键) 切换当前行注释状态。和 `M-;` 的区别:
- `M-;` 在无 region 时把注释加到**行尾**
- `comment-line` 把注释加到**行首**

如果你想要"toggle 当前行的注释",绑一个键到 `comment-line`:

```elisp
(global-set-key (kbd "M-:") 'comment-line)
```

(注意 `M-:` 默认是 eval-expression,绑之前考虑别的键)

### 6.4 注释的创造性用法

1. **临时禁用代码**: 选中代码块,`M-;` 注释掉,做实验,`M-;` 恢复
2. **大段文档**: 在 Lisp 里 `;; ...` 多行可以写文档,选 region `M-;` 一键加
3. **代码 review**: 在行尾加 `;; FIXME` 标记,用 occur (`M-s o FIXME RET`) 全部列出
4. **多语言混合**: 在 markdown 里嵌 Python 代码,`M-;` 会根据当前 mode 选对注释符
5. **`comment-box`**: `M-x comment-box` 把 region 包在一个"注释框"里 (用 `#` 或 `;` 围成矩形)

---

## 7. Imenu: 跳到任意定义

### 7.1 imenu 的本质

imenu 是 Emacs 的"跳定义"机制。它扫描当前 buffer,建立一个**定义索引** (函数名、类名、变量名等),然后让你选一个跳过去。

```
[扫描 buffer] ──> [建立索引]
                  function foo: line 10
                  function bar: line 25
                  class Baz: line 50
                ↓
[用户选] ──> [跳到 line]
```

imenu 不需要任何外部工具 (不需要 ctags、不需要 LSP),纯靠 mode 的 indexer。每个 major mode 自己实现 `imenu-generic-expression` (一个正则),imenu 用它扫 buffer。

**与 LSP 的区别**:
- LSP 跨文件、语义精确、需要 backend (slow startup)
- imenu 只在当前 buffer、靠正则 (可能漏)、零依赖

日常代码导航,**imenu 比 LSP 快 10 倍**,因为不需要 IPC 通信。LSP 留给跨文件场景。

### 7.2 调用 imenu

最基本调用:

```
M-x imenu RET
```

minibuffer 提示 "Index item:",你可以输入函数名,补全 (按 TAB):

```
M-x imenu RET
Index item: fact[TAB]
             ↳ completes to "factorial"
RET → 跳到 factorial 定义
```

imenu 也支持** residency**: 第二次调 `M-x imenu`,它会记住上次的位置,可以快速跳回。

### 7.3 consult-imenu (`M-g i`)

如果你装了 `consult` 包 (Module 4 推荐),`consult-imenu` 提供更好的 UI——它用 vertico/ivy 风格的 live filter,让你**输入即过滤**:

```
M-g i
[Type to filter definitions]
def[space]
  factorial
  defun helper
  defvar *counter*
```

`consult-imenu` 比 vanilla `imenu` 快得多,因为它的补全是 live filter (输入即过滤),vanilla imenu 是一次性补全。

**建议**: 装 consult 是 Module 4 的事。Module 1 先用 vanilla `imenu`,理解机制后再升级。

### 7.4 imenu-generic-expression

mode 怎么知道哪些是"定义"? 通过 `imenu-generic-expression`——一个正则的列表。

例子 (Emacs Lisp):

```elisp
(setq imenu-generic-expression
      `(("Defuns" "^(defun[ \t]+\\(\\(?:\\sw\\|\\s_\\)+\\)" 1)
        ("Defmacros" "^(defmacro[ \t]+\\(\\(?:\\sw\\|\\s_\\)+\\)" 1)
        ("Defvars" "^(defvar[ \t]+\\(\\(?:\\sw\\|\\s_\\)+\\)" 1)))
```

每个元素是 `(category regexp group)`:
- `"Defuns"` 是分类标签
- 正则匹配 `(defun name`,捕获组 1 是 name
- `1` 是 group 编号

自定义这个变量可以扩展 imenu。比如你想让 imenu 列出所有 `TODO` 注释:

```elisp
(add-to-list 'imenu-generic-expression
             '("TODOs" "^[ \t]*;+ *TODO: \\(.*\\)" 1))
```

重启 mode 后 `M-x imenu` 就有 "TODOs" 分类。

### 7.5 imenu 的陷阱

**陷阱 1: 大文件索引慢**。imenu 扫整个 buffer,10000 行的文件可能要几秒。如果你在巨型 buffer 反复 `M-x imenu`,会卡。

**解决**:
```elisp
(setq imenu-auto-rescan t)        ; 自动重扫
(setq imenu-auto-rescan-maxout 600000) ; 文件超过 600KB 不自动扫
```

**陷阱 2: mode 没实现 imenu indexer**。某些小众 mode 可能没写 `imenu-generic-expression`,`M-x imenu` 报 "No items"。这时候要么自己写,要么用 isearch + `C-s def RET`。

**陷阱 3: 索引可能漏**。imenu 靠正则匹配,如果代码风格特殊 (比如用宏定义函数),正则可能漏掉。imenu 是"近似",LSP 才是"精确"。

---

## 8. which-function-mode

`which-function-mode` 是 imenu 的副产品: 它在 **mode line 显示当前光标所在的函数名**。

```elisp
(which-function-mode 1)
```

开了之后,mode line 多一块:

```
-UU-:**--F1  test.el   Top   (factorial)           All L1     (Elisp)
                                     ↑ which-function 显示
```

光标在 `factorial` 函数体内时,mode line 显示 `(factorial)`。移到 `helper` 函数,显示 `(helper)`。

**为什么有用**:
- 长函数里你滚到中间,不知道自己在哪个函数
- mode line 实时告诉你
- 不需要查 imenu,抬眼就看到

**底层**: which-function-mode 复用 imenu 的索引。每次光标移动,它查"当前 point 落在哪个 imenu 项的范围里",显示那个项的名字。

**陷阱**: 和 imenu 一样,大文件下 which-function 可能让光标移动变慢 (每次都要查索引)。如果遇到这种情况,关掉 `which-function-mode` 或限制它只在特定 mode 启用:

```elisp
(setq which-func-modes '(emacs-lisp-mode python-mode c-mode))
```

---

## 9. xref: 跨文件跳转

### 9.1 xref 的角色

xref 是 Emacs 25+ 的统一"代码引用"API。它把不同 backend (LSP、etags、grep) 统一到一个接口下:

| 命令 | 键位 | 作用 |
|---|---|---|
| `xref-find-definitions` | `M-.` | 跳到光标处符号的定义 |
| `xref-find-references` | `M-?` | 列出所有引用光标处符号的地方 |
| `xref-pop-marker-stack` | `M-,` | 跳回上一个位置 |
| `xref-find-apropos` | `C-M-.` | 按名字搜符号 |

xref 的核心思想: **无论 backend 是什么,用户用同样的键**。

### 9.2 M-. (find-definitions)

光标在符号上,按 `M-.`:

```python
def helper():
    pass

def main():
    helper()    # 光标在 helper 上,按 M-.
```

如果 backend 找到了 `def helper()` 的定义,光标跳过去。如果**多个定义** (比如同名函数在多个文件),xref 弹一个 `*xref*` buffer 让你选。

**底层**: xref 不自己"懂"代码。它问 backend "符号 X 定义在哪?":
- 装 eglot (LSP): backend 是 LSP server (语义精确)
- 装 etags: backend 是 TAGS 文件 (从 `M-x visit-tags-table` 加载)
- 没装任何 backend: xref 退化为 grep (慢,不准)

**Module 1 假设没装 backend**,所以 `M-.` 在大多数情况下会用 grep 或 etags。Module 5 装 eglot 后 `M-.` 变成 LSP 精确跳转。

### 9.3 M-? (find-references)

`M-?` 列出所有**引用**光标处符号的位置:

```python
def helper():
    pass

def main():
    helper()    # 光标在 helper 上

def other():
    helper()    # 这里也调了
    helper()    # 还有这里
```

按 `M-?` 弹一个 `*xref*` buffer:

```
3 matches in 2 files
main.py:7:  helper()
main.py:10: helper()
main.py:11: helper()
```

在 `*xref*` buffer 里:
- `RET` 跳到该位置
- `o` 在另一个 window 打开
- `e` 进入 `xref-edit-mode` 批量编辑所有匹配 (强大的重构工具)

### 9.4 M-, (pop-marker-stack)

`M-.` 跳到定义后,怎么回原位? 用 `M-,` (xref-pop-marker-stack):

```
当前在 main.py:7, M-. → 跳到 helper 定义
M-, → 回到 main.py:7
```

`M-,` 维护一个**marker stack**: 每次 `M-.` 推一个位置,`M-,` pop 一个。可以连按 `M-, M-, M-,` 走历史栈。

**对比 isearch C-o**: isearch 跳完按 `C-o` (occur) 也"回原位",但 isearch 的回原位是单步的——只能回上一个匹配。xref 的 marker stack 是栈,能回多层。

### 9.5 xref 的陷阱

**陷阱 1**: 没 backend 时 `M-.` 经常失败或慢。Module 1 不要期待 `M-.` 像 IDE 一样工作。它能用,但不是主力工具——主力是 imenu 和 isearch。

**陷阱 2**: marker stack 跨 buffer 时,可能跳到不可达的 buffer (比如被 kill 了)。`M-,` 报 "No more" 是正常的——栈空了。

**陷阱 3**: 多个 backend 同时装时,xref 优先级由 `xref-backend-functions` 决定。可能你以为用的是 LSP,实际用的是 grep。用 `M-x xref-backend-functions` 检查当前 backend。

---

## 10. Hideshow: 代码折叠

### 10.1 hs-minor-mode

`hs-minor-mode` (hideshow) 是 Emacs 内置的代码折叠 minor mode:

```elisp
(add-hook 'prog-mode-hook #'hs-minor-mode)
```

开了之后,代码块可以折叠:

```
(defun factorial (n)
  (if (<= n 1)
      1
    (* n (factorial (- n 1)))))

;; 折叠 (hs-hide-block) 后:
(defun factorial (n) ...)
```

`...` 表示被折叠的内容。

### 10.2 hs 命令

| 命令 | 键位 | 作用 |
|---|---|---|
| `hs-hide-block` | `C-c @ C-c` (或 `C-c @ h`) | 折叠当前 block |
| `hs-show-block` | `C-c @ C-s` | 展开当前 block |
| `hs-toggle-hiding` | `C-c @ C-c` | 切换折叠/展开 |
| `hs-hide-all` | `C-c @ C-M-h` | 折叠所有 block |
| `hs-show-all` | `C-c @ C-M-s` | 展开所有 block |
| `hs-hide-level` | `C-c @ C-l` | 折叠到第 N 层 |
| `hs-show-region` | `C-c @ C-r` | 展开选中区域 |

键位是 `C-c @` 前缀 (因为 `@` 看起来像 hide 的"包裹")。很多用户会重绑:

```elisp
(define-key hs-minor-mode-map (kbd "C-c h") 'hs-hide-block)
(define-key hs-minor-mode-map (kbd "C-c s") 'hs-show-block)
(define-key hs-minor-mode-map (kbd "C-c t") 'hs-toggle-hiding)
```

### 10.3 折叠的创造性用法

1. **看函数签名**: `hs-hide-all` 后整个文件只剩 `(defun foo) ...` `(defun bar) ...` 一列签名。一秒看完所有 API。
2. **专注当前函数**: 折叠其他所有 block,只展开你正在改的。减少视觉干扰。
3. **代码 review**: 折叠实现细节,只看签名 + docstring,快速理解模块 API。
4. **临时屏蔽**: 把一段你不想看的代码折叠,而不是删除。
5. **嵌套探索**: `hs-hide-level` 折到第 2 层,看 2 层结构;折到第 1 层,看 1 层 (最粗)。

**陷阱**: hs-minor-mode 折叠后,sexp 移动 (`C-M-f`) 会**跳过折叠内容**——它把折叠的 sexp 当一个整体。这是好事 (避免进入隐藏的代码),但你要意识到: 你"看不到"的内容,`C-M-f` 也不会进入。如果你想看,先 `hs-show-block`。

### 10.4 其他折叠 minor mode

Emacs 还有别的折叠方案:
- `outline-minor-mode`: 用正则识别"标题" (比如 markdown 的 `#`)
- `origami`: 第三方包,比 hs 更智能
- `lsp-bridge-fold`: LSP-based 折叠

hs-minor-mode 是最简单的开箱即用方案。Module 1 用它,够用。

---

## 11. 微练习 (30 题)

### Sexp 移动基础 (8 题)

1. 在 Lisp buffer 里,光标在 `(defun foo ...)` 的 `d` 上,按 `C-M-f` 跳过 `defun` 到空格前
2. 继续按 `C-M-f` 跳过 `foo`、`(args)`、整个 body
3. 按 `C-M-b` 反向跳回 (4 次回到原位)
4. 光标在 `(if (foo) (bar))` 的 `if` 上,按 `C-M-d` 进入 `(foo)`
5. 在 `(foo)` 内按 `C-M-u` 回到 `(if` 的 `(`
6. 按 `C-M-a` 跳到当前 defun 开头
7. 按 `C-M-e` 跳到当前 defun 末尾
8. 按 `C-u 3 C-M-a` 跳到上面第 3 个 defun

### Sexp 操作 (6 题)

9. 光标在 `(foo bar baz)` 的 `bar` 前,按 `C-M-k` 杀 `bar`,然后 `C-y` 召回
10. 按 `C-M-SPC` 选中当前 sexp
11. 按 `C-u 2 C-M-SPC` 选中 2 个 sexp
12. 在 `(a b)` 的 `a` 前按 `C-M-t` 交换两个 sexp
13. 按 `C-M-% \\([a-z]+\\) RET \\1! RET` 给所有小写词加 `!`
14. 用 `M-;` 注释当前行

### 括号匹配 (4 题)

15. 检查 `show-paren-mode` 是否开 (`C-h v show-paren-mode RET`)
16. 如果没开,`M-x show-paren-mode RET`
17. 在一个 6 层嵌套的括号里,光标在最深层,连按 `C-M-u 6` 跳到最外层
18. `M-x check-parens RET` 检查 buffer 是否有 unbalanced 括号

### Comment commands (4 题)

19. 选中一个 5 行代码块,按 `M-;` 注释,再按 `M-;` 取消
20. 在空行按 `M-;` 加注释符
21. 在有注释的行按 `M-;` 把注释延伸到下一行
22. `M-x comment-box RET` 把 region 包进注释框

### Imenu (4 题)

23. `M-x imenu RET` 列当前 buffer 所有定义,选一个跳过去
24. 在 minibuffer 输入函数名前缀,按 TAB 补全
25. `M-x imenu` 第二次,看它是否记住上次位置
26. `C-h v imenu-generic-expression RET` 看当前 mode 的索引正则

### which-function-mode (2 题)

27. `M-x which-function-mode RET` 开启
28. 移动光标到不同函数,看 mode line 是否更新

### xref (3 题)

29. 光标在某符号上,按 `M-.` 尝试跳到定义 (可能失败,看错误信息)
30. 按 `M-?` 查引用 (可能也失败,因为没 backend)
31. 如果你成功用 `M-.` 跳过,按 `M-,` 回原位

### Hideshow (3 题)

32. `M-x hs-minor-mode RET` 开启
33. `C-c @ C-c` 折叠当前 block
34. `C-c @ C-M-h` 折叠所有 block,只看签名

---

## 12. 实战练习

### 任务 1: Lisp buffer 飞速导航

打开任意 .el 文件 (例如 Emacs 自带的 `simple.el`):

1. `M-<` 到文件头
2. `C-M-a` 到第一个 defun 开头
3. `C-M-e` 到该 defun 末尾
4. `C-M-a C-M-f` 跳到 defun 名
5. `C-M-d` 进入参数列表
6. `C-M-f` 跳过参数
7. `C-M-u` 出来
8. `C-u 10 C-M-e` 跳到第 10 个 defun 末尾

目标: 30 秒内完成以上 8 步。

### 任务 2: 用 imenu 列所有 Python 方法定义

打开一个有 5+ 个方法的 Python 文件:

1. `M-x imenu RET`
2. 看 minibuffer 的候选
3. 选一个方法跳过去
4. 用 `consult-imenu` (如果装了 consult) 做 live filter

如果你没装 consult,可以 `M-x imenu` 输入部分名字 + TAB 补全。

### 任务 3: hs-minor-mode 隐藏所有函数体

1. `M-x hs-minor-mode RET`
2. `C-c @ C-M-h` (hs-hide-all) 折叠所有
3. buffer 现在只剩 `(defun foo) ...` 一列签名
4. 用 `C-n`/`C-p` 浏览签名
5. 在某个签名上按 `C-c @ C-s` 展开
6. 看完按 `C-c @ C-c` 折回

目标: 30 秒内看完整个模块的 API。

### 任务 4: 用 xref 做一次定义-引用往返

(需要装 eglot 或 etags,Module 1 暂时跳过)

如果环境允许:

1. 光标在符号上,按 `M-.` 跳定义
2. 在 `*xref*` buffer 看 references
3. 按 `M-,` 回原位

### 任务 5: 用 M-; 批量注释

1. 选中一段 20 行代码
2. `M-;` 注释
3. `C-/` 撤销
4. 再次 `M-;` 注释
5. 在 region 里加一个 `FIXME` 标记
6. 用 `M-s o FIXME RET` 列所有 FIXME

### 任务 6: sexp 重构

打开一个 Lisp 文件:

1. 找一个 `(let ((a 1) (b 2)) ...)` 形式
2. 光标移到 `(a 1)` 前
3. `C-M-t` 交换 `(a 1)` 和 `(b 2)`
4. 现在 binding 顺序反了
5. `C-M-k` 杀掉 `(b 2)`
6. `C-y` 在新位置召回

### 任务 7: 多层嵌套的 C-M-u

构造一个 5 层嵌套:

```elisp
((((((foo))))))
```

光标在最内 `foo` 上。连按 `C-M-u` 5 次,每次上一层。第 6 次应该报错或停在 buffer 头。

### 任务 8: imenu + which-function 配合

1. `M-x which-function-mode RET` 开启
2. `M-x imenu RET` 跳到一个函数
3. 看 mode line 是否更新显示当前函数
4. 滚动到另一个函数,看 mode line 跟着变

---

## 13. 创造性用法汇总

1. **`C-M-u` 上帝视角**: 在深嵌套代码里迷路时,连按 `C-M-u` 直到最外层。这是 Emacs 用户的"home button"。
2. **`hs-hide-all` 看签名**: 阅读新代码库时,折叠所有函数体,只看签名 + docstring,秒懂 API surface。
3. **`M-;` 双向切换**: `M-;` 既是注释也是取消注释,根据上下文自动决定。养成"想注释就 `M-;`"的肌肉记忆,不用区分两个命令。
4. **imenu 当作 outline**: `M-x imenu` 列出的所有定义就是 buffer 的 outline。比 outline-minor-mode 还快。
5. **`C-M-k` 替代 `C-k`**: 代码场景永远优先 `C-M-k`,它保证结构正确。`C-k` 容易留下半截括号。
6. **`C-M-SPC` + `M-w` 复制 sexp**: 比手动选 region 快得多,且保证选对边界。
7. **`C-M-n` 跨函数跳**: 在一系列函数调用之间,用 `C-M-n` 一次跳一个,比 `C-s` isearch 更精确。
8. **sexp 移动测语法**: 如果 `C-M-f` 报 "scan error",说明代码有括号不匹配。这是最快的语法检查——不用编译,光按几下就知道。
9. **`C-M-a`/`C-M-e` 在 Python 也工作**: 在 Python mode 里,`C-M-a` 跳到上一个 `def` 顶部。跨语言肌肉记忆。
10. **imenu-generic-expression 自定义**: 在你的 mode 加自定义 indexer,让 imenu 列出 TODO、FIXME、特定宏等。

---

## 14. 陷阱与反例

### 陷阱 1: 括号不匹配时 sexp 命令报错

```elisp
(defun foo (bar baz
  (something))
```

(注意 `(bar baz` 没闭合)

按 `C-M-f` 会报 "Scan error: containing expression ends prematurely"。

**解决**: 先 `M-x check-parens RET` 找到问题位置,修复后再用 sexp 命令。

### 陷阱 2: 大文件 imenu 慢

10000 行的 .el 文件,`M-x imenu` 可能要 3-5 秒。如果反复调用,每次都卡。

**解决**:
```elisp
(setq imenu-auto-rescan t)
(setq imenu-auto-rescan-maxout 600000)
```
首次扫描后缓存,后续秒级。但文件超过 600KB 不自动重扫,避免每次编辑都重算。

### 陷阱 3: which-function-mode 拖慢光标

某些 mode (尤其是大量 buffer 修改的) 下,which-function-mode 每次光标移动都重查索引,可能让 Emacs 卡顿。

**解决**: 限制 which-function 只在常用 mode 启用:
```elisp
(setq which-func-modes '(emacs-lisp-mode python-mode c-mode c++-mode))
```

### 陷阱 4: xref 没 backend 时退化

`M-.` 在没装 eglot/etags 的 buffer 里,会用 grep 退化——搜索同名 symbol。这通常不准 (会跳到注释里、字符串里、同名变量)。

**解决**: Module 1 接受这个限制,主力用 imenu。Module 5 装 eglot 后再依赖 `M-.`。

### 陷阱 5: hs-minor-mode 改变 sexp 移动行为

折叠的 block 在 `C-M-f` 看来是一个 sexp——它跳过整个折叠内容,不进入。这是设计,但如果你不知道,会以为 `C-M-f` 漏跳。

**解决**: 折叠的 block 用 `C-c @ C-s` 展开后再用 sexp 命令,或者记住"折叠即不可见,sexp 也尊重"。

### 陷阱 6: M-; 在嵌套 region 行为不一致

选中 region 后 `M-;` 注释,但如果 region 起点在行中,行为可能不直观。比如:

```
|foo     ; 选 foo 到下行的 bar
bar|
```

`M-;` 可能在 foo 行加 `;`,bar 行加 `;`,但 foo 行的注释位置取决于 mode。**永远选整行 region** (`C-a C-SPC C-n ...`),避免半行 region。

### 陷阱 7: sexp 命令依赖 syntax table

在 fundamental-mode 里 (没有 syntax table 设置),`C-M-f` 行为和 lisp-mode 完全不同。在 markdown-mode 里,sexp 边界可能是 `*` 或 `` ` ``。

**解决**: 知道你当前 major mode 的语法约定。`C-h m` 看 mode 文档,理解它的 syntax table。

### 陷阱 8: imenu 索引可能漏定义

imenu-generic-expression 是正则匹配,如果代码用 macro 定义函数 (比如 `(defhelper foo ...)` 而不是 `(defun foo ...)`),imenu 漏掉。

**解决**: 手动加自定义 indexer,或者装 LSP (eglot) 用语义索引。

---

## 15. 自测

1. `C-M-f` 和 `M-f` 的区别?
2. `C-M-u` 和 `C-M-d` 的关系?
3. `C-M-a` 跳到哪里? `C-M-e` 呢?
4. `C-M-n` 和 `C-M-f` 的区别?
5. `C-M-k` 和 `C-k` 的区别?
6. `M-;` 在不同情况下做什么?
7. imenu 和 xref-find-definitions 的区别?
8. which-function-mode 显示什么?
9. `M-,` 干啥?
10. hs-minor-mode 的 `C-c @ C-M-h` 折叠什么?

**答案**:
> 1. `C-M-f` 跳 sexp (语法边界),`M-f` 跳 word
> 2. `C-M-u` 向上跳一层 (到外层开括号),`C-M-d` 向下钻一层 (进入下一个嵌套)
> 3. `C-M-a` 当前 defun 开头,`C-M-e` 末尾
> 4. `C-M-n` 跳到下一个闭括号 (跨 list),`C-M-f` 跳到下一个 sexp 末尾 (sexp 粒度)
> 5. `C-M-k` 杀一个 sexp (结构完整),`C-k` 杀到行尾 (可能留半截括号)
> 6. 当前行无注释 → 行尾加注释;有注释 → 延伸;region 激活 → 注释/取消
> 7. imenu 在当前 buffer 靠正则索引 (快),xref 跨文件靠 backend (准)
> 8. mode line 显示当前光标所在的函数名
> 9. xref-pop-marker-stack,跳回上一个 `M-.` 的位置
> 10. hs-hide-all,折叠所有 block

---

## 16. 毕业检查

- [ ] 30 题至少完成 24
- [ ] 能用 `C-M-f`/`C-M-b`/`C-M-u` 在 Lisp 代码里飞
- [ ] 能用 `M-;` 注释/取消注释 region
- [ ] 能用 `imenu` 跳到任意定义
- [ ] 开了 `which-function-mode` 和 `show-paren-mode`
- [ ] 能用 `hs-minor-mode` 折叠所有函数只看签名

---

## 17. 这把工具在整体里的位置

Module 1 的 8 个 drill 现在完整了:

| Drill | 主题 | 本质 |
|---|---|---|
| 01 | Movement | 字符/word/行级移动 |
| 02 | Kill Ring | 删除与召回 |
| 03 | Search | 查找与替换 |
| 04 | Mark & Rectangle | 选区与列操作 |
| 05 | Kmacro | 重复自动化 |
| 06 | Windows & Frames | 视图组织 |
| 07 | Help System | 自我学习 |
| **08** | **Code Navigation** | **代码结构导航** |

Drill 01-07 处理**通用文本**,Drill 08 处理**代码结构**。两者组合,你能在任何 buffer 里飞——既包括 README、log、配置文件,也包括 Lisp/Python/C 源代码。

Drill 08 也是后面模块的桥梁:
- Module 3 (Elisp) 会大量用 `C-M-f`/`C-M-u` 在 Lisp 里导航
- Module 5 (Power User) 装 eglot 后 `M-.` 变成精确 LSP 跳转
- Module 7 (写包) 在写自己的 major mode 时,要实现 `imenu-generic-expression`

把这个 drill 练熟,你后面 5 个月的学习速度会显著提升。

完成后回到 capstone,验证你的代码导航能力。

---

**下一步**: 完成 capstone (`/home/sun/src/learning/emacs-learning/01-survival/capstone.md`),在 2000 行代码重构里用上 sexp 命令。
