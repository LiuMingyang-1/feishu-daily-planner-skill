---
name: lark-personal-tasks
description: "使用用户的飞书 Tasks 多维表格作为个人待办和排期的唯一写入源。用户确认后写入 Status=Doing 与 Scheduled Start/End；Apple 日历只通过 Pulse 生成的 /api/calendar.ics 订阅展示。当用户说我的待办、Tasks、今天做什么、安排今天、给任务排时间、改时间块、复盘今天时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 飞书个人 Tasks（Apple 日历经 Pulse ICS）

用户任务的唯一数据源是名为 **Tasks 的飞书多维表格（Bitable/Base）**，不是飞书原生“任务/待办”（task app）。待办和排期只写入这张表。用户确认排期草稿后，定位或创建记录，写入 `Status=Doing` 以及 `Scheduled Start` / `Scheduled End`。**只写表格即完成本 skill 的排期写入。**

Apple 日历是下游展示：Pulse 同步同一张 Tasks 表，并生成 `/api/calendar.ics`；用户用 ICS 订阅在 Apple 日历里查看。本 skill 不读写飞书日历，不创建或维护 `Calendar Event ID`，也不另外直接创建 Apple 事件。不要配置订阅、不要轮换 feed token、不要代发 Pulse 同步或部署。用官方 `lark-cli` 的用户身份操作。若环境中已安装 `lark-shared`、`lark-base` Skills，操作前按需阅读；否则以当前 CLI 的 `--help` 和 `lark-cli schema` 核对命令。不要改用飞书原生任务清单（task app），也不要另建 Projects 等表。

## 固定位置

- Base：[Tasks](https://scn0cjx73cs5.feishu.cn/base/KkKJbeFuaayQHysyH1Wc9frRnPe)，`base_token`: `KkKJbeFuaayQHysyH1Wc9frRnPe`，`table_id`: `tbl9BFvxgcbQgJ5D`，时区 `Asia/Shanghai`
- 视图：`Inbox`、`Today`（今天到期且非 Done，**不是**当天时间块）、`Upcoming`（有 Due 且未完成、按 Due 升序）、`All`、`看板`（按 Tags 分组）、`今日安排`（Status=Doing）、`日历`（Tasks 表上按 Scheduled Start/End 的视图，不是飞书日历，也不是 Apple 日历）。看当天 Tasks 时间块时，以下文重叠条件为准，不要只靠视图里开始时间落在今天的行。
- Pulse（下游，只解释、不操作）：Worker 从这张 Tasks 表同步；`/api/calendar.ics` 供 Apple 日历订阅。行为以下文「下游契约」为准。

## 字段语义

| 字段 | 用途 |
| --- | --- |
| Task | 任务标题；一条记录是一个待办。 |
| Status | Inbox=已捕获、尚未决定今天做（默认）；Doing=决定今天做，可以还没有时间段；Done=完成。 |
| Type | 新建记录默认 `Task`。只有用户明确要求提醒时才写 `Reminder`。已有 `Reading` 保持原值，不改写成 `Task` 或 `Reminder`。 |
| Due | 用户自己的目标日期，可带时间。排期不修改 Due。没有单独的 Deadline 字段。Due 不是时间块。 |
| Priority | P0=今天必须处理；P1=重要；P2=普通（默认）；P3=有空再处理。P3 的校园/招聘/宣讲只要落在扫描窗内仍必须列出。 |
| Project | Research / OPC / School / Personal；未明确指定时留空。Campus 是 Tag，不是 Project。 |
| Tags | 多选：学习/科研/待阅读/生活/Campus；未明确指定时留空，按需增添选项。 |
| Notes | 任务背景、下一步或补充信息。 |
| Created | 自动创建时间，只读。 |
| Scheduled Start / Scheduled End | 计划做这件事的时间段；未排期留空。这是「计划做」，Due 是「用户目标」，两者分开。确认后的排期写入这两项，并由 Pulse 同步进 ICS。结束时间必须晚于开始时间。 |
| Calendar Event ID | **废弃。** 历史飞书日历技术字段。新流程不写入、不依赖、不维护，也不批量清空。记录里的旧值默认保留。 |

不要为填满字段猜测 Project、Tags、Due 或具体时刻。**创建前先用 `+record-search` 按标题检索，命中同一任务就更新该记录，避免重复。** 同一任务要占多段时间时，拆成多条独立记录，一条一段，每条自己的一对 Scheduled Start/End；不要在一条记录上虚构第二套排期字段。

## 下游契约（Pulse → Apple，只读说明）

核对过的公开实现是 Pulse `main` @ `b0b84c551a9ab90889a0e7abdb7e6fb28365e3d0`。README 原文：`No separate calendar write or Feishu-calendar sync is needed.` 对应代码是 `src/feishu-transform.ts` 与 `src/ics.ts`。本 skill 不修改 Pulse。

- ICS 收录两条 Scheduled 都非空、且结束晚于开始的记录，不限 Status、不限 Type（含 Done 历史）。过滤依据是 **Scheduled Start 的时间戳**落在约「现在减 3 天」到「现在加 14 天」，不是自然日闭区间。
- 有 `record_id` 时，稳定 UID 为 `record_id@pulse`。改同一条记录的起止时间后，订阅端按该 UID 就地更新。
- Pulse 今日日程卡比 ICS 更窄：Status=Doing、两条 Scheduled 都非空，且 Scheduled Start 的上海日历日是今天。跨午夜、开始于昨天的块可能仍占今天，但不会因为「结束落在今天」进入这张今日卡。安排今天时不要用这张卡代替下面的重叠扫描。
- 同步大约每小时由 Worker cron 执行（UTC 整点）；已登录用户也可以在 Pulse 里手动刷新，并带有短冷却。Apple 日历再按自己的订阅节奏重拉（系统里可为每小时、每天或手动）。`REFRESH-INTERVAL` 只是提示。**Tasks 写入成功不等于 Apple 已经显示。** 向用户说明要等同步和订阅刷新，不要报成即时出现。

本仓库和上述 Pulse 代码都没有 Apple Calendar、EventKit 或 CalDAV 的只读或写入步骤。环境里若另有已经核实可用的只读能力，可以用它核对课表、邀请等非 Tasks 占用，并说明读到了什么。否则必须写明这些关键约束尚未核验，请用户确认空档。不能从 Tasks 查询为空推断 Apple 上的其他来源没有占用，也不能编造读取接口。

## 排期原则

- 用户只要求记下待办时不写 Scheduled，Status 保持 Inbox。只有用户要求安排时间段时才写 `Scheduled Start` / `Scheduled End`。
- Doing 表示决定今天做，可以没有时间段。时间段与用户讨论并确认后再写。
- 排期不改变 Due，也不另造 Deadline。
- 结合明确的 Due、优先级、任务背景和用户给出的当天约束提出方案。优先少量核心任务。日常作息（吃饭、健身、通勤、回工位等）不是今天的硬约束；只有用户把它说成今天的约束时才遵守。不要把它写成 Tasks，除非用户要求记录。
- 不安排专门的休息、机动或出发/通勤块。空档留空，不建占位记录。正式活动按实际起止安排，活动注明地点。
- 相似任务不要连续安排，同时遵守依赖顺序。专注任务约 1 小时，轻量试用以约 30 分钟为参考，按用户给出的时长调整。
- 一个时间块只放一个任务。多段拆成多条记录。
- 新建或调整排期必须先交出草稿（任务、日期、起止，活动注明地点），用户认可该版本后才能写入。信息已经齐全、或用户只说「安排一下」，也不能跳过确认。同一已认可版本执行时不必逐步再问；草稿有任何变化，必须重新确认后再写。

## 扫描今天

用户要安排今天时，先扫描再出草稿。确认前不改 Status、不写 Scheduled、不碰任何日历。日期窗口和时区由程序确定，禁止靠模型心算「三天后是几号」。分页必须完整，截断不算扫完。

1. 用程序取 `Asia/Shanghai` 的今天 `today`，再算出 `window_end = today + 3 个自然日`。窗口含首尾：今天与第 3 天都在内，比较用当地日历日，不用 UTC 的日历日。例如今天是 11 月 5 日，则未完成且 Due 落在当地 11 月 8 日结束前的记录都在窗内；更早的过期未完成同样在窗内。
2. 优先用当前 CLI 文档中的日期过滤拉取「Status ≠ Done 且 Due 非空 且 Due ≤ window_end」。不要编造 `--help` 里没有的过滤参数。过滤不可用时，改为分页列出未完成且有 Due 的记录，再由程序按 `window_end` 切开，并在报告中写明已降级手切。
3. 列表若有下一页（`has_more`、page token 或 CLI 等价字段），必须翻完。

必扫集合，缺一不可。报告里每一块都要出现；某块为空就写「无」。

| 块 | 条件 | 说明 |
| --- | --- | --- |
| A. Doing | Status=Doing | 含昨天留下的；旧时间段可以已经过期，仍要列出 |
| B. Due 窗 | 未完成、Due 非空、Due ≤ window_end | 含过期、含 Type=Reminder、含 P0–P3 |
| C. 校园/招聘/宣讲 | Tags 含 Campus，或标题/Notes 写明招聘、宣讲，且日期在窗内 | **不看 Priority**。P3 也必须点名，不得当成「有空再处理」跳过 |
| D. Inbox | Status=Inbox | 可抽进今天的候选。Reminder 默认只盯、不占时间块 |
| E. 当天 Tasks 时间块 | 两个 Scheduled 都非空，且 Start < 次日当地零点，并且 End > 当日当地零点 | 这是重叠，覆盖跨午夜。只看 Start 落在今天会漏掉前一天跨过来的块 |

`Today` 视图（当天 Due 且未完成）并入 B，不要另讲一套名单。

漏判：

- 窗内的 Campus、招聘、宣讲一律单独点名，即使用户没把它选作重点。
- Due 为空，但 Notes 或标题里有明确日期：列出来请用户核实要不要补 Due。不要静默丢掉，也不要擅自写入 Due。能用程序判断时，注明该日期在窗内、已过期还是在窗外。
- 时区一律按 `Asia/Shanghai`。A、B、C 都要能在报告里对上；某块为空写「无」。

报告先写当地时刻，以及 E 是否已有 Tasks 时间块。接着按块列出 Doing、Due 窗、校园/招聘/宣讲、Inbox 候选，再给草稿表。草稿是单任务时间块（任务、起止；活动写地点）。不要用休息或机动占空档。E 只说明 Tasks 来源的占用。课表和邀请若尚未用已核实的只读能力核对，必须写「尚未核验」，不能写成空闲。

草稿优先消化 Doing 和窗内 Due，再从 Inbox 抽重点。Reminder 默认不进时间块。用户说稍后再排的时段留空。

## 操作流程

1. `lark-cli auth status` 检查用户身份；需要授权时按 `lark-shared` 处理。首次写入前用 `lark-cli base +field-list --base-token KkKJbeFuaayQHysyH1Wc9frRnPe --table-id tbl9BFvxgcbQgJ5D --as user` 确认字段仍匹配（至少含 Task、Status、Due、Scheduled Start、Scheduled End；Type、Priority、Tags 等按表里实际存在的字段使用）。
2. 查询：`+record-list` 指定上述 Base/table，需要时加 `--view-id`。按标题定位用 `+record-search`，确定唯一记录后再 `+record-upsert`。有分页就翻完。
3. 规划今天时走「扫描今天」，不要只读 `Today` 视图。
4. 当天已有 Tasks 时间块只用 E 的重叠条件。不要读取飞书日历。Tasks 侧没有重叠，不能推断 Apple 上其他来源没有占用。
5. 用户已认可排期草稿后：复用同一任务的已有记录，否则新建。新建时若有 Type 字段，写入 `Task`；只有这份草稿明确是提醒才写 `Reminder`。不要改已有 `Reading`。写入 `Scheduled Start`、`Scheduled End`，并把 Status 设为 Doing。用户只要求记下 Inbox 待办、不要排期时，不写这两个时间字段，也不要改成 Doing。不要创建飞书日历事件，不要写 `Calendar Event ID`，不要调用 Apple 日历写入。写完读回该记录，核对 Status 与两个 Scheduled。向用户说明状态、Due、时间段，并说明 Apple 要等 Pulse 同步和订阅刷新，现在还不能当成已经显示。
6. 改已确认的排期：若草稿相对上次认可有变化，先重新确认。然后更新**同一条**记录的两个 Scheduled（以及该范围要求的 Status）。同一段不要另建一条来顶替。多出来的时间段另建记录，一条一段，并先按标题检索避免重复。Base 写入失败就如实报告，不要用飞书日历或 Apple 接口做补偿写入。
7. 完成：只把 Status 改为 Done，保留 Scheduled Start/End 作为历史。不要因为完成去删 ICS 历史、清 Scheduled，或改动 `Calendar Event ID`。
8. 取消尚未执行的排期：只按用户明确点名的范围，清空这些记录的 Scheduled Start 与 Scheduled End。范围里包含「不再决定今天做」时，才把 Status 从 Doing 改回 Inbox。不要清空或改写 `Calendar Event ID`。删除记录用 `+record-delete`，目标必须明确，不要凭标题猜测。

历史飞书日历事件和已有的 `Calendar Event ID` 默认保留，不清理。日常排期不维护它们。用户若另行要求清理旧飞书日程，先单独确认范围；那不属于本流程。

写入后只读回 Tasks 记录做核对。CLI 参数和权限可能变化，执行前以当前 `--help`、`lark-cli schema` 和已安装的飞书 Skill 为准。

## 废弃：飞书日历读写

下列做法已经废弃，**不得再执行**，也没有替代用的日历写入步骤：把飞书个人日历当作排期的读取或写入目标；调用飞书日历 agenda、freebusy；对飞书日历事件做 create、patch、delete、get；把 `Calendar Event ID` 当作同步键；Base 写入失败时按该 ID 回滚或删改飞书事件。旧文档里的个人日历 ID、事件参数和时间戳写法都不再有效。历史字段值和历史飞书事件默认留在原处。
