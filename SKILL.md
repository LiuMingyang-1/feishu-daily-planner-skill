---
name: personal-tasks
description: "管理用户的个人待办与今日安排：任务存在 Pulse（Cloudflare Worker + D1）里，通过认证 HTTP API 增删改查；可结合飞书日历查看当天安排、给任务排时间块并同步日历事件。当用户说我的待办、Tasks、今天做什么、安排今天、给任务排时间、改时间块、复盘今天时使用。"
metadata:
  requires:
    bins: ["curl", "lark-cli"]
---

# 个人 Tasks 与日历

任务的唯一数据源是 **Pulse 的 D1 `tasks` 表**，通过 `https://pulse.lmy3243635445.workers.dev` 的 HTTP API 访问（`Authorization: Bearer $AGENT_TOKEN`）。它取代了旧的飞书多维表格——不要再往 bitable 写任务，历史记录已全部迁入 D1（原 `rec…` record_id 保留为主键）。飞书日历仍可用于查看当天安排；是否同步日历事件见下文「排期」。任务卡片（待办/今日安排/Reminder/ICS 订阅）全部由 D1 现算，写完 API 即生效，没有同步延迟。完整字段语义与 API 细节见 Pulse 仓库 `docs/TASKS.md`。

## API

- `GET /api/tasks` → `{tasks: [...]}`；`GET /api/tasks/:id` → 单行
- `POST /api/tasks` → 建任务，`title` 必填，201 返回整行（新 id `t_<uuid>`）
- `PATCH /api/tasks/:id` → 部分更新；`done` 0→1 自动盖 `completed_at` 并把 `status` 提到 `Done`
- `DELETE /api/tasks/:id` → 删除
- `POST /api/todo/toggle` → `{id, done, note?}`：完成/撤销，note 作为一行反思原子追加进 `notes`

认证用 `AGENT_TOKEN`（Pulse 部署的 secret；本机 `.dev.vars` / 环境变量里有）。读也需要认证；`GET /api/widgets` 例外（看板公开读）。

## 字段语义

| 字段 | 用途 |
| --- | --- |
| title | 任务标题；一行是一个待办。 |
| status | Inbox=刚记录（默认）；Doing=决定今天要做；Done=完成（打钩会自动置）；Waiting 等自定义标签撤销打钩时会被保留。 |
| done | 完成复选框——**唯一的完成态权威**，status 只是标签。 |
| plan_date | `YYYY-MM-DD`（上海时区）：选中某一天要做、但没有时间块 → 出现在「今日安排 · 未排期」。 |
| due | 截止/到期硬死线，可带时间；**不等于排期**。 |
| priority | P0=今天必须处理；P1=重要；P2=普通（默认）；P3=有空再处理。 |
| type | `Task`（默认，进待办卡）；`Reminder`（进提醒卡，不进未排期）；`Reading` 等其余类型只在 API 层可见。 |
| scheduled_start / scheduled_end | ISO 时间段，即"计划做"的时间块。合法区间（end>start）覆盖当天 → 上今日安排时间轴 + ICS 订阅。 |
| calendar_event_id | 若排期时同步建了飞书日历事件，存其事件 ID；纯 D1 排期留空。 |
| notes | 任务背景、下一步或完成反思（打钩时的 note 逐行追加在此）。 |
| project | Research / OPC / School / Personal；未明确指定时留空。 |
| tags | 自由文本或 JSON 数组字符串：学习/科研/待阅读/生活/Campus；未明确指定时留空。 |
| completed_at | done 0→1 自动盖章；撤销不清除。当天完成的任务沉到待办卡完成区。 |
| created_at / updated_at | 服务端维护，只读。 |

不要为填满字段猜 project、tags、due 或具体时刻。**创建任务前先 `GET /api/tasks` 按标题检索，优先复用已有行，避免重复任务。**

## 排期原则

- 用户只要求记录待办时不写时间块；只有用户要求安排/写入日历时才填 `scheduled_start`/`scheduled_end`。
- `plan_date` 表示"决定今天做但没排钟点"，和 status 无关；时间块可与用户讨论后再排。
- 结合明确截止时间、优先级、任务背景和用户给出的当天约束提出方案；优先少量核心任务，不塞满日历，不把过去的作息当作今天的固定规则。
- 不安排专门的休息、机动或出发/通勤提醒块；空档直接留空，不创建占位事件；正式活动按实际起止时间安排。
- 相似任务不要连续安排，中间穿插不同类型的任务；同时遵循依赖顺序，例如先学教程，再写 CLI，不能为了交替而倒置。
- 专注任务以约 1 小时为参考；轻量试用通常约 30 分钟。根据用户给出的任务时长调整，不机械凑满整点。
- 一个任务只关联一个主时间段；同一任务的其余时间块可在日历里另建普通事件占格，不回写任务行。
- 新建或调整排期必须先向用户交付具体排期表（任务、日期、起止时间，活动注明地点），待用户认可该版本后，才能写入。即使信息齐全、方案确定，或用户笼统要求"安排一下"，也不能跳过确认；已认可的同一版本执行时无需逐步重复确认，若再改方案须重新认可。

## 操作流程

1. 取 `AGENT_TOKEN`：优先环境变量，否则读 Pulse 仓库 `.dev.vars`（`AGENT_TOKEN=` 行）。`curl -s https://pulse.lmy3243635445.workers.dev/api/health` 确认服务在。
2. 查询：`GET /api/tasks` 全量返回（百条量级，直接拉全表即可），按 title/status/plan_date/due 自行过滤；排期前先读一遍当天 `plan_date` 与 `scheduled_*` 已占的格。
3. 规划"今天做什么"时不要只看 plan_date=今天：还要检查其余未完成项（含 Inbox、未来到期的 P1 等）。
4. 当天日程：若需要飞书日历视角，`lark-cli calendar +agenda --start <当天00:00+08:00> --end <次日00:00+08:00> --as user`；找空档用 `+freebusy`。注意 Pulse 的任务时间块走 ICS 订阅进 iOS 日历，**不在飞书日历里**，两边要对账时别忘了课程表 push（courses）也在同一 ICS 里。
5. 排期（两步，缺一不可）：
   a. `PATCH /api/tasks/:id` 写 `scheduled_start`/`scheduled_end`（ISO，含 +08:00 偏移）——这步已让任务上 Pulse 时间轴与 ICS 订阅；
   b. 若用户也在飞书日历里看日程：用 `lark-cli calendar events create` 在个人主日历 `feishu.cn_RgTfhrnm2Babh1MHEonaHd@group.calendar.feishu.cn` 建 `busy`、`private`、无视频会议的事件（标题=任务名，描述可写 Pulse 任务链接），成功后把返回的事件 ID 写进 `calendar_event_id`。`timestamp` 是**秒级** Unix 时间戳；传 `"timezone":"Asia/Shanghai"` 并读回核对。
   日历步失败时：D1 时间块保留有效，如实报告"已上 Pulse/ICS，飞书日历未同步"。
6. 改排期：若 `calendar_event_id` 非空，先改同一个飞书事件再 PATCH 新时间；为空直接 PATCH。ID 失效时先查明，不要悄悄建第二个事件。若日历改成功而 D1 失败，恢复日历或清理事件，再如实报告。
7. 完成：`PATCH {done:true}` 或 `POST /api/todo/toggle {id, done:true, note}`（有反思时）。不要因此自动删除历史日历事件。
8. 复盘：把当天未完成、plan_date=今天 的任务 `PATCH {plan_date:null}` 放回 Inbox；有 `scheduled_*` 但未执行的清掉两个字段（没执行的时间段不算历史记录），有 `calendar_event_id` 的删对应事件并清空该字段。删除任务用 `DELETE`，需要明确目标，不要凭标题猜测。

写入后 `GET /api/tasks/:id` 读回核对，向用户说明任务状态、截止时间、排期时间段及日历同步情况。API 可能演进，以 Pulse 仓库 `docs/TASKS.md` 为准。
