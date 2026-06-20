# 学习进度跟踪表

> 每完成一个模块,更新这个表格,并在 `logs/` 下写一篇日志。
> 不要跳模块,后面的依赖前面的。

---

## 总览表

| # | 模块 | 状态 | 开始日期 | 完成日期 | 实际用时 | 毕业项目 |
|---|---|---|---|---|---|---|
| 0 | 心法与装备 | ⬜ 未开始 | | | | |
| 1 | 编辑生存 | ⬜ | | | | |
| 2 | Dired + Buffer/Window | ⬜ | | | | |
| 3 | Elisp 入门 (Chassell) ★ | ⬜ | | | | |
| 4 | 配置体系 + Minor Mode | ⬜ | | | | |
| 5 | Power User 工作流 | ⬜ | | | | |
| 6 | Elisp 深度 (Reference) ★ | ⬜ | | | | |
| 7 | 写一个真实的包 | ⬜ | | | | |
| 8 | 开源贡献 | ⬜ | | | | |

**状态图例**: ⬜ 未开始 / 🟡 进行中 / ✅ 完成 / ⏸️ 暂停

---

## 模块进度细分

### Module 0: 心法与装备 (预计 2 天, 6 小时)

| 任务 | 完成 | 用时 | 备注 |
|---|---|---|---|
| 读 `00-mindset/README.md` | ⬜ | | |
| 读 `00-mindset/concept-anchor.md` | ⬜ | | |
| 读 `00-mindset/intro-reading.md` | ⬜ | | |
| 完成 `00-mindset/exercises.md` | ⬜ | | |
| 从源码编译 Emacs | ⬜ | | |
| 完成 `M-x help-with-tutorial` | ⬜ | | |
| 写出 10 行 init.el | ⬜ | | |
| 完成 `00-mindset/capstone.md` | ⬜ | | |

---

### Module 1: 编辑生存 (预计 1 周, 22.5 小时)

| 任务 | 完成 | 用时 | 备注 |
|---|---|---|---|
| 读 `01-survival/README.md` | ⬜ | | |
| 读 `01-survival/concept-anchor.md` | ⬜ | | |
| Drill 01: Movement (30 题) | ⬜ | | |
| Drill 02: Kill Ring (20 题) | ⬜ | | |
| Drill 03: Search (25 题) | ⬜ | | |
| Drill 04: Mark & Rectangle (20 题) | ⬜ | | |
| Drill 05: Kmacro (15 题) | ⬜ | | |
| Drill 06: Windows & Frames (20 题) | ⬜ | | |
| Drill 07: Help System (25 题) ★ | ⬜ | | |
| Drill 08: Code Navigation (30 题) ★ | ⬜ | | sexp 移动/操作、imenu、which-function、xref、hideshow |
| 完成 capstone (重构 2000 行) | ⬜ | | |

---

### Module 2: Dired + Buffer/Window (预计 4 天, 12 小时)

| 任务 | 完成 | 用时 | 备注 |
|---|---|---|---|
| 读 `02-dired/README.md` | ⬜ | | |
| 读 `02-dired/concept-anchor.md` | ⬜ | | |
| 完成 `02-dired/drills.md` (40 题) | ⬜ | | |
| 完成 capstone (整理混乱目录) | ⬜ | | |

---

### Module 3: Elisp 入门 (Chassell, 5 周, 100 小时) ★核心★

| 周 | 任务 | 完成 | 用时 | 备注 |
|---|---|---|---|---|
| W1 | Lists + Evaluation (10 个 list 函数) | ⬜ | | |
| W2 | defun + let + if (5 个 buffer 函数) | ⬜ | | |
| W3 | Conditionals + Recursion | ⬜ | | |
| W4 | Lambda + mapcar + sequences | ⬜ | | |
| W5 | Capstone: TODO 管理器 (200+ 行) | ⬜ | | |

---

### Module 4: 配置体系 + Minor Mode (2 周, 40 小时)

| 任务 | 完成 | 用时 | 备注 |
|---|---|---|---|
| 读 `04-config-system/README.md` | ⬜ | | |
| 读 `concept-anchor.md` | ⬜ | | |
| 读 `ref-deep-dive.md` | ⬜ | | |
| 学 use-package | ⬜ | | |
| 学 keymaps | ⬜ | | |
| 学 hooks | ⬜ | | |
| 学 define-minor-mode | ⬜ | | |
| 完成 500 行 init.el | ⬜ | | |
| 完成自定义 minor mode | ⬜ | | |

---

### Module 5: Power User (3 周, 60 小时)

| 任务 | 完成 | 用时 | 备注 |
|---|---|---|---|
| Org-mode (capture/agenda/babel) | ⬜ | | |
| Magit (完整 Git 流程) | ⬜ | | |
| project.el + 补全栈 (vertico/corfu) | ⬜ | | |
| eglot (LSP) | ⬜ | | |
| Org literate 配置 | ⬜ | | |
| Calendar + Diary (`calendar-diary.md`) | ⬜ | | Emacs 内置时间系统 |
| GUD 调试器 (`gud-debugging.md`) | ⬜ | | gdb/pdb 集成 |
| Mule 国际化 (`mule-international.md`) | ⬜ | | 编码/输入法/字体 |
| Misc + Amusements (`misc-amusements.md`) | ⬜ | | Emacs 游戏/batch/emacsclient |
| Magit PR 毕业项目 | ⬜ | | |

---

### Module 6: Elisp 深度 (Reference, 8 周, 160 小时) ★核心★

| 周 | 主题 | 完成 | 用时 | 备注 |
|---|---|---|---|---|
| W1 | 数据类型 (objects/numbers/strings/lists/seq/hash) | ⬜ | | |
| W2 | 求值/变量/函数 | ⬜ | | |
| W3 | 宏/加载/编译 | ⬜ | | |
| W4 | 调试/Edebug | ⬜ | | |
| W5 | Minibuffer/Commands/Keymaps | ⬜ | | |
| W6 | Modes/Files/Buffers/Windows | ⬜ | | |
| W7 | Text/Display/Search/Syntax | ⬜ | | |
| W8 | Processes/OS/Package | ⬜ | | |
| Capstone | 写一个完整 major mode (500+ 行 + ERT) | ⬜ | | |

---

### Module 7: 写一个真实的包 (3 周, 60 小时)

| 任务 | 完成 | 用时 | 备注 |
|---|---|---|---|
| 选题 + 设计 | ⬜ | | |
| 实现 (TDD) | ⬜ | | |
| ERT 测试 | ⬜ | | |
| 文档 (README + Texinfo) | ⬜ | | |
| 提交 MELPA | ⬜ | | |
| **包被 MELPA 合并** | ⬜ | | |

---

### Module 8: 开源贡献 (持续)

| 任务 | 完成 | 用时 | 备注 |
|---|---|---|---|
| 订阅 emacs-devel | ⬜ | | |
| Star 10 个常用包 | ⬜ | | |
| 读 1 个包的源码 | ⬜ | | |
| 找 good first issue | ⬜ | | |
| 提第一个 PR | ⬜ | | |
| **PR 被合并** | ⬜ | | |

---

## 里程碑检查

- [ ] **3 个月**: 脱离鼠标 + 500 行 init.el + 能读 elisp + 写过 minor mode
- [ ] **6 个月**: 写过 major mode + 发布过 MELPA 包
- [ ] **9 个月+**: 给他人包合并过 PR + 读 emacs-devel

---

## 真实能力自测 (每完成一个模块做一遍)

完成 Module 6 后,以下 8 题应该全部能答出:

- [ ] 1. 解释 `lexical-binding: t` 的作用
- [ ] 2. 描述闭包 `(let ((x 1)) (lambda () x))` 的行为
- [ ] 3. 区分 `eq`/`equal`/`eql` 对 `'a` `"a"` `1` `1.0` 的行为
- [ ] 4. 用 `make-process` 跑 `git status` 并把输出插入 buffer
- [ ] 5. 解释 `font-lock-defaults` 的结构
- [ ] 6. 写一个继承 `prog-mode` 的 `define-derived-mode`
- [ ] 7. 解释 `with-current-buffer` 与 `save-excursion` 的区别
- [ ] 8. 用 Edebug 单步执行函数
- [ ] 9. 用 `narrow-to-defun` 限制 buffer 操作范围
- [ ] 10. 用 `peg` 解析一段简单 DSL (URL / JSON 子集)
- [ ] 11. 定义一个 mode-specific abbrev table,带 hook 自动加 timestamp

---

## 用法说明

1. **每开始一个模块**: 把状态改为 🟡,填开始日期
2. **每完成一个子任务**: 打 ✅,记实际用时
3. **完成模块**: 写 `logs/module-XX.md` 日志 (心得、踩坑、收获)
4. **每 3 个月**: 复盘,调整后面的节奏
5. **诚实**: 不要打 ✅ 除非真的完成且理解

---

## 当前状态

**正在学习**: (空)
**下一步**: 打开 `00-mindset/README.md` 开始 Module 0
**最近更新**: 补充 Narrowing/PEG/Abbrevs/Misc 四处缺口
