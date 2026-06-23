# 第 3 章：多 agent 是怎么调度的

> CrewAI 的多 agent 调度不是一个隐藏的中央 scheduler，而是两种 process 加一组 delegation tools。

## 3.1 两种 process

当前源码只实现：

```text
Process.sequential
Process.hierarchical
```

位置：`lib/crewai/src/crewai/process.py:1`

官方 Processes 文档也把 consensual 标为 planned。

## 3.2 sequential：任务列表就是调度计划

sequential 模式里，Crew 按任务列表顺序推进：

```text
task1 -> task2 -> task3
```

每个 task 上可以绑定自己的 agent：

```python
Task(description="Research...", agent=researcher)
Task(description="Write...", agent=writer)
```

源码里 `_get_agent_to_use()` 在 sequential 下直接返回 `task.agent`：

`lib/crewai/src/crewai/crew.py:1685`

这说明 sequential 下“谁做什么”主要由 task 配置决定。

## 3.3 hierarchical：manager agent 接管任务分配

hierarchical 模式先创建 manager：

`lib/crewai/src/crewai/crew.py:1489`

如果用户传了 `manager_agent`：

- 设置 `allow_delegation=True`
- 禁止 manager 自带 tools

如果没有传：

- 用 `manager_llm` 创建一个默认 manager agent
- 给它注入 `AgentTools(agents=self.agents).tools()`
- 开启 `allow_delegation=True`

位置：`lib/crewai/src/crewai/crew.py:1494`

这个 manager agent 的本质很普通：它也是一个 Agent，只是拥有“委派给其他 agent”的工具。

## 3.4 delegation tools 是关键

`AgentTools` 会创建两个工具：

- `DelegateWorkTool`
- `AskQuestionTool`

源码：

- `lib/crewai/src/crewai/tools/agent_tools/agent_tools.py:1`
- `DelegateWorkTool`：`lib/crewai/src/crewai/tools/agent_tools/delegate_work_tool.py:1`
- `AskQuestionTool`：`lib/crewai/src/crewai/tools/agent_tools/ask_question_tool.py:1`

真正执行委派的地方在 `BaseAgentTool._execute()`：

`lib/crewai/src/crewai/tools/agent_tools/base_agent_tools.py:46`

它会：

1. 根据 coworker 名称匹配 agent role。
2. 创建一个新的 `Task(description=task, agent=selected_agent, expected_output=...)`。
3. 调用 `selected_agent.execute_task(task_with_assigned_agent, context)`。

源码证据：

- 匹配 agent：`lib/crewai/src/crewai/tools/agent_tools/base_agent_tools.py:80`
- 创建 task：`lib/crewai/src/crewai/tools/agent_tools/base_agent_tools.py:112`
- 调用被委派 agent：`lib/crewai/src/crewai/tools/agent_tools/base_agent_tools.py:120`

## 3.5 hierarchical 模式下每个原始 task 谁执行？

关键函数：

`Crew._get_agent_to_use()`：`lib/crewai/src/crewai/crew.py:1685`

```text
if process == hierarchical:
    return self.manager_agent
return task.agent
```

也就是说，原始 task 先交给 manager agent。manager agent 如果需要专家，就通过 delegation tool 调其他 agent。

这和很多人想象的不一样：

> hierarchical 不是 Crew 在 Python 层自动挑选最合适的 worker，而是 manager agent 在 LLM 层通过工具进行委派。

## 3.6 tools 是如何动态注入的

`Crew._prepare_tools()` 会根据 agent 和 task 动态添加工具。

位置：`lib/crewai/src/crewai/crew.py:1616`

它会检查：

- agent 是否 `allow_delegation`
- 是否 hierarchical
- 是否允许 code execution
- 是否 multimodal
- 是否有 platform apps
- 是否有 MCP
- 是否有 memory
- 是否有 input files

memory tool 注入在：

`lib/crewai/src/crewai/crew.py:1652`

delegation tool 注入在：

`lib/crewai/src/crewai/crew.py:1791`

## 3.7 async task 不是多 agent 调度

`Task.async_execution=True` 只表示这个 task 可以并发执行，不等于 agent 之间自动协作。

源码：

- async 分支开 future：`lib/crewai/src/crewai/crew.py:1568`
- 遇到同步任务时处理 futures：`lib/crewai/src/crewai/crew.py:1579`

多 agent 协作主要靠：

- task 绑定不同 agent
- hierarchical manager
- delegation tools
- task context

## 经典问题

### Q1：多 agent 之间是直接互相发消息吗？

不是直接发消息。CrewAI 把“问同事 / 委派工作”实现成工具调用。一个 agent 调工具，工具内部创建临时 task，并调用另一个 agent 的 `execute_task()`。

### Q2：manager agent 为什么不能有自己的 tools？

源码里如果自定义 manager agent 带 tools，会清空并抛错。原因是 hierarchical manager 的工具集应该主要是 delegation tools，否则 manager 容易绕过团队分工自己做事。

源码：`lib/crewai/src/crewai/crew.py:1498`

### Q3：sequential 能不能委派？

可以，但取决于 agent 的 `allow_delegation`。在 sequential 下，如果当前 task 的 agent 允许 delegation，Crew 会把其他 agents 作为 delegation candidates 注入。

源码：`lib/crewai/src/crewai/crew.py:1630`

