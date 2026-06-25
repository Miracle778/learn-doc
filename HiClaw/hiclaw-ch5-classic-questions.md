# 第 5 章：经典问题源码剖析清单

> 本章目标：用问题反推源码。你可以把这一章当成复习题，也可以当成面试/自测清单。

## 问题 1：HiClaw 里的 Agent 到底由哪些东西组成？

先给答案：

```text
Agent = runtime + workspace + prompt files + skills + communication identity + model/tool credentials + persistent files
```

源码入口：

```text
manager/scripts/init/start-manager-agent.sh
worker/scripts/worker-entrypoint.sh
manager/agent/AGENTS.md
manager/agent/worker-agent/AGENTS.md
hiclaw-controller/internal/agentconfig/generator.go
```

你应该看懂：

- runtime 是 OpenClaw/CoPaw/Hermes/OpenHuman。
- workspace 里有 `SOUL.md`、`AGENTS.md`、`memory/`、`skills/`、`openclaw.json`。
- Matrix userId 和 accessToken 决定 Agent 以谁的身份说话。
- Gateway key 决定它能访问哪些模型和工具。

## 问题 2：Agent 的记忆怎么做？

源码入口：

```text
manager/agent/AGENTS.md
manager/agent/worker-agent/AGENTS.md
hiclaw-controller/internal/agentconfig/generator.go
worker/scripts/worker-entrypoint.sh
```

你应该看懂三层：

### 第一层：文件记忆

```text
memory/YYYY-MM-DD.md
MEMORY.md
```

Agent 每次醒来先读；重大事件写回。

### 第二层：结构化状态

```text
~/state.json
shared/tasks/{task-id}/meta.json
```

这不是“记忆”，但决定调度系统是否知道谁在忙。

### 第三层：OpenClaw 插件

```text
memory-core
memorySearch
session.resetByType
```

这在 `GenerateOpenClawConfig` 里生成。

## 问题 3：上下文怎么做？

源码入口：

```text
manager/agent/AGENTS.md
manager/agent/skills/task-management/references/finite-tasks.md
manager/agent/worker-agent/AGENTS.md
worker/scripts/worker-entrypoint.sh
```

你应该能画出：

```text
当前触发消息
  + Matrix history section
  + SOUL.md / AGENTS.md / SKILL.md
  + memory today/yesterday
  + state.json
  + task spec/result files
  + shared knowledge/files
```

其中 Matrix 消息负责触发，文件系统负责承载大上下文。

## 问题 4：上下文怎么压缩？

先说结论：HiClaw 没有把压缩做成一个单独的“上下文压缩服务”。它的压缩策略是外置化和分层读取。

源码入口：

```text
hiclaw-controller/internal/agentconfig/generator.go
manager/agent/AGENTS.md
manager/agent/HEARTBEAT.md
```

你应该看懂：

- `session.resetByType` 每天 reset DM/group 会话。
- `memory/YYYY-MM-DD.md` 保留最近关键事实。
- `MEMORY.md` 保留长期经验。
- `state.json` 只保留活跃任务索引。
- `spec.md/result.md` 把任务上下文从聊天中移出。

这是一种工程压缩：不是总是总结聊天，而是把上下文拆成可按需读取的文件。

## 问题 5：多 Agent 之间怎么调度？

源码入口：

```text
manager/agent/skills/task-management/SKILL.md
manager/agent/skills/task-management/references/worker-selection.md
manager/agent/skills/task-management/references/finite-tasks.md
manager/agent/skills/task-management/scripts/manage-state.sh
manager/agent/HEARTBEAT.md
```

你应该能复述：

1. Manager 选择 Worker 或 Team。
2. 创建任务目录。
3. 写 `meta.json` 和 `spec.md`。
4. 推 MinIO。
5. 发 Matrix @mention。
6. 写 `state.json`。
7. Worker 执行并写 `result.md`。
8. Manager 收尾。

## 问题 6：Team Leader 如何调度团队？

源码入口：

```text
manager/agent/skills/team-management/SKILL.md
manager/agent/skills/team-management/references/team-task-delegation.md
manager/agent/team-leader-agent/skills/project-management/references/dag-execution.md
plugins/teamharness/skills/team/task-delegation/SKILL.md
```

你应该看懂：

- Manager 只和 Leader 交互。
- Leader 再调 team workers。
- 复杂项目用 DAG 表达依赖。
- ready nodes 可以并行派发。
- Manager heartbeat 只追 Leader，不绕过团队边界。

## 问题 7：Worker 是怎么被创建出来的？

源码入口：

```text
manager/agent/skills/worker-management/SKILL.md
manager/agent/skills/worker-management/references/create-worker.md
hiclaw-controller/api/v1beta1/types.go
hiclaw-controller/internal/service/provisioner.go
hiclaw-controller/internal/service/deployer.go
hiclaw-controller/internal/backend/interface.go
```

你应该能说清：

```text
hiclaw create worker
  -> WorkerSpec
  -> ProvisionWorker
  -> DeployWorkerConfig
  -> backend.Create
  -> worker-entrypoint.sh
  -> openclaw gateway run
```

## 问题 8：权限怎么做？

源码入口：

```text
hiclaw-controller/internal/service/provisioner.go
hiclaw-controller/internal/gateway/*
hiclaw-controller/internal/oss/*
hiclaw-controller/internal/agentconfig/generator.go
```

你应该看懂：

- Matrix token 决定房间身份。
- Gateway key 决定 LLM/MCP 访问。
- MinIO policy 限制对象存储路径。
- Matrix allowlist 限制谁能唤醒 Worker。
- ChannelPolicy 可以扩展或收紧通信白名单。

## 问题 9：为什么 @mention 要用 full Matrix ID？

源码入口：

```text
manager/agent/AGENTS.md
manager/agent/worker-agent/AGENTS.md
hiclaw-controller/internal/agentconfig/generator.go
```

你应该看懂：

- group room 里 `requireMention: true`。
- `@alice` 不等于 `@alice:matrix-domain`。
- 不完整 mention 可能不会唤醒 Agent。
- 无意义 @mention 会造成循环。

## 问题 10：为什么 Worker 的任务结果不直接发聊天？

源码入口：

```text
manager/agent/skills/task-management/references/finite-tasks.md
manager/agent/worker-agent/AGENTS.md
worker/scripts/worker-entrypoint.sh
```

你应该看懂：

- 聊天适合通知，不适合承载完整 artifact。
- `result.md` 可持久化、可审计、可被 Manager 拉取总结。
- 大文件和中间产物应该留在任务目录。

## 问题 11：为什么 Worker 可以被自动暂停？

源码入口：

```text
manager/agent/HEARTBEAT.md
manager/agent/skills/worker-management/scripts/lifecycle-worker.sh
manager/agent/skills/task-management/scripts/manage-state.sh
```

你应该看懂：

- `state.json` 记录活跃任务。
- heartbeat 根据活跃任务判断是否应该追问或保活。
- idle Worker 可以暂停节省资源。
- 未登记任务会导致 Worker 被误判为空闲。

## 问题 12：HiClaw 和常见 Agent 框架的关系是什么？

你可以这样理解：

```text
ReAct / Agent runtime
  解决：单个 Agent 如何推理、调用工具、生成动作。

AgentScope / multi-agent framework 思路
  解决：多个 Agent 如何交互、组织、容错。

HiClaw
  解决：把多 Agent 协作做成可部署系统：
       IM 协作、对象存储、控制器、网关、权限、容器生命周期、技能分发。
```

外部参考：

- ReAct: https://arxiv.org/abs/2210.03629
- AgentScope: https://arxiv.org/abs/2402.14034
- Kubernetes Operator Pattern: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/
- Matrix Client-Server API: https://spec.matrix.org/latest/client-server-api/
- MCP Documentation: https://modelcontextprotocol.io/

## 最后一张复习图

```text
Admin message
  |
  v
Matrix wakes Manager
  |
  v
Manager reads SOUL + memory + skill
  |
  v
Manager writes task files to shared/tasks
  |
  v
Manager records state.json
  |
  v
Manager @mentions Worker/Leader
  |
  v
Worker pulls files with hiclaw-sync
  |
  v
Worker executes with model + tools + MCP
  |
  v
Worker writes result.md and pushes MinIO
  |
  v
Worker @mentions Manager
  |
  v
Manager updates meta/state/memory and reports Admin
```

如果你能不看文档把这张图从源码里解释出来，你就已经掌握了 HiClaw 的核心 Agent 应用开发模型。

