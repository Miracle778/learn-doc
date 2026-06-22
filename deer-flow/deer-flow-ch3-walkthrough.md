# 第 3 章 精讲版：Agent 的记忆（Long-Term Memory）是怎么做的

> 这一章讲 DeerFlow 最"反直觉"的子系统——**长期记忆没有向量数据库**。
>
> **阅读方法**：和前几章一样，逐段过代码。
>
> **配套文件**（建议同时打开）：
> - `agents/memory/storage.py`（存哪）
> - `agents/memory/queue.py`（怎么排队写）
> - `agents/memory/updater.py`（怎么用 LLM 抽取）
> - `agents/memory/message_processing.py`（怎么检测纠错/强化）
> - `agents/memory/prompt.py`（抽取 prompt + 注入格式化）
> - `agents/memory/summarization_hook.py`（摘要前冲洗）
> - `agents/middlewares/memory_middleware.py`（触发写入）
> - `agents/middlewares/dynamic_context_middleware.py`（注入读取）
> - `config/memory_config.py`（配置）

---

# 🎯 先建立心理模型：没有向量库，靠什么"记住"用户？

很多人一听到"长期记忆"就想到：embedding + 向量库 + 相似度检索。DeerFlow **不用这套**。

我 grep 了整个 `deerflow` 包，`embedding|vector|chroma|faiss|pinecone` **零匹配**。

DeerFlow 的方案是：

```
写入：对话结束后，用 LLM 把对话"提炼"成结构化 JSON（摘要 + 置信度排序的事实）
存储：存成一个 JSON 文件（按 user_id + agent_name 隔离）
读取：每次对话前，把整个 JSON 加载、格式化、塞进上下文（按置信度贪心打包到 token 预算）
```

**核心思想**：

1. **记忆量级有限**（单用户上限 100 条 fact + 几段摘要），全量加载也就几 KB；
2. **"找相关记忆"交给 LLM 自己**——它读完 `<memory>` 段，自己决定哪些 fact 与当前问题相关；
3. **不用维护额外的 embedding 模型和向量库**，运维成本极低。

**代价**：记忆量级上去（比如 1000+ facts）后，全量加载会撑爆 token 预算。那时才需要考虑分层摘要或检索。**先简单后复杂**——这是好的工程判断。

---

# 📖 3.1 记什么：结构化 JSON Schema

**文件**：`agents/memory/storage.py:24-40`，`create_empty_memory()`：

```python
def create_empty_memory() -> dict[str, Any]:
    return {
        "version": "1.0",
        "lastUpdated": utc_now_iso_z(),
        "user": {
            "workContext":      {"summary": "", "updatedAt": ""},  # 工作背景
            "personalContext":  {"summary": "", "updatedAt": ""},  # 个人背景
            "topOfMind":        {"summary": "", "updatedAt": ""},  # 当前关注（更新最频繁）
        },
        "history": {
            "recentMonths":        {"summary": "", "updatedAt": ""},  # 近 1-3 月
            "earlierContext":      {"summary": "", "updatedAt": ""},  # 3-12 月
            "longTermBackground":  {"summary": "", "updatedAt": ""},  # 基础/持久
        },
        "facts": [],   # 离散事实数组
    }
```

存三类信息：

1. **结构化摘要**（字符串字段）：`user.*` 是当前状态，`history.*` 是时间分层；
2. **离散事实数组**，每条形如（`updater.py:658-665`）：

```python
{
    "id": "fact_xxxx",
    "content": "用户偏好用 TypeScript 而非 JavaScript",
    "category": "preference",   # preference|knowledge|context|behavior|goal|correction
    "confidence": 0.85,
    "createdAt": "2026-06-14T...",
    "source": "thread_abc123",  # 来自哪个会话
}
```

`correction` 类还会带可选的 `sourceError`（"之前的错误做法"）。

**没有实体抽取、没有知识图谱**——纯叙事摘要 + 置信度排序的事实列表。

---

# 📖 3.2 存哪里：可插拔的 JSON 文件存储

**文件**：`agents/memory/storage.py`

## 3.2.1 抽象 + 默认实现

- **抽象**：`MemoryStorage` ABC（storage.py:43-59），三个方法 `load / reload / save`；
- **默认实现**：`FileMemoryStorage`（storage.py:62-189）；
- **可插拔**：`MemoryConfig.storage_class` 默认 `"deerflow.agents.memory.storage.FileMemoryStorage"`，通过 `importlib` 动态加载（storage.py:196-231）。

## 3.2.2 路径解析（按 user_id + agent_name 隔离）

**storage.py:84-102**，`_get_memory_file_path` 的优先级：

1. `user_id` + `agent_name` → `get_paths().user_agent_memory_file(user_id, agent_name)`
2. `user_id` only → `config.storage_path`（如设了绝对路径）或 `get_paths().user_memory_file(user_id)`
3. `agent_name` only（legacy）→ `get_paths().agent_memory_file(agent_name)`
4. 都没有 → `config.storage_path` 或 `get_paths().memory_file`

**注意**：`storage_path` 如果设了**绝对路径**，就**跳过 per-user 隔离**（所有用户共享一个文件）——这是给"单用户部署"的便捷选项，多用户环境别用。

## 3.2.3 原子写（防写坏）

**storage.py:160-189**，`save` 的核心：

```python
def save(self, memory_data, agent_name=None, *, user_id=None) -> bool:
    file_path = self._get_memory_file_path(agent_name, user_id=user_id)
    file_path.parent.mkdir(parents=True, exist_ok=True)
    memory_data = {**memory_data, "lastUpdated": utc_now_iso_z()}   # 浅拷贝 + 加时间戳
    temp_path = file_path.with_suffix(f".{uuid.uuid4().hex}.tmp")
    with open(temp_path, "w", encoding="utf-8") as f:
        json.dump(memory_data, f, indent=2, ensure_ascii=False)
    temp_path.replace(file_path)          # ★ POSIX 原子重命名
    ...
```

**为什么先写临时文件再 rename？** 因为 `json.dump` 写到一半如果进程崩了，原文件就坏了。先写 `.{uuid}.tmp`，写完再 `replace`（原子操作），要么成功要么没动——不会留下半个文件。

**浅拷贝 + 加时间戳**：`{**memory_data, "lastUpdated": ...}` 创建新 dict，不改调用方的对象。这样即使 `save` 失败，调用方持有的 memory_data 还是干净的。

## 3.2.4 mtime 缓存（避免每次都读盘）

**storage.py:123-143**，`load` 的核心：

```python
def load(self, agent_name=None, *, user_id=None) -> dict[str, Any]:
    file_path = self._get_memory_file_path(...)
    current_mtime = file_path.stat().st_mtime if file_path.exists() else None
    cache_key = self._cache_key(user_id, agent_name)

    with self._cache_lock:
        cached = self._memory_cache.get(cache_key)
        if cached and cached[1] == current_mtime:   # ★ mtime 没变 → 用缓存
            return cached[0]
        # mtime 变了或没缓存 → 重新读盘
        memory_data = self._load_memory_from_file(file_path)
        self._memory_cache[cache_key] = (memory_data, current_mtime)
        return memory_data
```

**缓存键** = `(user_id, agent_name)`。**失效条件** = 文件的 `mtime` 变了。

**为什么用 mtime 而不是内容 hash？** 因为 `stat()` 是 O(1) 系统调用，而算 hash 要读整个文件。记忆文件很小（几 KB），但每轮对话都要 `load` 一次，频繁读盘 + 算 hash 浪费。mtime 检查几乎零成本。

`reload`（storage.py:145-158）强制失效缓存——用于外部修改后强制重读。

---

# 📖 3.3 怎么写：去抖队列 + LLM 抽取 + JSON 合并

这是记忆子系统的核心管道，分三个组件。

## 3.3.1 Queue：去抖 + 单例

**文件**：`agents/memory/queue.py`。`MemoryUpdateQueue` 是单例（`get_memory_queue()`，queue.py:265-275），内置**去抖定时器**。

### 去抖 key 和 flag 合并

**queue.py:43-50**，`_queue_key`：

```python
@staticmethod
def _queue_key(ctx: ConversationContext) -> tuple:
    return (ctx.thread_id, ctx.user_id, ctx.agent_name)
```

**queue.py:117-144**，`_enqueue_locked`：

```python
def _enqueue_locked(self, ctx):
    key = self._queue_key(ctx)
    for i, existing in enumerate(self._queue):
        if self._queue_key(existing) == key:
            # ★ 合并 flag（OR 语义），永不丢失信号
            ctx.correction_detected = ctx.correction_detected or existing.correction_detected
            ctx.reinforcement_detected = ctx.reinforcement_detected or existing.reinforcement_detected
            self._queue.pop(i)
            break
    self._queue.append(ctx)
```

**关键设计**：同一 key 的新对话**替换**旧的（messages 不拼接，用最新的），但 `correction_detected` / `reinforcement_detected` 标志用 **OR 合并**——一旦检测到纠错信号，即使被去抖覆盖了也**永不丢失**。

### add vs add_nowait

| 方法 | 延迟 | 用在哪 |
|---|---|---|
| `add()`（queue.py:52-88） | `config.debounce_seconds`（默认 30s） | `MemoryMiddleware.after_agent`——每轮对话后 |
| `add_nowait()`（queue.py:90-115） | 0（立即） | `memory_flush_hook`——摘要前冲洗 |

**为什么两种？** 普通对话不需要立即更新（30s 内可能还有新对话，去抖合并省 LLM 调用）。但摘要前冲洗**必须立即**——因为摘要马上要删掉原始消息，再不抽取就晚了。

### ContextVar 陷阱

**queue.py:67-71** 的 docstring 明确警告：

```python
user_id: The user ID captured at enqueue time. Stored in ConversationContext so it
    survives the threading.Timer boundary (ContextVar does not propagate across
    raw threads).
```

`threading.Timer` 在另一个线程触发，**ContextVar 不会自动传播过去**。所以 `user_id` 在入队时就被显式捕获存进 `ConversationContext`，而不是在 timer 回调里再读。

> 💡 **学习启示**：跨线程传 ContextVar，必须在创建任务时**显式捕获**存进数据结构。这是个容易踩的坑（第 2 章的 SubagentExecutor 也用 `copy_context()` 解决类似问题）。

## 3.3.2 Updater：LLM 驱动的抽取引擎

**文件**：`agents/memory/updater.py`。`MemoryUpdater`（updater.py:380-684）是核心。

### 为什么"故意用同步 invoke"？

**updater.py:503-509** 的 docstring：

```python
"""Pure-sync memory update using ``model.invoke()``.

Uses the *sync* LLM call path so no event loop is created.  This
guarantees that the langchain provider's globally cached async
httpx ``AsyncClient`` / connection pool (the one shared with the
lead agent) is never touched — no cross-loop connection reuse is
possible.
"""
```

翻译：用同步 `model.invoke()`，**不碰** langchain 的 async httpx 连接池（那个池和 lead agent 共享）。如果用 async，会触发跨事件循环的连接复用 bug（issue #2615）。

**updater.py:539-598**，`update_memory` 的处理：

```python
def update_memory(self, ...):
    try:
        asyncio.get_running_loop()
        # 有事件循环在跑 → 卸载到独立线程池（同步 invoke，不碰 async 池）
        future = _SYNC_MEMORY_UPDATER_EXECUTOR.submit(self._do_update_memory_sync, ...)
        return future.result()
    except RuntimeError:
        # 没事件循环 → 直接同步调
        return self._do_update_memory_sync(...)
```

**`_SYNC_MEMORY_UPDATER_EXECUTOR`**（updater.py:34-38）是 4 worker 的 `ThreadPoolExecutor`，atexit 注册关闭。

> 💡 **反直觉但重要的经验**：**async 不一定比 sync 好，关键看资源竞争**。这里 sync + 线程池反而避免了 async 连接池冲突。

### `_apply_updates`：合并逻辑（最关键）

**updater.py:600-684**。三个动作：

#### ① 更新摘要字段（仅当 shouldUpdate=true）

```python
# User sections
for section_key in ("workContext", "personalContext", "topOfMind"):
    section = update_data["user"].get(section_key, {})
    if section.get("shouldUpdate") and section.get("summary"):
        current_memory["user"][section_key] = {
            "summary": section["summary"],
            "updatedAt": now,
        }
```

只有 LLM 在 JSON 里标了 `shouldUpdate=true` 才更新——避免无意义的覆盖。

#### ② 删除 facts（按 id）

```python
facts_to_remove = set(update_data.get("factsToRemove", []))
current_memory["facts"] = [f for f in current_memory["facts"] if f["id"] not in facts_to_remove]
```

#### ③ 添加新 facts（置信度阈值 + 去重 + 上限驱逐）

```python
for fact in new_facts:
    confidence = fact.get("confidence", 0.5)
    if confidence >= config.fact_confidence_threshold:   # ★ 默认 0.7
        fact_key = _fact_content_key(normalized_content)  # casefold 去重
        if fact_key in existing_fact_keys:
            continue
        fact_entry = {"id": f"fact_{uuid.uuid4().hex[:8]}", ...}
        current_memory["facts"].append(fact_entry)

# ★ 超过上限 → 按置信度降序保留 top N
if len(current_memory["facts"]) > config.max_facts:   # 默认 100
    current_memory["facts"] = sorted(
        current_memory["facts"],
        key=lambda f: f.get("confidence", 0),
        reverse=True,
    )[: config.max_facts]
```

**三个安全机制**：

1. **置信度阈值**（`fact_confidence_threshold=0.7`）：低置信度的 fact 不存——避免噪音；
2. **casefold 去重**（`_fact_content_key`，updater.py:371-377）：`content.strip().casefold()`——大小写不敏感去重，防止"用户喜欢 TS"和"用户喜欢 ts"存两条；
3. **max_facts 驱逐**：满了按置信度降序保留 top 100——**按置信度驱逐**，低质量的先被丢掉。

### 上传洗刷（防"幽灵文件"）

**updater.py:337-368**，`_strip_upload_mentions_from_memory`：

```python
_UPLOAD_SENTENCE_RE = re.compile(
    r"[^.!?]*\b(?:"
    r"upload(?:ed|ing)?(?:\s+\w+){0,3}\s+(?:file|files?|document|...)"
    r"|file\s+upload"
    r"|/mnt/user-data/uploads/"
    r"|<uploaded_files>"
    r")[^.!?]*[.!?]?\s*"
)
```

它从摘要和 facts 里**洗掉上传相关的句子**。原因（docstring 350-353）：

> uploaded files are session-scoped; persisting upload events makes the agent search for non-existent files later.

上传文件是**会话级**的，下次会话文件可能已经不在了。如果把"用户上传了 data.csv"记进 memory，下次会话 agent 会去找 data.csv——找不到，困惑。所以必须洗掉。

## 3.3.3 信号检测：识别"纠错"和"强化"

**文件**：`agents/memory/message_processing.py`

### `detect_correction`（message_processing.py:88-97）

扫最近 6 条消息（`messages[-6:]`），匹配中英文纠错模式：

```python
_CORRECTION_PATTERNS = (
    re.compile(r"\bthat(?:'s| is) (?:wrong|incorrect)\b", re.IGNORECASE),
    re.compile(r"\byou misunderstood\b", re.IGNORECASE),
    re.compile(r"\btry again\b", re.IGNORECASE),
    re.compile(r"\bredo\b", re.IGNORECASE),
    re.compile("不对"),
    re.compile("你理解错了"),
    re.compile("你理解有误"),
    re.compile("重试"),
    re.compile("重新来"),
    re.compile("换一种"),
    re.compile("改用"),
)
```

### `detect_reinforcement`（message_processing.py:100-109）

同样的窗口，匹配强化模式：

```python
_REINFORCEMENT_PATTERNS = (
    re.compile(r"\byes,? (exactly|perfect|that's right|...)\b", re.IGNORECASE),
    re.compile(r"\bperfect[.!?]|$"),
    re.compile(r"\bthat's (exactly)? (right|correct|what I wanted)\b", re.IGNORECASE),
    re.compile(r"\bkeep (doing )?that\b", re.IGNORECASE),
    re.compile("对,就是这样"),
    re.compile("完全正确"),
    re.compile("继续保持"),
    ...
)
```

**关键规则**：`reinforcement_detected = not correction_detected and detect_reinforcement(...)`——**纠错优先**，检测到纠错就不检测强化（两个信号互斥）。

这些 flag 变成 prompt 提示（`_build_correction_hint`，updater.py:397-420）：

- 纠错 → "record the correct approach as category 'correction', confidence >= 0.95"
- 强化 → "record as category 'preference' or 'behavior', confidence >= 0.9"

---

# 📖 3.4 怎么读：全量加载 + 置信度排序 + 贪心 token 预算

**文件**：`agents/memory/prompt.py:319-439`，`format_memory_for_injection`。

## 3.4.1 整体流程

1. 空 memory → 返回 `""`；
2. 构建 User Context / History 段；
3. **facts 按 confidence 降序**（prompt.py:380-384）；
4. **贪心 token 预算打包**（prompt.py:388-419）；
5. 还超预算就按字符比例截断到 95%。

## 3.4.2 贪心 token 预算打包（核心）

```python
# 算一次基础 token（User Context + History 段）
base_tokens = _count_tokens("\n".join(lines))
running_tokens = base_tokens

for fact in sorted_facts:
    line = f"- [{category} | {confidence:.2f}] {content}"
    if category == "correction" and sourceError:
        line += f" (avoid: {sourceError})"   # ★ 纠错类特殊格式
    
    line_tokens = _count_tokens("\n" + line)
    if running_tokens + line_tokens <= max_tokens:   # ★ 还装得下
        lines.append(line)
        running_tokens += line_tokens
    else:
        break                                          # 装不下就停
```

**本质是"按置信度的贪心背包"**——高置信度的 fact 优先装进 token 预算。`max_tokens` 默认 2000（`MemoryConfig.max_injection_tokens`）。

## 3.4.3 token 计数双模式

**prompt.py:263-289**，`_count_tokens`：

```python
def _count_tokens(text, encoding_name="cl100k_base", *, use_tiktoken=True):
    if not use_tiktoken:
        return _char_based_token_estimate(text)       # char 模式
    encoding = _get_tiktoken_encoding(encoding_name)
    if encoding is None:
        return _char_based_token_estimate(text)       # tiktoken 加载失败 → fallback
    try:
        return len(encoding.encode(text))
    except Exception:
        return _char_based_token_estimate(text)
```

| 模式 | 配置 | 特点 |
|---|---|---|
| `tiktoken`（默认） | `token_counting: "tiktoken"` | 准确，但首次需联网下载 BPE 数据 |
| `char` | `token_counting: "char"` | 无需联网，CJK 感知（CJK 算 2 char/token，拉丁算 4 char/token） |

**tiktoken 的失败保护**（prompt.py:196-240）：`_get_tiktoken_encoding` 有 600s 冷却缓存（`_TIKTOKEN_RETRY_COOLDOWN_S`），防止 BPE 下载失败时雪崩重试。

---

# 📖 3.5 怎么注入：DynamicContextMiddleware（不在 MemoryMiddleware！）

> ⚠️ **最容易搞混的点**：`MemoryMiddleware` **不负责注入**，它只在 `after_agent` 触发更新入队。**真正注入记忆的是 `DynamicContextMiddleware`**。

## 3.5.1 为什么分离？

| 中间件 | 干什么 | hook |
|---|---|---|
| `MemoryMiddleware` | **写**：对话后触发记忆更新入队 | `after_agent` |
| `DynamicContextMiddleware` | **读**：对话前注入记忆 + 日期 | `before_agent` |

**读写分离**——写入是异步的（去抖队列），读取是同步的（每轮注入）。分开中间件让职责清晰。

## 3.5.2 "冻结快照"注入模式

**文件**：`agents/middlewares/dynamic_context_middleware.py`

**目的**（模块 docstring，1-27 行）：让 system prompt **完全静态**（prefix cache 友好），把每用户/每轮变化的数据（记忆 + 日期）作为隐藏的 `<system-reminder>` 注入。

### 首轮注入（ID 替换技巧）

**dynamic_context_middleware.py:138-161**，`_make_reminder_and_user_messages`：

```python
@staticmethod
def _make_reminder_and_user_messages(original, reminder_content):
    stable_id = original.id or str(uuid.uuid4())
    reminder_msg = HumanMessage(
        content=reminder_content,
        id=stable_id,                          # ★ 复用原 msg 的 ID
        additional_kwargs={"hide_from_ui": True, _DYNAMIC_CONTEXT_REMINDER_KEY: True},
    )
    user_msg = HumanMessage(
        content=original.content,
        id=f"{stable_id}__user",               # ★ 原 ID 加后缀
        name=original.name,
        additional_kwargs=original.additional_kwargs,
    )
    return reminder_msg, user_msg
```

**为什么要 ID 替换？** 因为 LangGraph 的 `add_messages` reducer 按 ID 去重——如果新消息的 ID 和旧消息一样，就**原地替换**。所以 `reminder_msg` 复用原 user msg 的 ID，替换它；`user_msg` 用 `{id}__user` 作为新 ID 追加。

**效果**：reminder 插在 user msg 前面，且**第一条消息整个会话冻结**——内容永不再变，所以每轮都能命中 prefix cache。

### 同一天 vs 跨午夜

**dynamic_context_middleware.py:163-203**，`_inject`：

- `last_date is None`（首轮）→ 注入完整 reminder（memory + date）；
- `last_date == current_date`（同一天）→ `return None`，啥也不做；
- `last_date != current_date`（跨午夜）→ 注入轻量的"日期更新"reminder（只有 date，没有 memory）。

### 5 秒超时保护

**dynamic_context_middleware.py:209-232**，`abefore_agent`：

```python
try:
    return await asyncio.wait_for(
        asyncio.to_thread(self._inject, state),
        timeout=_INJECT_TIMEOUT_SECONDS,   # 5.0s
    )
except TimeoutError:
    logger.warning("injection timed out; skipping memory/date injection for this turn")
    return None   # 优雅降级：这轮不注入记忆
```

**为什么 5 秒？** 因为 `_inject` 会读记忆文件 + 可能触发 tiktoken BPE 冷下载（OS TCP 超时约 26 分钟！）。如果 gateway 预热没预热 tiktoken，第一次注入会卡 26 分钟——前端全挂。5 秒超时强制降级。

---

# 📖 3.6 两条写入路径

记忆更新有**两条入口**，都异步：

## 3.6.1 主路径：MemoryMiddleware.after_agent

**文件**：`agents/middlewares/memory_middleware.py:52-110`

```python
def after_agent(self, state, runtime: Runtime) -> dict | None:
    config = self._memory_config or get_memory_config()
    if not config.enabled:
        return None
    ...
    filtered_messages = filter_messages_for_memory(messages)
    correction_detected = detect_correction(filtered_messages)
    reinforcement_detected = not correction_detected and detect_reinforcement(filtered_messages)
    user_id = get_effective_user_id()    # ★ 入队时捕获（防 ContextVar 跨线程丢失）
    queue.add(                            # ★ 去抖 add（30s）
        thread_id=..., messages=filtered_messages,
        agent_name=self._agent_name, user_id=user_id,
        correction_detected=..., reinforcement_detected=...,
    )
    return None    # ★ 不改 state，不注入
```

**注意它返回 `None`**——不改 state，不注入。它**唯一的作用**就是入队。

## 3.6.2 摘要前冲洗：memory_flush_hook

**文件**：`agents/memory/summarization_hook.py:12-34`

```python
def memory_flush_hook(event: SummarizationEvent) -> None:
    if not get_memory_config().enabled or not event.thread_id:
        return
    filtered_messages = filter_messages_for_memory(list(event.messages_to_summarize))
    ...
    queue.add_nowait(   # ★ 立即 add（delay 0）
        thread_id=event.thread_id, messages=filtered_messages,
        ...
    )
```

这是个 `BeforeSummarizationHook`，在 `DeerFlowSummarizationMiddleware` **删消息之前**触发（summarization_middleware.py:209-210 的 `_fire_hooks`）。

**为什么必须在摘要前？** 因为摘要是**不可逆压缩**——多轮对话被压成一句话。如果先摘要再抽取记忆，LLM 看到的是压缩后的摘要，**抽不出细节事实**（比如"用户提到了项目代号 X"）。所以要在压缩前先冲洗进记忆队列。

---

# 📖 3.7 配置一览

**文件**：`config/memory_config.py:8-75`

| 字段 | 默认值 | 含义 |
|---|---|---|
| `enabled` | `True` | 是否启用记忆 |
| `storage_path` | `""` | 存储路径（空 = per-user 隔离） |
| `storage_class` | `"...FileMemoryStorage"` | 存储实现类（可插拔） |
| `debounce_seconds` | `30` | 去抖延迟（1-300） |
| `model_name` | `None` | 抽取用的模型（None = 默认模型） |
| `max_facts` | `100` | 最大 fact 数（10-500） |
| `fact_confidence_threshold` | `0.7` | fact 置信度阈值（0-1） |
| `injection_enabled` | `True` | 是否注入记忆到上下文 |
| `max_injection_tokens` | `2000` | 注入的 token 预算（100-8000） |
| `token_counting` | `"tiktoken"` | token 计数模式（tiktoken/char） |

---

# 🎯 现在回头检验：你掌握了吗？

### 问题 1：为什么不用向量数据库做语义检索？

<details>
<summary>🔍 点击看答案</summary>

DeerFlow 的取舍是「**记忆量级有限 + 检索精度靠 LLM 自己**」：

- 单用户记忆上限 `max_facts=100` + 几段摘要，全量加载几 KB；
- 这个量级下，**按置信度排序 + 贪心 token 打包**比 embedding 检索更简单、更可控、更便宜（无需 embedding 模型和向量库运维）；
- "找相关记忆"交给 LLM——它读完 `<memory>` 段自己决定哪些 fact 相关。

**代价**：记忆量上去（1000+ facts）后会失效。那时再考虑分层摘要或检索。**先简单后复杂**。

</details>

### 问题 2：用户说"你上次理解错了"，agent 怎么记住这个纠错？

<details>
<summary>🔍 点击看答案</summary>

完整链路：

1. 用户发"不对，你理解错了" → `MemoryMiddleware.after_agent` 触发；
2. `detect_correction()` 命中，`correction_detected=True`；
3. 30s 后 `MemoryUpdater` 启动，`_build_correction_hint()` 在 prompt 里加："record with category='correction', confidence>=0.95"；
4. LLM 抽取出 `{category:"correction", content:"...", confidence:0.95, sourceError:"之前的错误做法"}`；
5. `confidence=0.95 >= 0.7`，写入 facts；
6. 下次注入时，correction 类有特殊格式：`- [correction | 0.95] 不要用 X (avoid: X 的描述)`。

</details>

### 问题 3：记忆更新为什么"故意用同步 invoke"而不是 ainvoke？

<details>
<summary>🔍 点击看答案</summary>

因为 LangChain 的 async 客户端用了一个**全局共享的 httpx 连接池**。如果记忆更新和 lead agent 在同一个事件循环里 async 跑，两者会抢连接池，触发跨循环连接复用 bug（issue #2615）。

解法：**同步 `invoke` + 把整个调用卸载到独立线程池**（`_SYNC_MEMORY_UPDATER_EXECUTOR`）。记忆更新跑在自己的线程里，用自己的同步 httpx 客户端，完全不碰 async 池。

**反直觉但重要**：async 不一定比 sync 好，关键看资源竞争。

</details>

### 问题 4：`memory_flush_hook` 为什么必须在摘要前触发？

<details>
<summary>🔍 点击看答案</summary>

因为摘要是**不可逆压缩**——多轮原始对话被压成一句话。如果先摘要再抽取记忆，LLM 看到的是压缩后的摘要，**抽不出细节事实**。

所以在 `DeerFlowSummarizationMiddleware` 删消息之前（`_fire_hooks`），先 `add_nowait()` 把原始消息塞进记忆队列——保证 LLM 抽取时看到完整原始对话。

**数据保全**：压缩不可逆，所以要在压缩前做信息抽取。

</details>

### 问题 5：为什么"注入记忆"和"触发记忆更新"是两个不同的中间件？

<details>
<summary>🔍 点击看答案</summary>

**读写分离**：

- `MemoryMiddleware`（`after_agent`）负责**写**——对话后触发记忆更新入队（异步去抖）；
- `DynamicContextMiddleware`（`before_agent`）负责**读**——对话前注入记忆 + 日期到上下文。

分开的原因：写入是异步的（去抖队列，30s 后批量处理），读取是同步的（每轮都要注入）。职责不同，时序不同，放一个中间件里会耦合。分开后各自清晰。

</details>

---

# 📝 一页纸总结（第 3 章精华）

```
┌──────────────────────────────────────────────────────────────┐
│  DeerFlow 长期记忆：无向量库，靠 LLM 抽取 + 置信度排序         │
└──────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼──────────────────────┐
        ▼                     ▼                      ▼
    写入路径                 存储                    读取路径
    MemoryMiddleware         FileMemoryStorage       DynamicContextMiddleware
    (after_agent)            (JSON 文件)             (before_agent)
        │                     │                      │
    detect_correction/       原子写(tmp+rename)      format_memory_for_injection
    reinforcement            mtime 缓存              (置信度排序+贪心token打包)
        │                     │                      │
    queue.add (去抖30s)       per-user 隔离          冻结快照注入
        │                     │                      │ (ID替换+hide_from_ui)
    memory_flush_hook         可插拔(ABC+importlib)   │
    (摘要前 add_nowait)                              5s 超时降级
        │                                            │
    MemoryUpdater                                  <system-reminder>
    (LLM 抽取)                                     <memory>...</memory>
        │                                          <current_date>...</current_date>
    _apply_updates:                                </system-reminder>
    • 置信度阈值 0.7
    • casefold 去重
    • max_facts=100 驱逐
    • 上传洗刷
```

**5 句话记住**：

1. **没有向量库**——全量加载 + 置信度排序 + 贪心 token 预算打包，LLM 自己找相关记忆；
2. **写入是去抖队列**——`MemoryMiddleware.after_agent` 入队（30s 去抖），`memory_flush_hook` 摘要前立即冲洗；
3. **LLM 抽取 + 三重安全**——置信度阈值 0.7 + casefold 去重 + max_facts 按置信度驱逐 + 上传洗刷；
4. **故意用同步 invoke**——避开 async httpx 连接池冲突，卸载到独立线程池；
5. **注入靠 DynamicContextMiddleware**（不是 MemoryMiddleware）——冻结快照 + ID 替换 + 5s 超时降级，为 prefix cache 保持 system prompt 静态。
