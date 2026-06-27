# Fast Selection: 如何像 Emacs 极客一样瞬间选中

> 这个文件教你**不用 `C-SPC + 挪光标`** 的选区方式——这才是真正的"极客选法"

---

## 0. 为什么 C-SPC + 挪光标是反模式

新手都这样选:
1. `C-SPC` 设 mark
2. 各种 `C-f` `C-n` `M-f` 挪到终点
3. region 才选好

这是**反 Emacs 的**。Emacs 给了你 30+ 种"一键选好语义单位"的命令,完全不需要挪光标。

**真相**: 看视频里的 Emacs 极客选东西,你看到的不是"挪光标",是:
- 按 `M-h` → 整段亮
- 按 `C-M-h` → 整个函数亮
- 按 `C-=` (expand-region) → 一键从 word 扩到 sexp 再扩到 defun
- 选完立刻 `C-w` / `M-%` / `C-.` 操作

读完这个文件,你**几乎不会再按 `C-SPC`**。

---

## 1. 内置 mark 命令 (语义选)

Emacs 内置一堆 `mark-X` 命令——**按一个键直接选好"语义单位"**:

| 键位 | 命令 | 选什么 |
|---|---|---|
| `M-h` | `mark-paragraph` | 当前段 (空行分隔) |
| `C-M-h` | `mark-defun` | 当前函数/defun |
| `C-M-SPC` | `mark-sexp` | 当前 sexp (括号块/字符串/单词) |
| `M-@` | `mark-word` | 当前 word |
| `C-x h` | `mark-whole-buffer` | 整个 buffer |
| `C-x C-p` | `mark-page` | 当前 page (用 `\f` 分隔) |
| `C-M-@` | `mark-sexp` (变体) | 同 C-M-SPC |
| `mark-end-of-sentence` | | 到句尾 |

这些是**Emacs 第一性原理**——文本有结构 (词/句/段/函数/page),Emacs 知道结构,所以能"语义选"。

### 1.1 演示: mark-defun

打开一个 .py 文件,光标放在任意函数内部:

```
def calculate(x, y):
    """Add x and y."""
    return x + y     ← 光标在这行
```

按 `C-M-h`。整个 `def calculate ... ` 自动高亮。**0 秒选中一个函数**。

### 1.2 演示: mark-paragraph

文档里光标在某段中间,按 `M-h`——整段高亮。

### 1.3 演示: mark-sexp

光标在 `(foo bar baz)` 的开头或内部,按 `C-M-SPC`——整个 `(foo bar baz)` 高亮。

sexp 是"语义单元"——括号表达式、字符串、单个 symbol 都算 sexp。

---

## 2. 重复按 mark 键 = 扩大

**这是新手不知道的进阶用法**。

mark 命令支持"重复按 = 扩大"。例:
- 按 `M-h` 一次 → 选当前段
- 按 `M-h M-h` → 当前段 + 下一段
- 按 `M-h M-h M-h` → 再下一段

`C-M-h` 同理——重复按扩大函数边界 (当前函数 + 下一个函数)。

为什么这样设计? 因为很多场景"我要选 N 段",重复按比"设 mark + 跳到终点"快得多。

---

## 3. expand-region (极客最爱的杀手锏)

`expand-region` 是 Magnar Sveen (Emacs Rocks 作者) 2012 年写的包。**它就是你看到的"快速选"的真相**。

### 3.1 安装

```bash
M-x package-install RET expand-region RET
```

### 3.2 配置

Doom `config.el`:
```elisp
(map! :n "C-=" #'er/expand-region
      :n "C--" #'er/contract-region)
```

普通 Emacs init.el:
```elisp
(use-package expand-region
  :bind (("C-=" . er/expand-region)
         ("C--" . er/contract-region)))
```

### 3.3 用法

光标在 `foo` 这个单词中间:
- 按 `C-=` → 选 `foo` (整个 word)
- 再 `C-=` → 选 `foo bar` (整个 sexp)
- 再 `C-=` → 选 `(foo bar baz)` (外层括号)
- 再 `C-=` → 选 `defun ... (foo bar baz) ...` (整个 defun)
- 再 `C-=` → 选整个 buffer
- `C--` 反向缩回

**一个键从单词扩到 buffer**。

### 3.4 原理

`expand-region` 用 syntax table + parse-partial-sexp 判断"现在最合理的边界"。语言感知 (支持 Python/JS/Elisp/Rust/C++/HTML/Markdown 等)。

每个语言有"expansion rule"——`er/mark-word` → `er/mark-symbol` → `er/mark-inside-quotes` → `er/mark-outside-quotes` → `er/mark-inside-pairs` → `er/mark-method-call` → ...

### 3.5 配合 try-expand

`er/try-region-settings` 可以加 try 函数。例:
```elisp
(setq er/try-expand-list '(er/mark-word
                            er/mark-symbol
                            er/mark-method-call
                            er/mark-comment
                            er/mark-url
                            er/mark-email
                            er/mark-defun))
```

让 expand-region 更智能——遇到 URL 直接跳到 URL 范围。

---

## 4. multiple-cursors (多光标同时编辑)

选中一处后,你想"对每个相似地方都做同一操作"。这就是 VS Code 的"多光标",Emacs 也有,而且更强。

### 4.1 安装 + 配置

```elisp
(use-package multiple-cursors
  :bind (("C->" . mc/mark-next-like-this)
         ("C-<" . mc/mark-previous-like-this)
         ("C-c C->" . mc/mark-all-like-this)
         ("C-S-c C-S-c" . mc/edit-lines)))
```

### 4.2 用法

例子: 你想把所有 `foo` 改成 `bar`。

**方法 A**: 全局
```
M-x mc/mark-all-like-this RET foo RET
```
所有 `foo` 都变 cursor,你输 `bar`,同时改。

**方法 B**: 递增
```
光标在第一个 foo 上
C->    下一个 foo 也变 cursor
C->    再下一个
C->    再下一个
输入 bar
```

**方法 C**: 行模式
```
选中 10 行
mc/edit-lines
每行都有一个 cursor
```

### 4.3 mc 高级

- `mc/mark-all-like-this-in-defun` — 只在当前函数内
- `mc/mark-all-symbols-like-this` — symbol 语义 (不是字面字符串)
- `mc/mark-next-like-this-word` — 只匹配 word 边界
- `mc/insert-numbers` — 多个 cursor 插入递增数字 (1, 2, 3, ...)
- `mc/sort-regions` — 对每个 cursor 位置排序
- `mc/reverse-regions` — 反转

`mc/insert-numbers` 是神器——你 50 个 cursor,按 `M-x mc/insert-numbers`,自动 1, 2, 3, ..., 50。

---

## 5. embark-act (选中后立刻菜单)

Doom 已经装了 embark。光标在某个东西上按 `C-.`:

- 在 word 上 `C-.` → 菜单 (查找/复制/翻译/拼写/...)
- 在 url 上 `C-.` → 菜单 (打开/copy/shorten)
- 在文件名上 `C-.` → 菜单 (打开/dired/grep)
- 在 sexp 上 `C-.` → 菜单 (eval/evaluate-replace/...)
- 在 color hex 上 `C-.` → 菜单 (copy/preview/insert)

embark **自动识别**"光标处是什么东西",给对应菜单。**不用选 region,直接对 thing-at-point 操作**。

### 5.1 配置

```elisp
(use-package embark
  :bind (("C-." . embark-act)
         ("M-." . embark-dwim)))
```

`embark-dwim` 是"做最合理的事"——在 url 上打开,在 symbol 上跳定义,在 defun 上 eval。

### 5.2 实战场景

光标在 `https://example.com` 上:
- `C-.` → 弹菜单
- 按 `b` (browse) → 浏览器打开
- 或 `w` (copy) → 复制到 kill-ring
- 或 `s` (shorten) → 用 short.io 缩短

光标在 `my-function` symbol 上:
- `C-.` → 弹菜单
- 按 `d` (definition) → 跳定义
- 按 `r` (references) → 查引用
- 按 `e` (eval) → eval

---

## 6. thing-at-point 系列 (Lisp 级别的"选")

写 elisp 时,这些函数返回光标处的东西:

```elisp
(thing-at-point 'word)      ; "hello"
(thing-at-point 'symbol)    ; "my-variable"
(thing-at-point 'sexp)      ; "(foo bar baz)"
(thing-at-point 'line)      ; "current line\n"
(thing-at-point 'sentence)
(thing-at-point 'paragraph)
(thing-at-point 'list)      ; "(...)" 当前列表
(thing-at-point 'defun)     ; 当前函数
(thing-at-point 'url)       ; "https://..."
(thing-at-point 'email)
(thing-at-point 'filename)
(thing-at-point 'number)
(thing-at-point 'whitespace)
(thing-at-point 'page)
```

每个有对应 mark:

```elisp
(mark-thing 'word)
(mark-thing 'sexp)
;; 等等
```

### 6.1 实战: 写自己的快捷命令

```elisp
(defun my-copy-url-at-point ()
  (interactive)
  (let ((url (thing-at-point 'url)))
    (when url
      (kill-new url)
      (message "Copied: %s" url))))

(global-set-key (kbd "C-c u") #'my-copy-url-at-point)
```

光标在 URL 上,`C-c u` 一键复制——不用选中再 M-w。

### 6.2 bounds-of-thing-at-point

如果你要"选"而不是"取值":
```elisp
(bounds-of-thing-at-point 'word)
;; → (5 . 10)  ; word 在 buffer 5-10 之间
```

---

## 7. avy (跳到任意位置,瞬间选)

`avy` 让你按 2-3 个字符跳到屏幕任意位置——**不用挪光标,直接飞过去**。

### 7.1 安装 + 配置

```elisp
(use-package avy
  :bind (("C-:" . avy-goto-char)        ; 输入 1 字符,跳
         ("C-'" . avy-goto-char-2)      ; 2 字符
         ("M-g f" . avy-goto-line)
         ("M-g w" . avy-goto-word-1)
         ("M-g e" . avy-goto-word-0)))
```

### 7.2 用法

光标在文件头,你想跳到屏幕中间的某个 `foo`:
- 按 `C-:` 然后输入 `f`
- 屏幕上所有 `f` 字符位置出现字母 (a, b, c, ...)
- 按对应字母,光标飞过去

这是**最快的"远距离跳"**——不需要 `C-s foo RET`,直接 `C-:` `f` `x` (3 键)。

### 7.3 配合 mark

```elisp
(defun my-avy-mark (char)
  "Set mark at avy target."
  (interactive "cChar: ")
  (let ((origin (point)))
    (avy-goto-char char)
    (push-mark origin t)
    (activate-mark)))

(global-set-key (kbd "C-x C-j") #'my-avy-mark)
```

按 `C-x C-j` + 字符,瞬间"从当前位置 mark 到屏幕任意位置"。**这是最快的"选两处之间"**。

---

## 8. selected (选了之后弹出操作菜单)

`selected` 包——选中任何东西后,**自动**激活一个临时 keymap:

```elisp
(use-package selected
  :config
  (selected-global-mode 1)
  (define-key selected-key-map (kbd "u") #'upcase-region)
  (define-key selected-key-map (kbd "l") #'downcase-region)
  (define-key selected-key-map (kbd "s") #'sort-lines)
  (define-key selected-key-map (kbd "r") #'replace-string)
  (define-key selected-key-map (kbd "c") #'comment-or-uncomment-region)
  (define-key selected-key-map (kbd "f") #'fill-region)
  (define-key selected-key-map (kbd "q") #'selected-off))
```

用法:
- `M-h` 选段
- 现在按 `u` (大写)、`l` (小写)、`s` (排序)、`r` (替换)、`c` (注释)、`f` (填充)
- 按 `q` 退出 selected mode

**这是 Sublime/VS Code 没有的**——选后临时菜单,执行完自动失效。

---

## 9. 实战 workflow 例子

### 场景 1: 把整个函数的 `foo` 改名 `bar`

**笨办法** (新手):
```
C-SPC
C-M-h  ; 实际新手不知道这个,挪光标选
M-% foo RET bar RET
```

**极客办法 A** (mark + replace):
```
光标在 defun 内任意位置
C-M-h              ; 选整个 defun
M-% foo RET bar RET
!                  ; 全替换
```

**极客办法 B** (expand-region + mc):
```
光标在 foo 上
C-= C-= C-=        ; expand 到 defun
C-> C-> C-> ...    ; 选所有 foo 为 multicursor
bar
```

### 场景 2: 给每个 TODO 加 deadline

```
光标在 TODO 列表第一行
mc/edit-lines      ; 每行一个 cursor
END                ; 每行到行尾
" : DEADLINE <2026-07-01>"
```

或者:
```
mark-whole-buffer (C-x h)
mc/mark-all-like-this RET TODO RET
每行 TODO 都 cursor
end-of-line
插入 deadline
```

### 场景 3: 选 defun 并 eval

`*scratch*` 里:
```
C-M-h              ; 选 defun
C-x C-e            ; Doom 默认 eval-region
```

### 场景 4: 翻转一个 url 路径

```
光标在 url 上
C-. (embark-act) → u (url)
选 "shorten" 或 "open"
```

### 场景 5: 选两段不相邻的代码

```
C-M-h    ; 选第一段
M-w      ; 复制
C-g      ; 取消 region
avy 跳到第二段
C-M-h    ; 选第二段
M-w      ; 复制 (kill-ring 现在有 2 项)
```

### 场景 6: 把代码块整体右移

```
C-M-h              ; 选代码块
C-x r t SPC SPC SPC SPC RET  ; 矩形每行前加 4 空格
```

或更简单:
```
C-M-h
C-x TAB            ; Doom 的 indent-rigidly
4 SPC              ; 移动 4 空格
```

---

## 10. 完整 Doom 配置 (复制即用)

```elisp
;;; ~/.config/doom/config.el

;; expand-region (语义扩张)
(map! :n "C-=" #'er/expand-region
      :n "C--" #'er/contract-region)

;; multiple-cursors
(use-package multiple-cursors
  :config
  (map! :n "C->" #'mc/mark-next-like-this
        :n "C-<" #'mc/mark-previous-like-this
        :n "C-M->" #'mc/mark-all-like-this-in-defun
        :i "C-M->" #'mc/mark-all-like-this))

;; 选 region 后的临时菜单
(use-package selected
  :config
  (selected-global-mode 1)
  (map! :map selected-key-map
        "u" #'upcase-region
        "l" #'downcase-region
        "c" #'comment-or-uncomment-region
        "s" #'sort-lines
        "r" #'replace-string
        "f" #'fill-region
        "q" #'selected-off))

;; 自定义快捷 mark
(map! :n "M-SPC" #'mark-sexp
      :leader "v w" #'mark-word
      :leader "v d" #'mark-defun
      :leader "v p" #'mark-paragraph
      :leader "v b" #'mark-whole-buffer)

;; thing-at-point 操作
(defun my/duplicate-thing-at-point ()
  "Duplicate the sexp at point."
  (interactive)
  (let ((bounds (bounds-of-thing-at-point 'sexp)))
    (when bounds
      (copy-region-as-kill (car bounds) (cdr bounds))
      (end-of-thing 'sexp)
      (insert " ")
      (yank))))

(map! :n "C-c d" #'my/duplicate-thing-at-point)

;; avy mark
(defun my-avy-mark (char)
  "Mark from current point to avy target."
  (interactive "cChar: ")
  (let ((origin (point)))
    (avy-goto-char char)
    (push-mark origin t)
    (activate-mark)))

(map! :n "C-x C-j" #'my-avy-mark)
```

---

## 11. 推荐的"5 分钟练习"

每天练 5 分钟:

1. 打开任意代码文件
2. **只用** `C-M-h` / `M-h` / `C-=` 选东西,不挪光标
3. 选完立刻按 `C-w` 杀 / `M-w` 复制 / `M-%` 替换 / `C-.` 操作
4. 练 1 周,养成"语义选择"的肌肉记忆

1 周后,你会**几乎不用 `C-SPC`**。

---

## 12. 进阶: 键盘操作的极限

学完上述,你已经是 80% 的 Emacs 极客。最后 20%:

### 12.1 跳过选区,直接"对 thing 操作"

最高境界是**根本不选**——`embark` 在光标处操作:
```
光标在 url 上 → C-. → b → 浏览器打开
光标在 defun 上 → C-. → e → eval
光标在 word 上 → C-. → s → 拼写检查
```

不选 region,直接对 thing-at-point 操作。**这是最快的**。

### 12.2 用 macro 代替选区

有时"选+操作"不如"录一个宏":
```
F3
C-s foo
M-d
RET
F4
C-u 10 C-x e
```

这比"选 + query-replace" 还快。

### 12.3 Lispy / Paredit (Lisp 结构编辑)

写 Elisp 时,`lispy` 或 `paredit` 让你"操作 S-表达式":
- `(` → 自动配对
- `M-s` → split sexp
- `M-j` → join
- `M-r` → raise (用子 sexp 替换父)
- `C-u M-d` → 不同单位杀

这是"超越选区"——直接用结构操作。

---

## 13. 总结 (给极客的真话)

Emacs 极客的"快速选" = **3 个包**:

1. `expand-region` (`C-=` 扩张)
2. `multiple-cursors` (`C->` 多选)
3. `embark` (`C-.` 对 thing-at-point 操作)

加上内置 4 个键:
- `C-M-h` mark-defun
- `M-h` mark-paragraph
- `C-M-SPC` mark-sexp
- `C-x h` mark-whole-buffer

这 7 个键组合覆盖 95% 的"选+操作"场景。`C-SPC + 挪光标` 留给最后 5% 罕见情况。

---

## 14. 自测

1. `C-M-h` 选什么?
2. expand-region 怎么从 word 扩到 defun?
3. multiple-cursors `C->` 干啥?
4. embark 在 url 上按 `C-.` 弹什么菜单?
5. `thing-at-point 'url` 返回什么?

**答案**:
> 1. mark-defun,当前函数
> 2. 按 C-= 重复,每次扩大边界
> 3. 选下一个相同 word (添加一个 multicursor)
> 4. URL 操作菜单 (open/copy/shorten)
> 5. 光标处的 URL 字符串

---

## 15. 下一步

加这份配置后,练 1 周。你会发现"挪光标"越来越少,"语义选"越来越多。

1 周后,进入下一个极客境界: **结合 AI** (gptel + selected + region)——选中代码,AI 一键解释/重构/翻译。
