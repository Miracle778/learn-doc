# 第 4 章 ReAct 主循环：Agent 是如何“想、做、再想”的？

> 这一章是整套文档的核心。请打开 `src/agentscope/agent/_agent.py`，沿着 `_reply_impl()` 一路读到 `_reasoning_impl()`、`_batch_tool_calls()`、`_execute_tool_call()`。

---

## 4.1 先看主干方法列表

`Agent` 的运行主线：

```text
reply_stream()
  └─ _reply()
      └─ _reply_impl()
          ├─ _check_incoming_event()
          ├─ _handle_incoming_event() 或 _handle_incoming_messages()
          ├─ compress_context()
          ├─ _reasoning()
          │   └─ _reasoning_impl()
          ├─ _batch_tool_calls()
          ├─ _execute_sequential_tool_calls()
          ├─ _execute_concurrent_tool_calls()
          └─ _execute_tool_call()
```

`reply_stream()` 对外只暴露事件流；真正逻辑在 `_reply_impl()`。

---

## 4.2 `_reply_impl()` 第一步：区分新输入和 continuation

`inputs` 可能是：

- `Msg` / `list[Msg]`：用户新消息
- `UserConfirmResultEvent`：用户确认工具调用
- `ExternalExecutionResultEvent`：外部工具执行结果
- `None`：无新用户输入，继续当前状态；常用于 wakeup run

源码先把它分成两类：

```text
event = UserConfirmResultEvent / ExternalExecutionResultEvent / None
msgs  = Msg / list[Msg] / None
```

然后调用 `_check_incoming_event(event)` 判断当前 agent 是否正在等事件。

### 为什么要这么复杂？

因为 agent 可能在上一轮暂停了：

```text
模型请求 Bash
  → 权限系统要求用户确认
  → Agent 发 RequireUserConfirmEvent
  → 本轮 reply 暂停
  → 用户确认后下一次调用继续同一个 reply
```

如果 agent 正在等确认，而你又传了 `None` 或新 user message，状态就会乱。所以 `_check_incoming_event()` 会严格校验。

---

## 4.3 新消息路径：写入 context，开始 reply

如果不是 continuation，就走：

```python
await self._handle_incoming_messages(msgs)
self.state.reply_id = _generate_id()
self.state.cur_iter = 0
yield ReplyStartEvent(...)
```

`_handle_incoming_messages()` 会做输入合法性检查：

- 必须是 `Msg`
- 不能是 system role
- 不能带 tool_call/tool_result/thinking

然后 append 到 `state.context`。

这就是用户消息进入短期上下文的位置。

---

## 4.4 ReAct loop 的主循环

核心结构：

```text
while cur_iter < max_iters:
    action, data = _check_next_action()

    if action == "exit":
        yield final Msg
        return

    if action == "reasoning":
        await compress_context()
        async for evt in _reasoning():
            if evt is final Msg:
                yield ReplyEndEvent
                yield evt
                return
            yield evt

    for batch in await _batch_tool_calls():
        execute batch
        if requires outside interaction:
            yield waiting AssistantMsg
            return

    cur_iter += 1

# 超过 max_iters
Yield ExceedMaxItersEvent + ReplyEndEvent + fallback AssistantMsg
```

你可以把它理解成一个状态机：

```text
检查是否有未执行工具
  ├─ 没有：调模型 reasoning
  └─ 有：执行工具 acting

reasoning 后：
  ├─ 只有文本：结束
  └─ 有 tool_call：下一轮执行工具

acting 后：
  ├─ 工具结果写回 context
  └─ 下一轮 reasoning
```

---

## 4.5 `_reasoning_impl()`：模型调用如何变成事件和消息？

`_reasoning_impl()` 做几件事：

1. 发 `ModelCallStartEvent`
2. 调 `_prepare_model_input()` 准备 messages + tools
3. 调 `_call_model()`
4. 把模型流式返回转换成各种 block events
5. 发 block end events
6. 发 `ModelCallEndEvent`
7. 把完整模型输出保存进 `state.context`
8. 如果没有 tool call，产出 final `AssistantMsg`

关键点：模型响应既要发给前端，又要写进上下文。

```text
ChatResponse chunk
  → _convert_chat_response_to_event() 给前端看
  → completed_response.content 保存到 context
```

---

## 4.6 `_check_next_action()` 的直觉

虽然这份文档不逐行贴 `_check_next_action()`，但你要理解它的角色：它检查上下文最后的 assistant message 里是否还有没处理完的 tool call。

如果：

- 工具调用还在 `pending/allowed/submitted/asking` 某些状态，可能要执行或等待。
- 没有可执行工具调用，就该继续 reasoning。
- 已经有最终 assistant message，可能 exit。

这让 Agent 可以在多次 HTTP 调用之间恢复工具生命周期。

---

## 4.7 工具 batch：为什么不是按 tool_call 顺序一个个执行？

模型一次可能输出多个 tool call。AgentScope 根据工具属性分批：

- 并发安全工具：可以 concurrent
- 不安全工具：sequential

例如：

```text
Grep README + Glob *.py       可以并发
Write file + Edit file        应该串行
add_memory                    不适合和其他写 memory 并发
```

`_execute_concurrent_tool_calls()` 用 `asyncio.gather(return_exceptions=True)`，并且通过 queue 把各个工具的事件流汇总出来。

这个设计避免一个工具失败就取消其他工具，同时还能完整产出每个工具的事件。

---

## 4.8 外部交互：为什么会暂停 reply？

两类事件会让执行暂停：

- `RequireUserConfirmEvent`：需要用户确认工具调用
- `RequireExternalExecutionEvent`：工具要外部系统执行

一旦遇到，Agent 会返回一个 AssistantMsg：

```text
Waiting for tool calls to be confirmed or executed from outside ...
```

这不是最终回答，而是告诉服务层：当前 reply 暂停，等下一次事件继续。

这就是 human-in-the-loop 和 external tool execution 的基础。

---

## 4.9 超过 max_iters 怎么办？

如果循环次数超过 `react_config.max_iters`：

1. 发 `ExceedMaxItersEvent`
2. 记录 warning
3. 发 `ReplyEndEvent`
4. 返回一个 assistant fallback message

为什么还要发 `ReplyEndEvent`？因为前端/SSE 客户端通常等 terminal event。如果没有 `ReplyEndEvent`，UI 可能一直显示“运行中”。

---

## 4.10 本章检查题

1. 新 user message 是在哪一步写入 `state.context` 的？
2. continuation event 是如何恢复等待中的工具调用的？
3. reasoning 前为什么先调用 `compress_context()`？
4. 为什么模型没有 tool call 时，`_reasoning_impl()` 会直接 yield final `AssistantMsg`？
5. 并发工具调用失败时，为什么不取消其他工具？
