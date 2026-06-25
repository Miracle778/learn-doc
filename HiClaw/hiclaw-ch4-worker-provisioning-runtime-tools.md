# 第 4 章：Worker 从创建到可执行的完整链路

> 本章目标：从 `hiclaw create worker` 往下追，理解一个 Worker 是如何拥有账号、房间、模型权限、文件系统、技能和容器的。

## 4.1 从 API 类型开始看

打开：

```text
hiclaw-controller/api/v1beta1/types.go
```

先读 `WorkerSpec`：

```go
type WorkerSpec struct {
    Model        string
    Runtime      string
    Image        string
    WorkerName   string
    Identity     string
    Soul         string
    Agents       string
    Skills       []string
    RemoteSkills []RemoteSkillSource
    McpServers   []MCPServer
    Package      string
    Expose       []ExposePort
    ChannelPolicy *ChannelPolicySpec
    Resources    *AgentResourceRequirements
    ContainerManaged *bool
    State *string
}
```

这里有一个很强的 Kubernetes/Operator 味道：`WorkerSpec` 不是“立即执行的命令”，而是“期望状态”。

你声明：

- 用哪个 model。
- 用哪个 runtime。
- 需要哪些 skills。
- 需要哪些 MCP servers。
- 是否暴露端口。
- 容器是否由 controller 管。
- 期望状态是 Running / Sleeping / Stopped。

Controller 负责把这个 spec 落地。

## 4.2 runtime 为什么是 API 的一等字段

`WorkerSpec.Runtime` 支持：

```text
openclaw | copaw | hermes | openhuman
```

打开：

```text
hiclaw-controller/internal/backend/interface.go
```

你会看到：

```go
const (
    RuntimeOpenClaw  = "openclaw"
    RuntimeCopaw     = "copaw"
    RuntimeHermes    = "hermes"
    RuntimeOpenHuman = "openhuman"
)
```

以及 `ResolveRuntime`：

```go
func ResolveRuntime(reqRuntime, fallback string) string {
    if reqRuntime != "" {
        return reqRuntime
    }
    if fallback != "" {
        return fallback
    }
    return RuntimeOpenClaw
}
```

这里的设计点是：CRD 不直接写死 schema default，而是允许 controller 根据环境变量决定默认 Worker runtime。这让安装时的默认 runtime 可以生效。

## 4.3 Provisioner：创建基础设施身份

打开：

```text
hiclaw-controller/internal/service/provisioner.go
```

重点读 `ProvisionWorker`。

它按步骤做：

```text
Step 1: load or generate credentials
Step 2: register Matrix account
Step 3: create MinIO user and policy
Step 4: create Matrix room
Step 4b: have worker join its own room
Step 5: create Higress gateway consumer and authorize AI routes
```

这对应一个 Worker 的“基础设施身份证”：

- Matrix user：能进房间说话。
- Matrix room：Human + Manager/Leader + Worker 的协作空间。
- MinIO user/policy：能读写自己的 workspace 和 shared。
- Higress consumer/key：能访问 LLM 和 MCP。

关键片段：

```go
workerMatrixID := p.matrix.UserID(workerName)
managerMatrixID := p.matrix.UserID("manager")
adminMatrixID := p.matrix.UserID(p.adminUser)
```

这说明 Worker 的名字不仅是容器名，也会变成 Matrix localpart 和对象存储路径的一部分。

再看创建房间：

```go
roomInfo, err := p.matrix.CreateRoom(ctx, matrix.CreateRoomRequest{
    Name:          fmt.Sprintf("Worker: %s", workerName),
    Topic:         fmt.Sprintf("Communication channel for %s", workerName),
    Invite:        invite,
    PowerLevels:   powerLevels,
    RoomAliasName: roomAliasLocalpart("worker", workerName),
})
```

HiClaw 把每个 Worker 都绑定到一个可恢复的 room alias，这有利于 reconcile。

## 4.4 一个很细但很重要的 readiness 注释

`ProvisionWorker` 里有一段注释很值得认真读：

```text
"membership = join" is necessary but NOT sufficient for "worker is ready to process messages".
```

意思是：Worker 加入房间不等于它已经能处理消息。某些 runtime 在首次 sync catch-up 时会丢掉早到的消息。

这也是为什么 HiClaw 文档和测试里经常强调 at-least-once send、等待 ready、不要过早假设 Worker 可用。

Agent 应用开发常见坑就是：

```text
账号创建成功 != Agent 可处理任务
房间加入成功 != runtime 已经 ready
容器 running != 模型/网关/文件同步都 ready
```

## 4.5 Deployer：把 Agent 的“灵魂和工具”推到存储

打开：

```text
hiclaw-controller/internal/service/deployer.go
```

重点读 `DeployWorkerConfig`。

它负责：

- 生成 `openclaw.json`。
- 写 `SOUL.md`。
- 写 `mcporter-servers.json`。
- 写 Matrix password。
- 合并/推送 `AGENTS.md`。
- 推 builtin skills。
- 推 on-demand skills / remote skills。

简化主线：

```text
localAgentDir = /root/hiclaw-fs/agents/<name>
agentPrefix   = agents/<name>

seed local files
generate openclaw.json
put agents/<name>/openclaw.json
put SOUL.md if first deploy
put mcporter config if MCP declared
put Matrix password if needed
prepare AGENTS.md
push builtin skills
```

这回答了一个非常关键的问题：

> Worker 的 prompt 和技能到底是谁发给它的？

答案：Controller 的 Deployer 把它们推到对象存储，Worker 启动时从对象存储拉。

## 4.6 AGENTS.md 的合并和协调上下文注入

`DeployWorkerConfig` 里有：

```go
d.prepareAndPushAgentsMD(...)
d.pushBuiltinSkills(...)
```

团队场景还有：

```go
InjectCoordinationContext
```

它会给 Team Leader 注入：

- team name
- team room id
- leader DM room id
- worker list
- admin id
- heartbeat interval

这说明 Team Leader 不是靠“猜测自己有哪些队友”来调度，而是启动时就拿到结构化协调上下文。

## 4.7 MCP 工具是怎么接进来的

先看 CRD：

```text
hiclaw-controller/api/v1beta1/types.go
```

`MCPServer`：

```go
type MCPServer struct {
    Name      string
    URL       string
    Transport string
}
```

注释里写得很清楚：

```text
The controller translates this slice directly into mcporter-servers.json
and injects Authorization: Bearer <consumer-key>.
```

再看 `deployer.go`：

```go
if len(req.McpServers) > 0 {
    mcporterJSON, err := d.agentConfig.GenerateMcporterConfig(req.GatewayKey, req.McpServers)
    ...
    d.oss.PutObject(ctx, agentPrefix+"/mcporter-servers.json", mcporterJSON)
}
```

最后 Worker entrypoint：

```text
worker/scripts/worker-entrypoint.sh
```

会把 `mcporter-servers.json` 链到默认路径：

```bash
MCPORTER_DEFAULT="${WORKSPACE}/config/mcporter.json"
MCPORTER_COMPAT="${WORKSPACE}/mcporter-servers.json"
ln -sfn "${MCPORTER_DEFAULT}" "${MCPORTER_COMPAT}"
export MCPORTER_CONFIG="${MCPORTER_DEFAULT}"
```

整体链路：

```text
WorkerSpec.mcpServers
  -> GenerateMcporterConfig
  -> agents/<worker>/mcporter-servers.json or config/mcporter.json
  -> worker file-sync
  -> mcporter CLI
  -> Higress MCP endpoint with Bearer gateway key
```

## 4.8 Worker 容器生命周期抽象

打开：

```text
hiclaw-controller/internal/backend/interface.go
```

`WorkerBackend` 定义了统一接口：

```go
type WorkerBackend interface {
    Available(ctx context.Context) bool
    Create(ctx context.Context, req CreateRequest) (*WorkerResult, error)
    Delete(ctx context.Context, name string) error
    Start(ctx context.Context, name string) error
    Stop(ctx context.Context, name string) error
    Status(ctx context.Context, name string) (*WorkerResult, error)
}
```

这让 HiClaw 能同时支持：

- Docker/Podman 本地模式。
- Kubernetes Pod 模式。
- remote/pip worker 等不由 controller 管容器的模式。

这也是 Agent 应用从 demo 走向产品时必须做的抽象：不要把“执行体”写死成一个本地进程。

## 4.9 Worker entrypoint 的同步策略

再次打开：

```text
worker/scripts/worker-entrypoint.sh
```

它有两个同步方向：

### Local -> Remote

Worker 自己写出的文件会推回 MinIO，但排除敏感/缓存/运行时文件：

```bash
mc mirror "${WORKSPACE}/" "${HICLAW_STORAGE_PREFIX}/agents/${WORKER_NAME}/" --overwrite \
  --exclude "openclaw.json" \
  --exclude "credentials/**" \
  --exclude ".cache/**" \
  --exclude ".openclaw/matrix/**"
```

### Remote -> Local

Manager 管理的配置会 fallback 拉取：

```bash
mc cp openclaw.json
mc mirror skills/
mc mirror shared/
```

设计原则写在注释里：

```text
The party that writes a file is responsible for:
1. Pushing it to MinIO immediately
2. Notifying the other side via Matrix @mention
```

这句话几乎可以当作 HiClaw 文件协作协议的核心。

## 4.10 本章经典问题

### 问题 1：一个 Worker 为什么需要 Gateway consumer？

因为 Worker 调 LLM 和 MCP 都经过 Higress。Gateway consumer key 是它访问 AI route 和 MCP route 的身份凭证。

源码对照：

```text
hiclaw-controller/internal/service/provisioner.go
hiclaw-controller/internal/agentconfig/generator.go
```

### 问题 2：为什么 `SOUL.md` 是 seed-only？

因为 Agent 可能在运行中自我调整或用户可能修改它。Controller 第一次创建时可以播种，后续 reconcile 不能无脑覆盖，否则会丢用户/Agent 自定义。

源码对照：

```text
hiclaw-controller/internal/service/deployer.go
```

### 问题 3：为什么 runtime 切换是破坏性操作？

因为切 runtime 会删除旧容器、用新镜像重建。Matrix account、room、MinIO 数据可以保留，但容器本地缓存、进程内 session、临时文件会丢。

源码对照：

```text
manager/agent/skills/worker-management/SKILL.md
manager/agent/skills/worker-management/scripts/update-worker-config.sh
```

### 问题 4：为什么 MCP 配置里要注入同一个 GatewayKey？

因为 HiClaw 让 Worker 用同一个 consumer key 访问 LLM 和 MCP。这样权限可以在网关层统一管理，Worker 不需要直接持有每个上游服务的原始密钥。

源码对照：

```text
hiclaw-controller/api/v1beta1/types.go
hiclaw-controller/internal/service/deployer.go
manager/agent/worker-agent/skills/mcporter/SKILL.md
```

