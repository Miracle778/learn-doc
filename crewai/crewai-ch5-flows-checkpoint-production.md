# 第 5 章：Flow、checkpoint 和生产化视角

> 前几章讲的是 Crew 和 Agent 怎么跑。这一章看更生产化的东西：Flow、checkpoint、事件上下文。

## 5.1 Flow 是确定性编排

官方 Flows 文档强调三件事：

- state management
- event-driven architecture
- conditional logic / branching

源码入口：

`lib/crewai/src/crewai/flow/flow.py`

这个文件现在主要是兼容导出层。真正实现被拆成：

```text
crewai.flow.dsl
crewai.flow.flow_definition
crewai.flow.runtime
crewai.experimental.conversational_mixin
```

位置：`lib/crewai/src/crewai/flow/flow.py:1`

## 5.2 Flow 的三个核心装饰器

### `@start`

标记入口节点。

源码：`lib/crewai/src/crewai/flow/dsl/_start.py:18`

### `@listen`

监听上游方法或路由事件。

源码：`lib/crewai/src/crewai/flow/dsl/_listen.py:18`

### `@router`

根据返回值发出路由事件。

源码：`lib/crewai/src/crewai/flow/dsl/_router.py:97`

这三个装饰器最终都会生成 `FlowMethodDefinition`，再交给 runtime 执行。

## 5.3 Flow runtime 做什么

runtime 入口：

`lib/crewai/src/crewai/flow/runtime/__init__.py:1`

它负责：

- kickoff / resume
- listener dispatch
- state proxy
- tracing
- event bus
- persistence
- memory scope/slice
- checkpoint

从 import 列表就能看出来，Flow runtime 是 CrewAI 生产化能力的汇合点。

## 5.4 Flow 可以调用 Crew 和 Agent

Flow definition action 支持调用 crew 或 agent。

源码：

- `CrewAction.run()`：`lib/crewai/src/crewai/flow/runtime/_actions.py:124`
- `AgentAction.run()`：`lib/crewai/src/crewai/flow/runtime/_actions.py:163`

`CrewAction` 会 load crew，然后：

```python
return await crew.kickoff_async(inputs=inputs)
```

位置：`lib/crewai/src/crewai/flow/runtime/_actions.py:160`

`AgentAction` 会 load agent，然后：

```python
return await agent.kickoff_async(...)
```

位置：`lib/crewai/src/crewai/flow/runtime/_actions.py:186`

所以 Flow 和 Crew 的关系可以这样理解：

```text
Flow 控制确定性流程
Crew 负责自治 agent 协作
Flow 可以在某个节点调用 Crew
Crew 内部又会调 Agent / Task / Executor
```

## 5.5 checkpoint 恢复不只是读 JSON

官方 Checkpointing 文档说：checkpoint 可以保存执行状态，失败后从最后 checkpoint 恢复，跳过已完成任务。

源码里恢复很细：

`Crew.from_checkpoint()`：`lib/crewai/src/crewai/crew.py:404`

恢复后还要 `_restore_runtime()`：

`lib/crewai/src/crewai/crew.py:452`

它会做这些事：

1. 从 runtime state 找出 started tasks。
2. 判断 hierarchical 下正在执行的是 manager 还是 task agent。
3. 重新绑定 agent.crew。
4. 如果 executor 正在恢复，重绑 executor.crew / executor.agent。
5. 重新绑定 task.agent。
6. 恢复原始 task description / expected_output。
7. 恢复 inputs / kickoff event id / train flag。
8. 重新绑定 memory views。
9. 恢复 event scope。

关键位置：

- 重绑 agent/executor：`lib/crewai/src/crewai/crew.py:476`
- 重绑 task.agent：`lib/crewai/src/crewai/crew.py:489`
- memory view 重绑：`lib/crewai/src/crewai/crew.py:514`
- event scope 恢复：`lib/crewai/src/crewai/crew.py:547`

## 5.6 为什么 memory view 要重绑

`MemoryScope` / `MemorySlice` 是对 `Memory` 的视图。checkpoint JSON 不能保存 live `Memory` 对象，所以恢复后需要重新 bind。

源码注释说得很清楚：

`lib/crewai/src/crewai/crew.py:514`

如果不重绑，恢复后的 scope/slice 第一次使用会报错。

## 5.7 ExecutionContext 保存什么

文件：

`lib/crewai/src/crewai/context.py`

`ExecutionContext` 是一组 `contextvars` 的快照：

- current task id
- flow request id
- flow id
- flow method name
- event id stack
- last event id
- triggering event id
- emission sequence
- platform token

捕获：

`capture_execution_context()`：`lib/crewai/src/crewai/context.py:80`

恢复：

`apply_execution_context()`：`lib/crewai/src/crewai/context.py:101`

这就是为什么 CrewAI 能在线程、异步、checkpoint 之间尽量保持事件链和执行上下文。

## 经典问题

### Q1：什么时候用 Crew，什么时候用 Flow？

用 Crew：你希望 agent 自主完成任务，允许工具调用、委派、推理、尝试。

用 Flow：你需要确定性的业务流程，比如先校验输入，再查数据库，再调用 Crew，再人工审批，再写回系统。

常见组合是：Flow 包住 Crew。

### Q2：checkpoint 为什么要恢复 event scope？

因为生产系统不只关心“结果能不能继续跑”，还关心 tracing、事件父子关系、已发事件序号。如果恢复后事件链断了，监控和审计会很难看懂。

### Q3：AgentExecutor 本身继承 Flow，和用户写的 Flow 是一回事吗？

它们使用同一套 Flow 思想和 runtime 能力，但粒度不同：

- 用户 Flow：业务流程。
- AgentExecutor Flow：单个 agent 执行一个任务时的内部状态机。

这也是 CrewAI 当前源码里很值得学习的设计：把 agent 推理循环抽象成可观察、可路由、可恢复的状态机。

