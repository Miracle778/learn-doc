# 第 8 章 Middleware：如何不改 Agent 核心就扩展能力？

> 这一章读 `MiddlewareBase` 和 Agent 里各个 middleware chain。目标是学会判断：一个能力应该写进 Agent 核心，还是做成 middleware。

---

## 8.1 配套源码

打开：

- `src/agentscope/middleware/_base.py`
- `src/agentscope/agent/_agent.py`
- `src/agentscope/middleware/_longterm_memory/_mem0/_middleware.py`
- `src/agentscope/app/middleware/_inbox_middleware.py`
- `src/agentscope/app/middleware/_tool_offload_middleware.py`
- `src/agentscope/middleware/_tts_middleware.py`

---

## 8.2 MiddlewareBase 提供哪些 hook？

AgentScope 2.0 的 middleware hook：

```text
on_reply             包住整个 reply 过程
on_reasoning         包住一次 reasoning/model phase
on_acting            包住单个工具原始执行
on_model_call        包住底层模型 API 调用
on_compress_context  包住上下文压缩
on_system_prompt     顺序转换 system prompt
list_tools           middleware 可以贡献工具
get_middleware_key   middleware 状态 key
```

这些 hook 覆盖了 agent 生命周期的主要插入点。

---

## 8.3 洋葱模式：on_reply / on_reasoning / on_acting / on_model_call

以 `_reply()` 为例，如果没有 middleware：

```python
async for item in self._reply_impl(inputs=inputs):
    yield item
```

如果有 middleware，会构造一个 chain：

```text
mw0.on_reply(
  next_handler = mw1.on_reply(
    next_handler = _reply_impl
  )
)
```

所以第一个 middleware 是最外层。

一个 logging middleware 可以：

```python
async def on_reasoning(agent, input_kwargs, next_handler):
    print("before")
    async for evt in next_handler(**input_kwargs):
        yield evt
    print("after")
```

这就是洋葱模型。

---

## 8.4 Transformer 模式：on_system_prompt

`on_system_prompt` 不用洋葱模式，而是顺序转换：

```text
prompt0 = base system prompt
prompt1 = mw0.on_system_prompt(prompt0)
prompt2 = mw1.on_system_prompt(prompt1)
prompt3 = mw2.on_system_prompt(prompt2)
```

为什么？因为 system prompt 是一个字符串，最自然的扩展方式就是流水线式修改。

典型用途：

- memory middleware 追加 memory tool instructions
- workspace/offloader 追加“如何读取 offloaded 内容”的说明
- 某些安全 middleware 追加策略提示

---

## 8.5 `is_implemented()`：怎么知道 middleware 实现了哪个 hook？

`MiddlewareBase.is_implemented(hook_name)` 比较 subclass 方法和 base 方法是否相同。

```text
如果子类 override 了 on_reply → True
否则 → False
```

Agent 初始化时会根据这个结果把 middleware 放进对应列表。

这避免运行时每次都 try/except 调不存在的 hook。

---

## 8.6 `on_reasoning` 和 `on_model_call` 的区别

这两个容易混。

`on_reasoning` 包住的是一次 reasoning phase，包括：

- 发 `ModelCallStartEvent`
- 准备 model input
- 调模型
- 转换流式输出事件
- 保存 response 到 context
- 判断是否 final message

`on_model_call` 更底层，只包住真正的模型 API 调用：

```text
messages + tools + tool_choice + current_model → ChatResponse
```

所以：

- 想在模型调用前往 context 注入 HintBlock，用 `on_reasoning`。
- 想替换模型、打模型 API 日志、做 fallback，自定义 `on_model_call`。

---

## 8.7 `on_acting` 的边界

`on_acting` 只包住：

```python
toolkit.call_tool(tool_call, state)
```

不包含：

- tool 是否存在
- JSON schema 校验
- 权限检查
- tool result 写回 context

这个边界是故意的。它让 ToolOffloadMiddleware 可以把“纯工具执行”放到后台，而不会在后台随便改 agent context。

---

## 8.8 middleware 贡献工具：`list_tools()`

Mem0Middleware 就是例子：

- static_control：`list_tools()` 返回空
- agent_control/both：返回 `search_memory` 和 `add_memory`

这说明 middleware 不只是“拦截流程”，也可以给 agent 增加新工具。但工具是否注册进 toolkit，要由上层装配代码显式处理。

---

## 8.9 经典问题：什么时候该写 middleware？

适合 middleware：

- 能力横切多个 agent，不属于某个具体工具
- 需要在生命周期节点前后插逻辑
- 不应该污染 Agent 核心循环
- 可以作为可选插件启用/禁用

例子：

```text
长期记忆       on_reply + on_system_prompt + list_tools
Inbox 注入     on_reasoning
TTS            监听输出文本事件并生成音频
Tool offload   on_acting
Tracing        on_model_call / on_reply
预算控制       on_model_call 或 on_reasoning
```

不适合 middleware：

- `ToolCallBlock` 状态机这种 Agent 核心语义
- `_prepare_model_input()` 这种基础上下文构造
- 权限状态写回这种必须一致的流程

---

## 8.10 本章检查题

1. 为什么第一个 middleware 是最外层？
2. `on_reasoning` 和 `on_model_call` 哪个更适合做 token budget 检查？
3. `InboxMiddleware` 为什么用 `on_reasoning`，而不是 `on_reply`？
4. 为什么 `on_system_prompt` 不需要 `next_handler`？
5. 一个 middleware 如果要保存跨轮状态，应该放到哪里？
