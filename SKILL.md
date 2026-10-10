---
name: lark-personal-tasks
description: "使用用户的飞书 Tasks 多维表格管理个人待办，并结合个人日历查看当天安排、规划或调整单个任务的时间段。当用户说我的待办、Tasks、今天做什么、安排今天、给任务排时间、改时间块、复盘今天时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 飞书个人 Tasks 与日历

用户任务的唯一数据源是名为 **Tasks 的飞书多维表格（Bitable/Base）**，不是飞书原生“任务/待办”（task app）。飞书日历是用户查看每日安排的时间轴，不替代 Tasks 表。用户认可排期表后，先定位或创建 Tasks 表记录并更新当天状态，再同步个人主日历，最后回写排期字段与事件 ID；只更新表格不更新日历不算完成排期。用官方 `lark-cli` 的用户身份操作。若环境中已安装 `lark-shared`、`lark-base`、`lark-calendar` Skills，操作前按需阅读；否则以当前 CLI 的 `--help` 和 `lark-cli schema` 核对命令。不要改用飞书原生任务清单（task app），也不要另建 Projects 等表。

## 固定位置

- Base：[Tasks](https://scn0cjx73cs5.feishu.cn/base/KkKJbeFuaayQHysyH1Wc9frRnPe)，`base_token`: `KkKJbeFuaayQHysyH1Wc9frRnPe`，`table_id`: `tbl9BFvxgcbQgJ5D`，时区 `Asia/Shanghai`
- 个人主日历：`feishu.cn_RgTfhrnm2Babh1MHEonaHd@group.calendar.feishu.cn`
- 视图：`Inbox`、`Today`（今天到期且非 Done，**不是**当天日历安排）、`Upcoming`（有 Due 且未完成、按 Due 升序）、`All`、`看板`（按 Tags 分组）、`今日安排`（Status=Doing）、`日历`（按 Scheduled Start/End）。看当天实际安排要读飞书日历。

## 字段语义

| 字段 | 用途 |
| --- | --- |
| Task | 任务标题；一条记录是一个待办。 |
| Status | Inbox=刚记录（默认）；Doing=决定今天要做，可能还没排时间段；Done=完成。 |
| Due | 截止/到期硬死线，可带时间；**不等于日历排期**。 |
| Priority | P0=今天必须处理；P1=重要；P2=普通（默认）；P3=有空再处理。 |
| Project | Research / OPC / School / Personal；未明确指定时留空。注意 Campus 是 Tag，不是 Project。 |
| Tags | 多选：学习/科研/待阅读/生活/Campus；未明确指定时留空，按需增添选项。 |
| Notes | 任务背景、下一步或补充信息。 |
| Created | 自动创建时间，只读。 |
| Scheduled Start / Scheduled End | 计划做这件事的日历时间段；未排期留空。这是"计划做"，Due 是"必须完成"，两者分开。 |
| Calendar Event ID | 对应主日历事件的 ID，技术字段，在各视图中隐藏；未排期留空。 |

不要为填满字段猜 Project、Tags、Due 或具体时刻。**创建任务前先用 `+record-search` 按标题检索，优先复用已有记录，避免重复任务。**

## 排期原则

- 用户只要求记录待办时不创建日历事件；只有用户要求安排/写入日历时才创建或修改日程。
- Status 设为 Doing 表示"决定今天做"，不要求一定已有排期；时间段可与用户讨论后再排。
- 结合明确截止时间、优先级、任务背景和用户给出的当天约束提出方案；优先少量核心任务，不塞满日历，不把过去的作息当作今天的固定规则。
- 不安排专门的休息、机动或出发/通勤提醒块；用户在任务中自行休息、自己掌握出发时间。空档直接留空，不创建占位事件；正式活动按实际起止时间安排。
- 相似任务不要连续安排，中间穿插不同类型的任务；同时遵循依赖顺序，例如先学教程，再写 CLI，不能为了交替而倒置。
- 专注任务以约 1 小时为参考；轻量试用通常约 30 分钟。根据用户给出的任务时长调整，不机械凑满整点。
- 一个任务只关联一个主时间段（写回 `Calendar Event ID`）；同一任务的其余时间块可另建普通日历事件占格，不回写。
- 新建或调整排期必须先向用户交付具体排期表（任务、日期、起止时间，活动注明地点），待用户认可该版本后，才能写入日历及任务表的排期字段。即使信息齐全、方案确定，或用户笼统要求“安排一下”，也不能跳过确认；已认可的同一版本执行时无需逐步重复确认，若再改方案须重新认可。

## 操作流程

1. `lark-cli auth status` 检查用户身份；需要授权时按 `lark-shared` 处理。首次写入前用 `lark-cli base +field-list --base-token KkKJbeFuaayQHysyH1Wc9frRnPe --table-id tbl9BFvxgcbQgJ5D --as user` 确认字段仍匹配。
2. 查询：`+record-list` 指定上述 Base/table 和 `--view-id` 按视图读取；按标题定位用 `+record-search`，确定唯一记录 ID 后才用 `+record-upsert` 更新。
3. 规划"今天做什么"时不要只读 `Today`：还要检查其余未完成项（含 Inbox、未来到期的 P1 等）；按需用视图或搜索即可，不必每次全表读取。
4. 当天日程：`lark-cli calendar +agenda --start <当天00:00+08:00> --end <次日00:00+08:00> --as user`；找空档用 `+freebusy`。多个可行方案需用户选择时，按 `lark-calendar` 的排期指引展示候选。
5. 新排期：先确认任务未关联现有事件，再在**个人主日历**创建一个 `busy`、`private`、无视频会议的事件；标题用任务名，描述可链接 Tasks Base。优先 `lark-cli calendar events create`，显式传 `vchat: {"vc_type":"no_meeting"}`，避免快捷命令默认附带视频会议。**`start_time`/`end_time` 的 `timestamp` 是秒级 Unix 时间戳（UTC 秒，非毫秒）**；常见错误是传毫秒或时区用错导致偏移数小时。传秒级时间戳 + `"timezone":"Asia/Shanghai"`，建完用 `calendar events get` 读回核对显示时间。成功后写入 `Scheduled Start`/`Scheduled End` 与返回的 `Calendar Event ID`，并把 Status 设为 Doing。
6. 改排期：若有 `Calendar Event ID`，先读取该事件，修改**同一个事件**，再更新两个时间字段。ID 无效或记录含糊时先查明，不要悄悄创建第二个时间段。若日历写入成功而 Base 写入失败，恢复原日程或清理刚创建的事件，再如实报告实际状态。
7. 完成：只将 Status 设为 Done，不要因此自动删除历史日历事件。复盘时，把当天未完成、仍是 Doing 的任务改回 Inbox，删除对应日程并清空三个排期字段（没执行的时间段不算历史记录）；只有用户要求取消排期时，才对其他任务同样处理。删除记录或事件用 `+record-delete` / `calendar events delete`，需要明确目标，不要凭标题猜测。

写入后读取记录和对应日程核对结果，向用户说明任务状态、截止时间、排期时间段及是否存在未同步部分。CLI 参数和权限可能更新，执行前以当前 `--help` / `lark-cli schema` 及飞书专用 Skill 为准。
