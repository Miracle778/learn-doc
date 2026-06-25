# 第 16 章 安全模型：AgentScope 的安全边界在哪里？

> 这一章专门提炼安全相关功能。AgentScope 的安全不是一个单独的 `security.py`，而是分布在消息校验、工具权限、workspace 隔离、人机确认、session lock、team tool gating、长期记忆隔离等多个层面。

---

## 16.1 先回答：安全相关内容原来分散在哪些章节？

如果你想快速回看，安全相关内容主要在这些章节：

- 第 2 章：消息角色校验，防止用户伪造 tool call / tool result
- 第 3 章：`PermissionContext` 是 session state 的一部分，权限规则可持久化
- 第 4 章：continuation event 校验，避免错误恢复等待中的工具调用
- 第 5 章：工具 schema 校验、权限决策、人机确认、工具结果写回
- 第 6 章：工具结果截断和上下文压缩，避免超长输出污染上下文
- 第 8 章：middleware 边界，避免扩展能力直接破坏 Agent 核心状态机
- 第 9 章：team leader/worker 能力隔离，跨 session inbox/wakeup 协作
- 第 11 章：`session_run` 分布式锁，避免并发 run 写坏同一 session state

这一章把它们串成一个完整安全模型。

---

## 16.2 配套源码地图

安全相关源码主要看：

- `src/agentscope/permission/_types.py`
- `src/agentscope/permission/_context.py`
- `src/agentscope/permission/_rule.py`
- `src/agentscope/permission/_decision.py`
- `src/agentscope/permission/_engine.py`
- `src/agentscope/tool/_base.py`
- `src/agentscope/tool/_builtin/_bash.py`
- `src/agentscope/tool/_builtin/_read.py`
- `src/agentscope/tool/_builtin/_write.py`
- `src/agentscope/tool/_builtin/_edit.py`
- `src/agentscope/message/_base.py`
- `src/agentscope/message/_block.py`
- `src/agentscope/agent/_agent.py`
- `src/agentscope/app/_service/_chat.py`
- `src/agentscope/app/_service/_toolkit.py`
- `src/agentscope/app/_tools/_agent_create.py`
- `src/agentscope/app/_tools/_team_say.py`
- `src/agentscope/app/message_bus/_base.py`
- `src/agentscope/workspace/`

---

## 16.3 安全边界总图

```text
外部用户输入
  │
  ├─ Msg role/content 校验
  │    └─ user 不能携带 tool_call / tool_result / thinking
  │
  ▼
Agent ReAct loop
  │
  ├─ continuation event 校验
  │    └─ 只有 agent 正在等待确认/外部结果时才能恢复
  │
  ├─ tool call 输入校验
  │    ├─ 工具是否存在
  │    ├─ JSON repair
  │    └─ jsonschema.validate
  │
  ├─ PermissionEngine
  │    ├─ mode: DEFAULT / ACCEPT_EDITS / EXPLORE / BYPASS / DONT_ASK
  │    ├─ deny_rules / ask_rules / allow_rules
  │    ├─ tool.check_permissions()
  │    └─ safety ASK / bypass-immune / suggested rules
  │
  ├─ human-in-the-loop
  │    ├─ RequireUserConfirmEvent
  │    └─ UserConfirmResultEvent
  │
  ├─ workspace / working directory
  │    ├─ path realpath 校验
  │    ├─ ACCEPT_EDITS 只自动允许工作目录内编辑
  │    └─ Docker / E2B sandbox 可隔离执行环境
  │
  ├─ tool result 控制
  │    ├─ tool_result_limit
  │    ├─ 截断
  │    └─ offloader
  │
  └─ 服务层并发保护
       ├─ MessageBus.session_run(session_id)
       └─ state 保存必须在 lock 内
```

安全模型的核心思想是：**不要信任模型输出，也不要信任外部输入；所有能产生副作用的动作都必须经过结构化校验和权限决策。**

---

## 16.4 第一层：消息角色校验，防止用户伪造内部事件

源码：

- `src/agentscope/message/_base.py`
- `src/agentscope/agent/_agent.py` 的 `_handle_incoming_messages()`

`Msg.validate_role_content()` 限制不同 role 能携带哪些 block：

```text
user    只能携带 text / data
system  只能携带 text
assistant 可以携带 tool_call / tool_result / thinking / hint 等
```

`_handle_incoming_messages()` 还会拒绝外部输入中的：

- system role message
- tool_call block
- tool_result block
- thinking block

这很重要。否则恶意用户可以构造：

```text
UserMsg(content=[ToolCallBlock(name="Bash", input="...")])
```

让系统误以为模型要执行工具。

AgentScope 的做法是：用户输入只能是用户输入；工具调用必须来自 assistant/model 输出，并进入工具生命周期。

---

## 16.5 第二层：工具调用前的结构化校验

源码：

- `src/agentscope/agent/_agent.py` 的 `_execute_tool_call()`

模型生成 `ToolCallBlock` 后，不会直接执行。先走：

```text
toolkit.check_tool_available()
  → _json_loads_with_repair(tool_call.input, tool.input_schema)
  → jsonschema.validate(parsed_input, tool.input_schema)
```

这层防的是：

- 模型调用不存在的工具
- 模型传错参数
- JSON 不完整或格式错误
- 参数绕过工具 schema

如果失败，AgentScope 会把错误变成 `ToolResultBlock(state="error")` 写回上下文，而不是直接崩溃。这样模型下一轮能修正参数。

---

## 16.6 第三层：PermissionContext 保存权限状态

源码：

- `src/agentscope/permission/_context.py`
- `src/agentscope/state/_state.py`

`PermissionContext` 包含：

```python
mode: PermissionMode
working_directories: dict[str, AdditionalWorkingDirectory]
allow_rules: dict[str, list[PermissionRule]]
deny_rules: dict[str, list[PermissionRule]]
ask_rules: dict[str, list[PermissionRule]]
```

它存在 `AgentState.permission_context` 里，所以是 session state 的一部分。

这意味着：

- 用户批准的 rule 可以持久化到当前 session。
- session 恢复后权限上下文仍然存在。
- worker agent 可以从 leader 继承 permission mode、working directories、rules。
- workspace workdir 可以在 `ChatService` 里加入 permission context。

权限不是 prompt 里的几句话，而是结构化运行时状态。

---

## 16.7 第四层：PermissionMode 五种模式

源码：

- `src/agentscope/permission/_types.py`
- `src/agentscope/permission/_engine.py`

AgentScope 支持五种 mode：

```text
DEFAULT
  默认安全模式。没有 allow rule 或工具明确 ALLOW 时，大多需要 ASK。

ACCEPT_EDITS
  快速开发模式。自动允许工作目录内的读写和部分路径受控的文件系统命令。

EXPLORE
  只读探索模式。允许 Read/Grep/Glob 和只读 bash；修改操作 DENY。

BYPASS
  跳过大多数权限检查。适合你完全信任且有 sandbox 的环境。

DONT_ASK
  无人值守模式。所有 ASK 都转成 DENY，避免后台任务卡住或乱执行。
```

这五种模式回答的是不同运行场景：

- 人在旁边看着：`DEFAULT` 或 `ACCEPT_EDITS`
- 只想让 agent 调研代码：`EXPLORE`
- 完全隔离沙箱里跑：谨慎用 `BYPASS`
- 定时任务、后台任务：`DONT_ASK`

---

## 16.8 PermissionEngine 的评估顺序

源码：

- `src/agentscope/permission/_engine.py`

不同 mode 有不同 `_check_*` 方法，但核心思想是有顺序的。

### DEFAULT 模式

```text
1. deny rules 命中 → DENY
2. ask rules 命中 → ASK
3. tool.check_permissions()
   ├─ ALLOW / DENY → 直接返回
   ├─ safety ASK → ASK，且 allow rules 不能覆盖
   └─ PASSTHROUGH / 普通 ASK → 继续
4. allow rules 命中 → ALLOW
5. 默认 → ASK
```

这个顺序很讲究：

- deny 最高优先级。
- safety ASK 比 allow rule 更强，防止危险操作被宽泛 allow rule 绕过。
- 没有明确允许时默认询问用户。

### EXPLORE 模式

```text
1. deny rules → DENY
2. ask rules → ASK
3. tool.check_read_only()
   ├─ True → ALLOW
   └─ False → DENY
```

注意：EXPLORE 不看 allow rules。因为“只读”承诺不能被 allow rule 打破。

### ACCEPT_EDITS 模式

```text
1. deny rules
2. ask rules
3. read-only fast path → ALLOW
4. tool.check_permissions()，例如检查写入路径是否在 working directory
5. allow rules
6. 默认 ASK
```

这是开发时常用的模式：读操作不打扰，工作目录内编辑自动允许，其他危险操作仍然问。

### BYPASS 模式

BYPASS 会跳过 ASK，包括一些 safety ASK。源码注释非常直白：只适合 sandbox/container 或完全信任 agent 的环境。

### DONT_ASK 模式

DONT_ASK 的不变量是：永远不返回 ASK。所有 ASK 都转成 DENY。

这对后台任务很关键：没人确认时，不应该让任务挂住，更不应该默认执行危险动作。

---

## 16.9 第五层：工具自己的安全判断

源码：

- `src/agentscope/tool/_base.py`
- `src/agentscope/tool/_builtin/_bash.py`
- `src/agentscope/tool/_builtin/_read.py`
- `src/agentscope/tool/_builtin/_write.py`
- `src/agentscope/tool/_builtin/_edit.py`
- `src/agentscope/tool/_builtin/_grep.py`
- `src/agentscope/tool/_builtin/_glob.py`

每个工具都有一些安全属性：

```text
is_read_only
is_concurrency_safe
is_external_tool
is_state_injected
check_read_only()
check_permissions()
match_rule()
```

例子：

- `Read/Grep/Glob` 是只读，通常 `is_concurrency_safe=True`
- `Write/Edit` 会修改文件，不能随意并发
- `Bash` 静态上不是只读，但会用 bash parser 判断某条 command 是否只读
- MCP tool 可以根据 annotation 判断 read-only hint

工具还可以做路径安全判断：

```text
_path_in_allowed_working_path(file_path, context)
_is_dangerous_path(file_path)
```

`_path_in_allowed_working_path()` 会用 `realpath` 展开路径，处理 symlink、`/tmp` 到 `/private/tmp` 这种别名，避免简单字符串前缀判断被绕过。

---

## 16.10 第六层：human-in-the-loop 确认

源码：

- `src/agentscope/agent/_agent.py`
- `src/agentscope/event/_event.py`

当权限决策是 `ASK` 或 `PASSTHROUGH`：

```text
ToolCallBlock.state = ASKING
yield RequireUserConfirmEvent(tool_calls=[...])
return  # 当前 reply 暂停
```

用户确认后，会传回 `UserConfirmResultEvent`。

`_handle_incoming_event()` 会处理：

- 用户同意：tool call state 改成 `ALLOWED`
- 用户拒绝：生成 `ToolResultBlock(state="denied")`
- 用户修改 tool call：更新工具名和 input
- 用户接受 suggested rules：写入 PermissionEngine

这说明确认不是“点一下继续”那么简单；用户可以修改参数，也可以把确认转成后续复用的规则。

---

## 16.11 第七层：continuation event 校验

源码：

- `src/agentscope/agent/_agent.py` 的 `_check_incoming_event()`

Agent 暂停后，不是什么事件都能恢复它。

`_check_incoming_event()` 会检查当前最后一条 assistant message：

- 哪些 tool call 处于 `ASKING`
- 哪些 tool call 处于 `SUBMITTED` 且还没有 result

然后校验传入事件：

- 如果 agent 没在等确认，却收到 `UserConfirmResultEvent` → 报错
- 如果确认事件包含不在等待列表里的 tool call id → 报错
- 如果 agent 正在等确认，但你传 `None` → 报错

这层防的是错误恢复、重复确认、跨 reply 混淆、外部结果错投。

---

## 16.12 第八层：workspace / sandbox 隔离

源码：

- `src/agentscope/workspace/`
- `src/agentscope/app/workspace_manager/`
- `src/agentscope/app/_service/_chat.py`

Workspace 相关安全有两层。

### 工作目录权限

`ChatService` 会把 workspace workdir 加入当前 session 的 `permission_context.working_directories`。

这样 `ACCEPT_EDITS` 模式可以只自动允许工作目录内文件修改，而不是允许整个文件系统。

### 执行环境隔离

AgentScope 支持本地、Docker、E2B 等 workspace。Docker/E2B 让工具执行发生在容器/沙箱里，适合更高风险的自动化任务。

但是要注意：sandbox 不是替代权限系统的理由。尤其是 BYPASS 模式，源码注释也强调应只在 sandboxed / containerized 或完全信任场景使用。

---

## 16.13 第九层：工具结果安全与上下文污染控制

源码：

- `src/agentscope/agent/_agent.py` 的 `_split_tool_result_for_compression()`
- `ContextConfig.tool_result_limit`

安全不只是“能不能执行危险命令”，还包括“工具返回的内容会不会污染或撑爆上下文”。

AgentScope 对超长工具结果会：

1. 估算 token
2. 超过 `tool_result_limit` 时截断
3. 如果有 offloader，把剩余内容外置
4. 在保留结果里加入 `<system-reminder>`，告诉模型内容被省略

这防止一次 `cat huge.log`、`grep`、`bash` 输出把后续模型上下文挤爆。

---

## 16.14 第十层：Team 安全边界

源码：

- `src/agentscope/app/_service/_toolkit.py`
- `src/agentscope/app/_tools/_team_tool_base.py`
- `src/agentscope/app/_tools/_agent_create.py`
- `src/agentscope/app/_tools/_team_say.py`

Team 安全主要在三个点。

### 工具分配按角色区分

`get_toolkit()` 里：

```text
source == "team" → worker 只拿 TeamSay
普通 user agent → leader 拿 TeamCreate / AgentCreate / TeamSay / TeamDelete
```

worker 不能继续创建 worker，避免 team 无限递归扩张。

### team tool 运行时检查

Team tools 虽然 `check_permissions()` 总是 allow，但它们在 `__call__()` 内部检查运行时状态：

- 当前 session 是否属于 team
- 当前 session 是否是 leader
- team 是否存在
- worker name 是否唯一
- recipient 是否在当前 team directory 里
- 不能给自己发 TeamSay

也就是说，team tools 的安全不是靠 PermissionEngine，而是靠工具附加逻辑和 storage state。

### worker 权限继承受模板控制

`AgentCreate` 创建 worker 时，会根据 `SubAgentTemplate` 合并 leader 权限：

- `override_leader_mode`
- `extend_leader_permission_rules`
- `extend_leader_working_directories`

所以 worker 权限不是无脑复制 leader，而是模板驱动。

---

## 16.15 第十一层：服务层并发安全

源码：

- `src/agentscope/app/_service/_chat.py`
- `src/agentscope/app/message_bus/_base.py`

同一个 session 可能被多种来源触发：

- 用户发消息
- TeamSay 唤醒
- 后台工具完成
- 定时任务触发
- 外部执行结果返回

如果两个 run 同时修改 `AgentState.context`，状态会损坏。

所以 ChatService 用：

```python
async with self._message_bus.session_run(session_id):
    run agent
    upsert reply message
    update session state
```

`session_run()` 是 per-session distributed lock。状态保存必须在 lock 内完成，防止另一个进程读到旧 state。

---

## 16.16 第十二层：长期记忆隔离

源码：

- `src/agentscope/middleware/_longterm_memory/_mem0/_middleware.py`

Mem0Middleware 搜索时用 filters：

```text
user_id
agent_id 可选
```

`scope_search_by_agent=True` 时，会同时按 user 和 agent 隔离。否则按 user 共享。

这不是传统权限系统，但属于数据隔离安全：不同用户的 memory 不能串；是否跨 agent 共享也要显式配置。

---

## 16.17 经典问题：AgentScope 的安全主线是什么？

一句话：

> 用户输入不能伪造内部动作；模型输出不能绕过工具 schema 和权限系统；工具执行不能越过工作目录和模式策略；服务层不能并发写坏 session；多 agent 不能越权创建或投递到非团队成员。

对应链路：

```text
Msg validation
  → ToolCall schema validation
  → PermissionEngine
  → Tool-specific safety checks
  → Human confirmation
  → Workspace boundary
  → Tool result truncation
  → Session lock
  → Team role gating
  → Memory namespace filters
```

---

## 16.18 学习建议：安全源码怎么读？

建议顺序：

1. 先读 `permission/_types.py`，理解五种 PermissionMode。
2. 再读 `permission/_engine.py`，逐个看 `_check_default`、`_check_explore`、`_check_accept_edits`、`_check_bypass`、`_check_dont_ask`。
3. 回到 `agent/_agent.py` 的 `_execute_tool_call()`，看 PermissionDecision 如何影响工具生命周期。
4. 读 `tool/_base.py` 的路径检查和 dangerous path。
5. 读 built-in tools 的 `check_permissions()`，尤其 Bash/Read/Write/Edit。
6. 读 `ChatService` 的 session lock，理解并发安全。
7. 读 Team tools，看角色能力隔离。

---

## 16.19 本章检查题

1. 为什么 user message 不能携带 `ToolCallBlock`？
2. `DEFAULT`、`EXPLORE`、`ACCEPT_EDITS` 三种模式最核心的差异是什么？
3. `DONT_ASK` 为什么把 ASK 转成 DENY，而不是 ALLOW？
4. `BYPASS` 为什么必须谨慎使用？
5. safety ASK 为什么能绕过 allow rule？
6. `PermissionContext.working_directories` 如何参与文件写入安全？
7. Team tools 为什么不完全依赖 PermissionEngine？
8. `session_run` 保护了哪类安全问题？
9. 长期记忆为什么也需要 user_id / agent_id 隔离？
