# Package Design

> 学完这个文件,你能设计一个清晰、可维护的包

代码是设计的结果,不是设计的起点。新手最常犯的错误是直接开 editor 敲代码——3 天后发现 API 设计错了,推倒重写。本节教你**先想清楚再写**:命名怎么取、API 边界怎么划、状态怎么放、错误怎么处理、hook 怎么暴露。这些决策一旦做出就很难改(改了就破坏 backward compatibility),所以**设计阶段投入 1 小时,省下实现阶段 10 小时**。

---

## 1. 设计原则

四个原则是软件工程的通用智慧,但放在 Emacs 包语境下有具体含义。

### 1.1 YAGNI (You Aren't Gonna Need It)

不要"未来可能用"的功能。
**现在的需求**最清晰。
未来需求会变,加错了功能反而拖累。

YAGNI 的逻辑是"未来不可预测"。你以为"以后会需要 undo/redo",于是花 3 天实现 undo tree——结果用户根本没 undo 过,你的 undo tree 反而和 Emacs 内置 `undo-tree` 包冲突。这种"为想象中的未来写代码"是新手最大的时间黑洞。

实践规则:

- 只实现"现在明确需要"的功能
- 如果"以后可能要",等真的需要时再加——那时候需求清晰
- 加一个功能前问自己:"如果不加,会死吗?" 答案是不会,就不加

Emacs 圈的好例子:`avy`——只做一件事(基于字符的快速跳转),做透了。它没扩展成"avy + isearch + occur" 大杂烩,反而成为经典。

### 1.2 Single Responsibility

一个包干一件事。
- 不要"my-utility" 大杂烩
- 要"my-todo" "my-snippet" 等专注

"大杂烩"包的典型死法:开始想做 todo,加了 calendar 功能,又加了 pomodoro,最后变成 5000 行的"个人助手"——没人知道它的边界,没人愿意装(怕影响别的),也没人愿意贡献(看不懂)。**专注的包容易懂、容易测、容易维护**。

判断"是不是单一职责"的简单测试:**用一句话能描述你的包吗?** 不能就是太复杂了。

- `magit`: Git porcelain for Emacs (一个职责:Git 操作)
- `org-roam`: Reproducible research with org-mode (一个:笔记网络)
- `vertico`: Completion based on default interface (一个:垂直补全 UI)

反例:`projectile` 是社区最大的"项目导航"包,但因为它什么都想做(运行命令、跳转、grep、切换 buffer),竞争对手 `project.el` (Emacs 内置)虽然功能少但职责清晰,正在替代它。

### 1.3 Convention over Configuration

合理默认,不要让用户配置一切。

这条原则的含义:**开箱即用**。用户装完包,不需要配置就能用——配置是"调优",不是"启动"。如果一个包必须配置才能用,大部分用户会在配置阶段放弃。

实践规则:

- 95% 用户用默认值就够了——不要把每个值都暴露成 defcustom
- 只暴露"经常改"的配置,边缘配置 hardcode 在代码里
- 配置项有清晰 default,不要 `defcustom my-foo nil`——用户不知道 nil 是什么意思

好的例子:`which-key` 装完即用——它已经预设了所有显示规则,只有少数几个 defcustom 让重度用户调。

### 1.4 Backward Compatibility

发布后,API 改要谨慎。
- 加默认值的新参数: OK
- 删参数: deprecate 先,删后
- 改语义: 大版本 (2.0)

发布 = 承诺。用户开始用你的包,他们写代码调你的公开 API。**改 API = 破用户代码**。一旦破,用户就迁走——而且会留下差评。

具体实践:

- 删函数前,先 `defalias` 到新名字,docstring 标记 "deprecated, use X instead"
- 改参数顺序不可能兼容,所以**API 设计阶段就考虑参数顺序**——重要的在前,可选的在后
- 用 SemVer:1.x → 1.y 兼容,1.x → 2.x 不保证兼容

Emacs 圈的反例:`helm` 多次 breaking change,导致大量用户迁移到 `ivy`/`selectrum`/`vertico`。破坏兼容性的代价是整个用户群流失。

---

## 2. 命名

命名是 API 设计的 90%。好的名字让用户不用看文档就猜到功能;坏的名字让用户找不到、记不住、用不对。**花 1 小时想名字,值**。

### 2.1 包名

- 短 (1-3 词)
- 描述功能
- 全小写连字符
- 不冲突

例子: `magit`, `org-roam`, `vertico`, `corfu`

分析这几个好名字:

- `magit`: magic + git(暗示"Git 像变魔术一样简单")。短、独特、易记
- `org-roam`: org + roam(漫游)。描述功能(笔记间漫游),前缀 `org` 表示和 org-mode 集成
- `vertico`: vertical + company(垂直补全)——但拼写独特,易搜索
- `corfu`: completion + region + function——但实际是 Greek pie,短、独特、好记

注意这些名字都**独特**——保证搜索时第一个就是你。`my-todo` 这种 generic 名字会淹没在 todo-related 包海里。如果你命名困难,可以用一个不相关但独特的词(`corfu` 就是这样)。

### 2.2 前缀

所有函数/变量用统一前缀 (包名)。这避免 namespace 冲突——Elisp 没有 namespace,所有符号共享一个空间。

```elisp
;; 包名: my-todo
my-todo-add
my-todo-done
my-todo-save
my-todo--internal-helper
my-todo-file (variable)
my-todo-list (variable)
my-todo-mode-hook
```

不要:

```elisp
add-todo                         ; 没前缀,可能冲突
mt-add                           ; 太短,不清晰
my-todo-add-todo                 ; 冗余
```

分析反例:

- `add-todo` 没有 namespace 前缀,如果别的包也叫 add-todo 就冲突
- `mt-add` 缩写太短,用户看不出 `mt` 是什么
- `my-todo-add-todo` 重复——`my-todo-add` 就够了,包名已经暗示了 "todo"

前缀的深层意义:**它是你的 namespace**。Emacs Lisp 没有 Common Lisp 的 package 系统(虽然 `cl-defpackage` 存在但很少用),所有符号全局可见,所以用前缀模拟 namespace。`my-todo-` 就是你的"包名空间"。

---

## 3. API 设计

API 是包的脸面。**API 设计的目标:让对的事情容易做,错的事情难做**。

### 3.1 公开 API vs 内部

```elisp
;; 公开 (用户/其他包调用):
my-todo-add                     ; 命令
my-todo-save
my-todo-mode                    ; minor mode

;; 内部 (实现细节):
my-todo--format-item
my-todo--write-file
my-todo--parse-line
```

内部用 `--`。

为什么区分公开和内部?因为你**承诺**维护公开 API(改它要保证兼容),但**保留改内部 API 的自由**(随时可以重构)。如果没有这个区分,你每次重构都破坏用户——你的包就变成"维护噩梦"。

`--` 的具体语义:

- `my-func`: 公开,用户可以调,承诺维护
- `my--func`: 内部,用户不应该调,随时可能改
- `my--category-func`: 更深层的内部,几乎肯定会被重构

社区工具(`checkdoc`、`package-lint`)会警告跨包调用 `--` 函数。这帮助用户识别"不该用"的 API。

### 3.2 自定义组

`defgroup` + `defcustom` 是 Emacs 的配置系统。用户通过 `M-x customize` 看到这些配置,改它们不需要写代码。

```elisp
(defgroup my-todo nil
  "Todo manager."
  :group 'tools
  :prefix "my-todo-")

(defcustom my-todo-file "~/todos.el"
  "File for storing todos."
  :type 'file
  :group 'my-todo)
```

逐部分解释:

- `defgroup` 创建一个 custom group——用户在 `M-x customize` 里看到这个 group 的名字。`:group 'tools` 把它放在 `tools` 大类下(也可以是 `convenience`、`editing` 等)。`:prefix "my-todo-"` 让 customize 自动包含所有 `my-todo-` 开头的变量。
- `defcustom` 类似 `defvar`,但加上了 `:type`(类型声明,告诉 customize 怎么显示编辑器)和 `:group`(归属)。

`:type` 是关键——它决定了 customize UI 怎么显示:

- `'file`: 文件选择器
- `'string`: 文本输入框
- `'integer`: 数字输入框
- `'(choice (const :tag "Option A" a) (const :tag "Option B" b))`: 单选
- `'(repeat string)`: 多个 string 的列表

**defcustom vs defvar**:`defvar` 给"不应该用户改"的内部状态,`defcustom` 给"用户应该调"的配置。新手常犯的错误是用 `defvar` 暴露本该 defcustom 的配置——用户就只能 `(setq ...)` 改,不能用 customize UI。

### 3.3 命令 API

命令是用户通过 `M-x` 或快捷键调用的函数。`interactive` 让 Elisp 函数变成命令。

```elisp
(defun my-todo-add (text priority tags)
  "Add a new todo.
TEXT is the todo description.
PRIORITY is an integer 1-5.
TAGS is a list of strings."
  (interactive
   (list (read-string "Todo: ")
         (read-number "Priority (1-5): " 3)
         (split-string (read-string "Tags (comma): ") "," t)))
  ...)
```

`interactive` 的几种形式:

- `(interactive)`: 不需要参数,直接调用函数
- `(interactive "sPrompt: ")`: 单个 string 参数(s 表示 string)
- `(interactive "nPrompt: ")`: 单个 number 参数(n 是 number)
- `(interactive (list ...))`: 多个参数,每个用不同 reader(最灵活)

`interactive` 形式只在**用户调用**时执行——`M-x my-todo-add` 会 prompt。**程序化调用**(别的代码 `(my-todo-add "foo" 3 nil)`)跳过 interactive,直接传参。所以一个函数既能交互用又能编程用。

这是 Emacs API 设计的精髓——**命令和函数是同一个东西**,只是入口不同。用户交互调 vs 程序调,行为一致(除了 interactive 部分)。这让你的包既能给用户用,也能给其他包编程用——双倍价值。

### 3.4 函数 API

提供**编程接口**,让其他包能用:

```elisp
(defun my-todo-add-internal (text priority tags)
  "Add todo programmatically.
Returns the new item."
  ...)

(defun my-todo-get-all ()
  "Return list of all todos."
  ...)

(defun my-todo-filter-by-tag (tag)
  "Return todos with TAG."
  ...)
```

注意命名:

- `add-internal` vs `add`:`add` 是命令(有 interactive),`add-internal` 是函数(无 interactive,无 prompt)。这种"双 API"模式很常见——内部用 `-internal`,外部用主名。
- `get-all`、`filter-by-tag`:predicate-like 命名,清晰说明功能。

为什么函数 API 重要?因为**生态系统的核心是组合**。你的 todo 包如果能被 org 包调用,用户就能用 org UI 操作 todo——你的用户群瞬间翻倍。如果你只暴露命令(`M-x`),其他包无法集成。

其他包:

```elisp
(let ((todos (my-todo-get-all)))
  (dolist (todo todos)
    (my-process todo)))
```

他们调你的 `get-all` 拿数据,自己处理。这种"数据 API + UI API 分离"是 Unix 哲学的核心——你的包做好一件事(get/manage todos),其他包组合它。

---

## 4. 状态管理

状态是 bug 的主要来源。Emacs 的状态有几种类别:全局、buffer-local、闭包、struct——选对类别,bug 减半。

### 4.1 全局状态

```elisp
(defvar my-todo-list nil
  "Current list of todos.")

(defvar my-todo-file "~/.emacs.d/todos.el"
  "File for storage.")
```

全局状态是默认值。优点:简单,任何地方都能访问。缺点:**隐式依赖**——任何代码都能改 `my-todo-list`,debug 时不知道谁改的。

实践建议:全局状态**最小化**——只放真的需要全局的(如缓存、配置)。改全局状态前 `(when (boundp 'my-todo-list) ...)` 防止未初始化。用 `defvar` 而不是 `setq`——`defvar` 只在未绑定时赋值,不覆盖用户配置。

### 4.2 buffer-local

```elisp
(defvar my-todo-display-mode nil)
(make-variable-buffer-local 'my-todo-display-mode)
```

buffer-local 让每个 buffer 有独立的变量值——A buffer 的 `my-todo-display-mode` 不影响 B buffer。这是 mode 状态的标准做法。

`make-variable-buffer-local` 把变量标记为"每个 buffer 独立"。从此 `(setq my-todo-display-mode t)` 在 buffer A 里设,不影响 buffer B。什么时候用 buffer-local?**任何和 buffer 相关的状态**:当前 mode 是否启用、buffer 特有的设置、临时状态(光标位置)。

不用 buffer-local 的反例:你把 mode 状态放在全局变量,用户在 buffer A enable mode,切到 buffer B 发现 mode 还"启用"——但 B 不该启用。这是常见的状态管理 bug。

### 4.3 闭包

闭包是 lexical binding 下的"私有状态"——比全局变量更安全。

```elisp
(defvar my-todo-cache
  (let ((data (make-hash-table :test 'equal)))
    (lambda (op &rest args)
      (pcase op
        ('get (gethash (car args) data))
        ('put (puthash (car args) (cadr args) data))))))

(funcall my-todo-cache 'put "key" "value")
(funcall my-todo-cache 'get "key")
```

这段代码做了什么:`let` 创建局部 `data`(hash table),返回一个 lambda 捕获 `data`。lambda 通过 `op` 参数决定操作。**`data` 对外部不可见**——只有通过 lambda 才能访问。

闭包的好处:**封装**。`data` 不会被外部代码意外改坏。这是"对象导向"在 Elisp 里的实现(用闭包模拟 object)。劣势:复杂,不如 cl-defstruct 直观。

### 4.4 cl-defstruct

`cl-defstruct` 是 Common Lisp 风格的 struct——比纯 list 更结构化、更类型安全。

```elisp
(require 'cl-lib)

(cl-defstruct my-todo-item
  text priority tags done)

(setq item (make-my-todo-item :text "Buy milk" :priority 3))
(my-todo-item-text item)
(setf (my-todo-item-done item) t)
```

`cl-defstruct` 自动生成 constructor(`make-my-todo-item`)、accessor(`my-todo-item-text`)、predicate(`my-todo-item-p`)、copy(`copy-my-todo-item`)。这比纯 list `(list "Buy milk" 3 nil nil)` 好太多——类型安全、字段名明确、未来加字段不破坏现有代码。

什么时候用 struct?**任何"有几个字段的复合数据"**:Todo 项、配置项、解析结果。不用 struct 的反例:你用 list 表示 todo,然后用 `(nth 2 todo)` 取 tags。3 个月后你忘了 2 是 tags 还是 priority——bug 自此而生。struct 的 accessor 名字是自文档的。

---

## 5. Hook 设计

Hook 是 Emacs 包之间通讯的标准机制。**好的包提供 hook,让用户和其他包能扩展**。

### 5.1 自定义 hook

```elisp
(defvar my-todo-added-hook nil
  "Run after a todo is added. Receives the new item.")

(defvar my-todo-saved-hook nil
  "Run after todos are saved.")

(defun my-todo-add-internal (...)
  ...
  (run-hook-with-args 'my-todo-added-hook item))
```

`defvar` 定义 hook(普通变量,初始 nil)。`run-hook-with-args` 在恰当时机触发——所有挂在 hook 上的函数会被调用,参数是 `item`。

别人:

```elisp
(add-hook 'my-todo-added-hook
          (lambda (item)
            (message "Added: %s" (my-todo-item-text item))))
```

这种"挂钩 + 扩展"模式是 Emacs 生态的核心。你的包不知道用户会怎么扩展,但你提供 hook,用户就能集成——比如有人写"todo 加到 org agenda"的 bridge,完全不用改你的代码。

### 5.2 hook 时机

Emacs 社区的 hook 命名约定,反映"事件时机":

- `before-X-hook`: 事件前——用户可以"修改输入"或"取消操作"
- `after-X-hook`: 事件后——用户可以"响应变化"
- `X-functions`: 替代默认行为——返回 non-nil 阻止默认

举例说明区别:

- `my-todo-before-add-hook`:todo 加之前触发。用户可以在这里 transform 输入(如 trim 空格),或 throw error 阻止加
- `my-todo-added-hook`:todo 加之后触发。用户可以在这里 update UI、通知、log
- `my-todo-add-functions`:替换 add 行为。返回 non-nil 跳过默认 add——用户可以完全自定义 add 逻辑

理解这三层,你的包就从一个"功能"变成一个"平台"。用户能在每个关键点插入逻辑,你的包成为可扩展的核心。

---

## 6. 错误处理

错误处理决定用户体验。**好的错误处理:用户错→清晰提示;系统错→优雅降级**。

### 6.1 用户错误

```elisp
(defun my-todo-delete (n)
  (interactive "nIndex: ")
  (if (>= n (length my-todo-list))
      (user-error "Index out of range")
    (setq my-todo-list (delq (nth n my-todo-list) my-todo-list))
    (my-todo-refresh)))
```

`user-error` 不进 debugger (用户错误,不是 bug)。

`user-error` vs `error` 的区别很关键:

- `user-error`:用户操作错误(选了不存在的 index)。Emacs 显示在 echo area,不进 debugger。`debug-on-error` 不触发。
- `error`:程序错误(不该发生的事)。进 debugger。`debug-on-error` 触发。

新手常犯的错误:用 `error` 报用户错。结果是用户每次 typo 都看到 backtrace——非常难用。**铁律:用户错误用 `user-error`,程序 bug 用 `error`**。

### 6.2 系统错误

```elisp
(defun my-todo-load ()
  (condition-case err
      (with-temp-buffer
        (insert-file-contents my-todo-file)
        (setq my-todo-list (read (current-buffer))))
    (file-error
     (message "Cannot load todos: %s" (cadr err))
     nil)))
```

`condition-case` 是 Emacs 的 try/catch:第一个 form 是 try body,每个 `(ERROR-TYPE HANDLER)` 是 catch clause——特定 error type 时调 handler。handler 接收 `err`,是 `(ERROR-SYMBOL . DATA)` 的 list。

上面代码捕获 `file-error`(文件不存在、权限问题等),显示友好消息,返回 nil(优雅降级)。**没有 condition-case,文件丢失会让用户看到 backtrace**——不可接受。

错误处理层次:用户错用 `user-error`(不进 debugger);可恢复系统错用 `condition-case`(优雅降级);不可恢复用 `error`(进 debugger 让用户报告)。每一层有合适的处理方式。不要"用 condition-case 吞掉所有 error"——这会掩盖真正的 bug。

---

## 7. 文档

文档不是事后补充——是 API 设计的一部分。**函数没有 docstring 就没设计完**。

### 7.1 docstring

每个 public 函数必须有 docstring:

```elisp
(defun my-todo-add (text priority tags)
  "Add a new todo item.

TEXT is the description (string).
PRIORITY is an integer 1-5.
TAGS is a list of strings.

Returns the new item."
  ...)
```

docstring 的规则:

- **第一行**简洁(~70 字符),总结功能——显示在 `M-x` 的 echo area
- **空行后**详细描述
- 参数逐一说明(类型、语义)
- 复杂函数加 `\(fn ARG1 ARG2)` 显式签名(byte-compiler 帮你检查)

**docstring 是契约**。你写了 `PRIORITY is an integer 1-5`,就承诺接受 1-5。如果你后面改成 0-10,就是 breaking change。所以 docstring 要准确,不要写"差不多"。

### 7.2 Commentary

`;;; Commentary:` 是包级别的长描述:

```elisp
;;; Commentary:
;; My Todo is a simple todo manager.
;;
;; Usage:
;;   M-x my-todo RET
;;
;; Features:
;;   - Add/remove todos
;;   - Tags and priorities
;;   - Persistent storage
```

`M-x finder-commentary` 显示这个。用户可以用它快速了解你的包——比 README 轻(用户不用切到浏览器)。**commentary 应该是 README 的浓缩版**——主要功能、用法、特性。

### 7.3 README

README 是 GitHub 看到的门面——决定用户是否装你的包。

```markdown
# my-todo

Simple todo manager for Emacs.

## Installation

```elisp
(use-package my-todo
  :ensure t
  :bind ("C-c t" . my-todo))
```

## Usage

| Key | Action |
|-----|--------|
| a | Add |
| d | Done |
| x | Delete |
| s | Save |
| q | Quit |

## Configuration

```elisp
(setq my-todo-file "~/todos.el")
```

## License
GPL v3+
```

README 的核心:**Installation 必须简单**。用户 clone 或 `package-install` 后 30 秒内能用。如果你的安装步骤要 5 步,大部分用户在 step 2 就放弃了。详细 README 见 `docs-texinfo.md`。

---

## 8. 实战: 设计一个 snippet 包

把上面的原则应用到一个具体例子。

### 8.1 选题

`yasnippet` 强大但复杂。
写一个极简的 my-snippet:
- 文件式 snippet (每个 snippet 一个文件)
- 触发 keyword + TAB
- 简单 `$1` `$2` placeholder

分析选题:符合 YAGNI(只做核心)、Single Responsibility(只做 snippet 展开)、Convention over Configuration(snippet 文件按目录组织,不配置)。

### 8.2 API

设计 API 时先想"用户和别的包会怎么调":

```elisp
(my-snippet-load-dir PATH)
(my-snippet-expand NAME)
(my-snippet-add NAME CONTENT)
```

三个函数,每个职责单一:`load-dir` 加载、`expand` 展开、`add` 添加。其他都是内部。

### 8.3 命令

```elisp
M-x my-snippet-insert
M-x my-snippet-edit
```

命令是 UI 层,内部调 API。`insert` 调 `expand`,`edit` 让用户改 snippet 文件。

### 8.4 文件格式

`~/.config/emacs/snippets/python/def.snippet`:

```
def ${1:name}(${2:args}):
    ${3:pass}
```

这是 yasnippet 兼容格式——好处是用户从 yasnippet 迁移时 snippet 文件不用改。**遵循行业标准 = 减少迁移成本**。

---

## 9. 创造性用法

把上面的设计原则用在新场景:

1. **逆向工程别人的 API**:看 org、magit 的公开函数,分析它们的 `defcustom` / hook 设计——偷学
2. **API 设计 review**:写完 API 后,假装自己是用户用 5 个场景调它——哪里别扭就改
3. **mock user 测试**:让朋友看你的 README,5 分钟能不能装上——不能就改进
4. **API evolution tracking**:每发一个版本,记下"破坏了什么"——强迫自己想兼容性
5. **设计 review checklist**:写一份"设计 review"的清单(命名一致吗?API 边界清晰吗?hook 够吗?错误处理对吗?)——每次发版前过一遍

---

## 10. 自测

1. 公开和内部函数命名区别?
2. defgroup 干啥?
3. hook 命名约定?
4. user-error vs error?
5. Commentary vs README?

**答案**:
> 1. 公开 my-func;内部 my--func。工具会警告跨包调 `--`
> 2. 创建 custom group,可 customize。`:prefix` 自动包含前缀变量
> 3. X-hook(normal hook)或 X-functions(替换默认行为);before-/after- 区分时机
> 4. user-error 是用户错误,不 debug;error 是真正 bug,进 debugger
> 5. Commentary 在 .el 文件头(`M-x finder-commentary` 看);README 是 GitHub 主页

---

## 11. 下一步

进入 `ert-testing.md`。
