# Module 2: Dired + Buffer/Window 管理 — 像 God 一样操作文件

> **目标**: 把文件管理完全迁到 Emacs 内,像操作文本一样操作文件
> **时长**: 4 天 (~12 小时)
> **难度**: ★★★☆☆ (Dired 的范式与文件管理器不同)
> **依赖**: Module 1 (移动/搜索/region 命令)
> **核心产出**: 10 分钟内用 dired 整理一个混乱目录

---

## 0. 这个模块在学什么

你已经能在 Emacs 里编辑文件了。但你打开的"文件"是从哪里来的?

回想一个典型工作日: 你打开 Nautilus/Finder/Explorer,双击进文件夹,右键"重命名",用鼠标拖拽归档,再切回 Emacs 继续敲字。每一次鼠标点击,你都在做一次**应用切换**——大脑要从"编辑模式"切到"操作系统模式",再切回来。看起来没什么,但一天累积几十次,注意力被切碎了。

更微妙的是: 你在两个世界里用的是两套心智模型。Emacs 里你用 isearch 找单词,用 query-replace 改一组词,用 kmacro 录操作。文件管理器里这些都没有——它有自己的"右键菜单",有它自己的"批量选中"逻辑,有它自己的"压缩/解压"对话框。**你被迫维护两套工具**,只为做本质上同一件事: 操作"信息单元"。

**Module 2 的解决方案**: 用 Dired,把文件管理也变成 buffer 编辑。

这是一次范式转移,不是技巧叠加。完成后你的工作流会发生质变:

- 不开文件管理器,在 Emacs 里看目录、改文件名、删文件——和编辑代码用同一套肌肉记忆
- 用 wdired 直接编辑文件名,**像编辑代码一样**——这意味着 isearch、query-replace、kmacro、矩形操作全部可用
- 跨多个目录做批量操作——标记 100 个文件,一次操作
- 管理几十个 buffer 不迷路——ibuffer 给你"文件管理器式"的 buffer 视图

更深一层: 一旦你接受了"目录也是文本",你的脑里就少了一层抽象。你不再问"这个文件在哪、叫什么",而是问"这一行文本怎么改"。**统一性**是 Emacs 哲学的核心回报。

---

## 1. 第一性原理: 为什么 Dired 这样设计?

### 1.1 一个反直觉的设计

让我们先用第一性原理推导 Dired 的存在。从最基本的事实出发:

**事实 1**: 目录的本质是"一个文件名列表"。Unix 一切皆文件,目录也是文件——它内部存着一组 `(name, inode)` 的映射。

**推论 1**: 既然目录是列表,而列表可以用文本表示,那么目录就可以渲染成一段文本。`ls` 命令做的事情就是这个——把目录的内部表示翻译成可读字符串。

**推论 2**: Emacs 最擅长操作什么? 文本。Emacs 有全世界最强的文本编辑命令: isearch、query-replace、矩形编辑、kmacro、occur……

**推论 3**: 既然如此,为什么不把目录列表塞进一个 buffer,然后用 Emacs 的所有文本操作命令来处理它? 这就是 Dired 的核心洞见。**目录列表 = buffer = 可编辑文本**。

**推论 4**: 一旦目录列表是 buffer,你就可以**编辑它**。但文件系统不允许任意编辑——你把一行删了,真要删文件吗? 你把一行改名了,真要 rename 吗? Dired 的答案是"标记 + 操作"两步走: 你先在 buffer 里标记意图 (用 `d`、`D`、`m` 等字符),然后按一个键 (`x`) 让 Emacs 把这些意图翻译成实际的文件系统调用。

**推论 5**: 既然可以"编辑"目录,那能不能**直接编辑文件名**,改完提交? 可以——这就是 WDired (`C-x C-q`)。它把"文件操作"完全降维成"文本编辑"。这是别的文件管理器做不到的事,因为它们把文件名当成不可触碰的 UI 元素,而 Emacs 把文件名当成 buffer 里的一行字符。

这条推理链解释了为什么 Dired 比所有 GUI 文件管理器都强: 它没有发明新的 UI 范式,而是复用了 Emacs 已经积累 40 年的文本操作能力。

普通文件管理器:
- 双击文件 → 打开
- 右键 → 菜单
- 拖拽 → 移动
- 鼠标为主

Dired:
- 文件名是 buffer 里的一行文本
- 你**编辑**这些文本行
- 不同的字母触发不同操作 (d=delete 标记, r=rename, c=copy)
- 键盘为主

新手第一次看 Dired 会觉得"反人类",因为它看起来像 `ls` 的输出,而不是一个 UI:
```
/home/sun/:
total 40
drwxr-xr-x  2 sun sun 4096 Jun 19 20:55 .
drwxr-xr-x 12 sun sun 4096 Jun 19 20:54 ..
-rw-r--r--  1 sun sun  148 Jun 19 20:55 ROADMAP.md
-rw-r--r--  1 sun sun  191 Jun 19 20:55 PROGRESS.md
...
```

像 ls 输出。然后呢?

**关键认知**: Dired buffer 是**可编辑的 buffer**。每一行就是一个文件,你可以对它做任何 Emacs 文本操作。Dired 只是把"操作文本"和"操作文件系统"用一个**约定**关联起来: 某些字符出现在行首 (`*`、`D`、`C`...) 意味着"对这个文件做某事",按 `x` 时把这些意图一起提交执行。

这就是为什么 Dired 命令的名字是 `dired-mark` 而不是 `mark-file`——你确实是在"标记 buffer 的一行"。文件系统的实际操作发生在这之后。

- 你可以选中行 (`m` 标记)
- 可以删除行 (`k` 隐藏, `x` 真执行)
- 可以重命名 (`R`)
- 可以**直接编辑文件名** (`C-x C-q` 进入 wdired)

### 1.2 历史背景

Dired = **Di**rectory **Ed**itor。1980 年代由 Richard Mlynarik 写,后来融入标准 Emacs。

它的设计灵感来自 Emacs 的"buffer 即世界"哲学: 既然 buffer 是统一的文本容器,那目录列表也应该是 buffer;操作文件 = 操作 buffer 行。这不是"为了与众不同而与众不同",而是 Emacs 整体设计的自然推论——如果 Emacs 一切皆 buffer,那么目录不能例外。

一旦接受这个统一性,Dired 比 GUI 文件管理器**强 10 倍**的能力就自然浮现:

- **批量操作**: 标记一组文件,一次操作。GUI 文件管理器要 Ctrl-点击逐个选,Dired 用正则一秒标记几百个。
- **正则标记**: `%m \.py$` 标记所有 .py 文件。GUI 做不到。
- **跨目录操作**: 用 `i` 把多个子目录"插入"到一个 dired buffer,然后跨目录批量操作。GUI 必须打开多个窗口。
- **WDired**: 直接编辑文件名! 这是杀手锏,GUI 完全没有对应物。
- **操作预览**: 标记后再确认,所有改动先以字符显示在 buffer 里,你有时间审视。

### 1.3 "Buffer 操作即文件操作"

让我用一个例子建立直觉。

假设你 dired 一个目录:

```
~/projects/foo/
  - main.py
  - test.py
  - notes.txt
  - backup.bak
  - old_main.py
```

你想:
1. 删 backup.bak 和 old_main.py
2. 把 notes.txt 改名 README.md
3. 把 test.py 移到 tests/ 目录

**普通方法**:
- 找文件管理器,右键删
- 右键改名
- 拖拽移动
- 5 分钟,鼠标累

**Dired 方法**:

```elisp
;; 在 dired buffer 里:
;; 移到 backup.bak,按 d (标记删除)
;; 移到 old_main.py,按 d
;; 移到 notes.txt,按 R (rename),输入 README.md
;; 移到 test.py,按 R,输入 tests/test.py
;; 按 x (执行所有标记)
```

或者更高级,正则标记:
```
%m \.bak$ RET     标记所有 .bak
%m \.old$ RET     标记所有 .old
D                 标记删除
x                 执行
```

**全程键盘,几秒钟。**

注意这个例子里的关键转换: 你**没在操作文件**,你在**编辑 buffer 里的字符**——按 `d` 在行首插入字符 `D`,按 `R` 触发一个询问。所有这些操作都发生在 buffer 里,只有按 `x` 时 Emacs 才把标记翻译成真正的 `unlink()` 系统调用。这种"延迟执行"是 Dired 的安全网: 你可以慌乱地按一堆 `d`,真删之前有清晰的视觉提示和最后确认。

### 1.4 为什么这套设计经久不衰

思考一下: Dired 的核心 API 在过去 40 年几乎没变。为什么?

因为它建在两个稳定的基石上: (1) 文件系统的语义 (`readdir`、`rename`、`unlink`) 没变; (2) Emacs 的 buffer/文本操作语义没变。Dired 只是这两者之间一层薄薄的胶水,所以它既不会过时,也不需要频繁"升级"。

对比现代文件管理器: 它们的 UI 每隔几年重做一次,你刚熟悉一套快捷键就过时了。Dired 不存在这个问题。学一次,用一辈子。这是 Unix 哲学的回报:**简单抽象长寿,复杂 UI 易朽**。

---

## 1.5 Dired 的创造性应用 (意想不到的用法)

Dired 不只是"看文件、改文件名"。它的一些用法会让你重新认识"文件管理":

1. **批量加前缀 (WDired 矩形编辑)**: 选中 20 个文件名行,`C-x r t prefix_ RET` 一秒加前缀。在 GUI 里要写脚本。

2. **跨文件 query-replace**: 标记所有 `.py`,`Q old_func RET new_func RET`,在所有文件里执行 query-replace。等于把 sed -i 升级成"交互式 + 可视化"。

3. **图片批量缩放**: `! convert ? -resize 50% thumb_? RET`——`?` 替换为每个文件名。ImageMagick 在 Dired 里像批处理脚本一样自然。

4. **跨目录 diff**: 开两个 Dired (左右 window),`=` diff 两个同名文件。版本对比不用切 Git。

5. **Wdired 改扩展名**: 标记所有 .jpg,query-replace `.jpg` → `.png`,提交。批量改后缀的 5 秒方案。

6. **Dired 作为项目导航**: `i` 插入所有子目录,一个 buffer 看整个项目的目录结构。比 treemacs 轻。

7. **批量 chmod**: 标记所有 .sh,`M 755 RET`。比 `find . -name '*.sh' -exec chmod 755 {} \;` 直观。

8. **临时图片预览**: 装 `peep-dired`,上下键浏览时旁边实时预览图片。比 Finder 的 Quick Look 还快。

9. **Dired 作为版本管理**: `~` (dired-flag-backup-files) 标记所有 `FILE~` backup,`x` 一次清理。Emacs 自带的 backup 系统配合 Dired 是天然的工作流。

10. **Dired 作为任务清单**: 用 `%g PATTERN` 标记内容含某个 TODO 关键词的文件——你的项目里所有"未完成"的工作一眼可见。

这些用法之所以可能,全部归功于 Dired 的"一切皆文本"哲学。GUI 文件管理器是封闭的——它们提供什么功能,你就只能用什么功能。Dired 是开放的——任何能作用于文本的命令,都可以作用于文件。

---

## 2. 这个模块的内容

模块分 4 天,每天一个递进的主题。第一性原理已经讲清楚为什么 Dired 长这样,接下来我们要把它落到肌肉记忆上。

### 2.1 Dired 基础 (Day 1)

第一天先建立"buffer 即目录"的直觉。学会进入、导航、标记、做最常用的 5 个操作 (打开/复制/重命名/删除/创建目录)。**重点不是背键位,是建立"按字母 → buffer 变化 → 文件系统变化"的因果链**。

- 进入 Dired
- 导航 (上下、目录切换)
- 标记/取消标记
- 基本操作 (open/rename/copy/delete)

### 2.2 Dired 高级 (Day 2)

第二天进入 Dired 的"超能力区": 正则标记、子目录递归、批量操作、shell 命令。这一天你会开始感到"惊异"——一个简单的 `%m \.py$` 就能让 100 个文件同时被选中,这在 GUI 里要 Ctrl-点击到手抽筋。

- 正则标记
- 子目录递归
- 批量操作
- shell 命令 (`!`)

### 2.3 WDired: 直接编辑文件名 (Day 3)

第三天学 WDired——Emacs 最被低估的特性之一。`C-x C-q` 把 Dired buffer 变成可编辑——你可以直接改文件名、用 query-replace 批量改名、用矩形操作加前缀、用 kmacro 录制改名宏。改完 `C-c C-c` 提交,Emacs 实际执行 rename 操作。

这意味着**所有 Emacs 的文本编辑能力 (kmacro, isearch, replace, 矩形) 都可以用于文件操作**。这是别的文件管理器做不到的——它们有自己的 UI,而 Emacs 把一切统一为文本。WDired 是 Module 2 的"顿悟时刻"。

- 进入 wdired (`C-x C-q`)
- 编辑文件名像编辑代码
- 矩形操作 / query-replace / kmacro 都能用
- 提交 (`C-c C-c`)

### 2.4 Buffer 管理 (Day 4)

第四天转到 Buffer 管理。学完 Dired 后你会发现: 你的 buffer 数量爆炸——每个 Dired、每个文件都是 buffer。怎么不迷路? 答案是 `ibuffer`,一个 Dired 风格的 buffer 列表。它支持标记、过滤、分组、批量操作——和 Dired 用同一套心智模型。**这是 Module 2 的另一条第一性原理线索**: 既然 Dired 能把目录当 buffer 操作,那为什么不能把 buffer 列表也当 buffer 操作? ibuffer 就是这个问题的答案。

- ibuffer (比 list-buffers 强)
- buffer 分组
- 批量关闭
- buffer 操作

---

## 2.5 为什么 buffer 和 window 分离? (Module 1 复习 + 深入)

Module 1 我们学了 buffer 和 window 的区别。Module 2 的 Dired 经验让我们能更深入理解这个分离:

**数据 vs 视图**: buffer 是数据 (文件内容、目录列表、shell 输出……),window 是视图 (一块屏幕区域显示某个 buffer)。这是经典的 MVC 分离。Emacs 把这个分离做得极端——同一个 buffer 可以同时显示在多个 window 里,你在一边改,另一边同步更新。

**为什么 mark 在 buffer 不在 window?** 因为 mark 是数据的一部分 ("我标记了什么"),它属于 buffer 的状态,不属于窗口。所以你在 window A 设了 mark,切到 window B (显示同一 buffer) 也能用 `C-u C-SPC` 跳回去。Window 是临时的,buffer 是持久的。

**为什么有 indirect buffer?** 同一份数据可以有多个"视图"——不同的 narrow、不同的 major mode 偶尔、不同的光标位置。Indirect buffer 共享底层文本,但维护自己的窗口状态。这是"同一数据多视图"的极致表达。

**为什么有 buffer-local 变量?** 因为有些配置只在特定 buffer 有意义——比如 `dired-directory` (每个 Dired buffer 指向不同目录)、`fill-column` (代码 vs 文档不同宽度)。Buffer-local 让你 per-buffer 配置,不用全局污染。Dired 大量使用 buffer-local 变量。

这些"为什么"在 Day 4 的 ibuffer 学习中会变得具体——你会看到 ibuffer 也是 buffer,它显示的也是 buffer 列表这个"数据"。Emacs 的一致性让你触类旁通。

---

## 3. 准备

加到你的 init.el:

下面这段配置不是"必备",但能把 Dired 从"能用"提升到"舒服"。每一行配置都解决一个具体痛点:

```elisp
;;; Dired 配置
(setq dired-dwim-target t           ; 智能选另一个 window 作为目标
      dired-recursive-copies 'always
      dired-recursive-deletes 'top
      dired-listing-switches "-alh"  ; ls -alh
      dired-auto-revert-buffer t     ; 文件变了,dired 自动刷新
      dired-clean-confirm-killing-deleted-buffers nil)
```

`dired-dwim-target` 是**最有价值的一项**: 当你按 `C` (复制) 时,如果另一个 window 显示了一个 Dired buffer,Emacs 自动把那个目录作为默认目标。这意味着分屏左右各开一个 Dired,复制/移动就是"按 C,回车"两步。GUI 文件管理器做不到这么顺。

`dired-recursive-copies 'always` 让你按 `C` 复制目录时不再问"递归吗?",默认递归。`dired-recursive-deletes 'top` 删除目录时也类似 (但只在顶层问一次,避免每个文件都确认——这是安全 vs 烦人的平衡点)。

`dired-listing-switches "-alh"` 传给底层 `ls`——`a` 显示隐藏文件,`l` 长格式,`h` 人类可读大小 (1.2K 而不是 1228)。这是 Unix 工具哲学的体现: Dired 不重新发明文件列表,它复用 `ls`,你给什么 flag 就显示什么。

`dired-auto-revert-buffer t` 是关键: 文件被外部改了 (比如你同时跑了 shell 命令改了文件),Dired 自动刷新。否则你要手动 `g`,很容易忘。

```elisp
;;; 让 dired 不重复开 buffer (旧 buffer 复用)
(put 'dired-find-alternate-file 'disabled nil)
```

这行启用 `(` (dired-find-alternate-file),让你按 `RET` 进入子目录时**复用同一个 buffer**,而不是每次新开一个。这是 Dired 用户必备——否则你浏览几下目录,buffer 列表爆炸。

默认这个命令被 disable 是历史原因 (它有一些边角 case)。但对现代用户来说,启用它体验提升巨大。

```elisp
;;; 颜色 (Emacs 28+ 内置 diredfl 包)
(when (fboundp 'diredfl-mode)
  (add-hook 'dired-mode-hook #'diredfl-mode))
```

`diredfl-mode` 给 Dired 加颜色——目录蓝、可执行绿、symlink 青……颜色帮你一眼识别文件类型。Emacs 28+ 内置,无需装包。

```elisp
;;; 隐藏细节 (按 ( 切换)
(setq dired-hide-details-hide-symlinks nil)
(add-hook 'dired-mode-hook #'dired-hide-details-mode)
```

`dired-hide-details-mode` 默认隐藏权限/owner/size 那一列,只看文件名——视图清爽。需要细节时按 `(` 切换显示。**这是 Dired 节制原则的体现**: 默认显示最少必要信息,需要时再展开。

如果你想要更强体验,装 `dirvish` 或 `dired+` 包 (Module 5 之后)。它们提供图片预览、现代 UI 等。但**先掌握原生 Dired**,再升级——否则你会被花哨功能分心,错过 Dired 的本质。

---

## 4. 这个模块不教什么

避免范围蔓延是 Emacs 学习的关键——Emacs 太大了,什么都想学结果什么都没学会。下面这些主题**有价值但不在本模块**:

- ❌ treemacs / neotree (项目树侧栏) — 那是另一种范式,与 Dired 不冲突但思路不同 (侧栏 vs 主 buffer)。可选。
- ❌ fd / ripgrep 集成 (Module 5) — 现代 grep 替代品,但你要先掌握 Dired 的原生 `A` 搜索。
- ❌ 图片预览 (dirvish 提供,Module 5) — 花哨但不是核心。
- ❌ Magit (Module 5) — Git 集成,完全独立的话题。
- ❌ Tramp 远程文件 (附录提及,不深入) — 远程 Dired 强大但需要单独模块。

这个模块专注**经典 Dired + WDired + Buffer 管理**。先把这些基础打牢,Extensions 自然水到渠成。

---

## 5. 毕业检查

完成这个模块后,你应该能:

### 概念题

下面 8 题考验你**理解**,不是记忆。如果某题答不上来,说明那个部分的"为什么"没真正吃透,回去重读对应章节。

1. Dired buffer 和普通 buffer 的区别? (提示: 内容从哪来?)
2. WDired 是什么? 它和 Dired 的关系? (提示: 同一份数据,不同视图)
3. ibuffer 比 list-buffers 强在哪? (提示: Dired 风格 vs 静态列表)
4. `dired-dwim-target` 干啥? (提示: 双窗口工作流)
5. 怎么标记所有 .py 文件? (提示: 正则)
6. 怎么跨目录递归删除所有 *.bak 文件? (提示: `i` + `%m` + `D` + `x`)
7. 怎么在多个 buffer 间快速切换? (提示: `C-x left/right` 或 ibuffer)
8. 怎么批量关闭一组 buffer (例如所有 .py)? (提示: ibuffer + `% f` + `D` + `x`)

### 实操题

1. 10 分钟内整理一个 50 文件的混乱目录
2. 用 wdired 把 20 个 `IMG_001.jpg` 改成 `vacation_001.jpg`
3. 用 ibuffer 关闭所有 `*Help*` buffer

如果实操题 1 你能 7 分钟内完成,你已经超过大多数 Emacs 用户。

---

## 6. 常见陷阱 (反例)

学习 Dired 时,新手最容易掉进这几个坑:

**陷阱 1**: 以为 `d` 就是删除。错——`d` 只是**标记**待删除 (`D` 字符),按 `x` 才执行。新手常因为看到 `D` 字符就慌,以为文件已经没了。其实只要不按 `x`,你可以 `u` 取消,什么都不会发生。

**陷阱 2**: WDired 提交后报错。如果你改的文件名包含非法字符 (如 `/` 在文件名中间被解释成路径但目标目录不存在),Emacs 会报错。提交后看 `*WDired errors*` buffer,里面会列出哪些文件没成功改名。**改完先 `C-c C-c`,有错就检查**。

**陷阱 3**: `R` 不是"rename"是"move"。文件名带路径就等于移动。`R file.txt RET /tmp/file.txt RET` 把文件移到 /tmp。新手以为 rename 只能改文件名,其实可以同时改路径。

**陷阱 4**: Dired 默认不显示隐藏文件 (dotfiles)。`dired-listing-switches` 加 `a` 才显示。如果你的 `.config` 找不到,是这个原因。

**陷阱 5**: `!` shell 命令里的 `?` 是文件名占位符,不是正则。新手写成 `! convert $1 -resize 50% out.jpg` 是错的——应该是 `! convert ? -resize 50% thumb_? RET`。

**陷阱 6**: ibuffer 默认按访问时间排序,新手以为 buffer 顺序乱了。按 `,` 切换排序方式 (名字/大小/mode/时间)。

记住这些陷阱,你能省下大量调试时间。

---

## 7. 下一步

打开 `02-dired/concept-anchor.md` 看 Emacs manual 的对应章节。Concept Anchor 把手册的"为什么"和"内部机制"展开讲。

然后 `02-dired/drills.md` 做练习。Drills 是 40 题渐进式实操,把概念变成肌肉记忆。

最后 `02-dired/capstone.md` 做毕业项目——10 分钟整理一个 50 文件的混乱目录。完成它你就真的"会用 Dired"了。
