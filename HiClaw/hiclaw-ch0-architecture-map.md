# 第 0 章：先建立 HiClaw 的大脑地图

> 本章目标：你先不要急着看某个函数。先知道 HiClaw 这套 Agent 应用的“角色、房间、文件、控制器、网关”分别在干嘛。

## 0.1 一句话理解 HiClaw

HiClaw 是一个用 IM 协议组织多 Agent 协作的系统。

它没有把“多 Agent”做成一个进程里的多个 Python object，也没有让 Manager 直接调用 Worker 的函数。它选择了一个很工程化的模型：

```text
Human/Admin
  在 Matrix 房间里下达任务
        |
        v
Manager Agent
  读规则、选 Worker、写任务文件、发 @mention、追踪 state.json
        |
        v
Worker Agent / Team Leader
  拉取任务文件、执行、写结果、推回对象存储、在 Matrix 回报
        |
        v
Controller
  把 Worker/Team/Manager CR 或 CLI 请求落成账号、房间、网关消费者、配置文件、容器
```

这一点很关键。很多 Agent demo 的核心问题是“模型如何调用工具”；HiClaw 的核心问题更接近：

> 一个 Agent 团队如何被可靠地创建、授权、通信、持久化、恢复和监督？

## 0.2 先看官方架构图

打开：

```text
docs/architecture.md
```

这份文档已经把 HiClaw 分成几层：

- `hiclaw-controller`：Go 写的控制面，负责 CRD、REST API、Worker/Manager/Team/Human 生命周期。
- `manager`：Manager Agent 容器，里面有 OpenClaw 或 CoPaw 运行时、HiClaw CLI、Agent prompts 和 skills。
- `worker` / `copaw` / `hermes` / `openhuman`：不同 Worker runtime。
- `helm` / `install`：部署层。
- Matrix/Tuwunel：Agent 和 Human 的通信总线。
- MinIO/OSS：共享文件系统和持久化上下文。
- Higress：LLM、MCP、HTTP 暴露和消费者鉴权网关。

你可以把它想象成一家公司：

```text
Matrix Room = 会议室
MinIO/OSS   = 文件柜
Higress     = 工具和模型的门禁
Controller  = 行政/IT 部门，负责开账号、开权限、分配电脑
Manager     = 项目经理
Worker      = 执行同事
Team Leader = 小组负责人
Human       = 老板/需求方/监督者
```

## 0.3 HiClaw 的“源码分工”

学习时不要把目录当成平铺文件。你要按职责分层读：

### 第一层：Agent 会读什么

这些不是普通文档，而是运行时 prompt/protocol：

```text
manager/agent/AGENTS.md
manager/agent/HEARTBEAT.md
manager/agent/SOUL.md
manager/agent/skills/*/SKILL.md
manager/agent/worker-agent/AGENTS.md
manager/agent/team-leader-agent/AGENTS.md
```

重点理解：

- Agent 每次醒来要先读什么。
- 什么情况必须用哪个 skill。
- 任务怎么登记、怎么派发、怎么完成。
- 哪些行为会造成循环、丢消息、权限泄露。

这就是 HiClaw 的第一个设计特色：很多业务规则写在“Agent 自己会读的文件”里。

### 第二层：Agent 怎么变成可运行配置

看：

```text
manager/scripts/init/start-manager-agent.sh
manager/scripts/init/upgrade-builtins.sh
worker/scripts/worker-entrypoint.sh
hiclaw-controller/internal/agentconfig/generator.go
```

这些文件负责：

- 初始化 Manager workspace。
- 把内置 prompt/skills 合并到实际工作区。
- 生成 `openclaw.json`。
- 配 Matrix channel、模型网关、memory-core 插件、heartbeat。
- Worker 启动时从 MinIO 拉配置，再启动 OpenClaw gateway。

### 第三层：控制面如何创建和管理 Agent

看：

```text
hiclaw-controller/api/v1beta1/types.go
hiclaw-controller/internal/service/provisioner.go
hiclaw-controller/internal/service/deployer.go
hiclaw-controller/internal/backend/interface.go
hiclaw-controller/internal/controller/worker_controller.go
```

这些文件回答：

- `WorkerSpec` 里能声明什么？
- Matrix 用户、房间怎么创建？
- Gateway consumer 怎么创建和授权？
- 配置文件、skills、SOUL.md 怎么推到 OSS？
- Docker/Kubernetes backend 怎么统一抽象？

## 0.4 一条用户消息的生命线

假设 Human 在 Manager DM 里说：

```text
请创建一个前端 Worker，并让它实现登录页
```

大致会发生：

1. Matrix 插件唤醒 Manager。
2. Manager 读 `AGENTS.md`，知道要读 SOUL、memory、相关 skill。
3. Manager 进入 `worker-management`，创建 Worker。
4. Controller 收到 `hiclaw create worker` 请求，创建 Worker CR 或直接走 REST 后端。
5. Provisioner 创建 Matrix 用户、Worker 房间、MinIO 用户、Higress consumer。
6. Deployer 生成 `openclaw.json`、推 `SOUL.md`、`AGENTS.md`、skills 到 `agents/<worker>/`。
7. Backend 创建 Worker 容器。
8. Worker entrypoint 从 MinIO 拉配置，启动 OpenClaw。
9. Manager 创建任务目录 `shared/tasks/{task-id}/`，写 `meta.json` 和 `spec.md`。
10. Manager 把任务加入 `~/state.json`。
11. Manager 在 Worker 房间 @mention Worker。
12. Worker 拉取任务文件，执行，写 `result.md`，推回 MinIO。
13. Worker @mention Manager 完成。
14. Manager 拉结果、更新 `meta.json` 和 `state.json`，写 memory，通知 admin。

这条链路就是后面几章的主线。

## 0.5 本章经典问题

### 问题 1：HiClaw 为什么不用 HTTP 直接让 Manager 调 Worker？

因为 HiClaw 的目标不是单次 RPC，而是人可见、可介入、可审计的协作。Matrix Room 天然保存对话上下文，人和多个 Agent 都能看见同一条时间线。HTTP 更适合命令调用，但不适合人类随时插话、补充需求和观察过程。

源码对照：

```text
manager/agent/AGENTS.md
manager/agent/worker-agent/AGENTS.md
hiclaw-controller/internal/service/provisioner.go
```

### 问题 2：为什么 Worker 是“无状态”的？

准确说：Worker 容器本身尽量无状态，状态外置到 MinIO/OSS。Worker 可以被停掉、重建、切 runtime，只要 `agents/<name>/` 和 `shared/` 还在，身份、任务和技能就能恢复。

源码对照：

```text
worker/scripts/worker-entrypoint.sh
manager/agent/worker-agent/AGENTS.md
manager/agent/skills/worker-management/SKILL.md
```

### 问题 3：HiClaw 的 Agent 应用开发最值得学什么？

不是某个 prompt 写法，而是四个工程模式：

- 通信协议外置：Matrix 房间作为协作总线。
- 上下文外置：任务、记忆、共享文件进入对象存储。
- 能力外置：skills 和 MCP 作为可扩展能力。
- 生命周期外置：Controller 用期望状态驱动真实资源。

