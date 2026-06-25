# 第 14 章 和官方资料 / GitHub 对照

> 这一章把源码学习路线和官方材料对应起来。官方文档用于校准概念，源码用于理解实现细节。

---

## 14.1 README 里的 AgentScope 2.0 定位

README 强调 AgentScope 2.0 是面向生产的智能体框架，核心能力包括：

- 事件系统：对应 `message/`、`event/`、`Msg.append_event()`、SSE replay/publish
- 权限系统：对应 `permission/`、`ToolCallBlock.state`、`RequireUserConfirmEvent`
- 多租户多会话服务：对应 `app/storage/`、`ChatService`、`SessionRecord`
- Workspace / sandbox：对应 `workspace/`、工作目录权限、tool offload
- Middleware：对应 `MiddlewareBase` 和各种 hook
- Agent Team：对应 `TeamCreate`、`AgentCreate`、`TeamSay`、`MessageBus`、`WakeupDispatcher`
- Mem0 长期记忆：对应 `middleware/_longterm_memory/_mem0/`

---

## 14.2 官方 Message/Event 文档怎么对源码

官方讲 message/event 时，你应该回到这些源码：

- `src/agentscope/message/_base.py`
- `src/agentscope/message/_block.py`
- `src/agentscope/event/_event.py`
- `src/agentscope/app/_service/_chat.py`

重点看：

```text
AgentEvent 流式产生
  → MessageBus 发布给前端
  → Msg.append_event 重建完整 assistant message
  → Storage 持久化 message
```

---

## 14.3 官方 Middleware 文档怎么对源码

官方讲 middleware hook 时，对应：

- `src/agentscope/middleware/_base.py`
- `src/agentscope/agent/_agent.py` 中 `_reply()`、`_reasoning()`、`_acting()`、`_call_model()`、`compress_context()`

学习重点：

- 哪些 hook 是 async generator
- 哪些 hook 是 awaitable 返回
- `on_system_prompt` 为什么是 transformer pipeline
- middleware 如何提供 tools

---

## 14.4 官方 Agent Team 文档怎么对源码

官方文档会讲“leader 创建 worker，worker 汇报”。源码里要看：

```text
TeamCreate    建 team
AgentCreate   建 worker agent/session，投递初始任务，唤醒
TeamSay       路由 team message 到 inbox，唤醒目标 session
InboxMiddleware  drain inbox，把 HintBlock 注入 context
WakeupDispatcher  收到 wakeup 后运行 ChatService.run(None)
```

这套实现的关键词是：session 间消息协作。

---

## 14.5 官方 Context / Memory 概念怎么对源码

短期上下文：

- `AgentState.context`
- `AgentState.summary`
- `ContextConfig`
- `compress_context()`

长期记忆：

- `Mem0Middleware`
- `search_memory` / `add_memory`
- mem0 filters：`user_id` / `agent_id`

要特别注意：官方文档里“memory”可能泛指记忆能力，但源码层面要分清 context compression 和 long-term memory。

---

## 14.6 参考链接

- AgentScope GitHub: https://github.com/agentscope-ai/agentscope
- AgentScope Docs: https://docs.agentscope.io/
- Message and Event: https://docs.agentscope.io/v2/building-blocks/message-and-event
- Middleware: https://docs.agentscope.io/v2/building-blocks/middleware
- Agent Team: https://docs.agentscope.io/zh/v2/deploy/agent-team
- AgentScope paper: https://arxiv.org/abs/2402.14034

---

## 14.7 使用官方文档的正确姿势

建议顺序：

1. 先读官方文档建立概念词汇。
2. 回到本文档对应章节读源码流程。
3. 再回官方文档看 API 使用方式。
4. 最后自己写一个最小 demo 验证。

不要只读官方 API 示例。Agent 应用开发真正难的是状态边界和生命周期，而这些必须对照源码才能学扎实。
