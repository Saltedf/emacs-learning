# Capstone: Module 0 毕业项目

> **目标**: 验证你真的掌握了 Module 0 的心智模型 + 装备
> **用时**: 1-2 小时
> **完成标志**: 所有任务打勾

---

## 项目概述

这个 capstone 包含 4 个子项目:

1. **环境验证** (15 分钟): 确认 Emacs 编译正确,功能完整
2. **init.el 完整化** (30 分钟): 把最小 init.el 扩展到 ~30 行,加入自己的偏好
3. **心智模型速记** (15 分钟): 不看任何资料,写下 5 条第一性原理
4. **第一次"住在 Emacs 里"** (30 分钟): 用 Emacs 做一件真实的小任务

---

## 子项目 1: 环境验证 (15 分钟)

### Task 1.1: 版本与功能检查

在 shell 跑这些命令,记录输出:

```bash
emacs --version
```

期望: `GNU Emacs 30.2` (或更高)

```bash
emacs --batch --eval='(message "lisp: %s" (version))'
```

期望: 显示完整版本字符串

```bash
emacs --batch --eval='(message "native: %s" (if (native-comp-available-p) "yes" "no"))'
```

期望: `native: yes`

```bash
emacs --batch --eval='(message "ts: %s" (if (treesit-available-p) "yes" "no"))'
```

期望: `ts: yes`

### Task 1.2: 启动测试

```bash
time emacs --batch --eval='(sleep-for 0.1)'
```

期望: 总时间 < 1 秒 (说明 Emacs 启动很快,native comp 工作)

```bash
emacs -Q --eval='(message "%s" (+ 1 2))'
```

期望: 启动一个 GUI 窗口,关闭它,minibuffer 显示 `3`

### Task 1.3: Daemon 模式

启动 daemon:

```bash
emacs --daemon &
```

等待 "Started Emacs Daemon" 输出。

连接:

```bash
emacsclient -c
```

应该秒开一个 GUI 窗口。

在窗口里 `M-x`,运行几个命令,感受 daemon 模式的速度。

关闭 frame (不杀 daemon): `C-x 5 0` 或关掉窗口。

再开一个:

```bash
emacsclient -c
```

应该又是秒开。

杀掉 daemon:

```bash
emacsclient --eval '(kill-emacs)'
```

### Task 1.4: 干净模式自检

```bash
emacs -Q
```

在干净 Emacs 里:
- 看到默认主题 (浅色,不是你的 modus-vivendi)
- 看到工具栏
- `C-h v user-init-file RET` 应该显示 nil (没加载 init.el)

退出 `C-x C-c`。

**记录**: 看到默认 Emacs 的样子,你应该理解你 init.el 改了什么。

### ✓ 子项目 1 检查

- [ ] Emacs 30.2+ 装好
- [ ] native comp 工作
- [ ] tree-sitter 工作
- [ ] 启动 < 1 秒
- [ ] daemon 模式秒开
- [ ] 干净模式自检通过

#### 为什么这些检查很重要

每一项不只"打勾就完",背后是 Module 0 的概念:

- **Emacs 30.2+**: 你享受最新包生态 (有些包要求 29+)
- **native comp**: Elisp 不会"慢",Magit、LSP 等大包流畅
- **tree-sitter**: 现代语法高亮、增量解析,VS Code 同款技术
- **启动 < 1 秒**: native comp + 你的 init.el 不重的证据
- **daemon 秒开**: 你的 Emacs 是"持续运行的服务器",不是"开开关关的应用"
- **干净模式自检**: 你**理解**你 init.el 改了什么,不是抄的

如果某项不过,不要"打勾凑数"。回到 README.md / concept-anchor.md 重读相关章节,**理解为什么没过**,然后修复。这是 capstone 的真正目的——**自检理解,不是完成任务**。

---

## 子项目 2: init.el 完整化 (30 分钟)

### Task 2.1: 在最小配置基础上加这些

打开你的 `~/.config/emacs/init.el`,加入这些 (在原有内容基础上):

```elisp
;;; 更好的默认值
(setq-default
 indent-tabs-mode nil               ; 用空格,不用 tab
 tab-width 4                         ; tab 宽度 4
 fill-column 80                      ; fill 宽度 80
 truncate-lines t)                   ; 不自动折行

(setq
 history-length 1000                 ; 历史更长
 use-short-answers t                 ; y-or-n-p 代替 yes-or-no-p
 confirm-kill-processes nil          ; 退出时不问
 load-prefer-newer t                 ; 加载更新的 .el
 enable-recursive-minibuffers t      ; minibuffer 里可以再开 minibuffer
 read-process-output-max (* 1024 1024) ; 进程输出 buffer 大
 )

;;; UX 增强
(global-hl-line-mode 1)              ; 高亮当前行
(show-paren-mode 1)                  ; 高亮配对括号
(electric-indent-mode 1)             ; 自动缩进
(savehist-mode 1)                    ; minibuffer 历史
(recentf-mode 1)                     ; 最近文件
(winner-mode 1)                      ; window 配置 undo/redo

;;; 字符编码
(prefer-coding-system 'utf-8)
(set-default-coding-systems 'utf-8)
(set-terminal-coding-system 'utf-8)
(set-keyboard-coding-system 'utf-8)

;;; 警告
(setq warning-minimum-level :emergency) ; 减少警告噪音

;;; 行号设置
(setq display-line-numbers-type 'relative) ; 相对行号
(global-display-line-numbers-mode 1)

;;; Dired
(setq dired-dwim-target t            ; dired 复制时智能选另一个 window
      dired-recursive-copies 'always
      dired-recursive-deletes 'top
      dired-listing-switches "-alh") ; ls 的开关

;;; 模式
(global-so-long-mode 1)              ; 长行优化
(global-visual-line-mode 1)          ; 视觉换行
(pixel-scroll-precision-mode 1)      ; 平滑滚动 (Emacs 29+)

;;; Completions
(setq completion-styles '(basic substring initials flex partial-completion))
```

### Task 2.2: 添加几个有用的键位

```elisp
;;; 关闭窗口快捷键
(global-set-key (kbd "C-x k") 'kill-this-buffer) ; 不问,直接 kill 当前 buffer
(global-set-key (kbd "C-c C-k") 'kill-buffer-and-window)
(global-set-key (kbd "C-s-<return>") 'ansi-term) ; Cmd+Return 开终端 (mac)

;;; 切换 window
(global-set-key (kbd "M-o") 'other-window) ; 比 C-x o 短

;;; 复制当前行
(defun my-duplicate-line ()
  "Duplicate current line."
  (interactive)
  (let ((col (current-column)))
    (move-beginning-of-line 1)
    (kill-line)
    (yank)
    (newline)
    (yank)
    (move-to-column col)))
(global-set-key (kbd "C-c d") 'my-duplicate-line)
```

### Task 2.3: 主题与字体 (再次检查)

```elisp
;;; 主题
(load-theme 'modus-vivendi t)

;;; 字体
(set-face-attribute 'default nil
                    :family "JetBrains Mono"
                    :height 140
                    :weight 'regular)

;; 中文字体
(when (find-font (font-spec :family "Sarasa Mono SC"))
  (set-fontset-font t ?中 "Sarasa Mono SC"))
```

### Task 2.4: 保存并应用

`C-x C-s` 保存。

`M-x eval-buffer RET` 应用。

观察变化:
- [ ] 高亮当前行
- [ ] 配对括号高亮
- [ ] 相对行号
- [ ] 字体变了 (JetBrains Mono)
- [ ] 平滑滚动

如果某个不工作,用 `C-h v` 查变量,用 `C-h f` 查函数。

### Task 2.5: 验证理解

对每一行配置,你应该能回答:

1. 它设置什么变量或调用什么函数? (用 `C-h v` 或 `C-h f`)
2. 它改变 Emacs 的什么行为?
3. 它在你后续使用中会带来什么体验差异?

随机抽 5 行,写下它们的解释:

```
1. (setq tab-width 4)
   - 设置 tab-width 变量为 4
   - tab 显示宽度为 4 个空格
   - 让代码缩进更紧凑

2. [你的]

3. [你的]

4. [你的]

5. [你的]
```

### ✓ 子项目 2 检查

- [ ] init.el 至少 50 行
- [ ] 10+ 个 `setq` 设置的变量,每个都查过文档
- [ ] 3+ 个自定义键位
- [ ] 1+ 个自定义函数
- [ ] 保存后 eval-buffer,所有设置生效
- [ ] 能解释每一行

#### 为什么这 50 行配置是"你的"

考虑两种学 Emacs 的方式:

**方式 A (普通)**: 复制粘贴别人的 1000 行配置,看效果惊艳,然后大部分不懂。
- 后果: 你"用着别人的 Emacs",不是"用着你的 Emacs"
- 半年后某个行为你不满意,你不知道是哪行控制的
- 一年后某个包升级,配置冲突,你不知道怎么修

**方式 B (本教程)**: 自己写 50 行,每行 `C-h v` 理解。
- 后果: 这 50 行是**你的资产**,任何一行你都能解释
- 半年后加新功能,你知道加在哪、为什么
- 一年后你的配置变成 200 行,**每一行都是你写的**

**这是为什么本 capstone 强调"能解释每一行"**——50 行能解释的配置 > 1000 行不能解释的配置。

#### 第一性原理: init.el 是程序,不是配置

提醒: 你的 init.el 不是"配置文件",是**程序**。每次 Emacs 启动都执行它。所以:

- 性能: `eval-buffer` 要快 (避免 `require` 没必要的包)
- 副作用: `(global-set-key ...)` 影响所有 buffer,要小心
- 错误处理: 一个 `setq` 报错,后面的不执行——用 `condition-case` 包
- 可测试性: 你可以 `(eval-buffer)` 重跑,不需要重启

把这 50 行当作"小型软件项目"对待——加注释、分模块、能调试。Module 4 会更系统讲 init.el 工程化。

---

## 子项目 3: 心智模型速记 (15 分钟)

### Task 3.1: 关掉所有资料

合上 README.md、concept-anchor.md、intro-reading.md。
关闭浏览器。
只用 `*scratch*` buffer。

### Task 3.2: 写下五条第一性原理

在 `*scratch*` 里,用 Elisp 注释格式写下五条:

```elisp
;; Emacs 第一性原理 (从记忆中写):

;; 1.
;;

;; 2.
;;

;; 3.
;;

;; 4.
;;

;; 5.
;;
```

每条至少 2-3 句话解释。

### Task 3.3: 写下 Frame/Window/Buffer 关系

```
;; Frame:
;; Window:
;; Buffer:
;;
;; 关系:
;;
```

### Task 3.4: 写下你为什么学 Emacs

```
;; 我学 Emacs 是因为:
;;
;;
;;
;;
```

至少 3 个理由。

### Task 3.5: 评分

对比你的速记和 README.md 的原文。

- 5 条第一性原理全写出? [yes/no]
- 解释清楚 (不是只写标题)? [yes/no]
- Frame/Window/Buffer 关系正确? [yes/no]
- 至少 3 个学 Emacs 的理由,且是你自己的 (不是抄书)? [yes/no]

如果 3/4 通过,合格。

### ✓ 子项目 3 检查

- [ ] 5 条第一性原理写出 (不看资料)
- [ ] Frame/Window/Buffer 关系正确
- [ ] 至少 3 个你自己的学 Emacs 理由

---

## 子项目 4: 第一次"住在 Emacs 里" (30 分钟)

### Task 4.1: 选一个真实任务

挑一个你今天要做的小任务:
- 写一封邮件草稿
- 整理一份 todo
- 记录一个想法
- 给一段代码写注释
- 学习一个新概念

### Task 4.2: 完全在 Emacs 里完成

要求:
- 用 Emacs 创建文件 (`C-x C-f`)
- 写内容
- 保存 (`C-x C-s`)
- 不离开 Emacs (不用浏览器、其他编辑器)
- 用 `*scratch*` 记录你的思考

### Task 4.3: 用 help system

完成过程中,如果遇到 "Emacs 怎么做 X":
- 不要 Google
- 用 `C-h a` (apropos) 搜命令
- 用 `C-h i` 进 Info
- 用 `C-h f` 查函数

记录你查了什么:

```
1. 我用 C-h a 搜了: ____, 找到: ____
2. 我用 C-h f 查了: ____
3. 我用 C-h i 进了: ____ 手册的 ____ 章节
```

### Task 4.4: 写日志

打开 `logs/module-00.md` (如果不存在就创建),写一篇学习日志:

```markdown
# Module 0 学习日志

**日期**: 2026-XX-XX
**用时**: ___ 小时
**完成度**: ___%

## 我学到的最重要的概念

[3-5 段]

## 让我"啊哈"的瞬间

[具体时刻]

## 我踩的坑

[列表]

## 我的疑问 (留给后面 Module 解决)

[列表]

## 下一步

进入 Module 1: 编辑生存
```

### ✓ 子项目 4 检查

- [ ] 选了真实任务,完全在 Emacs 里完成
- [ ] 用了 help system 至少 3 次
- [ ] 写了 `logs/module-00.md` 学习日志

#### 为什么"住在 Emacs 里"是 Module 0 的终极测试

考虑你完成这个子项目的过程:
- 你**没有切到浏览器** Google
- 你**没有切到其他编辑器** 抄代码
- 你**用 Emacs 内置工具** (help、Info、apropos) 解决问题
- 你**用 Elisp 探索** (在 `*scratch*` 里 eval)

这是 Emacs 用户**真正的工作方式**。一旦你能在一个真实任务里**完全**用 Emacs 完成,你就跨过了"会按键位"到"会用 Emacs"的门槛。

#### 创造性应用: 这次任务的"啊哈时刻"

如果你卡住了,以下是几个真实"啊哈"的可能:

**啊哈 1: 发现 `C-h a` 能搜命令**

你想"如何统计 buffer 的词数",不知道函数名。`C-h a count RET`——找到 `count-words`、`count-words-region`、`count-matches` 等命令。**完全自给自足**。

**啊哈 2: 发现 `M-!` 能跑 shell 命令**

你想"看看当前目录有哪些文件",`M-! ls -la RET`——shell 输出插到 buffer。**不需要切终端**。

**啊哈 3: 发现 Info 手册比博客详细**

你查"org-mode 怎么用",`C-h i m Org Mode RET`——内置 Org 手册 1000+ 页,**比任何博客都全**。

**啊哈 4: 发现 `*scratch*` 是 REPL**

你想"试一下 `format-time-string` 怎么用",`*scratch*` 里 `(format-time-string "%Y-%m-%d")`,`C-x C-e`,立即看到 `"2026-06-20"`。**5 秒搞定**。

这四个"啊哈"循环是 Emacs 用户每天的日常。一旦你习惯了,你会**反感其他编辑器**——它们没有这种"自给自足"的循环。

---

## 最终评分 (4 个子项目加起来)

| 子项目 | 满分 | 你的得分 |
|---|---|---|
| 1. 环境验证 | 25 | /25 |
| 2. init.el 完整化 | 30 | /30 |
| 3. 心智模型速记 | 25 | /25 |
| 4. 第一次住在 Emacs 里 | 20 | /20 |
| **总分** | **100** | **/100** |

- 80+: 合格毕业,进入 Module 1
- 60-80: 重做薄弱部分
- <60: 重读 README.md 和 concept-anchor.md,理解后再做

---

## 不要作弊

- 不要复制粘贴配置 (除非完全理解)
- 不要 Google 答案 (用 `C-h`)
- 不要跳过任何 task
- 不要"以后再做"——现在做

---

## 完成后

1. 更新 `PROGRESS.md`: Module 0 状态 ✅
2. 在 `logs/module-00.md` 写最终心得
3. 准备进入 Module 1 (`01-survival/README.md`)

恭喜! 你已经从"Emacs 当记事本"升级到"Emacs 极客的起点"。
真正的旅程从这里开始。

#### 完成后你应该有的"新世界观"

完成 Module 0 后,你应该有以下思维转变:

1. **从"按键位"到"调函数"**: 看到任何 Emacs 行为,你不再问"按什么键",你问"这调用什么函数"。函数可以 `C-h f` 查,可以重定义。

2. **从"配置文件"到"程序"**: 你的 init.el 不是"settings.json",是小型 Elisp 程序。你可以加逻辑、加条件、加宏。

3. **从"打开的文件"到"buffer"**: 你不再把文件、终端、目录当不同东西,它们都是 buffer,共享同一套编辑原语。

4. **从"读文档"到"自文档"**: 你不需要 Google,你用 `C-h` 系列。文档和代码同源,不会漂移。

5. **从"重启生效"到"热重载"**: 改任何东西不需要重启,`C-x C-e` 立即生效。你的工作流是连续的。

6. **从"用别人写的"到"自己写"**: 不存在"等作者加功能",任何小需求你都能 5 分钟写出来。

这 6 个转变是 Emacs 的"思考方式"。Module 1-8 会在这基础上系统深入。**你已经跨过了最难的一关——建立心智模型**。

#### 下一步: Module 1 的核心

Module 1 是"编辑生存",教你**键盘流**——默认键位的精确使用,不依赖鼠标。这是把 Module 0 的"心法"应用到**手感**上。

完成 Module 1 后,你能在 Emacs 里**飞快**地编辑文本,完全不用鼠标,效率超过 VS Code。这是 Emacs 的"基本盘"。

加油!

---
