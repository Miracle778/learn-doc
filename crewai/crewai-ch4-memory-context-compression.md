# 第 4 章：记忆、上下文、压缩

> 这一章很重要。agent 应用开发里最容易混淆三件事：memory、task context、context window compression。
>
> 它们不是一回事。

## 4.1 三个概念先分开

```text
task context
  上游 task 的输出，传给当前 task。

memory
  跨任务、跨运行保存的可检索事实，由 Memory 管理。

context window compression
  LLM 报上下文超限时，对 executor.messages 做总结压缩。
```

一句话：

- task context 是“这次工作流里前面步骤的结果”。
- memory 是“以前保存过、现在可召回的经验和事实”。
- compression 是“当前 LLM 对话太长了，先总结再继续跑”。

## 4.2 Memory 是统一记忆系统

核心类：

`lib/crewai/src/crewai/memory/unified_memory.py:76`

它的 docstring 已经说得很清楚：

```text
Unified memory:
- LLM analyzed
- adaptive-depth recall
- scope/slice views
- pluggable storage
```

关键字段：

- `llm`：用于分析保存内容和复杂查询。
- `storage`：默认 `lancedb`。
- `embedder`：用于向量搜索。
- `recency_weight`
- `semantic_weight`
- `importance_weight`
- `root_scope`

位置：`lib/crewai/src/crewai/memory/unified_memory.py:88`

## 4.3 Crew 开启 memory 时发生什么

`Crew.memory` 字段支持：

```text
False / None
True
Memory
MemoryScope
MemorySlice
```

字段位置：`lib/crewai/src/crewai/crew.py:227`

当 `memory=True` 时，`Crew.create_crew_memory()` 会创建一个 `Memory` 实例，并设置 root scope：

```text
/crew/{crew_name}
```

位置：`lib/crewai/src/crewai/crew.py:640`

这意味着：一个 crew 的记忆默认会被归到自己的命名空间下，不是所有 crew 混在根目录里。

## 4.4 Memory 保存：remember 与 remember_many

单条保存：

`Memory.remember()`：`lib/crewai/src/crewai/memory/unified_memory.py:430`

它是同步的，会等待保存完成并返回 `MemoryRecord`。

多条保存：

`Memory.remember_many()`：`lib/crewai/src/crewai/memory/unified_memory.py:523`

它是非阻塞的，把编码和保存丢到后台线程池里，后续 `recall()` 会先 `drain_writes()`，确保读取前写入完成。

读屏障：

`lib/crewai/src/crewai/memory/unified_memory.py:711`

## 4.5 保存前会做 LLM 分析

LLM 分析 schema 在：

`lib/crewai/src/crewai/memory/analyze.py:37`

`MemoryAnalysis` 会产出：

- `suggested_scope`
- `categories`
- `importance`
- `extracted_metadata`

这说明 CrewAI 的 memory 不是单纯“向量库存字符串”。它会先让 LLM 给记忆打结构化标签，再进入存储。

如果你想把长文本拆成原子记忆，用：

`extract_memories_from_content()`：`lib/crewai/src/crewai/memory/analyze.py:155`

## 4.6 Memory 召回：shallow 与 deep

召回入口：

`Memory.recall()`：`lib/crewai/src/crewai/memory/unified_memory.py:681`

两种 depth：

```text
shallow
  直接 embed query，搜索 storage，按 composite score 排序。

deep
  启动 RecallFlow：
  - LLM 分析长查询
  - 拆 recall_queries
  - 选择 candidate scopes
  - 多 query、多 scope 并行搜索
  - 根据 confidence 决定是否 deeper exploration
  - 去重、综合评分、排序
```

源码：

- shallow 分支：`lib/crewai/src/crewai/memory/unified_memory.py:734`
- deep 分支启动 RecallFlow：`lib/crewai/src/crewai/memory/unified_memory.py:763`
- RecallFlow 定义：`lib/crewai/src/crewai/memory/recall_flow.py:58`
- query 分析：`lib/crewai/src/crewai/memory/recall_flow.py:180`
- 选 scopes：`lib/crewai/src/crewai/memory/recall_flow.py:243`
- 搜索：`lib/crewai/src/crewai/memory/recall_flow.py:266`
- 深挖路由：`lib/crewai/src/crewai/memory/recall_flow.py:271`
- 结果合成：`lib/crewai/src/crewai/memory/recall_flow.py:343`

## 4.7 Memory 如何进入 agent

有两个入口。

### 自动入口：追加到 task prompt

`Agent._retrieve_memory_context()` 会在执行任务前 recall：

`lib/crewai/src/crewai/agent/core.py:511`

逻辑是：

```text
如果 agent.memory 或 crew._memory 存在：
  query = task.description
  matches = unified_memory.recall(query, limit=5)
  拼成 "Relevant memories:"
  追加到 task_prompt
```

这属于自动注入，不需要 agent 主动调用工具。

### 主动入口：注入 memory tools

`Crew._prepare_tools()` 中，如果存在 memory，会调用 `_add_memory_tools()`：

- 检测 memory：`lib/crewai/src/crewai/crew.py:1652`
- 注入工具：`lib/crewai/src/crewai/crew.py:1761`

工具定义：

`lib/crewai/src/crewai/tools/memory_tools.py:1`

两个工具：

- `Search memory`
- `Save to memory`

如果 memory 是 read-only，只提供 recall，不提供 remember。

源码：`lib/crewai/src/crewai/tools/memory_tools.py:104`

## 4.8 context window compression

压缩入口：

`handle_context_length()`：`lib/crewai/src/crewai/utilities/agent_utils.py:712`

当 LLM 报上下文超限：

- 如果 `respect_context_window=True`，调用 `summarize_messages()`。
- 否则抛 `SystemExit`，提示用更小输入或 RAG tools。

压缩函数：

`summarize_messages()`：`lib/crewai/src/crewai/utilities/agent_utils.py:920`

它会：

1. 保留 system messages。
2. 收集 user messages 里的 files。
3. 把非 system messages 按估算 token 切块。
4. 每块调用 LLM 总结。
5. 清空原 messages。
6. 放回 system messages。
7. 追加一个 summary message。

这说明压缩的是 executor 内部的 `messages`，不是 memory，也不是 task context。

在新 executor 中，context 超限会变成路由：

- LLM call 捕获 context error：`lib/crewai/src/crewai/experimental/agent_executor.py:1442`
- recovery 节点：`lib/crewai/src/crewai/experimental/agent_executor.py:2701`

## 经典问题

### Q1：CrewAI 的 memory 会自动保存每个 task 输出吗？

本章源码可以确认两种明确路径：执行前自动 recall，执行中通过 `Save to memory` 工具主动保存。是否自动保存某类输出，要继续追事件监听或具体调用点，不能想当然认为“所有输出自动进记忆”。在当前主链路里，`Task._execute_core()` 负责生成 `TaskOutput`，没有直接调用 `Memory.remember()`。

### Q2：memory 和 knowledge 有什么区别？

memory 偏运行中积累、可写、可召回；knowledge 偏预置知识源、RAG 查询。`Agent.execute_task()` 中先处理 memory context，再处理 knowledge retrieval，二者都可能进入 task prompt，但生命周期不同。

源码：

- memory retrieval：`lib/crewai/src/crewai/agent/core.py:807`
- knowledge retrieval：`lib/crewai/src/crewai/agent/core.py:809`

### Q3：压缩会不会丢掉 system prompt？

不会。`summarize_messages()` 先保存 `system_messages`，清空后再放回去。

源码：`lib/crewai/src/crewai/utilities/agent_utils.py:947`

### Q4：为什么 `remember_many()` 要后台保存？

因为 agent 调工具保存多条记忆时，不应该阻塞当前推理太久。CrewAI 通过后台线程池异步保存，同时在 recall 前用 `drain_writes()` 做读屏障，兼顾速度和一致性。

