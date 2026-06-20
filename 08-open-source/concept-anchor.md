# Concept Anchor: Editor Manual (Module 8)

> 替代 `emacs-manual-30.2/glossary.texi` + `ack.texi` + `gnu.texi` 复习

这个文件是 Module 8 的**术语锚点**。你已经在 Module 0-7 见过这些术语的零散用法,这里把它们集中复习,因为贡献开源时你会反复看到这些词——读 docstring、读 commit log、和 maintainer 邮件往来,术语必须精确。Maintainer 不喜欢"那个东西" 这种模糊说法,你必须能说出 "kill-ring" 和 "kill" 的区别。

Emacs 的术语大多是 1970-80 年代定的,有些来自 Lisp Machine 时代 (Zmacs),有些来自更早的 TECO Emacs。这些词不"现代化"是有理由的——50 年的文档、代码、社区记忆都基于这些词。改一个词相当于改 API,会破坏所有现存文档。

---

## 1. Glossary (glossary.texi)

### 1.1 完整术语表

(Mono 0 的 concept-anchor 提过部分,这里完整)

下面的术语在 Module 8 里你需要**熟练用**——和 maintainer 沟通、写 docstring、读别人的代码。每个术语都给了简短解释,因为开源贡献里精确表达最重要。

| 术语 | 含义 |
|---|---|
| **Abbrev** | 缩写展开 |
| **Auto Fill** | 自动换行 |
| **Auto-save** | 自动保存 (#FILE#) |
| **Backup** | 备份 (FILE~) |
| **Balanced Parentheses** | 配对括号 |
| **Binding** | 键位 → 命令的映射 |
| **Buffer** | 文本容器 |
| **Byte Compile** | 编译到 .elc |
| **Coding System** | 字符编码 |
| **Command** | interactive 函数 |
| **Comment** | 注释 |
| **Completion** | 补全 |
| **Condition-case** | 异常 |
| **Custom** | customize 系统 |
| **Daemon** | 后台 Emacs |
| **Defun** | 函数定义 |
| **Echo Area** | 最底消息区 |
| **Face** | 颜色/字体规格 |
| **File Local Variable** | 文件局部变量 |
| **Fill** | 自动换行 |
| **Font Lock** | 语法高亮 |
| **Form** | Lisp 表达式 |
| **Frame** | GUI 窗口 |
| **Function** | 函数 |
| **Garbage Collection** | GC |
| **Hook** | 钩子 (函数列表) |
| **Isearch** | 递增搜索 |
| **Keymap** | 键位映射 |
| **Key Sequence** | 键序列 |
| **Kill Ring** | kill 历史 |
| **Kill** | 杀 (进 ring) |
| **Lambda** | 匿名函数 |
| **Lexical Binding** | 词法作用域 |
| **Local Variable** | 局部变量 |
| **Macro** | Lisp 宏 |
| **Major Mode** | 主模式 |
| **Mark** | 标记 |
| **Mark Ring** | mark 历史 |
| **Menu Bar** | 顶部菜单 |
| **Minibuffer** | 命令交互 buffer |
| **Minor Mode** | 次模式 |
| **Mode Line** | 状态行 |
| **Narrowing** | 限制视图 |
| **Native Compilation** | 编译到原生码 |
| **Package** | Lisp 包 |
| **Point** | 光标位置 |
| **Predicate** | 返回 bool 的函数 |
| **Prefix Argument** | 前缀参数 |
| **Process** | 外部进程 |
| **Quote** | 引用 (不 eval) |
| **Region** | point 和 mark 之间 |
| **Register** | 命名 slot |
| **Regular Expression** | 正则 |
| **S-expression** | Lisp 表达式 |
| **Shell Mode** | shell buffer |
| **Syntax Table** | 语法表 |
| **Text Properties** | 文本属性 |
| **Tool Bar** | 图标工具栏 |
| **Tooltip** | 悬停提示 |
| **Undo** | 撤销 |
| **Variable** | 变量 |
| **Window** | 分屏区域 |
| **Window Manager** | 窗口管理器 |
| **Yank** | 粘贴 |

### 1.2 几个容易混的术语

贡献时最容易混的是这几对:

- **Buffer vs Window**: Buffer 是文本容器 (在内存里),Window 是显示 buffer 的屏幕区域。一个 buffer 可以被多个 window 显示;一个 window 一次只显示一个 buffer。
- **Frame vs Window**: Frame 是 GUI 窗口 (一个 OS-level 窗口);Window 是 Frame 内的分割区域。这个区别是 Emacs 用 X11 时代的术语——其他编辑器叫 "window" 的,Emacs 叫 "frame"。
- **Kill vs Delete**: Kill 进 kill-ring (可以 yank 回来);Delete 不进 (永久消失)。Delete 永远比 Kill 危险。
- **Point vs Mark**: Point 是光标当前位置;Mark 是"标记"位置 (用 `C-SPC` 设置)。Region 是 point 和 mark 之间的文本。
- **Major Mode vs Minor Mode**: 每个 buffer 一个 major mode (定义语法、键位、缩进);minor mode 可叠加多个 (highlight-chars、whitespace、linum 等)。

这几个区别在做贡献时每天都会遇到——maintainer 不会替你纠正这些词。

---

## 2. Acknowledgments (ack.texi)

### 2.1 关键贡献者

(详见 Module 0)

Emacs 50 年历史,贡献者上万人。最关键的几位你应该认识:

- **Richard Stallman**: 创始人,1976 年开始写 TECO Emacs,1984 年开始写 GNU Emacs。没有他就没有 Emacs,没有 FSF,没有 GPL。
- **David Lawrence**: 早期 posting 系统 (Gnus 的前身)
- **Jamie Zawinski**: XEmacs (1991 年 fork 出去的版本,XEmacs 和 GNU Emacs 分裂了 20 年,2015 年基本死掉)
- **Michael McNamara**: 各种贡献

完整列表见 `C-h i m emacs RET Acknowledgments RET`。每次 Emacs release,这个列表都会更新——如果你想成为 Emacs 历史的一部分,你的名字可以出现在这里。

### 2.2 FSF

Free Software Foundation 是 Emacs 背后的法律实体:

- **资助开发**: 雇佣一些核心 maintainer (如 Eli Zaretskii 部分时间)
- **维护 Savannah**: GNU 自己的代码托管 (git.savannah.gnu.org)
- **法律保护 GPL**: 起诉违反 GPL 的公司,保证 Emacs 的 license 有牙齿

https://www.fsf.org/

FSF 完全靠捐赠运作。如果你从 Emacs 受益多,可以考虑每年捐 $50-100——这是回报的最简单方式。

---

## 3. GNU Project

(Module 0 学过,这里强调贡献)

### 3.1 自由软件哲学

GNU 项目是 1983 年 Stallman 发起的,目标是创造一个完全自由的操作系统。Emacs 是 GNU 最早的产品之一 (1985 年第一个 release)。哲学:

- **用户控制软件,不是被软件控制**——闭源软件让开发者控制用户,这是道德错误
- **源码必须可获取**——没有源码,"修改"的权利是空的
- **修改和分发是权利**——不是被授予的特权

### 3.2 实践

作为 Emacs 用户,你已经是 GNU 哲学的实践者。如果想更深入:

- **用自由软件替代专有**——比如用 GIMP 替代 Photoshop,用 LibreOffice 替代 Word
- **贡献自由软件**——写代码、写文档、报 bug、捐钱
- **宣传理念**——告诉朋友为什么软件自由重要

很多 Emacs 极客的"扩展使命"是推广 GNU 哲学。这不是政治——这是关于软件应该怎么做的根本信念。

---

## 4. 你的位置

学完 Module 0-7,你已经是 Emacs 极客——你能用 Emacs 高效工作、写任意 Elisp、做自己的包、发布到 MELPA。

Module 8 让你**融入社区**。这一步不是技术上的,而是社交上的:你从"用户"变成"贡献者"。社区接受你的标志是你的第一个 PR 被合并——那一刻,你的名字永久进了某个项目的 git history。

这也是回报的起点。Module 0-7 你吸收;Module 8 你释放。这种"吸收-释放"的循环是开源社区的生命——每一代用户都从上一代吸收,然后给下一代释放。你现在是这个链条的一环。

---

## 5. 自测

1. **buffer** 是什么?
2. **point** 和 **mark** 区别?
3. **kill** 和 **delete** 区别?
4. **major mode** 和 **minor mode** 区别?
5. **lexical binding** 是什么?

**答案**:
> 1. 文本容器,与 window 解耦——buffer 在内存,window 显示 buffer
> 2. point 是当前位置;mark 是标记位置 (用 C-SPC 设置)
> 3. kill 进 kill-ring (可 yank);delete 不进 (永久消失)
> 4. major 每 buffer 一个,定义语法和键位;minor 多个可叠加
> 5. 词法作用域,定义时捕获变量 (vs dynamic binding,运行时查找)

---

## 6. 下一步

进入 `ref-deep-dive.md` 看 internals。
