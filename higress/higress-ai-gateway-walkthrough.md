# Higress AI 网关源码精讲（手把手带读）

> 这份文档是给"想学 Agent / AI 应用开发，但看着 Higress 一堆插件不知道从哪下手"的同学准备的。
> 我们用**对话式**的方式，**对照源码**一层层拆，把"AI 网关到底在干什么、怎么干的"讲清楚。
>
> **阅读方法**：每段代码都配了"这是什么"和"为什么这样写"。看到代码先自己读一遍，猜猜在干嘛，再看下面的解释。
>
> **配套源码**：建议同时打开这些文件对照看（路径相对于仓库根 `plugins/wasm-go/extensions/`）：
> - `hello-world/main.go`（最简插件）
> - `ai-proxy/main.go`（AI 网关心脏）
> - `ai-proxy/provider/provider.go`（Provider 抽象核心）
> - `ai-proxy/provider/openai.go`、`claude.go`（典型 Provider）
>
> Higress 仓库根：`/Users/miracle778/Project/higress`

---

# 🎯 先建立一个心理模型：AI 网关到底在干嘛？

在讲代码之前，先用**最朴素的语言**讲清楚 Higress 作为 AI 网关在做什么。

## 一个比喻：Higress 是 AI 世界的"超级前台"

想象你开了一家公司，有几十个不同供应商的"AI 员工"（OpenAI、通义千问、Claude、DeepSeek、智谱……），每个都只会说自己方言（API 协议各不相同）。客户（你的应用 / Agent）来找你，你不能让客户去学几十种方言。

```
                  客户（你的 App / AI Agent）
                  统一说 OpenAI 方言（/v1/chat/completions）
                          │
                          ▼
              ┌───────────────────────────┐
              │   Higress AI 网关（前台）   │
              │                           │
              │  ① 协议翻译（ai-proxy）     │  OpenAI 方言 → 各家方言
              │  ② 模型映射                │  "gpt-4" → "qwen-max"
              │  ③ 鉴权托管                │  客户不用知道各家 API Key
              │  ④ 流量治理（限流/配额/LB） │  按 token 限流、按前缀缓存路由
              │  ⑤ 增强（缓存/RAG/历史）    │  命中缓存直接返回
              │  ⑥ 安全（内容审核）         │  拦截敏感请求/响应
              │  ⑦ 可观测（token 统计）     │  谁用了多少 token
              │  ⑧ MCP 托管                │  把 API 包装成 AI 能调的工具
              └─────────────┬─────────────┘
                            │
        ┌───────────┬───────┴───────┬───────────┐
        ▼           ▼               ▼           ▼
   ┌────────┐ ┌────────┐      ┌────────┐   ┌────────┐
   │ OpenAI │ │ 通义千问│      │ Claude │   │ DeepSeek│  ...50+
   └────────┘ └────────┘      └────────┘   └────────┘
```

**关键认知**：

1. **整个网关对外只暴露一张统一的脸**——OpenAI 协议（`/v1/chat/completions`）。客户端不管你后端接的是哪家，都按 OpenAI 格式发请求。
2. **`ai-proxy` 插件是心脏**——它做协议翻译、模型映射、鉴权托管，是所有 AI 能力的基础。
3. **其他 AI 插件是"中间件"**——挂在 `ai-proxy` 之上或之下，各管一摊（限流、缓存、安全……）。
4. **所有插件都是 Wasm**——用 Go/Rust 写，编译成 Wasm，沙箱隔离，能热更新（流量无损）。

Higress 这个"前台"的核心代码，就是回答一个问题：

> **「如何用一套 Wasm 插件机制，把几十家 AI 供应商统一成一张 OpenAI 协议的脸，并在上面叠加治理、增强、安全能力？」**

这份文档就按这个主线，一章一章拆。

---

# 📖 第 0 章 架构全景：Higress 是什么，AI 能力在哪

## 0.1 Higress 的技术底座

Higress 是云原生 API 网关，内核是 **Istio + Envoy**。但你不用被这两个词吓到，先建立直觉：

| 层 | 是什么 | 类比 |
|---|---|---|
| **Envoy** | C++ 写的高性能代理，真正转发流量的 | 公司的"门卫+快递员" |
| **Istio** | 控制面，把配置（路由、证书）下发成 Envoy 能懂的 xDS | 公司的"行政部" |
| **Wasm 插件** | 用 Go/Rust 写的小程序，编译成 Wasm，挂在 Envoy 的请求处理流程里 | 门卫桌上贴的"工作守则" |
| **控制台** | Web UI，点一点就能配路由、加插件 | 公司的"OA 系统" |

**为什么 AI 场景特别需要这套东西？** 看 README 的核心优势就懂了：

1. **配置变更毫秒级生效、无 reload**——AI 长连接（SSE 流式）场景，Nginx reload 会断流，Envoy 不会；
2. **真正的完全流式处理**——Wasm 插件能逐 chunk 处理 SSE 报文，AI 大带宽场景省内存；
3. **Wasm 沙箱热更新**——插件逻辑改了，实时生效，连接不断，对 AI 业务无感。

## 0.2 AI 网关能力总览

打开 `README_ZH.md`，AI 网关的核心能力是这 8 项（对应 8 类插件）：

```
┌─────────────────────────────────────────────────────────────┐
│  能力分类          代表插件              解决什么问题          │
├─────────────────────────────────────────────────────────────┤
│ ① 协议统一        ai-proxy              OpenAI↔各家方言互译   │
│ ② 流量治理        ai-load-balancer      多模型负载/前缀缓存路由│
│                   ai-token-ratelimit    按 token 限流          │
│                   ai-quota              按 consumer 配额        │
│                   model-router/mapper   按模型/意图分流         │
│ ③ 上下文增强      ai-cache              响应缓存（精确+语义）   │
│                   ai-rag / ai-search    检索增强               │
│                   ai-history            对话历史注入            │
│                   ai-prompt-*           提示词装饰/模板         │
│                   ai-context-limit      输入 token 预检         │
│                   ai-transformer        LLM 驱动的报文改写      │
│                   ai-json-resp          强制 JSON 输出          │
│                   ai-intent             意图识别分流            │
│                   ai-image-reader       图片理解               │
│ ④ Agent 编排      ai-agent              ReAct 工具调用循环      │
│ ⑤ 安全防护        ai-security-guard     双向内容审核            │
│                   ai-data-masking       敏感数据脱敏（Rust）    │
│ ⑥ 可观测          ai-statistics         token/延迟/链路追踪     │
│ ⑦ MCP 托管        mcp-server/router     把 API 变成 AI 工具     │
│ ⑧ 模型管理        model-mapper          模型名映射              │
└─────────────────────────────────────────────────────────────┘
```

## 0.3 插件体系全景

在仓库根跑一下 `ls plugins/`，能看到四类插件实现：

| 目录 | 语言 | 运行时 | 数量 | 典型 |
|---|---|---|---|---|
| `plugins/wasm-go/extensions/` | Go→Wasm | proxy-wasm | 60+ | ai-proxy、ai-cache、mcp-server |
| `plugins/wasm-rust/extensions/` | Rust→Wasm | proxy-wasm | 5 | ai-data-masking、request-block |
| `plugins/wasm-cpp/` | C++ | 内置 | — | （Envoy 原生 filter） |
| `plugins/golang-filter/` | Go（原生） | Envoy Go filter | 2 | mcp-server、mcp-session |

**为什么 MCP 有两套实现（Wasm + 原生 Go）？** 第 6 章会讲——因为 MCP 的 SSE 长连接和并发 fan-out 用原生 Go filter 更顺手，而轻量工具托管用 Wasm 更隔离。这是个值得学的工程取舍。

> 💡 **学习提示**：`plugins/wasm-go/extensions/` 是绝对的主战场，90% 的 AI 插件在这。后面所有章节默认指这个目录。

## 0.4 学习路线建议

```
第1章（Wasm插件骨架）── 必须先懂，是所有插件的地基
      │
      ▼
第2章（ai-proxy心脏）── 最核心，花最多时间
      │
      ├──▶ 第3章（流量治理）── 工程价值最高
      ├──▶ 第4章（增强插件）── 最像"Agent 开发"
      ├──▶ 第5章（安全可观测）── 生产必备
      └──▶ 第6章（MCP托管）── Agent 生态前沿
```

**建议**：第 1、2 章逐行读，第 3-6 章挑 2-3 个最感兴趣的深读，其余当字典查。

---

# 📖 第 1 章 Wasm 插件骨架：一个 Higress 插件长什么样

在看复杂的 `ai-proxy` 之前，先用一个最简单的插件建立直觉。这一章是后面所有章节的地基。

## 1.1 最简单的插件：hello-world

**文件**：`plugins/wasm-go/extensions/hello-world/main.go`（去掉版权头就 20 行）

```go
package main

import (
	"net/http"
	"github.com/higress-group/proxy-wasm-go-sdk/proxywasm"
	"github.com/higress-group/proxy-wasm-go-sdk/proxywasm/types"
	"github.com/higress-group/wasm-go/pkg/log"
	"github.com/higress-group/wasm-go/pkg/wrapper"
)

func main() {}

func init() {
	wrapper.SetCtx(
		"hello-world",
		wrapper.ProcessRequestHeadersBy(onHttpRequestHeaders),
	)
}

type HelloWorldConfig struct {}

func onHttpRequestHeaders(ctx wrapper.HttpContext, config HelloWorldConfig, log log.Log) types.Action {
	err := proxywasm.AddHttpRequestHeader("hello", "world")
	if err != nil {
		log.Critical("failed to set request header")
	}
	proxywasm.SendHttpResponseWithDetail(http.StatusOK, "hello-world", nil, []byte("hello world"), -1)
	return types.ActionContinue
}
```

**逐行翻译**：

| 代码 | 含义 |
|---|---|
| `func main() {}` | 空的。Wasm 插件不是可执行程序，入口在 `init()` |
| `wrapper.SetCtx("hello-world", ...)` | ★ **注册插件**：名字叫 `hello-world`，告诉 SDK 这个插件要 hook 哪些阶段 |
| `wrapper.ProcessRequestHeadersBy(onHttpRequestHeaders)` | 只 hook 一个阶段：**请求头阶段**。`By` 后缀表示配置结构体在单独的包里（这里其实就在 main 包，但约定用 `By` 变体时多一个 `log` 参数） |
| `onHttpRequestHeaders(ctx, config, log)` | hook 回调：每个请求到达时被调用 |
| `proxywasm.AddHttpRequestHeader("hello", "world")` | 给请求加一个 header |
| `proxywasm.SendHttpResponseWithDetail(...)` | 直接短路返回 200 + "hello world"，**不转发到上游** |
| `return types.ActionContinue` | 告诉 SDK："我处理完了，继续" |

**这就是一个完整的 Higress 插件**。它的核心心智模型是：

> 🎯 **一个插件 = 一个名字 + 一组"阶段 hook" + 一份配置。**
>
> `wrapper.SetCtx` 是插件的"注册中心"，你声明"我叫什么、我要在哪些时机插手"，SDK 就会在对应时机调你的回调。

## 1.2 HTTP 处理的四个阶段——插件能插手的时机

一个 HTTP 请求经过网关，Envoy 会把它拆成几个阶段。Wasm 插件可以在每个阶段插手。看 `ai-proxy/main.go:106-117` 注册了哪些：

```go
func init() {
	wrapper.SetCtx(
		pluginName,
		wrapper.ParseOverrideConfig(parseGlobalConfig, parseOverrideRuleConfig),
		wrapper.ProcessRequestHeaders(onHttpRequestHeader),       // ① 请求头
		wrapper.ProcessRequestBody(onHttpRequestBody),            // ② 请求体
		wrapper.ProcessResponseHeaders(onHttpResponseHeaders),    // ③ 响应头
		wrapper.ProcessStreamingResponseBody(onStreamingResponseBody), // ④ 流式响应体（逐 chunk）
		wrapper.ProcessResponseBody(onHttpResponseBody),          // ⑤ 缓冲响应体（完整）
		wrapper.WithRebuildMaxMemBytes[config.PluginConfig](200*1024*1024),
	)
}
```

画成图：

```
   客户端请求
       │
       ▼
  ┌─────────────┐
  │ ① 请求头     │  onHttpRequestHeader   ← 能改 :path、:authority、加/删 header
  │   阶段       │                          此时还没读 body
  └──────┬──────┘
         │ 决定要读 body
         ▼
  ┌─────────────┐
  │ ② 请求体     │  onHttpRequestBody     ← 能改整个请求 body（改完转发）
  │   阶段       │                          ★ AI 协议翻译、模型映射在这做
  └──────┬──────┘
         │ 转发到上游 LLM
         ▼
  ┌─────────────┐
  │ ③ 响应头     │  onHttpResponseHeaders ← 能改响应 header、决定是否读 body
  └──────┬──────┘
         │
    ┌────┴────────┐
    │ 是流式(SSE)? │
    ├是──────────是┤
    ▼              ▼
 ┌──────────┐  ┌──────────┐
 │④ 流式逐  │  │⑤ 缓冲完整│
 │  chunk   │  │  body    │
 │ onStream │  │ onResp   │
 └──────────┘  └──────────┘
       │              │
       └──────┬───────┘
              ▼
          客户端
```

**关键认知**：

1. **④ 和 ⑤ 是互斥的选择**：流式响应走 `ProcessStreamingResponseBody`（每个 SSE chunk 调一次），非流式走 `ProcessResponseBody`（攒完一次性给）。`ai-proxy` 两个都注册了，运行时根据响应类型选。
2. **不是所有插件都注册全部阶段**。`hello-world` 只注册①；`ai-cache` 注册①②③④（流式）；`model-mapper` 只注册①②。**用不到的阶段就不注册，SDK 不会调你。**
3. **`ParseOverrideConfig`** 是配置解析，在所有阶段之前、插件加载时调用一次（后面 1.4 讲）。

> 💡 **为什么 AI 网关特别需要④流式阶段？** 因为 LLM 响应是 SSE 流（`data: {...}\n\n` 一段段吐）。网关要能**逐 chunk 改写**（比如把 DeepSeek 的流式格式翻译成 OpenAI 格式），不能等全部生成完再处理——那延迟就爆炸了。`ProcessStreamingResponseBody` 就是为此设计。

## 1.3 配置解析：ParseConfig vs ParseOverrideConfig

插件需要配置（比如 `ai-proxy` 要配"接哪个 provider、API Key 是啥"）。配置怎么传进来？看两种写法：

**简单写法**（model-mapper，`main.go:38`）：
```go
wrapper.SetCtx("model-mapper",
	wrapper.ParseConfig(parseConfig),   // ← 全局配置解析
	...
)

func parseConfig(json gjson.Result, config *Config) error {
	config.modelKey = json.Get("modelKey").String()
	...
}
```

**覆盖写法**（ai-proxy，`main.go:108-110`）：
```go
wrapper.SetCtx(pluginName,
	wrapper.ParseOverrideConfig(parseGlobalConfig, parseOverrideRuleConfig),  // ← 全局 + 路由级覆盖
	...
)
```

区别在于：

| 写法 | 含义 | 适用 |
|---|---|---|
| `ParseConfig` | 一份配置 | 简单插件 |
| `ParseOverrideConfig(global, override)` | 全局配置 + 每条路由可覆盖 | 复杂插件，不同路由用不同配置 |

看 `ai-proxy` 的覆盖逻辑（`main.go:135-151`）：
```go
func parseOverrideRuleConfig(json gjson.Result, global config.PluginConfig, pluginConfig *config.PluginConfig) error {
	*pluginConfig = global               // ① 先继承全局配置
	pluginConfig.FromJson(json)          // ② 再用路由级配置覆盖
	pluginConfig.Validate()              // ③ 校验
	pluginConfig.Complete()              // ④ 填充默认值
	return nil
}
```

**这就是"继承+覆盖"模式**——全局配一份默认，特定路由按需覆盖。和 DeerFlow 里"用户请求 → agent 配置 → 全局默认"的三级解析异曲同工。

## 1.4 异步暂停与恢复——最核心的控制流模式 ★

这是整个 Higress 插件体系**最关键**的设计，必须彻底搞懂。

**问题**：Wasm 插件里，你要调外部服务（比如查 Redis 缓存、调 embedding 接口）。但 Wasm 是**单线程同步**的，你直接阻塞等待会卡死整个 Envoy worker。

**解法**：**暂停 + 回调恢复**。看 `ai-cache` 的请求阶段（`core.go` 附近，简化）：

```go
func onHttpRequestBody(ctx, config, body) types.Action {
	// 1. 算出缓存 key
	key := buildCacheKey(body)
	// 2. 异步查 Redis（不阻塞！）
	redisClient.Get(key, func(resp) {
		// 4. Redis 返回后在回调里处理
		if cacheHit {
			proxywasm.SendHttpResponse(200, ...)  // 命中缓存，直接返回
		} else {
			proxywasm.ResumeHttpRequest()        // 没命中，恢复原请求转发
		}
	})
	// 3. 返回 ActionPause，告诉 SDK："先别转发，等我"
	return types.ActionPause
}
```

**执行时序**：

```
[T=0] onHttpRequestBody 被调
[T=1] 发起异步 Redis Get
[T=2] return ActionPause  ← SDK 暂停请求，不转发
      ...（Envoy worker 去处理别的请求，不阻塞）
[T=3] Redis 返回，触发回调
[T=4] 回调里决定：命中→SendHttpResponse 直接答；未命中→ResumeHttpRequest 恢复转发
```

**几个 Action 的区别**（这是新手最容易混的）：

| 返回值 | 含义 | 场景 |
|---|---|---|
| `types.ActionContinue` | 我处理完了，继续下一阶段 | 不需要异步，直接放行 |
| `types.ActionPause` | 暂停请求，等我回调里 Resume | 要异步查 Redis/调外部服务 |
| `types.HeaderStopIteration` | 请求头阶段暂停，等读 body | 请求头里要先决定要不要读 body |

> 💡 **学习启示**：Higress 里所有"需要查外部"的插件（ai-cache、ai-rag、ai-search、ai-quota、ai-token-ratelimit、ai-agent……）**全是这个模式**：`ActionPause` → 异步调用 → 回调里 `Resume`。记住这个，看任何插件都不会迷路。

## 1.5 HttpContext——贯穿请求的"口袋"

和 DeerFlow 的 `RunnableConfig` 一样，Higress 也有一个贯穿单个请求的"大口袋"——`wrapper.HttpContext`。

`ai-proxy/main.go` 里大量用到它：

```go
ctx.SetContext(provider.CtxKeyApiName, apiName)           // 存：当前请求的 API 类型
apiName, _ := ctx.GetContext(provider.CtxKeyApiName).(provider.ApiName)  // 取
ctx.SetContext("needClaudeResponseConversion", true)      // 存：需要 Claude 转换标记
```

**为什么需要它？** 因为 Wasm 插件是**无状态**的——同一个插件实例处理成千上万个并发请求。每个请求的"中间状态"（这是什么 API、要不要转换、流式 buffer 到哪了）必须挂在 `ctx` 上，靠 `ctx` 区分不同请求。

`ai-proxy` 在 ctx 里存了一堆东西（看 `provider.go:184-194`）：
```go
CtxKeyApiName              // 当前 API 类型
ctxKeyIsStreaming          // 是不是流式
ctxKeyStreamingBody        // 流式 buffer（跨 chunk 的不完整数据）
ctxKeyOriginalRequestModel // 原始模型名
ctxKeyFinalRequestModel    // 映射后的模型名
ctxKeyApiKey               // 用的哪个 API token
...
```

**一个经典场景**：SSE 流的某个 chunk 可能是不完整的 JSON（被网络切断了）。`ctxKeyStreamingBody` 把不完整部分存起来，下个 chunk 来了拼上再解析。`provider.go:1091-1096` 的 `ExtractStreamingEvents` 就这么干。

## 1.6 出站 HTTP 调用——插件怎么调外部服务

很多 AI 插件要自己调 LLM 或外部服务（ai-agent、ai-transformer、ai-rag、ai-search……）。怎么调？用 `wrapper.HttpClient`。

看 `ai-rag/main.go` 怎么建客户端：
```go
// 建一个指向 DashScope 服务的 HTTP 客户端
dashscopeClient, _ := wrapper.NewClusterClient(wrapper.FQDNCluster{
	ServiceName: config.dashscope.serviceFQDN,
	Port:        config.dashscope.servicePort,
	Host:        config.dashscope.serviceHost,
})

// 异步调用
dashscopeClient.Post(path, headers, body, func(respHeaders, respBody) {
	// 回调里处理响应
})
```

**两种 cluster**：
- `wrapper.FQDNCluster{}`：用 Envoy 已有的集群（按 FQDN 服务发现），最常用；
- `wrapper.DnsCluster{}`：DNS 解析的集群（ai-transformer 用）。

**关键**：所有出站调用都是**异步回调**的，和 1.4 的暂停/恢复模式配套——发起调用、`return ActionPause`、回调里 `Resume`。

## 1.7 小结：一个 Higress 插件的"五要素"

```
┌──────────────────────────────────────────┐
│  一个 Higress Wasm 插件 = 五要素           │
│                                          │
│  ① 名字        wrapper.SetCtx("xxx")     │
│  ② 配置        ParseConfig / ParseOverride│
│  ③ 阶段 hook   ProcessRequestHeaders...   │
│  ④ HttpContext 贯穿请求的"口袋"            │
│  ⑤ 异步模式    ActionPause → 回调 → Resume│
└──────────────────────────────────────────┘
```

记住这五要素，第 2-6 章的每个插件都是它的变体。

---

# 📖 第 2 章 ai-proxy 深度剖析：AI 网关的"心脏"

`ai-proxy` 是整个 AI 网关最核心、最复杂的插件（105 个 Go 文件、4 万多行、50+ provider）。这一章逐行拆它的核心。

## 2.1 ai-proxy 解决什么问题

没有 `ai-proxy`，你要接 5 家 AI 供应商，客户端代码得写 5 套（5 种鉴权、5 种请求格式、5 种流式解析）。有了 `ai-proxy`：

```
客户端永远发 OpenAI 格式 /v1/chat/completions
         │
    ai-proxy 插件接管
         │
   ┌─────┴─────┐
   │  翻译+映射  │  根据 provider type：
   │           │   - 改 :path（OpenAI的/v1/chat/completions → 千问的/api/v1/services/aigc/...）
   │           │   - 改 :authority（→ dashscope.aliyuncs.com）
   │           │   - 改鉴权（Authorization: Bearer xxx → 换成各家要的格式）
   │           │   - 改 body（OpenAI格式 → 各家格式，或反过来）
   └─────┬─────┘
         ▼
   真正的 LLM 供应商
```

它做的事拆开是 6 件：**协议翻译、模型映射、鉴权托管、流式处理、token 管理、失败容错**。我们一个个看。

## 2.2 main.go 的总调度——五个 phase hook

`ai-proxy/main.go:106-117` 注册了 5 个 hook（1.2 节贴过）。每个 hook 的职责：

| Hook | 函数 | 干什么 |
|---|---|---|
| ① 请求头 | `onHttpRequestHeader` | 识别 API 类型、设 token、改 path/host、决定要不要读 body |
| ② 请求体 | `onHttpRequestBody` | 协议翻译、模型映射、上下文注入 |
| ③ 响应头 | `onHttpResponseHeaders` | 失败检测、token 健康检查、决定流式/缓冲 |
| ④ 流式响应体 | `onStreamingResponseBody` | 逐 chunk 翻译 SSE 流 |
| ⑤ 缓冲响应体 | `onHttpResponseBody` | 完整响应翻译 |

**主流程**是：①识别 → ②翻译请求 → 转发 → ③④/⑤翻译响应。

## 2.3 ApiName 路由：从 URL 识别"这是什么请求"

`ai-proxy` 要翻译，先得知道"客户端发的是哪种请求"。怎么知道？看 URL 路径。`main.go:53-101` 定义了一张映射表：

```go
pathSuffixToApiName = []pair[string, provider.ApiName]{
	{provider.PathOpenAIChatCompletions, provider.ApiNameChatCompletion},  // /v1/chat/completions → 聊天补全
	{provider.PathOpenAIEmbeddings, provider.ApiNameEmbeddings},           // /v1/embeddings → 向量化
	{provider.PathOpenAIImageGeneration, provider.ApiNameImageGeneration}, // /v1/images/generations → 生图
	...
	{provider.PathAnthropicMessages, provider.ApiNameAnthropicMessages},   // /v1/messages → Claude格式
	...
}
```

`getApiName`（`main.go:748-764`）就是个查表：
```go
func getApiName(path string) provider.ApiName {
	for _, p := range pathSuffixToApiName {
		if strings.HasSuffix(path, p.key) {   // 后缀匹配
			return p.value
		}
	}
	for _, p := range pathPatternToApiName {
		if p.key.MatchString(path) {          // 正则匹配（带 id 的路径）
			return p.value
		}
	}
	return ""
}
```

**ApiName 是什么？** 看 `provider.go:34`：`// ApiName 格式 {vendor}/{version}/{apitype}`，比如 `"openai/v1/chatcompletions"`、`"anthropic/v1/messages"`、`"gemini/v1beta/generatecontent"`。

> 💡 **设计要点**：OpenAI 是事实标准，所以 ApiName 以 `openai/...` 为基准。其他厂商的接口（Claude 的 messages、Gemini 的 generateContent）单独命名。`ai-proxy` 用 ApiName 决定"按什么协议翻译"。

## 2.4 ★★★ Provider 抽象：基于接口的可选 handler 模式

这是 `ai-proxy` **最核心的设计**，也是整个项目最值得学的架构。理解了它，50+ provider 的代码你就懂了一大半。

**问题**：50+ 个 AI 供应商，每家的 API 都不一样。怎么用一套代码统一管理？

**Higress 的解法**：**一个基础接口 + 一堆可选接口，Provider 想实现哪个就实现哪个，main.go 用类型断言分发。**

看 `provider.go:265-309` 的接口定义：

```go
// 基础接口：所有 provider 必须实现
type Provider interface {
	GetProviderType() string
}

// 下面这些是"可选能力接口"，provider 按需实现：

type RequestHeadersHandler interface {
	OnRequestHeaders(ctx wrapper.HttpContext, apiName ApiName) error
}
type RequestBodyHandler interface {
	OnRequestBody(ctx wrapper.HttpContext, apiName ApiName, body []byte) (types.Action, error)
}
type StreamingResponseBodyHandler interface {
	OnStreamingResponseBody(ctx, name ApiName, chunk []byte, isLastChunk bool) ([]byte, error)
}
type TransformRequestHeadersHandler interface {
	TransformRequestHeaders(ctx, apiName ApiName, headers http.Header)
}
type TransformRequestBodyHandler interface {
	TransformRequestBody(ctx, apiName ApiName, body []byte) ([]byte, error)
}
type TransformResponseHeadersHandler interface { ... }
type TransformResponseBodyHandler interface { ... }
type StreamingEventHandler interface {
	OnStreamingEvent(ctx, name ApiName, event StreamEvent) ([]StreamEvent, error)
}
```

**main.go 怎么用？** 每个阶段都用类型断言判断"当前 provider 实现了这个接口吗"，实现了就调。看 `main.go:283-309`：

```go
if handler, ok := activeProvider.(provider.RequestHeadersHandler); ok {
	// 当前 provider 实现了 RequestHeadersHandler → 调它的 OnRequestHeaders
	providerConfig.SetApiTokenInUse(ctx)
	err := handler.OnRequestHeaders(ctx, apiName)
	...
	_, hasRequestBodyHandler := activeProvider.(provider.RequestBodyHandler)
	hasRequestBody := ctx.HasRequestBody()
	if hasRequestBody && hasRequestBodyHandler {
		// 有 body 且 provider 实现了 body handler → 暂停读 body
		return types.HeaderStopIteration
	}
	ctx.DontReadRequestBody()  // 否则不读 body
	return types.ActionContinue
}
```

**画成图**：

```
       activeProvider (某个具体 provider，如 openaiProvider)
              │
   main.go 在每个阶段做类型断言：
              │
   ┌──────────┼──────────┬──────────┬──────────┐
   ▼          ▼          ▼          ▼          ▼
 实现       实现       实现       没实现      实现
 RequestHeaders  RequestBody  TransformReq  →跳过   StreamingResp
 Handler?  Handler?   BodyHandler?           BodyHandler?
   │          │          │                      │
   ▼          ▼          ▼                      ▼
 调它的     调它的     调它的                调它的
 OnRequestHeaders  OnRequestBody  TransformRequestBody  OnStreamingResponseBody
```

**为什么这个设计牛？**

1. **零侵入扩展**：加一个新 provider，只要实现它需要的接口。比如 OpenAI 协议和目标一致，就只实现 `TransformRequestHeaders`（改 host/path）+ `OnRequestBody`（触发默认翻译）；Claude 协议差很多，就额外实现 `TransformRequestBody`/`TransformResponseBody` 做完整翻译。
2. **provider 之间不互相影响**：每个 provider 是独立的 struct，改一个不会碰别人。
3. **main.go 极其稳定**：不管加多少 provider，main.go 的分发逻辑一行都不用改——它只认接口，不认具体类型。

> 💡 **类比 DeerFlow**：DeerFlow 的中间件靠"实现哪些 hook"来扩展，Higress 的 provider 靠"实现哪些接口"来扩展。**都是"可选能力 + 分发器"模式**——这是处理"大量变体 + 统一调度"的经典解法。

## 2.5 一个 Provider 是怎么造出来的：openai.go 全流程

以最典型的 `openai.go` 为例，看一个 provider 长什么样。

### 2.5.1 注册：providerInitializers

`provider.go:224-262` 是个全局注册表：
```go
providerInitializers = map[string]providerInitializer{
	providerTypeOpenAI:  &openaiProviderInitializer{},
	providerTypeClaude:  &claudeProviderInitializer{},
	providerTypeQwen:    &qwenProviderInitializer{},
	providerTypeDeepSeek:&deepseekProviderInitializer{},
	providerTypeGemini:  &geminiProviderInitializer{},
	... // 50+
}
```

每个 initializer 实现两个方法（`provider.go:211-214`）：
```go
type providerInitializer interface {
	ValidateConfig(*ProviderConfig) error
	CreateProvider(ProviderConfig) (Provider, error)
}
```

`CreateProvider(config)` 是工厂方法：传配置，返回一个 Provider 实例。`provider.go:920-926`：
```go
func CreateProvider(pc ProviderConfig) (Provider, error) {
	initializer, has := providerInitializers[pc.typ]
	if !has {
		return nil, errors.New("unknown provider type: " + pc.typ)
	}
	return initializer.CreateProvider(pc)
}
```

**这就是"注册表 + 工厂"模式**——配置里写 `type: "qwen"`，就去注册表找 `qwenProviderInitializer`，调它的 `CreateProvider` 造一个 qwen provider。

### 2.5.2 openaiProvider 的结构

`openai.go:82-121`：
```go
func (m *openaiProviderInitializer) CreateProvider(config ProviderConfig) (Provider, error) {
	if config.openaiCustomUrl == "" {
		config.setDefaultCapabilities(m.DefaultCapabilities())  // 注册它能处理哪些 API
		return &openaiProvider{
			config:       config,
			contextCache: createContextCache(&config),
		}, nil
	}
	// 有自定义 URL 的分支...
}

type openaiProvider struct {
	config             ProviderConfig
	customDomain       string
	customPath         string
	isDirectCustomPath bool
	contextCache       *contextCache
}
```

### 2.5.3 它实现了哪些接口？

`openai.go` 实现了 4 个：

```go
// 1. 基础接口（必须）
func (m *openaiProvider) GetProviderType() string { return providerTypeOpenAI }

// 2. RequestHeadersHandler —— 请求头阶段：改 path/host、设鉴权
func (m *openaiProvider) OnRequestHeaders(ctx, apiName) error {
	m.config.handleRequestHeaders(m, ctx, apiName)
	return nil
}

// 3. RequestBodyHandler —— 请求体阶段：触发协议翻译
func (m *openaiProvider) OnRequestBody(ctx, apiName, body) (types.Action, error) {
	if !m.config.needToProcessRequestBody(apiName) {
		return types.ActionContinue, nil
	}
	return m.config.handleRequestBody(m, m.contextCache, ctx, apiName, body)
}

// 4. TransformRequestHeadersHandler —— 改请求头（path、host、Authorization）
func (m *openaiProvider) TransformRequestHeaders(ctx, apiName, headers) { ... }

// 5. TransformRequestBodyHandler —— 改请求体（注入 responseJsonSchema）
func (m *openaiProvider) TransformRequestBody(ctx, apiName, body) ([]byte, error) { ... }
```

### 2.5.4 看 TransformRequestHeaders：鉴权托管的精髓

`openai.go:132-203` 是 OpenAI provider 改请求头的逻辑，最能体现"鉴权托管"：

```go
func (m *openaiProvider) TransformRequestHeaders(ctx, apiName, headers) {
	// ① 改 path：用 capability 映射
	if m.isDirectCustomPath {
		util.OverwriteRequestPathHeader(headers, m.customPath)
	} else if apiName != "" {
		util.OverwriteRequestPathHeaderByCapability(headers, string(apiName), m.config.capabilities)
	}

	// ② 改 host：默认 api.openai.com，或自定义
	if m.customDomain != "" {
		util.OverwriteRequestHostHeader(headers, m.customDomain)
	} else {
		util.OverwriteRequestHostHeader(headers, defaultOpenaiDomain)  // "api.openai.com"
	}

	// ③ 鉴权：按优先级找 token
	var token string
	if len(m.config.apiTokens) > 0 {
		token = m.config.GetApiTokenInUse(ctx)         // 优先用配置的 apiTokens
	} else {
		// 没配 token，就从请求头里找（authHeaderKey → x-api-key/x-authorization → Authorization）
		...
	}
	// ④ 设 Authorization: Bearer xxx
	if token != "" {
		if !strings.HasPrefix(token, "Bearer ") { token = "Bearer " + token }
		util.OverwriteRequestAuthorizationHeader(headers, token)
	}
	headers.Del("Content-Length")
}
```

**精髓在哪？** 客户端**不需要知道**真实的 API Key——它随便发个 token（甚至不发），网关用配置好的 `apiTokens` 替换。这就是"鉴权托管"：**API Key 集中在网关管理，客户端零感知**。

## 2.6 协议转换：OpenAI ↔ Claude

很多客户端用 Claude SDK（`/v1/messages`），但后端想接便宜的 DeepSeek（OpenAI 协议）。`ai-proxy` 能自动转换。看 `main.go:248-263`：

```go
// 检测到 Claude 请求，但 provider 不原生支持 Claude 协议
if apiName == provider.ApiNameAnthropicMessages && !providerConfig.IsSupportedAPI(provider.ApiNameAnthropicMessages) {
	// 把 /v1/messages 改成 /v1/chat/completions
	newPath := strings.Replace(path.Path, provider.PathAnthropicMessages, provider.PathOpenAIChatCompletions, 1)
	proxywasm.ReplaceHttpRequestHeader(":path", newPath)
	apiName = provider.ApiNameChatCompletion
	// 标记：响应要转回 Claude 格式
	ctx.SetContext("needClaudeResponseConversion", true)
}
```

然后在请求体阶段（`provider.go:1213-1238`），把 Claude 的 body 转成 OpenAI：
```go
needClaudeConversion, _ := ctx.GetContext("needClaudeResponseConversion").(bool)
if needClaudeConversion {
	// 提取 Claude 的 thinking 配置
	thinkingType := gjson.GetBytes(body, "thinking.type").String()
	ctx.SetContext(ctxKeyClaudeThinkingType, thinkingType)
	// 用 ClaudeToOpenAIConverter 把请求体转成 OpenAI 格式
	converter := &ClaudeToOpenAIConverter{}
	body, err = converter.ConvertClaudeRequestToOpenAIWithOptions(body, ...)
}
```

响应阶段（`main.go:640-666`）再转回 Claude：
```go
func convertStreamingResponseToClaude(ctx, data) ([]byte, error) {
	if !needsClaudeResponseConversion(ctx) { return data, nil }
	// 从 ctx 拿 converter（保持跨 chunk 状态）
	converter := &provider.ClaudeToOpenAIConverter{}
	claudeChunk, err := converter.ConvertOpenAIStreamResponseToClaude(ctx, data)
	return claudeChunk, nil
}
```

**完整流程**：

```
客户端(Claude格式) ──/v1/messages──▶ ai-proxy
                                        │
                         检测: 是Claude请求,但provider不原生支持
                                        │
                         ① 改 path → /v1/chat/completions
                         ② 标记 needClaudeResponseConversion
                         ③ 请求体 Claude→OpenAI 转换
                                        │
                                        ▼
                                  LLM(OpenAI格式响应)
                                        │
                         ④ 流式响应 OpenAI→Claude 转换
                                        │
                                        ▼
客户端(收到Claude格式流) ◀──────────────┘
```

> 💡 **学习启示**：这是"协议适配器"模式——对外保持一种协议（Claude），对内用另一种（OpenAI），中间双向翻译。转换器对象存在 ctx 里保持跨 chunk 状态，这是处理流式转换的关键。

## 2.7 模型映射 modelMapping

客户端发 `model: "gpt-4"`，但你想路由到通义千问的 `qwen-max`。`modelMapping` 配置搞定。看 `provider.go:1022-1061` 的 `doGetMappedModel`：

```go
func doGetMappedModel(model string, modelMapping map[string]string) string {
	// ① 精确匹配
	if v, ok := modelMapping[model]; ok { return v }

	for k, v := range modelMapping {
		// ② 前缀通配：key 以 * 结尾
		if strings.HasSuffix(k, wildcard) {
			k = strings.TrimSuffix(k, wildcard)
			if strings.HasPrefix(model, k) { return v }
		}
		// ③ 正则：key 以 ~ 开头
		if strings.HasPrefix(k, "~") {
			re := regexp.MustCompile(strings.TrimPrefix(k, "~"))
			if re.MatchString(model) { return re.ReplaceAllString(model, v) }
		}
	}
	// ④ 全局通配 *
	if v, ok := modelMapping[wildcard]; ok { return v }
	return ""
}
```

**四级匹配**：精确 > 前缀 > 正则 > 通配。正则那条特别强——`~gpt-(\d+)` → `qwen-max-$1` 能把 `gpt-4` 映射成 `qwen-max-4`。

## 2.8 API Token 管理：failover 与 retry

生产环境一个 provider 配多个 API Key（token），某个挂了要自动切。`provider.go` 有一套完整机制：

**① 随机选择 + 消费者亲和**（`provider.go:813-832`）：
```go
func (c *ProviderConfig) selectApiToken(ctx) string {
	ctxApiName := ctx.GetContext(CtxKeyApiName)
	apiName := string(ctxApiName.(ApiName))
	// 有状态 API（如 Responses、Files）用消费者亲和，保证同一消费者固定用同一 token
	if isStatefulAPI(apiName) {
		consumer := c.getConsumerFromContext(ctx)  // 从 x-mse-consumer 头取
		if consumer != "" {
			return c.GetTokenWithConsumerAffinity(ctx, consumer)  // FNV hash 固定选一个
		}
	}
	return c.GetRandomToken()  // 无状态 → 随机
}
```

**② failover**（`provider.go:324` 的 `failover` 字段）：token 连续失败超阈值就移出列表，定期健康检查，恢复后加回。

**③ retryOnFailure**（`provider.go:329`）：失败立即重试别的 token。

**④ 响应阶段健康检查**（`main.go:376-396`）：上游返回非 200 时调 `OnRequestFailed`，决定要不要 failover：
```go
status, _ := proxywasm.GetHttpResponseHeader(":status")
if err != nil || status != "200" {
	action := providerConfig.OnRequestFailed(activeProvider, ctx, apiTokenInUse, apiTokens, status)
	...
}
// 成功了就重置失败计数
providerConfig.ResetApiTokenRequestFailureCount(apiTokenInUse)
```

> 💡 **设计要点**：有状态 API（Files、Batches、FineTuning）用消费者亲和，是因为这些 API 的状态绑在特定账号上——同一消费者的后续请求必须打到同一 token，否则找不到之前上传的文件。这是个很细致的工程考量。

## 2.9 流式响应处理：SSE 事件提取

LLM 流式响应是 SSE 格式（`event:xxx\ndata:{...}\n\n`）。`ai-proxy` 要逐 chunk 解析、翻译。核心是 `provider.go:1091-1167` 的 `ExtractStreamingEvents`：

```go
func ExtractStreamingEvents(ctx, chunk []byte) []StreamEvent {
	body := chunk
	// ① 拼上之前 buffer 的不完整数据
	if bufferedStreamingBody, has := ctx.GetContext(ctxKeyStreamingBody).([]byte); has {
		body = append(bufferedStreamingBody, chunk...)
	}
	// 统一换行符
	body = bytes.ReplaceAll(body, []byte("\r\n"), []byte("\n"))

	defer func() {
		// ④ 把没解析完的尾巴存回 ctx，等下个 chunk
		if eventStartIndex >= 0 {
			ctx.SetContext(ctxKeyStreamingBody, body[eventStartIndex:])
		}
	}()

	// ② 逐字符状态机解析
	var events []StreamEvent
	for i := 0; i < length; i++ {
		ch := body[i]
		// 按 \n 分行，按 : 分 key:value
		// 遇到空行 → 一个 event 完成
	}
	return events
}
```

**两个关键设计**：
1. **跨 chunk 拼接**：SSE event 可能被网络切断，`ctxKeyStreamingBody` buffer 不完整部分，下个 chunk 拼上；
2. **手写状态机解析**：不依赖外部 SSE 库（Wasm 环境受限），逐字符解析，处理 `\r`/`\n`/`\r\n` 三种换行。

main.go 的 `onStreamingResponseBody`（`main.go:426-533`）拿到 events 后，调 provider 的 `OnStreamingResponseBody` 或 `OnStreamingEvent` 逐个翻译。

## 2.10 小结：ai-proxy 的架构精髓

```
┌──────────────────────────────────────────────────────────────┐
│  ai-proxy = main.go(调度器) + provider/(50+ 变体)             │
│                                                              │
│  main.go 的职责（稳定不变）：                                  │
│   - 5 个 phase hook 分发                                      │
│   - ApiName 路由（URL → 接口类型）                             │
│   - Claude/OpenAI 协议自动转换编排                            │
│   - 流式 SSE 解析调度                                          │
│                                                              │
│  provider 的职责（每个变体不同）：                             │
│   - 实现需要的可选接口（OnRequestHeaders/TransformReqBody...）│
│   - 改 path/host/Authorization（鉴权托管）                    │
│   - 协议翻译（请求/响应、流式/非流式）                         │
│   - 模型映射                                                  │
│                                                              │
│  ★ 核心模式：基础接口 + 可选接口 + 类型断言分发                │
│  ★ 加新 provider：实现接口，注册到 providerInitializers，完事  │
└──────────────────────────────────────────────────────────────┘
```

**5 句话记住**：
1. **ai-proxy 是协议翻译器**——对外 OpenAI 协议，对内各家方言，中间双向翻译；
2. **Provider 是"可选接口"组合**——实现哪些接口就具备哪些能力，main.go 用类型断言分发；
3. **50+ provider 用注册表+工厂管理**——`providerInitializers` map，配置 `type` 决定造哪个；
4. **鉴权托管**——客户端不用知道 API Key，网关用 `apiTokens` 替换，支持 failover/retry/消费者亲和；
5. **流式转换靠 ctx 保状态**——`ClaudeToOpenAIConverter`、`ctxKeyStreamingBody` 都存 ctx，跨 chunk 保持连续。

---

# 📖 第 3 章 AI 流量治理插件

`ai-proxy` 解决"能不能通"，这章的插件解决"通得好不好"——负载均衡、限流、配额、分流。

## 3.1 ai-load-balancer：AI 感知的负载均衡

普通 LB 按连接数/请求数分配，但 AI 场景有特殊需求：**前缀缓存亲和**。vLLM 等推理引擎会缓存对话前缀的 KV cache，如果连续对话路由到同一 pod，能复用缓存，大幅降延迟。

`ai-load-balancer` 提供 5 种策略（`main.go:41-93`）：

| lb_policy | 干什么 |
|---|---|
| `prefix_cache` | 按对话前缀哈希路由，最大化 KV cache 复用 |
| `global_least_request` | 全局最少在途请求 |
| `endpoint_metrics` | 按 pod 实时指标（vLLM /metrics）调度 |
| `cluster_metrics` | 集群级指标调度 |
| `cluster_hash` | 集群级哈希 |

### 3.1.1 prefix_cache 的精髓：前缀树亲和

`prefix_cache/lb_policy.go` 把对话历史按 `user` 轮切成段，每段 SHA1 哈希，传给 Redis Lua 脚本找"最长已存在前缀"：

```go
// 拼接 role:content，按 user 轮切段，每段 SHA1
for _, msg := range messages {
	segment := msg.Role + ":" + content
	hashes = append(hashes, sha1(segment))
}
// Redis Lua：XOR 相邻哈希形成树路径，找最长已存在前缀 → 那个 host
redis.Eval(luaScript, hashes)
// 选中的 host
proxywasm.SetUpstreamOverrideHost(selectedHost)
```

**为什么用 Redis + Lua？** 因为"找最长前缀 + 选 host + 计数"必须是**原子操作**，否则并发下会选错。Lua 脚本在 Redis 里原子执行。

**注意**：`HandleHttpRequestHeaders` 返回 `HeaderStopIteration`（暂停），因为 `SetUpstreamOverrideHost` 只在 header 暂停时生效。流结束后 `HandleHttpStreamDone` 把在途计数 -1。

### 3.1.2 endpoint_metrics：vLLM 指标感知

读每个 pod 的 `/metrics`（vLLM 暴露的 Prometheus 指标，如 `vllm:num_requests_running`），按指标调度，还有 `maxRate` 本地限速防止单 pod 过载。

> 💡 **学习启示**：AI 负载均衡的精髓是"**业务感知**"——前缀缓存亲和、推理队列长度、显存占用，这些是通用 LB 不懂的。把业务语义注入调度策略，是 AI 网关区别于普通网关的关键。

## 3.2 ai-token-ratelimit：按 token 限流

普通限流按"请求数"，但 AI 一个请求可能消耗 10 token 也可能 10000 token。按 token 限流才合理。难点：**请求来时还不知道消耗多少 token**（响应还没生成）。

`ai-token-ratelimit` 的解法是**两阶段固定窗口**（`main.go`）：

```
请求阶段(onHttpRequestHeaders):
  Lua 脚本 GET 当前窗口已用 token（只读，不扣）
  if 已用 > 阈值 → 拒绝（429）
  else → ResumeHttpRequest 放行
  return HeaderStopAllIterationAndWatermark（暂停）

响应阶段(onHttpStreamingBody):
  用 tokenusage 累加本次 input+output token
  流结束(endOfStream)时:
  Lua 脚本 INCRBY 真正扣减（只在 current<=阈值 时扣）
```

**关键设计**：请求时**只查不扣**（预检），响应时**按实际用量扣**（实扣）。这样 counter 反映真实消耗，不会因为预扣过多而误杀。

Lua 脚本保证原子性：`RequestPhaseFixedWindowScript`（只读 GET+TTL）、`ResponsePhaseFixedWindowScript`（条件 INCRBY+EXPIRE）。

## 3.3 ai-quota：长效配额

和限流的"时间窗口"不同，配额是"总额度"——某消费者总共能用 100 万 token，用完为止。`ai-quota/main.go`：

- **扣减**：响应流结束时 `redisClient.DecrBy(prefix+consumer, totalToken)`；
- **预检**：请求时 `Get` 配额，≤0 就 403 "No quota left"；
- **admin API**：巧妙地复用路由路径，`/v1/chat/completions/quota?refresh` 刷新配额，`/quota/delta` 增减配额。`getOperationMode`（`main.go:280-297`）按 URL 后缀区分模式。

**消费者身份**来自 `x-mse-consumer` 头（由鉴权插件设置）——这是 Higress 生态的统一约定，ai-statistics、ai-security-guard 都用它。

## 3.4 model-router / model-mapper：模型分流

两个轻量插件，都只 hook 请求阶段：

**model-router**（`main.go`）：把 `model: "provider/model"` 拆开，provider 写到 `x-higress-llm-model` 头供路由匹配。还有 `autoRouting`——当 `model=higress/auto` 时，用正则匹配用户消息内容选模型（意图路由）：

```go
if config.enableAutoRouting && modelValue == AutoModelPrefix {
	userMessage := extractLastUserMessage(body)  // 取最后一条 user 消息
	if matchedModel, found := matchAutoRoutingRule(config, userMessage); found {
		// 正则匹配用户消息 → 选模型
		proxywasm.ReplaceHttpRequestHeader("x-higress-llm-model", targetModel)
	}
}
```

**model-mapper**（`main.go`）：纯模型名映射（精确/前缀/通配），把映射结果写到 `x-higress-llm-model-final` 头。注意它**按 key 字母序排序**再匹配（`main.go`），为了复刻 C++ 版 nlohmann::json 的迭代顺序——保证跨语言行为一致。

> 💡 **三者关系**：`model-router` 管"路由到哪个 provider"（改路由头），`model-mapper` 管"模型名最终是什么"（改 body+头），`ai-proxy` 的 `modelMapping` 是 provider 内部的映射。层层叠加。

---

# 📖 第 4 章 AI 增强插件

这章的插件最像"Agent 开发"——它们有的调 LLM、调检索、改 prompt、做循环。是学习 Agent 应用开发最好的素材。

## 4.1 ai-cache：精确 + 语义向量缓存

**目标**：相同问题直接返回缓存，省 token 省延迟。难点是"相同"怎么定义——字面相同？还是语义相似？

`ai-cache` 提供两级：

1. **精确缓存**：用 `cacheKeyStrategy`（`lastQuestion`/`allQuestions`）算 key，Redis GET；
2. **语义缓存**：`enableSemanticCache` 开启后，缓存 miss 时把 query 做 embedding，去向量库（Milvus/Qdrant/Chroma...）找相似度 ≥ threshold 的历史问答。

**核心流程**（`core.go`）：

```
请求来 → 算 cacheKey → Redis GET
  ├─ 命中 → SendHttpResponse(缓存答案，按 OpenAI 格式包装)  ★ 短路
  └─ 未命中 → 若开语义缓存:
        embeddingProvider.GetEmbedding(key)  → 向量化 query
        vectorProvider.QueryEmbedding(vec)   → 相似检索
        if top1.score ≥ threshold → 返回相似答案
        else → 放行到 LLM
放行后 → 响应流式累加完整答案 → Redis SET + 向量库存 embedding+answer
```

**三个可插拔 provider**（接口隔离）：
- `cache`：Redis（存键值对）；
- `embedding`：DashScope/OpenAI/Cohere...（算向量）；
- `vector`：Milvus/Qdrant/Chroma/Elasticsearch/Pinecone...（向量检索）。

**设计要点**：
- 响应模板化（`ResponseTemplate`/`StreamResponseTemplate`）——缓存命中时把答案包装成 LLM 的 wire format，客户端无感知；
- `x-higress-skip-ai-cache: on` 头可跳过缓存；
- tool_calls 响应不缓存（`processSSEMessage` 检测到 tool_calls 就跳过）。

> 💡 **Agent 开发启示**：语义缓存是 RAG 的反面——RAG 是"检索增强生成"，语义缓存是"检索替代生成"。两者共享 embedding + 向量检索的基础设施。

## 4.2 ai-rag：检索增强生成

最简 RAG 模式。`ai-rag/main.go` 的 `onHttpRequestBody`：

```
取最后一条 user 消息 → DashScope text-embedding-v2 向量化
  → DashVector 查 topk 相关文档（score ≤ threshold）
  → 改写请求：删掉原最后 user 消息
              插入检索到的文档作为多条 user 消息
              再追加原问题 "现在，请回答以下问题:\n<query>"
  → ReplaceHttpRequestBody → ResumeHttpRequest
```

**两级异步回调嵌套**（embed → query → rewrite），全程 `ActionPause` + `Resume`。只调 embedding 服务，不调 chat LLM（chat LLM 是路由上游）。

## 4.3 ai-search：LLM 改写 + 多引擎搜索

比 ai-rag 复杂得多。`ai-search/main.go`：

1. **查询改写**：可选地用 LLM 把用户问题拆成多个子查询（`searchRewrite`），按引擎类型分（`internet:`/`private:`/`arxiv-category:`）；
2. **并行搜索**：每个引擎（Bing/Google/arxiv/Elasticsearch/Quark）实现 `engine.SearchEngine` 接口，`executeSearch` 并行 fan-out，结果去重；
3. **prompt 填充**：把搜索结果填进模板（`{search_results}`/`{question}`/`{cur_date}`），替换请求 body；
4. **引用注入**：响应阶段把引用列表追加到答案，还要处理 `<think>...</think>` 推理块（引用放 `</think>` 之后），流式跨 chunk 检测半截 `</think>` 标签。

**引擎接口**（`engine/types.go`）让搜索后端可插拔，是策略模式的典型。

## 4.4 ai-history：对话历史注入

按消费者身份（`Authorization` 头）把历史存 Redis，下次请求 prepend 最近 N 轮。`ai-history/main.go`：

- 请求时 `redisClient.Get(prefix+identity)` 取历史，prepend 到 messages；
- 响应流式累加答案，结束时存回 Redis（`{user, assistant}` 对，trim 到 `FillHistoryCnt*2`）；
- 还复用路由路径提供 `/ai-history/query` 查询端点。

## 4.5 ai-prompt-decorator / ai-template / ai-transformer

**ai-prompt-decorator**：在 messages 前插固定消息、后插固定消息、对内容做正则/字面替换，还支持地理变量 `${geo-city}`（配合 geo-ip 插件）。

**ai-prompt-template**：服务端命名模板，客户端发 `{template:"简历", properties:{name:"张三"}}`，服务端展开成完整 prompt。`{{key}}` 占位符替换。

**ai-transformer**（最特别）：用 LLM 改写**整个 HTTP 报文**（头+体）。把当前请求头+体序列化成字符串，喂给 LLM（qwen-max）让它按自然语言 prompt 重写，再把 LLM 输出解析回头+体替换。请求/响应对称处理。**这是"LLM as transformer"的极端案例**——连报文改写都交给 LLM。

## 4.6 ai-context-limit：输入 token 预检

在请求到达 LLM **之前**估算 token 数，超限直接拒（模拟 OpenAI 的 `context_length_exceeded` 错误）。`ai-context-limit/tokenizer.go`：

```go
//go:embed bpe/o200k_base.tiktoken  // ★ 嵌入式 BPE 词表
```

**为什么嵌入词表？** Wasm 环境运行时不能下载文件，所以把 tiktoken 的 `o200k_base` 词表用 `//go:embed` 编译进二进制，离线分词。乘 `buffer_ratio`（默认 1.10）留余量。多模态请求（含图片）直接放行（无法估算）。

## 4.7 ai-json-resp：强制 JSON + 自纠正循环

强制 LLM 输出符合 JSON Schema 的结果，不对就让它自己改。`ai-json-resp/main.go` 的 `recursiveRefineJson`：

```
调 LLM → 提取 JSON → 校验 Schema
  ├─ 合格 → 返回
  └─ 不合格 → 把错误信息作为 system 消息追加 → 再调 LLM "请修正" → 递归（最多 maxRetry 次）
```

**重入检测**：用 `X-HIGRESS-AI-JSON-RESP: true` 头标记，防止无限递归。这个插件**自己调 LLM**（替换原上游），所以要替换鉴权。

## 4.8 ai-intent：意图识别分流

调一个分类 LLM 把用户问题分类（如"金融|电商|法律"），结果通过 `proxywasm.SetProperty([]string{"intent_category"}, ...)` 发布成 Envoy 属性，供下游路由/打标用。**不修改原请求**，纯旁路分类。

## 4.9 ai-agent：ReAct 工具调用循环 ★

这是最像"Agent"的插件——把网关变成自主 Agent。`ai-agent/main.go` 实现 ReAct（Reason+Act）循环：

```
用户问题 → 填 ReAct prompt（列出工具描述）→ 调 LLM（强制 stream=false）
  ↓
解析 LLM 输出: "Action: xxx\nAction Input: yyy\nFinal Answer: zzz"
  ├─ Final Answer → 完成（可选再调 LLM 格式化成 JSON）
  └─ Action → 找到对应工具 API → HTTP 调用 → 拿到结果
        → 包装成 "Observation: 结果" 追加到对话
        → 再调 LLM → 递归解析（最多 maxIterations 次）
```

**关键设计**：
- **工具来自 OpenAPI spec**：`initAPIs`（`config.go:248-345`）解析配置里的 OpenAPI 3.0 spec，提取路径/方法/参数成工具；
- **自己调 LLM**：建 `LLMClient`（`wrapper.HttpClient`），POST 到配置的 LLM 服务；
- **内存对话存储**：`dashscope.MessageStore` 存对话历史；
- **格式还原**：原请求可能是流式的，agent 内部强制非流式，最后再把答案包装成流式格式返回。

> 💡 **Agent 开发启示**：ReAct 是最经典的 Agent 模式——"思考→行动→观察→再思考"循环。`ai-agent` 把这个循环做到了网关层，客户端发一个问题，网关自己多轮调 LLM+工具，直到得出答案。和 DeerFlow 的 LangGraph 图是两种 Agent 实现风格：网关式（声明式配置）vs 框架式（代码编排）。

---

# 📖 第 5 章 AI 安全与可观测

## 5.1 ai-security-guard：双向内容审核

唯一 hook **全部五个阶段**的插件——请求和响应都要审。`main.go` 按 `action` 分发到 `text_moderation_plus` 或 `multi_modal_guard` handler。

**双向审核**：
- **请求阶段**：提取用户 prompt，调阿里云绿网（lvwang）内容安全服务，命中风险就拦截/脱敏；
- **响应阶段**：流式累加 LLM 输出，提交审核，命中就注入拦截响应。

**风险决策**（`config.go:798-832`）有 5 级优先级：消费者-维度 > 消费者-全局 > 全局-维度 > 全局-全局 > 默认 block。支持 per-consumer 策略（exact/prefix/regexp 匹配）。

**响应格式**：`legacy` 把拒答放 `message.content`，`structured` 嵌 `x_higress_guardrail` 块——为兼容不同客户端 SDK。

## 5.2 ai-statistics：token / 延迟 / 链路追踪

`ai-statistics/main.go`（单文件 1430 行）是可观测核心。采集：

- **token 用量**：input/output/total token，按 `route.cluster.model.consumer` 维度打 counter；
- **首字延迟** `LLMFirstTokenDuration`：第一个 chunk 到达时间；
- **服务延迟** `llm_service_duration`：总耗时；
- **ARMS span 属性**：`gen_ai.request.model`、`gen_ai.usage.*_tokens`；
- **问答内容**、**tool_calls**、**reasoning**。

**核心抽象**：声明式 attribute DSL——每个 attribute 配 `value_source`（request_header/body、response_streaming_body...）+ gjson path + rule（first/replace/append）。`shouldBufferStreamingBody` 在配置时就算出"要不要 buffer 流式体"，避免无谓 buffer。

**token 解析**靠共享的 `pkg/tokenusage.GetTokenUsage(ctx, data)`——ai-token-ratelimit、ai-quota 也用它。这是"token 计费"的统一基础设施。

## 5.3 ai-data-masking：敏感数据脱敏（Rust）

`plugins/wasm-rust/extensions/ai-data-masking/`——用 Rust 写的 Wasm 插件，对请求/响应里的敏感信息（手机号、身份证...）脱敏。**为什么用 Rust？** 脱敏要正则匹配大量文本，Rust 的正则引擎性能远超 Go，且 Wasm 沙箱里 Rust 内存安全。这体现了"按场景选语言"的工程智慧。

---

# 📖 第 6 章 MCP Server 托管

MCP（Model Context Protocol）是 Anthropic 提出的"AI 友好的 API"标准——让 AI Agent 能统一调用各种工具。Higress 能把现有 API 托管成 MCP Server，这是它作为 AI 网关的前沿能力。

## 6.1 MCP 是什么，为什么网关要托管

```
传统：AI Agent 要调 10 个工具 → 每个 API 协议不同 → Agent 写 10 套适配
MCP：  AI Agent 统一用 MCP 协议 → 网关把 10 个 API 包装成 MCP 工具 → Agent 一套协议搞定

┌──────────┐   MCP协议   ┌───────────┐   各家API   ┌────────┐
│ AI Agent │ ──────────▶ │  Higress  │ ──────────▶ │ REST/  │
│          │  tools/list │  (MCP托管) │  tools/call │ DB/... │
└──────────┘  tools/call └───────────┘             └────────┘
```

Higress 在 MCP 托管上叠加了它擅长的：统一鉴权、限流、审计、可观测、无损热更新。

## 6.2 两条实现栈

Higress 有**两套** MCP 实现，这是个值得深究的工程决策：

| 栈 | 目录 | 运行时 | 适用 |
|---|---|---|---|
| **Wasm 栈** | `wasm-go/extensions/mcp-server/` + `mcp-router/` + `pkg/mcp/` | proxy-wasm | 轻量工具托管、REST→MCP、MCP代理 |
| **原生 Go 栈** | `golang-filter/mcp-server/` + `mcp-session/` | Envoy Go filter | 重型工具（DB）、SSE 长连接 fan-out |

### 6.2.1 Wasm 栈：mcp-server SDK

`pkg/mcp/server/plugin.go` 是 SDK 核心。`server.Initialize()`（`plugin.go:673`）注册到 wrapper：

```go
wrapper.SetCtx("mcp-server",
	wrapper.PrePluginStartOrReload[McpServerConfig](onPluginStartOrReload),
	wrapper.ParseConfigWithContext(parseConfig),
	wrapper.ProcessRequestHeaders(onHttpRequestHeaders),
	wrapper.ProcessRequestBody(onHttpRequestBody),
	wrapper.ProcessResponseHeaders(onHttpResponseHeaders),
	wrapper.ProcessStreamingResponseBody(onHttpStreamingResponseBody),  // SSE 代理
	...
)
```

**四种 Server 实现**（`pkg/mcp/server/`）：

| 实现 | 文件 | 干什么 |
|---|---|---|
| `MCPServer` | `base_server.go` | 托管编译期注册的 Go 工具（如 quark-search） |
| `RestMCPServer` | `rest_server.go` | **REST→MCP**：用 Go 模板把 REST API 变成 MCP 工具 |
| `McpProxyServer` | `proxy_server.go` | **MCP→MCP 代理**：转发到后端 MCP server |
| `ComposedMCPServer` | `composed_server.go` | **toolSet 聚合**：把多个 server 的工具合成一个端点 |

**JSON-RPC 处理**（`pkg/mcp/utils/json_rpc.go`）：`HandleJsonRpcMethod` 解析 `id/method/params`，调注册的 handler。关键支持**异步暂停**——handler 设 `CtxNeedPause=true` 返回 `ActionPause`，等异步后端调用完成再 resume。这是 MCP 代理多轮握手（initialize→notify→call）的基础。

**协议版本协商**（`plugin.go:511`）：支持 `2024-11-05`、`2025-03-26`、`2025-06-18` 三个版本，客户端要的版本不支持就降级到最新。

### 6.2.2 Wasm 栈：mcp-router（toolSet 路由）

`mcp-router/main.go` 是 `ComposedMCPServer` 的配套——当多个 server 聚合成一个端点，工具名变成 `${serverName}___${toolName}`（`___` 是分隔符）。客户端调这个组合名时，mcp-router 拆开、改写 `:authority`/`:path`/`params.name`，让 Envoy 重新路由到正确的后端 server：

```go
parts := strings.SplitN(toolName, consts.ToolSetNameSplitter, 2)  // 拆 "server___tool"
// 找到 server 配置
proxywasm.ReplaceHttpRequestHeader(":authority", targetServer.Domain)
proxywasm.ReplaceHttpRequestHeader(":path", targetServer.Path)
sjson.SetBytes(rawBody, "params.name", actualToolName)  // 改回真实工具名
```

### 6.2.3 原生 Go 栈：mcp-server + mcp-session

为什么还要原生 Go 栈？因为 MCP 的 SSE 长连接需要**跨 worker fan-out**：

```
客户端开一个 SSE 长连接(GET /sse) ←── 一个 Envoy worker 持有
客户端发 tools/call(POST /message) ←── 可能落到另一个 worker
                                      │
                          怎么把结果送回 SSE 连接？
```

**解法**：Redis pub/sub 做 SSE 投递总线。`mcp-session/common/sse.go` 的 `HandleSSE`：
1. 生成 sessionId，订阅 Redis 频道 `mcp-server-sse:<sessionId>`；
2. 任何 worker 处理 POST tools/call 后，把结果 `Publish` 到这个频道；
3. 持有 SSE 连接的 worker 收到消息，`InjectData` 推给客户端。

**这样 SSE 长连接和消息处理解耦**——不管 POST 落到哪个 worker，结果都能送到 SSE 连接。这是横向扩展的关键。

`mcp-session` 还管：路径重写（SSE upstream 的 endpoint URL 改回网关路径）、per-user 配置（`x-higress-mcpserver-config` base64 头）、per-user 限流（Redis Lua 固定窗口）。

`mcp-server`（原生）托管**进程内 Go 工具**（gorm 查库、rag 检索、higress-api...），用 `init()` 注册到 `GlobalRegistry`。请求直接 `SendLocalReply` 短路，**不转发上游**。

## 6.3 两条栈怎么选

```
要托管的是……
   │
   ├─ 轻量 HTTP 工具 / REST API → Wasm 栈 RestMCPServer（隔离好、热更新）
   ├─ 已有 MCP server 要代理 → Wasm 栈 McpProxyServer
   ├─ 进程内重型工具（查 DB）→ 原生栈 mcp-server（性能好、能直接连库）
   └─ 需要 SSE 跨 worker fan-out / 大规模 → 原生栈 mcp-session（Redis 总线）
```

> 💡 **工程启示**：同一能力两套实现不是重复，是"按场景选运行时"——Wasm 胜在隔离和热更新，原生 Go 胜在性能和长连接。这种取舍在大型系统里很常见。

---

# 🎯 自测：你掌握了吗？

用这些问题检验理解。先自己想，再对照下面的提示。

**问题 1**：一个 Higress Wasm 插件的"五要素"是什么？
<details><summary>提示</summary>
名字（SetCtx）、配置（ParseConfig）、阶段 hook（ProcessRequestHeaders...）、HttpContext（请求口袋）、异步模式（ActionPause→Resume）。</details>

**问题 2**：ai-proxy 为什么用"基础接口 + 可选接口 + 类型断言分发"而不是继承？
<details><summary>提示</summary>
Go 没有继承；可选接口让 provider 按需实现能力，零侵入扩展；main.go 只认接口不认具体类型，加新 provider 不改分发逻辑。</details>

**问题 3**：ai-token-ratelimit 为什么是"两阶段"（请求只查不扣，响应实扣）？
<details><summary>提示</summary>
请求来时不知道消耗多少 token（响应没生成）。请求阶段只读预检防超发，响应阶段按实际 input+output token 扣减，counter 反映真实消耗。</details>

**问题 4**：ai-cache 命中缓存时，怎么让客户端"无感知"？
<details><summary>提示</summary>
用 ResponseTemplate/StreamResponseTemplate 把缓存答案包装成 LLM 的 wire format（OpenAI 的 choices.0.message.content 或流式 delta），客户端以为是真的 LLM 响应。</details>

**问题 5**：MCP 的 SSE 长连接，为什么原生栈要用 Redis pub/sub？
<details><summary>提示</summary>
SSE 长连接(GET)和一个 tools/call(POST)可能落到不同 Envoy worker。Redis 频道做投递总线，任何 worker 处理完 POST 都能 Publish 到 sessionId 频道，持有 SSE 的 worker 订阅后 InjectData 推给客户端。横向扩展必需。</details>

**问题 6**：ai-proxy 的 Claude↔OpenAI 协议转换，跨 chunk 状态怎么保持？
<details><summary>提示</summary>
转换器对象（ClaudeToOpenAIConverter）和流式 buffer（ctxKeyStreamingBody）都存在 HttpContext 里，同一个请求的每个 chunk 都能拿到同一个 converter 实例，保持状态连续。</details>

---

# 📝 一页纸总结

```
┌─────────────────────────────────────────────────────────────────┐
│  Higress AI 网关 = Istio/Envoy 底座 + Wasm 插件生态               │
│  对外统一 OpenAI 协议，对内接 50+ AI 供应商                        │
└─────────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
   第1章 插件骨架        第2章 ai-proxy心脏    第3-6章 叠加能力
   ┌────────────┐      ┌─────────────────┐   ┌──────────────────┐
   │①名字 SetCtx │      │ main.go 调度器   │   │ 流量治理:          │
   │②配置 Parse  │      │  5个phase hook   │   │  load-balancer    │
   │③阶段hook    │      │  ApiName路由     │   │  token-ratelimit  │
   │  ReqHdr     │      │  Claude↔OpenAI   │   │  quota            │
   │  ReqBody    │      │  SSE流式调度     │   │  model-router     │
   │  RespHdr    │      ├─────────────────┤   │ 增强:             │
   │  StreamResp │      │ provider 抽象    │   │  cache(语义)      │
   │  RespBody   │      │  基础接口+可选   │   │  rag/search       │
   │④HttpContext │      │  接口+类型断言   │   │  history          │
   │⑤异步模式    │      │  50+注册表工厂   │   │  prompt-*/transf  │
   │ Pause→Resume│      │  鉴权托管        │   │  context-limit    │
   │            │      │  failover/retry  │   │  json-resp        │
   │            │      │  模型映射4级     │   │  intent/agent(ReAct)│
   └────────────┘      └─────────────────┘   │ 安全可观测:        │
                                              │  security-guard   │
                                              │  statistics       │
                                              │  data-masking     │
                                              │ MCP托管(两栈):     │
                                              │  Wasm: server/router│
                                              │  原生: server/session│
                                              └──────────────────┘
```

**7 句话记住**：

1. **所有插件共享同一骨架**——`SetCtx` 注册 + 阶段 hook + HttpContext + 异步 Pause/Resume；
2. **ai-proxy 是协议翻译器**——基础接口 + 可选接口 + 类型断言分发，50+ provider 用注册表+工厂管理；
3. **鉴权托管**——客户端不知 API Key，网关用 apiTokens 替换，支持 failover/retry/消费者亲和；
4. **流式转换靠 ctx 保状态**——converter 和 streamingBody 都存 ctx，跨 chunk 连续；
5. **AI 流量治理"业务感知"**——按 token 限流（两阶段）、前缀缓存路由、vLLM 指标调度；
6. **增强插件分三类**——纯改写（prompt-*）、旁路观察（statistics）、自己调 LLM（agent/transformer/json-resp/intent/search）；
7. **MCP 两栈按场景选**——Wasm 胜隔离热更新，原生 Go 胜性能和 SSE 跨 worker fan-out（Redis 总线）。

---

# 🚀 接下来怎么学？

### 1. 动手跑
按 README 用 Docker 启动 all-in-one，配一个 ai-proxy 接通义千问，用 curl 发 OpenAI 格式请求看效果。

### 2. 改插件
在 `hello-world` 基础上加一个"打印请求 model 字段"的 hook，编译成 Wasm，挂到网关看日志。

### 3. 深读源码（推荐顺序）
- **必读**：`ai-proxy/main.go`（调度）→ `provider/provider.go`（接口）→ `provider/openai.go`（最简 provider）→ `provider/claude.go`（复杂翻译）
- **进阶**：`ai-cache/core.go`（语义缓存）→ `ai-agent/main.go`（ReAct 循环）→ `ai-load-balancer/prefix_cache/lb_policy.go`（前缀树）
- **前沿**：`pkg/mcp/server/plugin.go`（MCP SDK）→ `golang-filter/mcp-session/filter.go`（SSE fan-out）

### 4. 对照 DeerFlow 学 Agent
Higress 的 `ai-agent`（网关式声明 Agent）和 DeerFlow 的 LangGraph（框架式代码 Agent）是两种 Agent 范式，对照读能加深对"Agent 怎么造"的理解。

---

**到这里，Higress 作为 AI 网关的能力地图和核心源码你应该有谱了。** 🎉

如果某个插件或某段代码想更细地拆（比如"prefix_cache 的 Redis Lua 脚本逐行讲"或"claude_to_openai 转换器细节"），告诉我具体哪段，我可以再展开。
