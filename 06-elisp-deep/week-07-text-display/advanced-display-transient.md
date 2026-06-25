# 高级 text/display property + button + transient.el

这不是一份 reference。Emacs Lisp manual 已经把每个函数都列了。我想做的事情不一样——我想带你从零开始,写一个能用的 git status buffer,然后在这个过程中把 text property、display property、button.el、transient.el 这四样东西全摸一遍。等我们写完,你会理解 Emacs 的显示系统为什么是这个样子,以及为什么 magit 长那样。

我会用一个贯穿全文的项目,叫 `my-gs` (my git status)。每一节给它加一个特性。等到最后一节,你手里有一个能跑、能用、能扩展的工具。它不是 magit——magit 有几十万行——但它是 magit 的骨架,而这个骨架就是 Emacs 显示系统的全部精华。

写作风格说明: 我会花大量篇幅讲故事和讲为什么,而不是列 API。原因很简单——API 你自己 `(info "(elisp) Text Properties")` 就能查到。但 text property 为什么是 text property,button.el 解决了什么问题,transient 为什么不让用 `completing-read`——这些是读了 manual 也学不到的。如果你只想要 reference,关掉这篇去看 elisp manual;如果你想知道"老程序员怎么想这些问题",继续读。

---

## Part 1: 设计一个 git status buffer

故事是这样开始的。某天你觉得 magit 太重了——你想,我只想要看一眼仓库里哪些文件改了,不用 stage,不用 commit,不用看 diff,就一眼。你 `M-x shell RET git status RET`,出来一堆带 ANSI 颜色的乱码,在 Emacs 的 shell buffer 里丑得不行。你想,我自己写一个吧。

最朴素的写法是这样: 开一个 buffer,把 `git status --porcelain` 的输出塞进去。`--porcelain` 让输出对脚本友好,每行一个文件,前面两个字符是状态码。

```elisp
(defun my-gs--raw-status ()
  "Return list of (status file) pairs."
  (with-temp-buffer
    (call-process "git" nil t nil "status" "--porcelain")
    (goto-char (point-min))
    (let (result)
      (while (not (eobp))
        (let ((line (buffer-substring (line-beginning-position)
                                      (line-end-position))))
          (when (>= (length line) 3)
            (push (cons (substring line 0 2)
                        (substring line 3))
                  result))
          (forward-line 1)))
      (nreverse result))))
```

我先讲讲这段在干什么,因为里面有几个 Emacs Lisp 的细节如果不讲你可能会卡住。

`call-process` 是同步调用外部程序的函数。第一个参数是程序名 `"git"`,第二个是 `nil` (不指定 input file),第三个是 `t` (output 直接插到当前 buffer),第四个是 `nil` (不显示在 echo area),后面是命令行参数。这是同步的——它会阻塞 Emacs 直到 git 返回。对 git status 这种毫秒级的命令没问题,但如果你将来调用 `git log` 一万条 commit 就要小心了。

`with-temp-buffer` 创建一个临时 buffer,在里面执行 body,然后杀掉。我们的 call-process 输出就写在这个临时 buffer 里,不污染用户当前 buffer。这是 Emacs Lisp 处理外部命令输出的标准模式。

`buffer-substring` 拿一段位置区间的字符串。`line-beginning-position` 和 `line-end-position` 给当前行的起止 position。`(point-min)` 是 buffer 开头。`eobp` 是 "end of buffer predicate"。`forward-line 1` 移动到下一行开头。

`push ... result` 加到表头。所以最后 `(nreverse result)` 翻回来——这是 Lisp 的常见模式,因为 `push` 到表头是 O(1),加到表尾是 O(n)。

好,现在我们有了一个数据源。接下来开个 buffer 显示它。

```elisp
(defun my-gs-buffer ()
  "Show git status in a buffer."
  (interactive)
  (let ((buf (get-buffer-create "*my-gs*")))
    (with-current-buffer buf
      (read-only-mode -1)
      (erase-buffer)
      (dolist (entry (my-gs--raw-status))
        (insert (format "%s %s\n" (car entry) (cdr entry))))
      (goto-char (point-min))
      (read-only-mode 1))
    (switch-to-buffer buf)))
```

`get-buffer-create` 拿一个名字对应的 buffer,不存在就建。`with-current-buffer` 切到那个 buffer 跑 body,跑完切回来。我们先关掉 read-only-mode (因为如果之前开过),erase-buffer 清空,然后把每条 entry 插进去,最后开 read-only 防止用户误编辑。

`M-x my-gs-buffer RET`,你看到一个 buffer,里面长这样:

```
 M src/main.c
 M README.md
?? new-file.txt
```

很丑。但能跑。这是 v0.0.1。

现在停下来想一个问题: 这个 buffer 和一个普通的 text buffer 有什么区别?

答案是——没有任何区别。

这听起来像个废话,但这是 Emacs 显示系统最核心的事实。Buffer 就是一串字符。你看到的"红色文字"、"可点击按钮"、"图标",不是 buffer 的属性,而是**字符**的属性。Emacs 的 redisplay 引擎在把字符画到屏幕上之前,会去查每个字符的 text property,根据 property 决定怎么画。

这意味着: 你想要红色,你就给字符加一个 `'face` 属性;你想要可点击,你就给字符加一个 `'keymap` 属性;你想要显示成图标,你就给字符加一个 `'display` 属性。属性不在 buffer 元数据里,属性在字符上。

为什么这样设计? 历史原因。Emacs 70 年代末期诞生的时候,内存极小,设计一个 buffer 类型同时支持文本和"对象"(按钮、图片)太重了。所以选择最简单的: buffer 只存字符,额外属性存在一个稀疏的 property interval table 里——一段连续字符如果 property 一样,只存一条记录。这样大部分没属性的文本零开销,只有你显式 propertize 的地方才占额外内存。

这种设计有一个深远的好处——Emacs 的所有"buffer 操作"函数 (`search-forward`、`re-search-forward`、`buffer-substring`、`goto-char`、kill/yank) 都直接对字符工作,不需要知道 text property。这意味着 text property 是"零侵入"的: 你写一个搜索函数,它自动支持任何 text property 的 buffer;你写一个高亮功能,它自动配合搜索工作。如果 Emacs 当年选择"对象树"模型 (像 DOM),每个操作都要遍历对象,代码量爆炸。

另一个推论是——buffer 序列化到磁盘 (save buffer) 只存字符,不存 text property。这是个 limitation,但也是 feature: text files 永远是 text files,不管你在 Emacs 里把它装点得多花哨。Org-mode 文件、markdown 文件,你在 vim 里打开还是文本。这种"显示和存储分离"是 Unix 哲学的体现。

但是——这里有个微妙的问题。如果你的 major mode 想"恢复"text property (比如 org-mode 打开 .org 文件,重新加 face/font-lock text property),怎么做到? 答案是 font-lock-mode 和 after-change-functions。major mode 通常不在文件 read 时直接 propertize,而是注册一个 font-lock keywords 模式——Emacs 每次显示一行时,font-lock 引擎根据 keywords 给那行字符贴 face 属性。这是 lazy + cached 的。Magit、org、prog-mode 都用 font-lock 系统。我们 my-gs 不用 font-lock,因为我们直接 propertize 字符串后 insert——这是"静态"buffer (内容固定) 的做法。如果是"动态"buffer (用户可以编辑),就得用 font-lock。

理解这一点之后,后面所有东西都是细节。face 是 text property,keymap 是 text property,display 是 text property,button 是 text property 的语法糖,连 image 也是 text property。

我们要做的事就是: 给字符贴属性。贴属性。再贴属性。

让我先讲一个具体场景,让你感受"text property 是显示的全部"这件事。

场景一: 你想做一个 org-mode-like 的 outline。Level 1 heading 显示成大字、蓝色、加粗。怎么做? Level 2 heading 显示成中字、绿色。怎么做?

新手可能会想: "Org-mode 一定有某种 'heading' 对象,我创建一个 heading 对象,设置它的 level 属性,然后加到 buffer 里。"——错。Org-mode 的 heading 就是一行以 `*` 开头的文本。`*` 的数量决定 level。Font-lock 看到 `^\\*\\s-.*` 这种正则,给 match 部分贴上 `'org-level-1` face。就这么简单。没有 heading 对象。

场景二: 你想在 mode-line 显示当前 git branch。新手可能想: "mode-line 是 UI 组件,branch 是它的一个子组件,我要在 UI 树里加一个 branch component。"——错。mode-line 是一个字符串 format,format 里有一个 `(:eval ...)` 形式的占位符,Emacs 每次 redisplay 时 eval 那个表达式,得到一个字符串,字符串里贴上 face text property,显示出来。Branch 不是 component,branch 就是一个字符串。

场景三: 你想做一个 button。新手: "Button 是 widget,我要 instantiate 一个 widget。"——错。Button 是一段字符串,带 `'button`、`'action`、`'keymap` 等 text property。没有 widget instance,只有 propertized string。

这三个场景的共同点: Emacs 没有"组件"概念,只有"字符串 + 属性"。这种简化让你失去了一些 (没有 button.getEnabled()、button.setStyle() 之类的 OO 接口),但换来了系统的整体简洁——所有 buffer 操作都通用,所有 buffer 互通有无,所有 buffer 可以序列化到纯文本。

如果你能内化这一点,后面的所有 API 都只是"在什么属性下做什么事"的细节。如果你不能内化,后面每个 API 你都会觉得"为什么不抽象成对象?"——答案是: 抽象的代价不值。

---

## Part 2: face property 上色

现在我们让 buffer 好看一点。git status 输出的前两个字符是状态码——`M` 是 modified,`A` 是 added,`D` 是 deleted,`??` 是 untracked。我们想让不同的状态用不同的颜色。

先讲 face。Face 在 Emacs 里是"一组显示属性"——包含字体、前景色、背景色、下划线、粗体、斜体等。Face 可以是 named (一个 symbol,定义在 face 列表里) 也可以是 anonymous (一个 plist,比如 `(:foreground "red" :weight bold)`)。

最简单的用法是 text property:

```elisp
(insert (propertize "M" 'face 'font-lock-warning-face))
(insert " ")
(insert (propertize "file.txt" 'face 'default))
(insert "\n")
```

`propertize` 是一个函数,接收一个字符串、一组 property pair,返回一个新的字符串(原字符串加上指定属性)。第一个调用把字符串 `"M"` 加上 `'face` 属性值为 `'font-lock-warning-face`,返回带属性的字符串 `"M"`。然后 `insert` 把它塞进当前 buffer。

注意——你以为 `insert` 插入的是字符,实际上它插入的是**带属性的字符**。Buffer 里存的就是这堆带属性的字符,redisplay 引擎读这些属性决定怎么画。

`font-lock-warning-face` 是 Emacs 内置的 face,通常显示成橙色或红色。`'default` 是默认 face (就是普通文字的样子)。Emacs 内置了一堆 face,`M-x list-faces-display` 可以看。

为什么用内置 face 而不是 `(:foreground "red")`? 因为用户的主题可能已经定义了 `font-lock-warning-face` 的颜色,可能是红色,可能是橙色,可能是粉色——你用 named face,用户的主题自动生效。你硬编码 `"red"`,主题就被破坏了。这是个品味问题,但在 Emacs 社区,用 named face 是约定。

现在我们把这个用到 my-gs 里:

```elisp
(defun my-gs--file-line (status file)
  "Return a propertized line for one git status entry."
  (let ((face (pcase status
                ("M " 'font-lock-warning-face)
                ("M"  'font-lock-warning-face)
                ("A " 'font-lock-type-face)
                ("A"  'font-lock-type-face)
                ("D " 'error
                ("D"  'error))
                ("??" 'shadow))))
    (concat (propertize status 'face face)
            " "
            (propertize file 'face 'default)
            "\n")))
```

我先承认这段有个 bug——D 那行括号匹配错了。我们等下修。先讲 pcase。

`pcase` 是 pattern match,Emacs 24 引入。等价的 cond 写法是:

```elisp
(cond
 ((string= status "M ") 'font-lock-warning-face)
 ((string= status "M")  'font-lock-warning-face)
 ((string= status "A ") 'font-lock-type-face)
 ...)
```

pcase 比 cond 紧凑。但更重要的是,pcase 是 pattern matching——它可以 match 数据结构,不只是相等。这里我们只用了 string equality 的 pattern (叫 `_` pattern 不用了,直接字面字符串就行),所以效果上等于 cond。但养成 pcase 习惯,后面 match list / vector / plist 就舒服了。

修了括号 bug 的版本:

```elisp
(defun my-gs--file-line (status file)
  "Return a propertized line for one git status entry."
  (let ((face (pcase status
                ("M " 'font-lock-warning-face)
                ("M"  'font-lock-warning-face)
                ("A " 'font-lock-type-face)
                ("A"  'font-lock-type-face)
                ("D " 'error)
                ("D"  'error)
                ("??" 'shadow)
                (_    'default))))
    (concat (propertize status 'face face)
            " "
            (propertize file 'face 'default)
            "\n")))
```

注意我加了 `(_ 'default)` 作为 fallback——任何没 match 上的状态码用 default face 显示。`_` 是 wildcard pattern,永远 match。pcase 没 fallback 会返回 `nil`,然后 face 为 nil 等于 default,但显式写出来更清楚。

git status --porcelain 的状态码其实是两个字符。第一列是 staged 状态,第二列是 unstaged 状态。`"M "` 表示 staged-modified-但没-unstaged-change;`" M"` 表示 unstaged-modified-没-staged;`"MM"` 表示两者都有。为了简单,我们这版只看头一种情况。后面 Part 8 整合的时候再精细化。

更新 my-gs-buffer:

```elisp
(defun my-gs-buffer ()
  "Show git status in a buffer."
  (interactive)
  (let ((buf (get-buffer-create "*my-gs*")))
    (with-current-buffer buf
      (read-only-mode -1)
      (erase-buffer)
      (dolist (entry (my-gs--raw-status))
        (insert (my-gs--file-line (car entry) (cdr entry))))
      (goto-char (point-min))
      (read-only-mode 1))
    (switch-to-buffer buf)))
```

只改了一行: `(insert (format ...))` 换成 `(insert (my-gs--file-line ...))`。其他没动。这就是 propertize 的优雅之处——你给的数据还是字符串,只是带属性的字符串。

跑一下。现在 buffer 长这样 (假设 dark theme):

```
 M src/main.c        <- M 是橙色
 M README.md         <- M 是橙色
?? new-file.txt      <- ?? 是灰色 (shadow)
```

漂亮了一点。

但是 stop 一下。我刚才说 "M 是橙色"——你怎么验证? 你可以 `(get-text-property (point) 'face)` 在某个字符上看它的 face 属性。把光标移到 M 上,`M-: (get-text-property (point) 'face) RET`,echo area 显示 `font-lock-warning-face`。这就是属性,字面意义的属性,挂在那个字符上。

Text property 有几个核心操作:

- `put-text-property` ——给一段字符贴属性
- `add-text-properties` ——类似但用 plist 一次贴多个
- `get-text-property` ——读光标处的属性
- `text-properties-at` ——读光标处所有属性 (alist)
- `remove-text-properties` ——删属性
- `set-text-properties` ——完全替换所有属性
- `next-single-property-change` ——找下一个属性变化点

propertize 是个 helper,本质上是 `let ((s (copy-sequence string))) (put-text-property 0 (length s) prop val s) s`。它创建新字符串,不修改原字符串。所以可以放心用——不会改你的 literal。

那么 face 这个属性怎么知道有哪些 key? 看 `(info "(elisp) Face Attributes")`。但通常你不直接设 face 属性,而是引用 named face。named face 用 `defface` 定义。如果你想自定义一个 face (比如公司品牌色),你 defface 一个,然后用 symbol 引用:

```elisp
(defface my-gs-modified-face
  '((t :inherit font-lock-warning-face :weight bold))
  "Face for modified files in my-gs."
  :group 'my-gs)
```

`defface` 第一个参数是 face 名,第二个是 spec——一个列表,每个元素是 `(condition attributes...)`。`t` 表示无条件 (所有 display 都用)。`:inherit` 让它继承另一个 face,然后你可以 override 一些属性。

define face 之后,你就可以 `'my-gs-modified-face` 当 text property 用。

### Face 之外的显示属性: invisible / intangible

讲两个特殊属性,跟 face 一类但功能不在"颜色",而在"显示与否"。

**`'invisible` 属性**。设为非 nil 时,这段字符变成不可见——视觉上不显示,但 buffer 里还在。Org-mode 的折叠就是 `'invisible` 实现的。`(set-invisible-overlay ...)` 或 `(propertize text 'invisible t)`。

invisible 配合 `(enable-visual-newline)`、`(buffer-invisibility-spec)` 等可以做复杂折叠。具体看 `(info "(elisp) Invisible Text")`。

**`'intangible` 属性**。设为非 nil 时,光标无法进入这段字符。光标移动时自动跳过。这用于"显示标记但不能编辑/停留"的区间,比如 image display property 上 (image 占字符位置但你不希望光标进去)。

这两个属性很微妙,容易引入 bug。一般不要乱用——你想要折叠用 `(setq buffer-invisibility-spec ...)`,你想让光标跳过用 `(field ...)` text property (更现代)。

### Line-prefix / wrap-prefix

两个相关属性。`'line-prefix` 是 buffer 里每行的开头 prefix——redisplay 自动在每行渲染前加这个 prefix。`'wrap-prefix` 是软 wrap 时的续行 prefix。

Org-mode 的 quote block `>` 缩进就用 wrap-prefix。Outline 的层级缩进也用。

简单例子——你想要 my-gs buffer 每行前面有竖线分隔:

```elisp
(propertize "first line\nsecond line\n"
            'line-prefix (propertize "|" 'face 'shadow))
```

整个字符串 `'line-prefix` 设为 `"|"`——每行显示前自动加 `|`。Buffer 数据没变。

这是 display 的"per-line"版本——比手动每行 insert 字符简洁。

### Field 属性

`'field` 是个"光标语义"属性。设了 `'field` 的字符区间被识别为一个 field——Emacs 的 forward-word、beginning-of-line 等命令会停在 field 边界。这用于 minibuffer 的 prompt 部分 (光标不会跑到 prompt 里)。

button 用 `'field` 让 button 区间有清晰边界。你也可以用——比如让用户在 file button 上按 `M-f` (forward-word) 时停在 button 边界。

这些是 face 之外的"显示相关"text property。一般不常用,但知道有这些工具,需要时能找到。

---

## Part 3: keymap property (button 基础)

现在 buffer 有了颜色。但是它不能交互——我把光标移到 `src/main.c` 上,按 RET,什么都没发生。magit 的体验是: 光标在文件上,RET 就 diff,`e` 就 ediff,s 就 stage。我们至少要实现 RET 打开文件。

Emacs 的 keymap 大家都用过——`C-x C-s` 之所以能保存,是因为 global-map 里 `C-x C-s` 绑到 `save-buffer`。但是 keymap 不只是全局的。每个 major mode 有自己的 keymap (叫 local-map)。更细粒度地,**buffer 里每个字符都可以有自己的 keymap**——通过 `'keymap` text property。

这意味着: 你可以让 buffer 里某段字符的按键行为完全不同。`src/main.c` 这段字符上,RET 是"打开文件";别处 RET 是普通的 newline。这就是 text property keymap 的威力。

我们给文件名 propertize 一个 keymap:

```elisp
(defun my-gs--file-line (status file)
  "Return a propertized line for one git status entry."
  (let* ((face (pcase status
                 ("M " 'font-lock-warning-face)
                 ("M"  'font-lock-warning-face)
                 ("A " 'font-lock-type-face)
                 ("A"  'font-lock-type-face)
                 ("D " 'error)
                 ("D"  'error)
                 ("??" 'shadow)
                 (_    'default)))
         (file-map (make-sparse-keymap)))
    (define-key file-map (kbd "RET") #'my-gs-open-file)
    (define-key file-map [mouse-1] #'my-gs-open-file)
    (concat (propertize status 'face face)
            " "
            (propertize file
                        'face 'default
                        'keymap file-map
                        'help-echo "mouse-1, RET: open file")
            "\n")))
```

逐行讲。

`let*` 和 `let` 的区别: `let*` 让后面的 binding 能用前面的 binding。这里我用 `let*` 是因为 `file-map` 在同一个 let 里就能用 (虽然这里没依赖,但习惯上建 keymap 用 let*)。

`make-sparse-keymap` 创建一个空的 keymap。"sparse" 意味着内部是 alist,适合少量 binding。`make-keymap` 创建 char-table-based keymap,适合密集 binding (像 global-map)。99% 情况用 sparse。

`define-key` 给 keymap 加一个 binding。`file-map` 是 keymap。`(kbd "RET")` 是把字符串 `RET` 转成 event symbol——`kbd` 是个 helper,让 `"RET"` 这种字符串表达 `"\r"` 或 `[return]`。`[mouse-1]` 是直接的 event vector,表示鼠标左键。第三个参数是要绑的命令,这里 `#'my-gs-open-file` ——`#'` 是 `(function ...)` 的简写,等价于 `(lambda ...)` 或 symbol 的 function 引用。

然后 propertize。注意我同时设了三个属性: `'face`、`'keymap`、`'help-echo`。propertize 接受任意多个 pair。

`'help-echo` 是鼠标 hover 时显示的提示。Emacs 内置识别这个属性——你 hover 鼠标在带这个属性的字符上,echo area 显示提示文本。完全免费,你不用写任何代码。

然后定义 `my-gs-open-file`:

```elisp
(defun my-gs-open-file ()
  "Open file at point."
  (interactive)
  (let ((file (get-text-property (point) 'my-gs-file)))
    (when file
      (find-file file))))
```

但是等等——我们还没在 propertize 里设 `'my-gs-file` 属性。`get-text-property` 会返回 nil,`when` 不执行。

这暴露了一个关键问题: **光标在 file.txt 上,my-gs-open-file 怎么知道是哪个 file?**

最直观但错误的回答: 从 buffer 文本 parse。`(buffer-substring (line-beginning-position) (line-end-position))` 拿到当前行,然后 split,取第二个 token。但这有几个问题: 文件名里有空格怎么办? 文件名里有 unicode 怎么办? 你已经在数据里了,为什么还要 parse 一遍?

正确做法: 用 text property 存数据。这听起来怪——text property 不是"显示属性"吗?

不是。text property 是**任意数据**。是的,face、keymap、display 这些是 Emacs 内置识别的 property,但你可以贴任何 symbol-key 的属性:

```elisp
(propertize file 'keymap file-map 'help-echo "..." 'my-gs-file file)
```

`'my-gs-file` 是我们自定义的 property,值是 file 路径。Emacs 不会用它做任何显示,它就是数据。等用户按 RET,我们的命令读这个属性,拿到 file 路径,打开它。

这是 Emacs 显示系统一个深刻的设计——text property 是 buffer 上的**通用 metadata layer**。你可以把它当作数据库的一个 row,字符位置是 row id,任何属性是 column。face 是其中一个 column,keymap 是另一个 column,你的 `'my-gs-file` 是又一个 column。

完整 file-line:

```elisp
(defun my-gs--file-line (status file)
  "Return a propertized line for one git status entry."
  (let* ((face (pcase status
                 ("M " 'font-lock-warning-face)
                 ("M"  'font-lock-warning-face)
                 ("A " 'font-lock-type-face)
                 ("A"  'font-lock-type-face)
                 ("D " 'error)
                 ("D"  'error)
                 ("??" 'shadow)
                 (_    'default)))
         (file-map (make-sparse-keymap)))
    (define-key file-map (kbd "RET") #'my-gs-open-file)
    (define-key file-map [mouse-1] #'my-gs-open-file)
    (concat (propertize status 'face face)
            " "
            (propertize file
                        'face 'default
                        'keymap file-map
                        'help-echo "mouse-1, RET: open file"
                        'my-gs-file file)
            "\n")))
```

现在按 RET 在文件名上——`find-file` 打开它。漂亮。

但要再停下来想想——这种"text property 存数据"有几个坑,你需要知道。

第一个坑: text property 是 sticky 的。当你 `insert` 一个字符到一个 propertized 区间,新插入的字符默认**不**继承周围属性。但如果你用 `insert-and-inherit`,它会继承。这是 buffer 操作的细节。在 my-gs 里我们不操心,因为我们一次性 insert 一行,不编辑。但如果用户修改了 buffer (虽然我们设了 read-only),属性会变。

第二个坑: yank (粘贴) 时 text property 会跟着走。如果用户复制一段带 `'my-gs-file` 属性的字符到别的 buffer,属性还在那里——别的 buffer 不知道这个属性是啥,但属性在。一般无害,但如果你给 property 起的名字和别处冲突,可能怪事。

第三个坑: text property 不序列化。如果你 `(write-file "foo.txt")`,property 不进文件 (除非你显式 propertize 后用某些特殊编码)。这是好事——文件只存字符,property 在 Emacs 内部。

理解了这些之后,你应该能用 text property 干很多事。比如你想给文件加一个"上次修改时间",propertize 时贴 `'my-gs-mtime` 属性;hover 显示。你想标记某些文件是 "已经 review 过",贴 `'my-gs-reviewed t`。

这是 Lisp 的味道——简单的数据模型 (字符 + 属性),无限的组合可能。

### Keymap 的 lookup 顺序

讲个细节——当用户按一个键,Emacs 怎么决定执行什么命令?

Emacs 的 keymap lookup 是有优先级的:

1. **`overriding-local-map`** ——最高优先级。一般 nil,但某些 minor mode 临时设它 (比如 isearch 在搜索时设它)。
2. **`overriding-terminal-local-map`** ——per-terminal override。
3. **text property keymap at point**——我们刚学的。如果光标在带 `'keymap` text property 的字符上,这个 keymap 优先。
4. **`minor-mode-overriding-map`** ——minor mode 的 override。
5. **minor mode keymaps**——所有启用的 minor mode 的 keymap (按 `minor-mode-map-alist` 顺序)。
6. **`major-mode`'s local map**——current buffer 的 major mode keymap (`foo-mode-map`)。
7. **`global-map`**——最后 fallback。

所以 text property keymap 排在 minor mode 和 major mode 之上,只比临时 override 低。这意味着——你给文件名贴了 `'keymap` text property 绑 RET,即使在 my-gs-mode 里你也绑了 RET,在 file 按钮上按 RET 用 text property 的 (打开文件),不在 file 按钮上用 mode 的 (什么也不做)。这正是我们想要的。

但是——如果你在 my-gs-mode-map 里 RET 绑到别的命令,而你想让 button 优先,text property keymap 自然优先。但如果你想让 RET 在 button 上还是 mode 的行为 (不太常见),你得用 `:filter` 或 override text property keymap。一般用不上。

### Mouse event 处理

`[mouse-1]` 我们绑了。但鼠标事件有几个:

- `[mouse-1]` ——左键单击
- `[mouse-2]` ——中键单击 (paste)
- `[mouse-3]` ——右键单击 (menu)
- `[down-mouse-1]` ——左键按下 (dragging 用)
- `[drag-mouse-1]` ——左键拖动结束
- `[double-mouse-1]` ——双击

button.el 默认绑 mouse-1 和 mouse-2。绑 mouse-2 一般是"在另一个 window 打开" (像 web 浏览器中键)。我们 my-gs 不绑 mouse-2,但你可以加:

```elisp
(define-key file-map [mouse-2]
            (lambda () (interactive)
              (let ((file (get-text-property (point) 'my-gs-file)))
                (when file (find-file-other-window file)))))
```

Mouse event 的 `(point)` 是事件发生时光标位置——不是事件之后。`(posn-point (event-start last-input-event))` 更可靠。

### `kbd` 和 event 的微妙

讲讲 `(kbd "RET")` 和 `[return]` 的区别——新手会困惑。

- `(kbd "RET")` 返回 symbol `RET`,等价于 `13` (control-m) 的 ASCII。这是 terminal-style 的"按 enter"。
- `[return]` 是 GUI-style 的"按 enter 键"——某些 GUI Emacs 上,它和 RET 是不同 event。

历史: 终端 Emacs 只有 ASCII,RET 是 Ctrl-M (13)。GUI Emacs 引入了"function key"概念——`<return>` 是真正的 enter 键,`<backspace>`、`<tab>`、`<delete>` 等也是。

绑 `(kbd "RET")` 在终端和 GUI 都工作——Emacs 内部 GUI 的 `<return>` 默认会 map 到 RET。但有些 user config 改这个 mapping。最安全做法: 绑 RET (`(kbd "RET")`)——大多数情况没问题。

Mouse 事件 `[mouse-1]` 是 GUI-only (终端不发 mouse 事件除非 `xterm-mouse-mode`)。绑了 mouse 不会有副作用——GUI 工作终端不工作,不影响 keyboard binding。

### 三种"text property 存数据"的真实场景

我讲三个场景,理解"text property 是通用 metadata"。

**场景 1: hover 显示 diff stat**。光标在 file button 上时,你想在 echo area 显示这个文件的 diff stat (`+15 -3 lines`)。

你可以 my-gs-mode 用 `cursor-sensor-mode` (一个 minor mode),它会检测光标进入/离开某段文字,触发函数。函数读 `'my-gs-file` 属性,call git diff --stat,显示。

```elisp
(cursor-sensor-mode 1)

;; 在 file button 上加 'cursor-sensor-functions 属性
(propertize file
            'cursor-sensor-functions
            (list (lambda (window prev-pos symbol)
                    (when (eq symbol 'entered)
                      (let ((f (get-text-property (point) 'my-gs-file)))
                        (when f (message "diff: %s"
                                         (my-gs--diff-stat f))))))))
```

每次光标进入 file 区间,函数触发,diff stat 显示在 echo area。这是"text property 当数据库"的最佳例子。

**场景 2: mark files for batch operation**。用户在 file button 上按 `m`,标记这个文件。然后 transient menu 提供 "act on marked"。

实现: `'my-gs-marked` 属性。

```elisp
(defun my-gs-toggle-mark ()
  (interactive)
  (let* ((btn (button-at (point)))
         (marked (and btn (button-get btn 'my-gs-marked))))
    (when btn
      (button-put btn 'my-gs-marked (not marked))
      (my-gs--refresh-line))))  ;; 重画当前行显示 * 标记
```

然后批操作遍历 buffer:

```elisp
(defun my-gs-marked-files ()
  (let (files)
    (save-excursion
      (goto-char (point-min))
      (while (setq btn (next-button (point)))
        (goto-char (button-start btn))
        (when (button-get btn 'my-gs-marked)
          (push (button-get btn 'my-file) files))
        (goto-char (button-end btn))))
    files))
```

`next-button` 是 button.el 内置——遍历 buffer 找下一个 button。这是 button.el 的好处之一——它给你 `next-button`、`previous-button`、`button-map` 等通用遍历 API。

**场景 3: 缓存计算结果**。my-gs--diff-stat 调 git 比较慢 (几十毫秒)。你不想每次 hover 都调。可以把结果 cache 在 text property:

```elisp
(defun my-gs--diff-stat (file)
  (let ((btn (button-at (point))))
    (or (button-get btn 'my-gs-cached-stat)
        (let ((stat (my-gs--compute-diff-stat file)))
          (button-put btn 'my-gs-cached-stat stat)
          stat))))
```

第一次算,缓存到 button property。之后读 cached。这又是"text property 当数据库"。Magit 大量用这种缓存模式——diff 结果、process 输出、commit metadata 都缓存在 text property。

这三个场景展示——text property 不只是"显示属性",是 Lisp 程序员手中的通用工具。

---

## Part 4: button 包

我刚才讲的"keymap + 数据属性 + help-echo"模式,Emacs 内置一个包封装了——button.el。1995 年加入 Emacs。它的动机就是: 每次都 make-sparse-keymap、define-key RET、define-key mouse-1、设 help-echo、加一个数据属性——太重复了。封一层。

button.el 是 Emacs 早期开发 (1990s) 的产物。原作者已经不可考,但代码风格是典型 "Lisp machine 思想"——把一个抽象封装到极致,接口简洁。button.el 不到 1000 行,提供完整的 button 抽象。

它的核心思路是: button 不是新东西,button 就是一段 text property + 一些约定。约定包括: 一个 `'button` 属性标记"这是按钮",一个 `'category` 属性指向 button 类型(类似 face 的 named face),一个 `'action` 属性是点击时调用的函数,一组标准 keymap 让 RET/TAB/mouse 都工作。

为什么是 `'button` 属性 + `'category` 属性? 因为 Emacs 内置一个机制叫 "property category"——`'category` 属性的值是一个 symbol,这个 symbol 的 `category-default` property (用 `get` 查的) 提供其他属性的默认值。这是 Emacs 的"prototype-based inheritance"。

举个例子:

```elisp
(put 'my-category 'face 'bold)
(put 'my-category 'keymap some-map)

(propertize "text" 'category 'my-category)
```

这段 propertize 看起来只设了 `'category`,但实际效果是 face = bold,keymap = some-map——因为查 `'face` 时,Emacs 先查 text property `'face`,没找到,再查 `'category` symbol 的 `'face` property。

这让 button 类型系统廉价——button type 就是一个 symbol,定义 type 时给 symbol put 各种 property,button 引用这个 symbol 当 category,自动继承所有 property。define-button-type 就是封装这个。

最简单的用法:

```elisp
(require 'button)

(insert-button "file.txt"
               'action (lambda (btn) (find-file (button-get btn 'my-file)))
               'my-file "file.txt")
```

`insert-button` 是一个函数,第一个参数是要显示的文字,后面是 plist,贴到 button 上。`'action` 是 button 内置识别的属性——点击时调用的函数,函数接收 button 本身 (不是事件)。

`button-get` 是从 button 读属性——给定 button 起始位置和属性名,返回值。我们之前用 `get-text-property` 也行,但 button.el 提供 button-get 更方便 (它会先找到 button 的起止点,再读)。

我们的 my-gs 用 button 重写:

```elisp
(defun my-gs--file-line (status file)
  "Return a propertized line for one git status entry, using button."
  (let* ((face (pcase status
                 ("M " 'font-lock-warning-face)
                 ("M"  'font-lock-warning-face)
                 ("A " 'font-lock-type-face)
                 ("A"  'font-lock-type-face)
                 ("D " 'error)
                 ("D"  'error)
                 ("??" 'shadow)
                 (_    'default)))
         (status-str (propertize status 'face face))
         (file-str   (propertize "" 'face 'default)))
    (with-temp-buffer
      (insert status-str)
      (insert " ")
      (insert-button file
                     'action (lambda (btn)
                               (find-file (button-get btn 'my-file)))
                     'my-file file
                     'face 'default
                     'help-echo "mouse-1, RET: open file")
      (insert "\n")
      (buffer-string))))
```

注意我用 with-temp-buffer 构造字符串——这是因为 insert-button 直接插到当前 buffer。我想要的是一个字符串。`buffer-string` 拿到 buffer 全部内容 (带属性)。

这样 my-gs-buffer 改成:

```elisp
(dolist (entry (my-gs--raw-status))
  (insert (my-gs--file-line (car entry) (cdr entry))))
```

还是一行,只是内部用了 button。

button.el 提供的几个核心 API:

`insert-button` ——在 point 处插入一个 button,返回 button。`make-button` ——把 buffer 里一段已存在的区间标记成 button (不修改文本)。`button-at` ——给定 position,返回那个 position 上的 button 对象 (或 nil)。`button-get` / `button-put` ——读/写 button 属性。`button-activate` ——以编程方式"点击"一个 button。`forward-button` / `backward-button` ——导航到下一个/上一个 button。`button-label` ——读 button 的文字。

button 的几个好处:

第一,标准化。RET、mouse-1、TAB (button 之间跳) 都自动工作。你自己写 keymap 要手绑三个。

第二,accessibility。Emacs 的 screen reader (emcspeak) 识别 button,能告诉盲人用户"这里有一个按钮"。你自己写 keymap 就没这待遇。

第三,跨 platform。Button 在终端 Emacs 和 GUI Emacs 都工作——mouse 事件正确处理。

为什么不用 button? 当你想要 button 之外的细粒度控制时。比如 magit 的 hunk line——它不只是"点击",它根据光标位置变化行为 (RET 在 hunk 上是 expand,在 file 上是 diff)。Magit 用自己写的 keymap + text property,不用 button。但你刚入门,先用 button。

Button 还有几个进阶特性。

**Button types**。`define-button-type` 让你定义一类 button 的"模板"——所有这类 button 共享一些默认属性。比如:

```elisp
(define-button-type 'my-gs-file-button
  'action (lambda (btn) (find-file (button-get btn 'my-file)))
  'help-echo "mouse-1, RET: open file"
  'face 'default)
```

然后:

```elisp
(insert-button file
               :type 'my-gs-file-button
               'my-file file)
```

`:type` 指定 button type,button 从 type 继承 action、face、help-echo。你只需要给 button 自己特有的属性 (`my-file`)。这是 DRY。

define-button-type 还支持继承:

```elisp
(define-button-type 'my-gs-diff-button
  :supertype 'my-gs-file-button
  'action (lambda (btn) (my-gs-diff (button-get btn 'my-file))))
```

`:supertype` 让 diff-button 继承 file-button 的所有属性,只 override action。

**forward-button / backward-button**。用户按 TAB 在 button 之间跳,需要这两个。但 button.el 默认 keymap 不绑 TAB——你需要自己绑,或者用 `button-buffer-map`。这是 button.el 一个被吐槽的地方,稍微"不完整"。

**mode-line button**。Mode line (底部状态条) 也能放 button——text property 在 mode-line 上一样工作。但 mode-line 的字符串结构特殊,要用 mode-line-%-constructs。这是另一个坑,先不展开。

---

## Part 5: display property 高级

到这里我们一直在改"颜色"和"按钮"。但 Emacs 有个更狠的属性: `'display`。

`'display` 是 text property 的瑞士军刀。它可以让一个字符**显示成完全不同的东西**。注意——是"显示成",不是"变成"。Buffer 里的数据还是原字符,但屏幕上画的不是它。

这是 Emacs 显示系统一个反直觉的设计——**屏幕上看到的和 buffer 里的字符可以不一致**。

最简单例子:

```elisp
(propertize "x" 'display "MODIFIED")
```

把字符 `x` 显示成 8 个字符 `MODIFIED`。Buffer 里还是 1 个字符 `x`。光标移动按 buffer 字符走——按一下右移一个 `x`,不是 8 个 `MODIFIED`。`(buffer-substring (point-min) (point-max))` 返回 `"x"`,不是 `"MODIFIED"`。

为什么这样设计? 因为有时候你想美化显示,但不想污染数据。Org-mode 里的 `*bold*` 显示成 **bold**,但 buffer 里还是 `*bold*`——你可以 export 到其他格式,可以 search,可以编辑。Magit 的 commit hash 显示成缩写,但点开能展开。这些都是 display property 的功劳。

display property 的值可以是:

- 一个字符串 ——替换显示
- 一个 list ——更复杂的指令,比如 `(height 2.0)`、`(:family "Mono")`、`(image ...)`
- 一个 image object ——直接显示图片

我们来逐个看。

### 5.1 改变显示文本

刚才的 `"MODIFIED"` 例子。应用到 my-gs——把 `"M "` 显示成 `"✎ "` (一个 emoji 表示修改):

```elisp
(propertize "M " 'display "✎ ")
```

等等——`✎` 是 unicode 字符,Emacs 27+ 在 GUI 能显示。终端可能不行。这就是为什么用 display property 的另一个好处: 你可以让原数据保持 ASCII (`M `),显示美化 (✎),export/搜索时用原 ASCII 数据。万一你的终端不支持 emoji,改 display property 一处,buffer 不动。

更激进一点——用更短的显示:

```elisp
(propertize "modified" 'display "M")
```

显示成 `M`,buffer 里是 `modified`。光标移动按 8 字符走,你看到 1 字符。这种"显示短缩,数据完整"在 dashboard 类 buffer 很常见。

### 5.2 改变大小

```elisp
(propertize "Git Status" 'display '(height 2.0))
```

`'(height 2.0)` 是 display property 的 list form——`(keyword value...)`。`:height` 控制字号倍数 (相对于当前 face)。2.0 是两倍高。

我们想在 my-gs buffer 顶部显示一个大字标题。后面 Part 8 会做。

类似的还有 `:width`、`:rise` (上下偏移,做上标下标)。

### 5.3 改变字体

```elisp
(propertize "code" 'display '(:family "JetBrains Mono"))
```

`:family` 改字体。这是 display property 而不是 face 的属性,因为——等等,face 也有 :family。区别: face 是 sticky 的 (一段字符共享 face),display 是单个字符的指令。实际上想改字体一般用 face,不用 display property。

display property 的 :family 用途: 你想让一段文字临时变字体,不想定义新 face。

### 5.4 嵌入图片

```elisp
(let ((img (create-image "~/icon.png" 'png nil :height 30)))
  (insert (propertize " " 'display img)))
```

这是 Emacs 显示图片的核心方法。`create-image` 加载一个图片文件,返回一个 image descriptor (内部是 vector)。然后我们 propertize 一个空格 `'display` 成 image。

为什么用空格? 因为 display property 还是占字符位置。空格 1 个字符——buffer 里有它,显示成 30 像素高的图片。光标可以停在这个空格上,搜索能 match 这个空格。

`:height 30` 是 create-image 的 keyword argument,把图片缩放到 30 像素高。

支持的图片格式取决于 Emacs 编译选项。Linux 上一般支持 png、jpeg、svg、gif。`(image-type-available-p 'png)` 检查。

终端 Emacs 不支持图片。所以如果你的代码用了 image,要 featurep 检查:

```elisp
(when (display-images-p)
  (let ((img (create-image ...)))
    ...))
```

### 5.5 space 属性

```elisp
(propertize " " 'display '(space :width 50))
```

`(space :width N)` 让一个空格显示成 N 个字符宽。Buffer 里还是 1 个空格。

这在做 column alignment 时有用。比如你想让 buffer 第 50 列开始显示文件名:

```elisp
(insert (propertize " " 'display '(space :width 50)))
```

不管前面是什么,从这开始到第 50 列。

还有 `:align-to` ——对齐到第 N 列。`(space :align-to 50)` 不管前面是 5 字符还是 49 字符,这空格后面正好是第 50 列。这比手算 width 强多了。

Magit 用 `:align-to` 对齐它的 hunk header。Org-mode 表格用类似机制对齐。

### 5.6 实战: 状态图标用 emoji

我们让 my-gs 显示更"现代"一点——每个状态用 emoji。

```elisp
(defun my-gs--status-icon (status)
  "Return a display property string for STATUS."
  (let ((icon (pcase status
                ("M " "📝")
                ("M"  "📝")
                ("A " "✨")
                ("A"  "✨")
                ("D " "🗑")
                ("D"  "🗑")
                ("??" "❓")
                (_    "  "))))
    (propertize "XX" 'display icon)))
```

注意这里 `propertize "XX"` 用了 `"XX"` 而不是 `" "`——因为 emoji 显示占两个字符宽。我们 buffer 里也存 2 字符,display 成 emoji,光标行为正常。

但是 emoji 在 Emacs 终端是噩梦。终端不支持,需要 GUI + 合适字体 (Symbola、Noto Emoji 等)。你的用户可能没装。所以这种"美化"要慎重——考虑 fallback。

更稳的做法: 用 text character 替代 emoji。

```elisp
(pcase status
  ("M " "*")  ;; ASCII *
  ("A " "+")
  ("D " "-")
  ("??" "?"))
```

朴素但稳。或者用 Unicode 几何字符 (Emacs 26+ 在终端一般都能显示):

```elisp
(pcase status
  ("M " "●")  ;; filled circle
  ("A " "◉")  ;; fisheye
  ("D " "○")  ;; empty circle
  ("??" "◌")) ;; dotted circle
```

或者用 Emacs 的 `all-the-icons` 包 (需要安装),它处理 fallback、字体检测——但增加依赖。Magit 自己定义了一套图标系统。

我们这里选用 ASCII + emoji 二选一,通过变量切换:

```elisp
(defcustom my-gs-use-emoji nil
  "If non-nil, use emoji icons.  Requires GUI with emoji font."
  :type 'boolean
  :group 'my-gs)

(defun my-gs--status-icon (status)
  "Return a display property string for STATUS."
  (if my-gs-use-emoji
      (let ((icon (pcase status
                    ("M " "📝") ("M" "📝")
                    ("A " "✨") ("A" "✨")
                    ("D " "🗑") ("D" "🗑")
                    ("??" "❓")
                    (_    "  "))))
        (propertize "XX" 'display icon))
    (pcase status
      ("M " "*") ("M" "*")
      ("A " "+") ("A" "+")
      ("D " "-") ("D" "-")
      ("??" "?")
      (_   " "))))
```

`defcustom` 定义一个用户可定制的变量——出现在 `M-x customize-group RET my-gs RET` 界面里。`:type 'boolean` 告诉 customize 显示成 checkbox。`:group 'my-gs` 是 customize 组 (要先用 `defgroup` 定义,这里略过)。

### Display property 几个进阶概念

**`when` and `must`**。display 的值可以是 `(when condition ...)` ——根据 condition 选择是否应用。这用于自适应显示 (比如不同 window 大小不同显示)。

**Spec list**。`(slice x y w h)` 切图片的一部分显示。`(relative-w 0.5)` 按比例宽度。`(space ...)` 我们讲过。

**Mixing display with image**。Magit 用的: 在 image 上叠文字。需要 `(image ... :mask heuristic)` 等。复杂。

最后强调——display property 是**显示**。`(buffer-substring ...)` 返回原字符。`(save-excursion ... search-forward)` 搜原字符。所以你的数据保持纯净,显示可以花哨。这个分离是 Emacs 设计的高明之处。

### 三种"显示 ≠ 数据"的真实场景

我讲三个场景,让你理解为什么 display property 重要。

**场景 A: 隐藏长字符的中间部分**。你有一个 40 字符的 git commit hash `a1b2c3d4e5f6...`,想显示成 `a1b2c3d4...e5f6`(头 8 + 尾 4)。但用户复制时,你想让完整 hash 进 kill ring。

实现:

```elisp
(let ((hash "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6"))
  (propertize hash
              'display (concat (substring hash 0 8) "..." (substring hash -4))))
```

`(propertize hash 'display "...")` ——hash 是 40 字符,display 显示 15 字符。Buffer 里 40 字符,redisplay 画 15 字符。光标移动按 40 字符走——所以光标在 hash 上按右箭头要按 40 次才出来。Magit 解决这个问题: 它让 hash 区间 `(field ...)` text property + cursor-intangible,光标跳过。这是更高级技巧。

复制呢? 用户选中 hash 区间,`M-w`,kill ring 存的是 buffer-substring——也就是完整 40 字符。Perfect。如果用 substring 直接插入显示版本,kill ring 就只有 15 字符,丢失了。这是 display property 的优雅——显示美化,数据完整。

**场景 B: 不同 window 不同显示**。你的 buffer 在两个 window 显示。你想: 一个 window 显示完整文件名,另一个 window (窄) 只显示 basename。

display property 支持 `when` 条件:

```elisp
(propertize file
            'display `((when (>= (window-width) 80) ,file)
                       (t ,(file-name-nondirectory file))))
```

但 `when` 的 condition 只在 redisplay 时求值一次——window resize 不会自动更新。要触发更新得 `(force-window-update)`。这是 display property 一个 limitation,但基本能工作。

Magit 用类似机制——它的 status buffer 在 narrow window 显示简短 hunk header,wide window 显示完整。

**场景 C: 显示折叠 (类似 org-toggle-fold)**。Org-mode 的 TAB 折叠是用 `'invisible` text property,不是 `'display`。但 `'display` 也能做类似事:

```elisp
(propertize (buffer-substring start end)
            'display "")
```

整段 display 显示成空字符串——视觉上隐藏。但 buffer 数据还在,光标还能进去。比 `'invisible` 灵活 (可以显示 "[folded]" 这种),但通常用 `'invisible`。

### Display property 的 pitfalls

display property 不是没毛病。几个坑:

第一,**光标移动反直觉**。光标按 buffer 字符移动,但视觉上看到的是 display 字符。如果 display "x" 显示成 "MODIFIED",光标在 x 上按右箭头,光标"消失" (移动到下一字符),但视觉上你以为还在 MODIFIED。用户会困惑。解决: 用 cursor-intangible 属性,让光标跳过这种"视觉长,buffer 短"的字符。

第二,**复制行为反直觉**。复制视觉选中的文本,得到 buffer 字符,不是 display 字符。如果 display 显示 "MODIFIED" 而你选中复制,得到 "x"。新手困惑。

第三,**search 行为反直觉**。`isearch` 搜 buffer 字符,不搜 display 字符。display 显示 "MODIFIED" 但 isearch "MOD" 找不到。新手困惑。

第四,**打印/导出反直觉**。`(buffer-substring)` 拿到 buffer 字符。如果你 export buffer 到 HTML,可能丢 display property (取决于 export 函数)。

这些 pitfall 的根源是设计选择——display 是显示 hint,不是数据。Emacs 选择了"数据优先",这是 Unix 哲学。但它意味着 display property 是"高级用户工具",新手用着可能困惑。Magit、org 用得溜,因为它们的 UX 设计考虑了这些。

所以——display property 是好东西,但谨慎用。如果用错了,用户困惑。如果用对了,体验跃升。

---

## Part 6: button 高级

回到 button。前面讲了 insert-button、button-get、button types。这里讲几个进阶。

### Button 的内部表示

Button 本质上就是 text property。具体来说,一个 button 是一段连续字符,这些字符有 `'button` 属性 = t,`'category` 属性指向 button 类型,可能还有 `'action`、`'face` 等。`button-at` 函数检查这些属性来判断"这是不是 button"。

`button-start` 和 `button-end` 返回 button 的边界——它们用 `next-single-property-change` 找 `'button` 属性变化的位置。

你可以手写 button——不用 insert-button,直接 propertize:

```elisp
(let ((map (make-sparse-keymap)))
  (define-key map (kbd "RET") (lambda () (interactive) ...))
  (define-key map [mouse-1] (lambda () (interactive) ...))
  (propertize "click me"
              'button t
              'category 'my-button-type
              'keymap map
              'face 'link
              'mouse-face 'highlight
              'action (lambda (btn) (message "clicked"))
              'help-echo "click me"))
```

这是 button.el 内部做的事——只是它帮你 make-sparse-keymap + define-key + 设属性。`'mouse-face` 是鼠标 hover 时显示的 face (区别于 `'face` 是默认显示)。

### forward-button / backward-button

用户在 button 之间跳,用 `forward-button` 和 `backward-button`。这俩是命令,可以绑键:

```elisp
(define-key my-gs-mode-map (kbd "TAB") 'forward-button)
(define-key my-gs-mode-map (kbd "<backtab>") 'backward-button)
```

但有个细节——forward-button 默认会跳过 invisible button。如果你的 button 设了 `'invisible` 属性 (display 上的 invisible),它跳过。这通常是好的。

### Button 在 mode line

Mode line 是 Emacs 底部状态条。Mode line 的内容是一个复杂 format (参见 `mode-line-format`),里面可以包含 `(:eval ...)` 形式的 sexp。`(:eval)` 求值的字符串里可以 propertize——所以 mode line 也可以有 button。

但是 mode line 不是 buffer,property 检测有特殊机制。Mode line 的 button 需要一个特殊的 `local-map` property 指向 mode-line keymap。Emacs 内置 `mode-line-highlight` face、`mode-line-buffer-id` 等已经用这种机制。

简单例子:

```elisp
(setq mode-line-format
      '(:eval (propertize " [git] "
                          'local-map (make-mode-line-mouse-map
                                      'mouse-1 (lambda () (interactive)
                                                 (my-gs-buffer)))
                          'mouse-face 'mode-line-highlight)))
```

`make-mode-line-mouse-map` 是 mode line 专用的 helper,创建只绑鼠标的 keymap。

Magit 的 mode-line segment 显示当前 branch、ahead/behind——也是这种机制。

### Button 的 face

Button 默认用 `'button` face (蓝色加下划线,链接风格)。你可以通过 `'face` 属性 override:

```elisp
(insert-button "danger"
               'face 'error
               'action (lambda (btn) (delete-file ...)))
```

但通常用 button type + face:

```elisp
(define-button-type 'my-gs-danger
  'face 'error
  'mouse-face 'highlight)
```

### Button 的 limitation

Button.el 有几个老问题。

第一,button 默认不识别 touchpad tap。一些 platform 上 mouse-1 事件行为不一致。

第二,button 在终端 Emacs 上 mouse support 取决于终端模拟器。xterm 工作,但有些 terminal 不发 mouse event。

第三,button 不能 nested——一段文字是 button A 的一部分,不能同时是 button B 的一部分。因为 text property 是一层。Magit 有时候想要"在文件 button 里有 hunk button",只能用更复杂的 scheme (区分 position,自己写 keymap)。

第四,button 在 narrowed region 里行为奇怪——button-end 可能跑到 narrowed 之外。

这些问题都不致命,但你应该知道。Magit 早期用 button.el,后来迁移到自己写的 text-property keymap 系统,主要就是这些限制。

---

## Part 7: transient.el

到这里我们有一个能看的 git status buffer,RET 能打开文件。但是 git workflow 不只是打开文件——你要 pull、push、branch switch。这些操作你想从 git status buffer 触发,但**不想给每个操作绑一个键**——绑不过来。

Emacs 传统做法是 `completing-read`——按一个前缀键 (比如 `g`),弹一个 completing-read 让用户选 "pull / push / branch"。但是 completing-read 是单选,体验不好——用户记不住"是 g 还是 G,是 p 还是 P"。

更好的做法是**弹一个菜单**,显示所有相关命令和它们的快捷键。用户看到菜单,选一个,菜单消失,执行命令。如果命令有 argument (比如 pull --rebase),菜单支持选 argument。

这就是 Magit 的 popup,后来被作者 Jonas Bernoult 抽出来独立成 transient.el。

Jonas Bernoult 是 Magit 的核心维护者。他大概是 2010 年代开始搞 Magit,2014 年左右 popup 系统成形。一开始 popup 是 magit-popup.el,内嵌在 Magit。2017-2018 年间他意识到这个系统其实通用——任何需要"一组相关命令"的包都能用。于是改名 transient.el,extracted 成独立包,放进 ELPA。Emacs 28+ 内置。

Jonas 这个人有点像 Emacs 社区的 Linus——技术品味强烈,不太妥协。他在 transient 的 README 里直接写: "transient 不是 hydra,不是 which-key,不要拿它们比"。他对 transient 的设计哲学坚持——transient 是"prefix + infix command"模式,严格按照 Emacs 命令系统设计,不是"任意 popup menu"。

为什么叫 transient? 因为这些菜单是"临时的"——你选完命令就消失,不像 traditional UI 弹一个永远存在的对话框。这种"临时菜单"模式在 vim、tmux 里很常见 (那里叫 "transient popup" 或 "which-key")。

transient 的设计有几个核心理念,理解了之后,API 就顺了:

**理念 1: 命令是一等公民**。transient 不是"弹一个 menu,点选项",而是"调用一个 prefix command,prefix command 显示菜单,你按一个键触发 suffix command"。每个 menu item 都是一个 Emacs command (`interactive`)。这保持了 Emacs 一致性——所有东西是 command,可以被 `M-x` 调用,可以绑键,可以 describe (C-h f)。Magit 的 transient menu 里的每个 action 都是个独立命令,用户可以从 `M-x` 直接调,也可以 describe 它。

**理念 2: argument 是 infix command**。"--rebase" 这种参数不是 menu item 的 flag,它本身是个 command。用户按 `r`,执行这个 infix command,效果是 toggle 当前 prefix 的 --rebase argument。然后用户按 `p` 执行 pull command,pull command 通过 `transient-args` 读 argument list。这让 argument 是"first-class"的——它们独立于 command 存在,可以组合。

**理念 3: prefix 是非阻塞的**。transient 弹出菜单后,用户可以无视菜单继续操作 Emacs (切换 buffer、编辑文件)。菜单是 hint,不是 modal。这种"不阻塞"是 Emacs 哲学的延续——Emacs 极少有 modal UI。

**理念 4: 嵌套是自然的**。一个 prefix 的 suffix command 可以是另一个 prefix,弹子菜单。这让复杂 workflow (git rebase --interactive 的 8 个步骤) 可以组织成嵌套菜单。

这些理念合在一起,让 transient 适合"复杂的、多参数的、组合式的" workflow。git 是完美用例——git 有几十个命令,每个命令有几个到十几个常用 argument。Magit 把它们组织成 transient 网格,用户几步按键完成复杂操作。

如果只是"弹一个 menu 选一个",transient overkill,用 completing-read 更简单。

transient.el 的核心 API 是一个宏: `transient-define-prefix`。它定义一个"prefix command"——本身是个命令,调用它弹菜单。

```elisp
(require 'transient)

(transient-define-prefix my-git-menu ()
  "My git menu."
  ["Actions"
   ("s" "status" my-git-status)
   ("p" "pull"   my-git-pull)
   ("P" "push"   my-git-push)]
  [["Branch"
    ("b" "switch" my-git-branch-switch)
    ("B" "create" my-git-branch-create)]
   ["Other"
    ("q" "quit" transient-quit-one)]])
```

逐 token 讲。

`transient-define-prefix` 是个 macro。它的第一个参数是要定义的命令名 (`my-git-menu`)。第二个是 docstring (`"My git menu."`)。然后是 body——一组 vector,每个 vector 是一行 (或一组) menu item。

vector 的 element 可以是:
- 一个 string ——作为标题 (header)。比如 `"Actions"`、`"Branch"`。
- 一个 vector ——作为一组 (一组共享标题,显示在一列)。
- 一个 menu item ——`("key" "description" command)`。

最外层是 vector `["Actions" ...]`,里面是 menu items。也可以是嵌套 vector `[[...][...]]`,每个嵌套是一列。

menu item 是个 list `("key" "desc" cmd)`。`"key"` 是快捷键,用户按这个键执行 cmd。`"desc"` 是描述。`cmd` 是要执行的命令。

所以 `("s" "status" my-git-status)` 意思: 用户按 `s`,执行 `my-git-status`。

`("q" "quit" transient-quit-one)` 意思: 用户按 `q`,执行 `transient-quit-one`——这是 transient 内置的"关闭菜单"命令。

定义之后,`M-x my-git-menu RET`,弹一个菜单 buffer:

```
Actions
s status
p pull
P push

Branch      Other
b switch    q quit
B create
```

按 s/p/P/b/B/q 触发对应命令。其他键无效。C-g 关菜单。

这是个非常紧凑的"快捷菜单"——用户看到所有选项,选一个就走。

注意 transient 不阻塞——它显示菜单 buffer,等你按键。在等的时候你可以正常用 Emacs (虽然一般不会)。按键之后,如果是普通命令,执行然后菜单消失;如果是 infix command (set argument),执行后菜单保留。

### Argument (infix command)

transient 的真正威力是 argument——菜单可以让用户设 argument,后续命令读这些 argument。

例子: pull 默认 `git pull`。但有时候你想要 `git pull --rebase`。可以做成:

```elisp
(transient-define-prefix my-git-menu ()
  "My git menu."
  [:description "Arguments"
   ("r" "rebase" "--rebase")]
  ["Actions"
   ("s" "status" my-git-status)
   ("p" "pull"   my-git-pull :if-not-prefix "--rebase")]
  ...)
```

`("r" "rebase" "--rebase")` ——这是个 infix command。按 `r` 切换 `--rebase` argument。再次按 `r` 关闭。

`:if-not-prefix` 是一个条件——这个 menu item 只在 `--rebase` 没 set 时显示。

然后在 my-git-pull 里读 argument:

```elisp
(defun my-git-pull (&optional args)
  "Run git pull with ARGS."
  (interactive (list (transient-args transient-current-command)))
  (let ((rebase? (member "--rebase" args)))
    (message "Pulling %s..." (if rebase? "with rebase" ""))
    (apply #'call-process "git" nil nil nil "pull"
           (when rebase? (list "--rebase")))))
```

`transient-args` 返回当前 transient 的所有 argument list。`transient-current-command` 是当前 transient 的 symbol。

`interactive (list (transient-args transient-current-command))` 是个 trick——interactive form 在用户调用命令时求值。如果命令是从 transient 调的,`transient-current-command` 是那个 transient,能拿到 args;如果直接 `M-x my-git-pull RET`,没有 transient,args 是 nil。

这种 argument 系统让 transient 远超 completing-read。completing-read 让你选一个 option,transient 让你组合多个 option 然后执行命令。

### Argument 类型

transient 的 argument 不只是 on/off 切换。它支持几种类型,实现不同的 UX:

**Switch**。`:class transient-switch` ——on/off 切换,我们上面用的 `("--rebase")` 就是 switch。按一次设,再按一次取消。显示在 menu 上,旁边可能有 `[X]` 或 `[ ]` 标记当前状态。

**Option**。`:class transient-option` ——有值的 argument,比如 `--author=NAME`。用户按键,在 minibuffer 输入值。例如:

```elisp
(transient-define-prefix my-git-commit-menu ()
  "Commit menu."
  [:description "Arguments"
   ("a" "author" "--author=" :class transient-option)]
  ["Actions"
   ("c" "commit" my-git-do-commit)])
```

用户按 `a`,transient 弹 minibuffer "Author: ",输入 `John <john@x.com>`,设置 `--author=John <john@x.com>`。然后 `c` 执行 commit,commit 命令读 `--author=` 这个 argument。

**Choice**。`:class transient-switches` ——互斥的多选一,比如 `--sort=date` 或 `--sort=name`,二选一。transient 显示成 radio:

```elisp
("s" "sort by" "--sort=%s" :class transient-switches
 :choices ("date" "name" "size"))
```

用户按 `s` 循环 choice。

这些类型让 transient 可以表达复杂的 CLI argument 模式。git 这种 CLI 工具特别合适——几十个命令,每个有几十个 argument,transient 把它们组织成可视化菜单。

### Reading argument 在 command 里

刚才的 `(transient-args transient-current-command)` 返回 list。但有时候你要更精细——比如想知道某个 argument 是否 set,而不是 list 里有没有:

```elisp
(transient-arg-value "--author=" args)  ;; 拿 --author= 的 value
(transient-suffix--scope suffix)         ;; 拿特定 suffix 的 scope
```

实际 API 在 `(info "(transient) Argument Parsing")`。但 90% 情况 `(member "--rebase" args)` 就够了。

### Suffix 的属性

每个 menu item (suffix) 可以有 `:if`、`:if-not-prefix`、`:inapt-if` 等条件属性。这让 menu 是动态的——根据当前状态显示/隐藏某些项。

例如:

```elisp
("u" "unstage" my-gs-unstage-file
 :if (lambda () (my-gs-staged-p (point))))
```

`:if` 是个函数,返回 non-nil 时显示 unstage。这让 transient 根据上下文 (当前文件是否 staged) 显示不同 menu。Magit 大量用这个——比如 rebase menu 在你已经 in rebase 时显示 "continue/abort",不在 rebase 时显示 "start rebase"。

### Layout 灵活性

transient layout 用 vector + nested vector,可以做出复杂结构:

```elisp
(transient-define-prefix my-git-menu ()
  "Complex menu."
  [:description "Pull" :if my-git-in-branch-p
   ("p" "pull" my-git-pull)]
  [:description "Push"
   ("P" "push" my-git-push)
   ("f" "force" "--force" :class transient-switch)]
  [:description "Branch"
   [("b" "switch" my-git-branch-switch)
    ("c" "create" my-git-branch-create)]
   [("d" "delete" my-git-branch-delete)
    ("r" "rename" my-git-branch-rename)]])
```

外层是 column group,中间是 row,内层是 column。transient 自动 layout,wrap 到合适的列数。

学习 transient layout 最快的方式是看 magit 源码。`magit-commit.el`、`magit-pull.el`、`magit-push.el` 都有 transient 定义。读它们你能看到 Jonas 怎么用 layout 表达 git 的复杂 CLI。

### transient 与 button + display 整合

transient 可以嵌套——一个 transient 的 menu item 可以打开另一个 transient:

```elisp
(transient-define-prefix my-git-branch-menu ()
  "Branch submenu."
  ["Actions"
   ("s" "switch" my-git-branch-switch)
   ("c" "create" my-git-branch-create)
   ("d" "delete" my-git-branch-delete)])

(transient-define-prefix my-git-menu ()
  "My git menu."
  ["Actions"
   ("s" "status" my-git-status)
   ("b" "branch" my-git-branch-menu)]  ;; b opens submenu
  ...)
```

按 `b` 打开 branch submenu。子菜单按 `s` 切换 branch,然后 submenu 关闭,回到主菜单 (或主菜单也关闭,看你 config)。

### transient layout 高级

transient 的 vector structure 几个细节:

- 一个 vector 是一行 (横向)。里面的 items 横着排。
- 嵌套 vector 是分组。多个 group 竖着排。
- 每个 group 可以有 header (第一个字符串)。
- `:description` 关键字设 group 描述。

```elisp
(transient-define-prefix my-git-menu ()
  "My git menu."
  [:description "Actions" ;; 第一行
   ("s" "status" my-git-status)
   ("p" "pull"   my-git-pull)]
  [:description "Branch"
   ("b" "switch" my-git-branch-switch)]
  [:description "Other"
   ("q" "quit"   transient-quit-one)])
```

这会显示:

```
Actions
s status
p pull

Branch
b switch

Other
q quit
```

垂直布局。横向要嵌套:

```elisp
[[("s" "status" my-git-status)]
 [("b" "branch" my-git-branch-switch)]]
```

外层 vector 是 row,内层 vector 是 column (反过来 Magit 文档叫法可能不同)。具体读 `(info "(transient)")`。

### transient 与 button + display 整合

transient 自身用的不是 button——它自己画 buffer,有自己的 keymap。但 transient 的设计哲学 (text-property-based,keyboard-first) 和 button 一脉相承。

transient 的菜单 buffer 实际上是这样的: 创建一个 buffer,插入带 face 的文字描述 (`s status`),设一个 buffer-local keymap 绑 `s` 到对应命令。和 button 的实现思路一样,只是 transient 用 buffer-local keymap 而不是 text property keymap (因为整个 buffer 都是 menu,不需要 per-character keymap)。

### 嵌套 transient

transient 可以嵌套——一个 transient 的 menu item 可以打开另一个 transient:

---

## Part 8: 整合到 my-gs 项目

现在我们把所有东西整合——写完整版的 my-git-status-buffer。

特性:
1. 顶部用 display property 大字显示 "Git Status: /path/to/repo"
2. 每行用 emoji/ASCII 图标显示状态
3. 文件名是 button,RET/mouse-1 打开文件
4. 在文件 button 上按 `d` 显示 diff (这个需要先做 git diff 调用)
5. 按 `?` 弹 transient 菜单 (status, pull, push, branch)
6. 自定义 major mode 提供命令

完整代码:

```elisp
;;; my-gs.el --- Simple git status buffer  -*- lexical-binding: t; -*-

(require 'button)
(require 'transient)

;;; Customization

(defgroup my-gs nil
  "Simple git status buffer."
  :group 'tools)

(defcustom my-gs-use-emoji nil
  "If non-nil, use emoji icons."
  :type 'boolean
  :group 'my-gs)

(defface my-gs-title-face
  '((t :inherit font-lock-function-name-face :height 1.5 :weight bold))
  "Face for the title in my-gs buffer."
  :group 'my-gs)

;;; Data source

(defun my-gs--raw-status ()
  "Return list of (status . file) pairs from git status --porcelain."
  (with-temp-buffer
    (call-process "git" nil t nil "status" "--porcelain")
    (goto-char (point-min))
    (let (result)
      (while (not (eobp))
        (let ((line (buffer-substring (line-beginning-position)
                                      (line-end-position))))
          (when (>= (length line) 3)
            (push (cons (substring line 0 2)
                        (substring line 3))
                  result))
          (forward-line 1)))
      (nreverse result))))

(defun my-gs--repo-root ()
  "Return the git repository root directory."
  (with-temp-buffer
    (call-process "git" nil t nil "rev-parse" "--show-toplevel")
    (string-trim (buffer-string))))

;;; Button type

(define-button-type 'my-gs-file-button
  'face 'default
  'mouse-face 'highlight
  'help-echo "RET: open file\nd: diff file"
  'action (lambda (btn)
            (find-file (button-get btn 'my-file))))

;;; Rendering

(defun my-gs--status-face (status)
  "Return face for STATUS string."
  (pcase status
    ((or "M " " M" "MM") 'font-lock-warning-face)
    ((or "A " " A" "AA") 'font-lock-type-face)
    ((or "D " " D" "DD") 'error)
    ("??" 'shadow)
    (_    'default)))

(defun my-gs--status-icon (status)
  "Return display string for STATUS."
  (if my-gs-use-emoji
      (pcase status
        ((or "M " " M" "MM") "📝")
        ((or "A " " A" "AA") "✨")
        ((or "D " " D" "DD") "🗑")
        ("??" "❓")
        (_    "  "))
    (pcase status
      ((or "M " " M" "MM") "M")
      ((or "A " " A" "AA") "A")
      ((or "D " " D" "DD") "D")
      ("??" "?")
      (_    " "))))

(defun my-gs--render-line (status file)
  "Render one line of the buffer as a string."
  (let* ((face (my-gs--status-face status))
         (icon (my-gs--status-icon status))
         ;; icon takes 2 chars in buffer, display as icon
         (icon-str (propertize "XX" 'display icon 'face face))
         (file-str (with-temp-buffer
                     (insert-button file
                                    :type 'my-gs-file-button
                                    'my-file (expand-file-name
                                              file
                                              (my-gs--repo-root)))
                     (buffer-string))))
    (concat icon-str " " file-str "\n")))

(defun my-gs--render-title (path)
  "Render the title line for PATH."
  (let ((title (concat "Git Status: " path)))
    (concat (propertize title 'face 'my-gs-title-face) "\n\n")))

;;; Commands

(defun my-gs-open-file ()
  "Open file at point."
  (interactive)
  (let ((btn (button-at (point))))
    (when btn
      (button-activate btn))))

(defun my-gs-diff-file ()
  "Diff the file at point."
  (interactive)
  (let ((btn (button-at (point))))
    (when btn
      (let ((file (button-get btn 'my-file)))
        (my-gs--run-git-diff file)))))

(defun my-gs--run-git-diff (file)
  "Run git diff on FILE, show in a buffer."
  (let ((buf (get-buffer-create "*my-gs diff*")))
    (with-current-buffer buf
      (read-only-mode -1)
      (erase-buffer)
      (call-process "git" nil t nil "diff" "--" file)
      (goto-char (point-min))
      (diff-mode)
      (read-only-mode 1))
    (display-buffer buf)))

;;; Transient menu

(transient-define-prefix my-gs-menu ()
  "My git status menu."
  ["Actions"
   ("r" "refresh"  my-gs-refresh)
   ("f" "fetch"    my-gs-git-fetch)
   ("p" "pull"     my-gs-git-pull)
   ("P" "push"     my-gs-git-push)
   ("b" "branch"   my-gs-git-branch)]
  ["View"
   ("d" "diff"     my-gs-diff-file)
   ("RET" "open"   my-gs-open-file)]
  ["Quit"
   ("q" "quit"     quit-window)])

(defun my-gs-refresh ()
  "Refresh the my-gs buffer."
  (interactive)
  (my-gs-buffer))

(defun my-gs-git-fetch ()
  "Run git fetch."
  (interactive)
  (call-process "git" nil nil nil "fetch")
  (message "Fetched."))

(defun my-gs-git-pull ()
  "Run git pull."
  (interactive)
  (call-process "git" nil nil nil "pull")
  (message "Pulled.")
  (when (derived-mode-p 'my-gs-mode)
    (my-gs-refresh)))

(defun my-gs-git-push ()
  "Run git push."
  (interactive)
  (call-process "git" nil nil nil "push")
  (message "Pushed."))

(defun my-gs-git-branch ()
  "Branch operations (placeholder)."
  (interactive)
  (message "Branch operations not implemented."))

;;; Major mode

(defvar my-gs-mode-map
  (let ((map (make-sparse-keymap)))
    (define-key map (kbd "?")     #'my-gs-menu)
    (define-key map (kbd "r")     #'my-gs-refresh)
    (define-key map (kbd "d")     #'my-gs-diff-file)
    (define-key map (kbd "RET")   #'my-gs-open-file)
    (define-key map (kbd "q")     #'quit-window)
    (define-key map (kbd "TAB")   #'forward-button)
    (define-key map (kbd "<backtab>") #'backward-button)
    map)
  "Keymap for `my-gs-mode'.")

(define-derived-mode my-gs-mode special-mode "My-GS"
  "Major mode for `my-gs-buffer'.
\\{my-gs-mode-map}")

;;; Entry point

(defun my-gs-buffer ()
  "Show git status in a buffer."
  (interactive)
  (let ((root (my-gs--repo-root))
        (buf  (get-buffer-create "*my-gs*")))
    (with-current-buffer buf
      (read-only-mode -1)
      (erase-buffer)
      (insert (my-gs--render-title root))
      (dolist (entry (my-gs--raw-status))
        (insert (my-gs--render-line (car entry) (cdr entry))))
      (goto-char (point-min))
      (my-gs-mode)
      (read-only-mode 1))
    (switch-to-buffer buf)))

(provide 'my-gs)
;;; my-gs.el ends here
```

逐段讲解。

**Customization 段**。defgroup 定义 group,defcustom 定义可定制变量,defface 定义可定制 face。`:group 'tools` 把 my-gs 放在 tools 组下 (在 customize 顶层能看到)。所有 face / variable 都关联到 `:group 'my-gs`,所以 `M-x customize-group RET my-gs RET` 能看到全部。

**Data source**。`my-gs--raw-status` 我们之前写过,这里加了 `my-gs--repo-root`——通过 `git rev-parse --show-toplevel` 拿到仓库根目录。`string-trim` 去掉 trailing newline。

**Button type**。`define-button-type` 定义 button 类型——共享 action (打开文件)、face、help-echo。`:type 'my-gs-file-button` 在 insert-button 时引用。

**Rendering**。`my-gs--status-face` 用 pcase + or pattern——`(or "M " " M" "MM")` match 这三种状态码 (staged-modified / unstaged-modified / both)。`my-gs--status-icon` 同样,根据 `my-gs-use-emoji` 返回 emoji 或 ASCII。

`my-gs--render-line` 构造一行。`icon-str` 用 `"XX"` 占位 + display property 显示 icon。`file-str` 用 with-temp-buffer + insert-button 构造带 button 的字符串。注意 button 的 `'my-file` 属性是绝对路径——`(expand-file-name file (my-gs--repo-root))`。git status 返回相对路径,我们展开成绝对,这样 find-file 直接能用。

`my-gs--render-title` 把 title 字符串贴上 `'my-gs-title-face`——这个 face 我们在 defface 里设了 `:height 1.5 :weight bold`。所以 title 显示成 1.5 倍字号、加粗。

**Commands**。`my-gs-open-file` 和 `my-gs-diff-file` 都通过 `button-at` 拿当前 button——`(button-at (point))` 返回当前光标位置的 button (或 nil)。如果是 button,`button-activate` 模拟一次"点击"——这会触发 button type 定义里的 action。

`my-gs--run-git-diff` 单独一个 helper——开一个 `*my-gs diff*` buffer,塞 git diff 输出,启用 `diff-mode` (Emacs 内置,语法高亮 diff)。`display-buffer` 显示 buffer (不切换到它,用 `display-buffer` 而不是 `switch-to-buffer`,因为我们想让用户留在 status buffer)。

**Transient menu**。前面讲过的 `transient-define-prefix`。布局: Actions / View / Quit 三行。`("r" "refresh" my-gs-refresh)` 等用户按 r 触发 refresh。

**Major mode**。`define-derived-mode` 派生自 `special-mode` (special-mode 是 Emacs 内置的"只读 buffer 的基础 mode",像 `*Help*`、`*Messages*` 用)。我们定义 `my-gs-mode-map` 绑了 `?`、`r`、`d`、`RET`、`q`、`TAB`。`define-derived-mode` 第二个参数是 parent mode,第三个是 name (modeline 显示),第四个是 docstring。docstring 里的 `\\{my-gs-mode-map}` 自动展开成 keymap 描述——显示在 `C-h m`。

为什么用 special-mode? 因为 special-mode 提供了一些 "special buffer" 行为——`q` 绑到 `quit-window`,buffer 默认 read-only,C-c C-q 之类的特殊操作。Help、Messages、Info 都用 special-mode 派生。

但是——`q` 在 special-mode 里默认绑到 `quit-window`,我们在 my-gs-mode-map 里又显式绑了一遍,因为 transient menu 里也用了 `q`,我们要保持一致。

**Entry point**。`my-gs-buffer` 是主入口。它和之前的 v0.0.1 几乎一样——开 buffer、erase、insert 数据——只是现在用 `my-gs--render-title` 和 `my-gs--render-line`,最后启用 `my-gs-mode` (而不是 `read-only-mode 1`,因为 special-mode 已经是 read-only 的)。

注意 `(my-gs-mode)` 的位置——放在 read-only-mode 之前,因为 my-gs-mode (via special-mode) 会设置一些 buffer-local variable,erase 后 insert 完才启用 mode,这样 mode hook 在正确状态下运行。

实际上更好的写法是:

```elisp
(with-current-buffer buf
  (my-gs-mode)              ;; 先启用 mode
  (let ((inhibit-read-only t))  ;; 临时允许写
    (erase-buffer)
    (insert ...))
  (goto-char (point-min)))
```

`inhibit-read-only` 是个变量,non-nil 时允许写 read-only buffer。这是 Emacs 处理 read-only buffer 写入的标准 idiom。我之前的代码用 `(read-only-mode -1) ... (read-only-mode 1)` 也能工作,但不够 idiomatic。inhibit-read-only 更好——它不切换 mode,只是临时绕过。

把整个文件 load 起来,`M-x my-gs-buffer RET`,你看到一个 buffer:

```
Git Status: /home/user/myrepo

📝 src/main.c
📝 README.md
❓ new-file.txt
```

(如果 `my-gs-use-emoji` 是 t;否则用 M/M/??)

光标移到 src/main.c,RET 打开。`d` 显示 diff。`?` 弹 transient 菜单。`r` 刷新。`q` 关闭。

差不多 150 行代码,一个能用的 git status 工具。这就是 Emacs 的 power——你不需要 Electron,不需要 React,你只需要 text property 和一些约定。

### 与 magit-status 比较

诚实地讲: my-gs 缺很多。

- 没 staging。Magit 的 `s` (stage)、`u` (unstage) 是核心。我们没做。
- 没 commit。Magit 有 `c` 弹 commit transient,支持很多 commit 选项。
- 没 log。Magit 的 `l` 是 magit-log,显示 commit 历史。
- 没 diff expand/collapse。Magit 在 status buffer 里直接 expand hunk 显示 diff,光标在 hunk 上按 TAB 折叠。
- 没 rebase。Magit 的 rebase interactive 是杀手锏。
- 没 stashes,没 submodules,没 forge (PR 集成)。

但是——my-gs 是学习 Emacs 显示系统的好工具。它覆盖了:
- text property (face)
- button (insert-button, button-at, button-get, button type)
- display property (icon 替换、字号变化)
- transient (popup 菜单)
- major mode (keymap, parent mode)

这些是 Emacs 90% 的"扩展 UI"模式。理解了 my-gs,你看 magit 源码会容易很多——因为 magit 也是这些原语,只是更多更精细。

### Magit 是怎么扩展这个模型的

讲讲 Magit 在这个骨架上加了什么,让你知道"前面还有什么"。

**1. Section 抽象**。Magit 自己定义了一个 "section" 类型——比 button 更大的结构。一个 section 包含: heading、children、visibility state (expanded/collapsed)、keymap、数据。Section 嵌套形成树 (status buffer 顶层 section = 仓库,子 section = staged/untracked/modified groups,孙 section = 文件,曾孙 section = hunks)。

Magit 用 text property 标记 section 边界,但 section 本身是 Emacs Lisp object (一个 struct),存在 `magit-section--cache` buffer-local variable 里。Text property 只用来标记 "这个字符属于 section X"——X 是个 id。

Section 是 button 的"超集"——它支持折叠、嵌套、批操作。如果你做复杂 UI,section 模式值得学。Magit 把 section 抽到独立的 magit-section.el,你可以 `M-x package-install RET magit-section RET` 单独用。

**2. Process buffer**。Magit 跑 git 命令时,显示 process buffer——带 face 高亮的 git 输出。这给我们展示了 "如何把外部命令输出展示得好看":每行 parse,根据内容贴 face (`error` face 给 "ERROR:" 开头的行,`success` face 给成功消息)。

**3. Diff buffer with hunks**。Magit 的 diff 不只是 plain text——它把 diff 解析成 hunks,每个 hunk 是个 section,可以折叠。diff 行用 `diff-added`、`diff-removed` face 高亮。Magit 还允许在 diff buffer 里直接编辑 (修改 + 行,自动 adjust - 行)——这是 buffer 编辑的高级用法。

**4. Log buffer with graph**。Magit 的 log 显示 commit graph (合并/分叉的 ASCII 图)。这用 `(space :align-to N)` 对齐到列,显示 graph 字符。

**5. Worktree 支持**。Magit 可以同时管多个 worktree。这涉及多个 buffer 同步状态——通过 `magit-refresh-buffer` 系统。

这些都是 my-gs 之上扩展。如果你有兴趣,挑一个加到 my-gs——比如 stage 操作 (`s` 切换 staged/unstaged,运行 `git add`/`git restore --staged`)。

### 一个真实场景: 加 stage 操作

我手把手加一个 stage 操作,让你看 my-gs 怎么扩展。

需求: 文件按钮上按 `s`,运行 `git add <file>` (如果未 staged) 或 `git restore --staged <file>` (如果已 staged)。然后刷新 buffer。

```elisp
(defun my-gs-stage-file ()
  "Stage or unstage file at point."
  (interactive)
  (let ((btn (button-at (point))))
    (when btn
      (let* ((file (button-get btn 'my-file))
             (status (button-get btn 'my-status))
             (relative (file-relative-name file (my-gs--repo-root))))
        (if (string-prefix-p " " status)  ;; unstaged
            (call-process "git" nil nil nil "add" "--" relative)
          (call-process "git" nil nil nil "restore" "--staged" "--" relative))
        (my-gs-refresh)))))

;; 在 my-gs-mode-map 加
(define-key my-gs-mode-map (kbd "s") #'my-gs-stage-file)
```

我们在 file button 上存 `'my-status` 属性 (修改 my-gs--render-line 加上)。`string-prefix-p " " status` 检查状态码第一个字符是不是空格——空格表示 unstaged (`" M"`)。

`file-relative-name` 把绝对路径转回相对——git 命令需要相对路径。

`my-gs-refresh` 重画 buffer。

这就是一个完整功能的扩展——30 行代码。你能看到 Emacs 的扩展性来自哪里: text property 让数据流向命令,命令调外部工具,然后重画 buffer。所有逻辑都是这种"取属性 + 调命令 + 重画"的循环。

### 整体复盘

我们这一节做了什么:

- 定义 customize group / variable / face
- 用 git porcelain 输出做数据源
- 用 button type + button properties 做 interactive 文件按钮
- 用 display property 显示状态图标
- 用 display property + face 做大字 title
- 用 transient 定义 popup menu
- 用 define-derived-mode 定义 major mode
- 用 `?` key 触发 transient
- 整合所有部件

整个文件 150 行。Magit 是这种结构的几十万行版本。差异在 scale 和精细度,不在本质。

如果你能完整理解这 150 行,Magit 源码的可读性会大幅提升。你不再被"这是 button? section? overlay?"困惑——你知道每块是什么,为什么这样。

这就是"用项目学 Emacs"的价值。读 manual 你学到 API,做项目你学到设计。

---

## Part 9: 进一步

Emacs 还有几个没讲的"高级显示"工具,简短介绍。

**widget.el**。widget 比 button 更高级——它是"表单组件",类似 HTML form。Emacs 的 `M-x customize` 界面就是 widget 实现的: 文本框、checkbox、radio button、下拉菜单、按钮。widget 维护一个 widget 对象 (不只 text property),有自己的 value、callback。适合做复杂的配置 UI。学习曲线陡,不推荐小工具用。

**tabulated-list-mode**。这是 Emacs 内置的"表格 list" mode。`M-x list-packages`、`M-x proced` 都用它。你提供一个 tabulated list data (一个 vector of entries,每个 entry 是 `[id [col1 col2 ...]]`),tabulated-list-mode 帮你渲染对齐的表格,支持 sort、filter。如果你做"显示 list of items with columns"的工具,先看 tabulated-list-mode。

**button + image (Magit 风格)**。Magit 的 hunk line 上有"展开/折叠"图标——这是 button + image 的组合。button 的 face 不是普通 face,是带 image 的 display property。点击 button 触发 expand/collapse。实现复杂但视觉好。

**overlay vs text property**。我没讲 overlay——overlay 是 text property 的"非数据"版本。text property 是 buffer 数据的一部分 (字符上的属性);overlay 是 buffer 上独立的 interval object,有起止位置,有自己的 property。修改 overlay 不修改字符。用途: highlight 临时区间 (像 isearch 高亮)、读 buffer 时临时标记。一般 UI 永久内容用 text property,临时效果用 overlay。

**cursor-sensor-mode**。让 cursor 进入/离开某段文字时触发函数——基于 text property。可以实现"光标停在文件名上,echo area 显示 diff stat"这种。

**pixel-resize**。Emacs 27+ 支持像素级 height——`:height 100` 是 100 像素高 (不是 100 倍)。这让 display property 能做更精细的字号控制。

**line-prefix / wrap-prefix**。这两个 text property 让你给 buffer 每行加 prefix——line-prefix 是首行 prefix,wrap-prefix 是续行 prefix。Org 的 quote block 缩进、outline 的层级缩进,都用这俩。

**shr / eww**。shr 是 HTML-to-text 渲染引擎 (用 text property)。eww 是基于 shr 的 Emacs 浏览器。看 shr 源码是学高级 text property 用法的好教材——它用 face、display、keymap、button 实现完整的 HTML 渲染。

shr 把 `<a href>` 渲染成 button (鼠标可点,RET 打开),`<b>` 渲染成 bold face,`<img>` 渲染成 image display property,`<table>` 用 space :width 对齐成表格。所有这些只用 text property。读 `shr.el` 源码,你会看到所有今天学的技术的真实应用。

**emoji 包 (emojify, emacs-emoji)**。Emacs 28+ 内置 emoji 显示支持,但需要正确字体。emojify 是个第三方包,提供 emoji 渲染 + emoji 输入。它用 display property 把 unicode 替换成 image (老版本),或直接信任 unicode emoji (新版本)。看 emojify 源码也是学 display property 的好教材。

**all-the-icons / icon-frame**。这俩包提供 Nerd Font / SVG 图标库。它们集成到 dired、ibuffer、treemacs 等工具,用 display property 替换字符成图标。看源码学习"图标系统的工程化"——如何 fallback (终端不支持)、如何 cache (图片加载慢)、如何自定义。

**lsp-mode / eglot 的 flymake 显示**。LSP 错误下划线、flymake diagnostics——它们用 text property 的 `'face` (with underline style)、`'help-echo` (hover 显示错误)、`'flymake-overlay` (overlay 而非 text property)。如果你做编辑器扩展,看 flymake.el 怎么把 diagnostics 渲染到 buffer。

**company / corfu 的 popup**。这俩是 completion UI。它们**不用 text property**——它们用 child frame 或 overlay 在 buffer 上方/下方"浮"一个 popup。child frame 是 Emacs 26+ 引入的"独立小窗口",可以做 shadow、transparency。这是 Emacs UI 的另一条路径——不修改 buffer,而是叠一层窗口。corfu 是现代实现,读源码学这条路径。

**tree-sitter / treesit**。Emacs 29+ 内置 tree-sitter。treesit 不直接 propertize——它给你一个 incremental parse tree,你 (或 font-lock 系统) 用它 propertize buffer。如果你做 syntax highlighting,treesit 是未来。

**`window-text-pixel-size`**。Emacs 27+ 引入——以像素为单位测量文字宽度。这让"按像素对齐"成为可能 (而不是按字符)。某些复杂 layout 用它精确测量。

---

## 设计哲学回顾

走完这一路,回头看看 Emacs 显示系统的整体设计哲学。

**字符 + 属性**,而不是对象树。这是 Emacs 最重要的设计选择。它简化了所有 buffer 操作 (search、kill/yank、io),代价是"组件抽象"较弱 (你没法 button.setEnabled())。

**显示与数据分离**。`'display` 属性让 buffer 数据保持纯净,显示可以美化。这跟 Unix "tools process text, end of story" 哲学一致。

**约定优于配置**。Button 不是新对象类型,它是一组 text property 约定 (`'button`、`'category`、`'action`)。任何代码读这些约定都识别 button。这让 button 跨包互通,不需要 import / type system。

**Major mode + minor mode 装饰**。Major mode 提供 buffer 主要行为 (像 my-gs-mode),minor mode 提供横切行为 (linum、flyspell)。这俩协同——buffer 显示是它们共同作用的结果。

**Lisp machine 的味道**。这一切设计——buffer 当结构化数据、property 当 metadata、face 当 attribute——是 Lisp machine 时代 (Symbolics、MIT LispM) 的设计哲学。Emacs 是这套哲学的最后一个活化石。理解 Emacs 你就理解了 Lisp machine。

这套哲学的代价:

**学习曲线陡**。新手不知道 text property 怎么用,觉得 Emacs UI"老土"。但老土是表面——内部是精妙的设计。

**调试难**。Text property 看不见——你得用 `(get-text-property (point) ...)` 查。Emacs 没 devtools 那种 UI inspector。社区做了一些工具 (`what-cursor-position`、`describe-text-properties`),但不直观。

**性能**。大量 text property 的 buffer,redisplay 慢。Emacs 27+ 做了很多优化 (cache、batching),但极限情况下还是慢——所以 magit 在超大仓库卡。

但 Emacs 的 trade-off 给了它寿命——40 年前的 .el 文件今天还能跑。这种 longevity 没有第二个编辑器框架能做到。

---

## 结语

我跟你走了一遍。从最朴素的 `(insert "M file.txt")`,到 face、keymap、display property、button、transient,最后整合成一个能用的工具。

你应该有的认知:

第一,Emacs 的显示系统是**字符 + 属性**。没有 component,没有 widget tree。所有"组件"都是 text property。

第二,text property 不只是显示。它是 buffer 上的通用 metadata。face 是显示,keymap 是交互,你的自定义 property 是数据。

第三,button.el 是 text property 的语法糖,有标准化但灵活性低。Magit 自己写,因为它需要更细粒度。

第四,display property 是"显示 ≠ 数据"的关键。你想美化显示但不污染数据,用 display。

第五,transient 是 popup 菜单标准,Jonas Bernoult 从 Magit 抽出来。适合"一组相关命令"的场景。不要用 completing-read 当万金油。

如果这五条你都理解了,这份教程就达到了目的。

最后的最后——我希望你不仅学到了 API,更学到了**老程序员面对问题的思考方式**: 先想数据模型,再想显示,最后想交互。Emacs 之所以 40 年还能用,就是它把数据模型 (字符 + 属性) 想透了。其他东西都是这一层的衍生。

去读 magit 源码吧。`magit-status.el`、`magit-diff.el`、`magit-process.el`。你会看到所有今天讲的东西,以更复杂、更精细的形式出现。然后你会感激,Emacs 给了你这套原语。
