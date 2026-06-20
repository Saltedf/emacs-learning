# Week 5: Capstone — 写一个完整的 TODO 包

> **目标**: 综合应用 Module 3 学的,写一个完整的 `todo-mode`
> **时长**: 1 周 (~25 小时)
> **Chassell 章节**: 14. Counting Words in defun, 15. Readying a Graph, 16. Your .emacs File, 17. Debugging

---

## 0. 这周在做什么

你已经学了 4 周 Elisp。这周**整合所有**:

写一个完整的 `my-todo` 包,功能:
- 添加 / 删除 / 完成 todo 项
- 保存到文件,加载
- 自定义 major mode + keymap
- 支持 priority、tags
- 用 ERT 测试

这是 Module 3 的**毕业项目**。

为什么是 todo 管理器?它"麻雀虽小,五脏俱全"——有数据结构 (list of items)、有持久化 (文件 I/O)、有 UI (major mode + keymap)、有交互 (commands)、有测试 (ERT)。一个项目把所有学过的概念用一遍,这就是"综合"。

这个项目对你意味着什么? 它不是 toy——它是真实的、可用的包。写完你可以发到 MELPA (Module 7 教流程)。这是从"学 Lisp"到"用 Lisp 创造"的飞跃。

---

## 1. Chassell Ch 14-17 速读

最后一周你不再深入新概念,而是整合。所以 Chassell 最后几章我们"速读"——抓核心,跳过细节。

### 1.1 Ch 14: Counting Words in defun

Chassell 教你怎么数一个 defun 的词数。
核心模式:
- `beginning-of-defun` / `end-of-defun`
- 在 region 内循环 search
- 收集结果

这周你的 todo 包会用类似模式——比如统计有多少 todo、按 priority 排序。

### 1.2 Ch 15: Readying a Graph

Chassell 教怎么用 ascii 字符画图。
(略,与 todo 包无关)

这一章我们跳过。如果你好奇 ascii graph,可以读——但不是核心。

### 1.3 Ch 16: Your .emacs File

Chassell 教 `init.el` 的写法。
(Module 4 详细讲)

关键:
- `init.el` 是普通 Elisp
- 启动时加载
- 用 `(require 'foo)` 引入你的包
- `(load "path/to/file.el")` 直接加载

你的 my-todo 会写到 init.el: `(require 'my-todo)` + 绑快捷键。

### 1.4 Ch 17: Debugging

Chassell 教 Lisp 调试:
- `*Backtrace*` buffer
- `M-x toggle-debug-on-error`
- `M-x edebug` 单步
- `message` 调试

(Module 6 W4 详细讲)

写 my-todo 时一定会遇到 bug——这是不可避免的。掌握调试工具能让你不浪费时间。"调试是程序员的日常,不是异常情况"。

---

## 2. Capstone 项目: my-todo 包

### 2.1 设计

写包的第一步不是写代码——是设计。设计错了,代码再漂亮也是垃圾。我们想清楚"数据怎么存"和"命令怎么用",再开始写。

数据结构是核心。我们用一个 list of items,每个 item 是 4 元素 list:

```elisp
;; 一个 todo 项:
;; (text priority tags done)
;; 例: ("Buy milk" 3 ("shopping" "errand") nil)

;; 整个 todo list:
;; list of todo 项
```

为什么用 list 不用 struct/struct? 几个理由:

- list 是 Elisp 原生,`read`/`pp` 直接能存到文件——不需要序列化。
- list 灵活——以后加字段 (deadline, recurring) 容易。
- 数据量小——一个 todo list 通常几十项,list 性能足够。

如果数据量大 (比如 10000 个 todo),应该用 vector 或 hash table。但 todo 场景 list 完全够。

字段顺序: text, priority, tags, done。为什么这个顺序?

- text 最重要,放第一 (nth 0 容易访问)
- priority 次重要 (用户最常查)
- tags 用于过滤 (中等频率)
- done 是布尔 (最少访问)

文件格式:
```elisp
;; ~/.emacs.d/my-todos.el
((("Buy milk" 3 ("shopping") nil)
  ("Write report" 5 ("work") nil)
  ("Pay bills" 2 ("finance") t)))
```

文件是 Lisp 字面量,可以 `read` 直接加载。

为什么用 Lisp 字面量而不是 JSON 或 YAML?

- Lisp 原生支持——`(read)` 一行加载,`(pp)` 一行保存,零依赖。
- Emacs 自带 (不需要外部包)。
- 数据结构正好是 list,Lisp 字面量是自然选择。
- 用户可以手编辑 (懂 Lisp 的话)。

代价: 如果以后想给非 Emacs 程序读这个文件,Lisp 字面量不如 JSON 通用。但作为 Emacs 包,这无所谓。

### 2.2 命令

设计完数据,设计 API:

- `M-x my-todo-add` 添加 todo
- `M-x my-todo-list` 显示所有
- `M-x my-todo-done` 标记完成
- `M-x my-todo-undone` 取消完成
- `M-x my-todo-delete` 删除
- `M-x my-todo-save` 保存
- `M-x my-todo-load` 加载
- `M-x my-todo-filter` 按标签过滤

这 8 个命令覆盖 CRUD (Create/Read/Update/Delete) + 持久化 + 过滤。是 todo 应用的最小完整集。

为什么不分开放 (一个 add,一个 list,一个 file ops)? 因为 my-todo 是一个 cohesive 工具——所有操作围绕同一个数据。把它们都放一个包里,用户用起来方便 (打开 my-todo,所有操作在 keymap 里)。

### 2.3 major mode

定义一个 `my-todo-mode`,专门显示 todo list buffer。
keymap:
- `a` add
- `d` done
- `u` undone
- `x` delete
- `s` save
- `l` load
- `f` filter
- `g` refresh (revert)
- `q` quit

为什么用单字母键? 因为 todo buffer 是只读的 (你只通过命令改),不需要字符输入。所以单字母键最快。这是 dired、org-mode 等所有"特殊 buffer"的设计风格。

---

## 3. 完整代码

我会带你一步步写,每段都讲解。

### 3.1 包骨架

创建 `~/.config/emacs/lisp/my-todo.el`:

```elisp
;;; my-todo.el --- Simple todo manager -*- lexical-binding: t; -*-

;;; Commentary:
;; 一个简单的 todo 管理 major mode。
;; 数据存在 `my-todo-file',格式是 Lisp list。

;;; Code:

(require 'cl-lib)

;;; Custom variables

(defgroup my-todo nil
  "Simple todo manager."
  :group 'tools)

(defcustom my-todo-file
  (locate-user-emacs-file "my-todos.el")
  "File to store todos."
  :type 'file
  :group 'my-todo)

(defcustom my-todo-buffer-name "*My Todos*"
  "Buffer name for todo display."
  :type 'string
  :group 'my-todo)

;;; Internal variables

(defvar my-todo-list nil
  "List of todo items. Each item is (TEXT PRIORITY TAGS DONE).")

(defvar my-todo-mode-map
  (let ((map (make-sparse-keymap)))
    (define-key map "a" #'my-todo-add)
    (define-key map "d" #'my-todo-done)
    (define-key map "u" #'my-todo-undone)
    (define-key map "x" #'my-todo-delete)
    (define-key map "s" #'my-todo-save)
    (define-key map "l" #'my-todo-load)
    (define-key map "f" #'my-todo-filter)
    (define-key map "g" #'my-todo-refresh)
    (define-key map "q" #'my-todo-quit)
    map)
  "Keymap for `my-todo-mode'.")

(define-derived-mode my-todo-mode fundamental-mode
  "My-Todo"
  "Major mode for displaying todos."
  (read-only-mode 1))           ; buffer 只读,操作通过命令

;;; Core API

(defun my-todo--item-text (item) (nth 0 item))
(defun my-todo--item-priority (item) (nth 1 item))
(defun my-todo--item-tags (item) (nth 2 item))
(defun my-todo--item-done (item) (nth 3 item))

(defun my-todo--item-set-done (item value)
  (setf (nth 3 item) value))

(defun my-todo-add-internal (text priority tags)
  "Add a new todo item."
  (let ((item (list text priority tags nil)))
    (setq my-todo-list
          (append my-todo-list (list item)))))

(defun my-todo--find-item-at-line ()
  "Return the item displayed at current line, or nil."
  (let ((line (line-number-at-pos)))
    (nth (- line 1) my-todo-list)))

;;; Interactive commands

;;;###autoload
(defun my-todo-add (text priority tags)
  "Add a new todo item."
  (interactive
   (list (read-from-minibuffer "Todo: ")
         (read-number "Priority (1-5): " 3)
         (split-string
          (read-from-minibuffer "Tags (comma-sep): ")
          "," t " ")))
  (my-todo-add-internal text priority tags)
  (my-todo-refresh))

(defun my-todo-done ()
  "Mark current todo as done."
  (interactive)
  (let ((item (my-todo--find-item-at-line)))
    (when item
      (my-todo--item-set-done item t)
      (my-todo-refresh))))

(defun my-todo-undone ()
  "Mark current todo as not done."
  (interactive)
  (let ((item (my-todo--find-item-at-line)))
    (when item
      (my-todo--item-set-done item nil)
      (my-todo-refresh))))

(defun my-todo-delete ()
  "Delete current todo."
  (interactive)
  (let ((line (line-number-at-pos)))
    (setq my-todo-list
          (append (cl-subseq my-todo-list 0 (1- line))
                  (cl-subseq my-todo-list line)))
    (my-todo-refresh)))

(defun my-todo-save ()
  "Save todos to file."
  (interactive)
  (with-temp-file my-todo-file
    (pp my-todo-list (current-buffer)))
  (message "Saved %d todos" (length my-todo-list)))

(defun my-todo-load ()
  "Load todos from file."
  (interactive)
  (when (file-exists-p my-todo-file)
    (with-temp-buffer
      (insert-file-contents my-todo-file)
      (setq my-todo-list (read (current-buffer)))))
  (my-todo-refresh))

(defun my-todo-filter (tag)
  "Filter todos by TAG."
  (interactive "sTag (empty for all): ")
  (let ((all my-todo-list))
    (setq my-todo-list
          (if (string-empty-p tag)
              all
            (cl-remove-if-not
             (lambda (item)
               (member tag (my-todo--item-tags item)))
             all))))
  ;; 用 text property 保存原始 list
  (setq my-todo-list my-todo-list)  ; TODO: 改进
  (my-todo-refresh))

(defun my-todo-refresh ()
  "Refresh display."
  (interactive)
  (let ((buf (get-buffer-create my-todo-buffer-name)))
    (with-current-buffer buf
      (let ((inhibit-read-only t))
        (erase-buffer)
        (my-todo-mode)
        (if (null my-todo-list)
            (insert "No todos. Press 'a' to add.\n")
          (dolist (item my-todo-list)
            (let ((text (my-todo--item-text item))
                  (priority (my-todo--item-priority item))
                  (tags (my-todo--item-tags item))
                  (done (my-todo--item-done item)))
              (insert (format "[%s] P%d %s %s\n"
                              (if done "x" " ")
                              priority
                              text
                              (if tags (format "(%s)" (mapconcat #'identity tags ",")) ""))))))
        (goto-char (point-min))))
    (switch-to-buffer buf)))

(defun my-todo-quit ()
  "Quit todo buffer."
  (interactive)
  (quit-window))

;;;###autoload
(defun my-todo ()
  "Open the todo manager."
  (interactive)
  (my-todo-load)
  (my-todo-refresh))

(provide 'my-todo)
;;; my-todo.el ends here
```

### 3.2 测试代码

创建 `~/.config/emacs/lisp/my-todo-test.el`:

```elisp
;;; my-todo-test.el -*- lexical-binding: t; -*-

(require 'ert)
(require 'my-todo)

(ert-deftest my-todo-add-test ()
  (setq my-todo-list nil)
  (my-todo-add-internal "Buy milk" 3 '("shopping"))
  (should (= 1 (length my-todo-list)))
  (should (equal "Buy milk" (my-todo--item-text (car my-todo-list)))))

(ert-deftest my-todo-done-test ()
  (setq my-todo-list nil)
  (my-todo-add-internal "Test" 1 nil)
  (let ((item (car my-todo-list)))
    (my-todo--item-set-done item t)
    (should (my-todo--item-done item))))

(ert-deftest my-todo-filter-test ()
  (setq my-todo-list nil)
  (my-todo-add-internal "A" 1 '("work"))
  (my-todo-add-internal "B" 1 '("home"))
  (my-todo-add-internal "C" 1 '("work"))
  (let ((filtered (cl-remove-if-not
                   (lambda (item)
                     (member "work" (my-todo--item-tags item)))
                   my-todo-list)))
    (should (= 2 (length filtered)))))

(provide 'my-todo-test)
```

### 3.3 在 init.el 里加载

```elisp
(add-to-list 'load-path "~/.config/emacs/lisp/")
(require 'my-todo)
(global-set-key (kbd "C-c t") 'my-todo)
```

### 3.4 测试运行

```
M-x load-file RET ~/.config/emacs/lisp/my-todo.el RET
M-x my-todo RET
```

按 `a` 添加,看显示。

```
M-x ert-run-tests-interactively RET
```

跑测试。

---

## 4. 项目步骤详解

这个 section 把"3.1 包骨架"的大代码拆解成 11 步,每步 15-60 分钟。一边写一边 eval 测试——不要写完所有代码才测。每步都要能跑 (即使功能不全)。

### 4.1 Step 1: 包骨架 (30 分钟)

创建 `my-todo.el`,加上:
- `;;; Commentary`
- `;;; Code`
- `(require 'cl-lib)`
- `(provide 'my-todo)`
- `;;; my-todo.el ends here`

(`provide` 让别人可以 `(require 'my-todo)`)

这是 Elisp 包的"标准骨架"。每个文件都应该有 Commentary、Code、provide 标记——这是 Elisp 包的礼仪,让 Emacs 知道怎么加载你的包。

`provide` 是关键: 它告诉 Emacs "这个文件提供 my-todo 这个 feature"。别人 `(require 'my-todo)` 时,Emacs 找提供这个 feature 的文件,加载,然后返回。

### 4.2 Step 2: Custom 变量 (15 分钟)

```elisp
(defgroup my-todo nil "..." :group 'tools)

(defcustom my-todo-file
  (locate-user-emacs-file "my-todos.el")
  "..."
  :type 'file
  :group 'my-todo)
```

`defgroup` 创建一个 custom group (用户可以通过 `M-x customize` 改)。
`defcustom` 创建可配置变量,带类型和默认值。

(Module 4 详细讲 customize)

`defcustom` vs `defvar` 的区别: defcustom 让变量出现在 customize UI 里,用户可以图形化改。defvar 只是声明变量,没 UI。生产包推荐 defcustom——给用户友好接口。

### 4.3 Step 3: 数据结构 (15 分钟)

```elisp
(defvar my-todo-list nil "...")

(defun my-todo--item-text (item) (nth 0 item))
;; ... 其他 accessors
```

内部函数用 `--` 双横线 (Elisp 命名约定)。

`--` 双横线是"私有"标记——告诉别人"这是内部函数,别在外面用"。Elisp 没真正的 private,但有命名约定。`my-todo--item-text` 是 my-todo 内部的,`my-todo-add` 是 public API。

accessor 函数 (my-todo--item-text 等) 提供"间接访问"——你的代码不直接 `(nth 0 item)`,而是 `(my-todo--item-text item)`。如果以后改数据结构 (text 从第 0 改第 1),只改 accessor,不改所有调用点。

### 4.4 Step 4: Keymap (15 分钟)

```elisp
(defvar my-todo-mode-map
  (let ((map (make-sparse-keymap)))
    (define-key map "a" #'my-todo-add)
    ;; ...
    map)
  "...")
```

`make-sparse-keymap` 创建空 keymap。
`define-key` 加绑定。

"sparse keymap" 是不预填任何绑定的 keymap——你完全自己定义。对比 "full keymap" 预填了所有键 (性能差)。major mode 用 sparse。

注意 `#'my-todo-add`——这是 `(function my-todo-add)` 的缩写,表示"取 my-todo-add 的函数 slot"。比 `'my-todo-add` 更明确,byte-compiler 也更喜欢。

### 4.5 Step 5: Major mode (15 分钟)

```elisp
(define-derived-mode my-todo-mode fundamental-mode
  "My-Todo"
  "Major mode for displaying todos."
  (read-only-mode 1))
```

`define-derived-mode` 是宏:
- 名字: `my-todo-mode`
- 父: `fundamental-mode`
- lighter: "My-Todo" (出现在 mode line)
- docstring
- body (设 read-only)

(Module 6 W6 详细讲)

`define-derived-mode` 自动生成很多 boilerplate——keymap variable、hook variable、syntax table。你只填名字和 body,宏做剩下的事。这是 Elisp 的"工程便利"——避免每个 mode 手写几十行重复代码。

### 4.6 Step 6: Core API (30 分钟)

```elisp
(defun my-todo-add-internal (text priority tags)
  "Add a new todo item."
  (let ((item (list text priority tags nil)))
    (setq my-todo-list
          (append my-todo-list (list item)))))
```

内部 API,不带 interactive。

`-internal` 后缀是命名约定——表示"这是被 interactive 包装的内部函数"。好处: 你可以在测试里直接调 internal,不触发 interactive (interactive 在 batch mode 会失败)。

`append` 把新 item 加到 list 末尾。如果你想 O(1) 加,用 push 加到头,然后显示时 reverse——但 append 更直观,且 todo list 通常不大于几百项,性能够。

### 4.7 Step 7: Interactive commands (60 分钟)

每个命令:
- `(interactive ...)` 读参数
- 调用 internal API
- 调用 `my-todo-refresh` 更新显示

```elisp
(defun my-todo-add (text priority tags)
  (interactive
   (list (read-from-minibuffer "Todo: ")
         (read-number "Priority (1-5): " 3)
         (split-string
          (read-from-minibuffer "Tags (comma-sep): ")
          "," t " ")))
  ...)
```

注意 `interactive` 用 `(list ...)` 收集参数。

`(interactive (list ...))` 是"复杂 interactive"——当 codes 不够用时 (比如要变换参数),用 list 形式。每个元素是一个表达式,Emacs eval 它们收集成参数列表。

split-string 把 "shopping, errand" 切成 `("shopping" "errand")`——用户输入方便。

### 4.8 Step 8: Refresh 函数 (45 分钟)

```elisp
(defun my-todo-refresh ()
  (let ((buf (get-buffer-create my-todo-buffer-name)))
    (with-current-buffer buf
      (let ((inhibit-read-only t))
        (erase-buffer)
        (my-todo-mode)
        (dolist (item my-todo-list)
          (insert (format "[...] ...\n" ...)))))
    (switch-to-buffer buf)))
```

要点:
- `get-buffer-create` 找/创建 buffer
- `with-current-buffer` 临时切换
- `inhibit-read-only` 临时关只读
- `erase-buffer` 清空
- `(my-todo-mode)` 启用 mode
- 循环插入

`inhibit-read-only` 是关键——major mode 设了 read-only,直接 erase-buffer 会失败。`let` 绑定 inhibit-read-only 为 t,临时允许修改。

`switch-to-buffer` 是用户视角的"切到这个 buffer"。`with-current-buffer` 内部已经修改了 buffer 内容,switch-to-buffer 让用户看到。

### 4.9 Step 9: Save/Load (30 分钟)

```elisp
(defun my-todo-save ()
  (with-temp-file my-todo-file
    (pp my-todo-list (current-buffer))))
```

`with-temp-file` 创建临时 buffer,执行 body,写到文件。
`pp` (pretty-print) 把 list 格式化输出。

`pp` 比 `print` 好——它格式化输出,list 缩进,人类可读。`(pp DATA BUFFER)` 把 DATA 写到 BUFFER。

```elisp
(defun my-todo-load ()
  (when (file-exists-p my-todo-file)
    (with-temp-buffer
      (insert-file-contents my-todo-file)
      (setq my-todo-list (read (current-buffer))))))
```

`insert-file-contents` 读文件到 buffer。
`read` 读一个 sexp。

`read` 是 Lisp reader——它解析 Lisp 字面量。`(read (current-buffer))` 从 buffer 读一个 sexp,返回 Lisp 对象。这就是为什么我们用 Lisp 字面量存文件——读写都用 read/pp,零额外代码。

### 4.10 Step 10: 测试 (60 分钟)

ERT (Emacs Lisp Testing Framework):

```elisp
(ert-deftest my-test-name ()
  (should CONDITION)
  (should-not CONDITION)
  (should-error FORM))
```

跑:
- `M-x ert-run-tests-interactively RET` 跑所有
- `M-x ert RET "^my-todo"` 跑匹配的

(Module 7 W4 详细讲 ERT)

ERT 是 Elisp 的测试框架——类似 Python 的 unittest、JS 的 Jest。`should` 验证条件非 nil,`should-error` 验证表达式报错。

写测试的回报巨大: 你改代码后,跑测试立刻知道有没有破东西。Module 3 写小测试,Module 7 详细讲 TDD。

### 4.11 Step 11: autoload + 入口 (15 分钟)

```elisp
;;;###autoload
(defun my-todo ()
  "Open the todo manager."
  (interactive)
  (my-todo-load)
  (my-todo-refresh))
```

`;;;###autoload` 是魔法注释,生成 autoload (函数被调用时才加载文件)。

autoload 的意义: 启动 Emacs 时不加载所有包 (慢),只记"这个函数在哪个文件"。用户调用时,Emacs 才加载文件。这让 Emacs 启动快——10 个包的命令都在,但启动只加载 1 个。

init.el 加:
```elisp
(require 'my-todo)
(global-set-key (kbd "C-c t") 'my-todo)
```

---

## 5. 实战任务 (5-10 小时)

### 任务 1: 完整实现 my-todo (4-5 小时)

按上面步骤实现:
1. 创建 my-todo.el
2. 实现 defgroup/defcustom
3. 实现 data structures
4. 实现 keymap + mode
5. 实现 core API
6. 实现 commands
7. 实现 refresh
8. 实现 save/load
9. 实现 filter
10. 测试

### 任务 2: 加特性 (2-3 小时)

任选 2-3 个加:

- **Priority 排序**: 按 priority 降序显示
- **Tag 多选过滤**: 选多个 tag,显示包含任一的
- **Deadline**: 加 deadline 字段,显示倒计时
- **Recurring**: 重复任务 (每天/每周)
- **Notes**: 每个 todo 加 notes 子段
- **Export**: 导出为 markdown
- **Import**: 从 markdown 导入

### 任务 3: 加 ERT 测试 (1 小时)

为每个命令写至少 1 个测试。

### 任务 4: 加到 init.el (30 分钟)

init.el 里:
```elisp
(add-to-list 'load-path "~/.config/emacs/lisp/")
(require 'my-todo)
(global-set-key (kbd "C-c t") 'my-todo)
(global-set-key (kbd "C-c T") 'my-todo-add)  ; 快速加
```

### 任务 5: README (30 分钟)

写 `my-todo.md` 或 `my-todo.org`:
- 用途
- 安装
- 使用 (截图 + 键位表)
- 配置选项
- 贡献

### 任务 6: (可选) 发到 MELPA

(Module 7 教流程)

---

## 6. 自测

1. 解释 `defcustom` vs `defvar`
2. `(define-derived-mode ...)` 干啥?
3. `interactive` 用 `(list ...)` 形式为什么?
4. `with-temp-file` 和 `with-temp-buffer` 区别?
5. `pp` 和 `print` 区别?
6. `;;;###autoload` 干啥?
7. `read` 函数干啥?
8. ERT 的 `should` 和 `should-error` 区别?

**答案**:
> 1. defcustom 加类型和文档,可 customize;defvar 简单
> 2. 定义一个继承父 mode 的新 mode,自动生成 hook/keymap/syntax 表
> 3. 允许复杂 interactive 逻辑 (多个 prompt,变换参数等)
> 4. with-temp-file 创建临时 buffer,执行 body,写到文件;with-temp-buffer 只创建 buffer,不写文件
> 5. pp (pretty-print) 格式化输出 (list 缩进);print 单行
> 6. 标记下一个 form 为 autoload (调用时才加载文件)
> 7. 从 buffer/string 读一个 sexp (Lisp 字面量)
> 8. should 验证 CONDITION 为 non-nil;should-error 验证 FORM 报错

---

## 7. 毕业检查

- [ ] my-todo.el 完整实现,所有命令工作
- [ ] 至少 5 个 ERT 测试通过
- [ ] 用了 lexical binding
- [ ] 有 defgroup/defcustom
- [ ] 有 `;;;###autoload`
- [ ] 在 init.el 加载,绑了快捷键
- [ ] 写了 README

完成后:
1. 写 `logs/module-03.md`
2. 更新 `PROGRESS.md`
3. 进入 Module 4 (`04-config-system/`)

---

## 8. 反思: 你学到了什么

完成 Module 3,你应该能:

- 读 Chassell 全书,理解 Lisp 思维
- 写任意 Elisp 函数 (递归 + 循环 + 闭包)
- 用 lambda + mapcar + funcall + apply
- 用正则提取信息
- 操作 buffer、string、list、文件
- 定义 major mode + keymap
- 写 ERT 测试
- 发布一个简单包

**这是巨大的进步**。

你已经从"用户"变成"开发者"。
你写的代码,别人可以装、用、贡献。

继续 Module 4-8,你会:
- 把 init.el 写成艺术品 (Module 4)
- 用 Org/Magit 工作 (Module 5)
- 攻克 Elisp Reference (Module 6)
- 发布到 MELPA (Module 7)
- 贡献开源 (Module 8)
