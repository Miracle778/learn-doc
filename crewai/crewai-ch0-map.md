# 第 0 章：先建立心理模型

> CrewAI 可以先理解成一个“项目执行系统”：
>
> `Crew` 是项目组，`Task` 是工作单，`Agent` 是具备角色、目标、工具和 LLM 的执行者，`AgentExecutor` 是真正跑 ReAct / plan-and-execute 循环的发动机。

## 0.1 一句话总览

CrewAI 的主线不是“一个 agent 聊天”，而是：

```text
Crew.kickoff()
  -> 选择 process: sequential / hierarchical
  -> 遍历 Task
  -> 为每个 Task 选 Agent 和 tools
  -> Task.execute_sync()
  -> Agent.execute_task()
  -> AgentExecutor.invoke()
  -> LLM / tool / observation 循环
  -> TaskOutput
  -> CrewOutput
```

对照源码：

- `Crew` 字段定义：`lib/crewai/src/crewai/crew.py:220`
- `Crew.kickoff()`：`lib/crewai/src/crewai/crew.py:980`
- sequential / hierarchical 分支：`lib/crewai/src/crewai/crew.py:1037`
- `Task` 字段定义：`lib/crewai/src/crewai/task.py:114`
- `Agent` 字段定义：`lib/crewai/src/crewai/agent/core.py:171`
- `AgentExecutorState`：`lib/crewai/src/crewai/experimental/agent_executor.py:126`

## 0.2 四层结构

你可以把 CrewAI 分成四层：

```text
应用编排层
  Crew / Flow / Project annotations / YAML config

任务执行层
  Task / TaskOutput / guardrails / context / async task

Agent 执行层
  Agent / AgentExecutor / prompt / tools / LLM call / parser

能力支撑层
  Memory / Knowledge / RAG storage / events / checkpoint / tracing / MCP / A2A
```

这四层要分开看。很多初学者会混淆：

- `Crew` 不直接调用 LLM，它组织任务。
- `Task` 不会自己思考，它把任务交给 agent。
- `Agent` 本身更像配置和入口，它把 prompt、tools、LLM、callbacks 装配成 executor。
- `AgentExecutor` 才是真正反复调用 LLM、解析 action、执行工具、追加 observation 的循环。

## 0.3 和官方文档对齐

官方文档把 agent 描述为能执行任务、做决策、使用工具、协作、记忆和委派的单元。源码里这些能力分别落在不同位置：

| 能力 | 源码落点 |
| --- | --- |
| role / goal / backstory | `Agent` 字段 |
| 使用工具 | `Agent.create_agent_executor()` + `parse_tools()` |
| 多 agent 协作 | `AgentTools` + delegation tools |
| 记忆 | `Memory` + `memory_tools` + `Agent._retrieve_memory_context()` |
| context window | `handle_context_length()` + `summarize_messages()` |
| 计划和推理 | `AgentExecutorState.plan/todos/observations` |

## 0.4 第一遍应该怎么读

第一遍不要从所有目录开始扫。照这个顺序：

1. 打开 `crew.py`，只看 `kickoff()` 和 `_execute_tasks()`。
2. 打开 `task.py`，只看 `_execute_core()`。
3. 打开 `agent/core.py`，只看 `execute_task()` 和 `create_agent_executor()`。
4. 打开 `experimental/agent_executor.py`，只看 `AgentExecutorState` 和 `call_llm_and_parse()`。
5. 再回头看 memory、context、Flow。

## 经典问题

### Q1：CrewAI 的最小运行单位是 Agent 还是 Task？

从用户 API 看，最小运行入口可以是 `Agent.kickoff()` 或 `Crew.kickoff()`；但在 Crew 主流程里，真正被调度的是 `Task`。`Crew._execute_tasks()` 遍历任务列表，调用 `task.execute_sync()` 或 `task.execute_async()`，再由 task 交给 agent 执行。

源码证据：

- `Crew._execute_tasks()` 遍历 task：`lib/crewai/src/crewai/crew.py:1553`
- 同步 task 执行：`lib/crewai/src/crewai/crew.py:1585`
- `Task._execute_core()` 调 `agent.execute_task()`：`lib/crewai/src/crewai/task.py:790`

### Q2：CrewAI 是 LangChain 包装器吗？

不是。README 明确说 CrewAI 是 independent of LangChain；本地源码里核心 executor、flow、memory 都在 CrewAI 自己的包里。它会支持 MCP、OpenAI-compatible LLM、tools 等生态，但核心调度不是 LangChain agent 的薄封装。

### Q3：为什么源码里既有 Crew 又有 Flow？

`Crew` 偏“自治协作”：让 agent 根据任务和工具自主执行。

`Flow` 偏“确定性工作流”：用 `@start`、`@listen`、`@router` 把 Python 方法组织成事件驱动的流程。官方 Flows 文档也强调 state management、event-driven architecture、conditional logic。

看源码时记住：Flow 是更底层的编排模型，AgentExecutor 也继承 `Flow[AgentExecutorState]`，说明 CrewAI 正在把 agent 执行循环也建模成一种 flow。

