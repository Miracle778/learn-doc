# DeerFlow 2.0 源码学习指南 —— 从零到一搞懂 Agent 应用开发

> 本指南基于本地 `deer-flow` 源码（DeerFlow **2.0**，字节跳动开源的 "super agent harness"）逐章节剖析。
> 所有代码引用格式为 `文件路径:行号`，可直接在 IDE 中跳转。
>
> **阅读前提**：了解 Python、asyncio、LangChain/LangGraph 基本概念（Tool、Message、StateGraph、ReAct）。
>
> **学习路径建议**：第 0 章 → 第 1 章 → 第 2 章 → 第 3 章 → 第 4 章 → 第 5 章 → 第 6 章 → 第 7 章 → 第 8 章。
> 每章末尾有「经典问题与源码剖析」，看完章节再做题，做完题再回头看代码，效果最好。

---

## 目录

- [第 0 章 总览：DeerFlow 是什么、为什么这样设计](#第-0-章-总览deerflow-是什么为什么这样设计)
- [第 1 章 Agent 是怎么搭起来的（LangGraph + 中间件管道）](#第-1-章-agent-是怎么搭起来的langgraph--中间件管道)
- [第 2 章 多 Agent 调度：Lead Agent 与 Subagents](#第-2-章-多-agent-调度lead-agent-与-subagents)
- [第 3 章 Agent 的记忆（Long-Term Memory）是怎么做的](#第-3-章-agent-的记忆long-term-memory是怎么做的)
- [第 4 章 上下文工程（Context Engineering）与压缩](#第-4-章-上下文工程context-engineering与压缩)
- [第 5 章 Runtime：Checkpointer、Store、Events、Runs（状态管理）](#第-5-章-runtimecheckpointerstoreeventsruns状态管理)
- [第 6 章 安全与稳定性：Guardrails、循环检测、HITL](#第-6-章-安全与稳定性guardrails循环检测hitl)
- [第 7 章 周边：工具 / MCP / Skills / Sandbox / Models](#第-7-章-周边工具--mcp--skills--sandbox--models)
- [第 8 章 对外接口：FastAPI Gateway 与 SSE 流式](#第-8-章-对外接口fastapi-gateway-与-sse-流式)
- [附录 A 推荐的学习顺序与动手实验](#附录-a-推荐的学习顺序与动手实验)
- [附录 B 术语表](#附录-b-术语表)

---

## 第 0 章 总览：DeerFlow 是什么、为什么这样设计

### 0.1 一句话定位

DeerFlow（**D**eep **E**xploration and **E**fficient **R**esearch **Flow**）2.0 是一个 **"super agent harness"（超级智能体外壳/框架）**。它编排 **sub-agents（子智能体）**、**memory（记忆）** 和 **sandboxes（沙箱）**，通过可扩展的 **skills（技能）** 完成几乎任何任务。

> ⚠️ 注意：DeerFlow 2.0 是**完全重写**，与 1.x（Deep Research 框架）没有共享代码。本指南只讲 2.0。

### 0.2 它解决什么问题

一个"能干活"的 Agent 应用要回答 7 个核心问题：

| 问题 | DeerFlow 的答案 | 对应代码 |
|---|---|---|
| ① Agent 怎么思考、决策？ | LangGraph + ReAct + 中间件管道 | `agents/factory.py`, `agents/lead_agent/` |
| ② Agent 怎么分工？ | Lead Agent 编排 + Subagents 执行 | `subagents/`, `tools/builtins/task_tool.py` |
| ③ Agent 怎么记住用户？ | 结构化 JSON 长期记忆（无向量库） | `agents/memory/` |
| ④ 上下文太长怎么办？ | 摘要 + 工具输出外置 + 动态上下文 | `agents/middlewares/summarization_middleware.py` 等 |
| ⑤ 状态怎么持久化？ | LangGraph Checkpointer + Store + 事件日志 | `runtime/` |
| ⑥ 怎么不失控？ | Guardrails + 循环检测 + 安全终止 + HITL | `guardrails/`, `agents/middlewares/` |
| ⑦ 怎么对外提供服务？ | FastAPI + SSE 流式（兼容 LangGraph 协议） | `backend/app/gateway/` |

### 0.3 顶层架构图

```
┌─────────────────────────── 浏览器 / SDK 客户端 ───────────────────────────┐
└───────────────────────────────────┬───────────────────────────────────────┘
                                    │ HTTP / SSE
                                    ▼
┌──────────────────────── FastAPI Gateway (端口 8001) ──────────────────────┐
│  /api/threads   /api/threads/{id}/runs/stream   /api/models  /api/skills  │
│  AuthMiddleware → CSRFMiddleware → CORSMiddleware                          │
└───────────────────────────────────┬───────────────────────────────────────┘
                                    │ start_run() → asyncio.create_task()
                                    ▼
┌─────────────────────── Runtime（runs/worker.py: run_agent） ───────────────┐
│  Checkpointer(sqlite/postgres)  +  Store  +  StreamBridge  +  RunJournal   │
└───────────────────────────────────┬───────────────────────────────────────┘
                                    │ graph.astream(stream_mode=...)
                                    ▼
┌─────────────────── Lead Agent（make_lead_agent）──────────────────────────┐
│  create_agent(model, tools, middleware=[...15 个中间件...],                │
│               system_prompt, state_schema=ThreadState)                     │
│                                                                            │
│  中间件管道（关键顺序）：                                                  │
│  DynamicContext → Skill → Summarization → Todo → TokenUsage → Title        │
│  → Memory → ViewImage → DeferredToolFilter → SubagentLimit → Loop          │
│  → 自定义 → SafetyFinishReason → Clarification(永远最后)                    │
└──────┬───────────────────────────────────────────────────────┬────────────┘
       │ 直接调工具                                              │ task 工具
       ▼                                                          ▼
┌──────────────────────┐                          ┌──────────────────────────┐
│  内置工具 / MCP 工具   │                          │  Subagent Executor       │
│  bash/read_file/...  │                          │  (独立事件循环 + 线程池)   │
│  web_search/...      │                          │  general-purpose / bash  │
└──────────────────────┘                          └──────────────────────────┘
       │                                                       │
       ▼ 共享 Sandbox + ThreadData + ThreadId                  │
┌──────────────────────────────────────────────────────────────────────────┐
│  Memory（JSON 文件，带去抖队列）   Skills（SKILL.md）   Sandbox（本地/Docker）│
└──────────────────────────────────────────────────────────────────────────┘
```

### 0.4 关键技术栈

- **核心框架**：LangGraph（状态图）+ LangChain（Agent / Tool / Message 抽象）
- **Web**：FastAPI + SSE（Server-Sent Events）
- **持久化**：SQLite（默认）/ Postgres；内存兜底
- **配置**：`config.yaml`（Pydantic 模型校验）+ `extensions_config.json`（MCP/Skills 运行时配置）
- **模型接入**：完全配置驱动，通过 `use:` 字段反射加载（`langchain_openai:ChatOpenAI`、`langchain_anthropic:ChatAnthropic`、vLLM、DeepSeek、Kimi 等）

### 0.5 仓库目录速查（核心代码都在这里）

```
deer-flow/
├── backend/
│   ├── langgraph.json              ← 声明 graph 入口 = make_lead_agent
│   ├── app/gateway/                ← FastAPI 服务（对外）
│   │   ├── app.py                  ← create_app()
│   │   ├── routers/                ← threads.py / thread_runs.py / runs.py ...
│   │   └── auth/                   ← 鉴权
│   └── packages/harness/deerflow/  ← ★ 核心库（harness = "外壳"）
│       ├── agents/
│       │   ├── lead_agent/         ← 主智能体（prompt + 工厂）
│       │   ├── factory.py          ← create_deerflow_agent（SDK 级工厂）
│       │   ├── thread_state.py     ← ThreadState（贯穿会话的状态）
│       │   ├── features.py         ← RuntimeFeatures 声明式开关
│       │   ├── memory/             ← 长期记忆（storage/queue/updater/prompt）
│       │   └── middlewares/        ← ★ 15+ 个中间件（本章后面会逐个讲）
│       ├── subagents/              ← 子智能体（executor/registry/config）
│       ├── tools/                  ← 工具系统 + 内置工具
│       ├── mcp/                    ← MCP 客户端
│       ├── skills/                 ← SKILL.md 加载/安装
│       ├── sandbox/                ← 沙箱抽象（local/docker）
│       ├── models/                 ← create_chat_model + 各 provider
│       ├── guardrails/             ← 工具调用前的策略授权
│       ├── runtime/                ← Checkpointer/Store/Events/Runs/StreamBridge
│       ├── persistence/            ← ORM 模型（threads_meta/run_events 等）
│       └── config/                 ← 各子系统的 Pydantic 配置
└── frontend/                       ← Next.js 前端（不在本指南范围）
```

> 📌 **从哪里开始读代码？**
> 先读 `backend/langgraph.json`（5 行）→ `agents/lead_agent/agent.py:make_lead_agent` → `agents/factory.py:_make_lead_agent`。这三处串起来就是"一个 Agent 是怎么被造出来的"。

---

## 第 1 章 Agent 是怎么搭起来的（LangGraph + 中间件管道）

### 1.1 入口：langgraph.json

`backend/langgraph.json`：

```json
{
  "graphs": { "lead_agent": "deerflow.agents:make_lead_agent" },
  "auth": { "path": "./app/gateway/langgraph_auth.py:auth" },
  "checkpointer": { "path": ".../runtime/checkpointer/async_provider.py:make_checkpointer" }
}
```

这意味着：整个系统对外只暴露**一个图**——`lead_agent`，由 `make_lead_agent` 工厂函数创建。所谓的"多 Agent"其实是**一个主图 + 子图（subagent）** 的嵌套结构。

### 1.2 `make_lead_agent`：配置驱动的总装线

文件：`backend/packages/harness/deerflow/agents/lead_agent/agent.py`，但真正的装配在 `agents/factory.py:_make_lead_agent`。

读 `factory.py:409-540`，流程是：

```python
def _make_lead_agent(config: RunnableConfig) -> CompiledStateGraph:
    # 1) 合并 configurable + context 为运行时配置
    runtime_cfg = _get_runtime_config(config)

    # 2) 解析一堆开关
    thinking_enabled   = ...
    model_name         = ...
    is_plan_mode       = ...
    subagent_enabled   = ...
    max_concurrent     = ...
    is_bootstrap       = ...
    agent_name         = ...   # 自定义 agent 才有

    # 3) 加载该 agent 的配置 + 可用 skills
    agent_cfg = load_agent_config(...)
    skills    = _available_skill_names(...)

    # 4) 解析模型名（带兜底）
    model_name = _resolve_model_name(request=model_name, agent=..., global=...)

    # 5) 注入 LangSmith trace 元数据
    config["metadata"] = {...}

    # 6) ★ 在图根节点一次性挂 tracing 回调（关键不变量！）
    tracing_callbacks = build_tracing_callbacks()
    if tracing_callbacks:
        config["callbacks"] = [*existing, *tracing_callbacks]

    # 7) 过滤工具、组装 deferred tools
    final_tools = filter_tools_by_skill_allowed_tools(tools, skills, ...)
    deferred    = assemble_deferred_tools(final_tools, enabled=...)

    # 8) 造图
    return create_agent(
        model=create_chat_model(name=model_name, thinking_enabled=..., attach_tracing=False),
        tools=final_tools,
        middleware=build_middlewares(...),
        system_prompt=apply_prompt_template(...),
        state_schema=ThreadState,
    )
```

这里 `create_agent` 是 `langchain.agents.create_agent`——LangChain 1.0 的新版 Agent 构造器，本质是一个 LangGraph 的 ReAct 循环（`model node ↔ tools node`）。

> 💡 **为什么 tracing 要在图根挂一次，而每个 `create_chat_model` 都传 `attach_tracing=False`？**
> 见 `factory.py:1-19` 的模块文档：因为子图（subagent）、中间件（标题生成、摘要）都会各自 `create_chat_model`，如果每个都自己挂回调，会出现**重复 span**。统一在根节点挂一次回调，由 LangGraph 的回调传播机制下发给所有子调用，是最干净的做法。这是一个值得记住的工程模式。

### 1.3 三种 Agent 模式：bootstrap / default / custom

`factory.py:486-540` 根据 `is_bootstrap` 和 `agent_name` 分三种装配：

| 模式 | 触发条件 | 特点 |
|---|---|---|
| **Bootstrap** | `is_bootstrap=True` | 只给 `{bootstrap}` skill + `setup_agent` 工具；用于首次创建一个自定义 agent |
| **Custom** | `agent_name` 非空 | 加 `update_agent` 工具，让 agent 编辑自己的 SOUL.md / config.yaml |
| **Default** | 都不满足 | 无自更新工具，是日常对话用的通用 agent |

### 1.4 ThreadState：贯穿会话的状态

文件：`agents/thread_state.py`。`ThreadState(AgentState)` 继承自 LangChain 的 `AgentState`（提供 `messages` 字段），并扩展：

| 字段 | 类型 | Reducer | 作用 |
|---|---|---|---|
| `messages` | list[Message] | `add_messages` | 对话历史（继承自 AgentState） |
| `sandbox` | `SandboxState \| None` | — | 当前沙箱句柄（{sandbox_id}） |
| `thread_data` | `ThreadDataState \| None` | — | 工作目录路径（workspace/uploads/outputs） |
| `title` | `str \| None` | — | 自动生成的会话标题 |
| `artifacts` | `list[str]` | `merge_artifacts`（去重保序） | 产物文件路径 |
| `todos` | `list \| None` | `merge_todos`（last-non-None） | TodoList 状态 |
| `uploaded_files` | `list[dict] \| None` | — | 上传文件清单 |
| `viewed_images` | `dict[str, ViewedImageData] | {}` | `merge_viewed_images` | base64 图片注入 |
| `promoted` | `PromotedTools \| None` | `merge_promoted` | deferred tool 的提升记录 |

> 🔑 **Reducer 是 LangGraph 的核心概念**：当多个节点并发写入同一字段时，reducer 决定如何合并。比如 `merge_artifacts` 会保序去重；`merge_promoted` 用 `catalog_hash` 做版本隔离。看 `thread_state.py:21-84` 可以学到几个经典 reducer 写法。

### 1.5 中间件管道：Agent 的"洋葱模型"

这是 DeerFlow 最值得学的部分。`build_middlewares`（`factory.py:270-377`）按**严格顺序**组装约 15 个中间件，每个中间件可以在 `before_model` / `after_model` / `wrap_model_call` / `wrap_tool_call` / `before_agent` / `after_agent` 等 hook 点插逻辑。

```
请求 ──▶ ToolErrorHandling（基础错误处理）
      ──▶ DynamicContext（注入 <memory> + <current_date> 到首条 user msg）
      ──▶ SkillActivation（处理 /skill-name 斜杠激活）
      ──▶ Summarization（按 token 阈值压缩历史）
      ──▶ Todo（plan 模式：注入 todo 提醒，阻止提前收尾）
      ──▶ TokenUsage（统计 token、给每步打 attribution）
      ──▶ Title（异步生成会话标题）
      ──▶ Memory（after_agent 触发记忆更新入队）
      ──▶ ViewImage（视觉模型才装）
      ──▶ DeferredToolFilter（隐藏 MCP 工具 schema，需先 tool_search）
      ──▶ SubagentLimit（限制单轮并行 task ≤ 3）
      ──▶ LoopDetection（检测重复工具调用）
      ──▶ [自定义中间件]
      ──▶ SafetyFinishReason（处理 content_filter 等安全终止）
      ──▶ Clarification（★ 永远最后；拦截 ask_clarification 实现 HITL）
                                              ──▶ 模型 / 工具
```

**关键设计点**（务必理解，后面会反复用到）：

1. **顺序很重要**。比如 `ClarificationMiddleware` 必须最后装——它的 `after_model` 在 LangChain 里是**逆序执行**的，所以最后装的反而最先看到模型输出，可以第一时间拦截 `ask_clarification`。同理 `SafetyFinishReasonMiddleware` 故意排在自定义中间件之后（`factory.py:371`）。

2. **"延迟注入"模式**：很多中间件（Loop、Todo）在 `after_model` 里**检测**到问题，但**不在那一刻加消息**——而是把警告暂存到 `_pending_warnings`，等到下一轮的 `wrap_model_call` 才追加。
   > 为什么？因为 OpenAI/Moonshot 校验「每个 AIMessage(tool_calls) 后面必须紧跟对应的 ToolMessage」；如果你在两者中间插一条 HumanMessage，下一次请求会被服务端拒绝（详见 `loop_detection_middleware.py:18-38` 的注释）。这是一个**血泪教训型**的工程约束。

3. **声明式 vs 命令式**。`agents/factory.py`（SDK 级）提供了 `RuntimeFeatures`（`features.py`）声明式写法：

   ```python
   @dataclass
   class RuntimeFeatures:
       sandbox: bool | AgentMiddleware = True
       memory: bool | AgentMiddleware = False
       summarization: Literal[False] | AgentMiddleware = False
       subagent: bool | AgentMiddleware = False
       ...
   ```

   每个字段可以传 `True`（用默认实现）/ `False`（禁用）/ 一个 `AgentMiddleware` 实例（替换）。配合 `@Next(anchor)` / `@Prev(anchor)` 装饰器还能声明"我要排在某个内置中间件的前/后"。看 `features.py:42-63` 和 `factory.py:306-379`（SDK）。

### 1.6 system prompt 的组装

文件：`agents/lead_agent/prompt.py`。核心是 `SYSTEM_PROMPT_TEMPLATE`（`prompt.py:364`）和 `apply_prompt_template()`。

值得注意的设计：

- **Prompt 尽量保持静态**（`prompt.py:799-803`）——为了让 LLM 提供商的 **prefix cache** 命中。所有"会变"的东西（日期、用户记忆）**不进 system prompt**，而是通过 `DynamicContextMiddleware` 注入到第一条 user message 的 `<system-reminder>` 里（详见第 4 章）。
- **按特性拼装**：`subagent_enabled=True` 时追加 `<subagent_system>` 段（`prompt.py:236-361`），教模型如何 DECOMPOSE → DELEGATE → SYNTHESIZE。
- **加载 SOUL.md**：每个自定义 agent 有自己的"灵魂文件"（`SOUL.md`），通过 `get_agent_soul(agent_name)`（`prompt.py:677-682`）注入，实现 agent 人格化。

---

### 🎯 第 1 章 经典问题与源码剖析

#### Q1：LangChain 1.0 的 `create_agent` 和老版 `AgentExecutor` 有什么区别？为什么 DeerFlow 选它？

**A**：老版 `AgentExecutor` 是一个"封装好的对象"，难以在中间环节插逻辑；而 `create_agent` 返回的是一个 **LangGraph CompiledGraph**，本质是 `model node ↔ tools node` 的图。这样：

1. 可以用 **Agent Middleware**（`AgentMiddleware`）在任意 hook 点（`before_model`、`wrap_tool_call` 等）插逻辑，DeerFlow 的 15 个中间件就是这样挂上去的；
2. 状态走 `state_schema`（DeerFlow 用 `ThreadState`），可以用 LangGraph 的 reducer 机制精细控制并发合并；
3. 天然支持 `graph.astream(stream_mode="values")` 流式，方便 SSE 推给前端。

**对照源码**：`factory.py:520-540` 调用 `create_agent`；`agents/factory.py:61-147`（SDK）还提供了一个更底层的 `create_deerflow_agent`，介于 `create_agent` 和 `make_lead_agent` 之间，方便 SDK 用户不写 config.yaml 直接造 agent。

#### Q2：为什么中间件要分 `before_model` / `after_model` / `wrap_model_call` / `wrap_tool_call` 这么多 hook？

**A**：因为不同 hook 点能"看到"的状态不同：

| Hook | 时机 | 典型用途 |
|---|---|---|
| `before_agent` | 整个 agent run 开始 | 清理上一轮残留（如 loop warning） |
| `before_model` | 调模型前 | 检查要不要摘要、注入提醒 |
| `wrap_model_call` | 包裹模型调用 | 改 request（增删消息、过滤工具） |
| `after_model` | 模型返回后 | 解析 usage、检测循环 |
| `wrap_tool_call` | 包裹工具调用 | guardrail 拦截、预算控制 |
| `after_agent` | 整个 agent run 结束 | 触发记忆更新、生成标题 |

**对照源码**：

- `summarization_middleware.py:189-193` 用 `before_model` 在每次调模型前检查 token；
- `loop_detection_middleware.py:534-540` 用 `after_model` 在模型返回后统计工具调用；
- `loop_detection_middleware.py:579-593` 用 `wrap_model_call` 把警告追加到下次请求；
- `clarification_middleware.py` 用 `wrap_tool_call` 在工具执行前拦截 `ask_clarification`。

#### Q3：15 个中间件顺序错了会出什么问题？举一个具体例子。

**A**：举两个真实约束：

1. **`ClarificationMiddleware` 必须最后装**（`factory.py:376`）。因为它的 `after_model` 要拦截 `ask_clarification` 工具调用并 `goto=END` 中断。如果它前面还有别的 `after_model` 在跑（比如 TokenUsage 已经记了一笔），逻辑就乱了。LangChain 的 `after_model` 是**按列表逆序执行**的，所以最后装的最早看到模型输出。

2. **`SummarizationMiddleware` 必须早装**（`agent.py:263` 注释："should be early to reduce context before other processing"）。如果在它之前有中间件已经在改 `messages`，那么"该被摘要掉的旧消息"可能已经被别的东西引用了，造成不一致。

**学习建议**：把 `factory.py:260-377` 的注释读一遍，里面把每一步为什么要这个顺序都写清楚了。

---

## 第 2 章 多 Agent 调度：Lead Agent 与 Subagents

> 这是 DeerFlow 最有意思的部分。它不是"多个独立 Agent 通过消息总线通信"那种 actor 模型，而是 **Lead Agent 作为编排者，通过 `task` 工具把子任务委派给 Subagent**，Subagent 跑完把结果作为 ToolMessage 返回，Lead Agent 再综合。

### 2.1 三个相关工具，各司其职

| 工具 | 文件 | 何时用 | 谁能用 |
|---|---|---|---|
| `task` | `tools/builtins/task_tool.py` | 委派一个子任务给 subagent | 仅 `subagent_enabled=True` 的 lead agent |
| `setup_agent` | `tools/builtins/setup_agent_tool.py` | 首次创建一个自定义 agent | 仅 bootstrap agent |
| `update_agent` | `tools/builtins/update_agent_tool.py` | 修改已存在的自定义 agent | 仅 custom agent（`agent_name` 非空） |

### 2.2 `task` 工具的生命周期（最重要！）

文件：`tools/builtins/task_tool.py:229-441`。当一个 lead agent 决定委派任务时：

```
Lead Agent 调用 task(description, prompt, subagent_type)
   │
   ▼ task_tool.py:234-241  校验 subagent_type 是否可用；sandbox 模式下拒绝 bash
   ▼ task_tool.py:258-270  从 runtime.state / context 提取父上下文：
   │                       sandbox_state, thread_data, thread_id, parent_model, trace_id
   ▼ task_tool.py:175-183  合并父 skill 白名单与子 agent 的 skill 白名单
   ▼ task_tool.py:301-312  构造 SubagentExecutor
   ▼ task_tool.py:316      executor.execute_async(prompt, task_id=tool_call_id)
   │                       （★ 总是异步后台执行，立即返回 PENDING）
   ▼
   ┌──────────── task_tool 自己的轮询循环（每 5s）────────────┐
   │  while not 超时:                                            │
   │      result = get_background_task_result(task_id)          │
   │      通过 get_stream_writer() 推 SSE：                       │
   │         - task_started                                     │
   │         - task_running（每条新 AIMessage 推一次）            │
   │         - task_completed / task_failed / ...               │
   │      if result.is_terminal: break                          │
   └─────────────────────────────────────────────────────────────┘
   ▼ task_tool.py:373-415  返回带前缀的状态字符串：
       "Task Succeeded. Result: ..."
       "Task failed. Error: ..."
       "Task cancelled by user."
       "Task timed out. Error: ..."
       "Task polling timed out after N minutes..."
```

**为什么这样设计**（异步 + 轮询，而不是同步阻塞）？

- 因为 lead agent 是在 LangGraph 的 **ToolNode** 里执行 `task` 工具，如果同步阻塞，整个图就卡住了，SSE 也没法实时推送进度；
- 异步执行 + `get_stream_writer()` 推 SSE，前端就能实时看到"子 agent 正在思考..."这种中间状态；
- 取消是**协作式**的：调 `request_cancel_background_task(task_id)` 会设置 `result.cancel_event`，子 agent 在每个 `astream` chunk 之间检查这个 event（`executor.py:552-559`）。

### 2.3 SubagentExecutor：在"隔离的事件循环"里跑

文件：`subagents/executor.py`。这是整个多 agent 系统最精巧的部分。

#### 2.3.1 状态机

```python
class SubagentStatus(Enum):
    PENDING   # 刚入队
    RUNNING   # 执行中
    COMPLETED # 成功
    FAILED    # 失败
    CANCELLED # 被用户取消
    TIMED_OUT # 超时
```

`SubagentResult`（`executor.py:67-133`）持有：task_id、trace_id、status、result、error、ai_messages、token_usage_records、cancel_event 等。**关键方法** `try_set_terminal(...)`（`executor.py:100-133`）用 `_state_lock` 保证**第一个终态转换生效**——超时、取消、worker 完成可能同时发生，必须有一个赢。

#### 2.3.2 三种执行入口

```python
execute()          # executor.py:713-755  通用入口
                   #   - 检测当前是否在事件循环里
                   #   - 在 → 转给 _execute_in_isolated_loop
       _execute_in_isolated_loop()    # executor.py:677-711
                   #   - 用一个"长生命周期的隔离事件循环"
                   #     _get_isolated_subagent_loop() (executor.py:202-230)
                   #     跑在名为 "subagent-persistent-loop" 的 daemon 线程上
                   #   - 用 copy_context() 传播 ContextVar（如 user_id）
                   #   - 不在 → asyncio.run(_aexecute)
execute_async()   # executor.py:757-818  lead agent 走的就是这个
                   #   - submit 到 _scheduler_pool（ThreadPoolExecutor, max=3）
                   #   - 池里再 submit 到隔离循环
                   #   - 结果存到 _background_tasks dict
```

> 🔑 **为什么要"隔离的事件循环"？**
> 因为 lead agent 自己就在一个事件循环里跑（FastAPI 的）。如果 subagent 在同一个循环里 `await`，会阻塞 lead agent 的 SSE 推送。而**为每次调用新建/关闭一个循环**又会破坏共享的 httpx 连接池（连接复用失效、TLS 握手开销）。所以 deer-flow 用一个**常驻的隔离循环**，跑在独立 daemon 线程上，既不阻塞主循环，又能复用连接池。这是个很值得抄的工程模式。

#### 2.3.3 `_aexecute` 的核心循环

```python
async def _aexecute(self, task, ...):
    # 1) 装一个 SubagentTokenCollector 到 callbacks（按 caller="subagent:{name}" 标记）
    # 2) _build_initial_state：拼装 SystemMessage（含 system_prompt + skill + deferred 段）+ HumanMessage(task)
    #    ★ 注意：把 system_prompt 作为 SystemMessage 注入，而不是 create_agent 的参数
    #    （避免出现两条 SystemMessage）
    # 3) 把父级的 sandbox、thread_data 写入子 state（共享！）
    # 4) agent.astream(state, config=run_config, context=context, stream_mode="values")
    #    每个 chunk：
    #      - 检查 cancel_event（协作式取消）
    #      - 按 id 去重新抓 AIMessage，追加到 result.ai_messages
    #      - 这些就是 task_tool 推 task_running SSE 的数据源
    # 5) 从最后一条 AIMessage 抽取最终结果
    # 6) try_set_terminal(COMPLETED | FAILED | ...)
```

### 2.4 Subagent 配置与注册

#### 2.4.1 SubagentConfig

文件：`subagents/config.py:10-35`：

```python
@dataclass
class SubagentConfig:
    name: str
    description: str
    system_prompt: str | None = None
    tools: list[str] | None = None
    disallowed_tools: list[str] | None = field(default_factory=lambda: ["task"])  # ★ 防递归
    skills: list[str] | None = None
    model: str = "inherit"      # 继承父 agent 的模型
    max_turns: int = 50
    timeout_seconds: int = 900  # 15 分钟
```

**默认禁用 `task` 工具**——防止 subagent 再派 subagent，形成无限递归。

#### 2.4.2 Registry 的三层解析

文件：`subagents/registry.py:50-116`：

```
1. 内置 subagent（BUILTIN_SUBAGENTS：general-purpose / bash）  registry.py:66
2. config.yaml 里的 custom_agents                              registry.py:22-47
3. agents 段的 per-agent 覆盖 + 全局默认                        registry.py:75-116
```

`get_available_subagent_names`（`registry.py:150-165`）还会根据 `is_host_bash_allowed()` 过滤掉 `bash`——这是**沙箱感知**的可见性闸门。

#### 2.4.3 两个内置 subagent 的差异

| 维度 | `general-purpose` | `bash` |
|---|---|---|
| 角色 | 通用复杂任务 | 命令执行专家 |
| 工具 | `None`（**继承全部**父工具） | `["bash","ls","read_file","write_file","str_replace"]`（锁定 5 个） |
| 禁用 | `task` / `ask_clarification` / `present_files` | 同左 |
| max_turns | 100 | 60 |
| 可见性 | 总是可见 | 仅 `is_host_bash_allowed=True` 时可见 |

### 2.5 状态契约（status_contract）：告别脆弱的字符串匹配

> 这是一个真实演进案例：早期版本靠前端匹配 `"Task Succeeded. Result:"` 这样的前缀字符串来识别子 agent 状态（issue #3146），非常脆弱。后来引入了结构化契约。

文件：`subagents/status_contract.py`。核心做法是把状态信息塞进 `ToolMessage.additional_kwargs`：

```python
SUBAGENT_STATUS_KEY = "subagent_status"
SUBAGENT_ERROR_KEY  = "subagent_error"
SubagentStatusValue = Literal["completed", "failed", "cancelled", "timed_out", "polling_timed_out"]

# task_tool 返回的字符串 → 状态值（按特异性从高到低，处理子串重叠）
_PREFIX_TO_STATUS = (
    ("Task Succeeded. Result:", "completed"),
    ("Task polling timed out",  "polling_timed_out"),
    ("Task timed out",          "timed_out"),
    ("Task cancelled by user",  "cancelled"),
    ("Task failed.",            "failed"),
    ("Error",                   "failed"),
)
```

前后端共享一份 JSON 契约 `contracts/subagent_status_contract.json` 作为 single source of truth。

### 2.6 Token 归集：子 agent 的 token 怎么算到父 agent 头上

文件：`subagents/token_collector.py`。`SubagentTokenCollector(BaseCallbackHandler)` 每个 subagent 执行装一个，挂 `on_llm_end`：

- 按 `run_id` 去重；
- 读 `usage_metadata`（input/output/total），缺失时 `total = input + output`；
- 跳过零 usage；
- 追加 `{source_run_id, caller, input_tokens, output_tokens, total_tokens}` 记录。

这些记录通过 `SubagentResult.token_usage_records` → `task_tool._report_subagent_usage` → **父级 `RunJournal.record_external_llm_usage_records`**（`task_tool.py:145-163`），实现 token 归集。

> 💡 **架构启示**：多 agent 系统的 token 计费是一个容易被忽视的难点。子 agent 是独立图，它的 `usage_metadata` 不会自动流回父图。DeerFlow 的做法是「子 agent 装 callback 采集 → 通过 task 工具回传 → 父级 journal 聚合」，并强调 `result.usage_reported` 单次上报不变量。

---

### 🎯 第 2 章 经典问题与源码剖析

#### Q1：Lead Agent 怎么决定"自己干"还是"派给子 agent"？

**A**：靠 **prompt 引导 + 工具存在性**，不是硬编码分支。`prompt.py:236-361` 的 `<subagent_system>` 段给了明确规则：

- **DELEGATE**：当任务是"多步探索 + 行动"、可并行、需要长时间专注时；
- **DON'T DELEGATE**：单步动作、有顺序依赖、超简单的读取（prompt.py:292-298, 350-354 给了反例）。

同时 `SubagentLimitMiddleware`（`subagent_limit_middleware.py`）在 `after_model` 里硬限制：单轮并行 `task` 调用 ≤ 3 个（`MAX_CONCURRENT_SUBAGENTS`，clamped 到 [2,4]），超出就只保留前 3 个。这比纯靠 prompt 可靠。

#### Q2：子 agent 跑了 10 分钟，用户中途想取消，会发生什么？

**A**：协作式取消，分三层：

1. 前端调 `POST /api/threads/{id}/runs/{run_id}/cancel`（`thread_runs.py:224`）；
2. RunManager 设置 `abort_event` + `abort_action="interrupt"`；
3. **task_tool 的轮询循环**（`task_tool.py:413`）调 `request_cancel_background_task(task_id)`，设置 `result.cancel_event`；
4. **subagent 的 astream 循环**（`executor.py:552-559`）在每个 chunk 边界检查 `cancel_event`，命中就 break；
5. `try_set_terminal(CANCELLED)`；
6. task_tool 轮询发现终态，推 `task_cancelled` SSE，返回 `"Task cancelled by user."`。

注意：**长工具调用内部不会被中断**（比如子 agent 正在跑一个 5 分钟的 bash 命令），只能等这个工具调用返回后，在下个 chunk 检查点取消。这是协作式取消的固有局限。

#### Q3：为什么 subagent 默认禁用 `task` 工具？如果我就是想要"孙子 agent"怎么办？

**A**：禁用是为了**防止递归爆炸**和**token 失控**。如果你真的需要多层委派，可以在 `SubagentConfig.disallowed_tools` 里去掉 `"task"`，但要自己控制 `max_turns` 和 `timeout_seconds`，并准备好应对：

- token 归集变复杂（每层都要装 collector）；
- 状态契约要正确传递（祖孙三层的 status 字段）；
- 调试和 trace 会非常难看。

DeerFlow 的态度是「**默认扁平，最多一层委派**」——这是个值得借鉴的工程取舍。

#### Q4：`SubagentResult.try_set_terminal` 为什么要加锁？

**A**：因为终态可能**同时**被多个源触发：

- 超时定时器到期 → 想设 `TIMED_OUT`；
- 与此同时 worker 正常完成 → 想设 `COMPLETED`；
- 与此同时用户点了取消 → 想设 `CANCELLED`。

如果不加锁，可能出现"先 COMPLETED 又被 TIMED_OUT 覆盖"的诡异状态。`executor.py:100-133` 用 `_state_lock` + `_terminal` 标志保证**第一个调用赢**，后续调用直接返回 `False`。这是处理并发状态机的标准模式。

---

## 第 3 章 Agent 的记忆（Long-Term Memory）是怎么做的

> ⚠️ **重要且反直觉**：DeerFlow 的长期记忆**没有向量数据库、没有 embedding、没有语义检索**。我 grep 了整个 `deerflow` 包，`embedding|vector|chroma|faiss|pinecone` **零匹配**。
>
> 它用的是 **"结构化 JSON 文档 + LLM 抽取 + 置信度排序 + 贪心 token 预算打包"** 的方案。这一点非常值得学——它挑战了"长期记忆 = 向量库"的思维定势。

### 3.1 记什么：结构化 JSON Schema

文件：`agents/memory/storage.py:24-40`，`create_empty_memory()`：

```python
def create_empty_memory() -> dict[str, Any]:
    return {
        "version": "1.0",
        "lastUpdated": utc_now_iso_z(),
        "user": {
            "workContext":      {"summary": "", "updatedAt": ""},  # 工作背景
            "personalContext":  {"summary": "", "updatedAt": ""},  # 个人背景
            "topOfMind":        {"summary": "", "updatedAt": ""},  # 当前关注（更新最频繁）
        },
        "history": {
            "recentMonths":        {"summary": "", "updatedAt": ""},  # 近 1-3 月
            "earlierContext":      {"summary": "", "updatedAt": ""},  # 3-12 月
            "longTermBackground":  {"summary": "", "updatedAt": ""},  # 基础/持久
        },
        "facts": [],   # 离散事实，每条带置信度
    }
```

所以存三类信息：

1. **结构化摘要**（字符串字段）：当前状态 + 时间分层；
2. **离散事实数组**，每条形如（`updater.py:658-665`）：
   ```python
   {
       "id": "fact_xxxx",
       "content": "...",
       "category": "preference"|"knowledge"|"context"|"behavior"|"goal"|"correction",
       "confidence": 0.85,
       "createdAt": "...",
       "source": thread_id,
   }
   ```

**没有实体抽取、没有知识图谱**——纯叙事摘要 + 置信度排序的事实列表。

### 3.2 存哪里：可插拔的 JSON 文件存储

文件：`agents/memory/storage.py`。

- **抽象**：`MemoryStorage` ABC + 默认实现 `FileMemoryStorage`；
- **可插拔**：`MemoryConfig.storage_class` 默认 `"deerflow.agents.memory.storage.FileMemoryStorage"`，通过 `importlib` 动态加载（`storage.py:196-231`）；
- **路径解析**（`storage.py:84-102`）：按 `(user_id, agent_name)` 隔离；
- **原子写**：`save()`（`storage.py:160-189`）先写 `.{uuid}.tmp` 再 `temp_path.replace(file_path)`——POSIX 原子重命名；
- **缓存**：`_memory_cache` 按 `(user_id, agent_name)` 缓存，靠 `file_path.stat().st_mtime` 失效——只有文件改动才重读；
- **线程安全**：`threading.Lock` 保护所有缓存读写。

### 3.3 怎么写：去抖队列 + LLM 抽取 + JSON 合并

这是记忆子系统的核心管道，分三个组件。

#### 3.3.1 Queue：去抖 + 单例

文件：`agents/memory/queue.py`。`MemoryUpdateQueue` 是单例（`get_memory_queue()`，`queue.py:265-275`），内置**去抖定时器**：

- `add()`（`queue.py:52-88`）：入队 + 启动/重启 `threading.Timer(config.debounce_seconds)`（默认 30s）；
- `add_nowait()`（`queue.py:90-115`）：入队 + 立即调度（delay=0）；
- **去抖 key** = `(thread_id, user_id, agent_name)`（`_queue_key`，`queue.py:43-50`）。同一 key 的新对话**替换**旧的，但**合并** `correction_detected` / `reinforcement_detected` 标志（OR 语义，已检测到的信号永不丢失）；
- 定时器触发后，`_process_queue()`（`queue.py:166-214`）加锁、复制队列、清空、对每个 context 调 `MemoryUpdater().update_memory(...)`，每两次之间 sleep 0.5s 防限流。

> 📌 **ContextVar 陷阱**（`queue.py:67-69`）：`user_id` 在入队时就被捕获存进 `ConversationContext`，因为 `threading.Timer` 在另一个线程触发，**ContextVar 不会自动传播过去**。这是个容易踩的坑。

#### 3.3.2 Updater：LLM 驱动的抽取引擎

文件：`agents/memory/updater.py`。`MemoryUpdater`（`updater.py:380-684`）：

1. `_prepare_update_prompt()`（`updater.py:422-449`）：加载当前记忆 → 格式化对话 → 拼装 `MEMORY_UPDATE_PROMPT`；
2. `_get_model()`：用 `config.model_name`（或默认）+ `create_chat_model(thinking_enabled=False)`；
3. **故意用同步** `model.invoke()`（`updater.py:494-537`）——注释（`updater.py:484-492`）解释这是为了避免踩到与 lead agent 共享的 langchain async httpx 连接池 bug（#2615）；
4. **事件循环里则卸载到线程池**：`update_memory()` 检测 `asyncio.get_running_loop()`，如果有，submit 到 `_SYNC_MEMORY_UPDATER_EXECUTOR`（4 worker `ThreadPoolExecutor`，`updater.py:34-38`，atexit 注册）；
5. **响应解析**（`updater.py:313-331`）：用 `re.finditer(r"\{", ...)` + `json.JSONDecoder().raw_decode()` 找第一个含必需顶层键 `{user, history, newFacts, factsToRemove}` 的 JSON 对象——能容忍 thinking trace 和 markdown fence；
6. `_apply_updates()`（`updater.py:600-684`）：
   - 仅当 `shouldUpdate=true` 才更新对应摘要字段；
   - 按 id 删 fact；
   - **仅当 `confidence >= config.fact_confidence_threshold`（默认 0.7）** 才加新 fact，且**按 casefold content 去重**；
   - **超过 `max_facts`（默认 100）时，按 confidence 降序保留 top N**——这是**按置信度驱逐**策略；
7. **上传洗刷**（`updater.py:337-368`）：`_strip_upload_mentions_from_memory()` 移除文件上传相关的句子——因为上传是 session 级的，记下来下次会话找不到文件。

#### 3.3.3 信号检测：识别"纠错"和"强化"

文件：`agents/memory/message_processing.py`。入队前会跑两个检测：

- `detect_correction()`（`message_processing.py:88-97`）：扫最近 6 条消息，匹配中英文纠错模式（"that's wrong"、"不对"、"你理解错了"）；
- `detect_reinforcement()`（`message_processing.py:100-109`）：扫强化模式（"perfect"、"完全正确"、"继续保持"）。仅在没检测到纠错时检查。

这些 flag 变成 prompt 提示（`_build_correction_hint()`，`updater.py:397-420`），例如：

> "Pay special attention to what the agent got wrong ... record the correct approach as a fact with category 'correction' and confidence >= 0.95"

### 3.4 怎么读：全量加载 + 置信度排序 + 贪心 token 预算

文件：`agents/memory/prompt.py:319-439`，`format_memory_for_injection()`：

1. 构建 User Context / History / Facts 几段；
2. **facts 按 confidence 降序**（`prompt.py:380-384`）；
3. **贪心 token 预算打包**（`prompt.py:388-419`）：算一次基础 token，然后逐条追加 fact，只要 `running_tokens + line_tokens <= max_tokens` 就加。本质是**按置信度的贪心背包**——高置信度优先；
4. **token 计数双模式**（`_count_tokens()`，`prompt.py:263-289`）：
   - `tiktoken`（默认，准但首次需联网下载 BPE）——有 600s 冷却失败缓存防雪崩（`prompt.py:196-240`）；
   - `char`（CJK 感知的字符估算，无需联网）；
5. 还超预算就按字符比例截断到 95%。

### 3.5 怎么注入：DynamicContextMiddleware（不在 MemoryMiddleware！）

> ⚠️ **容易搞混**：`MemoryMiddleware` **不负责注入**，它只在 `after_agent` 触发更新入队（`memory_middleware.py:52-110`，注释在 28-36 行）。

**真正注入记忆的是 `DynamicContextMiddleware`**（详见第 4 章）。注入格式是包在 `<system-reminder>` 里：

```
<system-reminder>
<memory>
...format_memory_for_injection 的输出...
</memory>

<current_date>2026-06-14, Saturday</current_date>
</system-reminder>
```

### 3.6 何时更新：两条路径，都异步

1. **主路径**：`MemoryMiddleware.after_agent()`（`memory_middleware.py:52-110`）——每轮对话后入队，去抖 30s 后批量更新；
2. **摘要前冲洗**：`memory_flush_hook`（`summarization_hook.py:12-34`）——一个 `BeforeSummarizationHook`，在摘要**删消息之前**触发，把"即将被摘要掉"的消息用 `add_nowait()` 立即入队，保证记忆抽取发生在原始消息被压缩之前。

---

### 🎯 第 3 章 经典问题与源码剖析

#### Q1：为什么不用向量数据库做语义检索？

**A**：DeerFlow 的取舍是「**记忆量级有限 + 检索精度靠 LLM 自己**」：

- 单用户的记忆上限是 `max_facts=100` + 几段摘要，全量加载也就几 KB；
- 在这种量级下，**按置信度排序 + 贪心 token 打包**比 embedding 检索更简单、更可控、更便宜（无需额外 embedding 模型和向量库运维）；
- "找相关记忆"这件事交给 LLM 自己——它读完 `<memory>` 段，自己决定哪些 fact 与当前问题相关。

这个方案的代价：记忆量上去后（比如 1000+ facts）会失效，因为 token 预算装不下。那时再考虑分层摘要或检索。**先简单后复杂**，是好的工程判断。

#### Q2：用户说"你上次理解错了"，agent 怎么记住这个纠错？

**A**：完整链路：

1. 用户发"不对，你理解错了" → `MemoryMiddleware.after_agent` 触发；
2. 入队前 `detect_correction()` 命中，`correction_detected=True`；
3. 30s 后 `MemoryUpdater` 启动，`_build_correction_hint()` 在 prompt 里加："Pay special attention to what the agent got wrong... record with category='correction', confidence>=0.95"；
4. LLM 抽取出 `{category: "correction", content: "...", confidence: 0.95, sourceError: "之前的错误做法"}`；
5. 因为 `confidence=0.95 >= 0.7`，写入 facts；
6. 下次注入时，`format_memory_for_injection` 对 correction 类有特殊格式（`prompt.py:406-407`）：
   ```
   - [correction | 0.95] 不要用 X 方式 (avoid: X 方式的描述)
   ```

#### Q3：记忆更新为什么"故意用同步 invoke"，而不是 ainvoke？

**A**：见 `updater.py:484-492` 注释。原因是 LangChain 的 async 客户端用了一个**全局共享的 httpx 连接池**。如果记忆更新和 lead agent 在同一个事件循环里 async 跑，两者会抢连接池，可能触发 #2615 那种连接复用 bug。

解法是：**同步 `invoke` + 把整个调用卸载到独立线程池**（`_SYNC_MEMORY_UPDATER_EXECUTOR`）。这样记忆更新跑在自己的线程里，用自己的同步 httpx 客户端，完全不碰 async 池。

这是个**反直觉但很重要**的工程经验：**async 不一定比 sync 快/好，关键看资源竞争**。

#### Q4：`memory_flush_hook` 为什么必须在摘要前触发？

**A**：因为摘要会把多轮原始对话**压缩成一句话**。如果先摘要再抽取记忆，LLM 看到的就是压缩后的摘要，**抽不出细节事实**（比如"用户提到了一个具体的项目代号 X"）。

所以在 `DeerFlowSummarizationMiddleware` 删消息之前（`summarization_middleware.py:209-210` 的 `_fire_hooks()`），先 `add_nowait()` 把这些原始消息塞进记忆队列——保证 LLM 抽取时看到的是完整的原始对话。

这是个**数据保全**的精妙设计：**压缩不可逆，所以要在压缩前做信息抽取**。

---

## 第 4 章 上下文工程（Context Engineering）与压缩

> "Context Engineering" 是 2025 年下半年 LLM 圈的热词，比"Prompt Engineering"更高一层：**怎么管理送进模型上下文窗口的全部内容**（system prompt、历史消息、工具 schema、工具输出、注入的提醒）。
>
> DeerFlow 在这一层做了 5 件事：① 静态 prompt + 动态 reminder 分离；② 摘要压缩；③ 工具输出预算；④ 工具 schema 延迟加载；⑤ token 归因统计。

### 4.1 DynamicContext：让 system prompt 永远静态（prefix cache 友好）

文件：`agents/middlewares/dynamic_context_middleware.py`。**目的**（模块 docstring，1-27 行）：让 system prompt 跨用户/会话完全一致，最大化 **prefix cache 命中**，同时把每用户/每轮变化的数据（记忆 + 日期）作为隐藏的 `<system-reminder>` 注入。

**"冻结快照"设计**：

1. **首轮**（`_inject`，`dynamic_context_middleware.py:177-190`）：找第一条真实 user msg，构建完整 reminder（`<memory>` + `<current_date>`），用**ID 替换技巧**插入（`_make_reminder_and_user_messages`，138-161 行）：
   - 一条新的 `reminder_msg` **复用原 msg 的 ID**（这样 LangGraph 的 `add_messages` reducer 会原地替换）；
   - 原内容变成新 `user_msg`，id 为 `"{stable_id}__user"`；
   - reminder 带 `additional_kwargs={"hide_from_ui": True, "dynamic_context_reminder": True}`。
   - **第一条消息整个会话冻结**——内容永不再变，所以每轮都能命中 prefix cache。

2. **同一天**：`return None`，啥也不做。
3. **跨午夜**：注入一条轻量的"日期更新"reminder。

**识别技巧**：用 `additional_kwargs["dynamic_context_reminder"]` 标志判断，**不靠内容子串匹配**（`is_dynamic_context_reminder`，64-66 行），避免用户消息里恰好包含 `<system-reminder>` 被误判。

**async 安全**（`abefore_agent`，210-232 行）：把 `_inject` 卸载到 `asyncio.to_thread`，**5s 超时**保护——防 tiktoken BPE 冷下载卡住。

### 4.2 Summarization：什么时候压缩、压什么、怎么插回去

文件：`agents/middlewares/summarization_middleware.py`。继承 LangChain 的 `SummarizationMiddleware`。

#### 4.2.1 何时触发

`_maybe_summarize()`（`summarization_middleware.py:195-219`）：

```python
total_tokens = self.token_counter(messages)
if not self._should_summarize(messages, total_tokens): return None
cutoff_index = self._determine_cutoff_index(messages)
if cutoff_index <= 0: return None
```

触发条件由 `SummarizationConfig.trigger`（`config/summarization_config.py:32-38`）配置，支持三种 type（可列表，OR 语义）：

- `messages`（消息数）；
- `tokens`（绝对 token）；
- `fraction`（占模型 max input 的百分比）。

默认 `enabled=False`，需显式开启。**触发时机**：`before_model`（每次调模型前），所以中间件要**早装**。

#### 4.2.2 压什么（不压什么）

两个**保留逻辑**（DeerFlow 的特色）：

1. **`_partition_with_skill_rescue()`**（272-318 行）：**最近加载的 skill 文件不被摘要**。
   - `_find_skill_bundles()`（320-380 行）：找"AIMessage + 配对 ToolMessage"组，其中工具调用读的是 `/mnt/skills` 下的文件；
   - `_select_bundles_to_rescue()`（382-408 行）：按三个预算挑选——`preserve_recent_skill_count=5`、`preserve_recent_skill_tokens=25000`、`preserve_recent_skill_tokens_per_skill=5000`；
   - 被救的 AIMessage **克隆**（拆分 tool_calls：被救的留空内容，其余留下继续被摘要）。

2. **`_preserve_dynamic_context_reminders()`**（254-270 行）：**dynamic-context 的 `<system-reminder>` 不被摘要**。

#### 4.2.3 怎么插回去

`_build_new_messages()` 覆盖（248-252 行）：

```python
return [HumanMessage(content=f"Here is a summary of the conversation to date:\n\n{summary}", name="summary")]
```

摘要是一条 `HumanMessage(name="summary")`——前端隐藏，模型可见。新消息列表：

```python
{"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES), *new_messages, *preserved_messages]}
```

`RemoveMessage(id=REMOVE_ALL_MESSAGES)` **清空全部历史**，然后塞回摘要 + 保留消息。

**摘要模型**（`_create_summary`，133-177 行）：用专门的 model 副本，打 `TAG_NOSTREAM` 标签——**不让摘要 LLM 调用流式推给前端**（否则前端会看到一个幻影 AI 消息）。

### 4.3 ToolOutputBudget：工具输出预算

文件：`agents/middlewares/tool_output_budget_middleware.py`。这是**对单次工具调用的输出做收容**，不是对话摘要。

**两层策略**（`_budget_content`，325-415 行）：

1. **外置（首选）**：`len(content) > externalize_min_chars`（默认 12000 字符）时：
   - 全量写盘（`_externalize`，120-149 行）→ 返回虚拟路径 `/mnt/user-data/outputs/...`；
   - 远程沙箱用 `_externalize_to_sandbox()` 直接写沙箱文件系统，并 `test -s ... && echo OK` 校验；
   - 内联内容替换为 `_build_preview()`（206-231 行）：**头 2000 + 尾 1000 字符**，按行边界对齐，附引用：
     ```
     [Full bash output saved to /mnt/user-data/outputs/... (12345 chars, ~3086 tokens).
      Use read_file with start_line and end_line to access specific sections. 9345 chars omitted.]
     ```
   - 模型可按需 `read_file` 具体段落。

2. **兜底截断**（无盘可用）：`_build_fallback()`（234-275 行），头 8000 + 尾 3000，max 30000。

**两个 hook 点**：

- `wrap_tool_call`（581-613 行）：**工具调用时**预算（新鲜结果）；
- `wrap_model_call`（617-643 行）：**历史 ToolMessage** 也预算（`_patch_model_messages`）。

**豁免工具**（`exempt_tools`，默认 `["read_file"]`）——防止 "持久化 → read_file → 又持久化" 死循环。

### 4.4 DeferredToolFilter：MCP 工具 schema 延迟加载

文件：`agents/middlewares/deferred_tool_filter_middleware.py`。**目的**：MCP 工具很多时，全量 schema 会**撑爆上下文**。所以默认只给模型**工具名**，模型要用了再 `tool_search` 提升详细 schema。

**机制**：

- `wrap_model_call`：`_filter_tools()` 从 `request.tools` 移除 `deferred - promoted` 的 schema；
- `wrap_tool_call`：模型若强调延迟工具，返回错误 `ToolMessage`：`"Error: Tool 'X' is deferred... Call tool_search first"`；
- 提升状态从 `state["promoted"]` 读，按 `catalog_hash` 隔离——防止持久化的过期 promotion 暴露被重命名的工具。

`tool_search` 工具（`tools/builtins/tool_search.py`）支持三种查询：

- `select:Read,Edit`（精确）；
- `notebook jupyter`（关键词正则）；
- `+slack send`（require token，按 rest 排序）；

最多返回 5 个，返回的是 `Command(update={"promoted": ...})`——**靠图状态传递 promotion**，不靠 ContextVar。

### 4.5 TokenUsage：归因统计

文件：`agents/middlewares/token_usage_middleware.py`。**只统计不强制**（硬预算在 Summarization 那里）。

两个职责：

1. **记 token**：从最后一条 AIMessage 的 `usage_metadata` 读 input/output/total；
2. **打步骤归因**（`_build_attribution`，231-264 行）：给每个 AI 步骤打 `additional_kwargs["token_usage_attribution"]`：
   ```python
   {"version": 1, "kind": "todo_update"|"subagent_dispatch"|"tool_batch"|"final_answer"|"thinking",
    "shared_attribution": bool, "tool_call_ids": [...], "actions": [...]}
   ```
   - `write_todos` → diff 出 `todo_start`/`complete`/`update`/`remove` 动作；
   - `task` → `subagent` 动作；
   - `web_search` → `search` 动作；
   - 前端靠这个精确标注每一步。
3. **子 agent token 回滚**（281-314 行）：`task` 工具完成后，按 `tool_call_id` 弹出缓存的子 agent usage，**合并回派发的 AIMessage 的 `usage_metadata`**。

---

### 🎯 第 4 章 经典问题与源码剖析

#### Q1：为什么要"静态 prompt + 动态 reminder"分离？直接把日期塞 system prompt 不行吗？

**A**：因为 LLM 提供商（OpenAI/Anthropic 等）有 **prefix cache**——如果多次请求的前缀完全一致，缓存命中，**费用减半、延迟降低**。

如果把"今天日期"写进 system prompt，那么：

- 每天 0 点一过，prompt 就变了，缓存全失效；
- 不同用户的记忆不同，prompt 也不同，缓存命中率几乎为 0。

DeerFlow 的做法是：**system prompt 跨用户跨日期完全一致**，把日期/记忆作为隐藏 reminder 注入第一条 user msg。这样 prefix cache 命中率最大化。

**对照源码**：`dynamic_context_middleware.py:177-190` 的"首轮冻结"——第一条 reminder 一旦写入，整个会话不再变。

#### Q2：摘要为什么不直接覆盖旧消息，而要用 `RemoveMessage(REMOVE_ALL)`？

**A**：因为 LangGraph 的 `messages` 字段用 `add_messages` reducer，默认是 **append + 按 id 替换**。如果你只 append 一条 summary 消息，旧的几十条消息**还在**，根本没压缩。

`RemoveMessage(id=REMOVE_ALL_MESSAGES)` 是 LangGraph 的特殊指令，告诉 reducer "清空全部"。然后你再 append summary + 保留消息，才算真正的压缩。

**对照源码**：`summarization_middleware.py:211-219`。

#### Q3：skill 文件为什么不被摘要？这不是浪费上下文吗？

**A**：因为 skill 是**指令性**的内容（"用这个工具时必须遵守 X 流程"），被摘要成一句话后，模型就**忘记怎么用工具**了，会反复犯错。

DeerFlow 的策略：**最近 5 个 skill 文件、每个最多 5000 token、总共最多 25000 token** 保留。超出才摘要。这是个**指令保全 vs 上下文成本**的权衡。

**对照源码**：`summarization_middleware.py:382-408` 的 `_select_bundles_to_rescue()`。

#### Q4：工具输出超长，为什么不直接截断，而要"外置 + 预览"？

**A**：因为**直接截断会丢信息**。比如 agent 跑了 `grep -r foo .`，输出 5 万行，截断后只看到头尾，中间的命中行全丢了，agent 不得不重跑。

DeerFlow 的"外置 + 预览"方案：

1. 全量输出写盘，返回虚拟路径；
2. 内联只给头 2000 + 尾 1000 字符的预览；
3. 模型看到预览后，**自己决定**用 `read_file(start_line, end_line)` 读哪段。

这把"读多少"的决定权交回模型，既省 token 又不丢信息。

**对照源码**：`tool_output_budget_middleware.py:206-231` 的 `_build_preview()`。

---

## 第 5 章 Runtime：Checkpointer、Store、Events、Runs（状态管理）

> 这一章讲 DeerFlow 怎么把 LangGraph 的运行时（状态持久化、事件流、运行管理）跑起来。如果你用过 LangGraph Platform，会发现 DeerFlow 的 Runtime 几乎是它的**自托管平替**。

### 5.1 Checkpointer vs Store：两个不同的持久化

这俩都是 LangGraph 的接口，DeerFlow 直接复用上游实现（不是自己重写）：

| 概念 | 接口 | 存什么 | 后端 |
|---|---|---|---|
| **Checkpointer** | `langgraph.types.Checkpointer` | 每个超级步的图状态（channels + 消息历史 + pending writes）——让 `graph.astream(...)` 能按 `thread_id` 恢复 | memory / sqlite / postgres |
| **Store** | `langgraph.store.base.BaseStore` | 跨线程长期 KV：`store.put(("threads",), thread_id, {...})`——线程列表、用户数据 | 同上 |

**关键**：`runtime/store/provider.py:51-96` 显示 **checkpointer 和 store 共用同一个 `CheckpointerConfig`**——必须用同一种持久化技术。

### 5.2 同步单例 vs 异步上下文管理器：两套并行 API

DeerFlow 给**每个**持久化 provider（checkpointer、store、stream_bridge）都写了两套 API：

| 场景 | 模式 | 代表函数 |
|---|---|---|
| **CLI / 嵌入式 SDK**（无 FastAPI） | 全局单例 + 双重检查锁 | `get_checkpointer()`（`provider.py:107-145`） |
| **FastAPI 服务端** | async context manager，无全局状态 | `async with make_checkpointer(...)`（`async_provider.py:167-202`） |

**同步单例**的双重检查锁（`provider.py:107-145`）：

```python
_checkpointer: Checkpointer | None = None
_checkpointer_lock = threading.Lock()
def get_checkpointer() -> Checkpointer:
    if _checkpointer is not None: return _checkpointer        # 快速路径
    ensure_config_loaded()
    with _checkpointer_lock:                                  # 加锁
        if _checkpointer is not None: return _checkpointer    # 二次检查
        ...
        _checkpointer = _sync_checkpointer_cm(config).__enter__()
```

**异步路径**用 `AsyncSqliteSaver` / `AsyncPostgresSaver`，配 `psycopg_pool.AsyncConnectionPool` + TCP keepalive（`async_provider.py:50-67`）。

### 5.3 Events vs Runs：三个不同的概念

| 概念 | 定义 | 代码 |
|---|---|---|
| **RunRecord** | 一次 agent 调用的内存句柄（run_id、thread_id、status、asyncio.Task、abort_event、token 统计） | `runs/manager.py:74-103` |
| **RunStatus** | `pending / running / success / error / timeout / interrupted` | `runs/schemas.py:6-14` |
| **RunEvent** | 单条持久化记录（thread_id、run_id、event_type、category、content、metadata、seq） | `runtime/events/store/base.py` |

RunEvent 的 `category` 字段区分：

- **`message`**：前端可显示（human/ai/tool 消息）；
- **`trace`**：调试/审计（生命周期、错误）；
- **`middleware`**：中间件审计（如安全终止事件）。

### 5.4 三种事件存储后端

`make_run_event_store(config)`（`runtime/events/store/__init__.py:5-23`）按 `run_events.backend` 路由：

| 后端 | 特点 |
|---|---|
| `memory` | 进程内 dict，默认 |
| `db` | SQLAlchemy async ORM，写 `run_events` 表。**单调 seq**：SQLite 用 `SELECT max(seq) FOR UPDATE`，Postgres 用 `pg_advisory_xact_lock(hashtext(thread_id))` |
| `jsonl` | 每个 run 一个文件 `.deer-flow/threads/{tid}/runs/{rid}.jsonl`，per-thread `asyncio.Lock` 串行写。**多进程会出问题**，docstring 明确说"多进程用 db" |

**用户隔离的 `AUTO` 哨兵**（`db.py:74-90`）：仓库方法接收 `user_id: str | None | _AutoSentinel = AUTO`，三态：

- `AUTO`：读 contextvar，未设置就报错；
- 显式 `str`：覆盖；
- 显式 `None`：不加 WHERE 子句（迁移/CLI 用）。

### 5.5 `run_agent`：后台异步执行器

文件：`runtime/runs/worker.py:124-437`，`async def`，跑在 `asyncio.Task` 里。步骤：

1. **建 RunJournal**（`worker.py:171-180`）作为 LangChain callback；
2. **快照运行前 checkpoint**（186-201 行）——深拷贝，用于回滚；
3. **发 `metadata` 事件**（带 run_id + thread_id）；
4. **构造 agent + 装入 Runtime**（213-230 行）：
   ```python
   Runtime(context={thread_id, run_id, app_config, "__run_journal": journal}, store=store)
   config["configurable"]["__pregel_runtime"] = runtime
   ```
5. **挂 checkpointer 和 store** 到编译后的图（271-274 行）：
   ```python
   agent.checkpointer = checkpointer
   agent.store = store
   ```
6. **流式**：`graph.astream(stream_mode=lg_modes)`——支持 `values`/`updates`/`messages`/`custom`/`tasks`/`debug`/`checkpoints`，但**跳过 `events`**（需 `astream_events`，与 `values` 不兼容）；
7. **终态分支**（`abort_event`）：
   - `rollback` → `RunStatus.error` + `_rollback_to_pre_run_checkpoint()`（回放快照 + 重放 pending writes）；
   - `interrupt` → `RunStatus.interrupted`；
   - LLM 错误 → `RunStatus.error`；
   - 否则 → `RunStatus.success`；
8. **finally**：刷 journal、持久化完成数据、同步 title、`bridge.publish_end(run_id)`、60s 后清理。

**RunManager** 还提供：

- `create_or_reject`（`manager.py:497-579`）：原子 check-then-insert，用 `multitask_strategy`（reject/interrupt/rollback）消除 TOCTOU 竞态；
- `reconcile_orphaned_inflight_runs`（581-633 行）：重启后把"持久化但无本地 task"的 run 标记为 error；
- `shutdown`（648-738 行）：drain 在飞 run 再关 checkpointer 池（修过 #3373 那种 langgraph 内部 `_checkpointer_put_after_previous` 与关池的竞态）。

所有 store 写都过 `_call_store_with_retry`（139-167 行），重试策略 `max_attempts=5, initial_delay=0.05, backoff=2, max_delay=1.0`，匹配 `"database is locked"`、`SQLITE_BUSY` 等可重试错误。

### 5.6 StreamBridge：生产者/消费者解耦

文件：`runtime/stream_bridge/base.py`。`StreamBridge` 抽象 worker（生产者）↔ SSE 端点（消费者）的通道，模仿 LangGraph Platform 的 Queue + StreamManager 分离。

契约：`publish(run_id, event, data)` / `publish_end(run_id)` / `subscribe(run_id, last_event_id)`。两个哨兵：

- `HEARTBEAT_SENTINEL`（空闲超时发）；
- `END_SENTINEL`（`publish_end` 后发一次）。

`MemoryStreamBridge`（`stream_bridge/memory.py:25-133`）是唯一实现（Redis 是 `NotImplementedError`）。per-run `_RunStream` 持有：事件列表（cap 256）、`asyncio.Condition`、`ended` flag、`start_offset`（旧事件驱逐后前进）。订阅者落后于保留窗口则从 `start_offset` 恢复。

### 5.7 RunJournal：LangChain callback → 事件存储的桥

文件：`runtime/journal.py`。`RunJournal(BaseCallbackHandler)` 夹在 LangChain 回调机制和可插拔 `RunEventStore` 之间。关键设计（docstring 7-15 行）：

- **故意不实现 `on_llm_new_token`**——只在 `on_llm_end` 记完整消息；
- 用 `on_chat_model_start`（不是 `on_chain_start`）抓首条 human msg——因为那里的消息是完整结构化的，没被 checkpoint trimming 压缩。

**调用方识别**（`journal.py:428-436`）：`_identify_caller(tags)` 找 `"lead_agent"` / `"subagent:{name}"` / `"middleware:{name}"` tag，驱动 per-caller token 分桶。

**缓冲异步刷写**：`_put` 追加到 `_buffer`，到 `flush_threshold=20`（367-380 行）就刷。`_flush_sync`（382-405 行）试 `asyncio.get_running_loop()`，有就 `create_task(_flush_async)`，并防并发（`if self._pending_flush_tasks: return`）；没有就留在 buffer 等 worker 的 `finally: await journal.flush()`。

---

### 🎯 第 5 章 经典问题与源码剖析

#### Q1：Checkpointer 和 Store 有什么区别？为什么不用一个？

**A**：

- **Checkpointer** 存的是"图执行的进度"——每个超级步的 channels、消息、pending writes。它是为了让 `graph.astream(thread_id=X)` 能**从上次中断处恢复**。结构是按 `(thread_id, checkpoint_id)` 的历史栈。
- **Store** 存的是"跨线程的长期数据"——比如线程列表、用户偏好。结构是命名空间 KV，`put((prefix,), key, value)`。

**为什么分开**？因为访问模式不同：checkpointer 是**按线程、按步**频繁读写（每次 model call 都写）；store 是**偶尔写、经常扫**（列线程列表）。分开后能各自优化索引。这也是 LangGraph 上游的设计。

#### Q2：为什么 `run_agent` 要先快照 checkpoint，再跑？

**A**：为了支持**回滚**（`multitask_strategy="rollback"`）。如果用户在 run 跑到一半时发起冲突的 run（同一 thread），策略是 rollback：把当前 run 标 error，**把状态回滚到这次 run 开始前**，再跑新的。

`_rollback_to_pre_run_checkpoint()`（`worker.py:456-546`）的做法：把快照作为新 checkpoint 重放，然后按 task_id 分组重新应用 pending writes。这是个**非平凡**的操作——你不能直接"删 checkpoint"，因为别的 run 可能引用了它；只能追加一个"回到过去"的新 checkpoint。

#### Q3：RunJournal 为什么"故意不实现 on_llm_new_token"？

**A**：因为 `on_llm_new_token` 一个 token 触发一次，写入事件存储会**洪水般刷盘**。DeerFlow 的取舍是：**流式 token 直接走 SSE 给前端**（通过 `graph.astream(stream_mode="messages")`），**事件存储只记完整消息**（`on_llm_end`）。

这样事件存储的写入频率可控（每个 LLM 调用一次），便于持久化和重放；流式体验靠 SSE 实时通道，互不干扰。

#### Q4：多进程部署为什么不能用 jsonl 事件存储？

**A**：因为 jsonl 的 **seq 是进程内计数器**（`jsonl.py` 里的 `asyncio.Lock` 也只锁单进程）。两个进程同时往同一个 `{run_id}.jsonl` 追加，会出现：

- seq 重复或乱序；
- 文件交错损坏。

docstring（`jsonl.py` 头部）明确警告："Multi-process deployments sharing the same directory will produce duplicate or non-monotonic seq values. Use `DbRunEventStore` for multi-process."

`db` 后端用 `pg_advisory_xact_lock`（Postgres）或 `SELECT max(seq) FOR UPDATE`（SQLite）保证跨进程单调，是多进程唯一正确选择。

---

## 第 6 章 安全与稳定性：Guardrails、循环检测、HITL

### 6.1 Guardrails：工具调用前的策略授权

文件：`guardrails/`。**注意**：DeerFlow 的 "guardrails" 是 **pre-tool-call 授权**，不是输出内容审核。

**契约**（`guardrails/provider.py`）：

```python
@dataclass
class GuardrailRequest:
    tool_name: str
    tool_input: dict[str, Any]
    agent_id: str | None = None

@dataclass
class GuardrailDecision:
    allow: bool
    reasons: list[GuardrailReason]
    policy_id: str | None = None

class GuardrailProvider(Protocol):
    def evaluate(self, request: GuardrailRequest) -> GuardrailDecision: ...
    async def aevaluate(self, request: GuardrailRequest) -> GuardrailDecision: ...
```

字段名（`policy_id`、reason `code` 如 `"oap.tool_not_allowed"`）对齐 **OAP（Open Agent Protocol）**。

**内置 provider**（`builtin.py:6-23`）：`AllowlistProvider`——简单的 allow/deny 工具列表，deny 时返回 `code="oap.tool_not_allowed"`。

**中间件**（`middleware.py:20-98`）：`GuardrailMiddleware` 实现 `wrap_tool_call` / `awrap_tool_call`，在工具执行前调 provider：

- **`GraphBubbleUp` 重新抛出**（63-65 行）：保留 LangGraph 的中断/暂停信号——guardrail **不能吞掉 HITL 中断**；
- **fail_closed**（默认）：provider 异常时合成 deny，`code="oap.evaluator_error"`；
- **deny** 时返回错误 `ToolMessage`（status="error"），让 agent 自己换方案。

### 6.2 LoopDetection：防止 agent 死循环

文件：`agents/middlewares/loop_detection_middleware.py`。**P0 安全中间件**，防止 agent 反复调同一工具直到撞 LangGraph 递归上限。

#### 6.2.1 双层检测

**Layer 1：基于 hash（相同调用集）**

`_hash_tool_calls`（142-160 行）把每轮的工具调用归一化为 `(name, stable_key)`，**排序后 MD5**——顺序无关。hash 追加到 per-thread 滑动窗口（`window_size=20`）：

- `count >= warn_threshold`（默认 3）→ 排队警告；
- `count >= hard_limit`（默认 5）→ 硬停。

**Layer 2：per-tool-type 频率**（399-436 行）

捕获 hash 检测漏掉的情况——比如 40 次不同的 `read_file`（路径不同，hash 不同）。每个工具类型有独立计数：默认 warn 30、hard 50。可用 `tool_freq_overrides` 给 `bash` 这种批量场景调高。

**stable key 推导**（`_stable_tool_key`，99-139 行）避免过拟合噪声：

- `read_file` → `path:{bucket}`，行号按 200 行分桶（读附近行算同一次）；
- `write_file` / `str_replace` → 全 args hash（内容敏感）；
- 其他工具 → 只取显著字段 `("path", "url", "query", "command", "pattern", "glob", "cmd")`。

#### 6.2.2 关键的 hook 顺序设计

docstring（18-38 行）讲得很清楚：

> `after_model` 在模型 emit `AIMessage(tool_calls)` 后立即触发，此时 ToolNode 还没跑，没有配对的 ToolMessage。如果这时插消息，会落在 assistant 的 tool_calls 和它们的 response 之间——OpenAI/Moonshot 下一次请求会报 `"tool_call_ids did not have response messages"`。Anthropic 还禁止中途插 SystemMessage。**所以延迟到 `wrap_model_call`，那时所有 ToolMessage 都已就位，警告追加在末尾，配对完整。**

所以：

- `after_model`（534-540 行）：检测，硬停就直接清空 tool_calls 强制文本输出（`_build_hard_stop_update`，457-475 行：清 tool_calls/function_call，flip `finish_reason: "tool_calls"→"stop"`），否则排队警告（capped 4 条/run）；
- `wrap_model_call`（579-593 行）：排空 pending warnings，追加一条 `HumanMessage(name="loop_warning")`；
- `before_agent` 清上一轮残留；`after_agent` 清本轮残留。

### 6.3 SafetyFinishReason：处理内容审核中途截断

文件：`agents/middlewares/safety_finish_reason_middleware.py` + `safety_termination_detectors.py`。

**问题**（#3028）：提供商可能 `finish_reason=content_filter`（OpenAI）/ `stop_reason=refusal`（Anthropic）/ `finish_reason=SAFETY`（Gemini）**但还返回部分 tool_calls**——LangChain 的 router 当成完整响应派发，agent 看到被截断的 `write_file`，尝试修复，又被过滤，**死循环**。

**解法**：三个 detector（`safety_termination_detectors.py`）：

- `OpenAICompatibleContentFilterDetector`：`finish_reason ∈ {"content_filter"}`；
- `AnthropicRefusalDetector`：`stop_reason == "refusal"`；
- `GeminiSafetyDetector`：`finish_reason ∈ {SAFETY, BLOCKLIST, PROHIBITED_CONTENT, SPII, RECITATION, IMAGE_SAFETY, ...}`。

中间件**排在 LoopDetection 之后**（factory 里 `after_model` 逆序，最后注册的最先看）。检测到就：克隆消息、清 tool_calls、打 `additional_kwargs["safety_termination"]={...}`、发 `safety_termination` SSE、写 `middleware:safety_termination` 审计记录。**故意不记 tool arguments**（那是被过滤的内容）。

### 6.4 Clarification：Human-in-the-Loop

文件：`agents/middlewares/clarification_middleware.py`。拦截 `ask_clarification` 工具调用（`wrap_tool_call`），格式化问题（带类型 icon：❓ missing_info / 🤔 ambiguous / 🔀 approach_choice / ⚠️ risk_confirmation / 💡 suggestion），返回 `Command(update={"messages": [tool_message]}, goto=END)`——**中断执行**，把问题抛给用户，等回答再继续。

### 6.5 Todo：防止"提前收尾"

文件：`agents/middlewares/todo_middleware.py`。继承 LangChain 的 `TodoListMiddleware`。两个额外职责：

1. **上下文丢失检测**（`before_model`，120-156 行）：摘要把原 `write_todos` 调用截掉后，模型"忘记"自己有 todo。中间件检测到 `state["todos"]` 非空但消息里没有 `write_todos` 调用，注入 `HumanMessage(name="todo_reminder")`；
2. **提前退出防护**（`after_model`，265-309 行）：模型给了最终回复（无 tool_calls）但 todo 未完成，排队 `todo_completion_reminder` + `return {"jump_to": "model"}` 强制再来一轮。Capped 2 次/run 防无限循环。

---

### 🎯 第 6 章 经典问题与源码剖析

#### Q1：循环检测为什么不直接在 `after_model` 加警告消息？

**A**：见 6.2.2——会破坏 AIMessage(tool_calls) 和 ToolMessage 的配对。这是 OpenAI/Moonshot API 的硬性校验：每个 tool_call 必须有对应的 tool response，且两者必须**相邻**（中间不能插别的角色消息）。

DeerFlow 的"延迟注入"模式（`after_model` 检测 + 排队 → `wrap_model_call` 排空 + 追加）是处理这个约束的标准答案。

#### Q2：Guardrail 拒绝工具调用时，为什么返回 ToolMessage 而不是抛异常？

**A**：因为抛异常会被 ToolErrorHandlingMiddleware 捕获，agent 看到的是一个"工具崩溃"，会**重试**——陷入"调用-崩溃-重试"循环。

返回 `ToolMessage(status="error", content="Guardrail denied: ... Choose an alternative approach.")` 让 agent **理解约束**并主动换方案。这是把 guardrail 决策**翻译成 agent 能读懂的反馈**，而不是让它瞎重试。

**对照源码**：`guardrails/middleware.py:63-88`。

#### Q3：内容审核（content_filter）截断了 tool_calls，为什么会让 agent 死循环？

**A**：完整链条：

1. 模型生成 `write_file(content="敏感内容...")` 的部分 tool_calls；
2. 提供商检测到违规，`finish_reason=content_filter`，但**已生成的 tool_calls 还在响应里**；
3. LangChain 不区分"完整 tool_calls"和"被截断的 tool_calls"，照样派发给 ToolNode；
4. `write_file` 写了个**残缺文件**；
5. 下一轮模型看到残缺文件，尝试 `str_replace` 修复；
6. 又触发 content_filter……死循环。

DeerFlow 的 `SafetyFinishReasonMiddleware` 在第 3 步之前**清空 tool_calls**，让模型看到的是"我刚才的输出被安全过滤了"，从而换思路而不是修残文件。

#### Q4：HITL 的 Clarification 怎么实现"暂停-等用户-继续"？

**A**：用 LangGraph 的 `Command(goto=END)`。`clarification_middleware.py:117-156`：

1. 模型调 `ask_clarification(question="...", options=[...])`；
2. 中间件 `wrap_tool_call` 拦截，格式化问题，构造 `ToolMessage`（id=`clarification:{tool_call_id}`）；
3. 返回 `Command(update={"messages": [tool_message]}, goto=END)`——**图执行到此结束**，run 状态变 `interrupted`；
4. 前端收到 `interrupt` 事件，弹问题给用户；
5. 用户回答 → 前端发新一轮 run，把答案作为新 HumanMessage 注入 → 图从头跑，模型看到答案继续。

这是 LangGraph 实现 HITL 的标准模式：**用图的终止来表达"暂停"，用新 run 表达"继续"**。

---

## 第 7 章 周边：工具 / MCP / Skills / Sandbox / Models

### 7.1 Tools 系统

文件：`tools/tools.py`。`get_available_tools()`（44 行）按顺序构造、按名去重：

1. config 加载的工具（按 `groups` 过滤，非 host-bash 时剥掉 bash）；
2. 内置工具（`present_file`、`ask_clarification`，条件性加 `view_image`、`skill_manage`、`task`）；
3. MCP 工具（缓存，`tag_mcp_tool` 打标）；
4. ACP 工具。

每个 config 工具靠反射 `resolve_variable(cfg.use, BaseTool)` 加载。**name 不匹配会告警**（#1803）。

**同步包装**（`tools/sync.py`）：`make_sync_tool_wrapper(coro, tool_name)` 用 `ThreadPoolExecutor(max=10)` 把 async-only 工具桥接给 sync agent caller。

### 7.2 MCP 客户端

DeerFlow 是 **MCP 客户端**（不托管 MCP server）。用 `langchain-mcp-adapters` 的 `MultiServerMCPClient`。

文件：`mcp/tools.py:182` `get_mcp_tools()`：

1. 每次从盘重读 `ExtensionsConfig.from_file()`（让 Gateway-API 编辑立即生效）；
2. 注入 OAuth headers（sse/http）；
3. `MultiServerMCPClient(servers_config, tool_interceptors=..., tool_name_prefix=True)`；
4. `await client.get_tools()` 发现 schema；
5. **stdio 工具包一层 `_make_session_pool_tool`**（107 行）——连续调用复用持久 session（按 `(server_name, thread_id)` 键，Playwright 这种有状态 server 需要）；http/sse 不包（#3203 anyio TaskGroup 跨任务清理问题）；
6. 每个工具打 sync wrapper。

**缓存**（`mcp/cache.py`）：模块单例 + `_config_mtime` 失效检查；LangGraph Studio 那种"已有循环跑"的场景用 `ThreadPoolExecutor + asyncio.run` 兜底。

### 7.3 Skills（SKILL.md）

**定义**：一个 skill = 一个目录，含 `SKILL.md` 文件。`Skill` dataclass（`skills/types.py:20`）。

**SKILL.md 格式**（`skills/parser.py:66` `parse_skill_file`）：YAML frontmatter（`---` 围栏）+ 正文。必需字段 `name`、`description`；可选 `license`、`allowed-tools`。

**存储**（`skills/storage/`）：`SkillStorage` ABC（模板方法模式，`load_skills` 等具体流程是 final，组合抽象原子操作）+ `LocalSkillStorage` 文件系统实现。`get_or_new_skill_storage()`（`storage/__init__.py:15`）单例工厂。

**安装安全**（`skills/installer.py`）：`safe_extract_skill_archive` 防 zip slip、symlink、zip bomb（512MB 上限）、macOS metadata；`_scan_skill_archive_contents_or_raise` 跑安全扫描，scripts 要显式 allow。

**斜杠激活**（`agents/middlewares/skill_activation_middleware.py`）：

- `parse_slash_skill_reference`（`skills/slash.py:29`，正则 `^/([a-z0-9-]+)(?:\s+|$)`）识别 `/skill-name task`；
- 保留名 `{"bootstrap","help","memory","models","new","status"}` 忽略；
- `_resolve_activation` 校验 skill 已装、已启用、对当前 agent 可用；
- `_read_skill_content` 读文件，**路径包含检查** `resolved_file.relative_to(resolved_root)` 防逃逸；
- **构建一条 `HumanMessage`（不是 SystemMessage！）**（`_build_activation_message`，250 行），包在 `<slash_skill_activation>` XML 信封里，插在激活目标之前，`hide_from_ui=True`。

> 💡 注意：skill 内容是作为 **HumanMessage** 注入的，不是 SystemMessage。这避免了"多条 SystemMessage"问题，也便于按 id 去重（`_has_existing_activation_for_target`）。

### 7.4 Sandbox

**抽象**（`sandbox/sandbox.py`）：`Sandbox(ABC)` 定义 `execute_command`/`read_file`/`write_file`/`glob`/`grep`/`update_file` 等。

**Provider 抽象**（`sandbox/sandbox_provider.py`）：`SandboxProvider(ABC)`：

- 类属性 `uses_thread_data_mounts`、`needs_upload_permission_adjustment`；
- 抽象方法 `acquire(thread_id) -> sandbox_id` / `get(sandbox_id)` / `release(sandbox_id)`；
- `get_sandbox_provider()` 反射加载单例。

**Local 实现**（`sandbox/local/`）：

- `LocalSandboxProvider`：通用单例（id `"local"`）+ **LRU 按线程缓存**（OrderedDict，cap 256）；
- 静态路径映射：`/mnt/skills` → host skills dir（只读）+ config 的 mounts；保留前缀 `/mnt/skills`、`/mnt/acp-workspace`、`/mnt/user-data` 不可被用户 mount 覆盖；
- 按线程映射：`/mnt/user-data/{workspace,uploads,outputs}` → `{base}/users/{user_id}/threads/{thread_id}/...`；
- `LocalSandbox._resolve_path_with_mapping` 强制 `resolved_path.relative_to(local_root)`，逃逸抛 `PermissionError(EACCES)`；
- 只读 mount 上写文件抛 `OSError(EROFS)`；
- `execute_command` 跑 `subprocess.run([shell, "-c", cmd], timeout=600)`；`_resolve_paths_in_command` 把容器路径翻译成宿主路径，`_reverse_resolve_paths_in_output` 把输出里的宿主路径翻译回容器路径——**让 LLM 看到一致的虚拟路径**。

**Remote（Docker）**：把上述路径 bind-mount 进容器，`Sandbox` 接口共享给 lead agent 和 subagent（task_tool 透传 `sandbox_state`）。

### 7.5 Models：完全配置驱动

文件：`models/factory.py:82` `create_chat_model(name, thinking_enabled, *, app_config, attach_tracing, **kwargs)`：

1. `model_config = config.get_model_config(name)`，兜底 `config.models[0]`；
2. `model_class = resolve_class(model_config.use, BaseChatModel)`——provider 靠 `use:` 反射（如 `langchain_openai:ChatOpenAI`）；
3. **thinking 配置**（128-160 行）：三种模式合并 `when_thinking_enabled` / `when_thinking_disabled`：
   - OpenAI 兼容：`extra_body.thinking.type` + `reasoning_effort`；
   - vLLM/Qwen：`extra_body.chat_template_kwargs.thinking`；
   - 原生 Anthropic：`thinking={"type": "disabled"}` 构造参数。
4. **Codex Responses API**（166-179 行）：`thinking_enabled` → `reasoning_effort`（none/low/medium/high/xhigh）；
5. **流式默认**（34-79 行）：`_enable_stream_usage_by_default` 给带 `base_url` 的 OpenAI 兼容模型开 `stream_usage=True`（否则 token 跟踪失效）；`_apply_stream_chunk_timeout_default` 注入 `stream_chunk_timeout=240s`（默认 60s 对 DeepSeek-R1/GPT-5 这种 reasoning 模型太激进）；
6. **tracing**（198-203 行）：`attach_tracing=True` 时把 Langfuse/LangSmith callback 追加到 `model.callbacks`；图根已挂的（`make_lead_agent`、`TitleMiddleware`）必须传 `False` 防重复 span。

**支持的 provider**（`models/` 目录）：claude / vllm / mindie / openai_codex 全套自定义 provider + patched_deepseek/mimo/minimax/openai/stepfun。

---

### 🎯 第 7 章 经典问题与源码剖析

#### Q1：MCP 工具太多撑爆上下文，怎么办？

**A**：DeerFlow 的方案是**延迟加载**（详见 4.4）：MCP 工具的 schema 默认**不发给模型**，只发工具名列表（`<available-deferred-tools>` 段）。模型要用时先调 `tool_search`，提升具体 schema 后才能调用。`DeferredToolFilterMiddleware` 在 `wrap_model_call` 里过滤掉未提升的工具 schema，在 `wrap_tool_call` 里拦截强调。

这把"上下文成本"和"工具发现"解耦——你可以接 100 个 MCP server，而上下文只占当前轮真正需要的几个 schema。

#### Q2：本地 sandbox 怎么做到"路径隔离"又不让 LLM 看到宿主真实路径？

**A**：靠**双向路径翻译**：

- `_resolve_paths_in_command`（命令执行前）：把 LLM 写的容器路径（`/mnt/user-data/workspace/foo.py`）翻译成宿主真实路径；
- `_reverse_resolve_paths_in_output`（命令执行后）：把输出里的宿主路径翻译回容器路径。

LLM 全程只看到 `/mnt/...` 这种虚拟路径，**不知道自己的代码实际跑在宿主的哪个目录**。这既有安全意义（不泄露宿主结构），也有一致性意义（跨 local/docker sandbox 行为一致）。

#### Q3：为什么 skill 内容用 HumanMessage 注入，不用 SystemMessage？

**A**：两个原因：

1. **避免多条 SystemMessage**：create_agent 已经用 system_prompt 参数注入了一条 SystemMessage，再追加会变成两条，某些模型行为异常；
2. **便于按 id 去重**：HumanMessage 带 `additional_kwargs[_SLASH_SKILL_ACTIVATION_KEY]=True` 标志和稳定的 `__slash_activation` id 后缀，`_has_existing_activation_for_target` 能精确检测"这条 skill 已经注入过了"，避免重复注入。

#### Q4：`stream_chunk_timeout=240s` 这个默认值改大了，为什么？

**A**：因为 reasoning 模型（DeepSeek-R1、GPT-5、o1 这种）会**先思考几十秒到几分钟**才吐第一个 token。LangChain 默认 60s，对这种模型会**误判超时断连**，导致 agent 莫名其妙失败。

DeerFlow 改成 240s，覆盖大部分 reasoning 时长。这是个**生产环境踩坑后**的调优，值得抄。

---

## 第 8 章 对外接口：FastAPI Gateway 与 SSE 流式

### 8.1 FastAPI 应用：`create_app()`

文件：`backend/app/gateway/app.py:247`。模块级 `app = create_app()`（418 行）给 uvicorn 用。

**中间件**（341-357 行，按顺序）：`AuthMiddleware`（fail-closed 兜底）→ `CSRFMiddleware`（double-submit cookie）→ 可选 `CORSMiddleware`。

**Lifespan**（161 行）：加载 config、预热 tiktoken（`token_counting="char"` 时跳过）、进入 `langgraph_runtime(app, startup_config)` async CM（装 StreamBridge + RunManager + checkpointer + store）、`_ensure_admin_user`（首启建管理员 + 孤儿线程迁移）、启 IM channel service。

**路由**（361-403 行）：`models`/`mcp`/`memory`/`skills`/`artifacts`/`uploads`/`threads`/`agents`/`suggestions`/`channels`/`assistants_compat`/`auth`/`feedback`/`thread_runs`/`runs`。`/health` 返回 `{"status": "healthy"}`。

### 8.2 Threads CRUD（`/api/threads`）

| 方法 | 路由 | 函数 | 权限 |
|---|---|---|---|
| POST | `` | create_thread | — |
| POST | `/search` | search_threads | — |
| GET | `/{id}` | get_thread | `threads:read` + owner_check |
| PATCH | `/{id}` | patch_thread | `threads:write` + owner_check |
| DELETE | `/{id}` | delete_thread | `threads:delete` + owner_check |
| GET/POST | `/{id}/state` | get/update_thread_state | read/write |
| POST | `/{id}/history` | get_thread_history | read |

后端同时用 `thread_store`（ThreadMetaStore：sqlite/postgres/memory）和 `checkpointer`（LangGraph）。状态从 checkpoint 的 `pending_writes`/`tasks` 派生（`_derive_thread_status`）。

**安全细节**：`_SERVER_RESERVED_METADATA_KEYS = {"owner_id", "user_id"}`——客户端传的 metadata 里这些字段会被剥离，**防身份伪造**。`update_thread_state` 写新 checkpoint 用 uuid6（time-ordered），是 INSERT 不是 REPLACE。

### 8.3 Runs：SSE 流式

两套路由：

- **`/api/threads/{thread_id}/runs/*`**（`thread_runs.py`）：有 thread 上下文；
- **`/api/runs/*`**（`runs.py`）：无状态，没 thread_id 自动建临时 uuid4。

`RunCreateRequest`（`thread_runs.py:37`）镜像 LangGraph Platform 协议：`assistant_id`、`input`、`command`、`metadata`、`config`、`context`（DeerFlow 扩展：`model_name`、`thinking_enabled`、`reasoning_effort`、`is_plan_mode`、`subagent_enabled`、`max_concurrent_subagents`、`agent_name`、`is_bootstrap`）、`stream_mode`、`multitask_strategy`、`on_disconnect`、`on_completion`。

**关键端点**：

- `POST /{thread_id}/runs/stream` → `stream_run`，返回 `StreamingResponse` + `Content-Location` 头；
- `POST /{thread_id}/runs/wait` → 阻塞到完成，序列化最终 checkpoint；
- `POST /{thread_id}/runs/{run_id}/cancel` 带 `action=interrupt|rollback`、`wait=true|false`；
- `GET|POST /{thread_id}/runs/{run_id}/stream` → 加入现有 run 的流，或 cancel-then-stream。

### 8.4 SSE 流式机制

文件：`backend/app/gateway/services.py`。

- **`start_run()`**（278 行）：解析 `agent_factory = make_lead_agent`（所有 `assistant_id` 都映射到同一工厂，路由靠 `cfg["agent_name"]`）；校验模型白名单（309 行）；enforce 线程 ownership（329-331 行）；`run_mgr.create_or_reject` 建 `RunRecord`；`build_run_config` + 合并 context 覆盖；`asyncio.create_task(run_agent(...))`。
- **`format_sse(event, data, event_id)`**（47 行）：字段顺序 `event:` → `data:` → 可选 `id:` → 空行。匹配 LangGraph Platform wire format（`useStream` React hook 消费）。
- **`sse_consumer()`**（401 行）：async generator。读 `Last-Event-ID` 头做续传；消费 `bridge.subscribe(run_id, last_event_id)`；`HEARTBEAT_SENTINEL` → `": heartbeat\n\n"` 注释；`END_SENTINEL` → `event: end` + return；轮询 `request.is_disconnected()`；finally 在 `on_disconnect=cancel` 时取消 run。
- **`wait_for_run_completion`**（435 行）：只在观测到 `END_SENTINEL` 时返回 True，避免序列化半成品 checkpoint（#3265）。

### 8.5 Auth

`@require_permission(resource, action, owner_check=False)` 装饰器。`Permissions`：`THREADS_READ/WRITE/DELETE`、`RUNS_CREATE/READ/CANCEL`。`AuthContext.has_permission` 查 `"resource:action"` 成员。owner_check 路径用 `thread_store.check_access` 按认证用户过滤。IM channel 代替用户行动时是 internal system role，豁免 owner_check。

`inject_authenticated_user_context`（`services.py:166`）把服务端 `user_id` 盖进 `config["context"]`——让后台工具在请求 handler 返回后仍能持久化用户级文件。

---

### 🎯 第 8 章 经典问题与源码剖析

#### Q1：为什么 SSE 的 `format_sse` 要严格控制字段顺序？

**A**：因为前端用 LangGraph 的 `useStream` React hook，它按 SSE 规范解析：`event:` 行声明事件类型，`data:` 行是 JSON payload，`id:` 行是用于断线重连的 `Last-Event-ID`。顺序乱了 hook 会解析失败。

DeerFlow 故意复刻 LangGraph Platform 的 wire format，这样前端可以**直接用官方 SDK**，不用自己写客户端。

#### Q2：客户端断连了，run 怎么办？

**A**：看 `on_disconnect` 配置：

- `"cancel"`（默认）：`sse_consumer` 的 finally 块检测到 `request.is_disconnected()` 后取消 run（设 abort_event）；
- `"continue"`：run 继续在后台跑，客户端可以用 `Last-Event-ID` 重连 `GET /runs/{id}/stream` 续上。

`wait_for_run_completion` 还会**只在看到 END_SENTINEL 时返回 True**——避免客户端断连时序列化半成品 checkpoint 让前端看到错误状态（#3265）。

#### Q3：为什么所有 `assistant_id` 都映射到同一个 `make_lead_agent`？

**A**：因为 DeerFlow 的"多 agent"不是"多个独立 graph"，而是**一个 graph + 配置路由**。`make_lead_agent(config)` 接收 `config["configurable"]["agent_name"]`，内部根据它决定：

- 加载哪个 SOUL.md（人格）；
- 装哪些 skill；
- 是否加 `update_agent` 工具；
- 用什么 system prompt。

所以"创建一个新 agent"不是部署一个新服务，而是**写一份 SOUL.md + config 片段**。这种"配置即 agent"的设计让 agent 管理非常轻量。

#### Q4：`multitask_strategy` 的三种策略有什么区别？

**A**：当同一 thread 已有 run 在跑，又来一个 run 请求时：

- **`reject`**：直接拒绝新请求（409）；
- **`interrupt`**：中断旧 run（设 interrupted），再跑新的；
- **`rollback`**：把旧 run 标 error 并**回滚到它开始前的 checkpoint**，再跑新的。

`create_or_reject`（`manager.py:497-579`）用原子 check-then-insert 消除 TOCTOU 竞态——避免"检查时没冲突，插入时冲突了"。

---

## 附录 A 推荐的学习顺序与动手实验

### A.1 第一周：跑起来 + 通读主线

1. **跑起来**：按 `Install.md` 用 Docker 启动，在前端发几条消息，观察 SSE 流（浏览器 DevTools 的 Network 面板）。
2. **通读主线代码**（按顺序）：
   - `backend/langgraph.json`（5 行，入口声明）
   - `agents/lead_agent/agent.py`（很短，re-export）
   - `agents/lead_agent/prompt.py`（读 SYSTEM_PROMPT_TEMPLATE，理解 prompt 工程）
   - `agents/factory.py:_make_lead_agent`（agent 装配总装线）
   - `agents/thread_state.py`（状态结构）
3. **实验**：在 prompt.py 里改一行 system prompt（比如加一句"回答时先说喵"），重启服务，验证生效。

### A.2 第二周：中间件管道

1. **通读**：`agents/middlewares/__init__.py` + `factory.py:build_middlewares`（看顺序）。
2. **精读 5 个核心中间件**：
   - `dynamic_context_middleware.py`（注入）
   - `summarization_middleware.py`（压缩）
   - `loop_detection_middleware.py`（防循环）
   - `clarification_middleware.py`（HITL）
   - `memory_middleware.py`（触发记忆）
3. **实验**：写一个最简单的自定义中间件，在 `after_model` 里把每条 AIMessage 的 token 数打到日志，挂到 `extra_middleware`。

### A.3 第三周：多 Agent 调度

1. **通读**：`subagents/` 整个目录。
2. **精读**：
   - `tools/builtins/task_tool.py`（委派工具的全流程）
   - `subagents/executor.py:_aexecute`（子 agent 执行循环）
   - `subagents/status_contract.py`（状态契约）
3. **实验**：在 `subagents/builtins/` 下仿照 `general_purpose.py` 写一个自己的 subagent（比如"代码审查专家"），在 config.yaml 注册，让 lead agent 派任务给它。

### A.4 第四周：记忆 + 上下文

1. **通读**：`agents/memory/` 整个目录。
2. **精读**：
   - `memory/updater.py`（LLM 抽取）
   - `memory/prompt.py:format_memory_for_injection`（注入格式化）
   - `memory/queue.py`（去抖队列）
3. **实验**：把 `max_facts` 改成 5，跟 agent 聊 20 轮，观察 fact 怎么被按置信度驱逐。

### A.5 第五周：Runtime + 持久化

1. **通读**：`runtime/` 整个目录。
2. **精读**：
   - `runtime/runs/worker.py:run_agent`（后台执行器）
   - `runtime/checkpointer/provider.py` vs `async_provider.py`（同步单例 vs 异步 CM）
   - `runtime/journal.py`（事件记录）
3. **实验**：把 checkpointer 从 memory 切到 sqlite，重启服务，验证会话能恢复。

### A.6 第六周：对外接口 + 部署

1. **通读**：`backend/app/gateway/`。
2. **精读**：
   - `app.py:create_app` + lifespan
   - `routers/thread_runs.py` + `services.py:sse_consumer`
3. **实验**：用 `curl` 直接发 SSE 请求，观察事件流；用 `Last-Event-ID` 头测试断线重连。

---

## 附录 B 术语表

| 术语 | 含义 |
|---|---|
| **Harness** | "外壳/框架"，DeerFlow 自称 super agent harness，强调它是个"装载和编排 agent 的框架" |
| **Lead Agent** | 主智能体，整个系统的编排者，由 `make_lead_agent` 创建 |
| **Subagent** | 子智能体，由 lead agent 通过 `task` 工具委派任务 |
| **Thread** | 会话，由 `thread_id` 标识，状态持久化在 checkpointer |
| **Run** | 一次 agent 调用，由 `run_id` 标识，对应一次 `graph.astream` |
| **Checkpoint** | LangGraph 的状态快照，按超级步存，支持恢复 |
| **Store** | LangGraph 的跨线程长期 KV 存储 |
| **Middleware** | Agent 中间件，在 model/tool 调用前后插逻辑 |
| **Reducer** | LangGraph 字段合并函数，处理并发写入 |
| **Skill** | 一份 SKILL.md 文件，定义一项可被斜杠激活的能力 |
| **SOUL.md** | 自定义 agent 的"灵魂文件"，定义人格和系统提示 |
| **Sandbox** | 工具执行环境（本地/Docker），提供路径隔离 |
| **MCP** | Model Context Protocol，工具协议标准 |
| **Deferred Tool** | 延迟加载 schema 的 MCP 工具，需先 `tool_search` |
| **HITL** | Human-in-the-Loop，人在回路（如 clarification） |
| **Guardrail** | 工具调用前的策略授权（非内容审核） |
| **Prefix Cache** | LLM 提供商的前缀缓存，prompt 一致时费用减半 |
| **Debounce** | 去抖，事件累积到一定时间/数量才处理 |
| **TOCTOU** | Time-of-Check-to-Time-of-Use，检查-使用竞态 |

---

## 结语：从 DeerFlow 学到的 Agent 工程原则

通读完全部源码，你会提炼出几条**可迁移的 Agent 工程原则**：

1. **静态 prompt + 动态 reminder 分离**——为 prefix cache，也为可维护性；
2. **异步 + 轮询 + 协作式取消**——长任务的 SSE 友好模式；
3. **延迟注入**——尊重 "AIMessage(tool_calls) ↔ ToolMessage 配对"的硬约束；
4. **结构化契约代替字符串匹配**——status_contract 是经典案例；
5. **配置即 agent**——一个 graph + 配置路由，而非 N 个服务；
6. **没有向量库也能做长期记忆**——量级有限时，LLM 抽取 + 置信度排序 + 贪心打包更简单可控；
7. **隔离的事件循环**——既不阻塞主循环，又复用连接池；
8. **同步单例 + 异步 CM 两套 API**——CLI 和服务端各取所需；
9. **压缩前先抽取**——摘要不可逆，信息保全要在压缩前；
10. **fail-closed + 翻译成 agent 能懂的反馈**——guardrail 拒绝时返回 ToolMessage 而非抛异常。

把这些原则内化，你不仅能读懂 DeerFlow，也能在自己的 Agent 项目里做出更稳健的设计。

**Happy hacking! 🦌**
