# 第 4 章 精讲版：上下文工程（Context Engineering）与压缩

> "Context Engineering" 是 Agent 工程的核心议题：**怎么管理送进模型上下文窗口的全部内容**——system prompt、历史消息、工具 schema、工具输出、注入的提醒。
>
> DeerFlow 在这一层做了 5 件事：
> ① 静态 prompt + 动态 reminder 分离（DynamicContext，第 1 章讲过注入机制，这章讲它和摘要的配合）；
> ② 摘要压缩（SummarizationMiddleware）；
> ③ 工具输出预算（ToolOutputBudgetMiddleware）；
> ④ 工具 schema 延迟加载（DeferredToolFilter，第 1 章 merge_promoted 讲过 state 侧，这章讲 middleware 侧）；
> ⑤ token 归因统计（TokenUsageMiddleware）。
>
> **配套文件**（建议同时打开对照看）：
> - `agents/middlewares/summarization_middleware.py`
> - `agents/middlewares/dynamic_context_middleware.py`
> - `agents/middlewares/tool_output_budget_middleware.py`
> - `agents/middlewares/deferred_tool_filter_middleware.py`
> - `agents/middlewares/token_usage_middleware.py`
> - `config/summarization_config.py` / `tool_output_config.py` / `token_usage_config.py`

---

# 🎯 先建立心理模型：上下文窗口是个"昂贵的小房间"

LLM 的上下文窗口就像一个**租金昂贵的小房间**。每次调模型，你要把"东西"搬进这个房间：

```
┌──────────── 上下文窗口（比如 128K token）────────────────┐
│  ① system prompt（固定，~2K）                              │
│  ② 历史消息（会变长！）                                    │
│  ③ 工具 schema（每个工具几百 token，几十个就上万）          │
│  ④ 工具输出（bash 跑一下可能几万 token）                   │
│  ⑤ 注入的提醒（memory、date、todo reminder）              │
│                                                           │
│  → 越装越多，最终塞爆 → 模型变慢、变贵、甚至报错            │
└───────────────────────────────────────────────────────────┘
```

**Context Engineering 就是"房间管理"**——决定什么进、什么出、怎么压缩、怎么省。DeerFlow 用 5 个中间件各管一件事：

| 中间件 | 管什么 | 一句话策略 |
|---|---|---|
| DynamicContext | ① system prompt 静态化 + ⑤ 注入 | 静态 prompt 留给 prefix cache，动态内容注入第一条 user msg |
| Summarization | ② 历史消息 | 超阈值就把旧消息压成摘要，但救下 skill 文件和 reminder |
| ToolOutputBudget | ④ 工具输出 | 超长输出写盘 + 给头尾预览，模型按需 read_file |
| DeferredToolFilter | ③ 工具 schema | MCP 工具默认藏 schema，模型 search 了才暴露 |
| TokenUsage | 统计 | 记 token + 给每步打 attribution（不强制，只观测） |

我们一个个深入。

---

# 📖 4.1 SummarizationMiddleware——什么时候压缩、压什么、怎么插回去

**文件**：`agents/middlewares/summarization_middleware.py`。继承 LangChain 的 `SummarizationMiddleware`。

这是上下文工程的**重头戏**。我们按"触发→分区→生成摘要→插回"四步讲。

## 4.1.1 何时触发：`_maybe_summarize`

**文件**：`summarization_middleware.py:195-219`

```python
def _maybe_summarize(self, state: AgentState, runtime: Runtime) -> dict | None:
    messages = state["messages"]
    self._ensure_message_ids(messages)

    total_tokens = self.token_counter(messages)           # 算总 token
    if not self._should_summarize(messages, total_tokens):  # 够不够阈值？
        return None

    cutoff_index = self._determine_cutoff_index(messages)   # 从哪切？
    if cutoff_index <= 0:
        return None

    # ★ 分区：哪些压、哪些留
    messages_to_summarize, preserved_messages = self._partition_with_skill_rescue(messages, cutoff_index)
    messages_to_summarize, preserved_messages = self._preserve_dynamic_context_reminders(messages_to_summarize, preserved_messages)
    
    # ★ 触发 before_summarization 钩子（记忆冲洗！）
    self._fire_hooks(messages_to_summarize, preserved_messages, runtime)
    
    # ★ 生成摘要
    summary = self._create_summary(messages_to_summarize)
    
    # ★ 构建新消息列表
    new_messages = self._build_new_messages(summary)

    return {
        "messages": [
            RemoveMessage(id=REMOVE_ALL_MESSAGES),   # ★ 清空全部历史
            *new_messages,                            # 摘要
            *preserved_messages,                      # 保留的近期消息
        ]
    }
```

**触发时机**：`before_model`（每次调模型前），所以中间件要**早装**（第 1 章讲过）。

**`_should_summarize` 和 `_determine_cutoff_index` 是继承的**（没 override），靠配置驱动。看 `config/summarization_config.py:21-53`：

```python
class SummarizationConfig(BaseModel):
    enabled: bool = False                    # 默认关，需显式开
    model_name: str | None = None            # 用哪个模型做摘要（None=轻量模型）
    trigger: ContextSize | list[ContextSize] | None = None   # 触发阈值
    keep: ContextSize = ContextSize(type="messages", value=20)  # 保留多少
    trim_tokens_to_summarize: int | None = 4000    # 摘要前最多喂多少 token
    ...
```

**trigger 支持三种 type**（`ContextSizeType = Literal["fraction", "tokens", "messages"]`）：

- `messages`：消息数（比如 50 条触发）；
- `tokens`：绝对 token 数（比如 4000）；
- `fraction`：占模型 max input 的百分比（比如 0.8 = 80%）。

**可以传列表**（OR 语义）——任何一个阈值满足就触发。比如 `[{'type':'messages','value':50}, {'type':'tokens','value':4000}]` 表示"50 条消息 **或** 4000 token 就压缩"。

**`keep`** 决定保留多少近期消息（默认 20 条）。`_determine_cutoff_index` 据此算出 cutoff。

## 4.1.2 压什么（不压什么）——两个保留逻辑

DeerFlow 的特色在于它**不傻压**，有两个保留逻辑。

### 保留逻辑 ①：skill 文件救援（`_partition_with_skill_rescue`）

**文件**：`summarization_middleware.py:272-318`

```python
def _partition_with_skill_rescue(self, messages, cutoff_index):
    to_summarize, preserved = self._partition_messages(messages, cutoff_index)  # 先按 cutoff 分

    if self._preserve_recent_skill_count == 0 or self._preserve_recent_skill_tokens == 0 or not to_summarize:
        return to_summarize, preserved    # 救援关闭 → 直接返回默认分区
    ...
    bundles = self._find_skill_bundles(to_summarize)     # 找 skill 读取的"捆绑组"
    selected = self._select_bundles_to_rescue(bundles)   # 按预算选要救的
    # 把被救的 AIMessage 克隆（拆分 tool_calls），从 to_summarize 移到 preserved
    ...
    return remaining, rescued + preserved
```

**什么是"skill bundle"？** 看 `_find_skill_bundles`（320-380 行）：

```python
# 找这种结构：AIMessage(tool_calls=[读 skill 文件]) + 配对的 ToolMessage(skill 内容)
# 判断条件：
#   1. tool_call 的 name 在 {read_file, read, view, cat} 里
#   2. 读的路径在 /mnt/skills 下（skills_container_path）
```

**为什么要救 skill 文件？** 因为 skill 是**指令性内容**（"用这个工具时必须遵守 X 流程"）。如果被摘要压成一句话，模型就**忘记怎么用工具**了，会反复犯错。

**救援预算**（`_select_bundles_to_rescue`，382-408 行）有三个上限：

```python
for bundle in reversed(bundles):                          # 从最新的开始
    if kept >= self._preserve_recent_skill_count:         # ① 数量上限（默认 5 个）
        break
    if bundle.skill_key in seen_skill_keys:               # 去重（同一个 skill 不重复救）
        continue
    if bundle.skill_tool_tokens > self._preserve_recent_skill_tokens_per_skill:  # ② 单 skill token 上限（默认 5000）
        continue                                            # 超大的 skill 整个不救
    if total_tokens + bundle.skill_tool_tokens > self._preserve_recent_skill_tokens:  # ③ 总 token 预算（默认 25000）
        continue
    selected.append(bundle)
    total_tokens += bundle.skill_tool_tokens
    kept += 1
```

| 预算 | 默认值 | 含义 |
|---|---|---|
| `preserve_recent_skill_count` | 5 | 最多救 5 个 skill 文件 |
| `preserve_recent_skill_tokens` | 25000 | 救援总 token 不超过 25000 |
| `preserve_recent_skill_tokens_per_skill` | 5000 | 单个 skill 超过 5000 token 就不救（太大了不值得） |

**克隆技巧**（300-318 行）：被救的 AIMessage 要**克隆**——把 tool_calls 拆成"被救的"（content 清空）和"留下的"（继续被摘要）。因为一个 AIMessage 可能同时调了读 skill 和别的工具，不能整个搬走。

### 保留逻辑 ②：dynamic context reminder 保留

**文件**：`summarization_middleware.py:254-270`

```python
def _preserve_dynamic_context_reminders(self, messages_to_summarize, preserved_messages):
    reminders = [msg for msg in messages_to_summarize if is_dynamic_context_reminder(msg)]
    if not reminders:
        return messages_to_summarize, preserved_messages
    remaining = [msg for msg in messages_to_summarize if not is_dynamic_context_reminder(msg)]
    return remaining, reminders + preserved_messages   # 把 reminder 从"待摘要"挪到"保留"
```

**为什么要保留 reminder？** 因为 DynamicContextMiddleware 注入的 `<system-reminder>`（含 memory + date）是**会话级冻结**的（第 1 章 1.7.3 讲过）。如果摘要把它压掉了，DynamicContext 会误把摘要消息当成"第一条 user message"，在错误的位置重新注入。

## 4.1.3 怎么生成摘要——TAG_NOSTREAM

**文件**：`summarization_middleware.py:120-131, 141-161`

```python
# __init__ 里：构建一个打了 TAG_NOSTREAM 的 summary model
existing_tags = list((getattr(self.model, "config", None) or {}).get("tags") or [])
merged_tags = [*existing_tags, TAG_NOSTREAM] if TAG_NOSTREAM not in existing_tags else existing_tags
self._summary_model = self.model.with_config(tags=merged_tags)

def _summarize_with(self, messages_to_summarize):
    prompt = self._build_summary_prompt(messages_to_summarize)
    response = self._summary_model.invoke(prompt, config={"metadata": {"lc_source": "summarization"}})
    return response.text.strip()
```

**为什么要 `TAG_NOSTREAM`？** 因为摘要 LLM 调用是在中间件的 hook 里跑的，它的 token 流如果被 messages-tuple stream callback 捕获，会**作为幻影 AI 消息推给前端**——用户会莫名其妙看到一段摘要文字。`TAG_NOSTREAM` 告诉流式系统"别推这个调用的流"。

**`_build_summary_prompt`**（179-187 行）：先用 `self._trim_messages_for_summary`（受 `trim_tokens_to_summarize=4000` 控制）裁剪，再用 `get_buffer_string` 格式化（避免 metadata 膨胀 token），最后填进 `summary_prompt`。

## 4.1.4 怎么插回去——RemoveMessage + summary 消息

**文件**：`summarization_middleware.py:247-252`

```python
@override
def _build_new_messages(self, summary: str) -> list[HumanMessage]:
    return [HumanMessage(content=f"Here is a summary of the conversation to date:\n\n{summary}", name="summary")]
```

摘要是一条 `HumanMessage(name="summary")`——**前端隐藏**（靠 name），**模型可见**。

**新消息列表**（`_maybe_summarize` 返回的）：

```python
{
    "messages": [
        RemoveMessage(id=REMOVE_ALL_MESSAGES),   # 清空全部历史
        *new_messages,                            # 摘要（1 条）
        *preserved_messages,                      # 保留的近期消息
    ]
}
```

**`RemoveMessage(id=REMOVE_ALL_MESSAGES)`** 是 LangGraph 的特殊指令，告诉 `add_messages` reducer "清空全部"。然后 append 摘要 + 保留消息——这才是真正的压缩。

> 💡 **为什么不直接覆盖旧消息？** 因为 `add_messages` reducer 默认是 append + 按 id 替换。只 append 摘要，旧消息**还在**，根本没压。必须用 `RemoveMessage(REMOVE_ALL)` 清空，再塞回摘要 + 保留消息。

## 4.1.5 before_summarization 钩子——记忆冲洗

**文件**：`summarization_middleware.py:421-443`

```python
def _fire_hooks(self, messages_to_summarize, preserved_messages, runtime):
    if not self._before_summarization_hooks:
        return
    event = SummarizationEvent(
        messages_to_summarize=tuple(messages_to_summarize),
        preserved_messages=tuple(preserved_messages),
        thread_id=_resolve_thread_id(runtime),
        agent_name=_resolve_agent_name(runtime),
        runtime=runtime,
    )
    for hook in self._before_summarization_hooks:
        try:
            hook(event)
        except Exception:
            logger.exception(...)    # 一个 hook 失败不影响摘要
```

**关键**：钩子在 `_create_summary` **之前**触发（209-210 行）。这保证了**记忆抽取发生在原始消息被压缩之前**——第 3 章讲的 `memory_flush_hook` 就挂在这里，用 `add_nowait` 把即将被摘要的消息立即塞进记忆队列。

**钩子隔离**：一个 hook 失败不影响别的 hook，也不影响摘要本身（try/except 包裹）。

---

# 📖 4.2 DynamicContext——静态 prompt 的守护者

**文件**：`agents/middlewares/dynamic_context_middleware.py`

第 1 章 1.7.3 讲过它的注入机制（frozen-snapshot + ID-swap + midnight crossing）。这章补充它和摘要的**配合关系**。

## 4.2.1 核心矛盾：动态内容 vs prefix cache

LLM 提供商有 **prefix cache**——如果多次请求的前缀完全一致，缓存命中，**费用减半、延迟降低**。

但如果把"今天日期"写进 system prompt：

- 每天 0 点 prompt 就变 → 缓存失效；
- 不同用户记忆不同 → prompt 不同 → 缓存命中率几乎 0。

**DeerFlow 的解法**：system prompt **完全静态**（跨用户跨日期一致），动态内容（memory + date）作为隐藏 reminder 注入第一条 user msg。

## 4.2.2 冻结快照注入（回顾）

`_inject`（163-203 行）的三种情况：

| 情况 | 触发条件 | 做什么 |
|---|---|---|
| **首轮** | `last_date is None` | 找第一条 user msg，注入完整 reminder（memory+date），用 ID-swap 技巧 |
| **同一天** | `last_date == current_date` | `return None`（啥也不做） |
| **跨午夜** | `last_date != current_date` | 找最后一条 user msg，注入轻量 date-update reminder |

**ID-swap 技巧**（`_make_reminder_and_user_messages`，138-161 行）：

```python
reminder_msg = HumanMessage(
    content=reminder_content,
    id=stable_id,                    # ★ 复用原 msg 的 id → add_messages 原地替换
    additional_kwargs={"hide_from_ui": True, _DYNAMIC_CONTEXT_REMINDER_KEY: True},
)
user_msg = HumanMessage(
    content=original.content,
    id=f"{stable_id}__user",         # ★ 原 id 加后缀
    name=original.name,
)
return reminder_msg, user_msg
```

reminder **复用原 msg 的 id**，所以 LangGraph 的 `add_messages` reducer 会**原地替换**它（保持位置）。原内容变成新 msg（id 加 `__user` 后缀）追加在后面。这样第一条 reminder 一旦写入，**整个会话不再变**——prefix cache 每轮都能命中。

## 4.2.3 5 秒超时保护

**文件**：`dynamic_context_middleware.py:209-232`

```python
async def abefore_agent(self, state):
    try:
        return await asyncio.wait_for(
            asyncio.to_thread(self._inject, state),
            timeout=_INJECT_TIMEOUT_SECONDS,    # 5.0s
        )
    except TimeoutError:
        logger.warning("DynamicContextMiddleware: injection timed out (%.1fs); skipping...", _INJECT_TIMEOUT_SECONDS)
        return None    # 优雅降级：这轮不注入
```

**为什么要 5 秒超时？** 因为 `_inject` 会做两件可能阻塞的事：

1. **读记忆 JSON 文件**（sync 文件 I/O）；
2. **tiktoken BPE 编码**（首次需联网下载，可能卡几十分钟）。

如果不超时保护，一个冷启动的 tiktoken 下载会**饿死并发 HTTP handler**（issue #3402）。5 秒超时让它**优雅降级**——这轮不注入 memory，但 agent 还能跑。

## 4.2.4 和 Summarization 的配合

两个中间件有**双向耦合**：

1. **Summarization 保留 reminder**：`_preserve_dynamic_context_reminders` 把 reminder 从"待摘要"挪到"保留"（4.1.2 讲过）；
2. **DynamicContext 排除 summary 消息**：`_is_user_injection_target`（83-85 行）排除 `name == "summary"` 的消息——摘要消息不会被误当成注入目标。

这种**互相避让**保证了两者不打架。

---

# 📖 4.3 ToolOutputBudget——工具输出预算

**文件**：`agents/middlewares/tool_output_budget_middleware.py`

这是对**单次工具调用的输出做收容**，不是对话摘要。解决的问题：agent 跑 `grep -r foo .`，输出 5 万行，直接塞进上下文会撑爆。

## 4.3.1 两层策略

`_budget_content`（325-415 行）的决策树：

```
工具输出 content
   │
   ├── len(content) <= threshold(12000) 且 <= fallback_max(30000)
   │      → 不处理（None）
   │
   ├── len(content) > threshold(12000)
   │      ├── 有 sandbox 且 provider.uses_thread_data_mounts
   │      │      → _externalize（写宿主盘）→ _build_preview（头尾预览）
   │      ├── 有 sandbox（远程，非 mount）
   │      │      → _externalize_to_sandbox（写沙箱盘）→ _build_preview
   │      ├── 无 sandbox 但有 outputs_path
   │      │      → _externalize（写宿主盘）→ _build_preview
   │      └── 外置失败 / 无盘
   │             → 走兜底
   │
   └── len(content) > fallback_max(30000)
          → _build_fallback（头尾截断 + 标记）
```

### 外置（首选）：`_externalize`

**文件**：`tool_output_budget_middleware.py:120-149`

```python
def _externalize(content, *, tool_name, tool_call_id, outputs_path, storage_subdir) -> str | None:
    if os.path.isabs(storage_subdir) or ".." in storage_subdir:
        return None                              # ★ 路径安全检查
    storage_dir = os.path.join(outputs_path, storage_subdir)
    os.makedirs(storage_dir, exist_ok=True)
    filename = _build_externalized_filename(tool_name=tool_name, tool_call_id=tool_call_id)
    filepath = os.path.join(storage_dir, filename)
    if not os.path.abspath(filepath).startswith(os.path.abspath(storage_dir)):
        return None                              # ★ 二次路径逃逸检查
    with open(filepath, "w", encoding="utf-8") as f:
        f.write(content)
    return f"{_VIRTUAL_OUTPUTS_BASE}/{storage_subdir}/{filename}"   # 返回虚拟路径
```

全量输出写盘，返回**虚拟路径** `/mnt/user-data/outputs/.tool-results/xxx.log`。

**远程沙箱**用 `_externalize_to_sandbox`（152-198 行）直接写沙箱文件系统，并用 `test -s ... && echo OK` 校验可读性（拒绝给模型一个读不了的路径）。

### 预览：`_build_preview`

**文件**：`tool_output_budget_middleware.py:206-231`

```python
def _build_preview(content, *, tool_name, virtual_path, head_chars, tail_chars):
    total = len(content)
    head_end = _snap_to_line_boundary(content, min(head_chars, total))   # 头 2000 字符（按行对齐）
    tail_start = max(head_end, total - tail_chars)                       # 尾 1000 字符
    head = content[:head_end]
    tail = content[tail_start:] if tail_start < total else ""
    omitted = total - len(head) - len(tail)
    ref = f"\n\n[Full {tool_name} output saved to {virtual_path} ({total} chars, ~{total // 4} tokens). Use read_file with start_line and end_line to access specific sections. {omitted} chars omitted from this preview.]\n\n"
    parts = [head, ref]
    if tail:
        parts.append(tail)
    return "".join(parts)
```

**头 2000 + 尾 1000 字符**的预览，中间夹一条引用：告诉模型"完整输出存在哪、怎么读"。

模型看到预览后，**自己决定**用 `read_file(start_line, end_line)` 读哪段——把"读多少"的决定权交回模型。

### 兜底截断：`_build_fallback`

**文件**：`tool_output_budget_middleware.py:234-275`

无盘可用时，头 8000 + 尾 3000 字符截断，加标记：

```
[... 12345 chars omitted from bash output. Persistent storage unavailable. Consider narrowing the query or using more specific parameters.]
```

## 4.3.2 两个 hook 点

**文件**：`tool_output_budget_middleware.py:581-643`

| Hook | 时机 | 作用 |
|---|---|---|
| `wrap_tool_call` | 工具调用时 | 预算**新鲜结果**（有 sandbox + outputs_path，能外置） |
| `wrap_model_call` | 调模型前 | 预算**历史 ToolMessage**（无 sandbox，只能兜底截断） |

**为什么历史消息也要预算？** 因为外置是在工具调用时做的——但如果工具调用时没外置成功（比如当时没 sandbox），那条 ToolMessage 就带着超长内容进了历史。下次调模型前，`wrap_model_call` 会扫历史 ToolMessage，对超长的做兜底截断。

## 4.3.3 豁免工具

**文件**：`config/tool_output_config.py:55-58`

```python
exempt_tools: list[str] = Field(default=["read_file", "read_file_tool"])
```

**为什么 read_file 要豁免？** 防止死循环：

```
1. bash 输出 5 万字符 → 外置到文件 → 返回预览
2. 模型调 read_file 读那段文件 → 输出又超长 → 又外置到文件 → 又返回预览
3. 模型又调 read_file 读新文件 → ...无限循环
```

豁免 read_file，让它原样返回（read_file 本身就支持 start_line/end_line 分段读，不需要再外置）。

## 4.3.4 配置一览

**文件**：`config/tool_output_config.py`

| 字段 | 默认值 | 含义 |
|---|---|---|
| `enabled` | `True` | 开关 |
| `externalize_min_chars` | `12000` | 超过就外置（0 = 禁外置） |
| `preview_head_chars` | `2000` | 预览头 |
| `preview_tail_chars` | `1000` | 预览尾 |
| `fallback_max_chars` | `30000` | 兜底截断上限（0 = 禁截断） |
| `fallback_head_chars` | `8000` | 兜底头 |
| `fallback_tail_chars` | `3000` | 兜底尾 |
| `storage_subdir` | `.tool-results` | 外置子目录 |
| `exempt_tools` | `["read_file"]` | 豁免工具 |
| `tool_overrides` | `{}` | per-tool 阈值覆盖 |

---

# 📖 4.4 DeferredToolFilter——工具 schema 延迟加载

**文件**：`agents/middlewares/deferred_tool_filter_middleware.py`

第 1 章 merge_promoted 讲了 state 侧（promoted 字段 + catalog_hash），这章讲 middleware 侧。

## 4.4.1 问题：MCP 工具 schema 撑爆上下文

DeerFlow 接几十个 MCP 工具，每个 schema 几百 token。全发给模型，几万 token 就没了。

**解法**：默认只发工具**名**（在 system prompt 的 `<available-deferred-tools>` 里），schema 藏起来。模型要用了先调 `tool_search` 提升 schema。

## 4.4.2 两个 hook

**文件**：`deferred_tool_filter_middleware.py:76-93`

```python
@override
def wrap_model_call(self, request, handler):
    return handler(self._filter_tools(request))    # ★ 调模型前删掉隐藏工具的 schema

@override
def wrap_tool_call(self, request, handler):
    blocked = self._blocked_tool_message(request)  # ★ 模型若调隐藏工具，返回错误
    if blocked is not None:
        return blocked
    return handler(request)
```

### `_filter_tools`——删 schema

```python
def _filter_tools(self, request):
    hide = self._hidden(request.state)    # 延迟工具 - 已提升 = 仍隐藏
    if not hide:
        return request
    active = [t for t in request.tools if t.name not in hide]   # 删掉隐藏的
    return request.override(tools=active)
```

### `_blocked_tool_message`——拦调用

```python
def _blocked_tool_message(self, request):
    name = request.tool_call.get("name")
    if name not in self._hidden(request.state):
        return None
    return ToolMessage(
        content=f"Error: Tool '{name}' is deferred and has not been promoted yet. Call tool_search first...",
        status="error",
    )
```

模型如果绕过 schema 直接调延迟工具（虽然没 schema 它很难调对），返回错误 ToolMessage 让它先 search。

## 4.4.3 catalog_hash 隔离（回顾）

```python
def _promoted(self, state):
    promoted = state.get("promoted")
    if promoted and promoted.get("catalog_hash") == self._catalog_hash:   # ★ 双保险
        return set(promoted.get("names") or [])
    return set()
```

运行时读 state 时**再校验一次 hash**——即使 state 里存了旧 hash 的 promotion，也不认。这和第 1 章 merge_promoted 的 reducer 校验形成**双保险**。

---

# 📖 4.5 TokenUsageMiddleware——统计与归因

**文件**：`agents/middlewares/token_usage_middleware.py`

**只统计不强制**——硬预算在 Summarization 那里，这里只做观测。

## 4.5.1 两个职责

### ① 记 token

```python
usage = getattr(last, "usage_metadata", None)
if usage:
    logger.info("LLM token usage: input=%s output=%s total=%s%s",
                usage.get("input_tokens"), usage.get("output_tokens"),
                usage.get("total_tokens"), detail_suffix)
```

从最后一条 AIMessage 的 `usage_metadata` 读，打日志。

### ② 打步骤归因

`_build_attribution`（231-264 行）给每个 AI 步骤打 `additional_kwargs["token_usage_attribution"]`：

```python
{
    "version": 1,
    "kind": "todo_update" | "subagent_dispatch" | "tool_batch" | "final_answer" | "thinking",
    "shared_attribution": bool,      # 这一步是否多个 action 共享
    "tool_call_ids": [...],
    "actions": [...]
}
```

`_infer_step_kind`（206-217 行）判断这一步是什么类型：

| 条件 | kind |
|---|---|
| 有 tool_calls，且都是 write_todos | `todo_update` |
| 有 tool_calls，且是 task | `subagent_dispatch` |
| 有其他 tool_calls | `tool_batch` |
| 无 tool_calls，有 content | `final_answer` |
| 无 tool_calls，无 content | `thinking` |

`actions` 由 `_describe_tool_call`（135-203 行）生成，每个工具类型有不同描述：

- `write_todos` → diff 出 `todo_start`/`complete`/`update`/`remove` 动作；
- `task` → `{"kind":"subagent", ...}`；
- `web_search` → `{"kind":"search", ...}`；
- `present_files` → `{"kind":"present_files", ...}`。

**前端靠这个精确标注每一步**——"这一步在更新 todo"、"这一步在派子 agent"。

## 4.5.2 子 agent token 回滚

**文件**：`token_usage_middleware.py:281-314`

```python
# 倒着扫连续的 ToolMessage
idx = len(messages) - 2
while idx >= 0:
    tool_msg = messages[idx]
    if not isinstance(tool_msg, ToolMessage):
        break
    subagent_usage = pop_cached_subagent_usage(tool_msg.tool_call_id)  # 按 tool_call_id 弹出缓存
    if subagent_usage:
        # 往前找派发它的 AIMessage
        dispatch_idx = idx - 1
        while dispatch_idx >= 0:
            candidate = messages[dispatch_idx]
            if isinstance(candidate, AIMessage) and _has_tool_call(candidate, tool_msg.tool_call_id):
                # ★ 把子 agent token 合并进这条 AIMessage 的 usage_metadata
                merged = {
                    **prev,
                    "input_tokens": prev_input + subagent_input,
                    "output_tokens": prev_output + subagent_output,
                    "total_tokens": prev_total + subagent_total,
                }
                state_updates[dispatch_idx] = candidate.model_copy(update={"usage_metadata": merged})
                break
            dispatch_idx -= 1
    idx -= 1
```

**这段在做**：当一个 `task` 工具完成（ToolMessage 出现），把缓存的子 agent token（第 2 章讲过 `task_tool._report_subagent_usage` 缓存的）**合并回派发它的那条 AIMessage**。这样一条派发了 N 个并行子 agent 的 AIMessage，会累积所有 N 个子 agent 的 token。

---

# 🎯 现在回头检验：你掌握了吗？

### 问题 1：为什么要"静态 prompt + 动态 reminder"分离？

<details>
<summary>🔍 点击看答案</summary>

为了 **prefix cache 命中**。LLM 提供商对前缀一致的请求费用减半、延迟降低。

如果把日期/记忆写进 system prompt，每天 0 点 prompt 就变、不同用户 prompt 也不同，缓存全失效。

DeerFlow 让 system prompt **跨用户跨日期完全静态**，动态内容作为隐藏 reminder 注入第一条 user msg。第一条 reminder 一旦写入，整个会话不再变（frozen-snapshot），每轮都命中缓存。

</details>

### 问题 2：摘要为什么不直接覆盖旧消息，而要用 `RemoveMessage(REMOVE_ALL)`？

<details>
<summary>🔍 点击看答案</summary>

因为 `add_messages` reducer 默认是 **append + 按 id 替换**。只 append 摘要，旧消息**还在**，根本没压。

`RemoveMessage(id=REMOVE_ALL_MESSAGES)` 是 LangGraph 的特殊指令，告诉 reducer "清空全部"。然后 append 摘要 + 保留消息，才算真正压缩。

</details>

### 问题 3：skill 文件为什么不被摘要？这不是浪费上下文吗？

<details>
<summary>🔍 点击看答案</summary>

因为 skill 是**指令性内容**（"用这个工具时必须遵守 X 流程"）。被摘要成一句话后，模型**忘记怎么用工具**，会反复犯错。

DeerFlow 的策略：最近 5 个 skill 文件、每个最多 5000 token、总共最多 25000 token **保留**。超出才摘要。这是"**指令保全 vs 上下文成本**"的权衡。

</details>

### 问题 4：工具输出超长，为什么不直接截断，而要"外置 + 预览"？

<details>
<summary>🔍 点击看答案</summary>

因为**直接截断会丢信息**。比如 `grep -r foo .` 输出 5 万行，截断后只看到头尾，中间命中行全丢了，agent 不得不重跑。

DeerFlow 的"外置 + 预览"：全量写盘返回虚拟路径，内联只给头 2000 + 尾 1000 字符预览，模型看到预览后**自己决定**用 `read_file(start_line, end_line)` 读哪段。把"读多少"的决定权交回模型，既省 token 又不丢信息。

</details>

### 问题 5：为什么 read_file 在 ToolOutputBudget 里被豁免？

<details>
<summary>🔍 点击看答案</summary>

防止 **persist → read → persist 死循环**：

1. bash 输出超长 → 外置到文件 → 返回预览；
2. 模型调 read_file 读那段文件 → 输出又超长 → 又外置 → 又返回预览；
3. 模型又调 read_file → 无限循环。

豁免 read_file，让它原样返回（read_file 本身支持 start_line/end_line 分段读，不需要再外置）。

</details>

### 问题 6：Summarization 和 DynamicContext 怎么避免互相打架？

<details>
<summary>🔍 点击看答案</summary>

**双向避让**：

1. **Summarization 保留 reminder**：`_preserve_dynamic_context_reminders` 把 DynamicContext 注入的 `<system-reminder>` 从"待摘要"挪到"保留"——不压缩它；
2. **DynamicContext 排除 summary 消息**：`_is_user_injection_target` 排除 `name == "summary"` 的消息——摘要消息不会被误当成注入目标。

如果 Summarization 压掉了 reminder，DynamicContext 会误把摘要消息当成"第一条 user message"，在错误位置重新注入。两个中间件**互相感知**对方的消息类型，才能和平共处。

</details>

---

# 📝 一页纸总结（第 4 章精华）

```
┌──────────────────────────────────────────────────────────────┐
│  上下文窗口管理 = 5 个中间件各管一件事                          │
└──────────────────────────────────────────────────────────────┘
        │
        ├─▶ DynamicContext：静态 prompt + 动态 reminder 分离
        │     • frozen-snapshot 注入第一条 user msg（prefix cache）
        │     • ID-swap 技巧 + hide_from_ui + 5s 超时降级
        │
        ├─▶ Summarization：历史消息压缩
        │     • trigger（messages/tokens/fraction，OR 语义）
        │     • skill 救援（5 个 / 25K token / 5K per-skill）
        │     • reminder 保留（不压 dynamic context）
        │     • RemoveMessage(REMOVE_ALL) + summary HumanMessage
        │     • TAG_NOSTREAM 防幻影消息
        │     • before_summarization 钩子（记忆冲洗）
        │
        ├─▶ ToolOutputBudget：工具输出收容
        │     • 外置（threshold=12000）→ 头尾预览 + 虚拟路径
        │     • 兜底截断（max=30000）→ 头尾 + 标记
        │     • 豁免 read_file（防死循环）
        │     • wrap_tool_call（新鲜结果）+ wrap_model_call（历史）
        │
        ├─▶ DeferredToolFilter：MCP 工具 schema 延迟加载
        │     • 默认藏 schema，tool_search 提升后才暴露
        │     • catalog_hash 双保险（reducer + 运行时各校验一次）
        │     • wrap_model_call 删 schema + wrap_tool_call 拦调用
        │
        └─▶ TokenUsage：统计与归因
              • 记 token（log usage_metadata）
              • 打 attribution（kind: todo/subagent/tool_batch/final/thinking）
              • 子 agent token 回滚（合并回派发 AIMessage）
```

**5 句话记住**：

1. **静态 prompt + 动态 reminder 分离**——为 prefix cache，动态内容注入第一条 user msg 且冻结；
2. **摘要是 RemoveMessage(REMOVE_ALL) + summary + preserved**——不是简单覆盖，且救下 skill 文件和 reminder；
3. **工具输出外置 + 预览**——全量写盘给虚拟路径，内联只给头尾，模型按需 read_file；
4. **MCP 工具 schema 延迟加载**——默认藏 schema，tool_search 提升后才暴露，catalog_hash 防名字复用；
5. **TokenUsage 只观测不强制**——记 token + 打 attribution + 子 agent token 回滚到派发消息。
