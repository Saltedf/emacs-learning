# Documentation: README + Texinfo

> 学完这个文件,你能写出生产级文档

文档是包的"接口"——没有文档,代码再好也没人用。新手最常犯的错:**代码写完忘文档**,或者**文档写得像内部备忘**。本节教你写两类文档:**README**(GitHub 看到的门面,面向"决定要不要装"的人)和 **Texinfo 手册**(Emacs 内 `C-h i` 看到的深度文档,面向"已经装了在用"的人)。

为什么有两套?因为受众不同:

- README 给"浏览者"——他们 30 秒决定装不装,需要"卖点 + 安装 + 快速用法"
- 手册给"使用者"——他们已经装了,需要"每个函数怎么用、每个配置什么意思、如何扩展"

优秀的包两者都有。只 README 没 Texinfo,用户深入用就抓瞎;只 Texinfo 没 README,新用户不会装。

---

## 1. README.md

README 是 GitHub 项目主页的第一屏——决定 80% 用户是否继续读。**糟糕的 README 比没有 README 更糟**,因为它给人"作者不认真"的印象。

### 1.1 标准结构

```markdown
# package-name

One-line description.

[![Build Status](badge-url)](link)
[![MELPA](melpa-badge)](link)
[![License](license-badge)](link)

## Description

Long description. 2-3 paragraphs.

Why this package exists. What it does.

## Screenshots

![screenshot](url)

## Installation

```elisp
(use-package package-name
  :ensure t
  :config ...)
```

Or manual:

```bash
git clone https://github.com/you/package-name
```

Add to init.el:
```elisp
(add-to-list 'load-path "/path/to/package-name")
(require 'package-name)
```

## Usage

Quick start:

```elisp
M-x package-name
```

Key bindings:

| Key | Action |
|-----|--------|
| `a` | Add |
| `d` | Delete |
| `s` | Save |

## Configuration

```elisp
(use-package package-name
  :ensure t
  :custom
  (package-name-option-1 'value)
  :bind ("C-c p" . package-name))
```

## FAQ

**Q: How to ...?**
A: ...

## Comparison

vs other-package:
- pros
- cons

## Contributions

Pull requests welcome. See CONTRIBUTING.md.

## License

GPL v3+
```

每一节的作用:

- **标题 + 一行描述**: 用户看一眼知道是什么
- **Badges**: Build Status(过 CI)、MELPA(已发布)、License——三个徽章告诉用户"这个包是认真的"
- **Description**: 长描述,2-3 段。第一段最重要——为什么存在,解决什么问题
- **Screenshots**: 一图胜千言。UI 类包没截图等于残废
- **Installation**: 必须有 `use-package` 模板(因为大多数用户用 use-package)+ 手动安装步骤
- **Usage**: 快速上手——3 行代码能让用户跑起来
- **Configuration**: 常用配置示例
- **FAQ**: 高频问题——预先回答,减少 issue
- **Comparison**: 和类似包对比——诚实地说优劣,而不是假装"我是唯一选择"
- **Contributions**: 鼓励 PR,链 CONTRIBUTING.md
- **License**: 必须,放结尾

README 的核心是"**装到能用的最短路径**"——从 GitHub 主页到用户在自己 Emacs 里调出你的命令,步骤越少越好。

### 1.2 例子

一个真实的 README:

```markdown
# my-todo

A simple todo manager for Emacs.

[![License GPL 3](https://img.shields.io/badge/license-GPLv3-blue.svg)](LICENSE)

## Description

my-todo is a minimalist todo manager. It stores todos
as a plain Lisp file, supports tags and priorities, and
integrates with `org-agenda`.

## Installation

```elisp
(use-package my-todo
  :ensure t
  :bind ("C-c t" . my-todo))
```

## Usage

`C-c t` opens the todo list.

| Key | Action |
|-----|--------|
| `a` | Add todo |
| `d` | Mark done |
| `x` | Delete |
| `s` | Save |
| `l` | Load |
| `q` | Quit |

## Configuration

```elisp
(setq my-todo-file "~/org/todos.el")
```

## License

GPL v3+
```

注意这个 README 的特点:

- 描述**第一段就讲清楚做什么**(minimalist todo manager)
- 描述**第二段讲卖点**(plain Lisp 文件、tags、org-agenda 集成)
- Installation 只有 use-package 模板——大多数用户不需要手动
- Usage 是一个表格——一目了然
- Configuration 只有一个示例——多余配置在 Texinfo

**对比其他生态系统**:VS Code Marketplace 要求详细 README、徽章、GIF;npm 包的 README 通常简单(因为 npm 网站渲染好);cargo 的 README 通常和文档分离(docs.rs 自动从 docstring 生成)。Emacs 的 README 介于 VS Code 和 npm 之间——需要但不极繁。

---

## 2. Texinfo

### 2.1 什么是 Texinfo

Texinfo 是 GNU 的文档系统:
- 一种源码 (`.texi`)
- 多种输出: Info, HTML, PDF, man

Emacs 内置所有手册都是 texi。

Texinfo 是 GNU 项目 1986 年的发明——Richard Stallman 设计的"单源多输出"系统。一份 `.texi` 文件,通过不同工具变成 Info(Emacs 内读)、HTML(浏览器)、PDF(打印)、man page(命令行)。这个"单源多输出"是 80 年代的前瞻设计,直到今天还是 GNU 标准的文档格式。

Emacs 内置所有手册都是 texi:`C-h i` 进 Info,看到的 `(emacs)`、`(elisp)`、`(org)` 都是 texi 编译的 .info 文件。Texinfo 是 GNU 文化的活化石。

为什么用 Texinfo 而不是 Markdown?

1. **结构化更强**: Texinfo 有 node、chapter、section、subsection 的严格层级,适合大文档;Markdown 平面,大文档难导航
2. **交叉引用**: `@xref{node}` 让文档间精确跳转,Markdown 只有 URL
3. **索引**: Texinfo 自动生成函数索引、变量索引——查 API 极快
4. **Emacs 原生集成**: `C-h i` 看,`M-x info-lookup-symbol` 跳 API——无缝

劣势:学习曲线比 Markdown 陡。但**严肃的包应该有 Texinfo**,因为它是 Emacs 文化的标准。

### 2.2 基础语法

一个最小的 Texinfo 手册:

```texi
\input texinfo   @c -*- mode: texinfo; -*-
@setfilename my-package.info
@settitle My Package Manual

@copying
This manual is for My Package version 1.0.

Copyright @copyright{} 2026 Your Name.
@end copying

@titlepage
@title My Package Manual
@author Your Name
@page
@vskip 0pt plus 1filll
@insertcopying
@end titlepage

@contents

@node Top
@top My Package

@menu
* Introduction::
* Installation::
* Usage::
* Configuration::
* Index::
@end menu

@node Introduction
@chapter Introduction

My Package is a...

@node Installation
@chapter Installation

@example
M-x package-install RET my-package RET
@end example

@node Usage
@chapter Usage

@table @kbd
@item C-c p
Open My Package.
@item a
Add new item.
@end table

@node Configuration
@chapter Configuration

@defvar my-package-file
File to store data.
@end defvar

@defun my-package-add ITEM
Add ITEM.
@end defun

@node Index
@unnumbered Index
@printindex fn

@bye
```

逐块解释:

- **`\input texinfo`**: 加载 texinfo macro 包。类似 LaTeX 的 `\documentclass`
- **`@setfilename`**: 输出 Info 文件名
- **`@settitle`**: 文档标题(用于 HTML 的 `<title>`)
- **`@copying ... @end copying`**: 版权声明。`@copyright{}` 是 © 符号
- **`@titlepage`**: PDF 标题页(只在 PDF 输出显示)
- **`@contents`**: 自动生成目录
- **`@node Top`**: 顶层 node。Texinfo 的"node"是文档的导航单元——每个 node 是一个跳转目标
- **`@top`**: 顶层章节标题
- **`@menu ... @end menu`**: 菜单,列出子 node。**菜单必须严格反映 node 结构**——少一个就链接断
- **`@node NAME`**: 声明 node,后面的 `@chapter` 是显示标题
- **`@table @kbd`**: 表格,keybindings 风格
- **`@defvar NAME`**: 变量定义(自动进索引)
- **`@defun NAME ARGS`**: 函数定义(自动进索引)
- **`@printindex fn`**: 打印函数索引
- **`@bye`**: 文件结束

### 2.3 命令

Texinfo 有上百个命令,常用这些:

| 命令 | 用途 |
|---|---|
| `@chapter TITLE` | 章节 |
| `@section TITLE` | 节 |
| `@subsection TITLE` | 子节 |
| `@node NAME` | 节点 (Info 跳转用) |
| `@menu` | 菜单 |
| `@table` | 描述表 |
| `@item` | 列表项 |
| `@code{...}` | 代码 |
| `@var{...}` | 变量 |
| `@file{...}` | 文件名 |
| `@command{...}` | 命令 |
| `@key{...}` | 键 |
| `@kbd{...}` | 键序列 |
| `@samp{...}` | 引用 |
| `@defun NAME ARGS` | 函数定义 |
| `@defvar NAME` | 变量定义 |
| `@defmac NAME ARGS` | 宏定义 |
| `@example` | 例子 |
| `@lisp` | Lisp 例子 |
| `@noindent` | 不缩进 |
| `@ref{NODE}` | 引用 |
| `@xref{NODE}` | 交叉引用 |
| `@url{URL}` | URL |
| `@email{ADDR}` | email |
| `@center TEXT` | 居中 |
| `@itemize` | 无序列表 |
| `@enumerate` | 有序列表 |
| `@quotation` | 引用块 |
| `@flushleft` `@flushright` | 对齐 |
| `@page` | 分页 |
| `@bye` | 文件结束 |

### 2.4 编译

把 .texi 变成各格式:

```bash
makeinfo my-package.texi          ; → .info
makeinfo --html my-package.texi   ; → HTML
texi2pdf my-package.texi          ; → PDF
```

`makeinfo` 是 Texinfo 主工具,生成 Info 或 HTML。`texi2pdf` 走 LaTeX → PDF 路径。**生成 .info 后,用户在 Emacs 里 `C-h i m My Package RET` 就能看**——这是 Texinfo 的核心价值,文档活在 Emacs 里,和代码无缝集成。

### 2.5 Emacs 整合

`.dir-locals.el`:
```elisp
((texinfo-mode . ((fill-column . 70))))
```

`.dir-locals.el` 是 Emacs 的"项目局部变量"——它让进入这个目录的所有 buffer 自动应用变量。`fill-column 70` 是 Texinfo 的传统宽度(GNU 标准)。这让你写 texi 时 `M-q` 自动按 70 列重排。

---

## 3. 在线文档

Texinfo 生成的 Info 主要给 Emacs 用户看,但用户在 GitHub 或浏览器看时需要 HTML 版本。GitHub Pages 自动部署 HTML 是标准做法。

### 3.1 GitHub Pages

`.github/workflows/pages.yml`:

```yaml
name: Pages

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: purcell/setup-emacs@master
      with:
        version: 29.1
    - name: Build docs
      run: makeinfo --html --output=docs doc/manual.texi
    - uses: peaceiris/actions-gh-pages@v3
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_dir: ./docs
```

这个 workflow 每次 push 到 main 时:

1. 装 Emacs
2. `makeinfo --html` 把 texi 编译成 HTML,输出到 docs/
3. `peaceiris/actions-gh-pages` 把 docs/ 部署到 GitHub Pages

结果:`https://you.github.io/my-package/` 自动有最新文档。**不用手动维护网站**——文档源码改一次,CI 自动更新网页。

### 3.2 readme.org (用 org-mode 写 README)

很多 Emacs 老兵用 org-mode 写 README——org 的 export 功能强,可以同时生成 Markdown(GitHub 渲染)和 HTML(网页)。

```org
* Overview

My package description.

* Installation
#+begin_src emacs-lisp
(use-package my-package :ensure t)
#+end_src
```

可以用 `M-x org-md-export-to-markdown` 转换为 README.md。优势:org 是 Emacs 原生格式,你写文档时能用 org 的所有功能(folding、链接、表格)。劣势:多一步 export,容易忘。

---

## 4. docstring 也要好

Texinfo 是"包级别"文档,docstring 是"函数级别"文档。两者互补——Texinfo 讲整体设计,docstring 讲单个 API。

### 4.1 第一行

简洁 (~70 字符),总结功能:

```elisp
(defun my-todo-add (text priority tags)
  "Add a new todo item with TEXT, PRIORITY, and TAGS."
  ...)
```

第一行 docstring 显示在 `M-x` 的 echo area——用户调命令时看到。**所以第一行必须简洁清晰**——不能写一段,要一句话。~70 字符是经验值,超出会被截断。

### 4.2 完整 docstring

```elisp
(defun my-todo-add (text priority tags)
  "Add a new todo item with TEXT, PRIORITY, and TAGS.

TEXT is a string describing the todo.
PRIORITY is an integer from 1 to 5, where 5 is most urgent.
TAGS is a list of strings.

Returns the new item as a list (TEXT PRIORITY TAGS DONE).

\(fn TEXT PRIORITY TAGS)"
  ...)
```

完整 docstring 结构:

- 第一行总结
- 空行后逐参数说明(类型、语义)
- 返回值说明
- `\(fn ...)` 显式签名(byte-compiler 验证)

最后的 `\(fn ...)` 用反斜杠转义——因为单独的 `(fn ...)` 在 docstring 里有特殊含义(被 help-mode 解释)。加 `\` 让它显示为字面文本。

### 4.3 查 docstring

```
C-h f my-todo-add RET
```

看格式化效果。

**重要:写完 docstring 立刻用 `C-h f` 看效果**——很多 docstring 在源码里看着 OK,在 help buffer 里渲染糟糕(参数名没高亮、空行不对)。`C-h f` 是 WYSIWYG 的反馈。

---

## 5. 创造性用法

Texinfo 的强大超出"写文档":

1. **导出多格式**: 一个 `.texi` 出 Info + HTML + PDF + man——用户选喜欢的格式
2. **函数索引**: 自动从 `@defun` 生成"函数索引",用户 `M-x info-lookup-symbol` 跳到 API doc
3. **交叉引用**: `@xref{Configuration}` 在章节间跳,Info 模式下按 `Tab` 跟随
4. **菜单结构**: `@menu` 反映章节树,Info 模式下作为目录
5. **条件输出**: `@ifhtml ... @end ifhtml` 只在 HTML 显示;`@iftex` 只在 PDF 显示——同一源码针对不同输出
6. **嵌入示例**: `@example` 块自动等宽字体;`@lisp` 块按 Lisp 语法高亮
7. **自动生成版本**: `@set VERSION 1.0` 在多处引用,改一处全更新
8. **生成 man page**: `makeinfo --no-headers --output=man.1 manual.texi`——同一份源码出 man page

---

## 6. 实战

### Ex 6.1: 写 README

为你的 my-todo 写完整 README.md。要求:badges、description(2 段)、installation(use-package)、usage(表格)、configuration、license。

### Ex 6.2: 写 texi 手册

```bash
mkdir -p doc
touch doc/my-todo.texi
```

写完整手册:5 个 node(Introduction、Installation、Usage、Configuration、Index),包含至少 2 个 `@defun` 和 2 个 `@defvar`。

### Ex 6.3: 编译 texi

```bash
makeinfo doc/my-todo.texi          # → my-todo.info
makeinfo --html doc/my-todo.texi   # → HTML
```

两个都跑成功,确认无 error。

### Ex 6.4: 在 Emacs 里读

```
M-x info RET
```

`M-x info-find-node RET /path/to/my-todo.info RET`。从 Emacs 内部体验你的文档——这就是用户读你文档的方式。

### Ex 6.5: 完善 docstring

每个 public 函数都有完整 docstring(第一行总结 + 参数说明 + 返回值 + `\(fn ...)`)。`C-h f` 检查渲染。

---

## 7. 自测

1. README 标准章节?
2. Texinfo 的 `@defun` 干啥?
3. 怎么编译 texi 到 Info?
4. docstring 第一行应该多长?
5. README 和 Commentary 区别?

**答案**:
> 1. Description, Installation, Usage, Configuration, FAQ, License(可选 Comparison、Contributions)
> 2. 标记函数定义——自动进函数索引,Info 模式可查
> 3. `makeinfo FILE.texi`(生成 .info)
> 4. ~70 字符,一句话总结功能(显示在 `M-x` echo area)
> 5. README 在 GitHub(门面,给浏览者);Commentary 在 .el 文件头(`M-x finder-commentary` 看,给装包者)

---

## 8. 下一步

进入 `melpa-submission.md`。
