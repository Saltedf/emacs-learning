# Capstone: Module 2 毕业项目

> **目标**: 10 分钟内用 dired 整理一个混乱目录
> **用时**: 1 小时
> **难度**: ★★★☆☆

---

## 项目概述

你已经学完了 Module 2 的所有 drills。现在到了验收的时刻。

**任务**: 找一个真实的混乱目录 (~50 个文件),在 10 分钟内用 dired + wdired 把它整理成结构化目录。

Capstone 不是"考你记不记得键位",而是"考你能不能把概念串起来解决真实问题"。背键位是手段,解决问题是目的。在做的过程中,关注三件事:

1. **流程设计**: 你先做什么后做什么? 为什么这个顺序?
2. **工具选择**: 这个操作用 Dired、WDired 还是 ibuffer? 为什么?
3. **错误恢复**: 操作错了怎么办? 安全网是什么?

10 分钟的时限不是"逼你快",而是"逼你流畅"。如果你要停下来想键位,说明还没真正掌握——回去重做 drills。

---

## 准备

### 选择目录

Capstone 用真实目录最有意义。建议:

- `~/Downloads` (下载目录,通常很乱)
- 你的工作项目的某个杂乱子目录
- 或者用这个命令造一个测试目录:

```bash
mkdir -p ~/cleanup-capstone
cd ~/cleanup-capstone
# 造 50 个杂乱文件
for i in {1..15}; do echo "data $i" > "file$i.txt"; done
for i in {1..10}; do echo "img $i" > "IMG_$i.jpg"; done
for i in {1..8}; do echo "doc $i" > "document_$i.pdf"; done
for i in {1..5}; do echo "code $i" > "script$i.py"; done
for i in {1..3}; do echo "old $i" > "backup$i.bak"; done
for i in {1..3}; do echo "log $i" > "log_$i.log"; done
echo "music" > "song1.mp3"
echo "music" > "song2.mp3"
echo "archive" > "data.zip"
echo "temp" > "temp.tmp"
echo "readme" > "README.md"
```

这个测试目录刻意模拟真实 Downloads 的混乱——多类型、无结构、有垃圾。练习时它给你一个"安全试验场"——错了就 `rm -rf ~/cleanup-capstone` 重来。

**真实目录更难**,因为它有你不想丢的文件。Capstone 第一次用测试目录,熟练后再用真实 Downloads。

### 启用录像 (可选)

用 `M-x kmacro-start-macro` 录,或屏幕录像。

录像让你**回看自己的操作**——哪里慢了,哪里犹豫了,哪里手抖了。这是改进的最快方式。专业运动员看自己的录像,Emacs 极客也应该看自己的操作录像。

---

## 任务清单 (10 分钟内完成)

10 分钟拆成 5 个阶段: 观察、规划、执行、精修、验证。每个阶段有明确产出。

### Step 1: 观察现状 (1 分钟)

打开 dired:
```
C-x d ~/cleanup-capstone RET
```

`M-<` 到顶,浏览所有文件。

观察阶段的关键是"识别模式"——不是逐个文件看,而是看整体结构。这个目录里有哪些类型? 每类有多少? 有什么明显的垃圾?

识别分类:
- 图片: .jpg
- 文档: .pdf
- 代码: .py
- 备份: .bak
- 日志: .log
- 音乐: .mp3
- 压缩: .zip
- 临时: .tmp
- 文本: .txt
- Markdown: .md

**思考**: 这些类型需要不同的处理吗? .bak 和 .tmp 是垃圾 (删),其他要归类 (移到子目录)。.md 通常作为 README 留根目录。这种"分类决策"是 Capstone 的核心思维——不是机械操作,而是有判断。

### Step 2: 创建目标目录 (1 分钟)

按 `+` 创建子目录:

```
+ Pictures RET
+ Documents RET
+ Code RET
+ Logs RET
+ Music RET
+ Archives RET
+ Text RET
```

这一步"搭骨架"——后续归类就是把文件填进对应目录。先建好骨架,后面操作流畅。

`+` (dired-create-directory) 一次创建一个目录。你也可以批量创建 (用 shell 命令),但 `+` 直观。**这是"规划"阶段**——你知道最终结构,现在把它建出来。

### Step 3: 批量归类 (5 分钟)

按扩展名标记 + 移动。这是 Capstone 的主体——5 分钟处理 50 个文件,平均每个文件 6 秒。GUI 做不到这个速度。

下面每段是"一类文件"的处理。模式相同: 取消旧标记 → 标记新类型 → 移动。

**图片 (.jpg)**:
```
%m \.jpg$ RET     标记
R                  rename (这里用作 move)
输入: Pictures/ RET
```

或用 `C` (copy) 然后 `D` (delete)。但 `R` 直接移动更高效。

`R` 当目标是目录时,会把所有标记文件移过去,保持原名。这是 Dired 的"批量移动"模式——一个 `R` 命令处理多个文件。

**文档 (.pdf)**:
```
U                  先取消所有标记
%m \.pdf$ RET
R
Documents/ RET
```

注意每段开头 `U` 取消所有标记——避免上一类的标记残留。这是"操作纪律": 每个独立操作前先清空状态。

**代码 (.py)**:
```
U
%m \.py$ RET
R
Code/ RET
```

**备份 (.bak)**:
```
U
%m \.bak$ RET
D                  标记删除
x                  执行
y y y              确认每个 (或设 dired-deletion-confirms nil 跳过)
```

备份是垃圾——直接 `D` 删,不归类。注意 `D` 是标记,`x` 才执行。

**日志 (.log)**:
```
U
%m \.log$ RET
R
Logs/ RET
```

**音乐 (.mp3)**:
```
U
%m \.mp3$ RET
R
Music/ RET
```

**压缩 (.zip)**:
```
U
%m \.zip$ RET
R
Archives/ RET
```

**临时 (.tmp)**:
```
U
%m \.tmp$ RET
D
x
y
```

临时文件也是垃圾——直接删。

**文本 (.txt)**:
```
U
%m \.txt$ RET
R
Text/ RET
```

**Markdown (.md)**:
- `README.md` 留在根目录

README 是"门面"——通常放根目录让人一眼看到。其他 .md 也可以单独建 Docs 目录,但 Capstone 里就一个 README,留根目录。

### Step 4: WDired 高级操作 (2 分钟)

假设你想把 `file1.txt` 到 `file15.txt` 改成 `note_01.txt` 到 `note_15.txt`:

```
C-x C-q             进入 wdired
```

在 wdired 里:

这一步是 Capstone 的"精修"——归类后,有些文件名还不够好。WDired 让你像编辑文本一样改名。

**方法 1: query-replace**

- 选中所有 file*.txt 行
- `M-% file RET note_ RET`
- 按 `!` 替换所有

简单的字符串替换,适合"统一改前缀"。

**方法 2: kmacro (加 0 补全)**

(复杂,跳过)

如果要给数字加 0 补全 (1 → 01),query-replace 做不到——数字是变量。这时用 kmacro: 录一个宏"找下一个 fileN,改成 note_0N",跑 N 次。

(复杂,跳过——Capstone 不要求做到这步。Module 3 学 Elisp 后能用代码做。)

**方法 3: 手动 + 矩形**

- 选中 file1.txt 到 file9.txt 的 "file" 部分 (矩形)
- `C-x r t note RET` 把 "file" 改成 "note"

矩形编辑批量改多行同一列的文本。适合"统一改前缀"。

提交:
```
C-c C-c
```

提交后 WDired 翻译 buffer 改动成 `rename-file` 调用。如果有错,`*WDired errors*` buffer 会显示。

### Step 5: 验证 (1 分钟)

退出 wdired 后,在 dired 里看结构:

```
~/cleanup-capstone/
  - README.md
  - Archives/
    - data.zip
  - Code/
    - script1.py
    - ...
  - Documents/
    - document_1.pdf
    - ...
  - Logs/
    - log_1.log
    - ...
  - Music/
    - song1.mp3
    - song2.mp3
  - Pictures/
    - IMG_1.jpg
    - ...
  - Text/
    - note_01.txt
    - ...
```

完美。

验证阶段不是"看一眼就过"——要逐项检查: (1) 所有文件归类正确; (2) 没有遗漏; (3) 子目录结构合理。这一步养成"操作后验证"的习惯,是专业 Emacs 用户的标志。

---

## 评分

### 时间分

| 时间 | 分数 |
|---|---|
| < 7 分钟 | 10/10 |
| 7-10 分钟 | 8/10 |
| 10-15 分钟 | 6/10 |
| 15-20 分钟 | 4/10 |
| > 20 分钟 | 2/10 (重做 drills) |

时间分反映"流畅度"——你能不假思索地用对工具。如果你想超过 3 秒才想起来"标记是什么键",说明还不够流畅。

### 准确度分

- 所有文件归类正确: +10
- 漏 1-3 个: +7
- 漏 4-7 个: +5
- 漏 > 7: +2

准确度反映"严谨度"——你能做到不遗漏。漏文件通常是"忘记取消标记"或"过滤条件错了"。

### 无鼠标分

- 0 次鼠标: +5
- 1-3 次: +3
- > 3 次: 0

无鼠标分反映"沉浸度"——你完全在键盘流里。如果中途切到鼠标,说明某个操作你不会键盘版,回去补。

### 总分

满分 25。20+ 合格。

如果没到 20,不要气馁——回去重做 drills 的对应组,然后再来一次。Capstone 是"测试",不是"判决"。

---

## 高级挑战

如果基础 Capstone 你 7 分钟内完成,试试这些挑战——它们逼你用更高级的技巧。

### 挑战 1: 双窗口对比

`C-x 3` 分屏:
- 左: 原始目录
- 右: 目标目录

用 `C` (copy) 把文件分类复制 (而不是 `R` 移动)。

要求: 用 `dired-dwim-target` 智能选择目标 window。

这个挑战逼你用"双 Dired 工作流"——`dired-dwim-target` 让 `C`/`R` 自动选另一 window 的目录作为目标。`C RET` 两步完成移动,不用手输路径。

这种工作流的真实场景: 你从 `~/Downloads` 复制到 `~/Pictures/2024/`,左右分屏,Dired 自动认路。

### 挑战 2: 递归子目录

如果原始目录有深层结构:

```bash
mkdir -p ~/deep-test/{a,b/c,d/e/f}
echo "x" > ~/deep-test/a/keep.txt
echo "x" > ~/deep-test/a/old.bak
echo "x" > ~/deep-test/b/c/file.txt
echo "x" > ~/deep-test/b/c/old.bak
echo "x" > ~/deep-test/d/e/f/important.txt
```

任务:
- 进入 dired
- `i` 插入所有子目录
- `%m \.bak$ RET` 标记所有 .bak (递归)
- `D` 删除

这个挑战用"子目录插入"处理深层目录。`i` 把所有子目录插入一个 buffer,`%m` 跨目录标记,`D x` 一次清理。

真实场景: 项目目录树有几十层,你想清理所有 `*.pyc`——`i` 插入所有子目录,`%m \.pyc$` 标记,`D x` 一次完成。

### 挑战 3: 重命名 100 个文件

```bash
mkdir -p ~/rename-test
for i in {1..100}; do touch ~/rename-test/photo_$i.jpg; done
```

任务: 10 秒内把所有 `photo_N.jpg` 改成 `vacation_NN.jpg` (NN 是 0 补全)。

提示:
- wdired + kmacro
- 或 shell: `! mv ? $(echo ? | sed 's/photo_/vacation_0/') RET` (复杂)
- 或 `(dired-mark-files-regexp "photo_\\([0-9]+\\)")` + Elisp 批量改名

这个挑战最硬——0 补全不是 query-replace 能做的。WDired + kmacro 是答案: 录一个宏"找下一个 photo_N,改成 vacation_0N",跑 100 次。

更好的方案是 Elisp——但那是 Module 3 的事。Capstone 用 kmacro 就够了。**真实工作流里,你学到 Module 3 就能用代码做这个**,这就是为什么 Elisp 重要。

---

## 写日志

完成后,打开 `logs/module-02.md`:

```markdown
# Module 2 学习日志

**日期**: 2026-XX-XX
**用时**: ___ 小时
**完成度**: ___%
**Capstone 用时**: ___ 分钟
**Capstone 评分**: ___/25

## Dired 心得

- 第一次见 dired 的感受:
- 学完后的感受:
- 最让我"啊哈"的瞬间:

## WDired 心得

- 第一次用 wdired 的体验:
- 哪些场景最适合 wdired:

## ibuffer 心得

- 之前的 buffer 管理:
- 现在的 buffer 管理:
- 分组配置:

## Capstone 反思

- 时间花在哪了?
- 用了几次鼠标?
- 哪些操作可以更快?

## 我学到的最重要 3 件事

1.
2.
3.

## 下一步

进入 Module 3: Emacs Lisp 入门 (Chassell) ★核心★
```

写日志不是仪式,是**反思工具**。回答这些问题时,你会发现自己实际学到了什么、还差什么。这种元认知 (对学习本身的学习) 是深度学习的核心。

特别关注"最让我啊哈的瞬间"——那是你真正理解某个概念的标志。可能是"原来 Dired 就是 buffer"、"WDired 让 query-replace 作用于文件名"、"ibuffer 和 Dired 同构"等。这些"啊哈"积累起来,就是你 Emacs 思维的地基。

---

## 毕业仪式

如果你完成了 Module 2:

**恭喜**。

你已经把"文件管理"也搬进了 Emacs。不再需要切到文件管理器。你的整个工作流越来越集中在 Emacs 内。

这是从"Emacs 用户"到"Emacs 极客"的重要一步。你不再用"分散的工具"——文件管理器一个、编辑器一个、IDE 一个——而是用统一的 Emacs。所有操作共享同一套肌肉记忆,所有数据都是 buffer。

接下来的 Module 3 是**核心**:
- 5 周读完 Chassell
- 建立 Lisp 思维
- 写出真正的 Elisp 函数

这是从"Emacs 极客"到"Emacs 巫师"的转折点。Module 2 你学了"用 Emacs",Module 3 你会学"扩展 Emacs"。前者是消费,后者是创造。

学完 Module 3,你就能写自己的 Dired 命令、自己的 ibuffer filter、自己的批量改名工具——Emacs 真正变成"你的 Emacs",而不是"GNU 的 Emacs"。

更新 `PROGRESS.md`: Module 2 状态 ✅。

进入 `03-elisp-foundation/README.md`。

**Module 2 完成。Module 3 见。**
