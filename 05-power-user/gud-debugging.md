# GUD (调试器集成) 详解

> 学完这个文件,你能用 Emacs 内置的 GUD 调试 C/C++/Python/Perl/Java 程序

调试是编程工作流里最难自动化的部分——你看不到值、看不到栈、看不到正在执行的指令。1990 年代的解决方案是把调试器搬进编辑器,让代码和运行时数据**并排**显示。Emacs 的 GUD (Grand Unified Debugger) 就是这种思路的最早实践之一。

到了 2026 年,GUD 看起来"朴素"——LSP 的 DAP (Debug Adapter Protocol) 协议更现代、跨语言、跨工具。但 GUD 仍然是 Emacs 内置的、不需要任何额外配置就能跑的调试器——尤其是对 C/C++ 这种没有 LSP 调试器替代品的语言,GUD + GDB 仍然是黄金组合。

---

## 1. 为什么调试器要集成到编辑器

### 1.1 第一性原理推导

调试的本质是"**对照**"——你要把代码里的"我以为是 X"和运行时的"实际是 Y"放在一起看。这种对照工作流有几个推论:

- **第一性原理**: 调试需要看 (1) 代码、(2) 运行时变量、(3) 调用栈、(4) 程序计数器位置。
- **推论 1**: 这些信息分散在 CLI gdb 的输出里——你打 `print foo`,看到 `42`,然后要回到代码理解 `foo` 在哪里被赋值。这是**上下文切换**。
- **推论 2**: 编辑器已经显示了代码——它**有代码的位置信息**。如果调试器告诉编辑器"现在在第 42 行",编辑器可以在代码里高亮第 42 行。
- **推论 3**: 变量值可以**叠加在代码上**——鼠标 hover 在变量名上,显示当前值。这是 GUI 调试器的核心。
- **推论 4**: 多种调试器 (gdb, pdb, perldb, jdb) 应该共享一个 UI——它们的本质操作相同 (step/next/continue/breakpoint),只是底层命令不同。

这四条推论得到 GUD 的设计: **一个统一接口,把代码和运行时数据并排显示,支持多种调试器后端**。

### 1.2 GUD 是什么

GUD = Grand Unified Debugger。名字就揭示了设计哲学——"统一"所有调试器到一套 UI。

GUD 支持的后端:

```
GDB        GNU Debugger (C, C++, Rust, Go, Fortran, ...)
PDB        Python pdb (Python 标准库调试器)
PERLDB     Perl -d (Perl 内置调试器)
JDB        Java Debugger (JDK 自带)
BASHDB     Bash 调试器 (bashdb 包)
GUDBA      Common Lisp
SDB        经典 Unix sdb
DBX        经典 Unix dbx
```

无论用哪个,GUD 提供同一套命令——`C-c C-s` 永远是 step,`C-c C-n` 永远是 next。这种"learn once, debug any language"是 GUD 的最大价值。

### 1.3 历史

GUD 在 Emacs 19 (1994) 就集成了——这是 GDB 4.0 时代,GUI 调试器还很罕见。GUD 的设计参考了 Eclipse、CodeWarrior 等 IDE 的思路,但用 Emacs 的方式实现——所有 UI 都是 buffer,所有操作都是 keybinding。

后来 (2018 年) 微软推出 DAP (Debug Adapter Protocol),LSP 的姐妹项目——Emacs 社区出现了 `dap-mode` (基于 lsp-mode)。但 DAP 协议的客户端在 Emacs 里仍然依赖外部 server,而且 DAP server 比 LSP server 少得多——大多数语言没有 DAP server。所以 GUD 仍然不可替代,特别是 C/C++ 生态。

---

## 2. 启动 GUD

### 2.1 启动 GDB

最常见的入口——调试 C/C++ 程序:

```
M-x gdb RET
```

Emacs 会问 "Run gdb (like this): gdb -i=mi",回车默认就行。`-i=mi` 是关键——它让 GDB 用 **Machine Interface** 输出 (结构化),而不是默认的人类可读输出。GUD 解析 MI 输出,然后更新 source buffer (高亮当前行)、stack buffer、breakpoints buffer。

启动后你会看到**多个 buffer 同时打开**:
- **GUD buffer** (gdb 命令交互,你打 gdb 命令的地方)
- **Source buffer** (显示代码,断点和当前行高亮)
- 可能的辅助 buffer (Stack, Breakpoints, Locals, Registers, Threads, Memory)

这种"一个命令打开多个 buffer"的模式叫 **GDB Many Windows**。开启:

```
M-x gdb-many-windows
```

或配置为默认:

```elisp
(setq gdb-many-windows t)
```

### 2.2 启动 PDB (Python)

```
M-x pdb RET
```

问 "Run pdb (like this): pdb myscript.py",输完后启动。和 GDB 类似,但 Python 没有 MI 这种协议——pdb 的输出是文本,GUD 用正则解析。这导致 pdb 的集成比 gdb 简单但功能少 (没有 threads, registers 等)。

### 2.3 启动其他

```
M-x perldb RET    Perl
M-x jdb RET       Java
M-x bashdb RET    Bash (需要装 bashdb)
```

每个调试器都需要对应的可执行文件在 PATH 里——和 LSP server 类似。如果启动失败,先检查 `which gdb` / `which pdb`。

---

## 3. GUD 的 buffer 结构

理解 GUD 的关键是理解它的**多 buffer 协作**——每个 buffer 显示一种信息。

### 3.1 GUD buffer

主 buffer,显示调试器的输出和接受输入。你可以在这里打**任何 gdb/pdb 命令**——`break main`, `print x`, `backtrace`, `continue`。这相当于一个普通的 gdb/pdb REPL。

GUD buffer 的特殊之处在于——你在 source buffer 里按 `C-c C-s` (step),GUD buffer 里也会看到对应的 `step` 命令执行。**所有 GUI 操作都反映在 CLI 上**,这是 GUD 的"透明"哲学——它不隐藏底层,只是包装。

### 3.2 Source buffer

显示源代码,**断点和当前行高亮**:
- 断点: 行号旁有红点 (在 fringe 里)
- 当前行: 整行被高亮 (通常是浅黄色)

这是 GUD 最直观的部分——你看到光标"停在"代码的哪一行。当程序运行时,这个高亮会**实时跳到新位置**——程序 step 一下,高亮跟着走。

### 3.3 Stack buffer (GDB 专有)

显示调用栈:

```
#0  main () at main.c:42
#1  __libc_start_main (...) at libc-start.c:342
```

每行是一帧 (frame)。点一帧 → source buffer 跳到那一帧的代码。这是"上下导航"——你可以看调用链上任何层的局部变量。

### 3.4 Breakpoints buffer

列出所有断点:

```
Num     Type           Disp Enb Address    What
1       breakpoint     keep y   0x00...    main.c:42
        stop only if x > 10
2       breakpoint     keep y   0x00...    helper.c:100
```

可以**点击**断点启用/禁用,或在 source buffer 里 `C-c C-b` 切换。这种"集中管理"对复杂项目有用——几十个断点时,你需要一个列表。

### 3.5 Locals buffer

显示当前帧的所有局部变量及其值:

```
x = 42
y = "hello"
arr = [1, 2, 3]
```

像 IDE 的 "Variables" 视图——值会随程序执行更新。如果变量是 struct/list,可以展开看成员。

---

## 4. 基础操作

GUD 的核心 keybinding 都以 `C-c` 前缀——这是 minor mode 的标准前缀,GUD minor mode 在 source buffer 里激活。

### 4.1 执行控制

```
C-c C-c    continue (程序继续运行到下个断点)
C-c C-s    step (step into, 进入函数)
C-c C-n    next (step over, 不进入函数)
C-c C-u    step out (运行到当前函数结束)
C-c C-r    run (重新开始)
```

这五个是 99% 的调试操作——step / next / continue 反复用。`C-c C-u` (step out) 特别有用——你 step into 进了一个函数,发现走错了,`C-c C-u` 直接跳出。

记忆方法: **s = step (in), n = next (over), u = up (out)**——u 表示"向上跳出当前函数"。

### 4.2 断点

```
C-x C-a C-b    设置/取消断点 (gud-break)
C-x C-a C-d    删除断点 (gud-remove)
C-x C-a t      临时启用断点 (gud-tbreak)
```

注意是 `C-x C-a` 前缀 (不是 `C-c`)——这是历史原因,GUD 在不同版本里键位变了。Emacs 28+ 也提供 `C-c C-b` 作为别名。

更现代的等价物:

```
C-c C-b    gud-break (Emacs 27+)
```

操作: 在 source buffer 里把光标放在某行,`C-c C-b` 在那一行设断点——再按一次取消。这是 IDE 用户的本能操作,Emacs 也做到了。

### 4.3 查看变量

最强大的功能——GUD tooltip (鼠标 hover 看值):

```elisp
(gud-tooltip-mode 1)
```

开启后,**鼠标悬停在变量名上** (GUI Emacs),弹窗显示当前值。这是 IDE 经典体验,在 Emacs 里也能做到。

键盘等价:

```
C-x C-a C-p    print 表达式 (gud-print)
```

问 "Print expression:",你输 `arr[0]`,echo area 显示值。

---

## 5. GDB 高级用法

### 5.1 Conditional breakpoint

最强大的调试技巧之一——"只有满足条件才停下来"。

在 GUD buffer 里直接打:

```
(gdb) break main.c:42 if x > 10
```

这设置一个断点在 main.c:42,但只有当 `x > 10` 时才停下。配合 GUD 的 source buffer 高亮——红色断点会变成条件断点 (在 Breakpoints buffer 里看到条件)。

调试 "某个分支偶尔走错" 的 bug 时,conditional breakpoint 是**必备**——你不想每次循环都停下来,只想在出错时停。

### 5.2 Watch expression

监控某个表达式的值,每次单步都显示:

```
(gdb) display arr[i]
```

之后每次 step/next,echo area 自动打印 `arr[i]` 的当前值。多个 `display` 累加——你可以 watch `arr[0]`、`arr[1]`、`arr[2]` 三个元素,看它们的演化。

清除 watch:

```
(gdb) undisplay 1     ; 删除第 1 个 display
```

### 5.3 多线程

GDB 支持多线程程序:

```
(gdb) info threads          ; 列出所有线程
(gdb) thread 2              ; 切到线程 2
(gdb) thread apply all bt   ; 所有线程的 backtrace
```

如果开了 `gdb-many-windows`,会有专门的 **Threads buffer** 显示所有线程,点击切换。这是 GUI 调试器才有的能力,GUD 也支持。

### 5.4 Remote debugging

GDB 的远程调试协议——调试运行在另一台机器上的程序。典型场景:

1. 在服务器上跑 `gdbserver :1234 ./program`
2. 在 Emacs 里 `M-x gdb`,在命令里加 `--target=server:1234`

或用 TRAMP 直接调试远程的程序——Emacs 自动用 SSH 隧道连 gdbserver。这是**分布式调试**的强大工具。

---

## 6. PDB (Python) 集成

Python 调试比 C 简单——PDB 是标准库,不需要额外配置。

### 6.1 在代码里插入 breakpoint

最简单的用法——在 Python 代码里写:

```python
import pdb; pdb.set_trace()
```

(或 Python 3.7+: `breakpoint()`)

运行程序,执行到这一行时自动进入 PDB。在 Emacs 里运行这样的程序,PDB 自动 attach 到 GUD buffer——你可以用所有 `C-c C-s` 等键位操作。

### 6.2 直接启动

```
M-x pdb RET
Run pdb (like this): pdb /path/to/script.py
```

或在 Python 代码 buffer 里:

```
M-x python-pdb-debug-at-point
```

(这需要 `python` 包的 pdb 集成)。

### 6.3 PDB 命令

PDB 命令和 GDB 类似但不完全一样:

```
(Pdb) p x           ; print
(Pdb) pp obj        ; pretty-print (适合大对象)
(Pdb) l             ; list 周围代码
(Pdb) w             ; where (backtrace)
(Pdb) b func        ; break at function
(Pdb) c             ; continue
(Pdb) s             ; step into
(Pdb) n             ; next
(Pdb) u             ; up frame
(Pdb) d             ; down frame
```

这些在 GUD buffer 里都可以打——GUD 不挡你用 CLI。

---

## 7. 与 dap-mode 的对比

### 7.1 DAP 协议

DAP (Debug Adapter Protocol) 是微软 2018 年推出的协议——LSP 的姐妹项目。LSP 解决"编辑器懂代码",DAP 解决"编辑器能调试"。

DAP 架构:
```
Editor (dap-mode) ←→ DAP server (debugpy, delv, codelldb...) ←→ Debugger
```

### 7.2 dap-mode

`dap-mode` 是 Emacs 的 DAP 客户端,基于 `lsp-mode`:

```elisp
(use-package dap-mode
  :ensure t
  :config
  (dap-ui-mode)
  (dap-tooltip-mode)
  (require 'dap-python)
  (require 'dap-lldb))
```

启动调试:

```
M-x dap-debug
```

选择配置 (Python / LLDB / Go 等),开始调试 session。

### 7.3 GUD vs DAP

| | GUD | DAP |
|---|---|---|
| 后端 | GDB / PDB / JDB | DAP servers |
| 配置 | 几乎零 | 需要 launch.json 类似配置 |
| 覆盖语言 | C/C++/Java/Python 等 | 多但少 (有些语言没 DAP) |
| UI | 朴素,buffer 为基础 | 现代,类似 VSCode |
| 启动速度 | 极快 (直接调用 gdb) | 慢 (server 启动 + handshake) |
| 维护 | Emacs 内置 (官方) | 第三方包 |

经验法则:
- **C/C++**: 用 GUD + GDB (LLDB 的 DAP 还不成熟)
- **Python**: 都可以,DAP 略方便 (UI 漂亮)
- **Rust / Go**: 用 DAP (delve 和 codelldb 都是 DAP)
- **Java**: 用 DAP (jdtls 提供) 或 JDB

GUD 是**任何机器都能跑**的保底——DAP 需要 server 装好,在服务器上可能没装。

### 7.4 eglot 和 DAP

注意一个常见误解: **eglot 不包含调试**。eglot 只做 LSP (编辑器侧的"懂代码"),不处理调试。Emacs 的调试和 LSP 是**两个独立系统**:
- 编辑器懂代码 → eglot / lsp-mode
- 编辑器调试 → GUD / dap-mode

你完全可以同时用 eglot (编辑) + GUD (调试)——这是常见组合,尤其是 C++ (clangd LSP + GDB GUD)。

---

## 8. Lisp 视角

GUD 的所有功能都是函数——你可以写 Elisp 自动化调试。

### 8.1 程序化设断点

```elisp
(defun my/set-breakpoint-at-error (file line)
  "在 FILE 的 LINE 行设条件断点,条件为 assert 失败。"
  (gud-call (format "break %s:%d if result != 0" file line)))
```

`gud-call` 是向 GUD 发命令的核心函数——它确保命令在正确的 GUD session 里执行。

### 8.2 在测试失败处自动断点

更复杂的——读 `*compilation*` buffer 的错误,在错误处设断点:

```elisp
(defun my/break-at-first-error ()
  "在第一个编译错误处设断点。"
  (interactive)
  (with-current-buffer "*compilation*"
    (goto-char (point-min))
    (when (re-search-forward "\\([^:]+\\):\\([0-9]+\\):" nil t)
      (let ((file (match-string 1))
            (line (string-to-number (match-string 2))))
        (gud-call (format "break %s:%d" file line))))))
```

这种"调试器 + 编译输出"的集成——只有可编程的编辑器能做。

### 8.3 Edebug (Elisp 自己的调试器)

注意——调试 Elisp 本身不用 GUD,用 **Edebug**:

```
M-x edebug-defun    ; 在函数上调用,设断点
```

之后这个函数被调用时,进入 Edebug 模式——`SPC` step,`n` next,`c` continue,`b` 设条件断点。Edebug 在 Module 6 + 7 详讲。

---

## 9. 创造性例子

### 9.1 远程调试 (TRAMP + GDB)

调试服务器上的程序,不用 SSH 出去——TRAMP + GDB:

1. `C-x C-f /ssh:server:/path/to/program` 打开远程程序源码
2. `M-x gdb` 启动——Emacs 自动在远程 SSH 上跑 gdb
3. 所有断点、step 操作都通过 SSH 隧道

这是 Emacs 在分布式场景下的杀手锏——你**在本地 Emacs 里调试远程代码**,体验和本地一模一样。VSCode 的 Remote-SSH 也是这个模式,但 Emacs 20 年前就有。

### 9.2 多线程调试

调试并发 bug——同步问题是 C/C++ 最难的:

```
(gdb) info threads
(gdb) thread apply all bt
```

第一行列出所有线程 (每个有 ID 和当前函数),第二行打印**所有线程的 backtrace**。在 GUD 里配合 `gdb-many-windows` 的 Threads buffer——点不同线程,Locals buffer 和 Stack buffer 同步更新。

找出"线程 A 在等锁,锁在线程 B 手里"这种死锁,`info threads` + 看每个线程的 backtrace 是黄金手段。

### 9.3 Conditional breakpoint 捕捉偶发 bug

"1000 次循环里有 1 次结果错"——你不愿 step 一千次:

```
(gdb) break main.c:42 if result != expected
(gdb) continue
```

只有当 `result != expected` 时才停。1000 次循环可能跑 0.5 秒就触发——你看一眼栈,bug 当场暴露。

### 9.4 Watch expression 跟踪状态

调试排序算法,跟踪每次 swap 后的状态:

```
(gdb) watch arr
(gdb) continue
```

`watch` 是 GDB 的**数据断点**——当 `arr` 被修改时停下来。看每次修改的栈,你能看出"谁改的、改成什么"。这是 C/C++ 调试的瑞士军刀——尤其排查"不该变却变了"的内存错误。

### 9.5 Reverse debugging

GDB 7+ 支持**反向执行**:

```
(gdb) record
(gdb) continue
(gdb) reverse-step    ; 往回 step!
```

`record` 开启记录,之后程序运行被录制。`reverse-step` / `reverse-continue` 让你**回到过去**——这是 GUI 调试器没有的强大能力。比如你看到 segfault,`reverse-step` 几次就能找到谁写了非法指针。

### 9.6 自动生成调试脚本

把调试 session 录制成脚本:

```
(gdb) save breakpoints /tmp/breakpoints.gdb
(gdb) save gdb-index /path/to/bin
```

下次直接 `source /tmp/breakpoints.gdb` 恢复断点。对**反复调试同一个 bug** 的工作流有用——你不用每次重设 10 个断点。

---

## 10. 陷阱

### 10.1 没装 GDB

最常见——`M-x gdb` 失败,提示 "gdb: command not found"。检查:

```bash
which gdb
gdb --version
```

如果没有,装:

- Ubuntu/Debian: `sudo apt install gdb`
- macOS: `brew install gdb` (注意 macOS 的 gdb 需要 codesign 才能 attach 进程)
- Arch: `sudo pacman -S gdb`

macOS 上的 gdb 特别麻烦——Apple 默认用 lldb,装 gdb 需要签名证书。许多人放弃,直接用 lldb + DAP。

### 10.2 没有 debug symbols

调试 release 编译的二进制——你看到的所有变量都是 `<optimized out>`:

```
(gdb) print my_var
$1 = <optimized out>
```

编译时要加 `-g`:

```bash
gcc -g -O0 mycode.c -o mycode
```

`-O0` 关闭优化 (`-O2` 会让变量被优化掉,行号错乱)。**调试时永远用 `-g -O0`**,发布时切回 `-O2`。

### 10.3 GDB / Emacs 版本不匹配

GUD 解析 GDB 的 MI 输出——如果 GDB 版本太新或太旧,MI 格式可能变化。Emacs 28 配 GDB 13 没问题,但 Emacs 25 配 GDB 14 可能出 bug。

经验: **保持 Emacs 和 GDB 都接近最新**。Linux 发行版通常没问题,但 macOS Homebrew 的 GDB 偶尔有兼容问题。

### 10.4 GUD tooltip 只在 GUI 工作

`gud-tooltip-mode` 用 Emacs 的 tooltip 功能——这在 GUI Emacs (X/Wayland/macOS window) 里工作,但在 **TTY Emacs** 里**完全没用** (终端没法弹 tooltip)。

TTY 用户的替代:用 `C-x C-a C-p` (print) 在 echo area 看值。功能等价,只是不"hover"。

### 10.5 pdb 的输出解析脆弱

PDB 没有结构化输出协议 (像 GDB MI)——GUD 用正则解析文本输出。如果 PDB 版本变了输出格式 (例如 Python 3.13 改了 prompt),GUD 可能解析失败,断点不工作。

解决: **用 `realgud` 包** (`M-x realgud:pdb`)——它更现代,解析更健壮。但 realgud 是外部包,需要 `use-package realgud :ensure t`。

### 10.6 启动路径问题

`M-x gdb` 问 "Run gdb like this: gdb -i=mi ./myprogram"——路径是相对**当前 default-directory**。如果你在错误的目录下启动,程序找不到。

确保 `M-x gdb` 之前先 `M-x cd` 到正确目录,或用 `C-u M-x gdb` 让 Emacs 问目录。

---

## 11. 练习

1. 写一个简单的 C 程序 (循环 + 函数调用),编译成 `-g -O0`,用 `M-x gdb` 调试
2. 在 main 里设断点,run,然后 step / next 各一次,观察 source buffer 高亮
3. 设 conditional breakpoint: 只在循环计数 = 5 时停
4. 开启 `gud-tooltip-mode`,鼠标 hover 变量看值 (GUI Emacs)
5. 用 `display` watch 一个变量,每次 step 看演化
6. 写一个简单的 Python 脚本,用 `M-x pdb` 调试,设断点 + step
7. 在 Python 代码里写 `breakpoint()`,从命令行跑,看 Emacs 自动 attach
8. 用 `info threads` + 多线程程序,找出主线程和工作线程的栈
9. 用 `record` + `reverse-step`,体验反向调试
10. 配置 `gdb-many-windows`,在 Stack buffer 点不同帧看 source buffer 跳转
11. 在 Breakpoints buffer 启用/禁用一个断点,看 source buffer 标记变化
12. 配置 `dap-mode`,跑一次 Python 调试 session (`M-x dap-debug`)
13. 写一个 Elisp 函数,自动在 `*compilation*` buffer 的第一个错误处设断点
14. 用 TRAMP 打开远程程序,启动远程 GDB,体验远程调试
15. 调试一个 segfault: 用 `bt` 看栈,`frame N` 切换,`print` 看变量
16. 用 `watch` 设数据断点,跟踪一个变量被谁修改

---

## 12. 自测

1. GUD 是什么的缩写?支持哪些调试器?
2. `M-x gdb` 时 `-i=mi` 参数是干嘛的?
3. `C-c C-s` 和 `C-c C-n` 区别?
4. 怎么开启 "鼠标 hover 看变量值"?
5. conditional breakpoint 怎么设?
6. GUD 和 dap-mode 关系?
7. 为什么调试时要 `-g -O0` 编译?

**答案**:
> 1. Grand Unified Debugger;支持 GDB/PDB/JDB/PERLDB 等
> 2. `-i=mi` 让 GDB 用 Machine Interface 输出 (结构化,GUD 能解析)
> 3. C-c C-s = step into (进入函数);C-c C-n = next/step over (不进入)
> 4. `(gud-tooltip-mode 1)`
> 5. `break file:line if condition`
> 6. DAP 是 LSP 的姐妹协议 (微软,2018);dap-mode 是 Emacs 的 DAP 客户端,UI 更现代但需要 server;GUD 内置,适合 C/C++
> 7. `-g` 加 debug symbols;`-O0` 关闭优化 (避免变量被优化掉,行号错乱)

---

## 13. 下一步

读完这个文件,你已经会用 Emacs 调试多种语言的程序。

进入 Module 5 的其他专题 (`mule-international.md`) 学国际化。
