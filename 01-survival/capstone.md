# Capstone: Module 1 毕业项目

> **目标**: 在 30 分钟内,完全无鼠标,重构一段 2000 行代码
> **用时**: 1-2 小时
> **难度**: ★★★★☆

---

## 项目背景

你已经在 Module 1 完成了 8 个 drill,做了 185 道微练习。
现在到了综合验收的时刻。

**任务**: 拿一份真实的、~2000 行的代码文件,完成一系列重构任务,**全程不离开键盘**。

这不是为了快,是为了证明: **你已经脱离了鼠标**。

### 为什么是 2000 行

2000 行是一个有意义的规模。它大到足以包含各种编辑场景 (函数、类、注释、import、嵌套结构),小到你能在一个 session 内完成。如果你能在 30 分钟内重构 2000 行,你就能在一周内重构整个项目。

更重要的是: 2000 行逼你**用批量操作**。一行一行手动改 2000 行需要几小时,用 query-replace、kmacro、rectangle 才能 30 分钟完成。这个 capstone 测试的不是你的打字速度,是你**选择正确工具的能力**。

### 为什么 30 分钟

30 分钟是一个心理阈值。超过 30 分钟,你会开始疲劳、犯错、想用鼠标"快速搞定"。30 分钟内完成,说明你的肌肉记忆足够流畅,不需要停下来想键位。

如果你第一次超时,不要沮丧——回去重做相关 drill,然后再试。大多数学生需要 2-3 次尝试才能 30 分钟内完成。

---

## 准备

### 选择一份代码

找一个 ~2000 行的代码文件。建议:
- 你工作/学习中的某份代码
- 或一个 Emacs 自带的 .el 文件 (例如 `~/src/learning/emacs-learning/emacs-lispref-30.2/emacs-lispref-30.2/variables.texi` 或随便一个)
- 或一份开源项目代码

为了演示,我们假设你选了某个代码文件 `~/refactor-target.py`。

### 启用录像 (可选但推荐)

用 kmacro 录下你 30 分钟的所有操作:

```
M-x kmacro-start-macro-or-insert-counter RET
... 开始工作 ...
M-x kmacro-end-or-call-macro RET
```

录完用 `C-h l` (view-lossage) 看你的输入历史。

或者用屏幕录像软件录全屏。

回看时你能看到自己的肌肉记忆是否流畅。

---

## 任务清单 (30 分钟内完成)

### 任务组 A: 浏览与定位 (5 分钟)

1. 打开文件 `C-x C-f ~/refactor-target.py RET`
2. 跳到第 500 行: `M-g g 500 RET` (或 `M-g M-g`)
3. 跳到文件头: `M-<`
4. 跳到文件尾: `M->`
5. 用 isearch 找下一个 `def`: `C-s def RET`
6. 跳到 `class Foo` 的定义: `M-x imenu RET Foo RET` (或 `M-s i`)
7. 看当前函数: `M-x beginning-of-defun` (`C-M-a`),`M-x end-of-defun` (`C-M-e`)
8. 跳到匹配的括号: `C-M-n` / `C-M-p`
9. 用 sexp 移动: 在某行代码上,`C-M-f` 跳一个 sexp,`C-M-b` 跳回
10. 在嵌套括号里用 `C-M-u` 跳到外层 (至少 3 次,穿越层次)
11. 用 `M-x which-function-mode RET` 开启 (如未开),看 mode line 显示当前函数

### 任务组 B: 编辑 (10 分钟)

9. 把所有 `function` 改成 `func`: 
   - `M-% function RET func RET`
   - 按 `!` 替换剩余所有 (确认安全后)
10. 给所有 `// TODO` 加序号: 
    - 用 isearch 找第一个 TODO: `C-s TODO RET`
    - 设 mark: `C-SPC`
    - 找最后一个: `C-s TODO RET` 多次
    - `C-x r N` (rectangle-number-lines)
11. 把第 100-150 行的代码注释掉:
    - `M-g g 100 RET`
    - `C-SPC`
    - `M-g g 150 RET`
    - `M-;` (comment-region)
12. 在第 200 行后插入 5 行空行:
    - `M-g g 200 RET`
    - `C-o` 5 次 (open-line,不开新行)
    - 或 `C-e RET RET RET RET RET`
13. 把第 300-310 行的代码缩进:
    - `M-g g 300 RET`
    - `C-SPC`
    - `M-g g 310 RET`
    - `C-M-\` (indent-region)

### 任务组 C: 重构 (10 分钟)

14. 提取一段代码到新函数:
    - 选中第 500-520 行
    - `C-w` 杀掉
    - 移到文件某处
    - 写 `def new_func():` 然后 `RET`
    - `C-y` 粘贴
15. 把一段代码块整体右移 4 空格:
    - 选中 5 行
    - `C-x r t RET` (空字符串),然后... 不,正确方法:
    - 选中矩形 (每行前 0 列)
    - `C-x r t SPC SPC SPC SPC RET` (4 个空格)
16. 把 10 个相似的重命名:
    - 用 isearch + query-replace
    - 或 kmacro: `F3 C-s old-name M-% new-name RET RET F4`,然后 `C-u 10 C-x e`
17. **Sexp 操作 — 杀一个完整参数**:
    - 找到某个函数调用的参数列表 `(foo arg1 arg2)`
    - 光标移到 `arg2` 之前
    - `C-M-k` 杀掉 `arg2` (整个 sexp,结构完整)
    - `C-y` 召回到末尾,验证 kill-ring 内容
18. **Sexp 操作 — 选中和复制 sexp**:
    - 光标在某个嵌套表达式之前
    - `C-M-SPC` 选中一个 sexp
    - `M-w` 复制
    - 移到文件末尾,`C-y` 粘贴验证
19. **Sexp 操作 — 交换两个 sexp**:
    - 找到 `(let ((a 1) (b 2)) ...)` 或类似的成对形式
    - 光标在第一个 binding 前
    - `C-M-t` 交换两个 binding
    - 检查顺序是否反了
20. **Hideshow 折叠验证 API surface**:
    - `M-x hs-minor-mode RET` (如未开)
    - `C-c @ C-M-h` 折叠所有函数体
    - 浏览剩下的签名,确认能一眼看完整个模块的 API
    - `C-c @ C-M-s` 展开所有

### 任务组 D: 信息提取 (5 分钟)

21. 提取所有函数名到 buffer 末尾:
    - 用 kmacro:
      ```
      F3
      C-s def RET
      C-s SPC RET       ; def 后的空格
      M-f                ; 复制函数名
      M-w                ; 复制 (需要先 C-SPC)
      ...
      F4
      ```
      (实际操作可能需要多次调整,这是练习)
    - 或用 `imenu` 列表 + 手动复制
    - 或用 `hs-hide-all` 折叠后,看签名列表
22. 用 occur 列所有 `TODO`:
    - `M-s o TODO RET`
    - 在 `*Occur*` buffer 里看
    - `e` 进入 occur-edit,加优先级 (高/中/低)
    - `C-c C-c` 应用

### 任务组 E: 验收 (5 分钟)

23. 保存: `C-x C-s`
24. 退出 frame 但不杀 Emacs: `C-x 5 0` (daemon 模式)
25. 或保存退出: `C-x C-c`
26. 用 `git diff` 看你的修改
27. 反思: 哪些操作用了鼠标? (应该 0 个)

---

## 评分

### 时间分

| 时间 | 分数 |
|---|---|
| < 20 分钟 | 10/10 (你已经超越 Module 1 水平) |
| 20-30 分钟 | 8/10 |
| 30-45 分钟 | 6/10 |
| 45-60 分钟 | 4/10 |
| > 60 分钟 | 2/10 (重做几个 drill) |

### 准确度分

- 完成所有 27 个任务: +10
- 漏 1-3 个: +7
- 漏 4-7 个: +5
- 漏 > 7: +2

### 无鼠标分

- 0 次鼠标: +5 ★
- 1-3 次: +3
- 4-10 次: +1
- > 10 次: 0

### 总分

满分 25。20+ 合格毕业。

---

## 反思: 看你的录像

回看你的录像或 lossage,问自己:

### Q1: 哪些操作慢?

时间花在哪了?
- 想键位? → 重做相关 drill
- 等待响应? → 是不是开了太多 minor mode
- 找东西? → 用 isearch / imenu / occur 更快

### Q2: 哪些操作用了鼠标?

为什么用鼠标?
- 不知道键位? → `C-h` 查
- 习惯? → 强制改
- 真的没有键位? → 用 Elisp 自己写一个

### Q3: 哪些操作可以批量?

- 你是不是一个一个改? → query-replace
- 你是不是手动选 region? → 用 mark-paragraph / mark-defun
- 你是不是手动跳? → 用 imenu / register

### Q4: 你的瓶颈是什么?

如果让你再做一次类似的 2000 行重构,你会怎么提速?

## 反思的反思: 元认知

这个 capstone 的真正价值不是"完成 19 个任务",而是**让你看到自己的工作模式**。回看录像时,你会惊讶地发现:
- 你用了多少次鼠标 (比你以为的多)
- 你在哪些操作上卡住 (通常是没形成肌肉记忆的键位)
- 你用了哪些低效路径 (比如手动选 region 而不是 mark-defun)

这种**自我观察**是进步的关键。优秀的程序员不是"从不犯错",而是"能看到自己的错误并改正"。capstone + 录像回看是这个自我观察过程的工具。

具体建议: 录完 30 分钟,先不要急着改。看完整个录像,列出"我用了鼠标的次数"和"我卡住的键位"。这两个数字是你下一步训练的靶子。如果鼠标用了 5 次,下次目标 0 次;如果卡在 `C-M-s` (正则搜索),回去重做 search drill 的正则部分。

---

## 高级挑战 (可选)

如果你觉得上面的任务太简单,试试这些:

### 挑战 1: 双屏协作

分屏成 4 个 window,同时打开:
- 左上: 主代码文件
- 右上: 测试文件
- 左下: 文档
- 右下: shell

完成所有上述任务,但同时**用 `C-M-v` 滚动非当前 window** 看参考材料。

### 挑战 2: 不看 minibuffer

把 echo area 遮住 (用 `(setq echo-keystrokes 0)`)。

完成所有任务,不看你按的键的反馈。

### 挑战 3: 不用 query-replace

完成所有任务,但不允许用 `M-%` / `C-M-%`。
只能用 kmacro。

(这会逼你练 kmacro)

---

## 写日志

完成后,打开 `logs/module-01.md`:

```markdown
# Module 1 学习日志

**日期**: 2026-XX-XX
**用时**: ___ 小时
**完成度**: ___%
**Capstone 用时**: ___ 分钟
**Capstone 评分**: ___/25

## 我的 7 个 drill 心得

### 01 Movement
[用时 / 难点 / 收获]

### 02 Kill Ring
[用时 / 难点 / 收获]

...

## Capstone 反思

- 时间花在哪了?
- 用了几次鼠标?
- 哪些操作可以更快?

## 我学到的最重要 5 件事

1.
2.
3.
4.
5.

## 我还要加强的

- [ ]
- [ ]

## 下一步

进入 Module 2: Dired + Buffer/Window 管理
```

---

## 毕业仪式

如果你完成了 Module 1:

**恭喜**。

你已经从"Emacs 当记事本"升级到"Emacs 用户"。
你能在任何 buffer 里飞快地移动、选、删、改。
你能用 help system 自我学习。
你不再需要鼠标。

这是巨大的进步。

但你的旅程才刚开始。

Module 2-8 会带你:
- 把 Emacs 变成文件管理器 (Module 2)
- 真正学会 Elisp (Module 3, 6)
- 把配置变美 (Module 4)
- 用 Org/Magit 工作 (Module 5)
- 写自己的包 (Module 7)
- 给开源贡献 (Module 8)

**继续。**

更新 `PROGRESS.md`: Module 1 状态 ✅。

进入 `02-dired/README.md`。
