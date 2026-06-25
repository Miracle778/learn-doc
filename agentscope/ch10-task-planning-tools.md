# 第 10 章 任务规划工具：单 agent 内部如何管理 todo？

> 这一章读 Task tools。目标是把“单 agent 内部计划管理”和“多 agent team 调度”区分开。

---

## 10.1 配套源码

打开：

- `src/agentscope/tool/_task/_create_task.py`
- `src/agentscope/tool/_task/_list_task.py`
- `src/agentscope/tool/_task/_get_task.py`
- `src/agentscope/tool/_task/_update_task.py`
- `src/agentscope/tool/_task/_task_tool_base.py`
- `src/agentscope/state/_task.py`
- `src/agentscope/state/_state.py` 的 `TaskContext`
- `src/agentscope/app/_service/_toolkit.py`

---

## 10.2 Task tools 在哪里注册？

服务化场景中，`get_toolkit()` 会 always-on 注册：

```python
tools += [TaskCreate(), TaskList(), TaskGet(), TaskUpdate()]
```

所以普通 agent 和 team worker 都能用任务规划工具，除非你自定义 toolkit 时移除它们。

---

## 10.3 TaskContext 存在哪里？

`AgentState` 里：

```python
tasks_context: TaskContext = Field(default_factory=TaskContext)
```

`TaskContext` 里是：

```python
tasks: list[Task]
```

这意味着任务列表是 session state 的一部分，会随 `ChatService.update_session_state()` 持久化。

---

## 10.4 Task 和 Team 的区别

```text
Task tools
  一个 agent 内部的 todo 管理
  状态存在 AgentState.tasks_context
  适合拆步骤、标状态、记录依赖

Team tools
  多个 agent/session 协作
  状态存在 TeamRecord/AgentRecord/SessionRecord + MessageBus
  适合并行分工、不同权限模板、异步汇报
```

不要因为任务复杂就一定开 team。很多任务只需要 task list。

---

## 10.5 Task 工具解决什么问题？

LLM 自己在自然语言里写计划有几个问题：

- 容易忘记已经列过的步骤
- 状态不可查询
- 依赖关系不结构化
- 前端不好展示
- 中途压缩后细节可能丢

Task tools 把计划变成结构化状态：创建、查看、列出、更新。模型可以像操作项目管理表一样操作任务。

---

## 10.6 经典问题：什么时候应该让模型用 TaskCreate？

适合：

- 用户任务有多个步骤
- 需要跟踪哪些完成、哪些阻塞
- 工具调用很多，容易迷路
- 希望在压缩后仍保留任务结构

不适合：

- 简单问答
- 一次工具调用就能完成
- 模型已经明确知道下一步，不需要维护列表

---

## 10.7 和 context compression 的关系

Task 列表在 `AgentState.tasks_context`，不是普通 `state.context` 消息。因此上下文压缩不会把 task list 当消息压掉。

但模型要看到 task list，仍然需要通过工具查询或系统提示使用 task tools。结构化状态和模型上下文是两个层面。

---

## 10.8 本章检查题

1. Task 工具为什么需要 `is_state_injected`？
2. Task 状态和普通消息上下文有什么区别？
3. 如果 agent 被压缩上下文，task list 会不会丢？
4. 一个复杂任务什么时候只用 Task，什么时候升级到 Team？
