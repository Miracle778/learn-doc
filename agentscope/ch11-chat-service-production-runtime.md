# 第 11 章 服务化运行：ChatService 如何把 Agent 变成生产服务？

> 这一章读 `ChatService.run()`。目标是理解：生产里的 agent 不是直接 `agent.reply()`，而是要加载状态、装配依赖、加 session lock、发布事件、保存结果。

---

## 11.1 配套源码

打开：

- `src/agentscope/app/_service/_chat.py`
- `src/agentscope/app/_service/_toolkit.py`
- `src/agentscope/app/message_bus/_base.py`
- `src/agentscope/app/storage/_base.py`
- `src/agentscope/app/workspace_manager/_base.py`
- `src/agentscope/app/_router/_chat.py`
- `examples/agent_service/main.py`

---

## 11.2 ChatService 的职责

`ChatService` 是 HTTP chat endpoint 和 wakeup dispatcher 共用的运行入口。

它的职责不是“写业务 prompt”，而是保证每次 agent run 都：

- 加载正确的 agent/session/workspace/model
- 装配完整工具和 middleware
- 在分布式 session lock 内运行
- 把事件发布给前端
- 重建并保存 assistant message
- 保存更新后的 AgentState
- 捕获异常，避免后台触发器崩掉

---

## 11.3 `run()` 和 `_run_impl()` 的分工

`run()` 是外层保护：

```python
try:
    await self._run_impl(...)
except Exception:
    logger.exception(...)
```

为什么吞掉异常？因为 `ChatService.run()` 可能由 HTTP 请求、wakeup dispatcher、后台任务完成、定时器触发。某次 run 失败不应该把整个 dispatcher 或服务进程打崩。

真正逻辑在 `_run_impl()`。

---

## 11.4 第一步：加载记录和 workspace

`_run_impl()` 先加载：

```text
agent_record = storage.get_agent(user_id, agent_id)
session_record = storage.get_session(user_id, agent_id, session_id)
workspace = workspace_manager.get_workspace(...)
```

如果缺 agent/session，直接 404。

然后把 workspace workdir 加入 `session_record.state.permission_context.working_directories`。

这一步很关键：工具权限和工作目录不是全局固定的，而是每个 session 的 workspace 注入到 permission context。

---

## 11.5 第二步：组装 Toolkit

调用：

```python
toolkit = await get_toolkit(...)
```

`get_toolkit()` 聚合：

1. workspace built-in tools：Bash / Read / Write / Edit / Grep / Glob 等
2. Task tools：TaskCreate/List/Get/Update
3. Background task control：ToolStop
4. Schedule tools：如果 session 有 model config
5. Team tools：leader/worker 不同
6. extra tools：用户自定义 factory
7. workspace skills 和 MCPs

所以 Agent 看到的工具不是硬编码在 Agent 类里，而是每次 run 根据 session/workspace/app 配置动态组装。

---

## 11.6 第三步：组装 Middlewares

框架默认 middlewares：

```text
InboxMiddleware
StateChangeMiddleware
ToolOffloadMiddleware
TTSMiddleware（如果 session 配了 TTS）
extra_agent_middlewares（用户扩展）
```

顺序很重要：framework-supplied first，然后 extras。

`InboxMiddleware` 让 team message / background result 能在 reasoning 前进入 context。

`StateChangeMiddleware` 用于把状态变化通知出去。

`ToolOffloadMiddleware` 让长时间工具可以后台运行并通过 inbox/wakeup 回传结果。

---

## 11.7 第四步：加载模型和 fallback

从 `session_record.config.chat_model_config` 加载主模型。

如果有 fallback config，也加载 fallback model，然后传给：

```python
ModelConfig(fallback_model=fallback_model)
```

`Agent._call_model()` 会基于 `model_config` 处理主模型失败后的 fallback。

---

## 11.8 第五步：创建 Agent

核心代码形态：

```python
agent_state = session_record.state
agent_state.session_id = session_id
agent = Agent(
    name=agent_record.data.name,
    system_prompt=agent_record.data.system_prompt,
    model=model,
    toolkit=toolkit,
    model_config=ModelConfig(fallback_model=fallback_model),
    context_config=agent_record.data.context_config,
    react_config=agent_record.data.react_config,
    state=agent_state,
    middlewares=middlewares,
    offloader=workspace,
)
```

注意：

- context/react config 来自 `AgentRecord.data`，说明 agent 定义控制行为策略。
- state 来自 `SessionRecord.state`，说明每个 session 有自己的运行历史。
- offloader 是 workspace，说明超长内容可以落到 workspace。

---

## 11.9 第六步：wakeup guard

如果 `input_msg is None`，说明这是 wakeup run。此时如果 agent 正在等待工具确认或外部执行结果，就跳过。

为什么？因为 wakeup 只是为了处理 inbox；如果 agent 当前卡在工具确认，必须等确认事件，而不能用 `None` 继续。

这保护了 agent 状态机不被 team message 或后台唤醒打断。

---

## 11.10 第七步：session lock 内运行

核心：

```python
async with self._message_bus.session_run(session_id):
    async for event in agent.reply_stream(...):
        await session_publish_event(session_id, event)
        reply_msg.append_event(event)

    storage.upsert_message(reply_msg)
    storage.update_session_state(agent.state)
```

为什么必须在 lock 内保存 state？源码注释说得很清楚：如果先释放 lock，再保存 state，另一个进程可能拿到 lock 并读取旧 state，造成状态覆盖或丢失。

---

## 11.11 事件发布：replay log + live pub/sub

`MessageBus.session_publish_event()` 做：

```text
log_append(session_events_key, event)
publish(session_events_key, event + _entry_id)
```

为什么两份？

- replay log：客户端晚连接还能补当前 run 的事件
- live publish：在线客户端实时显示

run 结束后，`session_run.__aexit__()` 会 trim replay log，因为完整 message 已经持久化到 storage。

---

## 11.12 新消息和 continuation 的保存差异

Case A：新 user message / input_msg=None

- 如果有 user msg，先保存 user msg 到 storage
- agent 产生事件
- 根据 ReplyStartEvent 创建空 assistant Msg
- append_event 重建 reply
- 保存 reply 和 state

Case B：UserConfirmResultEvent / ExternalExecutionResultEvent

- 从 storage 取已有 reply msg
- 先把 continuation event append 到 reply msg
- agent 继续产生事件
- append_event 到同一个 reply msg
- 保存更新后的 reply 和 state

这保证工具确认前后的内容属于同一个 assistant reply。

---

## 11.13 本章检查题

1. 为什么 ChatService 要同时被 HTTP endpoint 和 WakeupDispatcher 复用？
2. AgentRecord 和 SessionRecord 分别承担什么？
3. 为什么 workspace workdir 要加入 permission_context？
4. session lock 保护了哪些写操作？
5. 为什么 replay log 在 run 结束后可以 trim？
6. continuation event 为什么要更新已有 reply message？
