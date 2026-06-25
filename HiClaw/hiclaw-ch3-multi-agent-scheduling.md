# 第 3 章：多 Agent 调度和任务协议

> 本章目标：搞懂 Manager 怎么把任务派给 Worker，Team Leader 又怎么进一步调度团队。重点不是“发一句话”，而是完整任务协议。

## 3.1 HiClaw 的调度不是函数调用

在很多教学 demo 里，多 Agent 调度可能长这样：

```python
planner.run()
coder.run(task)
reviewer.run(result)
```

HiClaw 不是这样。它的调度是一个跨系统协议：

```text
选择 Worker/Team
  -> 创建 shared/tasks/{task-id}/
  -> 写 meta.json 和 spec.md
  -> 推到 MinIO
  -> 写 state.json
  -> Matrix @mention 唤醒执行者
  -> Worker file-sync 拉取任务
  -> Worker 写 plan/result 并推回 MinIO
  -> Worker @mention Manager 完成
  -> Manager 拉结果、更新状态、通知 admin
```

这套设计牺牲了一点直接性，但换来了：

- 人类可见。
- 过程可追踪。
- Worker 可重启。
- 大上下文可用文件承载。
- 任务状态可被 heartbeat 监督。

## 3.2 入口：task-management skill

打开：

```text
manager/agent/skills/task-management/SKILL.md
```

这个 skill 的 description 说明了触发场景：

```text
Use when admin gives a task to delegate to a Worker,
when a Worker reports task completion,
when managing recurring scheduled tasks,
or when you need to check worker availability.
```

重点看 gotchas：

- Delegation-first：优先派 Worker。
- 每个任务必须注册到 `state.json`。
- 必须先 push task files 到 MinIO，再通知 Worker。
- Worker 完成后不能只在聊天里说收到，必须拉结果、更新 meta/state、写 memory、通知 admin。

这些规则是 HiClaw 多 Agent 可靠性的核心。

## 3.3 Step 0：如何选 Worker

打开：

```text
manager/agent/skills/task-management/references/worker-selection.md
```

选人流程：

1. 先看是否匹配 Team。
2. 再找 idle standalone worker。
3. 如果只有 busy worker，建议等或创建新 Worker。
4. 如果没有 Worker，建议导入/创建。
5. 只有 admin 明说“你自己做”，Manager 才自己执行。

这里有一个很重要的产品决策：

> Manager 是协调者，不是万事亲自做的超级 Agent。

源码里对应的命令是：

```bash
bash /opt/hiclaw/agent/skills/task-management/scripts/find-worker.sh
```

它会综合 Worker 的角色、skills、active tasks、container status。

## 3.4 有限任务协议

打开：

```text
manager/agent/skills/task-management/references/finite-tasks.md
```

核心步骤：

### 1. 生成 task id

```text
task-YYYYMMDD-HHMMSS
```

这个 ID 用于目录、状态、消息引用。

### 2. 写任务目录

```text
shared/tasks/{task-id}/meta.json
shared/tasks/{task-id}/spec.md
```

`meta.json` 是 Manager 维护的状态；`spec.md` 是 Worker 要读的完整需求。

### 3. 先推 MinIO

文档里强调：

```text
Worker cannot file-sync until files are in MinIO.
```

所以顺序必须是：

```text
写文件 -> mc cp 推送 -> 验证成功 -> Matrix 通知
```

这正是分布式系统里的“先提交状态，再发通知”。

### 4. 用 Matrix @mention 唤醒 Worker

通知内容类似：

```text
@{worker}:{domain} New task [{task-id}]: {title}.
Use your file-sync skill to pull the spec: shared/tasks/{task-id}/spec.md.
@mention me when complete.
```

注意：Manager 在 admin DM 里说“我让 Alice 做”是不够的。Worker 看不到 admin DM。必须发到 Worker 房间。

### 5. 写 state.json

```bash
bash /opt/hiclaw/agent/skills/task-management/scripts/manage-state.sh \
  --action add-finite \
  --task-id {task-id} \
  --title "{title}" \
  --assigned-to {worker} \
  --room-id {room-id}
```

如果跳过这一步，heartbeat 和 idle timeout 机制就不知道 Worker 正在干活。

## 3.5 Worker 收到任务后做什么

打开：

```text
manager/agent/worker-agent/AGENTS.md
```

Worker 的任务执行步骤：

1. `hiclaw-sync` 拉任务。
2. 读 `shared/tasks/{task-id}/spec.md`。
3. 创建 `plan.md`。
4. 执行任务。
5. 写 `result.md`。
6. 推回 MinIO。
7. @mention coordinator 汇报完成。
8. 写 memory。

这说明 Worker 的输出不是“聊天回复”本身，而是对象存储里的 artifact。聊天只是控制信号。

## 3.6 完成任务时 Manager 怎么收尾

`finite-tasks.md` 里 On completion：

1. 从 MinIO 拉任务目录。
2. 更新 `meta.json`：status = completed。
3. 从 `state.json` 移除任务。
4. 写 `memory/YYYY-MM-DD.md`。
5. 通知 admin。

这一整套动作解释了 `AGENTS.md` 里的硬规则：

```text
Worker reports completion -> load task-management skill and execute full flow.
Do NOT just acknowledge in chat.
```

因为如果只回复“收到”，系统状态会残留：

- `state.json` 还认为任务活跃。
- heartbeat 还会继续追问。
- admin 找不到结果汇总。
- Worker 可能被错误判断为忙碌。

## 3.7 Heartbeat：调度系统的巡检器

打开：

```text
manager/agent/HEARTBEAT.md
```

Heartbeat 会：

- 读 `state.json`。
- 检查 finite task 进展。
- 检查 team-delegated task。
- 检查 infinite task 是否到期。
- 扫 active project。
- 做容量评估。
- 检查 Worker 容器生命周期。
- 向 admin 汇报异常。

它不是“闲聊保活”，而是调度系统的后台巡检。

关键思想：

```text
调度时写 state.json
heartbeat 时读 state.json
完成时清 state.json
```

## 3.8 Team 调度：Manager 不直接管团队 Worker

打开：

```text
manager/agent/skills/team-management/SKILL.md
manager/agent/skills/team-management/references/team-task-delegation.md
```

Team 是：

```text
1 Team Leader + N Workers
```

规则很明确：

- Manager 只把任务派给 Team Leader。
- Team Leader 再拆解任务，分配给 team workers。
- Manager 不直接 @mention team workers。
- heartbeat 对 team-delegated task 也只问 Leader。

为什么？

因为团队内部有自己的上下文、角色、依赖和任务图。如果 Manager 越过 Leader 直接调 worker，会破坏团队局部自治。

## 3.9 DAG 项目模式

打开：

```text
manager/agent/team-leader-agent/skills/project-management/references/dag-execution.md
```

Team Leader 可以把复杂任务规划成 DAG：

```json
{
  "projectId": "project-xxx",
  "tasks": [
    {
      "taskId": "project-xxx-01",
      "title": "API contract design",
      "assignedTo": "backend-worker",
      "dependsOn": []
    },
    {
      "taskId": "project-xxx-02",
      "title": "Frontend UI",
      "assignedTo": "frontend-worker",
      "dependsOn": ["project-xxx-01"]
    }
  ]
}
```

好 DAG 节点的标准：

- 一个节点一个 Worker owner。
- 产出明确。
- 依赖只在真正需要上游结果时添加。
- 可动态重规划。

这就是多 Agent 并行协作的基本功：不是“让大家一起努力”，而是把任务切成可独立验收的节点。

## 3.10 本章经典问题

### 问题 1：为什么必须先推 MinIO 再 @mention？

因为 Worker 被 @mention 后第一步就是 `hiclaw-sync`。如果文件还没推上去，Worker 会拉到空目录或旧 spec，任务就会跑偏。

源码对照：

```text
manager/agent/skills/task-management/references/finite-tasks.md
manager/agent/worker-agent/skills/file-sync/SKILL.md
```

### 问题 2：为什么每个任务都必须进 state.json？

因为 heartbeat、idle timeout、容量评估都依赖它。没有 state.json，系统只能靠聊天历史猜测谁在忙，这在多 Agent 场景里很不可靠。

源码对照：

```text
manager/agent/HEARTBEAT.md
manager/agent/skills/task-management/scripts/manage-state.sh
```

### 问题 3：为什么 Team Worker 默认不接受 Manager 指令？

因为 team worker 的 coordinator 是 Leader。`agentconfig.GenerateOpenClawConfig` 会根据 `TeamLeaderName` 把 allowlist 从 Manager/Admin 改成 Leader/Admin。这样团队内部链路更清楚。

源码对照：

```text
hiclaw-controller/internal/agentconfig/generator.go
```

### 问题 4：HiClaw 的多 Agent 调度和 ReAct 有什么关系？

ReAct 讲的是单个 LLM 交替进行 reasoning 和 acting。HiClaw 在系统层把 acting 扩展成了：

- 调用脚本。
- 写状态。
- 发 Matrix 消息。
- 推拉对象存储。
- 创建真实 Worker。
- 授权 MCP/LLM 路由。

所以可以把 HiClaw 理解为：把 ReAct 风格的 Agent 放进一个可部署、可审计、可多 Agent 协作的工程系统里。

