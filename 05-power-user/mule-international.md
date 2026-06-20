# Mule (国际化) 详解

> 学完这个文件,你能用 Emacs 编辑任何语言的文本,处理编码问题,配置 CJK 字体

Mule (MULtilingual Enhancement) 是 Emacs 处理多语言文本的能力——这听起来"高大上",但实际就是回答几个朴素问题: 中文怎么输入?老文件打开是乱码怎么办?中英日韩字体怎么对齐?emoji 怎么显示?

在 2026 年的语境里,UTF-8 已经一统天下,大多数编辑器"默认就能用"多语言——但 Emacs 的 Mule 系统在底层做了几件**没别的编辑器做的事** (内置输入法、charset 分组、字体按 charset 配置),理解这些会让你在处理多语言文本时**不慌**——遇到乱码、字体错位、输入法问题,你知道每个按钮在哪。

---

## 1. 为什么编辑器要支持 Unicode

### 1.1 第一性原理推导

为什么这事是大事?

- **第一性原理**: 文本是字符序列。但计算机存的是字节——需要约定"字节 → 字符"的映射 (编码)。
- **推论 1**: 早期编辑器只支持 ASCII (单字节)——这只够英文。其他语言 (中文、日文、阿拉伯文) 用多字节,但每个文化有自己的方案 (GB2312, Shift-JIS, Big5, KOI8-R...)。
- **推论 2**: 同一字节流可能在不同编码下被解读为完全不同的字符——这是"乱码"的根源。
- **推论 3**: Unicode (1991 年) 提出一个统一方案——所有字符都给一个 codepoint (U+4E2D 是中)。UTF-8 是 Unicode 的一种编码方式 (变长字节)。
- **推论 4**: 一个**国际编辑器**应该: (a) 内部用 Unicode 表示字符 (避免数据丢失);(b) 读写外部文件时正确转换编码;(c) 让用户输入任何语言的字符;(d) 用合适字体显示任何字符。

这四条推论得到 Mule 的设计: **内部全 Unicode,外部接口 (文件读写) 可配置编码,内置输入法,字体按 charset 配置**。

Emacs 从 1997 年 (20.x) 起内置 Mule——这是 Emacs 走向国际化的里程碑。当时大多数编辑器还只能 ASCII。这种"早期投入国际化"的设计,让 Emacs 在 2026 年处理多语言仍然从容。

### 1.2 内部表示

关键洞察: **Emacs 内部不存"UTF-8 字节",它存"Unicode codepoint"**。这意味着你打开一个文件 (`utf-8`),Emacs 把它解码成内部字符 (codepoint),编辑;保存时再编码回 UTF-8。

这听起来细节,但意义巨大: **Emacs 内部永不丢信息**。即使你的文件是 GB2312 编码,Emacs 解码后内部仍然是 Unicode——你可以用 `set-buffer-file-coding-system` 改成 UTF-8 保存,**没有字符丢失**。这种"解码-编辑-编码"模型是 Mule 的核心。

### 1.3 Charset 概念

Emacs 用 **charset** (字符集) 这个抽象——比 Unicode 更细。Unicode codepoint 被分类到 charset:
- `ascii` (0-127)
- `latin-iso8859-1` (西欧扩展)
- `CJK` (中日韩汉字,在多个 charset: `chinese-gb2312`, `japanese-jisx0208`, `korean-ksc5601`)
- `emoji`
- 等等

这种分类让你可以**按 charset 设字体**——例如"所有 CJK 字符用思源黑体,emoji 用 Apple Color Emoji"。这就是为什么 Emacs 能做到完美的 CJK 字体对齐 (社区著名的痛点)。

---

## 2. 编码系统

### 2.1 常见编码

2026 年的编码生态:

```
UTF-8         现代标准,变长字节 (1-4),覆盖所有 Unicode
UTF-16        Windows 内部用,定长 2 字节 (大多数)
GBK / GB18030 中国大陆遗留 (Windows 中文版)
Big5          台湾/香港遗留
Shift-JIS     日本遗留
EUC-KR        韩国遗留
ISO-8859-1    西欧单字节遗留
```

UTF-8 一统天下,但**老文件**仍然存在——尤其是在中国大陆,1990-2010 年的文件大多是 GB2312/GBK,如果你用 UTF-8 打开会乱码。

Emacs 的编码符号:
- `utf-8` (默认,Unix line ending)
- `utf-8-unix` / `utf-8-dos` / `utf-8-mac` (显式行结束符)
- `gb2312`, `gbk`, `gb18030`
- `big5`
- `shift_jis` / `euc-jp`
- `iso-8859-1`

行结束符 (`-unix` / `-dos` / `-mac`) 是 Emacs 编码符号的一部分——一个编码符号 = 字符编码 + 行结束约定。

### 2.2 看当前编码

```
C-x RET f    set-buffer-file-coding-system (改)
C-h C RET    describe-coding-system (查看当前)
```

`C-h C RET` (大写 C) 在 echo area 显示当前 buffer 的编码——例如 "Coding system for saving this buffer: utf-8-unix"。这是诊断编码问题的第一步。

或看 mode line——Emacs 26+ 在 mode line 显示编码简写 (`:utf-8:` 或 `U:` 等)。

### 2.3 改编码

**场景一**: 打开了文件,想换编码保存:

```
C-x RET f    然后: utf-8-unix RET
C-x C-s      保存
```

文件被重写成新编码——内部字符不变,只是字节表示变了。这是"转码"的标准操作。

**场景二**: 文件已经是乱码 (解码用了错的编码)——需要重新解码:

```
M-x revert-buffer-with-coding-system RET
gb2312 RET
```

这告诉 Emacs "用 GB2312 重新解码这个文件"——如果文件本来就是 GB2312,现在显示正确。这是**修乱码**的核心命令。

**场景三**: 一次性命令前指定编码:

```
C-x RET c    universal-coding-system-argument
gb2312 RET
C-x C-f 文件名 RET
```

`C-x RET c` 设置一个"下一个命令用的编码"——之后的 `C-x C-f` 用这个编码打开文件。这只影响这一次操作,不改变全局。

### 2.4 默认编码

```elisp
(prefer-coding-system 'utf-8)
(set-language-environment "UTF-8")
(set-default-coding-systems 'utf-8)
(set-terminal-coding-system 'utf-8)
(set-keyboard-coding-system 'utf-8)
```

这五行是中文用户的标准 init.el 配置——把所有默认编码设为 UTF-8。99% 的情况下这就够了,你不需要再想编码。

`prefer-coding-system` 是关键——它告诉 Emacs "优先用 UTF-8 解码未知编码的文件"。配合自动检测 (下面),绝大多数文件打开就是对的。

### 2.5 自动检测

Emacs 有自动检测编码的能力——`set-language-environment` 配置后,Emacs 用一组优先级猜测未知文件的编码:

```elisp
(setq coding-system-for-read 'utf-8-with-signature)
(setq auto-coding-regexp-alist
      '(("gb\\(2312\\|k\\)" . gb18030)
        ("\\`Shift_JIS" . shift_jis)))
```

但自动检测**不可靠**——UTF-8 和 GB2312 的字节模式有重叠,Emacs 可能猜错。遇到乱码,你**手动用** `revert-buffer-with-coding-system` 切换。

### 2.6 看 char 详细信息

最重要的诊断工具:

```
C-u C-x =
```

(在光标字符上按)

这会显示一个完整 buffer,列出该字符的所有信息:
- character: 中
- codepoint: #x4E2D (U+4E2D)
- charset: `chinese-cns11643-1` (或 `CJK`)
- preferred coding system: `chinese-big5` / `gb2312`
- buffer file coding system: `utf-8-unix`
- font: `-...-Noto Sans CJK SC-...`
- face: `default`

这是诊断字符问题的瑞士军刀——比如某个字符显示成方块,你看 font 字段就知道用了什么字体,然后调整。

普通 `C-x =` 只显示 char 和 codepoint——`C-u C-x =` 显示全部。

---

## 3. 输入法

### 3.1 Emacs 内置输入法

Emacs 内置了一组输入法,基于 **Quail** 系统。Quail 是 Emacs 的"输入法框架"——和系统输入法 (fcitx, ibus, macOS pinyin) 是**两套独立系统**。

切换输入法:

```
C-\           toggle-input-method
M-x set-input-method RET chinese-py RET
M-x describe-input-method    看当前输入法
```

`C-\` 是关键——按一次开输入法,再按一次关。这个键你在用 Emacs 输入中文时会按几十次。

### 3.2 中文输入法

Emacs 内置的中文输入法:

```
chinese-py         全拼
chinese-tonepy     带声调全拼
chinese-bxnpy      双拼
chinese-cns        CNS 注音
chinese-punct      中文标点
chinese-py-punct   拼音 + 标点
chinese-4corner    四角号码
chinese-etzy       仓颉 (倚天版)
chinese-zozy       仓颉
chinese-ccdjpy     ?
```

最常用的是 `chinese-py` (全拼)。设默认:

```elisp
(setq default-input-method "chinese-py")
```

启动 Emacs 后,`C-\` 就用这个输入法。

### 3.3 Quail 系统简介

Quail (Quoted Input Attributed to Input Language) 是 Emacs 23+ 内置的输入法框架——它**完全在 Emacs 里**实现,不依赖外部 IME。

Quail 的工作方式:
1. 用户按键 (例如 "zhong")
2. Quail 查表,显示候选 (中, 种, 重, 钟, ...)
3. 用户选数字 (1, 2, 3, ...)
4. Quail 把候选字符插入 buffer

这种"框架 + 表"的设计让任何人都能写输入法——`chinese-py` 的核心就是一个 `~/.emacs.d/leim/quail/PY.el` 文件,存"zhong → 中,种,..."的映射。

但 Quail 的局限: 词组不全、智能联想弱、不能云输入。所以大多数人**用系统输入法**——它更智能。Emacs 输入法的价值是: (1) SSH 远程编辑时,没有系统 IME; (2) Emacs 完全自包含,不依赖系统配置。

### 3.4 与系统 IME 的关系

Emacs 接受系统 IME 的输入——你用 fcitx (Linux) 或 macOS pinyin,Emacs 接到字符插入。这通常是默认行为,不需要配置。

但有一个经典问题: **macOS GUI Emacs 用 IME 时,光标位置错位** (输入框弹在屏幕角落,不在光标处)。这是 Emacs 的 IME 集成 bug,2026 年仍有讨论。解决:

```elisp
(use-package emacs-everywhere
  :ensure t)
```

或换更现代的 patch (Emacs 28+ 改进了不少)。许多人索性用 Emacs 内置输入法 (`chinese-py`) 避免这个 bug——在 Emacs 内不再依赖 IME。

### 3.5 其他语言

```
japanese           日文
japanese-ascii     日文 + ASCII
japanese-kana-kun  仮名
korean-hangul      韩文
vietnamese-telex    越南文
russian-computer    俄文
hebrew              希伯来文
arabic              阿拉伯文
greek               希腊文
```

切换日语输入法:

```
M-x set-input-method RET japanese RET
```

之后输入日文假名。Emacs 内置支持几十种语言——完整列表 `M-x list-input-methods`。

---

## 4. 字符信息

### 4.1 describe-char

```
C-u C-x =    describe-char (光标处字符详情)
```

显示一个 buffer,包含:
- character (字符本身)
- charset (属于哪个字符集)
- codepoint (in hex: `#x4E2D`)
- code point in charset
- script (CJK, Latin, Cyrillic, ...)
- syntax (在 syntax table 里的类别)
- bidi-class (双向文本类型,for 阿拉伯文)
- category
- to input (Quail 怎么输入这个字符)
- buffer file coding system
- font (用哪个字体显示)
- face (面,颜色)

这是诊断字符的终极工具——任何"字符显示不对"的问题,先 `C-u C-x =` 看。

### 4.2 insert-char

输入任意 Unicode 字符:

```
C-x 8 RET    insert-char
```

可以按 codepoint (`4e2d` 输入"中") 或按名字 (`CJK UNIFIED IDEOGRAPH-4E2D` 或直接 `middle`)。Emacs 内置 Unicode 字符名表——你可以按"heart"找到 ❤,按"check"找到 ✓。

快捷插入常用字符:

```
C-x 8 e     emoji 选择器 (Emacs 28+)
C-x 8 '     Apostrophe variants
C-x 8 "     引号 variants
C-x 8 -     短横 variants
```

`C-x 8 e` 在 Emacs 28+ 是一个**emoji 选择器**——弹出 buffer,你浏览选择 emoji,RET 插入。这是 2026 年 Emacs 内置的好功能。

### 4.3 describe-char on emoji

emoji 是 Unicode 字符——比如 🎉 是 U+1F389。`C-u C-x =` 显示:

```
character: 🎉
codepoint: #x1F389
name: PARTY POPPER
general-category: So (Symbol, Other)
font: Apple Color Emoji
```

`name` 字段告诉你"这是 PARTY POPPER"——查询 emoji 的常用方式。

### 4.4 what-cursor-position

普通 `C-x =` (无 `C-u`) 显示简短信息:

```
char: 中 (0423554, 177620)
point=42 of 100 (42%) column=4
```

`char` 后面的数字是 octal (八进制,Emacs 老式习惯)。

---

## 5. 字体配置

### 5.1 基础

Emacs 字体配置的核心函数:

```elisp
(set-face-attribute 'default nil :font "JetBrains Mono-14")
```

这是英文字体的标准设置——14 号 JetBrains Mono。但这**只影响 ASCII**——CJK 字符不会自动用这个字体 (因为 JetBrains Mono 没有 CJK glyph)。

Emacs 找不到 glyph 时,fallback 到 fontset 的下一个字体——默认是某个系统 CJK 字体 (在 macOS 是 PingFang,在 Linux 是 Noto)。这通常**能用**,但 CJK 字体的字号、行高、基线可能和英文字体不匹配——出现"中文比英文高半行""中文方块不对齐"的问题。

### 5.2 fontset (字体集)

Emacs 用 **fontset** 概念——一个 fontset 是"按 charset 分配字体的表":

```elisp
(set-fontset-font t 'han "Noto Sans CJK SC")
(set-fontset-font t 'kana "Noto Sans CJK JP")
(set-fontset-font t 'hangul "Noto Sans CJK KR")
(set-fontset-font t 'cjk-misc "Noto Sans CJK SC")
(set-fontset-font t '(#x1F300 . #x1F9FF) "Apple Color Emoji")
```

- 第一个参数 `t`: 应用到所有 fontset (或 `'fontset-default`)
- 第二个参数: charset (或 cons 表示 codepoint 范围)
- 第三个参数: 字体名

`'han` 是中文汉字 charset,`'kana` 是日文假名,`'hangul` 是韩文。这种"按 charset 分字体"让你**精确控制每种字符的字体**——这是 Emacs 字体配置的灵魂,VSCode 等没有这种粒度。

### 5.3 CJK 字体对齐

最常见的痛点——中英文字体大小不一致,中文"飞起":

```elisp
(defun my/setup-cjk-fonts ()
  "对齐 CJK 字体。"
  (set-fontset-font t 'han "Source Han Sans CN")
  ;; 调整 CJK 字体的字号 scale
  (setq face-font-rescale-alist
        '(("Source Han Sans CN" . 1.0)
          ("PingFang SC" . 1.0))))

(when (display-graphic-p)
  (my/setup-cjk-fonts))
```

`face-font-rescale-alist` 是"字体缩放表"——某些字体的字号按比例缩放。1.0 表示不缩放;0.9 表示缩小到 90% (解决"CJK 字体看起来太大"的问题)。

调到完美对齐通常需要试验——你设好之后,打开一个中英混排的文件,看是否对齐。

### 5.4 列出可用字体

```
M-x describe-font RET   看当前字体
M-x font-selection      选字体 (如果有)
```

或用 Elisp:

```elisp
(font-family-list)     ; 返回所有可用字体 family
```

这返回系统所有字体——你 grep 找有没有"Source Han"或"Noto"。

---

## 6. 多语言文档

### 6.1 一个 buffer 混多语言

Emacs 完全支持——一个 buffer 里同时有中文、英文、日文、韩文、阿拉伯文、emoji。所有字符在内部都是 codepoint,显示用 fontset——fontset 自动按 charset 选字体。

```
你好世界 (CJK)
Hello, world (Latin)
こんにちは (Kana)
안녕하세요 (Hangul)
مرحبا (Arabic,RTL)
🎉 (Emoji)
```

这种"多语言共存"是 Emacs 的强项——你**不用切输入法也能编辑** (你切的是输入法,不是字符集)。`C-\` 切换输入法,只影响你**输入**时用什么——已有的字符都是 Unicode,显示和编辑都正常。

### 6.2 双向文本 (RTL)

阿拉伯文、希伯来文是 RTL (右到左)。Emacs 24+ 支持 bidi (双向文本) 显示:

```elisp
(bidi-paragraph-direction 'left-to-right)  ; 强制段落方向
```

通常 Emacs 自动检测——段落以 RTL 字符开始就 RTL 显示,以 LTR 字符开始就 LTR。这种"自动双向"和现代浏览器一样。

混合 LTR/RTL 的复杂情况 (例如阿拉伯文里嵌英文),Emacs 用 Unicode Bidirectional Algorithm (UAX #9) 处理——这是 2010 年代后的标准。

### 6.3 多语言搜索

`isearch` (`C-s`) 在多语言文本里完全工作——你可以搜中文、日文、emoji。Emacs 内部用 codepoint 比较,不区分语言。

正则匹配也行——你可以用 `[一-龥]` 匹配所有 CJK 汉字 (U+4E00 到 U+9FA5)。

---

## 7. 终端的国际化

### 7.1 TTY 限制

在终端 (`emacs -nw`) 跑 Emacs,有 Mule 的几个限制:

- **字体配置无效**: 终端用终端的字体,`set-fontset-font` 没效果。
- **Tooltip 无效**: 没有弹窗。
- **IME 集成**: 终端的 IME 是终端的事,Emacs 只是接收字符。
- **某些字符显示不正常**: emoji 在终端可能显示成方块或乱码。

所以如果你需要在终端用 Emacs 编辑多语言,**用 GUI Emacs 更好**。GUI Emacs 的 Mule 体验完整。

### 7.2 终端编码

在终端跑 Emacs,terminal 必须是 UTF-8:

```bash
$ echo $LANG
en_US.UTF-8
```

如果终端是 `C` 或 `POSIX` locale,Emacs 内部用 UTF-8 但终端不显示——字符会乱。配置 terminal locale (在 `.bashrc` / `.zshrc`):

```bash
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
```

---

## 8. Lisp 视角

### 8.1 字符 vs 字符串

在 Elisp 里,字符是**整数** (Unicode codepoint):

```elisp
?中                       ; => 20013 (U+4E2D)
?❤                        ; => 10084 (U+2764)
(char-to-string ?中)      ; => "中"
(string-to-char "中")     ; => 20013
```

`?中` 这种"问号 + 字符"语法在 Elisp 里表示字符。多字节字符也支持——`?中` 就是 20013。

字符串是字符序列:

```elisp
(length "中文")           ; => 2 (2 个字符)
(string-bytes "中文")     ; => 6 (UTF-8 编码 6 字节,每中文 3 字节)
```

`length` 返回**字符数**,`string-bytes` 返回**字节数**——这是 Mule 的精细处。在 Elisp 里你通常关心字符数 (你想知道"几个字"),所以 `length` 是对的。

### 8.2 编码转换

```elisp
(decode-coding-string (string-as-unibyte (encode-coding-string "中" 'gb2312)) 'gb2312)
; => "中"
(encode-coding-string "中" 'utf-8)        ; => #("\344\270\255" ...)
(decode-coding-string "\310\325\261\276" 'euc-jp)  ; => "日本"
```

这些函数用于处理字节流——通常你不需要直接调,但写网络/文件 IO 时可能用到。

### 8.3 提取 CJK 字符

实用函数——从一段文本里提取所有 CJK 字符:

```elisp
(defun my/extract-cjk (string)
  "提取 STRING 里所有 CJK 汉字。"
  (let ((result ""))
    (dolist (char (string-to-list string))
      (when (and (>= char #x4E00) (<= char #x9FFF))
        (setq result (concat result (char-to-string char)))))
    result))

(my/extract-cjk "Hello 世界 123 你好")  ; => "世界你好"
```

这是"过滤非中文字符"的常用 snippet——提取中文文本、统计中文比例等。

### 8.4 charset 查询

```elisp
(char-charset ?中)         ; => chinese-gb2312 (或 CJK)
(char-charset ?a)          ; => ascii
(char-charset ?❤)          ; => unicode (或具体子集)
(encode-char ?中 'iso-2022-jp)  ; => 编码到具体 charset
```

这些函数让你**程序化**处理字符——你可以写"扫描 buffer 找所有非 ASCII 字符并列表"的脚本。

---

## 9. 创造性例子

### 9.1 同时编辑中英日韩的笔记

一个 buffer 里有:

```
中文测试 (Chinese)
English text
日本語テスト (Japanese)
한국어 테스트 (Korean)
```

这是 Emacs 的常态——所有字符平等,显示和编辑都正常。配好 fontset 后,每种语言用对应字体,行高对齐。

### 9.2 提取所有 CJK 字符

写一个 Elisp 函数,扫描 buffer 提取所有 CJK:

```elisp
(defun my/buffer-cjk-chars ()
  "返回当前 buffer 的所有 CJK 字符 (去重)。"
  (interactive)
  (let ((chars (make-hash-table)))
    (save-excursion
      (goto-char (point-min))
      (while (not (eobp))
        (let ((c (char-after)))
          (when (and (>= c #x4E00) (<= c #x9FFF))
            (puthash c t chars)))
        (forward-char)))
    (let ((result ""))
      (maphash (lambda (k _v) (setq result (concat result (char-to-string k)))) chars)
      (message "%s" result)
      result)))
```

对统计字符使用、做 NLP 数据清洗有用。

### 9.3 查询 emoji 的 codepoint

`M-x insert-char` 或 `C-x 8 RET`,然后输 emoji 名字 (例如 "rocket")——Emacs 显示候选 (🚀, U+1F680)。或反向: 把光标放在 emoji 上,`C-u C-x =` 看 codepoint 和 name。

这对**编程里需要写 emoji** 有用——例如你写脚本输出 emoji,需要知道 codepoint 怎么写在源码里 (Python: `"\U0001F680"`, JavaScript: `"🚀"`)。

### 9.4 完美 CJK 字体

终极字体配置:

```elisp
(set-face-attribute 'default nil
                    :family "JetBrains Mono"
                    :height 140)

;; CJK 用思源黑体,完美对齐
(set-fontset-font t 'han      "Source Han Sans CN")
(set-fontset-font t 'kana     "Source Han Sans JP")
(set-fontset-font t 'hangul   "Source Han Sans KR")

;; Emoji 用 Apple Color Emoji (macOS) 或 Noto Color Emoji (Linux)
(set-fontset-font t '(#x1F300 . #x1FAFF) "Apple Color Emoji")

;; 全角符号
(set-fontset-font t 'cjk-misc "Source Han Sans CN")

;; 缩放 (CJK 字体看起来通常偏大,缩到 0.95)
(setq face-font-rescale-alist
      '((".*Source Han.*" . 0.95)
        (".*PingFang.*" . 1.0)
        (".*Apple Color Emoji.*" . 0.9)))
```

这是"业界最佳实践"——所有 CJK 用思源黑体 (Google/Adobe 联合开发的开源字体),emoji 用系统字体,缩放微调。配好之后,你的 Emacs 多语言显示比 VSCode 还漂亮。

### 9.5 古籍 ASCII art (画图)

ASCII art 在 CJK 环境下"被破坏"——因为 CJK 字符是"全角"(占 2 列),ASCII 是"半角"。一段用 ASCII 画的图,如果某些位置被 CJK 替换,就错位。

解决: 用 `M-x picture-mode`——这个 mode 强制所有字符按列算,无视 CJK 全角。或用 `quail-fullwidth` 输入法把 ASCII 转成全角版本。

### 9.6 区分 Unicode 同形字符

有些 Unicode 字符**视觉相同但 codepoint 不同**——例如:
- ASCII `A` (U+0041)
- 全角 `A` (U+FF21)
- 数学斜体 `𝐴` (U+1D434)

视觉上都像 A,但编码完全不同。这导致"看起来一样但不相等"——搜索失败、文件名碰撞。

`C-u C-x =` 是唯一可靠的诊断——告诉你这个"A"到底是哪个 codepoint。安全敏感场景 (密码、文件名) 这是真问题。

---

## 10. 陷阱

### 10.1 CJK 字体不对齐

最常见的痛点——中文比英文高半行、宽度不一致、行间距跳跃。原因: 英文字体和 CJK 字体的 metrics (字号、ascent、descent) 不同。

解决: 用 `face-font-rescale-alist` 调 scale,用思源黑体 (设计为对齐西文 metrics 的字体)。具体值需要试验——0.9, 0.95, 1.0 之间通常合适。

### 10.2 终端乱码

`emacs -nw` 打开 UTF-8 文件,中文显示成 `\344\270\255` 或 `???`。原因: 终端 locale 不是 UTF-8。

检查:

```bash
$ echo $LANG
$ locale
```

如果不是 `.UTF-8` 后缀,改 `.bashrc` / `.zshrc`:

```bash
export LANG=en_US.UTF-8
```

重启终端 (或 `source ~/.bashrc`),再用 Emacs。

### 10.3 编码丢失

打开 GB2312 文件,用 UTF-8 解码——看到的可能是 `????` 或乱码。如果你**直接保存**,这些字符可能丢失 (变成 `?`)。

预防: 配置 `prefer-coding-system`。如果已经发生,马上 `C-/` (undo) 撤销修改,然后 `M-x revert-buffer-with-coding-system RET gb2312 RET` 重新解码——只要你**没保存**,原始数据还在文件里。

### 10.4 Quail 输入法缺词

Emacs 的 `chinese-py` 输入法词组库小——你输入 "shijie",候选可能没有 "世界"。这在中文用户看来"难用"。解决方案:

- 用系统 IME (推荐): fcitx / ibus / macOS pinyin——这些有云输入、智能联想
- 装 Quail 扩展词库: 社区有 `pyim` 等增强包,提供更全的词组

99% 的中文 Emacs 用户用系统 IME——Emacs 内置输入法只在 SSH 等无 IME 场景用。

### 10.5 macOS 上 IME 光标位置错位

macOS GUI Emacs 用系统中文输入法时,候选框常常出现在屏幕角落而不是光标处——这是 Emacs 的 IME 集成 bug。

缓解:

```elisp
(setq use-posframe-tooltip t)  ; 使用 posframe 显示
```

或换 emacs-mac ( railwaycat 的 emacs-mac-port )——它的 IME 集成比官方 emacs 更好。或干脆用 `chinese-py` 输入法避免这个问题。

### 10.6 UTF-8 with BOM

Windows 程序 (尤其 Microsoft Office) 常用 **UTF-8 with BOM**——文件开头有 3 个字节 `\xEF\xBB\xBF`,作为"我是 UTF-8"的标记。Emacs 默认读 UTF-8 不期望 BOM——可能把这 3 字节识别为乱码。

解决: 用 `utf-8-with-signature` 编码:

```elisp
(setq buffer-file-coding-system 'utf-8-with-signature)  ; 当前 buffer
```

或读取时 `M-x revert-buffer-with-coding-system RET utf-8-with-signature RET`。

### 10.7 全角/半角空格混淆

中文文档常混用**全角空格** (U+3000) 和半角空格 (U+0020)——它们视觉上都是空白,但代码里完全不同。代码缩进如果是全角空格,Python 会报错。

诊断: `C-u C-x =` 看 codepoint。修复: `M-x query-replace RET C-q 0 0 0 0 RET SPC RET` (把全角空格 `C-q 3000` 替换为半角空格)。

---

## 11. 练习

1. 打开一个 UTF-8 文件,用 `C-h C RET` 看当前编码
2. 用 `C-u C-x =` 看光标字符的 codepoint 和 charset
3. 用 `C-x 8 RET` 输入一个 emoji (例如 🎉)
4. 用 `C-x 8 RET` 输入一个数学符号 (例如 ∑ 或 ∞)
5. 用 `C-\` 切换到 `chinese-py` 输入法,输入 "你好"
6. 配置 `(prefer-coding-system 'utf-8)`,打开 init.el 加这条
7. 找一个 GB2312 编码的老文件,用 `M-x revert-buffer-with-coding-system RET gb2312 RET` 重新解码
8. 用 `set-fontset-font` 配置 CJK 字体 (思源黑体或 Noto CJK)
9. 在 init.el 加 `face-font-rescale-alist`,微调 CJK 字体 scale
10. 写一个 Elisp 函数,提取 buffer 里所有 CJK 字符
11. 写一个 Elisp 函数,把所有全角空格 (U+3000) 替换为半角 (U+0020)
12. 用 `M-x describe-input-method RET chinese-py RET` 看拼音输入法的说明
13. 配置 Emacs 用全角标点 (`chinese-punct`)
14. 写一个 Elisp 函数,统计当前 buffer 的字符数、字节数、CJK 比例
15. 用 `insert-char` 输入你的名字 (如果是中文,看候选)
16. 打开一个阿拉伯文 (RTL) 网页或文件,看 Emacs 的双向显示

---

## 12. 自测

1. Mule 是什么的缩写?
2. Emacs 内部用什么表示字符?UTF-8 字节吗?
3. `C-\` 干啥?
4. `C-u C-x =` 显示什么?
5. 怎么改文件编码保存?
6. CJK 字体不对齐怎么解决?
7. 全角空格和半角空格 codepoint 区别?

**答案**:
> 1. MULtilingual Enhancement
> 2. Unicode codepoint (整数),不是字节。读文件时解码,保存时编码
> 3. toggle-input-method (开/关输入法)
> 4. 光标字符的完整信息 (codepoint, charset, font, face, ...)
> 5. `C-x RET f utf-8-unix RET C-x C-s`
> 6. 用 set-fontset-font 配 charset-specific 字体 + face-font-rescale-alist 调 scale
> 7. 全角 = U+3000,半角 = U+0020

---

## 13. 下一步

读完这个文件,你已经理解 Emacs 的国际化底层——Mule 系统。

进入 Module 5 的其他专题或进入 Module 6。
