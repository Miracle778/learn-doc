# 第 3 章 AgentState：上下文、摘要、工具缓存和任务列表都放在哪里？

> 这一章读 `AgentState`。你要理解：agent 的“记忆”不是一个东西，而是短期 context、压缩 summary、工具缓存、任务列表、权限上下文、中间件上下文的组合。

---

## 3.1 配套源码

打开：

- `src/agentscope/state/_state.py`
- `src/agentscope/state/_task.py`
- `src/agentscope/agent/_agent.py` 的 `_prepare_model_input()`
- `src/agentscope/app/_service/_chat.py` 中创建 Agent 和保存 state 的部分

---

## 3.2 `AgentState` 字段总览

核心字段：

```python
class AgentState(BaseModel):
    session_id: str
    summary: str | list[TextBlock | DataBlock]
    context: list[Msg]
    reply_id: str
    cur_iter: int
    permission_context: PermissionContext
    tool_context: ToolContext
    tasks_context: TaskContext
    middle_context: dict[str, Any]
```

先给每个字段一个直觉：

```text
session_id          当前会话身份
summary             被压缩掉的旧上下文摘要
context             近期未压缩的原始消息
reply_id            当前 assistant reply 的 id，事件都挂在它下面
cur_iter            当前 ReAct loop 迭代次数
permission_context  权限规则、工作目录、模式
工具 context         Read 缓存、激活的 tool group
tasks_context       TaskCreate/Update/List/Get 管理的 todo 列表
middle_context      middleware 跨轮保存自己的状态
```

---

## 3.3 两层短期上下文：summary + context

AgentScope 的短期上下文不是单独一个 messages 列表，而是两层：

```text
state.summary  = 旧消息压缩后的摘要
state.context  = 近期原始消息
```

模型输入由 `_prepare_model_input()` 生成：

```python
messages = [SystemMsg(...)]
if self.state.summary:
    messages.append(UserMsg(name="user", content=self.state.summary))
messages.extend(self.state.context)
```

这个设计很实用：

- 最近上下文保留原文，避免丢失细节。
- 旧上下文变 summary，避免爆 token。
- summary 作为一个 user message 放进去，让模型把它当作“之前工作摘要”。

---

## 3.4 `reply_id` 为什么在 state 里？

每次新 reply 开始时，`_reply_impl()` 会：

```python
self.state.reply_id = _generate_id()
self.state.cur_iter = 0
```

随后所有事件都带这个 `reply_id`：

- `ReplyStartEvent(reply_id=...)`
- `TextBlockDeltaEvent(reply_id=...)`
- `ToolCallEndEvent(reply_id=...)`
- `ReplyEndEvent(reply_id=...)`

服务层用它把事件追加到正确的 assistant message。对于等待用户确认或外部执行的场景，`reply_id` 还可以让下一次 continuation 接着同一个 reply 继续。

---

## 3.5 `cur_iter`：ReAct loop 的刹车计数器

`cur_iter` 记录当前 reply 已经进行了多少轮 reasoning-acting。

`_reply_impl()` 里有：

```text
while self.state.cur_iter < self.react_config.max_iters:
    ... reasoning / acting ...
    self.state.cur_iter += 1
```

如果模型陷入循环，比如一直读同一个文件、反复调用失败工具，`max_iters` 会让本轮 reply 结束并发 `ExceedMaxItersEvent`。

---

## 3.6 `PermissionContext`：权限是状态的一部分

`permission_context` 不是临时参数，因为权限会随对话变化。

例如模型要执行 Bash，权限系统可能要求用户确认。用户确认后，确认事件里可能带上新的 rule。`Agent._handle_incoming_event()` 会把规则加进 `PermissionEngine`，而 engine 绑定的是 `state.permission_context`。

这意味着：

- 用户批准过的目录或命令，可以在后续继续生效。
- team worker 可以继承 leader 已批准的权限规则。
- session 恢复后，权限状态仍在。

---

## 3.7 `ToolContext`：Read 缓存和 tool group 状态

`ToolContext` 里最值得看的是 Read 文件缓存：

```python
read_file_cache: list[ReadCacheEntry]
```

为什么 agent state 里要放文件缓存？因为 Read/Edit/Write 这类工具经常需要知道文件之前读到的内容，避免重复读取，也能辅助编辑工具判断文件是否变化。

缓存有两个限制：

- `max_cache_files`
- `max_cache_bytes`

并且会通过 mtime 检查缓存是否过期。

上下文压缩后，`_clear_unreserved_read_cache()` 会清理不再出现在保留上下文里的 Read 缓存，避免 state 越来越胖。

---

## 3.8 `TaskContext`：计划不是 prompt，而是结构化状态

`tasks_context` 保存 Task 工具维护的任务列表。

这和“让模型在回答里写 todo list”不一样。结构化 task 有状态、依赖、owner、metadata，可以被工具查询和更新。

所以 AgentScope 有两套“组织复杂任务”的机制：

- 单 agent 内部：Task tools + `state.tasks_context`
- 多 agent 协作：Team tools + message bus + worker sessions

---

## 3.9 `middle_context`：middleware 自己的跨轮记事本

middleware 有时需要跨 reply 保存状态，但又不应该污染 `context`。这时可以用 `middle_context`。

例如一个预算 middleware 可以记录累计费用，一个 tracing middleware 可以记录某些 run metadata。它属于 session state，会被 ChatService 持久化。

---

## 3.10 经典问题：AgentState 和长期记忆有什么区别？

`AgentState` 是 session 状态。它描述“这个会话做到哪里了”。

长期记忆，比如 Mem0，是跨 session 的外部记忆库。它描述“关于用户有哪些长期有用事实”。

对比：

```text
AgentState.context   当前会话近期原文
AgentState.summary   当前会话旧上下文摘要
Mem0 memory          跨会话可检索事实
```

压缩 summary 不应该当成长期记忆；长期记忆也不应该替代当前 session context。

---

## 3.11 本章阅读任务

1. 找到 `ChatService` 在哪里把 `session_record.state` 传给 `Agent`。
2. 找到 `ChatService` 在哪里把更新后的 `agent.state` 写回 storage。
3. 解释 `state.summary` 为什么被包装成 `UserMsg`，而不是 `SystemMsg`。
4. 如果一个 middleware 要保存“本 session 已经提醒过用户一次”，应该放在哪里？
