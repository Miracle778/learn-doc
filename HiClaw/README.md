# HiClaw Agent 应用开发源码学习路线

> 这套文档是给“想系统学习 Agent 应用开发，并且希望对着 HiClaw 源码一点一点读”的同学准备的。
>
> 读法很简单：每章先建立一个心理模型，再打开对应源码文件，最后用几个经典问题检验自己是否真的看懂。

## 你将学到什么

HiClaw 不是一个单 Agent demo。它更像一个“Agent 团队操作系统”：

- Human、Manager、Worker 都在 Matrix 房间里说话。
- Manager 负责协调、派单、建 Worker、建 Team、检查进度。
- Worker 负责执行具体任务。
- Controller 负责把 Worker/Manager/Team 这些“期望状态”变成真实账号、房间、网关消费者、对象存储文件和容器。
- MinIO/OSS 承担共享文件系统和持久上下文。
- Higress 承担 LLM 网关、MCP 网关、消费者鉴权。

所以学习 HiClaw，不能只问“模型怎么调用工具”。更应该问：

- Agent 的身份、规则、记忆、工具是怎么被装进运行时的？
- 多 Agent 之间为什么不直接函数调用，而是通过 IM 房间和共享文件协作？
- 任务为什么必须写 `meta.json`、`spec.md`、`state.json`？
- Worker 为什么可以随时销毁重建？
- 上下文为什么不全塞进 prompt，而是拆到 Matrix 历史、记忆文件、任务文件和对象存储里？

## 章节安排

1. [第 0 章：先建立 HiClaw 的大脑地图](./hiclaw-ch0-architecture-map.md)
2. [第 1 章：Agent 是怎样启动起来的](./hiclaw-ch1-agent-runtime-bootstrap.md)
3. [第 2 章：记忆、上下文和“压缩”到底在哪里](./hiclaw-ch2-memory-context-compression.md)
4. [第 3 章：多 Agent 调度和任务协议](./hiclaw-ch3-multi-agent-scheduling.md)
5. [第 4 章：Worker 从创建到可执行的完整链路](./hiclaw-ch4-worker-provisioning-runtime-tools.md)
6. [第 5 章：经典问题源码剖析清单](./hiclaw-ch5-classic-questions.md)

## 推荐阅读顺序

第一轮不要追细节。先按章节把主线跑通：

```text
docs/architecture.md
  -> manager/agent/AGENTS.md
  -> manager/agent/skills/task-management/SKILL.md
  -> manager/agent/skills/task-management/references/finite-tasks.md
  -> hiclaw-controller/api/v1beta1/types.go
  -> hiclaw-controller/internal/service/provisioner.go
  -> hiclaw-controller/internal/service/deployer.go
  -> worker/scripts/worker-entrypoint.sh
```

第二轮再追专题：

- 记忆：`manager/agent/AGENTS.md`、`manager/agent/worker-agent/AGENTS.md`、`hiclaw-controller/internal/agentconfig/generator.go`
- 调度：`task-management`、`team-management`、`team-leader-agent/skills`
- 上下文：Matrix 消息格式、`shared/tasks/`、`state.json`、`/root/hiclaw-fs`
- 工具：`mcporter`、`mcp-server-management`、`agentconfig.GenerateMcporterConfig`
- 容器生命周期：`worker_controller.go`、`backend/interface.go`、`worker-entrypoint.sh`

## 外部知识对照

这些资料适合边读源码边参考：

- ReAct 论文：[Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
- AgentScope 多 Agent 框架论文：[AgentScope: A Flexible yet Robust Multi-Agent Platform](https://arxiv.org/abs/2402.14034)
- Kubernetes Operator 概念：[Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- Matrix 协议文档：[Matrix Client-Server API](https://spec.matrix.org/latest/client-server-api/)
- Model Context Protocol：[MCP Documentation](https://modelcontextprotocol.io/)

