# Essential External Packages: dash.el / s.el / f.el / request.el / emacsql / async.el / package-lint

> 老程序员手把手带你过 Emacs Lisp 社区的"事实标准库"

Elisp 这门语言有点怪。它出生在 1985 年,从 GNU Emacs Lisp 这个名字就能看出来——它是"GNU Emacs 的 Lisp",不是给外人用的通用 Lisp。早期的 Emacs 维护者把绝大部分精力花在编辑器本身的功能上(缓冲区、窗口、minor mode 这些),语言层一直停留在"够用就行"的状态。结果就是:你去写一个稍微正经点的包,会发现 list 操作没有 `take`、string 操作没有 `split`(好吧,后来加了)、文件操作全是 `expand-file-name` / `file-name-directory` / `directory-files` 这种 1990 年代风格的低级函数,每个都得手写。

更糟的是 2000 年代的 Elisp 包作者们**每个都自己造轮子**。你看 magit 的早期源码,会发现它自己定义了 `magit-list-files`、`magit-filter`、`magit-mapcat` 这种内部工具函数。看 org-mode 的早期源码,又是另一套名字,`org-remove-if`、`org-map-entries`、`org-string-match-p`。每个包都重新发明一遍 `remove-if-not`、一遍 `string-trim`、一遍 `directory-files-recursively`。结果是什么?bug 越多,代码越长,用户装一个包等于把 1000 行重复逻辑塞进 `.emacs.d/elpa/`。

这个状态在 2012 年被打破了。Magnar Sveen——一个挪威的 Emacs 极客,Emacs Rocks! 视频系列作者——看了一阵 Clojure 后觉得"Emacs Lisp 也应该有 Clojure 那种现代 API",于是一口气写了三个包:**dash.el**(对 list)、**s.el**(对 string)、**f.el**(对 file)。这三个包不是 Emacs 官方维护的,但社区用疯了——到 2024 年,MELPA 上有超过 3000 个包依赖 dash.el,基本上你装任何稍微大点的包,dash 都会被一起拖进来。这就是所谓"事实标准"(de facto standard):不在 Emacs 源码树里,但人人都在用。

这一节我们就把这七个最常用的"基础库"过一遍:`dash` / `s` / `f` / `request` / `emacsql` / `async` / `package-lint`。每个我会讲**它的历史**(谁写的、为什么写、解决什么)、**核心 API**(不是全列出来,只讲你 80% 时间在用的)、**真实例子**(不是 `(s-upcase "a")` 这种玩具)、**陷阱**(每个包都有至少三个能让用户困惑的坑)、**设计取舍**(为什么用它而不是原生,什么时候不该用)。

学完之后你读别人代码会快十倍——因为现代 Elisp 包几乎都用这些库写。你自己写包也会快——不用再纠结 `cl-remove-if-not` 还是 `seq-filter`、不用再手写 `(split-string ...)` 的 trim 逻辑。这就是基础库的价值:**它把社区的碎片化统一了**。

但有个 warning 我要提前打:别滥用。一个 50 行的小包,如果依赖了 dash + s + f + cl-lib + map + seq 六个库,这是反面教材——用户为了用你 50 行代码,得装六个依赖。所以这一节最后我会讲"什么时候该用、什么时候不该用"。

---

## Part 1: 为什么有这些"基础库"

要理解为什么 dash/s/f 这么重要,得先看看它们出现之前 Elisp 的世界有多乱。

Elisp 的标准库一直被吐槽"薄"。你打开 `(info "(elisp) Lists")`,会看到 `car` / `cdr` / `cons` / `nth` / `nthcdr` / `member` / `delete` / `delq` / `remove` / `remq` / `reverse` / `sort` / `assoc` / `rassoc` / `assq` / `rassq` / `copy-sequence` / `length` / `safe-length` / `proper-list-p` / `setcar` / `setcdr`——这是 Elisp 关于 list 的**全部**标准 API。一个东西叫 `delete` 一个叫 `delq`,一个 `member` 一个 `memq`,一个 `assoc` 一个 `assq`,差别只是用 `equal` 还是 `eq` 比较——这是 1970 年代 Maclisp 的命名习惯,一直传到今天。

如果你要做"取前 N 个元素"这种操作,Elisp 标准库**没有** `take` / `drop`。你得自己写:

```elisp
(defun my-take (n list)
  (let ((result nil))
    (while (and (> n 0) list)
      (push (car list) result)
      (setq list (cdr list)
            n (1- n)))
    (nreverse result)))
```

或者用 `cl-subseq`:

```elisp
(cl-subseq '(1 2 3 4 5) 0 3)   ; → (1 2 3)
```

但 `cl-subseq` 是 `cl-lib` 的一部分,不是"Emacs 核心",需要 `(require 'cl-lib)`。而且 `cl-subseq` 只支持有起点和终点的切片,不能像 Clojure 的 `(take 3 coll)` 那样只给个数。

字符串更糟。Emacs 23 之前,**Elisp 标准库没有 `split-string`**。你要 split 一个 CSV 行 `"a,b,c"`,要么用 `replace-regexp-in-string` + `match-string` 写一堆,要么用 `split-string`(后来加的,但行为古怪——空字符串分隔符的行为、trim 的行为都改过几次)。`s-trim` 这种基础操作,Emacs 24 之前没有标准函数——你得用 `(replace-regexp-in-string "^\\s-+\\|\\s-+$" "" s)`,这写出来你自己都不忍心看。

文件操作类似。你想"递归找所有 .org 文件",Emacs 25 之前没有 `directory-files-recursively`。你得自己写:

```elisp
(defun my-find-org-files (dir)
  (let ((result nil))
    (dolist (entry (directory-files dir t))
      (cond
       ((string-prefix-p "." (file-name-nondirectory entry)))
       ((file-directory-p entry)
        (setq result (append result (my-find-org-files entry))))
       ((string-suffix-p ".org" entry)
        (push entry result))))
    (nreverse result)))
```

这 11 行代码,在 f.el 里就一行 `(f-files dir (lambda (f) (f-ext? f "org")) :recursive)`。这就是基础库的价值。

所以 2012 年 Magnar Sveen 写这三个包,本质上是**把社区的重复劳动收敛到一个地方**。dash.el 的灵感主要是 Clojure——Clojure 的 `-` 前缀(`-map`、`-filter`)是为了避免和 `cl-lib` 的 `cl-map` / `cl-remove-if` 冲突,同时也暗示"这是 Clojure-style 的 API"。s.el 是受 Python 的 string methods 启发(虽然 API 完全不同,但功能重叠)。f.el 受 Ruby 的 `File` / `Pathname` 启发(看 `f-exists?`、`f-directory?` 这种问号命名,一眼就是 Ruby 风)。

类比一下:JavaScript 社区的 lodash,Python 社区的 itertools + functools,Ruby 社区的 ActiveSupport。这些都是"标准库不够用,社区补丁"的经典模式。dash/s/f 在 Elisp 社区扮演的角色一模一样。

那"用还是不用"呢?我给几个判断:

**用**——你写的是一个正经包(>200 行),有自己的架构,依赖 dash 节省 30% 代码量是值得的。或者你做大量 list/string/file 处理,dash 让你不用写 5 个工具函数。或者你想用 `->>`(thread macro)——dash 提供的 thread macro 是社区主流,Emacs 25 也有内置 `thread-first` / `thread-last` 但用得少。

**不用**——你写的是一个超小工具(<100 行),只做一件事,装一个依赖反而增加用户负担。或者你的代码主要操作 buffer / window,根本不做 list 处理(比如 minor mode、theme)。或者你的代码是给 Emacs 核心提交的(Emacs 不会接受依赖外部包的代码)。

记住:每个外部依赖都是一个"安装步骤"。用户 `package-install` 你的包,package.el 会自动拉依赖,但如果某个依赖在 MELPA 改名了、或者从 MELPA 移到 GNU ELPA 了、或者依赖某个新 Emacs 版本——你的包就坏了。**依赖越少,生命周期越长**。

---

## Part 2: dash.el —— 现代 list 操作

dash.el 是这三个库里最早写、也最有名的。Magnar Sveen 在 2012 年 8 月开源第一版,初始只有十几个函数,后来社区不断贡献,到现在有 150+ 个函数。它的设计哲学是:**让 Elisp list 操作像 Clojure 一样顺手**。

为什么叫 dash?因为所有函数都以 `-` 开头(`-map`、`-filter`、`-reduce`)。这个约定是为了避免和 `cl-lib`(以 `cl-` 开头)以及 Emacs 内置函数(无前缀,如 `mapcar`)冲突。同时也成了 dash 的"视觉签名"——一眼看到 `-foo` 就知道是 dash 的。

dash 一开始只是 list 函数集合,后来加了一堆"高阶"特性:threading macro(`->>`、`->`)、destructuring (`-let`、`-lambda`)、combinators(`-compose`、`-partial`)、side-effect(`-each`、`-dotimes`)。所以你看到现在的 dash,某种程度上它已经不只是"list 库",而是"Elisp 缺失的函数式编程标准库"。

### 2.1 基础 list 操作

先看最基础的:

```elisp
(require 'dash)

(-map #'1+ '(1 2 3))                   ; → (2 3 4)
(-filter #'cl-oddp '(1 2 3 4 5))       ; → (1 3 5)
(-remove #'cl-oddp '(1 2 3 4 5))       ; → (2 4)
(-reduce #'+ '(1 2 3 4 5))             ; → 15
(-take 3 '(1 2 3 4 5))                 ; → (1 2 3)
(-drop 2 '(1 2 3 4 5))                 ; → (3 4 5)
(-partition 2 '(1 2 3 4 5 6))          ; → ((1 2) (3 4) (5 6))
(-zip '(1 2 3) '(a b c))               ; → ((1 a) (2 b) (3 c))
```

逐行讲。

`(-map #'1+ '(1 2 3))` 调用 `-map`,第一个参数是函数(用 `#'` 是简写,等价于 `(function 1+)`,告诉 byte-compiler 这个是函数引用,会被 byte-compile 优化),第二个是 list。返回 `(2 3 4)`——每个元素加 1。和 `mapcar` 的区别是什么?**几乎没区别**。`(-map fn list)` 就是 `(mapcar fn list)`。dash 提供 `-map` 主要是为了"完整 API"——你已经在用 dash 了,顺手用 `-map` 比 `mapcar` 一致性更好。性能上 `mapcar` 略快(因为原生),但差几个微秒可以忽略。

`(-filter #'cl-oddp '(1 2 3 4 5))` 返回 `(1 3 5)`——保留所有奇数。和 `cl-remove-if-not` 等价,但 `cl-remove-if-not` 是双重否定(名字绕),`-filter` 名字直观。**这是 dash 比 cl-lib 强的地方**:命名更好。

`(-remove #'cl-oddp ...)` 是 `-filter` 的反面,移除满足谓词的元素。和 `cl-remove-if` 等价。

`(-reduce #'+ '(1 2 3 4 5))` 把 list 折叠成一个值。`(reduce + '(1 2 3 4 5))` 等价于 `(+ (+ (+ (+ 1 2) 3) 4) 5)` = 15。`cl-reduce` 也有,但 `-reduce` 多了一个变体 `-reduce-r`(从右往左),还有 `-reduce-from`(可以指定初始值)。这些变体在 cl-lib 里都散在不同函数名下,dash 把它们统一了。

`(-take 3 '(1 2 3 4 5))` 取前 3 个,返回 `(1 2 3)`。cl-lib 没有 take,你得 `(cl-subseq list 0 (min 3 (length list)))`——丑死了。

`(-drop 2 '(1 2 3 4 5))` 丢掉前 2 个,返回 `(3 4 5)`。等于 `(nthcdr 2 list)`,但名字更清晰。

`(-partition 2 '(1 2 3 4 5 6))` 把 list 切成长度 2 的子 list,返回 `((1 2) (3 4) (5 6))`。这在处理"成对数据"时超有用,比如处理 `((key1 val1) (key2 val2))` 这种 alist 时。原生 Elisp 完全没有这个。

`(-zip '(1 2 3) '(a b c))` 把两个 list 配对,返回 `((1 a) (2 b) (3 c))`。处理"两个并行 list"时超有用——比如你有 `(names "alice" "bob")` 和 `(ages 30 25)`,想合并成 `(("alice" 30) ("bob" 25))`,就 `(-zip names ages)`。原生没有,得用 `-map-car` 或者手写循环。

### 2.2 与原生 / cl-lib 对照

最常见的问题:"dash vs cl-lib vs 原生,用哪个?"。下面是对照清单:

| dash | cl-lib | 原生 | 说明 |
|------|--------|------|------|
| `-map` | `cl-mapcar` | `mapcar` | dash 和原生几乎一样,cl-mapcar 多个 list 支持稍好 |
| `-filter` | `cl-remove-if-not` | (无) | `-filter` 名字最清晰 |
| `-remove` | `cl-remove-if` | `delete` (但 destructively) | `delete` 是 destructive 且用 `eq`,慎用 |
| `-reduce` | `cl-reduce` | (无) | 几乎一样 |
| `-take` | (无) | (无) | cl-subseq 模拟,但语义不同 |
| `-drop` | (无) | `nthcdr` | nthcdr 是 destructive 名字烂 |
| `-find` | `cl-find-if` | (无) | 返回第一个满足条件的元素 |
| `-any?` | `cl-some` | (无) | 是否有元素满足谓词 |
| `-all?` | `cl-every` | (无) | 是否所有元素都满足 |

实战建议:**优先用 dash**。理由:命名一致(`-` 前缀),功能更全(有 take/drop/partition/zip),thread macro 配套好。原生用 `mapcar` 和 `sort` 这些"经典"操作,cl-lib 只用 `cl-defstruct`、`cl-loop`、`cl-multiple-value-bind` 这种 dash 不覆盖的。

### 2.3 Thread 宏 —— dash 的杀手锏

dash 最 killer 的特性是 thread macro。先看一段没 thread 的代码:

```elisp
(reduce #'+
        (-filter (lambda (x) (> x 2))
                 (-map (lambda (x) (* x x))
                       '(1 2 3))))
```

这段代码做三件事:每个元素平方 → 保留 >2 的 → 求和。但读起来是**倒着**的——你从最里面 `(1 2 3)` 开始读,然后看到 `-map`、然后 `-filter`、然后 `reduce`。嵌套越多越难读。

thread macro 把这个嵌套**展平**成线性流程:

```elisp
(->> '(1 2 3)
     (-map (lambda (x) (* x x)))     ; → (1 4 9)
     (-filter (lambda (x) (> x 2)))  ; → (4 9)
     (-reduce #'+))                  ; → 13
```

`->>` 叫 thread-last,意思是"把上一个 form 的结果插到下一个 form 的**最后一个参数位置**"。所以 `'(1 2 3)` 作为 `-map` 的最后一个参数,`(-map ... '(1 2 3))` 算出 `(1 4 9)`;然后 `(1 4 9)` 作为 `-filter` 的最后一个参数,算出 `(4 9)`;然后 `(4 9)` 作为 `-reduce` 的最后一个参数,算出 `13`。

这就像 Unix 管道:`cat foo | grep bar | wc -l`。从上往下读,数据流向清楚。**写数据处理流水线时,thread 是首选**。

`->` 叫 thread-first,把结果插到第一个参数位置。用于"链式方法调用"风格的 API:

```elisp
(-> "hello"
    (upcase)              ; → "HELLO"
    (concat " world"))    ; → "HELLO world"
```

这里 `upcase` 接收一个参数,`(upcase "hello")` → `"HELLO"`;然后 `concat` 第一参数是 `"HELLO"`,第二参数是 `" world"`,得到 `"HELLO world"`。

什么时候用 `->>`(last) vs `->`(first)?经验:**list / sequence 操作几乎都 last**(因为 dash / cl-lib / seq 的函数都是 `(fn item ...args)`,被操作的项放最后?其实不是,dash 的 `-map` 是 `(fn fn list)`,list 在最后)。**对象方法风格几乎都 first**(比如 `concat`、`upcase`,被操作的字符串在第一个参数)。

dash 还提供 `-->`(thread with explicit placeholder),你用 `it` 标记位置:

```elisp
(--> "hello"
     (concat it " world")          ; → "hello world"
     (upcase it)                   ; → "HELLO WORLD"
     (s-split " " it))             ; → ("HELLO" "WORLD")
```

`it` 是 placeholder,指代上一个 form 的结果。这是 dash 独有的,cl-lib 没有对应。

### 2.4 实战:统计 MELPA 包依赖数

下面是一个真实场景。你想知道你 `.emacs.d/elpa/` 下哪些包占了最多空间——也就是哪个包目录最大。

```elisp
(defun my-melpa-stats ()
  "Return top 10 largest package directories in `package-user-dir'."
  (->> (directory-files package-user-dir t "[^.]")
       (-map (lambda (d)
               (cons (file-name-nondirectory d)
                     (-reduce #'+
                              (-map #'file-attribute-size
                                    (directory-files-and-attributes d))))))
       (-sort (lambda (a b) (> (cdr a) (cdr b))))
       (-take 10))))
```

逐行讲。

`(directory-files package-user-dir t "[^.]")` 列出 `package-user-dir`(Emacs 装包的目录,通常是 `~/.emacs.d/elpa/`)下的所有非隐藏条目。第二个参数 `t` 表示返回绝对路径(默认返回相对名)。第三个参数是 regexp,只匹配不以点开头的——这过滤掉 `.` 和 `..`。这一行给你一个全路径 list,比如 `("/home/user/.emacs.d/elpa/dash-20230810.1404" "/home/user/.emacs.d/elpa/s-20230810.1404" ...)`。

`(-map (lambda (d) ...))` 对每个目录算 `(cons 名字 大小)`。

`(file-name-nondirectory d)` 从全路径提取最后一段,比如 `"/home/user/.emacs.d/elpa/dash-20230810.1404"` → `"dash-20230810.1404"`。

`(directory-files-and-attributes d)` 返回这个目录下所有文件的列表,每个元素是 `(文件名 . 属性列表)`,属性列表第一个元素是 size。

`(-map #'file-attribute-size ...)` 对每个 `(文件名 . 属性)` 取 size。`file-attribute-size` 是个内置函数,从 file-attributes 返回的列表里提取 size。

`(-reduce #'+ ...)` 把所有 size 加起来,得到目录总大小。

`(-sort (lambda (a b) (> (cdr a) (cdr b))))` 按大小降序排。注意 `-sort` 是 non-destructive 的(原生 `sort` 是 destructive,会修改原 list——这是 dash 又一个优势)。

`(-take 10)` 取前 10 个。

返回类似 `(("org-9.6.15" . 4567890) ("magit-20240101.1234" . 3456789) ...)`,你可以看到哪些包占地方大。

这 7 行代码用了 dash 5 个函数,如果用原生写得写 20 行。**这就是 dash 的价值**。

### 2.5 陷阱

dash 有几个坑新手容易踩。

**陷阱 1:`-sort` vs `sort`**。原生 `sort` 是 destructive——它会**修改原 list 的 cons cell**。dash 的 `-sort` 是 non-destructive,返回新 list。如果你传一个常量 list 给 `sort`:

```elisp
(sort '(3 1 2) #'<)   ; 试图 sort 字面量 list,byte-compile 会警告
```

byte-compiler 会警告 "argument is a constant"。但 `-sort` 没事:

```elisp
(-sort #'< '(3 1 2))   ; → (1 2 3),原 list 不变
```

**陷阱 2:lazy 序列不存在**。dash 的 list 全部是严格的——所有元素在调用时都求值完。你写 `(-filter #'cl-oddp (my-huge-list))`,它会先**完整**调用 `my-huge-list`,生成完整 list,再 filter。不像 Haskell/Clojure 的 lazy seq,你不能用 dash 处理"无限序列"。如果你要 lazy,得用 generator / stream 库。

**陷阱 3:`-map` 的 fn 参数不是 function designator**。dash 大多数函数接受 function(可以用 `#'` 或 symbol),但**不接受 lambda 字面量直接传**——你得 quote 或用 `#'`:

```elisp
(-map (lambda (x) (* x 2)) '(1 2 3))   ; OK
(-map #'(lambda (x) (* x 2)) '(1 2 3)) ; OK
;; (-map (lambda ...) ...) 不 quote 也行,因为 lambda form 自求值
```

其实现在 `(lambda ...)` 在 lexical-binding 下自求值,所以可以直接传。但**对 defun 出来的 symbol 必须 `#'` 或 `'`**:

```elisp
(-map '1+ '(1 2 3))    ; OK
(-map #'1+ '(1 2 3))   ; OK,更快(byte-compile 优化)
```

**陷阱 4:thread 宏的展开**。`->>` 看起来像 Clojure 的 `->>`,但 Clojure 的 macro 在写代码时IDE 会高亮 placeholder。dash 的 `->>` 在 Elisp 里**没有 placeholder 高亮**——你写错位置 macro 不会报错,会静默按错误方式展开。比如:

```elisp
(->> '(1 2 3)
     (-map (lambda (x) (* x 2)))     ; 你期望 (-map fn '(1 2 3))
     (-filter (> 2)))                ; 期望 (-filter (> 2) '(2 4 6))?但 (> 2) 不是函数
```

`(-filter (> 2) ...)` 会试图调用 `(> 2)`,这是错的——`>` 至少要 2 个参数。你应该用 `(lambda (x) (> x 2))` 或 `(-partial #'> 2)`。thread macro 不会帮你检查这种逻辑错误。

**陷阱 5:版本兼容性**。dash 频繁加新函数,新加的函数在老版本 dash 里不存在。如果你的包用了一个 dash 1.x 没有的函数(比如 `-partition-by`),你必须在 `Package-Requires` 写 `(dash "2.x")`。MELPA 检查这个,但用户自己装时可能漏。

---

## Part 3: s.el —— 现代 string 操作

s.el 是 Magnar Sveen 同期写的"字符串版的 dash"。命名约定是 `s-` 前缀。和 dash 一样,目的也是把社区的字符串处理统一到一处。

为什么需要 s.el?Elisp 的字符串操作比 list 还乱。`split-string` 行为不一致(对空字符串分隔符、对正则的特殊字符),`replace-regexp-in-string` 的参数顺序反人类(替换字符串在 pattern 之前),`string-trim` 是 Emacs 24.4 才加的。你要做点稍微复杂的事,比如"把字符串按一行一行 split 然后去空行",得 5 行代码。

s.el 把这些都封装好。

### 3.1 基础

```elisp
(require 's)

(s-split "," "a,b,c")                   ; → ("a" "b" "c")
(s-join "-" '("a" "b" "c"))             ; → "a-b-c"
(s-trim "  hello  ")                    ; → "hello"
(s-replace "foo" "bar" "foo baz")       ; → "bar baz"
(s-upcase "hello")                      ; → "HELLO"
(s-ends-with? ".el" "foo.el")           ; → t
(s-contains? "ell" "hello")             ; → t
(s-match-strings-all "\\([a-z]+\\)\\([0-9]+\\)" "abc123")
(s-word-wrap 10 "hello world foo")
```

`(s-split "," "a,b,c")` 按逗号切分,返回 list。比 `split-string` 好在:对 `""` 分隔符的行为是"切成字符列表"(更直观),对正则字符串也能正确处理(默认不 trim)。

`(s-join "-" '("a" "b" "c"))` 用 "-" 连接 list 成字符串。和 `mapconcat` 一样,但名字直观(mapconcat 的"map"误导新手,以为是先 map 再 concat)。**这是 s.el 的典型优化:把原生命名改清楚**。

`(s-trim "  hello  ")` 去掉前后空白。Emacs 24.4+ 内置 `string-trim`,行为一样。所以 s-trim 在新 Emacs 上其实可以省,但写包兼容老 Emacs 时还是用 s-trim。

`(s-replace "foo" "bar" "foo baz")` 把所有 "foo" 替换成 "bar"。`replace-regexp-in-string` 也能做,但参数顺序反人类:

```elisp
(replace-regexp-in-string "foo" "bar" "foo baz")
;;            ↑ pattern    ↑ repl   ↑ string
```

新手总是搞反 pattern 和 repl。`s-replace` 用 `(from to string)` 顺序,**更符合人类思维**(从哪来到哪去,然后操作对象)。

`(s-upcase "hello")` 大写化。和 `upcase` 一样,但 s.el 提供了一整套对称:`s-upcase` / `s-downcase` / `s-capitalize` / `s-title-case`,命名一致。

`(s-ends-with? ".el" "foo.el")` 问号结尾返回 bool,检查是否以某后缀结尾。比 `(string-suffix-p ".el" "foo.el")` 名字清楚(string-suffix-p 是 Emacs 24+ 加的,但 `-p` 是 predicate 传统命名,s.el 用 `?` 是更现代风)。

`(s-contains? "ell" "hello")` 检查子串。比 `(string-match-p "ell" "hello")` 名字直观。

`(s-match-strings-all REGEX STRING)` 返回所有匹配,包括 group。比如 `(s-match-strings-all "\\([a-z]+\\)\\([0-9]+\\)" "abc123")` 返回所有匹配的 list,每个是 `(full-match group1 group2)`。这是处理"批量提取"的神器——原生你得手写 `while (string-match ...)` 循环,超烦。

`(s-word-wrap 10 "hello world foo")` 按单词换行,每行不超过 10 字符。原生没有,得用 `fill-region-to-left-margin` 或者 `wrap-region`——都是 buffer 操作,不是 string 操作。s.el 直接做 string→string。

### 3.2 实战:slugify

写博客 / 做静态网站生成器时常需要把标题转成 URL-safe slug:"Hello World!" → "hello-world"。

```elisp
(defun my-slugify (s)
  "Convert string S to a URL slug: lowercase, spaces to dashes, no punctuation."
  (->> s
       s-downcase
       (s-replace-all '((" " . "-") ("!" . "") ("?" . "")
                        ("." . "") ("," . "") ("'" . "")))))
```

逐行讲。

`(->> s ...)` thread-last,把 `s` 作为下面每个 form 的最后一个参数。

`s-downcase` 把整个字符串转小写。`"Hello World!"` → `"hello world!"`。

`(s-replace-all '((" " . "-") ...))` 是 s.el 的批量替换函数。参数是 alist,每个 `(from . to)`,按顺序应用。`s-replace-all` 等价于多个 `s-replace` 串起来,但只走一遍字符串(更高效)。结果是 `"hello-world"`。

注意 alist 顺序:先替换空格再替换标点。如果先替换 `"!"` 再替换 `" "`,结果一样,但如果有些替换会引入新字符(比如把 `"&"` 替换成 `"and"`,引入空格),顺序就重要。**写 s-replace-all 时要想清楚顺序**。

实战扩展:还可以用更狠的版本,只保留字母数字:

```elisp
(defun my-slugify-strict (s)
  "Strict slugify: only alphanumeric, rest to dash, collapse dashes."
  (->> s
       s-downcase
       (s-replace-regexp "[^a-z0-9]+" "-")
       (s-trim-left "-")
       (s-trim-right "-")))

(my-slugify-strict "Hello, World! How are you?")
;; → "hello-world-how-are-you"
```

`s-replace-regexp` 是 s.el 的正则替换。`"[^a-z0-9]+"` 匹配一个或多个非字母数字字符,替换成 `-`。然后用 trim 去掉首尾可能多出来的 dash。

### 3.3 陷阱

**陷阱 1:`s-split` 的保留空字符串**。`(s-split "," "a,,b")` 默认返回 `("a" "" "b")`——保留空段。但 CSV 处理时你通常不想要空段。s-split 有可选参数 TRIM,可以传 `" "` 之类的修剪;或者用 `(s-split "," "a,,b" t)` 跳过空——第三个参数是 OMIT-NULLS,行为像 Python 的 `str.split`。

**陷阱 2:`s-replace` 不做正则**。`s-replace` 是字面替换,不是正则。如果你想用正则,得用 `s-replace-all`(还是字面)或 `s-replace-regexp`(正则)。新手经常写 `(s-replace "[0-9]+" "" "abc123")` 期望去掉所有数字,但结果还是 `"abc123"`——因为 `s-replace` 在找字面字符串 `"[0-9]+"`。

**陷阱 3:`s-match-strings-all` 返回的 list 嵌套**。返回的是 list of list——每个匹配一个 list,第一个是整个匹配,后面是 groups。新手经常以为返回 flat list,然后 `(car (s-match-strings-all ...))` 取"第一个匹配",其实取到的是"第一个匹配的所有内容(包括 groups)",得 `(car (car ...))` 取第一个匹配的整段。

**陷阱 4:`s-trim` 在 Emacs 24.4+ 有原生 `string-trim`**。如果你不在乎兼容老 Emacs,用原生就行,s-trim 和 string-trim 行为一样。

**陷阱 5:`s-join` vs `mapconcat`**。`s-join` 是 `(separator list)`,`mapconcat` 是 `(function sequence separator)`。两者顺序差很多——mapconcat 多一个 function 参数(用来把每个元素转字符串),s-join 假设元素已经是字符串。所以 `(s-join "-" '(1 2 3))` 会出错(1 不是字符串),得 `(s-join "-" (-map #'number-to-string '(1 2 3)))`。

---

## Part 4: f.el —— 现代 file 操作

f.el 是 Magnar 三件套的第三个,对文件系统。命名 `f-` 前缀,问号结尾的返回 bool(`f-exists?`、`f-directory?`),其他返回值。

设计灵感明显是 Ruby 的 `File` 和 `Pathname` 类。Ruby 程序员看 f.el 会觉得很亲切——`f-exists?` 对应 `File.exists?`,`f-directory?` 对应 `File.directory?`,`f-ext` 对应 `File.extname`。

### 4.1 基础

```elisp
(require 'f)

(f-exists? "/etc/passwd")               ; → t
(f-read "/etc/hostname")                ; → "myhost\n"
(f-write "hello" "/tmp/x.txt")
(f-append "world" "/tmp/x.txt")
(f-mkdir "/tmp/a/b/c")                  ; 自动创建多层
(f-delete "/tmp/a")
(f-copy "/from" "/to")
(f-move "/from" "/to")
(f-ext "/path/file.txt")                ; → "txt"
(f-no-ext "/path/file.txt")             ; → "/path/file"
(f-join "a" "b" "c")                    ; → "a/b/c"
(f-dirname "/a/b/c.txt")                ; → "/a/b"
(f-filename "/a/b/c.txt")               ; → "c.txt"
(f-glob "*.el" package-user-dir)
(f-files "/home/user" (lambda (f) (f-ext? f "el")))
```

逐行讲。

`(f-exists? "/etc/passwd")` 返回 t 或 nil。和 `(file-exists-p "/etc/passwd")` 一样,但 `?` 结尾更清楚是 predicate。

`(f-read "/etc/hostname")` 读整个文件为字符串。原生没对应的单函数——你得:

```elisp
(with-temp-buffer
  (insert-file-contents "/etc/hostname")
  (buffer-string))
```

4 行变 1 行。**这是 f.el 最大的价值**——把 idiom 封装成函数。

`(f-write "hello" "/tmp/x.txt")` 写字符串到文件,覆盖。原生对应:

```elisp
(with-temp-file "/tmp/x.txt"
  (insert "hello"))
```

`(f-append "world" "/tmp/x.txt")` 追加。原生对应 `write-region`:

```elisp
(write-region "world" nil "/tmp/x.txt" 'append)
```

但 `write-region` 的参数顺序是 `(start end filename append visit lockname mustbenew)`,新手记不住。f-append 把它封装成 `(content filepath)`。

`(f-mkdir "/tmp/a/b/c")` 创建多层目录。原生 `make-directory` 也行,但需要传 `t` 作为 parents 参数:

```elisp
(make-directory "/tmp/a/b/c" t)
```

`(f-delete "/tmp/a")` 删文件或目录(递归)。原生 `delete-file` 只删文件,`delete-directory` 只删空目录——删非空目录得自己写递归。f-delete 统一处理。

`(f-copy "/from" "/to")` 复制文件或目录。原生 `copy-file` 行为类似,但目录复制需要 `copy-directory`,名字分裂。f-copy 统一。

`(f-move "/from" "/to")` 移动/改名。等于 `rename-file`,但 rename 在跨文件系统时会失败,f-move 会自动 fallback 到 copy + delete。

`(f-ext "/path/file.txt")` 取扩展名,返回 `"txt"`(不带点)。和 `file-name-extension` 一样。

`(f-no-ext "/path/file.txt")` 取去掉扩展名的路径,返回 `"/path/file"`。和 `file-name-sans-extension` 一样,但名字短。

`(f-join "a" "b" "c")` 路径拼接。这是 f.el 的神特性——你不用再写 `(expand-file-name "b" (expand-file-name "a" "/"))` 这种嵌套。f-join 接受任意个参数,智能处理 `/`:`(f-join "/usr" "local" "bin")` → `"/usr/local/bin"`。

`(f-dirname "/a/b/c.txt")` 取目录,返回 `"/a/b"`。和 `file-name-directory` 一样,但短。

`(f-filename "/a/b/c.txt")` 取文件名,返回 `"c.txt"`。和 `file-name-nondirectory` 一样,但名字短得多——`file-name-nondirectory` 这名字简直是惩罚。

`(f-glob "*.el" package-user-dir)` 通配符匹配。返回所有匹配的文件 list。原生没对应,得用 `directory-files` + regexp。

`(f-files "/home/user" predicate)` 列出所有文件(可带过滤 predicate)。第二个参数是函数,返回非 nil 的才保留。还有第三个可选参数 `:recursive`。

### 4.2 实战:递归找所有 .org 文件

```elisp
(defun my-all-org-files (dir)
  "Recursively find all .org files under DIR."
  (->> (f-files dir nil :recursive)
       (-filter (lambda (f) (f-ext? f "org")))
       (-map #'f-long)))
```

逐行讲。

`(f-files dir nil :recursive)` 列出 dir 下所有文件,递归(因为第三个参数是 `:recursive`)。第二个参数 `nil` 表示不过滤(predicate 为 nil = 全保留)。返回相对路径的 list。

`(-filter (lambda (f) (f-ext? f "org")))` 保留扩展名是 org 的。`f-ext?` 接收路径和扩展名,返回 bool。

`(-map #'f-long)` 把相对路径转成绝对路径。`f-long` 是 f.el 的"返回绝对规范路径"函数。

3 行代码搞定。原生你至少 10 行。

### 4.3 陷阱

**陷阱 1:`f-write` 会覆盖**。f-write 是**覆盖**写,不是追加。如果你想追加,用 `f-append`。新手经常把数据覆盖掉。

**陷阱 2:`f-files` 默认非递归**。第三个参数默认 nil,你得显式传 `:recursive` 或 t。所以 `(f-files dir)` 只列 dir 直接子文件,不进子目录。

**陷阱 3:`f-expand` vs `f-join`**。`f-expand` 是 "expand 相对路径为绝对路径",`f-join` 是"拼接路径组件"。`(f-expand "foo" "/usr")` → `"/usr/foo"`,行为像 `expand-file-name`。`(f-join "/usr" "foo")` → `"/usr/foo"`,看起来一样,但 f-join 不会展开 `~` 或环境变量:

```elisp
(f-join "~" "foo")    ; → "~/foo",字面拼接
(f-expand "foo" "~")  ; → "/home/user/foo",展开 ~
```

**陷阱 4:`f-mkdir` 已存在不报错**。如果目录已经存在,f-mkdir 不报错直接返回。这通常是你想要的,但如果你想"创建失败说明有问题"时,你得自己 `unless (f-exists? ...)` 检查。

**陷阱 5:f.el 不替代 `find-file`**。f.el 是文件**系统**操作(读写、复制、删除),不是 buffer 操作。`find-file` / `save-buffer` 这种"打开 buffer 编辑"还是用原生。f.el 是"批处理 / 后台"的文件操作。

---

## Part 5: request.el —— HTTP 客户端

request.el 是另一个时间段的产物——2012 年 Takafumi Arakaki(@tkf),一个在 ENS 的日本物理学家/程序员写的。他是 Emacs Python 社区(emacs-jupyter、epc)的核心贡献者。

要理解 request.el,得先看 Emacs 原生 HTTP 客户端:`url-retrieve`。这个函数是 1990 年代写的,API 极其难用:

```elisp
(let ((url-show-status nil))
  (url-retrieve "https://api.github.com/users/octocat"
                (lambda (status)
                  (goto-char (point-min))
                  (re-search-forward "^$" nil 'move)  ; skip headers
                  (forward-char)
                  (let ((data (buffer-substring (point) (point-max))))
                    (message "Got: %s" data)))))
```

这代码什么问题?**回调地狱**——`url-retrieve` 是异步的,你传一个 callback,它会在某个临时 buffer 里被调用,buffer 包含 HTTP headers + body,你得手动跳过 headers(`re-search-forward "^$"` 找空行)。错误处理烂:如果 HTTP 失败,status 里有 `:error`,但格式不规整。JSON 解析要自己 `(json-read-from-string data)`。**没 timeout、没重试、没 cookie 管理**——你需要的一切都没有。

Takafumi 看到这个,写了 request.el。设计目标:**API 像 Python 的 requests,行为像 url-retrieve(异步)但封装好,支持同步模式,自动 JSON 解析**。

### 5.1 基础

```elisp
(require 'request)

(request "https://api.github.com/users/octocat"
  :parser 'json-read
  :success (cl-function
            (lambda (&key data &allow-other-keys)
              (message "Got: %S" data))))
```

逐行讲。

`(request URL ...)` 是核心函数。第一个参数是 URL 字符串。后面是 keyword 参数,控制行为。

`:parser 'json-read` 指定响应 body 的解析函数。request.el 会在 buffer 里调用这个函数,结果作为 `:data` 传给你的 `:success` callback。`json-read` 是 Emacs 内置的 JSON parser。如果你想拿原始字符串,传 `:parser (lambda () (buffer-string))`,或者干脆不传(默认是 buffer-string)。

`:success (cl-function (lambda (&key data &allow-other-keys) ...))` 是成功回调。`cl-function` 是 cl-lib 的宏,让 lambda 支持 `&key` 参数解构——`&key data` 会从 request.el 传来的 keyword 参数列表里提取 `:data`。`&allow-other-keys` 表示"忽略其他 keyword"(因为 request 会传 `:response`、`:status-code` 等你可能不关心的)。

`data` 是解析后的内容——因为传了 `:parser 'json-read`,data 是 alist / plist 形式的 JSON。比如 `(("login" . "octocat") ("id" . 583231) ...)`。

request.el 关键的 keyword 参数:

- `:type` HTTP method,默认 GET。`:type "POST"` 或 `:type "PUT"`。
- `:params` query string 参数,alist。如 `:params '(("page" . "1") ("per_page" . "100"))`。
- `:data` POST body,字符串或 alist(自动 form-urlencoded)。
- `:headers` HTTP headers,alist。如 `:headers '(("Authorization" . "token xxx") ("Accept" . "application/json"))`。
- `:parser` body 解析函数,前面讲过。
- `:success` 成功(2xx)回调。
- `:error` 失败(4xx / 5xx)回调,签名同 success。
- `:complete` 无论成功失败都调用。
- `:sync` 是否同步。`:sync t` 是同步模式,callback 在 request 返回前调用。默认 nil(异步)。
- `:timeout` 超时秒数。

### 5.2 cl-function 解释

request.el 的 callback 用 `cl-function`,这是 cl-lib 提供的宏,让 lambda 支持 Common Lisp 风格的参数列表:

```elisp
(cl-function
 (lambda (&key data status-code &allow-other-keys)
   (message "Status: %S, Data: %S" status-code data)))
```

`&key` 表示参数按 keyword 传递——调用方传 `:data xxx :status-code 200`,lambda 里就能用 `data` 和 `status-code` 变量。`&allow-other-keys` 表示"即使有别的 keyword 也别报错"——request 会传一堆 keyword,你不全接收的话会报错。

为什么用 cl-function 而不是普通 lambda?因为普通 lambda 不支持 `&key`(Elisp lambda 只支持 `&optional` 和 `&rest`)。cl-function 是 cl-lib 给的"完整 CL 参数列表"支持。

### 5.3 实战:GitHub star 数查询

下面写一个真实工具——给一个 GitHub repo 名,返回 star 数。

```elisp
(require 'request)

(defun my-github-stars (repo)
  "Return star count of GitHub REPO (e.g. \"magit/magit\")."
  (let ((result nil))
    (request (format "https://api.github.com/repos/%s" repo)
      :parser (lambda () (json-read))
      :headers '(("Accept" . "application/vnd.github+json"))
      :sync t
      :success (cl-function
                (lambda (&key data &allow-other-keys)
                  (setq result (alist-get 'stargazers_count data))))
      :error (cl-function
              (lambda (&key error-thrown &allow-other-keys)
                (message "Error: %S" error-thrown))))
    result))

(my-github-stars "magit/magit")  ; → e.g. 5000+
```

逐行讲。

`(let ((result nil)) ...)` 定义一个本地变量 `result`,初始 nil。这个变量会被 callback 修改。`let` 在 lexical-binding 模式下创建闭包,callback 能"捕获"它。

`(request URL ...)` 发请求。`:sync t` 是关键——同步模式,request 会阻塞直到响应回来,callback 在 request 返回前执行完。这样我们能在 callback 里 setq `result`,然后函数返回时 result 已经被填充。

`(format "https://api.github.com/repos/%s" repo)` 把 repo 名(如 `"magit/magit"`)嵌入 URL。

`:parser (lambda () (json-read))` 解析 body 为 JSON。注意 parser 是无参函数,request 在 buffer 里调用它,buffer 内容是 HTTP body。

`:headers '(("Accept" . "application/vnd.github+json"))` 设置 Accept header。GitHub API 现在要求显式 Accept,不然会返回旧版本 API 的响应。

`:success ...` 2xx 回调。`(alist-get 'stargazers_count data)` 从 alist 取 `stargazers_count` 字段。**注意是 symbol 不是 string**——`json-read` 默认把 JSON key 转 symbol(可通过 `json-object-type` 配置成 string)。所以 JSON 里是 `"stargazers_count": 5000`,Elisp 里是 `('stargazers_count . 5000)`。

`:error ...` 错误回调。`error-thrown` 是 cons `(error . <message>)` 或类似。

最后 `result` 作为函数返回值——因为 `:sync t`,callback 已经执行过,result 已经填充。

### 5.4 sync 模式的坑

`:sync t` 让代码简单,但**阻塞整个 Emacs**——request 在底层用 `url-retrieve-synchronously`,这函数在等待 HTTP 响应期间不让 Emacs 处理用户输入。对一个 5 秒的 HTTP 请求,Emacs 会卡 5 秒,用户体验灾难。

所以 sync 模式**只适合**:

1. 命令行工具(脚本里调用,不在交互 Emacs)
2. 用户主动触发的"等待"操作(用户点了 M-x ...,期望等几秒)
3. 启动时初始化(用户能接受启动慢一点)

**不适合**:

1. 后台同步(应该用 async)
2. 实时反馈(应该 async + 进度显示)
3. 高频调用(会频繁卡)

异步模式(`:sync nil` 或省略)的"callback hell"问题怎么破?有几个模式:

**模式 1:链式 callback**——每个 callback 里启动下一个 request:

```elisp
(defun my-fetch-all-repos (user)
  "Fetch all of USER's repos, then fetch each repo's stars."
  (request (format "https://api.github.com/users/%s/repos" user)
    :parser #'json-read
    :success (cl-function
              (lambda (&key data &allow-other-keys)
                (--each data
                  (my-fetch-repo-stars (alist-get 'full_name it)))))))
```

`--each` 是 dash 的"对每个元素执行 side-effect"函数。

**模式 2:Promise / generator**——用 `emacs-promise` 或 `generator.el` 把 callback 变成 awaitable。但 Elisp 这两个库都不成熟,2024 年还有人在折腾。

**模式 3:把状态存到全局变量**——简单粗暴:

```elisp
(defvar my-github-queue nil
  "List of repos to fetch.")

(defun my-process-queue ()
  (if (null my-github-queue)
      (message "Done!")
    (let ((repo (pop my-github-queue)))
      (request (format "https://api.github.com/repos/%s" repo)
        :parser #'json-read
        :success (cl-function
                  (lambda (&key data &allow-other-keys)
                    (message "%s: %d stars" repo (alist-get 'stargazers_count data))
                    (my-process-queue)))))))
```

这是经典的"自己实现 coroutine"模式——用全局队列 + 递归调用。简单可靠。

### 5.5 陷阱

**陷阱 1:GitHub API 限流**。GitHub 未授权 API 每小时 60 次。你跑 100 个请求,第 61 个就 403。解决方案:用 Personal Access Token,加 `:headers '(("Authorization" . "token xxx"))`。

**陷阱 2:`:sync t` 和 error**。`:sync t` 模式下,如果请求失败,`:error` callback 会被调用,但**主调用 `(request ...)` 的返回值不是错误**——它返回一个 request object。错误信息只在 `:error` callback 里。所以 `:sync t` 模式你得在 error callback 里设置错误标志:

```elisp
(let ((result nil) (err nil))
  (request url
    :sync t
    :success (lambda (&key data &allow-other-keys) (setq result data))
    :error (lambda (&key error-thrown &allow-other-keys) (setq err error-thrown)))
  (if err
      (error "Failed: %S" err)
    result))
```

**陷阱 3:JSON 解析返回的 alist 用 symbol key**。`(json-read)` 默认返回 `((key1 . val1) (key2 . val2))` 形式,key 是 symbol。但有些 API 返回 snake_case key,有些 camelCase,你需要 `(alist-get 'some_field data)` 而不是 `(alist-get "some_field" data)`。可以通过 `json-object-type` 改成 `plist`(默认 alist),或者 `hash-table`。

**陷阱 4:HTTP 重定向**。request.el 默认跟随重定向,但 url-retrieve 在某些情况(HTTPS → HTTP)会卡住。如果遇到奇怪卡顿,试试 `:encoding 'binary`。

**陷阱 5:`request` 不是 `url.el`**。request.el 是 url.el 的包装,但它**不是所有 url.el 功能都有**。比如 SOCKS 代理支持需要你自己在 `(setq url-gateway-method 'socks)`。

---

## Part 6: emacsql —— SQL in Elisp

emacsql 是 2014 年 Christopher Wellons 写的。Wellons 是 Emacs 社区的"神"——他写了 SQLite 的 Emacs 绑定(`emacsql-sqlite`)、PDF 工具(`pdf-tools`)、图像浏览(`imenu-anywhere`)等十几个高质量包。他在 NASA 做航天软件工程师,写 Elisp 是业余爱好,代码质量极稳定。

为什么需要 emacsql?在 emacsql 之前,Elisp 操作数据库极其痛苦。如果你想在 Emacs 里存数据,选项只有:

1. **savehist / saveplace**——把数据存成 Elisp 表达式,load/save 整体。适合小数据。
2. **org-mode 表格**——人能读,机器难解析。
3. **bbdb**——专门给联系人,API 不通用。
4. **直接调 sqlite3 进程**——`(call-process "sqlite3" ...)` ,自己拼 SQL 字符串,容易 SQL injection。

emacsql 的设计目标:**让 Elisp 操作 SQL 数据库像操作 list 一样自然**。怎么做到?用 S-expression 表示 SQL——把 SQL 变成 Elisp 数据,然后 emacsql 编译成 SQL 字符串发给后端。

支持的后端:SQLite(原生 Emacs 29+ 也内置了)、PostgreSQL、MySQL。最常用的是 SQLite——单文件、无服务、Emacs 29+ 内置 `sqlite` 模块后,emacsql 性能很好。

### 6.1 基础

```elisp
(require 'emacsql)

;; 连接(内存数据库)
(defvar my-db (emacsql-connect :sqlite))

;; 创建表
(emacsql my-db [:create-table people
                ([(name text) (age integer)])])

;; 插入
(emacsql my-db [:insert-into people
                :values [("Alice" 30) ("Bob" 25) ("Carol" 35)]])

;; 查询
(emacsql my-db [:select [name age]
                :from people
                :where (> age 28)])
;; → (("Alice" 30) ("Carol" 35))
```

逐行讲。

`(emacsql-connect :sqlite)` 连 SQLite。`:sqlite` 关键字指定后端。不传文件路径 = 内存数据库(进程退出就没)。要持久化:

```elisp
(emacsql-connect :sqlite "~/my.db")
```

返回 connection 对象,你存到变量里后面用。

`emacsql` 是核心函数,签名 `(emacsql DB SQL-EXPR &rest ARGS)`。DB 是 connection,SQL-EXPR 是 vector(注意是 vector,不是 list!)。ARGS 是参数化查询的值。

`[:create-table people ([(name text) (age integer)])]` 是 SQL DDL 的 S-expression 形式。`[]` 表示 vector(emacsql 用 vector 因为它要区分"代码"和"数据"),`:create-table` 是 SQL 关键字(以 `:` 开头是 keyword),`people` 是表名(symbol),`([(name text) (age integer)])` 是列定义——每个 `(name type)` 一列。

为什么用 vector 不用 list?**因为 Elisp list 经常被求值**。vector 在 Elisp 里是自求值对象(`[a b c]` 求值还是 `[a b c]`)。emacsql 把 vector 当成"数据",在 macro/function 里展开成 SQL。如果用 list,你写 `'(:create-table ...)`,得 quote,而且 macro 展开会冲突。vector 干净。

`[:insert-into people :values [("Alice" 30) ("Bob" 25)]]` 插入多行。`[("Alice" 30) ...]` 是 row list,每行是一个 vector `[...]`。

`[:select [name age] :from people :where (> age 28)]` 是 SELECT。`[name age]` 是 column list(投影)。`:from people` 指定表。`:where (> age 28)` 是 WHERE 条件——`(> age 28)` 是**Elisp 表达式**,但 emacsql 把它"翻译"成 SQL `WHERE age > 28`。这是 emacsql 的魔法——你在 `:where` 里写的 Lisp 表达式会被分析、转成 SQL。

返回 `(("Alice" 30) ("Carol" 35))`——每行是一个 list,每个值是元素。Alice 30 岁(>28 满足),Bob 25(不满足),Carol 35(满足)。注意是 list of list,不是 vector——返回值用 list 因为 Elisp 处理 list 更顺手。

### 6.2 CRUD 全套

**Create (插入)**:

```elisp
(emacsql my-db [:insert-into people :values [("Alice" 30)]])
(emacsql my-db [:insert-into people (name age) :values [("Alice" 30)]])  ; 显式列
(emacsql my-db [:insert-into people (name) :values [("Dave")]])          ; 部分列
(emacsql my-db [:insert-into people :values [("Eve" 28)] :or-replace])   ; INSERT OR REPLACE
```

**Read (查询)**:

```elisp
(emacsql my-db [:select * :from people])                      ; 全部
(emacsql my-db [:select [name] :from people])                 ; 一列
(emacsql my-db [:select [name age] :from people
                :where (and (> age 25) (< age 40))])          ; 范围
(emacsql my-db [:select [name] :from people
                :order-by [age desc]                          ; 排序
                :limit 5])                                     ; 限制
(emacsql my-db [:select [(* count *)] :from people])          ; COUNT(*)
(emacsql my-db [:select [name] :from people
                :group-by [name]])                             ; GROUP BY
```

注意 `(* count *)` 这种写法——emacsql 把 SQL 函数用 `(* funcname args *)` 包起来,避免和 Elisp 函数调用混淆。

**Update**:

```elisp
(emacsql my-db [:update people
                :set [age 31]
                :where (= name "Alice")])
```

**Delete**:

```elisp
(emacsql my-db [:delete-from people :where (= name "Alice")])
```

### 6.3 事务

emacsql 支持事务:

```elisp
(emacsql-with-transaction my-db
  (emacsql my-db [:insert-into accounts :values [("alice" 100)]])
  (emacsql my-db [:insert-into accounts :values [("bob" 200)]])
  ;; 如果上面任何一句出错,整个事务回滚
  )
```

`emacsql-with-transaction` 是宏,wrap 在 BEGIN/COMMIT 里,任何错误自动 ROLLBACK。

### 6.4 参数化查询(防 SQL 注入)

不要拼字符串:

```elisp
;; 不要这样!
(emacsql my-db
         (vector :select :* :from 'people
                 :where `(= name ,user-input)))   ; 容易注入
```

应该用 emacsql 的参数化:

```elisp
(emacsql my-db [:select * :from people :where (= name $s1)]
         "Alice")
;; $s1 是 placeholder,1 表示第 1 个 vararg
;; s 表示 string(r = raw, i = int, etc.)
```

或者用 backquote + vector:

```elisp
(emacsql my-db `[:select * :from people :where (= name ,name)])
```

emacsql 看到 `[,name]` 会自动参数化,不会注入。**这是 emacsql 比"拼 SQL 字符串"安全的地方**。

### 6.5 实战:todo 应用

```elisp
(require 'emacsql)

(defvar my-todo-db (emacsql-connect :sqlite "~/todo.db")
  "Todo database connection.")

(emacsql my-todo-db [:create-table-if-not-exists todos
                     ([(id integer :primary-key)
                       (title text)
                       (done integer :default 0)
                       (created text :default (current_timestamp))])])

(defun my-todo-add (title)
  "Add a new todo with TITLE."
  (emacsql my-todo-db `[:insert-into todos (title)
                        :values ([(,title)])]))

(defun my-todo-list ()
  "Return all unfinished todos."
  (emacsql my-todo-db [:select [id title]
                       :from todos
                       :where (= done 0)
                       :order-by [created]]))

(defun my-todo-done (id)
  "Mark todo ID as done."
  (emacsql my-todo-db [:update todos
                       :set [done 1]
                       :where (= id $i1)]
           id))

(defun my-todo-search (substring)
  "Search todos by title substring."
  (emacsql my-todo-db [:select [id title]
                       :from todos
                       :where (like title $s1)
                       :order-by [created]]
           (format "%%%s%%" substring)))
```

逐行讲。

`(defvar my-todo-db (emacsql-connect :sqlite "~/todo.db") ...)` 定义全局变量存 DB connection。`:sqlite "~/todo.db"` 用文件持久化。注意 `defvar` 只在第一次加载时求值——如果你改了这个表达式,得 `eval-defun` 重新求值,或者重启 Emacs。

`[:create-table-if-not-exists todos (...)]` 创建表,IF NOT EXISTS 保证不会重复创建。每次启动都跑这句是安全的——是初始化模式。

列定义 `[(id integer :primary-key) (title text) (done integer :default 0) (created text :default (current_timestamp))]`。每列是 `(名字 类型 约束...)`。`:primary-key` 标记主键(SQLite 自动加 autoincrement)。`:default 0` 设置默认值。`:default (current_timestamp)` 用 SQL 函数作默认值。

`my-todo-add` 用 backquote:` `[:insert-into todos (title) :values ([(,title)])] `。backquote + `,title` 把变量值插入 vector。emacsql 自动参数化(,title 不是字符串拼接,是绑定参数)。

`my-todo-done` 用显式 placeholder `$i1`。`$i` 表示 integer(`r` raw, `s` string, `f` float),`1` 是第 1 个 vararg。`(emacsql db sql id)` 把 `id` 作为 `$i1` 的值。

`my-todo-search` 用 SQL `LIKE` 模糊匹配。`(format "%%%s%%" substring)` 生成 `%substring%`——`%%` 是 format 的 escape,表示字面 `%`。

返回值始终是 list of list——比如 `my-todo-list` 返回 `((1 "buy milk") (3 "wash car"))`。你可以用 dash 处理:

```elisp
(defun my-todo-display ()
  "Display todos in a buffer."
  (->> (my-todo-list)
       (-map (lambda (row)
               (format "[%d] %s" (car row) (cadr row))))
       (s-join "\n")
       (message)))
```

### 6.6 emacsql vs alist/plist

什么时候用 emacsql?**当数据量超过 1000 行,或者需要复杂查询(过滤、排序、聚合、join)**。

什么时候用 alist / plist?**当数据量小(<1000),结构简单(单层 key→value),或者需要 Elisp 原生操作**(比如 `(assoc 'foo list)` 在 plist 上比 SQL `SELECT WHERE` 快)。

中间地带:如果你的数据是 list of records(比如 1000 个联系人),用 emacsql 通常比用 list of alist 快——SQL 索引比 Elisp 的 `-filter` 快得多。但 emacsql 有"连接 + 序列化"开销,小数据反而慢。

实战建议:**用 emacsql 做持久化**(任何要存盘的数据),**用 Elisp list 做内存运算**(查询出来的结果用 dash 处理)。这是 emacsql 的设计哲学——SQL 用来"查",Elisp 用来"算"。

### 6.7 陷阱

**陷阱 1:emacsql 不是 ORM**。它不映射对象到表,不自动 join,不缓存。你写的就是 SQL,只是用 S-expression 写。如果你想要 ORM,得用 `org-ql` 或别的。

**陷阱 2:DB connection 是有状态的**。`(emacsql-connect :sqlite "x.db")` 打开文件,但**不会自动关闭**。Emacs 退出时 OS 清理,但中间如果 db 文件被其他进程占用,你会拿到锁错误。`emacsql-close` 显式关闭。

**陷阱 3:vector 不是 list**。`[:select * :from x]` 是 vector,不能用 `'(:select ...)`,也不能用 `'[:select ...]`(quote 一个 vector 是冗余的,但不会出错)。直接写 `[:select ...]` 就行。

**陷阱 4:`emacsql` 的 SQL 方言不完全跨后端**。SQLite 和 PostgreSQL 的某些语法不同(比如 `ON CONFLICT` vs `ON DUPLICATE KEY`),emacsql 尽量统一但没法完全。如果你写 PostgreSQL 特定查询,切到 SQLite 可能失败。

**陷阱 5:`:default (current_timestamp)` 在不同后端行为不同**。SQLite 接受,但 Postgres 需要 `CURRENT_TIMESTAMP`(大写无括号)。emacsql 文档建议用 `(datetime)` 或 `(now)` 抽象。

**陷阱 6:emacsql-sqlite (subprocess) vs emacsql-sqlite3 (builtin)**。Emacs 29+ 内置 sqlite 模块,emacsql 优先用它(快)。但 Emacs 28 及以下得用 subprocess 后端(慢,且并发有限)。`M-: (featurep 'sqlite)` 看你是不是内置。

---

## Part 7: async.el —— 异步编程

async.el 是 John Wiegley 2012 年写的。Wiegley 是 Emacs 社区的元老——他写了 `use-package`(Emacs 配置事实标准)、`auth-source`(认证管理)、还当过 Emacs 维护者(2014-2018)。

要理解 async.el,得先理解 Elisp 的并发模型。Emacs 26 之前**完全单线程**——你不能 fork 一个线程跑后台任务。Emacs 26 加了 `make-thread`,但这是**协作式线程**(线程只在显式 yield 点切换),不是真并发。而且大多数 Elisp 函数不是 thread-safe,你用 thread 容易 race condition。

async.el 用另一种思路:**用子进程跑 Elisp**。Emacs 可以通过 `start-file-process` / `make-process` 启动一个子进程(可以是另一个 emacs 实例),通过 stdin/stdout 通信。async.el 把这个机制包装成"在子进程里跑 lambda,结果回主进程"。

这听起来很 hack,但实际很好用——比如 byte-compile 一个大文件、跑 lint、扫描文件系统——这些操作不需要"并发",只需要"不卡 UI",async.el 完美。

### 7.1 基础

```elisp
(require 'async)

(async-start
 (lambda ()
   ;; 在子进程里跑
   (shell-command-to-string "find ~ -name '*.org'"))
 (lambda (result)
   ;; 在主进程里跑
   (message "Found orgs: %s" result)))
```

逐行讲。

`(async-start WORK FINISH)` 启动子进程跑 WORK 函数,WORK 完成后,结果序列化传回主进程,FINISH 函数以结果为参数被调用。

第一个 lambda 在**子进程**里跑。它做 `shell-command-to-string`——在子进程里 shell out,不会卡主进程。

第二个 lambda 在**主进程**里跑。它的参数是第一个 lambda 的返回值。

机制是什么?async.el 启动一个 `emacs --batch` 子进程,把第一个 lambda 的源码序列化(stdin 写 S-expression),子进程加载、求值、把返回值序列化写回 stdout。主进程 read stdout,把结果传给第二个 lambda。**这是 IPC,不是 thread**。

**重要限制**:lambda 之间传递的是 S-expression,所以**返回值必须是可打印的**——string、number、symbol、list、vector。**不能传 buffer、marker、window、process、frame**——这些是主进程的运行时对象,无法跨进程。

### 7.2 实战:异步 byte-compile

```elisp
(async-start
 (lambda ()
   (let ((default-directory user-emacs-directory))
     (byte-recompile-directory "." 0)))
 (lambda (result)
   (message "Compile done: %s" result)))
```

`(let ((default-directory user-emacs-directory)) ...)` 设置当前目录为 `.emacs.d`。`default-directory` 是 buffer-local 变量,所有相对路径基于它。`user-emacs-directory` 是 Emacs 配置目录,通常是 `~/.emacs.d/`。

`(byte-recompile-directory "." 0)` 递归 byte-compile 这个目录下所有 .el。`0` 表示"只重编译比 .elc 新的"。

这操作在大配置上要几分钟。在主进程跑会卡 UI——`M-x` 切窗口都卡。在 async 里跑,主进程继续响应,完成时 message 提示。

### 7.3 子进程隔离

子进程**完全独立**——它不知道主进程的 buffer、变量、mode 设置。你**不能**在 WORK lambda 里访问主进程的 buffer:

```elisp
;; 不会工作
(async-start
 (lambda ()
   (with-current-buffer "*scratch*"
     (buffer-string)))      ; 错!子进程没有 *scratch*
 (lambda (result) ...))
```

子进程是个 fresh emacs,没有你的 init.el(默认 async 不加载 init),没有你的 package。它只有 `async-before-async-action-function` 这些 hook 加载的东西。

如果你需要某些包,得在 lambda 里 require:

```elisp
(async-start
 (lambda ()
   (require 'f)
   (require 'dash)
   (->> (f-files "/some/path")
        (-filter (lambda (f) (f-ext? f "org")))
        (length)))
 (lambda (count)
   (message "Found %d org files" count)))
```

`require` 会在子进程的 `load-path` 里找 f 和 dash。**子进程的 load-path 默认是从 `exec-path` 推导的**,可能和你主进程不同。要保证一致,用 `(async-inject-var "load-path")` 或 `async-inject-environment`。

### 7.4 async-start 的变体

`(async-start-process NAME PROGRAM FINISH &rest ARGS)` 是另一个 API——启动外部程序(不是 Elisp lambda),收集输出:

```elisp
(async-start-process
 "grep"
 "grep"
 (lambda (proc)
   (message "grep done: %s"
            (with-current-buffer (process-buffer proc)
              (buffer-string))))
 "-r" "TODO" "~/projects/")
```

启动 `grep -r TODO ~/projects/`,完成后调用 callback。这相当于 `make-process` 的封装,但 API 简单。

### 7.5 async vs threads

Emacs 26+ 的 `make-thread`:

```elisp
(make-thread
 (lambda ()
   (sleep-for 5)
   (message "Done")))
```

这创建一个 thread,5 秒后 message。**但这 thread 不是并发的**——Elisp 是 GIL-like,只有一个 thread 在跑,其他在 wait(yield 点切换)。所以 `(sleep-for 5)` 期间这个 thread yield,主 thread 跑;5 秒后这个 thread 唤醒,等下次 yield 抢占。

thread 适合 **IO bound**(网络、磁盘等待)任务。async.el 适合 **CPU bound**(byte-compile、大计算)任务——真正占用另一个 CPU 核心。

实战:**优先用 async**。thread 的 API 不稳定,很多函数 thread-unsafe。async 用子进程,完全隔离,出问题不影响主进程。

### 7.6 陷阱

**陷阱 1:lambda 必须可序列化**。lambda 是 closure,它的 captured variables 跨进程无效。所以 async lambda 里访问主进程 lexical 变量会失败:

```elisp
(let ((x 10))
  (async-start
   (lambda ()
     (* x 2))   ; 错!x 在子进程不存在
   (lambda (result) ...)))
```

正确做法:把 x 作为参数 inline 进 lambda:

```elisp
(let ((x 10))
  (async-start
   `(lambda ()
      (* ,x 2))    ; backquote 把 x 的值插入
   (lambda (result) ...)))
```

注意 lambda 变成 backquote 形式——它实际上是构造一个新 list,把 x 的值嵌入。但这破坏 lexical binding(closure 失效),所以**只能 inline 简单值**(number、string、list)。

**陷阱 2:错误处理**。子进程出错,FINISH callback 不会被调用。你得自己检查 `(process-exit-status proc)`:

```elisp
(async-start
 (lambda () (error "boom"))
 (lambda (result)
   ;; 不会被调用!
   (message "Got: %s" result)))
```

要捕获错误,在 lambda 里用 `condition-case`:

```elisp
(async-start
 (lambda ()
   (condition-case err
       (do-something)
     (error (cons 'error (error-message-string err)))))
 (lambda (result)
   (if (and (consp result) (eq (car result) 'error))
       (message "Subprocess failed: %s" (cdr result))
     (message "Got: %s" result))))
```

**陷阱 3:子进程是 batch mode**。子进程的 emacs 是 `--batch`(无 UI、无 init),所以 `find-file`、`switch-to-buffer` 这些 UI 操作行为不同。`find-file` 在 batch mode 下不打开窗口,但仍创建 buffer,有时报错(比如 file 不存在)。

**陷阱 4:stdout 是 IPC 通道**。async.el 用子进程的 stdin/stdout 通信。**你的 lambda 不能往 stdout 写东西**——`(message "foo")` 默认写 stderr 还好,但 `(princ "foo")` 写 stdout 会污染 IPC 协议,async.el 解析会失败。

**陷阱 5:启动开销**。每次 async-start 启动一个 emacs,要几百毫秒。所以 async 不适合**短任务**(<100ms)——开销大于收益。适合**秒级以上**的任务。

---

## Part 8: package-lint —— 发布前必跑

如果你打算把包发到 MELPA,有个工具必跑:`package-lint`。这是 Steve Purcell 写的——Purcell 是 Emacs 社区"老前辈", 维护 MELPA 多年,自己写了几十个包(prelude、exec-path-from-shell、...),他写的 lint 工具基本是社区标准。

`package-lint` 检查你的 .el 文件,看是否符合 MELPA 提交规范。它不检查 bug——它检查"你的包会不会被 MELPA reviewer 拒"。

### 8.1 用法

```elisp
M-x package-install RET package-lint RET
M-x package-lint-current-buffer
```

或者在命令行:

```bash
emacs --batch --eval "(progn (package-initialize) (package-install 'package-lint))" \
      --file my-package.el \
      --eval "(package-lint-current-buffer)"
```

它会输出一系列 warning / error,告诉你哪里需要改。

### 8.2 它检查什么

**Header 完整性**:`Package-Requires`、`Version`、`URL`、`Keywords` 这些必填字段是否齐全。缺一个就 error。

**Emacs 版本兼容**:`Package-Requires: ((emacs "27.1"))` 声明要 27.1,但你用了 28+ 才有的函数(比如 `use-package`),会 error。

**cl vs cl-lib**:老 `cl.el`(已弃用)vs `cl-lib.el`(现代)。用了 `cl.el` 会 warn:应该用 `cl-lib`。

**未声明变量**:你用了 `package-user-dir` 但没 `(require 'package)`,会 warn。

**Deprecated API**:用了 `flet`(已弃用)而不是 `cl-flet`,会 warn。

**Scheme 形式**:文件开头 `;;; foo.el --- description -*- lexical-binding: t; -*-` 的格式。

**License**:必须有 `SPDX-License-Identifier` 或 `;; This file is free software...` 的声明。

### 8.3 配合 checkdoc 和 flycheck-package

`checkdoc` 是 Emacs 内置的 docstring 检查器。`M-x checkdoc` 在当前 buffer 跑,告诉你 docstring 哪里不规范:

- 函数有 args,docstring 必须按 args 顺序解释
- 第一行要是一句话总结
- 不要用"Return"开头(冗余,函数都 return)

`flycheck-package` 是 Purcell 写的另一个,把 package-lint + checkdoc 整合到 flycheck——你写代码时实时 lint,而不是手动跑。

实战配置:

```elisp
(use-package flycheck-package
  :after flycheck
  :config
  (flycheck-package-setup))
```

然后写包时,不规范的 docstring 会自动加波浪线。

### 8.4 实战:跑 package-lint 修一个包

假设你写了个包:

```elisp
;;; my-package.el --- Do cool stuff

;; Author: Me <me@example.com>

;;; Code:

(defun my-do-thing (x)
  "Do thing with X."
  (loop for i from 1 to x collect i))

(provide 'my-package)
;;; my-package.el ends here
```

跑 `M-x package-lint-current-buffer`,会报:

```
my-package.el:1:0: Error: Package-Requires header missing
my-package.el:6:11: Warning: Use cl-lib instead of cl
my-package.el:6:11: Error: `loop' is deprecated; use `cl-loop'
```

修:

```elisp
;;; my-package.el --- Do cool stuff -*- lexical-binding: t; -*-

;; Copyright (C) 2026 Me

;; Author: Me <me@example.com>
;; Version: 0.1
;; Package-Requires: ((emacs "27.1") (cl-lib "0.5"))
;; SPDX-License-Identifier: GPL-3.0-or-later

;;; Code:

(require 'cl-lib)

(defun my-do-thing (x)
  "Return list of numbers from 1 to X."
  (cl-loop for i from 1 to x collect i))

(provide 'my-package)
;;; my-package.el ends here
```

再跑 lint,没 error 了。这是发 MELPA 的基本要求。

### 8.5 陷阱

**陷阱 1:package-lint 不是 ERT**。它不测试你的代码对不对,只检查形式。要测对错,用 ert(参考 ert-testing.md)。

**陷阱 2:checkdoc 过度严格**。它要求每个函数有 docstring,args 在 docstring 里按顺序解释。有些 helper 函数(明显到不需要解释)也会被警告。你可以加 `;; checkdoc-params: ...` 或在 docstring 里用规范格式满足。

**陷阱 3:flycheck-package 性能**。实时 lint 大 buffer 时卡。可以 `(setq flycheck-package-checkers nil)` 关掉某些 checker,或者只在 save 时 lint。

---

## Part 9: 整合实战 —— 文件 metadata 数据库

下面把 dash + s + f + emacsql 整合起来,写一个**真实**工具——扫描某个目录,把所有文件的元数据(path、size、mtime、ext)存到 SQLite,提供查询接口。

这是 dash/s/f/emacsql 最典型的应用场景:**数据采集(f)+ 数据加工(dash + s) + 持久化(emacsql)**。

```elisp
;;; my-files-db.el --- Scan files into a queryable SQLite database -*- lexical-binding: t; -*-

;; Copyright (C) 2026 Me

;; Author: Me <me@example.com>
;; Version: 0.1
;; Package-Requires: ((emacs "27.1") (dash "2.19") (s "1.13") (f "0.20") (emacsql "3.0"))
;; SPDX-License-Identifier: GPL-3.0-or-later

;;; Commentary:
;; A small tool for scanning files into SQLite for ad-hoc queries.

;;; Code:

(require 'dash)
(require 's)
(require 'f)
(require 'emacsql)

(defvar my-files-db nil
  "Database connection for file metadata.")

(defun my-files-db-init (db-path)
  "Initialize the database at DB-PATH."
  (setq my-files-db (emacsql-connect :sqlite db-path))
  (emacsql my-files-db
           [:create-table-if-not-exists files
            ([(path text :primary-key)
              (size integer)
              (mtime text)
              (ext text)
              (depth integer)])]))

(defun my-files-scan (dir)
  "Scan DIR recursively, populate the database."
  (unless my-files-db
    (error "Call `my-files-db-init' first"))
  (let ((base-depth (length (s-split "/" (f-long dir)))))
    (->> (f-files dir nil :recursive)
         (-map (lambda (file)
                 (let* ((long (f-long file))
                        (size (f-size file))
                        (mtime (format-time-string
                                "%Y-%m-%dT%H:%M:%S"
                                (f-mtime file)))
                        (ext (or (f-ext file) ""))
                        (depth (- (length (s-split "/" long))
                                  base-depth)))
                   (list long size mtime ext depth))))
         (-filter (lambda (row) (> (cadr row) 0)))    ; skip empty files
         (--map (emacsql my-files-db
                        `[:insert-or-ignore-into files
                          (path size mtime ext depth)
                          :values ([(,@it)])])))))

(defun my-find-large-files (min-size)
  "Return paths of files larger than MIN-SIZE bytes."
  (->> (emacsql my-files-db
               `[:select [path size]
                 :from files
                 :where (> size ,min-size)
                 :order-by [size desc]])
       (-map #'car)))

(defun my-files-by-ext ()
  "Return list of (ext count total-size)."
  (emacsql my-files-db
           [:select [ext (funcall count *) (funcall sum size)]
            :from files
            :group-by ext
            :order-by [(funcall count *) desc]]))

(defun my-recently-modified (days)
  "Return files modified within the last DAYS days."
  (let ((cutoff (format-time-string
                 "%Y-%m-%dT%H:%M:%S"
                 (time-subtract (current-time) (days-to-time days)))))
    (->> (emacsql my-files-db
                 `[:select [path mtime]
                   :from files
                   :where (> mtime ,cutoff)
                   :order-by [mtime desc]])
         (-map #'car))))

(defun my-deepest-files (n)
  "Return top N files by depth."
  (->> (emacsql my-files-db
               [:select [path depth]
                :from files
                :order-by [depth desc]
                :limit ,$1]
               n)
       (-map #'car)))

(provide 'my-files-db)
;;; my-files-db.el ends here
```

逐行讲。

`defvar my-files-db nil` 全局变量存 DB connection,初始 nil(还没 init)。

`my-files-db-init` 接收 DB 路径,connect,创建表。`:create-table-if-not-exists` 保证幂等(多次调用不报错)。

表结构 5 列:`path` 作主键(防止重复插入)、`size` 字节、`mtime` ISO 8601 字符串、`ext` 扩展名(无扩展的存空串)、`depth` 目录深度(用于"找最深文件")。

`my-files-scan` 是采集函数:

- `(unless my-files-db (error ...))` 防御性检查
- `(s-split "/" (f-long dir))` 把目录路径按 / 切分,算 base-depth。比如 `/home/user/project` 切成长度 3 的 list,base-depth=3。
- `(f-files dir nil :recursive)` 递归列文件,nil = 不预过滤。
- `(-map (lambda (file) ...))` 对每个文件构造 row。`f-long` 转绝对规范路径。`f-size` 取大小。`(format-time-string "%Y-%m-%dT%H:%M:%S" (f-mtime file))` 把 mtime 格式化成 ISO 8601 字符串(SQLite 可比较)。`f-ext` 取扩展,无扩展返回 nil,所以 `(or (f-ext file) "")` 兜底。`depth` = 全路径切分数 - base-depth。
- `(-filter (lambda (row) (> (cadr row) 0)))` 过滤掉 size=0 的(空文件)。
- `(--map (emacsql ...))` 插入每一行。`--map` 是 dash 的"带 anaphoric it"宏——lambda 里可以用 `it` 指代当前元素。`(,@it)` 是 backquote splice,把 row list 的元素展开成 vector 元素。`:insert-or-ignore-into` = `INSERT OR IGNORE`,重复 path 时跳过(因为 path 是主键)。

`my-find-large-files` 查大于 min-size 的。`(cadr row)` 取 size(`(car row)` 是 path,`(cadr row)` 是 size)。`(> size ,min-size)` 是动态条件,用 backquote 注入。

`my-files-by-ext` 用 SQL 聚合:`GROUP BY ext`,对每个 ext 算 count(*) 和 sum(size)。`[:select [ext (funcall count *) (funcall sum size)] ...]` emacsql 的函数语法是 `(funcall name args...)`,避免和 Elisp 函数调用混淆。

`my-recently-modified` 查 N 天内修改的。`(days-to-time days)` 把天数转 time 元组,`(time-subtract (current-time) ...)` 算截止时刻,format 成 ISO 字符串,塞进 SQL。`:where (> mtime ,cutoff)` 字符串比较 ISO 8601 是字典序但等价于时间序——这是为什么用 ISO 格式。

`my-deepest-files` 取前 N 深。`$1` placeholder,传 `n` 作为参数。

### 9.1 用法

```elisp
(my-files-db-init "~/files.db")
(my-files-scan "~/projects")
;; 扫描完后:
(my-find-large-files 1000000)        ; 找 >1MB 的文件
(my-files-by-ext)                    ; 按扩展名统计
(my-recently-modified 7)             ; 最近 7 天修改
(my-deepest-files 10)                ; 最深的 10 个文件
```

实战场景:

- 你想知道 `~/projects` 里哪些 .org 文件最大,可能要拆分或归档
- 你想统计源代码扩展分布,看看主要用什么语言
- 你想找最近一周改过的文件,做"周报"
- 你想找最深的目录,看是不是有"路径过长"的 Windows 不兼容问题

这就是真实工具的样子——**不是 5 行 hello world,是 100 行解决一个真问题**。

### 9.2 设计取舍

**为什么用 SQLite 而不是 alist?** 数据量大(`~/projects` 几万文件),alist 查询 O(n),SQLite 索引 O(log n)。而且下次启动直接 attach 同一个 .db,不用重新扫描。

**为什么用 emacsql 而不是 sqlite3 命令?** 命令行要 fork 子进程,IPC 开销大。emacsql 直接调用内置(Emacs 29+)sqlite 模块,函数调用开销。而且 emacsql 防注入。

**为什么扫描后内存只保留结果 list?** emacsql 把数据存盘(磁盘),查询返回 list(内存)。你处理 list 用 dash。这是 Elisp + SQL 的混合范式——SQL 存查,Lisp 算。

**为什么不存更多元数据(mime type、hash)?** 取决于需求。本例只演示基础。要 hash 得 `(secure-hash 'sha256 (f-read file))`,但这对大文件慢——可能要 async。

---

## Part 10: 总结

这一节我们走了七个 Elisp 社区的核心外部包:`dash`、`s`、`f`、`request`、`emacsql`、`async`、`package-lint`。每个都有它的历史、它的核心 API、它的陷阱。

回顾一下整体图景:

- **dash** 让 list 操作像 Clojure。thread macro 是杀手锏。
- **s** 让 string 操作不痛苦。`s-replace` / `s-trim` / `s-split` 是日常。
- **f** 让 file 操作单行搞定。`f-files :recursive` 神特性。
- **request** 让 HTTP 不再写 callback hell。`cl-function` + keyword 参数好读。
- **emacsql** 让数据持久化优雅。S-expression SQL 是好抽象。
- **async** 让长任务不卡 UI。子进程模型简单可靠。
- **package-lint** 让你的包能发 MELPA。

**这些是事实标准**——不在 Emacs 核心,但人人用。学懂这些,你读别人代码快十倍(现代包都用这些库),你自己写包省一半时间(不用造轮子),你融入社区(看到 dash 风格不懵)。

但**别滥用**。一个 50 行的小工具,如果依赖 dash + s + f + cl-lib + map + seq + s.el + f.el 八个库,这是反面教材——用户为了你 50 行代码,装八个依赖。判断标准:

1. **如果只用一两个函数**,别依赖。原生 + 几行手写代码就行。
2. **如果代码主要做 list/string/file 操作**,dash/s/f 值得。
3. **如果代码主要做 buffer/window/mode 操作**,不需要这些。
4. **每次加依赖,想想用户**——他们 MELPA install 时,会拉一串 transitive deps,任何一个 broken 你的包就坏。

最后给个学习路径:

1. **先把 dash/s/f 装上,日常 config 里用**。这是最容易上手的。
2. **thread macro(`->>` / `->`)练熟**。这是 dash 最大价值。
3. **写一个 HTTP 工具用 request**。比如查 GitHub、查天气预报。
4. **写一个有持久化的应用用 emacsql**。todo / 笔记 / 书签 都行。
5. **写一个长任务用 async**。比如全目录搜索、批量 byte-compile。
6. **写完了跑 package-lint + checkdoc**,养成习惯。

到这一步,你写的包已经"长得像"社区标准包了。下一节我们会讲怎么把它发布到 MELPA。

参考:

- dash: https://github.com/magnars/dash.el
- s.el: https://github.com/magnars/s.el
- f.el: https://github.com/rejeep/f.el
- request: https://github.com/tkf/emacs-request
- emacsql: https://github.com/skeeto/emacsql
- async: https://github.com/jwiegley/emacs-async
- package-lint: https://github.com/purcell/package-lint

作者背景:

- **Magnar Sveen** — Emacs Rocks! 视频作者,Norway。dash/s/f 都是他的作品,还有 multiple-cursors、expand-region、wrap-region。他的代码风格鲜明,极大地影响了 2010 年代 Elisp 社区。
- **Takafumi Arakaki (@tkf)** — 日本物理学家,Emacs Python 生态核心。request、jupyter-emacs、epc 都是他的。
- **Christopher Wellons (@skeeto)** — 美国 NASA 航天软件工程师。emacsql、pdf-tools、elfeed、lfs.c 等高质量作品。代码风格极严谨。
- **John Wiegley** — Emacs 维护者(2014-2018),use-package 作者。async、auth-source、bbdb 都和他有关。
- **Steve Purcell (@purcell)** — MELPA 共同维护者。package-lint、flycheck-package、exec-path-from-shell 等几十个包。

学完这一节,你已经掌握了 Elisp 生态的"日常工具箱"。下一节进入 capstone——把这些工具组合起来做一个完整的包,发到 MELPA。
