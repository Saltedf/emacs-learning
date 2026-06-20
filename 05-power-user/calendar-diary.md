# Calendar + Diary 详解

> 学完这个文件,你能用 Emacs 内置的 Calendar 和 Diary 做日期计算、跟踪事件、配合 Org 工作

很多人以为 Org-agenda 是 Emacs 唯一的"时间管理"工具——但 Emacs 在 Org 出现之前 20 年就有自己的日历和日记系统。这个系统今天仍然内置,仍然有用,而且它的**底层概念** (日期算术、跨日历转换、Diary 文本格式) 是 Org-agenda 的基石。

学这一节的价值有两个: 第一,你在不想要 Org 的"重"的场景下 (一两个固定事件、只想查日历),Calendar/Diary 比 Org 更轻; 第二,理解了 Calendar/Diary,你会更深刻理解 Org-agenda 的设计——它不是凭空发明的,是在这个基础上演化的。

---

## 1. 为什么编辑器要内置日历

### 1.1 第一性原理推导

这个问题的答案需要从"文本编辑"出发:

- **第一性原理**: 编辑器处理**文本**。文本中常常包含**日期**——日志、日记、commit message、邮件、TODO。如果编辑器"懂"日期,它就能帮你在文本里做日期相关的事。
- **推论 1**: 日期是一个**结构化数据** (年月日),但日常我们写文本时是"字符串" (`2026-06-20`)。编辑器应该能**两者互通**——选中字符串,识别为日期,跳转到 calendar 里的对应位置。
- **推论 2**: 日期计算 (加减、间隔、星期) 是文本相关的常见操作——"100 天后是哪天""两个日期间隔多少天""某天是周几"。这些不应该让用户离开 Emacs 算。
- **推论 3**: 日期相关的**事件** (生日、纪念日、定期会议) 应该可以记录,并且能**自动提醒**。这些事件应该和编辑器"耦合"——它们和文档、TODO 同源。
- **推论 4**: 不同的文化和历法有不同的日期系统——Gregorian (公历)、Julian (儒略历)、Islamic (回历)、Hebrew (希伯来历)、Mayan (玛雅历)。一个全球编辑器应该理解所有这些。

这四条推论加起来,得到 Emacs Calendar 的设计: **一个集成在编辑器里的日历,可以做日期算术、记录事件、跨文化转换**。

### 1.2 文本即数据 (Diary 的设计哲学)

Diary 是 Emacs 的"日记"功能——它把所有事件存在一个**纯文本文件** (`~/diary`) 里。这听起来朴素,但它是 Emacs 的"反产品"哲学的体现:

- **商业日历** (Google Calendar、Outlook) 把数据存在**云端数据库**——你看不到原始格式,数据被锁定。
- **Emacs Diary** 把数据存在**纯文本文件**——你可以 `cat ~/diary`,可以 grep 它,可以版本控制它,可以写脚本批量修改。

这种"文本即数据"哲学在 Emacs 里到处都是——Magit 把 Git 状态放进 buffer,Org 把 TODO 放进文件,Dired 把目录放进 buffer。Calendar/Diary 是这个哲学的最早实践之一 (1990 年代就有了)。

Diary 的"文本"是什么样子? 大致这样:

```
June 20, 2026  birthdays
&%%(diary-anniversary 6 20 1990) Alice's birthday
&%%(diary-cyclic 7 6 1 2026) Weekly team meeting
&2026-06-25 Pay rent
```

每一行是一个 entry,左边是日期规范 (可以是具体日期、可以是 sexp 表达式),右边是事件描述。Emacs 的 calendar 读这个文件,在日历上**高亮有事件的日期**。这就是 Org-agenda 的原型——Org 里的 `SCHEDULED`、`DEADLINE`、`<2026-06-20>` 时间戳本质上是 Diary 语法的演化。

---

## 2. 打开 Calendar

### 2.1 基本入口

打开 calendar 用 `M-x calendar`:

```
M-x calendar RET
```

这会打开一个 3 个月的视图 (上个月、本月、下月),光标定位在**今天**。例如今天是 2026-06-20,你看到的就是 May/June/July 2026 三个月并排显示,6 月 20 那一格被高亮。

这个 buffer 是 Emacs 的"日历模式"——它有自己的一套键位,和普通 buffer 不同。看 mode line 会发现显示 "Calendar",这是它的身份标识。

为什么不直接在普通 buffer 里写日期?因为日历是一个**结构化视图**——你需要"看到月份的形状" (这周几、月头月尾、跨周)。表格形式的视图比纯文本直觉得多,这种"用 buffer 做结构化 UI"是 Emacs 的拿手好戏。

### 2.2 关闭

```
q    退出 (quit-window)
```

`q` 关闭 calendar buffer——这是 Emacs 里几乎所有 special buffer 的通用键 (Dired、Help、Magit 都用 `q`)。

---

## 3. Calendar 导航

Calendar 的导航键位很多,但都遵循"前缀 + 字母"的语义模式,记起来不难。

### 3.1 单日移动

最基础的——一天一天地挪:

```
C-f / 右箭头    前一天 (forward-day)
C-b / 左箭头    后一天 (backward-day)
C-n / 下箭头    下一日 (forward-week,通常是同列下移)
C-p / 上箭头    上一日 (backward-week)
```

注意是 `C-` 前缀,因为单字母在 calendar 里被分配给更常用的命令了。这看起来不直觉,但记住一个原则:**calendar 里 `C-` 前缀是"小步",无前缀是"大步"**。

### 3.2 大步移动

大步移动用 `M-` 前缀:

```
M-n    下个月 (calendar-scroll-month-forward)
M-p    上个月 (calendar-scroll-month-backward)
M-}    下一年 (calendar-forward-year)
M-{    上一年 (calendar-backward-year)
```

`M-n` / `M-p` 是 Emacs 全局约定 (next/previous 历史或元素),在 calendar 里也是这个意思——但是"月份级别"的 next/previous。`M-}` / `M-{` 是年份级别的——这是社区约定。

### 3.3 跳转

跳转到具体日期用 `g` (goto) 前缀:

```
.       今天 (calendar-goto-today)
g d     跳到指定日期 (calendar-goto-date)
g w     跳到某周开头 (calendar-goto-week)
g m     跳到某月开头 (calendar-goto-month)
g y     跳到某年开头 (calendar-goto-year)
g j     跳到 Julian 日期
g C-j   跳到 ISO 日期
```

`g d` 会问你 "Year (default 2026):" → "Month (default 6):" → "Day (default 20):",你输完就跳过去。这种"前缀 + 字母"的菜单是 Emacs 的老式 UI——后来 transient (Magit 用的) 把它做得更漂亮,但本质是同一个模式。

`.` (dot) 跳回今天是最高频的——任何时候迷路了,按 `.` 就回来。

### 3.4 滚动

如果要快速翻整个 3 月视图:

```
C-v     下一个 3 月 (scroll-calendar-left)
M-v     上一个 3 月 (scroll-calendar-right)
```

这是 Emacs 通用键——`C-v` / `M-v` 在所有 buffer 里都是"下/上翻页"。calendar 里"页"就是 3 个月。

---

## 4. 日期算术

Calendar 不只是"显示"——它还能**算**。这是它和纸质日历的最大区别。

### 4.1 计算间隔

```
M-=    计算两个日期间隔天数 (calendar-count-days-region)
```

操作: 先 mark 一个日期 (`C-SPC`),移动到另一个日期,按 `M-=`——echo area 显示"X days"。这和 region 概念一样——Emacs 的"mark + point"机制在 calendar 里也工作。

### 4.2 计算加减

"100 天后是哪天" 是极常见的需求——比如算 "项目启动 90 天后"、"信用卡到期日"。Emacs 提供:

```
+       今天加 N 天
-       今天减 N 天
```

按 `+` 会问"Number of days to add:",你输 100 回车,光标就跳到 100 天后的日期。`-` 同理。这个操作以**当前光标日期**为基准,不是"今天"——所以你先 `g d` 跳到任意日期,再 `+ 100`,得到那个日期 + 100 天的结果。

### 4.3 其他日期信息

```
p C    查看光标日期的 ISO 8601 格式 (calendar-iso-print-date)
p w    查看光标日期的所在周
p d    查看光标日期是一年中第几天 (day of year)
p j    Julian 日期
p m    Mayan 长历日期
```

`p` 是 "print" (echo area 显示) 的前缀——所有"打印日期信息"的命令都在 `p` 下。

---

## 5. 跨日历转换

这是 Emacs Calendar 最独特的能力之一——**同时显示多种历法**。

### 5.1 为什么需要

- **文化背景**: 伊斯兰用户需要知道公历日期对应的回历日期;犹太用户需要希伯来历;中国用户需要农历 (虽然 Emacs 没内置完整农历,但 Chinese Calendar 模式是社区维护的)。
- **历史研究**: 古代文献用 Julian 历或 Mayan 历——研究者需要把日期转成现代历法。
- **跨地区协调**: 全球团队常常需要"这次会议在所有相关历法中是哪天"。

### 5.2 支持的历法

Emacs 内置支持:

```
Gregorian (公历) — 默认
Julian (儒略历,1582 年前的欧洲标准)
Islamic (回历)
Hebrew (希伯来历)
Mayan (玛雅历)
French (法国大革命历)
Persian (波斯历)
Coptic (科普特历)
Ethiopic (埃塞俄比亚历)
ISO (周历)
```

转换命令是 `p X` 系列——`p` 打印,`X` 是历法首字母:

```
p j    Julian
p i    Islamic
p h    Hebrew
p m    Mayan
p f    French
p C    ISO 8601
```

按 `p h` 在 echo area 显示当前光标日期的希伯来历日期。这是一个"窗口"——可以窥见同一时刻在不同文化中的表述。

### 5.3 切换显示历法

```
M-x calendar-goto-julian-date
M-x calendar-goto-islamic-date
M-x calendar-goto-hebrew-date
```

这些 `goto-*` 命令接受**目标历法的输入** (例如回历的年月日),然后跳到那个日期在公历 calendar 上的位置。这是反向——从其他历法查公历。

---

## 6. 实用命令

### 6.1 节假日

```
M-x holidays         显示所有近期节假日
```

显示一个 buffer,列出"未来几个月的节假日"——这是 Emacs 内置的节假日数据 (包含各国主要节日)。在 calendar buffer 里也可以用 `a` (calendar-list-holidays)。

如果你需要配置某国的节假日:

```elisp
(setq calendar-holidays
      '((holiday-fixed 1 1 "New Year's Day")
        (holiday-fixed 7 4 "Independence Day (US)")
        (holiday-easter-etc 0 "Easter")
        (holiday-float 11 4 4 "Thanksgiving (US)")))
```

`holiday-fixed` 是固定日期,`holiday-float` 是"某月的第几个星期几" (11 月第 4 个周四),`holiday-easter-etc` 是相对 Easter 的偏移。这些函数是 Lisp——你可以写任意复杂的节假日规则。

### 6.2 月相

```
M-x lunar-phases     显示近期月相
```

显示一个 buffer,列出每天的月相 (New Moon、First Quarter、Full Moon、Last Quarter)。这听起来"小众",但对潮汐、夜钓、观星、宗教 (伊斯兰和犹太历法依赖月相) 的工作流有用。在 calendar buffer 里按 `M` (大写) 显示月相。

### 6.3 日出日落

```
M-x sunrise-sunset       显示今天的日出日落
S                        在 calendar buffer 里看光标日期的日出日落
```

这需要你的**经纬度**——配置:

```elisp
(setq calendar-latitude 39.9)
(setq calendar-longitude 116.4)
(setq calendar-location-name "Beijing")
(setq calendar-time-zone +480)
```

`+480` 是 UTC+8 (北京时间,以分钟计)。配好之后,sunrise-sunset 显示精确到分钟的日出日落时间。这在规划户外活动、计算白天长度时有用。

### 6.4 嵌入日历到文档

```
M-x insert-calendar          在当前 buffer 插入 1 个月日历
M-x insert-monthly-calendar  插入 1 个月
M-x insert-yearly-calendar   插入整年
```

例如你在写一个 markdown 文档,需要插入"2026 年 6 月日历表"——`insert-calendar` 直接生成。这是一种"用 Emacs 生成结构化文本"的小技能——其他编辑器做不到。

---

## 7. Diary 文件

### 7.1 基本格式

Diary 文件默认在 `~/diary` (或 `~/.emacs.d/diary`)。它就是纯文本,格式:

```
June 20, 2026  Pay rent
2026-06-25  Project deadline
6/25/2026  Meeting with team

&%%(diary-anniversary 6 20 1990) Alice's birthday
&%%(diary-cyclic 7 6 1 2026) Weekly team meeting every Monday

%%(diary-block 6 1 2026 6 30 2026) Conference travel period
```

Diary 支持多种日期语法:
- **具体日期**: `June 20, 2026` 或 `2026-06-20` 或 `6/20/2026`
- **每天**: `&` 开头表示"每天" (反复出现的)
- **Sexp**: `%%(...)` 是 Elisp 调用——`diary-anniversary` (周年纪念)、`diary-cyclic` (周期重复)、`diary-block` (日期范围)、`diary-float` (每月第几个星期几)

Sexp 形式是最强大的——它本质上是"用 Elisp 计算日期"。Emacs 内置了一组 diary-* 函数,你也可以写自己的:

```
&%%(let ((day (calendar-day-of-week date)))
    (or (= day 0) (= day 6)))
Weekend!
```

这个 entry 在"周末"显示——diary 系统会为每个日期调用这个 sexp,如果返回非 nil,事件就显示。

### 7.2 Diary 的 entry 类型

Calendar 模式里有 `i` 前缀的命令——它们**在 diary 文件里插入**对应类型的 entry (基于当前光标日期):

```
i d    insert daily (每天)
i w    insert weekly (每周)
i m    insert monthly (每月)
i y    insert yearly (每年)
i a    insert anniversary (周年)
i c    insert cyclic (周期)
i b    insert block (日期范围)
i f    insert float (浮动的,如某月第三个周一)
```

操作: 在 calendar 里 `g d` 跳到目标日期,按 `i d`,Emacs 自动打开 diary 文件 (没有就创建),插入一个 entry 模板,你填描述。这种"从 calendar 插入 diary"是流式工作流——你看着日历,顺手就把事件记了。

### 7.3 显示 diary

Diary 可以在两个地方显示:

**1. Calendar 里高亮**:

```
M-x diary-show-all-entries
```

或更常用的——diary 默认会在 calendar 里以**小三角** (`>` 在日期后面) 标记有事件的日期。`v d` (view-diary-entries) 显示光标日期的 diary 条目。

**2. 独立 buffer**:

```
M-x diary       显示所有日记条目
M-x diary-fancy-display   带颜色显示
```

`M-x diary` 把所有 entry 收集成一个 buffer——你可以浏览、搜索、跳转。这是 Agenda 的前身——Org agenda 就是模仿这个 + 加了 TODO 状态。

### 7.4 自动邮件提醒

```
M-x diary-mail-entries
```

这会**发邮件**给你列未来 7 天的 diary 条目。需要配置:

```elisp
(setq diary-mail-addr "you@example.com")
(setq diary-mail-days 7)
```

配合 cron 每天定时跑——这就是一个 DIY 的"每日提醒"系统。今天 (2026) 看起来朴素,但在 1990 年代,这是相当强大的功能。

---

## 8. 与 Org-agenda 的关系

### 8.1 层次

Emacs 的时间工具有三层:

```
calendar        日期算术、跨日历、节假日、月相
   ↓
diary           纯文本事件记录、自动提醒
   ↓
org-agenda      TODO + 事件 + 项目 + 文档 (高层抽象)
```

每一层都建立在下一层之上。Org-agenda 内部使用 calendar 的日期算术 (`org-read-date` 就是 calendar 的包装);Org 的时间戳 (`<2026-06-20>` 和 `<2026-06-20 +1w>` 这种 repeater) 是 Diary sexp 语法的简化版。

### 8.2 选择

什么场景用 Diary,什么用 Org?

| 场景 | 推荐 | 原因 |
|---|---|---|
| 项目 TODO | Org | 需要状态、priority、tags |
| 工作日志 | Org | 需要 capture、refile、链接 |
| 生日 / 周年纪念 | Diary | 简单,一辈子只需要一条 |
| 公共假期 (新年、国庆) | Diary 或 calendar-holidays | 全局生效 |
| 月度账单 (固定日期付款) | Diary | 比 Org capture 轻 |

经验法则: **如果一个事件"就是事件,没有 TODO 状态",用 Diary;如果一个事件是"任务",用 Org**。

### 8.3 Org 可以读 Diary

Org 甚至可以**显示 diary 文件里的 entry** 在 agenda 里:

```elisp
(setq org-agenda-include-diary t)
```

这样 `M-x org-agenda` 显示的内容里就**混入**了 diary 条目——它们和 Org 的 SCHEDULED/DEADLINE 平等显示。这是 Org 设计者的诚意——他们不强制你迁移所有 diary 到 org,允许共存。

---

## 9. 创造性例子

### 9.1 自动生日提醒

经典用法——给亲友记生日,**一辈子只输一次**:

在 `~/diary` 里:

```
&%%(diary-anniversary 6 20 1990) Alice's birthday (%d years)
&%%(diary-anniversary 3 15 1992) Bob's birthday (%d years)
```

`%d` 是年龄的占位符——diary 会自动算出"今年是这个生日第几年"。从此每次打开 calendar,生日那天就会高亮——比商业日历简单且永久 (不依赖任何服务)。

### 9.2 计算项目截止日

"项目启动后 90 天交付"——你不想数日历:

```
M-x calendar
g d    → 跳到启动日 2026-06-20
+      → 输 90 → 跳到 2026-09-18
i d    → 在那天插入 diary "Project deadline"
```

4 步完成——比任何 GUI 都快。计算的结果 (9 月 18 日) 还自动记进了 diary,成为永久记录。

### 9.3 Export 日历为 HTML

Diary 可以导出:

```
M-x diary-print-entries
```

或者更强大——用 `cal-html` 包 (Emacs 内置):

```
M-x cal-html-insert-year-html
```

这生成一个完整 HTML 年历,带链接到各月。对做项目报告、个人主页展示有用——纯 Emacs,不需要外部工具。

### 9.4 计算工资周期

假设你是双周付薪——每两周的周五发工资。可以用 cyclic:

```
&%%(diary-cyclic 14 6 5 2026) Payday
```

`diary-cyclic 14` 表示每 14 天重复一次,起点是 2026 年 6 月 5 日 (周五)。从此每次打开 calendar,发薪日都自动高亮。

### 9.5 计算每月第三个周三的会议

固定"每月第三个周三"是会议常见安排。用 float:

```
&%%(diary-float t 3 3) Monthly team meeting
```

`t` 表示每月,`3` 是周三 (周日 = 0),`3` 是第三个。Diary 自动算出每月第三个周三——比人工记忆可靠。

### 9.6 计算两个日期间隔

"从今天到年底还有多少天":

```
M-x calendar
.              → 跳到今天 2026-06-20
C-SPC          → mark 今天
g d 2026 12 31 → 跳到年底
M-=            → echo area 显示 "195 days"
```

195 天——这就是"到年底的天数"。同样可以算"距离某 event 还有多少天"——比心算准确,比外部工具快。

---

## 10. 陷阱

### 10.1 Diary 文件位置

默认是 `~/diary`,但有些系统 (尤其是 macOS) HOME 路径异常。检查:

```
C-h v diary-file RET
```

如果显示的不是你期望的路径,改:

```elisp
(setq diary-file "~/.emacs.d/diary")
```

很多人配了 diary 但 calendar 里看不到任何标记——99% 是路径不对。

### 10.2 日期格式敏感

Diary 解析日期格式时**非常严格**——`June 20, 2026` (英文月份名 + 逗号) 和 `June 20 2026` (无逗号) 行为不同。中国大陆用户常写 `2026年6月20日`——Diary **不识别**这种格式,会忽略。

最稳的格式是 ISO: `2026-06-20`——`calendar-date-style` 默认为 `european` 或 `american` 会影响解析顺序。如果你不确定,用 ISO + 明确:

```elisp
(setq calendar-date-style 'iso)
```

### 10.3 性能 (大量 entry)

Diary 默认在 calendar 启动时**扫描整个文件**——如果你的 diary 有几千条 (几十年累积),calendar 会变慢。解决:

```elisp
(setq diary-show-holidays-flag t)  ; 设为 nil 跳过节假日检查
(setq mark-diary-entries-in-calendar nil)  ; 不标记,提速
```

或更激进——把 diary 拆成多个文件 (按年分),只 include 当前年的。

### 10.4 Anniversary 的"起始年"语义

`diary-anniversary` 的第三个参数是**起始年**——例如 `(diary-anniversary 6 20 1990) Alice (%d years)` 在 2026 年 6 月 20 日显示"36 years"。但如果你输错起始年 (比如 1990 写成 19900),`%d` 会变成奇怪的负数。**记 anniversary 时务必 double-check 年份**。

### 10.5 时区配置

`sunrise-sunset` 完全依赖 `calendar-latitude` / `calendar-longitude` / `calendar-time-zone`——如果没配,Emacs 用默认值 (Cambridge, Massachusetts),结果完全错。配这三个变量是 sunrise-sunset 工作的前提。

---

## 11. 练习

1. 打开 calendar,跳到 2030 年 1 月 1 日,看是周几
2. 算 100 天后是哪天 (从今天 2026-06-20 算)
3. 算 2026-06-20 到 2026-12-31 的间隔天数
4. 在 `~/diary` 添加一条 "2026-12-25 Christmas" entry,打开 calendar 看标记
5. 加一条 anniversary: 你 (或假想的) 生日,自动显示年龄
6. 配置 sunrise-sunset 显示你所在城市的日出日落
7. 显示 2026 年所有中国节假日 (用 `holiday-fixed`)
8. 显示 2026 年 6 月的月相
9. 查询 2026 年 6 月 20 日的希伯来历日期
10. 查询 2026 年 6 月 20 日的伊斯兰历日期
11. 在当前 buffer (任意文本文件) 插入 2026 年 6 月的月历
12. 配置 `org-agenda-include-diary`,在 org-agenda 里看 diary 条目
13. 写一个 cyclic entry,每周一提醒"周报"
14. 写一个 float entry,每月第三个周五提醒"月度复盘"
15. 用 `M-x diary-mail-entries` 给自己发一封未来 7 天的日程邮件 (需要 mail 配置)
16. 导出 2026 年的 HTML 年历 (`cal-html-insert-year-html`)
17. 写一个自定义 diary sexp,只在工作日显示 "workday"

---

## 12. 自测

1. Diary 文件默认叫什么名字?在哪个目录?
2. `i a` 在 calendar 里干啥?
3. `+` 加 100 天是相对于什么日期?
4. `g d` 跳到日期,`.` 跳到哪?
5. Diary 和 Org-agenda 关系?
6. 怎么显示中文日期对应的希伯来历?
7. `%%(diary-cyclic 7 6 1 2026)` 是什么意思?

**答案**:
> 1. `~/diary` (或 `~/.emacs.d/diary`,取决于配置)
> 2. 插入 anniversary entry (周年纪念)
> 3. 相对于 calendar 里**当前光标**的日期
> 4. 今天 (`.` = calendar-goto-today)
> 5. Calendar 是基础 (日期算术),Diary 是事件记录,Org-agenda 是高层抽象 (TODO + 事件 + 项目)
> 6. 在 calendar 里跳到那天,按 `p h` (print hebrew)
> 7. 从 2026-06-01 起每 7 天重复一次

---

## 13. 下一步

读完这个文件,你已经理解了 Emacs 的"时间底层"——Calendar 和 Diary。

进入 Module 5 的其他专题 (`gud-debugging.md`, `mule-international.md`) 学更多内置功能。
