# MELPA Submission

> 学完这个文件,你能把包发布到 MELPA,让全世界用

---

## 0. MELPA 的故事

讲发布流程前,先理解 MELPA 是什么、为什么是 Emacs 生态的事实标准。

MELPA 是 Emacs 最大的社区包仓库,2012 年由 Donald Ephraem Buchanan 创建。MELPA 的核心创新是"**直接从 GitHub 拉**"——你提交 PR 到 melpa/melpa,加一个 recipe 文件(一行 Lisp 表达式),MELPA 服务器每天从你的 GitHub repo 拉代码,build 成 .tar 包。用户 `M-x package-install` 装的就是这个 build。

这种"**上游 = GitHub,MELPA = 镜像 + build**"的模式有几个好处:第一,作者完全控制代码(不需要 push 到 MELPA);第二,MELPA 自动 build,作者不用打包 release;第三,用户总是拿到最新代码(MELPA 每天更新)。缺点是"不稳定"——MELPA 是 bleeding edge,可能有 bug。所以又有 MELPA Stable(基于 git tag)。但对于活跃的包,用户通常用 MELPA 不用 Stable——最新版修了 bug,但 Stable 落后。

**对比其他生态的包管理**:

- **npm** (Node): 巨大,但 dependency hell 严重。npm 没审核,任何人可以发任何包——质量参差不齐
- **pip** (Python): 简单但依赖管理弱。pip 也是开放发布,无审核
- **cargo** (Rust): 现代,crates.io 有一定审核,语义化版本严格
- **VS Code Marketplace**: 商店模式,微软审核(但审核松)
- **ELPA/MELPA**: 简单,但靠约定。MELPA 不审核代码,只审核"是否符合基本规范"(命名、license、版本)

Emacs 的方式独特——**MELPA 只是镜像 + build 工具,源代码在 GitHub**。这意味着 MELPA 不像 npm 那样"集中存储",它是分散的。好处是去中心化,坏处是如果 GitHub 挂了,所有 MELPA 包都受影响(虽然 archive-contents 缓存能撑一会儿)。

理解这个背景,你才知道为什么要走"PR 到 melpa/melpa"的流程——你不是在"上传"代码,你是在"告诉 MELPA 服务器去哪里拉你的代码"。

---

## 1. MELPA 流程

### 1.1 准备清单

发布前:

- [ ] 包名不冲突 (搜索 melpa.org)
- [ ] header 完整 (Author, Version, Package-Requires, Keywords, URL)
- [ ] `;;;###autoload` 标记好
- [ ] `(provide 'my-package)` 末尾
- [ ] ≥ 10 个 ERT 测试,全通过
- [ ] byte-compile 无 warning
- [ ] README 完整
- [ ] 文档 (Texinfo 或 README 够)
- [ ] LICENSE (GPL v3+ 推荐)
- [ ] GitHub repo (public)

这个清单不是官僚主义——每一项都有理由:

- **包名不冲突**: 重名会让 MELPA build 失败,或导致用户装错包
- **header 完整**: MELPA 从 header 提取元数据,缺字段 build 出问题
- **autoloads + provide**: 保证用户装完能用、能 require
- **测试**: 没测试的包 = bug 多的包,差评率高
- **byte-compile 无 warning**: MELPA 评审硬性要求
- **README + LICENSE**: 用户和法律的最低需求

**没准备好就别提 PR**——MELPA 维护者是志愿者,review 你的烂代码是浪费他们的时间。先满足所有 checklist,再提。

### 1.2 GitHub repo

```bash
mkdir my-package
cd my-package
git init
touch my-package.el README.md LICENSE
git add .
git commit -m "Initial commit"
git remote add origin git@github.com:you/my-package.git
git push -u origin main
```

这是标准的 git init + GitHub 创建流程。几个细节:

- **目录名 = 包名**: 惯例,虽然不是强制的
- **首 commit 是 "Initial commit"**: 不要在首 commit 里加大量功能——历史清晰
- **ssh remote** (`git@github.com:`): 比 https 免输密码(如果你配了 ssh key)

### 1.3 打 tag

```bash
git tag -a v1.0 -m "First release"
git push origin v1.0
```

`-a` 创建 annotated tag(带 metadata,比轻量 tag 好)。`-m` 是 tag message。

为什么打 tag?MELPA Stable 从 tag 拉,不打 tag Stable 拿不到版本。即使你只用 MELPA(不用 Stable),打 tag 也是好习惯——明确标记版本边界。

---

## 2. MELPA Recipe

### 2.1 Recipe 是什么

MELPA 用 recipe 知道怎么构建你的包:

```elisp
(my-package :repo "you/my-package"
            :fetcher github
            :files ("*.el" "README.md" "LICENSE"))
```

Recipe 是一行 Lisp 表达式,告诉 MELPA 三件事:

1. **包名**: `my-package`(recipe 第一个元素)
2. **去哪拉**: `:repo` + `:fetcher`
3. **包含什么文件**: `:files`

这个设计的妙处:**作者完全不用关心 build**。MELPA 服务器读 recipe,clone 你的 repo,按 `:files` 选择文件,打包成 .tar,推到 archive。你只维护源码。

### 2.2 fetcher

`:fetcher` 告诉 MELPA 代码托管在哪:

| Fetcher | 含义 |
|---|---|
| `github` | GitHub |
| `gitlab` | GitLab |
| `sourcehut` | sourcehut |
| `git` | 任意 git URL |
| `wiki` | Emacs Wiki |

`github` 占绝大多数。`sourcehut` 是新兴的去中心化平台,Emacs 圈有人迁过去。`wiki` 是 Emacs Wiki(古老,不推荐——没版本控制)。`git` 是兜底,任何 git server 都能用(适合自托管)。

### 2.3 files

`:files` 列表告诉 MELPA 包含哪些:

```elisp
(my-package
 :repo "you/my-package"
 :fetcher github
 :files (:defaults "lisp/*.el" "snippets/*"))
```

`:defaults` 是魔法关键字——自动包含 `.el` 文件、README、LICENSE。通常 `:defaults` 就够了,只有特殊布局才需要显式列。

`:files ("lisp/*.el" "snippets/*")` 是 wildcard——包含 lisp/ 下所有 .el,和 snippets/ 下所有文件。这是处理"包代码分散在子目录"的情况。

### 2.4 recipe 写哪

放本地测试,正式发布 PR 到 `melpa/melpa` 的 `recipes/` 目录。

注意: **文件名就是包名,没有扩展**。`recipes/my-package`(不是 `my-package.recipe`)。MELPA build 脚本扫描 `recipes/` 目录,每个文件名就是一个包名。

---

## 3. 提 PR 到 MELPA

### 3.1 fork melpa/melpa

GitHub 上 fork https://github.com/melpa/melpa

fork 是 GitHub 的核心协作机制——你 fork 出自己的副本,改完后 PR 回原 repo。MELPA 用这个流程。

### 3.2 clone fork

```bash
git clone https://github.com/you/melpa.git
cd melpa
git remote add upstream https://github.com/melpa/melpa.git
git fetch upstream
git checkout -b add-my-package upstream/master
```

`upstream` 指向原 repo,让你能 fetch 最新。`add-my-package` 分支专门为加你的包——分支名清晰表达意图。

### 3.3 加 recipe

```bash
echo '(my-package :repo "you/my-package" :fetcher github)' > recipes/my-package
```

注意: **文件名就是包名,没有扩展**。`recipes/my-package`(不是 `my-package.recipe`)。

### 3.4 测试 recipe

```bash
# 在 melpa repo 里:
./run/maint/refresh-packages.sh my-package
```

应该成功 build,在 `packages/` 生成 `my-package-1.0.tar`。

这一步**必做**——很多 PR 因为 recipe 写错被拒。本地测过能 build,PR 通过率高。`./run/maint/refresh-packages.sh` 是 MELPA 的 build 脚本,它会真的 clone 你的 repo、按 recipe 打包。如果失败,看错误信息修。

### 3.5 commit + push

```bash
git add recipes/my-package
git commit -m "Add my-package"
git push origin add-my-package
```

commit message 简洁:"Add my-package"。MELPA 的 commit 历史是机器可读的——一致的格式让维护者能快速扫描。

### 3.6 提 PR

GitHub 上从你的 fork 提 PR 到 melpa/melpa。

PR 模板:

```
This PR adds `my-package`, a [description].

- Repo: https://github.com/you/my-package
- License: GPL v3+
- Tested with Emacs 27.2, 28.2, 29.4, 30.1
```

PR 描述要让维护者**5 秒内理解**——包做什么、在哪、license、测了哪些版本。如果你写得啰嗦,维护者跳过你的 PR 看 下一个。

### 3.7 等 review

MELPA 维护者会 review。
常见反馈:
- 改 license (GPL 必须有 header)
- 改命名
- 加测试

修改后再 push,PR 自动更新。

review 反馈是好事——意味着维护者看了你的代码。最糟的不是被拒,是被忽略。认真处理每条反馈——维护者是志愿者,他们花时间 review 你的代码是恩惠。

### 3.8 合并

合并后,~1 小时内 MELPA 会 build 你的包。
然后 `M-x package-install RET my-package RET` 就能装。

合并那一刻你的包**正式上线 MELPA**——全世界 Emacs 用户都能装。这是 Emacs 包开发者的高光时刻。

---

## 4. NonGNU ELPA (替代)

NonGNU ELPA 比 MELPA 严格,但官方支持。

NonGNU ELPA 是 2021 年 FSF 创建的,目的:**给"不想签 FSF copyright assignment 但想被官方认可"的作者一个出路**。它默认包含在 Emacs 28+ 的 package-archives 里——用户不用配置就能用。

### 4.1 流程

提交到 https://git.savannah.gnu.org/git/emacs/elpa.git

1. 注册 savannah 账号
2. 加 `packages/my-package/` 目录
3. 提交 patch 到 elpa-devel

 Savannah 是 FSF 的代码托管平台,老派但稳定。流程比 MELPA 麻烦(savannah UI 不好用),但官方认可度高。

### 4.2 优势

- FSF 官方
- Emacs 内置 package-archive 默认含它
- 更稳定的版本

### 4.3 劣势

- 必须 GPL v3+
- 需要版权 assignment (FSF paper)
- review 更慢

**策略**: 先发 MELPA(快,广覆盖),等包稳定 6-12 月再考虑 NonGNU ELPA(增加官方背书)。两者可以共存——用户的 archive 配置可以同时有 MELPA 和 NonGNU。

---

## 5. 版本管理

### 5.1 SemVer

`MAJOR.MINOR.PATCH`:
- MAJOR: 破坏 API
- MINOR: 加特性,不破坏
- PATCH: bug fix

SemVer 是社区标准,**MELPA 也用**。用户看到 1.2.3 就知道:1 是大版本,2 是特性版本,3 是修复版本。

### 5.2 在 .el 里设版本

```elisp
;; Version: 1.2.3
```

MELPA 从这读。

**重要**: version 在 header comment 里,不是 `(defconst my-version "1.2.3")`。MELPA 用正则扫 header comment,只认 `;; Version:`。如果你同时有 defconst,确保两者同步——不同步会迷惑用户。

### 5.3 tag

```bash
git tag -a v1.2.3 -m "Release 1.2.3"
git push origin v1.2.3
```

tag 名用 `v` 前缀——惯例(MELPA Stable 也用)。`v1.2.3` 比 `1.2.3` 更明显是版本号(不是文件名或参数)。

### 5.4 changelog

`CHANGELOG.md`:

```markdown
# Changelog

## [1.2.3] - 2026-06-19
- Fix bug in my-func
- Add my-new-command

## [1.2.0] - 2026-06-01
- Add feature X

## [1.0.0] - 2026-05-01
- Initial release
```

CHANGELOG 让用户知道"升级到新版有什么变化"。优秀的 CHANGELOG 按 SemVer 排序,每个版本列新增/修复/破坏。用户决定是否升级时,看 CHANGELOG 是最快方式。

---

## 6. 发布 checklist

- [ ] GitHub repo public
- [ ] Header 完整
- [ ] LICENSE 文件 (GPL v3+)
- [ ] README 完整
- [ ] 至少 10 个 ERT 测试
- [ ] byte-compile 无 warning
- [ ] CI 跑通 (GitHub Actions)
- [ ] 版本 tag (v1.0)
- [ ] MELPA recipe 写好,本地测试通过
- [ ] PR 到 melpa/melpa
- [ ] PR 被 review
- [ ] PR 合并
- [ ] 在 Emacs 里 `M-x package-install` 能装

最后一步"在 Emacs 里 `M-x package-install` 能装"是终极验证——你的包真的上线了,真的能装,真的能用。

---

## 7. 维护

发布不是终点,是开始。**长期维护比首发更难**。

### 7.1 处理 issue

- 回应每个 issue
- 重现 bug
- 加测试
- 修复

回应 issue 的速度决定用户感受。即使"我下周看"也比沉默好——用户知道你在意。重现 bug 时如果无法重现,问用户具体步骤、Emacs 版本、`emacs -Q` 下能否重现。

### 7.2 处理 PR

- review 代码
- 跑 CI
- 合并或建议改

PR 是社区贡献,要鼓励。即使代码不完美,提供具体的修改建议——"这里改成 X 会更好,因为 Y"。不要直接拒绝,那是伤贡献者的心。

### 7.3 发布新版本

- 改 `Version:` in .el
- 加 tag
- push
- MELPA 自动更新

MELPA 每天 build,所以你 push 后 24 小时内用户就能装到新版。如果想立刻让用户拿到,可以加 `(setq package-install-upgrade-warning ...)` 提示用户 refresh。

### 7.4 长期维护

- 不要 abandon
- 即使不再开发,接受 PR
- 标记 unmaintained 如果不维护

**最糟糕的事是 abandon**——用户装了你的包,你 3 年不维护,Emacs 升级了你的包崩了,所有用户受影响。如果真不能维护:

1. 在 README 加 "Looking for maintainer"
2. 标记 deprecated,推荐替代品
3. 转交社区(找愿意接手的)

负责任的 abandon 比假装维护好。

---

## 8. License 和 FSF 的角色

讲 license 和 FSF(自由软件基金会)——这是 Emacs 生态的法律基础。

GPL (GNU General Public License) v3+ 是 Emacs 包的标准 license。GPL 的核心是 "**copyleft**"——你可以免费用、改、发,但**你必须用同样的 license 发布修改**。这和 MIT/Apache 不同(那些允许闭源衍生)。

为什么 Emacs 包用 GPL? 因为 Emacs 本身是 GPL,**链接到 Emacs 的包"传染"GPL**。所以你的包要么 GPL,要么兼容(一些更宽松的 license)。

FSF (Free Software Foundation) 是 GPL 的守护者——他们法律追究违反 GPL 的人。这是 Emacs 生态的"法律基础"——没有 FSF,GPL 就是一纸空文。

**实践经验**: 你的包用 GPL v3+,LICENSE 文件放完整的 GPL 文本(从 fsf.org 复制),header 加 `SPDX-License-Identifier: GPL-3.0-or-later`。这三样齐了,MELPA 接受、用户放心、法律清晰。

如果你用其他 license(如 MIT),MELPA 也接受,但要确认和 Emacs 链接不冲突——MIT 兼容 GPL,所以可以。Apache 2.0 也兼容。**不兼容的 license**(如 proprietary)不能发 MELPA。

---

## 9. 实战

### Ex 9.1: 完成你的 my-todo 包

确保:
- [ ] header 完整
- [ ] ERT 测试 ≥ 10 个
- [ ] byte-compile 无 warning
- [ ] README 完整

### Ex 9.2: GitHub repo

```bash
cd ~/.config/emacs/lisp
mkdir my-todo-package
cp my-todo.el my-todo-package/
cp my-todo-test.el my-todo-package/test/
cd my-todo-package
echo "# my-todo" > README.md
echo "GPLv3 license text..." > LICENSE
git init
git add .
git commit -m "Initial release"
gh repo create my-todo --public --source=. --push
```

`gh repo create` 用 GitHub CLI 一键创建 public repo 并 push。前提是你装了 gh 并 `gh auth login`。

### Ex 9.3: 提 MELPA PR

按第 3 节流程。重点:**先本地测 recipe**,再 PR。

### Ex 9.4: 等 review + 修

通常 1-7 天 review。处理反馈,修完 push,PR 自动更新。

---

## 10. 自测

1. MELPA recipe 放哪?
2. recipe 的 :fetcher 有哪些?
3. 怎么测试 recipe?
4. PR 模板包括什么?
5. NonGNU ELPA 和 MELPA 区别?

**答案**:
> 1. melpa/melpa 的 recipes/ 目录,文件名 = 包名(无扩展)
> 2. github, gitlab, sourcehut, git, wiki
> 3. 在 melpa fork 里跑 `./run/maint/refresh-packages.sh NAME`
> 4. repo URL, license, tested versions(简洁 5 秒读懂)
> 5. NonGNU 更严格 (FSF,需要 copyright assignment),官方内置 archive;MELPA 社区自动 build,bleeding edge

---

## 11. 下一步

进入 `capstone.md` 做毕业项目。
