# 第 7 章 精讲版：周边系统——工具 / MCP / Skills / Sandbox / Models

> 前面几章讲的是"Agent 怎么思考、怎么调度、怎么记忆"。这一章讲 Agent 的"手和脚"——它用什么工具、怎么连 MCP、怎么加载技能、在哪跑命令、怎么选模型。
>
> **配套文件**：
> - `tools/tools.py` / `tools/sync.py` / `tools/builtins/`
> - `mcp/client.py` / `mcp/tools.py` / `mcp/cache.py` / `mcp/session_pool.py`
> - `skills/types.py` / `skills/parser.py` / `skills/storage/` / `skills/installer.py`
> - `agents/middlewares/skill_activation_middleware.py`
> - `sandbox/sandbox.py` / `sandbox/sandbox_provider.py` / `sandbox/local/`
> - `models/factory.py`

---

# 🎯 先建立心理模型：Agent 的"器官系统"

如果把 Agent 比作一个人：

```
┌─────────────────────────────────────────────┐
│  大脑 = LLM（models/factory.py）              │
│  手脚 = 工具（tools/）                         │
│  外接设备 = MCP 工具（mcp/）                   │
│  技能包 = Skills（skills/）                    │
│  工作台 = Sandbox（sandbox/）                  │
└─────────────────────────────────────────────┘
```

---

# 📖 7.1 Tools 系统

## 7.1.1 工具装配

**文件**：`tools/tools.py:44` `get_available_tools()`

按顺序构造、按名去重：

```python
def get_available_tools(model_name=None, groups=None, subagent_enabled=False, app_config=None):
    tools = []
    
    # 1. config 加载的工具（按 groups 过滤）
    for cfg in app_config.tools:
        if groups and cfg.group not in groups:
            continue
        tool = resolve_variable(cfg.use, BaseTool)    # 反射加载
        tools.append(tool)
    
    # 2. 内置工具
    tools.extend(BUILTIN_TOOLS)    # present_file, ask_clarification
    if model_supports_vision:
        tools.append(view_image_tool)
    if skill_evolution_enabled:
        tools.append(skill_manage_tool)
    if subagent_enabled:
        tools.extend(SUBAGENT_TOOLS)    # task_tool
    
    # 3. MCP 工具（缓存）
    mcp_tools = get_cached_mcp_tools()
    for t in mcp_tools:
        tools.append(tag_mcp_tool(t))
    
    # 4. ACP 工具
    ...
    
    # 按名去重
    return dedup_by_name(tools)
```

**注意 `subagent_enabled` 参数**：子 agent 获取工具时传 `False`——不带 `task` 工具，防递归（第 2 章讲过）。

## 7.1.2 同步包装器

**文件**：`tools/sync.py`

有些工具是 async-only 的，但 sync agent caller 需要同步调用。`make_sync_tool_wrapper` 用线程池桥接：

```python
_sync_executor = ThreadPoolExecutor(max_workers=10, thread_name_prefix="tool-sync")

def make_sync_tool_wrapper(coro, tool_name):
    def wrapper(*args, **kwargs):
        future = _sync_executor.submit(asyncio.run, coro(*args, **kwargs))
        return future.result()
    return wrapper
```

## 7.1.3 tool_search——延迟工具发现

**文件**：`tools/builtins/tool_search.py`

第 1 章 merge_promoted 和第 4 章 DeferredToolFilter 已讲过 state/middleware 侧。这里看工具本身：

```python
def build_tool_search_tool(catalog: DeferredToolCatalog) -> BaseTool:
    @tool
    def tool_search(query: str) -> Command:
        # 三种查询：
        # "select:Read,Edit"     → 精确选择
        # "notebook jupyter"     → 关键词正则
        # "+slack send"          → require token + rank by rest
        matched = catalog.search(query)[:MAX_RESULTS]    # 最多 5 个
        
        names = [t.name for t in matched]
        content = json.dumps([convert_to_openai_function(t) for t in matched])
        
        return Command(
            update={
                "promoted": {"catalog_hash": catalog.hash, "names": names},
                "messages": [ToolMessage(content=content, ...)],
            }
        )
    return tool_search
```

**关键**：返回 `Command(update={"promoted": ...})`——promotion 靠**图状态**传递，不靠 ContextVar。

---

# 📖 7.2 MCP 客户端

DeerFlow 是 **MCP 客户端**（不托管 MCP server）。用 `langchain-mcp-adapters` 的 `MultiServerMCPClient`。

## 7.2.1 三种传输

**文件**：`mcp/client.py:11`

```python
def build_server_params(server_name, config):
    if config.transport == "stdio":
        return {"command": config.command, "args": config.args, "env": config.env}
    elif config.transport == "sse":
        return {"url": config.url, "headers": config.headers}
    elif config.transport == "http":
        return {"url": config.url, "headers": config.headers}
```

## 7.2.2 工具加载

**文件**：`mcp/tools.py:182` `get_mcp_tools()`

```python
async def get_mcp_tools():
    # 1. 每次从盘重读 ExtensionsConfig（让 Gateway-API 编辑立即生效）
    extensions = ExtensionsConfig.from_file()
    
    # 2. 注入 OAuth headers（sse/http）
    # 3. 构造 MultiServerMCPClient
    client = MultiServerMCPClient(servers_config, tool_interceptors=..., tool_name_prefix=True)
    
    # 4. 发现工具 schema
    tools = await client.get_tools()
    
    # 5. ★ stdio 工具包一层 session pool
    for tool in tools:
        if tool.metadata.get("transport") == "stdio":
            tool = _make_session_pool_tool(tool)    # 复用持久 session
        # 6. 打 sync wrapper
        tool = make_sync_tool_wrapper(tool)
    
    return tools
```

**为什么要 session pool？** 因为有些 MCP server 是**有状态**的（比如 Playwright——你打开一个浏览器标签页，下次调用要操作同一个标签页）。如果每次调用都新建 session，状态就丢了。`_make_session_pool_tool` 按 `(server_name, thread_id)` 复用 session。

**http/sse 工具不包**——因为 anyio TaskGroup 跨任务清理有问题（issue #3203）。

## 7.2.3 缓存

**文件**：`mcp/cache.py`

```python
_mcp_tools_cache = None
_cache_initialized = False
_config_mtime = None

def get_cached_mcp_tools():
    if _is_cache_stale():    # config mtime 变了
        reset_mcp_tools_cache()
    if not _cache_initialized:
        initialize_mcp_tools()    # async，有锁
    return _mcp_tools_cache
```

**`_is_cache_stale`** 检查 `extensions_config.json` 的 mtime——管理员通过 API 改了 MCP 配置，下次获取工具就自动刷新。

---

# 📖 7.3 Skills（SKILL.md）

## 7.3.1 什么是 Skill

一个 skill = 一个目录，含 `SKILL.md` 文件。`Skill` dataclass（`skills/types.py:20`）：

```python
@dataclass
class Skill:
    name: str
    description: str
    license: str | None
    skill_dir: str
    skill_file: str           # SKILL.md 的路径
    category: SkillCategory   # PUBLIC（内置只读）或 CUSTOM（用户写的）
    allowed_tools: list[str] | None
    enabled: bool
```

**容器路径**：skill 在沙箱里的路径默认是 `/mnt/skills/{relative_path}`（`types.py:39-65`）。

## 7.3.2 SKILL.md 格式

**文件**：`skills/parser.py:66`

```markdown
---
name: docx
description: 创建和编辑 Word 文档
license: MIT
allowed-tools:
  - bash
  - read_file
  - write_file
---

# DOCX 技能

## 什么时候用
当用户需要创建 .docx 文件时...

## 怎么用
1. 先用 python-docx 库...
2. ...
```

YAML frontmatter（`---` 围栏）+ Markdown 正文。必需字段 `name`、`description`；可选 `license`、`allowed-tools`。

**`allowed-tools` 为空**表示"无工具 skill"（纯指令性，不限制工具）。

## 7.3.3 存储——模板方法模式

**文件**：`skills/storage/skill_storage.py`

`SkillStorage` 是 ABC，用**模板方法模式**——`load_skills` 等具体流程是 final（不可覆盖），组合抽象原子操作：

```python
class SkillStorage(ABC):
    def load_skills(self, enabled_only=True):
        # final 模板方法
        skills = []
        for skill_file in self._iter_skill_files():    # 抽象
            skill = self._parse_skill(skill_file)
            skills.append(skill)
        # 合并 enabled 状态（从 ExtensionsConfig 读）
        return self._merge_enabled(skills, enabled_only)
    
    @abstractmethod
    def _iter_skill_files(self): ...
    
    @abstractmethod
    def read_custom_skill(self, name): ...
    
    @abstractmethod
    def write_custom_skill(self, name, content): ...
```

`LocalSkillStorage` 是文件系统实现。`get_or_new_skill_storage()` 单例工厂——通过 `resolve_class(skills_config.use, SkillStorage)` 反射加载。

## 7.3.4 安装安全

**文件**：`skills/installer.py`

`safe_extract_skill_archive` 防御：

- **zip slip**（`is_unsafe_zip_member`，33 行）：路径含 `..` 或绝对路径；
- **symlink**（`is_symlink_member`，51 行）：符号链接可能指向沙箱外；
- **zip bomb**（512 MB 上限）；
- **macOS metadata**（`__MACOSX/`、`.DS_Store`）。

安装后 `_scan_skill_archive_contents_or_raise`（177 行）跑安全扫描，scripts 文件要**显式 allow**。

## 7.3.5 斜杠激活

**文件**：`agents/middlewares/skill_activation_middleware.py`

用户输入 `/docx 帮我生成简历` → 中间件读 `docx` 的 SKILL.md，注入上下文。

**解析**（`skills/slash.py:29`）：

```python
# 正则：^/([a-z0-9]+(?:-[a-z0-9]+)*)(?:\s+|$)
# 保留名（不激活）：bootstrap, help, memory, models, new, status
```

**注入方式**（`_build_activation_message`，250 行）：

```python
return HumanMessage(
    content=activation_content,    # 包在 <slash_skill_activation> XML 里
    id=f"{stable_id}__slash_activation",
    additional_kwargs={"hide_from_ui": True, _SLASH_SKILL_ACTIVATION_KEY: True},
)
```

**关键**：skill 内容作为 **HumanMessage** 注入，不是 SystemMessage——避免多条 SystemMessage，且便于按 id 去重。

**路径安全**（`_read_skill_content`，84-93 行）：`resolved_file.relative_to(resolved_root)` 防路径逃逸。

---

# 📖 7.4 Sandbox

## 7.4.1 抽象

**文件**：`sandbox/sandbox.py:6`

```python
class Sandbox(ABC):
    @abstractmethod
    def execute_command(self, command, timeout=None) -> CommandResult: ...
    @abstractmethod
    def read_file(self, path) -> str: ...
    @abstractmethod
    def write_file(self, path, content) -> None: ...
    @abstractmethod
    def list_dir(self, path) -> list[str]: ...
    @abstractmethod
    def glob(self, pattern) -> list[str]: ...
    @abstractmethod
    def grep(self, pattern, **kwargs) -> list[dict]: ...
    @abstractmethod
    def update_file(self, path, old_str, new_str) -> None: ...
```

## 7.4.2 Provider 抽象

**文件**：`sandbox/sandbox_provider.py:9`

```python
class SandboxProvider(ABC):
    uses_thread_data_mounts: bool = False          # 是否用 thread data 挂载
    needs_upload_permission_adjustment: bool = True  # 是否需要调上传文件权限
    
    @abstractmethod
    def acquire(self, thread_id: str) -> str: ...  # 获取 sandbox_id
    @abstractmethod
    def get(self, sandbox_id: str) -> Sandbox | None: ...
    @abstractmethod
    def release(self, sandbox_id: str) -> None: ...
```

`get_sandbox_provider()` 反射加载单例。

## 7.4.3 Local 实现

**文件**：`sandbox/local/local_sandbox_provider.py`

**两层**：

1. **通用单例**（id `"local"`）：给没有 thread 上下文的调用用；
2. **LRU 按线程缓存**（OrderedDict，cap 256）：给有 thread_id 的调用用，`release` 是 no-op（让 LRU entry 和 `_agent_written_paths` 在轮次间存活）。

**静态路径映射**（`_setup_path_mappings`，82 行）：

```
/mnt/skills         → host skills dir（只读）
config.sandbox.mounts 的条目
保留前缀 /mnt/skills, /mnt/acp-workspace, /mnt/user-data 不可被覆盖
```

**按线程映射**（`_build_thread_path_mappings`，187 行）：

```
/mnt/user-data/workspace → {base}/users/{user_id}/threads/{thread_id}/workspace
/mnt/user-data/uploads   → .../uploads
/mnt/user-data/outputs   → .../outputs
```

## 7.4.4 LocalSandbox 的双向路径翻译

**文件**：`sandbox/local/local_sandbox.py`

**核心设计**：LLM 全程只看到 `/mnt/...` 虚拟路径，不知道实际跑在宿主哪个目录。

```python
class LocalSandbox(Sandbox):
    def execute_command(self, command, timeout=600):
        # 1. 把容器路径翻译成宿主路径
        resolved = self._resolve_paths_in_command(command)
        # 2. 跑命令
        result = subprocess.run([shell, "-c", resolved], timeout=timeout)
        # 3. 把输出里的宿主路径翻译回容器路径
        output = self._reverse_resolve_paths_in_output(result.stdout)
        return CommandResult(stdout=output, ...)
```

**路径安全**（`_resolve_path_with_mapping`，127 行）：

```python
def _resolve_path_with_mapping(self, container_path):
    mapping = self._find_path_mapping(container_path)    # 最长前缀匹配
    local_path = mapping.local_path / container_path.relative_to(mapping.container_path)
    # ★ 强制检查：解析后的路径必须在 mapping 范围内
    resolved = local_path.resolve()
    if not str(resolved).startswith(str(mapping.local_path.resolve())):
        raise PermissionError(EACCES, "Path escape detected")
    return resolved
```

**只读检查**（`_is_read_only_path`，86 行）：在只读 mount 上写文件抛 `OSError(EROFS)`。

**`_agent_written_paths`**（84 行）：记录 agent 写过的文件。`read_file` 时只对 agent 写的文件做反向路径翻译——用户上传的文件不翻译（PR #1935 的 fix）。

## 7.4.5 Shell 检测

**文件**：`local_sandbox.py:285` `_get_shell`

```python
def _get_shell(self):
    for shell in ["/bin/zsh", "/bin/bash", "/bin/sh", "sh"]:
        if shutil.which(shell):
            return shell
    # Windows fallback: PowerShell / cmd
```

---

# 📖 7.5 Models——完全配置驱动

## 7.5.1 工厂函数

**文件**：`models/factory.py:82`

```python
def create_chat_model(name=None, thinking_enabled=False, *, app_config=None, 
                      attach_tracing=True, **kwargs) -> BaseChatModel:
    # 1. 解析 model config（兜底 config.models[0]）
    model_config = config.get_model_config(name) or config.models[0]
    
    # 2. 反射加载 model class
    model_class = resolve_class(model_config.use, BaseChatModel)
    #    e.g. "langchain_openai:ChatOpenAI" → ChatOpenAI
    
    # 3. thinking 配置（合并 when_thinking_enabled / when_thinking_disabled）
    if thinking_enabled:
        kwargs.update(model_config.when_thinking_enabled)    # {"extra_body": {"thinking": {"type": "enabled"}}}
    else:
        kwargs.update(model_config.when_thinking_disabled)
    
    # 4. 流式默认
    _enable_stream_usage_by_default(model_config, kwargs)    # stream_usage=True
    _apply_stream_chunk_timeout_default(model_config, kwargs) # stream_chunk_timeout=240s
    
    # 5. 造实例
    model = model_class(**kwargs)
    
    # 6. tracing
    if attach_tracing:
        model.callbacks.extend(build_tracing_callbacks())
    
    return model
```

## 7.5.2 thinking 配置——三种模式

| 模式 | 配置 | 适用 |
|---|---|---|
| OpenAI 兼容 | `extra_body.thinking.type` + `reasoning_effort` | GPT-5, DeepSeek-R1 |
| vLLM/Qwen | `extra_body.chat_template_kwargs.thinking` | vLLM 部署的 Qwen |
| 原生 Anthropic | `thinking={"type": "disabled"}` 构造参数 | Claude |

## 7.5.3 流式默认值调优

**`stream_chunk_timeout=240s`**（默认 60s）——reasoning 模型（DeepSeek-R1、GPT-5）会先思考几十秒到几分钟才吐第一个 token，默认 60s 会误判超时断连。

**`stream_usage=True`**——给带自定义 `base_url` 的 OpenAI 兼容模型开，否则 token 跟踪失效。

## 7.5.4 支持的 provider

```
models/
├── factory.py              ← 工厂
├── claude_provider.py      ← Anthropic Claude
├── vllm_provider.py        ← vLLM
├── mindie_provider.py      ← 华为 MindIE
├── openai_codex_provider.py ← OpenAI Codex Responses API
├── patched_deepseek.py     ← DeepSeek 补丁
├── patched_mimo.py         ← MiMo 补丁
├── patched_minimax.py      ← MiniMax 补丁
├── patched_openai.py       ← OpenAI 补丁
└── patched_stepfun.py      ← 阶步星辰补丁
```

任何 LangChain class 都可通过 `use:` 反射加载。

---

# 🎯 现在回头检验：你掌握了吗？

### 问题 1：MCP 工具太多撑爆上下文，怎么办？

<details>
<summary>🔍 点击看答案</summary>

**延迟加载**（第 4 章 DeferredToolFilter）：MCP 工具的 schema 默认**不发给模型**，只发工具名列表。模型要用时先调 `tool_search`，提升具体 schema 后才能调用。`DeferredToolFilterMiddleware` 在 `wrap_model_call` 里过滤掉未提升的工具 schema，在 `wrap_tool_call` 里拦截强调。

这把"上下文成本"和"工具发现"解耦——你可以接 100 个 MCP server，而上下文只占当前轮真正需要的几个 schema。

</details>

### 问题 2：本地 sandbox 怎么做到"路径隔离"又不让 LLM 看到宿主真实路径？

<details>
<summary>🔍 点击看答案</summary>

靠**双向路径翻译**：

- `_resolve_paths_in_command`（命令执行前）：把 LLM 写的容器路径（`/mnt/user-data/workspace/foo.py`）翻译成宿主真实路径；
- `_reverse_resolve_paths_in_output`（命令执行后）：把输出里的宿主路径翻译回容器路径。

LLM 全程只看到 `/mnt/...` 虚拟路径，不知道实际跑在宿主哪个目录。既有安全意义（不泄露宿主结构），也有一致性意义（跨 local/docker sandbox 行为一致）。

</details>

### 问题 3：为什么 skill 内容用 HumanMessage 注入，不用 SystemMessage？

<details>
<summary>🔍 点击看答案</summary>

两个原因：

1. **避免多条 SystemMessage**：create_agent 已用 system_prompt 参数注入了一条 SystemMessage，再追加会变成两条，某些模型行为异常；
2. **便于按 id 去重**：HumanMessage 带 `additional_kwargs[_SLASH_SKILL_ACTIVATION_KEY]=True` 标志和稳定的 `__slash_activation` id 后缀，能精确检测"这条 skill 已经注入过了"，避免重复注入。

</details>

### 问题 4：`stream_chunk_timeout` 为什么要改成 240s？

<details>
<summary>🔍 点击看答案</summary>

因为 **reasoning 模型**（DeepSeek-R1、GPT-5、o1）会先思考几十秒到几分钟才吐第一个 token。LangChain 默认 60s，对这种模型会**误判超时断连**，导致 agent 莫名其妙失败。

DeerFlow 改成 240s，覆盖大部分 reasoning 时长。

</details>

### 问题 5：MCP stdio 工具为什么要包 session pool？

<details>
<summary>🔍 点击看答案</summary>

因为有些 MCP server 是**有状态**的（如 Playwright——打开浏览器标签页后，后续操作要在同一个 session 里）。如果每次调用都新建 session，状态就丢了。

`_make_session_pool_tool` 按 `(server_name, thread_id)` 复用持久 session——同一线程对同一 server 的连续调用共享 session。

http/sse 工具不包，因为 anyio TaskGroup 跨任务清理有问题（issue #3203）。

</details>

---

# 📝 一页纸总结（第 7 章精华）

```
┌──────────────────────────────────────────────────────────────┐
│  Agent 的"器官系统"                                            │
└──────────────────────────────────────────────────────────────┘
        │
        ├─▶ Tools（手脚）
        │     • get_available_tools: config + builtin + MCP + ACP，按名去重
        │     • sync wrapper: ThreadPoolExecutor 桥接 async→sync
        │     • tool_search: 延迟发现，Command(update={"promoted":...})
        │
        ├─▶ MCP（外接设备）
        │     • 客户端（非 server），用 MultiServerMCPClient
        │     • 三种传输：stdio / sse / http
        │     • stdio 包 session pool（有状态 server 复用 session）
        │     • 缓存 + mtime 失效检查
        │
        ├─▶ Skills（技能包）
        │     • SKILL.md = YAML frontmatter + Markdown 正文
        │     • 模板方法模式存储（SkillStorage ABC + LocalSkillStorage）
        │     • 安全部署：防 zip slip / symlink / zip bomb
        │     • /skill-name 斜杠激活 → HumanMessage 注入（非 SystemMessage）
        │
        ├─▶ Sandbox（工作台）
        │     • Sandbox ABC + SandboxProvider ABC
        │     • Local: LRU 按线程缓存 + 路径映射
        │     • 双向路径翻译（容器↔宿主），LLM 只看虚拟路径
        │     • 路径安全：relative_to 检查防逃逸
        │     • 只读 mount 写文件抛 EROFS
        │
        └─▶ Models（大脑）
              • create_chat_model: 反射加载，配置驱动
              • thinking 三模式：OpenAI/vLLM/Anthropic
              • stream_chunk_timeout=240s（reasoning 模型友好）
              • attach_tracing=False 防重复 span
              • 10+ provider（claude/vllm/mindie/codex/patched_*）
```

**5 句话记住**：

1. **工具按 config+builtin+MCP+ACP 装配**，按名去重，子 agent 传 `subagent_enabled=False` 防递归；
2. **MCP stdio 工具包 session pool**——有状态 server 复用 session，按 `(server_name, thread_id)` 键；
3. **Skills 用 HumanMessage 注入**——避免多条 SystemMessage，便于按 id 去重，路径安全用 `relative_to` 检查；
4. **Sandbox 双向路径翻译**——LLM 只看 `/mnt/...` 虚拟路径，不知道宿主真实路径，`relative_to` 防逃逸；
5. **Models 完全配置驱动**——`use:` 反射加载，thinking 三模式，`stream_chunk_timeout=240s` 适配 reasoning 模型。
