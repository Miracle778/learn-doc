# 第 7 章 长期记忆：AgentScope 的 Mem0Middleware 怎么做？

> 这一章读 `Mem0Middleware`。目标是区分短期上下文压缩和长期记忆，并理解“框架自动控制”和“模型自主控制”两种 memory 模式。

---

## 7.1 配套源码

打开：

- `src/agentscope/middleware/_longterm_memory/_mem0/_middleware.py`
- `src/agentscope/middleware/_longterm_memory/_mem0/_tools.py`
- `src/agentscope/middleware/_longterm_memory/_mem0/_utils.py`
- `src/agentscope/middleware/_longterm_memory/_mem0/_agentscope_adapter.py`
- `examples/long_term_memory/mem0/README.md`

---

## 7.2 先分清三种“记忆”

AgentScope 里容易混淆三件事：

```text
state.context
  当前 session 的近期原始上下文

state.summary
  当前 session 的旧上下文压缩摘要

Mem0 long-term memory
  跨 session 的长期用户事实/偏好/历史记忆
```

长期记忆不是为了省 token，而是为了跨会话召回。

例如：

```text
context: 用户刚刚说“读 README”
summary: 之前已经读过 agent/_agent.py，发现 ReAct loop 在 _reply_impl
long-term memory: 用户偏好中文讲解，喜欢源码 walkthrough 风格
```

---

## 7.3 Mem0Middleware 的构造方式

`Mem0Middleware` 有两种 backend 构造路径：

1. 传 `chat_model + embedding_model`，middleware 内部构造 OSS `AsyncMemory`
2. 直接传已有 `mem0.AsyncMemory` 或 `AsyncMemoryClient`

源码里 `_resolve_client()` 会做校验：

- 如果传了 `client`，它优先，其他模型配置会被忽略并 warning。
- 如果不传 `client`，需要能构造 mem0 config。
- 必须是 async client；同步 Memory/MemoryClient 不支持。

这是服务化 agent 很重要的一点：memory 后端可能是本地 OSS，也可能是 hosted platform，AgentScope 不把它写死。

---

## 7.4 三种模式：static_control / agent_control / both

### static_control

框架自动控制：

```text
reply 前：根据用户输入 search memory
reasoning 前：把 memories 注入 context
reply 后：把本轮 user/assistant 对话写回 memory
```

agent 不需要知道有 `search_memory` / `add_memory` 工具。

### agent_control

模型自主控制：

```text
middleware 暴露 search_memory / add_memory 工具
system prompt 追加工具使用说明
模型自己决定何时查、何时写
```

### both

两者都开：自动检索写回兜底，模型也有工具主动权。

---

## 7.5 static_control 的源码时间线

`on_reply()` 是关键。

### 第一步：提取 query

```python
inputs = input_kwargs.get("inputs")
query_text = _extract_query_text(inputs)
```

它只从新输入里提取文本。没有 query，就不查 memory。

### 第二步：搜索 mem0

```python
memories = await self._async_search(
    query_text,
    user_id=user_id,
    agent_id=search_agent_id,
)
```

filter 至少包含 `user_id`，如果 `scope_search_by_agent=True` 且有 `agent_id`，还会按 agent 隔离。

### 第三步：等待 ReplyStartEvent 后注入

源码逻辑大概是：

```text
async for item in next_handler(...):
    if not injected and memories and isinstance(item, ReplyStartEvent):
        agent.state.context.append(_build_memory_message(memories))
        injected = True
    yield item
```

为什么等 `ReplyStartEvent`？因为 `_reply_impl()` 会先把用户消息 append 到 context，然后发 `ReplyStartEvent`。此时注入 memory，顺序是：

```text
UserMsg(用户新问题)
AssistantMsg(name="memory", HintBlock(相关长期记忆))
```

这正好让模型在本轮 reasoning 前看到记忆。

### 第四步：reply 后写回

`finally` 里如果有 `query_text` 和 `final_msg`，会把 user/assistant 对话写回 mem0。

`await_write=True` 时同步等待；`False` 时后台 `asyncio.create_task()`。

---

## 7.6 `_build_memory_message()` 为什么用 HintBlock？

检索结果被包装成：

```python
AssistantMsg(
    name="memory",
    content=[HintBlock(hint=content)],
)
```

不是 UserMsg，也不是 SystemMsg。

原因：

- memory 不是用户刚说的话。
- memory 是系统检索到的相关提示。
- user message 不能包含 HintBlock。
- assistant-role 容器可以装 HintBlock，formatter 再把 hint 转成模型可见输入。

---

## 7.7 agent_control 的两个工具

`_tools.py` 构造：

```text
search_memory
add_memory
```

### search_memory

输入：

```json
{
  "keywords": ["短关键词1", "短关键词2"],
  "limit": 5
}
```

实现特点：

- 每个 keyword 独立搜索
- `asyncio.gather()` 并发搜索
- 合并去重
- 没有结果时返回 `(no relevant memories found)`

这是给模型一个明确策略：不要用一大段自然语言乱查，拆成短关键词查。

### add_memory

输入：

```json
{
  "thinking": "为什么值得记",
  "content": ["要持久保存的事实1", "事实2"]
}
```

`thinking` 不写入 mem0，只留在工具结果里用于审计。真正写入的是 `content`。

为什么？因为 memory store 应该保存用户事实，不应该保存“模型为什么认为要保存”的自言自语。

---

## 7.8 写 memory 的 fallback

`_async_add_with_fallback()` 先正常调用 mem0 extraction：

```text
infer=True，让 mem0 判断提取哪些 memory
```

如果 mem0 认为没提取到任何东西，则 fallback：

```text
infer=False，直接保存 raw text
```

这保证 `add_memory` 工具不会“明明调用成功但什么都没存”。

---

## 7.9 经典问题：static_control 会不会让 context 越来越长？

会有这个风险。源码注释也提到：每轮检索到 memory 都会 append 一个 memory note 到 `state.context`。长会话下要依赖 `compress_context()`，或者开发者自己清理。

这正好说明：长期记忆检索结果进入当前 session 后，就变成短期 context 的一部分，也会受上下文压缩管理。

---

## 7.10 本章检查题

1. `static_control` 在哪个 hook 里工作？
2. 为什么 memory 注入要等 `ReplyStartEvent`？
3. `agent_control` 如何让模型知道 memory 工具存在？
4. `search_memory` 为什么支持多个 keywords？
5. `add_memory` 为什么不把 thinking 写进 mem0？
