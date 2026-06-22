# 第 2 章 精讲版：多 Agent 调度——Lead Agent 与 Subagents

> 这是 DeerFlow 最有意思、也最复杂的一章。
>
> **阅读方法**：和第 0-1 章一样，用"对话式"的方式，**一段一段**过代码。
>
> **配套文件**（建议同时打开对照看）：
> - `tools/builtins/task_tool.py`（委派工具的全流程）
> - `subagents/executor.py`（子 agent 执行引擎）
> - `subagents/config.py`（子 agent 配置）
> - `subagents/registry.py`（子 agent 注册）
> - `subagents/status_contract.py`（状态契约）
> - `subagents/token_collector.py`（token 归集）
> - `subagents/builtins/general_purpose.py` / `bash_agent.py`（内置子 agent）
> - `agents/middlewares/subagent_limit_middleware.py`（并行限制）

---

# 🎯 先建立心理模型：这不是"多个独立服务"

很多人听到"多 Agent"会以为是"多个独立服务通过消息总线通信"。DeerFlow **不是**这种。

DeerFlow 的多 Agent 是这样的：

```
                    用户发消息
                        │
                        ▼
              ┌─────────────────┐
              │  Lead Agent     │  ← 一个 LLM（项目经理）
              │  （主 agent）    │     会思考、会用工具
              └───────┬─────────┘
                      │
           ┌──────────┼──────────┐
           │          │          │
     自己直接   调 task 工具    自己直接
     用工具      派给子 agent    回答用户
           │          │          │
           ▼          ▼          ▼
      ┌────────┐ ┌──────────┐ ┌───────┐
      │bash 等 │ │Subagent  │ │ 文字  │
      │工具     │ │Executor  │ │ 回复  │
      └────────┘ │（跑子agent）│ └───────┘
                 └────┬─────┘
                      │
                 ┌────┴────┐
                 │ 子 agent │  ← 另一个 LLM（员工）
                 │（独立图）│     能力受限、专注某领域
                 └─────────┘
```

**三个关键认知**：

1. **只有一个入口**：用户只跟 Lead Agent 对话，不知道 Subagent 的存在；
2. **子 agent 是临时工**：Lead Agent 通过 `task` 工具"雇"一个子 agent 干活，干完就散；
3. **子 agent 是独立图**：它有自己的 LLM、自己的工具集、自己的对话历史，跑在一个**隔离的事件循环**里。

**Lead Agent 怎么决定"自己干"还是"派人干"？** 靠 prompt 引导 + 工具存在性（不是硬编码分支）。我们一步步看。

---

# 📖 2.1 三个工具，各司其职

整个多 Agent 系统围绕三个工具展开：

| 工具 | 文件 | 干什么 | 谁能用 |
|---|---|---|---|
| **`task`** | `tools/builtins/task_tool.py` | 把子任务委派给子 agent | 仅 `subagent_enabled=True` 的 lead agent |
| `setup_agent` | `tools/builtins/setup_agent_tool.py` | 首次创建一个自定义 agent | 仅 bootstrap agent |
| `update_agent` | `tools/builtins/update_agent_tool.py` | 修改已存在的自定义 agent | 仅 custom agent |

这章主要讲 `task`，另外两个在第 1 章的"三种装配"里提过，这里不重复。

---

# 📖 2.2 `task` 工具——多 Agent 的核心入口

这是整个章节最重要的代码。我们一段一段读。

## 2.2.1 工具签名和 docstring（task_tool.py:186-228）

```python
@tool("task", parse_docstring=True)
async def task_tool(
    runtime: Runtime,
    description: str,
    prompt: str,
    subagent_type: str,
    tool_call_id: Annotated[str, InjectedToolCallId],
) -> str:
    """Delegate a task to a specialized subagent that runs in its own context.

    Subagents help you:
    - Preserve context by keeping exploration and implementation separate
    - Handle complex multi-step tasks autonomously
    - Execute commands or operations in isolated contexts

    Built-in subagent types:
    - **general-purpose**: A capable agent for complex, multi-step tasks...
    - **bash**: Command execution specialist for running bash commands...

    When to use this tool:
    - Complex tasks requiring multiple steps or tools
    - Tasks that produce verbose output
    - When you want to isolate context from the main conversation
    - Parallel research or exploration tasks

    When NOT to use this tool:
    - Simple, single-step operations (use tools directly)
    - Tasks requiring user interaction or clarification
    """
```

**逐行解释**：

| 部分 | 含义 |
|---|---|
| `@tool("task", parse_docstring=True)` | LangChain 装饰器，把函数注册为名为 `task` 的工具；`parse_docstring=True` 表示用 docstring 生成工具描述发给模型 |
| `runtime: Runtime` | LangGraph 的运行时对象（`InjectedToolArg`），工具函数靠它访问图的状态和上下文——**不是模型填的参数** |
| `description: str` | 3-5 个词的简短描述（给日志/前端显示用） |
| `prompt: str` | 给子 agent 的详细任务指令 |
| `subagent_type: str` | 子 agent 类型（`general-purpose` / `bash` / 自定义） |
| `tool_call_id` | LangGraph 自动注入的工具调用 ID（不来自模型） |

**注意 docstring 里的 "When to use / When NOT to use"**——这是**给模型看的决策指南**。模型读完这段就知道什么时候该派子 agent、什么时候不该。这是 prompt engineering 的一部分。

## 2.2.2 校验和配置解析（task_tool.py:229-278）

```python
    runtime_app_config = _get_runtime_app_config(runtime)
    available_subagent_names = get_available_subagent_names(app_config=runtime_app_config)

    # 校验 subagent_type 是否可用
    config = get_subagent_config(subagent_type, app_config=runtime_app_config)
    if config is None:
        available = ", ".join(available_subagent_names)
        return f"Error: Unknown subagent type '{subagent_type}'. Available: {available}"
    
    # bash 子 agent 的额外校验
    if subagent_type == "bash":
        host_bash_allowed = is_host_bash_allowed(runtime_app_config)
        if not host_bash_allowed:
            return f"Error: {LOCAL_BASH_SUBAGENT_DISABLED_MESSAGE}"
```

**这段在干嘛**：

1. 拿到运行时的 app config；
2. 校验模型指定的 `subagent_type` 是否存在（`get_subagent_config`）——不存在就返回错误 + 列出可用的类型；
3. 如果是 `bash` 子 agent，额外校验 host bash 是否被允许——不允许就拒绝（安全考虑）。

**注意错误返回格式**：返回的是字符串，会作为 ToolMessage 的 content 返回给模型。模型看到 `"Error: Unknown subagent type..."` 就知道要换一个类型。

## 2.2.3 提取父上下文（task_tool.py:250-278）

```python
    # Extract parent context from runtime
    sandbox_state = None
    thread_data = None
    thread_id = None
    parent_model = None
    trace_id = None
    metadata: dict = {}

    if runtime is not None:
        sandbox_state = runtime.state.get("sandbox")        # ★ 沙箱状态
        thread_data = runtime.state.get("thread_data")      # ★ 工作目录
        thread_id = runtime.context.get("thread_id")        # ★ 线程 ID
        metadata = runtime.config.get("metadata", {})
        parent_model = metadata.get("model_name")           # ★ 父 agent 的模型名
        trace_id = metadata.get("trace_id") or str(uuid.uuid4())[:8]  # ★ trace ID
```

**这段极其关键**——它在提取**父 agent 的上下文**，准备传给子 agent。为什么？

因为子 agent 需要**继承**父 agent 的：

| 继承的东西 | 为什么 |
|---|---|
| `sandbox_state` | 子 agent 要用同一个沙箱（同一个文件系统） |
| `thread_data` | 子 agent 要在同一个工作目录操作 |
| `thread_id` | 日志、事件要归属同一个线程 |
| `parent_model` | 子 agent 可能"继承"父的模型（config.model="inherit"） |
| `trace_id` | 分布式追踪要把父子链起来 |

**如果不继承会怎样？** 子 agent 拿不到沙箱，没法读写文件；或者读写到一个完全不同的目录；或者日志查不到它属于哪个线程。

> 💡 **学习启示**：多 agent 系统设计的一个核心问题是**"上下文继承"**——子 agent 需要继承父 agent 的哪些状态？DeerFlow 的答案是：沙箱、工作目录、线程 ID、模型、trace。这是个值得抄的清单。

## 2.2.4 合并 skill 白名单（task_tool.py:272-278）

```python
    parent_available_skills = metadata.get("available_skills")
    if parent_available_skills is not None:
        overrides["skills"] = _merge_skill_allowlists(list(parent_available_skills), config.skills)
```

**这段在干嘛**：子 agent 的 skill 白名单 = **父 agent 的 skill 白名单 ∩ 子 agent 自己配置的 skill 白名单**（取交集）。

**为什么取交集？** 因为安全——子 agent 不能比父 agent 拥有**更多**权限。如果父 agent 没有"docx" skill 的权限，子 agent 也不能有。`_merge_skill_allowlists`（task_tool.py:175-183）做的就是这件事：

```python
def _merge_skill_allowlists(parent: list[str] | None, child: list[str] | None) -> list[str] | None:
    if parent is None:
        return child       # 父没限制 → 用子的
    if child is None:
        return parent      # 子没限制 → 用父的
    parent_set = set(parent)
    return [skill for skill in child if skill in parent_set]   # 都有限制 → 取交集
```

## 2.2.5 创建 executor 并异步启动（task_tool.py:280-316）

```python
    # 获取子 agent 的工具（subagent_enabled=False 防递归！）
    tools = get_available_tools(
        model_name=effective_model,
        groups=parent_tool_groups,
        subagent_enabled=False,           # ★ 子 agent 不能再派子 agent
    )

    # 创建 executor
    executor = SubagentExecutor(
        config=config,                    # 子 agent 配置
        tools=tools,                      # 子 agent 的工具集
        parent_model=parent_model,
        sandbox_state=sandbox_state,
        thread_data=thread_data,
        thread_id=thread_id,
        trace_id=trace_id,
    )

    # ★ 异步启动（不阻塞！）
    task_id = executor.execute_async(prompt, task_id=tool_call_id)
```

**两个关键点**：

### ① `subagent_enabled=False`——防递归

子 agent 获取工具时传 `subagent_enabled=False`，意味着子 agent 的工具集里**没有 `task` 工具**。这样子 agent 就不能再派"孙子 agent"。

为什么不允许多层嵌套？因为：

- **token 失控**：每层都有自己的 LLM 调用，3 层嵌套就是 3 倍 token；
- **调试噩梦**：祖孙三层的 trace 极其难看；
- **状态传递复杂**：每层都要装 collector、回传 token。

DeerFlow 的态度是"**默认扁平，最多一层**"——值得借鉴的工程取舍。

### ② `execute_async`——异步启动，不阻塞

注意这里是 `execute_async`，不是 `execute`。子 agent 在**后台**跑，`task_tool` 自己不阻塞，而是进入**轮询循环**（下一节讲）。

**为什么不阻塞等它跑完？** 因为子 agent 可能跑 15 分钟，如果 `task_tool` 阻塞等它：

- 整个 LangGraph 图就卡住了；
- SSE 没法实时推进度（用户看到"卡住了"）；
- 用户没法中途取消。

所以 DeerFlow 的设计是：**异步启动 + 轮询 + SSE 实时推进度 + 协作式取消**。

## 2.2.6 轮询循环——task_tool 的核心（task_tool.py:318-420）

这是 `task_tool` 最精彩的部分：

```python
    # 轮询超时：执行超时 + 60s 缓冲，每 5s 检查一次
    max_poll_count = (config.timeout_seconds + 60) // 5

    writer = get_stream_writer()
    # 发 task_started 事件
    writer({"type": "task_started", "task_id": task_id, "description": description})

    try:
        while True:
            result = get_background_task_result(task_id)    # 每 5s 查一次后台结果

            # 检查是否有新的 AI 消息 → 推 task_running 事件
            ai_messages = result.ai_messages or []
            current_message_count = len(ai_messages)
            if current_message_count > last_message_count:
                for i in range(last_message_count, current_message_count):
                    message = ai_messages[i]
                    writer({                                # ★ SSE 推给前端
                        "type": "task_running",
                        "task_id": task_id,
                        "message": message,
                    })
                last_message_count = current_message_count

            # 检查是否终态
            if result.status.is_terminal:
                break                                       # 跑完了，退出循环

            await asyncio.sleep(5)                          # 等 5s 再查
```

**这个循环在干三件事**：

1. **每 5 秒查一次**子 agent 的执行状态（`get_background_task_result`）；
2. **有新 AI 消息就推 SSE**（`task_running` 事件）——前端实时看到子 agent 的"思考过程"；
3. **终态就退出**（completed / failed / cancelled / timed_out）。

**为什么 5 秒？** 平衡实时性和性能。太频繁（比如 0.1s）会浪费 CPU；太慢（比如 60s）用户体验差。5 秒是个合理的折中。

## 2.2.7 终态处理——五种返回字符串（task_tool.py:367-415）

```python
            if result.status.is_terminal:
                # 清理后台任务
                cleanup_background_task(task_id)

                # 上报子 agent 的 token（如果还没上报过）
                if not result.usage_reported:
                    _report_subagent_usage(runtime, result)
                    result.usage_reported = True

                # 根据状态返回不同的字符串给模型
                if result.status == SubagentStatus.COMPLETED:
                    writer({"type": "task_completed", ...})
                    return f"Task Succeeded. Result: {result.result}"

                elif result.status == SubagentStatus.FAILED:
                    writer({"type": "task_failed", ...})
                    return f"Task failed. Error: {result.error}"

                elif result.status == SubagentStatus.CANCELLED:
                    writer({"type": "task_cancelled", ...})
                    return "Task cancelled by user."

                elif result.status == SubagentStatus.TIMED_OUT:
                    writer({"type": "task_timed_out", ...})
                    return f"Task timed out. Error: {result.error}"
```

**注意返回值**：返回的是**带前缀的字符串**，会作为 ToolMessage 的 content 返回给 lead agent。lead agent 读到 `"Task Succeeded. Result: ..."` 就知道子任务成功了。

**这些前缀不是随便写的**——它们是**状态契约**的一部分（2.5 节详讲），前端靠这些前缀识别子 agent 的最终状态。

**`_report_subagent_usage`** 把子 agent 的 token 记录上报给父 agent 的 RunJournal（第 1 章 1.10 节讲过）。`usage_reported` 标志保证**只上报一次**（防重复）。

---

# 📖 2.3 SubagentExecutor——子 agent 执行引擎

`task_tool` 是"派活"的，`SubagentExecutor` 才是"干活"的。这是整个多 agent 系统最精巧的部分。

## 2.3.1 状态机（executor.py:47-64）

```python
class SubagentStatus(Enum):
    PENDING   = "pending"    # 刚入队
    RUNNING   = "running"    # 执行中
    COMPLETED = "completed"  # 成功
    FAILED    = "failed"     # 失败
    CANCELLED = "cancelled"  # 被用户取消
    TIMED_OUT = "timed_out"  # 超时

    @property
    def is_terminal(self) -> bool:
        return self in {COMPLETED, FAILED, CANCELLED, TIMED_OUT}
```

**6 个状态，4 个是终态**。`is_terminal` 属性用于判断"是否该退出轮询循环"。

**为什么这么分？** 因为子 agent 有三种正常结束方式（成功/失败/超时）和一种外部干预（取消）。每种需要不同的返回字符串和前端处理。

## 2.3.2 SubagentResult——结果容器（executor.py:67-133）

```python
@dataclass
class SubagentResult:
    task_id: str
    trace_id: str
    status: SubagentStatus
    result: str | None = None
    error: str | None = None
    started_at: datetime | None = None
    completed_at: datetime | None = None
    ai_messages: list[dict[str, Any]] | None = None       # 子 agent 产生的每条 AI 消息
    token_usage_records: list[dict[str, int | str]] = ...  # token 记录
    usage_reported: bool = False                            # 是否已上报 token
    cancel_event: threading.Event = ...                    # ★ 协作式取消的信号灯
    _state_lock: threading.Lock = ...                      # ★ 状态锁
```

**三个字段值得特别关注**：

### ① `ai_messages`——子 agent 的思考过程

子 agent 跑的时候，每产生一条 AI 消息（比如"我打算先 ls 看看"），就被追加到 `ai_messages`。`task_tool` 的轮询循环拿这些消息推 `task_running` SSE 事件——**这就是前端看到"子 agent 正在思考..."的数据源**。

### ② `cancel_event`——协作式取消的信号灯

```python
cancel_event: threading.Event = field(default_factory=threading.Event, repr=False)
```

这是个 `threading.Event`（线程事件）。用户点"取消"时，`request_cancel_background_task(task_id)` 会调 `result.cancel_event.set()`。子 agent 的执行循环在每个 chunk 边界检查这个 event——**命中就 break**。

**为什么叫"协作式"？** 因为子 agent 不是被"杀掉"（强制终止），而是被"通知"（设了 flag），由子 agent **自己决定**何时退出。好处是不会留下半途的资源（比如打开的文件没关）；坏处是**长工具调用内部不会被中断**（比如子 agent 正在跑一个 5 分钟的 bash 命令，只能等它跑完才能取消）。

### ③ `try_set_terminal`——第一个终态转换赢（executor.py:100-133）

```python
    def try_set_terminal(self, status, *, result=None, error=None, ...) -> bool:
        """Set a terminal status exactly once.

        Background timeout/cancellation and the execution worker can race on the
        same result holder.  The first terminal transition wins; late terminal
        writes must not change status or payload fields.
        """
        if not status.is_terminal:
            raise ValueError(f"Status {status} is not terminal")

        with self._state_lock:                    # ★ 加锁
            if self.status.is_terminal:           # ★ 已经是终态了
                return False                       #    返回 False，不覆盖

            self.result = result
            self.error = error
            self.ai_messages = ai_messages
            self.token_usage_records = token_usage_records
            self.completed_at = completed_at or datetime.now()
            self.status = status
            return True                            # ★ 第一个调用赢，返回 True
```

**为什么要加锁？** 因为终态可能**同时**被多个源触发：

```
时间线：
  T=0     超时定时器到期 → 想设 TIMED_OUT
  T=0.001 worker 正常完成 → 想设 COMPLETED
  T=0.002 用户点了取消 → 想设 CANCELLED

三个几乎同时发生！
```

如果不加锁，可能出现"先 COMPLETED 又被 TIMED_OUT 覆盖"的诡异状态。`_state_lock` + `is_terminal` 检查保证**第一个调用赢**，后续调用返回 `False`。

> 💡 **学习启示**：处理并发状态机，"**第一个终态转换赢**"（first-write-wins）是个标准模式。用锁保护 + 检查标志位实现。

## 2.3.3 三种执行入口（executor.py:677-818）

`SubagentExecutor` 有三个入口方法，对应不同场景：

```
execute()                         ← 通用入口
  ├─ 检测当前是否在事件循环里
  │   ├─ 在 → 转给 _execute_in_isolated_loop
  │   └─ 不在 → asyncio.run(_aexecute)

execute_async()                   ← lead agent 走的就是这个（task_tool 调的）
  └─ submit 到 _scheduler_pool（ThreadPoolExecutor, max=3）
       └─ 池里再 submit 到隔离循环

_execute_in_isolated_loop()       ← "隔离事件循环"执行
  └─ 用 _get_isolated_subagent_loop() 的常驻循环
       └─ 跑在 "subagent-persistent-loop" daemon 线程上
       └─ copy_context() 传播 ContextVar
```

### 为什么要"隔离的事件循环"？（核心设计）

这是整个 SubagentExecutor 最精巧、也最容易被忽视的设计。

**问题**：lead agent 自己就在一个事件循环里跑（FastAPI 的）。如果子 agent 在**同一个循环**里 `await`，会**阻塞** lead agent 的 SSE 推送——前端看到的"卡住"。

**朴素的解法**：为每次子 agent 调用**新建一个事件循环**（`asyncio.new_event_loop()`），跑完关掉。

**为什么朴素解法不行？** 因为新建循环 + 关闭循环，会**破坏共享的 httpx 连接池**。LangChain 的 async 客户端用了一个全局 httpx 连接池（复用 TCP 连接、TLS 握手）。如果你在一个循环里创建了连接、在另一个循环里用，或者关闭了创建连接的循环，连接就废了——报 `RuntimeError: Event loop is closed` 或连接泄漏。

**DeerFlow 的解法**：用一个**常驻的隔离循环**（`_get_isolated_subagent_loop`，executor.py:202-230）：

```python
_isolated_subagent_loop: asyncio.AbstractEventLoop | None = None

def _get_isolated_subagent_loop() -> asyncio.AbstractEventLoop:
    global _isolated_subagent_loop
    if _isolated_subagent_loop is not None:
        return _isolated_subagent_loop              # 复用

    _isolated_subagent_loop = asyncio.new_event_loop()
    thread = threading.Thread(
        target=_run_isolated_loop_forever,
        name="subagent-persistent-loop",
        daemon=True,                                 # daemon：进程退出时自动结束
    )
    thread.start()
    return _isolated_subagent_loop
```

**特点**：

1. **进程级单例**：整个进程只有一个隔离循环，所有子 agent 调用复用它；
2. **跑在 daemon 线程上**：不阻塞主线程（lead agent 的循环）；
3. **永不关闭**：直到进程退出（daemon 线程自动结束）——所以连接池不会被破坏。

**`copy_context()` 传播 ContextVar**（executor.py:226）：

```python
ctx = copy_context()    # 复制当前线程的 ContextVar 快照
future = asyncio.run_coroutine_threadsafe(coro, loop)  # 在隔离循环里跑
```

为什么？因为 `threading.Event`/`threading.Timer` 在另一个线程触发，**ContextVar 不会自动传播**。如果不 copy，子 agent 在隔离循环里就拿不到 `user_id` 等 ContextVar 了。

> 💡 **学习启示**：在 async Python 里跨事件循环/跨线程传数据，`copy_context()` 是必须的。这是个容易踩的坑。

## 2.3.4 `_aexecute`——核心执行循环（executor.py:482-675）

这是子 agent **真正跑起来**的地方。简化版伪代码：

```python
async def _aexecute(self, task, ...):
    # 1. 装一个 SubagentTokenCollector 到 callbacks
    collector = SubagentTokenCollector(caller=f"subagent:{self._config.name}")
    run_config["callbacks"] = [collector]
    run_config["tags"] = [f"subagent:{self._config.name}"]    # ★ 给 RunJournal 识别用

    # 2. 构建初始状态
    state = self._build_initial_state(task)
    #    - 把 system_prompt + skill 内容 + deferred 段拼成一条 SystemMessage
    #    - 加一条 HumanMessage(task)
    #    - 继承父级的 sandbox、thread_data

    # 3. 创建子 agent（一个独立的 LangGraph 图）
    agent = self._create_agent()

    # 4. 流式跑！
    async for chunk in agent.astream(state, config=run_config, stream_mode="values"):
        # ★ 每个 chunk 检查取消信号
        if self._result.cancel_event.is_set():          # 协作式取消
            self._result.try_set_terminal(CANCELLED)
            break

        # 抓新的 AIMessage（按 id 去重）
        for msg in chunk["messages"]:
            if isinstance(msg, AIMessage) and msg.id not in seen_ids:
                self._result.ai_messages.append(msg.model_dump())   # ★ 给 task_tool 轮询用
                seen_ids.add(msg.id)

    # 5. 从最后一条 AIMessage 抽取最终结果
    final_message = ...
    self._result.try_set_terminal(COMPLETED, result=final_message)
```

**几个值得注意的细节**：

### ① `cancel_event` 在每个 chunk 边界检查

```python
if self._result.cancel_event.is_set():
    self._result.try_set_terminal(CANCELLED)
    break
```

注意是**每个 chunk 边界**，不是每条消息。chunk 是 `astream` 的最小产出单位。这意味着：

- 如果子 agent 正在等 LLM 返回（一个长 reasoning 调用），取消信号要**等这个 chunk 产出后**才能被检查到；
- 如果子 agent 正在跑一个 5 分钟的 bash 工具调用，取消信号要**等工具返回后**（工具返回产生一个 chunk）才能被检查到。

**这是协作式取消的固有局限**——不是即时取消，而是在"安全点"取消。

### ② `ai_messages` 按 id 去重

```python
for msg in chunk["messages"]:
    if isinstance(msg, AIMessage) and msg.id not in seen_ids:
        self._result.ai_messages.append(msg.model_dump())
        seen_ids.add(msg.id)
```

**为什么要去重？** 因为 LangGraph 的 `stream_mode="values"` 每次返回**完整状态**（所有消息），不是增量。所以 chunk 里会包含之前已经见过的消息。按 `msg.id` 去重，只追加新的。

### ③ `tags` 标记子 agent

```python
run_config["tags"] = [f"subagent:{self._config.name}"]
```

这个 tag 让**父 agent 的 RunJournal** 能识别"这次 LLM 调用来自子 agent"（第 1 章 1.10.4 节讲过 `_identify_caller`）。这样 token 能正确归到 `subagent` 桶。

---

# 📖 2.4 SubagentConfig——子 agent 的"身份证"

**文件**：`subagents/config.py:10-35`

```python
@dataclass
class SubagentConfig:
    name: str
    description: str
    system_prompt: str | None = None
    tools: list[str] | None = None
    disallowed_tools: list[str] | None = field(default_factory=lambda: ["task"])  # ★ 默认禁 task
    skills: list[str] | None = None
    model: str = "inherit"        # ★ 继承父 agent 的模型
    max_turns: int = 50
    timeout_seconds: int = 900    # 15 分钟
```

**逐个字段讲**：

| 字段 | 默认值 | 含义 |
|---|---|---|
| `name` | — | 子 agent 的标识名 |
| `description` | — | 给 lead agent prompt 看的描述 |
| `system_prompt` | `None` | 子 agent 的系统提示词（None 则用默认） |
| `tools` | `None` | 允许的工具列表（None = 继承全部父工具） |
| `disallowed_tools` | `["task"]` | **★ 禁用的工具（默认禁 task，防递归）** |
| `skills` | `None` | 允许的 skill 列表 |
| `model` | `"inherit"` | 模型名（"inherit" = 用父 agent 的模型） |
| `max_turns` | 50 | 最大对话轮数 |
| `timeout_seconds` | 900 | 超时时间（15 分钟） |

**最重要的设计**：

### `disallowed_tools` 默认包含 `["task"]`

这是**防递归**的关键。即使你在 config 里忘了禁 task，dataclass 的 `field(default_factory=lambda: ["task"])` 也会默认禁掉。

**model="inherit" 的解析**（`config.py:44-56`）：

```python
def resolve_subagent_model_name(config, parent_model, *, app_config=None) -> str:
    if config.model != "inherit":
        return config.model                          # 子 agent 指定了模型 → 用它的
    if parent_model:
        return parent_model                          # 继承父 agent 的模型
    # 都没有 → 用全局默认
    return app_config.models[0].name
```

三级优先：子 agent 自己配的 > 父 agent 的 > 全局默认。

---

# 📖 2.5 Registry——子 agent 的三层注册

这一节详细讲。很多同学看完觉得"三层注册"很抽象，其实它就是回答一个问题：

> **"lead agent 调 `task(subagent_type="xxx")` 时，那个 `xxx` 的配置从哪来？"**

答案就是 Registry。它按**三层优先级**解析配置，还有一层"可见性过滤"。我们一层层拆开看。

## 2.5.1 核心函数：`get_subagent_config`

**文件**：`subagents/registry.py:50-116`。这是整个 Registry 的核心。它接收一个名字，返回一个配置好的 `SubagentConfig`。

**注释（registry.py:53-57）直接说明了三层解析顺序**：

```python
def get_subagent_config(name: str, *, app_config=None) -> SubagentConfig | None:
    """Get a subagent configuration by name, with config.yaml overrides applied.

    Resolution order (mirrors Codex's config layering):
    1. Built-in subagents (general-purpose, bash)
    2. Custom subagents from config.yaml custom_agents section
    3. Per-agent overrides from config.yaml agents section (timeout, max_turns, model, skills)
    """
```

注释里提到"mirrors Codex's config layering"——这套分层是从 OpenAI Codex 那边学来的配置叠加模式。我们来逐层看。

## 2.5.2 第一层：内置子 agent

**文件**：`subagents/builtins/__init__.py:12-15`

```python
BUILTIN_SUBAGENTS = {
    "general-purpose": GENERAL_PURPOSE_CONFIG,
    "bash": BASH_AGENT_CONFIG,
}
```

就是一个字典，硬编码了两个内置子 agent。

**registry.py:65-70** 的第一步查询：

```python
    # Step 1: Look up built-in, then fall back to custom_agents
    config = BUILTIN_SUBAGENTS.get(name)        # 先查内置
    if config is None:
        config = _build_custom_subagent_config(name, app_config=app_config)  # 没有就查自定义
    if config is None:
        return None                              # 都没有 → 返回 None（不存在）
```

**注意这不是"三层叠加"，而是"两层查找"**：

- 先在 `BUILTIN_SUBAGENTS` 里找；
- 找不到再去 `custom_agents` 里找（下一层）；
- 都找不到才返回 `None`。

**为什么用"查找"而不是"合并"？** 因为一个名字不可能既是内置的又是自定义的——它们是**互斥**的。"general-purpose"这个名字已经被内置占了，你不能在 config.yaml 里再定义一个叫"general-purpose"的自定义子 agent（会被忽略）。

## 2.5.3 第二层：自定义子 agent（config.yaml）

**文件**：`subagents/registry.py:22-47`

```python
def _build_custom_subagent_config(name: str, *, app_config=None) -> SubagentConfig | None:
    """Build a SubagentConfig from config.yaml custom_agents section."""
    subagents_config = _resolve_subagents_app_config(app_config)
    custom = subagents_config.custom_agents.get(name)    # 从 config.yaml 读
    if custom is None:
        return None

    return SubagentConfig(
        name=name,
        description=custom.description,
        system_prompt=custom.system_prompt,
        tools=custom.tools,
        disallowed_tools=custom.disallowed_tools,
        ...
    )
```

用户在 `config.yaml` 里这样定义自定义子 agent：

```yaml
subagents:
  custom_agents:
    code-reviewer:                          # ← 子 agent 的名字
      description: "审查代码质量、找 bug"
      system_prompt: "你是一个严格的代码审查员..."
      tools: ["read_file", "bash"]          # 只给读和跑命令
      disallowed_tools: ["task"]            # 禁递归
      skills: ["lint"]                      # 只用 lint skill
      model: "inherit"                      # 继承父 agent 模型
      max_turns: 30
      timeout_seconds: 600
```

`_build_custom_subagent_config` 把这个 Pydantic 配置对象（`CustomSubagentConfig`，定义在 `config/subagents_config.py:34-68`）转成 `SubagentConfig` dataclass。

## 2.5.4 第三层：per-agent 覆盖 + 全局默认（★ 最复杂）

找到 config 后（无论来自内置还是自定义），registry 还会**叠加覆盖**。这是 `registry.py:72-116` 的 Step 2：

```python
    # Step 2: Apply per-agent overrides from config.yaml agents section.
    subagents_config = _resolve_subagents_app_config(app_config)
    is_builtin = name in BUILTIN_SUBAGENTS
    agent_override = subagents_config.agents.get(name)   # per-agent 覆盖

    overrides = {}

    # Timeout: per-agent override > 全局默认(仅内置) > config 自己的值
    if agent_override is not None and agent_override.timeout_seconds is not None:
        overrides["timeout_seconds"] = agent_override.timeout_seconds
    elif is_builtin and subagents_config.timeout_seconds != config.timeout_seconds:
        overrides["timeout_seconds"] = subagents_config.timeout_seconds

    # Max turns: 同上逻辑
    if agent_override is not None and agent_override.max_turns is not None:
        overrides["max_turns"] = agent_override.max_turns
    elif is_builtin and ...:
        overrides["max_turns"] = subagents_config.max_turns

    # Model: 仅 per-agent 覆盖（无全局默认）
    effective_model = subagents_config.get_model_for(name)
    if effective_model is not None and effective_model != config.model:
        overrides["model"] = effective_model

    # Skills: 仅 per-agent 覆盖（无全局默认）
    effective_skills = subagents_config.get_skills_for(name)
    if effective_skills is not None and effective_skills != config.skills:
        overrides["skills"] = effective_skills

    if overrides:
        config = replace(config, **overrides)    # dataclass 不可变更新
```

**这段有几个关键设计，我用一个问题串起来**：

### 问题：为什么 timeout/max_turns 有"全局默认"，而 model/skills 没有？

看 `config/subagents_config.py:71-91` 的 `SubagentsAppConfig`：

```python
class SubagentsAppConfig(BaseModel):
    timeout_seconds: int = 1800     # ★ 全局默认（30 分钟），仅对内置生效
    max_turns: int | None = None    # ★ 全局默认（可选），仅对内置生效
    agents: dict[str, SubagentOverrideConfig] = ...   # per-agent 覆盖
    custom_agents: dict[str, CustomSubagentConfig] = ...  # 自定义
```

**答案**：因为内置子 agent 的 timeout/max_turns 是"硬编码的默认值"（general-purpose 在自己的 config 里写了 150，bash 写了 60），用户可能想全局调大/调小，但又不想改源码。所以提供了 `subagents.timeout_seconds`（全局默认）和 `subagents.agents.xxx.timeout_seconds`（per-agent 覆盖）两个层级。

而自定义子 agent 在 config.yaml 里**自己就定义了** timeout/max_turns（`CustomSubagentConfig` 的默认值 900/50），它的"默认"就是它自己——不需要全局默认覆盖。

**注释（registry.py:73-77）说得很清楚**：

```python
    # Only explicit per-agent overrides are applied here. Global defaults
    # (timeout_seconds, max_turns at the top level) apply to built-in agents
    # but must NOT override custom agents' own values — custom agents define
    # their own defaults in the custom_agents section.
```

翻译："全局默认只覆盖内置 agent 的值，**不能**覆盖自定义 agent 自己的值——自定义 agent 在 custom_agents 段里定义了自己的默认值。"

### 四个字段的优先级一览

| 字段 | 内置子 agent | 自定义子 agent |
|---|---|---|
| `timeout_seconds` | per-agent 覆盖 > **全局默认(1800)** > 内置硬编码 | per-agent 覆盖 > 自定义自己的值 |
| `max_turns` | per-agent 覆盖 > 全局默认(可选) > 内置硬编码 | per-agent 覆盖 > 自定义自己的值 |
| `model` | per-agent 覆盖 > "inherit" | per-agent 覆盖 > 自定义自己的值 |
| `skills` | per-agent 覆盖 > None(继承全部) | per-agent 覆盖 > 自定义自己的值 |

**为什么 model/skills 没有"全局默认"？** 因为这俩字段太重要、太个性化——给所有子 agent 强制一个 model 或 skills 没意义，反而危险。所以只有 per-agent 覆盖，不提供全局默认。

## 2.5.5 可见性过滤：沙箱感知

**文件**：`subagents/registry.py:150-165`

```python
def get_available_subagent_names(*, app_config=None) -> list[str]:
    """Get subagent names that should be exposed to the active runtime."""
    names = get_subagent_names(app_config=app_config)   # 内置 + 自定义
    try:
        host_bash_allowed = is_host_bash_allowed(app_config) if hasattr(app_config, "sandbox") else is_host_bash_allowed()
    except Exception:
        return names    # 查不到就全暴露（宽松降级）

    if not host_bash_allowed:
        names = [name for name in names if name != "bash"]    # ★ 过滤掉 bash
    return names
```

**这是个"安全闸门"**：当 host bash 不被允许时（比如生产环境禁了在宿主机跑 bash），`bash` 子 agent 就从可见列表里**消失**。

**为什么"不展示 = 不可用"就够了？** 因为 lead agent 知道有哪些子 agent，靠的是 system prompt 里的描述（`_build_available_subagents_description`，prompt.py:184-211）。如果 bash 不在列表里，prompt 里就不会提它，模型自然不会尝试调用。这是"**眼不见心不烦**"的安全模式——比"展示了但调用时报错"体验好得多。

**`get_subagent_names`**（`registry.py:133-147`）负责合并内置+自定义的名字：

```python
def get_subagent_names(*, app_config=None) -> list[str]:
    names = list(BUILTIN_SUBAGENTS.keys())                  # 内置：general-purpose, bash
    subagents_config = _resolve_subagents_app_config(app_config)
    for custom_name in subagents_config.custom_agents:      # 自定义
        if custom_name not in names:                        # 去重（自定义不能覆盖内置名）
            names.append(custom_name)
    return names
```

## 2.5.6 完整的配置解析流程图

把上面几层串起来，调 `get_subagent_config("code-reviewer")` 的完整流程：

```
get_subagent_config("code-reviewer")
   │
   ▼ Step 1: 查找（互斥，不是叠加）
   BUILTIN_SUBAGENTS 里有 "code-reviewer" 吗？
   ├── 有 → config = GENERAL_PURPOSE_CONFIG / BASH_AGENT_CONFIG
   └── 没有 → custom_agents 里有吗？
       ├── 有 → config = _build_custom_subagent_config(...)  ★ 命中自定义
       └── 没有 → return None（不存在这个子 agent）
   │
   ▼ Step 2: 叠加覆盖（仅覆盖，不替换整个 config）
   is_builtin = "code-reviewer" in BUILTIN_SUBAGENTS?  → False（自定义）
   agent_override = agents.get("code-reviewer")?       → 可能有 per-agent 覆盖
   
   对每个字段（timeout/max_turns/model/skills）：
   ├── 有 per-agent 覆盖 → 用覆盖值
   ├── 没有覆盖 + 是内置 → 用全局默认（仅 timeout/max_turns）
   └── 没有覆盖 + 是自定义 → 保留 config 自己的值
   
   config = replace(config, **overrides)   # 不可变更新
   │
   ▼ 返回最终的 SubagentConfig
```

## 2.5.7 一个真实的 config.yaml 例子

把所有概念串起来，假设你有这样的配置：

```yaml
subagents:
  # 全局默认（仅对内置生效）
  timeout_seconds: 1800          # 内置子 agent 默认 30 分钟
  max_turns: 80                  # 内置子 agent 默认 80 轮

  # per-agent 覆盖
  agents:
    general-purpose:
      timeout_seconds: 600       # 给 general-purpose 单独调短到 10 分钟
      model: "claude-sonnet-4"   # 用更强的模型
    bash:
      max_turns: 100             # bash 允许跑更多轮（批量任务）

  # 自定义子 agent
  custom_agents:
    code-reviewer:
      description: "审查代码质量"
      system_prompt: "你是严格的代码审查员..."
      tools: ["read_file", "bash"]
      disallowed_tools: ["task"]
      model: "inherit"
      max_turns: 30
      timeout_seconds: 600
```

那么：

| 查询 | 结果 |
|---|---|
| `get_subagent_config("general-purpose")` | 内置 config + **timeout=600**（per-agent 覆盖）+ **max_turns=80**（全局默认）+ **model="claude-sonnet-4"**（per-agent 覆盖） |
| `get_subagent_config("bash")` | 内置 config + timeout=1800（全局默认）+ **max_turns=100**（per-agent 覆盖）+ model="inherit"（内置默认） |
| `get_subagent_config("code-reviewer")` | 自定义 config + timeout=600（自己的值，**不被全局 1800 覆盖**）+ max_turns=30（自己的值） |
| `get_subagent_config("xxx")` | `None`（不存在） |
| `get_available_subagent_names()`（host bash 禁用） | `["general-purpose", "code-reviewer"]`（bash 被过滤） |

## 2.5.8 两个内置子 agent 的差异

| 维度 | `general-purpose` | `bash` |
|---|---|---|
| 角色 | 通用复杂任务 | 命令执行专家 |
| 工具 | `None`（**继承全部**父工具） | `["bash","ls","read_file","write_file","str_replace"]`（锁定 5 个） |
| 禁用 | `task` / `ask_clarification` / `present_files` | 同左 |
| max_turns | 150（内置硬编码） | 60（内置硬编码） |
| 可见性 | 总是可见 | 仅 host bash 允许时 |

**为什么 general-purpose 继承全部工具但禁 task？** 因为它是"全能选手"，什么都能干（除了不能再派子 agent）。bash 子 agent 则是"专家"——只干命令执行，工具被锁定到 5 个，避免它越权干别的。

## 2.5.9 小结：Registry 的设计哲学

1. **查找用"互斥"，覆盖用"叠加"**——内置和自定义是互斥的（一个名字只属于一个来源），但 timeout/max_turns/model/skills 是叠加的（可以在基础值上覆盖）；
2. **内置有全局默认，自定义没有**——因为内置的硬编码值用户没法改源码，需要全局默认提供调整入口；自定义已经在 config.yaml 里写了自己的值，不需要再被全局覆盖；
3. **可见性过滤是安全闸门**——"不展示 = 不可用"比"展示但报错"体验好；
4. **用 `dataclasses.replace` 做不可变更新**——`SubagentConfig` 是 dataclass，`replace(config, **overrides)` 返回新实例，不改原对象，避免副作用。

---

# 📖 2.6 状态契约——告别脆弱的字符串匹配

**文件**：`subagents/status_contract.py`

这是个**真实演进案例**。早期版本前端靠匹配 `"Task Succeeded. Result:"` 这种前缀字符串识别子 agent 状态——非常脆弱（issue #3146）。后来引入了结构化契约。

**核心做法**：把状态信息塞进 `ToolMessage.additional_kwargs`，不靠字符串匹配：

```python
SUBAGENT_STATUS_KEY = "subagent_status"
SUBAGENT_ERROR_KEY  = "subagent_error"
SubagentStatusValue = Literal["completed", "failed", "cancelled", "timed_out", "polling_timed_out"]
```

**前缀 → 状态的映射**（`status_contract.py:55-62`）：

```python
_PREFIX_TO_STATUS = (
    ("Task Succeeded. Result:", "completed"),
    ("Task polling timed out",  "polling_timed_out"),   # ★ 注意顺序！
    ("Task timed out",          "timed_out"),
    ("Task cancelled by user",  "cancelled"),
    ("Task failed.",            "failed"),
    ("Error",                   "failed"),
)
```

**为什么顺序很重要？** 因为有子串重叠。`"Task polling timed out"` 包含 `"Task timed out"` 的子串。如果先匹配 `"Task timed out"`，`polling_timed_out` 也会被错误地映射成 `timed_out`。所以**从最特殊的到最一般的**排序。

**前后端共享一份 JSON 契约** `contracts/subagent_status_contract.json` 作为 single source of truth——后端测试和前端测试都引用它，保证一致性。

> 💡 **学习启示**：前后端约定不要靠"口口相传"的字符串格式，而应该有**一份机器可读的契约文件**（JSON Schema / Protobuf / OpenAPI），双端都引用它。这是"契约先行"的工程实践。

---

# 📖 2.7 Token 归集——子 agent 的 token 怎么算到父 agent 头上

**文件**：`subagents/token_collector.py`

这是个**架构难点**。子 agent 是独立图，跑在隔离的事件循环里，它的 LLM 调用**不会**自动触发父 agent RunJournal 的 `on_llm_end` 回调。

**解法分三步**：

### 第 1 步：子 agent 装 collector

```python
class SubagentTokenCollector(BaseCallbackHandler):
    def on_llm_end(self, response, *, run_id, **kwargs):
        if run_id in self._counted_run_ids:
            return                        # 去重
        self._counted_run_ids.add(run_id)
        
        usage = response.generations[0][0].message.usage_metadata
        self._records.append({
            "source_run_id": str(run_id),
            "caller": self._caller,       # "subagent:general-purpose"
            "input_tokens": usage["input_tokens"],
            "output_tokens": usage["output_tokens"],
            "total_tokens": usage.get("total_tokens") or input + output,
        })
```

每个子 agent 执行装一个 collector，挂 `on_llm_end`，记录每次 LLM 调用的 token。

### 第 2 步：collector 的 records 跟着 SubagentResult 回传

子 agent 跑完后，`SubagentResult.token_usage_records` 里存着所有的 token 记录。

### 第 3 步：task_tool 把 records 上报给父 journal

```python
# task_tool.py:145-163
def _report_subagent_usage(runtime, result):
    journal = runtime.context.get("__run_journal")
    if journal and result.token_usage_records:
        journal.record_external_llm_usage_records(result.token_usage_records)
        result.usage_reported = True
```

父 journal 的 `record_external_llm_usage_records`（第 1 章 1.10.6 节讲过）按 `source_run_id` 去重，累加进 `subagent_tokens` 桶。

**完整流程图**：

```
子 agent 在隔离循环里跑
  │
  ├── 每次 LLM 调用 → SubagentTokenCollector.on_llm_end
  │                   → 追加 {source_run_id, caller, input/output/total_tokens}
  │
  ▼ 跑完
SubagentResult.token_usage_records = [记录1, 记录2, ...]
  │
  ▼ task_tool 轮询发现终态
_report_subagent_usage(runtime, result)
  │
  ▼ 调 journal.record_external_llm_usage_records(records)
父 RunJournal
  ├── 按 source_run_id 去重
  ├── 累加进 _subagent_tokens 桶
  └── _total_tokens += 所有记录的 total
```

> 💡 **学习启示**：多 agent 系统的 token 计费是个容易被忽视的难点。子 agent 是独立图，usage 不会自动流回父图。解法是"**子端采集 → 回传 → 父端聚合**"三步走。

---

# 📖 2.8 SubagentLimitMiddleware——并行限制

**文件**：`agents/middlewares/subagent_limit_middleware.py`

Lead agent 一轮可能想同时派多个子 agent（比如"并行研究 3 个话题"）。但如果不限制，一轮可能派 10 个——每个都跑 15 分钟、都耗 token，系统会崩。

**`SubagentLimitMiddleware`** 在 `after_model` 里硬限制：

```python
class SubagentLimitMiddleware(AgentMiddleware):
    def __init__(self, max_concurrent: int = 3):
        self._max = _clamp_subagent_limit(max_concurrent)   # clamp 到 [2,4]

    def after_model(self, state, response):
        ai_message = response.message
        task_calls = [tc for tc in ai_message.tool_calls if tc["name"] == "task"]
        
        if len(task_calls) > self._max:
            # 只保留前 N 个，截掉超出的
            kept = task_calls[:self._max]
            # ... 修改 ai_message，只保留 kept 的 tool_calls ...
            return {"messages": [modified_message]}
        return {}
```

**默认 `max_concurrent=3`**，且被 `_clamp_subagent_limit` 限制在 `[2,4]` 范围内——即使配置写 `max_concurrent=10`，也只会用 4。

**为什么靠硬限制而不只靠 prompt？** 因为 prompt 只是"建议"，模型可能不听。硬限制是**保障**——即使模型一轮生成了 10 个 task 调用，中间件也只保留前 3 个。

---

# 🎯 现在回头检验：你掌握了吗？

### 问题 1：Lead Agent 怎么决定"自己干"还是"派给子 agent"？

<details>
<summary>🔍 点击看答案</summary>

靠 **prompt 引导 + 工具存在性**，不是硬编码分支。

- **prompt 层面**：system prompt 的 `<subagent_system>` 段给了明确规则——复杂多步任务 DELEGATE，简单单步任务 DON'T；
- **工具层面**：只有 `subagent_enabled=True` 时 lead agent 才有 `task` 工具——没有这个工具，模型想派也派不了；
- **硬限制**：`SubagentLimitMiddleware` 保证单轮并行 ≤ 3，超出截断。

三层保障：prompt 建议 + 工具存在性 + 中间件硬限制。

</details>

### 问题 2：子 agent 跑了 10 分钟，用户中途想取消，会发生什么？

<details>
<summary>🔍 点击看答案</summary>

**协作式取消**，分多层传递：

1. 前端调 `POST /runs/{id}/cancel` → RunManager 设 `abort_event`；
2. `task_tool` 的轮询循环调 `request_cancel_background_task(task_id)` → 设 `result.cancel_event`；
3. 子 agent 的 `astream` 循环在**下一个 chunk 边界**检查 `cancel_event` → 命中就 break；
4. `try_set_terminal(CANCELLED)`；
5. task_tool 推 `task_cancelled` SSE，返回 `"Task cancelled by user."`。

**注意**：长工具调用内部**不会被中断**（比如子 agent 正在跑 5 分钟的 bash 命令），只能等它返回后在下一个 chunk 检查点取消。这是协作式取消的固有局限。

</details>

### 问题 3：为什么子 agent 跑在"隔离的事件循环"里？不能直接用 lead agent 的循环吗？

<details>
<summary>🔍 点击看答案</summary>

两个原因：

1. **不能直接用 lead agent 的循环**：lead agent 在 FastAPI 的事件循环里跑，如果子 agent 在同一个循环里 `await`，会**阻塞** lead agent 的 SSE 推送——前端看到"卡住"；
2. **不能每次新建循环**：新建+关闭循环会**破坏共享的 httpx 连接池**（LangChain async 客户端的全局池），导致连接复用失效、TLS 握手开销、甚至 `RuntimeError: Event loop is closed`。

**解法**：用一个**常驻的隔离循环**，跑在 daemon 线程上——既不阻塞主循环，又能复用连接池。配合 `copy_context()` 传播 ContextVar（如 user_id）。

</details>

### 问题 4：`try_set_terminal` 为什么要加锁？

<details>
<summary>🔍 点击看答案</summary>

因为终态可能**同时**被多个源触发（竞态条件）：

- 超时定时器到期 → 想设 `TIMED_OUT`
- 与此同时 worker 正常完成 → 想设 `COMPLETED`
- 与此同时用户点了取消 → 想设 `CANCELLED`

如果不加锁，可能出现"先 COMPLETED 又被 TIMED_OUT 覆盖"的诡异状态。

`_state_lock` + `is_terminal` 检查保证**第一个调用赢**（first-write-wins），后续调用返回 `False`。这是处理并发状态机的标准模式。

</details>

### 问题 5：子 agent 的 token 怎么算到父 agent 的 RunJournal 头上？

<details>
<summary>🔍 点击看答案</summary>

三步走：

1. **子端采集**：子 agent 执行时装一个 `SubagentTokenCollector`（BaseCallbackHandler），挂 `on_llm_end`，按 `run_id` 去重记录每次 LLM 调用的 token；
2. **回传**：记录存在 `SubagentResult.token_usage_records` 里，跟着结果回传给 `task_tool`；
3. **父端聚合**：`task_tool` 调 `_report_subagent_usage` → `journal.record_external_llm_usage_records(records)`，父 journal 按 `source_run_id` 去重，累加进 `_subagent_tokens` 桶。

子 agent 是独立图，usage 不会自动流回父图，所以必须手动采集+回传+聚合。

</details>

### 问题 6：为什么子 agent 默认禁用 `task` 工具？

<details>
<summary>🔍 点击看答案</summary>

**防止递归爆炸**。如果子 agent 能再派子 agent（孙子 agent），会引发：

- **token 失控**：每层都有自己的 LLM 调用，N 层嵌套就是 N 倍 token；
- **调试噩梦**：祖孙三层的 trace 极其难看；
- **状态传递复杂**：每层都要装 collector、回传 token、维护 status contract。

DeerFlow 的态度是"**默认扁平，最多一层**"。具体实现靠 `SubagentConfig.disallowed_tools` 默认包含 `["task"]`（`config.py` 的 dataclass 默认值）。

如果真要多层，可以在 config 里去掉 `"task"`，但要自己应对上述问题。

</details>

---

# 📝 一页纸总结（第 2 章精华）

```
┌──────────────────────────────────────────────────────────────┐
│  Lead Agent 通过 task 工具委派任务给 Subagent                  │
│  （不是多个独立服务，而是一个主图 + 临时子图）                    │
└──────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼──────────────────────┐
        ▼                     ▼                      ▼
   task_tool              SubagentExecutor        SubagentConfig
   (派活+轮询)             (干活引擎)              (子 agent 身份证)
        │                     │                      │
        │  ① 校验类型          │  ① 状态机 6 态        │  • tools（继承/锁定）
        │  ② 提取父上下文       │  ② try_set_terminal   │  • disallowed=["task"]
        │  ③ execute_async     │     （加锁，先到先得）  │  • model="inherit"
        │  ④ 轮询 5s/次        │  ③ 隔离事件循环        │  • max_turns/timeout
        │  ⑤ 推 SSE 事件       │     （daemon线程+复用池）│
        │  ⑥ 终态返回带前缀字符串│  ④ aexecute 核心       │
        │                     │     （cancel检查/chunk）│
        │                     │  ⑤ TokenCollector       │
        │                     │     （采集→回传→聚合）    │
        ▼                     ▼                      │
┌──────────────────────────────────────────────────────┐       │
│  保障层                                                │       │
│  • SubagentLimitMiddleware：单轮并行 ≤ 3（clamp [2,4]）│       │
│  • status_contract：结构化契约代替字符串匹配            │       │
│  • Registry 三层注册：内置→custom→覆盖                  │       │
│  • 沙箱感知：host bash 不允许时 bash 子 agent 不可见     │       │
└──────────────────────────────────────────────────────┘───────┘
```

**5 句话记住**：

1. **Lead Agent 通过 `task` 工具委派**——`task_tool` 做校验→提取父上下文→异步启动→5s 轮询→推 SSE→终态返回带前缀字符串；
2. **子 agent 跑在隔离事件循环**——常驻 daemon 线程 + 复用连接池 + `copy_context()` 传播 ContextVar，既不阻塞主循环又不破坏连接池；
3. **协作式取消**——`cancel_event` 在每个 chunk 边界检查，长工具调用内部不会被中断；
4. **终态 first-write-wins**——`try_set_terminal` 加锁，超时/取消/完成竞态时第一个赢；
5. **Token 三步归集**——子端装 collector 采集 → SubagentResult 回传 → 父 journal 按 source_run_id 去重聚合。
