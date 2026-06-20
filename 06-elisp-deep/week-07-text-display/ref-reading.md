# Week 7: Text + Display + Search + Syntax

> **Ref 章节**: text, nonascii, searching, syntax, parsing, peg, display
> **目标**: 操作 buffer 文本、显示、搜索

这周是"视觉" 和 "搜索" 的深度——text property、overlay、font-lock 决定 buffer 怎么显示;syntax table、search 决定代码怎么被解析。理解这些让你能写真正的 major mode——不只是 indent,还有高亮、parse、跳转。

Emacs 区别于其他编辑器的核心: **文本属性是数据,不是 UI 状态**。语法高亮、超链接、字段标记都是 text property——它们和字符一起保存,可以 search、filter、transform。这让 Emacs 的"富文本" 不需要专门格式 (像 RTF)。

---

## 1. Text Properties (text.texi)

### 1.1 什么是 text property

text property 是 Emacs 的"持久元数据"机制——每个字符可以附带属性 (face、keymap、read-only、help-echo 等),这些属性**写入 buffer**,和字符一起保存。这让 Emacs 可以做"富文本"而不需要专门格式。

每个字符可以**附带属性**:
- face (颜色)
- category
- keymap
- mouse-face
- help-echo
- invisible
- intangible
- read-only
- field
- display
- ...

核心操作:

```elisp
(put-text-property START END PROP VALUE &optional OBJECT)
(get-text-property POS PROP &optional OBJECT)
(looking-at-p REGEXP)          ; (复习)
(text-properties-at POS &optional OBJECT)
(set-text-properties START END PROPS &optional OBJECT)
(remove-text-properties START END LIST &optional OBJECT)
(add-text-properties START END PROPS &optional OBJECT)
```

`put-text-property` 给 [START, END) 范围的字符加属性。`get-text-property` 读 POS 处某属性的值。注意属性是"附在字符上"的——如果你 delete-region 把这段文本删了,属性也消失。这和 overlay 不同 (overlay 是 buffer 上面的"图层",不附字符)。

### 1.2 例子

```elisp
(with-temp-buffer
  (insert "hello")
  (put-text-property 1 4 'face 'bold)        ; 前 3 字符 bold
  (buffer-string))
;; → "hello" (前 3 个 bold 显示)
```

```elisp
(put-text-property 1 (point-max) 'read-only t)
;; 整个 buffer 只读

(put-text-property 1 5 'mouse-face 'highlight)
;; 鼠标悬停高亮

(put-text-property 1 5 'help-echo "Click to open")
;; hover 显示
```

`'read-only` 是 text property 极强的应用——你可以让 buffer 的某段只读,其他段可编辑。Magit、Org 都用这个防止用户误改生成的 section。

`'help-echo` 让鼠标悬停显示 tooltip——给用户即时提示。

`'keymap` 给特定区域专属 keymap——让 buffer 的某段响应不同按键。这让你做"button"——点击跑代码。

### 1.3 text property = persistent

text property 写到 buffer 内部,和字符一起保存。
适合**长期**修改。

区别 overlay (临时): text property 跟字符走,你复制 (kill-ring-save) 文本时属性跟着。overlay 是 buffer 上的图层,你复制文本不会复制 overlay。

### 1.4 next-property-change

```elisp
(next-property-change POS &optional OBJECT LIMIT)
(previous-property-change POS &optional OBJECT LIMIT)
(next-single-property-change POS PROP &optional OBJECT LIMIT)
(text-property-any START END PROP VALUE &optional OBJECT)
(text-property-not-all START END PROP VALUE &optional OBJECT)
```

这些函数让你"按属性边界遍历"——找下一个属性变化的位置。font-lock、imenu 等用这个高效扫描 buffer。

`text-property-any` 检查 [START, END) 范围内是否有 PROP=VALUE——用于"找有这个属性的区域"。

### 1.5 font-lock 用 text property

```elisp
(font-lock-add-keywords nil
  '((("\\<\\(TODO\\|FIXME\\)\\>" 1 'font-lock-warning-face t))))
```

font-lock 把 face 写到 text property。这是为什么 `(get-text-property (point) 'face)` 在 keyword 处返回 `font-lock-keyword-face`——它是字符的属性,不是 buffer 状态。

Font-lock 用 text property 实现——它把 face 写到字符上,字符显示时 redisplay 引擎读 face 决定颜色。所以"语法高亮"在 Emacs 里不是 UI 状态,是**文本属性**——这意味着你可以 search、filter、transform 它,因为它就是数据。

---

## 2. Overlays (display.texi 部分)

### 2.1 什么是 overlay

**Overlay** = buffer 上的"图层"。
- 临时附加属性
- 不写到 buffer 内容
- 范围动态 (字符增删时调整)

overlay 和 text property 的区别: overlay 是 buffer 上"叠加"的对象,它指向一段范围,范围里的字符显示时叠加 overlay 属性。但 overlay 不附字符——如果你 delete-region,overlay 范围缩小;如果你在 overlay 范围外编辑,overlay 不变。

```elisp
(make-overlay START END &optional BUFFER FRONT-ADVANCE REAR-ADVANCE)
(overlay-put OVERLAY PROP VALUE)
(overlay-get OVERLAY PROP)
(move-overlay OVERLAY START END &optional BUFFER)
(delete-overlay OVERLAY)
(overlays-at POS &optional SORTED)
(overlays-in BEG END)
(next-overlay-change POS)
```

### 2.2 例子

```elisp
(let ((ov (make-overlay 1 5)))
  (overlay-put ov 'face 'highlight)
  (overlay-put ov 'help-echo "Hello")
  (overlay-put ov 'priority 100))
```

`make-overlay` 创建 overlay 范围 [1, 5)。`overlay-put` 给 overlay 加属性——属性和 text property 同名 (face、help-echo 等),但属于 overlay 而非字符。

### 2.3 text property vs overlay

| | text property | overlay |
|---|---|---|
| 持久 | ✅ (写入 buffer) | ❌ (临时) |
| 性能 | 高 (直接在 char) | 中 (单独 list) |
| 范围 | 静态 (要重新 set) | 动态 (自动调整) |
| 用途 | font-lock, 长期属性 | 临时高亮、tooltip、图片 |

经验:
- 临时高亮 → overlay
- 长期 face → text property (via font-lock)

为什么有两套? 历史原因 + 不同用例。text property 是 1985 设计 (字符是属性容器);overlay 是 1990s 加 (更灵活的"图层"概念)。font-lock 用 text property 因为它持久;isearch 高亮用 overlay 因为它临时。

### 2.4 实战: 临时高亮

```elisp
(defvar my-highlight-overlay nil)

(defun my-flash-region (start end)
  "Flash region for 0.5 seconds."
  (when my-highlight-overlay
    (delete-overlay my-highlight-overlay))
  (setq my-highlight-overlay (make-overlay start end))
  (overlay-put my-highlight-overlay 'face '(:background "yellow"))
  (run-with-timer 0.5 nil
                  (lambda ()
                    (when my-highlight-overlay
                      (delete-overlay my-highlight-overlay)
                      (setq my-highlight-overlay nil)))))
```

这是"闪烁高亮"的标准模式——创建 overlay,显示 0.5 秒,然后删除。`run-with-timer` 延时跑 cleanup。

---

## 3. Display (display.texi)

### 3.1 Faces

face 是 Emacs 的"样式"——颜色、字体、大小、下划线等。

```elisp
(defface my-face
  '((t :foreground "red" :weight bold))
  "My face.")
```

`defface` 定义 face——`((t ...))` 的 `t` 是 "默认 display spec",你可以加特定条件 (如 `((class color) (background dark)) ...)`)。

```elisp
;; 已有 face:
'font-lock-keyword-face
'font-lock-string-face
'font-lock-comment-face
'font-lock-function-name-face
'font-lock-variable-name-face
'font-lock-type-face
'font-lock-constant-face
'font-lock-builtin-face
'font-lock-warning-face
```

这些是 Emacs 内置 face——font-lock 默认用它们。你的 mode 可以直接用,保证一致性。

### 3.2 face 属性

```elisp
:family          ; 字体名
:height          ; 字号 (1/10 pt,如 140 = 14pt)
:weight          ; normal/bold/light/...
:slant           ; normal/italic/oblique
:underline       ; t / nil / color / (:color "red" :style wave)
:overline        ; 同上
:strike-through
:box             ; t / (:line-width N :color "red")
:inverse-video
:foreground
:background
:stipple
:inherit         ; 继承另一 face
:width           ; normal/condensed/...
```

`defface` 或 `set-face-attribute` 用这些 keyword 设置。`:height 140` 是 14pt (1/10 pt 为单位)。`:inherit` 让 face 继承另一个——`(:inherit font-lock-keyword-face :weight bold)` 创建一个基于 keyword 的 bold 版本。

### 3.3 set-face-attribute

```elisp
(set-face-attribute 'default nil
                    :family "JetBrains Mono"
                    :height 140)

(set-face-attribute 'font-lock-keyword-face nil
                    :foreground "pink"
                    :weight 'bold)
```

`set-face-attribute` 运行时改 face——`nil` 第二参数是 frame (nil = 所有 frame)。

### 3.4 fringe

```elisp
(fringe-mode '(8 . 8))

(set-fringe-bitmap-face 'left-triangle 'error)
```

fringe 是 window 左右的边带——显示 marker、wrap indicator 等。`fringe-mode` 设宽度,`set-fringe-bitmap-face` 改 bitmap 颜色。

### 3.5 display property

text property `display` 可让字符显示不同:

```elisp
(put-text-property 1 2 'display
                   '(image :type png :file "~/pic.png"))
;; 显示图片而不是字符

(put-text-property 1 2 'display
                   '(height 2.0))
;; 高度 2 倍

(put-text-property 1 2 'display
                   '(space-width 2.0))
;; 宽度 2 倍

(put-text-property 1 5 'display "(hidden)")
;; 显示 "(hidden)" 而不是 4 字符
```

`display` property 让字符显示**任意东西**——图片、不同尺寸、不同文本。这让 Emacs 能在 buffer 里嵌图片 (如 emoji、图像缩略图)、做"折叠"功能 (org-mode 的折叠用这个)。

---

## 4. Searching (searching.texi)

### 4.1 search

```elisp
(search-forward STRING &optional BOUND NOERROR COUNT)
(search-backward STRING &optional BOUND NOERROR COUNT)
(search-forward REGEXP)
(search-backward REGEXP)
(re-search-forward REGEXP &optional BOUND NOERROR COUNT)
(re-search-backward REGEXP &optional BOUND NOERROR COUNT)

(word-search-forward "hello world")    ; word search
(word-search-backward)
```

`search-forward` 字面 search (无 regex)。`re-search-forward` regex search。`word-search-forward` "word match"——忽略空白和标点,适合搜代码 (忽略 reformatting)。

### 4.2 BOUND / NOERROR

- BOUND: 位置限制
- NOERROR:
  - nil: 报错 if not found
  - t: 返回 nil if not found
  - `lambda`: quiet (no error)

```elisp
(re-search-forward "[0-9]+" nil t)   ; 找数字,找不到返回 nil
(re-search-forward "[0-9]+" 1000)    ; 在前 1000 字符内找
```

`NOERROR` 是关键——大多数场景你希望"找不到不报错" (用 t)。但偶尔你需要报错 (用 nil,显式处理 search-failed)。

### 4.3 match data

search 成功后,match data 存匹配信息——你可以取匹配的文本、子组。

```elisp
(match-beginning 0)             ; 最后匹配起点
(match-end 0)                   ; 终点
(match-string 0)                ; 匹配文本
(match-string-no-properties 0)

;; 分组:
(re-search-forward "\\([a-z]+\\)\\.\\([0-9]+\\)")
(match-string 1)               ; 第 1 组
(match-string 2)               ; 第 2 组

(set-match-data OLD-DATA)       ; 恢复
(save-match-data BODY...)       ; 保存,跑 body,恢复
```

`match-string 0` 是整个匹配,`match-string 1/2/...` 是 regex 的 capture group。

`save-match-data` 重要——如果你的代码用 match data,然后调别的函数 (可能也用 match data),match data 会被破坏。`save-match-data` 保存恢复。

### 4.4 replace

```elisp
(replace-match NEWTEXT &optional FIXEDCASE LITERAL STRING SUBEXP)
(query-replace FROM TO)         ; 交互
(perform-replace FROM TO FLAG STRING-RE LITERAL)  ; 内部

(replace-regexp-in-string REGEXP REP STRING &optional FIXEDCASE LITERAL SUBEXP START)
```

例子:
```elisp
(let ((case-fold-search nil))
  (while (re-search-forward "\\bfoo\\b" nil t)
    (replace-match "bar")))
```

`(let ((case-fold-search nil)) ...)` 让 search 大小写敏感——默认 Emacs 忽略大小写,这常导致意外替换。

### 4.5 thingatpt

thingatpt 让你"取 point 处的东西"——word、symbol、line、url 等。

```elisp
(thing-at-point 'word)
(thing-at-point 'symbol)
(thing-at-point 'line)
(thing-at-point 'sentence)
(thing-at-point 'paragraph)
(thing-at-point 'sexp)
(thing-at-point 'list)
(thing-at-point 'url)
(thing-at-point 'email)
(thing-at-point 'filename)
(thing-at-point 'number)
(thing-at-point 'defun)

(bounds-of-thing-at-point 'word)
(forward-thing 'word 1)
```

`thing-at-point` 是"智能取词"的标准 API——你不需要自己写 "找 word 边界" 代码,直接 `(thing-at-point 'word)`。

`(bounds-of-thing-at-point 'word)` 返回 (beg . end)——你可以 kill-region 那段。

---

## 5. Syntax Tables (syntax.texi)

### 5.1 什么是 syntax table

syntax table 决定每个字符的语法类别——是 word、whitespace、punctuation、open bracket 等。这是 Emacs 的"理解代码"基础——indent、movement、parse 都依赖它。

每个 buffer 有 syntax table:
- 决定字符类别 (word, whitespace, punctuation, ...)
- 决定 bracket 匹配
- 决定 comment 起止

### 5.2 char-syntax

```elisp
(char-syntax ?a)                ; → 'w' (word)
(char-syntax ?\s)               ; → ' ' (whitespace)
(char-syntax ?.)                ; → '.' (punctuation)
(char-syntax ?\()               ; → '(' (open)
(char-syntax ?\))               ; → ')' (close)
(char-syntax ?\n)               ; → '>' (comment end in some modes)
```

syntax class 用字符代码表示:
- `'w'` word
- `' '` whitespace
- `'.'` punctuation
- `'('`/`')'` open/close bracket
- `'"'` string quote
- `'<'`/`'>'` comment start/end
- `'\\'` escape

### 5.3 modify-syntax-entry

```elisp
(modify-syntax-entry ?_ "w")    ; _ 算 word
(modify-syntax-entry ?\" "\"")  ; " 是 string delimiter
(modify-syntax-entry ?\\ "\\")  ; \ 是 escape
(modify-syntax-entry ?\; "<")   ; 注释开始
(modify-syntax-entry ?\n ">")   ; 注释结束
```

`modify-syntax-entry` 改某字符的 syntax class。`"w"` 是 word class——`_` 算 word 一部分 (Python 标识符可以 `_foo`)。

C 风格注释 `/* */` 用复杂的 syntax entry: `(modify-syntax-entry ?/ ". 124b" st)`——`. ` 是 punctuation,`1`/`2`/`4` 是 comment 标志,`b` 是 "b style" (C-like)。

### 5.4 syntax-table

```elisp
(make-syntax-table)
(with-syntax-table TABLE BODY...)
(setq syntax-table (make-syntax-table))
```

`make-syntax-table` 创建标准 syntax table。`with-syntax-table` 临时用某 table 跑 body,恢复——用于"用其他 mode 的 syntax 跑代码"。

### 5.5 parse-partial

```elisp
(parse-partial-sexp FROM TO &optional TARGETDEPTH STOPBEFORE OLDEPTH STATE)
```

深入分析: 从 FROM 到 TO,跟踪嵌套、字符串、注释等。

返回 state:
- depth (嵌套层数)
- contains-start (string/comment start)
- ...

`parse-partial-sexp` 是 Emacs 的"代码 parse" 引擎——它返回当前 parse state (是否在 string、comment、嵌套深度等)。indent、font-lock、movement 都用它。

### 5.6 scan-lists

```elisp
(scan-lists FROM COUNT DEPTH)
;; 移动 N 个 list
(scan-sexps FROM COUNT)
```

`scan-lists` 跳过 N 个 list——`forward-list`、`backward-list` 用它。

---

## 6. Parsing Program Source (parsing.texi)

### 6.1 treesit (tree-sitter)

(Module 5 提过)

```elisp
(treesit-parser-create 'python)
(treesit-node-at (point))
```

tree-sitter 是 AST 解析,比正则准。

 Emacs 29+ 内置 tree-sitter——比 font-lock 的 regex 方法准确得多。treesit 让 indent、navigation、highlight 都基于 AST,不再需要脆弱的 regex。

---

## 7. PEG (peg.texi)

### 7.1 第一性原理: 为什么需要 parser

正则表达式是 Emacs 早期文本处理的核心——但正则有**根本性局限**: 它无法表达"嵌套结构"。`re-search-forward` 永远只能匹配扁平模式,无法递归。考虑解析一个嵌套 JSON:

```json
{"a": {"b": [1, [2, 3]]}}
```

正则写不出"匹配一个 JSON value (它可以是 string、number、array of values、object of values)"——因为这需要**递归**。正则是 FSM (有限状态机),没有 stack。

解决方案有三层:
1. **Regex** —— 扁平模式,适合 token、line 提取。简单快但弱。
2. **PEG (Parsing Expression Grammar)** —— 规则递归,能表达嵌套。Lisp 写的,集成度高。
3. **Tree-sitter** —— 工业级 AST 解析,C 库。强但重,Emacs 29+ 内置。

PEG 是介于 regex 和 tree-sitter 之间的"中间力量"——比 regex 强 (能递归、有 actions),比 tree-sitter 轻 (纯 Lisp,无外部依赖,启动无开销)。Emacs 29 把它内置到 lispref,作为"DSL 解析"的标准答案。

### 7.2 PEG vs CFG vs regex

- **Regex** (正则): 表达力弱 (无递归),实现简单 (NFA),适合 token。
- **CFG** (Context-Free Grammar): BNF 风格,有 ambiguous (要 disambiguation 算法,如 yacc 的 LALR)。语言理论强,实现复杂。
- **PEG** (Parsing Expression Grammar): CFG 变种,**无歧义**——用"优先选择" (`/` 操作符,先匹配左边) 代替 CFG 的 ambiguous。LL(1)+backtracking,实现简单。

PEG 的核心思想: **每条规则是"按顺序尝试,第一个成功就用"**——这避免了 CFG 的歧义问题,代价是某些 CFG 用 PEG 写会慢 (因 backtracking)。但对大多数 DSL,PEG 完美。

PEG 的语法:
- `A B`: 序列 (A 后 B)
- `A / B`: 选择 (优先 A,失败试 B)
- `A*`: 零或多个
- `A+`: 一或多个
- `A?`: 可选
- `&A`: lookahead (匹配但不消费)
- `!A`: negative lookahead (不匹配才成功)
- `(expr)`: 分组

### 7.3 Emacs peg 包 API

Emacs 29+ 内置 `peg` 包,API 简洁:

```elisp
(require 'peg)

;; 主入口: peg-run
(peg-run PEG-MATCHER &optional FAILURE-FUNCTION SUCCESS-FUNCTION)

;; 定义规则用宏:
(peg &rest PEXS)                       ; 编译规则,返回 matcher
(define-peg-rule NAME ARGS &rest PEXS) ; 命名规则,可复用
(define-peg-ruleset NAME &rest RULES)  ; 一组规则
(with-peg-rules RULES &rest BODY)      ; 局部规则 scope
(peg-parse &rest PEXS)                 ; 编译并立刻在 point 跑
```

`(peg "number" "digit+")` 编译规则——`"number"` 是规则名,`"digit+"` 是 PEX (parsing expression)。返回一个 matcher 对象,可传给 `peg-run`。

`peg-run` 是核心入口——它在 point 处跑 matcher,成功返回结果 (默认是匹配的字符串),失败调 failure-function (默认报错)。

### 7.4 内置规则

peg 包预定义了常用规则,省得你重写:

```elisp
peg-digits        ; [0-9]+
peg-number        ; 数字 (含负号、小数点)
peg-string        ; "..." 含 escape
peg-list          ; (a b c)
peg-symbol        ; symbol
peg-word          ; word char+
peg-spacing       ; 空白
peg-eol           ; 行尾
peg-bol           ; 行首
peg-bof           ; buffer 首
peg-eof           ; buffer 尾
```

这些规则覆盖大多数 token 类型——你只需组合,不重新定义。

### 7.5 实战: 解析简单 URL

```elisp
(require 'peg)

(define-peg-rule my-url-scheme
  (seq "http" (opt "s") "://"))

(define-peg-rule my-url-host
  (+ (any "a-z0-9.-")))

(define-peg-rule my-url-port
  (seq ":" (+ (range "0" "9"))))

(define-peg-rule my-url-path
  (* (any "a-z0-9/._-")))

(defun my-parse-url (str)
  (with-temp-buffer
    (insert str)
    (goto-char (point-min))
    (peg-run (peg my-url
                  (list my-url-scheme my-url-host
                        (opt my-url-port) my-url-path)))))
```

PEG 用 `seq`、`opt`、`any`、`range` 等组合规则。这比正则可读,且能复用 (URL 规则可被 HTTP、SMTP 等多个 parser 共享)。

### 7.6 实战: 解析 JSON 子集

```elisp
(define-peg-ruleset my-json-rules
  (json-value (or json-object json-array json-string json-number
                  "true" "false" "null"))
  (json-object (seq "{" (* (seq json-string ":" json-value
                               (opt ","))) "}"))
  (json-array (seq "[" (* (seq json-value (opt ","))) "]"))
  (json-string (seq "\"" (* (or (not (or "\"" "\\")) (seq "\\" peg-any))) "\""))
  (json-number (seq (opt "-") peg-digits (opt (seq "." peg-digits)))))
```

这种递归规则是 regex 写不出的——`json-value` 可以是 `json-object`,而 `json-object` 又含 `json-value`。PEG 自然处理这种递归。

### 7.7 Parsing Actions

PEG 不只匹配,还能在匹配时执行 action:

```elisp
(define-peg-rule my-int
  (seq peg-digits
       `(action -- (string-to-number (match-string)))))
```

`(action ...)` 在规则匹配成功后跑——`(match-string)` 取匹配文本,你可以转 number、push 到 stack、构造 AST。这让 PEG 不只是 "recognizer",而是 "parser with semantics"。

### 7.8 创造性用法 (5+)

**1. 解析 Makefile 提取目标**: 写一个 peg 规则匹配 `target: deps`,提取所有 target,生成 imenu 列表。比 regex 可靠,因为 deps 可能含嵌套引用。

**2. 解析 C header 提取函数签名**: `peg` 能处理 `int (*func)(int, char*)` 这种函数指针——正则做不到。

**3. 解析 React JSX**: JSX 有嵌套标签 `<div><span>..</span></div>`,必须递归。PEG 写出 JSX parser,生成 AST,然后做 lint/refactor。

**4. 解析配置文件 (TOML/INI/YAML subset)**: 大部分配置是 key=value,但值可以是嵌套 (table、list)。PEG 处理这种结构化数据比 regex 准。

**5. 自定义 DSL**: 你写自己的 init DSL (如 org-capture 模板),用 PEG 解析,然后生成 Elisp。比手写状态机简单。

**6. 解析 SQL 子集提取表名**: SQL 是高度嵌套的——JOIN、子查询、CTE。PEG 解析 SELECT 语句,提取所有引用的表,做 schema 影响分析。

**7. 自动生成 indentation engine**: 解析每行的"语法结构",根据嵌套深度算 indent。比 regex 启发式准确。

### 7.9 PEG vs tree-sitter

| 维度 | PEG | tree-sitter |
|---|---|---|
| 实现 | 纯 Lisp | C 库 |
| 速度 | 中 (backtracking) | 快 (增量 parse) |
| 错误恢复 | 弱 (失败就停) | 强 (能继续) |
| 启动开销 | 0 (require 即用) | 需装 grammar |
| AST | 自己用 action 构造 | 自动生成 |
| 用途 | DSL、一次性 parse | 真实语言 (Python/JS/...) |

经验:
- **小 DSL、配置文件、自定义格式** → PEG。无需 grammar 安装,代码内联,即写即用。
- **真实编程语言 (Python、JS、C)** → tree-sitter。预写 grammar 准确,有错误恢复,indent/highlight 都基于它。

PEG 的杀手锏是**"无需外部依赖"**——你的 package 用 PEG 解析 DSL,用户装 package 即用,不需要装额外 grammar。tree-sitter 需要用户装 grammar,跨平台分发麻烦。

### 7.10 局限

PEG 是 backtracking parser——某些规则组合会**指数慢**。比如 `(a* a)* b` 匹配 `aaaaa`——每个 `a*` 失败时 backtracking,组合起来指数。这叫 "PEG packrat 缓存问题"——解决方案是 packrat memoization,但 Emacs peg 包不实现它。

对真实 DSL 这通常不是问题,但要避免写"病态"规则 (大量嵌套 `*` 和 ambiguous 选择)。

---

## 8. nonascii.texi (国际字符)

### 8.1 编码

```elisp
(buffer-file-coding-system)     ; 当前 buffer 编码
(set-buffer-file-coding-system 'utf-8-unix)  ; 改
(prefer-coding-system 'utf-8)

(encode-coding-string STRING CODING)
(decode-coding-string BYTES CODING)
```

Emacs 默认 UTF-8——但有些 legacy 文件用其他编码 (Latin-1、GBK)。`set-buffer-file-coding-system` 转换。

### 8.2 字符操作

```elisp
(char-charset ?中)              ; → chinese-cns
(encode-char ?中 'chinese-cns)
(decode-char 'chinese-cns 20013)
```

### 8.3 input method

```elisp
(current-input-method)
(set-input-method "chinese-py")
(toggle-input-method)
```

`set-input-method` 切换输入法——`chinese-py` 是拼音。

---

## 9. 实战: 写 font-lock

```elisp
(defvar mylang-font-lock-keywords
  '(;; Comment
    ("//.*" . font-lock-comment-face)
    ;; String
    ("\"[^\"]*\"" . font-lock-string-face)
    ;; Keyword
    ("\\<\\(if\\|else\\|while\\|for\\|return\\|function\\)\\>" . font-lock-keyword-face)
    ;; Type
    ("\\<\\([A-Z][a-zA-Z0-9_]*\\)\\>" . font-lock-type-face)
    ;; Function name (定义)
    ("^\\s-*function\\s-+\\([a-z_][a-zA-Z0-9_]*\\)" . (1 font-lock-function-name-face))
    ;; Variable
    ("\\<\\([a-z_][a-zA-Z0-9_]*\\)\\s-*=" . (1 font-lock-variable-name-face))))

(define-derived-mode mylang-mode prog-mode "Mylang"
  "Major mode for Mylang code."
  (setq-local font-lock-defaults '(mylang-font-lock-keywords)))
```

每个 entry 是 `(REGEXP . FACE)` 或 `(REGEXP . (SUBEXP FACE))`。后者用 SUBEXP 从 REGEXP capture group 取要高亮的部分。

`\\<` 和 `\\>` 是 word boundary——确保 `if` 不匹配 `iff`。`\\s-*` 是 syntax-based 匹配 (任何 whitespace)。

---

## 10. 自测

1. text property 和 overlay 区别?
2. `put-text-property` 怎么用?
3. syntax table 决定什么?
4. `re-search-forward` 的 NOERROR 参数?
5. font-lock 怎么定义?

**答案**:
> 1. property 持久,overlay 临时
> 2. (put-text-property START END PROP VALUE)
> 3. 字符类别、bracket、comment
> 4. nil 报错;t 返回 nil
> 5. font-lock-defaults 设置 keywords list

---

## 11. 下一步

进入 `concept-anchor.md` + `exercises.md`,然后 `../week-08-processes-package/`。
