# 第 2 章：AgentExecutor 才是真正的执行发动机

> 如果说 `Agent` 是“一个员工的简历和工具箱”，那 `AgentExecutor` 就是“这个员工开始干活时的大脑循环”。

## 2.1 Agent 自己不是循环

`Agent` 定义了 role、goal、backstory、llm、tools、memory、planning、respect_context_window 等配置。

位置：`lib/crewai/src/crewai/agent/core.py:171`

但真正执行时，`Agent.execute_task()` 最后会调用：

```python
invoke_result = self.agent_executor.invoke(...)
```

位置：`lib/crewai/src/crewai/agent/core.py:905`

这意味着：读 CrewAI 的 agent，不要只停在 `Agent` 类。一定要继续看 executor。

## 2.2 executor 是怎么创建的

入口：

`Agent.create_agent_executor()`：`lib/crewai/src/crewai/agent/core.py:1074`

它做几件事：

1. `raw_tools = tools or self.tools or []`
2. `parse_tools(raw_tools)` 把 `BaseTool` 转成 `CrewStructuredTool`
3. `_build_execution_prompt(raw_tools)` 构造执行 prompt 和 stop words
4. 如果 executor 已存在，就更新参数
5. 否则实例化 `self.executor_class(...)`

关键参数包括：

- `llm`
- `task`
- `agent`
- `crew`
- `tools`
- `prompt`
- `original_tools`
- `stop_words`
- `max_iter`
- `tools_handler`
- `respect_context_window`
- `callbacks`
- `response_model`

这些参数在 `lib/crewai/src/crewai/agent/core.py:1101` 附近传入。

## 2.3 新 executor 的状态模型

当前默认 executor 是 `crewai.experimental.agent_executor.AgentExecutor`。

源码位置：`lib/crewai/src/crewai/experimental/agent_executor.py:164`

它继承：

```text
Flow[AgentExecutorState]
BaseAgentExecutor
```

状态模型在 `lib/crewai/src/crewai/experimental/agent_executor.py:126`：

```text
messages
iterations
current_answer
is_finished
ask_for_human_input
use_native_tools
pending_tool_calls
plan
plan_ready
todos
replan_count
observations
execution_log
```

这非常重要：CrewAI 的新 executor 不只是传统 ReAct while loop，而是把执行过程建模成 Flow 状态机。它既能跑普通 ReAct，也能支持 plan-and-execute、todo、observer、replan。

## 2.4 LLM 调用与解析

普通文本工具调用路径里的核心函数：

`call_llm_and_parse()`：`lib/crewai/src/crewai/experimental/agent_executor.py:1378`

它做的事：

```text
1. enforce_rpm_limit()
2. get_llm_response(...)
3. 如果 answer 是 Pydantic BaseModel，直接视为 AgentFinish
4. 否则 process_llm_response(answer, use_stop_words)
5. 把解析结果放到 state.current_answer
6. 如果上下文超限，返回 "context_error"
```

这里的 `process_llm_response()` 会把 LLM 输出解析成：

- `AgentAction`：说明还要调用工具。
- `AgentFinish`：说明可以给最终答案。

## 2.5 native tools 与文本 tools

CrewAI 同时支持两种工具调用方式：

1. native function calling：LLM 原生返回 tool calls。
2. text tool calling：LLM 生成类似 Action / Action Input 的文本，再由 parser 解析。

判断 native tool 支持的位置：

- `Agent._supports_native_tool_calling()`：`lib/crewai/src/crewai/agent/core.py:519`
- `AgentExecutor._check_native_tool_support()`：`lib/crewai/src/crewai/experimental/agent_executor.py:236`

如果 native tools 不支持，executor 可以降级到文本工具调用：

`lib/crewai/src/crewai/experimental/agent_executor.py:249`

## 2.6 max_iter 不是摆设

executor state 里有 `iterations`，executor 字段里有 `max_iter`。

当循环次数达到上限时，CrewAI 会尝试让 LLM 给出最终答案，而不是无休止调用工具。相关逻辑在：

`lib/crewai/src/crewai/utilities/agent_utils.py:300`

## 2.7 旧 executor 还在

源码里仍有 `CrewAgentExecutor`，并且 `agent/core.py` 里有兼容映射：

```python
_EXECUTOR_CLASS_MAP = {
    "CrewAgentExecutor": CrewAgentExecutor,
    "AgentExecutor": AgentExecutor,
}
```

如果传 `CrewAgentExecutor` 会有弃用警告。位置：`lib/crewai/src/crewai/agent/core.py:142`

学习建议：

- 先看新 `experimental/agent_executor.py`，理解当前方向。
- 再看旧 `agents/crew_agent_executor.py`，它的传统 ReAct loop 更直观，适合对照。

## 经典问题

### Q1：Agent 的 prompt 是在哪里生成的？

不是在 `Agent.execute_task()` 里硬编码字符串。它在 `_build_execution_prompt()` 中通过 `Prompts(...).task_execution()` 生成。

源码：`lib/crewai/src/crewai/agent/core.py:1037`

### Q2：为什么 executor 要保存 `messages`？

因为每次 LLM 调用、工具结果、观察结果都要进入下一轮上下文。`Task.context` 是任务之间的输出传递；`executor.messages` 是单个 agent 执行当前任务时的内部对话轨迹。

### Q3：为什么 AgentExecutor 要继承 Flow？

因为它不再只是 while loop。它有 plan、todo、observation、replan、context_error recovery 等多个路由节点。用 Flow 表达这些状态转移，比把所有逻辑塞进一个巨大循环更可控。

