# Drill 02: Kill Ring — 死亡与重生的栈

> **预计用时**: 1.5 小时
> **难度**: ★★☆☆☆
> **替代**: `emacs-manual/killing.texi` 全文
> **目标**: 30 秒内完成 杀-召回-循环替换

---

## 0. 学习目标

- [ ] 解释 kill 与 delete 的本质区别
- [ ] 描述 kill-ring 的数据结构 (list + 游标)
- [ ] 30 秒内: 杀一行 → 召回 → 替换为前一次
- [ ] 用 kmacro 批量处理 50 个"杀-改-粘"
- [ ] 写一个把当前行 kill 到 register 的命令

---

## 1. 第一性原理: 为什么是 "Kill" 不是 "Cut"?

### 1.1 历史背景

1970s 的 TECO 和 Emacs 都运行在 paper terminal 上。删除一段文字,物理上就是"消失"。Stallman 的设计直觉是:**被删除的文字应该有家可归**。

这不是显然的设计。当时的编辑器 (vi 的前身 ed, TECO) 都是"删了就删了"。Stallman 在 GNU Emacs 里加了 kill ring——一个**有循环指针的栈**。这在 1985 年是革命性的。

为什么 Stallman 这么在意"删除可召回"? 因为他在 MIT AI Lab 用过太多"删错了就完蛋"的编辑器。他相信**用户犯错是常态,软件应该宽恕**。kill ring 是这种哲学的体现——你永远可以召回最近 60 次删除的任何一段。

### 1.2 第一性原理推导

让我从根本推出 kill ring 的设计:

- **起点**: 用户删除文本。
- **推论 1 (可召回)**: 删除是危险的,用户可能后悔。所以删除的东西应该有地方去。 → kill ring。
- **推论 2 (多次保留)**: 用户可能多次后悔 (想召回上上次的,不是上次的)。所以多次删除应该都保留,而不是只保留最新。 → list,不是单值。
- **推论 3 (循环浏览)**: 用户召回时可能想看历史里的每一项,然后回到最近的。所以是 ring (循环),不是 stack (LIFO,弹一次就消失)。 → ring 结构。
- **推论 4 (容量上限)**: 不能无限保留 (内存)。所以有上限,默认 60。 → `kill-ring-max`。
- **推论 5 (跨 session 不需要)**: 进程结束后,kill ring 没意义了 (操作系统有 clipboard)。所以不持久化。 → 进程结束就丢。
- **推论 6 (系统同步)**: GUI 下 Emacs 应该和系统 clipboard 同步 (浏览器复制的能在 Emacs 粘)。 → `interprogram-cut-function`。

这 6 条推论完全推出了 kill ring 的当前设计。如果你设计一个新的编辑器,这 6 条也会引导你得到类似的结构。

### 1.3 Kill vs Delete 的核心区别

Emacs 故意分了两套删除命令。理解这个区分是掌握 kill ring 的前提。

| 操作 | 进入 kill-ring? | 行为 |
|---|---|---|
| `C-w` (kill-region) | 是 | 杀掉的文本进 kill-ring |
| `M-d` (kill-word) | 是 | 同上 |
| `M-k` (kill-sentence) | 是 | 同上 |
| `C-k` (kill-line) | 是 | 同上 |
| `DEL` (delete-backward-char) | 否 | 真删除,无召回 |
| `M-DEL` (backward-kill-word) | 是 | 杀掉进 ring |
| `M-\` (delete-horizontal-space) | 否 | 真删除 |

**记忆口诀**: 进入 kill-ring 的命令都带 "kill" 字样;真删除的命令带 "delete"。

**为什么分两套?** 因为删除有两种语义。你可能后悔的 (想召回),和你确定不要的 (不召回)。如果你只有一个"删除"操作,无法区分这两种意图,kill-ring 会被无用的内容 (比如删的空格) 污染。

具体例子: 你按 Backspace 删一个打错的字符,你不希望这个字符进 kill-ring (挤掉有用的东西)。所以 Backspace 是 `delete-backward-char`,不进 ring。但 `M-d` 杀一个词,你可能后悔 (想召回),所以进 ring。

这种"语义区分"的设计贯穿 Emacs,是它的核心哲学之一: **意图应该在键位层面表达,不是事后猜测**。

### 1.4 Kill-ring 的数据结构

在 `*scratch*` eval:

```elisp
kill-ring
;; => ("最近杀的" "前一次" "再前一次" ...)
```

```elisp
kill-ring-yank-pointer
;; => ("最近杀的" "前一次" ...) — 指向当前 yank 会取的那个
```

`C-y` 召回 `car` of `kill-ring-yank-pointer`。
`M-y` (after `C-y`) 把指针向后移一位,并替换刚才 yank 的内容。

**画图理解**:

```
kill-ring:        [A] -> [B] -> [C] -> nil
                   ^
                   |
yank-pointer ------+
```

`M-y` 后:

```
kill-ring:        [A] -> [B] -> [C] -> nil
                           ^
                           |
yank-pointer --------------+
```

到尾部后**循环回 A** (这是 kill-ring 与普通栈的本质区别)。

为什么用 list 加 pointer 实现环,而不是用真正的 ring buffer? 因为 Elisp 的 list 是 immutable cons cells,改变 pointer 只需要 `(setq pointer (cdr pointer))`,O(1)。如果用 vector 加 mod 运算,虽然也是 O(1),但 list 更"Elisp 风格"。

**陷阱**: `kill-ring-yank-pointer` 不是独立的 ring,它**指向 kill-ring 的某个 cons cell**。所以 `(cdr kill-ring-yank-pointer)` 是 ring 中下一项,`(setq kill-ring-yank-pointer (cdr kill-ring-yank-pointer))` 让指针前进。到尾部时 `(cdr ...)` 是 nil,这时 yank-pop 会把指针重置到 `kill-ring` 头——这就是"循环"。

---

## 2. 内联讲解: Kill 命令家族

### 2.1 Kill 命令对照表

| 命令 | 键位 | 杀什么 | 进 kill-ring? |
|---|---|---|---|
| `kill-region` | `C-w` | 选中的 region | ✅ |
| `kill-word` | `M-d` | 光标后一个 word | ✅ |
| `backward-kill-word` | `M-DEL` / `C-M-w`* | 光标前一个 word | ✅ |
| `kill-line` | `C-k` | 光标到行尾 | ✅ |
| `kill-sentence` | `M-k` | 光标到句尾 | ✅ |
| `kill-paragraph` | 无默认键 | 当前段 | ✅ |
| `kill-sexp` | `C-M-k` | 一个 sexp | ✅ |
| `zap-to-char` | `M-z CHAR` | 到下一个 CHAR (含) | ✅ |
| `zap-up-to-char` | `M-Z CHAR` | 到下一个 CHAR (不含) | ✅ |
| `kill-whole-line` | `C-S-backspace` | 整行含换行 | ✅ |
| `delete-backward-char` | `DEL` | 前一字符 | ❌ |
| `delete-char` | `C-d` | 后一字符 | ❌ |
| `delete-horizontal-space` | `M-\` | 周围空白 | ❌ |
| `delete-blank-lines` | `C-x C-o` | 多个空行合一 | ❌ |
| `delete-indentation` | `M-^` | 合并当前行和上一行 | ❌ |

\* `backward-kill-word` 默认是 `C-M-backspace` 或 `M-DEL`,有些终端会拦截 `C-M-` 组合。

### 2.2 `C-k` 的细节 (新手最容易踩的坑)

`C-k` 是 `kill-line`,看起来简单——杀到行尾。但它有几个让人困惑的边界情况。

打开一个空 buffer,输入 `abcdef`,光标在 `a`,按 `C-k`:

```
abcd|ef        ← 之前
abcd|          ← 之后 (光标没动,kill 了 "ef" 但没动换行)
```

注意光标**没动**,而且**换行没被杀**。`C-k` 杀的是"光标到行尾"的文本,不含换行符。

如果 `abcdef` 是 buffer 的最后一行 (无换行),按 `C-k` 后:

```
abcdef|        ← 之前
|             ← 之后 (空 buffer,光标在行首)
```

这种情况下 `C-k` 杀了所有内容,buffer 变空。

按 `C-u C-k` 或 `C-k C-k` (连按两次):

```
|ef           ← 之后 (kill 了 "abcd" + 换行,光标在新行行首)
```

这是 `C-k` 最让人困惑的地方: **第一次按杀到行尾,第二次按连换行符一起杀**。这导致 `C-k C-k` 杀整行 (含换行),而 `C-k` 一次只杀到行尾。

**底层机制**:

`kill-line` 调用 `kill-region`,后者调用 `kill-new` 把文本推入 `kill-ring`。但如果区域为空 (光标就在行尾),`kill-line` 会先调用 `(forward-line 1)`,然后 kill 该位置。这就是为什么 `C-k` 在行尾会"跨行"。

**创造性用法**:
- `C-k C-k` (双击): 杀整行含换行,常用于"删一行"。
- `C-u 5 C-k`: 一次杀 5 行 (含换行)。比按 5 次 `C-k C-k` 快。
- `M-0 C-k`: 杀从行首到光标。先 `C-a` 到行首,`C-u 0 C-k` 杀到原位置——但这其实是另一种方式: `M-0 C-k` 杀"行首到 point",配合 point 在中间位置使用。

### 2.3 Append Next Kill (`C-M-w`)

通常每次 `kill-*` 都会在 kill-ring 里增加一项。但有时你想合并多次 kill:

- `C-w` (kill 第一段)
- `C-M-w` (宣告下次 kill append,不增加新项)
- `C-w` (kill 第二段,这次与前一次合并)

**底层**: `interprogram-cut-function` 和 `kill-new` 检查 `last-command`:
如果是 `(kill-region kill-append ...)`,就把新文本 append 到 `car kill-ring`,而不是 cons 新项。

**实际场景**: 你想复制 3 段不连续的文本,合并粘到一处。
- 选中第一段,`M-w` (copy)
- 选中第二段,`M-w` 但**先 `C-M-w`**: `C-M-w M-w` (append copy)
- 选中第三段,`C-M-w M-w`
- `C-y` 一次粘全部三段

**陷阱**: `C-M-w` 必须**紧跟在前一次 kill/copy 之后**,且下一次 kill/copy 之前不能有别的命令。如果你中间按了 `C-n` (移动),`C-M-w` 的"append 标记"丢失,下次 kill 不 append。

**为什么 `C-M-w` 不常用?** 因为大多数人不知道它存在。但一旦掌握,你会发现它非常有用——尤其是"收集多段到一处"的场景。

### 2.4 Yanking (`C-y` 和 `M-y`)

`C-y` 调用 `yank`,做两件事:
1. 把 `kill-ring-yank-pointer` 重置到 `kill-ring` 头部
2. 在 point 插入 `(car kill-ring-yank-pointer)`,并把 point 设到插入文本的起始

紧接着按 `M-y` 调用 `yank-pop`:
1. 检查 `last-command` 是不是 `yank`,否则报错 "Previous command was not a yank"
2. 把 `kill-ring-yank-pointer` 向后移动一位 (循环到尾后回头部)
3. 删除刚才 yank 的文本
4. 插入新的 car

**循环机制**:

```elisp
;; 简化版 yank-pop 内部逻辑
(setq kill-ring-yank-pointer
      (or (cdr kill-ring-yank-pointer)  ; 向后移
          kill-ring))                    ; 到尾了,循环回头
```

**陷阱 1**: `M-y` 必须紧跟 `C-y`,否则报错 "Previous command was not a yank"。新手经常困惑——按了 `C-y`,想了想,按了 `C-l` (recenter),然后按 `M-y` 报错。解决: `M-y` 必须在 `C-y` 之后立刻按,中间不要按任何其他命令。

**陷阱 2**: `M-y` 在 `M-y` 之后也可以——你可以连续 `M-y M-y M-y` 浏览 kill-ring 的多项。这是因为 yank-pop 自己也是 yank 类命令,last-command 保持为 yank。

**陷阱 3**: `M-y` 之前如果 yank 了相同的内容 (比如 `C-y` 两次),行为可能让人困惑。yank-pop 删除"刚才 yank 的"内容,这通过 mark 记录位置。如果 yank 被部分修改过,yank-pop 可能删错。

### 2.5 Kill-ring 的容量与配置

```elisp
kill-ring-max        ;; 默认 60,可以改大
kill-ring            ;; 实际的 list
kill-ring-yank-pointer ;; 游标
interprogram-cut-function  ;; 同步到系统剪贴板的函数 (GUI 下)
interprogram-paste-function ;; 从系统剪贴板读
```

**陷阱**: kill-ring 默认 60 项。如果你 kill 100 次,前 40 个丢了。`setq kill-ring-max 200` 改大。

**陷阱**: GUI 下 kill 自动同步系统 clipboard。但 TTY 下不会——因为没有 X clipboard。`interprogram-cut-function` 设 nil 关闭同步。

**练习**: 在 `*scratch*` 里 eval `(setq kill-ring-max 200)`,然后连续 kill 100 段,用 `(length kill-ring)` 验证。

### 2.6 与 Register 的对比

| 维度 | Kill ring | Register |
|---|---|---|
| 数据结构 | list (FIFO 环) | 命名 slot (a-z 0-9) |
| 命名 | 匿名 | 你给名字 |
| 容量 | `kill-ring-max` (60 默认) | 任意多 |
| 持久化 | 进程结束就没 | 同 (除非持久化) |
| 用途 | 短期多次 yank | 长期持有特定文本/位置 |

Kill ring 像"近期剪贴板历史",register 像"我标记的固定位置"。

**何时用 register 而不是 kill ring?**
- 你要在整个 session 里反复粘贴同一段长文本 → register (kill ring 会被新内容挤掉)。
- 你要在两个特定位置之间反复跳 → register 位置 (`C-x r SPC` / `C-x r j`)。
- 你要存一个矩形区域 → register (`C-x r r`)。

**何时用 kill ring 而不是 register?**
- 你最近杀的几段,想召回 → kill ring (`C-y` `M-y`)。
- 你不确定要召回哪一段,想浏览 → kill ring。
- 跨 buffer 复制 → kill ring (`M-w` 在一个 buffer,`C-y` 在另一个)。

### 2.7 `M-w` (copy) 的细节

`M-w` (kill-ring-save) 复制 region,不删除。这是 Emacs 的 "Ctrl+C"。

变体:
- `M-w` 直接: 复制 region
- `M-w` 后立刻 `C-y`: 复制然后立即 yank (相当于 duplicate)
- 系统剪贴板同步: 默认 GUI 下,`M-w` 也更新系统剪贴板

**创造性用法**:
- **跨 buffer 复制**: 一个 buffer `M-w`,切到另一 buffer `C-y`,自动跨。这是 Emacs 的常规工作流——所有 buffer 共享同一个 kill-ring。
- **多次粘贴同一段**: `M-w` 一次,然后多次 `C-y`。每次 `C-y` 都从 kill-ring 头部取,粘的就是同一份。
- **复制整行不选 region**: `M-w` 在没有 active region 时复制当前行 (Emacs 25+ 行为)。比 `C-a C-SPC C-e M-w` 快得多。

变量 `mouse-drag-and-drop-region` 让你可以拖拽 region。

### 2.8 矩形 kill

矩形操作 (后续 Drill 04 详细):

```
C-x r k    kill-rectangle (杀矩形,进 kill-ring 单独区域)
C-x r y    yank-rectangle (粘矩形)
```

矩形 kill 不进 `kill-ring`,有单独的 `killed-rectangle` 变量。

为什么矩形不进 kill-ring? 因为 kill-ring 存的是字符串,矩形是 2D 数据。如果硬塞,需要某种"分隔符"约定,会很丑。所以 Emacs 给矩形单独的存储——`killed-rectangle` 变量,只存最近一个矩形。

**创造性用法**: 复制一列数据 (CSV 的某列),`C-x r M-w` (copy-rectangle-as-kill),移到目标位置,`C-x r y` 粘矩形。这在数据处理时很有用。

### 2.9 Kill ring 的创造性应用汇总

学完上面,试试这些意外用法:

1. **多段合并粘贴**: `C-w` → `C-M-w` (append) → `C-w` → `C-y` 一次粘两段。省去"粘两次"的麻烦。
2. **临时记事本**: 把 kill-ring 当 clipboard history,`M-x browse-kill-ring` 浏览 (装 browse-kill-ring 包)。
3. **跨 buffer 复制**: 一个 buffer `M-w`,切到另一 buffer `C-y`,自动跨。
4. **代码片段管理**: 把常用代码段 kill 到 register,需要时插入 (`C-x r s` / `C-x r i`)。
5. **替换 region 为历史**: 选 region → `M-y` 循环 kill-ring 替换。
6. **从外部粘贴长内容**: 浏览器复制一段长文本,在 Emacs `C-y`,自动进 kill-ring (GUI 下)。然后 `M-y` 看历史。
7. **`browse-kill-ring` 编辑历史**: 装这个包后 `M-x browse-kill-ring` 显示所有 kill-ring 内容,可以编辑、删除、选择粘贴。比 `M-y` 循环更直观。

---

## 3. 微练习 (40 分钟,25 题)

每题先在 `*scratch*` 或临时 buffer 准备文本,然后操作,最后 `C-/` 撤销。

### 基础 (★,10 题)

1. 输入 "hello world",光标在行首,杀 "hello " (`M-d`)
2. 召回 (`C-y`)
3. 召回前一次 (`C-y M-y`)
4. 连续按 `M-y` 5 次,观察指针循环
5. 杀一整行 (`C-k` 一次,观察是否包含换行)
6. 杀一整行含换行 (`C-k` 两次,或 `C-S-backspace`)
7. 杀到下一个 "world" (`M-z world RET`)
8. 在 buffer 输入 3 行文本,光标在第 2 行,`C-S-backspace` 杀整行,看光标去哪
9. `M-w` 复制一行 (不删),然后 `C-y` 粘贴
10. `C-u 3 C-k` 杀 3 行 (含换行)

### 中级 (★★,8 题)

11. 杀一个 sentence (`M-k`),注意 sentence 定义 (用 `C-h v sentence-end-double-space`)
12. 杀一段 paragraph (`M-{` 移段头,`M-h` mark-paragraph,`C-w`)
13. 杀矩形区域 (`C-SPC` 起点,移动,`C-x r k`)
14. **append next kill**: 杀第一段 (`C-w`),按 `C-M-w`,杀第二段,观察 kill-ring 只增加一项
15. 用 numeric prefix: `C-u 3 M-d` 一次杀 3 个 word
16. 用 `M-\` 删光标周围空格,`M-SPC` 只留一个空格
17. `C-x C-o` 合并多个空行为一个
18. `M-^` 合并当前行和上一行 (delete-indentation)

### 高级 (★★★,7 题)

19. 写一个函数: 把当前行 (不含换行) 追加到 register `a`:

    ```elisp
    (defun my-kill-line-to-register-a ()
      "Kill current line (without newline) into register a."
      (interactive)
      (let ((beg (line-beginning-position))
            (end (line-end-position)))
        (copy-to-register ?a beg end t))) ; t = delete source
    ```
    绑定到 `C-c k a`,试用。

20. 用 `M-x list-registers` 看所有 register
21. 用 `M-x browse-kill-ring` (装了 browse-kill-ring 包) 看 kill-ring 完整内容
    或直接 eval `kill-ring` 在 `*scratch*` 里看
22. 写一个 minor mode 让 `M-y` 总是弹出 menu (研究 `consult-yank-from-kill-ring` 怎么做的)
23. 你杀的内容会自动同步到系统剪贴板 (GUI 下)。研究变量 `interprogram-cut-function`,理解机制
24. 把 `(setq save-interprogram-paste-before-kill t)` 加到 init.el,然后从外部 (浏览器) 复制一段,在 Emacs 里 `C-y`,看会取到外部内容
25. **Capstone 微版**: 写一个命令 `my-kill-ring-rotate`,把 `kill-ring-yank-pointer` 强制重置到 head

### 答案 (题 25)

```elisp
(defun my-kill-ring-rotate ()
  "Reset kill-ring-yank-pointer to head."
  (interactive)
  (setq kill-ring-yank-pointer kill-ring)
  (message "kill-ring-yank-pointer reset to head"))
```

---

## 4. 实战练习 (20 分钟)

打开 `/home/sun/src/learning/emacs-learning/emacs-manual-30.2/killing.texi`,完成:

1. 把所有 `@section` 标题改成大写 (用 `C-x r t` string-rectangle)
2. 复制所有 `@code{...}` 内的命令名到 buffer 末尾
3. 用 kmacro 录一个"找下一个 `@code`,复制其中文本"的宏,运行 20 次

全程不要用鼠标。

**kmacro 录制提示**:

```
F3
C-s @code{
C-f
C-SPC
C-s }
C-b
M-w
M->           ; 跳 buffer 尾
C-y
C-n           ; 下一行
F4
```

`C-u 20 C-x e` 运行 20 次。

---

## 5. 自测 (5 分钟)

回答 (合上手册):

1. `C-k` 在行首按一次,会杀掉什么? 按两次呢?
2. `kill-ring` 默认长度是多少? 用哪个变量改?
3. `M-y` 不在 `C-y` 之后直接按,会怎么样?
4. 为什么 Emacs 把"删除"分成 kill/delete 两套?
5. 解释 `kill-ring-yank-pointer` 的作用,以及它如何实现 `M-y` 的循环。

**答案** (反白):
> 1. 按一次杀到行尾不含换行 (除非行尾就是 buffer 末尾);按两次杀到下一行行首 (含换行)
> 2. `kill-ring-max` 默认 60
> 3. 报错 "Previous command was not a yank"
> 4. 因为有些操作你确定不要 (如 `DEL` 退格),有些可能要召回。两套语义清晰。
> 5. 它是 kill-ring 的一个"游标",指向当前 yank 会取的位置。`M-y` 把它向后移动 (到末尾后回环),并 replace-region 替换刚才 yank 的内容。

---

## 6. 毕业检查

- [ ] 能在 30 秒内: 杀一段 → yank → M-y 替换为前两次杀的内容
- [ ] 能解释 kill-ring 是 list + yank-pointer 的组合,不是栈
- [ ] 能写出第 19 题的函数,并理解 `copy-to-register` 的第 4 个参数 `delete-flag`
- [ ] 完成第 4 节的实战练习

完成后在 `logs/module-01.md` 记 "Kill Ring 用时 X,熟练度 ★★★★☆",进入 `drills/03-search.md`。

---

## 7. 延伸 (可选)

- 看 `simple.el` 中 `kill-new`、`kill-append`、`current-kill` 的源码 (`C-h f kill-new RET` 然后点链接)
- 装 `browse-kill-ring` 包,体验 menu-style yank
- `consult` 包的 `consult-yank-from-kill-ring` 提供补全式 yank
