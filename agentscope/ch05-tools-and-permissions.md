# 第 5 章 工具系统：模型调用工具之前，框架做了哪些保护？

> 这一章读 `_execute_tool_call()`、`Toolkit`、`ToolBase`、`PermissionEngine`。目标是理解：工具调用不是模型说了就执行，中间有 schema、权限、外部确认、结果压缩等完整生命周期。

---

## 5.1 配套源码

打开：

- `src/agentscope/agent/_agent.py`：`_execute_tool_call()` / `_acting()` / `_acting_impl()`
- `src/agentscope/tool/_base.py`
- `src/agentscope/tool/_toolkit.py`
- `src/agentscope/permission/_engine.py`
- `src/agentscope/permission/_decision.py`
- `src/agentscope/message/_block.py`：`ToolCallBlock` / `ToolResultBlock`

---

## 5.2 工具调用生命周期总览

```text
ToolCallBlock(state=pending)
  │
  ▼
检查工具存在且可用
  │
  ▼
_json_loads_with_repair() 修复并解析 input
  │
  ▼
jsonschema.validate() 校验 tool.input_schema
  │
  ▼
PermissionEngine.check_permission()
  │
  ├─ ASK / PASSTHROUGH
  │     └─ state=asking，发 RequireUserConfirmEvent，暂停
  │
  ├─ DENY
  │     └─ 写 ToolResultBlock(state=denied)，返回给模型
  │
  └─ ALLOW
        ├─ state=allowed
        ├─ 如果 external tool：state=submitted，发 RequireExternalExecutionEvent
        └─ 否则 _acting() → toolkit.call_tool() → ToolResultBlock(state=success/error)
```

这条链路是 agent 安全的核心。

---

## 5.3 第一步：检查工具和解析输入

模型输出的工具参数是字符串，而且可能不是完美 JSON。AgentScope 先做修复和 schema 校验：

```text
_json_loads_with_repair(tool_call.input, tool.input_schema)
jsonschema.validate(parsed_input, tool.input_schema)
```

如果失败，不会抛给外部用户，而是通过 `_handle_error_tool_call()` 生成一个工具错误结果写回 context。

这样模型下一轮能看到：

```text
工具调用失败：参数 xxx 不符合 schema
```

然后模型有机会自我修正。

---

## 5.4 第二步：权限系统

权限决策有几类：

```text
ALLOW        允许执行
DENY         拒绝执行，结果写回模型
ASK          需要用户确认
PASSTHROUGH 交给外部确认/处理
```

如果是 `ASK` 或 `PASSTHROUGH`：

- tool call state 变成 `ASKING`
- 发 `RequireUserConfirmEvent`
- 当前 reply 暂停

用户确认后，会通过 `UserConfirmResultEvent` 回来。`_handle_incoming_event()` 会：

- 如果 confirmed：state 改成 `ALLOWED`
- 如果用户修改了 tool input：更新 tool_call.name/input
- 如果用户接受了建议规则：加入 permission engine
- 如果 denied：写一个 denied tool result

---

## 5.5 第三步：执行工具

真正执行在 `_acting_impl()`：

```python
async for chunk in self.toolkit.call_tool(tool_call, self.state):
    yield chunk
```

注意 `_acting()` 只是 raw execution hook，权限和 context 写入不在这里，而在 `_execute_tool_call()` 里。

为什么要这样分层？

- `on_acting` middleware 可以包装纯工具执行，比如 offload 到后台。
- 权限检查、输入校验、context mutation 留在 Agent 主流程里，避免后台任务乱改 state。

源码注释里也提醒：如果工具 `is_state_injected=True`，它拿到的是 live `agent.state`，并发/offload 时要非常谨慎。

---

## 5.6 工具结果如何进入上下文？

工具最终返回 `ToolResponse` 后，AgentScope 构造：

```python
ToolResultBlock(
    id=tool_call.id,
    name=tool_call.name,
    output=...,
    state=chunk.state,
)
```

然后做两件事：

1. `_save_to_context([reserved_tool_result_block])`
2. `_update_tool_call_state(tool_call.id, ToolCallState.FINISHED)`

这样上下文里会有完整轨迹：

```text
assistant message content:
  ToolCallBlock(id=abc, name=Read, input=...)
  ToolResultBlock(id=abc, name=Read, output=...)
```

`id` 对齐非常重要，formatter 和压缩逻辑都依赖它。

---

## 5.7 工具结果太长怎么办？

在写回 context 前，Agent 会调用：

```python
_split_tool_result_for_compression(tool_result_block)
```

如果 token 数没超过 `context_config.tool_result_limit`，原样保留。

如果超过：

1. 找一个边界，只保留前半部分
2. 如果边界 block 是文本，可以按比例截断
3. 剩余部分形成 `offload_tool_result_block`
4. 如果有 offloader，把剩余内容写到外部文件
5. 在保留结果末尾加 `<system-reminder>` 告诉模型结果被截断

这比简单 `output[:50000]` 更好，因为：

- 保留了结构化 block
- 可以 offload 其余内容
- 模型知道自己看到的是截断结果，不会误以为完整

---

## 5.8 经典问题：工具错误为什么也要写回 context？

如果工具参数错了、权限拒绝了、工具不存在了，直接结束会让模型没有修正机会。

把错误作为 `ToolResultBlock(state="error")` 写回 context 后，模型下一轮可能会：

- 修正 JSON 参数
- 换一个工具
- 向用户解释权限被拒绝
- 请求用户提供更多信息

这就是 ReAct 的“观察结果”思想：成功是观察，失败也是观察。

---

## 5.9 经典问题：工具权限为什么不只靠 tool 自己判断？

因为权限是跨工具、跨 session、跨用户确认的统一策略。如果每个工具自己判断，会出现：

- 规则分散，不好审计
- 用户确认结果难以复用
- worker 继承 leader 权限困难
- UI 很难统一展示“为什么要确认”

AgentScope 用 `PermissionEngine + PermissionContext + Tool.check_permissions()` 组合：工具能表达自己的安全语义，engine 统一决策和状态。

---

## 5.10 本章检查题

1. `_execute_tool_call()` 里哪些错误会变成 `ToolResultBlock`？
2. `ASK` 和 `DENY` 的行为差别是什么？
3. external tool 为什么会进入 `SUBMITTED` 状态？
4. 为什么 `_acting_impl()` 不负责写 context？
5. 如果你写一个会修改数据库的工具，应该把 `is_concurrency_safe` 设成什么？为什么？
