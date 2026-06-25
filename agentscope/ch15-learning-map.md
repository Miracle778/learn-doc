# 第 15 章 最后给你一张学习地图

> 这一章把整套源码学习压成一张地图。你可以把它当成复习清单，也可以当成之后读其他 agent 框架的对照表。

---

## 15.1 第一层：能跑起来

关键词：`Agent`、`ChatModelBase`、`Toolkit`、`UserMsg`

你应该能做到：

- 创建一个 agent
- 给它一个模型
- 注册几个工具
- 调用 `reply_stream()`
- 打印事件流

对应源码：

- `README_zh.md`
- `src/agentscope/agent/_agent.py`
- `src/agentscope/model/_base.py`
- `src/agentscope/tool/_toolkit.py`

---

## 15.2 第二层：懂消息结构

关键词：`Msg`、`TextBlock`、`ToolCallBlock`、`ToolResultBlock`、`HintBlock`、`AgentEvent`

你应该能解释：

- 为什么 message 不是 string
- tool call 如何流式累积
- tool result 如何回到上下文
- 前端事件如何重建 reply

对应源码：

- `src/agentscope/message/_base.py`
- `src/agentscope/message/_block.py`
- `src/agentscope/event/_event.py`

---

## 15.3 第三层：懂状态

关键词：`AgentState`、`summary`、`context`、`permission_context`、`tool_context`、`tasks_context`

你应该能解释：

- session 恢复靠什么
- context 和 summary 如何一起送给模型
- 权限为什么是状态
- Read 缓存为什么在 ToolContext

对应源码：

- `src/agentscope/state/_state.py`
- `src/agentscope/app/_service/_chat.py`

---

## 15.4 第四层：懂 ReAct loop

关键词：`_reply_impl`、`_reasoning_impl`、`_execute_tool_call`、`max_iters`

你应该能从任意一条用户消息追踪到：

```text
输入 → context → model input → tool call → permission → tool result → final answer
```

对应源码：

- `src/agentscope/agent/_agent.py`

---

## 15.5 第五层：懂上下文续航

关键词：`ContextConfig`、`SummarySchema`、`compress_context`、`offloader`

你应该能解释：

- 什么时候压缩
- 压缩哪些，保留哪些
- summary 长什么样
- 工具结果太长怎么处理

对应源码：

- `src/agentscope/agent/_config.py`
- `src/agentscope/agent/_agent.py`

---

## 15.6 第六层：懂长期记忆

关键词：`Mem0Middleware`、`static_control`、`agent_control`、`search_memory`、`add_memory`

你应该能解释：

- 长期记忆和上下文压缩的区别
- 自动检索如何注入 context
- memory tools 如何让模型主动控制记忆
- user_id/agent_id 如何隔离 memory

对应源码：

- `src/agentscope/middleware/_longterm_memory/_mem0/`

---

## 15.7 第七层：懂多 agent 协作

关键词：`TeamCreate`、`AgentCreate`、`TeamSay`、`InboxMiddleware`、`WakeupDispatcher`

你应该能画出：

```text
leader 创建 team
leader 创建 worker
worker inbox 收任务
wakeup dispatcher 启动 worker run
worker TeamSay 汇报 leader
leader inbox 收结果
leader 综合回答用户
```

对应源码：

- `src/agentscope/app/_tools/_team_*`
- `src/agentscope/app/middleware/_inbox_middleware.py`
- `src/agentscope/app/_manager/_wakeup_dispatcher.py`

---

## 15.8 第八层：懂生产化

关键词：`ChatService`、`StorageBase`、`MessageBus`、`session_run`、`session_publish_event`

你应该能解释：

- agent/session/model/workspace 如何装配
- 为什么同一 session 要加分布式锁
- event replay 和 live publish 的区别
- 为什么 state 保存必须在 lock 内

对应源码：

- `src/agentscope/app/_service/_chat.py`
- `src/agentscope/app/storage/_base.py`
- `src/agentscope/app/message_bus/_base.py`

---

## 15.9 一句话总结

AgentScope 2.0 的源码最值得学的不是某个 API，而是这些边界：

```text
结构化消息
  + 持久 AgentState
  + ReAct reasoning/acting loop
  + 工具 schema/权限/结果回写
  + summary/context 双层短期上下文
  + middleware 生命周期扩展
  + Mem0 长期记忆
  + MessageBus 驱动的多 agent session 协作
  + ChatService 生产化运行与持久化
```

这些边界学会了，再看 LangGraph、AutoGen、CrewAI 或自己写 agent runtime，都会更有底气。

---

## 15.10 最终自测

不看文档，试着回答：

1. 一条用户消息如何进入模型输入？
2. 一个 tool call 从生成到 finished 有哪些状态？
3. context compression 如何避免丢失最近细节？
4. Mem0Middleware 的三种 mode 分别适合什么？
5. leader 如何创建 worker 并让它自动开始任务？
6. ChatService 为什么需要 message bus？
7. 如果你要加一个新能力，什么时候写 tool，什么时候写 middleware，什么时候改 Agent 核心？

能答清楚这些，你就不是“会用 AgentScope”，而是真的开始理解 agent 应用框架了。
