# 第 8 章 精讲版：对外接口——FastAPI Gateway 与 SSE 流式

> 这是整个学习指南的最后一章。前面 7 章讲的都是"Agent 内部怎么运作"，这一章讲"外部怎么跟 Agent 通信"。
>
> **配套文件**：
> - `backend/app/gateway/app.py`
> - `backend/app/gateway/routers/threads.py`
> - `backend/app/gateway/routers/thread_runs.py` / `runs.py`
> - `backend/app/gateway/services.py`
> - `backend/app/gateway/authz.py`

---

# 🎯 先建立心理模型：Gateway 是"前台"

DeerFlow 的所有功能都通过一个 **FastAPI 服务**对外暴露。你可以把它理解成公司的"前台"：

```
┌─────────────────── 浏览器 / SDK ───────────────────┐
│  useStream React Hook（消费 SSE）                    │
└────────────────────────┬────────────────────────────┘
                         │ HTTP / SSE
                         ▼
┌──────────────── FastAPI Gateway（端口 8001）──────────┐
│                                                        │
│  中间件层：                                              │
│    AuthMiddleware（鉴权，fail-closed）                  │
│    → CSRFMiddleware（double-submit cookie）             │
│    → CORSMiddleware（跨域）                              │
│                                                        │
│  路由层：                                                │
│    /api/threads        ← 会话 CRUD                      │
│    /api/threads/{id}/runs/stream  ← ★ SSE 流式          │
│    /api/models         ← 模型列表                        │
│    /api/skills         ← 技能管理                        │
│    /api/mcp            ← MCP 配置                        │
│    /api/memory         ← 记忆管理                        │
│    /api/uploads        ← 文件上传                        │
│    /api/auth           ← 登录/注册                       │
│    ...                                                  │
│                                                        │
│  服务层：services.py                                    │
│    start_run() → asyncio.create_task(run_agent(...))   │
│    sse_consumer() → SSE 生成器                          │
│                                                        │
└────────────────────────┬───────────────────────────────┘
                         │
                         ▼
                    Runtime（第 5 章）
```

---

# 📖 8.1 FastAPI 应用：`create_app()`

**文件**：`backend/app/gateway/app.py:247`

```python
def create_app() -> FastAPI:
    app = FastAPI(title="DeerFlow API Gateway")

    # 中间件（按顺序）
    app.add_middleware(AuthMiddleware)      # ① 鉴权（fail-closed 兜底）
    app.add_middleware(CSRFMiddleware)      # ② CSRF 防护
    if cors_origins:
        app.add_middleware(CORSConfiguration)  # ③ 跨域

    # 路由
    app.include_router(models.router)
    app.include_router(mcp.router)
    app.include_router(memory.router)
    app.include_router(skills.router)
    app.include_router(threads.router)      # /api/threads
    app.include_router(thread_runs.router)  # /api/threads/{id}/runs
    app.include_router(runs.router)         # /api/runs
    app.include_router(auth.router)
    ...

    @app.get("/health")
    def health():
        return {"status": "healthy"}

    return app

app = create_app()    # 模块级，给 uvicorn 用
```

## 8.1.1 Lifespan——启动生命周期

**文件**：`app.py:161`

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # 启动
    load_config()
    apply_logging_level()
    if not auth_enabled:
        logger.warning("Auth is disabled!")
    
    # 预热 tiktoken（token_counting="char" 时跳过）
    if memory_config.token_counting == "tiktoken":
        warm_tiktoken_cache()
    
    # 进入 Runtime async context（装 StreamBridge + RunManager + checkpointer + store）
    async with langgraph_runtime(app, startup_config) as runtime:
        # 首启建管理员 + 孤儿线程迁移
        await _ensure_admin_user()
        # 启 IM channel service
        await im_channel_service.start()
        
        yield    # ← 服务运行中
    
    # 关闭
    await im_channel_service.stop()
```

**关键**：`langgraph_runtime(app, config)` 是个 async context manager，在里面 checkpointer/store/stream_bridge 都活着，出了 context 就清理。

---

# 📖 8.2 Threads CRUD（`/api/threads`）

**文件**：`backend/app/gateway/routers/threads.py`

| 方法 | 路由 | 函数 | 权限 |
|---|---|---|---|
| POST | `` | create_thread | — |
| POST | `/search` | search_threads | — |
| GET | `/{id}` | get_thread | `threads:read` + owner_check |
| PATCH | `/{id}` | patch_thread | `threads:write` + owner_check |
| DELETE | `/{id}` | delete_thread | `threads:delete` + owner_check |
| GET | `/{id}/state` | get_thread_state | `threads:read` |
| POST | `/{id}/state` | update_thread_state | `threads:write` |
| POST | `/{id}/history` | get_thread_history | `threads:read` |

## 8.2.1 双后端

后端同时用两个存储：

- **`thread_store`**（ThreadMetaStore：sqlite/postgres/memory）——存线程元数据（标题、状态、owner）；
- **`checkpointer`**（LangGraph）——存图状态（消息、todos、sandbox）。

**线程状态从 checkpoint 派生**（`_derive_thread_status`，188 行）：

```python
def _derive_thread_status(checkpoint):
    pending_writes = checkpoint.get("pending_writes", {})
    tasks = checkpoint.get("tasks", ())
    if pending_writes or tasks:
        return "running"
    return "idle"
```

## 8.2.2 安全细节

**身份防伪**（threads.py:41）：

```python
_SERVER_RESERVED_METADATA_KEYS = {"owner_id", "user_id"}
```

客户端传的 metadata 里这些字段会被**剥离**——防止用户伪造身份。

**时间有序 ID**（`update_thread_state`，544 行）：写新 checkpoint 用 `uuid6`（time-ordered），是 INSERT 不是 REPLACE——不覆盖历史 checkpoint。

---

# 📖 8.3 Runs——SSE 流式（核心）

这是最重要的部分——用户发消息后怎么实时看到 agent 的"思考过程"。

## 8.3.1 两套路由

| 路由前缀 | 文件 | 特点 |
|---|---|---|
| `/api/threads/{thread_id}/runs/*` | `thread_runs.py` | 有 thread 上下文 |
| `/api/runs/*` | `runs.py` | 无状态，没 thread_id 自动建临时 uuid4 |

## 8.3.2 RunCreateRequest——请求格式

**文件**：`thread_runs.py:37`

```python
class RunCreateRequest(BaseModel):
    assistant_id: str | None = None
    input: dict | None = None           # {"messages": [{"role":"user","content":"..."}]}
    command: dict | None = None          # LangGraph command（如 resume）
    metadata: dict | None = None
    config: dict | None = None           # configurable 等
    context: dict | None = None          # ★ DeerFlow 扩展
    stream_mode: list[str] | None = None  # ["values","messages"]
    multitask_strategy: str = "reject"   # reject/interrupt/rollback
    on_disconnect: str = "cancel"        # cancel/continue
    on_completion: str = "delete"        # delete/keep
```

**`context` 字段是 DeerFlow 扩展**（services.py:125 白名单）：

```python
# 这些是 DeerFlow 自己的运行时参数，不是 LangGraph 标准的
context_keys = {
    "model_name", "thinking_enabled", "reasoning_effort",
    "is_plan_mode", "subagent_enabled", "max_concurrent_subagents",
    "agent_name", "is_bootstrap",
}
```

## 8.3.3 关键端点

**文件**：`thread_runs.py`

| 方法 | 路由 | 函数 | 作用 |
|---|---|---|---|
| POST | `/{id}/runs` | create_run | 后台 run（不流式） |
| POST | `/{id}/runs/stream` | stream_run | ★ SSE 流式 |
| POST | `/{id}/runs/wait` | wait_run | 阻塞到完成 |
| POST | `/{id}/runs/{rid}/cancel` | cancel_run | 取消（action=interrupt/rollback） |
| GET\|POST | `/{id}/runs/{rid}/stream` | join_or_cancel_stream | 加入/取消重流 |
| GET | `/{id}/runs/{rid}/join` | join_run | 加入现有流 |

**`stream_run` 返回**：

```python
return StreamingResponse(
    sse_consumer(bridge, record, request, run_mgr),
    media_type="text/event-stream",
    headers={"Content-Location": f"/api/threads/{thread_id}/runs/{run_id}"},
)
```

---

# 📖 8.4 SSE 流式机制

**文件**：`backend/app/gateway/services.py`

## 8.4.1 `start_run`——启动后台 run

**文件**：`services.py:278`

```python
def start_run(body, thread_id, request):
    # 1. 解析 agent factory（所有 assistant_id 都映射到 make_lead_agent）
    agent_factory = resolve_agent_factory(body.assistant_id)
    
    # 2. 校验模型白名单
    validate_model(body.context.get("model_name"))
    
    # 3. 校验线程归属（这线程是你的吗？）
    if not is_internal_caller:
        thread_store.check_access(thread_id, current_user_id)
    
    # 4. 创建 RunRecord（原子，防并发冲突）
    record = run_mgr.create_or_reject(
        thread_id=thread_id,
        multitask_strategy=body.multitask_strategy,
        ...
    )
    
    # 5. 构建 config
    config = build_run_config(body, thread_id)
    
    # 6. ★ 启动后台任务！
    asyncio.create_task(run_agent(
        bridge, run_mgr, record, ctx,
        agent_factory, graph_input, config, stream_modes, ...
    ))
    
    return record
```

**关键认知**：`start_run` 不等 agent 跑完——它启动后台 task 后立即返回 `run_id`。前端拿 `run_id` 去 `GET /runs/{run_id}/stream` 订阅 SSE。

**为什么所有 `assistant_id` 都映射到同一个 `make_lead_agent`？** 因为 DeerFlow 的"多 agent"不是"多个独立 graph"，而是**一个 graph + 配置路由**。`make_lead_agent(config)` 内部根据 `config["agent_name"]` 决定加载哪个 SOUL.md、装哪些 skill。

## 8.4.2 `format_sse`——SSE 格式

**文件**：`services.py:47`

```python
def format_sse(event: str, data: str, *, event_id: str | None = None) -> str:
    lines = [f"event: {event}", f"data: {data}"]
    if event_id:
        lines.append(f"id: {event_id}")
    lines.append("")    # 空行结束
    lines.append("")
    return "\n".join(lines)
```

**字段顺序**：`event:` → `data:` → 可选 `id:` → 空行。匹配 LangGraph Platform wire format（`useStream` React hook 消费）。

## 8.4.3 `sse_consumer`——SSE 生成器

**文件**：`services.py:401`

```python
async def sse_consumer(bridge, record, request, run_mgr):
    # 读 Last-Event-ID 头（断线重连用）
    last_event_id = request.headers.get("Last-Event-ID")
    
    # 订阅 StreamBridge
    async for entry in bridge.subscribe(record.run_id, last_event_id=last_event_id):
        # 检查客户端是否断连
        if await request.is_disconnected():
            if on_disconnect == "cancel":
                run_mgr.cancel_run(record.run_id)
            break
        
        if entry is HEARTBEAT_SENTINEL:
            yield ": heartbeat\n\n"    # SSE 注释（保活）
        elif entry is END_SENTINEL:
            yield format_sse("end", "{}")
            return
        else:
            yield format_sse(entry.event, entry.data, event_id=entry.id)
```

**三个关键行为**：

### ① Last-Event-ID 断线重连

客户端断连后重连时带 `Last-Event-ID: {上次的 id}`，`bridge.subscribe` 从该 id 之后的事件开始重放——不丢事件。

### ② 心跳保活

`HEARTBEAT_SENTINEL` → `": heartbeat\n\n"`（SSE 注释格式）——浏览器不会显示，但保持连接活着，防代理超时断连。

### ③ 断连取消

`finally` 块在 `on_disconnect=cancel` 时调 `run_mgr.cancel_run(run_id)`——客户端断了就取消 run，不浪费资源。

## 8.4.4 `wait_for_run_completion`

**文件**：`services.py:435`

只在观测到 `END_SENTINEL` 时返回 True——避免序列化半成品 checkpoint（issue #3265）。如果客户端断连导致没看到 END，返回 False，调用方知道"run 没正常结束"。

---

# 📖 8.5 Auth——鉴权

**文件**：`backend/app/gateway/authz.py`

## 8.5.1 权限装饰器

```python
@require_permission("threads", "read", owner_check=True)
async def get_thread(thread_id, ...):
    ...
```

`@require_permission(resource, action, owner_check=False)` 装饰器：

1. 检查 `auth_context.has_permission(f"{resource}:{action}")`；
2. 如果 `owner_check=True`，还要检查 `thread_store.check_access(thread_id, user_id)`——这线程是不是你的。

## 8.5.2 权限常量

```python
class Permissions:
    THREADS_READ = "threads:read"
    THREADS_WRITE = "threads:write"
    THREADS_DELETE = "threads:delete"
    RUNS_CREATE = "runs:create"
    RUNS_READ = "runs:read"
    RUNS_CANCEL = "runs:cancel"
```

## 8.5.3 IM Channel 豁免

IM channel（飞书/钉钉等 IM 集成）代替用户行动时，是 **internal system role**——豁免 owner_check。因为 IM 消息触发时可能没有明确的"线程所有者"概念。

## 8.5.4 用户上下文注入

**文件**：`services.py:166`

```python
def inject_authenticated_user_context(config, request):
    user_id = request.auth.user_id
    config["context"]["user_id"] = user_id
```

把服务端 `user_id` 盖进 `config["context"]`——让后台工具在请求 handler 返回后（HTTP 上下文已销毁）仍能持久化用户级文件。

---

# 🎯 现在回头检验：你掌握了吗？

### 问题 1：为什么 SSE 的 `format_sse` 要严格控制字段顺序？

<details>
<summary>🔍 点击看答案</summary>

因为前端用 LangGraph 的 `useStream` React hook，它按 SSE 规范解析：`event:` 行声明事件类型，`data:` 行是 JSON payload，`id:` 行是用于断线重连的 `Last-Event-ID`。顺序乱了 hook 会解析失败。

DeerFlow 故意复刻 LangGraph Platform 的 wire format，这样前端可以**直接用官方 SDK**，不用自己写客户端。

</details>

### 问题 2：客户端断连了，run 怎么办？

<details>
<summary>🔍 点击看答案</summary>

看 `on_disconnect` 配置：

- `"cancel"`（默认）：`sse_consumer` 的 finally 块检测到 `request.is_disconnected()` 后取消 run（设 abort_event）；
- `"continue"`：run 继续在后台跑，客户端可以用 `Last-Event-ID` 重连 `GET /runs/{id}/stream` 续上。

`wait_for_run_completion` 还会**只在看到 END_SENTINEL 时返回 True**——避免客户端断连时序列化半成品 checkpoint 让前端看到错误状态（#3265）。

</details>

### 问题 3：为什么所有 `assistant_id` 都映射到同一个 `make_lead_agent`？

<details>
<summary>🔍 点击看答案</summary>

因为 DeerFlow 的"多 agent"不是"多个独立 graph"，而是**一个 graph + 配置路由**。`make_lead_agent(config)` 接收 `config["configurable"]["agent_name"]`，内部根据它决定：

- 加载哪个 SOUL.md（人格）；
- 装哪些 skill；
- 是否加 `update_agent` 工具；
- 用什么 system prompt。

所以"创建一个新 agent"不是部署一个新服务，而是**写一份 SOUL.md + config 片段**。这种"配置即 agent"的设计让 agent 管理非常轻量。

</details>

### 问题 4：`multitask_strategy` 的三种策略有什么区别？

<details>
<summary>🔍 点击看答案</summary>

当同一 thread 已有 run 在跑，又来一个 run 请求时：

- **`reject`**：直接拒绝新请求（409）；
- **`interrupt`**：中断旧 run（设 interrupted），再跑新的；
- **`rollback`**：把旧 run 标 error 并**回滚到它开始前的 checkpoint**，再跑新的。

`create_or_reject` 用原子 check-then-insert 消除 TOCTOU 竞态——避免"检查时没冲突，插入时冲突了"。

</details>

### 问题 5：为什么客户端传的 metadata 里 `owner_id` / `user_id` 会被剥离？

<details>
<summary>🔍 点击看答案</summary>

**防身份伪造**。如果允许客户端传 `owner_id`，恶意用户可以把别人的 thread 设成自己的，或者把自己的操作伪装成别人的。

`_SERVER_RESERVED_METADATA_KEYS = {"owner_id", "user_id"}`——这些字段只能由服务端（auth 中间件）写入，客户端传的会被静默丢弃。

</details>

---

# 📝 一页纸总结（第 8 章精华）

```
┌──────────────────────────────────────────────────────────────┐
│  Gateway = FastAPI 前台，对外暴露 REST + SSE                    │
└──────────────────────────────────────────────────────────────┘
        │
        ├─▶ 应用（app.py:create_app）
        │     • 中间件：Auth → CSRF → CORS
        │     • lifespan：load config → warm tiktoken → langgraph_runtime → ensure admin
        │     • 路由：threads / runs / models / skills / mcp / memory / uploads / auth
        │
        ├─▶ Threads CRUD（/api/threads）
        │     • 双后端：thread_store（元数据）+ checkpointer（图状态）
        │     • 状态从 checkpoint pending_writes/tasks 派生
        │     • 防身份伪造：owner_id/user_id 服务端保留
        │     • uuid6 时间有序 checkpoint ID
        │
        ├─▶ Runs SSE（核心）
        │     • start_run: 校验→create_or_reject→asyncio.create_task(run_agent)
        │     • 所有 assistant_id → make_lead_agent（配置即 agent）
        │     • context 扩展：model_name/thinking/subagent/agent_name 等
        │     • multitask_strategy: reject/interrupt/rollback
        │
        ├─▶ SSE 机制（services.py）
        │     • format_sse: event→data→id→空行（LangGraph wire format）
        │     • sse_consumer: Last-Event-ID 重连 + 心跳 + 断连取消
        │     • wait_for_run_completion: 只在 END_SENTINEL 返回 True
        │
        └─▶ Auth（authz.py）
              • @require_permission(resource, action, owner_check)
              • 权限：threads:read/write/delete, runs:create/read/cancel
              • IM channel 豁免 owner_check（internal system role）
              • inject_authenticated_user_context: 服务端 user_id 盖进 config
```

**5 句话记住**：

1. **Gateway 是 FastAPI 前台**——Auth→CSRF→CORS 中间件链，lifespan 装 Runtime，路由覆盖 threads/runs/models/skills/mcp/memory；
2. **Threads 双后端**——thread_store 存元数据，checkpointer 存图状态，状态从 pending_writes 派生，`owner_id` 服务端保留防伪造；
3. **SSE 是核心**——`start_run` 启后台 task 立即返回 run_id，`sse_consumer` 消费 StreamBridge 推 SSE，`Last-Event-ID` 断线重连；
4. **所有 assistant_id → make_lead_agent**——配置即 agent，路由靠 `agent_name`，不是多个独立服务；
5. **Auth 三层**——权限装饰器 + owner_check + IM 豁免，服务端 user_id 注入 config 让后台工具能持久化用户级文件。

---

# 🎉 全部 8 章完结！

恭喜你读完了全部 DeerFlow 2.0 源码学习指南！回顾一下你学到了什么：

| 章 | 主题 | 核心收获 |
|---|---|---|
| 0 | 总览 | 架构全景、7 个核心问题 |
| 1 | Agent 装配 | 5 样配料 + 15 个中间件 + ThreadState reducer + checkpointer/store + RunJournal |
| 2 | 多 Agent 调度 | task 工具 + SubagentExecutor + 隔离事件循环 + 状态契约 + Token 归集 |
| 3 | 长期记忆 | 无向量库方案 + 去抖队列 + LLM 抽取 + 置信度排序 + 贪心打包 |
| 4 | 上下文工程 | 静态 prompt + 摘要压缩 + 工具输出预算 + Schema 延迟加载 |
| 5 | Runtime | Checkpointer/Store/Events/Runs/StreamBridge/Journal |
| 6 | 安全与稳定性 | Guardrails + 循环检测 + 安全终止 + HITL + Todo 防提前收尾 |
| 7 | 周边系统 | 工具/MCP/Skills/Sandbox/Models |
| 8 | 对外接口 | FastAPI Gateway + SSE 流式 + Auth |

**最重要的 10 条 Agent 工程原则**（从全 8 章提炼）：

1. **静态 prompt + 动态 reminder 分离**——为 prefix cache；
2. **异步 + 轮询 + 协作式取消**——长任务的 SSE 友好模式；
3. **延迟注入**——尊重 "AIMessage(tool_calls) ↔ ToolMessage 配对"的硬约束；
4. **结构化契约代替字符串匹配**——status_contract 是经典案例；
5. **配置即 agent**——一个 graph + 配置路由，而非 N 个服务；
6. **没有向量库也能做长期记忆**——量级有限时，LLM 抽取 + 置信度排序更简单可控；
7. **隔离的事件循环**——既不阻塞主循环，又复用连接池；
8. **同步单例 + 异步 CM 两套 API**——CLI 和服务端各取所需；
9. **压缩前先抽取**——摘要不可逆，信息保全要在压缩前；
10. **fail-closed + 翻译成 agent 能懂的反馈**——guardrail 拒绝时返回 ToolMessage 而非抛异常。

把这些原则内化，你不仅能读懂 DeerFlow，也能在自己的 Agent 项目里做出更稳健的设计。

**Happy hacking! 🦌**
