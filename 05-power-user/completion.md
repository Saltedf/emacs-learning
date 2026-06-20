# Completion Stack 详解 (Vertico/Corfu/...)

> 这文件详细讲补全栈的每个组件

Emacs 的补全系统是它在 2020 年代最让人惊艳的设计——一套"小而美"的包组合起来,效果超过任何 IDE 的补全。这一节深度讲解每个组件,以及它们如何协作。

---

## 1. Emacs 补全的架构

### 1.1 三层

理解 Emacs 补全的第一步是"分层"——它不是单一系统,而是三层独立组件的协作:

```
1. Minibuffer (输入命令/参数)
   ↓
2. In-buffer completion (代码补全)
   ↓
3. UI (popup, vertical list, ...)
```

每层独立,可换。

这三层各自独立:
- **Minibuffer**: 你输入命令 (`M-x`)、文件路径 (`find-file`)、buffer 名 (`switch-to-buffer`) 的地方。这里用的是 minibuffer 补全。
- **In-buffer completion**: 你写代码时,在 buffer 内弹出的补全。这里用的是 completion-at-point。
- **UI**: 上面两层的"显示层"——用什么方式显示候选 (popup 框 vs 垂直列表)。

这种分层让组件可替换——比如 minibuffer UI 你可以选 Vertico (垂直)、Icomplete (内置垂直)、Helm (复杂)、Ivy (流行)。每层换不影响其他层。这是 Emacs "工具哲学"的体现。

### 1.2 历史演化

Emacs 补全系统的演化反映了整个 Emacs 社区的演化——从"内置够用"到"第三方强大",再回到"极简内置"。

- Emacs 24 (2012): `icomplete` (内置 minibuffer 增强)
- Emacs 25 (2017): `company` (主流代码补全)
- Emacs 27 (2020): `selectrum` / `ivy` / `helm` (现代 minibuffer)
- Emacs 28 (2022): `vertico` / `corfu` (新一代,极简)
- Emacs 29 (2023): 内置 `use-package`, `eglot`

现代 (2024+) 主流: **Vertico + Corfu + Consult + Orderless + Marginalia + Embark**

这个演化的关键转折是 2021-2022 年 Daniel Mendler 发布的 vertico/corfu——他用极简的代码 (每个包 < 500 行) 实现了 Ivy/Helm 的核心功能。这反映了一种"哲学回归"——Emacs 用户开始反思"大而全"的代价 (重、慢、难维护),回到"小而美"的 Unix 哲学。

---

## 2. completion-styles

### 2.1 是什么

`completion-styles` 是 Emacs 控制匹配算法的变量——它决定你输入的字符串如何匹配候选。这是补全的"核心算法"层。

变量 `completion-styles` 控制匹配算法:

```elisp
completion-styles
;; 默认值: ('basic ...)
```

| Style | 行为 |
|---|---|
| `basic` | 前缀匹配 |
| `partial-completion` | 部分匹配 (用 `*` `?`) |
| `emacs22` | Emacs 22 风格正则 |
| `substring` | 子串 |
| `flex` | 模糊 (字符顺序匹配) |
| `initials` | 缩写 (ih → insert-here) |
| `orderless` | 空格分隔,任意顺序 |

每个 style 是不同的匹配策略——理解它们的差异,才能选合适的:
- `basic`: 只匹配前缀。输入 "ins" 匹配 "insert" 但不匹配 "begin-ins"。
- `partial-completion`: 支持 `*` (任意字符) 和 `?` (单字符)。`ins*set` 匹配 `insert-setting`。
- `substring`: 任意位置的子串。`set` 匹配 `setting` 和 `offset`。
- `flex`: 字符按顺序匹配,但允许中间有其他字符。`is` 匹配 `insert-something` (i...s...)。
- `initials`: 缩写。`ih` 匹配 `insert-here` (取每个词的首字母)。
- `orderless`: 空格分隔的 token,每个 token 都要匹配,但顺序任意。`ins set` 匹配 `insert-setting` 和 `setting-insert`。

### 2.2 用 Orderless

Orderless 是社区主流选择——它最灵活,接近"模糊匹配"的体验:

```elisp
(use-package orderless
  :custom
  (completion-styles '(orderless basic)))
```

输入 `ins set` 匹配 `insert-setting`。
`basic` 是 fallback (当 orderless 匹配空时用)。

为什么 Orderless 这么好? 因为它的匹配方式接近**人脑思考**——你想到一个名字,脑子里浮现两个关键词 (ins 和 set),但记不清顺序。Orderless 让你直接输入这两个词,空格分隔,立刻匹配。这比"必须输入准确前缀"友好得多。

`basic` 作为 fallback 是必要的——某些场景 (比如输入空字符串) orderless 返回空,这时用 basic 避免显示"无候选"。

### 2.3 单独配置

`completion-category-overrides` 让你为不同类别设不同 style——这是精细控制:

```elisp
(completion-category-overrides
 '((file (styles basic partial-completion))
   (buffer (styles basic substring))
   (command (styles orderless))))
```

文件用 basic + partial (支持 `~/.config/em/in.el` 缩写)。
命令用 orderless。

这种"按类别定制"很有用:
- **文件路径**用 partial-completion——长路径用首字母缩写,如 `~/e/i.el` 匹配 `~/emacs/init.el`。
- **命令**用 orderless——记不住完整命令名时,输入几个关键词。
- **buffer**用 substring——只记得 buffer 名的一部分。

这种"针对性优化"是 Emacs 用户的核心技巧——你根据每种补全的使用习惯,定制匹配算法。

---

## 3. Vertico

### 3.1 角色

Vertico 是 **minibuffer UI**——它把候选从默认的"水平补全"改为"垂直列表":

Vertico 是 **minibuffer UI**:
- 把候选**垂直列表**显示
- 实时过滤
- 用方向键 / TAB 选

为什么"垂直列表"这么重要? 因为它**信息密度高**——你能一次看到 10-20 个候选,而不是默认的一个。这让你"挑"候选变成视觉任务 (一眼看到目标),而不是"试错"任务 (按 TAB 一个个看)。

实时过滤是另一个关键——你输入字符,列表立刻过滤。这种"输入即过滤"的反馈,让你不需要"完整输入"——输入一两个字符就够,候选已经缩小到几个。

### 3.2 替代品

minibuffer UI 是 Emacs 生态最丰富的领域之一——有十几个包可选:

| 包 | 风格 |
|---|---|
| Vertico | 极简,垂直 |
| Selectrum | 类似 Vertico,先出现 |
| Ivy | 流行,文件多 |
| Helm | 强大但重 |
| Icomplete | 内置,垂直 |
| Fido-mode | 内置,简化 icomplete |

新人推荐 **Vertico** (现代,极简,与内置一致)。

为什么选 Vertico?
- **极简**: < 500 行代码,你能读完。
- **与内置集成**: Vertico 不重新发明,只用 Emacs 内置的 completion API。这意味着它和任何使用标准 completion 的命令都兼容。
- **无依赖**: Vertico 只依赖 Emacs 内置功能。
- **可组合**: Vertico + Orderless + Consult + Marginalia + Embark,五个包各做一件事。

对比 Helm——Helm 是大包,自己一套 API。它开箱即用,但定制需要读大量代码。Vertico 反过来——开箱很简单,但通过组合 (Orderless, Consult 等) 达到甚至超过 Helm。

### 3.3 Vertico 用法

用 Vertico 时的交互流程:

```
M-x                进入命令补全
输入 "find file"   过滤
向下选
RET                执行
```

不用鼠标。

这是典型的"输入 → 过滤 → 选 → 执行"流程。整个交互只用键盘——你不需要碰鼠标。这种"键盘流"是 Emacs 用户高生产力的来源。

### 3.4 Vertico 配置

```elisp
(use-package vertico
  :init
  (vertico-mode 1)
  :custom
  (vertico-count 15)                 ; 显示 15 行
  (vertico-resize nil)               ; 不调整 minibuffer 大小
  (vertico-cycle t)                  ; 循环
  :bind (:map vertico-map
              ("C-j" . vertico-next)
              ("C-k" . vertico-previous)
              ("C-f" . vertico-scroll-up)
              ("C-b" . vertico-scroll-down)))
```

每个设置都有意义:
- `vertico-count 15`: 显示 15 行候选。你可以设大一点 (20-30),但太大 minibuffer 占太多屏幕。
- `vertico-resize nil`: 不自动调整大小——minibuffer 固定高度,避免抖动。
- `vertico-cycle t`: 在首尾循环。
- 键位 `C-j`/`C-k`: 用 Vim 风格的导航键——一些人 (evil 用户) 习惯这个。默认用 `C-n`/`C-p` 也可以。

### 3.5 vertico-directory

`vertico-directory` 是 Vertico 的一个扩展——专门优化 `find-file` 时的目录导航:

```elisp
(use-package vertico-directory
  :after vertico
  :ensure nil
  :bind (:map vertico-map
              ("RET" . vertico-directory-enter)
              ("DEL" . vertico-directory-delete-char)
              ("M-DEL" . vertico-directory-delete-word))
  :init
  (add-hook 'rfn-eshadow-update-overlay-hook #'vertico-directory-tidy))
```

`vertico-directory` 让 `find-file` 时 `DEL` 在 `/` 边界删整个目录。

这个细节极大改善 `find-file` 体验——默认情况下,`DEL` 删一个字符。但在路径里,你经常想"删整个目录名" (比如 `/home/user/projects/emacs/init.el` 你想退到 `/home/user/projects/`)。`vertico-directory-delete-char` 检测 `/` 边界,一次删一个目录。这让"路径回退"变得飞快。

---

## 4. Consult

### 4.1 角色

Consult 是补全栈的"命令包"——它提供大量"现代化"命令,每个命令用 minibuffer + Vertico + preview 的高质量交互:

Consult 提供"现代化"的版本:
- `consult-line` 替代 isearch
- `consult-buffer` 替代 switch-to-buffer
- `consult-grep` / `consult-ripgrep` 替代 grep
- `consult-yank-pop` 替代 yank-pop
- `consult-imenu` 替代 imenu

Consult 的核心价值是 **preview**——当你在候选列表上下移动时,目标位置实时显示。比如 `consult-line` (搜当前 buffer 的行),你按 `C-s` 输入 pattern,候选显示所有匹配的行——光标移到候选,buffer 自动跳到那一行,你能看到上下文。这种"实时预览"让你**不需要 enter** 就知道候选对不对。

这是 isearch 做不到的——isearch 是"输入即跳转",你不能"看候选再选"。Consult 把它变成"列表 + 预览"模式,信息密度更高。

### 4.2 主要命令

#### consult-line (替代 isearch)

```
C-s        consult-line
;; 输入 pattern,实时跳转,显示候选
```

支持:
- preview (光标移到候选,buffer 也跟着 preview)
- 多行匹配
- avy-style 跳转

`consult-line` 替代了默认的 isearch——它在 minibuffer 显示所有匹配行,你用方向键浏览,buffer 实时预览。比 isearch 的"按 C-s 一个个跳"直观得多。

#### consult-buffer

```
C-x b      consult-buffer
;; 列出 buffer + recent files + bookmarks
```

支持:
- 切 buffer
- 切到 recent file (直接打开)
- 切到 bookmark

`consult-buffer` 是 switch-to-buffer 的"加强版"——一个列表同时显示当前 buffer、最近文件、bookmark。你不需要分别 `switch-to-buffer`、`recentf-open`、`bookmark-jump`——一个 `C-x b` 全搞定。这种"多源聚合"是 Consult 的设计哲学。

#### consult-ripgrep / consult-grep

```
M-s r      consult-ripgrep
;; 项目内 ripgrep
```

支持:
- 输入 pattern 实时搜
- preview 候选
- 跨文件

`consult-ripgrep` 是项目内搜索的杀手锏——你输入 pattern,实时跨文件搜索,候选显示文件名: 行号: 内容,光标移到候选时,buffer 自动打开对应文件并定位。这是"全项目 grep + 即时预览"——比 `grep` 命令快得多。

ripgrep (rg) 是现代 grep 替代——Rust 写的,极快,默认忽略 `.gitignore`。`consult-ripgrep` 包装了它,加 preview。

#### consult-yank-pop

```
M-y        consult-yank-pop
;; kill-ring 候选菜单
```

替代默认 `M-y` 循环。

默认的 `M-y` (yank-pop) 是"循环"——你按 `M-y` 一次,kill-ring 显示下一项,需要按多次才能找到想要的。`consult-yank-pop` 改为列表——所有 kill-ring 项垂直显示,你按方向键选,RET 粘贴。这是巨大的体验提升。

#### consult-imenu

```
M-g i      consult-imenu
;; imenu 候选菜单
```

跳函数定义。

`imenu` 是 Emacs 内置的"buffer 结构导航"——它扫描当前 buffer,列出所有函数/类/变量定义。默认 imenu 是个简单列表,`consult-imenu` 把它变成 Vertico 列表——你能搜索、过滤、preview。在大文件 (几千行) 里非常有用。

#### consult-mark

```
M-x consult-mark           ; 跳 mark ring 候选
```

mark ring 是 Emacs 记录"之前光标位置"的栈——每次你做某些操作 (搜索、跳转),当前位置压栈。`consult-mark` 让你从 mark ring 选一个跳回去——比 `C-u C-SPC` (循环 pop) 直观。

#### consult-theme

```
M-x consult-theme         ; 选主题 (preview)
```

选主题时实时预览——你浏览候选,Emacs 实时切换主题颜色。这让你"看效果再选",而不是"切了不喜欢再切回去"。

### 4.3 consult 配置

Consult 的完整配置——展示所有有用的键位绑定:

```elisp
(use-package consult
  :ensure t
  :bind (;; C-c bindings (general)
         ("C-c h" . consult-history)
         ("C-c m" . consult-mode-command)
         ("C-c b" . consult-bookmark)
         ;; C-x bindings
         ("C-x M-:" . consult-complex-command)
         ("C-x b" . consult-buffer)
         ("C-x 4 b" . consult-buffer-other-window)
         ("C-x r b" . consult-bookmark)
         ;; M-g bindings
         ("M-g g" . consult-goto-line)
         ("M-g M-g" . consult-goto-line)
         ("M-g o" . consult-outline)
         ("M-g m" . consult-mark)
         ("M-g k" . consult-global-mark)
         ("M-g i" . consult-imenu)
         ("M-g I" . consult-project-imenu)
         ;; M-s bindings
         ("M-s f" . consult-find)
         ("M-s F" . consult-locate)
         ("M-s g" . consult-grep)
         ("M-s G" . consult-git-grep)
         ("M-s r" . consult-ripgrep)
         ("M-s l" . consult-line)
         ("M-s L" . consult-line-multi)
         ("M-s k" . consult-keep-lines)
         ("M-s u" . consult-focus-lines)
         ("M-s e" . consult-isearch-history)
         ;; Isearch
         (:map isearch-mode-map
               ("M-e" . consult-isearch)
               ("M-s e" . consult-isearch)
               ("M-s l" . consult-line))
         ;; Yank
         ("M-y" . consult-yank-pop))
  :custom
  (consult-narrow-key "<")               ; 用 < > narrow
  (consult-project-root-function #'project-root)
  :config
  (consult-customize
   consult-theme
   :preview-peek nil
   consult-ripgrep consult-git-grep consult-grep
   consult-bookmark consult-recent-file consult-xref
   :preview-key "M-."
   consult-buffer
   :preview-key "M-."))
```

这个配置体现了 Consult 的几个设计:
- `M-g` 前缀: goto 系列 (goto-line、imenu、mark)。
- `M-s` 前缀: search 系列 (grep、find、line)。
- `C-x b`: 替代默认 switch-to-buffer。
- `M-y`: 替代默认 yank-pop。
- `consult-narrow-key "<"`: 按 `<` 弹出"narrow"菜单——在 consult-buffer 里按 `b` 只看 buffer,`f` 只看文件,`r` 只看 recent。
- `:preview-key "M-."`: preview 默认是"光标移动时自动 preview"。但对某些命令 (grep、ripgrep),自动 preview 太慢,所以改为"按 M-. 手动 preview"——只有按 M-. 才 preview。这是性能和体验的平衡。

---

## 5. Marginalia

### 5.1 角色

Marginalia 是"注释层"——它在每个候选旁边显示元信息。这看起来简单,但极大提升决策效率:

Marginalia 在候选旁边显示元信息:

```
find-file: init.el
           500 lines, Emacs Lisp
find-file: README.md
           148 lines, Markdown
```

这种"一行注释"的价值是减少"打开看"的操作。找文件时,你立刻知道每个文件多大、什么类型。`switch-to-buffer` 时,知道每个 buffer 的 mode——立刻找到那个 Python buffer。`M-x` 时,知道每个命令的 keybinding——找到没绑定的命令。

Marginalia 的设计哲学: **信息密度**——一行额外信息可以省一次完整操作。这是 UI 设计的高级技巧——不增加交互,只增加信息。

### 5.2 配置

```elisp
(use-package marginalia
  :ensure t
  :init
  (marginalia-mode 1)
  :custom
  (marginalia-align 'right)
  (marginalia-ellipsis "…")
  (marginalia-field-width 50))
```

`marginalia-align 'right`: 注释右对齐——让候选名整齐,注释在右边。这是美学选择。
`marginalia-ellipsis "…"`: 注释太长时用省略号截断。
`marginalia-field-width 50`: 注释区域最大宽度——避免长注释挤掉候选。

### 5.3 自定义 marginalia

Marginalia 是可扩展的——你可以为自定义命令加 marginalia:

```elisp
(add-to-list 'marginalia-command-categories
             '(my-command . my-category))
```

这让 marginalia 知道"my-command 的候选应该当作 my-category 处理"——然后它会显示对应的注释格式。这是 Emacs 可扩展性的体现——一个工具 (marginalia) 可以适配任何新命令。

---

## 6. Embark

### 6.1 角色

Embark 是"上下文菜单"——这是补全栈最被低估的部分。它让你**对候选做更多事**,不只是选:

Embark 是"context menu":
- 在 minibuffer 候选上 `C-.`,弹出菜单
- 在 buffer 符号上 `C-.`,弹出菜单

这种"context menu"模式来自 IDE——右键弹菜单。但 Embark 把它变成键盘驱动——按 `C-.` 弹菜单,菜单是 keymap 形式 (按对应键执行 action)。这比鼠标右键快得多。

### 6.2 菜单选项

Embark 的菜单是**上下文敏感**——根据 candidate 类型不同:

依候选类型不同:

**buffer 候选**:
- switch / kill / rename / display

**file 候选**:
- open / copy / delete / magit

**command 候选**:
- execute / describe / bind to key

这种"按类型分流"是 Embark 的核心——它知道你在操作什么类型的对象,提供对应的 action。比如你在 `consult-buffer` 的候选上,选中一个 buffer,按 `C-.`——菜单显示 "switch / kill / rename"。你在 `find-file` 候选上按 `C-.`——菜单显示 "open / copy / magit-status"。

### 6.3 配置

```elisp
(use-package embark
  :ensure t
  :bind (("C-." . embark-act)
         ("M-." . embark-dwim)
         ("C-h B" . embark-bindings))
  :custom
  (embark-prompter 'embark-keymap-prompter)
  (embark-cycle-key ".")
  :config
  (add-to-list 'display-buffer-alist
               '("\\`\\*Embark Collect \\(Completions\\|Actions\\)\\*"
                 (display-buffer-in-direction)
                 (direction . right))))
```

每个设置:
- `C-.`: 主键——弹菜单。这是社区约定的 Embark 键。
- `M-.`: "dwim" (do what I mean)——执行最可能的 action (file=open, buffer=switch)。这是快速路径。
- `C-h B`: 看 keybindings——列出当前所有键位。Embark 把这做成了一个 powerful 的工具。
- `embark-prompter 'embark-keymap-prompter`: 用 keymap 风格的 prompter——显示按键提示,你按对应键。比"选数字"的 prompter 更高效。

### 6.4 embark-consult

```elisp
(use-package embark-consult
  :ensure t
  :after (embark consult)
  :hook
  (embark-collect-mode . consult-preview-at-point-mode))
```

让 Embark 在 consult 命令的候选上工作。

`embark-consult` 是 Embark 和 Consult 的"胶水"——它让 Embark 的 action 在 Consult 候选上工作。比如你在 `consult-ripgrep` 结果上按 `C-.`,可以选"在 dired 打开"、"在 magit 看"、"复制路径"——这些都是 embark-consult 提供的。

---

## 7. Corfu (代码补全)

### 7.1 角色

Corfu 是**代码补全 UI**——它在 buffer 内弹出候选框:

Corfu 是**代码补全 UI**:
- popup 框
- TAB/RET 选择
- 与 `completion-at-point` 集成

Corfu 和 Vertico 是兄弟——同一作者 (Daniel Mendler),同样设计哲学 (极简、与内置集成)。Vertico 是 minibuffer UI,Corfu 是 in-buffer UI。两个加起来覆盖了所有补全场景。

Corfu 的核心设计是"用 completion-at-point"——这是 Emacs 24+ 的标准补全 API。Corfu 只负责 UI (popup),不负责 source。这让 Corfu 和任何 completion-at-point 的 source 兼容——LSP (eglot)、cape、内置 dabbrev。

### 7.2 配置

```elisp
(use-package corfu
  :ensure t
  :init
  (global-corfu-mode 1)
  :custom
  (corfu-auto t)
  (corfu-auto-delay 0.2)
  (corfu-auto-prefix 2)
  (corfu-cycle t)
  (corfu-quit-no-match 'separator)
  (corfu-preview-current 'insert)
  (corfu-preselect 'prompt)
  :bind (:map corfu-map
              ("TAB" . corfu-next)
              ([tab] . corfu-next)
              ("S-TAB" . corfu-previous)
              ([backtab] . corfu-previous)
              ("RET" . nil)                ; 不让 RET 选
              ("M-d" . corfu-show-documentation)
              ("M-." . corfu-show-location)))
```

每个设置:
- `corfu-auto t`: 自动弹——输入字符时自动显示候选。
- `corfu-auto-delay 0.2`: 延迟 0.2 秒——避免每按一键就弹。
- `corfu-auto-prefix 2`: 至少 2 个字符——单字符补全候选太多。
- `corfu-preview-current 'insert`: 实时预览——选中的候选"虚拟插入"到 buffer,你能看到效果。
- `corfu-preselect 'prompt`: 默认选 prompt (光标位置),而不是第一个候选——这避免"按 TAB 误选"。
- `RET nil`: **关键**——RET 不选,正常换行。这是社区教训——之前很多人设 RET 选,结果写代码时按 RET 想换行,变成选了候选,非常烦。
- `M-d`: 显示 documentation——和 IDE 的"按 F1 看 doc"类似。
- `M-.`: 显示 location——跳到定义 (符号定义的地方)。

### 7.3 Cape (sources)

Cape 提供各种补全 source——它把"非 LSP"的补全也接到 completion-at-point:

```elisp
(use-package cape
  :ensure t
  :init
  (add-to-list 'completion-at-point-functions #'cape-dabbrev)
  (add-to-list 'completion-at-point-functions #'cape-file)
  (add-to-list 'completion-at-point-functions #'cape-elisp-block)
  (add-to-list 'completion-at-point-functions #'cape-keyword)
  (add-to-list 'completion-at-point-functions #'cape-abbrev)
  :config
  (advice-add 'pcomplete-completions-at-point :around #'cape-wrap-silent)
  (advice-add 'completion-at-point :around #'cape-wrap-nonexclusive))
```

- `cape-dabbrev`: 当前 buffer 词
- `cape-file`: 文件路径
- `cape-elisp-block`: Elisp 代码块
- `cape-keyword`: 语言关键字
- `cape-abbrev`: abbrev 表

这些 source 都注册到 `completion-at-point-functions`——Corfu 按顺序尝试。这意味着即使**没有 LSP**,你也有补全 (从当前 buffer 词、从文件路径、从关键字)。这是 Emacs 的优势——VSCode 没有 LSP 就没补全,Emacs 有"低级补全"作为底线。

### 7.4 Kind-icons (icons)

kind-icon 让补全候选显示 icon——函数、变量、类等:

```elisp
(use-package kind-icon
  :ensure t
  :after corfu
  :custom
  (kind-icon-default-face 'corfu-default)
  :config
  (add-to-list 'corfu-margin-formatters #'kind-icon-margin-formatter))
```

显示 function / variable / class 等的 icon。

这是视觉增强——你在候选列表里立刻看到 ƒ (函数) vs v (变量) vs C (类)。这种视觉信息让"扫一眼找到目标"更快。来自 LSP 的 Kind 信息 (LSP 协议提供 candidate kind)。

---

## 8. Yasnippet (代码模板)

Yasnippet 是"代码模板"系统——你输入 `def` TAB,展开成函数定义模板,光标停在函数名位置:

```elisp
(use-package yasnippet
  :ensure t
  :diminish yas-minor-mode
  :hook (prog-mode . yas-minor-mode)
  :config
  (yas-reload-all))

(use-package yasnippet-snippets
  :ensure t
  :after yasnippet)
```

输入 `def` TAB,展开成函数定义模板。

Yasnippet 的核心是"展开"——你输入一个 trigger (如 `def`),按 TAB,变成预设的模板。模板里可以有 placeholder (用 TAB 跳到下一个位置)。这避免了"重复输入样板代码"。

和 Corfu 的关系: Yasnippet 应该和 Corfu 配合——`yas-expand` 绑到 TAB 之外的键 (比如 `M-/`),让 TAB 给 Corfu 用。否则两者抢 TAB 键会冲突。

---

## 9. 完整补全栈配置

这是一个完整的补全栈配置,可以直接复制到 init.el:

```elisp
;;; init-completion.el -*- lexical-binding: t; -*-

(use-package vertico
  :init (vertico-mode 1))

(use-package orderless
  :custom
  (completion-styles '(orderless basic)))

(use-package marginalia
  :init (marginalia-mode 1))

(use-package consult
  :bind (("C-s" . consult-line)
         ("C-x b" . consult-buffer)
         ("M-y" . consult-yank-pop)
         ("M-g i" . consult-imenu)
         ("M-s r" . consult-ripgrep)))

(use-package embark
  :bind (("C-." . embark-act)
         ("M-." . embark-dwim)))

(use-package embark-consult
  :after (embark consult))

(use-package corfu
  :init (global-corfu-mode 1)
  :custom (corfu-auto t))

(use-package cape
  :init
  (add-to-list 'completion-at-point-functions #'cape-dabbrev)
  (add-to-list 'completion-at-point-functions #'cape-file))

(use-package which-key
  :init (which-key-mode 1)
  :custom (which-key-idle-delay 0.3))

(provide 'init-completion)
```

这个配置是"最小可用"——它装了核心 6 个包 (vertico, orderless, marginalia, consult, embark, corfu, cape) 加 which-key (按键提示)。配置简单,但效果已经超过任何 IDE 的补全。

which-key 是一个"辅助包"——你按 `C-x` 后等 0.3 秒,which-key 弹出"接下来可以按什么"的菜单。这是学新键位的神器——你不需要背,按 `C-x` 看菜单。

---

## 10. 实战

### 任务 1: 用 consult-line 替代 isearch

```
C-s
;; 输入 pattern
;; 候选垂直显示
;; 用方向键选,RET 跳
```

比 isearch 更直观。

体验差异: isearch 是"输入即跳",你必须按 C-s 多次找下一个。consult-line 是"列表 + preview",你浏览候选,buffer 实时预览,选对了 RET。

### 任务 2: 用 consult-buffer 切 buffer

```
C-x b
;; 显示 buffer 列表 + recent files + bookmarks
;; 输入过滤
;; RET 打开
```

体验差异: 默认 switch-to-buffer 只显示当前 buffer 列表。consult-buffer 同时显示 buffer、recent files、bookmarks——一个命令搞定三种切换。

### 任务 3: 用 consult-ripgrep 跨项目搜

```
M-s r
;; 输入 pattern
;; 实时搜
;; preview 候选
```

这是项目内搜索的杀手锏——ripgrep 极快,Consult 的 preview 让你"看候选再跳"。比 `M-x grep` + 手动跳到结果快得多。

### 任务 4: 用 Embark 批量操作

```
C-x b                 consult-buffer
在候选上 C-.          Embark act
按 k                  kill
```

这是 Embark 的"批量"工作流——你在 consult-buffer 列表里,看到几个不用的 buffer,按 `C-.` 弹菜单,按 `k` kill。这比"切到 buffer,按 `C-x k`,回到列表"快得多。

### 任务 5: 用 Corfu 代码补全

```
打开 python 文件
输入 "def"
候选弹出
TAB 选
RET (在你配置里可能 nil)
```

写代码时,候选自动弹出。TAB 选——这是和 VSCode 类似的体验,但完全键盘驱动。

---

## 11. 自测

1. Vertico, Orderless, Consult 各自角色?
2. Marginalia 干啥?
3. Embark 的 `C-.` 干啥?
4. Corfu 和 Cape 关系?
5. completion-styles 配置什么?

**答案**:
> 1. Vertico UI,Orderless 算法,Consult 命令
> 2. 候选旁边显示元信息
> 3. 在候选/symbol 上拉菜单
> 4. Corfu UI,Cape source
> 5. 匹配算法的顺序

---

## 12. 下一步

进入 `lsp-eglot.md`。
