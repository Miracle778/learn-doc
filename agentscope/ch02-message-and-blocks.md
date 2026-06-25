# 第 2 章 消息系统：为什么要把 message 拆成 block？

> 这一章读 `Msg` 和 `ContentBlock`。目标是理解 AgentScope 为什么不用纯字符串保存上下文，而要把文本、工具调用、工具结果、多模态数据、hint 都拆成结构化块。

---

## 2.1 配套源码

打开：

- `src/agentscope/message/_base.py`
- `src/agentscope/message/_block.py`
- `src/agentscope/event/_event.py`
- `src/agentscope/formatter/_formatter_base.py`

---

## 2.2 `Msg` 是 AgentScope 的信息载体

`Msg` 大概长这样：

```python
class Msg(BaseModel):
    name: str
    content: list[ContentBlock]
    role: Literal["user", "assistant", "system"]
    id: str
    metadata: dict
    created_at: str
    finished_at: str | None
    usage: Usage | None
```

你可以把它理解成一封结构化邮件：

```text
谁发的：name
发给模型时是什么角色：role
正文是什么：content blocks
是哪一条消息：id
什么时候开始/结束：created_at / finished_at
消耗多少 token：usage
```

关键是 `content` 不是字符串，而是 `list[ContentBlock]`。

---

## 2.3 ContentBlock 的六种核心形态

### TextBlock：普通文本

```python
TextBlock(type="text", text="...")
```

用户输入、助手回答、工具文本结果都可能用它。

### ThinkingBlock：模型思考/推理摘要

```python
ThinkingBlock(type="thinking", thinking="...")
```

不同模型供应商对 reasoning 的支持不同。AgentScope 用统一 block 承接供应商返回的思考摘要或 thinking 内容。

### DataBlock：多模态数据

```python
DataBlock(source=Base64Source(...) | URLSource(...))
```

图片、音频、视频等不能硬塞成字符串，所以用 `DataBlock` 描述来源和媒体类型。

### HintBlock：系统注入的提示

```python
HintBlock(hint="...", source="...")
```

这是读 AgentScope 必须重视的 block。它经常用于：

- team message
- inbox 注入
- 长期记忆检索结果
- 后台任务完成通知

它不是用户原始输入，也不是工具结果，而是一种“给模型看的运行时提示”。

### ToolCallBlock：模型要求调用工具

```python
ToolCallBlock(
    id="...",
    name="Read",
    input='{"file_path": "README.md"}',
    state="pending",
)
```

`input` 是 raw JSON string。为什么不直接存 dict？因为模型 streaming 时 JSON 可能是一段段增量生成的，先保存原始字符串更贴近模型输出。

### ToolResultBlock：工具执行结果

```python
ToolResultBlock(
    id=tool_call.id,
    name="Read",
    output=[TextBlock(text="...")],
    state="success",
)
```

`id` 和 `ToolCallBlock.id` 对应。后续压缩、formatter、模型上下文都依赖这对关系。

---

## 2.4 角色约束：为什么 user 不能带 tool_call？

`Msg.validate_role_content()` 对不同 role 做限制：

```text
user    只能包含 text / data
system  只能包含 text
assistant 可以包含 text / thinking / tool_call / tool_result / data / hint
```

这是安全边界。外部用户不能伪造一个 `ToolCallBlock` 让系统以为“模型请求执行工具”。工具调用必须来自 assistant 角色，并经过 schema 校验、权限检查、执行流程。

---

## 2.5 事件如何拼回消息？

`Msg.append_event()` 是服务层非常关键的拼装器。

模型流式输出时，Agent 不会等完整回答结束才给前端，而是不断发事件：

```text
ReplyStartEvent
ModelCallStartEvent
TextBlockStartEvent
TextBlockDeltaEvent
TextBlockDeltaEvent
TextBlockEndEvent
ToolCallDeltaEvent
ToolCallEndEvent
ToolResultStartEvent
ToolResultTextDeltaEvent
ToolResultEndEvent
ModelCallEndEvent
ReplyEndEvent
```

`append_event()` 根据事件类型更新 `Msg.content`：

- `TEXT_BLOCK_START`：追加空 `TextBlock`
- `TEXT_BLOCK_DELTA`：往对应 block 里追加文本
- `TOOL_CALL_DELTA`：累积工具名和 input
- `TOOL_RESULT_*`：追加工具结果
- `MODEL_CALL_END`：累加 usage
- `REPLY_END`：写 finished_at

所以 ChatService 可以边发布事件给前端，边在服务端重建完整 assistant message。

---

## 2.6 经典问题：为什么 tool call 和 tool result 都放在 assistant message 里？

因为从模型视角，工具调用是 assistant 发起的动作，工具结果是这个动作的观察结果。下一轮模型输入需要看到完整轨迹：

```text
assistant: 我要调用 Read({file_path: README.md})
assistant/tool result: README.md 内容是 ...
assistant: 根据内容，我总结如下 ...
```

如果工具结果不进 context，下一轮模型就不知道工具返回了什么。

---

## 2.7 经典问题：HintBlock 和 UserMsg 有什么区别？

Team message 明明也是“别人发来的话”，为什么不用 `UserMsg`？

因为 Team message 不是最终用户通过 UI 直接输入的需求，而是系统内部另一个 agent 发来的上下文提示。它需要：

- 保留来源，比如 `source="researcher"`
- 在前端可以单独渲染为 hint
- 在 formatter 中转换成适合模型理解的消息
- 不破坏用户真实对话历史

所以 AgentScope 用 `HintBlock` 表达“系统注入但模型可见”的信息。

---

## 2.8 本章阅读任务

对着源码回答：

1. `Msg.get_text_content()` 为什么只取 `TextBlock`？
2. `ToolCallBlock.state` 有哪些值？这些值对应工具生命周期的哪一步？
3. `ToolResultBlock.output` 为什么既可以是字符串，也可以是 `TextBlock/DataBlock` 列表？
4. `append_event()` 如何处理 usage？为什么 usage 放在 message 上？
