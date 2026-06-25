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
