# Silencing Prompts: 关掉 Emacs 的"确认"烦扰

> 学完这个文件,你能精准控制 Emacs 何时问你、何时直接执行

---

## 0. 为什么 Emacs 这么爱问

如果你从 VS Code 或 Vim 切到 Emacs,第一件让你崩溃的不是键位——是**确认弹窗**。

- 打开大文件? "really open? (y/n)"
- 杀进程? "kill processes? (y/n)"
- 退出 Emacs? "Active processes exist, kill them? (yes/no)"
- 文件有 local variables? "Set them? (! ? y n ...)"
- Projectile 保存缓存? "Delete cache? (y/n)"
- Dired 删文件? "Delete file X? (y/n)"

**每分钟能弹 5 次**。这不是 bug,是设计。

### 0.1 Emacs 的"谨慎"哲学

1976 年的 Emacs 设计假设: **用户会犯致命错误**。`C-x C-c` 退出会丢失所有未保存 buffer,所以问一下;`delete-file` 不可恢复,所以问一下;file-local variables 可能含恶意代码,所以问一下。

这是合理的——**1985 年的硬件和软件**。当时没有 Git (1985 没诞生),没有 autosave (有了但慢),没有 universal-undo-tree。删了就是删了,退出就是退出。

### 0.2 2026 年的现实

现在:
- Git 让删除可恢复
- `undo-tree` 让 99% 操作可撤销
- autosave + backup 双保险
- Emacs 进程可以 daemon 模式永不退出

但 Emacs 的默认行为没改 (向后兼容)。**默认值是给新手保护,你可以手动调到 expert 模式**。

这个文件教你怎么调。

---

## 1. 确认的来源 (Lisp 视角)

理解 prompt 从哪来,才能精准关掉它。Emacs 的"确认"分四类:

### 1.1 `y-or-n-p` / `yes-or-no-p` 函数

最直接的来源——某段代码主动调:

```elisp
(y-or-n-p "Delete file? ")        ;; → "Delete file? (y or n)"
(yes-or-no-p "Are you sure? ")    ;; → "Are you sure? (yes or no) "
```

这两个函数**故意**不同。`y-or-n-p` 按一个字符 (`y` 或 `n`),快。`yes-or-no-p` 要打完整 "yes" 或 "no",慢但**强制你思考**。

为什么 Emacs 区分这两个? `yes-or-no-p` 用于**不可逆**操作 (退出 Emacs、删整个目录),`y-or-n-p` 用于**可恢复**操作 (关一个 buffer)。这是 Emacs 给开发者的 API 约定。

Doom / Spacemacs / Prelude 都做了 `(setq use-short-answers t)`,让所有 `yes-or-no-p` 自动降级为 `y-or-n-p`。**这本身已经是"少烦"的第一步**。

### 1.2 `confirm-*` 变量

Emacs 内置一组变量,控制特定操作的"是否确认":

| 变量 | 控制什么 | 默认 |
|---|---|---|
| `confirm-kill-emacs` | `C-x C-c` 时问 | nil (不问) |
| `confirm-kill-processes` | 退出时杀进程问 | t (问) |
| `confirm-nonexistent-file-or-buffer` | `C-x b` 创建新 buffer 问 | t (问) |
| `confirm-elisp-eldoc-defun-default` | eldoc 操作确认 | nil |

这些是**用户可调**的——Emacs 把"是否确认"的决策权交给你。

### 1.3 `warning-*` 变量

不是确认,但**会弹消息**:

```elisp
(setq warning-minimum-level :emergency)
```

减少 warning 显示。

### 1.4 各包自定义的 prompt

有些包**自己**加 prompt,不走标准 API:

- Projectile: `projectile-cache-delete-prompt-function`
- Magit: `magit-no-confirm`
- Dired: `dired-deletion-confirms`
- Org: `org-confirm-babel-evaluate`

这些要**每个包单独关**——没有统一的"全局关闭"。

---

## 2. 精准关闭 (推荐)

这是**安全**的做法: 保留保护性确认,关掉烦人的。

加到你的 init.el (Doom 用户加到 `~/.config/doom/config.el` 或 `~/.doom.d/config.el`):

### 2.1 全局基础设置

```elisp
;;; silencing-prompts.el -*- lexical-binding: t; -*-

;; yes/no → y/n (Emacs 自动确认更短)
(setq use-short-answers t)
```

这一行**单独**就能让 90% 的 prompt 变短。`yes-or-no-p` 内部检查这个变量,如果 `t`,改用 `y-or-n-p` 的 UI。这是 Emacs 28 (2022) 的新特性,你装的 30.2 肯定支持。

为什么不直接 `(defalias 'yes-or-no-p 'y-or-n-p)`? 因为某些场合 Emacs 仍然**故意**用长版 (如 `confirm-kill-emacs` 时),强制你打字思考。`use-short-answers` 是 Emacs 官方的"折中",更稳。

### 2.2 关掉退出时的确认

```elisp
;; 退出 Emacs 时不问杀进程
(setq confirm-kill-processes nil)

;; 退出 Emacs 不二次确认 (慎用! C-x C-c 直接退)
;; (setq confirm-kill-emacs nil)
```

`confirm-kill-processes` 是**最烦**的一个。你 `C-x C-c` 时,如果有一个 `*shell*` 或 `* compilation*` 还活着,Emacs 弹 "Active processes exist; kill them and exit anyway? (yes or no)"。每次都要打 "yes"。

为什么有这个 prompt? 1990 年代编译一小时,C-c 误退是灾难。现在编译都在 CI 跑,本地进程死掉没所谓。**关掉合理**。

但 `confirm-kill-emacs` 我**建议保留** (如果你容易误按 C-x C-c)。C-x C-c 是**真正的核弹**——一旦退出,所有 buffer 状态、所有未保存的修改都没了 (除了 backup)。这个 prompt 是最后一道防线。

如果你用 daemon + emacsclient (Module 0 学过),`C-x C-c` 只关 frame,不杀 daemon,那这个 prompt 没意义,可以关。

### 2.3 关掉文件/buffer 创建确认

```elisp
;; C-x b 输入新名字,直接创建,不问
(setq confirm-nonexistent-file-or-buffer nil)
```

`C-x b foo` 切到 foo buffer,如果 foo 不存在,Emacs 默认问 "Buffer foo does not exist. Create? (y or n)"。一次 y/n 看似小事,但**你一天切 buffer 几百次**,累积烦人。

关掉后,直接创建。**反正 buffer 创建是廉价的** (一个内存对象)。

### 2.4 关掉大文件警告

```elisp
(setq large-file-warning-threshold nil)
```

默认打开 > 100MB 文件时,Emacs 弹 "File foo.bin is large (XXX MB), really open? (y or n)"。

为什么有这个? Emacs 加载大文件慢 (gap buffer 优化好,但 syntax 属性 + text property 加载会拖累)。100MB 的 JSON 打开可能卡 30 秒。

如果你**知道**自己在开大文件 (例如日志分析),关掉这个 prompt 省事。如果**不知道** (误点击),警告保护你。

**折中**: 把阈值调高 (例如 500MB):

```elisp
(setq large-file-warning-threshold 500000000)
```

或者用 `M-x vlf` (Very Large File) 包,只加载部分。

### 2.5 信任 file-local variables

```elisp
;; 自动信任所有 file-local variables (危险但省事)
(setq enable-local-variables :all)

;; 或者: 只信任安全变量
;; (setq enable-local-variables t)
```

这是 Emacs **最重要的安全机制之一**。文件头部可以写:

```
;; -*- lexical-binding: t; eval: (malicious-code); -*-
```

`eval:` 这种 file-local variable 在你打开文件时**自动执行**——如果文件来自陌生人,可能跑恶意代码。Emacs 默认问 "The following local variables are risky: ... allow? (y/n/!)"。

`!` 表示"这次允许并永久记住",但仍每次新文件都问。

关掉策略:

- `(setq enable-local-variables :all)` — 信任所有,最快但**最危险**
- `(setq enable-local-variables t)` — 信任,但 risky 的仍问
- 默认 nil — 都问

**建议**: 如果你的项目都是自己写的,`:all` 安全。如果你 clone 别人的代码,保留默认——file-local variables 是真实攻击面 (CVE 历史上有相关漏洞)。

### 2.6 Projectile 缓存

Projectile 会缓存项目文件列表加速。缓存过期时问 "Delete old cache? (y/n)"。

```elisp
(setq projectile-cache-delete-prompt-function #'ignore)
```

`projectile-cache-delete-prompt-function` 是个函数变量,调它问"要不要删旧缓存"。设为 `#'ignore` (一个内置 no-op 函数),它返回 nil (不问),projectile 直接删。这是**精准关 prompt** 的典型模式——找到 prompt 背后的函数,替换成 no-op。

### 2.7 Dired 递归和删除

```elisp
(setq dired-deletion-confirms nil         ; 不逐个确认删除
      dired-recursive-deletes 'always     ; 递归删不问
      dired-recursive-copies 'always      ; 递归复制不问
      dired-clean-confirm-killing-deleted-buffers nil)
```

Dired 删文件时,每个文件问一次 (`D` 标记后 `x` 执行,默认逐个 y/n)。`dired-deletion-confirms nil` 让 Dired 一次执行所有标记,不问。

`dired-recursive-deletes 'always` 让删目录时自动递归 (`top` 是顶层问,`always` 是永不问,`nil` 是每次问)。这是**潜在的核弹**——你 `D` 一个目录,里面 1000 个文件全删,如果目录里有重要东西就完了。

我的建议: `'always` + Git。如果你不用 Git,设 `'top` (只问顶层目录,内部不问)。

### 2.8 Magit 不问

Magit 也有自己的 prompt 系统:

```elisp
(setq magit-no-confirm '(stage-all-changes
                          magit-push-current
                          delete-some-branch))
```

`magit-no-confirm` 是个 symbol list,列哪些操作**不问**。常用的:

- `stage-all-changes` — `S` (magit-stage-modified) 不问 "Stage all?"
- `magit-push-current` — `P u` 推时不问
- `delete-some-branch` — 删分支不问
- `trash` — 删时不走 trash

Doom 默认已经设了一些。你可以加更多。

### 2.9 Org-babel 不问执行

```elisp
(setq org-confirm-babel-evaluate nil)
```

每次在 org 文件里 `C-c C-c` 跑代码块,默认问 "Evaluate this code block? (y/n)"。这是**安全考虑**——别人 org 文件可能有 `(shell-command "rm -rf /")`。

如果你只跑自己的 org 文件,关掉省事。

### 2.10 recentf 自动清理

```elisp
(setq recentf-auto-cleanup 'never)
;; 或: (setq recentf-auto-cleanup '(mode . idle 600))  ; idle 10 分钟后清理
```

recentf 默认启动时清理过期文件 (删了的文件从列表移除),但每次清理会问 "Cleanup recentf list? (y/n)"。

`'never` 完全关。如果你想清理,手动 `M-x recentf-cleanup`。

---

## 3. 完整推荐配置

整合上面所有:

```elisp
;;; init-silencing-prompts.el -*- lexical-binding: t; -*-

;; yes/no → y/n
(setq use-short-answers t)

;; 退出不问
(setq confirm-kill-processes nil)
;; (setq confirm-kill-emacs nil)  ; 慎用

;; buffer/file 创建不问
(setq confirm-nonexistent-file-or-buffer nil)

;; 大文件警告阈值调高 (而不是关掉)
(setq large-file-warning-threshold 500000000)

;; file-local variables 信任 (如果项目都是自己的)
(setq enable-local-variables :all)

;; Projectile 缓存
(setq projectile-cache-delete-prompt-function #'ignore)

;; recentf 不启动时清理
(setq recentf-auto-cleanup 'never)

;; Dired 递归
(setq dired-deletion-confirms nil
      dired-recursive-deletes 'always
      dired-recursive-copies 'always
      dired-clean-confirm-killing-deleted-buffers nil)

;; Magit
(setq magit-no-confirm '(stage-all-changes
                          magit-push-current
                          delete-some-branch))

;; Org babel
(setq org-confirm-babel-evaluate nil)

(provide 'init-silencing-prompts)
```

加到 `init.el` 的 require 列表,或 Doom 的 `config.el`,重启。

---

## 4. 核弹选项 (危险)

如果你真的彻底受够,可以让**所有** `y-or-n-p` 和 `yes-or-no-p` 返回 `t`:

```elisp
(fset 'y-or-n-p      (lambda (&rest _) t))
(fset 'yes-or-no-p   (lambda (&rest _) t))
```

`fset` 直接改函数 cell,把这两个函数替换成永远返回 `t` 的 lambda。**任何**调用它们的代码都"通过确认"。

**后果**:
- ✅ 永远不被烦
- ❌ `C-x C-c` 直接退出 (即使有未保存 buffer)
- ❌ `delete-file` 直接删 (无回收站如果设了 trash nil)
- ❌ file-local variables 自动 eval (有 `eval:` 都执行)
- ❌ 恶意 elisp 包的破坏性操作畅通无阻

**只在两种情况用**:

1. **纯只读 / demo 环境**: 给学生 demo 时,不想被 prompt 打断
2. **Emacs daemon 的 batch 模式**: 自动化脚本里,不能交互

生产配置**强烈不建议**。

---

## 5. 排查: 哪个 prompt 烦你

如果还有特定 prompt,定位它的命令:

### 5.1 prompt 出现时按 C-h

prompt 弹出后,**不要**按 y/n,先按 `C-h k`,然后按 y (或你看到的那个键)。Emacs 会显示 "y runs the command ..."——告诉你 prompt 来自哪个函数。

### 5.2 查 minibuffer 历史

```elisp
;; 在 *scratch* 里 eval:
(view-echo-area-messages)
;; 或 C-h e
```

`*Messages*` buffer 显示所有最近的 minibuffer 输出,你能看到 prompt 的完整文本。

### 5.3 grep 源码

如果知道 prompt 文本 (例如 "really delete"),grep 所有装了的包:

```bash
grep -r "really delete" ~/.config/emacs/elpa/
```

找到调用 `(y-or-n-p "really delete ...")` 的包,然后:

- 改该包的变量 (大部分包提供 `*-no-confirm` 选项)
- 或 `advice-add` 该函数让它不问

### 5.4 advice 关 prompt (精准核弹)

如果某个 prompt 没有官方选项关,用 advice:

```elisp
(defun my-skip-confirm (&rest args)
  "Always return t for any yes-or-no-p call."
  t)

(advice-add 'magit-pull-and-fetch-popup :before
            (lambda (&rest _) (message "fetching...")))
```

更精准: 只对某个调用方关:

```elisp
(defun my-no-confirm-foo (orig-fun &rest args)
  "Skip confirmation if called from foo."
  (if (eq this-command 'foo)
      t
    (apply orig-fun args)))

(advice-add 'y-or-n-p :around #'my-no-confirm-foo)
```

这是"编程级"控制——不是"全关",是"看场景"。

---

## 6. 何时**应该**保留确认

不要无脑全关。这些 prompt 是**安全网**,关了可能出大事:

| Prompt | 建议 | 理由 |
|---|---|---|
| 退出 Emacs 时未保存 | 保留 | 丢失编辑 |
| 删除受版本控制外的文件 | 保留 | 不可恢复 |
| file-local variables 的 `eval:` | 保留 | 恶意代码 |
| 提交 Git 时不带 message | 保留 | 错误的 commit message 难改 |
| 删除远端 (TRAMP) 文件 | 保留 | 远端没 trash |
| 执行 `(shell-command ...)` 的危险命令 | 保留 | 系统级影响 |

经验: **如果 prompt 是"撤销一次操作能恢复"的,关掉;如果是"不可逆"的,保留**。

---

## 7. 总结

Emacs 爱问 = 1985 设计 + 向后兼容。

**推荐的层次** (从安全到激进):

1. **Level 1 (最安全)**: 只加 `(setq use-short-answers t)`——`yes/no` 变 `y/n`,快一倍
2. **Level 2 (推荐)**: 加 §3 的完整配置——关掉烦人的,保留保护性的
3. **Level 3 (老手)**: Level 2 + 信任 file-local variables + Magit 全关
4. **Level 4 (核弹)**: `(fset 'yes-or-no-p #'ignore)`——所有 prompt 自动通过

**从 Level 2 开始**。用一段时间,如果某个 prompt 还烦,精准关掉它 (找变量或 advice)。最终你会到达 Level 3,既流畅又安全。

Level 4 **永远不要**——除非你清楚后果。

---

## 8. 实战练习

1. 加 `(setq use-short-answers t)`,重启 Emacs,记录 prompt 减少
2. 加 `confirm-kill-processes nil`,看退出是否顺畅
3. 加 `dired-recursive-deletes 'always`,试删一个有子目录的 dir
4. 写一个 advice,让 `magit-push-current` 不问 (`magit-no-confirm` 之外的方式)
5. 找一个你常用包的 prompt,grep 它的源码,看是哪个函数调的 `(y-or-n-p ...)`

---

## 9. 自测

1. `y-or-n-p` 和 `yes-or-no-p` 区别?
2. `use-short-answers t` 干啥?
3. `confirm-kill-processes` 和 `confirm-kill-emacs` 区别?
4. `enable-local-variables :all` 危险在哪?
5. `advice-add` 关 prompt 怎么写?

**答案**:

> 1. y-or-n-p 按单字符;yes-or-no-p 打字,慢但强制思考
> 2. 让 yes-or-no-p 内部用 y-or-n-p 的 UI,只输 y/n 不输 yes/no
> 3. 前者问"杀进程";后者问"退出 Emacs"。daemon 模式下后者无意义
> 4. 文件里的 `eval:` local variable 自动执行,可能恶意代码
> 5. `(advice-add 'FN :around (lambda (orig &rest _) t))`

---

完成后,你的 Emacs 应该比之前少 80% 的 prompt 烦扰。
