# Week 1: Lists + Evaluation — Lisp 的根基

> **目标**: 理解 list 数据结构、Lisp 求值规则、car/cdr/cons 三大操作
> **时长**: 1 周 (~20 小时)
> **Chassell 章节**: 1. List Processing, 2. Practicing Evaluation, 7. car/cdr/cons, 9. How Lists are Implemented

---

## 0. 这周在学什么

Lisp = LISt Processing。
要学 Lisp,**先彻底理解 list**。

这听起来很基础——list 不就是数组吗? 不。Lisp 的 list 是**特殊的数据结构**,它和代码"同构",这是 Lisp 区别于所有其他语言的根本。一旦你真正理解了 list,你就理解了 Lisp 的一半。剩下的一半 (函数、控制流、宏) 都建立在 list 之上。

这周你将建立:

1. **list 的数据直觉**: 看到 `(a b c)`,你想的是"3 元素 list"
2. **eval 的规则**: 看到 `(+ 1 2)`,你知道它如何变成 3
3. **car/cdr/cons 三大操作**: 拆、合 list 的基本原语
4. **底层实现**: cons cell 的内存模型

这周不要急着写复杂代码,**把基础打牢**。基础不牢,后面 W3 递归、W4 lambda 你会处处碰壁。Lisp 的学习曲线是"前慢后快"——这周慢一点,后面起飞。

---

## 1. Lisp 历史回顾 (Chassell Preface + Why + Lisp History)

要真正理解一个工具,你必须知道它从哪里来。Lisp 不是凭空设计出来的,它的每个特性都是 1958 年某个具体问题的解决方案。理解这些问题的来源,你就能理解为什么 Lisp 长成现在这样——它的"怪"都是有原因的。

### 1.1 Lisp 1958

John McCarthy 1958 年在 MIT 发明 Lisp。他本来在研究 AI (Artificial Intelligence),需要一个能表示"符号表达式"的语言。当时的主流语言 (Fortran, Algol) 是"数值"导向的,不适合符号计算。

McCarthy 的灵感来自两个源头:
- **Lambda Calculus** (Church 1930s): 数学家 Alonzo Church 发明的"函数演算",用来研究函数定义、应用和递归。Lambda Calculus 是函数式编程的数学基础。
- **IPL** (Information Processing Language, Newell & Simon 1956): 一种早期的"列表处理"语言,用于 AI 程序。IPL 有 list 但语法笨重。

McCarthy 把这两者结合: Lambda Calculus 的"函数是一等公民" + IPL 的"list 是数据结构",造出 Lisp。这个结合不是显然的——McCarthy 在 1960 年的论文 "Recursive Functions of Symbolic Expressions and Their Computation by Machine" 里第一次系统描述了 Lisp。

McCarthy 的两个关键设计:

1. **List as universal data structure**: 任何结构都能用 list 表示。这是简化——只学一种数据结构,就能表示一切。
2. **Code is data (homoiconicity)**: 程序本身是 list,可以被程序操作。这是革命性想法——让 Lisp 能"写 Lisp",从而能宏、能 DSL、能元编程。

这两个设计让 Lisp 适合**元编程** (写生成代码的代码)。今天所有"现代"特性 (宏、模板、代码生成),Lisp 60 年前就有了。

一个有趣的事实: McCarthy 后来说,他没想到 Lisp 会成为最古老的"活"语言。Fortran 比它早 1 年,但 Fortran 的"现代形态"和 1957 版完全不同。Lisp 的核心 (S-表达式、eval/apply、递归) 从 1958 到 2026 几乎没变。这是**简洁设计的胜利**。

### 1.2 Emacs Lisp 1985

Stallman 1985 年写 GNU Emacs,选了 Lisp 作为扩展语言。这不是随手选——Stallman 自己是 Lisp 文化的人,他在 MIT AI Lab 工作过,看着 Lisp Machine 长大。

为什么 Stallman 选 Lisp 而不是别的?有几个理由:

- **Lisp 动态**,适合"配置即代码": 用户可以在不重启的情况下改 Emacs 行为,这是静态语言做不到的。
- **Lisp 的 macro 系统允许 DSL**: 用户可以用宏定义自己的"领域语言",这是 GNU Emacs "可编程编辑器"理念的核心。
- **Stallman 自己是 Lisp 文化的人** (MIT AI Lab): 他熟 Lisp,知道 Lisp 适合做扩展。
- **Gosling Emacs (商业产品) 已经用 Mocklisp**: Stallman 想做出更好的版本,真 Lisp 而非 Mocklisp。

但 Emacs Lisp 没有照搬 Common Lisp (1984 标准化的"大 Lisp")。Stallman 简化了它:

- **Lisp-2** (变量和函数分开命名空间): 和 Common Lisp 一致
- **默认 dynamic binding** (Emacs 25+ 可选 lexical): Common Lisp 默认 lexical,Emacs 跟得慢
- **没有 CLOS** (对象系统): 太复杂,Stallman 不要
- **没有 conditions 系统** (只有 `condition-case`): Common Lisp 有更强大的状态系统,Emacs 简化
- **简单到一个人能写完**: Emacs 1985 的 elisp 部分基本是 Stallman 一个人写的

这种"简化"是双刃剑: 一方面,Emacs Lisp 容易学,新手几天就能写自己的命令;另一方面,它缺少一些现代特性 (像 Python/Clojure 的某些便利)。

### 1.3 为什么 Chassell 选择这种教学法

Chassell 注意到: 大多数 Lisp 教科书先讲"primitive data types" (数字、字符串),再讲"list"。这是按"复杂性"排序: 数字简单,list 复杂。

但 Chassell 反其道而行: **从 list 开始**。为什么?因为 Lisp 的灵魂是 list。如果你先学数字,你以为 Lisp 是"另一种算术语言",错过了核心。先学 list,你立刻进入"Lisp 思维"——看到任何东西都问"这是 list 吗? 怎么拆?"。

数学上,一个 list 要么是 `nil` (空),要么是 `(car . cdr)` (一个元素 + 剩下的 list)。这个递归定义本身就规定了 Lisp 的处理方式——递归。所以 Chassell 的"先 list"教学法**和 list 的数学结构对齐**,这就是为什么它效果好。

---

## 2. 第一性原理: List 是什么?

要从根本上理解 Lisp,你必须能从"第一性原理"推出 list 是什么、它怎么用。不要死记"`(car '(a b c))` 返回 a"——要理解为什么。

第一性原理推导 (5 步):

- **推论 1**: 我们要表示"有序集合"。最简单的有序是元素接元素的链式结构——`(a . (b . (c . nil)))`。每个节点是一个"cons cell": 一对指针 (car, cdr)。
- **推论 2**: 太啰嗦,加语法糖。`(a . (b . (c . nil)))` 写起来累,所以 Lisp 加了 `(a b c)` 的简写——这是语法糖,内部仍是嵌套 cons cell。
- **推论 3**: 既然是 cons cell 链,操作只需 car、cdr、cons。三种操作足以拆/合任何 list。其他 (nth、append、reverse) 都是这三种的组合。
- **推论 4**: 递归是天然操作模式。因为 list 自身递归定义 (空 list 或 car+cdr),处理算法自然就是"处理 car,递归处理 cdr"。
- **推论 5**: 同构——代码也是 list。既然 list 这么通用,代码也用 list 表示。eval 就是用 car 取函数,cdr 取参数。这是 Lisp 的"决定性天才": 用一种结构表示一切。

这 5 步推完,Lisp 的全部"奇怪"都消失了: quote 阻止 eval (因为代码即数据,要分清)、宏是操作 list (因为代码是 list)、递归自然 (因为 list 递归定义)。

### 2.1 直觉建立

**List 是一个有序的元素集合**。但这个定义太抽象,我们要更具体:

```
(a b c)         ;; 3 个元素的 list
()              ;; 空 list,等于 nil
(1)             ;; 1 个元素的 list
((a b) c)       ;; 嵌套 list: 第一个元素是 (a b),第二个是 c
("hello" 42 x)  ;; 不同类型的元素
```

**关键特征**:
- 有序: `(a b c)` ≠ `(b c a)`
- 元素可以是任何类型 (包括 list): list 可以嵌套,这让它能表示树形结构
- 可以是空的 (`nil`): 这点重要——空 list 不是"特殊状态",它是合法的 list

第 4 行 `((a b) c)` 展示了嵌套 list——它是 2 元素 list,第一个元素本身是个 list。这种嵌套能力让 list 能表示任意复杂结构 (JSON、XML、AST 都能编码为 list)。

### 2.2 代码 vs 数据

最关键的认知: **list 既是数据,也是代码**。这是 Lisp 和所有其他语言的根本区别。

```elisp
;; 数据视角: 这是一个 list,3 个元素
(+ 1 2)

;; 代码视角: 这是一个表达式,调用 + 函数,参数 1 2
(+ 1 2)
```

这两个视角不是"比喻",是字面事实。Lisp 解释器看到一个 list,默认把它当代码 eval;加 `'` 后当数据。

**Eval 行为**:

- 数据视角: 不 eval,就是 list
- 代码视角: eval,list 的第一个元素是函数,剩下是参数

```elisp
(+ 1 2)         ;; eval → 3 (调用 +)
'(+ 1 2)        ;; 不 eval (quote),返回 (+ 1 2) 这个 list
(list + 1 2)    ;; 构造 list,返回 (#<function +> 1 2)
```

第 3 行展示了一个微妙点: `(list + 1 2)` 把 `+` 作为参数传给 `list`,因为 `+` 在 list 的"参数位置"——Lisp eval 它 (得到 function),然后 `list` 把这些 function 和数字打包成 list。所以返回的 list 的第一个元素是 `#<function +>` 这个 function 对象。

### 2.3 Quote 的本质

`'foo` 是 `(quote foo)` 的语法糖。糖很重要——Lisp 程序员每天用 quote 几十次,完整写 `(quote foo)` 太累。`'` 是 reader 层的转换。

`(quote X)` 是 special form,返回 X 本身**不 eval**:

```elisp
foo              ;; → <value of variable foo>
'foo             ;; → foo (the symbol)
(quote foo)      ;; → foo (相同)

(+ 1 2)          ;; → 3
'(+ 1 2)         ;; → (+ 1 2)
(quote (+ 1 2))  ;; → (+ 1 2)
```

注意 `'foo` 和 `(quote foo)` 完全相同——Lisp reader 看到 `'foo` 立即转成 `(quote foo)`。所以"语法糖"在这里是字面的: `'` 是 read-time 转换。

**直觉**: `'` 是"等一下,先别 eval,给我原文"。

这个直觉有多重要?后面学宏时,你会反复用到"什么时候该 quote"。规则简单: 想要 symbol/list 本身就 `'`,想要它的值/求值结果就不 `'`。

### 2.4 在 Emacs 里试

光读不够,你必须**动手 eval** 才能记住。这是 Chassell 的核心教学法——"做中学"。打开 `*scratch*` 或 `M-x ielm`,把下面代码贴进去,光标放在末尾按 `C-x C-e`。

```elisp
(+ 1 2)              ;; C-x C-e → 3
(+ 1 2 3 4)          ;; → 10
(- 10 3)             ;; → 7
(* 2 3 4)            ;; → 24
(/ 100 7)            ;; → 14 (integer division)
(/ 100 7.0)          ;; → 14.2857... (float)
(mod 10 3)           ;; → 1
(1+ 5)               ;; → 6
(1- 5)               ;; → 4

(message "hello")    ;; → "hello"
(message "%d + %d = %d" 1 2 3)  ;; → "1 + 2 = 3"
```

注意几个 surprising 点:

- `(/ 100 7)` 是整数除法 (得 14),`(/ 100 7.0)` 是浮点除法。`7` vs `7.0` 决定结果类型——这是 Lisp 的"数值塔"设计。
- `(mod 10 3)` 返回 1,不是 0.333。`mod` 是模运算,不是除法。
- `(1+ 5)` 是 `(setq x 5) (1+ x)` 的"加 1"——名字怪 (是 `1+` 不是 `+1`),但好用。
- `message` 的 `%d`、`%s` 是格式化占位符,类似 C 的 `printf`。

试 `(quote foo)` 和 `'foo`:

```elisp
'foo                 ;; → foo
(quote foo)          ;; → foo
foo                  ;; → error: void-variable (除非 foo 已 bind)
```

第 3 行 `foo` 没 quote,Lisp 尝试 eval 它作为变量,但 `foo` 未定义,所以报 void-variable error。这是新手最常见的错误。

试 list 操作:

```elisp
'(+ 1 2)             ;; → (+ 1 2)
(list + 1 2)         ;; → (#<subr +> 1 2)
(car '(+ 1 2))       ;; → +
(cdr '(+ 1 2))       ;; → (1 2)
```

第 2 行 `(list + 1 2)` 返回的 list,第一个元素是 `#<subr +>`——`+` 函数本身 (subr 表示 built-in sub-routine)。这展示了"函数是一等公民": 函数可以作为 list 元素存在。

---

## 3. Eval 规则详解 (Chassell Evaluation)

Eval 是 Lisp 的"心脏"——它是一个函数: 输入 form,输出 value。理解 eval,你就理解了 Lisp 怎么运行。

### 3.1 求值的层次

Lisp 求值分多层。Chassell 用一个故事讲清楚——这个故事是"展开嵌套表达式":

```
(+ 1 (* 2 3))              ; 外层 list
     ↓ eval (+ 1 INNER)
     ↓   eval 1 → 1
     ↓   eval (* 2 3):
     ↓     eval (* 2 3)
     ↓       eval * → #<function *>
     ↓       eval 2 → 2
     ↓       eval 3 → 3
     ↓     → 6
     ↓   → 7
```

**规则**: Lisp eval 是**深度优先**的 (内层先 eval)。

这个规则看似简单,但有深刻含义: 任何复杂表达式都能用这一条规则展开。递归求值是 Lisp 的核心机制,所有"高阶"特性 (宏、closure、lazy eval) 都建立在它之上。

类比: 想象一个机械计算器,你输入 `(+ 1 (* 2 3))`,它一步步: 先算 `(* 2 3)` 得 6,再算 `(+ 1 6)` 得 7。Lisp 解释器就是这样的机器,只不过操作的不只是数字,还有 symbol、list、function。

### 3.2 三类 form

一个 form (表达式) 可以是:

1. **Atom (非 list)**:
   - 数字: eval 为自身
   - 字符串: eval 为自身
   - symbol: eval 为其 variable value
   - `nil` / `t`: 特殊常量

2. **List**:
   - 第一个元素是 symbol:
     - Special form (`if`/`let`/`defun`/`quote`/...): 特殊求值
     - 普通函数: eval 参数,调用
     - Macro: 展开,再 eval

3. **其他** (vector, record, etc.): eval 为自身

为什么分三类? 因为它们 eval 的方式不同。Atom 是"自求值"或"查变量"。List 是"调用函数或 special form"。其他 (vector 等) 是"自求值"。

最容易混淆的是 atom 里的 symbol——它和"自求值 atom" (数字、字符串) 不同,它要查变量值。所以 `(setq foo 5) foo` 求值为 5,但 `(setq foo 5) 42` 直接是 42 (数字自求值)。

### 3.3 Special forms

有些 symbol 不会被当函数调用,而是有**特殊求值规则**。这些叫 special forms,它们的求值规则和普通函数不同——比如 `if` 不会 eval 所有参数 (它选 then 或 else 一个 eval)。

| Special form | 行为 |
|---|---|
| `quote` | 不 eval |
| `if` | 条件 (不所有参数都 eval) |
| `let` | 局部 binding |
| `let*` | 顺序局部 binding |
| `setq` | 设置变量 |
| `defun` | 定义函数 |
| `defvar` | 定义变量 |
| `defmacro` | 定义宏 |
| `lambda` | 创建匿名函数 |
| `function` | 类似 quote,但用于函数 |
| `function` | (同 `'`) |
| `cond` | 多分支 |
| `while` | 循环 |
| `and` / `or` | 短路 |
| `condition-case` | 异常 |
| `save-excursion` | 保存 point/mark |
| `save-restriction` | 保存 narrowing |
| `track-mouse` | 鼠标事件 |

(Module 3 W3 和 Module 6 W2 详细讲)

为什么 special forms 存在?如果只有普通函数,所有参数都会被 eval——这没法做 `if` (因为 if 想条件性地 eval then 或 else,不是两个都 eval)。所以 Lisp 设计了一些"特殊求值"的 form,这就是 special forms。它们是语言的核心,不能被用户重定义。

宏 (macro) 是用户自定义的 special form——它允许你扩展语言。Module 6 详细讲。

### 3.4 在 Emacs 里看 eval

理论上讲完了,动手验证。`eval` 是函数,可以**手动**调用——把 list 当数据传给它:

```elisp
(+ 1 2)              ;; C-x C-e → 3
'(+ 1 2)             ;; → (+ 1 2)
(eval (+ 1 2))       ;; → 3 (eval 函数)
(eval '(+ 1 2))      ;; → 3
(eval (quote (+ 1 2)))  ;; → 3 (相同)
```

注意第 3 行 `(eval (+ 1 2))`: 先 eval 参数 `(+ 1 2)` 得 3,然后 `(eval 3)` 因为 3 自求值,得 3。这和第 4 行 `(eval '(+ 1 2))` 不同——后者 eval 参数 `'(+ 1 2)` 得 list `(+ 1 2)`,然后 `(eval (+ 1 2))` 把这个 list 当代码运行,得 3。

`eval` 是函数,可以**手动**调用。
但 Chassell 警告: **不要在日常代码里用 eval**。
(eval 让代码不可静态分析)

为什么不用 eval?因为它打破"代码即数据"的清晰边界。`eval` 把数据当代码——这听起来强大,实际上让你失去编译期检查、性能优化、代码可读性。99% 的"我需要 eval"的场景,应该用 `funcall`、宏或重新设计。

eval 的合法场景: 写 Lisp 解释器、REPL、读用户输入并执行。这些场景你 99% 不会遇到。

---

## 4. car / cdr / cons 三大操作 (Chassell Ch 7)

这三个名字是 Lisp 文化最著名的"怪东西"。每个新手第一次看到都问: "为什么叫 car? cdr? 这是什么意思?" 答案涉及计算机历史、文化传承和一点"无意义的归属感"。

### 4.0 名字的由来: 历史 + 文化

先讲清楚名字的来源,你才能在脑子里"接受"它们,而不是每次用都抗拒。

`car`、`cdr`、`cons` 不是任意命名,而是 1958 年 Lisp 在 IBM 704 计算机上的实现细节。IBM 704 有几个寄存器,其中两个的简写成了 Lisp 操作的名字:

- **car** = "Contents of Address Register" (地址寄存器内容) — 取 cons cell 的左半部分
- **cdr** = "Contents of Decrement Register" (减量寄存器内容) — 取 cons cell 的右半部分
- **cons** = "construct" (构造) — 构造一个新 cons cell

当时的 IBM 704 用"地址寄存器"和"减量寄存器"分别存 cons cell 的 car 和 cdr。McCarthy 团队顺手用寄存器名做操作名,只是省事。

70 年后我们仍用这怪名字,原因:

1. **历史包袱**: 几亿行 Lisp 代码用 `car`/`cdr`,改不了。
2. **短,打字快**: `car` 3 字符,`first` 5 字符。Lisp 函数调用多,短的占便宜。
3. **"无意义"反而让 Lisp 程序员有归属感**: 用 `car`/`cdr` 像"懂行的人"。学 Lisp 的"成年礼"就是理解这俩名字。

Scheme 用 `first`/`rest` 替代,但 Common Lisp 和 Emacs Lisp 保留传统。Module 3 我们用 `car`/`cdr`——以后看别人的代码也用这俩。

三个操作的语义:
- `car` 取第一个元素 (head)
- `cdr` 取剩下的 list (tail)
- `cons` 在前面加一个元素 (prepend)

下面逐个详细看。

### 4.1 car: 取第一个元素

现在让我们看 `car` 的具体行为。它的语义就是"取 list 的第一个元素",但有几个细节新手容易错。

```elisp
(car '(a b c))       ;; → a
(car '(1))           ;; → 1
(car '())            ;; → nil (空 list)
(car nil)            ;; → nil
```

注意第 3、4 行: `(car '())` 和 `(car nil)` 都返回 `nil`,**不报错**。这是 Elisp 设计的"宽容"——很多语言 (Python `lst[0]` on empty list) 会抛异常,Lisp 返回 nil。

为什么这么设计?因为递归处理 list 时,base case 经常是 `nil`。如果 `(car nil)` 报错,递归代码要加很多 check。返回 nil 让递归代码更简洁。

但代价是: 你可能误以为 list 非空,实际拿到 nil,bug 隐藏。所以要心里有数。

**直觉**: `car` 是"取头"。

### 4.2 cdr: 取剩下的 (除第一个)

`cdr` 是 `car` 的"另一半"。一个 cons cell 有两个 slot,`car` 取左,`cdr` 取右。当 cons cell 表示 list 时,cdr 是"剩下的 list"。

```elisp
(cdr '(a b c))       ;; → (b c)
(cdr '(1))           ;; → nil
(cdr '())            ;; → nil
(cdr nil)            ;; → nil
```

注意 `(cdr '(1))` 返回 `nil`,不是 `()`。在 Elisp 里,`nil`、`()`、`'()` 是同一个东西——空 list。这是 Lisp 的简化: nil 既是 false 又是空 list。

`(cdr '())` 返回 nil 也不报错,和 `car` 一样。

**直觉**: `cdr` 是"取尾"。

**为什么 car/cdr 不对称?** 你可能注意到: `car` 返回"一个元素",`cdr` 返回"一个 list"。它们看起来不对称——但其实是底层对称的: `car` 取 cons cell 的 CAR slot (单个值),`cdr` 取 CDR slot (这个 slot 的"值"恰好是另一个 list,因为 list 的递归定义)。理解了 cons cell,对称性就回来了。

### 4.3 cons: 把元素加到 list 头

`cons` 是 car/cdr 的"反向操作"——它构造一个新 cons cell。当 cdr 是 list 时,新 cons cell 也是一个 list (新元素在头)。

```elisp
(cons 1 '(2 3))      ;; → (1 2 3)
(cons 1 nil)         ;; → (1)
(cons 1 2)           ;; → (1 . 2) (这是 dotted pair,不是 list!)
(cons 'a '(b c))     ;; → (a b c)
```

第 3 行是关键陷阱: `(cons 1 2)` 返回 `(1 . 2)`,这是 **dotted pair**,不是 list。为什么?因为 list 要求 cdr 是 list 或 nil——`2` 既不是 list 也不是 nil,所以 `(1 . 2)` 不是 list。

这有啥影响? `length`、`mapcar`、`reverse` 等函数要求 proper list,给它们 dotted pair 会报错。所以 `cons` 默认用来"加到 list 头" (cdr 是 list),不要随便给它非 list 的 cdr。

**直觉**: `cons` 是"在前面加一个"。

### 4.4 组合用法

`car` 和 `cdr` 可以组合,Elisp 提供 `cadr`、`caddr` 等缩写:

```elisp
(car (cdr '(a b c)))         ;; → b (第二个元素)
(cadr '(a b c))              ;; → b (缩写:cadr = car of cdr)
(caddr '(a b c))             ;; → c (第三个元素)
(cddr '(a b c))              ;; → (c) (cdr of cdr)
(caar '((a b) c))            ;; → a (car of car)
```

`caar`、`cadr`、`cdar`、`cddr`、`caddr` 等组合,最多 4 层 (`caddddr`)。

这些缩写看起来怪,但有逻辑: 从右往左读,每个 `a` 是 car,每个 `d` 是 cdr。`cadr` = 先 cdr 再 car = 取第二个元素。`caddr` = cdr, cdr, car = 取第三个元素。

记忆口诀: "a 是头,d 是尾,从右往左剥洋葱"。

为什么不直接用 `(nth 1 list)`? 缩写更短,而且历史习惯。但超过 2 层 (`caddr` 及更深) 可读性下降,这时用 `nth` 更清楚。

### 4.5 List 的实际操作

**构造 list**:
```elisp
(cons 1 (cons 2 (cons 3 nil)))   ;; → (1 2 3)
(list 1 2 3)                      ;; → (1 2 3) (更简洁)
'(1 2 3)                          ;; → (1 2 3) (字面量)
```

**访问**:
```elisp
(car '(1 2 3))            ;; → 1
(cadr '(1 2 3))           ;; → 2
(nth 2 '(1 2 3))          ;; → 3 (nth 从 0 开始)
(nthcdr 2 '(1 2 3 4))     ;; → (3 4) (跳过前 N 个)
(last '(1 2 3))           ;; → (3) (最后一个 cons cell)
```

**修改** (实际是创建新 list,Lisp 数据不可变):
```elisp
(cons 0 '(1 2 3))         ;; → (0 1 2 3)
(append '(1 2) '(3 4))    ;; → (1 2 3 4) (拼接)
(reverse '(1 2 3))        ;; → (3 2 1)
```

### 4.6 在 Emacs 里练习

```elisp
(setq my-list '(a b c d e))

(car my-list)            ;; → a
(cdr my-list)            ;; → (b c d e)
(nth 2 my-list)          ;; → c
(length my-list)         ;; → 5
(member 'c my-list)      ;; → (c d e) (从 c 开始的子 list)
(member 'x my-list)      ;; → nil (没找到)
```

---

## 5. List 的底层: Cons Cell (Chassell Ch 9)

要真正"懂" list,你必须看穿它的"语法糖"——`(a b c)` 是写给人看的,机器里它是嵌套的 cons cell 链。这一节让你看清底层。

为什么看底层重要?因为很多"surprising 行为"只有看底层才能解释: 为什么 `(eq (list 1 2) (list 1 2))` 是 nil?为什么 `setcar` 会"莫名其妙"改两个 list?为什么 list 不擅长随机访问?答案都在 cons cell 的实现里。

### 5.1 Cons Cell 是什么

每个 cons cell 是**两个指针**。这是它最简单的定义: 两个内存位置,各存一个引用。

```
┌───┬───┐
│CAR│CDR│
└───┴───┘
```

CAR 指向"第一个值",CDR 指向"剩下的"。注意我说的是"指向"——cons cell 不存值本身,存的是指针。这是关键,因为它让 list 可以共享结构 (下一节讲)。

list `(a b c)` 实际是三个 cons cell 串起来:

```
┌───┬───┐   ┌───┬───┐   ┌───┬───┐
│ • │ •─┼──►│ • │ •─┼──►│ • │nil│
└─┬─┴───┘   └─┬─┴───┘   └─┬─┴───┘
  ↓           ↓           ↓
  a           b           c
```

每个 `•` 是一个 cons cell,左指针 (CAR) 指向元素,右指针 (CDR) 指向下一个 cell。
最后一个 cell 的 CDR 是 `nil` (表示 list 结束)。

这张图是理解所有 list 操作的基础。 `(car '(a b c))` 拿第一个 cons cell 的 CAR (即 `a`);`(cdr '(a b c))` 拿第一个 cell 的 CDR (即指向 `(b c)` 的指针);`(cons 'x '(a b c))` 造一个新 cell,car 是 `x`,cdr 指向原 list。

### 5.2 验证

在 Emacs 里,我们可以用 dotted pair 来"看见" cons cell:

```elisp
(cons 'a 'b)               ;; → (a . b) (dotted pair,不是 list)
(cons 'a nil)              ;; → (a)
(cons 'a (cons 'b nil))    ;; → (a b)
(cons 'a (cons 'b 'c))     ;; → (a b . c) (improper list)
```

`(a . b)` 是 dotted pair,**不是 list** (因为 CDR 不是 list 或 nil)。它打印成 `(a . b)`,中间有点,表示"这不是 list,是裸 cons cell"。

`(a b c)` 是 proper list,因为最后一个 CDR 是 nil。打印时省略最后的 `. nil`。

`(a b . c)` 是 improper list——前面像 list (a, b),但最后一个 CDR 是 `c` (不是 nil 也不是 cons cell),所以是 improper。这种结构很少用,但有些场景 (assoc list 的 dotted pair 形式) 会遇到。

**为什么区分 proper 和 improper?** 因为很多 list 函数 (`length`、`mapcar`、`reverse`) 假设 proper list,遇到 improper 会报错或行为奇怪。要让代码 robust,要检查 list 是否 proper。

### 5.3 共享结构

由于 cons cell 是指针,两个 list 可以**共享尾部**。这是 Lisp 的内存优化,但也是新手困惑点:

```elisp
(setq tail '(b c))
(setq l1 (cons 'a tail))   ;; → (a b c)
(setq l2 (cons 'x tail))   ;; → (x b c)
```

`l1` 和 `l2` 共享 `(b c)` 部分。在内存里:

```
l1 → ┌───┬───┐         ┌───┬───┐   ┌───┬───┐
     │ • │ •─┼──┐      │ • │ •─┼──►│ • │nil│
     └─┬─┴───┘  │      └─┬─┴───┘   └─┬─┴───┘
       │        └──────►↓           ↓
       a                b           c
                        ↑
l2 → ┌───┬───┐          │
     │ • │ •─┼──────────┘
     └─┬─┴───┘
       ↓
       x
```

`l1` 和 `l2` 的第二个 cell 是**同一个** cons cell (都指向 tail)。这是共享。

**重要**: Lisp 的"修改"通常创建新结构,而不是破坏性修改。所以 `(cons 'a tail)` 不改 tail,只是造一个新 cell 指向它。

但**破坏性修改** (`setcar`、`setcdr`) 可能影响共享结构,要小心:

```elisp
(setq tail '(b c))
(setq l1 (cons 'a tail))
(setcar tail 'Z)           ;; 改 tail 的 car
l1                         ;; → (a Z c) !! l1 也被影响了!
```

这是因为 `l1` 和 `tail` 共享 cell,改 cell 就改了两者。新手踩这个坑会浪费很多时间。

经验: 默认用"不可变"操作 (cons、append、reverse),只在性能关键时用破坏性操作 (setcar、setcdr、nreverse),并清楚记得共享关系。

### 5.4 内存效率

由于共享,Lisp list 比想象的节省内存。比如两个 list `(a b c)` 和 `(x b c)` 共享 `(b c)`,只占 4 个 cons cell (a, b, c, x),不是 6 个。

但 list 不擅长**随机访问** (`(nth 1000 list)` 要遍历 1000 个 cell,O(n))。这是因为 list 是链表——要找第 N 个,必须从头走 N 步。

对比: array (vector) 是 O(1) 随机访问,但插入/删除 O(n)。list 是 O(1) 插入头,O(n) 随机访问。两者各有优势。

要随机访问,用 vector 或 hash table (Module 6 学)。要顺序遍历或头插,用 list。这是数据结构选型的基本权衡。

---

## 6. Symbols 和 Variables (Chassell Variables)

这一节你第一次接触 Lisp 的"双命名空间"——Lisp-2。这是 Lisp 与 Python/JS 最不同之处之一,也是新手困惑的主要来源。

### 6.1 Symbol 是什么

Symbol 是一个**有名字的对象**。它不只是字符串——它有"身份"和"slot"。

```elisp
'foo                  ;; → foo (the symbol)
(intern "foo")        ;; → foo (从 string 找/创建 symbol)
(symbol-name 'foo)    ;; → "foo"
```

注意第 2 行 `intern`: 这是从字符串找/创建 symbol 的函数。Emacs 有一个全局的 `obarray` (object array) 存所有 symbol,`intern` 在里面查;没有就创建。所以 `foo` 这个 symbol 在整个 Emacs 里**唯一**——`(eq 'foo 'foo)` 永远 t。

每个 symbol 有 4 个 slot,这是 Lisp-2 的核心结构:

1. **name**: 字符串名字 (`"foo"`)
2. **value**: 变量值 (作为变量时,`(symbol-value 'foo)`)
3. **function**: 函数值 (作为函数时,`(symbol-function 'foo)`)
4. **property list**: 属性列表 (元数据,`(get 'foo 'prop)`)

这 4 个 slot 让一个 symbol 同时承担"变量名"和"函数名"两个角色。这就是 Lisp-2 的实现机制。

### 6.2 Lisp-2

Emacs Lisp 是 **Lisp-2**: 变量和函数有独立命名空间。这意味着同一个 symbol 名字,既可以作为变量、又可以作为函数,互不干扰。

```elisp
(list + 1 2)    ;; 这里 + 是变量吗? 不,是函数

(setq + 5)      ;; 把 + 设为变量值 5 (坏主意,但可以)
(+ 1 2)         ;; 仍然调用 + 函数,不是变量值 5

;; 要取变量值:
+               ;; → 5
(symbol-value '+)  ;; → 5

;; 要取函数值:
(function +)    ;; → #<function +>
#'              ;; → #<function +> (语法糖)
(symbol-function '+)  ;; → #<function +>
```

注意 `(setq + 5)` 后,`+` 作为变量是 5,作为函数仍是 `#<function +>`。两个空间不冲突。

**Lisp-1 (Scheme/Clojure)**: 变量和函数同一空间。`+` 是一个变量,它的值是函数。如果你想"重新定义 +",直接 `(define + ...)` 就行,但代价是名字冲突更频繁。

**Lisp-2 (Elisp/Common Lisp)**: 分开。好处:
- 不需要小心命名冲突: `(list list)` 中第一个 `list` 是函数,第二个可以是 list 名字的变量
- 函数名可以"借用"普通英语词 (`list`、`car`、`map`),不怕遮蔽变量

坏处:
- 有时要 `#'` 和 `funcall` 显式区分
- 多一点认知负担

Lisp-2 的代价是: 当函数存到变量里 (`(setq f (lambda ()))`),要调用就要 `(funcall f)` 而不是 `(f)`。因为 `(f)` 在 Lisp-2 是"调用名为 `f` 的函数",不是"调用变量 `f` 的值"。这是 Module 3 W4 大量遇到的事。

### 6.3 Setting variables

知道 symbol 有 value slot 后,我们看怎么设:

```elisp
;; setq (quote 第一个参数,所以不用 'foo)
(setq foo 5)            ;; → 5 (foo 现在是 5)

;; set (不 quote,要 'foo)
(set 'foo 5)            ;; → 5

;; let (local binding)
(let ((x 1) (y 2))
  (+ x y))              ;; → 3
;; x 和 y 在 let 外不存在
```

`setq` 是 `set quote` 的缩写——自动 quote 第一个参数。所以 `(setq foo 5)` 等于 `(set 'foo 5)`,但前者更简洁。

`let` 创建**局部**绑定——退出 let 后,变量消失。这是局部作用域,类似 Python 函数参数。但 Elisp 的 let 是 lexical (默认) 或 dynamic (如果变量是 defvar 的)。

(Module 3 W2 详细讲 let)

### 6.4 defvar vs setq

`defvar` 和 `setq` 都能设变量,但用途完全不同:

```elisp
(defvar my-var 5 "Documentation")
;; defvar:
;;   - 设默认值 (如果还没设)
;;   - 加 docstring
;;   - 标记为 "defvar" (不会被 let shadow)
;;   - 不覆盖已有值

(setq my-var 10)
;; setq:
;;   - 强制设值
;;   - 不加 docstring
;;   - 不标记 defvar
;;   - 覆盖已有值
```

**经验**: 配置变量用 `defvar`,临时变量用 `let`。

为什么这么分?`defvar` 有几个 "魔幻" 行为:

- 它**不会**覆盖已有值 (用户在 init.el 设过了,你 defvar 不动它)
- 它**永久**标记这个变量为 "special" (dynamic),即使代码切到 lexical binding 也不变
- 它的 docstring 出现在 `C-h v my-var`

所以 `defvar` 是"声明全局配置变量"的工具,`setq` 是"修改值"的工具。Module 4 详细讲配置体系时,你会大量用 `defvar` 和 `defcustom`。

---

## 7. Practice Exercises (Chassell Ex 1)

### Ex 1.1: 基础 list 操作

```elisp
(car '(1 2 3))                    ;; ?
(cdr '(1 2 3))                    ;; ?
(car (cdr '(1 2 3)))              ;; ?
(cadr '(1 2 3))                   ;; ?
(cons 1 '(2 3))                   ;; ?
(cons 1 (cons 2 '(3)))            ;; ?
(list 1 2 3)                      ;; ?
(length '(a b c d))               ;; ?
(nth 2 '(a b c d))                ;; ?
(reverse '(1 2 3))                ;; ?
(append '(1 2) '(3 4))            ;; ?
```

**答案** (反白):
> 1, (2 3), 2, 2, (1 2 3), (1 2 3), (1 2 3), 4, c, (3 2 1), (1 2 3 4)

### Ex 1.2: my-second
写一个函数 `my-second`,返回 list 的第二个元素 (不用 `cadr`):

```elisp
(defun my-second (list)
  ;; 填写
  )

(my-second '(a b c))     ;; → b
(my-second '(a))         ;; → nil
(my-second '())          ;; → nil
```

**思路**: 第二个元素 = `(car (cdr list))`。先 cdr 取"除第一个的",再 car 取那个的第一个 (即原 list 的第二个)。

**答案**:
> `(car (cdr list))`

### Ex 1.3: my-list-length
用 car/cdr/递归写 `my-length`:

```elisp
(defun my-length (list)
  (if (null list)
      0
      ;; 填写
      ))

(my-length '(a b c))    ;; → 3
(my-length '())         ;; → 0
```

**思路**: 递归。Base case: list 空,长度 0。Recursive: 1 + (my-length (cdr list))。把"剩下的长度"加 1 (当前元素)。

**答案**:
> `(1+ (my-length (cdr list)))`

### Ex 1.4: my-reverse
用递归写 `my-reverse`:

```elisp
(defun my-reverse (list)
  (if (null list)
      nil
      ;; 填写 (用 append)
      ))

(my-reverse '(a b c))   ;; → (c b a)
```

**思路**: 反转 = 拿第一个,放到"反转剩下的"末尾。`(append (my-reverse (cdr list)) (list (car list)))`——先反转 cdr,再把 car 接到末尾。

**答案**:
> `(append (my-reverse (cdr list)) (list (car list)))`

### Ex 1.5: my-member
用递归写 `my-member`:

```elisp
(defun my-member (elt list)
  (if (null list)
      nil
      (if (equal elt (car list))
          list
          ;; 填写
          )))

(my-member 'b '(a b c))    ;; → (b c)
(my-member 'x '(a b c))    ;; → nil
```

**思路**: 如果 car 等于 elt,返回整个 list (从 elt 开始)。否则递归 cdr 找。

**答案**:
> `(my-member elt (cdr list))`

### Ex 1.6: my-nth

```elisp
(defun my-nth (n list)
  (if (zerop n)
      (car list)
      ;; 填写
      ))

(my-nth 1 '(a b c))    ;; → b
(my-nth 0 '(a b c))    ;; → a
```

**答案**:
> `(my-nth (1- n) (cdr list))`

### Ex 1.7: my-butlast

写一个函数,返回除最后一个元素外的 list:

```elisp
(defun my-butlast (list)
  ;; 填写
  )

(my-butlast '(a b c))   ;; → (a b)
(my-butlast '(a))       ;; → nil
(my-butlast '())        ;; → nil
```

**思路**: 跟 my-last 反过来——当 cdr 是 nil 时 (即只剩一个),返回 nil。否则 cons 第一个到 my-butlast 剩下的。

**答案**:
```elisp
(if (or (null list) (null (cdr list)))
    nil
    (cons (car list) (my-butlast (cdr list))))
```

### Ex 1.8: 求和

写 `(my-sum LIST)`,返回 list 中所有数字之和:

```elisp
(defun my-sum (list)
  ;; 填写
  )

(my-sum '(1 2 3 4))     ;; → 10
(my-sum '())            ;; → 0
```

**思路**: Base: 空 list → 0。Recursive: `(car list) + (my-sum (cdr list))`。这是"线性递归"的经典例子。

**答案**:
```elisp
(if (null list)
    0
    (+ (car list) (my-sum (cdr list))))
```

### Ex 1.9: 深度复制 (实际不需要,Lisp 数据不可变)

写 `(deep-copy LIST)` 递归复制嵌套 list:

```elisp
(defun deep-copy (tree)
  (if (consp tree)
      (cons (deep-copy (car tree))
            (deep-copy (cdr tree)))
      tree))

(deep-copy '(1 (2 3) 4))   ;; → (1 (2 3) 4)
```

### Ex 1.10: count

写 `(my-count ELT LIST)`,数 ELT 在 LIST 出现几次:

```elisp
(defun my-count (elt list)
  ;; 填写
  )

(my-count 'a '(a b a c a))   ;; → 3
(my-count 'x '(a b c))       ;; → 0
```

**思路**: Base: 空 → 0。Recursive: 如果 car 等于 elt,1 + 递归;否则 0 + 递归。用 `(if (equal elt (car list)) 1 0)` 把 bool 转 0/1。

**答案**:
```elisp
(if (null list)
    0
    (+ (if (equal elt (car list)) 1 0)
       (my-count elt (cdr list))))
```

---

## 8. Buffer/Region 的简单应用 (Chassell Practicing Evaluation)

### 8.1 在 *scratch* 里 eval

```elisp
(+ 1 2)               ;; C-x C-e → 在 minibuffer 显示 3
(+ 1
   (* 2 3))           ;; C-x C-e → 7
```

### 8.2 eval 整个 buffer

`M-x eval-buffer RET` — eval 当前 buffer 所有 form。
用于 reload init.el。

### 8.3 eval region

选中 region,`M-x eval-region RET` — 只 eval region。

### 8.4 eval defun

光标在某个 `defun` 内部,`C-M-x` (eval-defun) — eval 这个 defun。
用于重新加载某个函数。

### 8.5 ielm (交互式 REPL)

`M-x ielm RET` 进入交互式 Elisp REPL。

```
ELISP> (+ 1 2)
3
ELISP> (defun square (x) (* x x))
square
ELISP> (square 5)
25
```

---

## 9. 实战练习: 第一个 Elisp 包

写一个简单的 `my-list-utils.el` 包:

```elisp
;;; my-list-utils.el --- Simple list utilities -*- lexical-binding: t; -*-

;;; Commentary:
;; 这个包提供简单的 list 操作函数,用于练习 Elisp。

;;; Code:

(defun my-second (list)
  "Return the second element of LIST."
  (car (cdr list)))

(defun my-third (list)
  "Return the third element of LIST."
  (car (cdr (cdr list))))

(defun my-length (list)
  "Return the length of LIST (recursive)."
  (if (null list)
      0
    (1+ (my-length (cdr list)))))

(defun my-reverse (list)
  "Reverse LIST (recursive)."
  (if (null list)
      nil
    (append (my-reverse (cdr list))
            (list (car list)))))

(defun my-sum (list)
  "Sum all numbers in LIST."
  (if (null list)
      0
    (+ (car list) (my-sum (cdr list)))))

(defun my-map (fn list)
  "Apply FN to each element of LIST, return new list."
  (if (null list)
      nil
    (cons (funcall fn (car list))
          (my-map fn (cdr list)))))

(provide 'my-list-utils)
;;; my-list-utils.el ends here
```

保存为 `~/.config/emacs/lisp/my-list-utils.el`。

测试:

```
M-x load-file RET ~/.config/emacs/lisp/my-list-utils.el RET
M-: (my-length '(a b c)) RET    ;; → 3
M-: (my-reverse '(1 2 3)) RET   ;; → (3 2 1)
M-: (my-sum '(1 2 3 4)) RET     ;; → 10
M-: (my-map #'1+ '(1 2 3)) RET  ;; → (2 3 4)
```

**这是你的第一个 Elisp 包!**

---

## 10. 自测

1. `(car '(1 2 3))` → ?
2. `(cdr '(1 2 3))` → ?
3. `(cons 1 '(2 3))` → ?
4. `'(+ 1 2)` → ? (注意 quote)
5. `(+ 1 2)` → ?
6. `(quote foo)` 等于什么简写?
7. 为什么 `(car '())` 不报错?
8. 解释 dotted pair `(a . b)`
9. `(list 1 2 3)` 和 `'(1 2 3)` 区别?
10. 什么是 cons cell? 画 `(a b c)` 的 cons cell 图。

**答案**:
> 1. 1
> 2. (2 3)
> 3. (1 2 3)
> 4. (+ 1 2) (list,没 eval)
> 5. 3
> 6. 'foo
> 7. 因为 (car nil) 定义为返回 nil
> 8. 一个 cons cell,car=a,cdr=b,但 cdr 不是 list,所以整体不是 proper list
> 9. 前者每次 eval 创建新 list;后者每次 eval 同一个 (字面量,可能被 byte-comp 共享)
> 10. 见 5.1 节图示

---

## 11. 毕业检查

- [ ] 10 道 exercise 全部完成
- [ ] my-list-utils.el 写完并测试通过
- [ ] 能用 cons cell 图解释 list 结构
- [ ] 能区分 `'foo`、`foo`、`(quote foo)`
- [ ] 能用递归写 list 操作

完成后:
- 写 `logs/week-01.md` 心得
- 进入 `concept-anchor.md` (editor manual 对照)
- 然后 `exercises.md` (自己设计的练习)
- 完成后进入 `../week-02-defun-let-if/`
