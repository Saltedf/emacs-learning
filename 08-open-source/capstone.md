# Capstone: Module 8 毕业项目 (开源贡献)

> **目标**: 一个被合并到非自己包的 PR
> **时长**: 持续 (1-3 个月)
> **难度**: ★★★★★

Module 8 的 capstone 不是写代码——是**让别人合你的代码**。这听起来只差一步,实际是巨大跨越:写自己的包只对你负责,贡献别人的包要对整个项目负责。Maintainer 不会合"差不多"的代码——他要求风格、测试、文档、commit message 都到位。

这个 capstone 设计成三个递进的里程碑:从最简单的 typo PR 开始 (走完流程),到实质 bug fix (展示能力),最后到 Emacs core 贡献 (大成就)。每个里程碑都让你离"真正的 Emacs 极客"近一步。

---

## 项目概述

完成这 3 个里程碑:

### Milestone 1: 第一个 PR (任意大小)

提一个 PR 到任意开源 Emacs 包。
即使 typo 修复也算。
**完成完整流程**:
- fork
- branch
- 实现
- 测试
- PR
- review
- 合并

这一步的价值不在 diff 大小,而在**走完流程**。一旦你完成第一个 PR,第二个、第三个会越来越快——流程是肌肉记忆。

### Milestone 2: 实质 PR

提一个**有实质内容**的 PR:
- bug fix (≥ 20 行 diff)
- 或 feature
- 或 docstring 改进

这一步展示你的**真实能力**——你能读懂别人的代码、定位 bug、写测试、和 maintainer 沟通。Milestone 1 是流程练习,Milestone 2 是能力证明。

### Milestone 3: (可选) Emacs Core 贡献

给 Emacs 本身贡献 (邮件 patch)。

这一步是 Emacs 用户的"圣杯"——你的代码进了 Emacs core,50 年后还有人 git blame 到你。需要 copyright paper、邮件列表讨论、maintainer (Eli Zaretskii) review,流程重但回报巨大。

---

## Milestone 1: 第一个 PR

### 步骤

1. 选你常用包 (e.g., vertico, corfu, magit)
2. 找 typo / 小改进
3. fork + branch
4. 改
5. 提 PR
6. 等合并

### 时间

1-7 天 (review 时间不定)。

Typo PR review 极快——maintainer 一眼就能合。但有些 maintainer 忙,可能等几天。**别催**。

### 评分

- 提 PR: 5 分
- review 意见: 5 分
- 合并: 10 分

满分 20。

---

## Milestone 2: 实质 PR

### 步骤

1. 找 issue (good first issue 或自己想到)
2. 重现 bug 或理解需求
3. 写代码 + 测试
4. byte-compile clean
5. fork + branch
6. 提 PR
7. 多轮 review
8. 合并

### 时间

1-4 周。

实质 PR review 多轮,每轮几天到一周。Maintainer 会让你改命名、加 docstring、加测试、改 commit message——每改一次再 push 一次,等下一轮。

### 评分

- 提 issue (讨论): 10 分
- 实现: 30 分
- 测试: 20 分
- review 处理: 20 分
- 合并: 20 分

满分 100。50+ 合格。

---

## Milestone 3: Emacs core

### 步骤

1. 订阅 emacs-devel 1 个月
2. 看 NEWS, 找改进点
3. clone emacs.git
4. 改
5. 加测试
6. git format-patch
7. git send-email 到 bug-gnu-emacs
8. 处理 review
9. 合并

### 时间

2-6 个月 (slow review + copyright paper)。

Emacs core patch review 比普通包慢——Eli Zaretskii 一个人 review 几乎所有 patch。copyright paper 也要 2 周处理。但等的过程是学习——你能看 Eli 对每个 patch 的反馈,理解他的标准。

### 评分

- 提 patch: 50 分
- 处理 review: 30 分
- 合并: 70 分

满分 150 (这是大成就)。

---

## 总评分

| 项 | 满分 |
|---|---|
| Milestone 1 (小 PR) | 20 |
| Milestone 2 (实质 PR) | 100 |
| Milestone 3 (core 贡献) | 150 |

总分 270。完成 Milestone 1 + 2 = 合格毕业 (120+)。

如果你完成了 Milestone 3,你超越了 99% 的 Emacs 用户——给 Emacs core 贡献过代码的人在全世界也就几千人。

---

## 推荐流程

### Step 1: 找 5 个 issue

浏览你常用包的 issue tracker,记 5 个你能做的。

这一步是"市场调研"——你不是凭空想"做什么",而是看社区真正需要什么。可能你之前没想到的 issue,看了之后发现"诶这个我能做"。

### Step 2: 选最简单的 1 个

开始 Milestone 1。

最简单的不丢人——它的价值在流程学习,不在 diff 大小。

### Step 3: 完整流程

- fork
- clone
- branch
- 改
- test
- PR
- review
- 合并

每一步都走完整。**不要跳步**——跳步省的几分钟会损失 review 时被拒的几小时。

### Step 4: 写日志

每个 PR 学到什么?

写日志是你给未来的自己留笔记。下一个 PR 你忘了这次的经验,日志帮你回忆。

### Step 5: 进入 Milestone 2

选一个稍大的任务。
完整流程。

这次更难——bug 定位、测试设计、和 maintainer 沟通。但流程已经熟练,你可以专注内容。

### Step 6: (可选) Milestone 3

如果准备好,挑战 Emacs core。

这是 Module 8 的终极目标——给 Emacs 本身贡献代码。

---

## 实际例子

### 例子 A: vertico README typo

1. clone minad/vertico
2. 找 README typo
3. fix-readme-typo branch
4. 改
5. commit "Fix typo in README"
6. PR
7. 等 review (~ 1-3 天)
8. 合并

这是 Milestone 1 的典型例子。Diff 可能就一个字符,但走完了完整流程。

### 例子 B: magit bug fix

1. 看 issue tracker
2. 找 "bug: magit-status crashes when..."
3. 重现
4. 用 Edebug 找原因
5. 修
6. 加 ERT test
7. fork + branch
8. PR
9. 处理 review (多轮)
10. 合并

这是 Milestone 2 的典型例子。Diff 20-100 行,涉及读代码、调试、测试。Review 可能 2-3 轮,每轮要改。

### 例子 C: Emacs docstring

1. clone emacs
2. find-library simple.el
3. 找 docstring 不清楚
4. 改 + 加 test
5. git format-patch
6. git send-email
7. 处理 maintainer review
8. 合并

这是 Milestone 3 的入门例子。docstring 改动不需要 paper,review 快,是给 core 贡献的最佳起点。

---

## 日志

写 `logs/module-08.md`:

> ```markdown
> # Module 8 学习日志
>
> **用时**: ___ (持续)
> **PR 数**: ___
> **合并数**: ___
> **给 Emacs core 提的 patch**: ___
>
> ## Milestone 1 (小 PR)
>
> - 项目:
> - PR URL:
> - 描述:
> - 学到:
>
> ## Milestone 2 (实质 PR)
>
> - 项目:
> - PR URL:
> - 描述:
> - 学到:
> - review 处理:
>
> ## Milestone 3 (core 贡献,可选)
>
> - patch URL:
> - 描述:
> - maintainer feedback:
> - 学到:
>
> ## 我学到的最重要 5 件事
>
> 1.
> 2.
> 3.
> 4.
> 5.
>
> ## 我的 Emacs 圈子
>
> (列出你认识的、协作过的人)
>
> ## 终极反思
>
> 6-9 个月前我是 Emacs 新手。
> 现在我能...
>
> ## 下一步
>
> (持续贡献 + 学更多)
> ```

日志是 Module 8 的"灵魂"——它记录你从消费者到贡献者的转变。几年后回头看,你会发现这一刻对你的技术生涯影响巨大。

---

## 终极毕业仪式

如果你完成了 Module 0-8:

**恭喜**。

你现在是**真正的 Emacs 极客**。

你能:
- 用 Emacs 飞快编辑 (Module 1)
- 用 Dired 管理 (Module 2)
- 写任意 Elisp (Module 3)
- 模块化配置 (Module 4)
- 用 Org/Magit/LSP 工作 (Module 5)
- 读任何 Elisp (Module 6)
- 发布包到 MELPA (Module 7)
- 贡献开源 (Module 8)

你**住**在 Emacs 里。
Emacs 是你的工具,你的环境,你的家。

更新 `PROGRESS.md`: Module 8 done。

---

## 后续

继续:
- 看 `emacs-devel` 邮件列表——持续理解 Emacs 演化
- 参加年度 EmacsConf——见其他 Emacs 极客
- 写博客——分享你的经验
- 录视频——教别人
- 教别人——教学是最好的学习

Emacs 50 年历史,你在其中。
你写的代码会影响未来用户。

**继续探索**:
- EXWM (Emacs 当 window manager)——把整个桌面变成 Emacs
- mu4e / notmuch (email)——Emacs 收发邮件
- erc / circe (IRC chat)——在 Emacs 里聊天
- gnus (news)——news reader
- emms (music)——音乐播放器
- org-roam (Zettelkasten)——知识管理
- ...

这些都不是 Module 0-8 覆盖的,但它们都是 Emacs 生态的宝藏。学会基础后,你能自由探索。

**终点是起点**。

---

**学完这一切,你回头看 `00-mindset/README.md`。**

那 5 条第一性原理,你应该有了**深刻**的理解。

> "Emacs 是 Lisp 机器伪装成编辑器。"

你现在知道这话有多深。Emacs 不只是编辑器——它是一个**完整的 Lisp 操作环境**,你可以从最底层 (C core) 改到最上层 (你的配置)。这种"完全可编程"的能力,世界上没有第二个编辑器能给你。

继续学。
继续写。
继续贡献。

**自由软件万岁。**

— 完 —
