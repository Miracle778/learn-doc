# 第 2 章：记忆、上下文和“压缩”到底在哪里

> 本章目标：不要把 HiClaw 误解成“有一个神秘 memory API”。HiClaw 的记忆和上下文是多层组合：文件记忆、任务状态、共享文件、Matrix 历史、OpenClaw memory-core、session reset。

## 2.1 先回答一个最容易误会的问题

问：HiClaw 的“记忆”是不是某个向量数据库？

答：不完全是。

HiClaw 至少有两类记忆：

1. HiClaw 自己显式要求 Agent 读写的文件记忆。
2. OpenClaw runtime 通过 `memory-core` 和可选 `memorySearch` 提供的框架记忆能力。

换句话说：

```text
HiClaw 显式记忆协议
  memory/YYYY-MM-DD.md
  MEMORY.md
  state.json
  shared/tasks/*
  shared/projects/*

OpenClaw 运行时能力
  memory-core plugin
  dreaming
  optional embedding memorySearch
  session reset
```

这两层是互补的。文件记忆可审计、可人工编辑、可跨容器恢复；框架记忆负责更自动化的提取和检索。

## 2.2 Manager 的记忆协议

打开：

```text
manager/agent/AGENTS.md
```

找到 `Every Session`：

```text
Before doing anything:

1. Read SOUL.md
2. Read memory/YYYY-MM-DD.md (today + yesterday)
3. If in DM with the human admin: also read MEMORY.md
```

这里能看出一个很朴素但有效的设计：

- 每日记忆：记录当天和昨天发生了什么。
- 长期记忆：只在 admin DM 场景加载，避免把敏感运行经验暴露到 group room。
- 先读记忆再行动：让一个“每次可能重新醒来”的 Agent 有连续性。

再看 `Memory` 段：

```text
Daily notes: memory/YYYY-MM-DD.md
Long-term: MEMORY.md
Write significant events: Worker performance, task outcomes, decisions, lessons learned
```

这不是自动魔法，而是明确要求 Agent 写文件。

### 为什么这样做？

因为 Agent 会被不同消息唤醒，运行上下文可能被 reset，也可能换容器。你不能假设“模型还记得刚才发生了什么”。所以 HiClaw 要求关键事实落盘。

## 2.3 Worker 的记忆协议

打开：

```text
manager/agent/worker-agent/AGENTS.md
```

Worker 的规则类似：

```text
Before doing anything:
1. Read SOUL.md
2. Read memory/YYYY-MM-DD.md (today + yesterday)
```

Worker 完成任务时还必须：

```text
Log key decisions and outcomes to memory/YYYY-MM-DD.md
```

这说明 HiClaw 的记忆不是 Manager 专属，每个 Agent 都有自己的 workspace 和记忆。

## 2.4 state.json：这不是记忆，但它比记忆更重要

打开：

```text
manager/agent/skills/task-management/references/state-management.md
manager/agent/skills/task-management/scripts/manage-state.sh
```

`state.json` 是 Manager 本地任务状态的单一事实源：

```json
{
  "admin_dm_room_id": null,
  "active_tasks": [],
  "updated_at": "..."
}
```

它和 `memory/YYYY-MM-DD.md` 的区别：

- memory 是叙事性记录，给 Agent 阅读理解。
- state.json 是结构化状态，给 heartbeat 和脚本判断。

`manage-state.sh` 做了几件关键事：

- 初始化文件。
- 防重复添加同一个 task。
- 用 tmp + mv 原子写入。
- 完成任务时从 `active_tasks` 移除。
- infinite task 执行后更新下一次触发时间。

关键片段：

```bash
jq --arg id "$TASK_ID" \
   --arg title "$TITLE" \
   --arg worker "$ASSIGNED_TO" \
   --arg room "$ROOM_ID" \
   '.active_tasks += [{
        task_id: $id,
        title: $title,
        type: "finite",
        assigned_to: $worker,
        room_id: $room
    }]
    | .updated_at = $ts' \
   "$STATE_FILE" > "$tmp" && mv "$tmp" "$STATE_FILE"
```

这就是 Agent 应用开发里很重要的经验：

> 不要让 LLM 自己随手改关键状态。让 LLM 调确定性脚本。

## 2.5 任务上下文：为什么要有 spec.md 和 result.md

打开：

```text
manager/agent/skills/task-management/references/finite-tasks.md
manager/agent/worker-agent/AGENTS.md
```

一次有限任务的上下文目录：

```text
shared/tasks/{task-id}/
├── meta.json
├── spec.md
├── base/
├── plan.md
├── result.md
└── progress/
```

这个目录承担了很多 prompt 本来可能承担的职责：

- `spec.md`：需求和验收标准。
- `meta.json`：任务状态、归属、时间。
- `base/`：只读参考材料。
- `plan.md`：Worker 的执行计划。
- `result.md`：最终结果。

为什么不直接在 Matrix 消息里塞完整需求？

因为消息适合“通知和唤醒”，不适合承载大上下文、附件、迭代产物。HiClaw 的设计是：

```text
Matrix 消息：短，负责唤醒和指路
MinIO 文件：长，负责完整上下文和产物
```

例如 Manager 通知 Worker：

```text
@worker:domain New task [task-id]: title.
Use your file-sync skill to pull the spec: shared/tasks/{task-id}/spec.md.
```

这条消息只告诉 Worker 去哪里取上下文。

## 2.6 Matrix 历史上下文：只看 Current message

Manager 和 Worker 的 `AGENTS.md` 都强调一种消息格式：

```text
[Chat messages since your last reply - for context]
... history messages ...

[Current message - respond to this]
... triggering message ...
```

规则是：

> History 只做上下文，真正要响应的是 Current message。

这是一个非常实用的多 Agent 防错设计。否则 Agent 可能看到历史里有人 @mention 过自己，就重复执行旧任务。

## 2.7 OpenClaw 层的 memory-core 和 session reset

打开：

```text
hiclaw-controller/internal/agentconfig/generator.go
```

在 `GenerateOpenClawConfig` 里可以看到：

```go
"session": map[string]interface{}{
    "resetByType": map[string]interface{}{
        "dm":    map[string]interface{}{"mode": "daily", "atHour": 4},
        "group": map[string]interface{}{"mode": "daily", "atHour": 4},
    },
},
"plugins": map[string]interface{}{
    "entries": map[string]interface{}{
        "memory-core": map[string]interface{}{
            "enabled": true,
            "config": map[string]interface{}{
                "dreaming": map[string]interface{}{
                    "enabled": true,
                },
            },
        },
    },
},
```

如果配置了 embedding model，还会有：

```go
defaults["memorySearch"] = map[string]interface{}{
    "provider": "openai",
    "model":    g.config.EmbeddingModel,
    "remote": map[string]interface{}{
        "baseUrl": aiGatewayURL + "/v1",
        "apiKey":  req.GatewayKey,
    },
}
```

这说明 HiClaw 对上下文膨胀的处理不是单点的“压缩函数”，而是：

- 会话每天 reset，避免无限增长。
- 重要事实写入文件，reset 后可重新加载。
- memory-core 可以在 runtime 层做更自动的记忆管理。
- embedding memorySearch 可用于语义检索。

## 2.8 HiClaw 的“压缩”应该怎么理解

如果你问“源码里有没有 summarize conversation 的压缩器”，目前从这些文件看，HiClaw 没有把压缩设计成一个显式的 `compress_context()` 函数。

它更像用了“上下文外置 + 人工/Agent 维护摘要”的方式：

```text
长对话      -> 每日 memory 摘要
长期经验    -> MEMORY.md
当前任务    -> state.json + meta.json
大材料      -> shared/tasks/base/
任务结果    -> result.md
运行规则    -> AGENTS.md / SKILL.md
模型会话    -> daily session reset
```

这是一种工程上的压缩：不是把所有 token 压成一段摘要，而是把不同类型的信息放到不同介质里，只在需要时读取。

## 2.9 本章经典问题

### 问题 1：为什么 Manager 在 DM 才读 MEMORY.md？

因为 `MEMORY.md` 可能包含 Worker 评估、任务模式、运营经验等敏感信息。在 Worker group room 中加载它，可能把不该出现的内容带进上下文或回复。

源码对照：

```text
manager/agent/AGENTS.md
```

### 问题 2：为什么 active task 不能只靠扫描 shared/tasks/*/meta.json？

因为扫描所有任务目录成本高、容易漏状态，而且 heartbeat 需要快速知道当前活跃任务。`state.json` 是专门为 heartbeat 和调度准备的结构化索引。

源码对照：

```text
manager/agent/HEARTBEAT.md
manager/agent/skills/task-management/references/state-management.md
```

### 问题 3：如果 Worker 容器被删了，它还记得任务吗？

容器本地的进程状态会丢，但持久状态还在：

- `agents/<worker>/memory/`
- `agents/<worker>/SOUL.md`
- `agents/<worker>/AGENTS.md`
- `shared/tasks/{task-id}/`
- Manager 的 `state.json`

Worker 重启后从 MinIO 拉回这些文件，再继续。

### 问题 4：HiClaw 的上下文压缩对 Agent 应用开发有什么启发？

不要迷信“把全部历史塞给模型”。更稳的做法是分层：

- 当前消息负责触发。
- 任务文件负责完整需求。
- 状态文件负责机器可读进度。
- 日记文件负责短期连续性。
- 长期记忆负责经验沉淀。
- reset 负责清理膨胀会话。

