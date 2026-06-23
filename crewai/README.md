# CrewAI 源码学习路线

> 这套文档是给“想学 agent 应用开发，但不想只停留在 API 使用”的读者准备的。
>
> 阅读方式：每章先建立心理模型，再按源码入口追调用链，最后用经典问题反过来检查你是否真的理解。

## 先说版本

本地源码位于：

`/Users/miracle778/Project/crewAI`

这份学习文档以本地源码为准。官方线上文档当前页面显示的是稳定文档版本，概念上有帮助，但本地源码里已经能看到更靠前的实现细节，例如：

- `crewai.experimental.agent_executor.AgentExecutor` 已经是默认 executor。
- `Memory` 是统一记忆系统，不再按 short-term / long-term / entity 拆成几套类。
- Flow、checkpoint、memory scope/slice、MCP、A2A 都已经进入核心包。

补充参考：

- 官方 Agents 文档：https://docs.crewai.com/en/concepts/agents
- 官方 Crews 文档：https://docs.crewai.com/en/concepts/crews
- 官方 Tasks 文档：https://docs.crewai.com/en/concepts/tasks
- 官方 Processes 文档：https://docs.crewai.com/en/concepts/processes
- 官方 Memory 文档：https://docs.crewai.com/en/concepts/memory
- 官方 Flows 文档：https://docs.crewai.com/en/concepts/flows
- GitHub 仓库：https://github.com/crewAIInc/crewAI

## 建议阅读顺序

1. `crewai-ch0-map.md`
   先建立总览：CrewAI 到底由哪些零件组成。

2. `crewai-ch1-kickoff-to-agent.md`
   从 `Crew.kickoff()` 一路走到 `Task.execute_sync()` 和 `Agent.execute_task()`。

3. `crewai-ch2-agent-executor.md`
   专门读 AgentExecutor：prompt 怎么来，LLM 怎么调，tool loop 怎么跑。

4. `crewai-ch3-multi-agent-orchestration.md`
   多 agent 如何调度：sequential、hierarchical、manager agent、delegation tools。

5. `crewai-ch4-memory-context-compression.md`
   记忆、上下文、压缩：Memory 怎么存、怎么召回、怎么注入 prompt，context 超限怎么总结。

6. `crewai-ch5-flows-checkpoint-production.md`
   生产化视角：Flow、checkpoint、事件、状态恢复。

## 最小源码书签

建议先把这些文件加入编辑器收藏：

```text
lib/crewai/src/crewai/crew.py
lib/crewai/src/crewai/task.py
lib/crewai/src/crewai/agent/core.py
lib/crewai/src/crewai/experimental/agent_executor.py
lib/crewai/src/crewai/agents/crew_agent_executor.py
lib/crewai/src/crewai/tools/agent_tools/base_agent_tools.py
lib/crewai/src/crewai/tools/memory_tools.py
lib/crewai/src/crewai/memory/unified_memory.py
lib/crewai/src/crewai/memory/recall_flow.py
lib/crewai/src/crewai/utilities/agent_utils.py
lib/crewai/src/crewai/context.py
lib/crewai/src/crewai/flow/runtime/__init__.py
```

## 这套文档的核心问题

读完以后，你应该能回答这些问题：

- 一个用户调用 `crew.kickoff()` 后，真实调用链是什么？
- CrewAI 的 “agent” 到底是配置对象、执行器，还是 ReAct 循环？
- 多 agent 调度是 scheduler，还是 manager agent 通过工具委派？
- task 的 `context` 到底是什么，为什么不是完整聊天历史？
- memory 是自动注入 prompt，还是 agent 主动用工具查？
- “压缩上下文”发生在什么时候？压缩的是 task context，还是 executor messages？
- Flow 和 Crew 是什么关系？为什么 AgentExecutor 本身也继承 Flow？
- checkpoint 恢复为什么要重新绑定 agent、executor、memory view 和 event scope？

