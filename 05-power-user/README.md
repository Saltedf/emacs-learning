# Module 5: Power User 工作流

> **目标**: 把日常编辑器工作流完全迁到 Emacs
> **时长**: 3 周 (~60 小时)
> **难度**: ★★★☆☆ (实操为主)
> **依赖**: Module 0-4 (有自己的 init.el)
> **核心产出**: Org literate 配置 + Magit PR + 现代补全栈

---

## 0. 这个模块在学什么

前面四个模块教的是"Emacs 是什么、怎么写"——你已经学完了键位、help、Dired、Elisp、模块化配置。但会写 Elisp 不等于会用 Emacs 做事,就像会调音不等于会作曲。

Module 5 的角色是**桥**:把工具知识接到真实工作流上。完成这个模块之前,Emacs 对你来说只是一个"能定制的好编辑器";完成之后,它变成你的笔记系统、Git 客户端、IDE、项目管理器——一个你每天 8 小时工作的环境。

你已经:
- 装好 Emacs,写过 Module 0 init.el
- 学会键位 + help system (Module 1)
- 学会 Dired (Module 2)
- 学会 Elisp (Module 3)
- 写了模块化 init.el (Module 4)

但你还**没用 Emacs 做真实工作**。这是一个非常现实的鸿沟:
- 写文档可能还在 Word/Typora/Markdown editor——你觉得这些"够用"
- 用 Git 可能还在命令行 / SourceTree / GitHub Desktop——你以为 CLI 够快了
- 代码补全可能还在 VSCode/JetBrains——你舍不得这些 IDE 的智能
- 看项目代码可能还在 tree-style 文件浏览器——你习惯鼠标点

这些"够用"是真的够用——但只是因为你**没体验过更好**。Module 5 不是教你怎么忍痛切换,而是教你怎么把 Emacs 配到比那些工具更好用。当配置完成后,你会**主动选择** Emacs——因为它真的更快、更连贯、更可控。

完成后你能用 Emacs 做:
- 笔记 / 待办 / 文档 (Org-mode)
- Git 全流程 (Magit)
- 项目管理 (project.el)
- 现代 IDE 补全 (Corfu + Vertico + Eglot)

### 0.1 这个模块的核心问题

Module 5 要回答的问题是: **一个 40 年的编辑器,凭什么能替代 2026 年的现代工具?**

答案分四层,对应这个模块的四个核心包:

1. **Org-mode** 凭什么替代 Notion/Obsidian/Roam? —— 因为它是纯文本、可编程、可扩展,且**整个工作流集中在一个工具里** (笔记 + TODO + 日历 + 文档 + 代码执行)。
2. **Magit** 凭什么替代 SourceTree/GitKraken/CLI? —— 因为它是 buffer,可搜索可编辑可脚本化,且**完全键盘驱动**。
3. **project.el + Vertico/Corfu/Consult** 凭什么替代 Helm/Telescope/IDE 文件树? —— 因为它们是 Unix 哲学的产物——每个包做一件事,组合起来强大,每个包 < 500 行。
4. **Eglot** 凭什么替代 VSCode 的 LSP? —— 因为它**与内置功能 (xref, flymake, eldoc) 集成**,没有重复造轮子,极简且快。

每一层背后都是**第一性原理**: 一个工具如果透明、可组合、可替换,它的长期价值会超过"大而全"的工具。Emacs 证明了这一点。

---

## 1. 学习单元

学习单元按"工作量从大到小"排序。Org-mode 是最大的 (5-7 天),因为它的概念多 (capture/agenda/babel/table/export/roam);Magit 是最快的上手 (3-4 天就能完成日常流程);补全栈和 LSP 各 3-4 天;Calendar/Diary/GUD/Mule 各 1-3 天,是内置功能的"补强"单元。

### 1.1 Org-mode (5-7 天)
- Capture / Agenda / Babel
- Org tables
- Org-roam (可选)

Org 是这个模块的**重头戏**,因为它体现了 Emacs 最强的"工作流整合"——一个 .org 文件同时是笔记、TODO、日历、文档、可执行代码。学 Org 不只是学语法,是学一种"文本即数据"的工作哲学。

### 1.2 Magit (3-4 天)
- magit-status
- Stage/Commit/Push
- Branch/Merge/Rebase
- Ediff

Magit 学得快,但要"肌肉记忆"——所有 Git 操作应该都不离开键盘。

### 1.3 Project + 补全 (3-4 天)
- project.el
- Vertico + Corfu + Orderless
- Consult + Marginalia + Embark

这部分是"看不见的提速"——配置好后,你会发现每次 `M-x`、每次切 buffer、每次找文件都比以前快几秒。

### 1.4 LSP (3-4 天)
- eglot (内置)
- flymake
- xref

LSP 让 Emacs 在大型代码库里"知道"你在做什么——跳定义、查引用、自动补全,这些是现代 IDE 的标配。

### 1.4b Calendar + Diary (1-2 天)
- Calendar 模式 (导航、日期算术、跨日历)
- Diary (`~/diary` 纯文本事件记录)
- 与 Org-agenda 的对比

这是 Emacs 内置的"时间底层"——Org-agenda 建立在它之上。简单场景 (生日、固定事件) 用 Diary 比 Org 轻。

### 1.4c GUD 调试器 (2-3 天)
- `M-x gdb` / `M-x pdb` 调试
- Source buffer 断点 + 当前行高亮
- Conditional breakpoint、watch、多线程
- 与 dap-mode 对比

GUD 是 Emacs 内置的调试器接口——对 C/C++ 调试尤其重要 (GDB 没有替代品)。

### 1.4d Mule 国际化 (2-3 天)
- 编码系统 (UTF-8, GB2312, etc.)
- 内置输入法 (Quail)
- 字符信息 (`C-u C-x =`)
- CJK 字体配置 (fontset)

Mule 让 Emacs 编辑任何语言——理解它是处理乱码、字体对齐、多语言文本的基础。

### 1.5 实战 (5+ 天)
- Org literate 配置
- Magit 完整 PR 流程
- 项目工作流

实战阶段把这些工具**串联**起来,在一个真实项目里使用。

### 1.6 Misc + Amusements (1 天,可选)

打开 `misc-amusements.md` 学 Emacs 的"工具 + 玩具"双重性——内置游戏 (tetris、snake、doctor、dunnet)、animation、batch mode、emacsclient。

这不是"娱乐"——它是理解 Emacs 文化的钥匙。学完你能用 `emacs --batch` 做 CI,用 emacsclient 跨终端复用进程,知道为什么 Emacs 内置 Eliza 精神分析师。

---

## 2. 现代 Emacs 工具栈 (2026 主流)

现代 Emacs (2024-2026) 的工具栈有一个明显的趋势: **小而美,组合式**。前一代 (2015-2020) 流行 Helm 和 Ivy 这种"大而全"的包——一个包几千行代码,把 UI、命令、source 都打包。新一代 (2021-) 流行 Vertico、Corfu 这种极简包——每个包 < 500 行,只做一件事,然后组合。

这反映了 Emacs 社区的一种哲学回归: **工具应该透明、可组合、可替换**。一个 < 500 行的包,你可以读懂源码,可以替换为别的——这是"工具"和"产品"的根本区别。VSCode 的补全是产品 (你不能换),Emacs 的补全是工具 (随时可换)。

下面五张图展示了五个领域的工具栈。

### 2.1 补全栈

补全栈的核心是"分层"——每层独立,可换:

```
Minibuffer 输入:
   ↓
Vertico (vertical UI)
   ↓
Orderless (匹配算法)
   ↓
Consult (高级命令: line, buffer, grep)
   ↓
Marginalia (候选旁边显示信息)
   ↓
Embark (从候选拉出菜单)
```

这五个包各自做一件事,但合起来效果惊人。比如 `M-x consult-buffer` 时:Vertico 提供垂直 UI,Orderless 让你能输入 "init el" 模糊匹配,Marginalia 在旁边显示文件行数和类型,Embark 让你按 `C-.` 弹菜单做更多操作 (kill / rename / open in dired)。一个命令,五个工具协作。

### 2.2 代码补全

代码补全同样分层——UI 和 source 分开:

```
输入字符:
   ↓
Corfu (popup UI)
   ↓
 Cape (各种 source: file, dabbrev, keyword)
   ↓
Completion-at-point (内置)
   ↓
eglot (LSP server,Module 5)
```

关键设计: Corfu 不关心 source 是什么。它可以来自 Cape (本地 buffer 词)、文件路径、或 LSP server。这种"UI / source 分离"是 IDE 没有的——VSCode 把补全 UI 和 LSP 强绑定,而 Emacs 你可以**不装 LSP 也有补全** (用 dabbrev 从当前 buffer 补词)。

### 2.3 Git

```
Magit (主入口)
   ↓
Forge (GitHub PR/Issue 集成)
   ↓
Git gutter (侧边栏显示 diff)
```

Magit 是单一工具——它没有"分层"设计,因为 Git 本身是一个完整的工具。但 Magit 之上可以加 Forge (PR review) 和 git-gutter (实时显示改动),形成 Git 工作流的完整栈。

### 2.4 Org

```
Org-mode (基础)
   ↓
Org-roam (Zettelkasten)
   ↓
Org-agenda (TODO 时间视图)
   ↓
Org-babel (literate programming)
```

Org 的"栈"是叠加功能,不是替换。Org-mode 本身就强大;加上 Org-roam 变成双链笔记系统;加上 Org-agenda 变成 GTD 工具;加上 Org-babel 变成可执行文档。所有这些共享同一个 .org 文件格式——**这是 Org 最强的设计**: 一个文件,无数用法。

### 2.5 LSP

```
eglot (内置 LSP client,推荐)
   ↓
LSP server (pyright, clangd, ...):
   ↓
Flymake (诊断)
   ↓
xref (跳定义)
```

或 `lsp-mode` (功能更全,但重)。

LSP 协议是微软 2016 年开源的——它让"编辑器 × 语言"的笛卡尔积变成了加法。eglot 不发明新概念,只是把 LSP 协议接到 Emacs 内置功能 (xref 跳定义、flymake 诊断、eldoc hover、completion-at-point 补全) 上。这种"复用内置"是 eglot 的核心哲学——也是它只有 ~1000 行代码的原因。

---

## 3. 这个模块不教什么

有些"流行的"Emacs 配置/工具,Module 5 不教——不是因为它们不好,而是因为它们要么是**个人偏好**,要么是**独立大主题**:

- ❌ evil-mode / spacemacs / doom (用默认键位)—— vim 键位是个人偏好;学默认键位能让你看懂所有文档和教程
- ❌ treemacs (个人偏好,不强求)—— 文件树是 IDE 习惯,Emacs 的 Dired + project.el 更高效
- ❌ email (mu4e/notmuch 单独学)—— 这是大主题,值得一个独立模块
- ❌ IRC / chat—— 通信工具,不影响编码工作流
- ❌ EXWM (Emacs X Window Manager,极客向)—— 用 Emacs 当窗口管理器,极客玩具,不适合新人

这些是"额外",Module 5 不深入。但**学完 Module 5 你能自己学这些**——因为它们都是同样的"配置 + 实操"模式。

---

## 4. 毕业检查

### 概念题

这些题目考"理解"——不只是会按键,要能解释清楚。如果你能用一段话讲明白每个概念,就过关了。

1. Org-mode 是什么? 它和普通 markdown 区别?
2. Capture/Agenda 的关系?
3. Magit 比 git CLI 强在哪?
4. Vertico/Corfu/Orderless 各自的角色?
5. eglot 和 lsp-mode 区别?
6. project.el 怎么识别项目边界?

### 实操题

这些题目考"会用"——在真实场景里能用出来:

1. 用 Org 写一个 capture 模板
2. 用 Magit 完成: branch → stage → commit → push
3. 用 Consult ripgrep 跨项目搜
4. 用 eglot 跳定义 + 看 hover
5. 用 Org-babel 跑 Python 代码块

---

## 5. 下一步

打开 `concept-anchor.md` 看 editor manual 对应章节。
然后各专题文件 (`org-mode.md`, `magit.md`, ...) 学细节。
