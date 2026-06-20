# First Pull Request

> 手把手带你提第一个 PR 到别人的包

第一个 PR 是个里程碑——它让你从"用户"变成"贡献者"。流程看起来繁琐,但每个步骤都有道理。这个文件**一步一步**带你走完,选择最简单的 PR (typo 修复) 作为示例,然后升级到 bug fix。

完成第一个 PR 后,你会发现开源贡献没那么神秘——它就是流程。后续 PR 会越来越快,因为流程已经熟练。

---

## 1. 准备

### 1.1 选包

选你**重度使用**的包。原因:你已经懂它的 API 和工作流,知道哪里不清楚、哪里有 bug;不熟悉的包你提不出有价值的改进。

例子: 假设你用 `vertico`。

> ```bash
> gh repo clone minad/vertico
> cd vertico
> ```

这里用 `gh repo clone` (GitHub CLI)。如果你想从 fork 开始 (而不是直接 clone upstream),用 `gh repo fork`——它自动 fork 到你账号 + clone + 加 remote。我们这里先 clone upstream,后面需要 fork 时再加。

### 1.2 跑测试

> ```bash
> make test
> ```

确保所有测试通过 (没你改之前)。这一步**非常重要**——如果原始代码就有失败的测试,你改完后 maintainer 会以为是你弄坏的。先确认 baseline。

如果测试不通过,可能是你的环境问题 (Emacs 版本、缺少依赖)。读项目的 CONTRIBUTING.md 看需要什么环境。

### 1.3 看 CONTRIBUTING

很多项目有 `CONTRIBUTING.md` 或 `.github/CONTRIBUTING.md`:
- **代码风格**——缩进、命名、长度限制
- **commit message 风格**——Conventional Commits? Emacs 风格?
- **PR 流程**——branch 命名、PR 模板、需要 sign-off?

**不读 CONTRIBUTING 就发 PR 是新手最大的失误**——maintainer 第一眼看你的 PR 就知道你读没读。读了 CONTRIBUTING 你的 PR 第一眼就"专业",合的概率高一倍。

---

## 2. 选 issue

### 2.1 找 good first issue

GitHub 上 `Issues` → filter `good first issue`。Maintainer 主动标记的"适合新人"任务——通常范围明确、不依赖深入理解。

但**不是所有 good first issue 都真的好做**。打开 issue 看历史——如果有人尝试过但放弃了,可能没那么简单。看 issue 的评论数,讨论多的话题通常复杂。

### 2.2 找 typo / 文档 bug

最简单的开始:
- README typo
- docstring 不清楚
- 例子错

这种 PR maintainer 一秒合——不需要 review 代码逻辑,只需要看拼写对不对。**最适合第一个 PR**。

怎么找 typo? 读项目的 README,仔细看每句话。或者用拼写检查工具 (aspell, ispell) 跑一遍。

### 2.3 找 bug

> ```bash
> # 看未解决 issues
> gh issue list -R minad/vertico
> ```

找一个你能重现的。Bug 的好 PR 候选:
- 有清晰重现步骤
- maintainer 还没自己修
- 没人 claim

bug PR 比 typo PR 难,但有学习价值——你能学到调试、测试、修复的完整流程。

### 2.4 自己想的特性

不要直接写,先 issue 讨论。Maintainer 不喜欢"unannounced feature PR"——你做了大量工作,但 maintainer 可能觉得这个 feature 不该加,你的工作白费。先开 issue 问"这个 feature 值得做吗?",讨论通过后再实现。

---

## 3. 实现流程

### 3.1 创建 branch

> ```bash
> git checkout -b fix-something
> ```

不要在 main 改。原因:如果你在 main 改,后面想同时维护多个 PR 就乱了;maintainer 让你 rebase 也麻烦。**每个 PR 一个 branch** 是铁律。

branch 名字要描述性——`fix-something`、`fix-window-crash`、`docs-clarify-foo`。不要用 `patch`、`tmp`、`wip`。

### 3.2 改代码

按项目风格改。读 CONTRIBUTING 和现有代码——如果项目用 2 空格缩进,你也用 2 空格;如果用 dash (`-`) 而不是 underscore (`_`),你也用 dash。

### 3.3 加测试

如果你的改动影响行为,加 ERT 测试。测试让 maintainer 安心——他知道以后改这个代码的人不会破坏你的修复。

测试最小化:一个测试只验证一件事。多个独立测试比一个巨大测试好。

### 3.4 跑测试

> ```bash
> make test
> ```

全通过。如果有测试失败,**先修测试**再发 PR——发一个有失败测试的 PR 是大忌。

### 3.5 byte-compile 检查

> ```bash
> emacs --batch -L . -f batch-byte-compile *.el
> ```

这条命令在 batch 模式下编译所有 .el 文件。`-L .` 把当前目录加到 load-path,`-f batch-byte-compile` 调用 byte-compile 函数。

无 warning。Warning 包括:未使用的变量、未定义的函数、过时的 API。Maintainer 要求 byte-compile clean——warning 往往预示潜在 bug。

### 3.6 commit

按项目风格 (CONTRIBUTING 通常规定):

> ```bash
> git commit -m "Fix X when Y is nil"
> ```

或 Conventional Commits 风格:

> ```bash
> git commit -m "fix(vertico): handle nil case in vertico--resize"
> ```

commit message 第一行是 summary (50 字符以内),空行后是详细描述 (如果有)。用现在时态——"Fix X" 而不是 "Fixed X"——git 习惯。

### 3.7 push

> ```bash
> git push origin fix-something
> ```

push 到 origin (upstream)。但等等——你不能 push 到 upstream (你不是 maintainer)!所以你需要 fork。

---

## 4. 提 PR

### 4.1 fork

GitHub 上 fork 项目 (如果还没)。Fork 把项目复制一份到你账号,你有 push 权限。从 fork 提 PR 到 upstream。

### 4.2 加 fork remote

> ```bash
> git remote add fork git@github.com:you/vertico.git
> git push -u fork fix-something
> ```

第一条命令把你 GitHub 上的 fork 加为 git remote (名叫 "fork")。第二条把你的 branch push 到 fork,`-u` 设置 upstream tracking。

之后 `git push` 就直接 push 到 fork。

### 4.3 在 GitHub 提 PR

> ```bash
> gh pr create --repo minad/vertico \
>   --base main \
>   --head you:fix-something \
>   --title "Fix X when Y" \
>   --body "Fixes #123.
>
> Problem: ...
> Solution: ...
> Test: ..."
> ```

`gh pr create` 命令行提 PR。参数:`--repo` 是 upstream,`--base` 是 upstream 的目标 branch,`--head` 是你 fork 的 branch (格式 `user:branch`),`--title` 和 `--body` 是 PR 标题和描述。

PR body 要清晰——maintainer 第一眼看 body 决定是否细看代码。

### 4.4 PR 模板

很多项目有 PR 模板 (`.github/pull_request_template.md`)——`gh pr create` 会自动加载。如果没有,自己写一个清晰的:

> ```markdown
> ## Description
>
> Fixes #123.
>
> Problem: When Y is nil, vertico would crash.
>
> Solution: Add null check before calling (foo Y).
>
> ## Testing
>
> - Added ERT test `vertico-test-y-nil`
> - Manually tested with M-x vertico
>
> ## Checklist
>
> - [x] Tests pass
> - [x] byte-compile clean
> - [x] Commit message clear
> ```

PR 模板的关键:**Problem/Solution/Testing** 三段。这让 maintainer 三秒理解你做了什么。

---

## 5. Review 流程

### 5.1 等 review

维护者会 review。可能 1-14 天——大 maintainer (Magit 的 Jonas) 可能忙,几天才回;小项目可能几小时。**别催**——催显得没耐心,maintainer 是志愿者。

如果两周没回应,可以礼貌 ping 一次:"Hi, just checking if this needs more work from my side." 一次,不要多次。

### 5.2 处理 feedback

reviewer 留 comment:
- 改 X
- 这里命名不好
- 加 docstring

这些都是正常的——senior engineer 的代码也被 review。不要 defensive,把 review 当学习机会。

修:

> ```bash
> git commit --amend              # 加到上次 commit
> # 或
> git commit -m "Address review"
> ```

`git commit --amend` 把当前修改加到上次 commit——保持 PR 是单 commit (很多项目要求 squash)。如果是多 commit PR,用普通 commit 加新 commit。

> ```bash
> git push origin fix-something
> ```

push 后 PR 自动更新——GitHub 显示新的 diff。Maintainer 会收到通知。

### 5.3 多轮 review

可能多轮。Maintainer 第一次 review 后,你改;他再 review;再改...直到满意。这是正常过程——好代码是反复打磨出来的。

保持耐心,礼貌回应。每个 comment 都回应——同意就说 "Done, thanks!",不同意就解释理由。**不要忽视 comment**——maintainer 会觉得你不在乎。

### 5.4 合并

维护者满意 → 合并。你成为该项目的贡献者!从这一刻起,你的 GitHub 头像出现在项目的 Contributors 列表,你的名字永久在 git log 里。

---

## 6. 例子: vertico 文档 typo

完整走一遍最简单的 PR——typo 修复。

### 6.1 找 typo

读 `README.md`,发现 typo:

> ```
> "vertico is a minimalistic vertical completion UI"
>                 ^^^^^^^^^^^^
> "minimalistic" 拼成 "minmalistic"
> ```

这种 typo maintainer 自己看不出来 (他写的时候脑补正确拼写),用户也懒得报。你修了,他一秒合。

### 6.2 流程

> ```bash
> gh repo clone minad/vertico
> cd vertico
> git checkout -b fix-readme-typo
> # 改 README.md
> git commit -m "Fix typo in README"
> git push origin fix-readme-typo
> gh pr create --repo minad/vertico \
>   --title "Fix typo in README" \
>   --body "s/minmalistic/minimalistic/"
> ```

PR body 极简——typo 不需要长篇大论。`s/old/new/` 是 sed 替换语法,程序员都懂。

合并成功!你的第一个 PR。

### 6.3 这是你的第一个 PR

简单,但**完整流程**——fork、branch、commit、push、PR、合并。后面所有 PR 都是这套流程,只是 diff 大小不同。

---

## 7. 例子: vertico bug fix

更复杂的例子——真 bug。

### 7.1 找 bug

`vertico-mode` 在 `window-size-change` 触发时崩溃。这种 bug 来自真实用户报告——你重现、定位、修复、加测试、提 PR。

### 7.2 重现

> ```
> 1. (vertico-mode 1)
> 2. 调整 window 大小
> 3. crash
> ```

清晰的重现步骤是 bug fix 的基础。如果你不能稳定重现,fix 也是猜的。

### 7.3 调试

> ```elisp
> (toggle-debug-on-error)
> ;; 重现
> ;; 看 backtrace
> ```

`toggle-debug-on-error` 让 Emacs 在出错时进 backtrace,而不是默默吞掉错误。看 backtrace 知道崩在哪一行。

发现:

> ```elisp
> (defun vertico--resize ()
>   (let ((height (window-size-change)))  ; ← 这里返回 nil
>     ...))
> ```

`window-size-change` 在某些情况返回 nil,后续代码假设它是 number,崩。

### 7.4 修

> ```elisp
> (defun vertico--resize ()
>   (let ((height (or (window-size-change) 0)))
>     ...))
> ```

用 `or` 提供 default——nil 时 fallback 到 0。最小化的修复,不引入新逻辑。

### 7.5 加测试

> ```elisp
> (ert-deftest vertico-test-resize ()
>   (should (vertico--resize)))    ; 不应 crash
> ```

测试验证 fix——以后改这个函数的人如果破坏了 nil 处理,测试会失败。

### 7.6 commit + push + PR

> ```bash
> git commit -m "Fix crash when window-size-change returns nil
>
> Fixes #123."
> git push origin fix-window-size-crash
> gh pr create ...
> ```

commit message 第一行 summary,空行后详细,引用 issue #123。这种格式让 GitHub 自动关联 issue——issue 在 PR 合并时自动关闭。

---

## 8. Commit 风格

不同项目用不同 commit 风格。读 CONTRIBUTING 看用哪个。

### 8.1 Conventional Commits

> ```
> type(scope): description
>
> [optional body]
>
> [optional footer]
> ```

type 是固定的几个值:`feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`。scope 是可选的范围标识 (包名、模块名)。

例:

> ```
> feat(vertico): add gradient face
> fix: handle nil case in vertico--resize
> docs(readme): fix typo
> test: add coverage for vertico--resize
> ```

Conventional Commits 是 Angular 项目发明的,被很多现代项目采用 (包括 Emacs 之外的)。优点是工具能自动生成 changelog。

### 8.2 Emacs 风格

老 Emacs 用自己的风格:

> ```
> Fix bug in foo-bar (Bug#12345)
>
> * lisp/foo.el (foo-bar): Check for nil.
> ```

格式 `* file (function): description.`——文件名加括号函数名加描述。这是 GNU changelog 格式,从 1980 年代就有了。Emacs core 强制用这个,有些 Emacs 包也用。

---

## 9. 常见错误

### 9.1 不读 CONTRIBUTING

每个项目风格不同。先读。Maintainer 第一眼看代码风格就知道你读没读。

### 9.2 PR 太大

不要一个 PR 改 1000 行。拆成多个小 PR——每个 PR 做一件事。大 PR review 慢、容易拒、合并难。

### 9.3 不写测试

测试让 reviewer 安心。小 fix 也加测试——一行 fix 配一个测试,完全合理。

### 9.4 commit message 差

"fix stuff" 不好。"Fix X in function Y when Z is nil" 好。commit message 是 PR 的"摘要"——maintainer 看一眼决定深不深看。

### 9.5 不耐心

review 慢。别催。Maintainer 是志愿者,有自己的工作和生活。两周以上没回应可以礼貌 ping 一次。

### 9.6 不礼貌

reviewer 说"这样更好",不要顶。学。即使你不同意,也要理性解释,不要情绪化。开源社区记忆长——一次冲突可能影响你以后所有 PR。

### 9.7 不响应

reviewer 留了 comment,你几天不回——maintainer 觉得你放弃了,可能 close PR。即使忙,也要回一句"会改,这周忙,下周一处理"。

### 9.8 改无关代码

PR 里加"顺便修了另一处的 typo"——不要!PR 应该 focused。无关改动放另一个 PR。

---

## 10. 实战

### Ex 1: 文档 typo PR

找一个你常用包的 README:
- 找 typo
- 提 PR

目标:走完完整流程,理解 fork/branch/commit/push/PR。

### Ex 2: docstring PR

> ```
> C-h f foo RET
> ```

看 docstring:
- 不清楚?
- 改 docstring
- 提 PR

docstring PR 比文档 PR 更有价值——docstring 是函数的"现场文档",每个用户都看。

### Ex 3: bug fix PR

找一个简单 bug:
- 重现
- 写测试
- 修
- 提 PR

这是"真正"的开源贡献——你修了真 bug。

### Ex 4: feature PR

跟维护者讨论一个新特性:
- 提 issue
- 讨论
- 实现
- 提 PR

最难,但最有价值。Feature PR 让项目长成你想要的样子。

---

## 11. 自测

1. 第一步: fork 还是 clone? (答: fork 然后 clone fork,或 clone upstream 后加 fork remote)
2. branch 名字怎么取? (答: 描述性,fix-foo-bar、add-feature)
3. commit message 风格? (答: Conventional Commits fix:/feat:/docs: 或 Emacs 风格)
4. PR 怎么提? (答: `gh pr create` 或 GitHub web UI)
5. review 怎么处理? (答: 改 commit、push、等下一轮)

---

## 12. 下一步

进入 `emacs-devel.md` 学 Emacs core 贡献。
