# Intro Reading: Chassell 的 Emacs Lisp 入门 (Module 0 节选)

> 这个文件**完全替代** `emacs-lispintro-30.2/emacs-lisp-intro.texi` 的以下章节:
> - Preface (前言)
> - Why (为什么学 Emacs Lisp)
> - On Reading this Text (如何读这本书)
> - Who You Are (你是谁)
> - Lisp History (Lisp 历史)
> - Note for Novices (新手提示)
> - Thank You (致谢)
>
> 完整的 Chassell 入门书在 Module 3 系统精读。Module 0 只读开头建立动机。

---

## 1. Preface (前言,替代原书 Preface)

### 1.1 谁写的这本书

**Robert J. Chassell** (1947-2017),FSF 的早期成员之一,Stallman 的同事。
他写了这本《Programming in Emacs Lisp: An Introduction》,目标是**让非程序员也能学会 Elisp**。

这本书的风格非常特殊:
- 每个概念都从最简单的例子开始
- 大量重复 (好的重复,加强记忆)
- 用真实 Emacs 命令 (如 `mark-whole-buffer`、`append-to-buffer`) 作为例子
- 每章有练习题

**版本说明**: 你目录里的版本是 30.2 (2025 年更新)。原书 1990 年代出版,内容核心未变。

### 1.2 为什么读这本书

不读 Chassell 直接读 Emacs Lisp Reference Manual 会怎样?
- Reference Manual 假设你懂 Lisp
- 没有循序渐进的例子
- 1100 页,密集术语,新手会淹没

Chassell 是**桥梁**: 它从"完全不懂 Lisp"到"能读 Reference Manual"。

Module 3 会系统读完整本书 (~5 周)。Module 0 你只读开头建立动机。

#### 深入: 为什么 Lisp 入门书这么少

考虑市面上 Lisp 书籍:
- 《ANSI Common Lisp》Paul Graham — 给程序员,假设懂编程
- 《SICP》 (MIT 6.001) — 用 Scheme 教计算机科学,不是入门
- 《Practical Common Lisp》— 偏实战
- 《Land of Lisp》— 用游戏教,有趣但浅

**几乎每本 Lisp 书都假设读者是程序员**。Chassell 是**少数为非程序员写的 Lisp 书**,而且写得非常好。这是为什么它是 Emacs 社区的"圣经"之一。

#### 反事实: 如果不读 Chassell 直接学 Lisp

你可能在几天内学会 Lisp 语法 (`(+ 1 2)`、`(defun foo () ...)`),但你会:
- 不理解**为什么** Emacs 是这样设计的
- 不知道怎么把"我想做的"翻译成"Elisp 怎么写"
- 缺乏对 docstring、hook、keymap、buffer-local 这些 Emacs 特有概念的直觉

Chassell 不只是教语法,它教**Emacs 思维方式**。这就是为什么 Module 3 花整整 5 周读它。

---

## 2. Why You Should Learn Emacs Lisp (替代 Why 章节)

### 2.1 三个理由

**理由 1: 自定义和扩展 Emacs**

Emacs 默认行为可以被任何人改变。但你**只能通过 Elisp** 改变它。
- 想让某个键做别的事? Elisp
- 想加一个新的命令? Elisp
- 想让 Emacs 在保存文件时自动跑 lint? Elisp
- 想让一个包和另一个包配合? Elisp

不会 Elisp → 你的 Emacs 只能用别人写好的。
会 Elisp → 你的 Emacs 完全为你定制。

**理由 2: 理解 Emacs 本身**

`C-h f` 给你函数文档,但**真正的逻辑在源码里**。
不会 Elisp,你看不懂源码,你只能"用"。
会 Elisp,你**看任何 Emacs 行为都明白底层发生了什么**。

**理由 3: 自动化日常任务**

```elisp
;; 一键插入当前日期
(defun insert-date ()
  (interactive)
  (insert (format-time-string "%Y-%m-%d")))

;; 一键替换某个 buffer 中所有 tab 为空格
(defun my-tabs-to-spaces ()
  (interactive)
  (save-excursion
    (goto-char (point-min))
    (while (re-search-forward "\t" nil t)
      (replace-match "    "))))
```

这种小工具,会 Elisp 后 5 分钟就能写一个。

#### 深入: 学 Elisp 的"复利效应"

考虑学习曲线:
- 第 1 周: 学会基本语法,能改一个键位
- 第 1 个月: 能写 20 行的函数,自动完成日常小任务
- 第 6 个月: 能写 100 行的"工作流",把多个包串起来
- 第 1 年: 能写自己的 minor mode,发布到 MELPA
- 第 2 年: 你不再"用 Emacs",你"住在自己定制的 Emacs"

这种"复利"是其他编辑器做不到的。VS Code 你能写扩展,但:
- 写扩展门槛高 (TypeScript、API、打包)
- 扩展不能直接调 VS Code 内部状态
- 调试要 launch extension host

Emacs 让你**从第一周就开始攒复利**——每一行 init.el 都是你的资产,且这些资产**永不作废** (Emacs 配置 30 年前写的今天还能跑)。

#### 创造性应用: Elisp 能做的"啊哈时刻"

**啊哈 1: 自动化论文写作**

```elisp
(defun my/org-to-arxiv ()
  "把当前 org 文件编译成 arXiv 提交格式。"
  (interactive)
  (org-latex-export-to-latex)
  (let ((tex-file (concat (file-name-base) ".tex")))
    (shell-command (format "latexmk -pdf %s" tex-file))))
```

整个论文工作流: 写、改、引用、图、表、编译、提交——全在 Emacs 里。

**啊哈 2: 自动化博客发布**

写 org-mode,`C-c C-e` 导出 HTML,用 Elisp 自动 push 到 GitHub Pages。从写到发布不离开 Emacs。

**啊哈 3: 用 Elisp 改变 Emacs 行为来适应你的工作**

```elisp
(defun my/save-and-compile ()
  "保存时自动编译 (只在 C 文件里)。"
  (when (and (buffer-file-name)
             (string-match "\\.c$" (buffer-file-name)))
    (compile (format "gcc %s -o /tmp/a.out" (buffer-file-name)))))
(add-hook 'after-save-hook 'my/save-and-compile)
```

每次保存 C 文件自动编译——但只对 C 文件,不影响其他。**精确控制**是 Elisp 的标志。

### 2.2 Chassell 的动机

Chassell 在书里说:
> "Many people think that learning to use a computer is inherently difficult...
> They think that the difficulty is intrinsic to computers.
> This is not so. Computers are tools. Tools are easy when you know how to use them."

> "很多人觉得用电脑很难,以为难度是电脑本身的。
> 错。电脑是工具。你知道怎么用,就不难。"

学 Elisp 不是为了"成为程序员",是为了**把电脑变成你的工具**。

#### 深入: Chassell 这段话的历史背景

1985 年,个人电脑刚刚普及。普通人第一次接触电脑,面对的是 DOS 命令行、WordStar、Lotus 1-2-3 这些"难用"的软件。当时的软件文化是"专家设计,普通人学"。

Chassell 这段话是对那个时代的反抗——他认为**电脑应该服从于人,而不是人服从于电脑**。学 Lisp 不是"成为程序员",是"夺回对自己工具的控制权"。

这个理念在 2026 年依然有力量——现代 SaaS 软件让用户更加被动 (你的数据在云端,你不能改软件行为)。学 Elisp 是**反潮流**——你拥有自己的工具,数据在本地,行为完全可控。

#### 反事实: 如果你不学 Elisp,会发生什么

- 你只能用别人写好的包,出问题时只能等修复
- 你的工作流受限于包作者的设计
- 你的"个性化"只到"开关某个选项"
- 5 年后你可能因为 Emacs 太"难用"而切回 VS Code——但你**永远不知道它真正能做什么**

学 Elisp 后:
- 你的工作流由**你自己**塑造,任何小需求都能实现
- 你的"个性化"达到"行为级"——任何按键、任何 hook、任何时机都能改
- 5 年后你的 Emacs 是你**独一无二的工具**,无人能复制

---

## 3. On Reading this Text (替代 On Reading this Text)

### 3.1 Chassell 的读书建议

> 这些建议是 Chassell 给读者的,但同样适用于我们这个教程。

1. **不要只读,要操作**
   - 每个 elisp 例子,在 `*scratch*` 里**实际 eval 一遍**
   - 改改参数,看结果变化
   - 不动手 = 没学会

2. **遇到不懂的术语,跳过去**
   - 第一遍不求"完全理解"
   - 读到后面,前面不懂的自然懂了
   - Lisp 是高度递归的语言,概念相互依赖

3. **重读**
   - Lisp 的概念需要多次接触才内化
   - 一遍不够,Module 6 还会再过一遍

4. **做练习**
   - 每章末尾的练习,自己做一遍
   - 做不出的看答案,理解后**合上答案再做一遍**

### 3.2 我们教程的额外建议

5. **每学一个概念,写在自己的 init.el 里**
   - 学到 `setq`,init.el 加一行 `(setq line-number-mode t)`
   - 学到 `defun`,init.el 加一个自己的函数
   - **配置是你的"产出"**

6. **每周写一篇学习日志** (`logs/`)
   - 用自己的话复述学到的
   - 解释为什么这个概念重要
   - 写不出来的 = 没真懂

7. **遇到 Emacs 行为不理解,用 Elisp 探索**
   - 看到 `(message "hello")` 不知道干啥?
   - 在 `*scratch*` 里 eval 一遍
   - 看 `*Messages*` buffer,看到 "hello"
   - 用 `C-h f message RET` 查文档

---

## 4. Who You Are (替代 Who You Are)

### 4.1 Chassell 假设的读者

> "You may be a non-programmer who wants to learn how to use Emacs...
> You may be a programmer who has never seen Lisp before..."

Chassell 假设的读者光谱:
- 完全不懂编程,只想自定义 Emacs
- 懂其他语言,不懂 Lisp
- 懂 Lisp (其他方言),不懂 Elisp

这本书对所有人都适用,因为**它从零开始**。

### 4.2 你的位置

根据你的初始状态:
- Emacs 经验: 当记事本用 (基本零基础)
- 编程经验: 有基础但不熟练
- Lisp 经验: 几乎无

**好消息**: Chassell 完全适合你。
Module 3 的 5 周精读会**让你建立 Lisp 思维**,不需要任何前置。

### 4.3 心态调整

新手学 Lisp 常见的卡点:
- "为什么这么多括号?" — 括号是 Lisp 的语法,你会爱上它的统一性
- "为什么不写 return?" — Lisp 的每个 form 自动返回值
- "为什么变量不用声明类型?" — Lisp 是动态类型
- "为什么递归这么常用?" — Lisp 鼓励递归思维

Chassell 会逐一解答这些卡点。Module 3 会系统学。

---

## 5. Lisp History (替代 Lisp History 章节)

### 5.1 Lisp 的诞生

**1958 年**, **John McCarthy** 在 MIT 发明了 Lisp (LISt Processing)。
这是**第二古老的高级语言** (仅次于 Fortran,1957)。

McCarthy 的设计灵感:
- **List Processing**: 一切是 list
- **Symbolic computation**: 处理符号,不只是数字
- **Homoiconicity**: 代码即数据 (代码本身是 list)
- **Garbage collection**: McCarthy 也是 GC 的发明者

### 5.2 Lisp 的演化

```
1958   LISP (McCarthy, MIT)
   │
   ├── 1960s  Lisp 1.5 (MIT)
   │
   ├── 1970s  MacLisp (MIT)
   │       │
   │       └── 1975  Scheme (Sussman + Steele)
   │
   ├── 1970s  Interlisp (BBN)
   │
   ├── 1980s  Common Lisp (标准化,Zetalisp + MacLisp 合并)
   │
   ├── 1985   Emacs Lisp (Stallman, GNU Emacs 的脚本语言)
   │
   ├── 1990s  Clojure (Hickey,2007,hosted on JVM)
   │
   └── 2000s  Racket (PLT,Scheme 的进化)
```

### 5.3 为什么 Emacs Lisp 长得不一样

Emacs Lisp 是 MacLisp 的"简化版":
- **Lisp-2**: 变量和函数有独立命名空间 (Common Lisp 也是 Lisp-2;Scheme 是 Lisp-1)
- **动态作用域** (历史遗留,Emacs 25+ 支持 lexical)
- **没有包系统** (用一个全局 `obarray`,所有 symbol 共享)
- **没有 reader macros** (不能像 Common Lisp 那样改语法)
- **简单到 Stallman 一个人能写完** (1985)

这些限制让 Elisp 不像 Common Lisp 那么"强大",但**简单到任何用户都能学**。

### 5.4 Lisp 的文化

Lisp 程序员有一个经典笑话:
> "In Lisp, the program is data, and data is the program."
> (Lisp 里,程序就是数据,数据就是程序)

这叫 **homoiconicity** (同像性),是 Lisp 的本质特征。

普通语言:
```
代码:  print("hello")
数据:  "hello"
```
代码和数据是分开的。

Lisp:
```
代码:  (print "hello")
数据:  (print "hello")  ;; ← 这是一个 list,第一个元素是符号 print,第二个是字符串
```
代码和数据**长得一样**。

这是为什么 Lisp 这么适合做宏 (macro)——你可以**写代码生成代码**,因为代码就是 list。

#### 深入: Homoiconicity 在 Emacs 里的具体体现

考虑这个真实场景: 你想定义一组类似模式的函数。

普通语言 (Python):
```python
def save_doc(): ...
def load_doc(): ...
def delete_doc(): ...
# 每个函数手写
```

Emacs Lisp 用宏:
```elisp
(defmacro my/define-file-op (name action)
  `(defun ,(intern (format "my-%s-file" name)) ()
     ,(format "Perform %s on current file." action)
     (interactive)
     (message "%s: %s" ,action (buffer-file-name))))

(my/define-file-op save "saving")
(my-define-file-op load "loading")  ; 注意: 调用 macro 用名字,不是引号
```

宏让你**写代码生成代码**。这在 Python/JS 里要么做不到,要么要用很丑的字符串拼接。

#### 创造性应用: 宏的"啊哈时刻"

**啊哈 1: 一个宏定义十个相似函数**

```elisp
(defmacro my/define-mode-hook (mode &rest body)
  `(add-hook ',(intern (format "%s-hook" mode))
             (lambda () ,@body)))

(my/define-mode-hook python-mode
  (setq indent-tabs-mode nil)
  (flycheck-mode 1))
```

这是 Lisp 的"元编程"——你不再写程序,你写**生成程序的程序**。

**啊哈 2: 自定义 DSL (领域特定语言)**

很多 Emacs 包都用宏定义自己的 DSL:
- `use-package` 是一个宏,看起来像配置,实际是生成大量 elisp
- `org-mode` 的 `org-map-entries` 是宏,让你像 SQL 一样查 org 文件
- `cl-defun` 是宏,提供 Common Lisp 风格的参数列表

这是普通配置 (JSON/YAML) 永远做不到的——JSON 是数据,不能"生成代码"。

### 5.5 经典 Lisp 应用

- **Emacs** (编辑器)
- **Autodesk AutoCAD** (绘图)
- **Grammarly** (写作辅助,Clojure)
- **Walmart pricing** (Clojure)
- **NASA Mars rover** 部分代码
- **Paul Graham 的 Viaweb** (第一个 SaaS,1995,Common Lisp)

Lisp 不是 dead 语言,它在某些领域 (AI、金融、复杂系统) 仍是首选。

#### 深入: 为什么高级程序员喜欢 Lisp

Paul Graham 的著名文章《Beating the Averages》讲了一个故事: 他的创业公司 Viaweb (1995,第一个 SaaS) 用 Common Lisp,**竞争对手用 C++**。结果是:
- Viaweb 1 个开发者 = 竞争对手 5-10 个开发者的产出
- 新功能 1 天上线,竞争对手 1 个月
- 最终 Yahoo 收购 Viaweb,变成 Yahoo Store

为什么 Lisp 这么高效? 因为它的**抽象层次更高**——你能用 100 行 Lisp 表达 1000 行 C++ 的功能,且更易读、更易改。

学 Elisp 不是"学个小语言",是**学一种强大的思维方式**。一旦掌握,你看其他语言会自动用 Lisp 思维审视——哪些是真正必要的,哪些是冗余的语法糖。

#### 反事实: 如果当年 Viaweb 用 Python 而不是 Lisp

1995 年 Python 还很年轻 (1991 发布),Rails 还没出现 (2004)。如果 Graham 用 Python:
- 可能也能快速开发,但不如 Lisp 灵活 (Python 没有宏)
- 代码会更长、抽象层次更低
- Yahoo Store 可能不是 1998 年收购,而是 2002 年

这是历史 fork 点——一个工具选择改变了创业公司的命运。

### 6. Note for Novices (替代 Note for Novices)

### 6.1 常见新手错误

Chassell 列出的新手坑,我加了几条:

1. **括号不配对**
   - Lisp 的括号必须严格匹配
   - Emacs 的 `show-paren-mode` 帮你高亮配对
   - 加到 init.el: `(show-paren-mode 1)`

2. **quote 忘了**
   - `(list a b)` 把 a 和 b 当变量 eval
   - `(list 'a 'b)` 把 a 和 b 当符号
   - 这是新手最常犯的错

3. **空格问题**
   - `(+ 1 2)` 对,`(+ 1 2)` 也对
   - `(+1 2)` 错 (被当成正数 +1)
   - Lisp 对空格很敏感

4. **大写小写**
   - Elisp 默认大小写敏感 (`foo` 和 `Foo` 是不同的符号)
   - 但 `setq` 和 `SetQ` 不同 — 你应该全用小写

5. **不读错误信息**
   - Lisp 报错时,error message 很有信息
   - 新手习惯按 `C-g` 跳过,**应该读**
   - 例: `wrong-number-of-arguments` 告诉你参数数不对

6. **不查类型**
   - `(+ 1 "hello")` 报错 — string 不能加到 number
   - 用 `type-of` 函数看类型
   - 用 `C-h f` 看函数期望什么类型

### 6.2 调试策略

Lisp 比 C/Java **更容易调试**:
- 没有编译-链接-运行的循环
- 用 `C-x C-e` 逐段 eval,定位问题
- `M-x edebug` 可以单步
- `message` 函数插入 print 调试

Module 3 第 17 章 (Debugging) 会系统讲。Module 7 还会深入。

#### 深入: 为什么 Lisp 比 Java/C 更易调试

考虑调试一个 Java 程序:
1. 写代码
2. `javac` 编译
3. 启动 JVM
4. 用 IDE 连接 debugger
5. 设断点
6. 重复运行到断点
7. 改代码,重新编译,重启 JVM

整个循环 30 秒到几分钟。**改一个值要重启整个应用**。

Lisp 调试:
1. 写代码
2. `C-x C-e` 立即 eval
3. 出错了,看错误信息 (Lisp 错误信息很详细)
4. 改代码,再 `C-x C-e`
5. 不重启,所有 buffer 状态保留

整个循环 5 秒。**这正是为什么 Emacs 用户"住在 Emacs 里"——他们的工作流是连续的**。

#### 创造性应用: 调试的"啊哈时刻"

**啊哈 1: `M-x edebug` 函数级单步**

把光标放在某个 defun 内,`M-x edebug-defun`。下次调用这个函数,Emacs 进入单步模式——你看到每一行执行,可以查看任意变量值,可以跳过、可以倒退。这是 IDE 级调试,但**不需要 IDE**。

**啊哈 2: `M-x toggle-debug-on-error`**

打开后,任何 error 触发时,Emacs 弹出 backtrace buffer——你能看到完整的调用栈和每一层的局部变量。一眼定位问题。

**啊哈 3: 在生产环境调试**

你的 Emacs 已经运行了 3 天,某个 hook 函数突然报错。你**不需要重启**——直接读 backtrace,改函数,`eval-defun`,问题立即修复,所有状态保留。这是 hot patching 在编辑器里的应用。

---

## 7. Thank You (替代 Thank You 章节)

### 7.1 Chassell 的致谢

Chassell 在书末感谢了一堆人,关键的:
- **Richard Stallman**: Emacs 的创造者,Chassell 的同事
- **MIT AI Lab**: Lisp 文化的发源地
- **FSF (Free Software Foundation)**: 资助这本书的写作

### 7.2 你应该感谢的人 (当你学完时)

学完 Module 6 后,你应该意识到 Emacs 生态是**几十年累积的礼物**:
- RMS 写了核心
- 数百个贡献者写了 mode、package
- Chassell 写了入门书
- Stefan Monnier、Eli Zaretskii 等维护者持续 review

回报的方式:
- **使用** 它 (用户基数本身就是力量)
- **写包** (Module 7)
- **贡献** (Module 8)
- **教别人** (写博客、做分享)

---

## 8. 总结: 你现在应该建立的认知

读完这个 intro-reading 后,你应该知道:

1. **Chassell 是谁,这本书为什么适合你**
2. **学 Elisp 的三个理由**: 自定义、理解、自动化
3. **如何读这本教程**: 操作 + 重读 + 做练习 + 写 init.el + 写日志
4. **Lisp 的简史**: McCarthy 1958 → MacLisp → Emacs Lisp 1985
5. **Lisp 的核心特征**: homoiconicity (代码即数据)
6. **新手的常见坑**: 括号、quote、空格、大小写、错误信息
7. **回报社区的方式**: 用、写包、贡献、教

---

## 9. 下一步

回到 `00-mindset/README.md`,继续 Module 0 的剩余部分 (编译安装 + init.el)。

或者直接打开 `00-mindset/exercises.md` 做这个模块的练习。
