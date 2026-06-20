# Capstone: Module 7 毕业项目 (发布到 MELPA)

> **目标**: 发布你的包到 MELPA,用户能 `package-install` 装
> **时长**: 2-3 周
> **难度**: ★★★★★

Capstone 是 Module 7 的实战检验。前面 6 个文件学了设计、测试、文档、发布的理论,现在是把它们**串起来做一个真实的东西**。这是从"学习者"到"作者"的最后一公里。**没有这一步,你学的所有东西都是纸上谈兵**——因为只有真正发到 MELPA,你才知道:

- 你的设计在陌生用户手里能不能用
- 你的测试在多版本 Emacs 上过不过
- 你的 README 够不够新人理解
- MELPA 维护者的 review 标准实际是什么

这个项目要 2-3 周,不是因为它特别难——而是因为它有**不可压缩的等待时间**:review 通常 1-7 天,处理反馈再 1-2 天。你不能加速,只能准备好。所以**先动起来**,把代码先 ship 出去,边等 review 边迭代。

---

## 项目概述

把你之前写的 (Module 3 的 my-todo 或 Module 6 的 mylang-mode) 升级到生产级别:
1. 完整 header
2. ≥ 10 个 ERT 测试
3. byte-compile 无 warning
4. README 完整
5. LICENSE 文件
6. GitHub repo
7. CI (GitHub Actions)
8. MELPA recipe
9. PR 到 melpa/melpa
10. 合并

这 10 步是 Module 7 所有内容的综合——每一步对应前面文件的一个章节。**做一遍,你就完整理解了"写包"这件事**。这是 Emacs 学习路径的"成年礼"——完成后你就是 Emacs 圈的正式贡献者,不再是消费者。

---

## 实施步骤

### Week 1: 完善

第一周专注**把代码搞成生产级别**。不写新功能——只 polish 已有的。

#### Day 1: header + 提供包

确保 `my-package.el`:

```elisp
;;; my-package.el --- My Package -*- lexical-binding: t; -*-

;; Copyright (C) 2026 Your Name

;; Author: Your Name <you@example.com>
;; Maintainer: Your Name <you@example.com>
;; Version: 1.0
;; Package-Requires: ((emacs "29.1"))
;; Keywords: convenience
;; URL: https://github.com/you/my-package
;; SPDX-License-Identifier: GPL-3.0-or-later

;;; Commentary:
;; Description.

;;; Code:

;; ... 代码 ...

(provide 'my-package)
;;; my-package.el ends here
```

这个 header 是包的"身份证"。每个字段都重要——Copyright (GPL 要求)、Author/Maintainer (用户能联系你)、Version (MELPA 读)、Package-Requires (依赖检查)、Keywords (MELPA 分类)、URL (用户找文档)、SPDX (法律标识)。

**最容易漏的是 Package-Requires 的 emacs 版本**。新手以为"我没用 29 特性,就不用声明"——但你的包可能在 27 上跑挂(老版本缺 `when-let` 等等)。**保守做法:声明你测过的最低版本**。

#### Day 2: autoloads

每个 public 命令加 `;;;###autoload`:

```elisp
;;;###autoload
(defun my-package-command ()
  ...)

;;;###autoload
(define-derived-mode my-package-mode ...)
```

`;;;###autoload` 让用户装包后**立刻能用**——不用重启 Emacs,不用手动 require。MELPA build 时扫描这个注释,生成 autoloads 文件,装包时自动加载。

规则:**用户入口**才 autoload——命令(`defun interactive`)、mode(`define-derived-mode`)、custom group。内部辅助函数不 autoload。

#### Day 3-4: ERT 测试

至少 10 个测试,覆盖:
- 核心功能 (add, delete, get)
- edge case (empty input, bad input)
- 关键交互

10 个测试不是随便的数字——这是 MELPA 评审的事实门槛。少于此数,review 时会被要求加。但 10 个**好测试**比 30 个垃圾测试有价值——每个测试都应该验证一个"如果 break 会怎样"的真实场景。

写测试的顺序:**先写已知的 bug**(如果你历史上修过 bug,每个 bug 都写一个 regression test),再写核心功能的 happy path,最后写边界 case。

#### Day 5: byte-compile + 修 warning

```bash
emacs --batch -L . -f batch-byte-compile my-package.el
```

修所有 warning:
- free variable → defvar
- unused lexical var → 删
- interactive-only → 调整

每个 warning 都是潜在 bug。`free variable` 说明你用了未 defvar 的变量——可能是 typo(用了 `my-foo` 但定义了 `my-bar`)。`unused lexical var` 说明你 `let` 了但没用——多半也是 typo。**零 warning 是 MELPA 的硬要求**。

#### Day 6-7: README

完整 README:
- Description
- Installation
- Usage
- Configuration
- FAQ
- License

README 是门面。Day 6-7 两天专门写——不要急。写完后**找朋友看 5 分钟能不能装上**——他们的反馈比你自己看 10 遍有用。

### Week 2: 发布

第二周专注**把代码变成"全世界能装"**。

#### Day 8: LICENSE

下载 GPL v3 文本到 LICENSE 文件。

从 https://www.gnu.org/licenses/gpl-3.0.txt 复制完整文本。MELPA 要求显式 LICENSE 文件——不是只 header 里写 `SPDX-License-Identifier`。

#### Day 9: GitHub repo

```bash
mkdir my-package
cp my-package.el my-package/
cd my-package
git init
git add .
git commit -m "Initial release v1.0"
git tag -a v1.0 -m "First release"
gh repo create my-package --public --source=. --push
git push origin v1.0
```

`gh repo create` 一键搞定创建 + push。前提是装了 gh 并登录。

#### Day 10: CI

`.github/workflows/test.yml`:

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        emacs: ['27.2', '28.2', '29.4']
    steps:
    - uses: actions/checkout@v3
    - uses: purcell/setup-emacs@master
      with:
        version: ${{ matrix.emacs }}
    - name: Byte compile
      run: emacs --batch -L . -f batch-byte-compile my-package.el
    - name: Run tests
      run: emacs --batch -L . -L test -l test/my-package-test.el -f ert-run-tests-batch-and-exit
```

CI 测试矩阵是关键——同时跑 27.2、28.2、29.4,保证跨版本兼容。`purcell/setup-emacs` 是社区维护的 action,装指定版本 Emacs。最后两步:byte-compile 检查零 warning,跑测试 batch 模式。

#### Day 11: MELPA recipe

Fork melpa/melpa,加 `recipes/my-package`:

```
(my-package :repo "you/my-package" :fetcher github)
```

本地测试:

```bash
cd /path/to/melpa-fork
./run/maint/refresh-packages.sh my-package
```

应该生成 `packages/my-package-1.0.tar`。

**这一步必做**——很多 PR 因为 recipe 错误被拒。本地测过能 build,PR 通过率高 80%。

#### Day 12: PR

```bash
cd /path/to/melpa-fork
git checkout -b add-my-package
git add recipes/my-package
git commit -m "Add my-package"
git push origin add-my-package
```

GitHub 上提 PR 到 melpa/melpa。PR 描述简洁:

```
This PR adds `my-package`, a [description].

- Repo: https://github.com/you/my-package
- License: GPL v3+
- Tested with Emacs 27.2, 28.2, 29.4
```

5 秒能读懂——维护者看了第一个 impression 好。

#### Day 13-14: 等 review

通常 1-7 天 review。
处理反馈:
- 改 license
- 改命名
- 加测试

修完 push,PR 自动更新。

review 反馈是好事——意味着维护者认真看了你的代码。**最糟糕的不是被拒,是被忽略**。维护者是志愿者,他们花时间 review 是恩惠,认真处理每条反馈。

### Week 3: 合并后

#### Day 15: 合并后

合并后,MELPA 在 1 小时内 build。

合并那一刻你的包**正式上线**——全世界 Emacs 用户都能 `package-install` 装。这是 Emacs 学习路径的高光时刻。

#### Day 16: 测试安装

```bash
emacs -Q
M-x package-refresh-contents RET
M-x package-install RET my-package RET
```

应该装上!

`emacs -Q` 启动干净 Emacs——确保你的安装是干净的,不依赖你自己 init.el 的特殊配置。`package-refresh-contents` 更新 archive 索引(否则可能拿到缓存)。`package-install` 装包。

如果这步成功,你的包真的发布成功了。**截图发到 r/emacs**,分享你的成就。

#### Day 17-21: 宣传

- 发到 r/emacs
- 发到 emacs-devel
- 写博客

宣传是负责任的——让可能受益的用户知道你的包存在。但**别过度宣传**——发一次高质量的介绍,不要 spam。

写作建议:不只是"我做了 X",而是"我解决 Y 痛点,做了 X 来解决。这是我的设计思路..."。读者感兴趣的是**故事和设计**,不是产品本身。

---

## 评分

### 包质量 (40 分)

| 项 | 满分 |
|---|---|
| 代码 ≥ 500 行 | 10 |
| 头文件完整 | 10 |
| `(provide ...)` | 5 |
| `;;;###autoload` | 5 |
| docstring 完整 | 10 |

### 测试 (25 分)

| 项 | 满分 |
|---|---|
| ≥ 10 ERT 测试 | 15 |
| 覆盖核心功能 | 10 |

### 文档 (15 分)

| 项 | 满分 |
|---|---|
| README 完整 | 10 |
| LICENSE | 5 |

### CI + MELPA (20 分)

| 项 | 满分 |
|---|---|
| GitHub Actions 配 | 5 |
| MELPA recipe | 5 |
| PR 提交 | 5 |
| 合并 | 5 |

**满分 100**。70+ 合格。

评分的核心是**完成度**——不是代码多么花哨。一个 500 行的实用工具,所有 checklist 满足,比 5000 行的"实验性大作"但没发布得分高。**完成 > 完美**。

---

## 日志

写 `logs/module-07.md`:

```markdown
# Module 7 学习日志

**用时**: ___ 小时
**Capstone 包名**: ___
**GitHub**: ___
**MELPA**: [合并 / PR 中 / 计划]

## 设计过程

## 实现挑战

## 测试心得

## 文档心得

## MELPA review

(如已合并)

## 我学到的最重要 3 件事

1.
2.
3.

## 下一步

进入 Module 8: 开源贡献
```

日志不是形式主义——它强迫你反思。**反思是学习的核心**——光做不想,你的经验无法沉淀。每个章节问"什么最难?为什么难?我学到了什么?"回答这些问题让你下次做得更好。

---

## 毕业仪式

如果你完成了 Module 7:

**恭喜**。

你的包**在 MELPA**。
任何人 `M-x package-install` 能装。
你**为社区做出了贡献**。

这是从"用户"到"贡献者"的关键转变。

想象一下:某个陌生人,在世界的某个角落,通过 `M-x package-install` 装了你的包,他的工作流因为你的代码变得更快、更愉快。你**没见过他,他没见过你**,但你的代码改善了另一个人的生活——这就是开源的魔法。

Emacs 已经存在 40 年(1985 年 GNU Emacs 首发)。你写的包会成为下一个 40 年生态的一部分。某天你不再维护这个包,但你的代码可能被 fork、被继承、被改进——技术意义上的"长生不老"。

继续 Module 8,给**别人的包**贡献。

更新 `PROGRESS.md`: Module 7 ✅。

进入 Module 8: `08-open-source/README.md`。
