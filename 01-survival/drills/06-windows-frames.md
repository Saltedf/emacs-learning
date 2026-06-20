# Drill 06: Windows & Frames — 一屏多视图的艺术

> **预计用时**: 1.5 小时
> **难度**: ★★☆☆☆
> **替代**: `emacs-manual/windows.texi` + `frames.texi` 的用户视角
> **目标**: 在一个屏幕里同时看 4 个文件,丝滑切换

---

## 0. 学习目标

读完这个 drill,你应该能够:

- [ ] 在 30 秒内分屏成 4 个 window,各看不同文件
- [ ] 用键盘在 window 之间无缝切换 (不用鼠标)
- [ ] 在"另一个 window"打开文件,而不离开当前 window
- [ ] 保存当前 window 布局,一键恢复
- [ ] 理解 frame、window、buffer 的三角关系

---

## 1. 故事: 你为什么需要分屏?

想象这个场景:

你正在写代码,左边是 `main.py`,右边是参考文档 `notes.md`。
你想看 `main.py` 第 100 行,同时看 `notes.md` 第 50 行。
你想复制 `notes.md` 的一段,粘到 `main.py`。

在普通编辑器里,你可能是这样的:
- 开两个窗口 (两个进程)
- 用操作系统的 alt-tab 切换
- 复制粘贴跨窗口麻烦

在 Emacs 里,你**完全在同一个进程内**做这件事:
- 一个 frame,两个 window (左右分屏)
- 切换 window 用 `C-x o` (不到 0.1 秒)
- 复制粘贴是同一个 kill-ring
- 每个 window 可以独立 scroll

**而且不止两个 window**——你可以分 3、4、6 个,任意布局。

---

## 2. 第一性原理: Window 不是 Buffer

新手最大的混淆: **window 和 buffer 是不是一回事?**

**不是**。让我用一个比喻。

### 2.1 "房间与窗户" 模型

想象你住在一栋楼里:

- **Buffer** = 房间 (每个房间有内容,有名字)
- **Window** = 墙上的窗户 (透过窗户看一个房间)
- **Frame** = 整栋楼 (一个 Emacs 进程可以有多个 frame)

关键认知:

```
- 房间一直存在,不管你开不开窗户看它
- 一扇窗只看一个房间
- 一扇窗只看房间的一部分 (你不能同时看到房间的所有角落)
- 多扇窗可以看同一个房间 (两个 window 显示同一个 buffer)
- 一面墙可以有多扇窗 (分屏)
```

这个比喻的精髓: **buffer 是数据,window 是视图**。数据独立存在,视图是"看数据的窗口"。这是 Model-View 分离的经典设计,后来被很多软件借鉴 (MVC 架构)。

为什么这个区分重要? 因为它解释了 Emacs 的几个反直觉行为:
- 你关一个 window,buffer 还在 (`C-x 0` 关 window,buffer 留着;`C-x k` 关 buffer)
- 两个 window 显示同一个 buffer,一个改了,另一个立刻看到 (因为底层 buffer 是共享的)
- buffer 里的 point 是 buffer 的属性,所有显示它的 window 共享同一个 point

最后一点常让人困惑: 你在 window A 把光标移到第 100 行,切到 window B (显示同一个 buffer),光标也在第 100 行。这不是 bug,是设计——point 属于 buffer,不属于 window。如果你要独立 point,用 `clone-indirect-buffer`。

### 2.2 实践验证

打开 Emacs,试这个序列:

1. `C-x C-f file1.txt RET` — 打开 file1 (现在有 1 个 window,看 file1 buffer)
2. `C-x 3` — 左右分屏 (现在有 2 个 window,都看 file1 buffer)
3. 光标在右 window,`C-x C-f file2.txt RET` — 右 window 现在看 file2
4. 左 window 还在看 file1 (它没动!)
5. `C-x o` — 切到左 window
6. `C-x b RET` — 切换左 window 的 buffer

**等等,让我重读一遍上面的序列,确认你真的做了**。

不是只读,要真的做。打开 Emacs,跟着步骤。

做完后,你应该看到类似这样:

```
┌────────────────┬────────────────┐
│ file1.txt      │ file2.txt      │
│ ...            │ ...            │
│                │                │
│ -uu-:**-F1 ... │ -uu-:**-F1 ... │
└────────────────┴────────────────┘
```

现在你已经**亲手验证**了: window ≠ buffer。两个 window 显示两个 buffer。

这个验证的价值: 你**亲手操作过**,而不仅仅是读。Emacs 的学习严重依赖肌肉记忆——读 10 遍不如做 1 遍。Module 1 的所有 drill 都强调"做",不是"读"。

---

## 3. 完整命令表 (慢一点,认真读)

### 3.1 创建/删除 window

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-x 2` | `split-window-below` | 上下分屏 (新 window 在下方) |
| `C-x 3` | `split-window-right` | 左右分屏 (新 window 在右侧) |
| `C-x 1` | `delete-other-windows` | 只保留当前,删其他 |
| `C-x 0` | `delete-window` | 删当前 window (不删 buffer) |

**注意**: `C-x 0` 和 `C-x 1` 删的是 window,不是 buffer。
buffer 还在 (用 `C-x b` 还能切回去)。
window 删了,要再分屏才能恢复 (没有"撤销删除 window"的直接命令)。

### 3.2 切换 window

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-x o` | `other-window` | 切到下一个 window (循环) |
| `M-- C-x o` | | 切到上一个 window |
| `M-0 C-x o` | 同上 | 也是上一个 (数字前缀) |
| `C-x 4 0` | `kill-buffer-and-window` | 关 buffer + window |
| `windmove-left` | (需绑定) | 切到左 window |
| `windmove-right` | (需绑定) | 切到右 window |
| `windmove-up` | (需绑定) | 切到上 window |
| `windmove-down` | (需绑定) | 切到下 window |

### 3.3 调整 window 大小

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-x ^` | `enlarge-window` | 增高 1 行 (前缀 N: 增 N 行) |
| `C-x }` | `enlarge-window-horizontally` | 增宽 1 列 |
| `C-x {` | `shrink-window-horizontally` | 减宽 1 列 |
| `C-x -` | `shrink-window-if-larger-than-buffer` | 缩到刚够装内容 |
| `C-x +` | `balance-windows` | 所有 window 等分 |

**前缀**: `C-u 10 C-x ^` 增高 10 行;`M-5 C-x }` 增宽 5 列。

### 3.4 在"其他 window"操作

这是 Emacs 一个非常优雅的设计:

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-x 4 C-f FILE` | `find-file-other-window` | 在另一个 window 打开文件 |
| `C-x 4 b` | `switch-to-buffer-other-window` | 在另一个 window 切 buffer |
| `C-x 4 d` | `dired-other-window` | 在另一个 window 开 dired |
| `C-x 4 r` | `find-file-read-only-other-window` | 只读 |
| `C-x 4 m` | `mail-other-window` | 写邮件 |
| `C-x 4 a` | `add-change-log-entry-other-window` | changelog |
| `C-x 4 f` | (同 C-f) | 别名 |
| `C-x 4 .` | `find-tag-other-window` | xref 跳定义 |

**`4` 的语义**: "在另一个 window 操作"。同理,`5` 是"在另一个 frame 操作"。

### 3.5 滚动其他 window

| 键位 | 命令 | 作用 |
|---|---|
| `C-M-v` | `scroll-other-window` | 下一个 window 向下滚 |
| `C-M-S-v` (或 `C-M-V`) | `scroll-other-window-down` | 下一个 window 向上滚 |

**神级技巧**: 你在左 window 写代码,右 window 在看文档。光标留在左,但你想看文档的不同部分——`C-M-v` 直接滚右 window,不离开你的代码光标。

这个键的重要性: 它解决了"边写边看"的认知负担。普通流程: 写代码 → 切到文档 window → 翻页 → 切回代码 window → 继续写。`C-M-v` 流程: 写代码 → 滚文档 → 继续写。光标永远在代码上,认知状态保持"写代码",只是余光扫文档。

**创造性用法**:
- 看长文档时,左 buffer 是大纲,右 buffer 是内容。`C-M-v` 滚右 buffer 读内容,左 buffer 留作导航。
- 调试时,上 buffer 是代码,下 buffer 是日志。`C-M-v` 滚日志找错误,代码 buffer 不动。
- 写测试时,左 buffer 是实现,右 buffer 是测试。`C-M-v` 滚测试看期望,实现 buffer 留光标。

### 3.6 Windows 的创造性应用汇总

1. **多视图对比**: 两个 window 显示同一个 buffer 的不同部分,对比阅读。
2. **代码 + 测试**: 左 window 代码,右 window 测试,kill-ring 跨 window 共享。
3. **代码 + 文档**: 主 window 代码,辅 window 文档,`C-M-v` 滚文档。
4. **代码 + Shell**: 上 window 代码,下 window shell,`M-x compile` 或 `ansi-term`。
5. **多文件导航**: 4 个 window 各开一个相关文件,`C-x o` 循环切。
6. **`winner-mode` 实验**: 大胆分屏,反正 `C-c left` 能撤销。
7. **window 配置存 register**: 把理想的 4-window 布局存到 register w,任何时候 `C-x r j w` 恢复。
8. **`pop-to-buffer` 临时窗口**: 让某些 buffer (如 help) 在临时 window 显示,关掉时不影响主布局。

---

## 4. 故事: 一个真实的双屏工作流

让我带你走一遍真实的工作流,看看分屏怎么用。

### 场景: 改 bug,需要看代码 + 日志 + 测试

启动 Emacs,打开你的项目。然后:

**Step 1**: 当前只有一个 window,显示 `main.py`。

**Step 2**: 你想看测试。`C-x 3` 左右分屏。光标现在在右 window。

**Step 3**: 在右 window 打开测试文件: `C-x C-f test_main.py RET`。

```
┌─────────────┬─────────────┐
│ main.py     │ test_main.py│
│             │             │
│ -F1 Top L1  │ -F1 Top L1  │
└─────────────┴─────────────┘
```

**Step 4**: 你想在底部看一个 shell。`C-x o` 切到右 window,然后 `C-x 2` 上下分屏:

```
┌─────────────┬─────────────┐
│ main.py     │ test_main.py│
│             ├─────────────┤
│             │             │
│             │             │
│ -F1 ...     │ -F1 ...     │
└─────────────┴─────────────┘
```

**Step 5**: 右下 window 跑 shell: `M-x ansi-term RET zsh RET`。

**Step 6**: 现在你想看 git log。切到左 window (`C-x o` 两次,或直接 `M-o` 如果你绑定了),`C-x 2` 分上下,`M-x magit-log` (或 `vc-print-log`)。

```
┌─────────────┬─────────────┐
│ main.py     │ test_main.py│
├─────────────┼─────────────┤
│ git log     │ shell       │
│ -F1 ...     │ -F1 ...     │
└─────────────┴─────────────┘
```

**Step 7**: 4 个 window 同时显示 4 个 buffer。用 `C-x o` 循环切换,用 `C-M-v` 滚非当前 window。

### 反思: 这个流程的关键

- **没有用鼠标**
- **每个 window 独立** (scroll、buffer、point)
- **kill-ring 跨 window 共享** (在 shell 里复制的,在代码里能粘)
- **整个过程在同一个 Emacs 进程内**

普通编辑器要做这事: 开 4 个文件 + 切窗口 + alt-tab。在 Emacs 里: 6 个键。

---

## 5. Windmove: 像方向键一样切 window

`C-x o` 循环切 window,在 4 个 window 时按几次才到位。
更好的方式: 用方向键直接指。

加到 init.el:

```elisp
(when (fboundp 'windmove-default-keybindings)
  (windmove-default-keybindings))
```

这会绑定:
- `S-<left>` → windmove-left
- `S-<right>` → windmove-right
- `S-<up>` → windmove-up
- `S-<down>` → windmove-down

现在 Shift + 方向键就能直接切到指定方向的 window。

(注意: 用 Shift 不是用 Control,因为 Control+方向键在很多 mode 有别的用途)

**我的推荐**: 绑定到 `M-o` + hjkl (Vim 风格):

```elisp
(global-set-key (kbd "M-o h") 'windmove-left)
(global-set-key (kbd "M-o l") 'windmove-right)
(global-set-key (kbd "M-o k") 'windmove-up)
(global-set-key (kbd "M-o j") 'windmove-down)
```

或更简单的 `M-o` 直接循环 (替代 `C-x o`):

```elisp
(global-set-key (kbd "M-o") 'other-window)
```

这样 `M-o` 比 `C-x o` 短,小拇指更舒服。

---

## 6. Frames: 多个物理窗口

Frame = 物理窗口 (GUI 下) 或整个终端 (TTY 下)。

### 6.1 创建/删除 frame

| 键位 | 命令 | 作用 |
|---|---|---|
| `C-x 5 2` | `make-frame-command` | 新建 frame |
| `C-x 5 0` | `delete-frame` | 删当前 frame |
| `C-x 5 1` | `delete-other-frames` | 只保留当前 |
| `C-x 5 o` | `other-frame` | 切换 frame |
| `C-x 5 C-f FILE` | `find-file-other-frame` | 在新 frame 打开文件 |
| `C-x 5 b` | `switch-to-buffer-other-frame` | 在新 frame 切 buffer |
| `C-x 5 d` | `dired-other-frame` | 在新 frame 开 dired |
| `C-x 5 r` | `find-file-read-only-other-frame` | 只读 |
| `C-x 5 .` | `find-tag-other-frame` | xref |
| `C-x 5 m` | `mail-other-frame` | 邮件 |

**`5` 的语义**: "在另一个 frame 操作"。

### 6.2 Frame vs Window: 什么时候用哪个?

| 场景 | 推荐 |
|---|---|
| 单显示器,代码 + 文档并排 | Window (`C-x 3`) |
| 多显示器,代码在这屏,聊天在那屏 | Frame (`C-x 5 2`) |
| 终端 Emacs,只能一个 frame | Window |
| 跨桌面切换工作 (workspace) | Frame |
| 短期看一个文件然后回去 | Window (在另一个 window 显示) |

**经验**: 大多数人用 window 就够了。Frame 是给多显示器或 daemon 模式 (`emacsclient -c`) 的。

### 6.3 Daemon + emacsclient (强烈推荐)

```bash
# 启动 daemon (后台)
emacs --daemon

# 任何时候秒开新 frame
emacsclient -c        # GUI
emacsclient -t        # TTY
emacsclient -e '(message "hi")'  # eval 表达式不进 frame
```

Daemon 模式的好处:
- Emacs 启动一次,持续运行
- buffer、window 状态、变量都保留
- emacsclient 秒开新 frame
- 关 frame 不杀 daemon

加到 init.el:
```elisp
(server-start)  ; 让普通 Emacs 也接受 emacsclient
```

或者用 systemd service 管理 daemon:
```ini
# ~/.config/systemd/user/emacs.service
[Unit]
Description=Emacs daemon

[Service]
Type=forking
ExecStart=/usr/bin/emacs --daemon
ExecStop=/usr/bin/emacsclient --eval "(kill-emacs)"
Restart=on-failure

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable emacs
systemctl --user start emacs
```

---

## 7. Window 配置管理

### 7.1 winner-mode: undo/redo window 布局

加到 init.el:

```elisp
(winner-mode 1)
```

然后:
- `C-c <left>` — winner-undo (恢复上一个 window 配置)
- `C-c <right>` — winner-redo

**场景**: 你分屏成 4 个 window,临时 `C-x 1` 只看一个。看完想恢复 4 个,`C-c <left>`。

### 7.2 Register 存 window 配置

```
C-x r w a    window-configuration-to-register (存当前布局到 register a)
C-x r j a    jump-to-register (恢复)
```

**场景**: 你有一个理想的 4-window 布局 (代码 + 测试 + shell + 文档)。
存到 register `w`。任何时候 `C-x r j w` 恢复。

### 7.3 Burly / persp-mode / eyebrowse

这些是高级包,提供"workspace"概念 (像 IDE 的多 workspace):
- `burly`: 保存/恢复 window 配置 + buffer 列表
- `persp-mode`: 虚拟 workspace
- `eyebrowse`: 简单 workspace 切换

Module 5 可以装一个。

---

## 8. 默认行为调整

### 8.1 split-window-preferred-function

`(setq split-width-threshold 160)` — 当 window 宽于 160 字符时,默认 `C-x C-f` 等命令会自动分屏。

默认 160,大多数显示器够宽。设 nil 禁用。

### 8.2 display-buffer-alist

控制"在新 window 显示 buffer"的行为:

```elisp
(setq display-buffer-alist
      '(("\\*Help\\*" . (display-buffer-same-window))
        ("\\*Completions\\*" . (display-buffer-at-bottom))
        ("\\*shell\\*" . (display-buffer-pop-up-window))))
```

Module 4 学 customize 时会深入。

### 8.3 switch-to-buffer-in-dedicated-window

```
(setq switch-to-buffer-obey-display-minibuffers-alist t)
```

防止不小心切到 minibuffer buffer。

---

## 9. 微练习 (20 题,45 分钟)

### 基础 (5 题)

1. `C-x 3` 左右分屏,`C-x o` 切右 window
2. 在右 window 打开另一个文件 (`C-x C-f`)
3. `C-x 1` 只留当前 window
4. `C-x 2` 上下分屏,`C-x o` 切下面
5. `C-x 0` 关当前 window

### 切换 (5 题)

6. `C-x o` 在 3 个 window 里循环
7. `M-- C-x o` 反向循环
8. 绑定 windmove,Shift+方向键切 window
9. (用 `windmove-default-keybindings` 后) 在 4 个 window 里用方向键任意切
10. `C-M-v` 滚另一个 window (光标不动)

### 大小 (3 题)

11. `C-x ^` 当前 window 增高 1 行
12. `C-u 5 C-x }` 增宽 5 列
13. `C-x +` 所有 window 等分

### "Other window" 操作 (4 题)

14. `C-x 4 C-f file.txt RET` 在另一个 window 打开文件
15. `C-x 4 b buffer-name RET` 在另一个 window 切 buffer
16. `C-x 4 d RET` 在另一个 window 开 dired
17. 在 dired 里,`o` (在另一个 window 打开文件)

### Frames (3 题)

18. `C-x 5 2` 新建 frame
19. `C-x 5 o` 切换 frame
20. `C-x 5 0` 关当前 frame

---

## 10. 实战练习 (15 分钟)

### 任务 1: 三屏代码工作流

打开你的项目:
1. `C-x C-f main.py`
2. `C-x 3` 左右分屏
3. `C-x o` 切右,`C-x C-f test_main.py`
4. `C-x o` 切左,`C-x 2` 上下分屏
5. `C-x o` 切下,`M-x ansi-term RET zsh RET`
6. 现在有 3 window: 左上是 main.py,左下是 term,右是 test

在这个布局里:
- 在 term 跑 `python -m pytest`
- 看测试失败,在 main.py 修
- 切到 test 看期望
- 反复迭代

用 `C-x o`、`C-M-v`、windmove,不用鼠标。

### 任务 2: Window 配置管理

1. 创建一个 4-window 布局 (代码/测试/shell/doc)
2. `C-x r w w` 存到 register w
3. `C-x 1` 只看代码 window,工作 5 分钟
4. `C-x r j w` 恢复 4-window 布局

### 任务 3: winner-mode

1. init.el 加 `(winner-mode 1)`,eval
2. 分屏成 4 个
3. `C-x 1` 只留一个
4. `C-c <left>` 恢复 4 个
5. `C-c <right>` redo 回到 1 个

---

## 11. 常见陷阱

### 11.1 `C-x 0` 关错 window

`C-x 0` 关的是**当前** window (光标所在)。
新手经常关错。

解决: 养成 `C-x o` 先切到要关的,再 `C-x 0`。

### 11.2 buffer 在哪个 window 显示?

Emacs 的"在哪个 window 显示 buffer"逻辑很复杂,由 `display-buffer-alist` 等变量控制。

如果命令在错的地方显示了 buffer,查 `C-h v display-buffer-alist`,或读 `(info "(elisp) Displaying Buffers")`。

### 11.3 minibuffer 占用一个 window?

当你按 `M-x`,minibuffer 占用 echo area,通常不"开 window"。
但有时 (复杂的命令),minibuffer 会开辟一个 window。

正常行为,不要慌。

### 11.4 dedicated window

`(set-window-dedicated-p window t)` 把一个 window 标记为 "dedicated",不会被 `switch-to-buffer` 替换 buffer。

新手不会主动设,但有些 mode (像 gdb) 会设。

---

## 12. 自测

1. window 和 buffer 的本质区别?
2. `C-x o` 干啥? `C-x 5 o` 干啥?
3. `C-x 4 C-f` 和 `C-x C-f` 区别?
4. `C-M-v` 是什么? 为什么有用?
5. winner-mode 干啥?

**答案**:
> 1. window 是显示区域,buffer 是文本容器;一个 window 显示一个 buffer;一个 buffer 可以被多个 window 显示
> 2. other-window 切下一个 window;other-frame 切下一个 frame
> 3. 前者在另一个 window 打开文件,光标也跳过去;后者在当前 window 打开
> 4. scroll-other-window,滚另一个 window 的内容,光标留在当前;用于"边写代码边看文档"
> 5. 让 window 布局可以 undo/redo,C-c left 撤销,C-c right 重做

---

## 13. 毕业检查

- [ ] 20 题至少完成 16
- [ ] 能在 30 秒内做出 4-window 布局
- [ ] windmove 或方向键切 window 流畅
- [ ] 用 register 存/恢复 window 配置
- [ ] 知道 frame 和 window 何时用哪个

完成后进入 `drills/07-help-system.md` ★最重要★。
