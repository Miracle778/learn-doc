# 第 1 章：Agent 是怎样启动起来的

> 本章目标：弄清楚 HiClaw 里的“Agent”不是凭空出现的。它是由镜像、启动脚本、workspace、prompt、skills、openclaw.json、Matrix token、Gateway key 一层层装配出来的。

## 1.1 先看 Manager 启动入口

打开：

```text
manager/scripts/init/start-manager-agent.sh
```

这个脚本做的事情很多，但主线只有一条：

```text
判断 Manager runtime
  -> 准备环境变量、域名、凭证
  -> 等待 Matrix / MinIO / Higress
  -> 同步 workspace
  -> 运行 upgrade-builtins.sh
  -> 获取 Matrix token
  -> 启动 OpenClaw 或 CoPaw Manager
```

重点看前半段：

```bash
MANAGER_RUNTIME="${HICLAW_MANAGER_RUNTIME:-openclaw}"
case "${MANAGER_RUNTIME}" in
    copaw)
        log "Manager runtime: CoPaw (Python workspace)"
        ;;
    *)
        log "Manager runtime: OpenClaw (Node.js gateway)"
        MANAGER_RUNTIME="openclaw"
        ;;
esac
```

这告诉你：Manager runtime 当前主要是两种：

- `openclaw`：Node/OpenClaw。
- `copaw`：Python/CoPaw。

Hermes、OpenHuman 在当前 API/Helm 里是 Worker runtime，不是 Manager runtime。

## 1.2 Agent workspace 是什么

打开：

```text
manager/agent/AGENTS.md
```

第一段就把 workspace 讲清楚了：

```text
Your workspace: ~/ (SOUL.md, openclaw.json, memory/, skills/, state.json, workers-registry.json)
Shared space: /root/hiclaw-fs/shared/
Worker files: /root/hiclaw-fs/agents/<worker-name>/
```

这三个路径非常重要：

- `~/` 是 Manager 自己的脑内工作台：身份、技能、记忆、任务状态。
- `/root/hiclaw-fs/shared/` 是大家协作的公共文件区。
- `/root/hiclaw-fs/agents/<worker-name>/` 是 Manager 能看到的 Worker 配置和状态镜像。

所以 HiClaw 的上下文不是只存在于聊天记录里，而是分散在：

```text
Matrix messages
workspace markdown files
state.json
shared/tasks/*
shared/projects/*
agents/<worker>/*
openclaw.json
```

## 1.3 内置 prompt 和 skills 怎么升级

打开：

```text
manager/scripts/init/upgrade-builtins.sh
```

这个脚本回答了一个常见问题：

> 镜像里更新了 AGENTS.md 或 SKILL.md，已经运行过的 Manager/Worker 怎么拿到新版本？

主线是：

1. 从 `/opt/hiclaw/agent` 读镜像内置版本。
2. 合并到 `/root/manager-workspace`。
3. `SOUL.md`、`AGENTS.md`、`HEARTBEAT.md` 用 builtin section 方式合并，尽量保留用户自定义内容。
4. `scripts/` 和 `references/` 直接覆盖，因为它们更像程序逻辑。
5. Worker builtins 推送到 MinIO 的 `shared/builtins/worker/`。
6. 已注册 Worker 的 `agents/<worker>/` 也会同步更新。

关键片段：

```bash
update_builtin_section "${WORKSPACE}/SOUL.md" "${AGENT_SRC}/SOUL.md"
update_builtin_section "${WORKSPACE}/AGENTS.md" "${AGENT_SRC}/AGENTS.md"

for skill_dir in "${AGENT_SRC}/skills"/*/; do
    skill_name=$(basename "${skill_dir}")
    _upgrade_skill_md "${skill_dir}SKILL.md" "${WORKSPACE}/skills/${skill_name}/SKILL.md"
done
```

这里的设计很值得学：

- prompt 是运行时协议，所以需要升级机制。
- 但 prompt 也可能被 Agent 或用户改过，所以不能简单覆盖。
- scripts/references 更接近确定性程序，覆盖更安全。

## 1.4 Worker 启动入口

打开：

```text
worker/scripts/worker-entrypoint.sh
```

这个脚本是 Worker 从“一个容器”变成“一个 Agent”的过程。

简化时间线：

```text
Step 1: 配置 mc alias，连上 MinIO/OSS
Step 2: 拉取 agents/<worker-name>/ 到本地 workspace
Step 3: 检查 openclaw.json / SOUL.md / AGENTS.md 是否存在
Step 4: 建 ~/.openclaw/openclaw.json 软链
Step 5: 启动本地到远端的变更同步
Step 6: 启动远端到本地的 fallback 同步
Step 7: 配置 mcporter
Step 8: 启动 openclaw gateway
```

看这段：

```bash
mc mirror "${HICLAW_STORAGE_PREFIX}/agents/${WORKER_NAME}/" "${WORKSPACE}/" --overwrite \
    --exclude ".openclaw/matrix/**" --exclude ".openclaw/canvas/**" --exclude "credentials/**"
mc mirror "${HICLAW_STORAGE_PREFIX}/shared/" "${HICLAW_ROOT}/shared/" --overwrite 2>/dev/null || true
```

这说明 Worker 的启动不是把配置 baked 到镜像里，而是启动时从对象存储拉取。镜像只是 runtime，真实身份和任务上下文在存储里。

## 1.5 openclaw.json 是 Agent 的运行时装配单

打开：

```text
hiclaw-controller/internal/agentconfig/generator.go
```

重点读 `GenerateOpenClawConfig`。

它生成的配置包含：

- `gateway`：本地 OpenClaw gateway 端口、认证 token、Control UI 设置。
- `channels.matrix`：Matrix homeserver、userId、accessToken、allowlist、requireMention、autoJoin。
- `models.providers.hiclaw-gateway`：模型通过 Higress 的 OpenAI-compatible API 访问。
- `agents.defaults`：workspace、primary model、timeout、并发数、subagents 并发数。
- `plugins.entries.memory-core`：启用 OpenClaw memory-core。
- `memorySearch`：如果配置 embedding model，就启用记忆搜索。
- `session.resetByType`：DM 和 group 每天 4 点 reset。

关键片段：

```go
"agents": map[string]interface{}{
    "defaults": map[string]interface{}{
        "timeoutSeconds": 1800,
        "workspace":      "~",
        "model": map[string]interface{}{
            "primary": "hiclaw-gateway/" + modelName,
        },
        "maxConcurrent": 4,
        "subagents": map[string]interface{}{
            "maxConcurrent": 8,
        },
    },
},
"session": map[string]interface{}{
    "resetByType": map[string]interface{}{
        "dm":    map[string]interface{}{"mode": "daily", "atHour": 4},
        "group": map[string]interface{}{"mode": "daily", "atHour": 4},
    },
},
"plugins": map[string]interface{}{
    "entries": map[string]interface{}{
        "matrix": map[string]interface{}{"enabled": true},
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

注意：这里能看到 OpenClaw 层的记忆插件，但 HiClaw 自己还额外定义了文件记忆协议。第 2 章会展开。

## 1.6 Matrix channel 的一个关键设计：必须 @mention

`buildMatrixChannelConfig` 里有：

```go
"groups": map[string]interface{}{
    "*": map[string]interface{}{"allow": true, "requireMention": true},
},
"autoJoin": "always",
```

这对应 `AGENTS.md` 里的规则：

```text
OpenClaw only processes messages that @mention you.
```

为什么要这样？

因为多 Agent 房间里消息很多。如果每条消息都唤醒所有 Agent，会很容易出现循环：

```text
Manager: Alice 做一下 X
Alice: 好的
Manager: 收到
Alice: 收到你的收到
...
```

所以 HiClaw 把“唤醒”变成显式协议：只有 full Matrix ID 的 @mention 才触发处理。

## 1.7 本章经典问题

### 问题 1：Agent 的“身份”在哪里？

主要在：

```text
SOUL.md
AGENTS.md
openclaw.json channels.matrix.userId
Matrix account
Gateway consumer key
```

`SOUL.md` 定义角色和长期人格；`AGENTS.md` 定义运行规则；`openclaw.json` 绑定通信账号、模型和工具。

### 问题 2：为什么 HiClaw 要把 skills 写成 Markdown？

因为 skills 同时是“给 Agent 看的操作手册”和“可版本化的能力包”。Agent 在运行时读 `SKILL.md`，再调用旁边的 scripts/references。这比把所有逻辑藏进模型 prompt 更可维护。

### 问题 3：为什么配置文件要从 MinIO 拉，而不是写死在镜像？

因为 Worker 要可重建、可切 runtime、可动态增减 skills/MCP。镜像固定 runtime，MinIO 固定身份和上下文。这样容器可以丢，Agent 的“记忆和工作台”还在。

