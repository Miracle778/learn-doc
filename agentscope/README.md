# AgentScope 2.0 源码学习文档目录（详细版）

> 按章节拆分版。每章都按“源码地图 → 调用链 → 关键设计 → 经典问题 → 阅读任务”的方式写，适合对照源码逐章学习。

## 章节列表

- [第 0 章：先建立心理模型：AgentScope 2.0 到底在解决什么？](./ch00-mental-model.md)
- [第 1 章：从 Hello Agent 开始：一个 Agent 是怎么装起来的？](./ch01-agent-assembly.md)
- [第 2 章：消息系统：为什么要把 message 拆成 block？](./ch02-message-and-blocks.md)
- [第 3 章：AgentState：上下文、摘要、工具缓存和任务列表都放在哪里？](./ch03-agent-state-and-context.md)
- [第 4 章：ReAct 主循环：Agent 是如何“想、做、再想”的？](./ch04-react-loop.md)
- [第 5 章：工具系统：模型调用工具之前，框架做了哪些保护？](./ch05-tools-and-permissions.md)
- [第 6 章：上下文压缩：AgentScope 如何处理长对话？](./ch06-context-compression.md)
- [第 7 章：长期记忆：AgentScope 的 Mem0Middleware 怎么做？](./ch07-long-term-memory-mem0.md)
- [第 8 章：Middleware：如何不改 Agent 核心就扩展能力？](./ch08-middleware.md)
- [第 9 章：多 Agent 调度：Team 不是函数调用，而是 session 间消息协作](./ch09-multi-agent-team-dispatch.md)
- [第 10 章：任务规划工具：单 agent 内部如何管理 todo？](./ch10-task-planning-tools.md)
- [第 11 章：服务化运行：ChatService 如何把 Agent 变成生产服务？](./ch11-chat-service-production-runtime.md)
- [第 12 章：建议学习顺序：从浅到深怎么读？](./ch12-learning-order.md)
- [第 13 章：经典剖析题：带着问题回源码](./ch13-classic-source-questions.md)
- [第 14 章：和官方资料 / GitHub 对照](./ch14-official-docs-and-github.md)
- [第 15 章：最后给你一张学习地图](./ch15-learning-map.md)
- [第 16 章：安全模型：AgentScope 的安全边界在哪里？](./ch16-security-model.md)

## 合订版

- [AgentScope 2.0 源码学习路线（详细合订版）](./agentscope-source-learning-walkthrough.md)
