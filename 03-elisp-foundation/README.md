# Module 3: Emacs Lisp 入门 (Chassell)

> **目标**: 系统读完 Chassell 的入门书,建立 Lisp 思维
> **时长**: 5 周 (~100 小时)
> **难度**: ★★★★☆ (核心,概念密集)
> **依赖**: Module 0-2 (基本操作)
> **核心产出**: 写一个完整的 todo-mode 包 (200+ 行)

---

## 0. 这个模块在学什么

你已经能在 Emacs 里飞快编辑文件了。但你**还没真正学过 Elisp**。Module 0 的最小 init.el 你写了,但可能不完全理解每一行;Module 1-2 的命令你熟了,但每个命令背后是 Elisp 函数——你没看过它们的源代码。

这个模块解决这个: **让你真正学会 Elisp**。从"会用 Emacs"升级到"能改造 Emacs"。

"真正学会"在这里有具体标准,不是模糊感觉。5 周后,你会:

- 读 Chassell 全书 17 章,理解每个概念 (不只是会做题)
- 建立 Lisp 思维 (递归、cons、引用、闭包)——能在脑子里把任意 list 代码"运行"出来
- 写过 30+ 个自己的 Elisp 函数——不是抄,是独立设计
- 写出第一个完整的 minor/major mode (todo-mode)——有 keymap、有 customize、有 ERT 测试
- 能读大部分 elisp 代码——包括你装的包的源码

这 5 条达成 4 条以上,才算"真正学会"。Module 6 会用更深的视角重读 Elisp Reference,那时你会感到"轻松"。

---

## 1. 为什么 Chassell?

Emacs Lisp 有两个官方手册,它们是 Emacs 自带的、和源代码同步更新的"权威文本":

- **emacs-lispintro** (Chassell): 入门,为非程序员写,大约 200 页
- **emacs-lispref**: 参考,为有经验的程序员写,1100+ 页密集术语

新手直接读 Lisp Reference 会**淹没**在 1100 页密集术语里。原因不是它写得差,而是它**假设你已经有 Lisp 思维**: 你要懂 S-表达式、quote、cons cell、Lisp-2、lexical binding 这些概念后,reference 才读得顺。它不"教",只"列"。

Chassell 是**桥梁**: 它从零开始,每个概念都有可跑的例子,大量重复加强记忆。它的目标读者是"完全不会编程"的人——这听起来低,但对转 Lisp 的人极其有用,因为 Lisp 的"思维翻转" (代码即数据) 比学语法要难得多。

### Chassell 的独特之处

Chassell 之所以成为 Emacs 社区公认的"入门首选",有几个具体的设计选择:

- **为非程序员写**: 它不假设你懂 C/Java/Python 的循环、变量、函数概念。所以它会从"什么是变量"开始讲,不像 SICP 假设你看过数学证明。
- **每个例子都可跑**: 在 `*scratch*` 里 `C-x C-e` eval 一遍就能验证。Chassell 反复强调"动手试",不让你只读不练。
- **大量重复**: 同一概念从不同角度讲。比如 `cons` 在第 1、7、9 章出现三次,每次深化 (引入→操作→底层)。
- **真实例子**: 用 `mark-whole-buffer`、`append-to-buffer` 等 Emacs 真实函数做例子,不是空想 toy example。
- **章末有练习**: 题目难度递进,而且答案在书里。

### 这本书的局限

Chassell 不是全能的。讲清楚它的边界,你才能正确地用它:

- **节奏慢**: 对有编程经验的人,前几章可能太啰嗦。第 1-2 章讲"什么是 list",对程序员可能是 5 分钟扫过的事。
- **不全面**: 不讲宏 (`defmacro`)、不讲 text property、不讲 process、不讲 EIEIO (对象系统)。这些在 Lisp Reference 里。
- **过时部分**: 有些例子用 dynamic binding (Emacs 25 前默认),而现代 Elisp 默认 lexical。我们会标注这些过时点,并教你正确的现代写法。

但作为**入门**它无可替代。Lisp Reference 像"字典",Chassell 像"入门课"。先上课,再查字典。

---

## 2. 5 周计划

5 周看起来很多,但 Lisp 的学习曲线决定它**必须慢**: 前 2 周打基础 (list, eval, defun),第 3 周是关键转折 (递归和闭包),第 4 周飞跃 (lambda + mapcar 高阶函数),第 5 周综合 (写真实包)。如果你跳过某周,后面会"撞墙"——比如你跳过 W3 闭包,W4 的 lambda 例子你看不懂。

每周一个文件夹,内容包含:
- `chassell-reading.md`: 内联讲解 Chassell 的章节 (替代原文)
- `concept-anchor.md`: editor manual 的对应章节 (穿插式)
- `exercises.md`: 自己设计的练习题 (不是书里的)

这三个文件有不同目的: `chassell-reading` 是"理论课",讲概念;`concept-anchor` 是"桥梁",把 Elisp 和你每天用的编辑器命令连起来 (比如 `C-s` 背后是 `isearch-forward` 函数);`exercises` 是"作业",让你动手巩固。三者按顺序读,但概念读完后立刻做 exercises (不要拖)。

| 周 | Chassell 章节 | 主题 | 产出 |
|---|---|---|---|
| W1 | 1. List Processing, 2. Practicing Evaluation, 7. car/cdr/cons, 9. How Lists are Implemented | Lists + Evaluation | 10 个 list 操作函数 |
| W2 | 3. How To Write Function Definitions, 4. A Few Buffer-Related Functions | defun + let + if | 5 个实用 buffer 函数 |
| W3 | 5. A Few More Complex Functions, 6. Narrowing and Widening, 11. Loops and Recursion | Conditionals + Recursion | length/recursion-based utilities |
| W4 | 8. Cutting and Storing Text, 10. Yanking Text Back, 12. Regexp Searches, 13. Counting via Repetition and Regexps | Lambda + mapcar + sequences | 重构日常重复代码 |
| W5 | 14. Counting Words in defun, 15. Readying a Graph, 16. Your .emacs File, 17. Debugging | 综合 | TODO 管理器 (200+ 行) |

表的"产出"列是每周必须完成的代码。如果产出没做出来,这周不算毕业。每个产出都直接用本周学的概念,所以"做产出"就是"复习"。

---

## 3. 第一性原理: Lisp 思维

在开始读 Chassell 之前,先建立 5 条 Lisp 心智模型。

这 5 条是 Lisp 与 C/Java/Python 的本质差异。理解它们,你后续学所有具体语法 (defun, lambda, if) 都会"水到渠成"——它们只是这 5 条原则的具体化。不理解它们,你学 Lisp 就是"背语法",记不住。

### 3.1 一切是 List

Lisp = **LIS**t **P**rocessing。这个名字是历史遗产: 1958 年 McCarthy 设计这门语言时,核心想法就是"用 list 表示一切"。70 年过去,这个想法被证明是 Lisp 的**根本优势**——它让代码和数据用同一种结构表示,从而让元编程 (写生成代码的代码) 极其自然。

所有代码、所有数据,本质是嵌套的 list:

```elisp
(+ 1 2 3)             ;; 一个 list,4 个元素: + 1 2 3
(defun foo (x) (* x x))  ;; 一个 list,3 个元素: defun (x) (* x x)
```

注意我们怎么"读"这两行: 第一行,我们说"`(+ 1 2 3)` 调用 `+` 函数";第二行,我们说"`defun` 定义 `foo`"。但其实这两行**结构相同**: 都是 list,第一个元素是 symbol,后面是参数。这种"统一性"是 Lisp 的核心。

**直觉**: 看到 Lisp 代码,把它想成"括号嵌套的树"。`(+ 1 (* 2 3))` 是树:

```
    +
   / \
  1   *
     / \
    2   3
```

树根是 `+`,左子是 `1`,右子是 `(* 2 3)` 这棵子树。这种"代码即树"的视角在所有 Lisp 里都适用,后面学宏、学代码变换时,你会反复用到。

### 3.2 代码即数据 (Homoiconicity)

普通语言 (Python, C, Java) 里,代码是字符串 (`"def foo(): pass"`),数据是对象 (dict, list)。它们是两个世界。编译器/解释器在它们之间架桥,但这桥对程序员不透明。

Lisp 不一样: 代码**就是** list 数据。这不是抽象口号,是字面事实:

```elisp
;; 这是一段代码:
(+ 1 2)
;; 这也是 list 数据 (3 个元素的 list):
(+ 1 2)
;; 它们长得一样!
```

你可能会问: "长得一样不代表是同一回事吧?" 不,Lisp 是真的一样。当你 `read` 一个文件,Lisp 把 `(+ 1 2)` 读成 list 数据;当你 `eval` 这个 list,Lisp 把它当代码执行。同一个对象,两种解释方式。

**为什么这重要**: 这个特性 (叫 homoiconicity,希腊语"同样的形象") 让 Lisp 有"宏"——一种能在编译期操作代码的机制。Python 的装饰器、Java 的注解、C 的 `#define`,都是 homoiconicity 的"低配版本",做不到 Lisp 宏能做到的事 (代码生成、DSL、编译期计算)。Module 6 会深入讲宏,Module 3 我们先打好基础。

具体好处:
- 你可以**生成代码** (宏): `(defmacro my-when (cond &rest body) `(if ,cond (progn ,@body)))` —— 几行代码生成 `if` 包装。
- 你可以**读代码**作为数据: `(read "(+ 1 2)")` 返回 list `(+ 1 2)`,可以分析它。
- 代码可被分析、变换、执行,都用同一套机制 (list 操作)。

### 3.3 Eval 求值

每个 Lisp form (表达式) 求值后产生一个值。"求值"是 Lisp 解释器的核心动作: 输入一个 form,输出一个值。

```
Form        Value
----        -----
1           1
"hello"     "hello"
+           <function +>
(+ 1 2)     3
foo         <value of variable foo>
'foo        foo (symbol, 不 eval)
```

这张表看似简单,但它定义了 Lisp 的全部语义。注意 `foo` 和 `'foo` 的区别: 同样的 symbol,加不加 `'` 决定它是变量还是字面量。这是 Lisp 的第一个"惊讶点"——Python 写 `foo` 永远是变量,Lisp 不行。

**Eval 规则**:
- 数字、字符串: eval 为自身 (这些叫 self-evaluating atom)
- symbol: eval 为它的 variable value (查符号表)
- list `(a b c)`:
  1. 看 `a` 是不是 special form (`if`/`let`/`defun` 等)
  2. 如果不是,eval `a` 得到 function,eval `b` `c` 得到参数
  3. 调用 function

这个规则**递归**应用: eval `(+ 1 (* 2 3))` 会先 eval `(* 2 3)` 得 6,再 eval `(+ 1 6)` 得 7。所有复杂表达式的求值都靠这一条规则展开。

### 3.4 Quote 阻止 Eval

`(quote foo)` 或 `'foo` 表示"不要 eval,给我 symbol 本身"。这是 Lisp 的"逃生舱"——默认情况下 Lisp 看到任何东西都 eval,quote 让你"暂停 eval,给我字面值"。

```
foo      → <value of variable foo>
'foo     → foo (the symbol)
(+ 1 2)  → 3
'(+ 1 2) → (+ 1 2) (the list,不 eval)
```

第 3 行和第 4 行的差异是 Lisp 最容易让新手困惑的点。`(+ 1 2)` 是"调用 + 加 1 2",但 `'(+ 1 2)` 是"给我这个 list,不要算"。理解了 quote,你就理解了为什么 `(list 'a 'b)` 和 `(list a b)` 行为不同 (前者用 symbol `a`、`b`,后者用它们的变量值)。

**直觉**: `'` 是"逃跑"——别 eval,给我原文。这个比喻在所有 Lisp 里都对。

### 3.5 Recursive Thinking

Lisp 鼓励递归思维。这不是"递归更好"的教条,是 list 结构天然契合递归:

- 处理一个 list = 处理第一个元素 + 递归处理剩下的

为什么是"天然的"?因为 list 本身就是递归定义的: 一个 list 要么是 `nil`,要么是 `(car . cdr)`——其中 `cdr` 又是一个 list。所以处理 list 的算法自然就是"处理 car,递归处理 cdr":

```elisp
(defun my-length (list)
  (if (null list)
      0
    (1+ (my-length (cdr list)))))
```

读法:
- 如果 list 是空,长度 0
- 否则,长度 = 1 + (剩下的长度)

这写法叫"递归的",但其实和数学归纳法一模一样: "空 list 长度 0;非空 list 长度 = 1 + 剩下 list 长度"。

**和命令式对比**:

```python
# Python
def my_length(lst):
    count = 0
    for x in lst:
        count += 1
    return count
```

Python 用 mutable state (`count`) 和循环。Lisp 版本没有 loop、没有 mutable state,只有递归。两者都正确,但思维方式不同:

- Python: "我有个计数器,逐个加"——操作一个 box
- Lisp: "空 list 长度 0;一个元素 + 剩下的长度"——结构归纳

(Module 3 W3 会深入讲为什么 Lisp 偏爱递归)

---

## 4. 学习节奏

### 每天的节奏 (建议 3-4 小时)

```
第 1 小时: 读 chassell-reading.md
  - 在 Emacs 里 *scratch* 跟着例子 eval
  - 不懂的地方标记,第二天回看

第 2 小时: 做 exercises.md 的题
  - 不看答案,先自己写
  - 卡住的看 chassell-reading 相关部分

第 3 小时: 写自己的"小项目"
  - 把今天学的应用到自己的 init.el
  - 写一个解决真实问题的小函数

第 4 小时 (可选): 读 Chassell 原文
  - 在 Emacs 里 C-h i m Emacs Lisp Intro RET
  - 看看 Chassell 怎么讲的 (我们教程吸收了核心,但原文有更多细节)
```

### 每周的节奏

```
Day 1-5: 上述每天节奏
Day 6:    整合本周学的,写"周报"
Day 7:    休息 / 复习
```

---

## 5. 准备: 工作环境

### 5.1 *scratch* buffer

`*scratch*` 是你的 Lisp playground。
默认 major mode 是 `lisp-interaction-mode`,允许 `C-x C-e` eval。

打开: `C-x b *scratch* RET`。

### 5.2 IELM (Interactive Emacs Lisp Mode)

更好的 REPL:

```
M-x ielm RET
```

进入 `*ielm*` buffer,你可以输入 Elisp 表达式,ENTER eval。

比 scratch 好:
- 显示 return value
- 支持 multi-line
- 支持 TAB 补全

### 5.3 配置: lexical binding

Chassell 的书用 dynamic binding (那时是默认)。
现代 Emacs (25+) 默认 lexical。在 `*scratch*` 或你的 .el 文件**首行**加:

```elisp
;; -*- lexical-binding: t; -*-
```

让该 buffer/file 用 lexical binding。

(Module 3 W3 详细讲 lex vs dyn)

### 5.4 自动 paren 高亮

确保 init.el 有:

```elisp
(show-paren-mode 1)
(electric-pair-mode 1)        ; 自动配对
(setq show-paren-style 'expression)  ; 高亮整个表达式
```

### 5.5 Paredit / smartparens (推荐)

装 `paredit` 或 `smartparens` 包:
- 括号不允许不平衡
- 帮你结构化编辑 S-表达式

```bash
M-x package-install RET paredit RET
```

加到 init.el:
```elisp
(add-hook 'emacs-lisp-mode-hook #'enable-paredit-mode)
(add-hook 'lisp-interaction-mode-hook #'enable-paredit-mode)
(add-hook 'ielm-mode-hook #'enable-paredit-mode)
```

但 Module 3 W1-W2 先别装,等熟悉了再装。

---

## 6. 评估方法

每周结束,问自己:

### Read 自检

- 这周读的章节,核心概念能复述吗?
- Chassell 的例子,我能不看书自己写吗?

### Drill 自检

- exercises.md 的题目,我做了多少?
- 卡住的题目,理解后能再做一遍吗?

### Build 自检

- 本周的"产出"完成了吗?
- 我能解释自己写的代码每一行吗?

如果有一项 < 60%,延长这周。

---

## 7. 毕业检查 (5 周后)

### 概念题

1. 解释 homoiconicity (代码即数据)
2. 区分 `(eq 'a 'a)` `(eq "a" "a")` `(equal "a" "a")`
3. 解释 `'foo` 和 `foo` 的区别
4. 解释 lexical vs dynamic binding
5. 什么是闭包 (closure)? 举个例子
6. `let*` 和 `let` 区别?
7. `funcall` 和 `apply` 区别?
8. 什么是 recursion? 用递归写一个 reverse

### 实操题

1. 写一个函数 `(my-filter PRED LIST)` 过滤 list
2. 写一个 `(count-words-region)` 数 region 内词数
3. 写一个 minor mode (用 `define-minor-mode`)
4. 写一个 `(my-mapcar FN LIST)` (不用 mapcar,自己实现)
5. 写一个 `(my-reverse LIST)` 用递归
6. 写一个 `(insert-date)` 函数,绑到 `C-c d`
7. 写一个 `(with-temp-buffer-message MSG)` 宏
8. 解释 `(defun foo () (let ((x 1)) (lambda () x)))` 返回的闭包

---

## 8. 不要做的

下面这些"陷阱"每个 Lisp 新手都踩过。提前警告,你能少走 6 个月弯路。

- ❌ **不要只读不做**: Lisp 必须动手,光读记不住。Lisp 的"思维翻转" (代码即数据、递归思维) 不是看就会的,要写代码"激活"肌肉记忆。每读完一个概念,立刻在 `*scratch*` eval 几个例子。
- ❌ **不要跳到 Module 6**: Lisp Reference 太难,先过 Chassell。Module 6 假设你已经有 Lisp 直觉——没有 Module 3 的训练,你读 reference 就是"看天书"。
- ❌ **不要装一堆包**: 用裸 Emacs 写 Elisp,先理解再装包。装包让你"看到结果",但不让你"理解原理"。Module 3 你应该用 vanilla Emacs + 内置工具 (`*scratch*`、`ielm`、`edebug`)。
- ❌ **不要急着写大项目**: 先把每周的 exercises.md 做完。Week 1-4 的 exercises 是"递进训练",跳过会"基础不牢"。Week 5 capstone 是综合——基础不牢时写 capstone 会卡死。
- ❌ **不要抄答案**: exercises.md 的题目,自己先想。卡 30 分钟再看答案。看完答案**关上重写**——否则你只"看懂",没"学会"。

这些"不要做"不是教条——每个都来自无数前人的血泪经验。你不听,后期会回来重学。

---

## 9. 下一步

进入 `week-01-lists-eval/chassell-reading.md` 开始第一周。
