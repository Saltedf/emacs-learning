# ERT Testing

> 学完这个文件,你能为包写完整的测试套件

## 0. 为什么需要测试 (第一性原理)

在讲 ERT 语法前,先想清楚**为什么写测试**——不理解动机,测试就变成走过场。

第一性原理推导:

1. **代码会变**——没有代码写完就不动的,你会改 bug、加功能、重构
2. **变可能引入 bug**——每次改 10 行,可能默默破坏了某个边缘 case
3. **bug 影响用户**——用户报告 bug 时,通常已经造成损失
4. **需要"自动检查"**——人脑无法预测每次改动的所有影响,需要机器帮验证
5. **检查 = 运行 + 验证**——运行代码,验证输出对不对
6. **把验证编码成可重复的形式 → 测试**——把"输入 X 应该输出 Y"写成代码
7. **测试应该可批量跑 → framework (ERT)**——不只是写测试,要能批量、自动化地跑
8. **CI 自动跑 → 持续集成**——每次 git push,CI 自动跑测试,bug 当天暴露

这个推导链告诉你:**测试不是"写完代码做的事",是"贯穿开发过程的事"**。最佳实践是 TDD (Test-Driven Development):先写测试(描述期望行为),再写实现(让测试过)。TDD 让你的代码"为测试设计",天然解耦、可测。

ERT (Emacs Regression Test) 是 Emacs 内置的测试 framework,Emacs 24+ 自带。"Regression"是关键词——它的核心价值不是"测试新功能",而是"防止旧功能回归"。每改一次代码,跑一次 ERT,确保没破坏历史功能。

---

## 1. ERT 基础

### 1.1 安装

ERT (Emacs Regression Test) Emacs 24+ 内置。

```elisp
(require 'ert)
```

`(require 'ert)` 加载 ERT library。Emacs 24+ 内置,不用装。把这一行放在 test 文件开头——和你的生产代码分离,test 文件单独 require。

### 1.2 ert-deftest

ERT 的核心 macro 是 `ert-deftest`,它定义一个测试。

```elisp
(ert-deftest NAME DOCSTRING KEYWORDS... BODY...)
```

它的结构类似 `defun`,但语义不同——`ert-deftest` 定义的不是一个可调用函数,而是一个"测试 case"。NAME 是测试名(惯例:`包名-测试场景-test`),DOCSTRING 描述测试意图,KEYWORDS 是可选的元数据(如 `:tags`、`:timeout`),BODY 是测试代码。

```elisp
(ert-deftest my-test ()
  "Test addition."
  (should (= (+ 1 2) 3)))
```

这个测试做一件事:验证 `(+ 1 2)` 等于 3。如果等于,测试 pass;不等于,`(should ...)` 抛 error,测试 fail。

### 1.3 should / should-not

ERT 的 assertion 几个变种:

```elisp
(should CONDITION)              ; CONDITION 必须 non-nil
(should-not CONDITION)          ; CONDITION 必须 nil
(should-error FORM)             ; FORM 必须报错
(should-error FORM :type 'error)  ; 特定 error type
(should-error FORM :exclude 'void-variable)  ; 排除某些
```

逐个看:

- `should` 是最常用的——期望表达式为 true。等价于 `assert` 但 ERT 报告更友好
- `should-not` 是反——期望 false
- `should-error` 期望 form 抛 error。用来测"错误处理对不对"——比如输入非法应该报错
- `should-error` 的 `:type` 限定 error 类型——防止"碰巧报了别的 error 也算 pass"
- `:exclude` 排除某些 error 类型——比如期望 `wrong-type-argument` 但不期望 `void-variable`(后者说明 typo 了符号名)

### 1.4 例子

实际例子最能说明用法:

```elisp
(ert-deftest my-square-test ()
  (should (= (my-square 5) 25))
  (should (= (my-square -3) 9))
  (should (= (my-square 0) 0)))

(ert-deftest my-sum-list-test ()
  (should (= (my-sum-list '(1 2 3)) 6))
  (should (= (my-sum-list '()) 0))
  (should (= (my-sum-list '(5)) 5)))

(ert-deftest my-error-test ()
  (should-error (my-func-bad-input nil)
                :type 'wrong-type-argument))
```

`my-square-test` 测三个 case:正数、负数、零。这是"边界 + 中心"——5 是典型 case,-3 测负数,0 测边界。**好测试覆盖正常 + 边界**。

`my-sum-list-test` 测三个 case:正常 list、空 list、单元素 list。空 list 是常见的边界——很多 bug 在空输入时才暴露。

`my-error-test` 测错误处理——传 nil 给不该接受 nil 的函数,应该报 `wrong-type-argument`。这保证你的错误处理不被无意改坏。

### 1.5 跑测试

写完测试要能跑。ERT 提供两种方式:交互和编程。

```elisp
;; 交互:
M-x ert-run-tests-interactively RET
M-x ert RET                    ; 选 "all"
M-x ert RET "^my-"             ; 正则匹配
M-x ert RET my-specific-test   ; 特定

;; 编程:
(ert "my-")                    ; 跑所有 "my-" 开头的
(ert-run-tests-interactively t)
(ert-run-tests-batch-and-exit)
```

交互模式打开一个 `*ert*` buffer,列出所有测试和结果(. pass, F fail)。编程模式 `ert-run-tests-batch-and-exit` 在终端打印结果并退出——这是 CI 用的,因为它让 Emacs 进程退出码反映测试结果(0 = 全过,1 = 有失败)。

`M-x ert RET "^my-"` 的正则匹配很有用——你只想跑自己包的测试,不想跑依赖包的。惯例:test 名都以 `包名-` 开头。

---

## 2. 测试模式

### 2.1 准备-执行-验证 (AAA)

AAA (Arrange-Act-Assert) 是经典测试模式。

```elisp
(ert-deftest my-todo-add-test ()
  ;; Arrange
  (let ((my-todo-list nil))
    ;; Act
    (my-todo-add-internal "test" 1 nil)
    ;; Assert
    (should (= 1 (length my-todo-list)))
    (should (equal "test" (car (car my-todo-list))))))
```

三个阶段:

- **Arrange**: 准备测试环境——重置状态、初始化数据
- **Act**: 调用要测的代码
- **Assert**: 验证结果

上面例子里 `let ((my-todo-list nil))` 是 Arrange(隔离状态),`my-todo-add-internal` 是 Act,两个 `should` 是 Assert。**AAA 让测试结构清晰**,读测试时一眼看出"做了什么、测了什么"。

### 2.2 多 case (数据驱动)

如果一个函数有多种输入 case,不要写 10 个 `ert-deftest`——用一个 dolist 循环。

```elisp
(ert-deftest my-square-test ()
  (dolist (case '((0 . 0)
                  (1 . 1)
                  (2 . 4)
                  (3 . 9)
                  (-4 . 16)))
    (should (= (my-square (car case)) (cdr case)))))
```

这测了 5 个 case,但代码只有 4 行。如果某个 case 失败,ERT 报告"在 dolist 第 N 个 case 失败"——和单独写测试一样可诊断,但代码更紧凑。

**数据驱动测试的核心优势**:加 case 不用改代码结构,只改 data list。鼓励你写更多 case,因为成本低。

### 2.3 :tags

`:tags` 给测试打标签,可以按标签选择跑哪些:

```elisp
(ert-deftest my-slow-test ()
  :tags '(slow)
  (sleep-for 5)
  ...)

;; 只跑 fast:
(ert '("fast" t))              ; 不,语法不同
(ert-tagged 'fast)             ; 跑 tagged 'fast'
```

slow 测试通常跑得久(如网络、IO),日常开发不跑,只在 CI 跑。`:tags '(slow)` 标记后,`(ert-tagged 'fast)` 跳过 slow 测试,快速反馈。

### 2.4 :timeout

测试可能挂起(如死循环、网络阻塞)。`:timeout` 防止一个测试拖垮整个 suite:

```elisp
(ert-deftest my-test ()
  :timeout 5                   ; 5 秒后超时
  ...)
```

5 秒后 ERT 强制 fail 这个测试,继续跑其他。CI 里这是必备——否则一个挂起的测试会让 CI 永远跑不完,被 GitHub Actions 自动取消。

### 2.5 beforeEach / afterEach

ERT 没有原生 setup/teardown(这是 ERT 的弱点,作者认为"用 macro 表达更清晰")。用 let + macro:

```elisp
(defmacro my-with-test-buffer (&rest body)
  `(with-temp-buffer
     (insert "test data")
     ,@body))

(ert-deftest my-test ()
  (my-with-test-buffer
   (should (string= "test data" (buffer-string)))))
```

这个 macro `my-with-test-buffer` 在每个测试里展开成"创建 temp buffer + 插入 test data + 跑 body"。每个用它的测试都自动有"准备好的 buffer",不用每个测试重复 setup 代码。

为什么 ERT 不提供内置 setup/teardown?设计哲学:**测试应该是独立的,不依赖全局状态**。如果你发现自己写大量重复 setup,说明你的代码耦合太重——应该重构让代码更可测。这是测试驱动设计的反作用力。

---

## 3. 测试 buffer 操作

Emacs 大部分代码操作 buffer——所以测 buffer 是 ERT 的高频场景。

### 3.1 with-temp-buffer

`with-temp-buffer` 创建一个临时 buffer,body 跑完后自动 kill。这是测试的"沙箱"——不污染用户当前 buffer。

```elisp
(ert-deftest my-buffer-test ()
  (with-temp-buffer
    (insert "hello")
    (should (string= "hello" (buffer-string)))))
```

这个测试在 temp buffer 里插 "hello",验证 buffer 内容是 "hello"。`buffer-string` 返回整个 buffer 内容。

### 3.2 测试 mode

测试 major mode 是否正确设置 buffer 状态:

```elisp
(ert-deftest my-mode-test ()
  (with-temp-buffer
    (my-mode)
    (should (eq major-mode 'my-mode))
    (should (eq (char-syntax ?_) ?w))))  ; syntax table
```

启用 `my-mode` 后,`major-mode` 应该是 `my-mode`。`(char-syntax ?_)` 返回 `_` 字符的 syntax class——`?w` 表示"word constituent"。这测了 syntax table 是否正确设置。

mode 测试的关键点:验证 mode 的**所有可见效果**——major-mode 变量、keymap、syntax table、font-lock、hook 触发。

### 3.3 测试 font-lock

font-lock(语法高亮)是 mode 的重要部分。测试它比较微妙:

```elisp
(ert-deftest my-font-lock-test ()
  (with-temp-buffer
    (my-mode)
    (insert "function foo() {}")
    (font-lock-fontify-buffer)
    (goto-char (point-min))
    (re-search-forward "function")
    (should (eq (get-text-property (match-beginning 0) 'face)
                'font-lock-keyword-face))))
```

流程:

1. 启用 mode
2. 插入代码
3. **显式调用 `font-lock-fontify-buffer`**——font-lock 默认是 deferred 的,不立即应用
4. 跳到 "function" 的位置
5. 检查那个位置的文字属性 `face` 是否是 `font-lock-keyword-face`

font-lock 测试是 ERT 最 tricky 的部分——因为 font-lock 依赖很多 buffer 状态(jit-lock 模式、deferred fontification)。**先 `font-lock-fontify-buffer` 强制同步**,否则可能测到 font 还没应用的状态。

### 3.4 测试文件

测试文件 I/O 必须用临时文件 + unwind-protect 清理:

```elisp
(ert-deftest my-file-test ()
  (let ((tmp-file (make-temp-file "test-" nil ".el")))
    (unwind-protect
        (progn
          (with-temp-buffer
            (insert "test data")
            (write-file tmp-file))
          (should (file-exists-p tmp-file))
          ...)
      (when (file-exists-p tmp-file)
        (delete-file tmp-file)))))
```

`make-temp-file` 创建唯一临时文件(避免测试间冲突)。`unwind-protect` 确保 body 报错时 cleanup 仍执行——否则测试失败后留下临时文件,污染文件系统。

**测试不能留垃圾**。文件、buffer、进程、网络连接——所有外部资源都要 cleanup。这是好测试的基本原则。`unwind-protect` 是 Emacs 的 try/finally,标准用法。

---

## 4. Mocking

Mocking 是测试隔离的关键——你的代码可能依赖外部(时间、网络、其他包),测试要"假装"这些依赖,才能可重复。

### 4.1 用 let 隔离

最简单的 mock 是 `let`——临时改一个变量的值。

```elisp
(ert-deftest my-test ()
  (let ((my-var 5))
    (should (= (my-func) 5))))
```

`let` 在 body 内把 `my-var` 设为 5,body 结束自动恢复。这是"变量 mock"——适合 mock 配置、状态。

`let` 的隔离是 Elisp 测试的基石。所有 dynamic variable 都能这样 mock——而 Elisp 大量使用 dynamic variable(配置、状态)。所以 Elisp 测试比想象的简单。

### 4.2 cl-letf (替换函数)

`cl-letf` 可以**临时替换函数定义**——这是真正的 function mock。

```elisp
(cl-letf (((symbol-function 'current-time) (lambda () '(0 0 0 0))))
  (should (time-equal-p (current-time) 0)))
```

`(symbol-function 'current-time)` 拿到 `current-time` 的函数 cell。`cl-letf` 在 body 内把这个 cell 替换成你的 lambda——之后所有调用 `current-time` 都返回 `(0 0 0 0)`。body 结束恢复原函数。

这是测试时间相关代码的标准技巧——时间不可控,但你可以 mock `current-time` 让它"固定"。同理可以 mock `random`、`network 进程`、任何外部依赖。

### 4.3 临时改 buffer

`cl-letf` 还能改 buffer-local 状态:

```elisp
(cl-letf (((buffer-name) "test.txt"))
  (should (string= (buffer-name) "test.txt")))
```

`(buffer-name)` 不带参数返回当前 buffer 的名字。`cl-letf` 让这个调用返回 "test.txt"——你的代码以为在 test.txt 里,实际不在。这测了"代码对 buffer name 的反应"。

---

## 5. CI (GitHub Actions)

CI 让测试自动化——每次 git push,远程服务器自动 clone、跑测试、报告结果。这是包发布前的最后一道防线。

### 5.1 配置文件

CI 配置是 YAML,GitHub Actions 读 `.github/workflows/*.yml`:

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        emacs-version: ['27.2', '28.2', '29.4', '30.1']
    steps:
    - uses: actions/checkout@v3
    - uses: purcell/setup-emacs@master
      with:
        version: ${{ matrix.emacs-version }}
    - uses: actions/cache@v3
      with:
        path: ~/.emacs.d
        key: ${{ runner.os }}-emacs-${{ matrix.emacs-version }}
    - name: Install dependencies
      run: |
        emacs --batch --eval "(progn (setq package-archives '((\"gnu\" . \"https://elpa.gnu.org/packages/\") (\"melpa\" . \"https://melpa.org/packages/\"))) (package-refresh-contents) (package-install 'ert-runner))"
    - name: Run tests
      run: |
        emacs --batch -L . -L test \
              --eval "(require 'ert)" \
              -f ert-run-tests-batch-and-exit
```

逐部分解释:

- `on:` 触发条件——push 到 main 或任何 PR 时跑
- `matrix.emacs-version`: **测试矩阵**——同时跑 4 个 Emacs 版本,保证你的包跨版本兼容。这是 MELPA 评审会看的指标
- `actions/checkout`: clone 你的 repo
- `purcell/setup-emacs`: 在 runner 上装指定版本 Emacs(社区维护的 action)
- `actions/cache`: 缓存 `~/.emacs.d`,加速重复跑(避免每次重新下载依赖包)
- `Install dependencies`: 用 `emacs --batch` 装 `ert-runner`(可选,简化测试运行)
- `Run tests`: `-L . -L test` 把当前目录和 test 目录加 load-path;`-f ert-run-tests-batch-and-exit` 跑所有测试,退出码反映结果

CI 是包质量的硬指标。**没有 CI 的包,用户不敢装**——因为他们不知道你的代码现在是不是 build 通过。

### 5.2 ert-runner (简化)

`ert-runner` 是一个社区包,简化测试运行:

```bash
emacs --batch --eval "(package-install 'ert-runner)"
```

装上后,运行测试就是 `ert-runner` 一个命令——它会自动找 test 文件、跑、报告。但内部还是 ERT,只是 UI 友好。

或写自己 Makefile:

```makefile
test:
	emacs --batch -L . -L test \
	  --eval "(let ((load-prefer-newer t)) (require 'ert))" \
	  -l test/my-test.el \
	  -f ert-run-tests-batch-and-exit
```

```bash
make test
```

Makefile 的好处:**用户不必记 emacs 命令**。`make test` 一键跑——贡献者上手快。

### 5.3 覆盖率

`undercover` 包测代码覆盖率——多少行被执行。

```elisp
(require 'undercover)
(undercover "my-package.el")
```

把这两行放在 test 文件开头,跑测试时 undercover 收集每个函数被调用的情况。CI 跑测试后上传到 coveralls.io,生成覆盖率徽章加到 README。

**覆盖率是参考指标,不是目标**。100% 覆盖不代表没 bug(可能你测错了期望值),50% 覆盖不代表有 bug(可能 50% 是冷代码)。但覆盖率低说明"没测到的代码多",是 review 的起点。

---

## 6. 创造性用法

ERT 不只是写 `should` assertion——这些用法扩展了它的价值:

1. **属性测试 (Property-based testing)**: 类似 QuickCheck。生成大量随机输入,验证不变量。Elisp 没原生支持,但可以用 `cl-loop` 生成随机数据 + dolist 验证。例:测试 `my-reverse` 的属性"reverse 两次等于原 list",`dotimes (1000 ...) (let ((input (random-list))) (should (equal input (my-reverse (my-reverse input)))))`
2. **Fixture 模式**: 用 macro 表达 setup/teardown。如 `(my-with-temp-todos ...)` 自动建临时 todo 文件,body 跑完删除。把 fixture 抽成 macro,测试代码减半
3. **Mock with cl-letf**: 替换 `current-time`、`call-process`、`network-process` 等外部依赖,测试隔离
4. **Snapshot 测试**: 把"期望的 buffer 内容"存成文件,测试时比较实际 buffer 和 snapshot。font-lock、format 类包常用
5. **Integration 测试**: 不只测单个函数,测多个包协作。如你的 todo 包 + org-mode 集成,启动两个包验证交互
6. **Benchmark**: `ert-deftest` + `benchmark-run` 测性能。例:`(should (< (car (benchmark-run 1 (my-slow-func))) 0.1))`——验证函数跑得够快
7. **生成测试数据**: 用 generative 函数造随机输入,找出未发现的边界 case。如生成随机 string,测 `my-parse` 不崩
8. **Regression library**: 每次修 bug,先写 `ert-deftest` 重现,再修。让历史 bug 永远不再回归
9. **Smoke test**: 简单的"加载测试"——`(ert-deftest my-load-test () (should (require 'my-package)))`——验证包至少能加载

---

## 7. 实战练习

### Ex 7.1: 简单 test

```elisp
(require 'ert)

(ert-deftest addition-test ()
  (should (= (+ 1 2) 3))
  (should (= (+ -1 1) 0)))

(ert-deftest string-test ()
  (should (string= (upcase "abc") "ABC")))
```

第一个练习很简单——验证 ERT 基础工作。把这段保存为 `test/my-test.el`,然后 `M-x ert RET`。

### Ex 7.2: 数据驱动

```elisp
(ert-deftest square-test ()
  (dolist (case '((0 . 0) (1 . 1) (5 . 25) (-3 . 9)))
    (should (= (* (car case) (car case))
               (cdr case)))))
```

数据驱动模式——一个 dolist 跑 4 个 case。注意 case 是 `(input . expected)` 的 cons cell。

### Ex 7.3: 错误测试

```elisp
(ert-deftest divide-test ()
  (should-error (/ 1 0) :type 'arith-error))
```

测错误处理——`(/ 1 0)` 应该抛 `arith-error`。`:type` 限定 error 类型,防止"碰巧抛了别的 error 也算 pass"。

### Ex 7.4: buffer 测试

```elisp
(ert-deftest buffer-test ()
  (with-temp-buffer
    (insert "hello")
    (forward-word -1)
    (should (looking-at "hello"))))
```

测 buffer 操作——插入文本,移光标,验证位置。

### Ex 7.5: 测试你的包

给你的 my-todo 写 ≥ 5 个测试,覆盖:

- 核心 add/delete/get
- 边界(空 list、超范围 index)
- 错误处理(非法输入)

---

## 8. 自测

1. `ert-deftest` 怎么用?
2. `should` 和 `should-error` 区别?
3. 怎么跑所有测试?
4. CI 配置文件放哪?
5. 怎么 mock 一个函数?

**答案**:
> 1. `(ert-deftest NAME () DOC BODY...)`——类似 defun,但定义测试 case
> 2. should 期望 non-nil;should-error 期望抛 error(可 `:type` 限定)
> 3. `M-x ert RET` 交互;`(ert-run-tests-batch-and-exit)` 编程(CI)
> 4. `.github/workflows/*.yml`,GitHub Actions 自动读
> 5. `cl-letf (((symbol-function 'fn) (lambda ...)))`——临时替换函数 cell

---

## 9. 下一步

进入 `docs-texinfo.md`。
