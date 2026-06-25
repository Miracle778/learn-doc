# 第 0 章 先建立心理模型：AgentScope 2.0 到底在解决什么？

> 这一章不急着读代码。先把 AgentScope 放进一个“真实 agent 产品”的脑图里：用户消息进来后，框架如何让一个 LLM 变成可运行、可恢复、可审计、可协作的 agent。

---

## 0.1 先别把 Agent 理解成“聊天函数”

很多人刚开始写 agent，会下意识写成这样：

```python
answer = llm.chat(messages)
```

这当然能聊天，但它不是完整 agent。真正的 agent 应用至少要回答这些问题：

1. 历史消息存在哪里？下次请求怎么恢复？
2. 模型要调用工具时，工具 schema 从哪里来？
3. 工具执行前要不要权限确认？
4. 工具结果太长怎么办？
5. 对话越来越长时怎么压缩？
6. 用户偏好、长期事实怎么跨会话保存？
7. 多个 agent 怎么互相发消息、被唤醒、并发执行？
8. 前端怎么显示模型流式输出、工具调用、工具结果？
9. 服务重启后，session 状态还能不能继续？

AgentScope 2.0 的源码，就是围绕这些问题建立边界。

---

## 0.2 用一张图建立整体结构

```text
用户 / 前端
  │
  │ HTTP / SSE
  ▼
ChatService.run()
  │
  ├─ 从 Storage 加载 AgentRecord / SessionRecord
  ├─ 从 WorkspaceManager 取 workspace
  ├─ get_toolkit() 组装工具、MCP、skills、team tools、task tools
  ├─ 组装 middlewares：Inbox / StateChange / ToolOffload / TTS / extras
  ├─ 根据 session config 加载 model / fallback model
  ├─ 用持久化 AgentState 创建 Agent
  │
  ▼
Agent.reply_stream()
  │
  ├─ _handle_incoming_messages()：用户消息进入 state.context
  ├─ compress_context()：必要时压缩旧上下文
  ├─ _reasoning_impl()：准备 model input，调模型
  ├─ _execute_tool_call()：校验工具、权限、执行、写回结果
  ├─ ReAct loop：继续 reasoning / acting，直到完成或 max_iters
  │
  ▼
事件流 AgentEvent
  │
  ├─ MessageBus.session_publish_event()：replay log + live publish
  ├─ Msg.append_event()：服务端重建 assistant reply
  └─ Storage.update_session_state()：保存新的 AgentState
```

这张图有两个关键认知：

- `Agent` 是推理-行动循环的核心，但它不负责整个产品的存储、SSE、session lock、多租户。
- `ChatService` 是产品化运行入口，它负责把 agent、工具、模型、中间件、workspace、storage、message bus 组装成一次完整运行。

---

## 0.3 AgentScope 的边界划分

先记住这几个层次，后面读源码会轻松很多。

```text
应用服务层 app/
  ChatService / Storage / MessageBus / WorkspaceManager / Team tools

Agent 执行层 agent/
  Agent / ReAct loop / context compression / model call / tool execution

状态与消息层 state/ message/ event/
  AgentState / Msg / ContentBlock / AgentEvent

扩展层 middleware/
  Memory / Inbox / TTS / ToolOffload / Tracing / Budget

能力层 tool/ model/ workspace/ permission/
  ToolBase / Toolkit / ChatModelBase / PermissionEngine / WorkspaceBase
```

你读代码时不要一上来就陷入模型适配器。模型适配器很多，但主干其实很清晰：

- `Agent` 关心“怎么运行一轮 agent”。
- `AgentState` 关心“运行中要保存什么”。
- `Msg` 和 `AgentEvent` 关心“信息如何结构化流动”。
- `Middleware` 关心“不改 Agent 核心如何插能力”。
- `ChatService` 关心“生产服务如何可靠运行 agent”。

---

## 0.4 一次用户消息的完整时间线

假设用户在前端输入：“帮我读一下 README_zh.md，并总结 AgentScope 的核心能力”。

```text
T0 前端发送消息
  POST /chat 或 session run 入口，构造 UserMsg

T1 ChatService.run()
  加载 agent/session/workspace/model/toolkit/middlewares

T2 创建 Agent
  Agent(state=session_record.state, model=model, toolkit=toolkit, ...)

T3 agent.reply_stream(UserMsg(...))
  用户消息被 deepcopy 后 append 到 state.context

T4 发 ReplyStartEvent
  服务层开始创建空 AssistantMsg，用后续事件逐块拼起来

T5 reasoning 前 compress_context()
  如果 system + summary + context + tools token 超阈值，就压缩旧消息

T6 _prepare_model_input()
  messages = SystemMsg + summary(UserMsg) + state.context
  tools = toolkit.get_tool_schemas(...)

T7 调模型
  模型可能输出文本，也可能输出 ToolCallBlock(name="Read", input="...")

T8 执行工具
  找工具 → JSON 修复 → schema 校验 → 权限系统 → call_tool → ToolResultBlock

T9 工具结果写回 context
  模型下一轮能看到 Read 的文件内容

T10 再次 reasoning
  模型根据工具结果总结

T11 结束
  没有新 tool_call，产生 final AssistantMsg，发 ReplyEndEvent

T12 持久化
  ChatService 保存 reply Msg 和新的 AgentState
```

这就是后面所有章节要拆的主线。

---

## 0.5 和 LangGraph 类框架的区别

如果你之前看过 LangGraph，会习惯把 agent 理解成一张状态图。AgentScope 2.0 不是以显式 graph 作为主抽象，而是以一个统一 `Agent` 类组织 ReAct loop，再通过 middleware 和服务层扩展。

两种思路都能做 agent：

```text
Graph-first 框架：节点和边是主角
AgentScope：Agent loop + structured events + state + middleware 是主角
```

AgentScope 的好处是：你可以非常直接地跟着 `_reply_impl()` 看完整执行路径，代码阅读门槛低；同时服务层又补齐了多 session、team、message bus、workspace、权限等生产问题。

---

## 0.6 本文档的阅读路线

建议顺序：

1. 第 1 章：Agent 如何装配
2. 第 2 章：消息和 block
3. 第 3 章：AgentState
4. 第 4 章：ReAct 主循环
5. 第 5 章：工具和权限
6. 第 6 章：上下文压缩
7. 第 7 章：长期记忆
8. 第 8 章：middleware
9. 第 9 章：多 agent team
10. 第 11 章：服务化运行

第 10、12、13、15 章更像复习和练习，可以穿插看。

---

## 0.7 本章检查题

1. 为什么不能把 agent 简化成一次 `llm.chat(messages)`？
2. `Agent` 和 `ChatService` 分别负责什么？
3. 哪些信息属于 session 短期状态？哪些信息属于长期记忆？
4. 为什么 AgentScope 要设计事件流，而不只是返回最终字符串？
