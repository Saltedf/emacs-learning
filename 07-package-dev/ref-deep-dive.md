# Ref Deep Dive: Lisp Reference (Module 7)

> 替代 `emacs-lispref-30.2/package.texi` + `tips.texi` + `compile.texi` + `debugging.texi` + `edebug.texi`

这个文件是 Module 7 的"开发者视角参考手册"——覆盖 Emacs Lisp Reference 里的五个关键章节。前面 `concept-anchor.md` 讲"用户怎么用包",这里讲"开发者怎么写包"。两份合起来就是包开发的完整图景。

这一节的内容比较密集,但你不需要一次读完。建议的用法:**先扫一遍知道有什么,等到 capstone 实战时遇到具体问题再回来查**。下面的每个小节都对应 Emacs Lisp Reference 的一个章节,你可以认为这是它们的"中文导读 + 实战点评"。

---

## 1. package.texi (开发者视角)

### 1.1 包结构

一个标准的 Emacs 包项目长这样:

```
my-package/
├── my-package.el           ; 主文件
├── my-package-pkg.el       ; descriptor (生成)
├── README.md
├── ChangeLog
├── LICENSE
└── test/
    └── my-package-test.el
```

每一行都有讲究。**主文件** `my-package.el` 是包的核心,文件名必须和包名一致(package.el 用文件名推导包名)。**pkg.el** 是 package descriptor——它声明包的元数据(版本、依赖),MELPA build 时自动生成,所以你**通常不手写**,放它在 `.gitignore` 里就行。**README.md** 是 GitHub 门面。**ChangeLog** 是版本历史(也可以用 `CHANGELOG.md` 替代)。**LICENSE** 是 license 文本(MELPA 要求显式文件)。**test/** 目录放测试,约定俗成叫 `test/` 或 `tests/`。

这种结构不是 Emacs 强制的——理论上你可以只有 `my-package.el` 一个文件就发布。但成熟的项目都遵循这个布局,因为:分离 test 和 production 代码、显式 license 文件让法律状态清晰、README 是社区通行做法。**遵循惯例 = 用户上手快**——用户 clone 你的 repo 就知道去哪找什么。

### 1.2 header

每个 `.el` 文件开头必须有标准 header。这是 Emacs 包生态最严格的约定之一,package.el 从这里读元数据。

```elisp
;;; my-package.el --- Short description -*- lexical-binding: t; -*-

;; Copyright (C) 2026 Your Name

;; Author: Your Name <you@example.com>
;; Maintainer: Your Name <you@example.com>
;; Version: 1.0
;; Package-Requires: ((emacs "29.1") (cl-lib "0.5"))
;; Keywords: convenience tools
;; URL: https://github.com/you/my-package
;; SPDX-License-Identifier: GPL-3.0-or-later

;;; Commentary:
;; Longer description.

;;; Code:

;; ... 代码 ...

(provide 'my-package)
;;; my-package.el ends here
```

逐行解释:

- **第一行** `;;; my-package.el --- Short description` 是"magic header"。三个分号开头,文件名 + ` --- ` + 一行描述。Emacs 的 `load` 函数和很多工具靠这一行识别文件是不是 Elisp 源。`-*- lexical-binding: t; -*-` 是 file-local variable,告诉 Emacs 这个文件用 lexical binding——**强烈推荐**,lexical 比 dynamic 快、错误少、闭包正确,Emacs 24+ 的标准。
- **Copyright** 必须有,GPL 要求显式 copyright 声明。年份写当前年份或发布年份范围(如 `2024-2026`)。
- **Author** 是你的名字和 email。用户 `M-x find-library` 能看到。
- **Maintainer** 是当前维护者(可以和 Author 不同,比如原作者不维护了)。
- **Version** 是 SemVer 版本号。MELPA 从这读(覆盖 git tag)。
- **Package-Requires** 是依赖列表。这是 critical:每个依赖是 `(包名 "最低版本")` 的 list,Emacs 启动时检查这些依赖是否满足,不满足会拒绝激活你的包。
- **Keywords** 是分类标签,MELPA 用它做分类浏览。常见:convenience、tools、languages、comm、docs。
- **URL** 是项目主页。`M-x package-install` 完后,用户在 describe-package 里点 URL 跳到你的 repo——这是用户找文档、报 bug 的入口。
- **SPDX-License-Identifier** 是 SPDX 标准 license 标识符。`GPL-3.0-or-later` 表示"GPL v3 或更新版本",是 Emacs 包的标准选择。
- **Commentary** 是长描述。用户 `M-x finder-commentary` 看到这个。
- **Code** 标记代码开始,纯约定。
- **provide** 在文件末尾,提供 `(require 'my-package)` 的钩子。
- **ends here** 是"文件结束"标记,纯约定——但所有内置包都有,体现了 Emacs 的严谨。

省略任何一个 header 都可能导致 MELPA 拒绝你的 PR。最常见被拒原因:`Package-Requires` 漏了 `emacs` 的最低版本(很多包要求 `emacs "27.1"` 或更高,因为用了 27+ 的新特性)。

### 1.3 必填 header

汇总表:

| 字段 | 含义 |
|---|---|
| `Author` | 你的名字 + email |
| `Version` | 语义化版本 (1.0, 1.1.0) |
| `Package-Requires` | 依赖 (Emacs 版本, 其他包) |
| `Keywords` | 分类 (在 elpa 找) |
| `URL` | 项目主页 |

MELPA 会校验这些字段。漏 `Package-Requires` 是新手最常见的错误——你以为"我不依赖什么",但你**至少依赖 Emacs 的某个版本**。如果你用了 28 才有的 `thing-at-point` 新行为,就必须写 `(emacs "28.1")`,否则用户在 27 上装会崩溃,留下差评。

### 1.4 包加载机制

Emacs 包系统的核心创新是 **autoload**——延迟加载。理解 autoload 是理解 Emacs 启动速度的关键。

```
;;;###autoload
(defun my-func () ...)

;;;###autoload
(define-derived-mode my-mode ...)
```

`;;;###autoload` 是 Emacs 包系统的"魔法注释"——它告诉包管理器"这个 form 需要被 autoload"。具体来说,跑 `update-directory-loads` 或 package-install 时,Emacs 扫描这个注释,在 autoloads 文件里生成对应的 `(autoload 'my-func "my-package" nil t)`。

效果是这样的:用户 `(require 'my-package)` 时,my-package.el **不加载**。只有当用户调用 `M-x my-func` 时,Emacs 才把 my-package.el load 进来。这就是"延迟加载"——你的包不增加 Emacs 启动时间,但功能完全可用。

这个设计是 Emacs 包生态的基石。没有它,装 100 个包会让 Emacs 启动慢到无法用——所有包都会在启动时加载,几百 MB 的 Elisp 全部 require 进来,启动从 0.5 秒变成 30 秒。**所有认真的包都用 `;;;###autoload` 标记入口函数**——这是 Emacs 30 年生态演化出来的最佳实践。

哪些 form 应该 autoload?规则:**用户入口**才 autoload,内部辅助函数不 autoload。具体来说:

- `defun` 且 `interactive`——是命令,用户会 `M-x` 调用,autoload
- `define-derived-mode` / `define-minor-mode`——用户要 enable,autoload
- `defcustom` 的 top-level 变量——可以 autoload(让用户在 customize 里看到)
- `defvar` 大部分不 autoload
- 内部 `my-func--helper` 不 autoload

### 1.5 autoload 文件

跑 `M-x update-directory-loads` 或打包时,生成 `my-package-autoloads.el`:

```elisp
;;; my-package-autoloads.el --- autoloads
(autoload 'my-func "my-package" nil t)
(register-definition-prefixes "my-package" '("my-"))
;;;***
```

这个文件是自动生成的,不要手改。两行做两件事:`autoload` 注册延迟加载的函数,`register-definition-prefixes` 告诉 Emacs"凡是 `my-` 开头的未定义符号,先尝试从 my-package 找"——这是 completion framework(vertico/company)能补全你包内函数的基础。

`package-install` 时,这个文件自动加载到 user-init。所以用户在 init.el 不用写 `(require 'my-package)`,只要装了,autoloads 就在 load-path 里——这就是为什么装一个包"立刻能用",不用重启。

### 1.6 provide

```elisp
(provide 'my-package)
```

末尾必须有。让别人 `(require 'my-package)` 工作。

`provide` 是 Emacs 的 module 机制的核心——它把符号 `my-package` 注册到 `features` 列表,告诉 Emacs "这个 feature 已经加载"。`(require 'my-package)` 检查 `features`,如果有就跳过,没有就 load 文件再检查。这是"加载一次"语义的实现。

为什么这件事对你重要?因为**你的包里如果用了 `require`,Emacs 会按依赖顺序加载**。比如你的包 `(require 'cl-lib)`,启动时 cl-lib 先加载,然后你的包加载——这保证了 `cl-lib` 的函数在你包里可用。漏了 `provide` 会导致 `require` 一直 reload,引发各种诡异 bug(比如 `defvar` 被重置)。

---

## 2. tips.texi

### 2.1 编码风格

Emacs Lisp 有 40 年的编码风格传统。遵循这些风格不是教条——它让你的代码和其他 Emacs 包看起来一致,用户读起来熟悉,贡献者上手快。所有风格规则在 Emacs Lisp Manual 的 "Tips" 章节里。

```elisp
;; 函数名: 小写连字符
my-func

;; 变量: 同上
my-var

;; 内部: 双横线
my--internal

;; 特殊变量 (dynamic): 星号
*my-state*

;; 常量: 加号
+my-const+

;; keymap: -map 后缀
my-mode-map

;; hook: -hook 后缀
my-mode-hook

;; 自定义 group: -group 后缀 (但通常同名)
```

每一行背后都有理由:

- **连字符**: Emacs Lisp 的传统(不像 Python 用下划线)。连字符在 Lisp 里是合法标识符,读起来更顺(`my-func` 比 `my_func` 自然)。
- **双横线表示内部**: Emacs Lisp 没有 namespace,但 `--` 是社区约定——其他包不应该调 `my--internal`。一些工具(如 `checkdoc`)会警告跨包调用 `--` 函数。
- **星号 dynamic 变量**: 老式约定,表示"会被 `let` 改的 special variable"。现在用 `defvar` 自动是 special,星号是历史遗留。
- **常量加号**: 类似 Common Lisp 习惯。Emacs 内置例子:`+float-pi+`、`+latin-iso8859-1-table+`。
- **keymap/hook/group 后缀**: 让人一眼看出类型。`(assq 'my-mode-map ...)` 一看就知道找 keymap。

### 2.2 度量

代码质量的硬指标:

- **函数行数**: < 50 行最好。Emacs Lisp 不擅长长函数,因为缺少 return/break,长函数嵌套深。如果函数超 50 行,大概率该拆。
- **圈复杂度**: 低 (少嵌套)。每个 `if`/`cond`/`while` 增加复杂度。用 early return 模式降低(Emacs 29+ 有 `when-let`、`if-let` 帮忙)。
- **注释**: 解释 why,不是 what。代码已经说了 what(`(setq x 5)` 就是设 x 为 5),注释要解释**为什么**(`;; 设置 x 为 5 是因为 foo 假设它是默认值`)。

### 2.3 命名约定

Emacs Lisp 的 prefix/suffix 约定,是社区 40 年沉淀的"语义协议"。看到名字就能猜功能。

- `with-X`: macro,设上下文
  - `with-current-buffer`, `with-temp-file`, `with-output-to-string`
  - 这些 macro 通常 `(let ... (unwind-protect (progn ,@body) cleanup))` 模式,保证清理
- `save-X`: macro,保存恢复
  - `save-excursion`, `save-restriction`, `save-match-data`
  - 和 with- 区别:save- 偏向"保存 buffer 状态后恢复",with- 偏向"创建上下文"
- `X-p`: predicate (返回 bool)
  - `stringp`, `numberp`, `bufferp`
  - `-p` 后缀是 Lisp 古老传统(最早 Scheme 用),Emacs Lisp 全盘继承
- `X-or-Y`: 有 default
  - `assoc-default`, `eq-or-`
- `define-X`: macro,定义
  - `define-minor-mode`, `define-derived-mode`
- `make-X`: constructor
  - `make-sparse-keymap`, `make-hash-table`

遵循这些约定让你的 API 更可猜。用户看到 `my-foo-p` 就知道是 predicate,看到 `with-my-context` 就知道是 macro。**可猜性是 API 易用性的核心**——用户不用查文档就知道怎么用。

### 2.4 性能提示

Elisp 慢——它是解释型(虽然可以 byte-compile),且没有现代优化。所以性能细节重要。

```elisp
;; 慢:
(while ... (setq result (append result (list x))))  ; 每次 O(n)

;; 快:
(while ... (push x result))                ; 每次 O(1)
(setq result (nreverse result))            ; 最后 O(n)
```

`append` 是 O(n) 因为它要复制整个 list。所以循环里 append 是 O(n²)——list 长一点就崩。`push` 是 O(1)(往前加 cons cell),最后 `nreverse` 一次性反转——O(n)。**几乎所有的 list 累积都应该用 push+nreverse 模式**。这是 Lisp 老兵的肌肉记忆。

```elisp
;; 慢:
(let ((s (buffer-substring ...)))
  (length s))                              ; 复制 string

;; 快:
(- end start)                              ; 直接算
```

`buffer-substring` 复制 buffer 内容到新 string——大 buffer 时是性能杀手。如果你只需要长度,直接 `(- end start)`,零分配。

更多性能诀窍:

- `eq` 比 `equal` 快(eq 比指针,equal 比内容)
- `aref` 比 `elt` 快(aref 是 array 专用,elt 要 type check)
- `cl-loop` 比 `while` 快(cl-loop 编译成更高效的字节码)
- 避免大 list 用 `mapcar`——用 `dolist` + 累积器
- 字符串拼接不要 `(concat a b c ...)`,用 `with-output-to-string` 或 `mapconcat`

### 2.5 byte-compile warning

byte-compile 是把 Elisp 源码编译成 bytecode(Emacs VM 执行),大幅提速且帮你抓 bug。warnings 不是噪声——它们指出真正的 bug。

常见 warning 和修复:

- **`free variable`**: 你用了未 defvar 的变量。修复:`defvar` 它(即使是 `(defvar my-var nil)`)。意义:dynamic 变量必须声明,否则 byte-compiler 假设它是 dynamic,可能引入微妙 bug。
- **`unused lexical var`**: 你 `let` 绑了变量但没使用。修复:删掉绑定。意义:多半是你 typo 了变量名(绑了 `foo`,用了 `bar`),byte-compiler 帮你抓出来。
- **`interactive-only`**: 这个函数只该交互用,不该程序调用。例:`previous-line`——programmatic 用 `(forward-line -1)`。修复:换成非交互版本。
- **`call to XXX with wrong number of arguments`**: 你调函数参数数错了。修复:看 docstring 改。

**目标:零 warning**。MELPA 评审会要求这个。

---

## 3. compile.texi

### 3.1 byte-compile

byte-compile 是把 `.el` 变成 `.elc`——后者是 bytecode,加载快 5-10x。命令行批量编译:

```bash
emacs --batch -L . -f batch-byte-compile my-package.el
```

`--batch` 让 Emacs 不进交互模式,跑完就退出。`-L .` 把当前目录加入 load-path(让 require 能找到)。`-f batch-byte-compile` 调用 batch-byte-compile 函数,参数是文件名。

或用 Makefile 自动化:

```makefile
ELS = $(wildcard *.el)
ELCS = $(ELS:.el=.elc)

all: $(ELCS)

%.elc: %.el
	emacs --batch -L . -f batch-byte-compile $<

clean:
	rm -f $(ELCS)
```

这个 Makefile 做三件事:`make` 编译所有 `.el`,`make clean` 清理,`%.elc: %.el` 定义 pattern rule——任何 `.elc` 依赖同名 `.el`,通过 `emacs --batch` 编译。Makefile 的好处是**增量编译**:只重编改过的文件。

CI 里你应该跑 `make` 检查零 warning——这是 MELPA 的硬性要求。

### 3.2 native comp

Emacs 28+ 引入 **native compilation**(gccemacs 项目合并)——把 bytecode 进一步编译成机器码,提速 2-4x。这对终端用户透明(他们的 Emacs 自动 native compile),但开发者应该测试自己的包在 native comp 下工作正常:

```bash
emacs --batch -L . \
  --eval "(let ((native-comp-async-query-on-exit nil)) \
            (native-compile-async \".\" 'recursively) \
            (while (native-comp-async-running-p) (sit-for 0.1)))"
```

这段代码:启用 native comp 的异步模式,递归编译当前目录,然后循环 `sit-for` 等待编译完成。`native-comp-async-query-on-exit nil` 防止 Emacs 在编译时问"要不要退出"。

native comp 大部分时候和 byte-compile 一样,但有些边缘 case 会触发问题——比如你用了 `eval-and-compile` 不当,或者 macro 展开依赖运行时状态。**CI 应该同时跑 byte-compile 和 native comp**,确保两种模式都过。

---

## 4. debugging.texi + edebug.texi

(Module 6 W4 已深入)

这里不重复,只补两个开发者高频用的工具:

1. **`toggle-debug-on-error`**: 开启后,任何 error 都进入 debugger 显示 backtrace。开发时几乎必备。
2. **`toggle-debug-on-quit`**: 开启后,`C-g` 也进 debugger。帮你定位"为什么这么慢"——按 C-g 看它卡在哪里。
3. **edebug**: 进入函数,逐 sexp 执行。`C-u C-M-x`(带前缀参数 eval-defun)激活。比 `message` debug 高效 10 倍。
4. **`trace-functions`**: 跟踪函数调用,记录参数和返回值。适合"这函数被谁调用、传了什么"的场景。

---

## 5. 自测

1. 包 header 必填什么?
2. `;;;###autoload` 干啥?
3. 内部函数命名约定?
4. 怎么 byte-compile 整个目录?
5. 怎么 fix "free variable" warning?

**答案**:
> 1. Author, Version, Package-Requires, Keywords, URL——MELPA 会校验
> 2. 标记下一个 form 为 autoload——让用户调用时才加载,不增加启动时间
> 3. 双横线 `my--internal`——社区约定,工具会警告跨包调用
> 4. `emacs --batch -f batch-byte-compile *.el`,或用 Makefile pattern rule
> 5. `defvar` 它——dynamic 变量必须声明,否则 byte-compiler 假设 dynamic 可能引入 bug

---

## 6. 下一步

进入 `package-design.md`。
