# 第 9 章 多 Agent 调度：Team 不是函数调用，而是 session 间消息协作

> 这一章读 Agent Team。重点理解：AgentScope 的多 agent 不是 leader 直接调用 worker 函数，而是创建 worker agent/session，通过 MessageBus 发 HintBlock，并用 WakeupDispatcher 唤醒 session。

---

## 9.1 配套源码

打开：

- `src/agentscope/app/_tools/_team_create.py`
- `src/agentscope/app/_tools/_agent_create.py`
- `src/agentscope/app/_tools/_team_say.py`
- `src/agentscope/app/_tools/_team_delete.py`
- `src/agentscope/app/_tools/_team_tool_base.py`
- `src/agentscope/app/_service/_toolkit.py`
- `src/agentscope/app/middleware/_inbox_middleware.py`
- `src/agentscope/app/message_bus/_base.py`
- `src/agentscope/app/_manager/_wakeup_dispatcher.py`
- `examples/agent_service/main.py`

---

## 9.2 Team 的核心实体

Team 能力涉及三类记录：

```text
TeamRecord
  team id、leader session_id、member agent ids、name、description

AgentRecord(source="team")
  worker agent 的定义：name、system_prompt、context/react config

SessionRecord
  每个 worker 有自己的 session、state、workspace_id、model config
```

注意：worker 是完整 agent，不是一个函数或线程。它有自己的 `AgentState`，自己的 session lock，自己的 inbox。

---

## 9.3 Toolkit 如何决定 leader/worker 拿到哪些 team tools？

看 `get_toolkit()`：

```text
如果 agent_record.source == "team":
    tools.append(TeamSay(role="worker"))
否则：
    tools += [TeamCreate, AgentCreate, TeamSay(role="leader"), TeamDelete]
```

这就是角色能力控制：

- 用户创建的普通 agent 可以成为 leader，拥有建队和创建 worker 的能力。
- team worker 只能通过 `TeamSay` 汇报或沟通，不能继续创建新 worker。

工具内部还会再次检查运行时状态：比如当前 session 是否已经在 team 里、是否是 leader。

---

## 9.4 `TeamCreate`：创建团队

`TeamCreate.__call__()` 做：

1. 从 storage 读取当前 session
2. 检查 session 是否存在
3. 检查当前 session 是否已经属于 team
4. 创建 `TeamRecord`
5. `upsert_team()` 保存 team
6. `set_session_team_id()` 把当前 session 标为 team leader session
7. 返回 team id

一个 session 只能属于一个 team。源码里明确防止重复创建。

---

## 9.5 `AgentCreate`：创建 worker agent + worker session

这是 Team 最关键的工具。

### 第一步：前置检查

它会检查：

- 当前 session 是否在 team 中
- team 是否存在
- 当前 session 是否是 team leader
- subagent_type 是否存在
- worker name 是否和已有 leader/worker 重名

为什么要检查 name 唯一？因为 `TeamSay(to=...)` 是按成员显示名路由，不是按 agent_id。重名会导致无法确定收件人。

### 第二步：根据模板生成 system prompt

默认模板大概包含：

```text
You are {member_name}, a member of team '{team_name}' led by {leader_name}.
Team purpose: {team_description}
Your role: {member_description}
You communicate ... through TeamSay.
```

这说明 worker 的“团队身份”和“汇报规则”不是靠 leader 每次口头提醒，而是固化进 worker system prompt。

### 第三步：创建 worker AgentRecord

```text
AgentRecord(source="team", data=AgentData(...))
```

`source="team"` 很重要：后续 `get_toolkit()` 会因此只给它 worker-side 工具。

### 第四步：创建 worker SessionRecord

worker session 会继承 leader 的模型配置：

```text
chat_model_config = leader_session.config.chat_model_config
fallback_chat_model_config = leader_session.config.fallback_chat_model_config
workspace_id = leader_session.config.workspace_id
```

权限上下文由模板和 leader 当前权限合并。

### 第五步：把 worker 加进 team.member_ids

```text
team.data.member_ids = [..., worker_agent.id]
upsert_team(team)
```

### 第六步：投递初始任务并唤醒

`AgentCreate` 会把 `prompt` 包成：

```text
HintBlock(
  hint='<team-message from="leader">\n...\n</team-message>',
  source=json.dumps({"label": "team_message", "sublabel": leader_name})
)
```

然后：

```text
message_bus.inbox_push(worker_session.id, hint)
message_bus.enqueue_wakeup(user_id, worker_session.id, worker_agent.id)
```

这一步回答了一个关键问题：worker 为什么创建后会自己开始工作？因为它的 inbox 收到任务，并且 wakeup dispatcher 会启动一次 `ChatService.run(input_msg=None)`。

---

## 9.6 `TeamSay`：成员之间如何发消息？

`TeamSay.__call__()` 做：

1. 读取当前 session，找到 team_id
2. 读取 team
3. 找 leader session 和 leader agent name
4. 遍历 member_ids，建立目录：`name -> (session_id, agent_id)`
5. 根据 `to` 决定单播或广播
6. 构造 `HintBlock(<team-message from="sender">...)</team-message>`
7. 对每个 recipient：`inbox_push + enqueue_wakeup`

所以 TeamSay 的本质是：

```text
把一段 team message 放进对方 session inbox，然后发 wakeup 信号
```

不是直接调用对方 agent。

---

## 9.7 `InboxMiddleware`：消息如何进入 worker 的上下文？

worker 被 wakeup 后，`ChatService.run(input_msg=None)` 创建 Agent 并进入 `reply_stream()`。

在 reasoning 前，`InboxMiddleware.on_reasoning()` 会：

1. `bus.inbox_drain(agent.state.session_id)`
2. 把 payload 反序列化成 `HintBlock`
3. append 到 `agent.state.context`
4. yield `HintBlockEvent` 给前端
5. 调 `next_handler()` 继续 reasoning

如果 context 最后一条是当前 agent 的 assistant message，就把 hint 追加进去；否则新建一个 assistant message 容器。

这就是跨 session 消息进入模型上下文的位置。

---

## 9.8 `WakeupDispatcher`：为什么需要唤醒器？

MessageBus 的 inbox 是队列，消息放进去后，如果目标 session 没有运行，就没人会读。

WakeupDispatcher 做的是：

```text
订阅 wakeup signal
  → dequeue_wakeups()
  → 如果 session 没在运行
  → 检查 session 还存在
  → registry.spawn(ChatService.run(input_msg=None))
```

这让 worker/leader 可以异步协作：一个 agent 发消息，另一个 agent 不需要正在运行，也会被唤醒处理。

---

## 9.9 为什么 ChatService 要跳过“正在等工具确认”的 wakeup？

`ChatService.run()` 里有一个 guard：如果 `input_msg is None`，但 agent 最后一条 assistant message 里有 `ASKING` 或 `SUBMITTED` 的 tool call，就跳过 wakeup。

原因：agent 正在等用户确认或外部执行结果。如果此时用 `None` 强行跑，会触发 `_check_incoming_event()` 报错。

inbox 消息不会丢，等用户确认/外部结果回来后，下一次 reasoning 时 `InboxMiddleware` 会 drain。

---

## 9.10 经典问题：多 agent 是并行的吗？

从系统角度看，可以并行。每个 worker 是独立 session，有自己的 session lock。不同 session 可以被不同进程/任务同时运行。

但同一个 session 内部不会并发运行，因为 `MessageBus.session_run(session_id)` 提供 per-session lock。

```text
leader session   一次只跑一个 run
worker A session 一次只跑一个 run
worker B session 一次只跑一个 run

leader、worker A、worker B 之间可以同时跑
```

---

## 9.11 本章检查题

1. `AgentCreate` 创建了哪些 record？
2. worker 的第一条任务是以什么形式送到 inbox 的？
3. `TeamSay(to=None)` 和 `TeamSay(to="researcher")` 有什么区别？
4. 为什么 worker 不能创建新的 worker？
5. wakeup run 为什么传 `input_msg=None`？
6. 为什么 session lock 仍然必要？
