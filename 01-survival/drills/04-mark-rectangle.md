# Drill 04: Mark & Rectangle — 选区与矩形的艺术

> **预计用时**: 1.5 小时
> **难度**: ★★★☆☆ (矩形是反直觉的)
> **替代**: `emacs-manual/mark.texi` + `regs.texi` 矩形部分
> **目标**: 5 秒内选中任意区域;10 秒内做矩形编辑

---

## 0. 学习目标

- [ ] 区分 point、mark、region、rectangle
- [ ] 用 `C-SPC` 设 mark,用 `C-x C-x` 交换
- [ ] 用 `M-h`、`C-M-h`、`C-x h` 快速选段/defun/buffer
- [ ] 用 `C-x r t` 在矩形每行加前缀 (经典场景: 注释代码块)
- [ ] 用 `C-x r k`/`C-x r y` 杀/粘矩形

---

## 1. 第一性原理: 为什么 Emacs 的 region 这么强?

### 1.1 Region 的本质

要理解 region,先要理解 point 和 mark。这两个都是**整数** (buffer 里的位置索引),不是"对象"。

**Region** = point 和 mark 之间的文本。

```
[Buffer 文本]
Hello, world!
   ↑ point (当前光标)
              ↑ mark (用 C-SPC 设)

[point 和 mark 之间 = region]
```

**与 VS Code 的区别**:
- VS Code: 选中 = 鼠标拖一段连续字符
- Emacs: region 由两个**整数** (point 和 mark) 定义,可以不"高亮"

`transient-mark-mode` (默认开) 让 region 高亮。
关了之后 region 还在,只是没颜色。

这个"region 不一定要高亮"的设计在 1985 年是激进的。当时大多数编辑器把"选中"等同于"高亮"——没有高亮就没有选中。Emacs 把 region 抽象成"两个位置",高亮只是视觉表示。这让 region 可以**程序化操作**: `(buffer-substring (region-beginning) (region-end))` 取 region 内容,`(delete-region (region-beginning) (region-end))` 删 region。Elisp 代码可以无视视觉,直接操作 region。

**创造性应用**: 你可以写 Elisp 函数检查 region 是否匹配某个模式,如果匹配就处理它。比如: 选 region,`M-x my-sort-words` 对 region 里的词排序。这种"对 region 操作"的命令在 Emacs 包里无处不在——`upcase-region`、`comment-region`、`indent-region`、`fill-region`、`sort-lines`... 数百个。

### 1.2 Mark 的双重作用

Mark 有两个用途,这经常让新手困惑:

**用途 1**: 标记 region 起点 (设置后,point 和 mark 之间是 region)
**用途 2**: 跳回的"书签" (mark ring 历史栈)

| 操作 | 用途 |
|---|---|
| `C-SPC` | 设 mark,激活 region |
| `C-SPC C-SPC` (双击) | 设 mark,不激活 region (只记位置) |
| `C-u C-SPC` | 跳上一个 mark (pop-mark-ring) |

为什么 mark 既是 region 起点又是"书签"? 因为这两个用途共享同一个底层机制: **mark 是一个位置,mark-ring 是位置历史**。设 mark 时,前一个 mark 进 ring。激活 region 只是"视觉显示 point-mark 之间",不影响 mark 的本质。

**理解关键**: 设 mark 后,即使你按 `C-g` 取消 region 高亮,mark **仍然存在**。你之后可以 `C-u C-SPC` 跳回那个 mark。这是"光标的足迹"——你走过的每个设过 mark 的地方都能跳回。

### 1.3 Mark ring 的循环

每次设 mark,前一个 mark 推入 `mark-ring`:

```
设 mark at A:   mark-ring = []
设 mark at B:   mark-ring = [A],    current mark = B
设 mark at C:   mark-ring = [B A],  current mark = C
```

`C-u C-SPC` 弹一次:

```
mark-ring = [A B],   current mark = B    (跳到 B)
再 C-u C-SPC:
mark-ring = [B A],   current mark = A    (跳到 A)
再 C-u C-SPC:
mark-ring = [A B],   current mark = B    (循环!)
```

注意是**循环**,不是只 pop。

为什么是循环而不是栈? 因为用户经常想"在 A 和 B 之间反复跳"——跳到 A 看一眼,跳回 B 继续,再跳到 A。如果 mark ring 是栈 (pop 后消失),你跳到 A 后 mark-ring 就没 A 了,无法再跳。ring 的循环让你可以**来回穿梭**历史位置。

这个 ring 设计是 Emacs 文化的一个标志: 用户的位置历史是一个**环形**,你可以转一圈回到起点。这比 VS Code 的"线性 history" (按一下 alt+left 走一步,不能循环) 更灵活。

---

## 2. Mark 命令对照

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-SPC` | `set-mark-command` | 设 mark + 激活 region |
| `C-SPC C-SPC` | 双击 | 设 mark,不激活 region |
| `C-u C-SPC` | | 跳上一个 mark |
| `C-x C-x` | `exchange-point-and-mark` | 交换 point 和 mark (region 范围不变,光标到另一端) |
| `C-x C-SPC` | `pop-global-mark` | 跳到 global mark (跨 buffer) |
| `M-@` | `mark-word` | 设 mark 在 word 末 |
| `C-M-@` | `mark-sexp` | 设 mark 在 sexp 末 |
| `C-M-SPC` | 同上 | mark-sexp (变体) |
| `M-h` | `mark-paragraph` | 选中当前段 |
| `C-M-h` | `mark-defun` | 选中当前函数 (lisp) |
| `C-x h` | `mark-whole-buffer` | 选中全 buffer |
| `C-x C-p` | `mark-page` | 选中当前 page |
| `M-x mark-end-of-sentence` | | 设 mark 在句尾 |
| `M-x mark-word` | | 设 mark 在 word 末 |

---

## 3. 内联讲解: Region 操作

### 3.1 基础 region 操作

选中 region 后,以下命令对 region 生效:

| 命令 | 键位 | 作用 |
|---|---|---|
| `kill-region` | `C-w` | 杀 region |
| `kill-ring-save` | `M-w` | 复制 region |
| `delete-region` | (无默认键) | 删 region (不进 kill-ring) |
| `upcase-region` | `C-x C-u` | 大写 |
| `downcase-region` | `C-x C-l` | 小写 |
| `comment-or-uncomment-region` | `M-;` | 注释/取消 |
| `indent-region` | `C-M-\` | 缩进 |
| `fill-region` | `M-x fill-region` | 填充 (自动换行) |
| `sort-lines` | `M-x sort-lines` | 排序 |
| `reverse-region` | `M-x reverse-region` | 反转 |
| `uniq-region` | (无默认) | uniq |
| `shell-command-on-region` | `M-|` | shell 命令处理 region |
| `eval-region` | `M-x eval-region` | eval region 里的 Elisp |
| `query-replace` (region) | `M-%` | 只在 region 内替换 |
| `align-regexp` | `C-u M-x align-regexp` | 对齐 |

### 3.2 数字前缀 + region 命令

很多 region 命令接受前缀:
- `C-u M-;` 注释,不问方式 (强制用 `;; ` 或 mode 默认)
- `C-u C-x C-u` 严格大写 (即使已是)

---

## 4. 第一性原理: 矩形 (Rectangle) 是什么?

### 4.1 普通选区 vs 矩形

矩形是 Emacs 一个被低估的功能。理解它需要从"普通 region"的局限开始。

普通 region 是**连续文本**:

```
[abcdef]
[ghijkl]
[mnopqr]
选 "bc" "hi" "no" (一个矩形列):

普通 region: 无法表达
矩形:      可以! 它是 [(行,列) 的集合]
```

普通 region 只能表达"从位置 A 到位置 B 的连续字符"。如果你想选"每行的第 2-4 列",普通 region 做不到——它必须包含中间的所有字符。

矩形解决了这个问题。它定义一个**二维区域**: 行范围 + 列范围。`(行 1 到 3, 列 2 到 4)` 是一个矩形,覆盖 3 行 × 3 列共 9 个字符位置,但跳过每行之外的内容。

### 4.2 矩形的定义

矩形 = 从 `(start-row, start-col)` 到 `(end-row, end-col)` 的"列范围"。

```
设置: point 在 (row=1, col=1), mark 在 (row=3, col=3)

行 1: a[b]cdef      ← 选 (1,1) 到 (1,3)
行 2: g[h]ijkl      ← 选 (2,1) 到 (2,3)
行 3: m[n]opqr      ← 选 (3,1) 到 (3,3)
```

注意矩形的"宽度"由最左和最右列决定,与每行实际字符无关。

**陷阱**: 矩形操作对**短行**有奇怪行为。如果你选的矩形列范围超出某行的实际长度,Emacs 会插入空格补齐。比如第 3 行只有 2 个字符,你要选列 1-3,操作后第 3 行会被补成 3 字符 (多一个空格)。

### 4.3 矩形命令

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-x r k` | `kill-rectangle` | 杀矩形 |
| `C-x r d` | `delete-rectangle` | 删矩形 (不进 ring) |
| `C-x r y` | `yank-rectangle` | 粘最近矩形 |
| `C-x r M-w` | `copy-rectangle-as-kill` | 复制矩形 |
| `C-x r o` | `open-rectangle` | 在矩形位置插入空白 (文本右移) |
| `C-x r c` | `clear-rectangle` | 清空矩形 (替换为空格) |
| `C-x r t STRING RET` | `string-rectangle` | 矩形每行替换为 STRING |
| `C-x r N` | `rectangle-number-lines` | 给矩形每行加行号 |
| `C-x r r REGISTER` | `copy-rectangle-to-register` | 复制到 register |
| `C-x r i REGISTER` | (相同键) | 从 register 粘矩形 |
| `M-x replace-rectangle` | | 矩形替换 (会问) |

`C-x r t` 是用得最多的一个——给代码块加注释、给一列数据加前缀,都用它。

### 4.4 实战场景 1: 给代码块加注释

```
foo
bar
baz
```

想改成:

```
// foo
// bar
// baz
```

操作:
1. 光标在 `foo` 的 `f`,设 mark (`C-SPC`)
2. 移到 `baz` 的 `b` (或者 `C-n C-n`)
3. 现在 region 是 3 行 (普通 region)
4. 按 `C-x r t`
5. 输入 `// RET`
6. 三行前面都加了 `// `

**关键**: 矩形宽度 = mark 和 point 之间的列差。如果都在第 1 列,宽度=0,所以 `C-x r t STRING` 在每行行首插入 STRING。

这是矩形最常用的场景。比 `M-;` (comment-region) 更灵活——`M-;` 用 mode 默认注释符,`C-x r t` 你指定任意字符串。你可以加 `> ` (引用)、`- ` (列表)、`TODO ` 等任意前缀。

### 4.5 实战场景 2: 移除前缀

```
// foo
// bar
// baz
```

去掉 `// `:
1. 光标在第 1 行的 `/`,设 mark
2. 光标在第 3 行的 `/` 之后 (空格之前)
3. `C-x r k` (杀矩形)
4. 三行的 `// ` 都没了

这是上一个场景的反向。注意光标要精确在 `/` 上 (列 0) 和 `/` 之后 (列 3),才能选中 `// ` 这 3 列。

### 4.6 实战场景 3: 提取一列数据

CSV:
```
name, age, email
alice, 30, alice@example.com
bob, 25, bob@example.com
```

提取 email 列:
1. 设 mark 在第 1 行的 `email` 的 `e`
2. 光标移到第 3 行的 `bob@example.com` 的 `m`
3. `C-x r M-w` (复制矩形)
4. 移到目标位置
5. `C-x r y` (粘矩形)

得到:
```
email
alice@example.com
bob@example.com
```

这个场景在数据处理时极有用。你有一份 CSV 或表格,想提取某列到另一个地方,不用写脚本,几个键搞定。

### 4.7 `C-x r o` (open-rectangle)

在矩形位置**插入空白**,文本右移:

```
abc
def
ghi
```

选中第 2 列的 3 行,`C-x r o`:

```
a  bc
d  ef
g  hi
```

(中间插入了 2 个空格)

这个命令用来"在文本中间留出空白",常用于: 在代码里给某段腾出空间、在文档里做缩进、给标题腾位置等。

### 4.8 `C-x r N` (rectangle-number-lines)

给矩形每行加序号:

```
foo
bar
baz
```

选中第 1 列 3 行,`C-x r N`:

```
1 foo
2 bar
3 baz
```

(默认从 1 开始)

**创造性用法**: 给列表加编号、给 TODO 加优先级、给测试用例加序号。`C-u C-x r N` 可以指定起始数字和格式。

### 4.9 矩形的创造性应用汇总

1. **加注释**: `C-x r t // RET` 给一段加注释 (最常用)。
2. **去注释**: `C-x r k` 杀矩形,去掉前缀。
3. **提取列**: `C-x r M-w` 复制一列数据,`C-x r y` 粘贴。
4. **批量缩进**: `C-x r t SPC SPC SPC SPC RET` 给一段代码加 4 空格缩进。
5. **加序号**: `C-x r N` 给列表加 1. 2. 3.。
6. **加前缀符号**: `C-x r t - RET` 把文本变成 markdown 列表。
7. **`open-rectangle` 腾空**: `C-x r o` 在文本中间留空白,文本右移。
8. **`clear-rectangle` 清空**: `C-x r c` 把矩形内容替换为空格,保留布局。
9. **存矩形到 register**: `C-x r r a` 把矩形存到 register a,`C-x r i a` 恢复。
10. **批量重命名**: 选中一列变量名,`C-x r t new_name RET` 全部替换。

---

## 5. 微练习 (40 分钟,20 题)

### Mark 基础 (5 题)

1. 光标在某位置,`C-SPC`,移到另一位置,region 高亮
2. `C-x C-x` 交换 point 和 mark,光标跳到另一端
3. `C-g` 取消 region (region 仍存在但不显示)
4. `C-SPC C-SPC` 设 mark 但不激活 region
5. `C-u C-SPC` 跳上一个 mark

### Mark 扩展 (5 题)

6. `M-h` 选中当前段
7. `C-x h` 选中全 buffer
8. `M-@` 选中当前 word
9. `C-M-SPC` 选中当前 sexp
10. `C-M-h` 选中当前 defun (lisp 文件)

### Mark ring (3 题)

11. 设 mark at A,移到 B 设 mark,移到 C 设 mark,`C-u C-SPC` 跳 B,再 `C-u C-SPC` 跳 A,再 `C-u C-SPC` 跳 B (循环)
12. 在 buffer 1 设 mark,切到 buffer 2,`C-x C-SPC` 跳回 buffer 1
13. 用 `M-x mark-ring` 看 mark-ring 内容 (作为 list)

### Region 操作 (3 题)

14. 选中 region,`C-w` 杀
15. 选中 region,`M-;` 注释
16. 选中 region,`C-x C-u` 大写

### 矩形 (4 题)

17. 三行文本,选中行首 3 列,`C-x r t // RET` 加注释
18. 用 `C-x r k` 杀矩形
19. `C-x r y` 粘回来
20. `C-x r N` 给每行加行号

---

## 6. 实战练习 (20 分钟)

### 任务 1: 代码块注释

打开一个 .py 文件,选中一段代码 5-10 行:
1. 光标在第一行行首
2. `C-SPC`
3. `C-n C-n C-n C-n C-n` (下 5 行)
4. `C-x r t # RET` (Python 注释 `#`)
5. 撤销 `C-/`

### 任务 2: 提取 CSV 一列

```
id,name,age
1,alice,30
2,bob,25
3,carol,28
```

提取 name 列:
1. 光标在 `name` 的 `n`
2. `C-SPC`
3. 移到 `carol` 的 `l`
4. `C-x r M-w` (copy-rectangle)
5. 移到文件末尾
6. `C-x r y`

得到:
```
name
alice
bob
carol
```

### 任务 3: 缩进代码块

把一段代码整体缩进 4 空格:
1. 选中代码块 (行首到行首)
2. `C-x r o` (open-rectangle),在选中位置插入空格
3. 但 `C-x r o` 会问宽度,默认是 region 宽度 (0 不行)

正确做法:
1. 选中代码块第一列到第一列 (column 0 到 column 0,但实际上要 4 列宽)
2. 或者用 `C-x r t STRING` 在每行加 4 空格

### 任务 4: 用 register 存矩形

```
AAA
AAA
AAA
```

选中矩形 `AAA`, `C-x r r a RET` (存到 register a)。

清空矩形: `C-x r c`。

恢复: `C-x r i a RET`。

---

## 7. 高级技巧

### 7.1 `delete-indentation` (合并行)

`M-^` (delete-indentation) 把当前行和上一行合并:

```
foo
    bar
```

光标在 bar,`M-^`:

```
foo bar
```

(自动处理空白)

### 7.2 `fill-paragraph` (`M-q`)

`M-q` 在当前段填充 (按 `fill-column` 自动换行):

```
This is a long line that goes on and on and on and on and on and on...
```

`M-q`:

```
This is a long line that goes on and on and on and on and on and
on...
```

(假设 fill-column = 70)

### 7.3 `align-regexp` (对齐)

```
name = "alice"
age = 30
email = "alice@example.com"
```

选中三行,`C-u M-x align-regexp RET =`:

```
name  = "alice"
age   = 30
email = "alice@example.com"
```

(对齐 `=`)

---

## 8. 自测

1. `C-SPC C-SPC` (双击) 和 `C-SPC` 的区别?
2. `C-x C-x` 干啥?
3. `M-h` 选中什么?
4. 矩形和普通 region 的区别?
5. `C-x r t` 和 `C-x r o` 的区别?

**答案**:
> 1. 双击设 mark 但不激活 region (不显示高亮);单点激活 region
> 2. exchange-point-and-mark,交换 point 和 mark
> 3. mark-paragraph,当前段
> 4. 矩形是"列"概念,跨多行同列;普通 region 是连续文本
> 5. t 替换矩形内容为字符串;o 在矩形位置插入空白,原文右移

---

## 9. 毕业检查

- [ ] 20 题至少完成 16
- [ ] 能用 `C-x r t` 加注释,`C-x r k` 去注释
- [ ] 能从 CSV 提取一列
- [ ] 能用 `M-h` / `C-x h` 快速选中

完成后进入 `drills/05-kmacro.md`。
