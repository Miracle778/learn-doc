# 第 13 章 经典剖析题：带着问题回源码

> 这一章用面试题/自测题的方式，把前面章节串起来。每个问题都给出源码路径和回答框架。

---

## 问题 1：用户发一条消息，到模型输入前经历了什么？

源码路径：

- `ChatService._run_impl()`
- `Agent._reply_impl()`
- `Agent._handle_incoming_messages()`
- `Agent.compress_context()`
- `Agent._prepare_model_input()`

答案框架：

```text
ChatService 加载 agent/session/workspace/model/toolkit/middleware
  → 创建 Agent(state=session_record.state)
  → agent.reply_stream(UserMsg)
  → _handle_incoming_messages 校验并写入 state.context
  → 设置 reply_id 和 cur_iter
  → ReplyStartEvent
  → compress_context 判断是否压缩
  → _prepare_model_input 生成 SystemMsg + summary + context + tools
```

关键点：模型输入不是用户消息本身，而是 system prompt、summary、context、tools 的组合。

---

## 问题 2：模型调用工具时，AgentScope 如何保证安全？

源码路径：

- `Agent._execute_tool_call()`
- `Toolkit.check_tool_available()`
- `_json_loads_with_repair()`
- `PermissionEngine.check_permission()`
- `RequireUserConfirmEvent`

答案框架：

```text
ToolCallBlock
  → 工具存在性检查
  → JSON repair
  → schema validation
  → permission decision
      DENY: 写 denied tool result
      ASK: 发 confirmation event 并暂停
      ALLOW: 执行
      external: 发 external execution event 并暂停
  → tool result 写回 context
```

关键点：失败、拒绝、等待确认都进入同一套工具生命周期。

---

## 问题 3：上下文太长时如何压缩？

源码路径：

- `ContextConfig`
- `Agent._compress_context_impl()`
- `Agent._split_context_for_compression()`
- `SummarySchema`

答案框架：

```text
count tokens(system + summary + context + tools)
  → 超过 trigger_ratio * context_size
  → split context：旧消息压缩，最近消息保留
  → 构造 compression messages
  → generate_structured_output(summary_schema)
  → summary_template 渲染 state.summary
  → state.context = msgs_to_reserve
```

关键点：不是截断，而是结构化摘要和近期原文结合。

---

## 问题 4：长期记忆如何进入当前对话？

源码路径：

- `Mem0Middleware.on_reply()`
- `_extract_query_text()`
- `_async_search()`
- `_build_memory_message()`

答案框架：

```text
on_reply 提取输入 query
  → mem0.search(filters=user_id/agent_id)
  → next_handler 进入 Agent._reply_impl
  → 等 ReplyStartEvent
  → append AssistantMsg(name="memory", HintBlock(...)) 到 state.context
  → agent reasoning 时看到 memory hint
  → reply 后把 user/assistant 对话写回 mem0
```

关键点：长期记忆检索结果进入 context 后，也会被短期上下文机制管理。

---

## 问题 5：Team worker 创建后为什么会自己开始工作？

源码路径：

- `TeamCreate.__call__()`
- `AgentCreate.__call__()`
- `MessageBus.enqueue_wakeup()`
- `WakeupDispatcher._drain_and_dispatch()`
- `InboxMiddleware.on_reasoning()`

答案框架：

```text
leader 调 TeamCreate 创建 team
  → leader 调 AgentCreate 创建 worker AgentRecord + SessionRecord
  → 初始任务封装成 HintBlock 写入 worker inbox
  → enqueue_wakeup
  → WakeupDispatcher 收到 wakeup，spawn ChatService.run(input_msg=None)
  → InboxMiddleware drain worker inbox
  → HintBlock 注入 context
  → worker reasoning 并执行任务
```

关键点：worker 是独立 session，被消息总线唤醒，不是 leader 函数调用。

---

## 问题 6：为什么 ChatService 要在 session lock 里保存 state？

源码路径：

- `ChatService._run_impl()`
- `MessageBus.session_run()`

答案框架：

```text
async with message_bus.session_run(session_id):
    run agent
    publish events
    upsert reply message
    update session state
```

如果先释放 lock 再保存 state，另一个进程可能立刻获得 lock，读到旧 state，然后两个 run 的更新互相覆盖。

关键点：锁保护的不只是 agent run，也保护 run 结束时的状态落盘。

---

## 问题 7：为什么 TeamSay 用 HintBlock，而不是 UserMsg？

源码路径：

- `TeamSay.__call__()`
- `InboxMiddleware.on_reasoning()`
- `message/_block.py` 的 `HintBlock`

答案框架：

Team message 是系统内部另一个 agent 的协作消息，不是最终用户输入。它需要保留 sender/source，也需要前端能渲染为 team hint，所以用 `HintBlock`。

`InboxMiddleware` 在 reasoning 前把 HintBlock 注入 context，让模型看到。

---

## 问题 8：为什么工具结果和工具调用不能在压缩时拆散？

源码路径：

- `Agent._split_context_for_compression()`

答案框架：

模型需要看到工具调用的因果关系：

```text
ToolCallBlock(id=abc, name=Read, input=...)
ToolResultBlock(id=abc, output=...)
```

如果只保留 result，不保留 call，模型不知道结果来自哪里；如果只保留 call，不保留 result，模型不知道观察结果。压缩边界要尽量维护这对关系。

---

## 问题 9：AgentScope 为什么既有 Task tools，又有 Team tools？

源码路径：

- `tool/_task/*`
- `app/_tools/_team_*`
- `state/_state.py`
- `app/storage/_model/_team.py`

答案框架：

Task tools 解决单 agent 内部计划管理；Team tools 解决多 agent/session 异步协作。

```text
Task: AgentState.tasks_context
Team: TeamRecord + worker AgentRecord/SessionRecord + MessageBus
```

---

## 问题 10：如果我要加一个“预算控制”能力，应该放哪里？

推荐：middleware。

可能 hook：

- `on_model_call`：统计/限制模型调用 token 或费用
- `on_reasoning`：在 reasoning 前检查预算，必要时注入提醒或中断
- `middle_context`：保存累计预算状态

不建议直接改 `_reply_impl()`，除非预算控制已经变成 Agent 核心语义。
