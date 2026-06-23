# 第 1 章：从 `Crew.kickoff()` 走到 `Agent.execute_task()`

> 这一章只追一件事：用户调用 `crew.kickoff()` 后，代码到底怎么把一个任务交给 agent。

## 1.1 时间线

假设你写了：

```python
result = crew.kickoff(inputs={"topic": "agent memory"})
```

源码里的时间线大致是：

```text
[1] Crew.kickoff()
    - 处理 checkpoint restore
    - 处理 stream
    - 准备 inputs / input_files
    - 根据 process 选择 sequential 或 hierarchical

[2] _run_sequential_process() / _run_hierarchical_process()
    - sequential: 直接执行 tasks
    - hierarchical: 先创建 manager_agent，再执行 tasks

[3] _execute_tasks()
    - 遍历 task 列表
    - 处理 conditional task
    - 处理 async task futures
    - 为当前 task 计算 context
    - 调 task.execute_sync()

[4] Task._execute_core()
    - 设置 current_task_id
    - 选 agent
    - 发 TaskStartedEvent
    - 调 agent.execute_task()
    - 包装成 TaskOutput
    - 发 TaskCompletedEvent

[5] Agent.execute_task()
    - 构造 task_prompt
    - 拼 task context
    - 召回 memory
    - 查询 knowledge
    - 准备 tools / training data
    - 调 agent_executor.invoke()
```

## 1.2 `Crew.kickoff()` 是总入口

核心源码在 `lib/crewai/src/crewai/crew.py:980`。

关键分支：

```text
if self.process == Process.sequential:
    result = self._run_sequential_process()
elif self.process == Process.hierarchical:
    result = self._run_hierarchical_process()
```

这个分支在 `lib/crewai/src/crewai/crew.py:1037`。

不要把 `process` 想复杂。源码当前只实现两个：

```python
class Process(str, Enum):
    sequential = "sequential"
    hierarchical = "hierarchical"
```

位置：`lib/crewai/src/crewai/process.py:1`

源码里还保留了 consensual 的计划注释；官方 Processes 文档也说 consensual 是 planned，不是当前实现。

## 1.3 sequential：顺序执行任务

sequential 非常直接：

```text
_run_sequential_process()
  -> _execute_tasks(self.tasks)
```

位置：`lib/crewai/src/crewai/crew.py:1485`

`_execute_tasks()` 是这一章最重要的函数。它维护三个核心变量：

- `task_outputs`：已经完成的任务输出。
- `futures`：异步任务的 future 列表。
- `last_sync_output`：最近一个同步任务输出，用来给 async task 做有限上下文。

关键代码位置：

- 初始化输出和 futures：`lib/crewai/src/crewai/crew.py:1549`
- 遍历任务：`lib/crewai/src/crewai/crew.py:1553`
- async task 分支：`lib/crewai/src/crewai/crew.py:1568`
- sync task 分支：`lib/crewai/src/crewai/crew.py:1578`
- 最终生成 CrewOutput：`lib/crewai/src/crewai/crew.py:1598`

## 1.4 Task context 是怎么来的

`Crew._get_context()` 是一个容易被忽略但很关键的小函数：

```text
如果 task.context 为空：返回 ""
如果 task.context 是 NOT_SPECIFIED：聚合前面 task_outputs 的 raw 输出
否则：只聚合 task.context 指定的那些任务输出
```

位置：`lib/crewai/src/crewai/crew.py:1837`

这解释了一个常见误区：

> Task context 不是完整聊天历史，也不是 executor messages。它是“上游任务输出”的文本聚合。

## 1.5 `Task._execute_core()` 做包装

同步路径：

- `Task.execute_sync()` 设置开始时间后进入 `_execute_core()`。
- `_execute_core()` 设置当前 task id，存文件输入，拿到 agent。
- 调用 `agent.execute_task(task=self, context=context, tools=tools)`。
- 将结果包装成 `TaskOutput`。

源码位置：

- `Task.execute_sync()`：`lib/crewai/src/crewai/task.py:581`
- `Task._execute_core()`：`lib/crewai/src/crewai/task.py:762`
- 调用 agent：`lib/crewai/src/crewai/task.py:790`
- 构造 `TaskOutput`：`lib/crewai/src/crewai/task.py:816`
- 发完成事件：`lib/crewai/src/crewai/task.py:874`

异步路径类似，只是 `Task.execute_async()` 会复制 `contextvars` 并开线程，`Task.aexecute_sync()` 用原生 async：

- 线程异步：`lib/crewai/src/crewai/task.py:596`
- 原生 async：`lib/crewai/src/crewai/task.py:627`

## 1.6 `Agent.execute_task()` 做 prompt 装配

`Agent.execute_task()` 是 agent 进入 executor 前的最后一站。

位置：`lib/crewai/src/crewai/agent/core.py:786`

关键步骤：

```text
task_prompt = _prepare_task_execution(task, context)
task_prompt = handle_knowledge_retrieval(...)
task_prompt = _finalize_task_prompt(task_prompt, tools, task)
result = _execute_without_timeout(task_prompt, task)
return _finalize_task_execution(task, result)
```

`_prepare_task_execution()` 里做了这些事：

- 注入日期。
- 重置 tools handler 的 last_used_tool。
- `task.prompt()` 生成基础任务 prompt。
- 加结构化输出 schema。
- 把 task context 格式化进 prompt。
- 召回 memory 并追加到 prompt。

位置：`lib/crewai/src/crewai/agent/core.py:535`

## 经典问题

### Q1：为什么 task.context 不能引用未来任务？

因为 `_execute_tasks()` 是按任务列表推进的，未来任务还没有 output。源码在 Crew validator 里检查 future task context，防止你声明一个无法满足的依赖。

### Q2：async task 的 context 为什么只拿 `last_sync_output`？

看 `lib/crewai/src/crewai/crew.py:1568`。async task 启动时不能等待所有并发任务都结束，否则就失去并发意义。它只拿最近同步输出作为稳定上下文；遇到下一个同步任务时，Crew 会先 drain futures。

### Q3：`Agent.execute_task()` 为什么不直接拼 tools？

tools 的准备有两层：

- prompt 层：`_finalize_task_prompt()` 调 `prepare_tools()`，让 agent 知道工具描述。
- 执行层：`Crew._prepare_tools()` 根据 delegation、memory、files、MCP 等动态注入工具。

所以 tools 既是 prompt 信息，也是 executor 可调用能力。
