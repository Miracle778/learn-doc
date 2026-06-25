# 第 5 章 精讲版：Runtime——Checkpointer、Store、Events、Runs（状态管理）

> 这一章讲 DeerFlow 怎么把 LangGraph 的运行时跑起来。
> 如果你用过 LangGraph Platform，会发现 DeerFlow 的 Runtime 几乎是它的**自托管平替**。
>
> **配套文件**（建议同时打开对照看）：
> - `runtime/checkpointer/provider.py` / `async_provider.py`
> - `runtime/store/provider.py`
> - `runtime/events/store/base.py` / `db.py` / `jsonl.py` / `memory.py`
> - `runtime/runs/worker.py` / `manager.py` / `schemas.py`
> - `runtime/stream_bridge/base.py` / `memory.py`
> - `runtime/journal.py`（第 1 章 1.10 节已详讲，这里补 runtime 视角）
> - `runtime/serialization.py` / `converters.py`
> - `runtime/user_context.py`

---

# 🎯 先建立心理模型：Runtime 是"剧场后勤"

如果把 Agent 比作一场舞台剧：

```
┌─────────────────────────────────────────────────┐
│  舞台（Agent 运行）                               │
│  演员（LLM）→ 道具（工具）→ 观众（用户）           │
└───────────────────────┬─────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│  后勤（Runtime）                                  │
│  ① 存档室（Checkpointer）— 保存每场演出的进度      │
│  ② 档案柜（Store）— 跨场次的长期资料               │
│  ③ 录像机（Events）— 把每场演出录下来              │
│  ④ 场记板（Runs）— 管理每场演出的状态              │
│  ⑤ 直播车（StreamBridge）— 实时推流给观众          │
│  ⑥ 速记员（RunJournal）— 记录每一步               │
└─────────────────────────────────────────────────┘
```

**关键认知**：Agent 本身（model + tools + middleware + prompt）是"剧本和演员"，Runtime 是"剧场后勤"——没有它，演出只能演一次，没法录像、没法续演、没法回放。

---

# 📖 5.1 Checkpointer vs Store：两个不同的持久化

这俩都是 LangGraph 的接口，DeerFlow 直接复用上游实现（不是自己重写）。

## 5.1.1 概念区分

| 概念 | 接口 | 存什么 | 比喻 |
|---|---|---|---|
| **Checkpointer** | `langgraph.types.Checkpointer` | 每个**超级步**的图状态（channels + 消息历史 + pending writes）——让 `graph.astream(...)` 能按 `thread_id` 恢复 | 游戏存档 |
| **Store** | `langgraph.store.base.BaseStore` | 跨线程长期 KV：`store.put(("threads",), thread_id, {...})`——线程列表、用户数据 | 档案柜 |

**一句话区分**：

- **Checkpointer 是"按 thread 的版本栈"**——每个超级步存一次，能回溯到任何中间状态；
- **Store 是"跨 thread 的扁平 KV"**——按 `(namespace_prefix, key)` 存，所有线程共享。

## 5.1.2 三种后端

**文件**：`runtime/checkpointer/provider.py:59-93`

```python
@contextlib.contextmanager
def _sync_checkpointer_cm(config: CheckpointerConfig) -> Iterator[Checkpointer]:
    if config.type == "memory":
        from langgraph.checkpoint.memory import InMemorySaver
        yield InMemorySaver()                # ← 内存版，重启就丢
        return

    if config.type == "sqlite":
        from langgraph.checkpoint.sqlite import SqliteSaver
        with SqliteSaver.from_conn_string(conn_str) as saver:
            saver.setup()
            yield saver                       # ← SQLite 文件持久化
        return

    if config.type == "postgres":
        from langgraph.checkpoint.postgres import PostgresSaver
        with PostgresSaver.from_conn_string(config.connection_string) as saver:
            saver.setup()
            yield saver                       # ← Postgres，生产级
        return
```

Store 的结构**一模一样**（`store/provider.py:51-96`），三种后端：`InMemoryStore` / `SqliteStore` / `PostgresStore`。

**关键细节**（`store/provider.py:52-58` 注释）：

> The `config` argument is a `CheckpointerConfig` instance — **the same object used by the checkpointer factory**.

**Checkpointer 和 Store 共用同一个配置**。必须用同一种后端——避免"checkpointer 在 sqlite，store 在 postgres"这种割裂配置。

## 5.1.3 配置

**文件**：`config/checkpointer_config.py:7-26`

```python
CheckpointerType = Literal["memory", "sqlite", "postgres"]

class CheckpointerConfig(BaseModel):
    type: CheckpointerType
    connection_string: str | None = None
```

就这么简单——一个 type + 一个 connection_string。

---

# 📖 5.2 同步单例 vs 异步上下文管理器：两套 API

这是 DeerFlow 给**每个**持久化 provider 都写的两套 API——一个值得学的工程模式。

## 5.2.1 为什么需要两套？

| 场景 | 模式 | 代表 |
|---|---|---|
| **CLI / 嵌入式 SDK**（无 FastAPI） | 全局单例 + 双重检查锁 | `get_checkpointer()` |
| **FastAPI 服务端** | async context manager，无全局状态 | `async with make_checkpointer(...)` |

**为什么服务端不用全局单例？** 因为 FastAPI 的 lifespan 要明确管理生命周期（启动建池、关闭拆池），全局单例做不到干净的 teardown。

## 5.2.2 同步单例——双重检查锁定

**文件**：`checkpointer/provider.py:107-145`

```python
_checkpointer: Checkpointer | None = None
_checkpointer_lock = threading.Lock()

def get_checkpointer() -> Checkpointer:
    if _checkpointer is not None:              # ★ 快速路径（无锁，99% 走这）
        return _checkpointer

    ensure_config_loaded()

    with _checkpointer_lock:                   # 加锁
        if _checkpointer is not None:          # ★ 二次检查（防并发重复创建）
            return _checkpointer

        config = get_checkpointer_config()
        checkpointer_ctx = _sync_checkpointer_cm(config)
        _checkpointer = checkpointer_ctx.__enter__()
        _checkpointer_ctx = checkpointer_ctx    # 存起来，reset 时能 __exit__
    return _checkpointer
```

**双重检查锁定（Double-Checked Locking）**：

1. 第一次检查**不加锁**（99% 的调用走快速路径，性能好）；
2. 第二次检查**加锁**（防止两个线程同时通过第一次检查后重复创建）。

**`reset_checkpointer()`**（148-162 行）调 `_checkpointer_ctx.__exit__()` 干净关闭——所以要把 ctx 存起来。

## 5.2.3 异步上下文管理器

**文件**：`checkpointer/async_provider.py:167-202`

```python
@contextlib.asynccontextmanager
async def make_checkpointer(app_config=None) -> AsyncIterator[Checkpointer]:
    async with ... as checkpointer:
        app.state.checkpointer = checkpointer   # 存到 app.state
        yield checkpointer                       # FastAPI lifespan 用
```

异步路径用 `AsyncSqliteSaver` / `AsyncPostgresSaver`，配 `psycopg_pool.AsyncConnectionPool` + TCP keepalive（`async_provider.py:50-67`）。

> 💡 **学习启示**：写一个既要给 CLI 用、又要给 Web 服务用的库时，准备两套 API 是常见做法。同步靠单例简化使用，异步靠 CM 保证生命周期清晰。

---

# 📖 5.3 Events vs Runs：三个不同的概念

很多人搞混这三个词。它们是**不同粒度**的东西：

## 5.3.1 RunRecord——一次 agent 调用的内存句柄

**文件**：`runs/manager.py:74-103`

```python
@dataclass
class RunRecord:
    run_id: str           # UUID4
    thread_id: str
    assistant_id: str
    status: RunStatus      # pending/running/success/error/timeout/interrupted
    task: asyncio.Task     # 后台协程
    abort_event: asyncio.Event
    abort_action: str      # "interrupt" 或 "rollback"
    # 聚合的 token 统计
    total_input_tokens: int
    lead_agent_tokens: int
    subagent_tokens: int
    middleware_tokens: int
    last_ai_message: ...
    first_human_message: ...
```

**RunRecord 是"场记板"**——记录这一场演出的元信息和状态。

## 5.3.2 RunStatus——6 种状态

**文件**：`runs/schemas.py:6-14`

```python
class RunStatus(StrEnum):
    pending = "pending"         # 刚创建
    running = "running"         # 执行中
    success = "success"         # 成功
    error = "error"             # 失败
    timeout = "timeout"         # 超时
    interrupted = "interrupted" # 被中断（HITL）
```

还有 `DisconnectMode`（`schemas.py:17-21`）：`cancel | continue`——客户端断连时怎么办。

## 5.3.3 RunEvent——单条持久化记录

**文件**：`runtime/events/store/base.py`

```python
# RunEvent 是个 dict，结构：
{
    "thread_id": "abc",
    "run_id": "run-456",
    "event_type": "llm.ai.response",   # 事件类型
    "category": "message",              # message / trace / error / middleware
    "content": {...},                   # 内容
    "metadata": {...},                  # 元数据
    "seq": 42,                          # ★ 线程内单调递增
    "created_at": "2026-06-25T10:00:00Z",
}
```

**`category` 字段区分**：

- **`message`**：前端可显示（human/ai/tool 消息）；
- **`trace`**：调试/审计（生命周期、错误）；
- **`middleware`**：中间件审计（如安全终止事件）。

**RunEventStore 的契约**（`base.py:17-109`）——5 个保证：

1. events 在 `put()` 后可检索；
2. `seq` 在 thread 范围内严格递增；
3. `list_messages()` 只返回 `category="message"`；
4. `list_events()` 返回该 run 的全部；
5. 返回的 dict 结构匹配 `RunEvent`。

---

# 📖 5.4 三种事件存储后端

**文件**：`runtime/events/store/__init__.py:5-23`

`make_run_event_store(config)` 按 `run_events.backend` 路由：

| 后端 | 文件 | 特点 |
|---|---|---|
| `memory` | `memory.py` | 进程内 dict，默认，重启就丢 |
| `db` | `db.py` | SQLAlchemy async ORM，写 `run_events` 表 |
| `jsonl` | `jsonl.py` | 每 run 一个 `.jsonl` 文件 |

## 5.4.1 db 后端——单调 seq + 用户隔离

**文件**：`runtime/events/store/db.py:25-315`

**单调 seq**（`db.py:92-112`）：

```python
# SQLite/其他：行级锁
SELECT max(seq) ... FOR UPDATE

# Postgres：咨询锁（因为聚合行没法行锁）
pg_advisory_xact_lock(hashtext(:thread_id)::bigint)
```

**为什么 Postgres 要用 advisory lock？** 因为 `SELECT max(seq)` 是聚合查询，不锁定具体行——两个并发事务可能读到相同的 max(seq)，然后都写 seq+1，造成重复。`pg_advisory_xact_lock` 按 thread_id 哈希加事务级锁，保证同一 thread 的写串行。

**JSON content 往返**（`db.py:31-60`）：非字符串 content 用 `json.dumps` 序列化，打 `metadata["content_is_json"]=True`；读时反序列化。trace content 截断到 `max_trace_content=10240` 字节。

**用户隔离的 `AUTO` 哨兵**（`db.py:74-90`）：

```python
def list_messages(..., user_id: str | None | _AutoSentinel = AUTO):
    ...
```

三态：

- `AUTO`：读 contextvar，未设置就报错；
- 显式 `str`：覆盖；
- 显式 `None`：不加 WHERE 子句（迁移/CLI 用）。

## 5.4.2 jsonl 后端——简单但单进程

**文件**：`runtime/events/store/jsonl.py:39-218`

- 每个 run 一个文件：`.deer-flow/threads/{thread_id}/runs/{run_id}.jsonl`；
- 进程内 seq 计数器；
- per-thread `asyncio.Lock` 串行写；
- 文件 I/O 卸载到 `asyncio.to_thread`。

**致命限制**（docstring 明确警告）：

> "Multi-process deployments sharing the same directory will produce duplicate or non-monotonic seq values. Use `DbRunEventStore` for multi-process."

**为什么？** 因为 seq 是进程内计数器 + `asyncio.Lock` 只锁单进程。两个进程同时写同一个 `.jsonl` 文件，seq 会重复或乱序，文件会交错损坏。

---

# 📖 5.5 `run_agent`——后台异步执行器

**文件**：`runtime/runs/worker.py:124-437`。`async def`，跑在 `asyncio.Task` 里。

这是 Runtime 的**核心函数**——把所有东西串起来。我们逐步看。

## 5.5.1 8 步流程

```python
async def run_agent(bridge, run_mgr, record, ctx, agent_factory, graph_input, config, stream_modes, ...):
    # 1. 建 RunJournal（作为 LangChain callback）
    journal = RunJournal(run_id=..., thread_id=..., event_store=..., 
                         progress_reporter=lambda snap: run_mgr.update_run_progress(...))
    
    # 2. 快照运行前 checkpoint（用于回滚）
    checkpoint = await checkpointer.aget_tuple(("thread_id",))
    pre_run_snapshot = deepcopy(checkpoint)
    
    # 3. 发 metadata 事件（带 run_id + thread_id）
    bridge.publish(run_id, "metadata", {...})
    
    # 4. 构造 agent + 装入 Runtime
    agent = agent_factory(config)      # ← make_lead_agent(config)
    runtime = Runtime(context={thread_id, run_id, app_config, "__run_journal": journal}, store=store)
    config["configurable"]["__pregel_runtime"] = runtime
    config["callbacks"].append(journal)   # journal 也作为 callback
    
    # 5. 挂 checkpointer 和 store 到图
    agent.checkpointer = checkpointer
    agent.store = store
    
    # 6. 流式跑
    async for chunk in agent.astream(graph_input, config=config, stream_mode=stream_modes):
        serialized = serialize(chunk, mode=mode)
        bridge.publish(run_id, mode, serialized)    # 推给 StreamBridge → SSE
    
    # 7. 终态分支
    if abort_event.is_set():
        if abort_action == "rollback":
            await _rollback_to_pre_run_checkpoint(...)   # 回滚
            status = RunStatus.error
        else:
            status = RunStatus.interrupted
    elif llm_error_detected:
        status = RunStatus.error
    else:
        status = RunStatus.success
    
    # 8. finally 收尾
    await journal.flush()                     # 刷干净 buffer
    run_mgr.update_run(record.run_id, status=status, ...)
    # 同步 title
    title = checkpoint.get("channel_values", {}).get("title")
    thread_store.update_display_name(thread_id, title)
    bridge.publish_end(run_id)                # 发 end 信号
    # 60s 后清理
```

## 5.5.2 回滚机制

**文件**：`worker.py:456-546`

当 `multitask_strategy="rollback"` 时，用户在 run 跑到一半时发起冲突 run：

1. 把当前 run 标 `error`；
2. **把状态回滚到这次 run 开始前**；
3. 再跑新的。

`_rollback_to_pre_run_checkpoint()` 的做法：把快照作为新 checkpoint **重放**，然后按 task_id 分组**重新应用 pending writes**。

**为什么不直接"删 checkpoint"？** 因为别的 run 可能引用了它；只能**追加**一个"回到过去"的新 checkpoint。

## 5.5.3 RunManager 的并发控制

**文件**：`runs/manager.py:106-738`

**`create_or_reject`**（497-579 行）——原子 check-then-insert，消除 TOCTOU 竞态：

```python
# multitask_strategy:
#   reject    → 直接拒绝新请求（409）
#   interrupt  → 中断旧 run（interrupted），再跑新的
#   rollback   → 标 error + 回滚到旧 run 开始前，再跑新的
```

**`reconcile_orphaned_inflight_runs`**（581-633 行）——重启后把"持久化但无本地 task"的 run 标记为 error。

**`shutdown`**（648-738 行）——drain 在飞 run 再关 checkpointer 池（修过 #3373 那种 langgraph 内部 `_checkpointer_put_after_previous` 与关池的竞态）。

**重试策略**（139-167 行）：所有 store 写过 `_call_store_with_retry`，`max_attempts=5, initial_delay=0.05, backoff=2, max_delay=1.0`，匹配 `"database is locked"`、`SQLITE_BUSY` 等可重试错误。

---

# 📖 5.6 StreamBridge——生产者/消费者解耦

**文件**：`runtime/stream_bridge/base.py`

## 5.6.1 契约

`StreamBridge` 抽象 worker（生产者）↔ SSE 端点（消费者）的通道，模仿 LangGraph Platform 的 Queue + StreamManager 分离。

```python
class StreamBridge(ABC):
    def publish(self, run_id: str, event: str, data: Any) -> None: ...
    def publish_end(self, run_id: str) -> None: ...
    async def subscribe(self, run_id: str, last_event_id: str | None = None) -> AsyncIterator[StreamEvent]: ...
    async def cleanup(self) -> None: ...
    async def close(self) -> None: ...
```

**两个哨兵**：

- `HEARTBEAT_SENTINEL`：空闲超时发（保活）；
- `END_SENTINEL`：`publish_end` 后发一次（告诉消费者"结束了"）。

**`StreamEvent`** 是个 frozen dataclass `{id, event, data}`，`id` 支持 SSE `Last-Event-ID` 重连。

## 5.6.2 MemoryStreamBridge

**文件**：`stream_bridge/memory.py:25-133`

唯一实现（Redis 是 `NotImplementedError`）。

**per-run `_RunStream`** 持有：

- 事件列表（cap `queue_maxsize=256`）；
- `asyncio.Condition`（通知消费者）；
- `ended` flag；
- `start_offset`（旧事件驱逐后前进）。

**订阅者落后于保留窗口怎么办？** 从 `start_offset` 恢复——可能丢一些旧事件，但能继续。

**关键设计**：worker 和 SSE 端点解耦——worker 往 bridge 推，SSE 端点从 bridge 拉。即使 SSE 端点断连重连，worker 不受影响。

---

# 📖 5.7 RunJournal——Runtime 视角补充

第 1 章 1.10 节已详讲 RunJournal 的内部机制。这里补充 **Runtime 视角**——它在 `run_agent` 里的角色。

## 5.7.1 两个挂载点

```python
# worker.py:171-235
journal = RunJournal(...)

# ① 作为 LangChain callback（自动捕获 LLM/工具调用）
config["callbacks"].append(journal)

# ② 作为 Runtime context（中间件手动写事件）
Runtime(context={..., "__run_journal": journal})
```

**为什么要两个挂载点？**

- **`config["callbacks"]`**：LangChain 自动在 LLM/工具调用时触发 journal 的 `on_llm_end` / `on_tool_end` 等——**自动捕获**；
- **`Runtime.context["__run_journal"]`**：某些中间件（如 `SafetyFinishReasonMiddleware`）不经过 LangChain 回调，需要**手动调** `journal.record_middleware(...)` 写事件。

## 5.7.2 节流的进度上报

RunJournal 的 `progress_reporter` lambda 调 `run_manager.update_run_progress(...)`，**5 秒节流**一次——避免高频写数据库。

## 5.7.3 兜底刷写

`worker.py` 的 `finally` 块调 `await journal.flush()`——保证 run 结束时 buffer 一定被刷干净，不丢事件。

---

# 📖 5.8 Serialization——序列化

**文件**：`runtime/serialization.py`

## 5.8.1 统一序列化器

`serialize_lc_object`（16-42 行）——递归序列化 LangChain 对象：

```python
def serialize_lc_object(obj):
    if isinstance(obj, dict):     # 递归 dict
        return {k: serialize_lc_object(v) for k, v in obj.items()}
    if isinstance(obj, (list, tuple)):  # 递归 list/tuple
        return [serialize_lc_object(item) for item in obj]
    if hasattr(obj, "model_dump"):       # Pydantic v2
        return obj.model_dump()
    if hasattr(obj, "dict"):             # Pydantic v1
        return obj.dict()
    return str(obj)                       # 兜底
```

**单一真源**——`worker.py`（SSE）和 `app/gateway/routers/threads`（REST）都用它，保证序列化一致。

## 5.8.2 channel_values 清理

`serialize_channel_values`（45-56 行）剥离内部 LangGraph key：

```python
def serialize_channel_values(values):
    return {
        k: serialize_lc_object(v)
        for k, v in values.items()
        if not k.startswith("__pregel_") and k != "__interrupt__"
    }
```

`__pregel_*` 和 `__interrupt__` 是 LangGraph 内部 key，不该暴露给前端。

## 5.8.3 converters——LangChain → OpenAI 格式

**文件**：`runtime/converters.py`

`langchain_to_openai_message`（21-71 行）把 LangChain 消息转成 OpenAI wire format：

- `HumanMessage` → `{"role": "user", ...}`；
- `AIMessage`（有 tool_calls）→ `{"role": "assistant", "tool_calls": [...]}`；
- `ToolMessage` → `{"role": "tool", "tool_call_id": ..., "content": ...}`。

**注意**：这个 converter 目前**没有**接入 journal（journal 直接用 `message.model_dump()`）。它存在是为了兼容需要 OpenAI 格式的消费者。

---

# 📖 5.9 user_context——用户隔离

**文件**：`runtime/user_context.py`

## 5.9.1 AUTO 哨兵

```python
_AutoSentinel = ...  # 特殊类型
AUTO = _AutoSentinel()

def resolve_user_id(user_id: str | None | _AutoSentinel = AUTO) -> str | None:
    if isinstance(user_id, _AutoSentinel):
        # 读 contextvar，未设置就报错
        ...
    return user_id  # 显式传的 str 或 None
```

三态语义：

| 传入值 | 含义 | 用途 |
|---|---|---|
| `AUTO` | 读 contextvar，未设置报错 | HTTP 请求（auth 中间件设了 contextvar） |
| 显式 `str` | 覆盖 | 测试、指定用户 |
| 显式 `None` | 不加 WHERE 子句 | 迁移、CLI、后台任务 |

## 5.9.2 跨边界传播

```python
def resolve_runtime_user_id(runtime) -> str | None:
    # 先查 runtime.context["user_id"]（后台任务可能设了）
    # 再查 contextvar
    # 最后用 DEFAULT_USER_ID
```

**为什么先查 runtime.context？** 因为后台任务（如 subagent）跑在隔离线程/循环里，contextvar 可能丢失。`runtime.context` 是显式传递的，更可靠。

---

# 🎯 现在回头检验：你掌握了吗？

### 问题 1：Checkpointer 和 Store 有什么区别？为什么不用一个？

<details>
<summary>🔍 点击看答案</summary>

- **Checkpointer** 存"图执行的进度"——每个超级步的 channels、消息、pending writes。结构是按 `(thread_id, checkpoint_id)` 的**版本栈**，让 `graph.astream(thread_id=X)` 能从上次中断处恢复；
- **Store** 存"跨线程的长期数据"——按 `(namespace_prefix, key)` 的扁平 KV，如线程列表。

**为什么分开？** 访问模式不同：checkpointer 是按线程按步频繁读写（每次 model call 都写）；store 是偶尔写、经常扫。分开后能各自优化索引。

</details>

### 问题 2：多进程部署为什么不能用 jsonl 事件存储？

<details>
<summary>🔍 点击看答案</summary>

因为 jsonl 的 **seq 是进程内计数器**，`asyncio.Lock` 也只锁单进程。两个进程同时往同一个 `{run_id}.jsonl` 追加，会出现：

- seq 重复或乱序；
- 文件交错损坏。

`db` 后端用 `pg_advisory_xact_lock`（Postgres）或 `SELECT max(seq) FOR UPDATE`（SQLite）保证跨进程单调，是多进程唯一正确选择。

</details>

### 问题 3：RunJournal 为什么"故意不实现 on_llm_new_token"？

<details>
<summary>🔍 点击看答案</summary>

因为 `on_llm_new_token` 一个 token 触发一次，一次 LLM 调用可能触发上千次。如果每次都写事件存储，会**洪水般刷盘**。

DeerFlow 的取舍：**流式 token 走 SSE 直推前端**（通过 `graph.astream(stream_mode="messages")`），**事件存储只记完整消息**（`on_llm_end`）。这样事件存储写入频率可控，流式体验靠 SSE 实时通道，互不干扰。

</details>

### 问题 4：`multitask_strategy` 的三种策略有什么区别？

<details>
<summary>🔍 点击看答案</summary>

当同一 thread 已有 run 在跑，又来一个 run 请求时：

- **`reject`**：直接拒绝新请求（409）；
- **`interrupt`**：中断旧 run（设 interrupted），再跑新的；
- **`rollback`**：把旧 run 标 error 并**回滚到它开始前的 checkpoint**，再跑新的。

`create_or_reject` 用原子 check-then-insert 消除 TOCTOU 竞态。

</details>

### 问题 5：StreamBridge 和 RunJournal 有什么区别？

<details>
<summary>🔍 点击看答案</summary>

- **StreamBridge** 是**直播**——实时推流给前端，推完就没了（除非前端自己存）；
- **RunJournal** 是**录像**——持久化记录到 RunEventStore，事后能查。

StreamBridge 解决"实时性"（前端立刻看到），RunJournal 解决"可追溯性"（事后能审计/调试/计费）。

</details>

---

# 📝 一页纸总结（第 5 章精华）

```
┌──────────────────────────────────────────────────────────────┐
│  Runtime = 剧场后勤                                           │
│  让 agent 能续演、能录像、能回放、能实时直播                    │
└──────────────────────────────────────────────────────────────┘
        │
        ├─▶ Checkpointer（游戏存档）
        │     • 按 thread 存图状态（超级步版本栈）
        │     • memory/sqlite/postgres 三后端
        │     • 同步单例 + 异步 CM 两套 API
        │
        ├─▶ Store（档案柜）
        │     • 跨 thread 存长期 KV（线程列表等）
        │     • 共用 CheckpointerConfig
        │
        ├─▶ Events（录像）
        │     • RunEvent = 单条持久化记录（category: message/trace/middleware）
        │     • seq 线程内单调递增
        │     • memory/db/jsonl 三后端（db 支持多进程）
        │     • AUTO 哨兵做用户隔离
        │
        ├─▶ Runs（场记板）
        │     • RunRecord = 一次 agent 调用的内存句柄
        │     • RunStatus: pending/running/success/error/timeout/interrupted
        │     • run_agent: 8 步流程（建 journal → 快照 → 造 agent → 挂 cp/store → astream → 终态分支 → 收尾）
        │     • multitask_strategy: reject/interrupt/rollback
        │     • 重试策略 + 孤儿 run 回收
        │
        ├─▶ StreamBridge（直播车）
        │     • 生产者(worker) ↔ 消费者(SSE) 解耦
        │     • HEARTBEAT/END 哨兵
        │     • Last-Event-ID 重连
        │
        ├─▶ RunJournal（速记员）
        │     • LangChain callback + Runtime context 双挂载
        │     • 同步入 buffer → 攒 20 条异步批量刷 → 失败回退重试
        │     • 按 caller 分桶统计 token
        │
        └─▶ Serialization + user_context
              • serialize_lc_object 统一序列化
              • AUTO 哨兵三态用户隔离
```

**5 句话记住**：

1. **Checkpointer 是存档（按 thread），Store 是档案柜（跨 thread）**——共用配置，三后端，两套 API；
2. **RunEvent 是录像**——seq 单调递增，category 分 message/trace/middleware，db 后端支持多进程；
3. **run_agent 是 8 步流程**——建 journal → 快照 → 造 agent → 挂 cp/store → astream → 终态 → 收尾；
4. **StreamBridge 是直播，RunJournal 是录像**——一个实时推流，一个持久化记录；
5. **multitask_strategy 三策略**——reject（拒绝）/ interrupt（中断）/ rollback（回滚到 run 前）。
