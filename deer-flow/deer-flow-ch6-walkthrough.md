# 第 6 章 精讲版：安全与稳定性——Guardrails、循环检测、HITL

> Agent 是个"会自己干活"的 LLM。但"自己干活"意味着它会**犯错**——死循环、调用危险工具、内容被审核截断、该问用户却自作主张。
>
> 这一章讲 DeerFlow 怎么用 6 个中间件保证 agent "不失控"。
>
> **配套文件**（建议同时打开对照看）：
> - `guardrails/provider.py` / `builtin.py` / `middleware.py`
> - `agents/middlewares/loop_detection_middleware.py`
> - `agents/middlewares/safety_finish_reason_middleware.py` / `safety_termination_detectors.py`
> - `agents/middlewares/clarification_middleware.py`
> - `agents/middlewares/todo_middleware.py`
> - `agents/middlewares/subagent_limit_middleware.py`
> - `config/loop_detection_config.py`

---

# 🎯 先建立心理模型：Agent 的 6 种"失控"和对应的"护栏"

```
Agent 可能的失控方式           →  DeerFlow 的护栏
─────────────────────────────────────────────────
① 调用不该调的工具              →  Guardrails（工具调用前授权）
② 反复调同一个工具（死循环）     →  LoopDetection（双层检测 + 硬停）
③ 内容被审核截断但还带 tool_calls →  SafetyFinishReason（清 tool_calls）
④ 该问用户却自作主张            →  Clarification（HITL 中断）
⑤ 没干完就提前收尾              →  TodoMiddleware（强制再来一轮）
⑥ 一轮派太多子 agent            →  SubagentLimit（截断到 3 个）
```

**关键认知**：这些护栏不是"事后补救"，而是**在中间件管道里拦截**——在模型输出到达工具/用户之前就处理掉。

---

# 📖 6.1 Guardrails——工具调用前的策略授权

**文件**：`guardrails/provider.py` / `builtin.py` / `middleware.py`

## 6.1.1 什么是 Guardrails（在 DeerFlow 里）

**注意**：DeerFlow 的 "guardrails" 是 **pre-tool-call 授权**（"这个工具能不能调"），**不是**输出内容审核（那是 SafetyFinishReason 的事，6.3 节讲）。

## 6.1.2 契约

**文件**：`guardrails/provider.py`

```python
@dataclass
class GuardrailRequest:
    tool_name: str
    tool_input: dict[str, Any]
    agent_id: str | None = None

@dataclass
class GuardrailDecision:
    allow: bool
    reasons: list[GuardrailReason] = field(default_factory=list)
    policy_id: str | None = None

@runtime_checkable
class GuardrailProvider(Protocol):
    name: str
    def evaluate(self, request: GuardrailRequest) -> GuardrailDecision: ...
    async def aevaluate(self, request: GuardrailRequest) -> GuardrailDecision: ...
```

字段名（`policy_id`、reason `code` 如 `"oap.tool_not_allowed"`）对齐 **OAP（Open Agent Protocol）**。

## 6.1.3 内置 provider

**文件**：`guardrails/builtin.py:6-23`

```python
class AllowlistProvider:
    """Simple allow/deny list guardrail provider."""
    
    def evaluate(self, request):
        tool = request.tool_name
        if self._denied and tool in self._denied:
            return GuardrailDecision(
                allow=False,
                reasons=[GuardrailReason(code="oap.tool_not_allowed", 
                                         message=f"Tool '{tool}' is denied")],
            )
        if self._allowed and tool not in self._allowed:
            return GuardrailDecision(
                allow=False,
                reasons=[GuardrailReason(code="oap.tool_not_allowed",
                                         message=f"Tool '{tool}' is not in allowlist")],
            )
        return GuardrailDecision(allow=True, reasons=[GuardrailReason(code="oap.allowed")])
```

就是简单的 allow/deny 工具列表。

## 6.1.4 中间件——怎么拦

**文件**：`guardrails/middleware.py:20-98`

`GuardrailMiddleware` 实现 `wrap_tool_call` / `awrap_tool_call`，在工具执行**前**调 provider：

```python
class GuardrailMiddleware(AgentMiddleware):
    def __init__(self, provider, *, fail_closed=True, passport=None):
        self._provider = provider
        self._fail_closed = fail_closed    # provider 异常时是否拒绝（默认 True）

    def wrap_tool_call(self, request, handler):
        decision = self._provider.evaluate(GuardrailRequest(
            tool_name=request.tool_call["name"],
            tool_input=request.tool_call.get("args", {}),
        ))
        if not decision.allow:
            return self._deny_tool_message(request, decision)   # 返回错误 ToolMessage
        return handler(request)    # 放行
```

**三个关键设计**：

### ① `GraphBubbleUp` 重新抛出

```python
# middleware.py:63-65, 86-88
except GraphBubbleUp:
    raise    # 保留 LangGraph 控制流信号（interrupt/pause/resume）
```

**为什么？** guardrail 不能吞掉 HITL 中断信号。如果 ClarificationMiddleware 设了 `goto=END`，那个信号是 `GraphBubbleUp` 异常——guardrail 必须让它通过。

### ② fail_closed（默认 True）

provider 异常时合成 deny（`code="oap.evaluator_error"`）——宁可错杀不可放过。

### ③ deny 时返回 ToolMessage 而非抛异常

```python
def _deny_tool_message(self, request, decision):
    return ToolMessage(
        content=f"Guardrail denied: tool '{name}' was blocked (oap.tool_not_allowed). "
                f"Reason: {reason}. Choose an alternative approach.",
        status="error",
    )
```

**为什么不抛异常？** 因为抛异常会被 ToolErrorHandlingMiddleware 捕获，agent 看到的是"工具崩溃"，会**重试**——陷入"调用-崩溃-重试"循环。返回 ToolMessage 让 agent **理解约束**并主动换方案。

---

# 📖 6.2 LoopDetection——防止 agent 死循环

**文件**：`agents/middlewares/loop_detection_middleware.py`（613 行）。**P0 安全中间件**。

## 6.2.1 问题：agent 会"撞墙"死循环

agent 可能反复调同一个工具（比如反复 `read_file` 同一个文件），直到撞 LangGraph 的递归上限——浪费 token、浪费时间、用户体验极差。

## 6.2.2 双层检测

### Layer 1：基于 hash（相同调用集）

**文件**：`loop_detection_middleware.py:142-160`

```python
def _hash_tool_calls(tool_calls):
    """Hash a set of tool calls, order-independent."""
    keys = sorted([_stable_tool_key(tc) for tc in tool_calls])
    return hashlib.md5(json.dumps(keys).encode()).hexdigest()
```

**关键**：**排序后** MD5——顺序无关（同一组工具调用的任意排列产生相同 hash）。

hash 追加到 per-thread 滑动窗口（`window_size=20`），然后统计：

- `count >= warn_threshold`（默认 3）→ 排队警告；
- `count >= hard_limit`（默认 5）→ **硬停**。

### Layer 2：per-tool-type 频率

**文件**：`loop_detection_middleware.py:399-436`

hash 检测漏掉的情况——比如 40 次不同的 `read_file`（路径不同，hash 不同）。每个工具类型有独立计数：

```python
freq[name] += 1    # 每轮累加
# 默认：warn 30, hard 50
# 可用 tool_freq_overrides 给 bash 调高
```

## 6.2.3 stable key 推导——避免过拟合

**文件**：`loop_detection_middleware.py:99-139`

```python
def _stable_tool_key(tool_call):
    name = tool_call["name"]
    args = tool_call.get("args", {})
    
    if name == "read_file":
        path = args.get("path", "")
        start = args.get("start_line", 1)
        bucket = (start - 1) // 200    # ★ 200 行分桶
        return f"read_file:{path}:{bucket}"
    
    if name in ("write_file", "str_replace"):
        return f"{name}:" + hashlib.md5(json.dumps(args).encode()).hexdigest()
    
    # 其他工具：只取显著字段
    salient = {k: args[k] for k in ("path", "url", "query", "command", "pattern", "glob", "cmd") if k in args}
    return f"{name}:{json.dumps(salient, sort_keys=True)}"
```

**为什么 read_file 要分桶？** 因为"读第 1-200 行"和"读第 50-250 行"本质上是**同一次操作**（在读同一片区域）。按 200 行分桶，读附近行算同一次——避免"只是行号不同"就绕过检测。

**为什么 write_file 要 hash 全部 args？** 因为写文件的内容**是敏感的**——同一路径可能合理地写入不同内容（迭代修改），不能按路径去重。

## 6.2.4 关键的 hook 顺序设计——延迟注入

**文件**：`loop_detection_middleware.py:18-38`（docstring）

> `after_model` 在模型 emit `AIMessage(tool_calls)` 后立即触发，此时 ToolNode 还没跑，没有配对的 ToolMessage。如果这时插消息，会落在 assistant 的 tool_calls 和它们的 response 之间——OpenAI/Moonshot 下一次请求会报 `"tool_call_ids did not have response messages"`。

**所以**：

- `after_model`（534-540 行）：**检测**，硬停就直接清 tool_calls 强制文本输出；否则**排队警告**（暂存 `_pending_warnings`，capped 4 条/run）；
- `wrap_model_call`（579-593 行）：**排空 pending warnings**，追加一条 `HumanMessage(name="loop_warning")`。

**硬停怎么做**（`_build_hard_stop_update`，457-475 行）：

```python
# 清空 tool_calls，强制模型只能输出文本
ai_message.tool_calls = []
ai_message.additional_kwargs.pop("function_call", None)
ai_message.response_metadata["finish_reason"] = "stop"  # was "tool_calls"
```

**为什么 not 直接中断图？** 因为硬停后让模型**自己总结**"我为什么要停"并输出最终答案，比直接报错体验好。

## 6.2.5 配置

**文件**：`config/loop_detection_config.py`

```python
class LoopDetectionConfig(BaseModel):
    enabled: bool = True
    warn_threshold: int = 3       # 相同 hash 出现 3 次警告
    hard_limit: int = 5           # 5 次硬停
    window_size: int = 20         # 滑动窗口 20 步
    max_tracked_threads: int = 100
    tool_freq_warn: int = 30      # 单工具类型 30 次警告
    tool_freq_hard_limit: int = 50  # 50 次硬停
    tool_freq_overrides: dict[str, ToolFreqOverride]  # per-tool 覆盖
```

`@model_validator` 强制 `hard_limit >= warn_threshold`。

---

# 📖 6.3 SafetyFinishReason——处理内容审核中途截断

**文件**：`agents/middlewares/safety_finish_reason_middleware.py` + `safety_termination_detectors.py`

## 6.3.1 问题：内容审核截断 + 部分工具调用 = 死循环

**真实场景**（issue #3028）：

```
1. 模型生成 write_file(content="敏感内容...") 的部分 tool_calls
2. 提供商检测到违规 → finish_reason=content_filter
   但已生成的 tool_calls 还在响应里！
3. LangChain 不区分"完整 tool_calls"和"被截断的"，照样派发给 ToolNode
4. write_file 写了个残缺文件
5. 下一轮模型看到残缺文件，尝试 str_replace 修复
6. 又触发 content_filter → 又截断 → 又修复 → 死循环
```

## 6.3.2 三个 detector

**文件**：`safety_termination_detectors.py`

```python
class OpenAICompatibleContentFilterDetector:
    # finish_reason ∈ {"content_filter"}
    # +Azure 的 content_filter_results

class AnthropicRefusalDetector:
    # stop_reason == "refusal"

class GeminiSafetyDetector:
    # finish_reason ∈ {SAFETY, BLOCKLIST, PROHIBITED_CONTENT, SPII, RECITATION, IMAGE_SAFETY, ...}
    # 排除 STOP, MAX_TOKENS 等正常值

def default_detectors():
    return [OpenAICompatibleContentFilterDetector(), 
            AnthropicRefusalDetector(), 
            GeminiSafetyDetector()]
```

## 6.3.3 中间件怎么处理

**文件**：`safety_finish_reason_middleware.py:207-262`

```python
def after_model(self, state, response):
    ai_message = response.message
    for detector in self._detectors:
        reason = detector.detect(ai_message)
        if reason:
            # ★ 清空 tool_calls！
            cleaned = ai_message.model_copy(update={
                "tool_calls": [],
                "additional_kwargs": {
                    **ai_message.additional_kwargs,
                    "safety_termination": {
                        "detector": detector.name,
                        "reason_field": reason.field,
                        "reason_value": reason.value,
                        "suppressed_tool_call_count": len(ai_message.tool_calls),
                    }
                }
            })
            # 发 safety_termination SSE 事件
            get_stream_writer()({"type": "safety_termination", ...})
            # 写审计记录到 RunJournal
            journal = runtime.context.get("__run_journal")
            if journal:
                journal.record_middleware("safety_termination", ...)
            return {"messages": [cleaned]}
    return {}
```

**关键**：

1. **清 tool_calls**——让模型看到的是"我的输出被安全过滤了"，而不是残缺的工具调用；
2. **打 `additional_kwargs["safety_termination"]` 标记**——可观测性（但不记 tool arguments，因为那是被过滤的内容）；
3. **发 SSE + 写 journal**——前端知道发生了什么，审计有记录。

## 6.3.4 顺序设计

中间件**排在 LoopDetection 之后**（factory 里 `after_model` 逆序，最后注册的最先看）。注释（`safety_finish_reason_middleware.py:25-33`）：

> "Registered after custom middlewares so that LangChain's reverse-order after_model dispatch runs Safety first; cleared tool_calls then flow through Loop/Subagent accounting without firing extra alarms."

Safety 先跑 → 清掉 tool_calls → 然后这些"已清"的消息流经 Loop/Subagent 计数器 → **不会触发误报**（因为没有 tool_calls 了，Loop 不会把它算作重复调用）。

---

# 📖 6.4 Clarification——Human-in-the-Loop

**文件**：`agents/middlewares/clarification_middleware.py`

## 6.4.1 什么时候需要 HITL

agent 遇到这些情况应该**问用户**而不是自作主张：

| 类型 | icon | 例子 |
|---|---|---|
| `missing_info` | ❓ | "你需要哪个项目的报告？" |
| `ambiguous_requirement` | 🤔 | "你说的'优化'是指性能还是代码可读性？" |
| `approach_choice` | 🔀 | "方案 A 更快但风险高，方案 B 更稳，选哪个？" |
| `risk_confirmation` | ⚠️ | "这会删除 100 个文件，确认吗？" |
| `suggestion` | 💡 | "我建议先跑测试再提交，可以吗？" |

## 6.4.2 实现——拦截 + 中断

**文件**：`clarification_middleware.py:117-156`

```python
class ClarificationMiddleware(AgentMiddleware):
    def wrap_tool_call(self, request, handler):
        name = request.tool_call.get("name")
        if name != "ask_clarification":
            return handler(request)    # 不是 clarification 工具，放行
        
        # ★ 拦截！不执行工具，直接构造 ToolMessage
        args = request.tool_call.get("args", {})
        question = args.get("question", "")
        clarification_type = args.get("type", "missing_info")
        options = args.get("options", [])
        
        # 格式化问题（带 icon + 编号选项）
        formatted = self._format_question(question, clarification_type, options)
        
        tool_message = ToolMessage(
            content=formatted,
            tool_call_id=request.tool_call["id"],
            id=f"clarification:{request.tool_call['id']}",    # 确定性 id
        )
        
        # ★ 中断！goto=END
        return Command(update={"messages": [tool_message]}, goto=END)
```

**关键**：返回 `Command(update={...}, goto=END)`——**图执行到此结束**，run 状态变 `interrupted`。

## 6.4.3 HITL 的完整流程

```
1. 模型调 ask_clarification(question="选 A 还是 B？", options=["A","B"])
2. ClarificationMiddleware 拦截 → 格式化 → Command(goto=END)
3. 图结束 → run 状态 = interrupted
4. 前端收到 interrupt 事件 → 弹问题给用户
5. 用户回答 "A"
6. 前端发新一轮 run，答案作为新 HumanMessage 注入
7. 图从头跑，模型看到答案 → 继续
```

**这是 LangGraph 实现 HITL 的标准模式**：用图的终止表达"暂停"，用新 run 表达"继续"。

---

# 📖 6.5 TodoMiddleware——防止"提前收尾"

**文件**：`agents/middlewares/todo_middleware.py`。继承 LangChain 的 `TodoListMiddleware`。

## 6.5.1 两个额外职责

### ① 上下文丢失检测（`before_model`，120-156 行）

```python
def before_model(self, state):
    todos = state.get("todos")
    if not todos:
        return None
    
    # 检查消息里有没有 write_todos 调用
    messages = state["messages"]
    has_write_todos = any(
        self._has_write_todos_call(msg) for msg in messages
    )
    
    if not has_write_todos:
        # ★ 摘要把 write_todos 调用截掉了，模型"忘记"自己有 todo
        return {"messages": [HumanMessage(
            content=f"提醒：你还有 {len(todos)} 个未完成的任务：\n{format_todos(todos)}",
            name="todo_reminder",
            additional_kwargs={"hide_from_ui": True},
        )]}
```

**问题**：SummarizationMiddleware 把旧的 `write_todos` 工具调用消息压缩掉了，但 `state["todos"]` 还在。模型看不到"自己设了 todo"的历史 → **忘记有任务**。中间件注入提醒，让模型重新意识到。

### ② 提前退出防护（`after_model`，265-309 行）

```python
@hook_config(can_jump_to=["model"])
def after_model(self, state, response):
    ai_message = response.message
    
    # 模型给了最终回复（无 tool_calls）但 todo 未完成
    if not self._has_tool_call_intent(ai_message):
        todos = state.get("todos") or []
        incomplete = [t for t in todos if t.get("status") != "completed"]
        
        if incomplete and self._completion_reminders < _MAX_COMPLETION_REMINDERS:  # 最多 2 次
            self._queue_completion_reminder(...)
            return {"jump_to": "model"}    # ★ 强制再来一轮！
```

**`return {"jump_to": "model"}`** 是 LangGraph 的指令——不结束图，强制再跑一轮 model。模型会看到提醒"你的 todo 还没做完"，被迫继续干活。

**`_MAX_COMPLETION_REMINDERS=2`**——最多提醒 2 次，防止无限循环。提醒 2 次后模型还坚持收尾，就放过它（可能模型判断剩下的任务不重要）。

## 6.5.2 延迟注入模式（和 LoopDetection 一样）

提醒消息在 `after_model` **排队**，在 `wrap_model_call` **排空追加**——同样的"不在 AIMessage(tool_calls) 和 ToolMessage 之间插消息"约束。

---

# 📖 6.6 SubagentLimitMiddleware——并行限制

**文件**：`agents/middlewares/subagent_limit_middleware.py`

第 2 章已详讲，这里补充核心逻辑：

```python
class SubagentLimitMiddleware(AgentMiddleware):
    def __init__(self, max_concurrent: int = 3):
        self._max = _clamp_subagent_limit(max_concurrent)   # clamp 到 [2,4]

    def after_model(self, state, response):
        task_calls = [tc for tc in response.message.tool_calls if tc["name"] == "task"]
        if len(task_calls) > self._max:
            # 只保留前 N 个，截掉超出的
            kept = task_calls[:self._max]
            modified = self._modify_message(response.message, kept)
            return {"messages": [modified]}
        return {}
```

**`_clamp_subagent_limit`** 把值限制在 `[2,4]`——即使配置写 `max_concurrent=10`，也只用 4。这是**硬保障**，不靠 prompt。

---

# 🎯 现在回头检验：你掌握了吗？

### 问题 1：循环检测为什么不直接在 `after_model` 加警告消息？

<details>
<summary>🔍 点击看答案</summary>

因为会破坏 **AIMessage(tool_calls) ↔ ToolMessage 的配对**。OpenAI/Moonshot API 的硬性校验：每个 tool_call 必须有对应的 tool response，且两者必须**相邻**（中间不能插别的角色消息）。

如果在 `after_model`（此时 ToolNode 还没跑）插一条 HumanMessage，它会落在 AIMessage(tool_calls) 和即将到来的 ToolMessage 之间——下一次请求被服务端拒绝。

解法是"延迟注入"：`after_model` 检测 + 排队 → `wrap_model_call` 排空 + 追加（此时所有 ToolMessage 都已就位）。

</details>

### 问题 2：Guardrail 拒绝工具调用时，为什么返回 ToolMessage 而不是抛异常？

<details>
<summary>🔍 点击看答案</summary>

抛异常会被 ToolErrorHandlingMiddleware 捕获，agent 看到的是"工具崩溃"，会**重试**——陷入"调用-崩溃-重试"循环。

返回 `ToolMessage(status="error", content="Guardrail denied: ... Choose an alternative approach.")` 让 agent **理解约束**并主动换方案。这是把 guardrail 决策**翻译成 agent 能读懂的反馈**。

</details>

### 问题 3：内容审核截断 tool_calls，为什么会让 agent 死循环？

<details>
<summary>🔍 点击看答案</summary>

完整链条：

1. 模型生成 `write_file(content="敏感内容...")` 的**部分** tool_calls；
2. 提供商检测到违规，`finish_reason=content_filter`，但**已生成的 tool_calls 还在响应里**；
3. LangChain 不区分"完整"和"被截断的"，照样派发给 ToolNode；
4. `write_file` 写了个**残缺文件**；
5. 下一轮模型看到残缺文件，尝试 `str_replace` 修复；
6. 又触发 content_filter → 又截断 → 又修复 → **死循环**。

`SafetyFinishReasonMiddleware` 在第 3 步之前**清空 tool_calls**，让模型看到"输出被安全过滤了"，从而换思路。

</details>

### 问题 4：HITL 的 Clarification 怎么实现"暂停-等用户-继续"？

<details>
<summary>🔍 点击看答案</summary>

用 LangGraph 的 `Command(goto=END)`：

1. 模型调 `ask_clarification(...)`；
2. 中间件拦截，返回 `Command(update={"messages": [tool_message]}, goto=END)`——**图终止**，run 状态 = `interrupted`；
3. 前端收到 interrupt 事件，弹问题给用户；
4. 用户回答 → 前端发新一轮 run，答案作为新 HumanMessage 注入 → 图从头跑。

用图的终止表达"暂停"，用新 run 表达"继续"——这是 LangGraph HITL 的标准模式。

</details>

### 问题 5：TodoMiddleware 为什么要在 `after_model` 检测但 `wrap_model_call` 注入？

<details>
<summary>🔍 点击看答案</summary>

和 LoopDetection 一样的原因——**不能在 AIMessage(tool_calls) 和 ToolMessage 之间插消息**，否则 OpenAI/Moonshot 报 `"tool_call_ids did not have response messages"`。

`after_model` 时 ToolNode 还没跑，没有配对的 ToolMessage。所以检测在 `after_model`，注入延迟到 `wrap_model_call`（此时所有 ToolMessage 已就位，追加在末尾安全）。

</details>

---

# 📝 一页纸总结（第 6 章精华）

```
┌──────────────────────────────────────────────────────────────┐
│  6 个中间件 = 6 道护栏，保证 agent 不失控                       │
└──────────────────────────────────────────────────────────────┘
        │
        ├─▶ Guardrails（工具授权）
        │     • wrap_tool_call 拦截，fail_closed
        │     • deny 返 ToolMessage（非异常）→ agent 换方案
        │     • GraphBubbleUp 透传（不吞 HITL 信号）
        │
        ├─▶ LoopDetection（防死循环）
        │     • Layer 1: hash 检测（同调用集，warn 3 / hard 5）
        │     • Layer 2: per-tool 频率（warn 30 / hard 50）
        │     • stable key: read_file 200 行分桶 / write_file 全 args hash
        │     • 延迟注入：after_model 检测 → wrap_model_call 注入
        │     • 硬停：清 tool_calls + flip finish_reason → 强制文本输出
        │
        ├─▶ SafetyFinishReason（内容审核截断）
        │     • 3 个 detector：OpenAI/Anthropic/Gemini
        │     • 清 tool_calls → 防残缺文件死循环
        │     • 打 safety_termination 标记 + SSE + journal 审计
        │     • 排在 Loop 之后（逆序 → 最先看 → 清完再过 Loop 计数）
        │
        ├─▶ Clarification（HITL）
        │     • 拦截 ask_clarification → Command(goto=END)
        │     • run 状态 = interrupted → 前端弹问题
        │     • 用户答 → 新 run → 图从头跑
        │
        ├─▶ TodoMiddleware（防提前收尾）
        │     • before_model: 上下文丢失检测（摘要把 todo 截掉了 → 注入提醒）
        │     • after_model: 提前退出防护（todo 没完 → jump_to:model 强制再来）
        │     • 最多提醒 2 次（防无限循环）
        │     • 延迟注入（同 LoopDetection）
        │
        └─▶ SubagentLimit（并行限制）
              • after_model: 单轮 task 调用 > max → 截断
              • clamp [2,4]，默认 3
              • 硬保障（不靠 prompt）
```

**5 句话记住**：

1. **Guardrails 是工具授权**——deny 返 ToolMessage 不抛异常，fail_closed，透传 GraphBubbleUp；
2. **LoopDetection 双层检测**——hash（相同调用集）+ per-tool 频率，延迟注入防 tool_calls 配对断裂；
3. **SafetyFinishReason 清 tool_calls**——防内容审核截断导致的残缺文件死循环，排在 Loop 之前；
4. **Clarification 用 `goto=END` 实现 HITL**——图终止 = 暂停，新 run = 继续；
5. **Todo 防提前收尾**——上下文丢失检测 + `jump_to:model` 强制再来，最多 2 次。
