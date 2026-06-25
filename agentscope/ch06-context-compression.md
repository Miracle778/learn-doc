# 第 6 章 上下文压缩：AgentScope 如何处理长对话？

> 这一章读 `compress_context()`。目标是理解 AgentScope 的短期记忆压缩策略：不是简单截断，而是“旧内容结构化摘要 + 最近内容原文保留 + 工具结果截断/offload”。

---

## 6.1 配套源码

打开：

- `src/agentscope/agent/_agent.py`
  - `compress_context()`
  - `_compress_context_impl()`
  - `_split_context_for_compression()`
  - `_split_tool_result_for_compression()`
  - `_prepare_model_input()`
- `src/agentscope/agent/_config.py`
  - `ContextConfig`
  - `SummarySchema`

---

## 6.2 压缩什么时候触发？

每次 reasoning 前，`_reply_impl()` 会调用：

```python
await self.compress_context()
```

真正判断在 `_compress_context_impl()`：

```text
kwargs = await _prepare_model_input()
estimated_tokens = await model.count_tokens(**kwargs)
threshold = trigger_ratio * model.context_size
if estimated_tokens < threshold:
    return
```

默认 `trigger_ratio=0.8`。意思是：当当前模型输入超过上下文窗口的 80% 时，启动压缩。

为什么不是 100%？因为压缩本身也要把待压缩内容、压缩 prompt、structured output schema 一起送给模型。如果等到 100% 再压，压缩请求自己可能已经放不进上下文。

---

## 6.3 `ContextConfig` 逐项解释

```python
trigger_ratio: float = 0.8
reserve_ratio: float = 0.1
compression_prompt: str = "..."
summary_template: str = "..."
summary_schema: dict = SummarySchema.model_json_schema()
tool_result_limit: int = 50000
```

字段含义：

```text
trigger_ratio      多长时触发压缩
reserve_ratio      压缩后保留多少近期原始上下文
decompression?     没有显式解压，summary 作为后续上下文提示
compression_prompt 告诉模型如何总结旧上下文
summary_schema     要求模型输出结构化摘要字段
summary_template   把结构化字段渲染成最终 summary 文本
tool_result_limit  单个工具结果进入 context 前的 token 上限
```

---

## 6.4 SummarySchema：摘要不是一句话

`SummarySchema` 包含这些字段：

- `task_overview`：用户核心请求和成功标准
- `current_state`：已经完成什么，生成/修改/分析了哪些文件
- `important_discoveries`：发现的约束、决策、错误和解决方式
- `next_steps`：下一步行动、阻塞点、优先级
- `context_to_preserve`：用户偏好、领域信息、承诺

这是很好的 agent 压缩实践：不要让模型“随便总结”，而是强制它输出可恢复工作的结构化字段。

---

## 6.5 压缩流程逐步拆解

### 第一步：准备当前模型输入并估 token

```text
_prepare_model_input()
  → SystemMsg(system prompt)
  → UserMsg(summary) if exists
  → state.context
  → tools schemas
```

压缩判断用的是完整模型输入，不只是 `state.context`。这是对的，因为 tool schemas 和 system prompt 也占 token。

### 第二步：分割旧消息和保留消息

调用：

```python
msgs_to_compress, msgs_to_reserve = await _split_context_for_compression(
    reserve_ratio * model.context_size,
    tools,
)
```

保留的是最近一段消息。更旧的消息进入压缩。

### 第三步：构造压缩请求

压缩请求大概是：

```text
SystemMsg(system prompt)
UserMsg(old summary)        # 如果已经有 summary
msgs_to_compress            # 要压缩的旧上下文
UserMsg(compression_prompt) # 要求生成 continuation summary
```

注意：旧 summary 会参与新 summary 生成。这意味着多次压缩时，summary 会滚动更新，而不是只总结这一次被压缩的片段。

### 第四步：结构化输出

调用：

```python
model.generate_structured_output(
    messages=messages,
    structured_model=cfg.summary_schema,
)
```

模型必须按 schema 输出字段，然后：

```python
self.state.summary = cfg.summary_template.format(**res.content)
```

### 第五步：更新 state

```text
state.summary = 新摘要
state.context = msgs_to_reserve
清理未保留 Read 缓存
如果有 offloader，把被压缩上下文外置，并在 summary 里加入路径提醒
```

---

## 6.6 `_split_context_for_compression()` 的关键细节

这个方法不是简单按消息数量切一半。它做的是 token-aware split：

1. 从最新消息往前数
2. 每次估算“从这个位置到末尾”的 token
3. 一旦超过 reserve token，就找到边界
4. 边界消息内部再按 block 切分
5. 避免保留 tool result 却丢失对应 tool call

为什么要按 block 切？因为一个 assistant message 可能很长，里面包含多个 tool call/result/text。只按 message 切可能粒度太粗。

为什么要避免 tool call/result 拆散？因为模型看到孤立工具结果会困惑：

```text
坏上下文：
  ToolResultBlock(id=abc, output="文件内容...")
  但没有 ToolCallBlock(id=abc, name="Read", input="...")
```

好的上下文应该保留完整因果：谁调用了什么工具，返回了什么。

---

## 6.7 压缩失败怎么办？

如果压缩请求本身超过上下文窗口，源码会设置 `context_overflow=True`。

如果结构化输出失败且是 overflow，AgentScope 会尝试丢掉更旧的待压缩消息，再次压缩：

```text
for i in range(1, len(msgs_to_compress)+1):
    messages = msgs_system + msgs_to_compress[i:] + compression_prompt
    if estimated < context_size * trigger_ratio:
        break
```

这是一种降级策略：宁可少压一些旧内容，也不要整个压缩失败。

---

## 6.8 工具结果截断和上下文压缩的关系

上下文压缩处理的是整体 context 太长；工具结果截断处理的是单个工具结果太长。

两者互补：

```text
单个 Read/Grep/Bash 输出爆炸
  → _split_tool_result_for_compression()

多轮对话累积太长
  → compress_context()
```

很多 agent bug 都来自忽视工具输出。比如 `cat huge.log` 一次返回几 MB，如果不截断，后面压缩也救不回来。

---

## 6.9 经典问题：压缩 summary 会不会污染用户消息？

`summary` 被包装成 `UserMsg(name="user", content=summary)` 放进模型输入。它不是最终用户真实说的话，而是一个系统风格的 `<system-info>` 文本。

这是一种折中：不同模型 API 对 system/developer/context 角色支持不一致，用 user message 放结构化系统信息更兼容；同时文本里用 `<system-info>` 标记提醒模型这是摘要。

---

## 6.10 本章检查题

1. 为什么压缩判断要把 tools schema 也算进 token？
2. `reserve_ratio` 太大时会发生什么？源码如何 fallback？
3. 多次压缩时，旧 summary 如何参与新 summary？
4. 为什么 Read 文件缓存要随 context 压缩清理？
5. 如果你要自定义摘要字段，应该改哪里？
