# 80/20 Emacs: 20 个高频键覆盖 80% 日常

> 这个文件不讲"所有有用的命令",只讲**真正高频的 20 个**——会用这 20 个,你 80% 的日常操作已经飞快
> 剩下的 20% (各种专业工具) 边用边学

---

## 0. 80/20 心法

学 Emacs 最大的坑是"想学完"。**永远学不完**——MELPA 上 5000+ 包,内置命令 5000+。

真正的极客**先掌握 20 个核心键**,把日常工作流跑顺,然后**按需扩展**。

下面是我观察 30+ Emacs 老用户得出的"最高频 20 个"。**先练这 20 个到肌肉记忆**,其他都是衍生。

---

## 1. 移动 (4 个) — 用得最多

### `C-s` / `C-r` isearch
**为什么高频**: 99% 的"去某处"用 isearch 比挪光标快 10 倍。你不用记 "100 行下",直接 `C-s foo`。

**进阶关键**: 搜的时候按 `C-w` — **把光标处 word 加进搜索词**。例: 搜 `getUserById` 时,光标在 `getUser`,按 `C-s C-w`,瞬间搜到所有 `getUser` 开头的位置。

`C-s C-s` — 重复上次搜索 (不用重输)。

### `M-<` / `M->` 头尾
跳 buffer 头/尾。简单但天天用。

### `M-g g` (goto-line) + `M-g i` (imenu)
- `M-g g 50 RET` — 跳第 50 行
- `M-g i` — 弹"函数列表"菜单,选一个跳过去 (替代"翻 1000 行找函数")

### `C-u C-SPC` mark ring 跳回
**最被低估**。每次设 mark 都进 ring,`C-u C-SPC` 在历史里跳。例: 你在 A 跳到 B 跳到 C,`C-u C-SPC` 回 B,再按回 A。

---

## 2. 选中 (3 个) — 80% 不用 C-SPC

### `C-M-h` mark-defun
按一下选**整个函数**。改函数内某个变量名时必备。

### `M-h` mark-paragraph
按一下选**段落**。Markdown 写作天天用。

### `C-=` expand-region (装包)
按一下扩一级——word → sexp → 字符串 → defun → buffer。**一键选到任意范围**,极客最常用。

---

## 3. 杀/粘/历史 (4 个) — kill ring 核心

### `C-w` / `M-w` / `C-y` / `M-y`
剪/复制/粘/换前一个。

**关键**: `M-y` 必须在 `C-y` 之后。粘完发现不是想要的,按 `M-y` 换成 kill-ring 里前一个,再按再换。

### `C-k` kill-line
杀到行尾。按两次杀含换行。

### `C-S-backspace` kill-whole-line
杀整行 (含换行)。比 `C-k C-k` 快。

### `M-w` 复制行
没选 region 时,`M-w` 复制当前行 (有的 mode)。否则用 `C-SPC C-e M-w` (mark 行尾 + 复制)。

---

## 4. 改 (3 个) — 替换三件套

### `M-%` query-replace
**最高频的批量改**。选 region (或全 buffer) → `M-% old RET new RET` → 按 `y`/`n`/`!`。

- `y` 替换一个
- `!` 替换剩余所有 (确认安全后用)
- `q` 退出
- `u` undo 上一次

### `C-M-%` regex query-replace
正则版。例: 把 `foo123` `foo456` 都改成 `bar123` `bar456`:
```
C-M-% foo\([0-9]+\) RET bar\1 RET
```
`\1` 引用第一个括号组。

### `M-s o` occur
列出所有匹配,在 occur buffer 里浏览。

**进阶**: 在 occur buffer 按 `e` 进入 `occur-edit-mode`——直接改 occur,改完 `C-c C-c` 回写到原文件。**跨多行同时改**。

---

## 5. 跑命令 (3 个) — 元操作

### `M-x` execute-extended-command
跑任意命令。配合 vertico/orderless,输入几个字母就能找到。

### `C-h k` describe-key
不知道某键干啥,按 `C-h k` + 那个键,立刻看文档。

### `C-h f` / `C-h v` / `C-h m` describe-function/variable/mode
查函数/变量/当前 mode。**最高频的 help 命令**,不会用 help = 不会用 Emacs。

---

## 6. 文件/buffer/window (3 个) — 切换

### `C-x C-f` find-file
打开文件。

### `C-x b` switch-to-buffer
切 buffer。装了 consult 后变成"智能切"——同时显示 recent files/bookmarks。

### `C-x o` other-window
切 window。 (Doom 默认绑 `M-o` 更短)

---

## 7. Eval (1 个) — Lisp 用户必备

### `C-x C-e` eval-last-sexp
在 *scratch* 或 .el 里 eval 光标前一个表达式。**改完代码立即看效果**。

进阶: `C-u C-x C-e` 把结果插入到 buffer 里 (而不是只显示在 minibuffer)。

---

## 8. 完整 20 个清单

按"如果只能记 20 个"的顺序:

| # | 键 | 命令 | 频率 |
|---|---|---|---|
| 1 | `C-s`/`C-r` | isearch (+C-w) | 每 30 秒 |
| 2 | `C-x C-s` | save | 每分钟 |
| 3 | `C-g` | quit | 每分钟 |
| 4 | `M-w`/`C-w`/`C-y`/`M-y` | kill ring | 每分钟 |
| 5 | `C-/` | undo | 每分钟 |
| 6 | `C-x b` | switch-buffer | 每分钟 |
| 7 | `C-x C-f` | find-file | 每 5 分钟 |
| 8 | `C-x C-e` | eval-last-sexp | 每 5 分钟 (Lisp) |
| 9 | `M-%` | replace | 每 10 分钟 |
| 10 | `M-h` / `C-M-h` | mark paragraph/defun | 每 10 分钟 |
| 11 | `C-x o` | other-window | 每 10 分钟 |
| 12 | `C-k` | kill-line | 每 10 分钟 |
| 13 | `M-x` | run command | 每 10 分钟 |
| 14 | `C-h k/f/v/m` | help system | 每小时 |
| 15 | `M-<` / `M->` | head/tail | 每小时 |
| 16 | `M-s o` | occur | 每小时 |
| 17 | `M-/` | hippie-expand (补全 word) | 每分钟 |
| 18 | `M-.` | find-definition | 每 10 分钟 |
| 19 | `C-u C-SPC` | mark ring jump | 每 10 分钟 |
| 20 | `C-x z` | repeat (按 z 继续) | 每小时 |

加 5 个"现代必装" (会一次,变肌肉记忆):
- `C-=` expand-region
- `C->` multiple-cursors
- `C-.` embark-act
- `M-g i` consult-imenu
- `C-:` avy-goto-char

**这 25 个 = 80% 日常。**

---

## 9. 练习计划 (3 周)

**Week 1**: 1-12 (基础)。目标: 不看键盘能按对。

**Week 2**: 13-20 + 现代必装。目标: 形成肌肉记忆。

**Week 3**: 综合练习——只用这 25 个键,完成你的日常工作。如果发现某操作"很慢",问"是不是用错了上面某个键"。

3 周后,你会**几乎不用鼠标**,飞快。

---

## 10. **不要**学的 (避免范围蔓延)

新手最大的坑: 想学太多。**先放下这些**:

- ❌ evil-mode (Vim 键位) — 默认键位够用,evil 是另一套体系
- ❌ 各种专业包 (mu4e, elfeed, exwm) — Module 5/6 之后再说
- ❌ Lisp 宏高级 — Module 6 W3 之后
- ❌ EIEIO/CLOS — Module 6 之后
- ❌ AI/LLM 集成 — 先把基础打牢

**先用 25 个键跑顺 1 个月**,再扩展。

---

## 11. 自测 (5 题)

合上手册,你能立刻答出吗?

1. 搜 "foo" 并把光标处 word 加进搜索词,怎么按?
2. 选整个函数,怎么按?
3. 粘贴后发现不是想要的,换成 kill-ring 里前一个,怎么按?
4. 把 region 里所有 `abc` 改成 `xyz`,最少的按键?
5. 跳回上一次 mark 位置,怎么按?

**答案**:
> 1. C-s C-w (C-w 取光标处 word)
> 2. C-M-h (mark-defun)
> 3. C-y M-y (粘贴后立刻按 M-y 切换)
> 4. M-% abc RET xyz RET ! (! 全替换)
> 5. C-u C-SPC (mark ring pop)

---

## 12. 真话

很多 Emacs 用户用了 5 年,还在按 `C-SPC + 挪光标`。**不是他们笨,是没人告诉他们这 20 个键**。

练好这 25 个,你的速度已经超过 80% 的 Emacs 用户。

剩下的 20%——专业包、特殊场景、Lisp 高级——按需学。**先让 80% 跑顺**。
