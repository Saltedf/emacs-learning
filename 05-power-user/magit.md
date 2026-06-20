# Magit 详解

> 学完这个文件,你能用 Magit 完成 Git 全流程,效率比 CLI 高 5 倍

Magit 是 Emacs 社区最被称赞的包之一——很多非 Emacs 用户也羡慕它。它被广泛认为是"最好的 Git 客户端",即使是 VSCode 用户也会承认 Magit 在某些方面无可匹敌。这一节会解释**为什么** Magit 这么好,以及怎么用它。

---

## 1. Magit 是什么

### 1.1 一句话

**Magit 是 Emacs 的 Git 接口**,被称为"最好的 Git 客户端"。

不是 GUI 客户端,是**交互式 buffer**。
所有 Git 操作通过 keybinding,但**可视**。

这三句话需要展开解释,因为它们抓住了 Magit 的本质:

- **"不是 GUI,是 buffer"**: GUI 客户端 (SourceTree、GitKraken) 用按钮和列表——你必须用鼠标。Magit 用一个 Emacs buffer 显示 Git 状态——这个 buffer 可以**像任何 buffer 一样导航、搜索、复制**。这是质的区别: 你可以用 `C-s` 搜 "modified" 跳到所有修改的文件,可以用 `M-w` 复制 commit hash,可以写 Elisp 脚本处理它。GUI 工具是黑盒,Magit 是透明的。

- **"keybinding 但可视"**: CLI 用户需要背命令 (`git rebase -i HEAD~3`),输入完整命令。Magit 用"transient menu"(弹出菜单)——按 `r` 弹出 rebase 菜单,显示所有选项,你按对应键 (`i` for interactive, `a` for abort, `c` for continue)。这种"按一个键看选项"的模式,让你**不需要背**——菜单即文档。

- **"比 CLI 快 5 倍"**: 这不是夸张。一个典型的 stage-commit-push 流程,CLI 是 `git add .` → `git commit -m "..."` → `git push` (~10 秒,如果还要输入 commit message 更久)。Magit 是 `C-x g` → `s` → `c c` → 写 message → `C-c C-c` → `P u` (~3 秒)。长期累积,这是几小时的时间。

### 1.2 第一性原理推导

为什么 Magit 比 CLI 和 GUI 都好?从第一性原理推导:

- **第一性原理**: Git 是版本控制,核心操作是 commit/diff/log/branch。但 Git CLI 有几百个命令,参数复杂,记不住。
- **推论 1**: 编辑器应该集成 Git。你在编辑器里改代码,看 diff 应该在同一环境——切换到终端是上下文切换。
- **推论 2**: 用 Emacs buffer 显示 Git 状态,可搜索可编辑。这是 Emacs 的优势——所有 buffer 平等,都有同样的导航/搜索/编辑能力。
- **推论 3**: 操作通过 keybinding,但有可视反馈。keybinding 比鼠标快,但需要"看"才能"操作"——buffer 显示状态,你看着操作。
- **推论 4**: 不需要鼠标,完全键盘。这是 Emacs 哲学的延续——手不离键盘。
- **推论 5**: 比任何 GUI Git 客户端都快——因为它是 buffer。GUI 工具是"应用",Magit 是"buffer"。应用需要启动、点击、切换;buffer 是你已经在的地方。

这个推导的关键洞察: **Magit 不是"Git 客户端",它是"把 Git 集成进 Emacs buffer 的接口"**。这听起来微妙,但实际差异巨大——Magit "免费"获得了所有 Emacs 能力 (search、edit、script、yank)。

### 1.3 vs CLI

CLI:
- 背命令 (`git rebase -i`, `git cherry-pick`)
- 输入完整命令
- 输出是文本流,看一眼就滚走
- 多步操作需要多次输入

Magit:
- 弹出菜单 (popup)
- 一键操作
- 看所有状态
- 状态持久 (buffer 在那里)

最重要的差异: **Magit 的状态是持久的**。你打开 magit-status,看到所有未提交的改动、所有 staged 文件、所有最近 commits——这个状态一直在那里。CLI 你看一眼 `git status`,然后做别的,状态就忘了——你需要再 `git status`。这种"持久状态"让 Magit 用户**对仓库状态有更好的直觉**。

### 1.4 vs GUI 客户端

| 工具 | 优点 | 缺点 |
|---|---|---|
| Git CLI | 强大、可脚本化 | 难记、文本流 |
| GitKraken/Sourcetree | 直观、好看 | 鼠标、慢、闭源 |
| GitHub Desktop | 简单 | 功能少、商业绑定 |
| Magit | 强大、键盘、可扩展 | 需要 Emacs |

Magit 是唯一同时"强大"和"键盘驱动"的——它继承了 CLI 的能力,但用 buffer 替代了文本流。

### 1.5 安装

Magit 是 Emacs 社区维护的包,不在 Emacs 内置——这是因为它迭代太快 (Git 本身每年都有新功能)。

```elisp
(use-package magit
  :ensure t
  :bind ("C-x g" . magit-status))
```

`C-x g` 是社区约定的 Magit 快捷键——几乎所有 Magit 用户都用这个。`magit-status` 是 Magit 的主入口。

(Emacs 29+ 内置 `magit-status`,但版本可能旧)

### 1.6 历史

Magit 由 Jonas Bernoult 在 2008 年创建。他是瑞典的开发者,长期维护 Magit + Emacs related 项目 (他也是 Forge、Transient 等的作者)。Magit 被广泛认为是"最好的 Git 客户端",甚至非 Emacs 用户也羡慕。

这反映了 Emacs 文化的力量——一个社区志愿者做的工具,功能超过商业软件。Jonas 的设计哲学是"pop-up driven"——所有操作通过 transient menu,让 Git 的复杂性变得可探索。这个哲学后来被 Emacs 内置采纳 (transient.el 现在是 GNU ELPA 包),影响了整个 Emacs 生态。

---

## 2. magit-status: 主入口

### 2.1 打开

`magit-status` 是 Magit 的核心——它打开一个 buffer,显示当前仓库的所有状态。这是你的"Git 主页":

```
M-x magit-status RET
;; 或
C-x g
```

显示:

```
Head:     master Pull (branch.tracking)
Branch:   master
Upstream: origin/master

Untracked files (1):
        new-file.txt

Unstaged changes (2):
        modified     file1.txt
        modified     file2.txt

Staged changes (1):
        new file     file3.txt

Recent commits:
        abc1234  Last commit message
        def5678  Earlier commit
        ...
```

这个 buffer 是 Magit 的灵魂——所有 Git 操作都从这里发起。它分**section**显示: Header (当前分支、上游)、Untracked (未跟踪文件)、Unstaged (修改未 stage)、Staged (已 stage)、Recent commits。每个 section 可以折叠/展开。

这种"全景状态"是 CLI 没有的——`git status` 只显示文件列表,不显示 commits。Magit 的 buffer 让你**一眼看到仓库全貌**——这是它最大的价值。

### 2.2 在 magit buffer 里导航

Magit buffer 的导航和普通 buffer 一样,但有几个 Magit 特有的快捷键:

```
n / p        下/上一行
TAB          展开当前 section
M-TAB        折叠所有
g            refresh
G            refresh all
q            quit (bury buffer)
```

`TAB` 是核心——每个 section 都可以折叠/展开。比如 "Untracked files (1)" 折叠时只显示计数,展开时列出文件。`M-TAB` 折叠/展开所有 section——快速切换"全景"和"细节"视图。

`g` (refresh) 在外部改动仓库时有用——比如你在终端 `git pull`,在 Magit buffer 按 `g` 刷新。Magit 通常会自动刷新,但手动 `g` 是确定性的。

### 2.3 在文件上操作

在 magit buffer 里,光标在文件行时:

```
TAB          diff 当前文件
RET          visit (打开)
SPC          show (peek)
```

`TAB` 是最有用的——展开文件,显示 diff。你**不需要打开文件**,在 magit buffer 里就能看到改动。这是 Magit 的关键工作流: 看 diff → 决定是否 stage → 按 `s`。

`RET` 打开文件 (跳到对应行)。`SPC` 是 "peek"——临时显示,不切换 window,适合"瞥一眼"。

---

## 3. Stage / Commit / Push

### 3.1 Stage

Stage 是 Git 的核心概念——"准备提交的改动"。Magit 的 stage 操作非常灵活:

```
s            stage 文件 (整文件)
S            stage 所有
u            unstage
U            unstage 所有
```

或部分 stage:

```
-            stage hunk (光标在 diff hunk)
            再次按 unstage
```

部分 stage (`-`) 是 Magit 杀手锏之一——你可以在一个文件里 stage 一部分改动,不 stage 另一部分。这在 CLI 里需要 `git add -p` (交互模式,难用),在 Magit 里就是光标移到 hunk 上按 `-`。

为什么这有用? 想象你改了一个文件——既有 bug 修复,又有新功能,还有一些格式整理。你想**分三个 commit**——一个 commit 一类改动。在 Magit 里,你 stage bug 修复 hunks → commit → stage 新功能 hunks → commit → stage 格式 hunks → commit。三个干净的 commit。CLI 做同样的事是痛苦的。

### 3.2 Commit

Commit 流程在 Magit 里是一个特殊 buffer:

```
c            commit popup
cc           commit (打开 *magit-edit* buffer 写 message)
C-a C-k 改 commit message
C-c C-c      保存并 commit
C-c C-k      取消
```

按 `c c` 后,Magit 打开一个 `*magit-edit*` buffer——这是一个特殊的 text-mode buffer,你在里面写 commit message。写完按 `C-c C-c` 提交,或 `C-c C-k` 取消。

这个设计的好处: commit message 是**完整的编辑器**——你有语法高亮 (git-commit-mode)、自动 fill、拼写检查 (如果装了 flyspell)、模板 (commit message convention)。CLI 的 `git commit -m` 只能写一行,长 message 要 `git commit` 然后用 vim/nano——Magit 直接用 Emacs,你熟悉的键位都在。

### 3.3 Amend

Amend 是"修改上次 commit"——你忘了改一个文件,或者 commit message 写错了。

```
ca           amend (修改上次 commit)
cc           改 message 后 C-c C-c
```

`c a` 打开 amend buffer——预填上次的 message,你修改或加新 staged 改动,`C-c C-c` 提交。Git 把这视为"修改"上次 commit (实际是创建新 commit 替换)。

注意: amend 已经 push 的 commit 需要 force push (`P f`)——这会重写历史,协作时要小心。

### 3.4 Push

Push 是把本地 commit 推到远程:

```
P            push popup
Pu           push upstream
Pp           push to specific
```

`P u` 是最常用的——push 到当前分支的 upstream (通常是 origin/branch-name)。`P p` 允许你指定远程和分支 (用于初次 push 一个新分支)。

### 3.5 完整流程

把上面的步骤串起来——这是一个完整的 feature 开发 → commit → push 流程:

```
1. C-x g            magit-status
2. 看修改
3. s / S            stage
4. c c              开始 commit
5. 写 message
6. C-c C-c          提交
7. P u              push
```

整个流程不离开 Emacs,几秒钟。

这是 Magit 的"杀手级体验"——从改代码到 push 完成整个链路,完全在键盘上完成,完全在 Emacs 内部。没有"打开终端",没有"切换应用",没有"鼠标"。这种**连贯性**是 Magit 比所有外部 Git 客户端强的根本原因。

---

## 4. Branch

### 4.1 Branch popup

Branch 操作集中在 `b` popup:

```
b            branch popup
bb           checkout existing branch
bc           create new branch
bm           rename branch
bk           delete branch
```

`b c` (create) 是最常用的——基于当前分支创建新分支。Magit 会问"基于哪个 commit" (默认当前 HEAD),然后问分支名。

### 4.2 在 branches buffer

要查看所有分支:

```
M-x magit-show-refs RET
;; 或在 magit-status 里 Y
```

显示所有 branch,可 checkout / merge。

这个 buffer 按"本地"、"远程"、"tags"分组,每个分支显示最近 commit。你可以快速对比分支 (`=` 键),找哪个分支领先/落后。

---

## 5. Merge / Rebase

### 5.1 Merge

Merge 是把另一个分支的改动合并到当前分支:

```
m            merge popup
mm           merge
ma           abort merge
```

`m m` 会问"合并哪个分支",然后执行。如果有冲突,Magit 自动进入冲突解决模式 (后面 Ediff 一节讲)。

### 5.2 Rebase

Rebase 是 Git 最强大的功能之一,也是最让新人害怕的。Magit 让 rebase 变得**可探索**——所有选项都在菜单里:

```
r            rebase popup
ri           interactive
rf           rebase --onto
rr           continue rebase
rs           skip
ra           abort
```

`r i` (interactive rebase) 是最常用的——它打开一个 buffer,显示最近 N 个 commit,你可以:
- `p` pick (保留)
- `r` reword (改 message)
- `e` edit (暂停,允许修改 commit)
- `s` squash (合并到上一个)
- `f` fixup (合并到上一个,丢弃 message)
- `d` drop (删除)

这是 Git 最强大的"重写历史"工具——你可以把几个 commit 合并成一个,或者重排顺序。Magit 让这个操作变得**可视化**——你看到一个 commit 列表,改前缀字母,`C-c C-c` 执行。CLI 的 `git rebase -i` 用 vim 打开一个文本文件,改起来累得多。

### 5.3 Cherry-pick / Revert

```
A            cherry-pick popup
V            revert popup
```

Cherry-pick 是"从另一个分支拿一个 commit"——不想 merge 整个分支,只要某个 commit。这在 bug 修复时常用: 修了 master 的 bug,想把这个 fix 应用到 release 分支。

Revert 是"反向一个 commit"——创建一个新 commit 撤销之前的改动。比 `git reset` 安全 (reset 重写历史,revert 创建新 commit,不影响协作)。

---

## 6. Diff / Log

### 6.1 Diff

Diff 是"看改动"——Magit 提供多种 diff:

```
d            diff popup
dd           diff current
dr           diff range
dw           diff working
```

`d d` 显示当前 working tree 和 HEAD 的 diff。`d r` (range) 让你指定任意两个 ref 的 diff (e.g., `master..feature`)。

在 diff buffer 里,你可以:
- `TAB` 折叠/展开 hunk
- `s` / `u` 直接 stage/unstage hunk
- `a` 应用 hunk 到 working tree
- `C-c C-c` 跳到源文件

### 6.2 Log

Log 是"看历史"——Magit 的 log buffer 是一个交互式的 commit 列表:

```
l            log popup
ll           log current branch
lo           log other
lf           log file
```

`l l` 显示当前分支的 commit 历史。每个 commit 显示 hash、author、date、message——颜色编码 (HEAD、当前分支、远程)。

在 log buffer:
```
RET          visit commit
=            diff with working
.            visit at point
```

`RET` 在 commit 上跳到"那个时刻"——你的 buffer 显示该 commit 的文件状态。这是"时间旅行"——你可以看三天前的代码长什么样。

### 6.3 Blame

Blame 是"找谁写的这行"——Git 的杀手级功能之一:

```
M-x magit-blame RET        ; blame 当前文件
```

显示每行的 commit 和 author。

Blame buffer 在每行旁边显示 commit hash、author、date。光标在 blame 行上按 `RET` 跳到那个 commit,按 `b` 显示更老的 blame (在那次 commit 之前的状态)。这是"考古"工具——找一行代码是哪个 commit 加的,为什么加。

---

## 7. Stash

Stash 是"临时存改动"——你正在改 feature,突然要切到 master 修 bug,但当前改动还没 commit。Stash 把改动存起来,working tree 变干净,你可以切分支。修完 bug,切回来,`stash pop` 把改动恢复。

```
z            stash popup
zz           stash
zi           stash index (只 stash 已 stage)
za           apply
zp           pop
zk           drop
```

Magit 让 stash 可视化——stash 列表在 magit-status buffer 里,你可以 apply/pop/drop。CLI 的 stash 是"看不见的堆栈",你 `git stash list` 才看到。

---

## 8. Submodule

Submodule 是 Git 管理子项目的方式——一个仓库里嵌入另一个仓库。复杂但有时必要 (比如 vendor 一个库):

```
o            submodule popup
oa           add submodule
ob           bootstrap (clone)
oi           init
ou           update
od           deinit
or           register
```

Magit 的 submodule 支持远比 CLI 易用——submodule 操作是出了名的复杂 (init/update/sync/recycle),Magit 把每个操作放在菜单里,你能看到所有选项。

---

## 9. Forge (GitHub PR/Issue)

### 9.1 安装

Forge 是 Magit 作者的另一个项目——它把 GitHub/GitLab PR 和 Issue 集成到 Magit。这意味着你**不离开 Emacs**就能 review PR、评论 issue、合并 PR。

```elisp
(use-package forge
  :after magit)
```

第一次 `forge-create-pullreq` 会问 GitHub token。

Forge 用 GitHub API——你需要一个 personal access token。第一次用会引导你创建。Forge 缓存数据到本地 sqlite,所以查询很快。

### 9.2 用法

```
M-x forge-dispatch RET
@            list issues/PRs
c i          create issue
c p          create pull request
```

Forge 在 magit-status buffer 里加一个 "Issues" 和 "Pull requests" section——显示所有 open 的 issue 和 PR。在 PR 上按 RET 进入 PR 详情 buffer,可以看 diff、评论、approve、merge。这是"完全在 Emacs 里做 GitHub review"的实现——对于不想切到浏览器的开发者,这是巨大的效率提升。

---

## 10. Ediff (合并冲突)

### 10.1 启动

冲突时:
```
e            ediff popup
```

或:
```
M-x ediff-files3 RET BASE LOCAL REMOTE
```

Ediff 是 Emacs 内置的三方合并工具——它在三个 buffer (BASE/LOCAL/REMOTE) 里显示冲突,让你选哪个版本。

### 10.2 在 ediff 里

```
a / b / c    从 A / B / C buffer 取
n / p        下/上一 conflict
q            退出
```

`a` 用 A (BASE, 共同祖先),`b` 用 B (LOCAL, 你的改动),`c` 用 C (REMOTE, 别人的改动)。按一个键就解决一个冲突——比手动编辑 `<<<<<<<` 标记快得多。

Ediff 的杀手锏是**实时显示**——你在 ediff control buffer 里按 `a`,合并 buffer 立刻更新为 A 版本。你可以试不同选项,看到效果,不满意再换。这种"交互式合并"比任何 GUI merge 工具都直观。

---

## 11. 实战

### 任务 1: 完整 feature 分支流程

这是一个最常见的 Git 工作流——从 master 创建 feature 分支,改代码,commit,push:

```
1. C-x g                  magit-status
2. b c                    create branch "feature/foo"
3. 输入 feature/foo RET
4. 修改文件
5. C-x g again
6. s                      stage
7. c c                    commit
8. 写 message, C-c C-c
9. P u                    push
10. 在浏览器开 PR (或 forge)
```

按熟练后,步骤 1-9 大概 30 秒 (不算写代码时间)。同样的流程 CLI 要 1-2 分钟 (输入多个命令、commit message、push URL 等)。

### 任务 2: 修复合并冲突

冲突是协作时不可避免的——你和新同事都改了同一行。Magit + Ediff 让解决冲突变成"按几个键":

```
1. m m                  merge feature/foo 到 master
2. 冲突
3. e                    ediff
4. 在 ediff buffer 选 a/b/c
5. q, save
6. 在 magit: r i / rr   rebase continue
```

### 任务 3: 看 commit 历史

```
l l                    log current
在 log buffer里 RET 看具体 commit
= 看与 working diff
```

这是"考古"工作流——找哪个 commit 引入了 bug,理解为什么某段代码这样写。Magit 的 log buffer 让这个过程流畅——按 `RET` 进入 commit,按 `=` 看 diff,按 `o` 在新窗口打开文件。

### 任务 4: amend 上次 commit

"Oops, 忘了 stage 一个文件"——这是 amend 的典型场景:

```
1. 改文件
2. C-x g
3. s
4. c a                  commit amend
5. 默认 message, C-c C-c
6. P f                  force push (要 confirm)
```

`P f` 是 force push——Magit 会问 "Are you sure?" 因为这会重写远程历史。这是 Magit 的安全网——危险操作有确认。

### 任务 5: 创造性用法

Magit 不只是 commit/push——一些意外但强大的用法:

1. **批量 stage hunks**: 在 magit buffer 逐个 hunk 按 `s`,精细控制 commit 内容。
2. **Commit amend**: `c a` 改上次 commit (force push 安全)——拯救"忘了 stage"的错误。
3. **Rebase interactive**: `r i` 编辑历史,合并 commits——让 commit 历史干净。
4. **Cherry-pick 跨分支**: `A` 选 commit 应用——bug 修复合并到 release 分支。
5. **Forge PR review**: 不离开 Emacs review GitHub PR——完全键盘的 PR review。
6. **Ediff 合并冲突**: 三方 merge 可视化——解决冲突比手动编辑快 10 倍。
7. **Worktree**: 同时看多个 branch——`magit-worktree` 让你在不同目录检出不同分支,不需要 stash。
8. **Bisect**: 二分找 bug——`M-x magit-bisect` 让 Git 二分查找自动化,找"哪个 commit 引入 bug"。
9. **Cherry-pick range**: 选一系列 commit 一起 cherry-pick。
10. **Stash with message**: `z z` 时加 message,以后 `z l` 看 stash 列表时知道每个 stash 是什么。
11. **Log for file**: `l f` 看单个文件的历史——找"这行什么时候改的"。
12. **Reflog**: `M-x magit-reflog` 看 HEAD 移动历史——拯救"误删 commit"。

---

## 12. 自定义配置

这是一个完整的 Magit 配置——展示了常见定制选项:

```elisp
(use-package magit
  :ensure t
  :bind (("C-x g" . magit-status)
         ("C-c g" . magit-dispatch)
         ("C-c f" . magit-file-dispatch))
  :custom
  (magit-display-buffer-function #'magit-display-buffer-same-window-except-diff-v1)
  (magit-bury-buffer-function #'magit-mode-quit-window)
  (magit-repository-directories '(("~/projects" . 2)))
  (magit-save-repository-buffers 'dontask)
  (magit-commit-show-diff nil)
  (git-commit-summary-max-length 72)
  :config
  ;; 关掉 vc (用 magit 替代)
  (setq vc-handled-backends nil)
  ;; 自动刷新
  (add-hook 'after-save-hook #'magit-after-save-refresh-status t))
```

每个设置都值得解释:
- `magit-display-buffer-function`: 控制 Magit buffer 在哪里显示——`same-window-except-diff-v1` 让 status 在当前 window,diff 在另一 window。这是个人偏好,有人喜欢全屏 Magit。
- `magit-repository-directories`: 让 Magit 知道你的项目在哪——`("~/projects" . 2)` 意思是 `~/projects` 下两层深度的目录都是仓库。这让 `magit-list-repositories` 工作。
- `magit-save-repository-buffers 'dontask`: 当你 commit 时,Magit 自动保存所有相关 buffer——不需要你手动 save。
- `git-commit-summary-max-length 72`: commit message 第一行限制 72 字符——这是 Git 社区约定。
- `vc-handled-backends nil`: 关掉内置 VC——避免和 Magit 冲突。
- `after-save-hook ... magit-after-save-refresh-status`: 保存文件后自动刷新 magit buffer——保证你看到最新状态。

---

## 13. Git gutter

Magit 是"按需查看"——你 `C-x g` 才看到状态。但有时候你想**实时看到**哪些行改了,不需要切到 magit buffer。这就是 git-gutter/diff-hl 的作用——它们在 fringe (左右边缘) 显示改动的行。

显示 inline 的 git diff:

```elisp
(use-package git-gutter
  :ensure t
  :diminish git-gutter-mode
  :init (global-git-gutter-mode 1)
  :custom
  (git-gutter:modified-sign " ")
  (git-gutter:added-sign " ")
  (git-gutter:deleted-sign " ")
  :config
  (set-face-background 'git-gutter:modified "yellow")
  (set-face-background 'git-gutter:added "green")
  (set-face-background 'git-gutter:deleted "red"))
```

配置后,你写代码时左边缘实时显示: 绿色 = 新增行,黄色 = 修改行,红色 = 删除行。这让你**在编辑时就知道改动范围**——不需要切到 magit。

或 `diff-hl` (类似,更现代):

```elisp
(use-package diff-hl
  :ensure t
  :init (global-diff-hl-mode 1))
```

diff-hl 比 git-gutter 更现代——它和 Magit/VC 集成更好,性能更高。新人推荐 diff-hl。

---

## 14. 自测

1. `C-x g` 干啥?
2. 怎么 stage 单个文件?
3. 怎么 stage 单个 hunk?
4. 怎么 commit?
5. 怎么 push?
6. amend 上次 commit 怎么做?
7. 怎么看 commit log?
8. 怎么 resolve merge conflict?

**答案**:
> 1. magit-status
> 2. 在文件行按 s
> 3. 在 hunk 行按 -
> 4. c c 写 message C-c C-c
> 5. P u
> 6. c a
> 7. l l
> 8. e ediff,选 a/b/c

---

## 15. 下一步

进入 `project-el.md`。
