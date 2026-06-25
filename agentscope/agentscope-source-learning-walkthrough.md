# AgentScope 2.0 源码学习路线（详细合订版）

> 本文件由拆分章节合并生成；建议优先阅读 README.md 中的分章节版本。

---

# 第 0 章 先建立心理模型：AgentScope 2.0 到底在解决什么？

> 这一章不急着读代码。先把 AgentScope 放进一个“真实 agent 产品”的脑图里：用户消息进来后，框架如何让一个 LLM 变成可运行、可恢复、可审计、可协作的 agent。

---

## 0.1 先别把 Agent 理解成“聊天函数”

很多人刚开始写 agent，会下意识写成这样：

```python
answer = llm.chat(messages)
```

这当然能聊天，但它不是完整 agent。真正的 agent 应用至少要回答这些问题：

1. 历史消息存在哪里？下次请求怎么恢复？
2. 模型要调用工具时，工具 schema 从哪里来？
3. 工具执行前要不要权限确认？
4. 工具结果太长怎么办？
5. 对话越来越长时怎么压缩？
6. 用户偏好、长期事实怎么跨会话保存？
7. 多个 agent 怎么互相发消息、被唤醒、并发执行？
8. 前端怎么显示模型流式输出、工具调用、工具结果？
9. 服务重启后，session 状态还能不能继续？

AgentScope 2.0 的源码，就是围绕这些问题建立边界。

---

## 0.2 用一张图建立整体结构

```text
用户 / 前端
  │
  │ HTTP / SSE
  ▼
ChatService.run()
  │
  ├─ 从 Storage 加载 AgentRecord / SessionRecord
  ├─ 从 WorkspaceManager 取 workspace
  ├─ get_toolkit() 组装工具、MCP、skills、team tools、task tools
  ├─ 组装 middlewares：Inbox / StateChange / ToolOffload / TTS / extras
  ├─ 根据 session config 加载 model / fallback model
  ├─ 用持久化 AgentState 创建 Agent
  │
  ▼
Agent.reply_stream()
  │
  ├─ _handle_incoming_messages()：用户消息进入 state.context
  ├─ compress_context()：必要时压缩旧上下文
  ├─ _reasoning_impl()：准备 model input，调模型
  ├─ _execute_tool_call()：校验工具、权限、执行、写回结果
  ├─ ReAct loop：继续 reasoning / acting，直到完成或 max_iters
  │
  ▼
事件流 AgentEvent
  │
  ├─ MessageBus.session_publish_event()：replay log + live publish
  ├─ Msg.append_event()：服务端重建 assistant reply
  └─ Storage.update_session_state()：保存新的 AgentState
```

这张图有两个关键认知：

- `Agent` 是推理-行动循环的核心，但它不负责整个产品的存储、SSE、session lock、多租户。
- `ChatService` 是产品化运行入口，它负责把 agent、工具、模型、中间件、workspace、storage、message bus 组装成一次完整运行。

---

## 0.3 AgentScope 的边界划分

先记住这几个层次，后面读源码会轻松很多。

```text
应用服务层 app/
  ChatService / Storage / MessageBus / WorkspaceManager / Team tools

Agent 执行层 agent/
  Agent / ReAct loop / context compression / model call / tool execution

状态与消息层 state/ message/ event/
  AgentState / Msg / ContentBlock / AgentEvent

扩展层 middleware/
  Memory / Inbox / TTS / ToolOffload / Tracing / Budget

能力层 tool/ model/ workspace/ permission/
  ToolBase / Toolkit / ChatModelBase / PermissionEngine / WorkspaceBase
```

你读代码时不要一上来就陷入模型适配器。模型适配器很多，但主干其实很清晰：

- `Agent` 关心“怎么运行一轮 agent”。
- `AgentState` 关心“运行中要保存什么”。
- `Msg` 和 `AgentEvent` 关心“信息如何结构化流动”。
- `Middleware` 关心“不改 Agent 核心如何插能力”。
- `ChatService` 关心“生产服务如何可靠运行 agent”。

---

## 0.4 一次用户消息的完整时间线

假设用户在前端输入：“帮我读一下 README_zh.md，并总结 AgentScope 的核心能力”。

```text
T0 前端发送消息
  POST /chat 或 session run 入口，构造 UserMsg

T1 ChatService.run()
  加载 agent/session/workspace/model/toolkit/middlewares

T2 创建 Agent
  Agent(state=session_record.state, model=model, toolkit=toolkit, ...)

T3 agent.reply_stream(UserMsg(...))
  用户消息被 deepcopy 后 append 到 state.context

T4 发 ReplyStartEvent
  服务层开始创建空 AssistantMsg，用后续事件逐块拼起来

T5 reasoning 前 compress_context()
  如果 system + summary + context + tools token 超阈值，就压缩旧消息

T6 _prepare_model_input()
  messages = SystemMsg + summary(UserMsg) + state.context
  tools = toolkit.get_tool_schemas(...)

T7 调模型
  模型可能输出文本，也可能输出 ToolCallBlock(name="Read", input="...")

T8 执行工具
  找工具 → JSON 修复 → schema 校验 → 权限系统 → call_tool → ToolResultBlock

T9 工具结果写回 context
  模型下一轮能看到 Read 的文件内容

T10 再次 reasoning
  模型根据工具结果总结

T11 结束
  没有新 tool_call，产生 final AssistantMsg，发 ReplyEndEvent

T12 持久化
  ChatService 保存 reply Msg 和新的 AgentState
```

这就是后面所有章节要拆的主线。

---

## 0.5 和 LangGraph 类框架的区别

如果你之前看过 LangGraph，会习惯把 agent 理解成一张状态图。AgentScope 2.0 不是以显式 graph 作为主抽象，而是以一个统一 `Agent` 类组织 ReAct loop，再通过 middleware 和服务层扩展。

两种思路都能做 agent：

```text
Graph-first 框架：节点和边是主角
AgentScope：Agent loop + structured events + state + middleware 是主角
```

AgentScope 的好处是：你可以非常直接地跟着 `_reply_impl()` 看完整执行路径，代码阅读门槛低；同时服务层又补齐了多 session、team、message bus、workspace、权限等生产问题。

---

## 0.6 本文档的阅读路线

建议顺序：

1. 第 1 章：Agent 如何装配
2. 第 2 章：消息和 block
3. 第 3 章：AgentState
4. 第 4 章：ReAct 主循环
5. 第 5 章：工具和权限
6. 第 6 章：上下文压缩
7. 第 7 章：长期记忆
8. 第 8 章：middleware
9. 第 9 章：多 agent team
10. 第 11 章：服务化运行

第 10、12、13、15 章更像复习和练习，可以穿插看。

---

## 0.7 本章检查题

1. 为什么不能把 agent 简化成一次 `llm.chat(messages)`？
2. `Agent` 和 `ChatService` 分别负责什么？
3. 哪些信息属于 session 短期状态？哪些信息属于长期记忆？
4. 为什么 AgentScope 要设计事件流，而不只是返回最终字符串？

---

# 第 1 章 从 Hello Agent 开始：一个 Agent 是怎么装起来的？

> 这一章读 `Agent.__init__`。目标不是会写示例，而是理解一个 agent 实例内部到底装了哪些“器官”。

---

## 1.1 配套源码

先打开这些文件：

- `README_zh.md`：Hello Agent 示例
- `src/agentscope/agent/_agent.py`：`Agent` 主类
- `src/agentscope/agent/_config.py`：`ModelConfig` / `ContextConfig` / `ReActConfig`
- `src/agentscope/tool/_toolkit.py`：工具集合
- `src/agentscope/middleware/_base.py`：middleware hook 定义
- `src/agentscope/permission/_engine.py`：权限引擎

---

## 1.2 最小 Agent 示例背后的含义

README 里大概是这样：

```python
agent = Agent(
    name="Friday",
    system_prompt="You're a helpful assistant named Friday.",
    model=DashScopeChatModel(...),
    toolkit=Toolkit(tools=[Bash(), Grep(), Glob(), Read(), Write(), Edit()]),
)
```

这不是随便几个参数，它们刚好对应 agent 的基本构成：

```text
name           agent 的身份，用于消息 sender、事件 name、工具上下文
system_prompt  agent 的长期行为约束，每次 model input 都会放到最前面
model          LLM 适配器，负责 generate/count_tokens/structured output
toolkit        agent 的行动能力集合，提供 tool schemas 和执行入口
state          agent 的持久状态，不传就新建 AgentState
middlewares    插入生命周期的扩展能力
offloader      长内容/上下文外置存储入口，常由 workspace 承担
configs        模型重试、上下文压缩、ReAct 最大轮数等运行策略
```

---

## 1.3 精读 `Agent.__init__`

`Agent.__init__` 做的事情可以分成五组。

### 第一组：保存身份和核心依赖

```python
self.name = name
self._system_prompt = system_prompt
self.model = model
self.state = state or AgentState()
```

注意这里的 `state or AgentState()`：

- 单机临时使用时，可以让 Agent 自己创建新状态。
- 服务化场景里，`ChatService` 会从 storage 取出 session 的 `AgentState` 传进来。

这就是“可恢复 agent”的关键。Agent 不是每次从空白开始，而是带着 session state 继续运行。

### 第二组：保存三类配置

```python
self.model_config = model_config
self.context_config = context_config
self.react_config = react_config
```

三类配置分工很清楚：

- `ModelConfig`：模型失败后要不要 fallback，重试几次。
- `ContextConfig`：什么时候压缩上下文，保留多少近期原文，工具结果上限多少。
- `ReActConfig`：一次 reply 里最多 reasoning-acting 几轮，被拒绝工具后是否停止。

不要把这些配置塞进 prompt。它们是运行时策略，应该由代码控制。

### 第三组：权限引擎

```python
self._engine = PermissionEngine(self.state.permission_context)
```

权限系统绑定的是 `state.permission_context`，不是临时变量。这意味着：

- 用户本轮确认过的规则可以写进 state。
- 后续工具调用可以复用这些规则。
- worker agent 也可以从 leader 继承部分权限规则。

### 第四组：工具与 offloader

```python
self.toolkit = toolkit or Toolkit()
self.offloader = offloader
```

`Toolkit` 是工具唯一入口：模型看到的 tool schema 来自它，工具执行也通过它。

`offloader` 用于处理“内容太大但不能彻底丢”的情况，例如：

- 压缩掉的旧 context 可以 offload 到 workspace。
- 超长 tool result 可以截断保留一部分，其余 offload。

### 第五组：middleware 分组

```python
self._reply_middlewares = [... if _.is_implemented("on_reply")]
self._reasoning_middlewares = [... if _.is_implemented("on_reasoning")]
self._acting_middlewares = [... if _.is_implemented("on_acting")]
self._model_call_middlewares = [... if _.is_implemented("on_model_call")]
self._system_prompt_middlewares = [... if _.is_implemented("on_system_prompt")]
self._compress_context_middlewares = [... if _.is_implemented("on_compress_context")]
```

这一段很重要。AgentScope 不是每次执行时都问 middleware“你支持哪个 hook”，而是在初始化时分好组。后面每个生命周期方法只遍历自己相关的 middleware。

---

## 1.4 经典问题：为什么 system prompt 不是固定字符串？

`Agent` 保存的是 `_system_prompt`，真正模型调用前会走 `_get_system_prompt()`。

`_get_system_prompt()` 会动态拼接：

1. 基础 system prompt
2. toolkit 中已激活 skill 的 instructions
3. workspace/offloader instructions
4. `on_system_prompt` middleware 的转换结果

所以最终送给模型的 system prompt 不是初始化传进来的那一小段，而是运行时动态构造的。

这解释了两个设计点：

- skill 启用后，可以自动把使用说明塞进 prompt。
- memory middleware 可以追加“你可以使用 search_memory/add_memory”的说明。

---

## 1.5 经典问题：为什么 Agent 构造函数不直接接 storage？

因为 `Agent` 不负责产品级持久化。它只关心当前 state 对象怎么变化。

持久化在 `ChatService` 做：

```text
Storage 取出 state
  → 创建 Agent(state=state)
  → Agent 修改 state
  → ChatService 把 state 写回 Storage
```

这样 `Agent` 本身保持轻量，既能嵌入脚本，也能被服务层复用。

---

## 1.6 本章阅读任务

请你对着源码回答：

1. `Agent.__init__` 里哪些字段会影响模型输入？
2. 哪些字段会影响工具执行？
3. 哪些字段会影响上下文压缩？
4. 如果你要给 Agent 加一个“每轮模型调用前检查预算”的能力，应该改 Agent 核心，还是写 middleware？为什么？

---

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

---

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

---

# 第 4 章 ReAct 主循环：Agent 是如何“想、做、再想”的？

> 这一章是整套文档的核心。请打开 `src/agentscope/agent/_agent.py`，沿着 `_reply_impl()` 一路读到 `_reasoning_impl()`、`_batch_tool_calls()`、`_execute_tool_call()`。

---

## 4.1 先看主干方法列表

`Agent` 的运行主线：

```text
reply_stream()
  └─ _reply()
      └─ _reply_impl()
          ├─ _check_incoming_event()
          ├─ _handle_incoming_event() 或 _handle_incoming_messages()
          ├─ compress_context()
          ├─ _reasoning()
          │   └─ _reasoning_impl()
          ├─ _batch_tool_calls()
          ├─ _execute_sequential_tool_calls()
          ├─ _execute_concurrent_tool_calls()
          └─ _execute_tool_call()
```

`reply_stream()` 对外只暴露事件流；真正逻辑在 `_reply_impl()`。

---

## 4.2 `_reply_impl()` 第一步：区分新输入和 continuation

`inputs` 可能是：

- `Msg` / `list[Msg]`：用户新消息
- `UserConfirmResultEvent`：用户确认工具调用
- `ExternalExecutionResultEvent`：外部工具执行结果
- `None`：无新用户输入，继续当前状态；常用于 wakeup run

源码先把它分成两类：

```text
event = UserConfirmResultEvent / ExternalExecutionResultEvent / None
msgs  = Msg / list[Msg] / None
```

然后调用 `_check_incoming_event(event)` 判断当前 agent 是否正在等事件。

### 为什么要这么复杂？

因为 agent 可能在上一轮暂停了：

```text
模型请求 Bash
  → 权限系统要求用户确认
  → Agent 发 RequireUserConfirmEvent
  → 本轮 reply 暂停
  → 用户确认后下一次调用继续同一个 reply
```

如果 agent 正在等确认，而你又传了 `None` 或新 user message，状态就会乱。所以 `_check_incoming_event()` 会严格校验。

---

## 4.3 新消息路径：写入 context，开始 reply

如果不是 continuation，就走：

```python
await self._handle_incoming_messages(msgs)
self.state.reply_id = _generate_id()
self.state.cur_iter = 0
yield ReplyStartEvent(...)
```

`_handle_incoming_messages()` 会做输入合法性检查：

- 必须是 `Msg`
- 不能是 system role
- 不能带 tool_call/tool_result/thinking

然后 append 到 `state.context`。

这就是用户消息进入短期上下文的位置。

---

## 4.4 ReAct loop 的主循环

核心结构：

```text
while cur_iter < max_iters:
    action, data = _check_next_action()

    if action == "exit":
        yield final Msg
        return

    if action == "reasoning":
        await compress_context()
        async for evt in _reasoning():
            if evt is final Msg:
                yield ReplyEndEvent
                yield evt
                return
            yield evt

    for batch in await _batch_tool_calls():
        execute batch
        if requires outside interaction:
            yield waiting AssistantMsg
            return

    cur_iter += 1

# 超过 max_iters
Yield ExceedMaxItersEvent + ReplyEndEvent + fallback AssistantMsg
```

你可以把它理解成一个状态机：

```text
检查是否有未执行工具
  ├─ 没有：调模型 reasoning
  └─ 有：执行工具 acting

reasoning 后：
  ├─ 只有文本：结束
  └─ 有 tool_call：下一轮执行工具

acting 后：
  ├─ 工具结果写回 context
  └─ 下一轮 reasoning
```

---

## 4.5 `_reasoning_impl()`：模型调用如何变成事件和消息？

`_reasoning_impl()` 做几件事：

1. 发 `ModelCallStartEvent`
2. 调 `_prepare_model_input()` 准备 messages + tools
3. 调 `_call_model()`
4. 把模型流式返回转换成各种 block events
5. 发 block end events
6. 发 `ModelCallEndEvent`
7. 把完整模型输出保存进 `state.context`
8. 如果没有 tool call，产出 final `AssistantMsg`

关键点：模型响应既要发给前端，又要写进上下文。

```text
ChatResponse chunk
  → _convert_chat_response_to_event() 给前端看
  → completed_response.content 保存到 context
```

---

## 4.6 `_check_next_action()` 的直觉

虽然这份文档不逐行贴 `_check_next_action()`，但你要理解它的角色：它检查上下文最后的 assistant message 里是否还有没处理完的 tool call。

如果：

- 工具调用还在 `pending/allowed/submitted/asking` 某些状态，可能要执行或等待。
- 没有可执行工具调用，就该继续 reasoning。
- 已经有最终 assistant message，可能 exit。

这让 Agent 可以在多次 HTTP 调用之间恢复工具生命周期。

---

## 4.7 工具 batch：为什么不是按 tool_call 顺序一个个执行？

模型一次可能输出多个 tool call。AgentScope 根据工具属性分批：

- 并发安全工具：可以 concurrent
- 不安全工具：sequential

例如：

```text
Grep README + Glob *.py       可以并发
Write file + Edit file        应该串行
add_memory                    不适合和其他写 memory 并发
```

`_execute_concurrent_tool_calls()` 用 `asyncio.gather(return_exceptions=True)`，并且通过 queue 把各个工具的事件流汇总出来。

这个设计避免一个工具失败就取消其他工具，同时还能完整产出每个工具的事件。

---

## 4.8 外部交互：为什么会暂停 reply？

两类事件会让执行暂停：

- `RequireUserConfirmEvent`：需要用户确认工具调用
- `RequireExternalExecutionEvent`：工具要外部系统执行

一旦遇到，Agent 会返回一个 AssistantMsg：

```text
Waiting for tool calls to be confirmed or executed from outside ...
```

这不是最终回答，而是告诉服务层：当前 reply 暂停，等下一次事件继续。

这就是 human-in-the-loop 和 external tool execution 的基础。

---

## 4.9 超过 max_iters 怎么办？

如果循环次数超过 `react_config.max_iters`：

1. 发 `ExceedMaxItersEvent`
2. 记录 warning
3. 发 `ReplyEndEvent`
4. 返回一个 assistant fallback message

为什么还要发 `ReplyEndEvent`？因为前端/SSE 客户端通常等 terminal event。如果没有 `ReplyEndEvent`，UI 可能一直显示“运行中”。

---

## 4.10 本章检查题

1. 新 user message 是在哪一步写入 `state.context` 的？
2. continuation event 是如何恢复等待中的工具调用的？
3. reasoning 前为什么先调用 `compress_context()`？
4. 为什么模型没有 tool call 时，`_reasoning_impl()` 会直接 yield final `AssistantMsg`？
5. 并发工具调用失败时，为什么不取消其他工具？

---

# 第 5 章 工具系统：模型调用工具之前，框架做了哪些保护？

> 这一章读 `_execute_tool_call()`、`Toolkit`、`ToolBase`、`PermissionEngine`。目标是理解：工具调用不是模型说了就执行，中间有 schema、权限、外部确认、结果压缩等完整生命周期。

---

## 5.1 配套源码

打开：

- `src/agentscope/agent/_agent.py`：`_execute_tool_call()` / `_acting()` / `_acting_impl()`
- `src/agentscope/tool/_base.py`
- `src/agentscope/tool/_toolkit.py`
- `src/agentscope/permission/_engine.py`
- `src/agentscope/permission/_decision.py`
- `src/agentscope/message/_block.py`：`ToolCallBlock` / `ToolResultBlock`

---

## 5.2 工具调用生命周期总览

```text
ToolCallBlock(state=pending)
  │
  ▼
检查工具存在且可用
  │
  ▼
_json_loads_with_repair() 修复并解析 input
  │
  ▼
jsonschema.validate() 校验 tool.input_schema
  │
  ▼
PermissionEngine.check_permission()
  │
  ├─ ASK / PASSTHROUGH
  │     └─ state=asking，发 RequireUserConfirmEvent，暂停
  │
  ├─ DENY
  │     └─ 写 ToolResultBlock(state=denied)，返回给模型
  │
  └─ ALLOW
        ├─ state=allowed
        ├─ 如果 external tool：state=submitted，发 RequireExternalExecutionEvent
        └─ 否则 _acting() → toolkit.call_tool() → ToolResultBlock(state=success/error)
```

这条链路是 agent 安全的核心。

---

## 5.3 第一步：检查工具和解析输入

模型输出的工具参数是字符串，而且可能不是完美 JSON。AgentScope 先做修复和 schema 校验：

```text
_json_loads_with_repair(tool_call.input, tool.input_schema)
jsonschema.validate(parsed_input, tool.input_schema)
```

如果失败，不会抛给外部用户，而是通过 `_handle_error_tool_call()` 生成一个工具错误结果写回 context。

这样模型下一轮能看到：

```text
工具调用失败：参数 xxx 不符合 schema
```

然后模型有机会自我修正。

---

## 5.4 第二步：权限系统

权限决策有几类：

```text
ALLOW        允许执行
DENY         拒绝执行，结果写回模型
ASK          需要用户确认
PASSTHROUGH 交给外部确认/处理
```

如果是 `ASK` 或 `PASSTHROUGH`：

- tool call state 变成 `ASKING`
- 发 `RequireUserConfirmEvent`
- 当前 reply 暂停

用户确认后，会通过 `UserConfirmResultEvent` 回来。`_handle_incoming_event()` 会：

- 如果 confirmed：state 改成 `ALLOWED`
- 如果用户修改了 tool input：更新 tool_call.name/input
- 如果用户接受了建议规则：加入 permission engine
- 如果 denied：写一个 denied tool result

---

## 5.5 第三步：执行工具

真正执行在 `_acting_impl()`：

```python
async for chunk in self.toolkit.call_tool(tool_call, self.state):
    yield chunk
```

注意 `_acting()` 只是 raw execution hook，权限和 context 写入不在这里，而在 `_execute_tool_call()` 里。

为什么要这样分层？

- `on_acting` middleware 可以包装纯工具执行，比如 offload 到后台。
- 权限检查、输入校验、context mutation 留在 Agent 主流程里，避免后台任务乱改 state。

源码注释里也提醒：如果工具 `is_state_injected=True`，它拿到的是 live `agent.state`，并发/offload 时要非常谨慎。

---

## 5.6 工具结果如何进入上下文？

工具最终返回 `ToolResponse` 后，AgentScope 构造：

```python
ToolResultBlock(
    id=tool_call.id,
    name=tool_call.name,
    output=...,
    state=chunk.state,
)
```

然后做两件事：

1. `_save_to_context([reserved_tool_result_block])`
2. `_update_tool_call_state(tool_call.id, ToolCallState.FINISHED)`

这样上下文里会有完整轨迹：

```text
assistant message content:
  ToolCallBlock(id=abc, name=Read, input=...)
  ToolResultBlock(id=abc, name=Read, output=...)
```

`id` 对齐非常重要，formatter 和压缩逻辑都依赖它。

---

## 5.7 工具结果太长怎么办？

在写回 context 前，Agent 会调用：

```python
_split_tool_result_for_compression(tool_result_block)
```

如果 token 数没超过 `context_config.tool_result_limit`，原样保留。

如果超过：

1. 找一个边界，只保留前半部分
2. 如果边界 block 是文本，可以按比例截断
3. 剩余部分形成 `offload_tool_result_block`
4. 如果有 offloader，把剩余内容写到外部文件
5. 在保留结果末尾加 `<system-reminder>` 告诉模型结果被截断

这比简单 `output[:50000]` 更好，因为：

- 保留了结构化 block
- 可以 offload 其余内容
- 模型知道自己看到的是截断结果，不会误以为完整

---

## 5.8 经典问题：工具错误为什么也要写回 context？

如果工具参数错了、权限拒绝了、工具不存在了，直接结束会让模型没有修正机会。

把错误作为 `ToolResultBlock(state="error")` 写回 context 后，模型下一轮可能会：

- 修正 JSON 参数
- 换一个工具
- 向用户解释权限被拒绝
- 请求用户提供更多信息

这就是 ReAct 的“观察结果”思想：成功是观察，失败也是观察。

---

## 5.9 经典问题：工具权限为什么不只靠 tool 自己判断？

因为权限是跨工具、跨 session、跨用户确认的统一策略。如果每个工具自己判断，会出现：

- 规则分散，不好审计
- 用户确认结果难以复用
- worker 继承 leader 权限困难
- UI 很难统一展示“为什么要确认”

AgentScope 用 `PermissionEngine + PermissionContext + Tool.check_permissions()` 组合：工具能表达自己的安全语义，engine 统一决策和状态。

---

## 5.10 本章检查题

1. `_execute_tool_call()` 里哪些错误会变成 `ToolResultBlock`？
2. `ASK` 和 `DENY` 的行为差别是什么？
3. external tool 为什么会进入 `SUBMITTED` 状态？
4. 为什么 `_acting_impl()` 不负责写 context？
5. 如果你写一个会修改数据库的工具，应该把 `is_concurrency_safe` 设成什么？为什么？

---

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

---

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

---

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

---

# 第 9 章 多 Agent 调度：Team 不是函数调用，而是 session 间消息协作

> 这一章读 Agent Team。重点理解：AgentScope 的多 agent 不是 leader 直接调用 worker 函数，而是创建 worker agent/session，通过 MessageBus 发 HintBlock，并用 WakeupDispatcher 唤醒 session。

---

## 9.1 配套源码

打开：

- `src/agentscope/app/_tools/_team_create.py`
- `src/agentscope/app/_tools/_agent_create.py`
- `src/agentscope/app/_tools/_team_say.py`
- `src/agentscope/app/_tools/_team_delete.py`
- `src/agentscope/app/_tools/_team_tool_base.py`
- `src/agentscope/app/_service/_toolkit.py`
- `src/agentscope/app/middleware/_inbox_middleware.py`
- `src/agentscope/app/message_bus/_base.py`
- `src/agentscope/app/_manager/_wakeup_dispatcher.py`
- `examples/agent_service/main.py`

---

## 9.2 Team 的核心实体

Team 能力涉及三类记录：

```text
TeamRecord
  team id、leader session_id、member agent ids、name、description

AgentRecord(source="team")
  worker agent 的定义：name、system_prompt、context/react config

SessionRecord
  每个 worker 有自己的 session、state、workspace_id、model config
```

注意：worker 是完整 agent，不是一个函数或线程。它有自己的 `AgentState`，自己的 session lock，自己的 inbox。

---

## 9.3 Toolkit 如何决定 leader/worker 拿到哪些 team tools？

看 `get_toolkit()`：

```text
如果 agent_record.source == "team":
    tools.append(TeamSay(role="worker"))
否则：
    tools += [TeamCreate, AgentCreate, TeamSay(role="leader"), TeamDelete]
```

这就是角色能力控制：

- 用户创建的普通 agent 可以成为 leader，拥有建队和创建 worker 的能力。
- team worker 只能通过 `TeamSay` 汇报或沟通，不能继续创建新 worker。

工具内部还会再次检查运行时状态：比如当前 session 是否已经在 team 里、是否是 leader。

---

## 9.4 `TeamCreate`：创建团队

`TeamCreate.__call__()` 做：

1. 从 storage 读取当前 session
2. 检查 session 是否存在
3. 检查当前 session 是否已经属于 team
4. 创建 `TeamRecord`
5. `upsert_team()` 保存 team
6. `set_session_team_id()` 把当前 session 标为 team leader session
7. 返回 team id

一个 session 只能属于一个 team。源码里明确防止重复创建。

---

## 9.5 `AgentCreate`：创建 worker agent + worker session

这是 Team 最关键的工具。

### 第一步：前置检查

它会检查：

- 当前 session 是否在 team 中
- team 是否存在
- 当前 session 是否是 team leader
- subagent_type 是否存在
- worker name 是否和已有 leader/worker 重名

为什么要检查 name 唯一？因为 `TeamSay(to=...)` 是按成员显示名路由，不是按 agent_id。重名会导致无法确定收件人。

### 第二步：根据模板生成 system prompt

默认模板大概包含：

```text
You are {member_name}, a member of team '{team_name}' led by {leader_name}.
Team purpose: {team_description}
Your role: {member_description}
You communicate ... through TeamSay.
```

这说明 worker 的“团队身份”和“汇报规则”不是靠 leader 每次口头提醒，而是固化进 worker system prompt。

### 第三步：创建 worker AgentRecord

```text
AgentRecord(source="team", data=AgentData(...))
```

`source="team"` 很重要：后续 `get_toolkit()` 会因此只给它 worker-side 工具。

### 第四步：创建 worker SessionRecord

worker session 会继承 leader 的模型配置：

```text
chat_model_config = leader_session.config.chat_model_config
fallback_chat_model_config = leader_session.config.fallback_chat_model_config
workspace_id = leader_session.config.workspace_id
```

权限上下文由模板和 leader 当前权限合并。

### 第五步：把 worker 加进 team.member_ids

```text
team.data.member_ids = [..., worker_agent.id]
upsert_team(team)
```

### 第六步：投递初始任务并唤醒

`AgentCreate` 会把 `prompt` 包成：

```text
HintBlock(
  hint='<team-message from="leader">\n...\n</team-message>',
  source=json.dumps({"label": "team_message", "sublabel": leader_name})
)
```

然后：

```text
message_bus.inbox_push(worker_session.id, hint)
message_bus.enqueue_wakeup(user_id, worker_session.id, worker_agent.id)
```

这一步回答了一个关键问题：worker 为什么创建后会自己开始工作？因为它的 inbox 收到任务，并且 wakeup dispatcher 会启动一次 `ChatService.run(input_msg=None)`。

---

## 9.6 `TeamSay`：成员之间如何发消息？

`TeamSay.__call__()` 做：

1. 读取当前 session，找到 team_id
2. 读取 team
3. 找 leader session 和 leader agent name
4. 遍历 member_ids，建立目录：`name -> (session_id, agent_id)`
5. 根据 `to` 决定单播或广播
6. 构造 `HintBlock(<team-message from="sender">...)</team-message>`
7. 对每个 recipient：`inbox_push + enqueue_wakeup`

所以 TeamSay 的本质是：

```text
把一段 team message 放进对方 session inbox，然后发 wakeup 信号
```

不是直接调用对方 agent。

---

## 9.7 `InboxMiddleware`：消息如何进入 worker 的上下文？

worker 被 wakeup 后，`ChatService.run(input_msg=None)` 创建 Agent 并进入 `reply_stream()`。

在 reasoning 前，`InboxMiddleware.on_reasoning()` 会：

1. `bus.inbox_drain(agent.state.session_id)`
2. 把 payload 反序列化成 `HintBlock`
3. append 到 `agent.state.context`
4. yield `HintBlockEvent` 给前端
5. 调 `next_handler()` 继续 reasoning

如果 context 最后一条是当前 agent 的 assistant message，就把 hint 追加进去；否则新建一个 assistant message 容器。

这就是跨 session 消息进入模型上下文的位置。

---

## 9.8 `WakeupDispatcher`：为什么需要唤醒器？

MessageBus 的 inbox 是队列，消息放进去后，如果目标 session 没有运行，就没人会读。

WakeupDispatcher 做的是：

```text
订阅 wakeup signal
  → dequeue_wakeups()
  → 如果 session 没在运行
  → 检查 session 还存在
  → registry.spawn(ChatService.run(input_msg=None))
```

这让 worker/leader 可以异步协作：一个 agent 发消息，另一个 agent 不需要正在运行，也会被唤醒处理。

---

## 9.9 为什么 ChatService 要跳过“正在等工具确认”的 wakeup？

`ChatService.run()` 里有一个 guard：如果 `input_msg is None`，但 agent 最后一条 assistant message 里有 `ASKING` 或 `SUBMITTED` 的 tool call，就跳过 wakeup。

原因：agent 正在等用户确认或外部执行结果。如果此时用 `None` 强行跑，会触发 `_check_incoming_event()` 报错。

inbox 消息不会丢，等用户确认/外部结果回来后，下一次 reasoning 时 `InboxMiddleware` 会 drain。

---

## 9.10 经典问题：多 agent 是并行的吗？

从系统角度看，可以并行。每个 worker 是独立 session，有自己的 session lock。不同 session 可以被不同进程/任务同时运行。

但同一个 session 内部不会并发运行，因为 `MessageBus.session_run(session_id)` 提供 per-session lock。

```text
leader session   一次只跑一个 run
worker A session 一次只跑一个 run
worker B session 一次只跑一个 run

leader、worker A、worker B 之间可以同时跑
```

---

## 9.11 本章检查题

1. `AgentCreate` 创建了哪些 record？
2. worker 的第一条任务是以什么形式送到 inbox 的？
3. `TeamSay(to=None)` 和 `TeamSay(to="researcher")` 有什么区别？
4. 为什么 worker 不能创建新的 worker？
5. wakeup run 为什么传 `input_msg=None`？
6. 为什么 session lock 仍然必要？

---

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

---

# 第 11 章 服务化运行：ChatService 如何把 Agent 变成生产服务？

> 这一章读 `ChatService.run()`。目标是理解：生产里的 agent 不是直接 `agent.reply()`，而是要加载状态、装配依赖、加 session lock、发布事件、保存结果。

---

## 11.1 配套源码

打开：

- `src/agentscope/app/_service/_chat.py`
- `src/agentscope/app/_service/_toolkit.py`
- `src/agentscope/app/message_bus/_base.py`
- `src/agentscope/app/storage/_base.py`
- `src/agentscope/app/workspace_manager/_base.py`
- `src/agentscope/app/_router/_chat.py`
- `examples/agent_service/main.py`

---

## 11.2 ChatService 的职责

`ChatService` 是 HTTP chat endpoint 和 wakeup dispatcher 共用的运行入口。

它的职责不是“写业务 prompt”，而是保证每次 agent run 都：

- 加载正确的 agent/session/workspace/model
- 装配完整工具和 middleware
- 在分布式 session lock 内运行
- 把事件发布给前端
- 重建并保存 assistant message
- 保存更新后的 AgentState
- 捕获异常，避免后台触发器崩掉

---

## 11.3 `run()` 和 `_run_impl()` 的分工

`run()` 是外层保护：

```python
try:
    await self._run_impl(...)
except Exception:
    logger.exception(...)
```

为什么吞掉异常？因为 `ChatService.run()` 可能由 HTTP 请求、wakeup dispatcher、后台任务完成、定时器触发。某次 run 失败不应该把整个 dispatcher 或服务进程打崩。

真正逻辑在 `_run_impl()`。

---

## 11.4 第一步：加载记录和 workspace

`_run_impl()` 先加载：

```text
agent_record = storage.get_agent(user_id, agent_id)
session_record = storage.get_session(user_id, agent_id, session_id)
workspace = workspace_manager.get_workspace(...)
```

如果缺 agent/session，直接 404。

然后把 workspace workdir 加入 `session_record.state.permission_context.working_directories`。

这一步很关键：工具权限和工作目录不是全局固定的，而是每个 session 的 workspace 注入到 permission context。

---

## 11.5 第二步：组装 Toolkit

调用：

```python
toolkit = await get_toolkit(...)
```

`get_toolkit()` 聚合：

1. workspace built-in tools：Bash / Read / Write / Edit / Grep / Glob 等
2. Task tools：TaskCreate/List/Get/Update
3. Background task control：ToolStop
4. Schedule tools：如果 session 有 model config
5. Team tools：leader/worker 不同
6. extra tools：用户自定义 factory
7. workspace skills 和 MCPs

所以 Agent 看到的工具不是硬编码在 Agent 类里，而是每次 run 根据 session/workspace/app 配置动态组装。

---

## 11.6 第三步：组装 Middlewares

框架默认 middlewares：

```text
InboxMiddleware
StateChangeMiddleware
ToolOffloadMiddleware
TTSMiddleware（如果 session 配了 TTS）
extra_agent_middlewares（用户扩展）
```

顺序很重要：framework-supplied first，然后 extras。

`InboxMiddleware` 让 team message / background result 能在 reasoning 前进入 context。

`StateChangeMiddleware` 用于把状态变化通知出去。

`ToolOffloadMiddleware` 让长时间工具可以后台运行并通过 inbox/wakeup 回传结果。

---

## 11.7 第四步：加载模型和 fallback

从 `session_record.config.chat_model_config` 加载主模型。

如果有 fallback config，也加载 fallback model，然后传给：

```python
ModelConfig(fallback_model=fallback_model)
```

`Agent._call_model()` 会基于 `model_config` 处理主模型失败后的 fallback。

---

## 11.8 第五步：创建 Agent

核心代码形态：

```python
agent_state = session_record.state
agent_state.session_id = session_id
agent = Agent(
    name=agent_record.data.name,
    system_prompt=agent_record.data.system_prompt,
    model=model,
    toolkit=toolkit,
    model_config=ModelConfig(fallback_model=fallback_model),
    context_config=agent_record.data.context_config,
    react_config=agent_record.data.react_config,
    state=agent_state,
    middlewares=middlewares,
    offloader=workspace,
)
```

注意：

- context/react config 来自 `AgentRecord.data`，说明 agent 定义控制行为策略。
- state 来自 `SessionRecord.state`，说明每个 session 有自己的运行历史。
- offloader 是 workspace，说明超长内容可以落到 workspace。

---

## 11.9 第六步：wakeup guard

如果 `input_msg is None`，说明这是 wakeup run。此时如果 agent 正在等待工具确认或外部执行结果，就跳过。

为什么？因为 wakeup 只是为了处理 inbox；如果 agent 当前卡在工具确认，必须等确认事件，而不能用 `None` 继续。

这保护了 agent 状态机不被 team message 或后台唤醒打断。

---

## 11.10 第七步：session lock 内运行

核心：

```python
async with self._message_bus.session_run(session_id):
    async for event in agent.reply_stream(...):
        await session_publish_event(session_id, event)
        reply_msg.append_event(event)

    storage.upsert_message(reply_msg)
    storage.update_session_state(agent.state)
```

为什么必须在 lock 内保存 state？源码注释说得很清楚：如果先释放 lock，再保存 state，另一个进程可能拿到 lock 并读取旧 state，造成状态覆盖或丢失。

---

## 11.11 事件发布：replay log + live pub/sub

`MessageBus.session_publish_event()` 做：

```text
log_append(session_events_key, event)
publish(session_events_key, event + _entry_id)
```

为什么两份？

- replay log：客户端晚连接还能补当前 run 的事件
- live publish：在线客户端实时显示

run 结束后，`session_run.__aexit__()` 会 trim replay log，因为完整 message 已经持久化到 storage。

---

## 11.12 新消息和 continuation 的保存差异

Case A：新 user message / input_msg=None

- 如果有 user msg，先保存 user msg 到 storage
- agent 产生事件
- 根据 ReplyStartEvent 创建空 assistant Msg
- append_event 重建 reply
- 保存 reply 和 state

Case B：UserConfirmResultEvent / ExternalExecutionResultEvent

- 从 storage 取已有 reply msg
- 先把 continuation event append 到 reply msg
- agent 继续产生事件
- append_event 到同一个 reply msg
- 保存更新后的 reply 和 state

这保证工具确认前后的内容属于同一个 assistant reply。

---

## 11.13 本章检查题

1. 为什么 ChatService 要同时被 HTTP endpoint 和 WakeupDispatcher 复用？
2. AgentRecord 和 SessionRecord 分别承担什么？
3. 为什么 workspace workdir 要加入 permission_context？
4. session lock 保护了哪些写操作？
5. 为什么 replay log 在 run 结束后可以 trim？
6. continuation event 为什么要更新已有 reply message？

---

# 第 12 章 建议学习顺序：从浅到深怎么读？

> 这一章是学习计划。它把前面内容拆成 8 个阶段，每个阶段都有目标、源码文件、必须回答的问题和小练习。

---

## 阶段 A：跑通最小 Agent

目标：知道 Agent 最小构成。

读：

- `README_zh.md` Hello Agent
- `src/agentscope/agent/_agent.py`：`__init__`、`reply_stream()`、`reply()`
- `src/agentscope/model/_base.py`
- `src/agentscope/tool/_base.py`

必须回答：

1. `Agent` 不传 `state` 时会发生什么？
2. `reply()` 和 `reply_stream()` 的返回有什么不同？
3. `toolkit` 为空时，agent 还能做什么？

练习：写一个只带模型、无工具的 agent，再写一个带 Read/Grep 的 agent，对比模型输入中的 tool schemas。

---

## 阶段 B：理解消息和事件

目标：知道前端流式输出和服务端持久化如何统一。

读：

- `src/agentscope/message/_base.py`
- `src/agentscope/message/_block.py`
- `src/agentscope/event/_event.py`

必须回答：

1. 为什么 `Msg.content` 是 block list？
2. `append_event()` 如何重建 assistant message？
3. tool call streaming 为什么需要 delta event？

练习：手动构造一个空 AssistantMsg，然后模拟 `TextBlockStart/Delta/End` 事件调用 `append_event()`。

---

## 阶段 C：理解 AgentState

目标：知道 agent 运行中哪些东西会持久化。

读：

- `src/agentscope/state/_state.py`
- `src/agentscope/state/_task.py`
- `src/agentscope/app/_service/_chat.py` 保存 state 的部分

必须回答：

1. `summary` 和 `context` 的关系是什么？
2. `permission_context` 为什么要保存在 state？
3. `tool_context.read_file_cache` 什么时候清理？

练习：画一张 AgentState 字段图，把每个字段标注“模型可见 / 工具可见 / 服务层可见”。

---

## 阶段 D：精读 ReAct loop

目标：能从用户输入追踪到最终回答。

读：

- `_reply_impl()`
- `_reasoning_impl()`
- `_batch_tool_calls()`
- `_execute_tool_call()`

必须回答：

1. 新消息路径和 continuation 路径如何分开？
2. `ReplyStartEvent` 在什么时候发？
3. 工具调用执行后为什么还要继续 reasoning？
4. max_iters 如何结束循环？

练习：给一段假想上下文，判断下一步 action 是 reasoning 还是 acting。

---

## 阶段 E：精读工具和权限

目标：理解工具生命周期和 human-in-the-loop。

读：

- `src/agentscope/tool/_base.py`
- `src/agentscope/tool/_toolkit.py`
- `src/agentscope/permission/_engine.py`
- `_execute_tool_call()`

必须回答：

1. 工具 input 是在哪里做 JSON repair 和 schema validation 的？
2. `ASK` 和 `DENY` 对 context 的影响有什么不同？
3. external tool 的结果如何回来？

练习：设计一个危险工具和一个只读工具，写出它们应该如何设置 `is_read_only`、`is_concurrency_safe`、权限行为。

---

## 阶段 F：精读上下文压缩

目标：知道长任务如何不断续航。

读：

- `ContextConfig`
- `SummarySchema`
- `compress_context()`
- `_split_context_for_compression()`
- `_split_tool_result_for_compression()`

必须回答：

1. 为什么 trigger_ratio 默认小于 0.9？
2. 压缩时为什么保留最近原文？
3. tool call/result 为什么不能拆散？
4. 工具结果截断和 context compression 有什么区别？

练习：假设 context 有 10 条消息，第 7 条是超长工具结果，描述压缩和工具结果截断分别会怎么处理。

---

## 阶段 G：精读 middleware 和长期记忆

目标：学会扩展 agent 行为。

读：

- `src/agentscope/middleware/_base.py`
- `src/agentscope/middleware/_longterm_memory/_mem0/_middleware.py`
- `src/agentscope/middleware/_longterm_memory/_mem0/_tools.py`

必须回答：

1. `on_reply` 适合做什么？
2. `on_reasoning` 和 `on_model_call` 差别是什么？
3. Mem0 static_control 为什么要等 ReplyStartEvent？
4. `add_memory` 为什么要 fallback 到 `infer=False`？

练习：设计一个“用户偏好提醒 middleware”：每轮 reply 前检索偏好，reasoning 前注入 HintBlock。

---

## 阶段 H：精读多 agent 和服务化

目标：理解生产级 agent 如何多 session 协作。

读：

- `src/agentscope/app/_service/_chat.py`
- `src/agentscope/app/_service/_toolkit.py`
- `src/agentscope/app/_tools/_agent_create.py`
- `src/agentscope/app/_tools/_team_say.py`
- `src/agentscope/app/middleware/_inbox_middleware.py`
- `src/agentscope/app/_manager/_wakeup_dispatcher.py`
- `src/agentscope/app/message_bus/_base.py`

必须回答：

1. worker 是如何被创建并唤醒的？
2. TeamSay 如何把消息送到目标 session？
3. ChatService 为什么要用 session lock？
4. replay log 和 live publish 分别服务什么场景？

练习：画一张 leader 创建两个 worker 并收集结果的时序图。

---

# 第 13 章 经典剖析题：带着问题回源码

> 这一章用面试题/自测题的方式，把前面章节串起来。每个问题都给出源码路径和回答框架。

---

## 问题 1：用户发一条消息，到模型输入前经历了什么？

源码路径：

- `ChatService._run_impl()`
- `Agent._reply_impl()`
- `Agent._handle_incoming_messages()`
- `Agent.compress_context()`
- `Agent._prepare_model_input()`

答案框架：

```text
ChatService 加载 agent/session/workspace/model/toolkit/middleware
  → 创建 Agent(state=session_record.state)
  → agent.reply_stream(UserMsg)
  → _handle_incoming_messages 校验并写入 state.context
  → 设置 reply_id 和 cur_iter
  → ReplyStartEvent
  → compress_context 判断是否压缩
  → _prepare_model_input 生成 SystemMsg + summary + context + tools
```

关键点：模型输入不是用户消息本身，而是 system prompt、summary、context、tools 的组合。

---

## 问题 2：模型调用工具时，AgentScope 如何保证安全？

源码路径：

- `Agent._execute_tool_call()`
- `Toolkit.check_tool_available()`
- `_json_loads_with_repair()`
- `PermissionEngine.check_permission()`
- `RequireUserConfirmEvent`

答案框架：

```text
ToolCallBlock
  → 工具存在性检查
  → JSON repair
  → schema validation
  → permission decision
      DENY: 写 denied tool result
      ASK: 发 confirmation event 并暂停
      ALLOW: 执行
      external: 发 external execution event 并暂停
  → tool result 写回 context
```

关键点：失败、拒绝、等待确认都进入同一套工具生命周期。

---

## 问题 3：上下文太长时如何压缩？

源码路径：

- `ContextConfig`
- `Agent._compress_context_impl()`
- `Agent._split_context_for_compression()`
- `SummarySchema`

答案框架：

```text
count tokens(system + summary + context + tools)
  → 超过 trigger_ratio * context_size
  → split context：旧消息压缩，最近消息保留
  → 构造 compression messages
  → generate_structured_output(summary_schema)
  → summary_template 渲染 state.summary
  → state.context = msgs_to_reserve
```

关键点：不是截断，而是结构化摘要和近期原文结合。

---

## 问题 4：长期记忆如何进入当前对话？

源码路径：

- `Mem0Middleware.on_reply()`
- `_extract_query_text()`
- `_async_search()`
- `_build_memory_message()`

答案框架：

```text
on_reply 提取输入 query
  → mem0.search(filters=user_id/agent_id)
  → next_handler 进入 Agent._reply_impl
  → 等 ReplyStartEvent
  → append AssistantMsg(name="memory", HintBlock(...)) 到 state.context
  → agent reasoning 时看到 memory hint
  → reply 后把 user/assistant 对话写回 mem0
```

关键点：长期记忆检索结果进入 context 后，也会被短期上下文机制管理。

---

## 问题 5：Team worker 创建后为什么会自己开始工作？

源码路径：

- `TeamCreate.__call__()`
- `AgentCreate.__call__()`
- `MessageBus.enqueue_wakeup()`
- `WakeupDispatcher._drain_and_dispatch()`
- `InboxMiddleware.on_reasoning()`

答案框架：

```text
leader 调 TeamCreate 创建 team
  → leader 调 AgentCreate 创建 worker AgentRecord + SessionRecord
  → 初始任务封装成 HintBlock 写入 worker inbox
  → enqueue_wakeup
  → WakeupDispatcher 收到 wakeup，spawn ChatService.run(input_msg=None)
  → InboxMiddleware drain worker inbox
  → HintBlock 注入 context
  → worker reasoning 并执行任务
```

关键点：worker 是独立 session，被消息总线唤醒，不是 leader 函数调用。

---

## 问题 6：为什么 ChatService 要在 session lock 里保存 state？

源码路径：

- `ChatService._run_impl()`
- `MessageBus.session_run()`

答案框架：

```text
async with message_bus.session_run(session_id):
    run agent
    publish events
    upsert reply message
    update session state
```

如果先释放 lock 再保存 state，另一个进程可能立刻获得 lock，读到旧 state，然后两个 run 的更新互相覆盖。

关键点：锁保护的不只是 agent run，也保护 run 结束时的状态落盘。

---

## 问题 7：为什么 TeamSay 用 HintBlock，而不是 UserMsg？

源码路径：

- `TeamSay.__call__()`
- `InboxMiddleware.on_reasoning()`
- `message/_block.py` 的 `HintBlock`

答案框架：

Team message 是系统内部另一个 agent 的协作消息，不是最终用户输入。它需要保留 sender/source，也需要前端能渲染为 team hint，所以用 `HintBlock`。

`InboxMiddleware` 在 reasoning 前把 HintBlock 注入 context，让模型看到。

---

## 问题 8：为什么工具结果和工具调用不能在压缩时拆散？

源码路径：

- `Agent._split_context_for_compression()`

答案框架：

模型需要看到工具调用的因果关系：

```text
ToolCallBlock(id=abc, name=Read, input=...)
ToolResultBlock(id=abc, output=...)
```

如果只保留 result，不保留 call，模型不知道结果来自哪里；如果只保留 call，不保留 result，模型不知道观察结果。压缩边界要尽量维护这对关系。

---

## 问题 9：AgentScope 为什么既有 Task tools，又有 Team tools？

源码路径：

- `tool/_task/*`
- `app/_tools/_team_*`
- `state/_state.py`
- `app/storage/_model/_team.py`

答案框架：

Task tools 解决单 agent 内部计划管理；Team tools 解决多 agent/session 异步协作。

```text
Task: AgentState.tasks_context
Team: TeamRecord + worker AgentRecord/SessionRecord + MessageBus
```

---

## 问题 10：如果我要加一个“预算控制”能力，应该放哪里？

推荐：middleware。

可能 hook：

- `on_model_call`：统计/限制模型调用 token 或费用
- `on_reasoning`：在 reasoning 前检查预算，必要时注入提醒或中断
- `middle_context`：保存累计预算状态

不建议直接改 `_reply_impl()`，除非预算控制已经变成 Agent 核心语义。

---

# 第 14 章 和官方资料 / GitHub 对照

> 这一章把源码学习路线和官方材料对应起来。官方文档用于校准概念，源码用于理解实现细节。

---

## 14.1 README 里的 AgentScope 2.0 定位

README 强调 AgentScope 2.0 是面向生产的智能体框架，核心能力包括：

- 事件系统：对应 `message/`、`event/`、`Msg.append_event()`、SSE replay/publish
- 权限系统：对应 `permission/`、`ToolCallBlock.state`、`RequireUserConfirmEvent`
- 多租户多会话服务：对应 `app/storage/`、`ChatService`、`SessionRecord`
- Workspace / sandbox：对应 `workspace/`、工作目录权限、tool offload
- Middleware：对应 `MiddlewareBase` 和各种 hook
- Agent Team：对应 `TeamCreate`、`AgentCreate`、`TeamSay`、`MessageBus`、`WakeupDispatcher`
- Mem0 长期记忆：对应 `middleware/_longterm_memory/_mem0/`

---

## 14.2 官方 Message/Event 文档怎么对源码

官方讲 message/event 时，你应该回到这些源码：

- `src/agentscope/message/_base.py`
- `src/agentscope/message/_block.py`
- `src/agentscope/event/_event.py`
- `src/agentscope/app/_service/_chat.py`

重点看：

```text
AgentEvent 流式产生
  → MessageBus 发布给前端
  → Msg.append_event 重建完整 assistant message
  → Storage 持久化 message
```

---

## 14.3 官方 Middleware 文档怎么对源码

官方讲 middleware hook 时，对应：

- `src/agentscope/middleware/_base.py`
- `src/agentscope/agent/_agent.py` 中 `_reply()`、`_reasoning()`、`_acting()`、`_call_model()`、`compress_context()`

学习重点：

- 哪些 hook 是 async generator
- 哪些 hook 是 awaitable 返回
- `on_system_prompt` 为什么是 transformer pipeline
- middleware 如何提供 tools

---

## 14.4 官方 Agent Team 文档怎么对源码

官方文档会讲“leader 创建 worker，worker 汇报”。源码里要看：

```text
TeamCreate    建 team
AgentCreate   建 worker agent/session，投递初始任务，唤醒
TeamSay       路由 team message 到 inbox，唤醒目标 session
InboxMiddleware  drain inbox，把 HintBlock 注入 context
WakeupDispatcher  收到 wakeup 后运行 ChatService.run(None)
```

这套实现的关键词是：session 间消息协作。

---

## 14.5 官方 Context / Memory 概念怎么对源码

短期上下文：

- `AgentState.context`
- `AgentState.summary`
- `ContextConfig`
- `compress_context()`

长期记忆：

- `Mem0Middleware`
- `search_memory` / `add_memory`
- mem0 filters：`user_id` / `agent_id`

要特别注意：官方文档里“memory”可能泛指记忆能力，但源码层面要分清 context compression 和 long-term memory。

---

## 14.6 参考链接

- AgentScope GitHub: https://github.com/agentscope-ai/agentscope
- AgentScope Docs: https://docs.agentscope.io/
- Message and Event: https://docs.agentscope.io/v2/building-blocks/message-and-event
- Middleware: https://docs.agentscope.io/v2/building-blocks/middleware
- Agent Team: https://docs.agentscope.io/zh/v2/deploy/agent-team
- AgentScope paper: https://arxiv.org/abs/2402.14034

---

## 14.7 使用官方文档的正确姿势

建议顺序：

1. 先读官方文档建立概念词汇。
2. 回到本文档对应章节读源码流程。
3. 再回官方文档看 API 使用方式。
4. 最后自己写一个最小 demo 验证。

不要只读官方 API 示例。Agent 应用开发真正难的是状态边界和生命周期，而这些必须对照源码才能学扎实。

---

# 第 15 章 最后给你一张学习地图

> 这一章把整套源码学习压成一张地图。你可以把它当成复习清单，也可以当成之后读其他 agent 框架的对照表。

---

## 15.1 第一层：能跑起来

关键词：`Agent`、`ChatModelBase`、`Toolkit`、`UserMsg`

你应该能做到：

- 创建一个 agent
- 给它一个模型
- 注册几个工具
- 调用 `reply_stream()`
- 打印事件流

对应源码：

- `README_zh.md`
- `src/agentscope/agent/_agent.py`
- `src/agentscope/model/_base.py`
- `src/agentscope/tool/_toolkit.py`

---

## 15.2 第二层：懂消息结构

关键词：`Msg`、`TextBlock`、`ToolCallBlock`、`ToolResultBlock`、`HintBlock`、`AgentEvent`

你应该能解释：

- 为什么 message 不是 string
- tool call 如何流式累积
- tool result 如何回到上下文
- 前端事件如何重建 reply

对应源码：

- `src/agentscope/message/_base.py`
- `src/agentscope/message/_block.py`
- `src/agentscope/event/_event.py`

---

## 15.3 第三层：懂状态

关键词：`AgentState`、`summary`、`context`、`permission_context`、`tool_context`、`tasks_context`

你应该能解释：

- session 恢复靠什么
- context 和 summary 如何一起送给模型
- 权限为什么是状态
- Read 缓存为什么在 ToolContext

对应源码：

- `src/agentscope/state/_state.py`
- `src/agentscope/app/_service/_chat.py`

---

## 15.4 第四层：懂 ReAct loop

关键词：`_reply_impl`、`_reasoning_impl`、`_execute_tool_call`、`max_iters`

你应该能从任意一条用户消息追踪到：

```text
输入 → context → model input → tool call → permission → tool result → final answer
```

对应源码：

- `src/agentscope/agent/_agent.py`

---

## 15.5 第五层：懂上下文续航

关键词：`ContextConfig`、`SummarySchema`、`compress_context`、`offloader`

你应该能解释：

- 什么时候压缩
- 压缩哪些，保留哪些
- summary 长什么样
- 工具结果太长怎么处理

对应源码：

- `src/agentscope/agent/_config.py`
- `src/agentscope/agent/_agent.py`

---

## 15.6 第六层：懂长期记忆

关键词：`Mem0Middleware`、`static_control`、`agent_control`、`search_memory`、`add_memory`

你应该能解释：

- 长期记忆和上下文压缩的区别
- 自动检索如何注入 context
- memory tools 如何让模型主动控制记忆
- user_id/agent_id 如何隔离 memory

对应源码：

- `src/agentscope/middleware/_longterm_memory/_mem0/`

---

## 15.7 第七层：懂多 agent 协作

关键词：`TeamCreate`、`AgentCreate`、`TeamSay`、`InboxMiddleware`、`WakeupDispatcher`

你应该能画出：

```text
leader 创建 team
leader 创建 worker
worker inbox 收任务
wakeup dispatcher 启动 worker run
worker TeamSay 汇报 leader
leader inbox 收结果
leader 综合回答用户
```

对应源码：

- `src/agentscope/app/_tools/_team_*`
- `src/agentscope/app/middleware/_inbox_middleware.py`
- `src/agentscope/app/_manager/_wakeup_dispatcher.py`

---

## 15.8 第八层：懂生产化

关键词：`ChatService`、`StorageBase`、`MessageBus`、`session_run`、`session_publish_event`

你应该能解释：

- agent/session/model/workspace 如何装配
- 为什么同一 session 要加分布式锁
- event replay 和 live publish 的区别
- 为什么 state 保存必须在 lock 内

对应源码：

- `src/agentscope/app/_service/_chat.py`
- `src/agentscope/app/storage/_base.py`
- `src/agentscope/app/message_bus/_base.py`

---

## 15.9 一句话总结

AgentScope 2.0 的源码最值得学的不是某个 API，而是这些边界：

```text
结构化消息
  + 持久 AgentState
  + ReAct reasoning/acting loop
  + 工具 schema/权限/结果回写
  + summary/context 双层短期上下文
  + middleware 生命周期扩展
  + Mem0 长期记忆
  + MessageBus 驱动的多 agent session 协作
  + ChatService 生产化运行与持久化
```

这些边界学会了，再看 LangGraph、AutoGen、CrewAI 或自己写 agent runtime，都会更有底气。

---

## 15.10 最终自测

不看文档，试着回答：

1. 一条用户消息如何进入模型输入？
2. 一个 tool call 从生成到 finished 有哪些状态？
3. context compression 如何避免丢失最近细节？
4. Mem0Middleware 的三种 mode 分别适合什么？
5. leader 如何创建 worker 并让它自动开始任务？
6. ChatService 为什么需要 message bus？
7. 如果你要加一个新能力，什么时候写 tool，什么时候写 middleware，什么时候改 Agent 核心？

能答清楚这些，你就不是“会用 AgentScope”，而是真的开始理解 agent 应用框架了。

---

# 第 16 章 安全模型：AgentScope 的安全边界在哪里？

> 这一章专门提炼安全相关功能。AgentScope 的安全不是一个单独的 `security.py`，而是分布在消息校验、工具权限、workspace 隔离、人机确认、session lock、team tool gating、长期记忆隔离等多个层面。

---

## 16.1 先回答：安全相关内容原来分散在哪些章节？

如果你想快速回看，安全相关内容主要在这些章节：

- 第 2 章：消息角色校验，防止用户伪造 tool call / tool result
- 第 3 章：`PermissionContext` 是 session state 的一部分，权限规则可持久化
- 第 4 章：continuation event 校验，避免错误恢复等待中的工具调用
- 第 5 章：工具 schema 校验、权限决策、人机确认、工具结果写回
- 第 6 章：工具结果截断和上下文压缩，避免超长输出污染上下文
- 第 8 章：middleware 边界，避免扩展能力直接破坏 Agent 核心状态机
- 第 9 章：team leader/worker 能力隔离，跨 session inbox/wakeup 协作
- 第 11 章：`session_run` 分布式锁，避免并发 run 写坏同一 session state

这一章把它们串成一个完整安全模型。

---

## 16.2 配套源码地图

安全相关源码主要看：

- `src/agentscope/permission/_types.py`
- `src/agentscope/permission/_context.py`
- `src/agentscope/permission/_rule.py`
- `src/agentscope/permission/_decision.py`
- `src/agentscope/permission/_engine.py`
- `src/agentscope/tool/_base.py`
- `src/agentscope/tool/_builtin/_bash.py`
- `src/agentscope/tool/_builtin/_read.py`
- `src/agentscope/tool/_builtin/_write.py`
- `src/agentscope/tool/_builtin/_edit.py`
- `src/agentscope/message/_base.py`
- `src/agentscope/message/_block.py`
- `src/agentscope/agent/_agent.py`
- `src/agentscope/app/_service/_chat.py`
- `src/agentscope/app/_service/_toolkit.py`
- `src/agentscope/app/_tools/_agent_create.py`
- `src/agentscope/app/_tools/_team_say.py`
- `src/agentscope/app/message_bus/_base.py`
- `src/agentscope/workspace/`

---

## 16.3 安全边界总图

```text
外部用户输入
  │
  ├─ Msg role/content 校验
  │    └─ user 不能携带 tool_call / tool_result / thinking
  │
  ▼
Agent ReAct loop
  │
  ├─ continuation event 校验
  │    └─ 只有 agent 正在等待确认/外部结果时才能恢复
  │
  ├─ tool call 输入校验
  │    ├─ 工具是否存在
  │    ├─ JSON repair
  │    └─ jsonschema.validate
  │
  ├─ PermissionEngine
  │    ├─ mode: DEFAULT / ACCEPT_EDITS / EXPLORE / BYPASS / DONT_ASK
  │    ├─ deny_rules / ask_rules / allow_rules
  │    ├─ tool.check_permissions()
  │    └─ safety ASK / bypass-immune / suggested rules
  │
  ├─ human-in-the-loop
  │    ├─ RequireUserConfirmEvent
  │    └─ UserConfirmResultEvent
  │
  ├─ workspace / working directory
  │    ├─ path realpath 校验
  │    ├─ ACCEPT_EDITS 只自动允许工作目录内编辑
  │    └─ Docker / E2B sandbox 可隔离执行环境
  │
  ├─ tool result 控制
  │    ├─ tool_result_limit
  │    ├─ 截断
  │    └─ offloader
  │
  └─ 服务层并发保护
       ├─ MessageBus.session_run(session_id)
       └─ state 保存必须在 lock 内
```

安全模型的核心思想是：**不要信任模型输出，也不要信任外部输入；所有能产生副作用的动作都必须经过结构化校验和权限决策。**

---

## 16.4 第一层：消息角色校验，防止用户伪造内部事件

源码：

- `src/agentscope/message/_base.py`
- `src/agentscope/agent/_agent.py` 的 `_handle_incoming_messages()`

`Msg.validate_role_content()` 限制不同 role 能携带哪些 block：

```text
user    只能携带 text / data
system  只能携带 text
assistant 可以携带 tool_call / tool_result / thinking / hint 等
```

`_handle_incoming_messages()` 还会拒绝外部输入中的：

- system role message
- tool_call block
- tool_result block
- thinking block

这很重要。否则恶意用户可以构造：

```text
UserMsg(content=[ToolCallBlock(name="Bash", input="...")])
```

让系统误以为模型要执行工具。

AgentScope 的做法是：用户输入只能是用户输入；工具调用必须来自 assistant/model 输出，并进入工具生命周期。

---

## 16.5 第二层：工具调用前的结构化校验

源码：

- `src/agentscope/agent/_agent.py` 的 `_execute_tool_call()`

模型生成 `ToolCallBlock` 后，不会直接执行。先走：

```text
toolkit.check_tool_available()
  → _json_loads_with_repair(tool_call.input, tool.input_schema)
  → jsonschema.validate(parsed_input, tool.input_schema)
```

这层防的是：

- 模型调用不存在的工具
- 模型传错参数
- JSON 不完整或格式错误
- 参数绕过工具 schema

如果失败，AgentScope 会把错误变成 `ToolResultBlock(state="error")` 写回上下文，而不是直接崩溃。这样模型下一轮能修正参数。

---

## 16.6 第三层：PermissionContext 保存权限状态

源码：

- `src/agentscope/permission/_context.py`
- `src/agentscope/state/_state.py`

`PermissionContext` 包含：

```python
mode: PermissionMode
working_directories: dict[str, AdditionalWorkingDirectory]
allow_rules: dict[str, list[PermissionRule]]
deny_rules: dict[str, list[PermissionRule]]
ask_rules: dict[str, list[PermissionRule]]
```

它存在 `AgentState.permission_context` 里，所以是 session state 的一部分。

这意味着：

- 用户批准的 rule 可以持久化到当前 session。
- session 恢复后权限上下文仍然存在。
- worker agent 可以从 leader 继承 permission mode、working directories、rules。
- workspace workdir 可以在 `ChatService` 里加入 permission context。

权限不是 prompt 里的几句话，而是结构化运行时状态。

---

## 16.7 第四层：PermissionMode 五种模式

源码：

- `src/agentscope/permission/_types.py`
- `src/agentscope/permission/_engine.py`

AgentScope 支持五种 mode：

```text
DEFAULT
  默认安全模式。没有 allow rule 或工具明确 ALLOW 时，大多需要 ASK。

ACCEPT_EDITS
  快速开发模式。自动允许工作目录内的读写和部分路径受控的文件系统命令。

EXPLORE
  只读探索模式。允许 Read/Grep/Glob 和只读 bash；修改操作 DENY。

BYPASS
  跳过大多数权限检查。适合你完全信任且有 sandbox 的环境。

DONT_ASK
  无人值守模式。所有 ASK 都转成 DENY，避免后台任务卡住或乱执行。
```

这五种模式回答的是不同运行场景：

- 人在旁边看着：`DEFAULT` 或 `ACCEPT_EDITS`
- 只想让 agent 调研代码：`EXPLORE`
- 完全隔离沙箱里跑：谨慎用 `BYPASS`
- 定时任务、后台任务：`DONT_ASK`

---

## 16.8 PermissionEngine 的评估顺序

源码：

- `src/agentscope/permission/_engine.py`

不同 mode 有不同 `_check_*` 方法，但核心思想是有顺序的。

### DEFAULT 模式

```text
1. deny rules 命中 → DENY
2. ask rules 命中 → ASK
3. tool.check_permissions()
   ├─ ALLOW / DENY → 直接返回
   ├─ safety ASK → ASK，且 allow rules 不能覆盖
   └─ PASSTHROUGH / 普通 ASK → 继续
4. allow rules 命中 → ALLOW
5. 默认 → ASK
```

这个顺序很讲究：

- deny 最高优先级。
- safety ASK 比 allow rule 更强，防止危险操作被宽泛 allow rule 绕过。
- 没有明确允许时默认询问用户。

### EXPLORE 模式

```text
1. deny rules → DENY
2. ask rules → ASK
3. tool.check_read_only()
   ├─ True → ALLOW
   └─ False → DENY
```

注意：EXPLORE 不看 allow rules。因为“只读”承诺不能被 allow rule 打破。

### ACCEPT_EDITS 模式

```text
1. deny rules
2. ask rules
3. read-only fast path → ALLOW
4. tool.check_permissions()，例如检查写入路径是否在 working directory
5. allow rules
6. 默认 ASK
```

这是开发时常用的模式：读操作不打扰，工作目录内编辑自动允许，其他危险操作仍然问。

### BYPASS 模式

BYPASS 会跳过 ASK，包括一些 safety ASK。源码注释非常直白：只适合 sandbox/container 或完全信任 agent 的环境。

### DONT_ASK 模式

DONT_ASK 的不变量是：永远不返回 ASK。所有 ASK 都转成 DENY。

这对后台任务很关键：没人确认时，不应该让任务挂住，更不应该默认执行危险动作。

---

## 16.9 第五层：工具自己的安全判断

源码：

- `src/agentscope/tool/_base.py`
- `src/agentscope/tool/_builtin/_bash.py`
- `src/agentscope/tool/_builtin/_read.py`
- `src/agentscope/tool/_builtin/_write.py`
- `src/agentscope/tool/_builtin/_edit.py`
- `src/agentscope/tool/_builtin/_grep.py`
- `src/agentscope/tool/_builtin/_glob.py`

每个工具都有一些安全属性：

```text
is_read_only
is_concurrency_safe
is_external_tool
is_state_injected
check_read_only()
check_permissions()
match_rule()
```

例子：

- `Read/Grep/Glob` 是只读，通常 `is_concurrency_safe=True`
- `Write/Edit` 会修改文件，不能随意并发
- `Bash` 静态上不是只读，但会用 bash parser 判断某条 command 是否只读
- MCP tool 可以根据 annotation 判断 read-only hint

工具还可以做路径安全判断：

```text
_path_in_allowed_working_path(file_path, context)
_is_dangerous_path(file_path)
```

`_path_in_allowed_working_path()` 会用 `realpath` 展开路径，处理 symlink、`/tmp` 到 `/private/tmp` 这种别名，避免简单字符串前缀判断被绕过。

---

## 16.10 第六层：human-in-the-loop 确认

源码：

- `src/agentscope/agent/_agent.py`
- `src/agentscope/event/_event.py`

当权限决策是 `ASK` 或 `PASSTHROUGH`：

```text
ToolCallBlock.state = ASKING
yield RequireUserConfirmEvent(tool_calls=[...])
return  # 当前 reply 暂停
```

用户确认后，会传回 `UserConfirmResultEvent`。

`_handle_incoming_event()` 会处理：

- 用户同意：tool call state 改成 `ALLOWED`
- 用户拒绝：生成 `ToolResultBlock(state="denied")`
- 用户修改 tool call：更新工具名和 input
- 用户接受 suggested rules：写入 PermissionEngine

这说明确认不是“点一下继续”那么简单；用户可以修改参数，也可以把确认转成后续复用的规则。

---

## 16.11 第七层：continuation event 校验

源码：

- `src/agentscope/agent/_agent.py` 的 `_check_incoming_event()`

Agent 暂停后，不是什么事件都能恢复它。

`_check_incoming_event()` 会检查当前最后一条 assistant message：

- 哪些 tool call 处于 `ASKING`
- 哪些 tool call 处于 `SUBMITTED` 且还没有 result

然后校验传入事件：

- 如果 agent 没在等确认，却收到 `UserConfirmResultEvent` → 报错
- 如果确认事件包含不在等待列表里的 tool call id → 报错
- 如果 agent 正在等确认，但你传 `None` → 报错

这层防的是错误恢复、重复确认、跨 reply 混淆、外部结果错投。

---

## 16.12 第八层：workspace / sandbox 隔离

源码：

- `src/agentscope/workspace/`
- `src/agentscope/app/workspace_manager/`
- `src/agentscope/app/_service/_chat.py`

Workspace 相关安全有两层。

### 工作目录权限

`ChatService` 会把 workspace workdir 加入当前 session 的 `permission_context.working_directories`。

这样 `ACCEPT_EDITS` 模式可以只自动允许工作目录内文件修改，而不是允许整个文件系统。

### 执行环境隔离

AgentScope 支持本地、Docker、E2B 等 workspace。Docker/E2B 让工具执行发生在容器/沙箱里，适合更高风险的自动化任务。

但是要注意：sandbox 不是替代权限系统的理由。尤其是 BYPASS 模式，源码注释也强调应只在 sandboxed / containerized 或完全信任场景使用。

---

## 16.13 第九层：工具结果安全与上下文污染控制

源码：

- `src/agentscope/agent/_agent.py` 的 `_split_tool_result_for_compression()`
- `ContextConfig.tool_result_limit`

安全不只是“能不能执行危险命令”，还包括“工具返回的内容会不会污染或撑爆上下文”。

AgentScope 对超长工具结果会：

1. 估算 token
2. 超过 `tool_result_limit` 时截断
3. 如果有 offloader，把剩余内容外置
4. 在保留结果里加入 `<system-reminder>`，告诉模型内容被省略

这防止一次 `cat huge.log`、`grep`、`bash` 输出把后续模型上下文挤爆。

---

## 16.14 第十层：Team 安全边界

源码：

- `src/agentscope/app/_service/_toolkit.py`
- `src/agentscope/app/_tools/_team_tool_base.py`
- `src/agentscope/app/_tools/_agent_create.py`
- `src/agentscope/app/_tools/_team_say.py`

Team 安全主要在三个点。

### 工具分配按角色区分

`get_toolkit()` 里：

```text
source == "team" → worker 只拿 TeamSay
普通 user agent → leader 拿 TeamCreate / AgentCreate / TeamSay / TeamDelete
```

worker 不能继续创建 worker，避免 team 无限递归扩张。

### team tool 运行时检查

Team tools 虽然 `check_permissions()` 总是 allow，但它们在 `__call__()` 内部检查运行时状态：

- 当前 session 是否属于 team
- 当前 session 是否是 leader
- team 是否存在
- worker name 是否唯一
- recipient 是否在当前 team directory 里
- 不能给自己发 TeamSay

也就是说，team tools 的安全不是靠 PermissionEngine，而是靠工具附加逻辑和 storage state。

### worker 权限继承受模板控制

`AgentCreate` 创建 worker 时，会根据 `SubAgentTemplate` 合并 leader 权限：

- `override_leader_mode`
- `extend_leader_permission_rules`
- `extend_leader_working_directories`

所以 worker 权限不是无脑复制 leader，而是模板驱动。

---

## 16.15 第十一层：服务层并发安全

源码：

- `src/agentscope/app/_service/_chat.py`
- `src/agentscope/app/message_bus/_base.py`

同一个 session 可能被多种来源触发：

- 用户发消息
- TeamSay 唤醒
- 后台工具完成
- 定时任务触发
- 外部执行结果返回

如果两个 run 同时修改 `AgentState.context`，状态会损坏。

所以 ChatService 用：

```python
async with self._message_bus.session_run(session_id):
    run agent
    upsert reply message
    update session state
```

`session_run()` 是 per-session distributed lock。状态保存必须在 lock 内完成，防止另一个进程读到旧 state。

---

## 16.16 第十二层：长期记忆隔离

源码：

- `src/agentscope/middleware/_longterm_memory/_mem0/_middleware.py`

Mem0Middleware 搜索时用 filters：

```text
user_id
agent_id 可选
```

`scope_search_by_agent=True` 时，会同时按 user 和 agent 隔离。否则按 user 共享。

这不是传统权限系统，但属于数据隔离安全：不同用户的 memory 不能串；是否跨 agent 共享也要显式配置。

---

## 16.17 经典问题：AgentScope 的安全主线是什么？

一句话：

> 用户输入不能伪造内部动作；模型输出不能绕过工具 schema 和权限系统；工具执行不能越过工作目录和模式策略；服务层不能并发写坏 session；多 agent 不能越权创建或投递到非团队成员。

对应链路：

```text
Msg validation
  → ToolCall schema validation
  → PermissionEngine
  → Tool-specific safety checks
  → Human confirmation
  → Workspace boundary
  → Tool result truncation
  → Session lock
  → Team role gating
  → Memory namespace filters
```

---

## 16.18 学习建议：安全源码怎么读？

建议顺序：

1. 先读 `permission/_types.py`，理解五种 PermissionMode。
2. 再读 `permission/_engine.py`，逐个看 `_check_default`、`_check_explore`、`_check_accept_edits`、`_check_bypass`、`_check_dont_ask`。
3. 回到 `agent/_agent.py` 的 `_execute_tool_call()`，看 PermissionDecision 如何影响工具生命周期。
4. 读 `tool/_base.py` 的路径检查和 dangerous path。
5. 读 built-in tools 的 `check_permissions()`，尤其 Bash/Read/Write/Edit。
6. 读 `ChatService` 的 session lock，理解并发安全。
7. 读 Team tools，看角色能力隔离。

---

## 16.19 本章检查题

1. 为什么 user message 不能携带 `ToolCallBlock`？
2. `DEFAULT`、`EXPLORE`、`ACCEPT_EDITS` 三种模式最核心的差异是什么？
3. `DONT_ASK` 为什么把 ASK 转成 DENY，而不是 ALLOW？
4. `BYPASS` 为什么必须谨慎使用？
5. safety ASK 为什么能绕过 allow rule？
6. `PermissionContext.working_directories` 如何参与文件写入安全？
7. Team tools 为什么不完全依赖 PermissionEngine？
8. `session_run` 保护了哪类安全问题？
9. 长期记忆为什么也需要 user_id / agent_id 隔离？
