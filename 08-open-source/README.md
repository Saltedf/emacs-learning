# Module 8: 开源贡献

> **目标**: 给已有 Emacs 包或 Emacs core 贡献代码
> **时长**: 持续 (终身学习)
> **难度**: ★★★★★
> **依赖**: Module 0-7 全部
> **核心产出**: 一个被合并到非自己包的 PR

---

## 0. 这个模块在做什么

Module 0 到 Module 7 都是**吸收**——你学 Emacs 的功能、写自己的配置、做自己的包。Module 8 是**释放**:你把你学到的东西回馈给那个让你受益的生态系统。

这个转变很重要。在你回报之前,你只是个消费者;回报之后,你成了共同体的一员。Emacs 已经活了将近 50 年 (1984 年 Stallman 开始写),之所以能活这么久,正是因为每一代用户里都有一些人选择回馈。Stallman、Stallman、Zawinski、Monnier、Egli、Corallo、Dmitry... 这些名字背后是数以千计的小贡献累积。现在轮到你。

你已经具备回报的全部条件:
- 用 Emacs 飞快 (Module 1-2)
- 写过自己的包 (Module 3-4)
- 用 Org/Magit/LSP 工作 (Module 5)
- 读过别人的源码 (Module 6)
- 发到 MELPA (Module 7)

剩下唯一缺的是**信心**和**流程知识**。Module 8 就是补这个。

---

## 1. 为什么贡献

### 1.1 第一性原理推导

贡献这件事不是义务,但如果你推到底层逻辑,它几乎是个"必然":

- **第一性原理**:你用了别人免费写的软件,从他们的劳动中受益了。Emacs core 是 40+ 年 1500 万行 C + Lisp 代码,Org 5 万行,Magit 3 万行——你下载、安装、用,这些都没有付出过任何金钱。
- **推论 1**:这种受益有"对称性义务"。如果一个社区只有消费者没有贡献者,它会死。Emacs 活着就是因为它一直有新的贡献者接棒。
- **推论 2**:贡献方式有多种——报 bug、写文档、提 PR、捐钱、帮别人答疑。**代码不是唯一**,但代码最持久。
- **推论 3**:代码贡献累积可以建立**信任**。一开始你只能提交小 patch,被 maintainer review,通过后你有"信用"。
- **推论 4**:信用累积到一定程度,你可能拿到 commit 权限——成为项目的"自己人"。这就是开源的晋升路径。
- **推论 5**:被信任后,你能影响项目的方向。比如决定某个特性怎么实现。这是单纯的"用户"永远做不到的。

这套逻辑和商业软件完全不同。商业软件里,你付钱,然后只能抱怨;开源里,你不付钱,但能改。**Emacs 选择开源不是因为它"友好",而是因为 Stallman 认为软件自由是用户的权利**。

### 1.2 道德回报

你用的所有包 (Org, Magit, Vertico, ...) 都是别人免费写的。

回报是应该的——但更重要的是,**回报让你也变得更好**。免费拿到工具的人心里有一种"债",这种债不还,会慢慢让你和社区疏远;还了,你会感到"这工具是我的"——参与感改变了你和工具的关系。Stallman 一直强调这种"用户所有者"的心理,他认为这是 GNU 项目最重要的精神资产。

### 1.3 技术成长

回报的过程本身就是最好的训练:

- 读别人的代码,你学**新技巧**。Emacs 包的作者大多是顶级 Lisp 黑客——读 Carsten Dominik (Org)、Jonas Bernoulli (Magit)、Daniel Mendler (Vertico) 的代码,等于在上大师课。
- 接受 review,你提高**代码质量**。Maintainer 的反馈会让你意识到自己代码里的盲点——风格、命名、边界情况、性能。这种反馈在商业公司里需要花几万块钱请 senior engineer 做,在开源社区里免费。
- 处理边界情况,你练**严谨思维**。Bug report 往往暴露你没想到的场景——空 list、超大 buffer、unicode 字符、嵌套调用。每修一个 bug 你就强一分。
- 跨人协作,你学**沟通**。Maintainer 是不同文化、不同行业、不同性格的人。学习如何用文字精确表达技术想法,是一项核心技能,在工作和生活里都用得上。

### 1.4 社区连接

- 你的 GitHub 有了真实贡献——简历上的加分项
- 认识其他 Emacs 极客——他们可能是你未来的合作者、雇主、朋友
- 影响项目方向——你能让 Org/Magit/Vertico 长成你想要的样子

---

## 2. 贡献类型

开源贡献不是只有"写代码"。它是一个**光谱**,从最轻到最重。新人往往以为只有"写大 feature"才叫贡献,所以不敢开始——其实最轻的贡献极其有价值,而且往往是 maintainer 最需要的。

下面按**入门难度**排序:

### 2.1 Bug report

发现 bug,报给 maintainer。很多人碰到 bug 自己默默重启 Emacs 就算了,但 maintainer 看不到你的崩溃,他不知道有 bug。Bug report 是**最被低估**的贡献——一个清晰的重现步骤能让 maintainer 节省几小时的猜测时间。

好的 bug report 三要素:
- **重现步骤**:一步步能照做,而不是"我用着用着就崩了"
- **期望 vs 实际**:你以为会发生什么,实际发生了什么
- **环境信息**:Emacs 版本、OS、相关包版本

`M-x report-emacs-bug` 会自动收集 Emacs 版本和加载的包,直接帮你格式化好。

### 2.2 Feature request

提建议。Feature request 不是命令,而是**讨论的起点**。你描述痛点,给一个粗略方向,问 maintainer "这个值得做吗?"。如果 maintainer 觉得对,他会回复讨论;如果他不感兴趣,你也学到了他对项目方向的判断。

好的 feature request 三要素:
- **痛点描述**:你为什么需要这个?具体场景?
- **建议方案**:大致方向 (不一定完整)
- **是否愿意实现**:如果你说"我可以实现",maintainer 接受概率高很多

### 2.3 文档 PR

修文档。这是**最容易开始**的贡献——你不需要懂代码,只需要懂"什么是清楚的写作"。

修文档包括:
- typo (拼写错误)
- 不清楚的句子
- 缺失的例子
- 过时的信息

很多 maintainer 烦的就是 README 上的 typo——他们自己看不出来,用户也懒得报。你修一个 typo,maintainer 一秒合 PR,你在 GitHub 上有了第一个 contribution。从这开始积累信心。

### 2.4 Bug fix PR

修 bug。比 typo 难,需要读源码、找原因、写测试、改代码。这是**真正进入项目**的门槛。一旦你能修别人的 bug,你就懂了这个项目的内部结构。

修 bug 的流程:
- 找原因 (Edebug、grep、复现)
- 写测试 (防回归)
- 修
- 验证测试通过

### 2.5 Feature PR

加新功能。最难,因为需要 maintainer 同意"这个功能值得加"。Feature PR 通常先有 issue 讨论,讨论通过后再实现。直接发 unannounced feature PR 大概率被拒——maintainer 觉得没讨论就发代码是浪费大家时间。

Feature PR 的流程:
- 先开 issue 讨论 (是不是值得做?API 怎么设计?)
- 实现 (在 branch 上)
- 测试
- 文档 (docstring + README + NEWS)

### 2.6 Review

帮别人 review PR。这是**进阶贡献**,适合你已经熟悉项目后。看别人的 PR,提建议,跑测试。Maintainer 会非常感激——大项目的 PR review 是瓶颈,你能帮分担,等于在分担项目的维护负担。

Review 要注意礼貌:不要"这写得不行",而要"这里如果用 X 函数会不会更简洁?"。建设性反馈是 review 的灵魂。

---

## 3. 给哪个项目

### 3.1 你常用的包

贡献你**用得最多**的包。这不是"道德正确",而是务实的判断:
- 你了解它的痛点——你能找到真的有用的改进点
- 你有动力——遇到 bug 你会真的去修
- 你知道正确方向——你不会提出和项目理念冲突的 feature

如果你只用过一次 vertico,去给它提 feature,大概率提的方向是错的。如果你用了 3 年 vertico,你的提案几乎一定切中要害。

### 3.2 good first issue

GitHub 上很多项目标 `good first issue`。这是 maintainer 主动标记的"适合新人"的任务——通常范围明确、依赖少、不需要深入理解整个项目。

但要注意:**不是所有标了 good first issue 的都真的好做**。有些是 maintainer 一时觉得简单标的,实际做起来发现涉及很多代码。打开 issue 看一下讨论,如果之前有人尝试过但卡住了,可能没那么简单。

### 3.3 推荐开始的项目

按"活跃度 × 社区友好度"排序:

- **vertico/corfu/consult**: 作者 Daniel Mendler 极其活跃,review 快,代码现代
- **use-package**: 中等,用户多,影响力大
- **org-mode**: 大项目,有很多子任务,但 review 慢 (Bastien 是 maintainer,事多)
- **magit**: 大项目,社区活跃,但代码复杂,新人入门难
- **emacs core**: 最权威,要求最高,流程最重 (邮件 + paper),但贡献最持久
- **小包**: 容易开始,但 maintainer 可能不活跃

### 3.4 不要碰的项目

- **不熟悉的项目**: 你不懂痛点,提的东西是噪音
- **abandon 的项目**: 没维护者 merge,你的 PR 永远等
- **政治化项目**: 争论多,精力消耗大
- **重写中的项目**: 大重构期间,maintainer 不接受小 PR

怎么判断 abandon? 看 GitHub Insights → commit 频率。如果近一年没 commit,基本 abandon 了。

---

## 4. 流程

下面是 GitHub 风格包的完整 PR 流程。Emacs core 流程不同 (邮件),见第 5 节。

### 4.1 选 issue

读 issue tracker,找一个你能做的。**做一个完整的 issue 比尝试 10 个半吊子的 issue 强**——前者让你完成一个 PR,后者让你心累放弃。

### 4.2 fork + clone

GitHub 的 fork 模型是这样的:你不能直接 push 到 `minad/vertico` (你不是 maintainer),所以你 fork 一份到自己账号下 `you/vertico`,push 到你的 fork,然后从 fork 向 upstream 提 PR。这个模型是 GitHub 的发明,不是 git 本身的——git 本身没有 PR 概念。

`gh repo fork` 是 GitHub CLI 的命令,它会自动:1) 在 GitHub 上 fork,2) 把 fork 加为 remote,3) clone 到本地。一条命令做完三件事。

> ```bash
> gh repo fork PROJECT
> cd PROJECT
> git checkout -b fix-something
> ```

branch 名字要描述性——`fix-something` 比 `patch` 好,`fix-window-crash` 比 `fix-something` 好。Maintainer 看 branch 名就知道你修了啥。Module 7 的 git 工作流里讲过这套命名约定。

**关键**: 永远在 branch 上改,不要在 main 改。如果你的 PR 后面 maintainer 让你改,你在 branch 上 amend 然后 force-push 到自己的 fork——这不会影响 upstream。如果你在 main 上改,事情就乱了。

### 4.3 实现 + 测试

写代码,跑测试。每个项目都有自己的测试命令——`make test` 是最常见的。Vertico、Corfu 这些现代项目用 Makefile 包装 ERT 测试。

测试通过后,**byte-compile 必须 clean**——`emacs --batch -L . -f batch-byte-compile *.el` 不能有 warning。Byte-compile warning 在很多项目里是硬要求,因为 warning 往往预示潜在 bug。

### 4.4 commit + push

> ```bash
> git commit -m "Fix X by Y"
> git push origin fix-something
> ```

commit message 要清楚——"Fix crash when window-size-change returns nil" 比 "fix stuff" 好一万倍。Maintainer 在 GitHub 看你的 PR 时,第一眼是 commit message。Message 模糊会让他怀疑你不懂自己改了什么。

Conventional Commits 风格 (`fix:`, `feat:`, `docs:`) 是现代项目的标准,但老 Emacs 项目 (Magit 之外的) 用自己的 `* file (function): description.` 风格。读 CONTRIBUTING.md 看。

### 4.5 提 PR

GitHub 上提 PR。PR 描述要包含:
- **Problem**: 你在解决什么问题?链接到 issue
- **Solution**: 你怎么解决的?
- **Test**: 怎么验证?

`gh pr create` 命令行提 PR,适合不喜欢切到浏览器的人。

### 4.6 等 review + 修

维护者 review,你修。可能多轮——大项目的 PR 平均要 2-3 轮 review。每一轮都可能让你:改命名、加 docstring、加测试、改算法。**别把这当否定**——maintainer 在帮你把代码打磨到能进 mainline 的质量。

### 4.7 合并

维护者合并。你的贡献进了项目。从这一刻起,你的名字永久出现在项目的 git log 里——50 年后还有人能 git blame 到你的代码。这是开源贡献最持久的奖励。

---

## 5. Emacs Core 贡献

### 5.1 为什么不一样

Emacs core 走邮件列表,**不接受 GitHub PR**。

这不是 maintainer 老顽固——而是 GNU 项目的 deliberate 选择。GitHub 是 Microsoft 的商业服务,GNU 项目坚持用自己的基础设施 (Savannah),避免依赖单一公司。如果某天 Microsoft 改 GitHub 条款,GNU 项目不会受影响。这种"基础设施独立性"是 GNU 哲学的核心实践。

另外,邮件列表的归档是**永久**的——所有 1990 年代以来的 emacs-devel 邮件都能在 lists.gnu.org 查到。GitHub 的 issue/PR 没有这种保证——公司可以删数据,可以改 API。邮件列表用 mbox 格式存储,只要 SMTP 还在就能读。

技术上,git 本身就是为邮件 workflow 设计的 (Linus 早期就是这样用)。`git format-patch` 和 `git send-email` 是 git 的标准命令,不是补丁工具。GitHub 的 PR 是 GitHub 加的一层 UI,不是 git 本身的特性。

### 5.2 copyright assignment

FSF 要求**大贡献者**签 copyright assignment paper。这是一份法律文件,你把代码的版权转给 FSF。这听起来可怕,但实际是好事:

1. FSF 持有完整版权,可以在法庭上**强制**执行 GPL——违反 GPL 的人会被 FSF 起诉。如果版权分散在 1000 个贡献者手里,没人有法律地位起诉侵权者。
2. 如果 Emacs 的某个 license 需要升级 (例如 GPL v4 出来),FSF 可以单方面决定,不需要找所有贡献者签字。Linux 内核就因为版权分散,至今无法从 GPLv2 升级到 GPLv3。
3. 你的贡献永久记录在 Emacs 的作者列表里——签了 paper 后,你就是 Emacs 的"法定作者"之一。

这是 GNU 项目独特的"集体所有权"模式——和 Linux (Linus 持有商标) 或 Rust (各公司持有) 完全不同。它源于 Stallman 的理念: **软件应该属于用户群体,不属于个人或公司**。

签 paper 是一次性过程 (大约 2 周,FSF 处理 paperwork),之后所有贡献都覆盖。这就是为什么很多 Emacs 包要求签 paper 才能合——保护整个生态的法律健康。

小贡献 (< 15 行) 不需要 paper。文档 typo、单行 fix 可以直接提 patch。

> ```elisp
> M-x describe-copying     ; 看详情
> ```

`describe-copying` 显示 Emacs 的 GPL license 全文和 copyright 信息。如果你想看 Emacs 的法律细节,这是入口。

### 5.3 流程

Emacs core 贡献的完整流程比 GitHub PR 长,但每一步都有道理:

1. **订阅 emacs-devel@gnu.org**——感受文化,看别人怎么提 patch
2. **提议** (邮件)——大改动先在邮件列表讨论,避免白做
3. **实现**——按 Emacs 风格写代码、加测试、改 NEWS
4. **用 `M-x report-emacs-bug` 提 patch**——它会自动格式化、附版本信息、cc 到 emacs-devel
5. **邮件讨论**——maintainer 和社区 review
6. **maintainer 接受**——他 commit 到 emacs.git

### 5.4 patch 格式

Emacs patch 是标准 git format-patch 输出,通过邮件发送。它包含作者信息、commit message、diff,可以直接 `git am` 应用。这种格式从 2005 年 git 诞生起就这样,从 1990 年代的 patch 工具就这样,从 1980 年代的 Usenet 就这样——40 年没变。

> ```
> From: You <you@example.com>
> Subject: [PATCH] Fix X in foo.el
>
> Description of the fix.
>
> ---
>  lisp/foo.el | 4 ++--
>  1 file changed, 2 insertions(+), 2 deletions(-)
>
> diff --git a/lisp/foo.el b/lisp/foo.el
> index abc..def 100644
> --- a/lisp/foo.el
> +++ b/lisp/foo.el
> @@ ...
> - old line
> + new line
> ```

这个格式有讲究:`From` 是你的真实身份 (Emacs 社区看重真名),`Subject` 的 `[PATCH]` 前缀让收件人知道这是 patch,`---` 后是 diffstat 和 diff。Maintainer 用 `git am` 一键应用。

生成这个格式用 git 自带的工具:

> ```bash
> git format-patch -1 HEAD            # 生成最新 commit 的 patch
> git send-email --to=bug-gnu-emacs@gnu.org 0001-*.patch
> ```

`git send-email` 不是 GitHub 风格的"提 PR",而是把 patch 作为邮件正文发到 bug-gnu-emacs@gnu.org (Emacs 的 bug tracker 邮件)。这封邮件会被自动归档到 debbugs.gnu.org,所有人都能看到并回复。这种"邮件即 issue tracker"的设计就是 GNU 项目的 debbugs 系统。

---

## 6. 社区

Emacs 社区分散在多个平台,没有"唯一入口"。这是 Emacs 文化的特点——它早于所有现代社交平台,所以每个时代都用当时的工具,从 1980s 的 Usenet 到 1990s 的 mailing list 到 2000s 的 IRC 到 2010s 的 Reddit 到 2020s 的 Matrix。所有这些今天都还活着。

### 6.1 邮件列表

| 列表 | 用途 |
|---|---|
| emacs-devel@gnu.org | Emacs 开发讨论 |
| emacs-tangents@gnu.org | 离题 (社会、文化、笑话) |
| bug-gnu-emacs@gnu.org | bug 报告和 patch |
| help-gnu-emacs@gnu.org | 用户求助 |
| org-mode-email list | Org 开发 |

订阅: https://lists.gnu.org/

邮件列表是**最权威**的渠道——所有 Emacs core 决策都在这里。如果你只看一个,看 emacs-devel。

### 6.2 IRC / Matrix

- `#emacs` on Libera Chat
- `#emacs-beginners`
- `#org-mode`
- Matrix: `#emacs:matrix.org`

IRC 是实时聊天,适合快速问问题。`#emacs-beginners` 是专为新人设的——不会有人嘲笑简单问题。

### 6.3 Reddit

- r/emacs (英文, 活跃)
- r/orgmode

Reddit 适合分享配置、讨论新闻、问"你们怎么用 Emacs 做 X"。活跃度高但深度不如邮件列表。

### 6.4 StackExchange

- Emacs StackExchange
- Stack Overflow (标 emacs)

StackExchange 适合具体问题——"怎么在 X 包里实现 Y"。答案会永久存在,搜索引擎能找到。

### 6.5 博客

- Emacs Wiki (社区编辑的 wiki,海量但有点乱)
- Sacha Chua's blog (每周 Emacs 新闻,持续 10+ 年)
- System Crafters (视频 + 博客,适合新人)
- Planet Emacsen (Emacs 博客聚合)

Sacha Chua 的 weekly Emacs news 是必读——她每周手动整理所有 Emacs 相关的活动、博客、新包。

### 6.6 视频频道

- System Crafters (YouTube)——新人友好的视频教程
- Protesilaos Stavrou——深度配置和哲学思考
- Emacsconf (年度会议)——Emacs 用户自己组织的会议,演讲质量极高

Emacsconf 是个亮点:全社区志愿者组织,完全免费,所有演讲都有录像和文字稿,在 emacsconf.org。

---

## 7. 文化

### 7.1 GNU 哲学

Emacs 是 GNU 项目的旗舰产品 ("GNU Emacs" 是全名)。理解 GNU 哲学能让你理解很多 Emacs 设计的"为什么":

- **软件应该尊重用户自由**——用户控制软件,不是被软件控制
- **源码必须可获取**——闭源软件是道德问题,不只是技术问题
- **修改和分享是权利**——不是"被允许",是"应该的"
- **商业可以,但不能锁用户**——卖自由软件可以,但买家必须得到同样的自由

这套哲学解释了为什么 Emacs 不学 VS Code 的 telemetry (用户行为上报),不学 Sublime 的付费 license,不学 JetBrains 的"社区版 vs 商业版"区分。Emacs 给所有人同样的全部功能,不区分。

理解这点能让你对很多 Emacs "怪"的地方有耐心——为什么 `M-x report-emacs-bug` 默认不发送 usage data,为什么没有 "Emacs Cloud",为什么 package 系统没有"官方商店"。这些都是 deliberate choice。

### 7.2 GPL

所有 Emacs 代码是 GPL v3+。你的贡献也 GPL v3+。

GPL v3+ 的 "+" 意思是"或更新版本"——如果将来 FSF 发布 GPL v4,你的代码自动适用 v4。这是 FSF 的前瞻性设计,避免 license 锁定。

如果你引用了其他 GPL 代码,你的代码必须也是 GPL——这是 GPL 的"传染性" (copyleft)。BSD/MIT 没有传染性,所以商业公司更喜欢 BSD/MIT——但他们就少了一个保证:他们的修改不必回馈社区。

### 7.3 礼貌

Emacs 社区相对礼貌——比 Reddit、比 Discord、比很多技术论坛都礼貌。这不是偶然,而是社区文化的 deliberate 选择:

- **用真实姓名**——邮件列表里大家都用真名,不用匿名 handle。这让人更有责任感。
- **谦虚,不傲慢**——Stallman 自己是创始人都经常被批评,没人有"特权"
- **帮助新人**——所有 senior 都回答新人问题,这是社区传承

Stallman 在邮件列表里和普通贡献者讨论的风格——你能看到他认真回复一个新人的 patch,有时接受,有时拒绝,但都给详细理由。这种文化从上而下。

### 7.4 多元

Emacs 社区有各国家、行业、年龄的人。Maintainer 来自美国、法国、德国、日本、巴西、俄罗斯——地域分布极广。年龄从大学生到退休教授都有。

保持开放心态——你看到的"奇怪"feature request 可能来自一个完全不同的工作流。不要急着否定。

---

## 8. 毕业检查

### 概念题

1. 怎么报 bug? (答: `M-x report-emacs-bug` 自动格式化)
2. fork + PR 流程? (答: fork → clone → branch → 改 → push fork → PR upstream)
3. Emacs core 怎么贡献? (答: 邮件 + patch,不走 GitHub)
4. copyright assignment 是什么? (答: 把版权转给 FSF,让其能法律执行 GPL)
5. emacs-devel 是什么? (答: Emacs 开发邮件列表,所有 core 决策在这里)

### 实操

1. 给一个包提 issue
2. 给一个包提 PR (即使小)
3. 订阅 emacs-devel
4. 给 Emacs core 提 patch (可选,大成就)

---

## 9. 下一步

打开 `concept-anchor.md` 看 glossary/ack。
然后 `reading-source.md` 学读源码。
然后 `first-pr.md` 做第一个 PR。
然后 `emacs-devel.md` 给 core 贡献。

最后 `capstone.md`: 一个被合并的非自己包 PR。
