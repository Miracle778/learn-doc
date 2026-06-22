# 第 0 章 + 第 1 章 超详细精讲版（手把手带读源码）

> 这份文档是给"看完总览还是懵"的同学准备的。
> 我们会用**对话式**的方式，**一行一行**过代码，把每个概念掰开揉碎。
>
> **阅读方法**：每一段代码我都配了"这是什么"和"为什么这样写"。
> 看到代码先自己读一遍，猜猜在干嘛，再看下面的解释。
>
> **配套文件**：建议同时打开这些源码文件对照看：
> - `backend/langgraph.json`
> - `backend/packages/harness/deerflow/agents/lead_agent/agent.py`
> - `backend/packages/harness/deerflow/agents/thread_state.py`
> - `backend/packages/harness/deerflow/agents/lead_agent/prompt.py`（只看 364 行附近）

---

# 🎯 先建立一个心理模型：DeerFlow 到底在干嘛？

在讲代码之前，先用**最朴素的语言**讲清楚 DeerFlow 在做什么。

## 一个比喻：DeerFlow 是一个"超能员工公司"

想象你开了一家公司，要接待各种客户需求。你不是雇一个全能员工，而是这么组织：

```
                    客户（浏览器用户）
                          │
                          ▼
              ┌───────────────────────┐
              │  前台（FastAPI 服务）   │  ← 接电话、登记、安排会议室
              └───────────┬───────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  项目经理（Lead Agent） │  ← 一个超级聪明的 LLM
              │                       │     会读、会写、会用工具
              │  自己能干的：直接干      │
              │  干不动的：派给下属      │
              └───┬───────────────┬───┘
                  │               │
        自己用工具 │               │ 派任务
        (bash等)   │               │
                  ▼               ▼
          ┌──────────────┐  ┌──────────────────┐
          │ 工具箱         │  │ 下属（Subagents） │
          │ bash/read_file│  │ general-purpose   │
          │ web_search... │  │ bash 专家         │
          └──────────────┘  └──────────────────┘
```

**关键认知**：

1. **整个公司对外只有一张脸**——前台。客户不管你内部怎么分工，他只发消息、收回复。
2. **项目经理（Lead Agent）是核心**。它是一个 LLM（比如 GPT-4 或 Claude），但它**不是孤零零地说话**，而是被"装配"了一堆"中间件"（可以理解成"工作守则"）。
3. **下属（Subagent）也是 LLM**，但能力被限制（比如 bash 专家只会跑命令），由项目经理通过一个叫 `task` 的工具委派。
4. **所有人共享一个"档案柜"**——记忆（用户偏好）、工具箱、工作目录。

DeerFlow 这个"公司"的代码，核心就是回答一个问题：

> **「一个 LLM 怎么被'装配'成一个能干活的 Agent？」**

第 0 章和第 1 章就是回答这个问题的。我们开始。

---

# 📖 第 0 章 重新讲：架构全景

## 0.1 从一个用户消息说起

假设你在浏览器里输入："帮我看看 /tmp 目录下有什么文件"。

会发生什么？我用**时间线**讲：

```
[T=0]   浏览器发 POST /api/threads/abc/runs/stream
        body: {"input": {"messages": [{"role":"user","content":"帮我看看 /tmp 目录下有什么文件"}]},
               "config": {"configurable": {"model_name": "gpt-4o"}},
               "context": {"agent_name": null},
               "stream_mode": ["values","messages"]}

[T=1]   FastAPI 收到请求 → AuthMiddleware 鉴权 → 路由到 thread_runs.py:stream_run
        → 调 services.py:start_run()

[T=2]   start_run() 做几件事：
        a) 解析 assistant_id（不管填啥都映射到 make_lead_agent）
        b) 校验 model_name 是否在白名单
        c) 校验线程归属（这线程是你的吗？）
        d) run_mgr.create_or_reject(...) 创建 RunRecord（原子操作，防并发冲突）
        e) asyncio.create_task(run_agent(...))  ← 在后台开一个协程跑 agent

[T=3]   worker.py:run_agent() 启动：
        a) 建 RunJournal（记事件用）
        b) 快照当前 checkpoint（万一要回滚）
        c) 调 agent_factory(config) —— 这就是 make_lead_agent(config)
           ★ 这里"造"出一个 agent 实例（一个 LangGraph 图）
        d) 把 checkpointer、store 挂到图上；把 RunJournal 挂到 callbacks + context
        e) graph.astream(input, stream_mode=...)  ← 开流！

[T=4]   ★★★ 重点：make_lead_agent(config) 内部发生了什么？★★★
        这就是第 1 章的主题。简单说：
        - 读 config 决定用哪个模型、开哪些开关
        - 装配一堆工具（bash、read_file、web_search...）
        - 装配一堆中间件（管压缩、管循环、管澄清...）
        - 拼装 system prompt
        - 调用 langchain.agents.create_agent(...) 造出图

[T=5]   图跑起来了，进入 ReAct 循环：
        模型 → "我得用 ls 工具" → ToolNode 执行 ls → 结果回模型
        → 模型 → "好，我看到了 /tmp 下有 a.txt、b.log，回复用户"

[T=6]   每一步都通过 graph.astream 流式吐 chunk → StreamBridge → SSE → 浏览器
        浏览器实时显示"正在思考...""调用 ls...""/tmp 下有 a.txt..."

[T=7]   图结束，worker.py 收尾：
        - 把最终状态存进 checkpointer（下次还能恢复）
        - RunJournal 把 buffer 刷干净，汇总 token 用量
        - 发 end 事件
        - MemoryMiddleware 触发：把这段对话塞进记忆队列（30s 后抽取）
```

**重点来了**：T=4 那一步——"造 agent"——就是整个系统的**装配车间**，也是第 1 章的全部内容。我们接下来就钻进 `make_lead_agent` 看它到底怎么"造"。

## 0.2 三个一定要先懂的概念

在第 1 章代码里，有三个词会反复出现。先建立直觉：

### 概念 ①：Graph（图）——LangGraph 的核心抽象

**别被"图"吓到**。你可以理解成一张**状态机流程图**：

```
   ┌─────────┐
   │  START   │
   └────┬─────┘
        ▼
   ┌─────────┐  调模型    ┌─────────┐
   │  model  │ ─────────▶ │  tools  │
   │  node   │ ◀───────── │  node   │
   └────┬─────┘  返回结果  └─────────┘
        │ 模型说"我做完了"
        ▼
   ┌─────────┐
   │   END    │
   └─────────┘
```

- **节点（Node）**：干活的函数。ReAct Agent 只有两个节点：`model`（调 LLM）和 `tools`（执行工具）。
- **边（Edge）**：节点之间的跳转。`model` 调完看输出——如果模型要求调工具，跳到 `tools`；否则跳到 `END`。
- **状态（State）**：在节点之间流动的数据。DeerFlow 用 `ThreadState`（一会儿细讲）。

**关键认知**：所谓"Agent"，在 LangGraph 里就是**这张图的一个完整运行**。每跑一次 `graph.astream(...)`，就是从 START 走到 END 的一次旅程。

### 概念 ②：Middleware（中间件）——洋葱模型

中间件是 LangChain 1.0 引入的概念。想象一个洋葱：

```
              ┌─────────────────────────────┐
              │   ClarificationMiddleware    │  ← 最外层
              ├─────────────────────────────┤
              │ SafetyFinishReasonMiddleware │
              ├─────────────────────────────┤
              │    LoopDetectionMiddleware   │
              ├─────────────────────────────┤
              │           ...               │
              ├─────────────────────────────┤
              │       【模型 / 工具】         │  ← 洋葱芯
              └─────────────────────────────┘
```

每次要调模型（或工具），请求都要**穿过所有中间件**，每层中间件可以：

- **改请求**：比如往消息列表里加一条提醒；
- **改响应**：比如检测模型输出是不是在死循环；
- **拦截**：比如 guardrail 拒绝某个工具调用；
- **完全旁路**：比如 ClarificationMiddleware 看到 `ask_clarification` 就直接中断整个流程。

**每个中间件有 6 个"插嘴"位置**（hook）：

```
before_agent → before_model → [wrap_model_call] → after_model
                                    ↓
                            [wrap_tool_call]
                                    ↓
                              after_agent
```

- `before_agent` / `after_agent`：整个 run 的开始/结束（一次）；
- `before_model` / `after_model`：每次调模型前后（多次，因为 ReAct 会循环）；
- `wrap_model_call` / `wrap_tool_call`：包裹模型/工具的调用本身，能改 request/response。

记住这个比喻，等会儿看代码时你会发现 DeerFlow 的每个中间件都挂在这些 hook 上。

### 概念 ③：RunnableConfig——运行时配置的"大口袋"

`RunnableConfig` 是 LangChain 的一个 `TypedDict`，长这样（简化）：

```python
{
    "configurable": {...},   # 普通配置（thread_id、model_name 等）
    "context": {...},        # LangGraph 运行时上下文（DeerFlow 用它传 app_config 等）
    "metadata": {...},       # 元数据（用于 trace）
    "callbacks": [...],      # 回调（tracing、journal）
    "tags": [...],           # 标签
}
```

你可以理解成**一个贯穿整个 run 的"大口袋"**，谁都能往里塞东西、从里取东西。

DeerFlow 的 `make_lead_agent(config: RunnableConfig)` 接收的就是这个口袋。等会儿你看代码时会看到大量 `cfg.get("xxx")`，都是在从这个口袋里掏东西。

---

好，三个概念建立了。现在我们正式开始读代码。🚀

---

# 📖 第 1 章 重新讲：Agent 是怎么造出来的

## 1.1 第一步：看入口文件 `langgraph.json`

**文件**：`backend/langgraph.json`（只有 17 行）

```json
{
  "$schema": "https://langgra.ph/schema.json",
  "python_version": "3.12",
  "dependencies": ["."],
  "env": ".env",
  "graphs": {
    "lead_agent": "deerflow.agents:make_lead_agent"
  },
  "auth": {
    "path": "./app/gateway/langgraph_auth.py:auth"
  },
  "checkpointer": {
    "path": "./packages/harness/deerflow/runtime/checkpointer/async_provider.py:make_checkpointer"
  }
}
```

**逐行翻译**：

| 字段 | 含义 |
|---|---|
| `python_version` | 用 Python 3.12 |
| `dependencies: ["."]` | 依赖当前目录（即 backend 包） |
| `env: ".env"` | 环境变量文件 |
| `graphs.lead_agent` | ★ **关键**：声明一个叫 `lead_agent` 的图，工厂函数是 `deerflow.agents:make_lead_agent` |
| `auth.path` | 鉴权钩子的位置 |
| `checkpointer.path` | checkpointer 工厂的位置 |

**最重要的认知**：

> 🎯 **整个 DeerFlow 对外只暴露"一个图"——`lead_agent`。**
>
> 你在浏览器里跟 DeerFlow 的每一次对话，最终都会调用 `make_lead_agent(config)` 来"造一个 agent 实例"跑这一轮。
>
> 所谓"多 Agent 系统"，其实是**这个图在运行时通过 `task` 工具内部派生子任务**，不是"多个独立的图"。

这个认知非常关键，会贯穿后面所有章节。

> 💡 **小知识**：`deerflow.agents:make_lead_agent` 这种写法叫 "import path"，意思是"从 `deerflow.agents` 模块导入 `make_lead_agent` 这个对象"。冒号前面是模块路径，后面是对象名。

## 1.2 第二步：跟着 import 找到 `make_lead_agent`

`langgraph.json` 说入口是 `deerflow.agents:make_lead_agent`。我们打开 `deerflow/agents/__init__.py`……（其实它是 `lead_agent/__init__.py` 再 re-export 的，链路是 `agents/__init__.py` → `lead_agent/__init__.py` → `lead_agent/agent.py`）。

**文件**：`backend/packages/harness/deerflow/agents/lead_agent/__init__.py`（只有 3 行）

```python
from .agent import make_lead_agent

__all__ = ["make_lead_agent"]
```

就是个 re-export。真正的代码在 `lead_agent/agent.py`。

## 1.3 第三步：读 `agent.py` 的模块文档（最上面那段）

**文件**：`backend/packages/harness/deerflow/agents/lead_agent/agent.py` 第 1-19 行

```python
"""Lead agent factory.

INVARIANT — tracing callback placement
======================================

Tracing callbacks (Langfuse, LangSmith) are attached at the **graph
invocation root** in :func:`_make_lead_agent` (see the
``build_tracing_callbacks()`` block that appends to ``config["callbacks"]``).
Every ``create_chat_model(...)`` call inside this module — and inside any
middleware reachable from this graph (e.g. ``TitleMiddleware``) — MUST pass
``attach_tracing=False``.
"""
```

**翻译成人话**：

> 这段文档定义了一个**铁律**（INVARIANT，不变量）：
>
> "Tracing 回调（用于把执行轨迹发给 Langfuse / LangSmith 这类观测平台）**只能挂在图的根节点一次**，挂在 `_make_lead_agent` 里。本模块以及图能到达的任何中间件（如 `TitleMiddleware`）里，凡是调 `create_chat_model(...)` 的地方，**必须传 `attach_tracing=False`**。"

**为什么这是个铁律？**

想象一下：

- `make_lead_agent` 造图时，会给图本身挂一套 tracing 回调（在根节点）；
- 图跑起来，会调模型——模型是 `create_chat_model(...)` 造的；
- 如果 `create_chat_model` **也**自己挂一套 tracing 回调（`attach_tracing=True`），就会**重复**：同一个 LLM 调用，根节点记一次、模型自己又记一次 → 你的 trace 面板里出现两条重复的 span。

更糟的是，Langfuse 的 handler 必须看到"根节点的开始"才会把 `session_id` / `user_id` 这些字段传到 trace 上。如果只在模型层挂，handler 看不到根节点，字段就丢了。

**结论**：tracing 必须挂一次、挂在根。这是 DeerFlow 团队踩坑后总结的工程纪律。代码里那四个 `attach_tracing=False` 调用点（bootstrap agent、default agent、summarization 中间件、TitleMiddleware）都要遵守。

> 📌 **学习提示**：读源码时，遇到全大写的 `INVARIANT` / `MUST` / `CRITICAL` 这种字眼，一定要停下来认真读——这是作者在用最强烈的语气提醒你"别动这里"。

## 1.4 第四步：看 `make_lead_agent` 函数（公开入口）

**文件**：`lead_agent/agent.py:402-406`

```python
def make_lead_agent(config: RunnableConfig):
    """LangGraph graph factory; keep the signature compatible with LangGraph Server."""
    runtime_config = _get_runtime_config(config)
    runtime_app_config = runtime_config.get("app_config")
    return _make_lead_agent(config, app_config=runtime_app_config or get_app_config())
```

**就 4 行**。它是 `langgraph.json` 注册的工厂函数，所以签名必须符合 LangGraph Server 的约定（只接收一个 `RunnableConfig`）。

它做两件事：

1. `_get_runtime_config(config)`：把 config 里的 `configurable` 和 `context` 合并成一个 dict（一会儿看这个函数）；
2. 调 `_make_lead_agent(config, app_config=...)`：真正的装配逻辑在私有函数里。

**为什么拆成 `make_lead_agent` 和 `_make_lead_agent` 两层？**

因为 `make_lead_agent` 的签名被 `langgraph.json` 钉死了（必须 `(config) -> graph`），但实际装配需要更多参数（比如测试时想注入 mock 的 `app_config`）。所以公开入口只做"参数解析"，私有函数做"真正的活"。这是个常见的工厂模式。

### 1.4.1 看一眼 `_get_runtime_config`

**文件**：`lead_agent/agent.py:55-61`

```python
def _get_runtime_config(config: RunnableConfig) -> dict:
    """Merge legacy configurable options with LangGraph runtime context."""
    cfg = dict(config.get("configurable", {}) or {})
    context = config.get("context", {}) or {}
    if isinstance(context, dict):
        cfg.update(context)
    return cfg
```

**它在干嘛**：把 `config["configurable"]`（传统 LangChain 配置位）和 `config["context"]`（LangGraph 新的运行时上下文位）**合并**成一个 dict。

**为什么这么做**？因为 LangGraph 的 `context` 是较新的字段，DeerFlow 为了兼容老用法，让两边都能传参。比如：

- 客户端可以传 `config.configurable.model_name = "gpt-4o"`（老风格）；
- 也可以传 `context.model_name = "gpt-4o"`（新风格）；
- 合并后都进 `cfg["model_name"]`。

## 1.5 第五步：进入 `_make_lead_agent`——装配车间

这是**最核心**的函数。我们一段一段读。

### 1.5.1 第一段：解析开关（agent.py:415-425）

```python
def _make_lead_agent(config: RunnableConfig, *, app_config: AppConfig):
    # Lazy import to avoid circular dependency
    from deerflow.tools import get_available_tools
    from deerflow.tools.builtins import setup_agent, update_agent
    from deerflow.tools.builtins.tool_search import assemble_deferred_tools

    cfg = _get_runtime_config(config)
    resolved_app_config = app_config

    thinking_enabled = cfg.get("thinking_enabled", True)
    reasoning_effort = cfg.get("reasoning_effort", None)
    requested_model_name: str | None = cfg.get("model_name") or cfg.get("model")
    is_plan_mode = cfg.get("is_plan_mode", False)
    subagent_enabled = cfg.get("subagent_enabled", False)
    max_concurrent_subagents = cfg.get("max_concurrent_subagents", 3)
    is_bootstrap = cfg.get("is_bootstrap", False)
    agent_name = validate_agent_name(cfg.get("agent_name"))
```

**这段在干嘛**：从 `cfg`（运行时配置）里掏出一堆"开关"。

**逐个开关讲**：

| 开关 | 默认值 | 含义 |
|---|---|---|
| `thinking_enabled` | `True` | 是否开"思考模式"（reasoning model，如 o1/Claude thinking） |
| `reasoning_effort` | `None` | 思考强度（low/medium/high），仅某些模型支持 |
| `requested_model_name` | — | 客户端指定的模型名（兼容 `model_name` 和 `model` 两种 key） |
| `is_plan_mode` | `False` | 是否开"计划模式"（带 TodoList） |
| `subagent_enabled` | `False` | 是否允许 lead agent 派子任务 |
| `max_concurrent_subagents` | `3` | 单轮最多并行派几个子任务 |
| `is_bootstrap` | `False` | 是否是"引导 agent"（用于首次创建自定义 agent） |
| `agent_name` | — | 自定义 agent 的名字（普通对话是 `None`） |

**注意第一行注释**：

```python
# Lazy import to avoid circular dependency
```

这是"延迟导入"，避免循环依赖。为什么会有循环依赖？因为 `tools` 模块可能反过来 import `agents` 模块的东西，如果在文件顶部 import，就会"鸡生蛋蛋生鸡"。把它放到函数内部，调用时才 import，就绕开了。

> 📌 **学习提示**：Python 项目里看到 `def func(): from xxx import yyy` 这种写法，99% 是为了避免循环依赖。这是个常见技巧。

### 1.5.2 第二段：加载 agent 配置 + 解析模型（agent.py:427-441）

```python
    agent_config = load_agent_config(agent_name) if not is_bootstrap else None
    available_skills = _available_skill_names(agent_config, is_bootstrap)
    # Custom agent model from agent config (if any), or None to let _resolve_model_name pick the default
    agent_model_name = agent_config.model if agent_config and agent_config.model else None

    # Final model name resolution: request → agent config → global default, with fallback for unknown names
    model_name = _resolve_model_name(requested_model_name or agent_model_name, app_config=resolved_app_config)

    model_config = resolved_app_config.get_model_config(model_name)

    if model_config is None:
        raise ValueError("No chat model could be resolved. ...")
    if thinking_enabled and not model_config.supports_thinking:
        logger.warning(f"Thinking mode is enabled but model '{model_name}' does not support it; fallback to non-thinking mode.")
        thinking_enabled = False
```

**这段在干嘛**：

1. 如果是自定义 agent（`agent_name` 非空），从磁盘加载它的配置（`load_agent_config`，去读 SOUL.md / config.yaml）；
2. 算出这个 agent 能用哪些 skill（`_available_skill_names`）；
3. **模型名三级解析**：客户端请求 → agent 配置 → 全局默认（看 `_resolve_model_name`）；
4. 兜底校验：模型不存在就报错；模型不支持 thinking 但用户开了 thinking，就关掉 thinking 并警告。

**"三级解析"是个常见模式**——优先级从高到低：用户每次请求指定的 > agent 配置里写的 > config.yaml 的全局默认。这样既能给用户灵活性，又有合理默认。

我们看一眼 `_resolve_model_name`：

```python
def _resolve_model_name(requested_model_name: str | None = None, *, app_config: AppConfig | None = None) -> str:
    app_config = app_config or get_app_config()
    default_model_name = app_config.models[0].name if app_config.models else None
    if default_model_name is None:
        raise ValueError("No chat models are configured. ...")

    if requested_model_name and app_config.get_model_config(requested_model_name):
        return requested_model_name                          # 用户要的能用 → 用

    if requested_model_name and requested_model_name != default_model_name:
        logger.warning(f"Model '{requested_model_name}' not found in config; fallback to default ...")
    return default_model_name                                 # 否则用默认
```

**核心逻辑**：

- 如果用户传了模型名**且**这个模型在 config.yaml 里**确实存在** → 用用户的；
- 否则 → 用默认（`config.models[0]`，即配置里第一个模型），并打 warning。

**为什么不直接报错？** 因为这是个生产系统，宁可降级也不能让用户对话失败。这种"软降级"思想在大型系统里很常见。

### 1.5.3 第三段：注入 trace 元数据 + 挂 tracing 回调（agent.py:454-482）

```python
    # Inject run metadata for LangSmith trace tagging
    if "metadata" not in config:
        config["metadata"] = {}

    config["metadata"].update(
        {
            "agent_name": agent_name or "default",
            "model_name": model_name or "default",
            "thinking_enabled": thinking_enabled,
            "reasoning_effort": reasoning_effort,
            "is_plan_mode": is_plan_mode,
            "subagent_enabled": subagent_enabled,
            "tool_groups": agent_config.tool_groups if agent_config else None,
            "available_skills": sorted(available_skills) if available_skills is not None else None,
        }
    )

    # Inject tracing callbacks at the graph invocation root ...
    tracing_callbacks = build_tracing_callbacks()
    if tracing_callbacks:
        existing = config.get("callbacks") or []
        if not isinstance(existing, list):
            existing = list(existing)
        config["callbacks"] = [*existing, *tracing_callbacks]
```

**这段在做两件事**：

#### (a) 把这一轮的关键信息塞进 `config["metadata"]`

为什么要塞？因为 tracing 平台（LangSmith / Langfuse）会读 `metadata` 来给 trace 打标签。塞进去之后，你在 trace 面板里就能按 `model_name=gpt-4o` 或 `subagent_enabled=true` 来筛选、查询 trace。这是可观测性的基础。

#### (b) ★ 在图根节点挂 tracing 回调 ★

这就是模块文档里那个"铁律"的具体实现。`build_tracing_callbacks()` 返回 Langfuse/LangSmith 的 handler 列表（如果配置了的话），追加到 `config["callbacks"]`。

**为什么这里挂，而不是模型挂？** 还记得模块文档说的——只有挂在根节点，Langfuse handler 才能看到 `on_chain_start(parent_run_id=None)`，从而正确传播 `langfuse_session_id` / `langfuse_user_id`。

### 1.5.4 第四段：装配的三个分支（agent.py:484-540）

这是 `_make_lead_agent` 的"主战场"。装配分**三种情况**：bootstrap / default / custom。

为了不让你迷路，我先画一张图：

```
                  ┌────────────── is_bootstrap? ──────────────┐
                  │                                            │
                  │ Yes                                        │ No
                  ▼                                            ▼
        ┌──────────────────┐                       ┌──────────────────┐
        │ Bootstrap 分支    │                       │  agent_name 非空? │
        │ (agent.py:486-511)│                       └──────┬───────────┘
        │                  │                              │
        │ • 工具 = 全部 +   │                       Yes    │ No
        │   setup_agent    │              ┌────────────────┴───────────┐
        │ • skill = 只有    │              ▼                            ▼
        │   "bootstrap"    │     ┌──────────────────┐         ┌──────────────────┐
        │ • prompt 简化     │     │ Custom 分支       │         │ Default 分支     │
        └──────────────────┘     │ (agent.py:513-540)│         │ (同 Custom，但   │
                                  │                  │         │  无 update_agent)│
                                  │ • 工具 = 全部 +   │         └──────────────────┘
                                  │   update_agent   │
                                  │ • skill = 该     │
                                  │  agent 配的      │
                                  │ • prompt 含 SOUL │
                                  └──────────────────┘
```

我们一个一个看。

#### 分支 A：Bootstrap Agent（agent.py:486-511）

```python
    if is_bootstrap:
        # Special bootstrap agent with minimal prompt for initial custom agent creation flow
        raw_tools = get_available_tools(model_name=model_name, subagent_enabled=subagent_enabled, app_config=resolved_app_config) + [setup_agent]
        filtered = filter_tools_by_skill_allowed_tools(raw_tools, skills_for_tool_policy)
        final_tools, setup = assemble_deferred_tools(filtered, enabled=resolved_app_config.tool_search.enabled)
        return create_agent(
            model=create_chat_model(name=model_name, thinking_enabled=thinking_enabled, app_config=resolved_app_config, attach_tracing=False),
            tools=final_tools,
            middleware=build_middlewares(
                config,
                model_name=model_name,
                available_skills=set(_BOOTSTRAP_SKILL_NAMES),   # ← 只有 "bootstrap" 这个 skill
                app_config=resolved_app_config,
                deferred_setup=setup,
            ),
            system_prompt=apply_prompt_template(
                subagent_enabled=subagent_enabled,
                max_concurrent_subagents=max_concurrent_subagents,
                available_skills=set(_BOOTSTRAP_SKILL_NAMES),
                app_config=resolved_app_config,
                deferred_names=setup.deferred_names,
            ),
            state_schema=ThreadState,
        )
```

**Bootstrap 是干嘛的？**

它是一个**特殊的一次性 agent**，专门用来"引导用户创建他们的第一个自定义 agent"。比如用户第一次打开 DeerFlow，没有 agent 可用，bootstrap agent 就出来问："你想创建一个什么样的 agent？" 然后调 `setup_agent` 工具把用户的回答写成 SOUL.md + config.yaml。

**为什么它的 skill 集合是 `{"bootstrap"}`？** 因为此时还没有"被创建的 agent"，所以不能用任何业务 skill，只能用一个最小化的"引导流程" skill。这是"先有鸡还是先有蛋"问题的解法——用一个**固定的引导 agent** 来创建其他 agent。

**装配要素**：

- **工具**：`get_available_tools(...)` 拿到所有内置工具 + `[setup_agent]`（创建 agent 的工具）；
- **过滤**：`filter_tools_by_skill_allowed_tools` 按 skill 白名单过滤（这里只允许 bootstrap skill）；
- **deferred 处理**：`assemble_deferred_tools` 把 MCP 工具标记为"延迟加载"；
- **造图**：调 `create_agent(...)`。

**注意 `create_agent` 的 5 个参数**：

```python
create_agent(
    model=...,          # 用哪个 LLM
    tools=...,          # 能用哪些工具
    middleware=...,     # 装哪些中间件
    system_prompt=...,  # 系统提示词
    state_schema=...,   # 状态结构（ThreadState）
)
```

这 5 个就是造一个 LangGraph Agent 的全部"配料"。后面 default/custom 两个分支也是这 5 样，只是内容不同。

> 💡 **注意**：所有 `create_chat_model(...)` 调用都传了 `attach_tracing=False`。这就是模块文档那个"铁律"的落地。

#### 分支 B & C：Custom / Default（agent.py:513-540）

```python
    # Custom agents can update their own SOUL.md / config via update_agent.
    # The default agent (no agent_name) does not see this tool.
    extra_tools = [update_agent] if agent_name else []
    # Default lead agent (unchanged behavior)
    raw_tools = get_available_tools(model_name=model_name, groups=agent_config.tool_groups if agent_config else None, subagent_enabled=subagent_enabled, app_config=resolved_app_config)
    filtered = filter_tools_by_skill_allowed_tools(raw_tools + extra_tools, skills_for_tool_policy)
    final_tools, setup = assemble_deferred_tools(filtered, enabled=resolved_app_config.tool_search.enabled)
    return create_agent(
        model=create_chat_model(name=model_name, thinking_enabled=thinking_enabled, reasoning_effort=reasoning_effort, app_config=resolved_app_config, attach_tracing=False),
        tools=final_tools,
        middleware=build_middlewares(
            config,
            model_name=model_name,
            agent_name=agent_name,
            available_skills=available_skills,
            app_config=resolved_app_config,
            deferred_setup=setup,
        ),
        system_prompt=apply_prompt_template(
            subagent_enabled=subagent_enabled,
            max_concurrent_subagents=max_concurrent_subagents,
            agent_name=agent_name,
            available_skills=available_skills,
            app_config=resolved_app_config,
            deferred_names=setup.deferred_names,
        ),
        state_schema=ThreadState,
    )
```

**Custom 和 Default 的唯一区别**：

```python
extra_tools = [update_agent] if agent_name else []
```

- **Custom agent**（`agent_name` 非空）：多一个 `update_agent` 工具，能改自己的 SOUL.md / config.yaml；
- **Default agent**（`agent_name` 是 `None`）：没有 `update_agent`，是日常对话用的通用 agent。

**其他都一样**：

- 工具：`get_available_tools(...)` + 可选的 `update_agent`，按 skill 白名单过滤；
- 中间件：`build_middlewares(...)`；
- Prompt：`apply_prompt_template(...)`（Custom 会注入 SOUL.md 内容）。

> 💡 **关键认知**：所谓"多 Agent"，本质上就是**同一个 `create_agent` 函数，用不同的工具集、中间件参数、prompt 配料，造出不同行为的实例**。不是"多个独立服务"。

### 1.5.5 小结：`_make_lead_agent` 干了什么？

用一句话总结：

> **`_make_lead_agent` 是个"装配车间"，它根据运行时配置，把 model + tools + middleware + system_prompt + state_schema 这 5 样配料组装成一个 LangGraph 图。**

装配的核心是这 5 样。我们接下来重点看其中两个**最复杂**的：

- **`state_schema=ThreadState`**：状态结构（1.6 节）；
- **`middleware=build_middlewares(...)`**：中间件管道（1.7 节）。

（`model` 和 `tools` 相对简单，`system_prompt` 等讲 prompt 工程时再说。）

## 1.6 配料之一：`ThreadState`——贯穿会话的状态

**文件**：`backend/packages/harness/deerflow/agents/thread_state.py`

这个文件**整个**就 96 行，但极其重要。我们一段一段读。

### 1.6.1 先看类的定义（thread_state.py:87-95）

```python
class ThreadState(AgentState):
    sandbox: NotRequired[SandboxState | None]
    thread_data: NotRequired[ThreadDataState | None]
    title: NotRequired[str | None]
    artifacts: Annotated[list[str], merge_artifacts]
    todos: Annotated[list | None, merge_todos]
    uploaded_files: NotRequired[list[dict] | None]
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
    promoted: Annotated[PromotedTools | None, merge_promoted]
```

**先解释语法**：

- `class ThreadState(AgentState)`：继承自 LangChain 的 `AgentState`。`AgentState` 已经定义了一个字段 `messages: list[BaseMessage]`——所以 `ThreadState` 自动有了 `messages`，再加上自己定义的 8 个字段。
- `NotRequired[...]`：TypedDict 的语法，表示这个字段是可选的（可以不出现）。
- `Annotated[类型, reducer]`：**LangGraph 的关键语法**——给字段附一个 **reducer（归约函数）**，定义多个节点并发写这个字段时怎么合并。

### 1.6.2 字段逐个讲

| 字段 | 类型 | Reducer | 用途 |
|---|---|---|---|
| `messages` | `list[BaseMessage]` | `add_messages`（继承） | 对话历史 |
| `sandbox` | `SandboxState \| None` | 无 | 当前沙箱句柄 |
| `thread_data` | `ThreadDataState \| None` | 无 | 工作目录路径 |
| `title` | `str \| None` | 无 | 会话标题 |
| `artifacts` | `list[str]` | `merge_artifacts` | 产物文件路径 |
| `todos` | `list \| None` | `merge_todos` | TodoList |
| `uploaded_files` | `list[dict] \| None` | 无 | 上传文件清单 |
| `viewed_images` | `dict[str, ViewedImageData]` | `merge_viewed_images` | 看过的图片 |
| `promoted` | `PromotedTools \| None` | `merge_promoted` | 延迟工具的提升记录 |

**注意三类**：

1. **无 reducer 的字段**（`sandbox`、`thread_data`、`title`、`uploaded_files`）：直接覆盖，后写胜前写；
2. **有 reducer 的字段**（`artifacts`、`todos`、`viewed_images`、`promoted`）：合并，具体怎么合并看 reducer；
3. **`messages`**：继承自 `AgentState`，用 LangGraph 内置的 `add_messages` reducer（按 id 去重 + 追加）。

### 1.6.3 深入看几个 reducer

reducer 是 LangGraph 的精髓。我们看 3 个典型的：

#### (a) `merge_artifacts`——保序去重合并（thread_state.py:21-28）

```python
def merge_artifacts(existing: list[str] | None, new: list[str] | None) -> list[str]:
    """Reducer for artifacts list - merges and deduplicates artifacts."""
    if existing is None:
        return new or []
    if new is None:
        return existing
    # Use dict.fromkeys to deduplicate while preserving order
    return list(dict.fromkeys(existing + new))
```

**它在干嘛**：把两个 list 合并，去重，**保持顺序**。

**为什么用 `dict.fromkeys`？** 这是个 Python 小技巧——从 Python 3.7 起，dict 保持插入顺序，所以 `dict.fromkeys([...])` 能"去重保序"。比 `list(set(...))`（去重但不保序）更合适。

**用在哪**：`artifacts` 字段记录 agent 产生的文件路径。比如 agent 第一轮生成了 `report.md`，第二轮又生成了 `data.csv`，第三轮又生成了 `report.md`（同一个文件）。reducer 会把它合并成 `["report.md", "data.csv"]`——去重，保序。

#### (b) `merge_todos`——"显式更新胜出"（thread_state.py:48-58）

```python
def merge_todos(existing: list | None, new: list | None) -> list | None:
    """Reducer for todos list - keeps the last non-None value.

    Semantics:
    - If `new` is None (node didn't touch todos), preserve `existing`.
    - If `new` is provided (even empty list), it represents an explicit
      update and wins over `existing`.
    """
    if new is None:
        return existing
    return new
```

**它在干嘛**：如果新值是 `None`，保留旧值；否则用新值（哪怕是空 list）。

**为什么这么设计**？因为 LangGraph 里每个节点都会跑一次 reducer（即使节点没改这个字段）。如果一个中间件只是"路过"没动 todos，LangGraph 会传 `new=None`——reducer 必须能识别"没改"和"显式清空"的区别。

- `new=None` → "我没动 todos" → 保留旧值；
- `new=[]` → "我故意清空 todos" → 用空 list。

这是个"**区分'未设置'和'显式空'**"的经典问题，在 API 设计里很常见。

#### (c) `merge_promoted`——版本隔离合并（thread_state.py:66-84）

```python
def merge_promoted(existing: PromotedTools | None, new: PromotedTools | None) -> PromotedTools | None:
    """Reducer for deferred-tool promotions, scoped by catalog hash.

    - new None/empty -> preserve existing (node didn't touch promotions).
    - catalog_hash changed -> replace wholesale, dropping stale names (prevents a
      persisted bare name from exposing a different tool after catalog drift).
    - same catalog_hash -> union names, dedupe, preserve order.
    """
    if not new:
        return existing
    if existing is None or existing.get("catalog_hash") != new["catalog_hash"]:
        return {
            "catalog_hash": new["catalog_hash"],
            "names": list(dict.fromkeys(new["names"])),
        }
    return {
        "catalog_hash": existing["catalog_hash"],
        "names": list(dict.fromkeys(existing["names"] + new["names"])),
    }
```

**背景**：`promoted` 记录"哪些延迟工具已经被模型主动 search 出来、可以用了"。结构是 `{catalog_hash, names}`。

**三种情况**：

1. `new` 是空/None → 没动，保留旧的；
2. `catalog_hash` 变了 → **整体替换**（catalog 漂移，旧名字可能指向不同的工具了，不能合并）；
3. `catalog_hash` 没变 → **并集合并**。

**为什么要 `catalog_hash`？** 因为工具列表会变（MCP server 重启、配置改动）。如果只按名字合并，可能出现"上次提升的 `read` 工具，这次变成了完全不同的实现"——非常危险。`catalog_hash` 是工具目录的哈希，变了就说明"工具的世界变了"，必须丢掉旧的。

**学习启示**：这是个**版本隔离合并**的模式——合并时带版本号，版本变了就不合并。在缓存、配置同步、状态合并场景都适用。

#### (c-补充) merge_promoted 彻底搞懂——一个完整的故事

如果你看完上面还觉得模糊，这里用一个**从头到尾的故事**把每个环节串起来。

**第 1 步：什么是"延迟工具（deferred tools）"？为什么需要它？**

DeerFlow 可以接入很多 MCP 工具（Slack、GitHub、Playwright……可能几十上百个）。如果把这些工具的**完整 JSON Schema**（参数定义、描述……）全发给模型，会**撑爆上下文窗口**。

DeerFlow 的解法是"延迟加载"：

```
初始状态（模型看到的）：
  system prompt 里只有工具名列表：
    <available-deferred-tools>
    slack_send_message
    slack_list_channels
    github_create_issue
    ...（100 个名字，但没有 schema）
    </available-deferred-tools>

  → 模型知道"有这些工具"，但不知道参数怎么填
  → 模型没法直接调用它们
```

**第 2 步：模型想用某个工具时怎么办？**

模型必须**先调 `tool_search` 工具**，把目标工具的 schema "搜索"出来。看 `tools/builtins/tool_search.py:130-158`：

```python
def build_tool_search_tool(catalog: DeferredToolCatalog) -> BaseTool:
    catalog_hash = catalog.hash     # ★ 把 catalog_hash 闭包进 tool_search

    @tool
    def tool_search(query: str, tool_call_id: ...) -> Command:
        matched = catalog.search(query)[:MAX_RESULTS]
        content = json.dumps([convert_to_openai_function(t) for t in matched], ...)
        names = [t.name for t in matched]
        return Command(
            update={
                "promoted": {"catalog_hash": catalog_hash, "names": names},  # ★ 写 state
                "messages": [ToolMessage(content=content, ...)],
            }
        )
```

**关键**：`tool_search` 返回一个 LangGraph 的 `Command(update={...})`，它把 `{catalog_hash, names}` 写进 `state["promoted"]` 字段。这就是 promotion（提升）——从"隐藏"变成"可见"。

**第 3 步：promote 之后，模型怎么"获得"调用权？**

靠 `DeferredToolFilterMiddleware`（`deferred_tool_filter_middleware.py:42-60`）：

```python
def _promoted(self, state) -> set[str]:
    promoted = (state or {}).get("promoted")
    if promoted and promoted.get("catalog_hash") == self._catalog_hash:  # ★ 再校验 hash
        return set(promoted.get("names") or [])
    return set()

def _hidden(self, state) -> set[str]:
    return set(self._deferred) - self._promoted(state)   # 延迟工具 - 已提升 = 仍隐藏

def _filter_tools(self, request: ModelRequest) -> ModelRequest:
    hide = self._hidden(request.state)
    active = [t for t in request.tools if t.name not in hide]  # 从发给模型的全集里删掉隐藏的
    return request.override(tools=active)
```

**每次调模型前**（`wrap_model_call`），这个中间件会把"还没 promoted 的延迟工具"的 schema **从 request.tools 里删掉**。模型只看到已提升的工具 schema。

**第 4 步：为什么合并时需要 `catalog_hash`？（核心问题）**

`catalog_hash` 是工具目录的 **SHA256 指纹**（`tool_search.py:66-70`）：

```python
@cached_property
def hash(self) -> str:
    canon = [{"name": t.name, "schema": convert_to_openai_function(t)} 
             for t in sorted(self.tools, key=lambda t: t.name)]
    blob = json.dumps(canon, sort_keys=True, ...)
    return hashlib.sha256(blob.encode()).hexdigest()[:16]
```

它把**所有延迟工具的名字+schema**序列化排序后算哈希。**任何工具的新增/删除/改名/改参数都会让 hash 变**。

现在看 merge_promoted 的**三种情况**，用真实场景讲：

**情况 A：`new` 是 None → 保留旧值**

```
某个中间件跑完（比如 DynamicContext），它根本没碰 promoted 字段。
LangGraph 传进来：new = None
→ 返回 existing（原样保留）

跟 merge_todos 一样：区分"没动"和"清空"。
```

**情况 B：`catalog_hash` 变了 → 整体替换（★ 最关键的安全机制）**

想象这个灾难场景（如果不整体替换）：

```
Day 1：工具目录 catalog_hash = "abc"
  模型调 tool_search("read")
  promoted = {"catalog_hash":"abc", "names":["read"]}
  （"read" 是"读取本地文档"工具）

Day 2：MCP server 重启，工具列表变了
  "read" 这个名字被复用给了完全不同的工具（比如"读取 GitHub repo"）
  新的 catalog_hash = "xyz"

Day 3：对话继续（thread 还活着，state 里有旧的 promoted）
  如果 reducer 只是简单合并 names：
    existing = {"catalog_hash":"abc", "names":["read"]}
    new      = {"catalog_hash":"xyz", "names":["write"]}
    合并     → {"names":["read","write"]}  ← 灾难！

  此时 "read" 已经指向完全不同的工具了！
  模型以为提升的是"读文档"，实际暴露的是"读 GitHub repo"
  → 可能误操作别人的仓库
```

**所以 `catalog_hash` 变了 = 工具世界变了，旧 promotion 全部作废**：

```python
# 情况 B：hash 变了，整体替换
existing = {"catalog_hash":"abc", "names":["read","old_tool"]}
new      = {"catalog_hash":"xyz", "names":["write"]}
→ 返回 {"catalog_hash":"xyz", "names":["write"]}
# 旧的 "read" 被丢弃了——因为它可能指向完全不同的工具！
```

**情况 C：`catalog_hash` 没变 → 并集合并**

```
同一个工具目录下，模型多次调 tool_search：
  第 1 次：promoted = {"catalog_hash":"abc", "names":["slack_send"]}
  第 2 次：promoted = {"catalog_hash":"abc", "names":["github_issue"]}

reducer 合并（hash 一样，走并集）：
  → {"catalog_hash":"abc", "names":["slack_send","github_issue"]}
```

用 `dict.fromkeys` 去重保序——防模型重复 search 同一个工具。

**双保险**：reducer 里校验 hash 一次，`_promoted()` 运行时读 state 时**再校验一次**。即使 state 里存了旧 hash 的 promotion，运行时也不会认。

**一句话总结 merge_promoted**：

> **它是个"带版本隔离的并集合并" reducer。同版本（hash 一样）→ 合并工具名；跨版本（hash 变了）→ 整体替换（旧 promotion 作废，防"名字复用"安全事故）；没动 → 保留旧值。**

---

> 📌 **Reducer 的核心价值**：让 LangGraph 能**安全地并发**。多个节点同时往同一个字段写，reducer 决定结果。没有 reducer，并发就会"丢更新"。

## 1.7 配料之二：`build_middlewares`——中间件管道

**文件**：`lead_agent/agent.py:270-377`

这是 DeerFlow 最复杂、也最值得学的部分。我们一段段读。

### 1.7.1 先看顶部的注释（agent.py:260-269）

```python
# ThreadDataMiddleware must be before SandboxMiddleware to ensure thread_id is available
# UploadsMiddleware should be after ThreadDataMiddleware to access thread_id
# DanglingToolCallMiddleware patches missing ToolMessages before model sees the history
# SummarizationMiddleware should be early to reduce context before other processing
# TodoListMiddleware should be before ClarificationMiddleware to allow todo management
# TitleMiddleware generates title after first exchange
# MemoryMiddleware queues conversation for memory update (after TitleMiddleware)
# ViewImageMiddleware should be before ClarificationMiddleware to inject image details before LLM
# ToolErrorHandlingMiddleware should be before ClarificationMiddleware to convert tool exceptions to ToolMessages
# ClarificationMiddleware should be last to intercept clarification requests after model calls
```

**这段注释是金矿**。它把"为什么这个顺序"全写出来了。逐条翻译：

| 注释 | 翻译 |
|---|---|
| ThreadData 必须在 Sandbox 前 | 因为 Sandbox 要用 thread_id，而 thread_id 由 ThreadData 准备 |
| Uploads 在 ThreadData 后 | Uploads 也要 thread_id |
| DanglingToolCall 早装 | 它要补全"缺失的 ToolMessage"，必须在模型看到历史之前补 |
| Summarization 要早 | 它压缩上下文，应该在别的中间件处理之前先压 |
| TodoList 在 Clarification 前 | TodoList 要能管理 todo，不能被 Clarification 抢先中断 |
| Title 在首轮对话后 | 标题生成需要先有对话内容 |
| Memory 在 Title 后 | 记忆更新排队，等标题稳定了再排 |
| ViewImage 在 Clarification 前 | 图片信息要先进 LLM 上下文 |
| ToolErrorHandling 在 Clarification 前 | 工具异常要先转成 ToolMessage |
| **Clarification 必须最后装** | 它要拦截 ask_clarification，必须最后看到模型输出 |

**学习启示**：**好的代码注释解释"为什么"，不解释"是什么"**。这段注释全是"为什么 X 必须在 Y 前/后"，是教科书级别的中间件顺序文档。

### 1.7.2 看 `build_middlewares` 函数签名（agent.py:270-299）

```python
def build_middlewares(
    config: RunnableConfig,
    model_name: str | None,
    agent_name: str | None = None,
    custom_middlewares: list[AgentMiddleware] | None = None,
    *,
    available_skills: set[str] | None = None,
    app_config: AppConfig | None = None,
    deferred_setup=None,
):
    """Build the lead-agent middleware chain based on runtime configuration.

    Public entry point for the lead agent's full middleware composition. Used by
    ``make_lead_agent`` and by the embedded ``DeerFlowClient`` (a lead-agent variant
    that needs the identical chain). Keep this name stable: ...
    """
    resolved_app_config = app_config or get_app_config()
    middlewares = build_lead_runtime_middlewares(app_config=resolved_app_config, lazy_init=True)
```

**函数签名**：

- `config`：运行时配置；
- `model_name`：解析后的模型名（决定要不要装 Vision 中间件）；
- `agent_name`：自定义 agent 名（决定 MemoryMiddleware 用哪个存储）；
- `custom_middlewares`：用户自定义中间件；
- `available_skills`：可用 skill 集合；
- `deferred_setup`：延迟工具的配置。

**第一行做的事**：

```python
middlewares = build_lead_runtime_middlewares(app_config=resolved_app_config, lazy_init=True)
```

调 `build_lead_runtime_middlewares` 拿到一批"基础中间件"（ThreadData、Sandbox、Uploads、DanglingToolCall、ToolErrorHandling 等运行时基础设施）。这是管道的"地基"，`lazy_init=True` 表示延迟初始化（第一次用时才真正初始化）。

> 💡 这个函数返回的具体是什么我没读到，但从命名和注释推断：它返回那批"必须按特定顺序装、但用户一般不动"的底层中间件。DeerFlow 把它们打包成一个工厂函数，避免每次手写。

### 1.7.3 逐个添加中间件（agent.py:300-376）

接下来是一长串 `middlewares.append(...)`。我们分组讲：

#### 第 1 组：动态上下文 + 技能激活（agent.py:300-313）

```python
    # Always inject current date (and optionally memory) as <system-reminder> into the
    # first HumanMessage to keep the system prompt fully static for prefix-cache reuse.
    from deerflow.agents.middlewares.dynamic_context_middleware import DynamicContextMiddleware

    middlewares.append(DynamicContextMiddleware(agent_name=agent_name, app_config=resolved_app_config))

    # Deterministically load a full SKILL.md when the user starts the turn with
    # /skill-name. This keeps the base system prompt metadata-only while giving
    # explicit user activation priority over model-side relevance guessing.
    from deerflow.agents.middlewares.skill_activation_middleware import SkillActivationMiddleware

    middlewares.append(SkillActivationMiddleware(available_skills=available_skills, app_config=resolved_app_config))
```

**两个中间件**：

1. **`DynamicContextMiddleware`**：把日期 + 用户记忆包成 `<system-reminder>` 注入到**第一条 HumanMessage**（不是 system prompt！）。
   
   **为什么这么干？** 看注释："to keep the system prompt fully static for prefix-cache reuse"——保持 system prompt **完全静态**，让 LLM 提供商的 prefix cache 命中（缓存命中费用减半、延迟降低）。这部分第 4 章会详讲。

2. **`SkillActivationMiddleware`**：处理用户用 `/skill-name` 斜杠激活技能的情况。比如用户输入 `/docx 帮我生成一个简历`，这个中间件会把 `docx` 这个 skill 的完整 SKILL.md 内容读出来，注入到上下文。

**为什么放第二个？** 因为它要在模型看到 user message 之前就插入 skill 内容，但要在 DynamicContext 之后（DynamicContext 改的是第一条 user msg，SkillActivation 改的是当前 user msg）。

#### 第 2 组：摘要（agent.py:315-318）

```python
    # Add summarization middleware if enabled
    summarization_middleware = _create_summarization_middleware(app_config=resolved_app_config)
    if summarization_middleware is not None:
        middlewares.append(summarization_middleware)
```

**`SummarizationMiddleware`**：当上下文超过阈值时，把旧消息压缩成摘要。

**为什么早装？** 看顶部注释："should be early to reduce context before other processing"——要在别的中间件处理之前先压。否则其他中间件看到的还是又臭又长的历史，浪费 token。

**条件装配**：`if summarization_middleware is not None`——只有配置 `summarization.enabled=True` 才装。默认是关的。

#### 第 3 组：Todo（agent.py:320-325）

```python
    # Add TodoList middleware if plan mode is enabled
    cfg = _get_runtime_config(config)
    is_plan_mode = cfg.get("is_plan_mode", False)
    todo_list_middleware = _create_todo_list_middleware(is_plan_mode)
    if todo_list_middleware is not None:
        middlewares.append(todo_list_middleware)
```

**`TodoMiddleware`**：只有在 plan 模式才装。它会：

- 给 agent 一个 `write_todos` 工具；
- 注入 TodoList 管理 prompt；
- 防止 agent 提前收尾（todo 没完成就强制再来一轮）。

#### 第 4 组：Token 统计 + 标题 + 记忆（agent.py:327-335）

```python
    # Add TokenUsageMiddleware when token_usage tracking is enabled
    if resolved_app_config.token_usage.enabled:
        middlewares.append(TokenUsageMiddleware())

    # Add TitleMiddleware
    middlewares.append(TitleMiddleware(app_config=resolved_app_config))

    # Add MemoryMiddleware (after TitleMiddleware)
    middlewares.append(MemoryMiddleware(agent_name=agent_name, memory_config=resolved_app_config.memory))
```

**三个**：

1. **`TokenUsageMiddleware`**：统计 token 用量，给每步打 attribution（这是哪一步、用了多少 token）；
2. **`TitleMiddleware`**：异步生成会话标题（首轮对话后）；
3. **`MemoryMiddleware`**：在 `after_agent` 触发记忆更新入队。

**为什么 Memory 在 Title 后？** 看顶部注释："MemoryMiddleware queues conversation for memory update (after TitleMiddleware)"——记忆更新需要标题先稳定。

#### 第 5 组：视觉 + 延迟工具 + Subagent 限制（agent.py:337-355）

```python
    # Add ViewImageMiddleware only if the current model supports vision.
    model_config = resolved_app_config.get_model_config(model_name) if model_name else None
    if model_config is not None and model_config.supports_vision:
        middlewares.append(ViewImageMiddleware())

    # Hide deferred tool schemas from model binding until tool_search promotes them.
    if deferred_setup is not None and deferred_setup.deferred_names:
        from deerflow.agents.middlewares.deferred_tool_filter_middleware import DeferredToolFilterMiddleware

        middlewares.append(DeferredToolFilterMiddleware(deferred_setup.deferred_names, deferred_setup.catalog_hash))

    # Add SubagentLimitMiddleware to truncate excess parallel task calls
    subagent_enabled = cfg.get("subagent_enabled", False)
    if subagent_enabled:
        max_concurrent_subagents = cfg.get("max_concurrent_subagents", 3)
        middlewares.append(SubagentLimitMiddleware(max_concurrent=max_concurrent_subagents))
```

**三个**：

1. **`ViewImageMiddleware`**：**只有模型支持 vision 才装**（条件装配）。处理图片查看；
2. **`DeferredToolFilterMiddleware`**：隐藏 MCP 工具的 schema，强制模型先 `tool_search` 才能用；
3. **`SubagentLimitMiddleware`**：限制单轮并行 `task` 工具调用 ≤ 3，超出截断。

#### 第 6 组：循环检测（agent.py:357-360）

```python
    # LoopDetectionMiddleware — detect and break repetitive tool call loops
    loop_detection_config = resolved_app_config.loop_detection
    if loop_detection_config.enabled:
        middlewares.append(LoopDetectionMiddleware.from_config(loop_detection_config))
```

**`LoopDetectionMiddleware`**：检测 agent 是不是在反复调同一个工具（死循环），是的话警告或硬停。这是个 P0 安全中间件。

#### 第 7 组：自定义中间件（agent.py:362-364）

```python
    # Inject custom middlewares before ClarificationMiddleware
    if custom_middlewares:
        middlewares.extend(custom_middlewares)
```

**用户自定义中间件**：插在 ClarificationMiddleware 前面，给用户一个扩展点。

#### 第 8 组：安全终止（agent.py:366-373）

```python
    # SafetyFinishReasonMiddleware — suppress tool execution when the provider
    # safety-terminated the response. Registered after custom middlewares so
    # that LangChain's reverse-order after_model dispatch runs Safety first;
    # cleared tool_calls then flow through Loop/Subagent accounting without
    # firing extra alarms. See safety_finish_reason_middleware.py docstring.
    safety_config = resolved_app_config.safety_finish_reason
    if safety_config.enabled:
        middlewares.append(SafetyFinishReasonMiddleware.from_config(safety_config))
```

**`SafetyFinishReasonMiddleware`**：处理模型被内容审核中途截断的情况（比如 OpenAI 返回 `finish_reason=content_filter` 但还带了部分 tool_calls）。

**为什么排在自定义中间件之后？** 看注释："LangChain's reverse-order after_model dispatch runs Safety first"——LangChain 的 `after_model` 是**逆序执行**的（最后装的最先跑）。所以 Safety 装在后面，它的 `after_model` 反而**最先**看到模型输出，能在死循环或 subagent 计数之前先清掉危险的 tool_calls。

> 💡 **这是个反直觉但重要的设计**：**"列表顺序" ≠ "执行顺序"**。LangChain 的 `after_model` 是逆序的，所以你想让某中间件**最先**看到输出，反而要把它**最后**装。这种"倒装"模式在事件系统里很常见。

#### 第 9 组：Clarification（永远最后）（agent.py:375-377）

```python
    # ClarificationMiddleware should always be last
    middlewares.append(ClarificationMiddleware())
    return middlewares
```

**`ClarificationMiddleware`**：拦截 `ask_clarification` 工具调用，实现 HITL（人在回路）。

**为什么必须最后装？** 同样的道理——逆序执行，最后装的最先看到 `after_model` 输出，能第一时间拦截 clarification 请求并中断流程。如果它在别的中间件之前装，那些中间件可能已经在改消息、记 token 了，逻辑就乱了。

### 1.7.4 中间件管道总结

把上面 9 组串起来，就是完整的中间件管道（**装配顺序**，注意执行顺序部分是逆的）：

```
[基础运行时中间件]                          ← build_lead_runtime_middlewares 提供
  → DynamicContext                         ← 注入日期+记忆
    → SkillActivation                      ← /skill 激活
      → Summarization (可选)                ← 上下文压缩
        → Todo (plan 模式)
          → TokenUsage
            → Title
              → Memory                     ← 触发记忆更新
                → ViewImage (vision 模型)
                  → DeferredToolFilter     ← MCP 工具延迟加载
                    → SubagentLimit
                      → LoopDetection
                        → [自定义中间件]
                          → SafetyFinishReason
                            → Clarification  ← 永远最后
```

**总数**：约 15 个中间件（部分条件装配）。

**这张图你记住，第 2-6 章的所有"魔法"都发生在这条管道里**：

- 第 2 章的 Subagent 调度 → SubagentLimit + task 工具；
- 第 3 章的记忆 → Memory + DynamicContext；
- 第 4 章的上下文压缩 → Summarization + DynamicContext + DeferredToolFilter；
- 第 6 章的循环检测、安全、HITL → LoopDetection + SafetyFinishReason + Clarification。

### 1.7.5 补充：装配顺序 vs 执行顺序——不同 hook 的规则不同！

这是一个**极其容易踩坑**的点，必须单独讲：**"装配顺序"（append 顺序）和"执行顺序"的对应关系，不同 hook 是不一样的！**

#### 先回顾 6 个 hook

每个中间件可以实现这 6 个 hook 中的任意几个：

```
before_agent → before_model → [wrap_model_call] → after_model
                                    ↓
                            [wrap_tool_call]
                                    ↓
                              after_agent
```

#### 核心：不同 hook 的执行顺序规则不同

假设 `build_middlewares` 按这个顺序 append：

```python
middlewares = []
middlewares.append(A())    # index 0（最先装）
middlewares.append(B())    # index 1
middlewares.append(C())    # index 2（最后装）
```

**各 hook 的执行顺序**：

| Hook | 执行顺序 | 类型 |
|---|---|---|
| `before_agent` | A → B → C | **正序** |
| `before_model` | A → B → C | **正序** |
| `wrap_model_call` | A(B(C(...))) | **洋葱式**（A 最外层） |
| `wrap_tool_call` | A(B(C(...))) | **洋葱式**（A 最外层） |
| **`after_model`** | **C → B → A** | **★ 逆序！** |
| `after_agent` | C → B → A | **★ 逆序！** |

#### 为什么 `after_*` 要逆序？

因为 `after_*` 是 `before_*` 的"配对收尾"。想象函数调用栈——**先入后出（LIFO）**：

```
正序进入（before）：  A 进 → B 进 → C 进 → 【干活】 
逆序退出（after）：                        【干完】 → C 出 → B 出 → A 出
```

这跟 Python 的 `with` context manager 或装饰器嵌套一模一样：

```python
# before/after 的等效嵌套
def combined():
    A.before_model()         # 第 1 个进
    B.before_model()         # 第 2 个进
    C.before_model()         # 第 3 个进
    # ====== model.invoke ======
    result = model.invoke()
    # ===========================
    C.after_model(result)    # 第 1 个出（逆序！）
    B.after_model(result)    # 第 2 个出
    A.after_model(result)    # 第 3 个出
```

**这才是"洋葱"的完整形态**——不只是 `wrap_model_call` 是洋葱，**整个 before→干→after 的流程都是洋葱**：

```
A.before_model()           ← 正序进
  └─ B.before_model()
       └─ C.before_model()
            ┌──────────────┐
            │ model.invoke  │  ← 洋葱芯
            └──────────────┘
       C.after_model()      ← 逆序出
  B.after_model()
A.after_model()
```

#### `wrap_model_call` 的洋葱式怎么理解？

`wrap_model_call` 是真正的"包裹"——它**包住整个模型调用**，能改 request 也能改 response。假设装配顺序还是 A、B、C：

```python
def A.wrap_model_call(request, handler):
    print("A: 进")
    response = handler(request)   # handler = B.wrap_model_call
    print("A: 出")
    return response

def B.wrap_model_call(request, handler):
    print("B: 进")
    response = handler(request)   # handler = C.wrap_model_call
    print("B: 出")
    return response

def C.wrap_model_call(request, handler):
    print("C: 进")
    response = handler(request)   # handler = model.invoke（真正的调用）
    print("C: 出")
    return response
```

**执行时**：

```
A: 进
  B: 进
    C: 进
      ===== model.invoke() =====
    C: 出
  B: 出
A: 出
```

**A 是最外层**（第一个装的），C 是最内层（最后装的）。A 第一个进、最后一个出，能看到完整的请求和响应——这就是"洋葱"的精髓。

#### 用 DeerFlow 真实例子验证

看 `agent.py:366-377`，装配顺序的**最后两个**：

```python
middlewares.append(SafetyFinishReasonMiddleware(...))  # 倒数第 2
middlewares.append(ClarificationMiddleware())          # 最后（index 最大）
```

**`after_model` 是逆序的**，所以执行顺序：

```
ClarificationMiddleware.after_model    ← 最后装的，after_model 最先执行 ★
  ↓ (如果没拦截)
SafetyFinishReasonMiddleware.after_model  ← 倒数第 2 装的，第 2 执行
  ↓
... 其他中间件（继续逆序）...
```

**这正是 DeerFlow 想要的**（注释在 `agent.py:366-373`）：

```python
# SafetyFinishReasonMiddleware — Registered after custom middlewares so
# that LangChain's reverse-order after_model dispatch runs Safety first;
# cleared tool_calls then flow through Loop/Subagent accounting without
# firing extra alarms.
```

翻译："Safety 装在后面 → 逆序执行时它**先跑** → 先清掉危险的 tool_calls → 然后这些已清的消息再流经 Loop/Subagent 计数器 → 不会触发误报。"

而 `ClarificationMiddleware` 装在**最最后** → 它的 `after_model` **最最最先跑** → 能第一时间拦截 `ask_clarification` → 在任何其他中间件处理之前就中断流程。

#### 一张图总结全部

```
装配顺序（append 顺序）：A → B → C → D → E

执行流程（假设都实现了所有 hook）：

  before_agent:  A → B → C → D → E                      (正序)
       │
       ▼
  ┌───────────────────────────────────────────────────┐
  │ before_model: A → B → C → D → E                    │ (正序)
  │      │                                             │
  │      ▼                                             │
  │  wrap_model_call: A(B(C(D(E(model)))))             │ (洋葱：A 最外)
  │      │                                             │
  │      ▼                                             │
  │  after_model:  E → D → C → B → A                   │ (★ 逆序)
  └───────────────────────────────────────────────────┘
       │ (ReAct 循环，每轮跑一遍)
       ▼
  ┌───────────────────────────────────────────────────┐
  │ wrap_tool_call: A(B(C(D(E(tool)))))                │ (洋葱：A 最外)
  └───────────────────────────────────────────────────┘
       │
       ▼
  after_agent:   E → D → C → B → A                      (★ 逆序)
```

#### 一句话记忆口诀

> **`before_*` 正序进，`after_*` 逆序出，`wrap_*` 洋葱包（先装的在外层）。**

**为什么这么设计？** 这样才能保证**最外层（最先装的）中间件"包"住所有内层**——它第一个进、最后一个出，拥有最完整的视野。这就是"洋葱模型"的精髓。

---

## 1.8 最后一个关键概念：`create_agent` 到底造了什么？

我们一直在说"`create_agent` 造一个图"，但具体是什么？这一节揭开谜底。

`create_agent` 是 `langchain.agents` 提供的工厂函数。它内部做的事（简化版）：

```python
def create_agent(model, tools, middleware, system_prompt, state_schema):
    # 1. 把中间件"编译"成各种 hook handler
    before_model_handlers = [m.before_model for m in middleware if hasattr(m, 'before_model')]
    after_model_handlers  = [m.after_model  for m in middleware if hasattr(m, 'after_model')]  # 注意：逆序
    wrap_model_handlers   = [m.wrap_model_call for m in middleware if hasattr(m, 'wrap_model_call')]
    wrap_tool_handlers    = [m.wrap_tool_call for m in middleware if hasattr(m, 'wrap_tool_call')]
    ...

    # 2. 定义两个节点
    def model_node(state):
        # 应用所有 before_model hook
        for h in before_model_handlers:
            state = h(state)
        # 应用 wrap_model_call（洋葱式包裹）
        request = ModelRequest(messages=state["messages"], tools=tools)
        for h in wrap_model_handlers:
            request = h(request)
        # 真正调模型
        response = model.invoke(request)
        # 应用 after_model hook
        for h in after_model_handlers:    # 逆序
            response = h(response)
        return {"messages": [response.message]}

    def tools_node(state):
        # 应用 wrap_tool_call hook
        for tool_call in last_ai_message.tool_calls:
            for h in wrap_tool_handlers:
                result = h(tool_call)
        return {"messages": [tool_messages]}

    # 3. 用 LangGraph 的 StateGraph 拼起来
    graph = StateGraph(state_schema)
    graph.add_node("model", model_node)
    graph.add_node("tools", tools_node)
    graph.add_edge(START, "model")
    graph.add_conditional_edges("model", should_continue, {"continue": "tools", "end": END})
    graph.add_edge("tools", "model")
    return graph.compile()
```

**关键认知**：

1. **中间件不是"装饰器"，而是"hook 注册器"**。每个中间件可以选实现任意几个 hook（`before_model`、`after_model`、`wrap_model_call`、`wrap_tool_call`、`before_agent`、`after_agent`），`create_agent` 会把所有中间件的同类 hook 收集起来，按顺序串成管道；
2. **`after_model` 是逆序执行的**（最后装的最先跑）——这就是为什么 Clarification 必须最后装；
3. **`wrap_model_call` / `wrap_tool_call` 是洋葱式的**（一层包一层，从外到里进，从里到外出）；
4. **最终的图只有两个节点**（`model` 和 `tools`），但通过中间件挂的 hook，行为极其丰富。

> 💡 **心智模型**：你可以把 LangGraph Agent 想成 "**两节点图 + 一堆 hook 拦截器**"。两节点图保证结构简单，hook 拦截器保证扩展性强。DeerFlow 的 15 个中间件就是 15 组 hook 拦截器，每个负责一件事。

---

## 1.9 图造出来后，还要"挂"两样东西：checkpointer 和 store

读到这里你会发现一个**看似矛盾**的点：

- 1.5 节说 `_make_lead_agent` 造图时用了 5 样配料（model / tools / middleware / prompt / state_schema）；
- 但这 5 样里**没有**"数据库"、"持久化"相关的东西。

那 DeerFlow 怎么做到"用户关浏览器、明天回来还能接着聊"？答案就是这一节要讲的：**图造出来后，运行时会"挂"上两样东西**——`checkpointer` 和 `store`。

### 1.9.1 先建立直觉：游戏存档 vs 档案柜

想象你在玩一个 RPG 游戏，打了 3 个小时，要睡觉了。明天接着玩，你需要两样东西：

| 你需要的 | 对应 LangGraph 的什么 | 比喻 |
|---|---|---|
| **存档**：记住"我角色几级、在哪个地图、背包里有啥" | **Checkpointer** | 游戏存档 |
| **公共资料柜**：所有存档共享的"怪物图鉴、成就列表" | **Store** | 游戏的 Wiki |

**关键区别**：

- **存档（checkpointer）** 是**跟某个角色绑定的**——每个角色有自己的存档，记录"这个角色玩到哪了"。你可以有多个存档（多个 thread）；
- **资料柜（store）** 是**所有存档共享的**——不管哪个角色，都能查同一份"怪物图鉴"。

回到 DeerFlow：

- **Checkpointer** 存的是**某个会话（thread）的执行进度**——消息历史、todos、sandbox 状态……整个 `ThreadState`；
- **Store** 存的是**跨所有会话的长期数据**——比如"会话列表"本身（前端侧边栏那个列表）。

### 1.9.2 看源码：挂载发生在哪？

**文件**：`runtime/runs/worker.py:270-274`

```python
# 4. Attach checkpointer and store
if checkpointer is not None:
    agent.checkpointer = checkpointer     # ← 挂 checkpointer
if store is not None:
    agent.store = store                   # ← 挂 store
```

**就这两行**。`agent` 是 `make_lead_agent(config)` 刚造出来的 LangGraph 图。给它两个属性赋值，图就"具备了持久化能力"。

**注意位置**：这两行在 `agent = agent_factory(config)` **之后**（worker.py:255-257 造图）、`agent.astream(...)` **之前**（worker.py 实际开跑）。也就是说：**先造图 → 再挂持久化 → 然后开跑**。

### 1.9.3 这俩具体是什么对象？

它们是 LangGraph 提供的**接口**（看 `provider.py` 顶部的 import）：

```python
# runtime/checkpointer/provider.py:27
from langgraph.types import Checkpointer      # Checkpointer 接口

# runtime/store/provider.py:28
from langgraph.store.base import BaseStore     # Store 接口
```

**接口是抽象的，具体实现有三种后端**（看 `_sync_checkpointer_cm`，`provider.py:59-93`）：

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

**一个重要细节**（看 `store/provider.py:52-58` 的注释）：

> The `config` argument is a `CheckpointerConfig` instance — **the same object used by the checkpointer factory**.

**Checkpointer 和 Store 共用同一个配置对象**。所以它俩**必须用同一种后端**（要么都 sqlite，要么都 postgres）。这是为了简化部署，避免"checkpointer 在 sqlite，store 在 postgres"这种割裂配置。

### 1.9.4 挂上之后，LangGraph 自动用它们干嘛？

这是最神奇的部分——**挂上之后，你基本不用管**，LangGraph 内部自动用：

```
┌─────────────────────────────────────────────────────────────┐
│ 用户第一次跟 thread_id="abc" 聊天                             │
│                                                              │
│  agent.astream(input, config={"configurable":{"thread_id":"abc"}})│
│     │                                                        │
│     ▼ LangGraph 内部                                          │
│     1. 开跑前：cp.aget_tuple(("abc",)) → 空（新会话）          │
│     2. 跑完超级步 1（model 调用完）：cp.aput(...)  ← 存档①      │
│     3. 跑完超级步 2（tools 调用完）：cp.aput(...)  ← 存档②      │
│     4. 跑完超级步 3（model 调用完）：cp.aput(...)  ← 存档③      │
│     5. 图结束                                                 │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 用户明天回来，同一个 thread_id="abc" 继续聊                    │
│                                                              │
│  agent.astream(new_input, config={"configurable":{"thread_id":"abc"}})│
│     │                                                        │
│     ▼ LangGraph 内部                                          │
│     1. 开跑前：cp.aget_tuple(("abc",)) → 拿到存档③            │
│     2. 把存档③ 的状态恢复进 ThreadState                       │
│        （messages、todos、sandbox 全回来了！）                 │
│     3. 把 new_input 追加进 messages                           │
│     4. 从恢复的状态继续跑                                     │
└─────────────────────────────────────────────────────────────┘
```

**关键点**：

1. **存档是"版本栈"**——每个超级步（super-step）存一次，形成 `[存档①, 存档②, 存档③]` 的历史。理论上能回溯到任何一个中间状态（这就是 `worker.py` 里"回滚到运行前 checkpoint"的实现基础）；
2. **`thread_id` 是关键**——LangGraph 靠 `config["configurable"]["thread_id"]` 知道"用哪个存档"。同一个 thread_id 续聊，就会读上次的档；不同 thread_id 就是全新会话；
3. **你不用手动调** checkpointer 的 API（除非要回滚）——LangGraph 自动管。

### 1.9.5 那为什么"挂上去"而不是"造图时传进去"？

你看 `_make_lead_agent` 造图时（`agent.py:493-511`）：

```python
return create_agent(
    model=...,
    tools=...,
    middleware=...,
    system_prompt=...,
    state_schema=ThreadState,
    # ← 注意：没传 checkpointer / store
)
```

**造图时没传**，是后来在 worker.py 里"挂"上去的。为什么？

因为 **checkpointer 是"运行时基础设施"，不是"agent 定义"的一部分**。

这是个重要的设计哲学，类比 Web 框架就明白了：

| Web 框架 | DeerFlow |
|---|---|
| 路由定义（`@app.get("/users")`） | agent 定义（model + tools + middleware + prompt） |
| 数据库连接（运行时注入） | checkpointer / store（运行时挂载） |
| 同一份路由代码，测试用 SQLite、生产用 Postgres | 同一个 agent，测试用 InMemorySaver、生产用 PostgresSaver |

所以 LangGraph 把 checkpointer 设计成"**可后挂的属性**"——agent 定义（是什么）和持久化环境（在哪跑）解耦。

### 1.9.6 同步单例 vs 异步上下文管理器：两套 API

DeerFlow 给每个持久化 provider 都写了**两套 API**（这是个值得学的工程模式）：

**同步单例**（CLI / 嵌入式 SDK 用）——`checkpointer/provider.py:107-145`：

```python
_checkpointer: Checkpointer | None = None       # 全局单例
_checkpointer_lock = threading.Lock()            # 双重检查锁

def get_checkpointer() -> Checkpointer:
    if _checkpointer is not None:                # ★ 快速路径（无锁）
        return _checkpointer

    ensure_config_loaded()

    with _checkpointer_lock:                     # 加锁
        if _checkpointer is not None:            # ★ 二次检查（防并发重复创建）
            return _checkpointer

        config = get_checkpointer_config()
        checkpointer_ctx = _sync_checkpointer_cm(config)
        _checkpointer = checkpointer_ctx.__enter__()   # 进 context manager
        _checkpointer_ctx = checkpointer_ctx           # 存起来，reset 时能 __exit__
    return _checkpointer
```

这是经典的**双重检查锁定（Double-Checked Locking）**模式：

- 第一次检查**不加锁**（99% 的调用走快速路径，性能好）；
- 第二次检查**加锁**（防止两个线程同时通过第一次检查后重复创建）。

**异步上下文管理器**（FastAPI 服务端用）——`checkpointer/async_provider.py`：

```python
@contextlib.asynccontextmanager
async def make_checkpointer(app_config=None) -> AsyncIterator[Checkpointer]:
    async with ... as checkpointer:
        app.state.checkpointer = checkpointer   # 存到 app.state
        yield checkpointer                       # FastAPI lifespan 用
```

**为什么两套？**

- **CLI 场景**没有事件循环，需要同步 API + 全局单例（启动一次、复用）；
- **服务端场景**有事件循环，且 FastAPI 的 lifespan 是 `async with`，需要 async CM；
- 服务端**故意不用全局单例**，因为 lifespan 要明确管理生命周期（启动建池、关闭拆池），全局单例做不到干净的 teardown。

**学习启示**：当你写一个既要给 CLI 用、又要给 Web 服务用的库时，准备两套 API 是常见做法。同步靠单例简化使用，异步靠 CM 保证生命周期清晰。

### 1.9.7 一个容易搞混的点：Store ≠ Memory

很多人第一次看会以为"Store 就是用户的长期记忆"。**不是**。

| 概念 | 存什么 | 哪里实现 |
|---|---|---|
| **Checkpointer** | 图的执行状态（按 thread） | `runtime/checkpointer/` |
| **Store** | 系统级元数据（跨 thread，简单 KV） | `runtime/store/` |
| **Memory**（第 3 章主题） | 用户长期偏好（跨 thread，需 LLM 维护） | `agents/memory/` |

**为什么 Memory 不放 Store？**

因为 Store 只是个**简单 KV**（`put((ns,), key, value)` / `get((ns,), key)`），而 Memory 需要：

- **复杂结构**（结构化摘要 + 置信度排序的 facts 列表）；
- **LLM 抽取 + 合并**逻辑（每次对话后用 LLM 更新）；
- **按置信度驱逐**策略（满了保留 top N）。

这些语义 KV 装不下，所以 DeerFlow 用了**独立的 JSON 文件存储**（`agents/memory/storage.py:FileMemoryStorage`）。

**三层持久化，各司其职**——这是 DeerFlow 架构的一个亮点：

```
┌──────────────────────────────────────────────────────┐
│  Checkpointer（图的存档）                              │
│  • 短期，按 thread                                     │
│  • 存：messages、todos、sandbox、所有 ThreadState 字段  │
│  • 用途：让对话能续聊、能回滚                           │
├──────────────────────────────────────────────────────┤
│  Store（系统档案柜）                                   │
│  • 中期，跨 thread                                     │
│  • 存：线程列表、用户元数据（简单 KV）                  │
│  • 用途：前端侧边栏的"我的会话列表"                    │
├──────────────────────────────────────────────────────┤
│  Memory（用户长期记忆，第 3 章详讲）                    │
│  • 长期，跨 thread                                     │
│  • 存：用户偏好、工作背景、离散事实（结构化 JSON）       │
│  • 用途：让 agent"认识"这个用户                        │
└──────────────────────────────────────────────────────┘
```

### 1.9.8 小结：1.9 节你该记住的

1. **图造出来后，运行时还要挂两样东西**：`agent.checkpointer = cp; agent.store = store`（worker.py:271-274）；
2. **Checkpointer = 游戏存档**：按 thread 存图的执行进度，让对话能续聊；
3. **Store = 档案柜**：跨 thread 存长期 KV（如线程列表）；
4. **三种后端**（memory / sqlite / postgres），checkpointer 和 store **共用配置**；
5. **挂载不传参**——因为持久化是"环境"，不是"agent 定义"，要解耦；
6. **两套 API**：同步单例（CLI 用）+ 异步 CM（服务端用）；
7. **Memory 不是 Store**——用户长期记忆在独立的 JSON 文件里，因为它需要 LLM 维护复杂结构。

---

## 1.10 还要挂一个"记录员"：RunJournal

1.9 节讲了图造完后要挂 `checkpointer` 和 `store`。其实 worker.py 在挂这俩的同时，还**挂了第三个东西**——`RunJournal`。它和前两个完全不同：前两个是"存状态"，它是"**记事件**"。

### 1.10.1 先建立直觉：法庭书记员

1.9 节用了"游戏存档"和"档案柜"比喻 checkpointer 和 store。RunJournal 用另一个比喻：

```
┌──────────────────────────────────────────────────┐
│  法庭（agent 运行过程）                             │
│                                                   │
│  律师发言 → 证人作证 → 律师反驳 → 法官裁决          │
│     │           │           │           │         │
│     ▼           ▼           ▼           ▼         │
│  ┌─────────────────────────────────────────────┐ │
│  │  书记员（RunJournal）                         │ │
│  │                                              │ │
│  │  把每一步记成"庭审记录"（RunEvent）             │ │
│  │  每条记录盖个 seq 序号                        │ │
│  │  攒够 20 条就批量归档（flush）                 │ │
│  └─────────────────────────────────────────────┘ │
│                       │                           │
│                       ▼                           │
│              ┌─────────────────┐                  │
│              │ 档案室           │                  │
│              │ (RunEventStore) │                  │
│              │ memory/db/jsonl │                  │
│              └─────────────────┘                  │
└──────────────────────────────────────────────────┘
```

- **书记员 = RunJournal**：在法庭里坐着，记录每一步；
- **庭审记录 = RunEvent**：一条条结构化记录（谁、什么时候、干了啥、用了多少 token）；
- **档案室 = RunEventStore**：记录最终归档的地方（三种后端）。

**和 checkpointer 的区别**（这是最容易搞混的）：

| | Checkpointer（1.9 节） | RunJournal（本节） |
|---|---|---|
| **比喻** | 游戏存档 | 法庭书记员 |
| **给谁用** | 给**机器**用（恢复图的状态） | 给**人**用（审计、调试、计费） |
| **存什么** | ThreadState 完整快照（messages、todos、sandbox） | 事件流（谁调了什么、用了多少 token） |
| **去掉会怎样** | agent 失忆，无法续聊 | agent 还能跑，但你不知道它干了啥 |

一句话：**checkpointer 是给机器看的"存档"，RunJournal 是给人看的"录像"**。

### 1.10.2 它是什么对象？挂在哪里？

RunJournal 是 `BaseCallbackHandler` 的子类（`runtime/journal.py:38`）：

```python
class RunJournal(BaseCallbackHandler):
    """LangChain callback handler that captures events to RunEventStore."""
```

**关键认知**：它**不是**中间件，**不是**工具，它是 **LangChain 回调处理器**。

挂载发生在 `runtime/runs/worker.py`（和挂 checkpointer/store 在同一个函数里）：

```python
# worker.py 大约 171-235 行附近
# 1. 建一个 RunJournal
journal = RunJournal(
    run_id=record.run_id,
    thread_id=thread_id,
    event_store=event_store,
    progress_reporter=lambda snap: run_manager.update_run_progress(...),
)

# 2. 把它塞进 config["callbacks"]——LangChain/LangGraph 会自动触发它的回调
config.setdefault("callbacks", []).append(journal)

# 3. 把它也塞进 Runtime.context，让中间件能手动拿它记事件
Runtime(context={..., "__run_journal": journal}, ...)
```

**两个挂载点**：

1. **`config["callbacks"]`**：LangChain/LangGraph 在每次 LLM 调用、工具调用时会**自动**触发 journal 的回调方法（`on_llm_end` 等）；
2. **`Runtime.context["__run_journal"]`**：让那些**不经过 LangChain 回调**的代码（比如某些中间件）也能**手动**往 journal 里写事件。

### 1.10.3 它记录哪些事件？

RunJournal 实现了这些回调方法，每个对应一类事件：

| 回调方法 | 触发时机 | 记录的事件 | category |
|---|---|---|---|
| `on_chain_start`（parent=None） | 整个 run 开始 | `run.start` | trace |
| `on_chain_end`（parent=None） | 整个 run 结束 | `run.end` | outputs |
| `on_chain_error` | run 出错 | `run.error` | error |
| `on_chat_model_start` | 模型开始调用 | `llm.human.input`（抽首条 user msg） | message |
| `on_llm_end` | 模型返回 | `llm.ai.response`（含 usage、latency） | message |
| `on_llm_error` | 模型调用失败 | `llm.error` | trace |
| `on_tool_end` | 工具执行完 | `llm.tool.result` | message |

**每条事件是一个 dict**（`journal.py:367-378`）：

```python
{
    "thread_id": "abc-123",
    "run_id": "run-456",
    "event_type": "llm.ai.response",
    "category": "message",
    "content": {... message.model_dump()},   # 完整消息内容
    "metadata": {
        "caller": "lead_agent",              # 谁调的
        "usage": {"input_tokens": 100, "output_tokens": 50, "total_tokens": 150},
        "latency_ms": 2300,                  # 这次调用花了多久
        "llm_call_index": 3,                 # 本次 run 的第几次 LLM 调用
    },
    "created_at": "2026-06-15T10:30:00Z",
}
```

事件存储会给每条事件分配一个**单调递增的 `seq`**（在 thread 范围内）。按 `seq` 排序就能**完整重放整个 run**——这就是"录像"的回放。

### 1.10.4 三个关键设计决策（值得学）

读 `journal.py:1-16` 的模块文档，里面写了三个"故意这么干"的决策：

#### 决策 ①：故意不实现 `on_llm_new_token`

> 模块文档第 8 行：`on_llm_new_token is NOT implemented -- only complete messages via on_llm_end`

**为什么？** `on_llm_new_token` **每个 token 触发一次**，一次 LLM 调用可能触发上千次。每次都写事件存储会**洪水般刷盘**。

DeerFlow 的取舍：

- **流式 token** 走 SSE 直推前端（通过 `graph.astream(stream_mode="messages")`）；
- **事件存储只记完整消息**（`on_llm_end`，每个 LLM 调用记一次）。

这样事件存储的写入频率可控，流式体验靠 SSE 实时通道，**互不干扰**。

> 💡 **学习启示**：实时流（给用户看的）和持久化记录（给审计用的）应该**分成两条通道**。混在一起会既卡又乱。

#### 决策 ②：用 `on_chat_model_start` 抓首条 human message

> 模块文档第 9-11 行：`on_chat_model_start captures structured prompts ... because it is more reliable than on_chain_start`

**为什么不用 `on_chain_start`？** 因为它**在每个节点**（model node、tools node）都触发，而且那时消息可能还没结构化。而 `on_chat_model_start` **只在真正调 LLM 时触发**，且消息是完整结构化的、没被 checkpoint trimming 压缩。所以抓首条 human message 更可靠。

#### 决策 ③：caller 识别靠 tags

**问题**：一次 run 里，LLM 可能被**三方**调用：

- 主 agent（lead_agent）
- 子 agent（subagent）
- 中间件（middleware，比如摘要中间件自己也会调 LLM）

**怎么区分？** 靠 LangChain 的 `tags` 参数（`journal.py:428-436`）：

```python
def _identify_caller(self, tags: list[str] | None) -> str:
    _tags = tags or []
    for tag in _tags:
        if isinstance(tag, str) and (tag.startswith("subagent:")
                                      or tag.startswith("middleware:")
                                      or tag == "lead_agent"):
            return tag
    return "lead_agent"   # 默认是主 agent
```

然后按 caller **分桶统计 token**（`journal.py:75-78`）：

```python
# Caller-bucketed token accumulators
self._lead_agent_tokens = 0
self._subagent_tokens = 0
self._middleware_tokens = 0
```

run 结束时，你就能看到："这次对话，主 agent 花 5000 token，子 agent 花 8000 token，中间件花 500 token"——**精确到调用方的 token 归因**。这对计费和成本优化极其重要。

### 1.10.5 核心机制：缓冲 + 异步批量刷写

这是 RunJournal 最精巧的部分，处理一个**根本矛盾**：

- **LangChain 回调是同步的**（`BaseCallbackHandler` 的方法是 `def`）；
- **事件存储是异步的**（`RunEventStore.put_batch` 是 `async`）。

**怎么在同步方法里写异步存储？** 看 `_put` 和 `_flush_sync`（`journal.py:367-405`）：

```python
def _put(self, *, event_type, category, content="", metadata=None):
    self._buffer.append({...})                       # ① 先塞进内存 buffer（O(1)，极快）
    if len(self._buffer) >= self._flush_threshold:   # ② 攒够 20 条
        self._flush_sync()                           #    就刷一次

def _flush_sync(self):
    if not self._buffer: return
    if self._pending_flush_tasks: return             # ③ 已有刷写在跑，跳过（防并发写 SQLite）
    try:
        loop = asyncio.get_running_loop()
    except RuntimeError:
        return                                        # ④ 没事件循环，留 buffer 等会儿
    batch = self._buffer.copy()
    self._buffer.clear()
    task = loop.create_task(self._flush_async(batch))  # ⑤ 异步刷
    self._pending_flush_tasks.add(task)

async def _flush_async(self, batch):
    try:
        await self._store.put_batch(batch)           # ⑥ 批量写事件存储
    except Exception:
        self._buffer = batch + self._buffer          # ⑦ 失败了把事件还回 buffer，下次重试
```

**7 步流程的核心思想**：**同步只入 buffer（不阻塞），异步批量刷盘（高效），失败回退重试（不丢数据）**。

**几个值得记的细节**：

- **防并发写**（第 ③ 步）：`if self._pending_flush_tasks: return`——已有刷写在跑就跳过，避免多个 task 同时写 SQLite 触发 `database is locked`；
- **兜底机制**：worker.py 的 `finally` 块会调 `await journal.flush()`，保证 run 结束时 buffer **一定被刷干净**；
- **失败回退**（第 ⑦ 步）：写失败时 `self._buffer = batch + self._buffer`，把事件**还回 buffer 头部**，下次 flush 重试——**永不丢事件**。

> 💡 **学习启示**：这是处理"同步回调 + 异步存储"矛盾的经典模式。写自己的 telemetry/observability 系统时很值得抄。

### 1.10.6 子 agent 的 token 怎么算进来？

这是个**架构难点**。子 agent 是**独立的 LangGraph 图**，跑在隔离的事件循环里（1.9 节没展开，详见主文档第 2 章），它的 LLM 调用**不会**自动触发父级 RunJournal 的 `on_llm_end`。

**那子 agent 的 token 怎么算到这次 run 头上？** DeerFlow 的解法（`journal.py:440-483`）：

```python
def record_external_llm_usage_records(self, records: list[dict]):
    """Record token usage from external sources (e.g., subagents)."""
    for record in records:
        source_id = str(record.get("source_run_id", ""))
        if source_id in self._counted_external_source_ids:
            continue                                    # ★ 去重（按 source_run_id）

        total_tk = record.get("total_tokens", 0) or 0
        ...
        self._counted_external_source_ids.add(source_id)
        self._total_tokens += total_tk

        caller = str(record.get("caller", ""))
        if caller.startswith("subagent:"):
            self._subagent_tokens += total_tk           # 算进 subagent 桶
```

**完整流程**：

```
子 agent 在隔离事件循环里跑
  → 自己装了 SubagentTokenCollector（也是 callback），记录用了多少 token
  → 跑完后，task_tool 把这些记录回传给父级
  → 父级调 journal.record_external_llm_usage_records(records)
  → 用 source_run_id 去重，累加进对应桶
```

这就是为什么 `task_tool.py:145-163` 有 `_report_subagent_usage(runtime, result)` 这步——**把子 agent 的 token 算到父 agent 的 journal 头上**。

> 💡 **学习启示**：多 agent 系统的 token 计费是个容易被忽视的难点。子 agent 是独立图，usage 不会自动流回父图。DeerFlow 的做法是「子 agent 装 collector 采集 → task 工具回传 → 父 journal 聚合」。

### 1.10.7 进度上报：让前端实时看到"跑到哪了"

RunJournal 还干一件事：**节流的进度上报**（`journal.py:532-581` 的 `_schedule_progress_flush`）。

每次 `on_llm_end` 累加完 token，会调 `_schedule_progress_flush()`，它把当前的 token 统计、消息数等**节流地**（每 5 秒最多一次）通过 `progress_reporter` 回调上报给 RunManager，RunManager 再更新 RunRecord，最终通过 StreamBridge 推给前端。

**为什么节流？** 因为 ReAct 循环可能一轮调好几次 LLM，每次都上报会让前端频繁刷新、SSE 消息爆炸。5 秒一次是体验和实时性的平衡。

### 1.10.8 RunJournal vs StreamBridge：录像 vs 直播

这俩最容易搞混。一张表说清：

| | StreamBridge | RunJournal |
|---|---|---|
| **比喻** | **直播** | **录像** |
| **推什么** | 实时 chunk（model 的每个输出、tool 的每次调用） | 持久化事件（结构化的 RunEvent） |
| **谁在看** | 前端在**实时看** | 人/分析工具**事后查** |
| **存哪** | 内存队列（推完就没了，除非前端自己存） | RunEventStore（永久保存） |
| **去哪了** | SSE 推给浏览器，浏览器关了就没了 | 写进 db/jsonl，重启还在 |

**它们是互补的，不是替代的**：

- StreamBridge 让用户**实时看到** agent 在干嘛（"正在思考...""调用 ls...""/tmp 下有 a.txt"）；
- RunJournal 让你**事后能查**这次 run 用了多少 token、调了哪些模型、哪里出了错。

### 1.10.9 中间件也能手动记事件

有些事件**不经过 LangChain 回调**，比如 `SafetyFinishReasonMiddleware` 检测到内容审核截断时，它想记一条审计事件。怎么记？

通过 `Runtime.context["__run_journal"]` 拿到 journal，调 `record_middleware`（`journal.py:489-508`）：

```python
# 伪代码（safety_finish_reason_middleware.py 里）
journal = runtime.context.get("__run_journal")
if journal:
    journal.record_middleware(
        tag="safety_termination",
        name="SafetyFinishReason",
        hook="after_model",
        action="suppress_tool_calls",
        changes={"reason": "content_filter", "suppressed_tool_call_count": 2},
    )
```

`record_middleware` 会产生一条 `category="middleware"` 的事件，方便事后审计"这次 run 里发生了几次安全终止"。

**双下划线前缀 `__run_journal`** 是 DeerFlow 的约定：标记"runtime 内部字段"，不出现在正常状态里，单元测试/子 agent/无事件存储的场景下它就是 `None`，中间件要自己判空。

### 1.10.10 小结：1.10 节你该记住的

1. **RunJournal 是 LangChain 回调处理器**，挂在 agent 运行过程中，记录每次 LLM/工具调用；
2. **它是"会话录像"**——和 checkpointer（给机器恢复用）正交，它是给人审计/调试/计费用的；
3. **记录 7 类事件**（run.start/end/error、llm.human.input/ai.response/error、llm.tool.result）；
4. **三个关键决策**：不记 token 流（只记完整消息）、用 on_chat_model_start 抓首条 user msg、按 tags 分 caller 桶；
5. **核心机制**：同步入 buffer → 攒 20 条异步批量刷 → 失败回退重试 → finally 兜底 flush；
6. **子 agent 的 token** 通过 `record_external_llm_usage_records` 手动累加（按 source_run_id 去重）；
7. **和 StreamBridge 互补**：Bridge 是直播（实时推前端），Journal 是录像（事后可查）。

---

# 🎯 现在回头检验：你掌握了吗？

我们用 9 个问题自测。**先自己想答案，再对照下面的解析**。

### 问题 1：用户在浏览器发一条消息，DeerFlow 内部经历的"主路径"是什么？

<details>
<summary>🔍 点击看答案</summary>

```
浏览器 → FastAPI(/api/threads/{id}/runs/stream)
       → services.py:start_run()
       → asyncio.create_task(run_agent(...))
       → worker.py:run_agent()
       → agent_factory(config)  ← 即 make_lead_agent(config)
       → _make_lead_agent()  ← 装配 5 样配料造图
       → graph.astream(...)  ← 开跑 ReAct 循环
       → 每步通过 StreamBridge 推 SSE → 浏览器
```

**关键点**：`make_lead_agent` 是**每次 run 都会调**的（除非有缓存）。所以装配逻辑不能太重。

</details>

### 问题 2：为什么 `create_chat_model` 必须传 `attach_tracing=False`？

<details>
<summary>🔍 点击看答案</summary>

因为 tracing 回调**已经在 `_make_lead_agent` 的图根节点挂了一次**（`config["callbacks"] = [*existing, *tracing_callbacks]`）。

如果模型自己再挂一次（`attach_tracing=True`），会导致：

1. **重复 span**——同一个 LLM 调用，根节点记一次、模型自己又记一次；
2. **Langfuse 字段丢失**——Langfuse handler 必须在根节点才能正确传播 `langfuse_session_id` / `langfuse_user_id`，在模型层挂就丢了。

这是 DeerFlow 团队踩坑后定的铁律（INVARIANT）。

</details>

### 问题 3：`ThreadState` 里 `todos` 字段的 reducer 为什么是 `merge_todos`（None 时保留旧值），而不是直接覆盖？

<details>
<summary>🔍 点击看答案</summary>

因为 LangGraph 里**每个节点都会跑一次 reducer**（即使节点没改这个字段，LangGraph 也会传 `new=None`）。

如果直接覆盖，一个没动 todos 的中间件就会**意外清空** todos。

`merge_todos` 通过区分 `None`（没动）和 `[]`（显式清空），实现了"未设置保留旧值，显式更新胜出"的语义。

这是 **"区分'未设置'和'显式空'"** 的经典 reducer 模式。

</details>

### 问题 4：为什么 `ClarificationMiddleware` 必须最后装？

<details>
<summary>🔍 点击看答案</summary>

因为 LangChain 的 `after_model` hook 是**逆序执行**的——列表里**最后装的中间件**，其 `after_model` 反而**最先跑**。

ClarificationMiddleware 要**第一时间**拦截 `ask_clarification` 工具调用并中断流程。如果它前面还有别的 `after_model` 在跑（比如 TokenUsage 已经记了一笔），逻辑就乱了。

所以"最后装 = 最先看到 after_model 输出"，这是个**反直觉但重要**的设计。

</details>

### 问题 5：用一句话描述 `create_agent` 造出来的"图"长什么样？

<details>
<summary>🔍 点击看答案</summary>

**两节点图 + 一堆 hook 拦截器**。

- **两节点**：`model`（调 LLM）和 `tools`（执行工具），通过条件边循环（ReAct）；
- **hook 拦截器**：15 个中间件挂的 `before_model` / `after_model` / `wrap_model_call` / `wrap_tool_call` / `before_agent` / `after_agent`，分别在 6 个时机插逻辑。

两节点保证结构简单，hook 拦截器保证扩展性强。

</details>

### 问题 6：为什么 checkpointer 和 store 是"造图后挂上去"，而不是"造图时传进去"？

<details>
<summary>🔍 点击看答案</summary>

因为 **checkpointer 是"运行时基础设施"，不是"agent 定义"的一部分**。

- **agent 定义**（model、tools、middleware、prompt、state_schema）= 这个 agent **是什么**（人格、能力）；
- **checkpointer / store** = 这个 agent **在什么环境跑**（用哪个数据库）。

类比 Web 框架：路由定义和数据库连接是分离的——同一份路由代码，测试用 SQLite、生产用 Postgres。

所以 LangGraph 把它们设计成"**可后挂的属性**"（`agent.checkpointer = cp`，在 worker.py:271-274），而不是 create_agent 的构造参数。这样同一个 agent 定义可以在不同环境用不同后端，**不用改 agent 代码**。

</details>

### 问题 7：DeerFlow 的"用户长期记忆"（比如"用户喜欢吃辣"）存在哪？Checkpointer？Store？还是别的地方？

<details>
<summary>🔍 点击看答案</summary>

**都不是！** 存在**单独的 JSON 文件**里（`agents/memory/storage.py` 的 `FileMemoryStorage`）。

DeerFlow 是**三层持久化**：

| 层 | 存什么 | 实现 |
|---|---|---|
| **Checkpointer** | 图的执行状态（按 thread，短期） | `runtime/checkpointer/` |
| **Store** | 系统级元数据（跨 thread，KV） | `runtime/store/` |
| **Memory** | 用户长期偏好（跨 thread，复杂结构） | `agents/memory/`（JSON 文件） |

Memory 不放 Store 的原因：Store 只是个简单 KV（`put/get`），而 Memory 需要 LLM 抽取、置信度排序、按置信度驱逐——这些语义 KV 装不下。

**三层各司其职**是 DeerFlow 架构的一个亮点。

</details>

### 问题 8：RunJournal 和 checkpointer 都在"记录"，它俩有啥区别？

<details>
<summary>🔍 点击看答案</summary>

**完全不同的用途**——一个是给机器用的，一个是给人用的：

| 维度 | Checkpointer | RunJournal |
|---|---|---|
| **比喻** | 游戏存档 | 法庭录像 |
| **目的** | 让图能**恢复**（续聊、回滚） | 让人能**审计/调试/计费** |
| **存什么** | ThreadState 完整快照（messages、todos、sandbox...） | 事件流（谁调了什么、用了多少 token） |
| **谁来读** | LangGraph 自动读（下次同 thread_id 续聊时） | 人/前端/分析工具主动查 |
| **粒度** | 每个超级步一个完整快照 | 每次调用一条事件 |
| **去掉会怎样** | agent 失忆，无法续聊 | agent 还能跑，但你不知道它干了啥、花了多少 token |

**一句话**：checkpointer 是给**机器**用的（恢复状态），RunJournal 是给**人**用的（理解发生了什么）。它们是**正交**的，各管一摊。

</details>

### 问题 9：RunJournal 为什么故意不实现 `on_llm_new_token`？

<details>
<summary>🔍 点击看答案</summary>

因为 `on_llm_new_token` **每个 token 触发一次**，一次 LLM 调用可能触发上千次。如果每次都写事件存储（SQLite/Postgres），会**洪水般刷盘**，严重拖慢 agent。

DeerFlow 的取舍是**分两条通道**：

- **流式 token** → 走 SSE 直推前端（通过 `graph.astream(stream_mode="messages")`），让用户实时看到打字效果；
- **持久化事件** → 只在 `on_llm_end` 记一次完整消息（含 usage/latency），写入事件存储。

这样事件存储的写入频率可控（每个 LLM 调用记一次），流式体验靠 SSE 实时通道，**互不干扰**。

**学习启示**：实时流（给用户看的）和持久化记录（给审计用的）应该**分成两条通道**。

</details>

---

# 📝 一页纸总结（第 0 + 1 章精华）

```
┌─────────────────────────────────────────────────────────────────┐
│  DeerFlow 对外只有"一个图"——lead_agent（langgraph.json 声明）    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  make_lead_agent(config) ← langgraph.json 注册的工厂              │
│    └─ _make_lead_agent(config, app_config) ← 真正的装配车间        │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
   解析开关              装配 5 样配料           返回 create_agent(...)
   (model/mode/...)      ┌──────────────┐         造出的图
                         │ ① model      │
                         │ ② tools      │
                         │ ③ middleware │  ← build_middlewares() 装的 15 个
                         │ ④ prompt     │
                         │ ⑤ state_schema│ ← ThreadState（9 个字段 + reducer）
                         └──────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  create_agent 造出来的图 = 两节点（model ↔ tools）+ 15 组 hook    │
│                                                                   │
│  中间件管道（装配顺序，注意 after_model 执行是逆序的）：             │
│  基础 → DynamicContext → Skill → Summarization → Todo →           │
│  TokenUsage → Title → Memory → ViewImage → DeferredToolFilter →   │
│  SubagentLimit → LoopDetection → [自定义] → SafetyFinishReason →  │
│  Clarification（永远最后）                                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  worker.py 运行时再"挂"三样东西（1.9 + 1.10 节）                   │
│                                                                   │
│    agent.checkpointer = cp   ← 游戏存档（按 thread 存图状态，给机器）│
│    agent.store = store       ← 档案柜（跨 thread 存 KV 元数据）     │
│    config["callbacks"] += journal                            │
│    Runtime.context["__run_journal"] = journal                │
│                              ← 法庭录像（记每步事件/token，给人）    │
│                                                                   │
│  ★ 挂载而非传参：持久化是"环境"，不是"agent 定义"，要解耦            │
│  ★ checkpointer/store 三种后端：memory / sqlite / postgres        │
│  ★ RunJournal：同步入 buffer → 攒 20 条异步批量刷 → 失败回退重试    │
└─────────────────────────────────────────────────────────────────┘
```

**5 句话记住**：

1. **整个系统对外是一张图**，由 `make_lead_agent` 工厂函数按需装配；
2. **装配 = 5 样配料**（model / tools / middleware / prompt / state_schema）；
3. **15 个中间件按严格顺序装**，逆序执行的 `after_model` 让"最后装 = 最先看到输出"；
4. **图造完后，worker 再"挂"上 checkpointer 和 store**——前者是按 thread 的"游戏存档"（让对话能续聊），后者是跨 thread 的"档案柜"（存线程列表等元数据）；用户长期记忆则在独立的 JSON 文件里（第三层）；
5. **worker 还挂了 RunJournal**——一个 LangChain 回调处理器，是这次 run 的"法庭录像"，记下每步事件和 token 用量（按 caller 分桶），和 StreamBridge 的"直播"互补。

---

# 🚀 接下来怎么学？

看完这份精讲，建议你做这 3 件事：

### 1. 动手验证

打开 `lead_agent/agent.py`，在 `_make_lead_agent` 里加一行 print：

```python
print(f"🔧 造 agent: model={model_name}, subagent={subagent_enabled}, plan={is_plan_mode}")
```

重启服务，发条消息，看日志。你会真实看到这个函数被调用。

### 2. 玩中间件

写一个最简单的自定义中间件（5 行），在 `after_model` 里打印每条 AI 消息的 token 数：

```python
from langchain.agents.middleware import AgentMiddleware

class MyMiddleware(AgentMiddleware):
    def after_model(self, state, response):
        msg = response.message
        if hasattr(msg, 'usage_metadata') and msg.usage_metadata:
            print(f"📊 这一步用了 {msg.usage_metadata.get('total_tokens')} tokens")
        return {}
```

挂到 `extra_middleware`（SDK 用法），看效果。

### 3. 继续往下读

第 0、1 章搞懂后，**强烈建议**第 2 章（多 Agent 调度）和第 3 章（记忆）二选一深读：

- 喜欢"架构 + 并发"的 → 第 2 章（task_tool + SubagentExecutor）；
- 喜欢"AI + 数据"的 → 第 3 章（Memory updater + 抽取 prompt）。

---

**到这里，第 0 章和第 1 章你应该彻底搞懂了。** 🎉

如果还有具体段落/代码行没明白，**告诉我具体是哪一段**（比如"1.7.3 第 8 组的 SafetyFinishReason 我还是没懂为什么逆序"），我可以再细讲。
