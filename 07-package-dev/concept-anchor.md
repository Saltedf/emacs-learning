# Concept Anchor: Editor Manual (Module 7)

> 替代 `emacs-manual-30.2/package.texi` + `trouble.texi` + `misc.texi`

这个文件是 Module 7 的"用户视角地图"。**写包之前,你要先理解"用户怎么用包"**——这是开发者最容易忘记的事:你以为自己在写包,但你的目标受众是用户,他们的心智模型由 Emacs 内置的 package 系统塑造。如果你不知道用户怎么找包、装包、报 bug,你就无法设计出符合用户预期的包。本节对应 Emacs Manual 里的 `package.texi` (用户怎么用包系统)、`trouble.texi` (出问题怎么排查)、`misc.texi` (其他工具)。这三个章节合起来就是"用户的包使用体验"。

---

## 1. package.texi (用户视角)

### 1.1 包列表

Emacs 内置一个包浏览器。打开它的命令是:

```
M-x package-list-packages
```

这会下载所有已配置 archive 的索引(`package-archives`),然后显示一个可排序、可过滤的列表。每一行是一个包:名字、版本、状态(installed/available/built-in)、简短描述。在这个列表里你可以:

- install / remove:光标移到某行,按 `i` 标记安装,`d` 标记删除,`x` 执行
- upgrade:按 `U` 标记所有可升级的包,`x` 执行
- 搜索 / 过滤:用标准的 `C-s`、`C-r`,或者绑定到 `package-menu-filter`

这是 Emacs 用户装包最直观的方式。但作为开发者,你要知道底层发生了什么:`package-list-packages` 调用 `package-refresh-contents`(如果不缓存)下载 archive 的 `archive-contents` 文件,解析后填充 `tabulated-list-mode`。这个流程对用户是透明的,但理解它有助于你设计 recipe——你的包在用户那里就是这么"被发现"的。

### 1.2 包源

Emacs 的包生态不是单一仓库,而是**多个并列的 archive**。每个 archive 有不同的审核标准、更新频率、license 要求。理解它们的区别是设计包发布策略的前提。

| 名字 | URL | 审核 |
|---|---|---|
| ELPA | https://elpa.gnu.org/packages/ | 严格 (FSF) |
| NonGNU ELPA | https://elpa.nongnu.org/nongnu/ | 中等 |
| MELPA | https://melpa.org/packages/ | 社区 |
| MELPA Stable | https://stable.melpa.org/packages/ | 社区稳定 |

四个 archive 的设计哲学截然不同。**ELPA** 是 FSF 官方运营的,要求作者签 copyright assignment 给 FSF,review 严格但稳定,默认包含在 Emacs 里。**NonGNU ELPA** 是较新的(2021 年)FSF 子项目,不要求 copyright assignment,审核比 ELPA 宽松,也默认包含——这是给"不想签 FSF paper 但想被官方认可"的作者的折中方案。**MELPA** 是社区最大的仓库,从 GitHub 自动 build,最 bleeding edge,绝大多数活跃包都在这里。**MELPA Stable** 基于 git tag,相对稳定但更新慢,而且很多作者不维护 tag 所以 Stable 经常缺包。

实际策略:**MELPA 是事实标准**。新包先发 MELPA,等成熟了可以同步到 NonGNU ELPA。ELPA 适合被 Emacs 核心团队认可的包(比如 org 就在 ELPA)。

### 1.3 用户配置

典型用户的 init.el 开头是这样的:

```elisp
(require 'package)
(setq package-archives
      '(("gnu"    . "https://elpa.gnu.org/packages/")
        ("nongnu" . "https://elpa.nongnu.org/nongnu/")
        ("melpa"  . "https://melpa.org/packages/")))
(package-initialize)
```

这段代码做了三件事。第一行 `(require 'package)` 加载 Emacs 内置的包管理 library。第二行 `setq package-archives` 配置 archive 列表——每个 entry 是 `(名字 . URL)` 的 cons cell,名字是用户友好的标识符,URL 是 archive 服务器。第三行 `(package-initialize)` 初始化包系统:它扫描 `package-user-dir` 下所有已安装包,加载它们的 `-autoloads.el`,设置好 load-path。

为什么这件事对你(开发者)重要?因为你的用户运行这段代码后,才能 `M-x package-install` 你的包。如果你的包只在 MELPA,但用户没配 melpa archive,他们就装不上。所以 README 的 installation 部分必须**告诉用户先配 archive**——很多新手不知道默认只有 ELPA 和 NonGNU。

### 1.4 Lisp 视角

`package.el` 既是用户命令的集合,也是可编程的 library。下面是开发者最常用的几个 API:

```elisp
(package-installed-p 'foo)
(package-install 'foo)
(package-delete 'foo)
(package-refresh-contents)
(package-list-packages)
(package--builtins)            ; 内置包
(package--activated-list)      ; 已激活
package-archives               ; list of (NAME . URL)
package-user-dir               ; 安装目录
package-quickstart-file
```

`package-installed-p` 是 predicate,返回某包是否已装——你的包如果有可选依赖,可以用它做条件加载,比如 `(when (package-installed-p 'magit) (require 'my-magit-integration))`。`package-install` 是命令式 API,程序化装包(可以在 init.el 里自动 bootstrap)。`package-refresh-contents` 重新下载 archive 索引,你的 CI 在跑测试前必须调它,否则可能装到过期版本。

注意 `package--builtins` 和 `package--activated-list` 用了双横线——这些是内部 API,理论上你不该依赖,但实际开发中偶尔有用(比如统计已激活包)。`package-archives` 和 `package-user-dir` 是 defcustom,用户可改。`package-quickstart-file` 是 Emacs 27+ 的新特性:它生成一个快照文件,启动时一次性加载所有包的 autoloads,大幅加速启动。

---

## 2. trouble.texi

### 2.1 报 bug

Emacs 内置一个 bug 报告命令,它会自动收集你的环境信息:

```
M-x report-emacs-bug
```

这个命令会打开一个 mail buffer,自动填好 Emacs 版本、系统、已装包等关键信息,你只需要描述问题。M-x report-emacs-bug 走的是 email 到 `bug-gnu-emacs@gnu.org` 邮件列表——这是 FSF 守护的官方 bug 通道,所有 bug 都归档在 `https://bugs.gnu.org/`。

为什么讲这个?因为你的包也会有 bug,你也需要类似的 bug 报告流程。**你的包的 bug 报告通道通常是 GitHub Issues**,但你应该模仿 Emacs 的好做法:让用户报告 bug 时自动收集环境信息。比如你的包可以提供一个 `my-package-report-bug` 命令,自动填上 Emacs 版本、包版本、相关配置——这样你 debug 时就不用问"什么版本?什么配置?"这种重复问题。

### 2.2 排查 init.el 问题

init.el 写错了,Emacs 启动可能直接挂或者部分功能失效。两个排查命令:

```bash
emacs --debug-init
emacs -Q
```

`emacs --debug-init` 让 Emacs 在 init.el 报错时进入 debugger,显示完整的 backtrace——你能立刻看到是哪一行报了什么错。这是修自己 init.el 的第一步。

`emacs -Q` (Q 表示 "quick",跳过用户 init) 启动一个**纯净的 Emacs**——不加载 init.el,不加载 site-lisp。这是排查"我的 bug 是包的问题还是配置的问题"的金标准:在 `emacs -Q` 下重现问题,如果能重现,说明是包的 bug;如果不能,说明是你的配置里某行代码搞坏了。作为开发者,**每次报 bug 你都应该让用户用 `emacs -Q` 重现**——否则你可能在 debug 别人的配置问题,白费时间。

### 2.3 你的包的 bug

你的包上线的第一天就会有人报 bug。这是必然的——再好的测试也覆盖不了所有用户场景。处理 bug 的标准流程:

- 在 GitHub issues 收。设置 issue template,要求用户报 Emacs 版本、包版本、复现步骤
- 用 ERT 重现。把 issue 转化为 ert-deftest——这能保证 bug 修了之后不会回归
- 修复,加测试。bug 修了但没测试 = 等于没修,因为下次改动又会触发

最后一点是关键:**bug fix 必须配 regression test**。这是 ERT 最有价值的应用——每次修 bug 时写一个 `ert-deftest`,把这个 bug 永久钉死,以后任何破坏它的改动都会立刻被 CI 拦截。优秀的包的 test suite 里 30%+ 都是历史 bug 的 regression test。

---

## 3. misc.texi

`misc.texi` 收录 Emacs 内置的一些"小工具"——shell、term、calendar、calc、gnus、eww。它们不是必须学的,但每个都是某个垂直领域的强者。你写包时可能需要和它们交互(比如你的包调用 calc 算表达式),所以至少要知道它们存在。

### 3.1 shell-mode / eshell

`M-x shell`, `M-x eshell`。前者调用系统的 shell(bash/zsh),后者是 Elisp 实现的 shell——可以在 shell 里直接写 Elisp。eshell 的设计哲学是"shell 应该和 Emacs 无缝集成",它的命令其实就是 Elisp 函数:`ls` 是 `(eshell/ls)`,`grep` 是 `(eshell/grep)`。你可以扩展 eshell 加自己的命令,这本身就是写包的好选题。

### 3.2 term / ansi-term

`M-x ansi-term` 启动一个真正的 terminal emulator——它模拟 VT100/ANSI,可以跑 vim、htop、tmux 这些全屏 TUI 程序。`shell-mode` 不行因为它只处理 line-mode 输出。term 的劣势是 Emacs 快捷键大部分被 terminal 抢走(`C-c C-c` 是 SIGINT 而不是 Emacs 命令),所以日常用 `shell`/`eshell`,只有跑 TUI 才用 term。

### 3.3 calendar / diary

(Module 5) Emacs 内置日历和日记,org-mode 就建立在 calendar 之上。

### 3.4 calc

`M-x calc` 是一个**惊人的计算器**。它支持任意精度、有理数、矩阵、代数化简、单位转换、二元运算——所有这些都在一个 Emacs buffer 里。calc 的设计者 Dave Gillespie 是 Emacs 圈传奇。calc 的强大在于它是"可编程"的:你可以在 calc 里写宏、定义函数,甚至把 calc 当成 symbolic math library 用(类似 Mathematica)。**写包时如果需要数学计算,先看 calc 能不能干**——不要重新发明轮子。

### 3.5 gnus

新闻/邮件阅读器。gnus 是 Emacs 圈最有历史的包之一(1994 年发布),设计极其独特——它把所有东西(邮件、新闻、RSS)抽象成"group"+ "article" 的概念,用同一套 UI 处理。学习曲线陡峭,但忠实用户极多。如果你写邮件相关的包,gnus 的设计哲学值得学习。

### 3.6 eww

Web 浏览器。基于 `shr`(Simple HTML Renderer),只显示文字和图片,不跑 JavaScript。eww 是"信息获取"利器:看文档、看 Wikipedia、看 GitHub README,没有任何 JS 广告/弹窗干扰。**你的包的文档如果要离线可读,生成 Info 格式**——eww 能完美渲染 Info,这也是为什么 GNU 项目坚持用 Texinfo 的原因之一。

---

## 4. 自测

1. ELPA 和 MELPA 区别?
2. 怎么装包?
3. 怎么报 Emacs bug?
4. 怎么排查 init.el 问题?

**答案**:
> 1. ELPA 官方严格 (FSF,需要 copyright assignment),MELPA 社区松 (GitHub 自动 build)。NonGNU ELPA 介于两者之间。
> 2. `M-x package-install RET 包名 RET`,或先 `M-x package-list-packages` 浏览。
> 3. `M-x report-emacs-bug`——会自动收集环境信息并发到 bug-gnu-emacs 邮件列表。
> 4. `emacs --debug-init` 看哪里报错;`emacs -Q` 看是不是配置问题。

理解这些"用户视角"的概念,你才能写出**让用户用得舒服**的包。下一步进入开发者视角。

---

## 5. 下一步

进入 `ref-deep-dive.md`。
