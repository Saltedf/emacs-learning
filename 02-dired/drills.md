# Drills: Dired + Buffer 管理 (40 题)

> **预计用时**: 4 小时
> **替代**: `emacs-manual/dired.texi` + `buffers.texi` 的练习
> **目标**: 10 分钟内整理一个混乱目录

这一份练习不只是"按键清单"。每道题前面都有"为什么"——解释这个操作解决了什么问题,内部发生了什么。**目标是建立直觉,不是肌肉记忆**。肌肉记忆是结果,直觉才是地基。

建议节奏: 第一组 30 分钟后停一下,合上电脑,回想"我刚才做了什么? 为什么这样做能起作用?"。如果说不清,回去重做。这就是从"会用"到"理解"的转化。

---

## 第一组: Dired 基础 (10 题,30 分钟)

### 准备

练习 Dired 需要一个"试验场"——可以乱改、可以删、不心疼的目录。下面这段 shell 命令造一个有 10+ 文件的小目录:

```bash
mkdir -p ~/dired-practice
cd ~/dired-practice
echo "content 1" > file1.txt
echo "content 2" > file2.txt
echo "content 3" > file3.txt
echo "hello" > a.py
echo "world" > b.py
echo "test" > test_a.py
echo "old backup" > backup.bak
echo "another old" > old_file.bak
mkdir sub1 sub2
echo "in sub1" > sub1/x.txt
echo "in sub2" > sub2/y.txt
```

这个目录刻意混合不同类型 (txt/py/bak) 和结构 (扁平 + 子目录),让后续练习有素材。**真实世界的目录比这复杂 100 倍,但练习的"模式"是一样的**——一旦你能熟练处理这个小目录,大目录只是规模问题。

### 题 1.1: 进入 dired

```
C-x d ~/dired-practice RET
```

应该看到所有文件。

`C-x d` 是 `dired` 命令的快捷键。它问你"哪个目录",默认是当前 buffer 的目录 (如果是文件 buffer) 或 home。你可以输任意路径——`/etc/`、`/tmp/`、`~/.emacs.d/`。

看到的内容是 `ls -alh` 的输出。注意 buffer 名就是目录路径 (`~/dired-practice/`)。这强化了"Dired 是普通 buffer"的概念——它没有特殊 UI,就是 buffer。

### 题 1.2: 导航

- `n` 或 `C-n`: 下一行
- `p` 或 `C-p`: 上一行
- `SPC`: 下一行 (移动 + 显示)
- 移到 `file1.txt`,按 `RET` 打开

导航在 Dired 里和普通 buffer 完全一样——因为**它就是普通 buffer**。`n`/`p` 是 Dired 给的更短键,但你完全可以用 `C-n`/`C-p`。**Emacs 不强制你学新键**,它复用你已经会的。

`RET` 打开文件——它调用 `dired-find-file`,提取当前行文件名,然后 `find-file`。如果当前行是目录,它会进入子目录的 Dired。

试一下: 移到 `sub1/`,按 `RET`,进入子目录的 Dired。再按 `^` 回到上级。

### 题 1.3: 上一级目录

- `^` 跳到 `~/`
- 再 `RET` 回 `~/dired-practice/`

`^` 是 Dired 专属的"返回上级"键。在 GUI 里你点"向上"按钮,在 Dired 里按一个字符。它实际调 `dired-up-directory`。

注意: `^` 默认开一个新 buffer 显示上级目录。这意味着你多次 `^` 后会有多个 Dired buffer——这通常不想要。解决: 在 init.el 里启用 `dired-find-alternate-file` (见 README 配置),让 `^` 复用 buffer。

### 题 1.4: 在另一个 window 打开

- 移到 `file1.txt`
- `o` 在另一个 window 打开
- 光标跳到新 window

`o` (other-window) 是高频操作——它 split window 然后在另一边打开文件。这构建了 Dired 的"双窗口工作流": 一边 Dired 浏览,另一边打开文件。

配合 `dired-dwim-target`,你的 `C`/`R` 操作会自动以另一 window 的 Dired 作为目标——分屏两边的 Dired 让复制/移动变成"按 C,回车"两步。这是 Dired 杀手级工作流,Module 2 的核心福利。

### 题 1.5: 只读查看

- 移到 `file2.txt`
- `v` 进入 view-mode (只读)
- `q` 退出 view

`v` 是 view-mode——只读打开文件,按 `q` 退出 (不 kill buffer,只是隐藏)。适合"我只想看一眼不想编辑"的场景。

view-mode 比 `f` (普通打开) + `C-x C-q` (转只读) 更轻量——它不会改 buffer 的 modified 状态,退出后不留痕迹。

### 题 1.6: 标记单个

- 移到 `a.py`
- `m` 标记 (前面加 `*`)
- 移到 `b.py`,`m` 标记
- 两个都有 `*`

`m` 标记当前行——在行首加字符 `*`。这是 Dired 的核心机制: **标记就是 buffer 里的字符**。

标记后你能一眼看到选中状态。GUI 文件管理器的"选中"只是高亮,Dired 的标记是显式字符——你能看出"是 `*` (通用)、`D` (删除)、`C` (复制)"等不同意图。**这是可视化优势**。

### 题 1.7: 取消标记

- `u` 取消当前
- `U` 取消所有

`u` 取消当前行的标记。`U` 取消 buffer 里所有标记——批量"重置"。

这是 Dired 的"安全网"——标记错了随时撤销。GUI 的 Ctrl-点击选中错了,要重新点;Dired 里 `U` 一下全部清空,重新开始。

### 题 1.8: 翻转标记

- 标记 `a.py` 和 `b.py`
- `t` 翻转: 现在所有其他文件被标记,这两个取消
- `U` 全部取消

`t` (toggle) 是"反选"——把标记和未标记互换。这看起来奇怪,其实非常有用: 你想"选中除了 .py 以外的所有",就 `%m \.py$` + `t`。**反选是高级用法的常见模式**。

### 题 1.9: 复制

- 移到 `file1.txt`
- `C` 复制
- 输入 `copy_of_file1.txt RET`

`C` (dired-do-copy) 复制文件。它会问你目标路径,默认填当前目录 + 原文件名。你只需要改名字部分。

注意: 如果有标记,`C` 会对所有标记文件操作——它问目标目录,然后把所有标记文件复制过去。这是批量复制的核心。

底层: `C` 调 `copy-file`。如果跨文件系统,自动 fallback 到 byte-copy。如果目标已存在,默认问覆盖。**Emacs 的设计是周全的**。

### 题 1.10: 重命名

- 移到 `a.py`
- `R` 改名
- 输入 `alpha.py RET`

`R` (dired-do-rename) 重命名。它实际是"移动"——可以跨目录、跨文件系统。`R a.py RET /tmp/alpha.py RET` 把文件移到 /tmp。

底层: `R` 调 `rename-file`。Unix 的 `rename` 在文件系统内是原子的,跨文件系统 Emacs 会自动用 copy+delete 模拟。

**新手常以为 R 只能改文件名**——其实它是个完整的 move 命令。理解了这点你就能用 `R` 做文件归类: 标记多个文件,`R` 到一个目录,所有文件被移过去。

---

## 第二组: Dired 标记高级 (10 题,40 分钟)

这一组进入 Dired 的"超能力区"。前面学了手动 `m` 标记,这里学"按模式批量标记"——这才是 Dired 真正强大的地方。GUI 文件管理器做不到的事,这里全都能做。

### 题 2.1: 标记扩展名

- `* .` 然后输入 `py RET`
- 所有 .py 被标记

`* .` (dired-mark-extension) 标记某扩展名的所有文件。它问你扩展名,然后扫描 buffer 匹配。

这是"按类型选择"的快捷方式。`%m \.py$` 也能做,但 `* .` 更短。**Emacs 给高频操作短键**。

### 题 2.2: 标记目录

- `* /` 标记所有目录
- `sub1`、`sub2` 应该有 `*`

`* /` 标记所有目录。类似的还有 `* *` (可执行)、`* @` (symlink)。这些"按属性标记"覆盖了常见选择需求。

试一下: 标记所有目录,然后 `R` (rename/move) 把它们移到别处——批量重组目录结构。

### 题 2.3: 正则标记

- `%m \.txt$ RET` 标记所有 .txt

`%m` 是最强大的标记命令——用正则。`%m ^test_` 标记所有 test_ 开头,`%m [0-9]\{4\}` 标记含 4 位数字。

正则的力量在于"描述模式"而非"列举"。你不需要知道有哪些文件,只需要描述"我要什么样的"。**这是从枚举到描述的范式跃迁**。

### 题 2.4: 内容标记

- `%g hello RET` 标记内容含 "hello" 的文件
- `a.py` 应该被标记

`%g` (dired-mark-files-containing-regexp) 是 grep + 标记。它打开每个文件搜内容,匹配的标记。

这是 Dired 的"内容感知"能力——它不只看文件名,还看文件内容。`%g TODO RET` 标记所有含 TODO 的文件,`%g FIXME RET` 标记所有待修复的代码。**这是项目管理神器**。

### 题 2.5: 批量复制

- 标记所有 .py (`%m \.py$ RET`)
- 按 `C` (copy)
- 输入 `sub1/ RET` (复制到 sub1 目录)
- 检查 `sub1/` 应该有所有 .py 的副本

这就是 Dired 批量操作的核心: 标记一组文件,一次操作。GUI 要 Ctrl-点击逐个选,Dired 用正则一秒标记几十个。

注意 `C` 问的是"目标目录" (不是单个目标文件)——因为有多个源文件,默认每个保持原名复制到目标目录。

### 题 2.6: 批量删除

- 标记所有 .bak (`%m \.bak$ RET`)
- 按 `D` (标记删除,加 D 字符)
- 按 `x` 执行
- 确认每个 (按 y 或 n)

注意两步: `D` 只是标记 (加 `D` 字符),`x` 才真删。**这是 Dired 的安全网**——你可以堆积一堆 `D`,审视后再 `x`。

`x` 会问每个文件"删除吗?",你可以逐个 y/n。设 `(setq dired-deletion-confirms nil)` 跳过单个确认 (老手做法)。

### 题 2.7: 操作的预览 + 取消

- `d` 标记某个文件 (加 D)
- `u` 取消
- `x` 执行剩余标记

这道题强调"标记 → 预览 → 执行"的模式。标记后你能看到所有 `D` 字符,审视后决定执行还是取消。**这是 GUI 没有的安全层级**。

练习时故意标错几个文件,然后 `u` 取消——体会"撤销"的顺畅。这种"操作可撤销"的安全感是 Dired 的设计哲学。

### 题 2.8: 创建目录

- `+ RET newdir RET` 创建 `newdir/`

`+` (dired-create-directory) 创建目录。可以一次创建多层: `+ a/b/c RET` 创建嵌套目录 (底层 `mkdir -p`)。

GUI 里创建目录要右键 → 新建文件夹 → 输名字,多步。Dired 里 `+` + 名字 = 两步。**键盘工作流的优势累积**。

### 题 2.9: 改权限

- `M` 改权限
- 输入 `644 RET`
- 文件权限变成 `-rw-r--r--`

`M` (dired-do-chmod) 改权限。底层调 `set-file-modes`。输入是 octal 数字 (644 = rw-r--r--)。

Dired 的 `M` 比 shell 的 `chmod` 直观——你能立即看到 buffer 里的权限列变化。批量 chmod 也很容易: 标记一组,`M 755 RET`,所有标记文件同时改。

### 题 2.10: 删除空目录

- `R` 把 `newdir` 改名到别处
- 或 `D` 删除

`D` 在目录上会问"递归删除吗?"——除非 `dired-recursive-deletes` 设为 `always`。空目录可以直接删,非空目录需要确认递归。

这道题练习"目录操作"——前面都是文件,目录有些特殊 (递归、子文件)。

---

## 第三组: 子目录递归 (5 题,20 分钟)

这一组学 Dired 的"子目录插入"——一个 buffer 看多个目录。这是 GUI 完全做不到的能力,Dired 的差异化优势。

### 题 3.1: 插入子目录

- 在 dired 里,移到 `sub1/`
- `i` 插入 (dired 显示 sub1 内容)
- 同样插 `sub2/`
- 现在 dired 一次显示三个目录

`i` (dired-maybe-insert-subdir) 把子目录的内容"插入"当前 Dired buffer。这是"多目录视图"。

为什么有用? 跨目录批量操作。比如 `tests/` 和 `docs/` 都有 .bak,你想一起删——`i` 插入两个子目录,`%m \.bak$` 标记所有 (跨目录),`D x` 一次完成。

GUI 做这个要开两个窗口,各自选中,各自删。Dired 一个 buffer 全搞定。

### 题 3.2: 折叠子目录

- `$` 折叠 sub1
- 再 `$` 展开
- `M-$` 折叠所有

`$` 折叠光标处的子目录——只显示标题行 (`sub1/`),内容隐藏。再 `$` 展开。

`M-$` 折叠所有插入的子目录——一键简化视图。

这是"信息分层"——你只看当前关心的部分,其余折叠。**Emacs 重视视图管理**,因为 buffer 内容多了会乱。

### 题 3.3: 跳到子目录头

- 移到 sub1 内部
- `^` 回到 `sub1/` 行
- (或 `M-j` 跳到任意插入的子目录)

当 buffer 里有多个插入的子目录时,跳转很重要。`^` 回到当前子目录的标题行,`M-j` 选任意插入的子目录跳过去。

### 题 3.4: 递归操作

- `dired-recursive-copies` 设为 `always`
- 标记 `sub1/`
- `C` 复制
- 输入 `sub2/ RET`
- sub1 整个目录被复制到 sub2 里

递归操作: 标记一个目录,`C` 会复制整个目录树。`dired-recursive-copies` 控制是否递归——`always` 总是递归,`top` 只在顶层问。

类似地,`D` 可以递归删除目录 (受 `dired-recursive-deletes` 控制)。**这是处理目录树的核心机制**。

### 题 3.5: 移除子目录插入

- 移到 `sub1/` 行
- `k` (kill-line 在 dired 里 = 移除该子目录的显示)
- 注意: 文件没删,只是 dired 不再显示

`k` 不是删除文件,是"从 Dired 视图移除该子目录的插入"。文件还在文件系统里,只是 buffer 不显示。

`g` (revert) 会重新生成 buffer,所有插入恢复默认。所以 `k` 是临时的视图调整。

新手常误以为 `k` 删了文件——其实是 buffer 操作,不是文件操作。**理解这一点很重要**: Dired 的很多命令是 buffer 操作,不直接对应文件系统变化。

---

## 第四组: WDired (10 题,30 分钟)

WDired 是 Module 2 的"顿悟时刻"。一旦你理解"文件名就是文本",所有 Emacs 文本编辑技能突然都作用于文件操作。这一组让你建立这个直觉。

### 题 4.1: 进入 wdired

```
C-x C-q
```

buffer 标题变成 WDired。

`C-x C-q` 在普通 buffer 是 `read-only-mode` 切换。在 Dired 里,它特殊——切换到 WDired mode。**同一键,不同 mode 有不同含义**,这是 Emacs 的"上下文敏感"传统。

进入 WDired 后,buffer 标题显示 `(WDired)`,文件名变成可编辑文本。其他列 (权限/size) 仍然只读——Emacs 通过 text property 标记可编辑区域。

### 题 4.2: 改一个文件名

- 移到 `alpha.py` (假设你之前改过)
- 用 `C-f`/`C-b` 等编辑,改成 `renamed.py`
- `C-c C-c` 提交
- 检查文件系统,文件被改名

这道题建立基本直觉: WDired 里改文本 = 改文件名。改完 `C-c C-c` 提交,Emacs 执行 `rename-file`。

试一下改一个不存在的路径: 把 `renamed.py` 改成 `nonexistent_dir/foo.py`,`C-c C-c` 会报错——目标目录不存在。`*WDired errors*` buffer 里会列出。**这是 WDired 的反馈机制**。

### 题 4.3: 批量改名

- 重新 `C-x C-q`
- 选中所有文件名 (`C-x h`)
- `M-% file RET item RET` 把 "file" 全改成 "item"
- `!` 替换所有
- `C-c C-c`

现在 file1.txt → item1.txt 等。

这道题展示 WDired 的杀手锏: **query-replace 作用于文件名**。GUI 文件管理器没有这个——你要写脚本或用第三方工具。WDired 让 Module 1 学的 query-replace 直接复用于文件操作。

`C-x h` (mark-whole-buffer) 选中整个 buffer。然后 `M-%` 限定在 region 内执行——只替换选中部分。这是 query-replace 的"region 模式"。

### 题 4.4: 用矩形编辑加前缀

- `C-x C-q`
- 选中所有 .py 文件的行首 (用 `C-SPC` + 移动)
- `C-x r t backup_ RET`
- 所有 .py 前加 `backup_`
- `C-c C-c`
- 文件被改名

矩形编辑 (`C-x r t` = string-rectangle) 在 WDired 里给文件名加前缀。选中多行的开头几列,`C-x r t PREFIX RET` 给每行插入 PREFIX。

这是 Module 1 学的矩形编辑的"文件版"——你之前用它给代码加注释,现在用它给文件加前缀。**同一技能,多个场景**。

### 题 4.5: 删除文件

- `C-x C-q`
- 移到某行,`C-k` 删整行
- `C-c C-c`
- 文件被删

WDired 里删整行 = 删文件。`C-k` (kill-line) 在 WDired 里有特殊语义——它不只是删文本,提交时 Emacs 会 `delete-file` 对应的文件。

**这是危险操作**——但 WDired 会问确认。新手练习时多检查,养成提交前审视的习惯。

### 题 4.6: 移动文件

- `C-x C-q`
- 移到 `item1.txt`,改名 `sub1/item1.txt`
- `C-c C-c`
- 文件被移到 sub1/

WDired 里改文件名为 `path/name` 等于移动。底层是 `rename-file`,它跨目录就移动。

这意味着你可以一次"归类"多个文件——WDired 里改路径,提交后所有文件被移到对应目录。**这是整理目录的高效方式**。

### 题 4.7: kmacro 录制改名

- `C-x C-q`
- 录一个宏: 找下一个 IMG_,改成 PIC_
- 跑 10 次
- `C-c C-c`

WDired 里 kmacro 完全适用——它录的是按键序列,和 buffer mode 无关。Module 1 学的 kmacro,在 WDired 里变成"文件操作录制"。

练习: 录一个宏"找下一个 IMG_,query-replace 成 PIC_",跑 10 次。这是 GUI 完全做不到的——批量改名脚本化但可视化。

### 题 4.8: 取消

- 改几个文件名
- `C-c C-k` 取消
- 改动不应用

`C-c C-k` (wdired-abort-changes) 放弃所有改动。和 `C-c C-c` 相反。

新手改错了就 `C-c C-k`,从头来。这是 WDired 的"安全出口"——任何改动都可以放弃。

### 题 4.9: query-replace-regexp

- `C-x C-q`
- `C-M-% \(item\)\([0-9]+\) RET \2_\1 RET` 把 "item1" 变 "1_item"
- `C-c C-c`

`C-M-%` 是 query-replace-regexp——用正则替换。捕获组 `\1`、`\2` 让你"重排"文件名。

`item1` 变 `1_item` 是"重排"——`\2` (数字) 在前,`\1` (item) 在后,中间加下划线。**这是改名的高级技巧**,GUI 完全做不到。

### 题 4.10: 整理目录

打开 ~/Downloads (假设很多杂乱文件):
- `C-x C-q`
- 用各种编辑技巧归类:
  - 所有 .jpg 改名 `Photos/name`
  - 所有 .pdf 改名 `Docs/name`
  - 所有 .py 改名 `Code/name`
- `C-c C-c`
- 文件被归类

这是 WDired 的"毕业题"——综合运用前面的技能。改路径 = 移动,query-replace = 批量改名,kmacro = 自动化。

练习时给自己限时 2 分钟。能做到就毕业 WDired。

---

## 第五组: Shell 命令 (5 题,15 分钟)

`!` 是 Dired 的"逃生舱"——任何 Dired 没内置的操作,你都可以用 shell 命令完成。这一组练 `!` 和 `&`。

### 题 5.1: 简单 shell

- 标记 `item1.txt`
- `!` shell command
- 输入 `wc -l ? RET` (`?` 是文件名)
- 看输出

`!` (dired-do-shell-command) 跑 shell 命令,`?` 替换为文件名。如果有多个标记,每个文件单独跑一次。

输出显示在 `*Shell Command Output*` buffer。这是"shell 集成"——你在 Dired 里调任何 Unix 工具。

### 题 5.2: 异步

- 标记一个文件
- `&` async shell
- 输入 `sleep 5; echo done & RET`

`&` (dired-do-async-shell-command) 异步跑——不阻塞编辑。长命令 (视频转换、下载) 必须用 `&`,否则 Emacs 卡住。

输出在 `*Async Shell Command*` buffer,完成时显示。**这是 Emacs 长任务的标准处理**: 后台跑,前台继续编辑。

### 题 5.3: 批量 shell

- 标记所有 .py
- `! wc -l RET`
- 输出每个文件的行数

多个标记时,`?` 被替换多次——每个文件一次 `wc -l`。输出汇总在 `*Shell Command Output*`。

这等于 shell 的 `wc -l *.py`,但通过 Dired 你能可视化选中文件,而不只是 glob。

### 题 5.4: 转换图片 (要 ImageMagick)

- 标记一个 .jpg
- `! convert ? -resize 50% thumb_? RET`
- 缩略图生成

`! convert ? -resize 50% thumb_? RET` 中两个 `?` 都被替换为文件名——`foo.jpg` 变成 `convert foo.jpg -resize 50% thumb_foo.jpg`。这是"批量转换"的标准模式。

GUI 文件管理器没有这个——你要写 shell 脚本或用图片管理软件。Dired + ImageMagick 是 Unix 哲学的极致组合。

### 题 5.5: gzip

- 标记一个大文件
- `Z` (compress)
- 文件变 .gz
- `Z` 再解压

`Z` (dired-do-compress) 智能压缩/解压——根据扩展名判断。`.gz` 文件按 `Z` 解压,普通文件按 `Z` 压缩成 `.gz`。

底层调 `compress-file`/`uncompress-file`,支持 gzip、bzip2、xz 等。**Dired 不重新发明压缩,它复用系统工具**。

---

## 第六组: ibuffer (10 题,30 分钟)

学完 Dired 后,你的 buffer 数量爆炸。ibuffer 是"Dired for buffers"——同样的标记 + 操作模式,作用于 buffer 而非文件。

### 准备

确保 init.el 有:
```elisp
(global-set-key (kbd "C-x C-b") 'ibuffer)
```

这一行替换默认的 `list-buffers`。**所有 Emacs 老手都这么做**。

打开几个 buffer:
```
C-x C-f ~/dired-practice/item1.txt RET
C-x C-f ~/dired-practice/item2.txt RET
C-x C-f ~/.emacs.d/init.el RET
M-x *scratch*
```

这些 buffer 会出现在 ibuffer 列表里,供后续练习。

### 题 6.1: 进入 ibuffer

```
C-x C-b
```

看到所有 buffer 列表。

ibuffer 显示所有 buffer,按访问时间倒序 (默认)。每个 buffer 一行,显示名字、size、major mode、文件路径。

注意 ibuffer 的键位和 Dired 几乎一样——`m`、`u`、`d`、`x`、`t`、`g`、`q`。**Emacs 的一致性**让你学一次用两处。

### 题 6.2: 标记 + 操作

- `m` 标记 item1.txt
- `m` 标记 item2.txt
- `S` 保存 (已保存的会 noop)
- `U` 取消标记

`S` (save) 批量保存所有标记 buffer。`U` 取消所有标记。和 Dired 的 `m`/`U` 完全同构。

这是 ibuffer 的核心: 标记一组 buffer,批量操作。GUI 编辑器没有这个——你要逐个 buffer 切换、保存、关闭。

### 题 6.3: 按名字标记

- `% f item RET`
- 标记所有名字含 "item" 的 buffer

`% f` 按正则标记 buffer 名字。等于 Dired 的 `%m`,但作用于 buffer 名。

这是清理一组 buffer 的标准方式: `% f \.py RET` + `D` (kill) 关闭所有 Python buffer。

### 题 6.4: 标记按 mode

- `* /` 标记所有 dired buffer
- `M-x` 切到 `* /` (modified) 试试其他

按 mode 标记——`* /` 标记所有 Dired buffer (你刚才开了多个),`* *` 标记所有 modified buffer。

`* /` 在 Dired 和 ibuffer 里含义不同——Dired 里是"标记目录",ibuffer 里是"标记 modified"。**同一个键在不同 mode 下有不同含义**,这是 Emacs 的"上下文敏感"。

### 题 6.5: 删除标记

- 标记几个 buffer (`m`)
- `D` 标记删除 (加 D)
- `x` 执行

ibuffer 里 `D` 标记 buffer 待删除,`x` 执行。和 Dired 完全同构。

如果有未保存改动,Emacs 会问"是否保存?"。**这是 ibuffer 的安全网**。

### 题 6.6: 过滤

- `/ n item RET` filter by name 含 "item"
- 只显示匹配的
- `/ /` 取消过滤

`/ n` (filter by name) 是"视图"——只显示匹配的 buffer,其他不显示。和"标记"不同: 标记是显式选中,过滤是视图调整。

`/ /` 取消所有 filter。组合多个 filter 可以"只显示 Python buffer 且 modified"。

### 题 6.7: 按 mode 过滤

- `/ m Text RET` 只显示 text-mode buffer

`/ m` (filter by mode) 按 major mode 过滤。例如 `/ m python-mode RET` 只显示 Python buffer。

这是"按类型聚焦"——你只关心某类 buffer 时,filter 让视图清爽。

### 题 6.8: 分组 (要配置)

如果 init.el 配了 `ibuffer-saved-filter-groups`,默认显示分组。
- `g` 刷新
- 看分组

分组让 ibuffer 自动归类 buffer——Emacs config 一组、Code 一组、Org 一组……视觉清爽。

这是 ibuffer 的"杀手锏"——把"几十个混乱 buffer"变成"几组分类清单"。配置一次,永久受益。

### 题 6.9: 排序

- `,` 切换排序方式 (按名字/大小/mode等)

ibuffer 默认按访问时间排序。按 `,` 循环切换排序方式——名字、size、mode、时间。

新手常以为 buffer 顺序乱了——其实是排序方式没选对。按 `,` 切换到合适的方式。

### 题 6.10: 标记所有未访问

- `* u` 标记所有"未访问过"的 buffer
- 这是清理临时 buffer 的好方法

`* u` (mark-untouched) 标记"打开过但没编辑过"的 buffer——通常是你点开看过但不需要的临时 buffer。标记后 `D` 批量关闭。

这是"清理 buffer 垃圾"的标准流程。每周做一次,你的 buffer 列表保持清爽。

---

## 实战练习 (30 分钟)

实战是检验学习成果的方式。下面 4 个任务模拟真实场景——整理 Downloads、批量改照片名、跨项目重构、清理子目录。每个都有时间预算,逼你流畅。

### 任务 1: 整理下载目录

```bash
ls ~/Downloads | head  # 看一眼
```

假设有 50 个杂乱文件。

打开 dired:
```
C-x d ~/Downloads RET
```

任务:
1. 标记所有 `.jpg .png .gif`,移到 `~/Pictures/`
2. 标记所有 `.pdf .docx`,移到 `~/Documents/`
3. 标记所有 `.py .js .ts`,移到 `~/Code/`
4. 标记所有 `.bak .tmp .old`,删除
5. 把剩下的按日期排序: `s` (dired-sort-toggle)

时间预算: 5 分钟。

这个任务的"模式"是: 按扩展名标记 → 移动/删除。每个扩展名两步 (`%m` + `R`/`D`),50 个文件 5 分钟绰绰有余。

老手做法: 一次性 `C-x 3` 分屏,左边源 Dired,右边目标 Dired,然后 `dired-dwim-target` 让 `R` 自动选另一 window 的目录——`R RET` 两步完成移动。

### 任务 2: 重命名照片

假设有 100 张 `IMG_XXXX.jpg`,改成 `vacation_XXXX.jpg`:

1. 进入 dired
2. `C-x C-q` 进 wdired
3. 选中所有 .jpg 行
4. `M-% IMG_ RET vacation_ RET` (用 region-restricted replace)
5. `!` 替换所有
6. `C-c C-c` 提交

时间预算: 1 分钟。

这个任务展示 WDired 的威力: GUI 要写脚本,WDired 用 query-replace 一秒搞定。

变体: 如果你想同时给文件名加日期前缀,用 kmacro + `format-time-string`。Module 1 学的 kmacro 在这里完美适用。

### 任务 3: 批量改代码

代码项目里 50 个 .py 文件,要把 `old_function` 改成 `new_function`:

1. 进入项目根 dired
2. `%m \.py$ RET` 标记所有 .py
3. `Q old_function RET new_function RET` 在所有文件里 query-replace
4. 逐个确认 (或 `!` 替换所有)
5. 完成后保存所有: `M-x ibuffer`, `* /` (modified), `S`

时间预算: 5 分钟。

`Q` (dired-do-find-regexp-and-replace) 是项目级重构工具。它打开每个标记文件,依次执行 query-replace——你能看到每次替换的上下文,避免误改。

完成后保存: ibuffer 标记 modified buffer,批量 `S` 保存。这是"Dired 改 + ibuffer 存"的标准流程。

### 任务 4: 子目录清理

```bash
mkdir -p ~/cleanup-test/{a,b,c}
echo "x" > ~/cleanup-test/a/keep1.txt
echo "x" > ~/cleanup-test/a/old.bak
echo "x" > ~/cleanup-test/b/keep2.txt
echo "x" > ~/cleanup-test/b/old.bak
echo "x" > ~/cleanup-test/c/keep3.txt
echo "x" > ~/cleanup-test/c/old.bak
```

任务:
1. dired `~/cleanup-test/`
2. `i` 插入 a, b, c 子目录
3. `%m \.bak$ RET` 标记所有 .bak (跨子目录)
4. `D` 标记删除
5. `x` 执行

时间预算: 2 分钟。

这个任务展示"子目录插入"的威力——一个 buffer 看多个目录,跨目录批量操作。GUI 要打开多个窗口,各自操作;Dired 一次完成。

关键: `i` 插入所有子目录后,`%m` 标记会作用于整个 buffer (跨子目录)。这是"统一视图"的红利。

---

## 自测 (10 题)

下面 10 题检验你**理解**。每题都对应一个核心概念,如果答不出说明那个部分没吃透。

1. 怎么标记所有 .jpg 文件?
2. WDired 怎么进入? 怎么提交?
3. `* /` 干啥? `* .` 干啥?
4. `i` 在 dired 里干啥?
5. ibuffer 比 list-buffers 强在哪?
6. `% g pattern RET` 干啥?
7. `Q OLD RET NEW RET` 在 dired 里干啥?
8. `!` 和 `&` 区别?
9. 怎么批量改 100 个文件名?
10. 怎么备份 Emacs 配置 (init.el)?

**答案**:
> 1. `%m \.jpg$ RET`
> 2. C-x C-q 进入,C-c C-c 提交
> 3. `* /` 标记所有目录;`* .` 标记某扩展名
> 4. insert subdirectory,把子目录内容插入当前 dired 显示
> 5. 标记、过滤、分组、批量操作
> 6. 标记内容含 pattern 的文件 (dired-mark-files-containing-regexp)
> 7. 在所有标记文件里 query-replace
> 8. `!` 同步 shell;`&` 异步 shell
> 9. WDired + M-% 或矩形或 kmacro
> 10. Emacs 自动 backup (FILE~);`make-backup-files t` + `version-control t` 多版本

---

## 常见陷阱 (反例)

练习时新手最常犯的错:

**陷阱 1**: 标记后按 `g` (revert) 丢失所有标记。`g` 重新生成 buffer。重新操作前要重新标记。

**陷阱 2**: WDired 改名后路径不存在,改名失败。提交后看 `*WDired errors*` buffer。

**陷阱 3**: `!` 里用 `$1` 而非 `?`——`?` 才是文件名占位符。

**陷阱 4**: ibuffer 默认按时间排序,新手以为顺序乱。按 `,` 切换排序。

**陷阱 5**: `* /` 在 Dired 和 ibuffer 里含义不同——Dired 里"标记目录",ibuffer 里"标记 modified"。

**陷阱 6**: WDired 里删整行 (`C-k`) 会真删文件,不只是删文本。提交前要审视。

**陷阱 7**: `R` 是 move,不只是 rename。`R foo RET /tmp/foo RET` 把文件移走。

**陷阱 8**: Dired 默认不显示 dotfiles。`dired-listing-switches` 要含 `a`。

记住这些陷阱,你省下大量调试时间。

---

## 毕业检查

- [ ] 40 题至少完成 32
- [ ] 实战 4 个任务在预算时间内完成
- [ ] 用 wdired 流畅,提交不卡顿
- [ ] 用 ibuffer 管理多 buffer 不迷路

完成后进入 `capstone.md`。

Capstone 是 Module 2 的最终验收——10 分钟整理一个 50 文件的混乱目录。如果你能流畅完成 drills 的所有题目,Capstone 就是水到渠成。
