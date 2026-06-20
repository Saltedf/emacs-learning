# Emacs 极客学习路线图

> 一份基于第一性原理 + 做中学的 Emacs 学习路线,从零基础到开源贡献者。
> 配套教程完全自包含,不需要再翻开三本官方手册。
> 预计时长: 6-7 个月 (按 3-4 小时/天的强度)。

---

## 你的起点与终点

**起点** (假设):
- Emacs 当记事本用,会基本移动
- 有编程基础但不熟练,无 Lisp 经验
- 能投入 3-4+ 小时/天

**终点** (能做这些):
- 完全脱离鼠标,日常编辑飞快
- 能读懂任何 elisp 代码,能写自己的包
- 给 melpa 包贡献过 PR,能跟 emacs-devel 邮件列表讨论

---

## 五条第一性原理 (贯穿全程)

1. **Emacs 是 Lisp 机器伪装成编辑器** — 学 Emacs = 学 Elisp
2. **一切皆 Buffer** — 文件、目录、shell、邮件都是 buffer
3. **自文档化** — `C-h f/v/k/m/i` 永远是第一手资料
4. **Live system** — 函数热重载,`eval-buffer` 立即生效
5. **可组合性** — 任何函数都可重定义,没有"插件边界"

---

## 三本手册的角色 (穿插式学习)

```
emacs-manual     → 概念锚点 (What/Why,用户视角)
emacs-lispintro  → 入门实现 (How simple,Chassell 的教学法)
emacs-lispref    → 深度机制 (How complete,源码级)
```

**重要**: 配套教程已**完全吸收**三本手册的内容,你不需要再翻原始 texi 文件。
教程中每个概念都从"用户视角 → 简单 Elisp → 深度实现"三层打通。

---

## 9 个模块总览

| # | 模块 | 时长 | 核心产出 | 毕业项目 |
|---|---|---|---|---|
| 0 | 心法与装备 | 2 天 (~6h) | 编译的 Emacs + init.el | 跑通内置 tutorial |
| 1 | 编辑生存 | 1 周 (~20h) | 脱离鼠标做日常编辑 | 30 分钟无鼠标重构 2000 行代码 |
| 2 | Dired + Buffer/Window | 4 天 (~12h) | 文件管理在 Emacs 内 | 10 分钟整理混乱目录 |
| 3 | Elisp 入门 (Chassell) ★ | 5 周 (~100h) | Lisp 思维 | 写一个 todo-mode (200+ 行) |
| 4 | 配置体系 + Minor Mode | 2 周 (~40h) | 自己的 init.el | 500 行模块化 + 自定义 minor mode |
| 5 | Power User 工作流 | 3 周 (~60h) | Org/Magit/补全栈 | Org literate 配置 + Magit PR |
| 6 | Elisp 深度 (Reference) ★ | 8 周 (~160h) | 能读懂任何 elisp | 写一个 major mode (500+ 行 + ERT) |
| 7 | 写一个真实的包 | 3 周 (~60h) | 完整的开发发布流程 | 包被 MELPA 合并 |
| 8 | 开源贡献 | 持续 | 融入社区 | 给他人包提被合并的 PR |

★ = 核心模块,工作量最大

---

## 螺旋上升 (Spiral Curriculum)

不是"读完一本再读下一本",而是 4 圈穿越:

- **第 1 圈** (M0-2): 装备 + 编辑生存 + 最小化 Elisp (让 Emacs 听话)
- **第 2 圈** (M3-4): 系统读完 Chassell + 配成自己风格
- **第 3 圈** (M5-6): 攻克 Elisp Reference 核心 + 写真实可用的包
- **第 4 圈** (M7-8): 进入开源生态,贡献代码

---

## 每个模块的三段式

```
Read   →  教程内联讲解 (替代手册某章,加上 Why + 图 + 多例)
Drill  →  30-50 个微练习,形成肌肉记忆
Build  →  一个毕业项目,综合应用
```

---

## 三个里程碑

**3 个月后** (M0-4 完成):
- ✅ 完全脱离鼠标
- ✅ 有 500 行的模块化 init.el
- ✅ 能读懂大部分 elisp 代码
- ✅ 写过自定义 minor mode

**6 个月后** (M0-7 完成):
- ✅ 写过完整的 major mode (带 font-lock + indent + ERT 测试)
- ✅ 发布过自己的包到 MELPA
- ✅ Org/Magit/补全栈全部熟练

**9 个月+** (持续贡献):
- ✅ 给 melpa 包贡献过被合并的 PR
- ✅ 读 emacs-devel 邮件列表,参与讨论
- ✅ (可选) 给 Emacs core 提过 patch

---

## 真实能力测试 (6 个月后能答出来)

1. 解释 `lexical-binding: t` 这个 file-local variable 干了什么
2. 描述 `(defun foo () (let ((x 1)) (lambda () x)))` 返回的闭包行为
3. 区分 `(eq 'a 'a)` `(eq "a" "a")` `(equal "a" "a")` `(eql 1 1.0)`
4. 写一个 `make-process` 跑 `git status` 并把输出插入当前 buffer
5. 解释 `font-lock-defaults` 的结构
6. 写一个 `define-derived-mode` 继承 `prog-mode`
7. 解释 `with-current-buffer` 与 `save-excursion` 的区别
8. 用 Edebug 单步执行一个函数

如果一半以上答不出来,**不要进入下一个模块**。

---

## 配套文件索引

- `MANUAL-CROSSREF.md` — 三本手册的章节对照表 (查"我现在学的对应手册哪里")
- `PROGRESS.md` — 学习进度跟踪表
- `00-mindset/` 到 `08-open-source/` — 9 个模块的教程
- `logs/` — 学习日志

---

## 学习纪律

- **不跳模块**: 后面的模块依赖前面的肌肉记忆和概念
- **每个毕业项目必须完成** (或至少认真尝试过)
- **教程自包含**: 不需要翻原始 texi 手册,所有内容在教程里
- **遇到问题先用 `C-h` 系列**: `C-h f`/`C-h v`/`C-h k`/`C-h i` 比 Google 更权威
- **每周写学习日志**: `logs/module-XX.md`

---

## 推荐配置 (硬件/软件)

- **Emacs 版本**: 30.2+ (与目录中的手册版本对齐)
- **操作系统**: Linux/macOS (Windows 也可,但某些包不友好)
- **编译选项**: `--with-native-compilation --with-tree-sitter --with-x-toolkit=gtk3 --with-pgtk` (Wayland)
- **键盘**: 强烈推荐 HHKB 或带真正 Ctrl/Meta 位置的键盘,减少小拇指痛

---

开始: 打开 `00-mindset/README.md`。
