# Drill 05: Keyboard Macros — 录制你的重复劳动

> **预计用时**: 1.5 小时
> **难度**: ★★★☆☆
> **替代**: `emacs-manual/kmacro.texi` 全文
> **目标**: 30 秒内把 100 行重复编辑自动化

---

## 0. 学习目标

- [ ] 录制一个宏,运行 N 次
- [ ] 用 `F3` 插入 counter
- [ ] 编辑已录制的宏
- [ ] 命名宏,转成函数,保存到 init.el

---

## 1. 第一性原理: 为什么用 Kmacro?

### 1.1 重复劳动的两种解决方案

每个程序员都遇到过这种场景: 你有一系列相似的操作要做 50 次。比如:
- 把 50 个 markdown 文件的标题改成锚点链接
- 在 100 行代码的每行末尾加分号
- 把 30 个函数名 `foo` 重命名成 `bar`

这类任务有两种自动化方案:

| 方案 | 适用 | 速度 |
|---|---|---|
| Elisp 函数 | 通用、可复用 | 写起来慢,运行飞快 |
| Kmacro | 一次性、特定 | 录起来快 |

**Kmacro 不是 Elisp 替代品**,而是"快速自动化"。
当你发现某段操作要做 50 次:
- 慢方法: 一行一行做
- 快方法: 录一个 kmacro,跑 50 次

**何时用 kmacro 而不是 Elisp?** 经验法则:
- 操作很简单 (几个键),但要做很多次 → kmacro
- 操作有复杂逻辑 (条件、循环、变量) → Elisp
- 你不确定操作是不是对,想"试一下" → kmacro (录错了改,不写代码)
- 这个操作你以后还会做,要可复用 → Elisp (写成函数)

### 1.2 Kmacro 的本质

Kmacro 录的是**键序列**。运行时,把键序列"回放"。

```
录制:    F3 ... 键序列 ... F4
         ↓
存储:    last-kbd-macro = <键序列>
         ↓
运行:    C-x e  →  回放 <键序列>
```

这个简单的模型有深刻含义。kmacro 录的不是"操作" (高层的),而是"键" (低层的)。所以 kmacro 回放时,每按一个键就触发对应的命令——和手动按完全一样。这意味着:
- 如果你的 kmacro 录了 `C-s`,回放时会执行 isearch (而不是某个抽象的"搜索"操作)
- 如果 mode 不同,同一个键可能跑不同命令,kmacro 在不同 mode 里行为可能不同
- minibuffer 输入也被记录 (因为输入也是键)

这个"录键不录操作"的设计让 kmacro 简单但强大。简单是因为不需要解析意图,直接回放键;强大是因为可以录任何能按出来的操作。

### 1.3 适用场景

- 批量改一系列相似的行
- 复制数据,粘贴到模板
- 加序号、加注释、加前缀
- 把一段 elisp 函数应用到 100 个文件

### 1.4 不适用

- 需要复杂逻辑 (条件、循环、异常处理)
- 需要数学计算 (counter 简单的可以)
- 长期使用 (那种应该写成 Elisp 函数)

**判断标准**: 如果录完宏后你想"这个宏我要存下来以后用",那就该写成 Elisp 函数。如果只是"这次用完就丢",kmacro 合适。

### 1.5 历史: 为什么 Emacs 有 kmacro

1970s 的编辑器大多有"录制宏"功能 (因为那时没有 Elisp 这种脚本)。Stallman 把这个传统带进 Emacs,即使 Emacs 有完整的 Elisp,kmacro 仍然保留——因为它是"快速自动化"的工具,不需要写代码。

很多现代编辑器 (VS Code、Sublime) 也有类似的"录制"功能,但通常不如 Emacs kmacro 强大。Emacs kmacro 的独特之处: counter、命名、编辑、绑定键、转成 Elisp 函数——它是一套完整的"宏工程"系统,不只是录制。

---

## 2. 基本流程

### 2.1 录制 + 运行

```
F3 (或 C-x ()
... 操作 ...
F4 (或 C-x ))
```

按 `F4` 后,宏**立即运行一次** (这是设计)。

### 2.2 再运行

```
C-x e    再运行一次 (kmacro-end-and-call-macro)
```

### 2.3 运行 N 次

```
C-u 5 C-x e     运行 5 次
C-u 100 C-x e   运行 100 次
M-5 C-x e       同上 (M-num 等于 C-u num)
```

### 2.4 在 region 每行运行

```
C-x C-k r    apply-macro-to-region-lines
```

选中 region (整行),宏在每行运行一次。

---

## 3. Counter

### 3.1 录制时插入 counter

```
F3    (开始录制 + 插入 counter 0)
F3    (继续录制 + 插入 counter 1)
F4    (结束)
```

### 3.2 重置 counter

```
C-x C-k C-c N RET   设 counter 起始值为 N
```

### 3.3 counter format

```
C-x C-k C-f FORMAT   设 counter 格式 (例 "%03d" 补零)
```

### 3.4 实战: 给 100 行加序号

```
foo
bar
baz
...
```

想变:

```
1. foo
2. bar
3. baz
...
```

操作:
1. 光标在 `foo` 行首
2. `F3` 开始录制
3. (counter 0 自动插入) `.` `SPC` (`.` 和空格)
4. `C-n` 下一行
5. `C-a` 行首
6. `F4` 结束
7. `C-u 99 C-x e` 运行 99 次

但 counter 从 0 开始,不对。
重置:
- `C-x C-k C-c 1 RET` 设起始为 1
- 再录

或录的时候按 `F3` 第一次插入的是 0,但 counter 是从那开始增的。
所以 `F3` 录的时候,counter 是 0,1,2,...

实际:
1. `C-x C-k C-c 1 RET` 设从 1 开始
2. 光标行首
3. `F3` (插入 1)
4. `. SPC`
5. `C-n C-a`
6. `F4`
7. `C-u 99 C-x e`

---

## 4. 命名 + 保存

### 4.1 给宏命名

```
C-x C-k n NAME RET   name-last-kbd-macro
```

变成一个 Elisp 函数。`M-x NAME` 调用。

### 4.2 绑到键

```
C-x C-k b KEY   kmacro-bind-to-single-key
```

绑到 `C-x C-k 1` (single-key)。可以 1-9 或 a-z。

### 4.3 生成 Elisp 代码 (保存到 init.el)

```
M-x insert-kbd-macro RET NAME RET
```

会在当前 buffer 插入:

```elisp
(fset 'my-macro
   [?\C-a ?\C-  ?\C-e ?\M-w ?\C-n ?\C-y])
```

把这个加到 `init.el`,宏就持久化了。

---

## 5. 编辑宏

### 5.1 进入编辑

```
C-x C-k e RET   edit-kbd-macro (然后输入宏名或 RET 编辑 last)
```

打开 `*edit-macro*` buffer:

```
Keyboard Macro:

Type C-h m in this buffer for more info.
;; Original keys: C-a M-f M-b C-n

C-a        ;; move-beginning-of-line
M-f        ;; forward-word
M-b        ;; backward-word
C-n        ;; next-line
```

每行是一个 event,可以增删改。

### 5.2 应用编辑

`C-c C-c` 提交,宏被更新。

### 5.3 步进执行

```
C-x C-k SPC   kmacro-step-edit-macro
```

单步执行宏,每步问要不要执行、要不要记录。

---

## 6. Kmacro Ring

宏也存 ring (和 kill ring 类似):

```
C-x C-k C-p   上一宏
C-x C-k C-n   下一宏
C-x C-k C-d   删除当前宏
C-x C-k l     list 最近宏
C-x C-k C-v   view 当前宏
```

`kmacro-ring-max` (默认 8) 控制大小。

---

## 7. Kmacro 内联讲解: 高级技巧

### 7.1 在 minibuffer 里用宏

录制时如果进入 minibuffer (例如 `C-s` 搜索),宏会记录所有 minibuffer 输入。
运行时也回放。

这是一个微妙的行为。kmacro 录的是**所有键事件**,包括 minibuffer 里输入的字符。所以你录的宏如果包含 `C-s foo RET`,回放时会执行"搜 foo 然后回车"——这通常是你想要的。

**陷阱**: 如果你录的宏依赖某个 buffer 状态 (比如 buffer 里有 "foo" 这个词),回放时 buffer 没那个词了,宏会卡在 isearch。所以 kmacro 对"结构性"操作 (移动、复制) 友好,对"内容相关"操作 (搜索特定词) 容易出错。

### 7.2 用 `\C-` 写键

`insert-kbd-macro` 输出会用向量格式:

```elisp
(fset 'my-macro
   [?\C-a ?\M-f ?\C-n])
```

向量里每个元素是一个 event:
- `?a` = 字符 a
- `?\C-a` = C-a
- `?\M-f` = M-f
- `return` = RET
- `[tab]` = TAB

这个格式不直观,但精确。`?\C-a` 是 Elisp 的字符字面量,表示 Ctrl+A 的字符码 (1)。

### 7.3 用 `kbd` 函数 (更易读)

```elisp
(fset 'my-macro (kbd "C-a M-f C-n"))
```

`kbd` 把字符串转成 event 向量。

`kbd` 是宏,它接受一个**字符串** (像 "C-a M-f C-n"),解析后返回一个事件向量。这是写 kmacro 的现代方式,比 `?\C-a` 字面量可读得多。

**创造性用法**: 你可以不录宏,直接用 `kbd` 写宏。比如你想让 `C-c d` 执行"行首插入 TODO: 然后下一行",写:
```elisp
(fset 'my-todo (kbd "C-a TODO: RET C-n"))
(global-set-key (kbd "C-c d") 'my-todo)
```
不用录,直接写。这种"代码即宏"的方式适合简单的、明确的操作。

### 7.4 在宏里跑 Elisp

按 `M-:` 在录制中跑一段 Elisp,会被记录。
运行时也跑。

例: 录制时 `M-: (insert (format-time-string "%Y-%m-%d")) RET`,宏记录了 `M-:` + 输入。

**这是 kmacro 的"杀手级特性"**。你可以在宏里嵌入任意 Elisp,让宏有计算能力。比如:
- `M-: (insert (number-to-string (+ 1 (random 100))))` 插入随机数
- `M-: (insert (format "%s" (buffer-name)))` 插入 buffer 名
- `M-: (shell-command "date" t)` 插入 shell 命令输出

这些 Elisp 在宏回放时执行,结果插入到 buffer。这让 kmacro 不再是"纯录键",而是"录键 + Elisp 嵌入"——强大得多。

### 7.5 kmacro 的创造性应用汇总

1. **批量加 TODO**: 录一个"找下一个 def,加 TODO 注释"的宏,跑 50 次。
2. **CSV 转表格**: 给每行加 `| ` 前缀和 ` |` 后缀。
3. **提取注释**: 找所有 `//` 开头的行,复制到 buffer 末尾。
4. **批量重命名**: 配合 query-replace 或 search,把一列变量改 名。
5. **加序号**: 用 counter 给 100 行加 `1.` `2.` `3.`...
6. **批量格式化日期**: 录一个"读日期,改格式"的宏。
7. **配合 Elisp**: `M-:` 在宏里跑任意 Elisp,让宏有计算能力。
8. **跨文件应用**: `dired` 标记多个文件,`C-x C-k r` 在每个文件里跑宏。
9. **步进调试**: `C-x C-k SPC` 单步执行宏,每步问要不要执行,debug 录错的宏。
10. **持久化宏**: `M-x insert-kbd-macro` 生成 Elisp 代码,粘到 init.el,宏永久可用。

---

## 8. 微练习 (15 题)

### 基础 (5 题)

1. 录一个宏: 行首插入 `// `,下一行。运行 10 次。
2. 录一个宏: 杀当前行,粘到 buffer 末尾。运行 5 次。
3. 录一个宏: 在当前 word 后加 ` - DONE`。运行 5 次。
4. 录一个宏: 跳到行尾,加 `;`,下一行。运行 10 次。
5. 录一个宏: `C-s` 搜 "TODO",`M-y` 粘 "FIXME" 替换。运行 5 次。

### Counter (3 题)

6. 给 20 行加序号 `1.`、`2.`、`3.`...
7. 给 20 行加 `001`、`002`、`003` 格式 (用 `C-x C-k C-f "%03d" RET`)
8. 录一个宏: counter + 一行模板,例如 `Item N: __`。运行 5 次。

### 命名保存 (3 题)

9. 录一个有用的宏,`C-x C-k n my-macro RET` 命名
10. `M-x insert-kbd-macro RET my-macro RET`,生成 Elisp 代码
11. 把代码粘到 init.el,重启 Emacs 验证 `M-x my-macro` 还能用

### 高级 (4 题)

12. `C-x C-k e RET` 编辑 last macro,改一两个键
13. `C-x C-k SPC` 步进执行一个宏,看每步干啥
14. 录一个宏调用 Elisp: `M-: (insert (number-to-string (+ 1 (string-to-number (thing-at-point 'line)))))` —— 但这其实不适合宏,用 Elisp 函数更好
15. **场景**: 你有 50 个 markdown 文件,每个都要把 `# Title` 改成 `# Title {#title-anchor}`。录一个宏,用 `dired` + `C-x C-k r` 批量应用

---

## 9. 实战练习 (20 分钟)

### 任务 1: CSV 转 Markdown 表格

```
name,age,email
alice,30,alice@example.com
bob,25,bob@example.com
```

转成:

```
| name  | age | email              |
|-------|-----|--------------------|
| alice | 30  | alice@example.com  |
| bob   | 25  | bob@example.com    |
```

宏可以处理一部分,但复杂的对齐还是用 `orgtbl-mode` 或 `align-regexp` 更好。

简化任务: 给每行加 `| ` 前缀和 ` |` 后缀:

```
| name,age,email |
| alice,30,... |
```

宏:
1. `F3` `| SPC` `C-e` `SPC |` `RET` `F4`
2. `C-u 3 C-x e`

### 任务 2: 批量加 TODO 注释

代码里有 50 个函数,你想给每个加 TODO 注释:

```
def foo():
    pass
```

变成:

```
def foo():
    # TODO: implement
    pass
```

宏:
1. 光标在 `def foo():`
2. `F3`
3. `C-s def` (跳到下一个 def)
4. `C-e` (行尾)
5. `RET`
6. `    # TODO: implement`
7. `F4`
8. `C-u 49 C-x e`

### 任务 3: 提取代码注释到单独 buffer

```
// TODO: fix this
function foo() { ... }

// FIXME: bug here
function bar() { ... }
```

把所有 `// TODO` 和 `// FIXME` 提取到 buffer 末尾。

宏:
1. `F3`
2. `C-s //` (找下一个注释)
3. `C-a C-SPC` (mark 行首)
4. `C-e` (到行尾)
5. `M-w` (复制)
6. `M->` (buffer 末尾)
7. `C-y RET` (粘贴 + 换行)
8. `M-<` (回 buffer 头)
9. `C-s //` (下一个注释,避免重复)
10. `F4`
11. `C-u 10 C-x e`

---

## 10. 自测

1. `F3` 和 `F4` 各干啥?
2. 怎么运行宏 100 次?
3. `C-x C-k r` 干啥?
4. 怎么给宏命名?
5. 怎么把宏保存到 init.el?

**答案**:
> 1. F3 开始录制,F4 结束录制并运行一次
> 2. `C-u 100 C-x e`
> 3. apply-macro-to-region-lines,在选中 region 的每行运行宏
> 4. `C-x C-k n NAME RET`
> 5. `M-x insert-kbd-macro RET NAME RET` 生成 Elisp 代码,粘到 init.el

---

## 11. 毕业检查

- [ ] 15 题至少完成 12
- [ ] 能用 counter 给 50 行加序号
- [ ] 能命名宏并保存到 init.el
- [ ] 能用 `C-x C-k r` 批量应用

完成后进入 `drills/06-windows-frames.md`。
