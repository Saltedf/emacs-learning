# Org-mode 详解

> 学完这个文件,你能用 Org 做笔记 / TODO / 日历 / 文档 / 编程

Org-mode 是 Emacs 的"杀手级应用"——很多用户从其他编辑器切到 Emacs,就是为了 Org。这种吸引力不来自花哨的功能,而来自 Org 的**统一哲学**: 把笔记、待办、日历、文档、代码都放在一个**纯文本**格式里。

这一节将深度讲解 Org——不只是按键,而是为什么 Org 这么设计,以及它的创造性用法。读完之后,你应该能用 Org 替代 Notion、Obsidian、Todoist、Trello、Word、Excel——不是模仿它们,而是用 Org 的方式做这些事。

---

## 1. Org 是什么

### 1.1 一句话

**Org-mode 是 Emacs 的杀手级应用**。

它是:
- 纯文本格式 (.org)
- 大纲编辑器
- TODO 管理
- 日历 + 计划
- 表格计算
- 文档编写
- 代码执行 (literate programming)
- ...

这个列表看起来是"很多功能"——但 Org 的真正力量是**这些功能共享同一个文件格式**。一个 .org 文件可以同时是项目笔记、TODO 列表、代码文档、数据报表。这种"统一"是其他工具无法做到的——Notion 把笔记和 TODO 分开 (databases vs pages),Roam 只做笔记,Todoist 只做 TODO。Org 全做。

### 1.2 vs Markdown

这是最常见的对比——很多用户问"为什么不用 Markdown + pandoc?"。答案在于"功能边界":

| | Org | Markdown |
|---|---|---|
| 大纲 | 原生 (folding) | 用 # (没原生) |
| TODO | 原生 (TODO/DOING/DONE) | 需扩展 |
| 表格 | 强 (含公式) | GFM 表格 (无公式) |
| 代码执行 | babel | 没有 |
| Capture | 原生 | 没 |
| Agenda | 原生 | 没 |
| 输出 | HTML/LaTeX/PDF | pandoc 任意 |

Org 是**整套工作流**,Markdown 是**文档格式**。

这个区别决定了用途: Markdown 适合"写一篇文档" (README、博客),Org 适合"管理工作" (笔记 + TODO + 日程 + 项目)。你可以用 Markdown 写 TODO 列表 (GFM 有 `- [ ]` 语法),但 Markdown 没有 agenda (跨文件视图)、没有 capture (快速输入)、没有 repeater (重复任务)——这些才是"工作流"的核心。

一个更哲学的说法: **Markdown 是给人看的,Org 是给工具用的**。Org 的元数据 (TODO keyword、SCHEDULED、DEADLINE、properties) 是**机器可解析**的——这就是为什么 Org 能做 agenda (扫描所有文件,提取时间信息)。Markdown 没有这种结构化元数据,所以做不出 agenda。

### 1.3 历史

Org-mode 由 Carsten Dominik 在 2003 年创建。他本来想解决自己的 GTD (Getting Things Done) 工作流——他需要一种"随时记录、统一查看"的系统,但当时的工具 (Outlook、纸笔、独立 TODO 应用) 都不够灵活。他在 Emacs 的 outline-mode 之上加了一层"TODO + 时间"——这就是 Org 的原型。

Org 的核心思想"纯文本 + 元数据"影响了无数工具 (Markdown task lists, Notion, Logseq, Roam),但 Org 仍然是功能最完整的。这反映了开源工具的力量——一个社区志愿者 (Carsten + 后来的 Bastien Guerry + 几百贡献者) 做的工具,功能超过商业软件。

2003 年 Carsten Dominik 创建。Bastien Guerry 在 2011 年接管维护,持续到今天 (2026)。Emacs 内置 (从 Emacs 22 起)。Org 现在是一个独立的 GNU 项目,有自己的 release 周期 (与 Emacs 不同步)。

---

## 2. 基础语法

### 2.1 Headings

Org 的核心是"大纲"——一个 .org 文件就是一棵树,每个节点是一个 heading。这种树状结构来自 Emacs 的 outline-mode,但 Org 加了大量语义 (TODO、properties、tags)。

```org
* Top level
** Second level
*** Third level
* Another top
```

`*` 数量决定级别。Org 的级别无上限,但实际上 4-5 级就够了——太深的大纲难以阅读。

这种"树状 + 缩进"的设计,让 Org 文件天然适合"层级信息"——文档章节、项目分解、思维导图。每个 heading 是一个"折叠单元",你可以只看顶层 (`S-TAB` 全折叠),也可以深入到细节 (`TAB` 展开)。

### 2.2 Folding

折叠是 Org 的核心交互——你**不需要看整个文件**,只看当前关心的部分。这让一个几千行的 Org 文件仍然可以高效导航。

```
TAB          折叠/展开当前
S-TAB        全部折叠/展开
C-u C-u TAB  全部展开 (强制)
```

`TAB` 是 context-sensitive——根据光标位置,它折叠/展开当前 heading。`S-TAB` 操作整个 buffer——循环切换"全展开 / 只看顶级 / 全折叠"。这是 Org 的"鸟瞰模式"——快速看文件结构。

### 2.3 Lists

当不想要 heading 的"重量" (heading 有 TODO、property 等语义),用 list:

```org
- item 1
- item 2
  - sub 1
  - sub 2
- item 3

1. first
2. second
3. third
```

List 比 heading 轻——它就是缩进的文本。适合简单的列举 (购物清单、步骤)。

### 2.4 强调

Org 的强调记号比 Markdown 多——这是 Org 为"长文档"设计的体现:

```org
*bold*
/italic/
_underline_
=verbatim/
~code~
+strike-through+
```

注意 Org 有 `_underline_` 和 `+strike-through+`——Markdown 没有。这反映了 Org 的目标用户 (写论文、写书) 比 Markdown 的目标用户 (写 README) 需要更多排版工具。

### 2.5 Links

Org 的 link 系统是它最强的功能之一——一个 link 可以指向**任何东西**: URL、本地文件、特定 heading、Org-roam 节点、邮件、IRC 消息。这是 Org 成为"信息枢纽"的基础。

```org
[[https://example.com][description]]
[[file:path/to/file][file]]
[[id:UUID][org-roam link]]
[[mailto:foo@bar.com]]
```

`C-c C-l` 创建/编辑 link。

第三种 `[[id:UUID][...]]` 是 Org-roam 的核心——它用 UUID (而不是文件路径) 标识一个节点。这意味着你可以**移动文件**而不破坏链接——UUID 是稳定的。这是 Org-roam 优于普通 wiki link 的地方。

### 2.6 代码块

Org-babel 是 Org 最让人惊喜的功能——它让 Org 成为"可执行文档"。你写一段代码块,按 `C-c C-c` 在 Org 里**直接执行**,结果嵌回文档。这就是 literate programming (Donald Knuth 1984 提出) 的实现。

````org
#+begin_src python
def hello():
    print("Hello")

hello()
#+end_src

#+RESULTS:
: Hello
````

`C-c C-c` 在代码块上执行 (要 org-babel 配置)。

这种"文档 + 代码 + 结果一体"的模式,对于数据科学、技术博客、配置文件都极其强大——你不再需要在文档和代码之间切换,它们就是一体的。后面 Babel 一节会详细讲。

---

## 3. TODO 管理

### 3.1 基本 TODO

Org 的 TODO 管理是最被模仿的功能——几乎所有现代笔记应用 (Todoist、Things、TickTick) 都借鉴了 Org 的 keyword 设计。但 Org 仍然是这些工具的"祖宗"。

```org
* TODO Buy milk
* DONE Submit report
* IN-PROGRESS Write article
```

`S-LEFT` / `S-RIGHT` 切换 TODO keyword。

关键设计: TODO 是 heading 的**前缀**,不是独立项。这意味着 TODO 继承 heading 的所有特性——可以折叠、可以嵌套、可以有 property、可以加 tag。这比 Todoist 的"独立 TODO 列表"灵活得多——你的 TODO 可以在一个项目大纲里,而不是孤立的列表。

### 3.2 自定义 keywords

Org 允许你完全自定义 workflow——这是它的杀手锏。你可以定义自己的状态序列,反映你的真实工作流。

```elisp
(setq org-todo-keywords
      '((sequence "TODO(t)" "IN-PROGRESS(p)" "|" "DONE(d!)" "CANCELED(c@)")))
```

这个配置定义了一个 workflow: TODO → IN-PROGRESS → (done state) DONE 或 CANCELED。

`|` 分隔 active 和 done states——左边是"进行中"的状态,右边是"终态"。这个区分影响 agenda 视图 (只显示 active TODO) 和 logging (进入终态会记录时间戳)。

`t/p/d/c` 是快捷键——你按 `C-c C-t t` 直接设为 TODO,不需要循环。这对于"批量修改状态"非常有用。

`!` 记录时间戳,`@` 记录 note——这是 GTD 的"CLOSED"机制。进入 DONE 时自动记录 "CLOSED: [2026-06-20 Fri 10:00]",进入 CANCELED 时弹 prompt 让你写原因。这是审计和回顾的基础。

### 3.3 Priority

Priority 是 TODO 的可选属性——`[#A]` 最高,`[#C]` 最低。agenda 按 priority 排序。

```org
* TODO [#A] Important task
* TODO [#B] Normal task
* TODO [#C] Low priority
```

`S-UP` / `S-DOWN` 改 priority。

Priority 的设计哲学: **不要太多级别**。A/B/C 三级够了——更多级别意味着你在"决定 priority"上花太多时间,违反 GTD 原则。

### 3.4 Tags

Tags 是"横向分类"——heading 的层级是纵向的 (树状),tags 是横向的 (跨树)。一个 TODO 可以属于"工作"树,但 tag 为 "urgent"。

```org
* TODO Buy milk :shopping:errand:
```

`C-c C-q` 加 tag。

Tags 是 agenda 过滤的基础——`C-c a m` (match) 可以按 tag 查询所有 TODO (跨文件)。比如 "找所有 tag 为 urgent 的 TODO"——一行命令。

### 3.5 Properties

Properties 是 key-value 元数据——你可以给 heading 加任意属性。这让 Org 成为轻量级数据库。

```org
* TODO Buy
:PROPERTIES:
:Effort:   30m
:Assigned: Alice
:END:
```

`C-c C-x p` 加 property。

Property 的强大在于它**可被查询**——你可以在 agenda 里用 "Effort < 30m" 过滤所有 TODO。这让 Org 可以做项目管理 (估时、分配)、CRM (客户信息)、bug tracker (priority、severity) 等。

### 3.6 Date/Time

时间是 Org TODO 的灵魂——没有时间的 TODO 只是"待办",有时间才是"计划"。

```org
* TODO Meeting
SCHEDULED: <2026-06-20 Mon 14:00>
DEADLINE: <2026-06-25>
```

`C-c C-s` schedule
`C-c C-d` deadline

SCHEDULED 和 DEADLINE 的区别非常重要:
- **SCHEDULED**: 你**计划**什么时候做这个任务。Agenda 在那天显示。
- **DEADLINE**: 任务**必须**在这天之前完成。Agenda 在临近时高亮警告 (默认提前 14 天)。

这个区分反映了真实工作——"周三开会" (SCHEDULED) 和"周五截止" (DEADLINE) 是不同的事。Org 把它们分开,agenda 才能正确显示。

### 3.7 Repeater

Repeater 是 Org 处理"重复任务"的机制——每天站会、每周回顾、每月付租金。这些任务你不能每次重新创建——Org 让你设一次,repeater 自动推进。

```org
* TODO Daily standup
SCHEDULED: <2026-06-20 Mon +1d>      ; 每天

* TODO Weekly review
SCHEDULED: <2026-06-20 Mon +1w>      ; 每周

* TODO Pay rent
SCHEDULED: <2026-06-20 Mon +1m>      ; 每月
```

`+1d` 的含义: "完成此任务后,自动 schedule 到 +1 天后"。当你按 DONE,repeater 自动把它改回 TODO 并推进时间。这是 Org 最优雅的设计之一——重复任务**永远只占一行**。

---

## 4. Capture

### 4.1 模板

Capture 是 Org 改变生活的功能——它让你"零摩擦"地记录任何东西。没有 capture,你要: 打开 org 文件 → 找到位置 → 输入 → 保存。有 capture,你只要: `C-c c t` → 输入 → `C-c C-c`。三步变两步,而且不需要"切换上下文"——你在写代码时想到一件事,`C-c c` 记下来,立刻回到代码。

模板让 capture 适应不同场景——TODO、note、journal、link,各有各的目标文件和格式:

```elisp
(setq org-capture-templates
      '(("t" "Todo" entry (file+headline "" "Inbox")
         "* TODO %?\n  %i\n  %a")
        ("n" "Note" entry (file "" )
         "* %?\n  %U\n")
        ("j" "Journal" entry (file+datetree "journal.org")
         "* %?\n  %U\n  %i")
        ("l" "Link" entry (file+headline "" "Links")
         "* %a\n  %?")))
```

模板的语法关键是 **placeholder**——`%?`、`%a`、`%U` 在 capture 时被替换为实际内容:

`%?` 是光标最终位置——capture 打开后,光标自动停在这里,你立刻可以输入。这是"零摩擦"的关键。
`%a` 是 annotation (来自 link/region)——如果你在浏览器复制了一个 URL,或者选中了一段文字,`%a` 会插入它。
`%U` 是时间戳——自动加上当前时间。这对于 journal、log 类的 capture 必不可少。

这种"模板 + placeholder"的设计,让 capture 既能"无脑快速记" (TODO),也能"结构化记录" (journal with date)。一个键位 (`C-c c`),适应所有场景。

### 4.2 调用

```
C-c c t    ; 创建 todo
C-c c n    ; 创建 note
C-c c j    ; 写日记
```

按 `C-c c` 后,弹出模板选择菜单。选一个 (按对应字母),capture buffer 打开。输入内容,`C-c C-c` 保存到目标文件,`C-c C-k` 取消。

### 4.3 例子

这是一个完整的 Org 配置——把 capture 绑到 `C-c c`,定义目录、默认文件、几个模板:

```elisp
(use-package org
  :bind (("C-c c" . org-capture))
  :custom
  (org-directory "~/Dropbox/org/")
  (org-default-notes-file (concat org-directory "inbox.org"))
  (org-capture-templates
   `(("t" "Todo" entry (file+headline ,org-default-notes-file "Tasks")
      "* TODO %?\n  %i\n")
     ("w" "Work Todo" entry (file+headline "work.org" "Tasks")
      "* TODO %?\n  %i\n  %a")
     ("j" "Journal" entry (file+datetree "journal.org")
      "* %?\n  Entered on %U\n  %i"))))
```

这个配置体现了"工作流分离"——个人 TODO 进 inbox.org,工作 TODO 进 work.org,journal 进 journal.org 的 datetree (按日期组织的树)。同一个 `C-c c` 入口,根据模板分流到不同位置。

`file+datetree` 是一个特别的 target——它会自动创建 (或找到) 当年的"年 → 月 → 日"树,把 entry 插到今天。这就是 journal 的工作方式——每天一个 heading,自动归档。这种"自动结构化"是 Org 比简单日记应用强的地方。

`Dropbox/org/` 目录的选择有讲究——把 org 文件放 Dropbox/iCloud/同步盘,你可以在多设备 (电脑、手机) 访问。手机上的 org-mode 客户端 (beorg, Orgro) 让你随时随地 capture。

---

## 5. Agenda

### 5.1 视图

Agenda 是 Org 的"统一视图"——它扫描所有 `org-agenda-files` 里的 .org 文件,把 TODO、时间、deadline 聚合到一个视图。这是 Org 替代 Todoist、Things 等 TODO 应用的核心。

```
C-c a a    agenda (本周)
C-c a t    todo list
C-c a m    match tag
C-c a s    search
```

`C-c a a` 显示本周日程——所有 SCHEDULED、DEADLINE、timed entry 按日期排列。`C-c a t` 显示所有 TODO (跨文件)。`C-c a m` 按 tag 过滤 (e.g., "work+urgent")。`C-c a s` 全文搜索。

这种"一个视图看到所有 TODO"是 Org 的核心价值——你不需要在每个 org 文件之间切换,agenda 把它们统一。

### 5.2 在 agenda 里

Agenda buffer 有自己的键位——你在里面可以**操作**TODO,不只是看:

```
f / b     下/上周
.         今天
d         day view
w         week view
m         month view
y         year view
SPC       在原文件光标位置
TAB       在另一个 window 打开
RET       打开
```

更重要的是,agenda 里可以直接改 TODO 状态——按 `t` 切换 TODO keyword,按 `,` 设 priority,按 `+`/`-` 调 schedule date。这意味着你**不需要打开原文件**就能管理任务。这是"批量操作"的体现——在 agenda 里扫一遍,标记完成的、推迟的、加 tag 的,几分钟搞定一周计划。

### 5.3 配置

这是一个完整的 agenda 配置——展示如何自定义视图:

```elisp
(use-package org
  :bind (("C-c a" . org-agenda))
  :custom
  (org-agenda-files (list org-directory))
  (org-agenda-start-on-weekday 1)        ; 周一开始
  (org-agenda-span 14)                   ; 显示 2 周
  (org-agenda-include-diary t)
  (org-agenda-time-grid
   '((daily today require-timed)
     (800 1000 1200 1400 1600 1800 2000)
     "......" "----------------"))
  (org-agenda-custom-commands
   '(("h" "Home"
      ((agenda "")
       (tags "home")
       (todo "TODO")))
     ("w" "Work"
      ((agenda "")
       (tags "work"))))))
```

`org-agenda-files` 是关键——它告诉 agenda 扫描哪些文件。设为 `(list org-directory)` 意味着整个 org 目录都被扫描。这意味着**你不需要手动"注册"文件**,放进目录就自动加入 agenda。

`org-agenda-custom-commands` 是 agenda 的杀手锏——你可以定义"复合视图"。上面定义了 `h` (Home) 和 `w` (Work) 两个视图,每个包含多个 block (agenda + tags + todo)。按 `C-c a h` 显示家里所有事 (本周日程 + 家相关 tag + 家相关 TODO)。这种"上下文切换"让你在不同角色之间快速切换。

---

## 6. Tables

### 6.1 创建

Org table 是 Org 最被低估的功能——很多人不知道 Org 可以当 Excel 用。一个 .org 文件可以同时是文档和报表。

```org
| Name | Age |
|------|-----|
| Alice | 30 |
| Bob   | 25 |
```

输入 `|Name|Age|` 然后 `TAB`,自动格式化。

这个"输入 | 自动格式化"的设计非常优雅——你不需要画表格边框,Org 替你画。每次按 TAB,列宽自动调整对齐。这让 Org table 比 Markdown table 易写得多 (Markdown table 必须手动对齐)。

### 6.2 操作

Org table 的导航和编辑键位非常多——它本质上是一个"小型电子表格":

```
TAB          next field (auto-format)
S-TAB        prev field
M-LEFT       move column left
M-RIGHT      move column right
M-UP         move row up
M-DOWN       move row down
M-S-LEFT     delete column
M-S-RIGHT    insert column
M-S-UP       delete row
M-S-DOWN     insert row
C-c -        insert hline
C-c ^        sort
```

这些键位让你可以**重构表格**——加列、删列、移动、排序——不需要重写。这是 Org table 优于"静态表格" (Markdown、HTML) 的地方。

### 6.3 公式

公式是 Org table 的杀手锏——它支持完整的 Emacs Calc (Emacs 内置的计算器) 语法。这意味着你的表格可以**计算**:

```org
| Qty | Price | Total |
|-----+-------|-------|
|   2 |   100 |   200 |
|   3 |    50 |   150 |
|-----+-------|-------|
|     |       |   350 |
#+TBLFM: $3=$1*$2::@4$3=vsum(@2..@-1)
```

`$3=$1*$2`: 第三列 = 第一 * 第二
`@4$3=vsum(@2..@-1)`: 第四行第三列 = vsum(第二行..上一行)

`C-c C-c` 在公式上,计算并填表。

公式的语法值得解释: `$N` 表示第 N 列,`@RN` 表示第 N 行。`$3=$1*$2` 是"对每一行,第三列 = 第一列 × 第二列"——这是行级公式。`@4$3=vsum(@2..@-1)` 是"第四行第三列 = 从第二行到当前行的第三列求和"——这是单元格级公式,用于合计行。

`vsum` 来自 Emacs Calc——Calc 是一个完整的科学计算器,支持向量、矩阵、复数、单位转换、微积分。Org table 公式 = Calc 函数。这意味着你可以在 table 里做非常复杂的计算 (统计、线性代数、符号运算)。

### 6.4 高级

```org
| Date         | Amount |
|--------------+--------|
| [2026-01-15] |    100 |
| [2026-02-20] |    200 |
#+TBLFM: $2=round($2*1.1)
```

这里 `round($2*1.1)` 把第二列乘以 1.1 (加 10% 税) 再四舍五入。一个公式,批量修改整列——比 Excel 拖动更快。

---

## 7. Org-Babel (Literate Programming)

### 7.1 配置

Babel 是 Org 执行代码的能力——它的名字来自"巴别塔",寓意"多语言"。一个 Org 文件可以同时跑 Python、R、SQL、Shell、Elisp——结果都嵌回文档。这是 literate programming 的现代实现。

Donald Knuth 1984 年提出 literate programming——他认为代码应该和文档混合,程序应该像"写给读者的故事"。这个理念在 1980 年代没流行 (那时 IDE 兴起),但在数据科学 (Jupyter, R Markdown) 时代复兴了。Org-babel 是这一理念的 Emacs 实现。

```elisp
(org-babel-do-load-languages
 'org-babel-load-languages
 '((python . t)
   (shell . t)
   (emacs-lisp . t)
   (sql . t)
   (ruby . t)
   (C . t)))
```

这个配置告诉 Org-babel 启用哪些语言。每个语言需要对应的"ob-XXX"包——但常见的 (python, shell, elisp) 都内置。`org-babel-do-load-languages` 一次性加载所有需要的语言支持。

### 7.2 代码块

代码块是 babel 的基本单位——`#+begin_src` 和 `#+end_src` 之间是一段代码:

````org
#+begin_src python :exports both
def fib(n):
    return n if n < 2 else fib(n-1) + fib(n-2)

print([fib(i) for i in range(10)])
#+end_src

#+RESULTS:
| 0 | 1 | 1 | 2 | 3 | 5 | 8 | 13 | 21 | 34 |
````

`:exports both` 是 header argument——它告诉 Org 导出时包含代码和结果。其他选项: `code` (只导代码)、`results` (只导结果)、`none` (都不导)。

按 `C-c C-c` 在代码块上执行——Python 解释器启动,代码运行,结果嵌入到 `#+RESULTS:` 下面。这个结果会被保存到文件——下次打开 Org 文件,结果就在那里。这让 Org 文件成为"自带数据的文档"。

### 7.3 操作

```
C-c C-c    执行代码块
C-c '      编辑代码 (用对应 mode)
C-c C-v t  tangle (导出代码到文件)
C-c C-v e  export
```

`C-c '` (org-edit-src-code) 是一个非常聪明的命令——它在单独的 buffer 里打开代码块,用对应语言的 major mode (Python、Shell、Elisp)。这意味着你**有完整语法高亮、缩进、补全**——和在 .py 文件里写代码一样。写完 `C-c C-c` 嵌回 Org。

`tangle` (C-c C-v t) 是 literate programming 的另一半——它把所有代码块导出为独立的源文件。这意味着你可以用 Org 写代码,但实际编译的是 tangled 出来的 .py / .c / .el 文件。这是 Org 用作"配置文件"的方式 (后面会讲 init.org)。

### 7.4 Literate 配置 (init.org)

这是 Org-babel 最实用的应用之一——用 Org 写 Emacs 配置:

```org
* Basics

#+begin_src emacs-lisp
(setq make-backup-files nil)
#+end_src

* UI

#+begin_src emacs-lisp
(load-theme 'modus-vivendi t)
#+end_src
```

`C-c C-v t` 把所有 src 导出到 `init.el`。

这种"配置即文档"的方式有巨大好处: 你的配置不再是一堆 `setq`,而是有**章节、说明、理由**的文档。读 init.org 时,你看到的是"为什么要这样配",而不只是"配了什么"。几年后回来修改,你能立刻理解自己当初的思路。

很多 Emacs 高手 (包括 Emacs 配置社区 r/emacs 的高赞配置) 都用 init.org——这是 literate programming 在配置管理上的最佳实践。

---

## 8. Export

### 8.1 命令

Org 的 export 系统是它的"输出层"——一个 Org 文件可以导出为 HTML、PDF、Markdown、ODT、LaTeX...这意味着你写一份 Org,输出多种格式。这是文档工作者的梦想。

```
C-c C-e h h    export to HTML
C-c C-e l p    export to LaTeX PDF
C-c C-e m m    export to markdown
C-c C-e o o    export to ODT
```

`C-c C-e` 打开 export dispatcher——一个 popup 显示所有导出选项。每个选项有快捷键 (hh = HTML, lp = LaTeX PDF),按对应键直接导出。

### 8.2 配置

```elisp
(setq org-export-with-toc t)
(setq org-html-postamble nil)
```

这些变量控制导出行为——`with-toc` 决定是否生成目录,`html-postamble` 决定 HTML 是否有页脚 (作者、日期等)。这些细节让你能定制输出格式。

对于学术写作,Org → LaTeX → PDF 是一个强大的工作流——你写 Org (比 LaTeX 易写得多),Org 自动转 LaTeX,然后 LaTeX 编译为 PDF。这意味着你**享受 LaTeX 的排版质量,但不需要写 LaTeX**。

---

## 9. Org-roam (Zettelkasten)

### 9.1 安装

Org-roam 把 Org 变成 Zettelkasten——一种"卡片盒"笔记系统,由德国社会学家 Niklas Luhmann 在 20 世纪发明。他一生写了 70 本书、400 篇论文,都基于他的 9 万张卡片。Org-roam 把这个系统数字化。

Zettelkasten 的核心是**双链**——每张卡片可以链接到其他卡片,被链接的卡片自动显示"backlink" (谁链接了我)。这种"网状结构"让笔记不再是孤立的,而是连接的——一篇笔记可以"激活"相关的所有笔记。

```elisp
(use-package org-roam
  :ensure t
  :custom
  (org-roam-directory "~/org/roam/")
  :init
  (org-roam-setup)
  :bind (("C-c n f" . org-roam-node-find)
         ("C-c n i" . org-roam-node-insert)
         ("C-c n l" . org-roam-buffer-toggle)))
```

`org-roam-directory` 指定卡片目录——所有 .org 文件都是"卡片"。`org-roam-setup` 初始化数据库 (sqlite) 用于快速查询。

### 9.2 用法

- `C-c n f` 找/创建节点
- `C-c n i` 插入链接到节点
- `C-c n i` 是核心——它让你在当前笔记里"提及"另一个笔记。这创建了链接,被链接的笔记自动获得 backlink。
- `C-c n l` 显示 backlinks buffer——侧边栏显示**所有链接到当前笔记的笔记**。

每个 .org 文件是一个"卡片"。
Org-roam 管理 backlinks (谁链接到这个卡片)。

Backlinks 的价值在于**意外发现**——你写一篇关于"信息论"的笔记,backlinks 显示你三年前写的"熵"笔记也链接到这里。这种"意外重连"是 Zettelkasten 的核心价值,org-roam 让它变成日常体验。

Org-roam vs Roam Research vs Obsidian: Org-roam 是开源的 (Roam 是闭源商业)、本地优先的 (Roam 在云端)、用纯文本的 (Roam 用数据库)。这意味着你的笔记永远在你手里——可以备份、git 跟踪、脚本处理。

---

## 10. 其他有用的

Org 生态有大量"扩展包"——每个针对一个特定场景。这里列几个最流行的:

### 10.1 Org-pomodoro

番茄钟——25 分钟专注,5 分钟休息。基于 TODO 的状态机: 开始番茄钟时,当前 TODO 状态变为 IN-PROGRESS;番茄钟结束,自动切到休息。

```elisp
(use-package org-pomodoro
  :ensure t
  :bind ("C-c C-x p" . org-pomodoro))
```

番茄钟。

### 10.2 Org-journal

每日日记——和 capture 的 datetree 类似,但专门为日记优化。

```elisp
(use-package org-journal
  :ensure t
  :custom
  (org-journal-dir "~/org/journal/"))
```

每日日记。

### 10.3 Org-ref

学术引用 (BibTeX 集成)——让 Org 成为论文写作工具。你可以在 Org 里插入 `\cite{key}`,org-ref 自动管理 BibTeX 文件,导出 LaTeX 时正确渲染引用。这是 Emacs 在学术界的优势之一。

### 10.4 Org-present / ox-reveal

Org 当演示文稿 (slides)——把 Org 文件导出为 reveal.js (HTML slides) 或 Beamer (LaTeX slides)。这意味着你可以用 Org 的所有特性 (代码块、表格、公式) 来做演示。比 PowerPoint 强大得多。

### 10.5 Org 的意外用法

Org 的强大在于它的"通用性"——同一套机制 (heading、property、tag、link) 可以用于无数场景:

1. **Literate programming**: org-babel 在 org 里跑 Python,文档+代码+结果一体——这是数据科学的杀手锏。
2. **Org as database**: org-roam 把 org 当 Zettelkasten,backlinks 自动——这是知识管理工具。
3. **Org as spreadsheet**: 表格 + 公式,计算报表——替代 Excel 做小型报表。
4. **Org as journal**: 每天一个 entry,自动 datetree——替代 Day One 等日记应用。
5. **Org as presentation**: ox-reveal 导出 reveal.js slides——替代 PowerPoint。
6. **Org as GTD inbox**: capture → agenda → file → done——这是 GTD 工作流的完整实现。
7. **Org as PM**: 项目状态报告,客户共享——一个 .org 文件可以发给客户,他们看 HTML 导出。
8. **Org as notebook**: 数学公式 (LaTeX),导出 PDF——替代 LaTeX 写论文。
9. **Org as CRM**: 用 properties 存客户信息,tags 标记状态——轻量级 CRM。
10. **Org as bug tracker**: TODO + priority + tag = bug。一个项目用一个 .org 文件管理所有 issue。
11. **Org as recipe book**: 食谱 + 配料表 + 步骤——一个 .org 文件管所有菜谱。
12. **Org as habit tracker**: `.+1d` repeater + agenda 习惯视图——量化自我。

---

## 11. 实战练习

### 任务 1: 配置 Org

加到 init.el (Module 4 已有),确认:
- `C-c c` capture 工作
- `C-c a` agenda 工作
- 表格 `TAB` 工作
- `C-c C-c` 在代码块工作

如果都工作,你已经有了完整的 Org 设置。接下来是在真实工作中使用——给自己一周时间,所有笔记、TODO、journal 都用 Org,不要回到旧工具。

### 任务 2: 用 Org 写日记

```
C-c c j RET
Today I learned...
C-c C-c
```

连续做一周。你会发现 Org 的 capture → datetree → agenda 流程比任何日记应用都快。

### 任务 3: 用 Org 做项目计划

```org
* Project X
** TODO Research [0/3]
*** TODO Read paper A
*** TODO Read paper B
*** TODO Summarize
** TODO Implementation
** TODO Testing
```

agenda 看所有 TODO。

`[0/3]` 是 Cookie——显示"完成 0/3"。每完成一个 TODO,Cookie 自动更新。这让进度可视化,不需要手动数。

### 任务 4: Literate 配置

把 init.el 重写为 init.org,每个配置在 src block,有说明。
`C-c C-v t` 生成 init.el。

这是 Module 4 的配置的"升级版"——同样的配置,但现在每行都有解释。你会发现,写文档的过程会**重新审视**你的配置——你会发现自己"为什么这样配"忘了,重新思考会让配置更清晰。

### 任务 5: Babel 跑数据

```org
#+begin_src python :session :exports both
import pandas as pd
data = pd.read_csv("data.csv")
data.describe()
#+end_src
```

C-c C-c,看结果。

`:session` 让代码块共享一个 Python 进程——多个代码块可以共享变量、DataFrame。这意味着一个 Org 文件就是一个交互式数据科学 notebook (类似 Jupyter)。

---

## 12. 自测

1. Org capture 的模板里 `%?` 和 `%a` 分别是?
2. Agenda 的 `C-c a a` 和 `C-c a t` 区别?
3. Org table 的公式怎么写?
4. Babel 怎么跑 Python?
5. Org-roam 的"backlink" 是什么?

**答案**:
> 1. %? 光标位置;%a annotation (来自 region 或 link)
> 2. agenda 日历视图;todo 是 todo list
> 3. `$N=expr` 第 N 列 = expr;`@RN$CN` = 第 RN 行第 CN 列
> 4. C-c C-c 在 #+begin_src python ... #+end_src
> 5. 谁链接到这个 note

---

## 13. 下一步

进入 `magit.md` 学 Git。
