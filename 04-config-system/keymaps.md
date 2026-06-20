# Keymaps 详解

> 学完这个文件,你能熟练操作 Emacs 的 keymap 系统

---

## 1. Keymap 的本质

### 1.1 数据结构

keymap 的本质是"按键序列 → 命令"的查找表。Emacs 内部把它实现成嵌套的 vector——顶层 vector 按 ASCII 字符索引,每个槽指向一个 command 或另一个 keymap(对于 prefix key)。这种结构让按键 lookup 是 O(1) 的向量访问,极快。

下面是 global-map 的概念示意。`?\C-a` 这种 control 字符直接作为 vector 索引,普通字符如 `?a` 也是。prefix key(如 C-x)指向另一个完整的 keymap——这就是为什么 `C-x C-f` 是两段查找: 先查 global-map[?\C-x] 得到 Control-X-prefix 这个 keymap,再查它[?\C-f] 得到 find-file:

```
global-map (vector indexed by char):
  [nil nil ... 
   ?a → self-insert-command
   ?A → self-insert-command (shift)
   ...
   ?\C-a → move-beginning-of-line
   ...
  ]
```

prefix key (如 `C-x`)指向**另一个 keymap**:

```
global-map[?\C-x] → Control-X-prefix (另一个 keymap)
   Control-X-prefix[?\C-f] → find-file
   Control-X-prefix[?\C-s] → save-buffer
   Control-X-prefix[b] → switch-to-buffer
   ...
```

这种"keymap 套 keymap"的设计让 Emacs 的键位空间可以无限扩展——`C-c` 是用户自定义 prefix,`C-c C-c`、`C-c C-d` 等组合几乎用不完。

### 1.2 sparse vs full keymap

Emacs 提供两种 keymap 实现: full keymap 启动时预分配所有 128 个 ASCII 槽(占内存,但 lookup 直接索引,快);sparse keymap 懒分配(像 hash table,只存非 nil 项,内存高效)。选择哪种取决于你的使用模式:

```elisp
(make-sparse-keymap)          ; 懒分配,只存非 nil 项
(make-keymap)                 ; full,128 entry
```

minor mode 用 sparse (内存高效)。
global-map 是 full (启动开销,但 lookup 快)。

为什么 minor mode 用 sparse?因为每个 minor mode 通常只绑几个键,full keymap 会浪费 100+ 个空槽。global-map 用 full 是因为它绑了几百个键,full 的索引速度优势值得。

### 1.3 active keymaps

Emacs 按下面优先级查找 active keymap——理解这个顺序是解决"键位不工作"的核心。注意 minor mode 排在 major mode 之前,这就是为什么你的 minor mode 键位能覆盖 major mode:

按优先级:

```
1. overriding-local-map        (rare, 编程用)
2. text property 'keymap       (光标处的)
3. overriding-terminal-local-map
4. minor-mode-overriding-map-alist
5. minor-mode-map-alist        (所有 active minor modes)
6. current-local-map           (major mode)
7. global-map
```

Emacs 按顺序查找,第一个找到的生效。这意味着 minor mode 能"抢先" major mode,而 text property keymap(给 buffer 中特定文本附加的 keymap)甚至比 minor mode 还优先——这是为什么 org 链接、按钮能拦截按键。

---

## 2. 查看当前 keymap

调试 keymap 的关键是知道"现在 active 的 keymap 里有什么"。Emacs 提供了几个查看命令。

### 2.1 describe-bindings

`C-h b` 列出当前 buffer 的**所有** active bindings——包括 global、major mode、所有 minor mode 的。这是排查"键位被谁绑了"的第一神器:

```
C-h b
```

显示当前 buffer 的所有 active bindings。

### 2.2 describe-key

如果你只想知道某个具体键绑到啥,用 `C-h k`:

```
C-h k KEY
```

显示 KEY 绑到啥。

### 2.3 active maps

在 Elisp 里查当前 active 的 keymap 列表——写工具脚本时有用:

```elisp
(current-active-maps t)       ; list of active keymaps
```

### 2.4 where-is

反向查询: 给一个命令,查它绑到哪些键。VS Code 没有等价功能,这是 Emacs 的优势——键位是双向可查的:

```
C-h w COMMAND
```

反向: 命令绑到哪些键。

---

## 3. define-key

### 3.1 基础

`define-key` 是绑键的原语,所有其他绑键函数(global-set-key 等)都是它的包装。第一个参数是 keymap,第二个是 key(可以用 kbd 字符串、vector、list 三种形式),第三个是 definition:

```elisp
(define-key KEYMAP KEY DEFINITION)

;; KEY 可以是:
(kbd "C-c d")                 ; 字符串 → kbd 解析
[?\C-c ?d]                    ; vector of events
[(control ?c) ?d]             ; list form
[C-c ?d]                      ; 简化 vector
```

### 3.2 DEFINITION

definition 可以是多种类型——command、lambda、keymap(变 prefix)、nil(解绑)、字符串(键盘宏)、menu item。这种多态让一个 API 能处理各种场景:

可以是:

| 类型 | 含义 |
|---|---|
| symbol | command |
| lambda / function | 函数 |
| keymap | 变成 prefix |
| nil | 解绑 |
| string | keyboard macro (按键序列) |
| `(menu-item ...)` | menu item |

```elisp
(define-key map (kbd "C-c d") #'my-func)
(define-key map (kbd "C-c d") nil)            ; 解绑
(define-key map (kbd "C-c") (make-sparse-keymap))  ; 变 prefix
```

### 3.3 global-set-key

`global-set-key` 是 `define-key global-map` 的简写——日常绑全局键用这个就行:

```elisp
(global-set-key KEY DEF) = (define-key global-map KEY DEF)
```

### 3.4 local-set-key

`local-set-key` 在当前 buffer 的 local map(major mode 的)里定义。但更推荐的做法是用 `define-key` + `with-eval-after-load` 直接改 major mode 的 keymap,这样所有同类 buffer 都生效,而不只是当前 buffer:

```elisp
(local-set-key KEY DEF)
;; 当前 buffer 的 local map 里定义
```

通常用 `define-key` 加 major mode map:

```elisp
(with-eval-after-load 'python
  (define-key python-mode-map (kbd "C-c C-c") #'python-shell-send-buffer))
```

`with-eval-after-load` 等 feature 加载后再跑——这很关键,因为 `python-mode-map` 在 python 包加载之前还不存在。

### 3.5 unbind

解绑就是把 definition 设成 nil:

```elisp
(global-unset-key KEY)
(local-unset-key KEY)
```

或 `(define-key map KEY nil)`。

---

## 4. kbd 函数

### 4.1 语法

`kbd` 把人类可读的字符串解析成 Emacs 内部的事件 vector。直接写 vector 很痛苦(要记得 `?\C-c` 这种转义、modifier 顺序),kbd 让你用自然语法——这是 Emacs 对"可读性"的细致考虑:

```elisp
(kbd "C-c d")            → [?\C-c ?d]
(kbd "M-x")              → [?\M-x]
(kbd "<f5>")             → [<f5>]
(kbd "C-M-s")            → [?\C-\M-s]
(kbd "C-c <return>")     → [?\C-c return]
(kbd "C-c RET")          → 同上
(kbd "M-<up>")           → [M-up]
(kbd "<mouse-1>")        → [mouse-1]
(kbd "C-M-S-<left>")     → [C-M-S-left]
```

### 4.2 modifiers

Emacs 支持六种 modifier——比一般编辑器多。Hyper、Super、Alt 这些"老牌"modifier 来自 Lisp machine 时代,现代键盘上通常映射到 Win/Option/Cmd:

| Modifier | 缩写 |
|---|---|
| Control | C |
| Meta | M |
| Shift | S |
| Super | s |
| Hyper | H |
| Alt | A |

### 4.3 special keys

Emacs 用 `<...>` 语法表示特殊键——功能键、方向键、鼠标事件。这种统一表示法让你可以绑任意输入设备:

`<f1>`, `<return>`, `<tab>`, `<backspace>`, `<delete>`, `<escape>`, `<up>`, `<down>`, `<left>`, `<right>`, `<home>`, `<end>`, `<prior>` (pageup), `<next>` (pagedown), `<insert>`。

鼠标: `<mouse-1>` (左键), `<mouse-2>` (中), `<mouse-3>` (右), `<double-mouse-1>`, `<triple-mouse-1>`, `<drag-mouse-1>`, `<wheel-down>`, `<wheel-up>`。

---

## 5. 修改内置 keymap

### 5.1 修改 global-map

直接改 global-map 是最简单的绑全局键方式。下面三段演示常见模式: 把无用的 C-z(默认 suspend)改成 undo、把 ESC 改成退出、解绑 C-x C-c(避免误退 Emacs):

```elisp
;; 把 C-z 改成 undo (默认 C-z 是 suspend)
(global-set-key (kbd "C-z") #'undo)

;; ESC 取消
(global-set-key (kbd "<escape>") #'keyboard-escape-quit)

;; 不用 C-x C-c (避免误退)
(global-unset-key (kbd "C-x C-c"))
```

### 5.2 修改 mode-map

改 major mode 的 keymap 时,要等 mode 加载完——否则 keymap 还不存在。`with-eval-after-load` 解决这个问题: 它注册一个"等 feature 加载后跑"的回调:

```elisp
(with-eval-after-load 'org
  (define-key org-mode-map (kbd "C-c C-x j") #'my-org-function))
```

`with-eval-after-load` 等 feature 加载后再跑 (此时 keymap 存在)。

### 5.3 给 prefix 加键

`C-c m` 这种 prefix 是 Emacs 留给用户的自定义空间(major mode 不会用)。下面演示如何定义一个自己的 prefix map 并绑到 `C-c m`,这样 `C-c m a`、`C-c m b` 就成了你的快捷命令空间:

```elisp
(define-prefix-command 'my-prefix-map)
(define-key my-prefix-map "a" #'my-action-a)
(define-key my-prefix-map "b" #'my-action-b)
(global-set-key (kbd "C-c m") 'my-prefix-map)
```

现在 `C-c m a` → action-a,`C-c m b` → action-b。

### 5.4 替换 prefix

更结构化的做法: 定义一个 keymap 变量,内部绑多个键,整体绑到一个 prefix。下面这个 `my-org-prefix-map` 是一个"org 操作集合",绑到 C-c o:

```elisp
(defvar my-org-prefix-map
  (let ((map (make-sparse-keymap)))
    (define-key map "a" #'org-agenda)
    (define-key map "c" #'org-capture)
    map)
  "My Org prefix.")

(define-key my-org-prefix-map (kbd "C-c o") 'my-org-prefix-map)
```

---

## 6. Keymap inheritance

Emacs 27+ 支持让一个 keymap 继承另一个——子 keymap 自动有父 keymap 的所有 binding。这让"扩展某个 mode 的 keymap"变得干净: 不污染原 keymap,只加自己的键。

### 6.1 set-keymap-parent

```elisp
(defvar my-extended-map
  (let ((map (make-sparse-keymap)))
    (set-keymap-parent map prog-mode-map)
    (define-key map (kbd "C-c C-z") #'my-repl)
    map))
```

`my-extended-map` 继承 `prog-mode-map`,加自己的键。

### 6.2 make-composed-keymap

`make-composed-keymap` 合并多个 keymap 成一个虚拟 keymap——查找时按顺序查每个。这种"组合"模式让你能"叠加"多个 keymap 而不修改它们:

```elisp
(make-composed-keymap MAP-LIST &optional PARENT)
```

合并多个 keymap。

---

## 7. text-property keymap

### 7.1 给 buffer 区域加 keymap

Emacs 有一个强大的特性: 你可以给 buffer 中**一段文本**附加 keymap——光标在这段文本上时,这些键位才生效。这是 org-mode 的链接、按钮、可点击元素背后的机制:

```elisp
(let ((map (make-sparse-keymap)))
  (define-key map [mouse-1] #'my-click)
  (define-key map (kbd "RET") #'my-enter)
  (put-text-property START END 'keymap map))
```

START 到 END 之间,点击鼠标或按 RET 触发自定义。

### 7.2 例子: 可点击的 link

下面是一个完整例子: 插入一段文本,让它变成可点击的"链接"——鼠标点或按 RET 都能打开 URL。这种"text property + keymap"的模式是 Emacs 创建交互式 buffer(*Help*、*Messages*、Magit status)的基础:

```elisp
(defun my-insert-link (text url)
  "Insert clickable TEXT linking to URL."
  (let ((map (make-sparse-keymap))
        (start (point)))
    (insert text)
    (let ((end (point)))
      (define-key map [mouse-1]
        (lambda () (interactive) (browse-url url)))
      (define-key map (kbd "RET")
        (lambda () (interactive) (browse-url url)))
      (put-text-property start end 'keymap map)
      (put-text-property start end 'face 'link)
      (put-text-property start end 'help-echo url))))
```

(Module 6 W7 详细讲)

---

## 8. menu

### 8.1 menu-bar

Emacs 的 menu bar(图形菜单)也是 keymap——只是 key 是 `[menu-bar ...]` 这种特殊 event。下面演示如何在 menu bar 加一个自定义菜单:

```elisp
(define-key global-map [menu-bar my-menu]
  (cons "My Menu" (make-sparse-keymap "My Menu")))

(define-key global-map [menu-bar my-menu my-action]
  '(menu-item "Do Something" my-action
              :help "Run my-action"))
```

### 8.2 easy-menu (简化)

直接用 `define-key` 写 menu 很啰嗦。`easy-menu` 包提供了声明式 API,更易写易读:

```elisp
(require 'easymenu)

(easy-menu-define my-menu global-map "My Menu"
  '("My Menu"
    ["Action A" my-action-a t]
    ["Action B" my-action-b t]
    "---"
    ["Quit" keyboard-quit t]))
```

---

## 9. Keymap 调试

### 9.1 单键失灵

键不工作是最常见的 Emacs 烦恼。第一步永远是 `C-h k KEY` 看它绑到啥。如果绑到非预期的命令,可能被某个 minor mode 抢了——`C-h m` 看 active minor modes。或者光标处的文本有 text property keymap(`C-u C-x =` 查看):

```
C-h k KEY
```

看绑到啥。如果不对:
- 是不是 minor mode 覆盖了? `C-h m` 看 active minor modes
- 是不是 text property? `C-u C-x =` 看 char 的 properties

### 9.2 prefix 失灵

如果 prefix(如 C-c)看起来"没反应"——其实 Emacs 在等你按下一个键。用 `C-h k C-c` 看 C-c 绑到哪个 prefix keymap,再看其内容:

`C-h k C-c` 看 C-c 绑啥。如果绑到 prefix keymap,看其内容。

### 9.3 哪个 minor mode 抢了键

下面这段代码扫描所有 active minor mode,看哪个绑了指定键——排查"键被抢"的神器:

```elisp
(defun which-active-modes ()
  (let ((active-modes nil))
    (mapc (lambda (mode)
            (when (and (boundp mode) (symbol-value mode))
              (push mode active-modes)))
          minor-mode-list)
    active-modes))

;; 用:
(dolist (mode (which-active-modes))
  (let ((map (symbol-value (intern (format "%s-map" mode)))))
    (when (and (keymapp map) (lookup-key map (kbd "KEY")))
      (message "%s binds KEY" mode))))
```

---

## 10. 实战: 个人键位库

下面是一个完整的"个人键位 minor mode"——把所有自定义键集中在一个 minor mode 里。这种模式的优势: 整体可开关、不污染 global-map、优先级高不会被 major mode 覆盖、可以放 GitHub 分享。几乎所有 Emacs 老用户都有这样一个 minor mode:

```elisp
(defvar my-keys-mode-map
  (let ((map (make-sparse-keymap)))
    ;; window
    (define-key map (kbd "M-o") #'other-window)
    (define-key map (kbd "M-O") (lambda () (interactive) (other-window -1)))
    ;; buffer
    (define-key map (kbd "M-k") #'kill-this-buffer)
    (define-key map (kbd "M-K") #'kill-buffer-and-window)
    ;; file
    (define-key map (kbd "C-c f s") #'save-buffer)
    (define-key map (kbd "C-c f r") #'recentf-open-files)
    ;; search
    (define-key map (kbd "C-c s") #'consult-line)
    ;; editing
    (define-key map (kbd "C-c d") #'my-duplicate-line)
    (define-key map (kbd "M-<up>") #'my-move-line-up)
    (define-key map (kbd "M-<down>") #'my-move-line-down)
    map)
  "Personal keymap.")

(define-minor-mode my-keys-mode
  "Personal keys."
  :global t
  :lighter " MK"
  :keymap my-keys-mode-map)

(my-keys-mode 1)
```

把所有个人键位放一个 minor mode,管理清晰,不会污染 global。

---

## 11. 自测

1. active keymap 顺序?
2. `kbd` 函数干啥?
3. `:global` minor mode 和普通 minor mode 在 keymap 的区别?
4. 怎么给 buffer 一段文本加自定义键?
5. 怎么查键位失灵的原因?

**答案**:
> 1. overriding > text prop > minor > major > global
> 2. 把字符串解析成 event vector
> 3. global minor mode 影响所有 buffer;普通的只当前 buffer
> 4. put-text-property 加 'keymap 属性
> 5. C-h k + C-h m (查 active minor modes)

---

## 12. 下一步

进入 `hooks.md`。
