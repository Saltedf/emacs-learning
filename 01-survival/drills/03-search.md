# Drill 03: Search & Replace — 找到任何东西,改任何东西

> **预计用时**: 2 小时
> **难度**: ★★★☆☆ (正则表达式部分)
> **替代**: `emacs-manual/search.texi` 全文
> **目标**: 5 秒内找到任意文本;10 秒内批量替换

---

## 0. 学习目标

- [ ] 用 isearch 在 3 秒内跳到任意位置
- [ ] 用 `M-s o` (occur) 列出所有匹配
- [ ] 用 `M-%` 做交互式替换
- [ ] 用 `C-M-s` 跑正则搜索
- [ ] 写一个简单的 regexp 匹配邮箱/URL

---

## 1. 第一性原理: 为什么是"递增"搜索?

### 1.1 历史背景

搜索是编辑器的灵魂。1970s 的搜索是**批量**的: 输入完整 query,ENTER,等结果。这是 Unix 工具 `grep` 的模型——它适合"找出所有匹配",但不适合"跳到匹配"。

1980s Stallman 在 Emacs 引入 **incremental search** (递增搜索):
- 你按一个字符,立刻跳到第一个匹配
- 再按一个,缩小范围
- 反馈循环 < 100ms

这在当时是革命性的——大多数编辑器要你输完 query 按 ENTER 才搜索。直到 1990s 其他编辑器才学会。`C-s` 是这个革命的核心键,backward 用 `C-r`。如果你要批量替换,`M-%` 进 query-replace,每个匹配问 y/n。

**第一性原理**: 反馈循环越短,认知负担越低。普通搜索要你在脑子里构建完整 query 再按确认,然后等结果。递增搜索让你"边想边搜",每按一个键都是反馈: "对了,继续",或者"错了,退一格"。这种循环让搜索变成**对话**,不是命令。

为什么这这么重要? 因为大多数"找东西"的操作不是一开始就知道完整目标。你记得有个函数叫 `something-with-foo`,但不记得全名。普通搜索要你猜完整名字再按 ENTER,猜错重来。递增搜索让你输入 `foo`,看到所有含 foo 的位置,继续输入 `with` 缩小范围,直到找到目标。

### 1.2 Emacs 搜索家族

Emacs 提供一整套搜索工具,各有适用场景:

| 工具 | 用途 | 速度 |
|---|---|---|
| Isearch (`C-s`/`C-r`) | 单 buffer,递增 | 极快 |
| Occur (`M-s o`) | 单 buffer,列所有匹配 | 快 |
| Replace (`M-%`/`C-M-%`) | 替换 | 中 |
| Imenu (`M-s i`) | 跳定义 | 快 |
| grep (`M-x grep`) | 多文件 | 慢 |
| rgrep (`M-x rgrep`) | 多文件递归 | 中 |
| project-find-regexp | 项目内 | 中 |
| xref-find-references | 跨文件,语义 | 慢 |

**选择决策树**:
- 在当前 buffer 找一个词跳过去 → isearch
- 在当前 buffer 列出所有匹配 → occur
- 替换某些文本 → query-replace
- 跳到某个函数/类定义 → imenu
- 跨文件找文本 → grep / rgrep / project-find-regexp
- 找符号的所有引用 (重构) → xref-find-references

理解每个工具的"舒适区",在合适场景选合适工具,是 Emacs 用户的必备技能。

---

## 2. Isearch (递增搜索) 内联讲解

### 2.1 基本流程

按 `C-s` 进入 isearch mode (向后搜索):

```
Isearch: []
```

每输入一个字符:
- 实时跳到第一个匹配的位置
- 高亮所有匹配 (`lazy-highlight`)

```
输入 f:    Isearch: f     ← 跳到第一个 'f'
输入 fo:   Isearch: fo    ← 跳到第一个 'fo'
输入 foo:  Isearch: foo   ← 跳到第一个 'foo'
```

按 `RET` 退出,光标停在当前匹配。
按 `C-g` 取消,光标回到搜索前的位置。

**这里有个微妙之处**: 按 `RET` 和按 `C-g` 行为不同。`RET` 是"我找到了,留在这里",光标停在当前匹配。`C-g` 是"我改变主意,回到开始",光标回到按 `C-s` 之前的位置。新手经常误按 `C-g`,失去搜索结果。

**底层机制**: isearch 在开始时记录 point,每次按字符都更新 point 到匹配。按 `RET` 退出时保留当前 point。按 `C-g` 退出时恢复原始 point。这两个退出方式各有用途,理解它们能避免困惑。

### 2.2 在 isearch 里

isearch 是一个**特殊模式**,有自己的 keymap。进入 isearch 后,普通键不再插入文本,而是触发 isearch 命令。

| 键 | 作用 |
|---|---|
| `C-s` | 下一个匹配 |
| `C-r` | 上一个匹配 |
| `RET` | 退出,留在当前 |
| `C-g` | 取消,回原位 |
| `C-w` | 把光标处的 word 加入搜索词 |
| `C-y` | 把光标处到行尾加入搜索词 |
| `M-y` | 从 kill-ring 粘贴到搜索词 |
| `M-c` | toggle case-sensitive |
| `M-r` | toggle regexp |
| `M-s w` | toggle word search |
| `M-s s` | toggle symbol search |
| `M-s o` | 从 isearch 进入 occur |
| `M-%` | 从 isearch 进入 query-replace |
| `C-M-w` | 删搜索词最后一个字符 |
| `C-M-y` | 把 char 加入搜索词 |
| `M-n` / `M-p` | search history (上/下一条) |
| `M-tab` | completion (在搜索词上) |

**最有用的几个**:
- `C-w`: 取光标处的词加进搜索词。如果你搜的词就在屏幕上,不要手敲,`C-s C-w` 一步到位。
- `M-y`: 从 kill-ring 粘到搜索词。如果你刚复制了一段长文本想搜它,`C-s M-y` 比手敲快。
- `M-s o`: 从 isearch 跳到 occur。你边搜边发现匹配太多,想看列表,`M-s o` 切换。
- `M-%`: 从 isearch 进 query-replace。你搜到一个词想批量替换,`M-%` 直接进入替换模式,以当前搜索词为 query。

### 2.3 重复搜索

`C-s C-s` (连按两次) 重复**上次**搜索 (从 search-ring 取)。
`C-r C-r` 同上,向前方向。

为什么要 `C-s C-s`? 因为大多数时候你重复搜索的是上次的词 (比如搜所有 `defun`,看完一个看下一个)。`C-s C-s` 让你不重新输入词,直接用上次的。这比 `M-p` 翻历史再 `RET` 快。

### 2.4 Word search

```
M-s w RET word RET   搜 "word" 整词 (不匹配 "keyword")
```

或 `C-s` 进入后按 `M-s w` 切换。

**为什么要 word search?** 普通 isearch 是子串匹配——搜 `foo` 会匹配 `foobar`、`afoobar`、`foo_`。但有时候你只想找独立的 `foo` 这个词。word search 把 `foo` 当作整词搜,只在词边界处匹配。

具体例子: 你在一个长 buffer 里搜 `is`,普通搜索会匹配 `this`、`his`、`exist` 等所有含 `is` 的地方。word search 只匹配独立的 `is`。这在阅读文档时很有用——你不想看每个含 `is` 的词。

### 2.5 Symbol search

```
M-s _ RET symbol RET   搜 symbol (受 syntax table 影响,通常 = word)
```

编程时,`M-s .` 搜光标处的 symbol。

**symbol 和 word 的区别**: 在大多数编程语言 mode 里,symbol 比 word 宽松——`foo-bar` 是一个 symbol (因为 `-` 是 symbol 字符),但是两个 word。所以编程时用 symbol search 更准确。

具体例子: 在 lisp-mode,你想搜 `my-function`,用 word search 会搜到 `my` 和 `function` 两个词。用 symbol search,`my-function` 是一个 symbol,直接匹配。

### 2.6 Case sensitivity

`case-fold-search` (默认 t):
- t: 大小写不敏感 (`foo` 匹配 `Foo`, `FOO`)
- nil: 严格

isearch 里 `M-c` 临时切换。

聪明的规则: 如果你的 query 全小写,大小写不敏感;有大写,敏感。
设 `(setq search-upper-case 'auto)` (默认值)。

**为什么要这个自动规则?** 因为你搜 `foo` 时,大多数时候你不在乎大小写 (找 foo/Foo/FOO 都行)。但搜 `Foo` 时,你**通常在乎** (你要的是大写 F 开头的)。Emacs 根据你的输入猜测意图,大多数时候猜对。

具体例子: 搜 `defun` 你想匹配 `defun`、`DEFUN` (常量名);搜 `MyClass` 你只想匹配 `MyClass` 不想匹配 `myclass`。这个规则让两种场景都满足。

---

## 3. 正则搜索 (`C-M-s`)

### 3.1 Elisp 正则语法

Emacs 正则和 POSIX/Perl 不完全一样。常见元字符:

| 元字符 | 含义 |
|---|---|
| `.` | 任意字符 (不含换行) |
| `*` | 0 或多个 |
| `+` | 1 或多个 (需 `(require 'cl-lib)` 前); 在 Emacs 24+ 默认支持 |
| `?` | 0 或 1 个 |
| `^` `$` | 行首 / 行尾 |
| `\b` | 词边界 |
| `\(...\)` | 分组 (括号要 escape!) |
| `\|` | 或 |
| `[abc]` | 字符类 |
| `[^abc]` | 取反 |
| `\d` | 数字 (有些版本) |
| `\s.` | syntax class (例 `\s(` 匹配开括号) |
| `\S.` | 非 syntax class |
| `\w` | word 字符 |
| `\W` | 非 word |
| `\<` `\>` | 词首/词尾 |

**注意: Emacs 26+ 也支持 `rx` 宏** (s-expression 风格的正则,可读性强):

```elisp
(rx (one-or-more digit) "." (one-or-more digit))
;; 等价于 "[0-9]+\\.[0-9]+"
```

Module 6 会详细学 `rx`。

### 3.2 实战正则

例 1: 找所有 email:

```
[\w.]+@[\w.]+
```

(简化版,不严格 RFC)

例 2: 找所有 URL:

```
https?://[\w./?-]+
```

例 3: 找所有 hex 颜色:

```
#[0-9a-fA-F]\{6\}
```

(注意 `{n}` 在 Emacs 正则要 escape!)

### 3.3 isearch 正则

`C-M-s` 进入正则向后搜。
`C-M-r` 向前。
或 `C-s` 后 `M-r` 切换。

### 3.4 例子

按 `C-M-s`,输入 `defun\s-+\(\w+\)`,找下一个 `defun foo` 形式。

---

## 4. Occur (`M-s o`)

### 4.1 基本用法

`M-s o` 跑 occur,把所有匹配显示在 `*Occur*` buffer:

```
15 matches for "function" in buffer: foo.js
   12:function foo(a, b) {
   18:function bar(c) {
   25:function baz() {
   ...
```

- 数字是行号
- `n`/`p` 上下移
- `RET` 或鼠标点击跳到原文
- `o` 在其他 window 显示
- `g` 刷新

### 4.2 Occur 编辑

`e` 进入 `occur-edit-mode`,可以直接改 `*Occur*` buffer 里的文本!
改完 `C-c C-c` 应用到原文件。

**实战**: 用正则搜出 50 处需要改的文本,在 occur 里批量改,一次应用。

### 4.3 Multi-occur

```
M-x multi-occur RET (选 buffers) RET pattern RET
M-x multi-occur-in-matching-buffers RET pattern RET
```

跨多个 buffer 搜索。

---

## 5. Replace

### 5.1 `replace-string` (无问)

```
M-x replace-string RET foo RET bar RET
```

把 buffer 里所有 "foo" 改成 "bar",不问。

**警告**: 替换完无法 region 限制,无法 undo (除了 buffer-undo-list)。新手用 `M-%`。

### 5.2 `query-replace` (`M-%`)

```
M-% foo RET bar RET
```

进入交互模式,每个匹配问:

```
Query replacing foo with bar: (? for help)
```

| 键 | 作用 |
|---|---|
| `y` / `SPC` | 替换当前,下一个 |
| `n` / `DEL` | 不替换,下一个 |
| `!` | 替换剩下所有 (不问) |
| `q` / `RET` | 退出 |
| `.` | 替换当前,然后退出 |
| `,` | 替换当前,但不前进 (再看一眼) |
| `^` | 回到上一个匹配 |
| `u` | 撤销上次替换 |
| `U` | 撤销所有替换 |
| `E` | 编辑 replacement (用复杂 Elisp) |
| `C-r` | 递归编辑 (临时改点东西再回来) |
| `C-w` | 删除当前匹配,进入递归编辑 |
| `C-l` | 重定位 (让你看上下文) |
| `?` | 帮助 |

### 5.3 正则替换 (`C-M-%`)

`C-M-% OLD RET NEW RET`

NEW 里可以用 `\1`、`\2` 引用 OLD 里的分组:

```
OLD: \(foo\)\(bar\)
NEW: \2\1
```

把 `foobar` 替换为 `barfoo`。

**Elisp 在 replacement 里**:
`C-M-% \(foo\) RET \,(concat "X" \1 "Y") RET`

把 `foo` 替换为 `XfooY`。`\,(...)` 在 replacement 里跑 Elisp!

### 5.4 Region 限制

选中 region 后 `M-%`,只在该 region 内替换。

### 5.5 Project-wide replace

用 `M-x project-find-regexp` 找,然后在 occur 里改。

或用 `wgrep` 包 (Module 5)。

### 5.6 搜索的创造性应用

学完基本搜索,试试这些高级用法:

1. **`C-s C-w` 取词搜**: 光标在一个词上,`C-s C-w` 立刻搜这个词的下一个出现。再 `C-s` 下一个。这比手敲词快 5 倍。
2. **`C-s M-y` 粘贴搜**: 你刚复制了一段 (浏览器、shell、其他 buffer),想搜它,`C-s M-y` 直接从 kill-ring 取,不手敲。
3. **isearch + occur 切换**: 你 `C-s` 搜一个词,发现匹配很多,想看列表。`M-s o` 从 isearch 进 occur,query 自动继承。
4. **isearch + query-replace**: 你 `C-s` 搜到一个词,决定批量替换。`M-%` 从 isearch 进 query-replace,query 自动继承。
5. **`M-s h r` 高亮**: `M-s h r REGEX` 在 buffer 里高亮所有匹配 REGEX 的地方,持久高亮 (不退出)。适合"标记所有 X 让我能看到分布"。
6. **`M-s h u` 取消高亮**: 取消上面设的高亮。
7. **`\,(elisp)` 在 replacement**: `C-M-% \(word\) RET \,(upcase \1) RET` 把每个匹配的 word 转大写。`\,(...)` 在 replacement 里跑 Elisp,这是 Emacs 的"超级武器"——任何文本变换都能做。
8. **`align-regexp` 配合搜索**: 选中一段,`C-u M-x align-regexp RET = RET` 把所有 `=` 对齐。底层就是用正则找 `=` 然后插空格。

---

## 6. Imenu (`M-s i` 或 `M-x imenu`)

### 6.1 跳定义

```
M-x imenu RET foo RET
```

跳到 `foo` 函数/类/方法定义。

imenu 用 major mode 提供的 imenu-generic-function 或 treesit 解析。

### 6.2 imenu 列表

```
M-x imenu RET RET   ; 第二个 RET 不输入,显示列表
```

(列表取决于补全 UI; 默认 *Completions* buffer)

### 6.3 自动 rescan

```
(setq imenu-auto-rescan t)
```

文件改了,imenu 索引自动更新。

---

## 7. 跨文件搜索

### 7.1 grep

```
M-x grep RET grep -nH pattern *.el RET
```

结果在 `*grep*` buffer,可点击跳转。

### 7.2 rgrep

```
M-x rgrep RET pattern RET *.el RET ~/src RET
```

递归 grep。

### 7.3 find-grep

```
M-x find-grep RET find . -type f -name "*.el" -exec grep -nH pattern {} +
```

### 7.4 project-find-regexp

```
M-x project-find-regexp RET pattern RET
```

当前项目内搜索。比 grep 更智能 (识别 .gitignore 等)。

### 7.5 rg / ag (要装)

`ripgrep` 和 `the_silver_searcher` 是最快的:
```
M-x package-install RET wgrep RET
M-x package-install RET rg RET
```

然后 `M-x rg` 或 `M-x ripgrep-regexp`。

---

## 8. 微练习 (40 分钟,25 题)

### Isearch 基础 (5 题)

1. 打开一个长文件,`C-s function`,跳到第一个匹配
2. 再 `C-s`,下一个匹配
3. `C-r`,反向
4. `RET`,留在当前
5. `C-s C-s`,重复上次搜索

### Isearch 中级 (5 题)

6. `C-s` 然后 `C-w`,把光标处的 word 加进搜索词
7. `C-s` 然后 `M-p`,看搜索历史
8. `C-s foo` 然后 `M-c`,切换大小写敏感
9. `M-s w` 切换 word search,试 `function` 整词 (不匹配 `functions`)
10. `C-s` 然后 `M-s o`,进入 occur

### 正则 (5 题)

11. `C-M-s defun\s-+`,找下一个 `defun ` 形式
12. `C-M-s [0-9]+`,找数字
13. `C-M-s \(foo\|bar\)`,找 `foo` 或 `bar`
14. 用 `rx` 宏写一个正则匹配邮箱: `(rx (one-or-more (any "a-zA-Z0-9.")) "@" (one-or-more (any "a-zA-Z0-9.")))`
15. 在 `*scratch*` eval `(re-search-forward "[0-9]+" nil t)`,看返回值

### Occur (4 题)

16. `M-s o function RET`,列出所有 "function" 出现
17. 在 occur buffer 里,`n`/`p` 移动,`RET` 跳
18. `e` 进入 occur-edit,改一处,`C-c C-c` 应用到原文件
19. `M-x multi-occur`,选两个 buffer 搜索

### Replace (4 题)

20. `M-% foo RET bar RET`,在 region 内替换
21. 在 query-replace 里按 `!` 替换所有
22. `C-M-% \(foo\) RET \1\1 RET`,把 `foo` 变 `foofoo`
23. `C-M-% \(word\) RET \,(upcase \1) RET`,把 word 大写

### 跨文件 (2 题)

24. `M-x rgrep RET function RET *.el RET ~/src RET`
25. `M-x project-find-regexp RET TODO RET` (如果你在一个 git 项目里)

---

## 9. 实战练习 (30 分钟)

### 任务 1: 提取所有 TODO 注释

打开一个代码项目:
```
M-x rgrep RET TODO RET *.py RET ~/your-project RET
```

在 `*grep*` buffer 里:
- 逐个看 TODO
- 想删的,`C-k` 在 grep buffer 里删行 (不影响原文件)
- 重要的,`m` 标记,然后 `M-x occur-mode` 处理

### 任务 2: 批量改函数名

假设你把 `foo()` 重命名为 `bar()`:
```
M-x project-find-regexp RET \bfoo\( RET
```

在结果 buffer 里:
- `w` (wgrep 模式,要装 wgrep 包)
- 直接编辑,把 `foo` 改 `bar`
- `C-x C-s` 应用所有

### 任务 3: 写一个正则匹配 IP 地址

```elisp
;; IP 地址: 4 段数字,每段 0-255
;; 简化版: \(\(?:[0-9]\{1,3\}\.\)\{3\}[0-9]\{1,3\}\)

(rx (repeat 3 (group (repeat 1 3 digit)) ".")
    (group (repeat 1 3 digit)))
```

在 buffer 里 `C-M-s` 然后输入这个 rx 表达式 (要先 `(require 'rx)` 或在 Elisp 里用)。

实际上,`rx` 是宏,不能直接在 isearch 用。
正确方式:

```elisp
(re-search-forward (rx (repeat 3 (repeat 1 3 digit) ".") (repeat 1 3 digit)))
```

在 `*scratch*` eval。

---

## 10. 自测 (5 分钟)

1. `C-s` 和 `C-M-s` 区别?
2. `M-s o` 跑什么命令?
3. `M-%` 在 query 里按 `!` 做什么?
4. `M-y` 在 isearch 里做啥?
5. `\(...\)` 在 Emacs 正则里是什么? 为什么不是 `(...)`?

**答案**:
> 1. 字符串搜索 vs 正则搜索
> 2. occur
> 3. 替换剩余所有,不问
> 4. 从 kill-ring 粘到搜索词
> 5. 分组 (Emacs 正则要求括号 escape,普通 `()` 是字面括号)

---

## 11. 毕业检查

- [ ] 25 题至少完成 20
- [ ] 能用 `M-s o` 列所有匹配并编辑应用
- [ ] 能写一个正则匹配邮箱/IP/URL
- [ ] 知道 `\,(elisp)` 在 replacement 里的用法

完成后进入 `drills/04-mark-rectangle.md`。
