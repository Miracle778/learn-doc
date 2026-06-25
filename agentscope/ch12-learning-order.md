# 第 12 章 建议学习顺序：从浅到深怎么读？

> 这一章是学习计划。它把前面内容拆成 8 个阶段，每个阶段都有目标、源码文件、必须回答的问题和小练习。

---

## 阶段 A：跑通最小 Agent

目标：知道 Agent 最小构成。

读：

- `README_zh.md` Hello Agent
- `src/agentscope/agent/_agent.py`：`__init__`、`reply_stream()`、`reply()`
- `src/agentscope/model/_base.py`
- `src/agentscope/tool/_base.py`

必须回答：

1. `Agent` 不传 `state` 时会发生什么？
2. `reply()` 和 `reply_stream()` 的返回有什么不同？
3. `toolkit` 为空时，agent 还能做什么？

练习：写一个只带模型、无工具的 agent，再写一个带 Read/Grep 的 agent，对比模型输入中的 tool schemas。

---

## 阶段 B：理解消息和事件

目标：知道前端流式输出和服务端持久化如何统一。

读：

- `src/agentscope/message/_base.py`
- `src/agentscope/message/_block.py`
- `src/agentscope/event/_event.py`

必须回答：

1. 为什么 `Msg.content` 是 block list？
2. `append_event()` 如何重建 assistant message？
3. tool call streaming 为什么需要 delta event？

练习：手动构造一个空 AssistantMsg，然后模拟 `TextBlockStart/Delta/End` 事件调用 `append_event()`。

---

## 阶段 C：理解 AgentState

目标：知道 agent 运行中哪些东西会持久化。

读：

- `src/agentscope/state/_state.py`
- `src/agentscope/state/_task.py`
- `src/agentscope/app/_service/_chat.py` 保存 state 的部分

必须回答：

1. `summary` 和 `context` 的关系是什么？
2. `permission_context` 为什么要保存在 state？
3. `tool_context.read_file_cache` 什么时候清理？

练习：画一张 AgentState 字段图，把每个字段标注“模型可见 / 工具可见 / 服务层可见”。

---

## 阶段 D：精读 ReAct loop

目标：能从用户输入追踪到最终回答。

读：

- `_reply_impl()`
- `_reasoning_impl()`
- `_batch_tool_calls()`
- `_execute_tool_call()`

必须回答：

1. 新消息路径和 continuation 路径如何分开？
2. `ReplyStartEvent` 在什么时候发？
3. 工具调用执行后为什么还要继续 reasoning？
4. max_iters 如何结束循环？

练习：给一段假想上下文，判断下一步 action 是 reasoning 还是 acting。

---

## 阶段 E：精读工具和权限

目标：理解工具生命周期和 human-in-the-loop。

读：

- `src/agentscope/tool/_base.py`
- `src/agentscope/tool/_toolkit.py`
- `src/agentscope/permission/_engine.py`
- `_execute_tool_call()`

必须回答：

1. 工具 input 是在哪里做 JSON repair 和 schema validation 的？
2. `ASK` 和 `DENY` 对 context 的影响有什么不同？
3. external tool 的结果如何回来？

练习：设计一个危险工具和一个只读工具，写出它们应该如何设置 `is_read_only`、`is_concurrency_safe`、权限行为。

---

## 阶段 F：精读上下文压缩

目标：知道长任务如何不断续航。

读：

- `ContextConfig`
- `SummarySchema`
- `compress_context()`
- `_split_context_for_compression()`
- `_split_tool_result_for_compression()`

必须回答：

1. 为什么 trigger_ratio 默认小于 0.9？
2. 压缩时为什么保留最近原文？
3. tool call/result 为什么不能拆散？
4. 工具结果截断和 context compression 有什么区别？

练习：假设 context 有 10 条消息，第 7 条是超长工具结果，描述压缩和工具结果截断分别会怎么处理。

---

## 阶段 G：精读 middleware 和长期记忆

目标：学会扩展 agent 行为。

读：

- `src/agentscope/middleware/_base.py`
- `src/agentscope/middleware/_longterm_memory/_mem0/_middleware.py`
- `src/agentscope/middleware/_longterm_memory/_mem0/_tools.py`

必须回答：

1. `on_reply` 适合做什么？
2. `on_reasoning` 和 `on_model_call` 差别是什么？
3. Mem0 static_control 为什么要等 ReplyStartEvent？
4. `add_memory` 为什么要 fallback 到 `infer=False`？

练习：设计一个“用户偏好提醒 middleware”：每轮 reply 前检索偏好，reasoning 前注入 HintBlock。

---

## 阶段 H：精读多 agent 和服务化

目标：理解生产级 agent 如何多 session 协作。

读：

- `src/agentscope/app/_service/_chat.py`
- `src/agentscope/app/_service/_toolkit.py`
- `src/agentscope/app/_tools/_agent_create.py`
- `src/agentscope/app/_tools/_team_say.py`
- `src/agentscope/app/middleware/_inbox_middleware.py`
- `src/agentscope/app/_manager/_wakeup_dispatcher.py`
- `src/agentscope/app/message_bus/_base.py`

必须回答：

1. worker 是如何被创建并唤醒的？
2. TeamSay 如何把消息送到目标 session？
3. ChatService 为什么要用 session lock？
4. replay log 和 live publish 分别服务什么场景？

练习：画一张 leader 创建两个 worker 并收集结果的时序图。
