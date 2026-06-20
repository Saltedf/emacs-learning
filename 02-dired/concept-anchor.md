# Concept Anchor: Emacs 编辑器手册 (Module 2)

> 这个文件**完全替代** `emacs-manual-30.2/` 的以下章节:
> - `dired.texi` (Dired, the Directory Editor)
> - `dired-xtra.texi` (Subdirectory Switches in Dired)
> - `buffers.texi` (Using Multiple Buffers)
> - `windows.texi` (Multiple Windows) (复习,Module 1 已学用户视角)
> - `files.texi` (File Handling) (节选)

---

## 1. Dired 内联讲解 (替代 dired.texi)

### 1.1 Dired 是什么?

**Dired** = **Di**rectory **Ed**itor。它把一个目录的内容渲染成一个 buffer,你可以"编辑"这个 buffer 来操作文件。

这句话听起来简单,但它的含义是深刻的。常规理解里,"编辑器编辑文本,文件管理器管理文件"——这是两种不同的工具,两套不同的操作逻辑。Dired 把这两个世界合并了: 目录的内容被翻译成一段文本放进 buffer,你对这段文本的"编辑意图"被翻译成文件系统操作。

这个设计有几个直接推论:

第一,**Dired 没有自己的 UI 组件**。它就是 buffer。你看到的 `ls` 风格的列表就是 buffer 的内容,可以用 Emacs 的所有命令导航——`C-s`、`M-x occur`、`C-v` 翻页——全部原生支持。不需要学新交互。

第二,**Dired 的所有"操作"都是 buffer 命令**。按 `m` 不是"调用文件系统 API",而是"在 buffer 当前行的开头插入字符 `*`"。文件系统操作发生在你按 `x` 时,Emacs 扫描 buffer 找出所有 `*` 标记,然后翻译成对应的系统调用。

第三,**Dired 是延迟执行的**。你可以在 buffer 里堆积一堆意图 (删除 A、复制 B、重命名 C),审视它们,然后一次性提交。这是 GUI 文件管理器做不到的——GUI 里每个操作立即执行,错了就要撤销。

进入 Dired 有三种方式,效果略有不同:

```elisp
;; 进入 dired
M-x dired RET ~/ RET
;; 或
C-x d ~/ RET
;; 或
C-x C-f ~/ RET  (find-file 一个目录也进入 dired)
```

打开后看到类似:

```
/home/sun/:
  total used in directory 32K available 100G
  drwxr-xr-x 2 sun sun 4.0K Jun 19 20:55 .
  drwxr-xr-x 12 sun sun 4.0K Jun 19 20:54 ..
  -rw-r--r-- 1 sun sun 148 Jun 19 20:55 ROADMAP.md
  -rw-r--r-- 1 sun sun 253 Jun 19 20:55 MANUAL-CROSSREF.md
  -rw-r--r-- 1 sun sun 191 Jun 19 20:55 PROGRESS.md
  drwxr-xr-x 2 sun sun 4.0K Jun 19 20:55 00-mindset/
  drwxr-xr-x 2 sun sun 4.0K Jun 19 20:55 01-survival/
  ...
```

**关键**: 每行是一个文件/目录,你可以用 Emacs 的所有编辑命令操作它们。

注意 Dired buffer 的格式——它就是 `ls -alh` 的输出。第一列是权限 (drwxr-xr-x),第二列是 link count,然后 owner、group、size、mtime、文件名。**Emacs 没有重新发明列表格式**,它复用了 Unix `ls`。这意味着: (1) 你在 shell 里能读 `ls` 输出就能读 Dired; (2) `dired-listing-switches` 直接控制传给 `ls` 的 flag,你想要 `-t` (按时间) 就改这个变量。

Dired buffer 里 Emacs "知道" 哪部分是文件名 (最后那列),靠的是正则解析。你按 `m` 时,Emacs 提取当前行的文件名,然后在内部数据结构里记录"这个文件被标记了"。这种"从文本反推数据"的设计贯穿整个 Dired。

### 1.2 在 Dired 里导航

Dired 的导航命令就是普通 buffer 的导航——加上几个 Dired 特有的便捷键。一旦你建立"这就是 buffer"的直觉,导航几乎不用学。

| 键位 | 命令 | 作用 |
|---|---|---|
| `n` / `C-n` / `SPC` | dired-next-line | 下一行 |
| `p` / `C-p` | dired-previous-line | 上一行 |
| `RET` | dired-find-file | 打开文件/进入目录 |
| `o` | dired-find-file-other-window | 在另一个 window 打开 |
| `^` | dired-up-directory | 上一级目录 |
| `i` | dired-maybe-insert-subdir | 把子目录插入当前 dired (一起看) |
| `j` | dired-goto-file | 跳到指定文件 (输入名字) |
| `M-<` / `M->` | | 头/尾 |

`n` 和 `p` (next/previous) 是 Dired 专属,但它们其实就是 `C-n`/`C-p` 的别名——Dired 给它们更短的键,因为这两个操作太频繁了。**Emacs 的传统: 高频操作给短键**。

`^` (上目录) 值得记住。在 GUI 文件管理器是"返回上级"按钮,在 Dired 就一个字符。它实际是 `dired-up-directory`,会新开一个 Dired buffer 显示上级目录 (除非你启用了 `dired-find-alternate-file`,见 README 配置)。

`i` 是 Dired 的"杀手锏"——把子目录"插入"当前 Dired。你可以同时在 buffer 里看到当前目录和几个子目录的内容,跨子目录批量操作。这是 GUI 完全做不到的。后面 §1.7 详细讲。

`j` 是"按名字跳转"。输入 `foo.txt`,光标直接跳到那行。比 isearch 还快——不需要敲完整文件名,Emacs 会增量匹配。

### 1.3 标记 (Marking) — Dired 的灵魂

Dired 的核心是**标记**。你先标记一组文件,然后批量操作。

为什么是两步? 想想 GUI 的痛苦: 你 Ctrl-点击 20 个文件,一不小心点错了位置,选中集就乱了。Dired 把"选择"和"操作"解耦——你慢慢标记 (可以撤销、可以翻转、可以用正则批量),确认后再一次性操作。**这是 "CQRS" (Command Query Responsibility Segregation) 在文件管理上的应用**——读 (标记) 和写 (执行) 分离。

标记还有一个好处: **可视化**。所有标记在 buffer 里以字符形式显示 (`*`、`D`、`C`...),你能一眼看到自己即将做什么。GUI 的"选中"只是高亮,看不出意图 (要删? 要复制? 要重命名?)。

| 键位 | 命令 | 作用 |
|---|---|---|
| `m` | dired-mark | 标记当前行 (前面加 `*`) |
| `u` | dired-unmark | 取消标记 |
| `U` | dired-unmark-all-marks | 取消所有标记 |
| `t` | dired-toggle-marks | 翻转标记 (标记↔未标记) |
| `*` / `* *` | dired-mark-executables | 标记所有可执行 |
| `* /` | dired-mark-directories | 标记所有目录 |
| `* @` | dired-mark-symlinks | 标记所有 symlink |
| `* s` | dired-mark-files-containing-regexp | (有点不同) |
| `* .` | dired-mark-extension | 标记某扩展名 |
| `% m` | dired-mark-files-regexp | **正则标记** (最常用) |
| `% g` | dired-mark-files-containing-regexp | 标记内容含某正则的文件 |

`%m` 是最强大的——用正则标记。`%m \.py$` 一秒标记所有 .py,`%m ^test_` 标记所有 test_ 开头。这种"基于模式的选择"是 Emacs 文本操作的直接复用——既然 Dired 是文本,正则就是天然的工具。

`%g` 更进一步: 它打开每个文件查内容。`%g TODO RET` 标记所有内容含 "TODO" 的文件——这等于 grep + 标记,你可以接着批量操作这些文件 (例如 `Q TODO RET DONE RET` 把 TODO 改成 DONE)。

`t` (翻转) 看起来奇怪,其实很有用: 你想"选中除了 .py 以外的所有",就先 `%m \.py$`,然后 `t`——非 .py 的都被标记了。

**标记字符**:

Dired 用不同字符表示不同意图,这些字符显示在每行行首:

- `*` — 通用标记 (用于大多数操作)
- `D` — 待删除标记
- `C` — 待复制
- `R` — 待重命名
- `!` 或其他 — 模式特定

实际上 `*` 是"通用选中",`D`/`C`/`R` 这些是"预承诺操作"——某些命令会直接打这种标记,你按 `x` 时它们各自执行。但日常用法 90% 是 `*` (通用) + 命令 (`C`/`R`/`D`)。

### 1.4 单文件操作

打开/查看文件,在 Dired 里就是"光标所在行 + 一个键"。

| 键位 | 命令 | 作用 |
|---|---|---|
| `f` | dired-find-file | 打开文件 (同 RET) |
| `e` | dired-find-file | 同上 |
| `o` | dired-find-file-other-window | 在另一个 window 打开 |
| `C-u o` | | 选择哪个 window |
| `v` | dired-view-file | 只读查看 |
| `^` | dired-up-directory | 上一级 |
| `RET` | dired-find-file | 同 f |
| `C-u RET` | | 用其他程序 (你输入) |

`f` 和 `e` 都是 `dired-find-file`——Emacs 有时给同一命令多个绑定,因为不同人记忆方式不同。`RET` 也一样。**Emacs 的传统: 同一操作给多个键**,适配不同肌肉记忆。

`v` (view-mode) 是查看模式——只读打开文件,按 `q` 退出。适合快速浏览不修改的场景。比 `f` 打开后 `C-x C-q` 转只读更轻量。

`o` 在另一个 window 打开是高频操作。它会自动 `split-window` 然后在另一边打开文件。配合 `dired-dwim-target`,你的双窗口工作流就成型了。

### 1.5 文件操作 (对标记或当前)

这一组是 Dired 的核心——对文件系统做实际改动的命令。

| 键位 | 命令 | 作用 |
|---|---|---|
| `C` | dired-do-copy | 复制 |
| `R` | dired-do-rename | 重命名/移动 |
| `D` | dired-do-delete | 删除 |
| `+` | dired-create-directory | 新建目录 |
| `M` | dired-do-chmod | 改权限 |
| `O` | dired-do-chown | 改 owner |
| `G` | dired-do-chgrp | 改 group |
| `T` | dired-do-touch | touch (改时间戳) |
| `S` | dired-do-symlink | 创建 symlink |
| `H` | dired-do-hardlink | 创建 hardlink |
| `Z` | dired-do-compress | 压缩/解压 |
| `!` | dired-do-shell-command | 跑 shell 命令 |
| `&` | dired-do-async-shell-command | 异步 shell |
| `=` | dired-diff | diff (跟另一文件) |
| `w` | dired-copy-filename-as-kill | 复制文件名到 kill-ring |
| `B` | dired-do-byte-compile | byte-compile .el |
| `L` | dired-do-load | load .el |
| `A` | dired-do-find-regexp | 在所有标记文件里搜 |
| `Q` | dired-do-find-regexp-and-replace | query-replace 在所有标记文件 |

每个命令背后都有一段逻辑。让我挑几个最常用的展开讲:

**`C` (dired-do-copy)**: 在 Dired 里按 `C` 复制文件,Emacs 做的事比"复制"复杂得多。它先询问目标路径 (默认填当前目录 + 原文件名),然后调用 `copy-file` 函数。如果目标已存在,默认会问覆盖;如果你设了 `dired-copy-preserve-time`,还会保留原 mtime。如果跨文件系统,会自动 fallback 到 byte-copy (因为 rename 不能跨 FS)。**这就是 Emacs 的哲学: 一个简单的按键背后是周全的逻辑**。

**`R` (dired-do-rename)**: 重命名也类似——它实际是"移动",可以跨目录、跨 FS,会保留权限。`R file.txt RET /tmp/file.txt RET` 把文件移到 /tmp。新手常以为 rename 只能改文件名,其实路径任意。

**`D` (dired-do-delete)**: 删除会先标记 (`d`),按 `x` 才执行,中间可以 `u` 取消。**这是设计上的安全网**——你可以慌乱地按 `d` 几次,真删之前有一次"刹车"机会。`dired-recursive-deletes` 控制是否递归删目录。

**`+` (dired-create-directory)**: 创建目录,可以一次创建多层 (`+ a/b/c RET`)。底层调 `make-directory`,Unix `mkdir -p` 的等价物。

**`M` (dired-do-chmod)**: 改权限。`M 755 RET` 改成 rwxr-xr-x。底层调 `set-file-modes`。比 shell 的 `chmod` 直观——你能看到 buffer 里的权限列马上变化。

**`Z` (dired-do-compress)**: 压缩/解压,根据扩展名自动判断。`.gz` 文件按 `Z` 解压,普通文件按 `Z` 压缩成 `.gz`。底层调 `compress-file`/`uncompress-file`。

**`!` (dired-do-shell-command)**: 跑 shell 命令,`?` 替换为文件名。这是 Dired 的"逃生舱"——所有 Dired 没覆盖的操作 (图片转换、grep 替代、git 操作),你都可以用 `!` 接 shell 命令完成。例如 `! convert ? -resize 50% thumb_? RET` 批量生成缩略图。**`!` 让 Dired 永远不会被"卡住"——任何 shell 能做的,Dired 都能做**。

**`Q` (dired-do-find-regexp-and-replace)**: 在所有标记文件里执行 query-replace。这是 Dired 的"杀手级"用法——你可以跨整个项目改名函数、修 bug。等于 `sed -i` 加上交互式确认。后面 §3.3 详细讲。

**`w` (dired-copy-filename-as-kill)**: 复制文件名到 kill-ring。之后 `C-y` 可以粘到任何地方。这解决了"我需要这个文件的完整路径"这个高频小需求——不用 `pwd` + 选中 + 复制,一键搞定。

### 1.6 操作的"批量"特性

`C`、`R`、`D` 等命令:
- 如果你**有标记**的文件,对**所有标记**操作
- 如果你**没标记**,对**当前行**操作

这是 Dired 的设计: "标记 + 操作"两步。

这个设计有个微妙之处: 同一个命令 (如 `C`) 行为依赖于当前状态 (是否有标记)。这在编程里通常是反模式 (函数行为应明确),但在交互式工具里很自然——用户的意图被"标记集"携带,命令读取它。**Emacs 经常这样: 命令的行为由上下文 (mark、prefix arg、buffer 状态) 决定**。学习 Emacs 就是学习这套"上下文敏感"的约定。

如果你想强制对当前行操作 (即使有标记),用 `C-u C` 之类——prefix arg 通常会覆盖默认行为。

### 1.7 子目录递归

Dired 默认只显示当前目录。这看起来是限制,其实是设计——避免 buffer 过长。但你可以扩展视图。

`i` (dired-maybe-insert-subdir) 把光标处的子目录内容"插入"到当前 Dired buffer。这样你可以同时看到 `~/foo/`、`~/foo/sub1/`、`~/foo/sub2/` 等的内容——一个 buffer 看多个目录。

```
i         把光标处的子目录"插入"到当前 dired (一起看)
```

为什么这有用? 想象一个项目结构: `src/`、`tests/`、`docs/` 各有文件。你要把 `tests/` 里所有 `.bak` 删掉,顺便检查 `docs/` 的 `.txt`。GUI 里你要切窗口。Dired 里你 `i` 插入 `tests/` 和 `docs/`,然后 `%m \.bak$` 标记所有 .bak (跨子目录),`D`、`x` 一次完成。

`$` 折叠/展开某个子目录的视图——折叠后只剩 `subdir/` 那一行,展开看到全部内容。`M-$` 折叠所有插入的子目录。这是"信息分层"——你只看当前关心的部分,其余折叠。

`k` (kill) 移除某个子目录的"插入"——但**不删文件**,只是不再显示。文件还在文件系统里,只是不在 Dired buffer 里了。`g` (revert) 会重新生成 buffer,所有插入恢复默认。

### 1.8 操作的预览 + 确认

Dired 是**安全的**。这点怎么强调都不过分。

- `d` 只是标记 `D`,不真删
- `x` 才执行所有 `D` 标记的删除
- 中间可以 `u` 取消

```
D ROADMAP.md         ← 按 d 标记的,黄色 D
D backup.bak
  notes.txt          ← 没标记的,空格
D old_main.py

按 x 执行:
  Delete ROADMAP.md? (y or n)
  Delete backup.bak? (y or n)
  Delete old_main.py? (y or n)
```

这个"标记 → 预览 → 执行"的模式是 Unix 工具罕见的——大多数 Unix 命令直接执行,错了就 undo (如果能的话)。Dired 把它做成默认行为,因为文件操作不可逆 (rm 没有 undo)。

可以设 `(setq dired-deletion-confirms nil)` 跳过单个确认——只看到汇总。**老手会开这个**,新手建议先保留确认,免得手抖。

`dired-clean-confirm-killing-deleted-buffers` 控制删除文件后是否问"是否同时关 Dired buffer"。一般设 nil 省事。

### 1.9 `!` 跑 shell 命令

`!` 是 Dired 的逃生舱——任何 Dired 没内置的操作,你都可以用 shell 命令完成。

```
! COMMAND RET     对标记文件跑 COMMAND (同步)
& COMMAND RET     异步
```

COMMAND 里 `?` 替换为文件名。如果有多个标记文件,`?` 会被替换多次 (每个文件跑一次命令)。

例:
- `! mv ? /tmp/ RET` 移到 /tmp
- `! chmod 644 ? RET` 改权限
- `! convert ? -resize 50% thumb_? RET` (ImageMagick)
- `& xdg-open ? & RET` 用系统默认程序打开

`&` (async) 是关键——长命令 (如视频转换) 不会卡住 Emacs。它在后台跑,完成时显示在 `*Async Shell Command*` buffer。**这是 Emacs 的设计: 长任务一律异步**,你的编辑不被阻塞。

`!` 和 `&` 之外,还有 `&` 的变种: 用 `*` 替换表示"所有文件一起作为一个命令的参数" (而不是每个文件一个命令)。例如 `! tar czf archive.tar.gz * RET` 把所有标记文件打成一个 tar 包。

Dired 的 `!` 还有一个隐藏功能: 它会智能识别"输出文件"。例如 `! convert ? -resize 50% thumb_? RET` 中 `thumb_?` 的 `?` 会被替换为对应输入文件名,所以 `foo.jpg` 会输出 `thumb_foo.jpg`。这种"批量转换"工作流是 Dired 比 GUI 强的典型例子。

### 1.10 Dired 的 minor modes

Dired 有几个配套 minor modes,各自解决一个具体问题。它们都是"在 Dired buffer 之上加一层行为",体现 Emacs 的可组合性。

`M-x dired-hide-details-mode` — 隐藏权限/owner/size,只看文件名。视图清爽。**这是"信息节制"原则**: 大多数时候你不关心权限,需要时按 `(` 切换显示。

`M-x dired-omit-mode` — 隐藏匹配 `dired-omit-files` 的文件 (默认 dotfiles 和 `#*`/`*#` autosave 文件)。Dired buffer 进一步清爽。按 `M-o` 切换。可以自定义 `dired-omit-files` 来匹配更多模式 (例如 `\\.pyc$"`)。

`M-x image-dired` — 缩略图模式。给图片生成 thumbnail,你可以用方向键浏览。适合管理照片库。

---

## 2. WDired (Writable Dired)

### 2.1 WDired 是什么?

WDired 是 Dired 的"编辑模式"。它解决一个根本性的限制: 普通 Dired 里你不能"直接改"文件名——必须按 `R` 触发询问。这虽然安全,但对于批量改名很笨拙。

WDired 让你**直接编辑 buffer 里的文件名**——就像编辑代码一样。

- `C-x C-q` 进入 wdired
- Dired buffer 变成可编辑
- 你可以直接改文件名 (像编辑代码)
- `C-c C-c` 提交应用
- `C-c C-k` 取消

`C-x C-q` 在普通 buffer 是 `read-only-mode` 的开关 (切换只读)。在 Dired 里它有特殊含义——切换到 WDired。**Emacs 经常这样: 同一键在不同 mode 下有不同含义**,因为 mode 携带了上下文。

WDired 的本质是: 它把 Dired buffer 从"只读视图"变成"可编辑文本",你改完后它解析这些改动,翻译成对应的 `rename-file` 调用。**这是 Dired 哲学的极致——既然文件名是文本,那就直接编辑文本**。

### 2.2 用法

进入 dired 后,按 `C-x C-q` 进入 WDired:
```
C-x d ~/ RET
```

进入 wdired:

```
C-x C-q
```

buffer 标题变成 `(WDired)`,所有文件名可编辑。

进入 WDired 后,buffer 仍然是 `ls` 风格的列表,但文件名那一列变成可编辑文本。其他列 (权限、size 等) 仍然只读——它们是"显示信息",改它们没意义。Emacs 通过 text property 标记哪些字符是文件名,所以你只能在文件名区域编辑。

**现在你可以**做任何文本编辑操作:

- 改文件名: 移到文件名,改字符。例如 `foo.txt` 改成 `foo.md`,提交后 Emacs 实际执行 `rename-file foo.txt foo.md`。
- 改路径 (移动文件): 改名字成 `path/to/newname`。WDired 会执行 `rename-file`,这等价于移动。所以 `foo.txt` 改成 `subdir/foo.txt` 会把文件移到 `subdir/`。
- 删文件: 删整行 (`C-k`)。提交时 Emacs 执行 `delete-file`。**这是个危险操作**——但 WDired 会问确认。
- 用矩形编辑: 选中多行,`C-x r t PREFIX RET` 给所有行加前缀。
- 用 query-replace: `M-% old RET new RET` 批量改名。
- 用 kmacro: 录制一个改名宏,跑 N 次。

提交:

```
C-c C-c    wdired-finish-edit (应用)
C-c C-k    wdired-abort-changes (取消)
C-c ESC    wdired-exit (无操作退出)
```

`C-c C-c` 是 WDired 的"commit"——把所有 buffer 改动翻译成文件系统操作。如果有错误 (例如新路径的目标目录不存在),Emacs 在 `*WDired errors*` buffer 里列出。**老手提交后第一件事就是检查这个 buffer**,确认没有意外。

`C-c C-k` 是"abort"——放弃所有改动,回到普通 Dired。

### 2.3 实战场景: 批量改名

让我们看一个 WDired 的典型场景,理解它为什么比 GUI 强 10 倍。

你有 20 个文件:
```
IMG_001.jpg
IMG_002.jpg
...
IMG_020.jpg
```

想改成:
```
vacation_001.jpg
vacation_002.jpg
...
vacation_020.jpg
```

WDired 方法:
1. 进入 dired,`C-x C-q` 进 wdired
2. 选中所有文件名 (mark + 操作,或直接 `C-x h`)
3. `M-% IMG_ RET vacation_ RET ! `
4. `C-c C-c`

10 秒搞定 20 个文件。

GUI 文件管理器做这个要写脚本 (F2 重命名循环),或者用第三方批量改名工具。WDired 的优势在于: 它**复用你已经会的 Emacs 文本编辑**。你不用学新工具,query-replace 你早就会了,矩形编辑 Module 1 也学了。所有这些技能自然地作用于文件名。

这就是 Emacs 的"统一性"红利的具体体现: 一个技能,多个场景。

### 2.4 WDired 的限制

WDired 只支持**改文件名/移动/删除**。它本质是"编辑文件名文本",所以只能做"对文件名文本的改动能描述的操作"。

不能:
- 创建新文件 (改新行不会创建——文件系统需要内容,空文件名没意义)
- 改文件内容 (那是普通 buffer 的事)
- 改权限 (那些是数字,不是文件名的一部分——用 `M` 在普通 Dired 里做)

这些限制是合理的。WDired 不试图"包办一切",它专注做一件事: 把改名/移动/删除做到极致。

### 2.5 WDired 的创造性用法

WDired 的强大不止于改名。几个意外用法:

1. **批量加序号**: 用 kmacro 给一组文件加序号。`F3 C-a 1 RET M-: (1+ ...) F4` 之类——把 Module 1 的 kmacro 技能直接用到文件名上。

2. **正则改名 + 重组**: `C-M-% \(IMG_\)\([0-9]+\) RET vacation_\2 RET` 把 `IMG_001` 变成 `vacation_001`——捕获组让你做"重排"式改名。

3. **批量加日期前缀**: 配合 `M-!` 插入 shell 命令输出,或者用 Emacs 的 `format-time-string` 在 kmacro 里插入当前日期。

4. **跨目录移动**: WDired 里把文件名改成 `subdir/name` 就等于移动。你可以一次把 20 个文件归类到不同子目录——只要在 WDired 里改路径。

5. **批量去前缀**: 选中所有行,`C-x r k` 删掉每行开头的 N 个字符 (矩形 kill)。一秒去掉 `IMG_` 这种前缀。

这些用法的核心都是: **WDired 让所有 Emacs 文本操作命令作用于文件**。你之前学的所有编辑技能,在 WDired 里都"变成文件操作"。这是其他文件管理器做不到的——它们有自己封闭的 UI 范式。

---

## 3. Dired 进阶: 正则批量

### 3.1 正则标记

正则标记是 Dired 最强大的"选择"机制。

```
%m PATTERN RET    dired-mark-files-regexp
```

标记所有文件名匹配 PATTERN 的。底层是 Emacs 的 `re-search-forward`——既然 Dired 是 buffer,正则搜索就是天然工具。

例:
- `%m \.py$ RET` 标记所有 .py
- `%m ^test_ RET` 标记所有 test_ 开头
- `%m [0-9]\{4\} RET` 标记文件名含 4 数字

正则的力量在于"模式",不是"列表"。GUI 让你 Ctrl-点击每个文件,你只能选中你已经看到的。Dired 让你描述"什么样的文件",它一次选中所有匹配——包括你没注意到的。**这是从"枚举"到"描述"的范式跃迁**,和 SQL 之于过程式编程是一样的革命。

### 3.2 内容标记

```
%g PATTERN RET    dired-mark-files-containing-regexp
```

打开每个文件,标记**内容含** PATTERN 的。这是 grep + 标记的组合。

例: `%g TODO RET` 标记所有含 TODO 的文件。然后你可以 `D` 删除它们,或 `Q TODO RET DONE RET` 把 TODO 改成 DONE。

`%g` 是项目管理的神器——你想知道哪些文件还没完成,一秒标记出来。GUI 完全做不到——它不知道文件内容。

### 3.3 多文件操作

标记后,`C`/`R`/`D` 等对**所有标记**操作。

`Q PATTERN RET REPLACEMENT RET` 在所有标记文件里 query-replace! 这是 Dired 的"杀手级"用法。

```
;; 标记所有 .py
%m \.py$ RET
;; query-replace
Q old_func RET new_func RET
;; (在所有 .py 文件里执行)
```

Emacs 会打开每个标记文件,依次执行 query-replace。你可以 `y`/`n` 每次替换,或者 `!` 全部替换。完成后保存所有文件。

这等于 `sed -i 's/old_func/new_func/g' *.py`——但有两个关键优势: (1) 交互式,你能看到每次替换; (2) Emacs 的正则比 sed 强 (支持 `\(...\)` 捕获、backreference 等)。

**这是项目级重构的 Emacs 方案**,不需要 IDE。配合 Module 5 的 project.el 或 projectile,你可以在整个项目里替换。

---

## 3.4 Dired + Wdired 组合工作流

最强大的工作流是 Dired + WDired 组合。一些例子:

**工作流 1: 整理照片库**
- Dired 进入照片目录
- `%m \.jpg$ RET` 标记所有 jpg
- `t` 翻转 (现在选中非 jpg——通常是无用文件)
- `D x` 删除非 jpg
- `C-x C-q` 进 WDired
- 用 query-replace 或矩形给剩下的 jpg 重命名 (加日期前缀)
- `C-c C-c` 提交

**工作流 2: 项目重构**
- Dired 进入项目
- `%m \.py$ RET` 标记所有 .py
- `Q old_api RET new_api RET` 批量改名
- `C-x C-b` 进 ibuffer
- `* /` 标记所有 modified
- `S` 批量保存

**工作流 3: 清理 backup**
- Dired 进入 `~/.emacs.d/`
- `~` (dired-flag-backup-files) 标记所有 `*~` backup
- 浏览确认
- `x` 清理

这些工作流体现了 Dired 的"组合性"——每个命令做一件事,组合起来解决复杂任务。**Unix 哲学的 Emacs 实践**。

---

## 4. Buffer 管理 (替代 buffers.texi)

### 4.1 为什么 buffer 管理是 Module 2 的话题

学完 Dired 后,你的 buffer 数量会爆炸——每个 Dired 是 buffer,每个打开的文件是 buffer,`*Help*`、`*Messages*`、`*shell*` 都是 buffer。Module 1 你学了基本切换 (`C-x b`),但 10+ buffer 时这不够用。

Emacs 的设计哲学在这里又一次显现: 既然 buffer 列表也是数据,为什么不让它也是 buffer? 这就是 `ibuffer` 的核心思想——和 Dired 同构,但作用于 buffer 而非文件。

**Emacs 的一致性是它的力量来源**: 你学会 Dired 的标记 + 操作两步走,在 ibuffer 里完全适用。学一次,用两处。

### 4.2 Buffer 列表

```
C-x C-b    list-buffers (在另一个 window 显示)
C-x b      switch-to-buffer (切换,默认上一个)
M-x ibuffer  更强的列表
```

`C-x C-b` 默认调 `list-buffers`,显示一个静态的 buffer 列表。你能在里面导航、按 RET 打开,但不能批量操作。这有点鸡肋——它"只是个列表",不是"可操作的视图"。

`M-x ibuffer` 是 `list-buffers` 的"Dired 化"版本——支持标记、过滤、分组、批量操作。**几乎所有 Emacs 老手都把 `C-x C-b` 重绑到 ibuffer**,因为后者完全包含前者的功能且更强。

### 4.3 ibuffer: 比 list-buffers 强

ibuffer 是 dired 风格的 buffer 管理器。

加到 init.el:
```elisp
(global-set-key (kbd "C-x C-b") 'ibuffer)
```

这一行替换之后,你的 `C-x C-b` 进入 ibuffer 而非 list-buffers。**这是 Emacs 用户的"必装配置"**。

ibuffer 里:

| 键 | 作用 |
|---|---|
| `m` | 标记 |
| `u` | 取消标记 |
| `d` | 标记删除 |
| `x` | 执行删除 |
| `RET` | 打开 |
| `o` | 在另一个 window 打开 |
| `S` | 保存 |
| `% f REGEX` | 按名字标记 |
| `% n REGEX` | 按名字标记 (别名) |
| `* /` | 标记所有 modified |
| `* u` | 标记所有未访问 |
| `t` | 翻转标记 |
| `/ n` | filter by name |
| `/ m` | filter by major mode |
| `/ *` | filter by modified |
| `g` | 刷新 |
| `q` | 退出 |
| `h` | 帮助 |

注意这张表和 Dired 的键位惊人地相似——`m`、`u`、`d`、`x`、`t`、`g`、`q` 全部一致。**这就是 Emacs 一致性的具体体现**。学会 Dired 后,ibuffer 几乎不用学。

几个值得强调的:

`% f REGEX` 按名字标记——你可以 `% f \.py RET` 标记所有 Python buffer,然后 `D` 一次关闭。这解决"我开了 20 个 .py buffer 想一起关"这个常见痛点。

`/ n`、`/ m` 是过滤——只显示匹配的 buffer。和"标记"不同,过滤是"视图",被过滤的不显示。`/ /` 取消过滤。

`* u` 标记"未访问"的 buffer——通常用来清理临时 buffer (你打开过但没编辑过的)。

### 4.4 ibuffer 分组

buffer 多了之后,平铺列表难看。分组让 buffer 按类别归类,视觉清爽。

```elisp
(setq ibuffer-saved-filter-groups
      '(("default"
         ("Emacs Config" (filename . ".emacs.d"))
         ("Code" (or (mode . python-mode)
                     (mode . js-mode)))
         ("Org" (mode . org-mode))
         ("Help" (name . "^\\*Help\\*"))
         ("Dired" (mode . dired-mode))
         ("Temp" (name . "^\\*.*\\*$")))))

(add-hook 'ibuffer-mode-hook
          (lambda ()
            (ibuffer-switch-to-saved-filter-groups "default")))
```

这个配置定义了一个名为 "default" 的分组方案。每个分组是一个 `(name . predicate)` 对,predicate 可以基于 mode、filename、name 等。打开 ibuffer 时,buffer 自动归类:

```
[ Emacs Config ]
  init.el                Python  123 line
[ Code ]
  main.py                Python  456 line
  test.py                Python  789 line
[ Org ]
  todo.org               Org      12 line
[ Dired ]
  ~/                     Dired
[ Temp ]
  *scratch*              Lisp Interaction
  *Messages*             Fundamental
```

视觉上清爽很多。每个分组有标题和折叠——你可以 `$` 折叠不关心的组。

分组是 ibuffer 的"杀手锏"——它把"几十个混乱的 buffer"变成"几组分类清单"。**这是 Emacs 用结构对抗混乱的典型例子**。

### 4.5 切换 buffer 的几种方式

Emacs 提供多种 buffer 切换方式,各有适用场景:

```
C-x b NAME RET          切换 (默认上一个)
C-x left/right          上一个/下一个 buffer (历史)
C-x C-b                 ibuffer (列表)
M-x switch-to-buffer-in-project   项目内切
(安装后) projectile / consult / ivy 的 buffer 切换
```

`C-x b` 配合 ido/icomplete/fuzzy (现代 Emacs 默认开) 是日常切换——输入几个字符就 fuzzy 匹配。

`C-x left/right` 按历史顺序切——适合"在两个 buffer 之间反复切"。

`C-x C-b` 进 ibuffer——适合"我不知道我要哪个,先看列表"。

Module 5 会介绍 `consult-buffer`、`projectile-switch-to-buffer` 等更强的工具。但**先掌握原生方式**,理解它们解决的问题,再升级。

### 4.6 关闭 buffer

```
C-x k RET               关闭当前 (k=this-buffer)
C-x k NAME RET          关闭指定
M-x kill-some-buffers   问每个,逐个关
M-x kill-matching-buffers   按名字关一组
```

`C-x k RET` 是日常——关闭当前 buffer。如果 buffer 有未保存改动,Emacs 会问。

`kill-matching-buffers` 危险——它按正则关闭一组 buffer,可能误关。**老手更倾向于用 ibuffer**,因为可视化的标记 + 操作更安全。

ibuffer 里:
- `d` 标记,`x` 批量关
- `* u` (标记未访问), `D` 批量关

这是"标记 + 操作"模式在 buffer 管理上的应用——和 Dired 完全同构。

---

## 5. File Handling (替代 files.texi 节选)

这一节讲 Emacs 怎么处理"文件"这个抽象。Dired 操作"文件名",这些命令操作"文件内容"和"文件元数据"。

### 5.1 find-file 的细节

`C-x C-f FILE RET` 打开文件。这是 Emacs 最古老的命令之一,但它的能力远不止"打开本地文件"。

变种:
- `C-x C-f /sudo::/etc/passwd RET` 用 tramp 以 root 打开
- `C-x C-f /ssh:user@host:/path RET` 用 tramp 远程打开
- `C-x C-f /ftp:...` FTP
- `C-x C-f FILE ~5~ RET` 打开某个 backup (Emacs 自动 backup)
- `C-x C-f #FILE# RET` 打开 autosave

`/sudo::` 是 **TRAMP** 的特性,Module 5 简略涉及。TRAMP 让 Emacs 像"本地编辑"一样编辑远程文件——你打开 `/ssh:host:/path/to/file`,Emacs 在后台用 ssh 拉取/保存。**这是 Emacs 的"透明远程"哲学**: 你不需要 FTP 客户端,文件路径里有协议就行。

打开 backup (`FILE ~5~`) 是 Emacs 数据安全的体现——每次保存都备份,你能回到任何版本。下面 §5.3 详细讲。

### 5.2 自动保存 (auto-save)

Emacs 自动每隔 N 个字符或空闲 N 秒保存到 `#FILE#`:

```elisp
(setq auto-save-default t           ; 默认开
      auto-save-timeout 30           ; 30 秒空闲
      auto-save-interval 300)        ; 或 300 字符
```

这两个触发条件是"或"关系——任何一个满足就保存。`auto-save-timeout` (空闲) 防止"打字时崩溃丢失",`auto-save-interval` (字符数) 防止"长时间高强度打字不空闲"。

注意 `#FILE#` 和原文件不同——它是临时副本,文件名带 `#`。这样设计的好处是: (1) 原文件不被污染; (2) 崩溃后你能 `M-x recover-this-file` 恢复。

崩溃后,`M-x recover-this-file RET` 或 `M-x recover-file RET` 恢复。Emacs 会比对原文件和 `#FILE#` 的时间戳,提示你恢复哪个。

**新手常嫌 `#FILE#` 文件烦**,但这是 Emacs 的安全网——一旦你因为崩溃丢失过一次重要数据,你就会感激它。Module 0 为了清爽关了它,Geek 应该重新开。

### 5.3 备份 (backup)

默认 Emacs 每次保存创建 `FILE~`:

```elisp
(setq make-backup-files t            ; 默认开
      version-control t              ; 多版本 backup
      kept-new-versions 5
      kept-old-versions 2
      backup-by-copying t)           ; 不破坏 hard link
```

效果: `foo.txt.~1~`、`foo.txt.~2~`... 保留多版本。

`version-control t` 是关键——它让 Emacs 保留多个 backup,而不是只一个。每次保存都生成新版本,自动清理超过 `kept-new-versions` 的旧版本。这意味着你可以回到任何一次保存前的状态——**这是 Emacs 内置的版本控制**。

`backup-by-copying t` 解决一个微妙问题: 默认 backup 用 rename (原子操作,快),但这会破坏 hard link。如果你的文件被 hard link 到多处,bakeup 会"切断"链接。`backup-by-copying` 改用 copy (慢但安全)。**生产环境推荐开**。

Module 0 的最小 init.el 关了 backup,因为新手烦 `~` 文件。但 Geek 应该开 backup + version-control,数据安全。**这是从新手到 Geek 的态度转变**: 接受一些"烦"以换取安全。

### 5.4 文件锁

Emacs 编辑文件时,创建 `.#FILE` 锁文件 (symlink 指向 user@host.pid)。防止两人同时编辑。

```elisp
(setq create-lockfiles t)    ; 默认开
```

锁文件是合作编辑的安全网——如果你的同事也用 Emacs 编辑同一文件,他会看到"文件被锁定"。新手烦 `.#[random]` 文件,可关。但生产环境推荐开,尤其是在共享文件系统上。

### 5.5 大文件警告

打开超大文件 (>100MB),Emacs 会问:

```
File foo.bin is large (100MB), really open? (y or n)
```

避免 Emacs 卡死。

为什么 Emacs 对大文件敏感? 因为 Emacs 的内部数据结构是"间隙缓冲"(gap buffer),适合频繁小改动,不适合大文件的随机访问。100MB 的文本会让光标移动变慢,搜索变慢,内存占用大。

```elisp
(setq large-file-warning-threshold 100000000)  ; 100MB
```

或用 `M-x vlf` (Very Large File) 包处理超大文件——它分页加载,不是全读进内存。**这是 Emacs 应对"非典型场景"的方式**: 默认假设普通文件,大文件用专门工具。

### 5.6 Auto-revert

文件被外部改了,Emacs 自动 reload:

```elisp
(global-auto-revert-mode 1)
```

`dired-mode` 也自动 revert (`dired-auto-revert-buffer`)。

这个功能解决"外部工具改了文件,Emacs 还显示旧的"的问题。比如你用 shell 跑 `git checkout`,Emacs 里的文件应该更新——auto-revert 自动做这件事。

`global-auto-revert-mode` 还会 revert dired——你在 shell 里 `touch newfile`,切回 Emacs 的 dired buffer,新文件自动出现。

---

## 6. 高级 Dired 技巧

### 6.1 在 dired 里跑命令

`!` shell 命令 (同步):
```
! convert ? -resize 50% thumb_? RET
```

`&` 异步:
```
& xdg-open ? & RET
```

`%` 对每个标记文件跑命令:
```
% !  (或没特殊,就 !)
```

同步 vs 异步的选择: 短命令 (< 1 秒) 用 `!`,长命令 (视频转换、大文件下载) 用 `&`。`&` 让命令在后台跑,你可以继续编辑。

Dired 的 shell 命令里 `?` 是"文件名"占位符。多个标记文件时,每个文件单独跑一次命令。如果你想把所有文件作为一个命令的参数 (例如 `tar czf out.tgz file1 file2 ...`),用 `*` 代替 `?`。

### 6.2 预览 (peep-dired)

`peep-dired` 包让你在 dired 上下/左右预览图片/文件:

```bash
M-x package-install RET peep-dired RET
```

加到 init.el:
```elisp
(add-hook 'dired-mode-hook #'peep-dired-hook)
```

进入 peep-dired 后,上下键浏览时旁边实时显示文件预览。对于图片库管理特别有用——比 Finder 的 Quick Look 还快,因为不需要鼠标。

### 6.3 dirvish

`dirvish` 是现代 Dired 增强:
- 图片预览侧栏
- 现代 UI
- 兼容 dired 命令

```bash
M-x package-install RET dirvish RET
```

Module 5 推荐用。dirvish 在不破坏 Dired 命令兼容性的前提下,加上现代 UI——比如选中文件时旁边显示缩略图。**先掌握原生 Dired 再升级 dirvish**,否则你会被花哨功能分心。

### 6.4 dired-x

dired-x 是内置扩展:
- `M-x dired-jump` (`C-x C-j`): 跳到当前 buffer 的 dired
- `M-x dired-virtual`: 虚拟 dired (基于任意文件列表)
- `M-x dired-omit-mode`: 隐藏文件

启用:
```elisp
(require 'dired-x)
```

`dired-jump` (`C-x C-j`) 是高频命令——你在编辑 `~/foo/bar.py`,按 `C-x C-j` 跳到 `~/foo/` 的 dired,光标停在 `bar.py` 上。**这是"文件 → 目录"的快速反向导航**,比 GUI 的"在文件管理器中显示"快得多。

`dired-virtual` 是隐藏宝藏——它能基于任意"文件列表"创建虚拟 dired。例如你可以 `find . -name '*.py'` 输出后,把这个列表当作 dired 操作。等于"搜索结果即文件管理器"。

### 6.5 wdired 配合 kmacro

WDired 是普通 buffer,可以用 kmacro:

```
C-x C-q
F3
M-s f file_\([0-9]+\).jpg RET
M-1 file_\1.jpg
... 编辑 ...
F4
C-c C-c
```

(Module 1 学的 kmacro 在 wdired 里能用)

这是 Emacs 一致性的极致——kmacro 录的是"按键序列",和 buffer 的 mode 无关。所以 Module 1 学的 kmacro 在 WDired 里完全适用。**你的肌肉记忆是通用的**。

---

## 6.6 更多创造性用法 (重点)

Dired 的"意外用法"远不止上面列的。下面 10 个场景展示 Dired 的灵活:

1. **作为项目导航**: `i` 插入所有子目录,一个 buffer 看整个项目。比 treemacs 轻,无侧栏占用。

2. **作为版本对比**: 开两个 Dired (左右),`=` diff 同名文件。比 `git diff` 直观,适合对比两份代码。

3. **作为 grep 替代**: `%g TODO RET` 标记所有含 TODO 的文件,然后 `A` 在所有标记文件里搜索——这等于 grep + Dired 可视化。

4. **作为图片批处理器**: 标记所有 .jpg,`! convert ? -resize 50% thumb_? RET`,一秒生成所有缩略图。

5. **作为代码重构工具**: `%m \.py$ RET` + `Q old_name RET new_name RET`,跨整个项目重命名函数。

6. **作为 chmod 批处理器**: 标记所有 .sh,`M 755 RET`,批量改权限。比 shell 的 `find ... -exec chmod` 直观。

7. **作为 backup 清理器**: `~` 标记所有 `FILE~` backup,`x` 清理。Emacs 的 backup 系统配合 Dired 是天然工作流。

8. **作为快捷方式管理器**: `S` (symlink) 给常用文件创建快捷方式。在 Dired 里管理 symlink 比图形界面直观。

9. **作为 tar/zip 处理器**: 标记文件,`! tar czf out.tgz * RET`。压缩 / 解压全在 Dired 里。

10. **作为照片整理**: `peep-dired` 浏览图片,边看边改名/移动。比 Finder 的照片管理顺畅。

这些用法的共同点: **Dired 不限制你能做什么**。任何能"操作文件名/文件"的任务,都可以在 Dired 里完成。这是 Emacs 一切皆 buffer 哲学的具体回报。

---

## 7. 这个锚点对照到 Elisp

每个交互式命令背后都有一个 Elisp 函数。理解这个映射是 Module 3 (Elisp) 的准备——你会看到,所有 Dired 操作都是函数调用,你可以在自己的代码里复用。

| Editor 概念 | Elisp |
|---|---|
| 打开文件 | `(find-file FILE)` |
| 保存 | `(save-buffer)` |
| 关闭 buffer | `(kill-buffer BUFFER)` |
| 切换 buffer | `(set-buffer BUFFER)` / `(switch-to-buffer BUFFER)` |
| Dired | `(dired DIR)` |
| WDired | `(wdired-change-to-wdired-mode)` |
| 标记 | `(dired-mark)`、`dired-mark-files-regexp` |
| ibuffer | `(ibuffer)` |

Module 3+ 会用这些函数写自己的工具。例如,你可以写一个函数: "在当前 Dired 里标记所有大于 1MB 的文件"——核心就是 `(dired-mark-files-regexp ...)` 加上文件大小检查。

理解了这个映射,你就理解了 Emacs 的"可编程性": **所有 UI 操作都是函数,所有函数都可以从代码调用**。这意味着你能自动化任何重复工作流。Module 2 学的命令,在 Module 3 会变成代码构建块。

---

## 8. 常见陷阱 (反例汇总)

学习 Dired 时新手最常踩的坑:

**陷阱 1**: 以为 `d` 就是删除。错——`d` 只是**标记** `D`,按 `x` 才执行。新手看到 `D` 字符就慌,其实只要不按 `x`,什么都不会发生。

**陷阱 2**: WDired 提交后部分改名失败。`*WDired errors*` buffer 里会列出哪些失败——通常是目标路径的目录不存在。改完 `C-c C-c` 后看一眼这个 buffer。

**陷阱 3**: `R` 不是"rename"是"move"。文件名带路径就等于移动。`R foo.txt RET /tmp/foo.txt RET` 把文件移到 /tmp。

**陷阱 4**: Dired 不显示 dotfiles。`dired-listing-switches` 要含 `a`。找不到 `.config` 是这个原因。

**陷阱 5**: `!` 里的 `?` 是文件名占位符,不是正则。`$1` 是错的,正确是 `?`。

**陷阱 6**: ibuffer 默认按访问时间排序,新手以为顺序乱了。按 `,` 切换排序方式。

**陷阱 7**: 多个 Dired 进入子目录时,buffer 名重复。Emacs 会用 `dir<2>` 区分,但容易混。启用 `dired-find-alternate-file` 让一个 buffer 复用。

**陷阱 8**: `C-x C-q` 在普通 buffer 是只读切换,在 Dired 是 WDired 切换。新手有时在普通 buffer 按 `C-x C-q` 以为进 WDired。

**陷阱 9**: 标记文件后按 `g` (revert) 会丢失标记。`g` 重新生成 buffer,所有标记消失。**需要重新操作前先取消 revert,或者用 `C-u g`**。

**陷阱 10**: `Q` (query-replace in files) 会修改文件但默认不保存。改完要在 ibuffer 里 `* /` (modified) + `S` 批量保存。

---

## 9. 自测

1. Dired buffer 和普通 buffer 的本质区别?
2. WDired 和 Dired 的关系?
3. `%m \.py$ RET` 干啥?
4. 怎么标记所有内容含 "TODO" 的文件?
5. `C-x C-q` 在 dired 里干啥?
6. ibuffer 比 list-buffers 强在哪?
7. `make-backup-files` 默认值?
8. `C-x C-f /sudo::/etc/ RET` 干啥?

**答案**:
> 1. Dired buffer 内容是**目录列表**,从文件系统生成;普通 buffer 是文本
> 2. WDired 是 dired 的可编辑模式;`C-x C-q` 切换
> 3. 标记所有 .py 文件
> 4. `%g TODO RET` (dired-mark-files-containing-regexp)
> 5. 进入 wdired (可编辑)
> 6. 标记、分组、过滤、批量操作;list-buffers 只是静态列表
> 7. t (开)
> 8. 用 TRAMP 以 root 权限打开 /etc/ 目录

---

## 10. 下一步

进入 `02-dired/drills.md` 做练习。

Drills 是 40 题渐进式实操,把上面讲的概念变成肌肉记忆。每题都有"为什么这样做"的解释,不只是"按这个键"。
