# Concept Anchor + Exercises: Week 3

## Concept Anchor: Editor (custom.texi + package.texi)

Emacs 的"可配置性"是它区别于其他编辑器的核心。VS Code 有 settings.json,但 Emacs 有 **customize system**——一个统一的 UI,让你用图形界面 (而非 JSON) 配置所有 package。这让非程序员也能配置 Emacs。

### custom.texi
- Module 4 已学 user 视角的 customization
- 这周深入 defcustom 的 keyword (`:type`, `:set`, `:initialize` 等)

`defcustom` 的 keyword 决定 customize UI 怎么显示:
- `:type 'integer` → 数字输入框
- `:type 'string` → 文本框
- `:type '(choice (const :tag "Option A" a) (const :tag "Option B" b))` → 下拉菜单
- `:type '(repeat string)` → list of string
- `:type 'alist` → key-value 列表

`:set` 是"自定义 setter"——当用户在 customize 改值时,跑这个函数 (而非直接 setq)。用于需要"副作用"的变量 (如改后要重启某些服务)。

`:initialize` 是"初始化逻辑"——`'custom-initialize-default` (默认)、`'custom-initialize-reset`、`'custom-initialize-changed`。

### package.texi
- Module 4 已学用户视角 (M-x package-install)
- Module 7 会深入"写包"

package system 是 Emacs 25+ 内置的 package manager——基于 ELPA/MELPA 仓库。它通过 `package.el` 实现,核心 API:
- `package-install` 装
- `package-refresh-contents` 刷新索引
- `package-list-packages` 列出所有
- `package-archives` 配置源 (GNU ELPA、MELPA、NonGNU ELPA)

Module 7 会讲怎么把你的包发布到 MELPA——这是从"用户"到"作者"的跃迁。

---

## Exercises: Week 3

这些练习考宏、加载、编译——三件 Module 6 W3 的核心事。

### 题 1: my-when (宏)

```elisp
(defmacro my-when (cond &rest body)
  `(if ,cond (progn ,@body)))
```

这是宏的最简单例子。注意 `,@body` splice——body 是 list (任意多个 form),splice 进 progn。

### 题 2: my-unless

```elisp
(defmacro my-unless (cond &rest body)
  `(if (not ,cond) (progn ,@body)))
```

`unless` 的标准实现——`(if (not X) BODY)` 的糖。

### 题 3: my-foreach (用 gensym)

```elisp
(defmacro my-foreach (var list &rest body)
  ;; 类似 dolist
  )
```

<details><summary>答案</summary>

```elisp
(defmacro my-foreach (var list &rest body)
  `(let ((lst ,list))
     (while lst
       (let ((,var (car lst)))
         ,@body)
       (setq lst (cdr lst)))))
```

注意我们用 `lst` 作为内部变量——但如果用户调用 `(my-foreach lst my-list ...)` (用户的 var 也叫 lst),就冲突。正确做法是 gensym:

```elisp
(defmacro my-foreach (var list &rest body)
  (let ((lst-sym (gensym)))
    `(let ((,lst-sym ,list))
       (while ,lst-sym
         (let ((,var (car ,lst-sym)))
           ,@body)
         (setq ,lst-sym (cdr ,lst-sym))))))
```

这样无论用户用什么变量名,内部 `lst-sym` 都是唯一的。

</details>

### 题 4: autoload 练习

给你的 my-todo 加 `;;;###autoload`,在 init.el 不 require。

这是真实 package 的标准做法——`;;;###autoload` 让 Emacs 第一次调用 my-todo 命令时才加载 my-todo.el。init.el 不需要 require,启动快。

### 题 5: byte-compile

`M-x byte-compile-file` 编译所有 .el。修复 warning。

修复 warning 是好习惯——`free variable` 警告通常是 typo,`unused` 警告是 dead code。CI 应该 fail on warning。

---

## 自测

1. 宏和函数本质区别?
2. backquote 怎么用?
3. 怎么避免变量捕获?
4. `;;;###autoload` 干啥?
5. byte-compile 好处?

**答案**:
> 1. 宏接收未 eval 的 form,生成代码;函数接收已 eval 的值
> 2. `` ` `` 不 eval;`,X` eval 后插入;`,@X` eval 后 splice list
> 3. gensym 创建唯一 symbol
> 4. 标记下一个 form 为 autoload,生成 autoloads.el
> 5. 加载快 10x,运行快 2-10x,隐藏源码

---

进入 `../week-04-debugging/`。
