# Misc + Amusements: Emacs 的"工具 + 玩具"双重性

> **Ref 章节**: emacs-manual/misc.texi (Amusements, Hyperlinking, Sorting, Emacs Server)
> **目标**: 理解为什么一个 40 年的编辑器内置了俄罗斯方块和 Eliza

---

## 0. 第一性原理: 为什么编辑器要内置游戏?

这听起来像无关紧要的"彩蛋",但其实是 Emacs 文化的核心问题。

VSCode 的设计哲学是"工具"——它严格专注于编辑代码,任何"非编辑"功能 (游戏、心理分析师、动画) 都被排斥。这是产品思维——工具的边界要清晰。

Emacs 的设计哲学是**"工具 + 玩具"**——它不区分。游戏和编辑器共享同一套 primitive (buffer、window、keymap、face),所以加游戏**几乎零成本** (每游戏 ~200 行 Elisp)。这种"无边界"让 Emacs 既是 IDE、又是邮件客户端、又是俄罗斯方块机。

更深层: **toy 是 tool 的雏形**。`M-x tetris` 用到的 redisplay、keymap、timer,正是真实工具 (Magit、Org) 用的。学游戏代码你能学到真实模式。GNU Emacs Manual 把 Amusements 单列一节 (`misc.texi` 的 "Games and Other Amusements"),不是凑数——是认可这种"玩中学"哲学。

Stallman 一直强调: 软件应该是 playground,不是 factory。Emacs 体现这一点。

---

## 1. Amusements (游戏)

### 1.1 列表

```
M-x tetris           俄罗斯方块 (经典 7 形状)
M-x snake            贪吃蛇 (越吃越长)
M-x pong             Pong (双人,用 keyboard)
M-x doctor           Eliza-style 精神分析师 (会话 AI)
M-x psychoanalyze-pinhead   长跑 doctor,用 pinhead 名言
M-x dunnet           文字冒险游戏 (Zork 风格)
M-x blackbox         推理游戏 (找隐藏原子)
M-x gomoku           五子棋 (vs Emacs)
M-x life             Conway's Game of Life (元胞自动机)
M-x solitaire        孔明棋 (Peg solitaire)
M-x mpuz             数学乘法谜题
M-x 5x5             Lights Out 谜题
M-x bubbles          消除游戏 (现代)
```

这些都在 Emacs Lisp 写的——源码在 `lisp/play/`。每个游戏 200-500 行 Elisp,展示如何用 Emacs primitive 做交互式 UI。

### 1.2 tetris (俄罗斯方块)

```
M-x tetris
```

操作: 方向键移动,上箭头旋转,空格直降。`g` 重新开始,`p` 暂停,`q` 退出。

tetris 展示 Emacs 的**定时器 + buffer drawing**——每 ~1 秒 timer 触发,redraw buffer 显示新状态。这是所有实时游戏的标准模式。

源码: `lisp/play/tetris.el` (~400 行)。读它你能学到: 怎么用 text property 显示方块,怎么设 timer,怎么处理按键。

### 1.3 doctor (Eliza 精神分析师)

```
M-x doctor
```

这是经典 Eliza——1966 年 Joseph Weizenbaum 写的"假装心理治疗师"程序。你输入"我觉得沮丧",它回"为什么你觉得沮丧?"。规则简单 (模式匹配),但效果出奇——人感觉被"倾听"。

Emacs 的 doctor 不只是娱乐——它展示了**pattern matching + 简单 state machine** 的力量。读 `lisp/play/doctor.el` 你能看到一个 ~600 行的"AI"——没有 ML,没有 LLM,只有规则。

Weizenbaum 后来写 "Computer Power and Human Reason" 警告 AI 的"伪智能"危险——doctor 是他的反面教材。但 Emacs 社区把它当玩具保留,提醒"1985 年的 AI 就这水平"。

### 1.4 dunnet (文字冒险)

```
M-x dunnet
```

dunnet 是一个完整的文字冒险游戏 (Zork 风格)——你用英文命令探索世界 (`go north`、`take key`、`unlock door`)。游戏有 ~10 个谜题,通关需 1-2 小时。

dunnet 的特殊价值: 它**展示 Elisp 能写复杂状态机**。所有房间、物品、谜题状态都在 Lisp 数据结构里。读 `lisp/play/dunnet.el` (~1500 行) 你能学到如何用 Elisp 写"stateful interactive fiction"。

很多 Emacs 老用户用 dunnet 作为"教程工具"——它强迫你学 Emacs 的交互模式 (minibuffer 输入、buffer 显示、key binding)。

### 1.5 life (康威生命游戏)

```
M-x life
```

Conway's Game of Life——元胞自动机。简单的规则 (活细胞 2-3 邻居活,死细胞 3 邻居复活) 产生复杂行为。

Emacs 的 life 展示**buffer 作为 grid 模拟器**——每个字符是一个 cell,redisplay 是 frame。读源码你能学到怎么高效 update buffer (只改变化 cell,不全 redraw)。

### 1.6 gomoku / 5x5 / mpuz / solitaire / blackbox (单机谜题)

这些是 puzzle——单局 5-30 分钟。每个用不同 Emacs primitive:
- **gomoku**: 简单 AI (minimax)
- **5x5**: linear algebra 解 Lights Out
- **mpuz**: 数字谜题
- **solitaire**: board 状态
- **blackbox**: logic deduction

每个都是"小而完整"的 Elisp 案例。

---

## 2. Animation

### 2.1 animate

```
M-x animate RET "Hello, Emacs!" RET
```

`animate` 在 buffer 里逐字符显示字符串——每个字符在不同位置浮现,然后 settle。是动画效果。

源码 `lisp/play/animate.el`——展示怎么用 `sit-for` + `goto-char` + `insert` 做字符级动画。

### 2.2 animate-birthday-present

```
M-x animate-birthday-present
```

是 animate 的预设版本——展示一个"生日快乐"动画。可以改源码做自定义。

---

## 3. Zones (zone-mode 屏保)

```
M-x zone
```

`zone` 是 Emacs 的"屏保"——buffer 内容开始扭曲、变形、消失。看起来像 CRT 损坏。

源码 `lisp/zone.el` + `lisp/zone-mg.el`。zone 是 Elisp 写"screen distortion"的练习——它临时改 text property 让字符显示不同。

可以设 idle 触发:

```elisp
(require 'zone)
(zone-when-idle 60)   ; 60 秒 idle 触发
```

---

## 4. Yow lines

```
M-x yow
```

`yow` 随机吐一句 Zippy the Pinhead 名言——例如 "All of life is a big surprise to a clam."。这是 Emacs 内置的 quote 库,~200 句。

yow 价值: 它是**obarray + random** 的最简单 demo——所有名言存在 list,random 选一句。读 `lisp/play/yow.el` (~50 行) 你能学 Elisp 字符串处理。

---

## 5. Animate cursor

```elisp
(run-with-idle-timer 1 t (lambda () (message "%s" (random 1000))))
```

(自定义) 你可以用 `run-with-idle-timer` 做 cursor 动画——这非内置,但 Elisp 几行就能写。

---

## 6. Hyperlinking (info-lookup, ffap)

misc.texi 的 "Hyperlinking" 节讲 Emacs 的超链接系统:

```
M-x ffap                  find file at point
M-x browse-url-at-point   打开 point 处的 URL
M-x goto-address-mode     让 buffer 里的 URL/email 可点击
```

`goto-address-mode` 让任何 buffer 变成"超链接文档"——它扫 URL/email,加 face + keymap,鼠标或 RET 点击打开。这是 Org 的 link 系统的简化版。

---

## 7. Sorting

```
M-x sort-lines            按字母排
M-x sort-numeric-fields   按数字排
M-x sort-fields           按 N 列排
M-x sort-regexp-fields    按正则提取的 key 排
M-x sort-pages            按 page 分隔排
M-x sort-paragraphs       按段落排
M-x reverse-region        反转
```

`sort-regexp-fields` 最强——它用正则提取 "sort key",按 key 排,但保留原始行。比如按 CSV 第二列排:

```
M-x sort-regexp-fields RET ^[^,]*,\\([^,]*\\) RET \1 RET
```

---

## 8. Recursive Edit

```
C-]     abort-recursive-edit
M-x top-level     退出所有 recursive edit
```

`recursive-edit` 让你在 minibuffer 操作时临时切回主 buffer——改点东西,然后 `C-M-c` (exit-recursive-edit) 回 minibuffer。复杂但强大。

---

## 9. Batch Mode

```
emacs --batch --eval "(message \"hello\")"
```

`--batch` 让 Emacs 跑完命令退出,不开 UI。这是用 Emacs 做"脚本引擎"——你能在 shell 调 Emacs 的 elisp。

```bash
# 用 Emacs 解析 JSON
emacs --batch --eval '
  (require '"'"'json)
  (princ (json-encode '"'"'(("key" . "value"))))'

# 批量改文件
emacs --batch --eval '
  (dolist (f (directory-files-recursively "." "\\.txt$'"'"'"))
    (with-temp-buffer
      (insert-file-contents f)
      (while (re-search-forward "foo" nil t) (replace-match "bar"))
      (write-region (point-min) (point-max) f)))'

# byte-compile 全目录
emacs --batch -f batch-byte-compile *.el

# 跑 ERT 测试
emacs --batch -l my-test.el -f ert-run-tests-batch-and-exit
```

batch 模式是 Emacs 的"CI 后端"——所有 MELPA 包的 CI 用它。你能在 shell 用 Elisp,不需要学别的语言。

---

## 10. Emacs Server / emacsclient

```
M-x server-start           启动 Emacs server
shell: emacsclient FILE    用运行的 Emacs 打开 FILE
shell: emacsclient -n -c   开新 frame,不阻塞
```

Emacs server 让你**复用一个 Emacs 进程**——避免每个文件开新 Emacs (启动慢)。这是长期使用 Emacs 的标配:开机启动一个 server,之后所有 `e FILE` 用 emacsclient。

```bash
# 在 .bashrc / .zshrc
export ALTERNATE_EDITOR=""   # 没 server 就开新的
alias e="emacsclient -n"
```

---

## 11. 15+ 练习

### Ex A.1: tetris
```
M-x tetris
```
玩 5 局,记录最高分。

### Ex A.2: snake
```
M-x snake
```
记录最长 snake。

### Ex A.3: pong
双人 (Player 1: W/S, Player 2: 方向键),玩 3 局。

### Ex A.4: doctor
```
M-x doctor
```
和 doctor 对话 5 分钟。最后它说什么?

### Ex A.5: psychoanalyze-pinhead
```
M-x psychoanalyze-pinhead
```
让 pinhead 自言自语。看 5 句。

### Ex A.6: dunnet
```
M-x dunnet
```
玩到通过第一个谜题 (拿 office 的 floppy)。

### Ex A.7: blackbox
```
M-x blackbox
```
解一个 board。

### Ex A.8: gomoku
```
M-x gomoku
```
vs Emacs,赢一局。

### Ex A.9: life
```
M-x life
```
看 100 代演化。

### Ex A.10: solitaire
```
M-x solitaire
```
解一局 (剩 1 个棋子)。

### Ex A.11: mpuz
```
M-x mpuz
```
解 3 个谜题。

### Ex A.12: 5x5
```
M-x 5x5
```
解 5x5 Lights Out。

### Ex A.13: bubbles
```
M-x bubbles
```
玩 3 局。

### Ex A.14: animate
```
M-x animate RET "Hello!" RET
```

### Ex A.15: zone
```
M-x zone
```
让 buffer 扭曲 5 秒。

### Ex A.16: yow
```
M-x yow
```
按 5 次,记下最有意思的一句。

### Ex A.17: batch eval
```bash
emacs --batch --eval '(message (+ 1 2 3))'
```
输出什么?

### Ex A.18: emacsclient
```
M-x server-start
# shell:
emacsclient -n /tmp/test.txt
```
确认用同一 Emacs 打开。

---

## 12. 创造性例子 (5+)

**1. 用 dunnet 学交互式 fiction 设计**: 读 `lisp/play/dunnet.el`,看它怎么建模世界 (alist of rooms/items)。然后写自己的 mini text adventure (10 个房间)。

**2. animate 自定义祝福**: 改 `animate-birthday-present` 显示同事名字 + 生日,放到 init.el 自动在他生日打开。

**3. batch 模式跑 elisp 工具链**: 写一个 CI 脚本——`emacs --batch` 跑 byte-compile + ERT + lint。这是 MELPA recipe 的核心模式。

**4. 用 life 模拟 cellular automata**: 改 `life` 的规则 (B3/S23 → B36/S23,HighLife),看不同规则下模式。

**5. 把 doctor 改成 "code reviewer"**: 改 doctor.el 的 pattern——你输入"我的代码不工作",它回"为什么你觉得你的代码不工作?"。轻松的 review 同事。

**6. 用 zone 做番茄钟**: `zone-when-idle 1500` (25 分钟)——番茄钟结束 buffer 开始扭曲,强制你休息。

**7. batch + Elisp 做 JSON 转换工具**:
```bash
emacs --batch --eval '
  (require (quote json))
  (let ((data (json-read-from-string
               (with-temp-buffer
                 (insert-buffer-substring (current-buffer))
                 (buffer-string)))))
    (princ (alist-get "name" data nil nil (quote equal))))' < input.json
```

**8. tetris 高分持久化**: 改 `tetris.el` 把高分写到文件,每次启动显示。

---

## 13. 自测

1. 为什么 Emacs 内置游戏?
2. `M-x doctor` 是什么?
3. dunnet 是什么类型的游戏?
4. `--batch` 干啥?
5. emacsclient 优势?
6. zone 是什么?

**答案**:
> 1. "工具 + 玩具"双重性,展示 Emacs primitive 灵活
> 2. Eliza 精神分析师,1966 年的程序
> 3. 文字冒险游戏 (Zork 风格)
> 4. 跑完命令退出,无 UI,适合脚本
> 5. 复用一个 Emacs 进程,启动快
> 6. 屏保,buffer 内容扭曲动画

---

## 14. 下一步

回到 `concept-anchor.md` 或 Module 6 学更深 Elisp。
