# Exercises: Module 0 心法与装备

> 完成所有 exercises 后,你应该能不假思索地回答所有毕业检查题。
> 用时: ~2 小时
> 不要跳过任何一题。

---

## 第一组: 心智模型 (10 题)

### A1. 五条第一性原理

把"Emacs 是 Lisp 机器伪装成编辑器"这条心智模型,用**自己的话**写在下面:

```
[你的回答]
```

提示: 试着用"VS Code 是 X,Emacs 是 Y"的对比格式。

#### 引导思考 (不是答案,是问题)

在回答前,先想这几个问题:

1. 当你打开 VS Code,主进程在干什么? 当你打开 Emacs,主进程在干什么?
2. VS Code 的扩展和主进程是同一个进程吗? Emacs 的"扩展"和主进程呢?
3. VS Code 的 `settings.json` 改完要做什么? Emacs 的 `init.el` 改完要做什么?
4. VS Code 的扩展 API 是白名单还是黑名单? Emacs 的"API"是什么?

把这些问题的答案串起来,你就自然得出"VS Code 是 X,Emacs 是 Y"的对比。

**这种引导思考方式是 Module 0 的核心方法**——我们不给你答案,我们给你**问对的问题**。

### A2. Buffer 的定义

技术层面,一个 buffer 由哪些组件构成? 列出至少 5 个:

```
1.
2.
3.
4.
5.
```

### A3. 自文档化的证据

打开 Emacs,完成这些操作并记录你看到了什么:

```
操作: C-h k C-g
看到: [填你看到的]

操作: C-h f find-file RET
看到: [填你看到的]

操作: C-h v kill-ring RET
看到: [填你看到的]

操作: C-h m
看到: [填你看到的]
```

### A4. Live system 演示

在 `*scratch*` 里完成:

```elisp
;; 1. 定义一个函数 my-square,接受一个数,返回平方
(defun my-square (n)
  ;; 把这里填完
  )

;; 2. eval 这个定义 (C-x C-e 在末尾)

;; 3. 调用 (my-square 5),应该在 minibuffer 看到 25
(my-square 5)
```

#### 引导思考

做完后,试这些"创造性扩展":

```elisp
;; 4. 不重启,直接改函数让它接受字符串 (parse 成数字)
(defun my-square (n)
  (if (stringp n)
      (setq n (string-to-number n)))
  (* n n))

;; 5. eval 新定义,然后试:
(my-square "7")     ; 应该输出 49
(my-square 7)       ; 应该输出 49

;; 6. 再改,让它返回带前缀的字符串
(defun my-square (n)
  (format "结果: %d" (* n n)))

(my-square 5)       ; 应该输出 "结果: 25"
```

**观察**: 你**从未重启 Emacs**,每次 eval 后函数立即变了。这是"Live System"的本质。

对比 VS Code: 写一个扩展函数,要重启 extension host,所有 buffer 状态丢失。

### A5. Frame/Window/Buffer 关系图

在纸上或 ASCII 画一个 Emacs 屏幕图,标出 Frame、Window、Buffer、Mode line、Echo area、Minibuffer。
然后把你画的图描述写下来 (或拍照)。

### A6. Mode line 解读

打开任意文件,看 mode line,解读每一部分:

```
你的 mode line:  [填你看到的]

例如: -U-:**-F  foo.txt   Top L1    (Fundamental) ----

- U: 
- :**: 
- F:
- foo.txt:
- Top:
- L1:
- (Fundamental):
```

### A7. Point 和 Mark

回答:

1. point 是什么? (技术定义)
2. mark 是什么?
3. region 是什么?
4. `C-SPC C-SPC` (双击)做什么?
5. `C-u C-SPC` 做什么?

### A8. 历史时间线

填空:

- ____ 年,Stallman 和 ____ 在 MIT 用 ____ 宏集合做出第一版 EMACS
- ____ 年,GNU Emacs 1.0 发布,用 ____ 写核心
- ____ 年,XEmacs fork
- ____ 年,package.el 引入 (Emacs ____)
- ____ 年,native compilation 正式 (Emacs ____)

### A9. 为什么选 Lisp

为什么 Stallman 1985 年重写 Emacs 时选了 Lisp? 给出至少 2 个理由:

```
1.
2.
```

### A10. 自由软件

GPL v3+ 的"4 项自由"是哪 4 项?

```
0:
1:
2:
3:
```

---

## 第二组: 编译安装验证 (5 题)

### B1. 验证 Emacs 版本

在 shell 里:

```bash
emacs --version
```

你应该看到什么?

```
[填你看到的]
```

如果不是 30.x,你需要重新装或编译。

### B2. 验证 native compilation

```bash
emacs --batch --eval='(message "native: %s" (if (native-comp-available-p) "yes" "no"))'
```

应该输出什么?

```
[填你看到的]
```

如果 `no`,说明你装的 Emacs 没开 native comp,重新编译加 `--with-native-compilation`。

### B3. 验证 tree-sitter

```bash
emacs --batch --eval='(message "tree-sitter: %s" (if (treesit-available-p) "yes" "no"))'
```

应该输出什么?

```
[填你看到的]
```

### B4. 启动时间

```bash
time emacs --batch --eval='(message "done")'
```

启动时间是多少?

```
[填你看到的]
```

应该 < 0.5 秒 (native comp 开了的话)。如果 > 2 秒,init.el 有问题或没开 native。

### B5. 干净启动测试

```bash
emacs -Q
```

在干净启动的 Emacs 里:
- 没有工具栏? [yes/no]
- 没有 menu bar? [yes/no]
- 是默认主题? [yes/no]
- 不会加载你的 init.el? [yes/no]

---

## 第三组: 内置教程 (1 题,30 分钟)

### C1. 完成 `M-x help-with-tutorial`

启动 Emacs,按 `C-h t` 进入官方教程。

教程大约 40 个小节,每节教你一两个键。**全部做完**,大约 30-45 分钟。

完成后,你应该掌握:
- [ ] 基本移动 (`C-p`/`C-n`/`C-b`/`C-f`)
- [ ] 词、句、段移动
- [ ] 文件操作 (`C-x C-f`/`C-x C-s`)
- [ ] buffer 切换 (`C-x b`/`C-x C-b`)
- [ ] 窗口分屏 (`C-x 2`/`C-x 3`/`C-x o`)
- [ ] 退出 (`C-x C-c`)
- [ ] 数字前缀 (`C-u` 和 `M-num`)
- [ ] 取消 (`C-g`)

记录你用了多久: ___ 分钟

记录你的感受 (什么最反直觉? 什么最爽?):

```
[你的感受]
```

---

## 第四组: Help System 实操 (10 题)

> 这是 Module 0 最重要的实操。Module 1 会更系统学 help system,这里先预热。

### D1. 查函数

`C-h f` 然后输入 `find-file` RET。

回答:
- `find-file` 是哪个文件里的函数? ____
- 它是 interactive 吗? ____
- 它的 docstring 第一句是什么? ____

### D2. 查变量

`C-h v` 然后输入 `kill-ring` RET。

回答:
- `kill-ring` 的当前值是什么? (前 3 项即可) ____
- 它的默认值是? ____
- 谁设的? ____

### D3. 查键位

`C-h k` 然后按 `C-g`。

回答:
- `C-g` 跑的是哪个命令? ____
- 这个命令做了什么 (1 句话)? ____

### D4. 反向查键位

`C-h w` 然后输入 `save-buffer` RET。

回答:
- `save-buffer` 绑定到哪些键? ____

### D5. 查当前 mode

`C-h m` 在 `*scratch*` buffer 里。

回答:
- `*scratch*` 的 major mode 是什么? ____
- 它有什么键位 (列 3 个)? ____
- 有哪些 minor mode 是 active 的? ____

### D6. 查所有键位

`C-h b` (describe-bindings)。

回答:
- `C-c` 开头的键有几个? ____
- `C-x` 开头的键有几个? ____

### D7. 进入 Info

`C-h i` 进入 Info 目录。

按 `m emacs RET` 进入 Emacs manual。

按 `m Kill and Yank RET` 看 Killing 章节。

按 `l` (lowercase L) 返回。

按 `q` 退出 Info。

### D8. Apropos 搜索

`C-h a` (apropos-command) 然后输入 `buffer` RET。

这是"按关键字搜命令"。回答:
- 找到几个命令? ____
- 列 3 个你觉得有用的: ____

### D9. 查 option

`C-h o` 然后输入 `rectangle` RET。

这是查 symbol (变量或函数)。回答:
- 找到什么? ____

### D10. Info 里查 Lisp 教程

`C-h i m Emacs Lisp Intro RET`。

这是 Chassell 的入门书 Info 版本 (Module 3 你会用它)。

按 `n` (next node) 翻几页感受。
按 `q` 退出。

---

## 第五组: 写配置实操 (5 题)

### E1. 找到你的 init.el

```bash
ls -la ~/.config/emacs/init.el 2>/dev/null || ls -la ~/.emacs.d/init.el 2>/null || ls -la ~/.emacs 2>/dev/null
```

记录哪个存在:

```
[填你的]
```

### E2. 写入最小配置

把 README.md 中的最小 init.el 写到你的 `init.el`。
保存后 `M-x eval-buffer RET`。

验证:
- [ ] 没有工具栏
- [ ] 没有滚动条
- [ ] 行号显示
- [ ] modus-vivendi 主题
- [ ] 字体改成 JetBrains Mono (或其他你有的等宽字体)

### E3. 自定义一个键

在 init.el 里加:

```elisp
(global-set-key (kbd "C-c t") 'load-theme)
```

eval 后,按 `C-c t` 应该提示选主题。

回答:
- `kbd` 函数干啥的? (用 `C-h f kbd RET` 查)
- `global-set-key` 接受几个参数?

### E4. 自定义一个函数

在 init.el 里加:

```elisp
(defun my-insert-date ()
  "Insert current date at point."
  (interactive)
  (insert (format-time-string "[%Y-%m-%d %a]")))

(global-set-key (kbd "C-c d") 'my-insert-date)
```

eval 后,在任意 buffer 按 `C-c d`,应该插入今天的日期。

回答:
- `interactive` 这一行的作用? (用 `C-h f interactive RET`)
- `format-time-string` 接受什么参数?

### E5. 解释每一行

对你的 init.el,**每一行**都用 `C-h f` 或 `C-h v` 查文档。
然后写下每行的作用:

```elisp
;; 例如:
(setq inhibit-startup-screen t)
;; → inhibit-startup-screen 是变量,t 是真值
;; → 作用: 启动时不显示欢迎屏
;; → 用 C-h v inhibit-startup-screen RET 查文档
```

对你的 init.el 做同样的分析 (写在你 init.el 的注释里)。

---

## 第六组: 反向题 (10 题,考验心智)

> 这组题不是"做",是"想"。

### F1. 如果让你重新设计 Emacs,你会保留 Lisp 底座吗?

写 3-5 句你的看法:

```
[你的回答]
```

#### 引导思考

回答前考虑这些子问题:

1. 如果不用 Lisp,Emacs 还能做到"自文档化"吗? (`C-h f` 需要从哪里取 docstring?)
2. 如果不用 Lisp,Emacs 还能做到"Live system"吗? (改代码不重启需要什么条件?)
3. 如果用 Python/Ruby 替代 Lisp,会有什么后果? (这些语言不是 homoiconic,宏不行)
4. 如果用 Lua/JavaScript,启动会快多少? 但失去什么?
5. RMS 1985 年选 Lisp,2026 年如果你重写,会选什么? 为什么?

**这类问题的价值不在"对错",在于训练你的设计思维**。即使你选"用 Python",你也要能解释为什么放弃 Lisp 的所有优势。

### F2. VS Code 的"配置 JSON"和 Emacs 的"配置 Elisp"本质区别是?

```
[你的回答]
```

#### 引导思考

具体回答这些:

1. JSON 能写 `if` 吗? 能写 `loop` 吗? 能定义函数吗?
2. JSON 改完要做什么? Elisp 改完要做什么?
3. VS Code 的 keybindings.json 是数据还是代码? Emacs 的 `(global-set-key ...)` 是数据还是代码?
4. 如果想"在 macOS 上把 Cmd 当 Ctrl",VS Code 怎么做? Emacs 怎么做? 哪个更灵活?
5. 哪个能表达"如果时间在 6-18 点用浅色主题,否则深色"? 哪个不能?

**结论**: JSON 是"声明式数据",Elisp 是"命令式代码"。前者简单但僵化,后者复杂但灵活。Emacs 选择后者,代价是入门门槛高,收益是天花板无限。

### F3. 为什么 Emacs 能存活 50 年?

写至少 3 个理由:

```
1.
2.
3.
```

### F4. 假设你完全不会 Elisp,你能用 Emacs 做什么?

```
[你的回答]
```

### F5. 假设你完全会 Elisp,你能用 Emacs 做什么 (不一样的事)?

```
[你的回答]
```

### F6. `*scratch*` buffer 为什么默认是 `lisp-interaction-mode`?

```
[你的回答]
```

(提示: 它和 eval-last-sexp 有关)

#### 引导思考

回答前考虑:

1. `*scratch*` 是关联文件的 buffer 吗? 不是。那它存在的目的是什么?
2. 启动 Emacs 时为什么不给个空白文档,而给一个 Lisp 交互环境?
3. `lisp-interaction-mode` 比 `emacs-lisp-mode` 多了什么? (用 `C-h m` 在两个 mode 里对比)
4. `C-x C-e` 在 `lisp-interaction-mode` 里能 eval,在 `text-mode` 里能吗? 试试。

**答案的方向**: Emacs 假设用户**愿意**写 Lisp——`*scratch*` 不是"草稿纸",是"REPL"。这反映了 Emacs 的根本态度: **你不是消费者,你是创造者**。

### F7. 为什么 Emacs 默认配置这么"难用"?

```
[你的回答]
```

(提示: 默认配置要兼容所有用户,包括没装 GUI 的 TTY)

#### 引导思考

具体回答:

1. 默认开 menu bar / tool bar: 是为了谁? (GUI 新手)
2. 默认没开行号: 为什么? (TTY 显示行号占宽度,1985 年 80 列宝贵)
3. 默认 `yes-or-no-p` 而非 `y-or-n-p`: 为什么? (避免误操作)
4. 默认开 backup files (`~`): 为什么? (数据安全 > 干净目录)
5. 默认没 evil-mode: 为什么? (Emacs 设计不假设 Vim 用户)

**深层原因**: Emacs 默认值是 **50 年的累积妥协**——每个选择都有理由,但合起来对现代用户不友好。这就是为什么本教程教你**第一件事就是改默认**——理解默认值的理由,然后按你的工作流改。

### F8. 你预测 Emacs 50 年后还存在吗? 为什么?

```
[你的回答]
```

### F9. 为什么很多高级程序员 (Stallman, McCarthy, Graham) 都喜欢 Lisp?

```
[你的回答]
```

### F10. 你学完这个教程后,最想做的第一件事是什么?

```
[你的回答]
```

---

## 答题方式

不要在脑子里"想"答案,**写下来** (在文本编辑器或纸上)。
写不出来 = 没真懂。重读 README.md 或 concept-anchor.md 相关章节。

---

## 完成标准

- [ ] 全部 41 题认真回答 (不一定全对)
- [ ] 第一组至少 7/10 自信答对
- [ ] 第三组 (内置 tutorial) 完整跑完
- [ ] 第四组至少 8/10 能完成操作
- [ ] 第五组的 init.el 至少 20 行,每行有 `C-h` 解释
- [ ] 第六组至少认真思考过每一题

完成后:
1. 在 `logs/module-00.md` 写心得 (3-5 段)
2. 进入 `00-mindset/capstone.md` 做毕业项目
