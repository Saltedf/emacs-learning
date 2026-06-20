# Capstone: Module 3 毕业项目 (TODO Manager)

> **目标**: 写一个完整的 my-todo 包
> **时长**: 1 周 (~25 小时)
> **难度**: ★★★★☆

---

## 项目概述

完整开发一个 todo 管理包,要求:

1. **数据持久化**: 保存到文件,加载
2. **Major mode**: 自定义模式 + keymap
3. **CRUD 操作**: 添加/查看/编辑/删除
4. **过滤**: 按标签、优先级
5. **测试**: ERT 至少 5 个测试
6. **文档**: README + docstring 完整
7. **集成**: 在 init.el 加载,绑快捷键

完整代码见 `chassell-reading.md`。
这个文件是**实施指南**。

为什么这个项目是 Module 3 的"毕业项目"?因为它把 W1-W4 学的所有概念用一遍:

- W1 list: todo 数据是 list of items,filter 用 list 操作
- W2 defun/let/if: 每个命令都是 defun,内部用 let/if
- W3 递归/闭包: 排序用递归思路,filter 状态可能用闭包
- W4 lambda/mapcar: filter 内部用 cl-remove-if-not,显示用 dolist

一个项目串起所有——这就是"综合训练"。

完成这个项目后,你不再是"学过 Lisp 的人",而是"写过 Lisp 包的人"。这两者在简历上、在贡献开源时,意义完全不同。

---

## 实施步骤 (5 天计划)

5 天拆解有讲究——每天一个"层次":

- Day 1: 骨架 (文件 + 框架)
- Day 2: 数据结构 (Core API)
- Day 3: 交互 (Interactive commands)
- Day 4: 持久化 (Save/Load/Filter)
- Day 5: 测试 + 整合 (ERT + init.el)

每天独立完成,不要跨天。这样如果某天卡住,你知道是哪一层的 bug。

### Day 1: 骨架 (3-4 小时)

**目标**: 创建文件,基础结构能跑。

骨架是"先有架子,再填肉"。先确保文件能 load、mode 能切换、buffer 能显示——即使内容是空的。这给你"工作基础"——后续每天在这个骨架上加功能。

#### Task 1.1: 创建文件

```bash
mkdir -p ~/.config/emacs/lisp
touch ~/.config/emacs/lisp/my-todo.el
```

#### Task 1.2: 写骨架

参考 `chassell-reading.md` 第 3.1 节,把骨架代码贴入。

包括:
- `;;; Commentary`
- `;;; Code`
- `(require 'cl-lib)`
- `;;; Custom variables`
- `;;; Internal variables`
- keymap
- `define-derived-mode`
- `(provide 'my-todo)`
- 结尾注释

#### Task 1.3: eval 测试

```
M-x load-file RET ~/.config/emacs/lisp/my-todo.el RET
```

确认无 error。

#### Task 1.4: 加 `my-todo-refresh` 空版

```elisp
(defun my-todo-refresh ()
  (interactive)
  (let ((buf (get-buffer-create my-todo-buffer-name)))
    (with-current-buffer buf
      (erase-buffer)
      (insert "TODO\n"))
    (switch-to-buffer buf)))
```

`M-x my-todo-refresh` 应该打开 buffer 显示 "TODO"。

#### Day 1 检查

- [ ] my-todo.el 创建
- [ ] load-file 不报错
- [ ] `M-x my-todo-mode` 能切到该 mode
- [ ] keymap 绑定工作 (`C-h m` 在 my-todo buffer 看到)

---

### Day 2: 数据结构 + Core API (4-5 小时)

Day 2 是项目的"心脏"——数据结构和核心 API。这些是其他命令的基础,做对了后面顺;做错了,后面全是补丁。

#### Task 2.1: 实现 accessors

```elisp
(defun my-todo--item-text (item) (nth 0 item))
(defun my-todo--item-priority (item) (nth 1 item))
(defun my-todo--item-tags (item) (nth 2 item))
(defun my-todo--item-done (item) (nth 3 item))

(defun my-todo--item-set-done (item value)
  (setf (nth 3 item) value))
```

`setf` 是 cl-lib 的"通用 setq",可以 set 各种位置。

为什么用 accessor 而不直接 `(nth 0 item)`?因为 accessor 提供"间接层"——以后改数据结构 (加 deadline,改字段顺序),只改 accessor,不改调用点。这是"封装"的基本功。

`setf` 是"广义赋值"——`(setf (nth 3 item) value)` 设置 item 的第 3 个元素为 value。比 `(setcar (nthcdr 3 item) value)` 这种"原始 cons cell 操作"清晰得多。

#### Task 2.2: 实现 add-internal

```elisp
(defun my-todo-add-internal (text priority tags)
  (let ((item (list text priority tags nil)))
    (setq my-todo-list
          (append my-todo-list (list item)))))
```

#### Task 2.3: 完整 refresh

参考 `chassell-reading.md` 第 3.1 节,实现完整 `my-todo-refresh`:
- 清空 buffer
- 启用 mode
- 循环 my-todo-list,插入格式化文本
- 切到 buffer

#### Task 2.4: 测试

```
M-x my-todo-add-internal RET "test" 1 nil RET
M-x my-todo-refresh
```

应该看到一行 todo。

#### Day 2 检查

- [ ] accessors 工作
- [ ] add-internal 工作
- [ ] refresh 显示列表

---

### Day 3: Interactive Commands (5-6 小时)

#### Task 3.1: my-todo-add

```elisp
(defun my-todo-add (text priority tags)
  (interactive
   (list (read-from-minibuffer "Todo: ")
         (read-number "Priority (1-5): " 3)
         (split-string
          (read-from-minibuffer "Tags (comma-sep): ")
          "," t " ")))
  (my-todo-add-internal text priority tags)
  (my-todo-refresh))
```

#### Task 3.2: my-todo-done / undone

```elisp
(defun my-todo--find-item-at-line ()
  (let ((line (line-number-at-pos)))
    (nth (- line 1) my-todo-list)))

(defun my-todo-done ()
  (interactive)
  (let ((item (my-todo--find-item-at-line)))
    (when item
      (my-todo--item-set-done item t)
      (my-todo-refresh))))
```

类似 undone。

#### Task 3.3: my-todo-delete

```elisp
(defun my-todo-delete ()
  (interactive)
  (let ((line (line-number-at-pos)))
    (setq my-todo-list
          (append (cl-subseq my-todo-list 0 (1- line))
                  (cl-subseq my-todo-list line)))
    (my-todo-refresh)))
```

#### Task 3.4: 在 my-todo buffer 测试

```
M-x my-todo RET
;; buffer 打开,显示 "No todos" 或列表
;; 按 a 添加
;; 按 d 标记完成
;; 按 x 删除
```

#### Day 3 检查

- [ ] `a` 添加 (3 个 prompt)
- [ ] `d` 标记完成
- [ ] `u` 取消
- [ ] `x` 删除
- [ ] `q` 退出

---

### Day 4: Save/Load + Filter (4-5 小时)

Day 4 加"持久化"——保存到文件,加载回。这是 todo 应用"跨 session"的关键。

#### Task 4.1: my-todo-save

```elisp
(defun my-todo-save ()
  (interactive)
  (with-temp-file my-todo-file
    (pp my-todo-list (current-buffer)))
  (message "Saved %d todos" (length my-todo-list)))
```

为什么用 `with-temp-file` 而不是 `(find-file ...)`?因为 find-file 会创建一个 buffer,污染用户的 buffer list。with-temp-file 用临时 buffer 写文件,完成后丢弃——干净。

#### Task 4.2: my-todo-load

```elisp
(defun my-todo-load ()
  (interactive)
  (when (file-exists-p my-todo-file)
    (with-temp-buffer
      (insert-file-contents my-todo-file)
      (setq my-todo-list (read (current-buffer)))))
  (my-todo-refresh))
```

#### Task 4.3: filter

```elisp
(defvar my-todo--original-list nil
  "Original list before filtering.")

(defun my-todo-filter (tag)
  (interactive "sTag (empty for all): ")
  (if (string-empty-p tag)
      (setq my-todo-list my-todo--original-list)
    (setq my-todo--original-list my-todo-list)
    (setq my-todo-list
          (cl-remove-if-not
           (lambda (item)
             (member tag (my-todo--item-tags item)))
           my-todo-list)))
  (my-todo-refresh))
```

#### Task 4.4: 测试 save/load

```
M-x my-todo RET
a → 添加几个 todo
s RET  (保存)
q     (退出)
重启 Emacs
M-x my-todo RET
l     (load,应该恢复)
```

#### Day 4 检查

- [ ] save 写文件
- [ ] load 读文件
- [ ] filter 按 tag
- [ ] 跨 session 数据保留

---

### Day 5: 测试 + 文档 + 整合 (3-4 小时)

#### Task 5.1: ERT 测试

创建 `my-todo-test.el`:

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

(ert-deftest my-todo-delete-test ()
  (setq my-todo-list nil)
  (my-todo-add-internal "A" 1 nil)
  (my-todo-add-internal "B" 1 nil)
  (setq my-todo-list (append (cl-subseq my-todo-list 0 0)
                             (cl-subseq my-todo-list 1)))
  (should (= 1 (length my-todo-list))))

(ert-deftest my-todo-filter-test ()
  (setq my-todo-list nil)
  (my-todo-add-internal "A" 1 '("work"))
  (my-todo-add-internal "B" 1 '("home"))
  (let ((filtered (cl-remove-if-not
                   (lambda (item)
                     (member "work" (my-todo--item-tags item)))
                   my-todo-list)))
    (should (= 1 (length filtered))
            (should (equal "A" (my-todo--item-text (car filtered)))))))

(provide 'my-todo-test)
```

跑:
```
M-x load-file RET my-todo-test.el RET
M-x ert-run-tests-interactively RET my-todo RET
```

#### Task 5.2: README

创建 `my-todo.md`:

```markdown
# my-todo

简单的 Emacs todo 管理。

## 安装

把 `my-todo.el` 加到 `load-path`,然后:

```elisp
(require 'my-todo)
(global-set-key (kbd "C-c t") 'my-todo)
```

## 使用

`M-x my-todo` 打开。

| 键 | 命令 |
|---|---|
| `a` | 添加 |
| `d` | 标记完成 |
| `u` | 取消完成 |
| `x` | 删除 |
| `s` | 保存 |
| `l` | 加载 |
| `f` | 过滤 |
| `g` | 刷新 |
| `q` | 退出 |

## 配置

```elisp
(setq my-todo-file "~/.config/emacs/todos.el")
```

## 测试

```
M-x ert-run-tests-interactively RET my-todo
```
```

#### Task 5.3: init.el 加载

```elisp
(add-to-list 'load-path "~/.config/emacs/lisp/")
(require 'my-todo)
(global-set-key (kbd "C-c t") 'my-todo)
```

eval 后 `C-c t` 应该打开 my-todo。

#### Task 5.4: 重启测试

完全重启 Emacs,`C-c t` 应该工作,数据应该加载。

#### Day 5 检查

- [ ] ERT 测试通过 (至少 4 个)
- [ ] README 写好
- [ ] init.el 集成
- [ ] 重启后工作

---

## 高级挑战 (可选)

### 挑战 1: 排序

按 priority 降序,done 排后:

```elisp
(defun my-todo--sort-list ()
  (setq my-todo-list
        (sort my-todo-list
              (lambda (a b)
                (cond
                 ((and (my-todo--item-done a)
                       (not (my-todo--item-done b))) nil)
                 ((and (not (my-todo--item-done a))
                       (my-todo--item-done b)) t)
                 (t (> (my-todo--item-priority a)
                       (my-todo--item-priority b))))))))
```

在 refresh 前调用。

### 挑战 2: Deadline

数据结构加 deadline:
```elisp
(text priority tags done deadline)
```

add 时可选输入 deadline。
显示时显示倒计时。

### 挑战 3: 重复任务

`done` 时,如果 recurring,生成下一个:
```elisp
(defun my-todo--maybe-recur (item)
  (let ((recurring (my-todo--item-recurring item)))
    (when recurring
      (let ((new (copy-tree item)))
        (my-todo--item-set-done new nil)
        ;; 更新 deadline
        (setq my-todo-list
              (append my-todo-list (list new)))))))
```

### 挑战 4: org-mode 集成

把 todo 导出为 org 文件:
```elisp
(defun my-todo-export-org (path)
  (interactive "FExport to: ")
  (with-temp-file path
    (insert "* Todos\n\n")
    (dolist (item my-todo-list)
      (insert (format "** %s %s\n"
                      (if (my-todo--item-done item) "DONE" "TODO")
                      (my-todo--item-text item))))))
```

### 挑战 5: multi-todo-lists

支持多个 todo list (work / home / shopping):

```elisp
(defvar my-todo-lists (make-hash-table :test 'equal))

(defun my-todo-switch (name)
  (interactive "sList name: ")
  (setq my-todo-current-list name)
  (setq my-todo-list (gethash name my-todo-lists))
  (my-todo-refresh))
```

---

## 反思

完成这个项目,你应该能:

- 设计一个 Lisp 数据结构 (list of items)
- 用 Lisp 文件 I/O 持久化
- 定义自己的 major mode
- 写 interactive 命令
- 用 ERT 测试
- 整合到 init.el
- 写文档

这是 Module 3 的总结。
也是从"学 Lisp"到"用 Lisp 创造"的飞跃。

---

## 日志

写 `logs/module-03.md`:

```markdown
# Module 3 学习日志

**日期**: 2026-XX-XX
**总用时**: ___ 小时
**Capstone 用时**: ___ 小时

## 5 周心得

### Week 1 (Lists)
- 心得:

### Week 2 (defun/let/if)
- 心得:

### Week 3 (Recursion)
- 心得:

### Week 4 (Lambda/Mapcar)
- 心得:

### Week 5 (Capstone)
- 心得:

## Capstone 项目

- 行数:
- 测试数:
- 用了哪些高级特性:

## 我学到的最重要 5 件事

1.
2.
3.
4.
5.

## 我还要加强的

- [ ]
- [ ]

## 下一步

进入 Module 4: 配置体系 + Minor Mode
```

---

## 毕业检查

- [ ] my-todo.el 完整 (≥300 行)
- [ ] my-todo-test.el 至少 4 个测试通过
- [ ] README 写好
- [ ] init.el 集成,`C-c t` 工作
- [ ] 跨 session 数据保留
- [ ] 至少完成 1 个高级挑战

更新 `PROGRESS.md`: Module 3 ✅。

进入 Module 4: `04-config-system/README.md`。

**恭喜**。

你已经从"用户"到"开发者"。
你写的 my-todo 是真实的、可用的包。
你已经走过 Module 3 (核心)。

继续 Module 4-8,你将:
- 把 init.el 变成艺术品 (Module 4)
- 用 Org/Magit 工作 (Module 5)
- 攻克 Elisp Reference (Module 6)
- 发布到 MELPA (Module 7)
- 贡献开源 (Module 8)
