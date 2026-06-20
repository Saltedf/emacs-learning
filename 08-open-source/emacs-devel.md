# Emacs Core 贡献 (emacs-devel)

> 学完这个文件,你能给 Emacs 本身贡献代码

给 Emacs core 贡献是最有挑战性也最有回报的开源活动。挑战在于流程重 (邮件、patch、copyright paper)、review 严格、文化独特。回报在于你的代码进了**Emacs 本身**——50 年历史的项目,全世界几十万用户,你的名字永久在 AUTHORS 文件里。

这个文件教你整个流程。读完你不仅会贡献,还会理解为什么 Emacs 选择这种"老派"方式,以及它的优势。

---

## 1. Emacs core 是什么

"Emacs core" 指 Emacs 项目的所有官方代码:
- **C 核心** (`src/*.c`)——评估器、GC、display engine、文件 I/O 等
- **内置 Lisp** (`lisp/*.el`)——所有 `M-x` 命令、`org`、`dired`、`isearch` 等
- **文档** (`doc/`)——Emacs manual、Elisp manual
- **测试** (`test/`)——ERT 测试

Git repo: https://git.savannah.gnu.org/git/emacs.git
镜像: https://github.com/emacs-mirror/emacs (只读 mirror,不能 PR)

Emacs core 是**地球上最古老的开源项目之一**——1985 年第一个 release,持续维护到现在。每一行代码都有几十年的"演化压力",改任何东西都要小心。

---

## 2. 不接受 GitHub PR

**重要**: Emacs core 不接受 GitHub PR。
所有贡献走**邮件列表**。

理由: GNU 项目传统,git send-email 流程。

这个选择有深层原因:

第一,**基础设施独立性**。GitHub 是 Microsoft 的商业服务,GNU 项目坚持用自己的 Savannah。如果某天 Microsoft 改 GitHub 条款 (比如关闭开源仓库、收费),GNU 项目不受影响。这种"基础设施独立性"是 GNU 哲学的核心实践。

第二,**永久归档**。邮件列表的归档 (mbox 格式) 永久存在 lists.gnu.org,从 1990 年代至今所有 emacs-devel 邮件都能查到。GitHub 的 issue/PR 没这种保证——公司可以删数据,可以改 API。邮件只要 SMTP 在就能读。

第三,**git 原生**。git 本身就是为邮件 workflow 设计的——Linus 早期就是这样用。`git format-patch` 和 `git send-email` 是 git 的标准命令。GitHub PR 是 GitHub 加的一层 UI,不是 git 本身。

第四,**开放访问**。任何人都能发邮件到 emacs-devel——不需要 GitHub 账号,不需要登录。这在网络受限地区 (如某些国家屏蔽 GitHub) 是优势。

对比其他开源生态:
- **Linux kernel**: 也用邮件,需要 maintainer Acked-by,极严格
- **VS Code**: Microsoft 主导,社区 PR 难合,Microsoft 内部决策
- **Rust**: RFC 流程,社区治理,有正式治理结构
- **Emacs**: 邮件 + maintainer 仲裁,传统 Unix 风格

---

## 3. 准备

### 3.1 订阅 emacs-devel

`emacs-devel` 是 Emacs 开发者的邮件列表。这里讨论 Emacs 的设计、规划未来版本、争论特性取舍。Eli Zaretskii (主 maintainer) 在这里和贡献者讨论。Stefan Monnier、Andrea Corallo 等核心维护者也在。

订阅这个列表是"进入 Emacs 内圈"的第一步。你不一定要发言,但**读**能让你理解 Emacs 的演化逻辑。比如"为什么不加某个看起来很有用的特性",你能在邮件里看到详细的技术论证。这种"历史决策"是其他编辑器看不到的——VSCode 的决策在 Microsoft 内部,你不知道。

发邮件到 `emacs-devel-join@gnu.org`,或 web 订阅:
https://lists.gnu.org/mailman/listinfo/emacs-devel

只读 1-2 周,感受文化。

邮件列表的礼仪和 Discord/Slack 不同。**回邮件用纯文本,不用 HTML**——HTML 邮件可能被自动拒收。**引用原邮件只引用相关部分**——不要 top-post (在原邮件上方回),要 inline 回 (在原邮件相关段落下回)。**签名用 `-- ` 分隔** (注意 `--` 后有个空格)。这些约定来自 1980s Usenet 时代,GNU 项目保留至今。

### 3.2 设置 git send-email

`git send-email` 把 patch 作为邮件发到指定地址。需要配置 SMTP 服务器。

> ```bash
> git config --global sendemail.smtpserver smtp.gmail.com
> git config --global sendemail.smtpserverport 587
> git config --global sendemail.smtpencryption tls
> git config --global sendemail.smtpuser you@example.com
> git config --global sendemail.smtppass your-app-specific-password
> ```

(具体看你邮箱——Gmail 用上面的,GitHub 邮箱可能不同)

Gmail 不允许直接用密码,需要"app-specific password" (在 Google 账号设置里生成)。这是 Gmail 的安全要求,不是 git 的。

配置完后 `git send-email --to=... patch-file` 就能发邮件。第一次测试可以发给自己 (cc 自己) 看格式对不对。

### 3.3 clone Emacs

> ```bash
> git clone https://git.savannah.gnu.org/git/emacs.git
> cd emacs
> ```

或镜像 (国内访问快):

> ```bash
> git clone https://github.com/emacs-mirror/emacs.git
> ```

镜像同步 Savannah,内容一样。但**你不能 push 到任何**——push 需要 commit 权限 (只有 maintainer)。你 clone 是为了改 + 测试 + format-patch。

### 3.4 编译

> ```bash
> ./autogen.sh
> ./configure --prefix=/tmp/emacs-dev --with-native-compilation --with-tree-sitter
> make -j$(nproc)
> ./src/emacs -Q
> ```

`./autogen.sh` 生成 configure 脚本 (需要 autotools)。`./configure` 检测系统、生成 Makefile。`make -j$(nproc)` 并行编译 (`$(nproc)` 是 CPU 核数)。`./src/emacs -Q` 跑你编译的 Emacs (不带配置,验证 baseline 工作)。

编译时间 ~10 分钟 (1.5M 行 C + Lisp)。第一次编译后增量编译快——`make` 只重编改变的文件。

测试编译成功——`./src/emacs -Q` 能开起来就行。

---

## 4. 找任务

### 4.1 看 TODO

Emacs 有 TODO 系统——maintainer 把没时间做的任务记下来,等贡献者来做。

> ```bash
> cat admin/notes/TODO
> cat etc/TODO
> ```

`etc/TODO` 是用户可见的 wishlist——里面有"如果有人想做,这些是好任务"。`admin/notes/TODO` 是 maintainer 内部笔记。

或在 Emacs 里:

> ```elisp
> M-x view-emacs-todo
> ```

这个命令显示 TODO 文件,有 navigation。

### 4.2 看 bug tracker

https://debbugs.gnu.org/

GNU 项目的 bug tracker (叫 debbugs)。每个 bug/patch 是一个数字,有 thread。找标 "wishlist" 或 "easy" 的 bug——这些是 maintainer 标的"适合新人"任务。

或 news group `gnu.emacs.bug`——bug tracker 的邮件镜像。

### 4.3 看 emacs-devel

讨论中可能有人说"这个应该改"。如果你能实现,直接发 patch 回复那个 thread。

### 4.4 自己发现

你用 Emacs,发现 bug 或想改进。先讨论 (邮件)——直接发 patch 可能被拒,因为 maintainer 可能觉得这个改动不该做。

---

## 5. copyright assignment

### 5.1 何时需要

FSF 要求大贡献者签 copyright assignment:
- 小贡献 (< 15 行): **不需要** paper
- 中等 (15-100 行): **看情况**——maintainer 决定
- 大贡献 (≥ 100 行, 或多次): **必须**签 paper

### 5.2 为什么需要

这是 GNU 项目独特的"集体所有权"模式——和 Linux (Linus 持有商标) 或 Rust (各公司持有) 完全不同。它源于 Stallman 的理念: **软件应该属于用户群体,不属于个人或公司**。

具体好处:
1. **FSF 持有完整版权**,可以在法庭上**强制**执行 GPL——违反 GPL 的人会被 FSF 起诉。如果版权分散在 1000 个贡献者手里,没人有法律地位起诉侵权者。
2. **License 升级灵活**——如果 GPL v4 出来,FSF 可以单方面决定 Emacs 升级到 v4。Linux 内核因为版权分散 (几千个贡献者),至今无法从 GPLv2 升到 v3。
3. **你的名字永久在 AUTHORS 文件**——签了 paper 后,你就是 Emacs 的"法定作者"。

这就是为什么很多 Emacs 包要求签 paper 才能合——保护整个生态的法律健康。

### 5.3 流程

联系 `assign@gnu.org`:
- 表格: PDF
- 签字,扫描或邮寄
- FSF 录入

一次性,后续贡献都覆盖。

### 5.4 时间

~2 周处理。这是 FSF 的人工流程,有 paperwork。第一次签等两周,之后所有贡献覆盖。

---

## 6. 实现流程

### 6.1 写代码

按 Emacs 风格:
- **标准 header**——每个 .el 文件开头有固定格式 (`;;; foo.el --- description  -*- lexical-binding: t; -*-`)
- **标准 formatting**——2 空格缩进、特定行宽
- **docstring 完整**——第一行句号结尾、参数大写、文档详细
- **加 ChangeLog**——commit message 用 GNU changelog 格式

读 Emacs 源码的 `CONTRIBUTE` 文件 (源码根目录) 看全部规则。Maintainer 不合风格不对的 patch,即使功能正确。

### 6.2 加测试

> ```elisp
> ;; test/lisp/foo-tests.el
> (require 'ert)
> (require 'foo)
>
> (ert-deftest foo-test-something ()
>   (should (eq (foo-func) ...)))
> ```

Emacs 的 test 在 `test/` 目录,镜像 `lisp/` 的结构——`lisp/foo.el` 对应 `test/lisp/foo-tests.el`。

`ert-deftest` 定义测试,`should` 断言。`eq` 比较身份,`equal` 比较内容,`eql` 比较 number/char。

跑:

> ```bash
> make lisp/foo-tests
> ```

只跑 foo 的测试。或 `make check` 跑所有测试 (慢,~30 分钟)。

### 6.3 byte-compile 检查

> ```bash
> make compile
> ```

无 warning。Emacs 对 byte-compile warning 严格——`unused lexical variable`、`free variable`、`obsolete function` 都必须修。

### 6.4 commit

Emacs commit message 风格:

> ```
> Fix bug in foo-bar (Bug#12345)
>
> * lisp/foo.el (foo-bar): Check for nil before calling (baz).
> * test/lisp/foo-tests.el (foo-test-bar-nil): Add test.
> * etc/NEWS: Mention.
> ```

格式:
- 第一行: 简洁,动词开头 (Fix, Add, Improve, Remove...)
- 空行
- 详细 (可选)
- 空行
- `* file (function): change description.`——GNU changelog 格式

`Bug#12345` 是 debbugs 的 bug 编号,如果你修的是 reported bug,引用它让 debbugs 自动关联。

### 6.5 etc/NEWS

如果加新特性或行为变,加 NEWS 条目:

> ```
> ** New command 'foo-bar'.
> Short description.
>
> ** 'foo-baz' now accepts a region.
> ```

NEWS 是 Emacs release 的"changelog",用户读它知道新版本有什么变化。每条用 `**` 开头,简短描述。

---

## 7. 提 patch

### 7.1 format-patch

> ```bash
> git format-patch -1 HEAD
> ```

生成 `0001-Fix-bug-in-foo-bar.patch`。`-1` 是最新 1 个 commit,可以 `-2`、`-3` 生成多个。文件名是 commit subject 的 slugified 版本。

Patch 文件包含: From (作者)、Subject、Date、commit message、diffstat、diff——可以直接 `git am` 应用。

### 7.2 send-email

> ```bash
> git send-email --to=bug-gnu-emacs@gnu.org \
>   --cc=emacs-devel@gnu.org \
>   0001-*.patch
> ```

`--to` 是 bug tracker (每个 patch 自动分配 bug 编号),`--cc` 是开发列表 (让大家看到讨论)。

发完邮件几小时后,debbugs 自动回复分配的 bug 编号——你的 patch 在 https://debbugs.gnu.org/cgi/bugreport.cgi?bug=NNNNN。

### 7.3 用 report-emacs-bug

或更简单,在 Emacs 里:

> ```
> M-x report-emacs-bug RET
> ```

这个命令:
- 自动收集 Emacs 版本、系统信息、loaded packages
- 帮你格式化邮件
- 如果 attach 文件 (patch),会自动 mime-encode
- 发到 bug-gnu-emacs@gnu.org 并 cc emacs-devel

对新手友好——不用配 git send-email,Emacs 自己用 smtpmail 发。但灵活性不如 git send-email (比如不能批量发多个 patch)。

---

## 8. 等回应

### 8.1 在 debbugs

每个 patch 在 https://debbugs.gnu.org/ 有 thread。你能看到所有回复、状态变化 (pending、accepted、closed)。

### 8.2 maintainer review

Eli Zaretskii (主 maintainer)、Stefan Monnier、Andrea Corallo 等会 review。

Eli 是 Emacs 事实上的 BDFL (Benevolent Dictator For Life)——他不是创始人,但维护 Emacs 20+ 年,对所有 patch 有最终决定权。他极其认真——每个 patch 都仔细 review,经常要求多轮修改。这种严格保证了 Emacs 代码质量。

可能反馈:
- 改 X——具体的代码修改
- 加 docstring——文档不够详细
- 改 NEWS——措辞不清
- 加测试——覆盖不够
- 拒——理由可能是"不在 scope"、"设计不对"、"已有别的方案"

接受拒绝都要冷静——Eli 的反馈几乎都有道理,即使你觉得委屈,理性解释你的观点,如果他坚持,接受。

### 8.3 修 + 重新提交

> ```bash
> # 改代码
> git commit --amend
> git format-patch -1 HEAD -o /tmp/patches/
> # 用 v2 标记:
> git send-email --to=bug-gnu-emacs@gnu.org \
>   --subject-prefix="PATCH v2" \
>   --in-reply-to="<original-message-id>" \
>   /tmp/patches/0001-*.patch
> ```

`--subject-prefix="PATCH v2"` 让 v2 标记为 v2——maintainer 知道这是修订版,不是新 patch。

`--in-reply-to="<original-message-id>"` 让 v2 在同一线程 (邮件客户端按 thread 显示)。Original message ID 是 v1 邮件的 Message-ID header,在 debbugs 上能看到。

`--in-reply-to` 让 v2 在同一线程。Maintainer 看到一个连贯的讨论,而不是分散的多个邮件。

---

## 9. 合并

maintainer 满意 → patch 进 emacs.git。

### 9.1 push 流程

maintainer 用 `git push` 到 Savannah (有 commit 权限)。GitHub mirror 自动同步 (每几分钟)。

历史上 Emacs 用 `bzr` (2010-2014),之前用 `cvs` (到 2009),再之前用 patches mailed to Stallman。现在 git 已经稳定。

### 9.2 你的名字在 ChangeLog

每个 commit 加 author 到 ChangeLog (Emacs 自动从 commit author 提取)。

`etc/AUTHORS` 文件列所有贡献者——你的名字永久在那里,作为 Emacs 历史的一部分。

### 9.3 在 emacs-30 / master

patch 进 master (开发分支)——下一个 major release (Emacs 31) 会包含。

或 backport 到 emacs-30 (稳定分支)——如果是 bug fix,会 backport 到当前稳定版 (Emacs 30.x)。Backport 由 maintainer 决定,不需要你做。

---

## 10. 例子: 修 docstring

最简单的 Emacs core 贡献——docstring 改进。

### 10.1 找

> ```
> M-x find-library simple.el RET
> C-s defun kill-region
> ```

看 docstring,发现 typo 或不清楚。

`M-x find-library` 打开 simple.el (Emacs 的基础命令库)。`C-s defun kill-region` 搜索 kill-region 函数定义。

### 10.2 改

> ```elisp
> (defun kill-region (beg end &optional region)
>   "Kill (cut) text between BEG and END.
> ..." ; 改这里
>   ...)
> ```

只改 docstring,不改代码。Docstring 改动不需要 copyright paper (< 15 行)。

### 10.3 commit

> ```
> Improve docstring of kill-region
>
> * lisp/simple.el (kill-region): Clarify that the killed text
> goes into the kill ring.
> ```

GNU changelog 格式。

### 10.4 send-email

> ```bash
> git format-patch -1 HEAD
> git send-email --to=bug-gnu-emacs@gnu.org \
>   --cc=emacs-devel@gnu.org \
>   0001-*.patch
> ```

### 10.5 合并

通常 docstring 改动 review 快,容易合并。Eli 通常 1-2 周回复,接受就 merge。

---

## 11. 例子: 加新特性

更复杂。

### 11.1 讨论

先邮件到 emacs-devel:

> ```
> Subject: Proposal: add new function foo-bar
>
> I'd like to add `foo-bar' that does X.
> Use case: Y.
> Implementation sketch: Z.
>
> Opinions?
> ```

讨论: 是否值得加? 命名? API?

讨论可能持续几周——Emacs 社区对加新 API 很谨慎 (因为加进去就难移除)。你需要说服 maintainer 这个 feature 真的有用。

### 11.2 实现

讨论通过后,写代码 + 测试 + 文档。

### 11.3 patch

> ```
> Add new function foo-bar
>
> * lisp/foo.el (foo-bar): New function.
> * test/lisp/foo-tests.el (foo-test-bar): Test.
> * etc/NEWS: Mention.
> * doc/lispref/foo.texi (Foo Bar): Document.
> ```

(同时改文档——加新函数必须更新 Elisp manual)

新特性的 commit 要列出所有改的文件——`* file (thing): description.` 格式。

### 11.4 send-email + 等

Feature patch review 通常 2-3 轮,持续 1-3 个月。需要 copyright paper (因为 ≥ 100 行)。

---

## 12. 文化

### 12.1 礼貌

Emacs 维护者都是志愿者——他们白天有全职工作,晚上和周末维护 Emacs。尊重他们的时间。

- 谢谢他们的时间——简单的 "Thanks for reviewing" 很重要
- 不要命令式语气——"You should..." 改成 "What do you think about..."
- 接受 feedback——即使不同意,也理性讨论

### 12.2 长视野

Emacs 50 年。你的贡献影响几代人。代码要**经得起时间考验**:
- **简洁**——简单代码 50 年后还能读懂
- **文档好**——没文档的代码等于没写
- **兼容性好**——不破坏现有用户的工作流

很多 Emacs 函数 1980 年代写的,今天还在用——因为作者考虑了向前兼容。这种"长视野"是 Emacs 文化的核心。

### 12.3 GNU 哲学

理解并尊重 GNU 理念:
- 自由软件
- GPL v3+
- copyright assignment 给 FSF

不分享这套理念的人通常不会留在 Emacs 社区——这是文化筛选。如果你觉得这些"过时"或"偏执",Emacs 可能不适合你;但如果你认同,这里是少数几个还能"按理念做软件"的地方。

### 12.4 不要太怪

Emacs 有 50 年历史。有些"怪"是历史遗留:
- `car` / `cdr` 不是 first/rest——这是 IBM 704 机器的指令名 (Contents of Address Register, Contents of Decrement Register)
- `defun` 不是 def-func——这是 Lisp 1.5 (1958 年) 的语法
- `nil` 是 false,`0` 是 true——这是 Lisp 传统,和 C 不同
- `kbd` 用奇怪的语法——这是历史累积

不要试图"修复"这些。学。这些"怪"是 Emacs 的身份——改了就不再是 Emacs。Maintainer 不会合"重命名 car/cdr" 之类的 patch。

---

## 13. 实战

### Ex 1: 看 NEWS

> ```bash
> cat etc/NEWS | head -100
> ```

看最新版本的改动。NEWS 是理解 Emacs 演化的最佳入口——每个 release 的所有变化都在。

### Ex 2: 订阅 emacs-devel

发邮件 `emacs-devel-join@gnu.org`。
读 1 周。

观察讨论风格、决策流程、maintainer 的关注点。

### Ex 3: 改 docstring

找一个 docstring 改进,提 patch。

最简单的 Emacs core 贡献——不需要 paper,review 快。

### Ex 4: 修 small bug

找一个 small bug,提 patch。

需要 paper (< 15 行可能不需要,但建议签)。Bug fix 是中阶贡献。

### Ex 5: 加特性

提议 + 实现 + 提 patch。

最难,需要 paper + 长期 review。但是 Emacs 历史的一部分。

---

## 14. 自测

1. 怎么 clone Emacs 源码? (答: `git clone https://git.savannah.gnu.org/git/emacs.git`)
2. 怎么编译? (答: `./autogen.sh && ./configure && make`)
3. copyright assignment 何时需要? (答: 大贡献 ≥ 100 行,或多次小贡献)
4. 怎么提 patch? (答: `git format-patch` + `git send-email` 到 bug-gnu-emacs)
5. patch v2 怎么标记? (答: `--subject-prefix="PATCH v2"` + `--in-reply-to`)

---

## 15. 下一步

进入 `capstone.md` 做 Module 8 毕业项目。
