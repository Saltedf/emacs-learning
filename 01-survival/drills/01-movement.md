# Drill 01: Movement — 像弹钢琴一样移动光标

> **预计用时**: 2 小时
> **难度**: ★☆☆☆☆ (概念简单,肌肉记忆需要时间)
> **替代**: `emacs-manual/basic.texi` 的移动章节
> **目标**: 不用鼠标,在任何 buffer 里快速定位

---

## 0. 学习目标

- [ ] 30 秒内,从 buffer 头到 buffer 尾 (任意文件)
- [ ] 5 秒内,跳到任意函数定义 (`imenu`)
- [ ] 不看键盘,完成字符/词/句/段/行 移动
- [ ] 用 prefix arg 精准跳 N 个单位
- [ ] 用 `C-l` 在 window 顶/中/底 之间重定位

---

## 1. 内联讲解: Emacs 的移动哲学

### 1.1 为什么不用方向键?

新手会想用 ↑↓←→ (PC 上的方向键)。这看起来是最自然的选择——所有现代编辑器都用方向键。但 Emacs 默认的移动是 `C-p` (上)、`C-n` (下)、`C-b` (左)、`C-f` (右)。

为什么不直接用方向键?

```
方向键位置: 远离 home row (手指要离开 jkl;)
C-p/n/b/f: 都在 home row 附近,不离开
```

这是**肌肉效率**的考量。home row 是 jkl;,你的食指、中指、无名指自然落下。方向键在键盘右下角,要用右手离开 home row 才能按。每次离开 home row,你失去 0.3-0.5 秒,而且要"重新定位"——眼睛可能要看键盘,注意力被打断。

Emacs 的设计假设: **专业打字员一天按几千次移动键**,这些操作必须能在 home row 完成。`C-p/n/b/f` 满足这个要求——Control 用小拇指按,p/n/b/f 是右手 home row 的位置,操作起来像"弹钢琴"。

**记忆口诀**:
- `C-p` = **p**revious (上一行)
- `C-n` = **n**ext (下一行)
- `C-b` = **b**ackward (向后 = 向左)
- `C-f` = **f**orward (向前 = 向右)

这 4 个字母不是随便选的——p (previous) 在前,n (next) 在后,b (backward) 在左,f (forward) 在右,字母本身暗示方向。这是 Stallman 在 1976 年的设计,沿用至今。

**第一性原理**: 长期编辑效率。手指不离 home row 是 Geek 之道。

但你**可以用方向键**,Emacs 也支持。等熟练后再切到 C-p/n 系列。

**历史小插曲**: 1976 年的 Space-cadet 键盘上,Control 键在 home row 旁边 (现在的 Caps Lock 位置),按 C-p 不用伸小拇指。现代 PC 键盘把 Ctrl 移到左下角,所以有了"Pascal 手腕"说法——Emacs 用户的小拇指容易疼。这就是为什么后来有 evil-mode (Vim 键位) 和 god-mode (改 modifier)。如果你想长期用 Emacs,考虑把 Caps Lock 改成 Ctrl (`setxkbmap -option ctrl:nocaps` on Linux,或在 macOS 系统设置里改)。

### 1.2 Emacs 的"移动单位"层次

Emacs 的移动命令有一个清晰的**层次结构**: 字符 < 词 < 句 < 段 < defun < buffer。每个层次有对应的 forward/backward 命令。

```
字符    <  词  <  句  <  段  <  defun  <  buffer
↓        ↓     ↓     ↓       ↓          ↓
C-f/b   M-f/b M-a/e M-{/}   C-M-a/e   M-</>
```

**单位越大,跳得越远**。看你要去哪儿,选合适单位。

这个层次结构反映**人类编辑的认知粒度**。你编辑文字时,思维单位不是"字符",而是"词"、"句"、"段"。比如"把这个词改成那个词"、"删这一段"、"跳到下一个函数"。Emacs 让每个思维单位都有一个对应的命令。

创造性用法: 想跳到当前函数末尾,不要 `C-n` 几十次,直接 `C-M-e` (end-of-defun)。想跳到当前段末,`M-}`。这种"按思维单位跳"的习惯一旦养成,你的编辑速度会提升一个数量级。

**指数思维**: `C-u 10 C-f` 一次跳 10 字符。但如果你要跳 100 字符,用 isearch (`C-s`) 加 `C-w` (取 word) 更快。这就是 Emacs 的"指数思维": 不要线性按 100 次,用更高级的单位。优秀的 Emacs 用户**几乎不按 10 次同样的键**——他们总是找更大的单位。

### 1.3 Forward vs Backward 的语义

Emacs 的 forward/backward 有歧义:

| 语境 | Forward | Backward |
|---|---|---|
| 字符 | 向右 | 向左 |
| 词 | 向右一词 | 向左一词 |
| 句 | 下一句 | 上一句 |
| 段 | 下一段 | 上一段 |
| Buffer | buffer 头 | buffer 尾 (混乱,见下) |

**Buffer 头/尾的特别说明**:
- `M-<` (beginning-of-buffer) = 跳到头
- `M->` (end-of-buffer) = 跳到尾

注意没有 "forward"/"backward",因为这是绝对位置。

为什么 forward 在 buffer 层级会混乱? 因为"forward"在不同视角下含义不同。从 point 视角,forward 是"buffer 增大方向" (向尾);从阅读视角,forward 是"下一行" (向尾)。两者一致。但 `M-<` 跳到头是"backward",`M->` 跳到尾是"forward"——和字符层的 forward (向右 = 向尾) 一致,但用 `<` `>` 符号表示更直观 (`<` 指向左/头,`>` 指向右/尾)。

### 1.4 Screen vs Window vs Buffer

新手经常混淆 screen、window、buffer。让我们厘清:

- **Buffer**: 所有文本 (可能比屏幕大)。一个 buffer 是 Emacs 内存里的一段字符串,可能 100 行也可能 100 万行。
- **Window**: 你看到的部分 (Frame 内分屏的一块)。一个 window 显示一个 buffer 的**一部分**。如果 buffer 比 window 大,window 显示 buffer 的某个范围,其余的需要 scroll 才能看到。
- **Screen**: 整个屏幕 (一个 Frame)。一个 Frame 内可以有多个 window。

移动命令分两类:
- **buffer 移动**: 改 point 在 buffer 中的位置
  - `C-f`/`C-b`/`C-n`/`C-p`/`M-f`/`M-b`/...
- **window 移动**: 改 window 显示 buffer 的哪部分 (point 不动)
  - `C-v` (scroll-up): 显示 buffer 的"下一屏"
  - `M-v` (scroll-down): 显示 buffer 的"上一屏"

这个区分很重要: **point 是 buffer 的属性,不是 window 的**。你 scroll window,buffer 里的 point 不动——只是你看不到了。scroll 回去,point 还在那里。如果你在 window A 看某个 buffer,window B 也看同一个 buffer,两个 window 共享同一个 point (除非你用 `clone-indirect-buffer`)。

理解这一点,你就理解了 `C-M-v` (scroll-other-window) 的妙处: 光标留在当前 window,但另一个 window 滚动——你可以在写代码的同时"翻"参考文档的页面,不需要切 window。

---

## 2. 基础移动 (10 题,30 分钟)

打开一个练习 buffer:

```
M-x find-file RET ~/emacs-practice.txt RET
```

输入以下文本作为练习材料:

```
The quick brown fox jumps over the lazy dog.
Pack my box with five dozen liquor jugs.
She sells seashells by the seashore.

How vexingly quick daft zebras jump!
The five boxing wizards jump quickly.
Sphinx of black quartz, judge my vow.

function foo(a, b) {
    return a + b;
}

function bar(c, d) {
    return c * d;
}
```

把光标放到 buffer 头 (`M-<`)。

### 题 1.1: 字符移动

- 光标在 `T` of "The"
- 按 `C-f`,光标移到 `h`
- 再按 3 次,光标在 `q`
- 现在按 `C-b` 5 次,光标回到 `T`

### 题 1.2: 用 prefix arg

- 光标在 `T`
- 按 `C-u 5 C-f` (或 `M-5 C-f`)
- 光标应该跳 5 个字符,到 `u` of "quick"

### 题 1.3: 上下移动

- 光标在第一行
- 按 `C-n`,光标到第二行
- 按 `C-p`,回到第一行
- 按 `C-u 3 C-n`,跳 3 行

### 题 1.4: 行首行尾

- 光标在第一行某处
- 按 `C-a`,光标到行首 (第一列)
- 按 `C-e`,光标到行尾

**注意 `C-a` 的细节**:
- 默认 `C-a` 跳到第一列
- 如果设了 `(setq-default visual-order-mode t)`,可能跳到第一个非空白字符
- `C-a C-a` (双击) 在两者间切换 (smart tab 模式)

### 题 1.5: 词移动

- 光标在 `The` 的 `T`
- 按 `M-f`,光标到 `quick` 的 `q`
- 再按 `M-f`,到 `brown` 的 `b`
- 按 `M-b`,回到 `quick` 的 `q`

**词的定义**: 由 `fill-prefix` 和 syntax table 决定。默认一个 word 是连续的字母数字。

**陷阱**: 词的定义**不是固定的**,它取决于当前 major mode 的 syntax table。在 lisp-mode 里,`-` 是 word 字符 (所以 `foo-bar` 是一个词);在 c-mode 里,`-` 不是 word 字符 (所以 `foo-bar` 是两个词)。这有时让人困惑: 在不同 mode 里 `M-f` 行为不同。

**创造性用法**: 用 `M-f` + `M-b` 在词之间弹跳,比 `C-f` 按几次快得多。如果你要选一个词,光标在词中任意位置,`M-b M-f` 把光标移到词首,然后 `M-@` (mark-word) 选中。整个流程 3 键。

### 题 1.6: 句子移动

- 光标在第一句 `The quick ... dog.`
- 按 `M-e`,跳到下一句首 `Pack ...`
- 按 `M-a`,回到上一句

**句子定义**: `.`/`?`/`!` 加空格或换行 (受 `sentence-end-double-space` 影响)。

**陷阱**: 句子的定义有个历史包袱。早期英文打字机时代,句号后要双空格 (那是固定宽度字体的规范)。所以 Emacs 默认认为句号后**两个空格**才结束一句。这导致 `M-a`/`M-e` 在现代单空格文本里行为奇怪——它可能跳过单空格的句号,因为认为不是句末。

**解决**: `(setq sentence-end-double-space nil)` 加到 init.el,让 Emacs 用单空格作为句末。但注意,有些排版习惯仍用双空格,所以默认保留。这是 Emacs 尊重历史习惯的体现。

### 题 1.7: 段落移动

- 光标在第一段
- 按 `M-}`,跳到下一段开头 (空行后)
- 按 `M-{`,回到上一段

**段落定义**: 空行分隔。

**陷阱**: `paragraph-start` 和 `paragraph-separate` 这两个变量控制段落边界。在有些 mode 里 (如 org),段落定义更复杂。如果 `M-}` 行为奇怪,`C-h v paragraph-start` 看当前设置。

### 题 1.8: Buffer 头尾

- 按 `M-<`,光标到 buffer 头
- 按 `M->`,光标到 buffer 尾
- 用 prefix: `C-u 5 M-<` 跳到 buffer 的 5% 位置 (高级特性)

### 题 1.9: 配对的括号

- 光标在 `function foo(a, b) {` 的 `{` 上
- 按 `C-M-n` (forward-list),跳到匹配的 `}`
- 按 `C-M-p` (backward-list),跳回 `{`

**list 跳转**: 在所有括号配对间跳,不只 `{}`,`()`, `[]` 都行。

### 题 1.10: Recenter

- 光标在某行
- 按 `C-l`,当前行居中
- 再按 `C-l`,当前行到顶
- 再按 `C-l`,当前行到底

`C-l` 是 `recenter-top-bottom`,在三个位置循环。

---

## 3. 词与 Sexp (5 题,15 分钟)

### 题 2.1: 标记 word

- 光标在 `The` 的 `T`
- 按 `M-@` (mark-word)
- region 被激活,选中 `The`

### 题 2.2: mark-sexp

- 光标在 `(a, b)` 的 `(`
- 按 `C-M-SPC` (mark-sexp)
- 整个 `(a, b)` 被选中

### 题 2.3: forward-sexp

- 光标在 `(a, b)` 的 `(`
- 按 `C-M-f` (forward-sexp)
- 光标跳到 `)` 之后

### 题 2.4: backward-sexp

- 光标在 `)` 之后
- 按 `C-M-b` (backward-sexp)
- 光标回到 `(`

### 题 2.5: transpose-sexps

- 光标在 `foo bar` 中间
- 按 `C-M-t` (transpose-sexps)
- 变成 `bar foo`

**sexp 定义**: 一个"完整表达式",可以是符号、字符串、(balanced parentheses)。

---

## 4. 跳转和书签 (5 题,15 分钟)

### 题 3.1: imenu

打开一个代码文件 (任意 .py 或 .js):
```
M-x find-file RET ~/some-code.py RET
M-x imenu RET foo RET
```
光标跳到 `foo` 函数定义。

(`imenu` 自动扫描代码结构,提供函数/类/方法列表)

### 题 3.2: imenu 列表

```
M-x imenu RET RET
```
(第二个 RET 不输入名字,显示所有候选列表)

### 题 3.3: register 跳转

- 在某行,按 `C-x r SPC m` (存 point 到 register m)
- 移动到 buffer 别处
- 按 `C-x r j m`,跳回刚才位置

### 题 3.4: bookmarks

```
M-x bookmark-set RET mybookmark RET   ; 设置书签
M-x bookmark-jump RET mybookmark RET  ; 跳到书签 (会重新打开文件)
M-x list-bookmarks                    ; 列所有书签
```

书签**跨 session 保留** (存在 `~/.emacs.d/bookmarks`)。

### 题 3.5: occur 跳转

```
M-s o function RET
```

`*Occur*` buffer 显示所有 "function" 的匹配,点击或 `RET` 跳转。

---

## 5. Scroll (5 题,15 分钟)

### 题 4.1: 上下翻屏

- 按 `C-v`,向下翻一屏 (window 显示 buffer 下半部分)
- 按 `M-v`,向上翻一屏

### 题 4.2: 翻页 (page)

如果你的文本有 `\f` (form feed,即 `C-q C-l`):

```
C-x ]    forward-page
C-x [    backward-page
```

实际场景: 在某些格式 (nroff, texinfo) 里 `\f` 分页。

### 题 4.3: 单行 scroll

```
M-1 C-v    向下滚 1 行
M-- C-v    向上滚 1 行
```

(数字前缀影响 `scroll-up-command` 的行数)

### 题 4.4: scroll 其他 window

如果你有两个 window (split 了):

```
C-M-v     scroll-other-window (向下)
C-M-S-v   scroll-other-window (向上)
```

光标留在当前 window,但另一个 window 滚动。**神技**: 看文档时翻代码不离开当前位置。

### 题 4.5: pixel scroll (Emacs 29+)

```
(pixel-scroll-precision-mode 1)
```

启用后,鼠标滚轮平滑滚动 (像素级)。

加到 init.el:
```elisp
(pixel-scroll-precision-mode 1)
```

---

## 6. 高级跳转 (5 题,30 分钟)

### 题 5.1: avy 跳转 (如果你装了 avy)

如果没装,跳到下一题。

```
M-x package-install RET avy RET
```

加到 init.el:
```elisp
(global-set-key (kbd "C-:") 'avy-goto-char)
(global-set-key (kbd "C-'") 'avy-goto-char-2)
(global-set-key (kbd "M-g f") 'avy-goto-line)
(global-set-key (kbd "M-g w") 'avy-goto-word-1)
```

试用:
- `C-:` 然后输入一个字符,屏幕上所有该字符位置出现字母提示
- 输入提示字母,光标瞬移过去

**avy 是 Geek 必装**。Module 4 之后推荐用。

### 题 5.2: xref (跳定义)

打开任何代码文件:
```
M-x find-file RET ~/some-code.py RET
M-.    (xref-find-definitions) 跳到光标处符号的定义
M-?    (xref-find-references) 查找所有引用
M-,    (xref-pop-marker-stack) 跳回
```

xref 是 Emacs 内置的"跳定义"框架,Module 5 会学。

### 题 5.3: 项目内搜索

```
M-x project-find-file     ; 在当前项目内找文件
M-x project-find-regexp   ; 在当前项目内搜文本
M-x project-switch-to-buffer  ; 项目内切 buffer
```

(project.el 是 Emacs 内置,Module 5 深入)

### 题 5.4: 最近文件

```
M-x recentf-open-files
```

显示最近打开的文件列表。

(你 init.el 里 `(recentf-mode 1)` 开了的话)

### 题 5.5: 注册 jump 历史

多次设 register:
```
C-x r SPC a     (位置 1)
... 移动 ...
C-x r SPC b     (位置 2)
... 移动 ...
C-x r SPC c     (位置 3)
```

跳来跳去:
```
C-x r j a
C-x r j b
C-x r j c
```

---

## 7. 综合练习 (5 题,30 分钟)

### 题 6.1: 速度测试

打开一个长文件 (~5000 行,可以拷贝某个 elisp 源码)。

任务:
1. 从 buffer 头,在 5 秒内到第 5000 行
2. 从第 5000 行,在 3 秒内跳到第一个 "defun"
3. 在 5 秒内跳到 buffer 中间

记录你的时间: ___ 秒

### 题 6.2: 不看键盘

合上书/教程,在 buffer 里完成:
1. 跳到行首 (`C-a`)
2. 跳到行尾 (`C-e`)
3. 向前一词 (`M-f`)
4. 向后一词 (`M-b`)
5. 下一行 (`C-n`)
6. 上一行 (`C-p`)
7. 跳到句首 (`M-a`)
8. 跳到段首 (`M-{`)
9. 跳到 buffer 头 (`M-<`)
10. 跳到 buffer 尾 (`M->`)

如果有一半卡住,重做 1-10 题。

### 题 6.3: 模拟编辑工作流

任务: 你在写代码,需要:
1. 跳到第 50 行 (`M-g g 50 RET` 或 `M-g M-g 50 RET`)
2. 跳到函数 `foo` 定义 (`M-x imenu RET foo RET`)
3. 看一下文件头 (`M-<`)
4. 跳回 foo (用 register 或 `M-,`)
5. 跳到匹配的 `}` (`C-M-n`)

记住每个键,完成全流程。

### 题 6.4: 滚动配合

两个 window (用 `C-x 3` 分屏):
- 左 window 看文档
- 右 window 看代码

任务: 光标留在右 window 写代码,但**滚动左 window** 看不同文档:
- `C-M-v` 左 window 向下滚
- `C-M-S-v` 左 window 向上滚

### 题 6.5: Kmacro 配合移动

录一个宏:
- 找下一个空行 (`C-s RET RET`)
- 在空行插入 `// empty`
- 移到下一行

```
F3
C-s RET RET
// empty
C-n
F4
```

运行 10 次: `C-u 10 C-x e`

---

## 8. 移动的"第一性原理"总结

### 8.1 选择移动单位的决策树

移动的本质是**减少认知负担**: 你脑子里有"要去哪",Emacs 提供命令让你"去那里"。命令越多越精细,但你不知道用哪个就越迷茫。下面这棵决策树是经验总结:

```
要移动到哪儿?
├── 几个字符:        C-f / C-b (+ prefix)
├── 一个词:          M-f / M-b
├── 一行内:          C-a (行首), C-e (行尾)
├── 上下几行:        C-p / C-n (+ prefix)
├── 一句话:          M-a / M-e
├── 一段:            M-{ / M-}
├── 函数定义:        C-M-a / C-M-e (lisp), imenu (通用)
├── 配对括号:        C-M-n / C-M-p / C-M-f / C-M-b
├── Buffer 头/尾:    M-< / M->
├── 指定行号:        M-g g
├── 指定字符:        C-s / C-r (isearch), avy (装了的话)
├── 指定符号:        M-. (xref), imenu
└── 跨 buffer:       C-x b, C-x r j (register), bookmark-jump
```

这棵树的优先级: **永远选最大可能的单位**。如果你要跳到 50 行后的某处,不要 `C-n 50`,用 `M-g g` 或 isearch。如果你要跳到当前函数末尾,不要 `C-n N`,用 `C-M-e`。每次你"按很多次同样的键",Emacs 都有更快的替代——找到它。

### 8.2 速度优化

速度优化本质是"找更大的单位"。下面是几个具体的反模式和正模式:

- **不要一次按很多 `C-f`**,用 `M-f` 或 `C-s` 跳
- **不要用鼠标滚轮**,用 `C-v` / `M-v`
- **不要 `C-n` 50 次**,用 `M-g g 50`
- **不要 scroll 找东西**,用 `C-s` (isearch) 或 `M-s o` (occur)

创造性思维: 任何"找东西"的操作,都应该用**搜索**而不是**滚动**。你的眼睛扫 1000 行找某个词需要 30 秒,isearch 1 秒。任何"跳到某行"的操作,用 `M-g g` 而不是 `C-n N`。任何"跳到下一个 X"的操作,用 isearch + `C-s` 而不是手动找。

### 8.3 常见反模式

- 鼠标点击移动 → 用键盘。鼠标要离开 home row,慢且打断流。
- 用方向键很远 → 用单位更大的命令。`C-f` 50 次比 `M-f` 10 次慢 5 倍。
- 找东西用 scroll → 用 isearch/occur/imenu。眼睛扫比搜索慢一个数量级。
- 用 pageup/pagedown → 用 C-v/M-v (键盘近,且方向键在 home row 之外)。

### 8.4 创造性应用

学完移动命令,试试这些"高级用法":

1. **Register 跳转做"代码地标"**: 在长文件的 5 个关键位置设 register (`C-x r SPC a` 到 `C-x r SPC e`),用 `C-x r j a` 到 `C-x r j e` 秒跳。比 isearch 快。
2. **`C-s C-w` 复制词到搜索**: 光标在一个词上,`C-s C-w` 立刻搜这个词的下一个出现。比手敲词快。
3. **`avy` 跳任意可见字符**: 装 avy 后,`C-:` 然后输入任意字符,屏幕上所有该字符位置出现字母提示,输入提示字母光标瞬移。这是"看见即到达"。
4. **`C-u C-SPC` 走 mark 历史**: 你设过的每个 mark 都在 ring 里,`C-u C-SPC` 反复按可以回到所有最近工作过的位置。比书签快。
5. **`M-g g` 配合 `imenu`**: 先 `M-x imenu` 跳到某函数,在那里 `C-x r SPC m` 存 mark,然后继续工作。需要回来时 `C-u C-SPC` 几次就回到。

---

## 9. 自测 (10 题)

回答 (合上教程):

1. `C-p` 跑哪个命令? ____
2. 跳到行首? ____
3. 跳到 buffer 尾? ____
4. 跳到下一个词? ____
5. 跳到上一段? ____
6. 跳到匹配的 `}`? ____
7. 跳到第 100 行? ____
8. 滚动其他 window (向下)? ____
9. 跳到当前函数定义 (lisp)? ____
10. 跳到当前 buffer 中某函数定义 (通用)? ____

**答案** (反白):
> 1. previous-line
> 2. C-a (move-beginning-of-line)
> 3. M-> (end-of-buffer)
> 4. M-f (forward-word)
> 5. M-{ (backward-paragraph)
> 6. C-M-n (forward-list)
> 7. M-g g (goto-line) 或 M-g M-g
> 8. C-M-v (scroll-other-window)
> 9. C-M-a (beginning-of-defun)
> 10. M-x imenu

---

## 10. 毕业检查

- [ ] 题 6.2 全部不卡顿完成
- [ ] 题 6.1 速度测试 < 20 秒
- [ ] 自测 10 题 ≥ 8 题答对
- [ ] 知道何时用 `C-f` 何时用 `M-f` 何时用 `C-s`

完成后:
- 在 `logs/module-01.md` 记 "Movement 用时 X 小时,熟练度 ★★★★☆"
- 进入 `drills/02-kill-ring.md`
