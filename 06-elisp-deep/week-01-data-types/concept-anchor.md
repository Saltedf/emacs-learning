# Concept Anchor: Editor Manual (Week 1)

> 这周你的"数据类型深入"对应编辑器的: 文本/字符/编码

Emacs 区别于其他编辑器的一个核心设计: **buffer 内部用 Unicode codepoint (整数) 表示字符**,而不是字节。这意味着你在 Elisp 里看到的 "abc" 是 3 个 codepoint 的 sequence,而不是 3 个字节。当你存到文件时,Emacs 用你指定的编码 (UTF-8、UTF-16、Latin-1 等) 转换。

这是巨大的简化——Lisp 代码不需要关心编码细节,只关心字符。底层编码只在"读写文件"、"网络传输"、"和外部系统交互"时才出现。

---

## 1. text.texi 的"字符"概念

### 1.1 字符 vs 字节

Emacs 内部用 Unicode:
- 一个 char 是一个 codepoint (整数)
- 1 byte = 8 bits
- 1 char 可能 1-4 bytes (UTF-8)

这个分离让你写 Elisp 时不用操心 ASCII vs 非 ASCII——`(length "hello")` 是 5,`(length "你好")` 也是 2,API 一致。对比其他语言: C 的 `char` 是 byte,处理 Unicode 要用 `wchar_t` 或外部库;Python 区分 `bytes` 和 `str`,你必须显式 decode/encode。Emacs 把这一切藏在内部。

```elisp
(char-to-string ?A)          ; → "A"
(char-to-string ?中)         ; → "中"
(char-to-string 20013)       ; → "中" (Unicode U+4E2D)
```

`?中` 是 character literal,直接是 codepoint 20013。`(char-to-string 20013)` 把 codepoint 转 1-char string。这两行本质相同。

### 1.2 编码

```elisp
(encode-coding-string "中" 'utf-8)
;; → 字节序列 (在 string 里)

(decode-coding-string STRING 'utf-8)
```

`encode-coding-string` 把 multibyte string 转成 unibyte string (字节序列)。这在网络传输、文件 IO 时必要——大多数协议假设字节流。

文件编码:
```elisp
(buffer-file-coding-system)   ; 当前 buffer 用什么编码
(set-buffer-file-coding-system 'utf-8-unix)
```

`utf-8-unix` 是复合 coding system——UTF-8 编码 + Unix 换行 (LF)。Windows 上可能用 `utf-8-dos` (CRLF)。如果你跨平台编辑文件,coding system 不匹配会导致 git 显示"整个文件都改了" (因为换行符变)。

### 1.3 字符分类

syntax table 决定每个字符的语法类别——是 word、whitespace、punctuation、open bracket 等。这是 Emacs 的"理解代码"基础。

```elisp
(char-syntax ?a)              ; → 'w' (word)
(char-syntax ?\s)             ; → ' ' (whitespace)
(char-syntax ?.)              ; → '.' (punctuation)
(char-syntax ?\()             ; → '(' (open)
```

`(char-syntax ?a)` 返回 character code (不是 symbol)——`'w'` 是 119 (w 的 codepoint),`' '` 是 32 (空格)。这些 syntax class 让 `forward-word`、`skip-syntax-forward` 等"按类别移动"工作。

(Module 6 W7 详细)

---

## 2. buffer-string 的底层

### 2.1 buffer 是 mutable string

buffer 内部和 string 共享数据模型——都是 mutable array of codepoint。但 buffer 多了: point、mark、narrowing、text properties、overlays 等"编辑器状态"。

```elisp
(buffer-substring START END)  ; 取一段
(buffer-substring-no-properties ...)  ; 无 text props
(buffer-string)               ; 全部
(set-buffer-multibyte t)      ; 多字节 mode
```

底层是 **gap buffer**: 字符串中间有 gap,允许高效插入。

```
[abc | def]
    gap
```

这个图展示了 gap buffer。`|` 是 gap——一段未使用内存,在哪里就让哪里"插入便宜"。point 在哪,gap 就在哪 (移动 point 时,内存里把 gap 移过去)。

插入: gap 移到 point,写新字符,gap 缩小。
删除: gap 扩大 (字符移到 gap 区域,但内存不释放)。

为什么这个设计? 1985 年的硬件上,如果每次插入都 shift 后面所有字符 (像 Python list.insert),编辑大文件会卡。Gap buffer 让连续插入是 O(1)。代价是 jump 后的第一次插入慢 (要移 gap),但 jump 不频繁。

VS Code 后期也加了类似优化 (叫 "piece table"),但 Emacs 从一开始就用 gap——这就是为什么 Emacs 编辑 100MB 文件也不卡。

### 2.2 point 是 integer

```elisp
(point)                       ; 当前位置 (1-based)
(point-min)
(point-max)
(char-after)                  ; 当前 char
(char-before)
```

point 是 1-based integer——buffer 第一个字符 position 1。`(point-min)` 通常 1,但如果 buffer 被 narrow (Module 4 学过),`point-min` 是 narrowed 范围的起点。

`(char-after)` 返回当前位置的字符 codepoint (integer),`(char-before)` 前一个。

### 2.3 position 操作

```elisp
(goto-char POS)
(forward-char N)
(forward-word N)
(skip-chars-forward " \t\n")
(skip-syntax-forward "w")
```

`goto-char` 直接跳,`forward-char` 相对移动。`skip-chars-forward` 跳过指定字符集合——`" \t\n"` 是空格、tab、换行。`skip-syntax-forward` 跳过指定 syntax class——`"w"` 是 word 字符。

这两个 skip 函数是写 mode 的核心——比如 indent 时跳过前导空白: `(skip-chars-forward " \t")`。

---

## 3. 数据持久化

### 3.1 序列化 list

Elisp 数据 (list、vector、hash) 到文件——用 Lisp 语法。这是 Elisp 的"原生存储格式"。

```elisp
(with-temp-file "data.el"
  (let ((print-length nil)
        (print-level nil))
    (pp my-data (current-buffer))))

(with-temp-buffer
  (insert-file-contents "data.el")
  (read (current-buffer)))
```

`print-length` 和 `print-level` 默认有限制 (防止巨大 list 卡住),设 `nil` 表示无限。`pp` 是 pretty-printer,让数据格式化对齐。

这个 pattern 是 Elisp 配置文件的基础——大多数 Emacs 包的"会话状态"、"history"、"cache"都用这个。对比其他编辑器: VS Code 用 JSON,Vim 用 vimscript literal,但 Elisp 用自己——好处是类型丰富 (可以存 function 引用、symbol),坏处是不通用 (外部工具读不懂)。

### 3.2 序列化 hash-table

hash-table **不能直接 read**——它没有 Lisp 字面量语法 (像 `#s(hash-table ...)` 是 Emacs 25+ 才有,且复杂)。常见做法: 转 alist 存,读回时重建 hash。

```elisp
;; hash-table 不能直接 read
;; 转为 alist
(let ((alist nil))
  (maphash (lambda (k v) (push (cons k v) alist)) my-hash)
  (pp alist (current-buffer)))
```

这个 limitation 是 Emacs 设计权衡——hash 内部用 hash function,不同 Emacs 版本可能不同,所以无法稳定序列化。alist 是 list,序列化稳定。

### 3.3 序列化对象 (cl-defstruct)

```elisp
(cl-defstruct person name age)
(setq p (make-person :name "Alice" :age 30))
(print p)                     ; → #s(person "Alice" 30)
(read "(person \"Alice\" 30)") ; 但读回不会自动转 struct
```

`#s(...)` 是 record/struct 的字面量语法——`(print p)` 输出 `#s(person "Alice" 30)`。但 `read` 读回时,它只是普通 vector (带 person 作为第 0 个元素),不会自动重建为 person 对象。

完整方案用 `read` + 自己 reconstruct,或用 `sergi` 包。

---

## 4. 字符串 = 字符 array

string 在 Elisp 里**就是字符 array**——你可以用 `aref`、`aset`、`length` 操作它,和 vector 几乎一样。

```elisp
(aref "abc" 0)               ; → 97 (a)
(aset "abc" 0 ?A)            ; → "Abc"
(length "abc")               ; → 3
(length "中文")               ; → 2 (Emacs 内部 Unicode)
(string-bytes "中文")         ; → 6 (UTF-8 编码)
```

`(length "中文")` 返回 2 (字符数),`(string-bytes "中文")` 返回 6 (UTF-8 字节数)。这两者关系: 中 = 3 字节 UTF-8,文 = 3 字节 UTF-8,共 6 字节。但 Elisp 里它们是 2 个 codepoint。

如果你处理 ASCII,`length` 和 `string-bytes` 总相等。处理中文/日文/emoji 才有差别——这是为什么很多老代码 (假设 1 char = 1 byte) 在 Unicode 上出 bug。

### 4.1 byte vs char

```elisp
(string-to-list "abc")       ; → (97 98 99)
(string-to-vector "abc")     ; → [97 98 99]
(append "abc" nil)           ; → (97 98 99) (string → list)
(concat '(97 98 99))         ; → "abc" (list → string)
```

string 和 list of char 是双向可转换的——`(string-to-list "abc")` 给 `(97 98 99)`,`(concat '(97 98 99))` 给回 `"abc"`。这让 string 处理极其灵活——你可以用所有 list 函数 (map、filter、reduce) 处理字符串。

### 4.2 multibyte vs unibyte

```elisp
(string-make-unibyte "abc")  ; → "abc" (1 byte per char)
(string-make-multibyte "abc") ; → "abc" (可多字节)
```

multibyte string: 每个 char 是 codepoint (1-4 字节内部表示)。
unibyte string: 每个 char 严格 1 字节 (0-255)。

默认 Emacs buffer 是 multibyte,允许 Unicode。有些低级操作需要 unibyte (如二进制数据、网络协议头)。一般你不需要直接操作——但理解这个区别能解释某些诡异行为。

---

## 5. 自测

1. `?中` 的 codepoint 是多少?
2. buffer 是 mutable string,Lisp 怎么取它内容?
3. `(length "中文")` 和 `(string-bytes "中文")` 区别?
4. 怎么序列化一个 list 到文件?
5. 怎么反序列化?

**答案**:
> 1. 20013
> 2. buffer-substring 或 buffer-string
> 3. length 返回 char 数 (2);string-bytes 返回 UTF-8 byte 数 (6)
> 4. pp 到 file
> 5. read 从 file

---

## 6. 下一步

进入 `exercises.md`。
