# Run Event 完整性保护方案演进记录

本文记录围绕 DeerFlow `run_events` 完整性保护的方案讨论过程：最初考虑过哪些方向，为什么逐步否掉，最后如何收敛到一个更小、更稳定、适合提交给维护者讨论的方案。

## 1. 背景问题

DeerFlow 当前已经有 `RunJournal` 和 `RunEventStore`：

- `RunJournal` 通过 LangChain callback 捕获 run 过程中的事件。
- `RunEventStore` 负责持久化这些事件。
- 当前 `run_events.backend` 支持 `memory`、`db`、`jsonl`。
- `db` 后端实际复用 DeerFlow 的 SQLAlchemy 持久层，当前数据库支持 `sqlite` 和 `postgres`。
- `jsonl` 后端按 `.deer-flow/threads/{thread_id}/runs/{run_id}.jsonl` 存储。

源码事实上，`RunJournal` 记录的是粗粒度事件，不是 token 级事件：

- `run.start`
- `run.end`
- `run.error`
- `llm.human.input`
- `llm.ai.response`
- `llm.tool.result`
- `llm.error`
- `middleware:{tag}`

`RunJournal` 明确没有实现 `on_llm_new_token`，因此 run event 数量通常不会像 tracing span 或 token stream 那样高频。

核心问题是：

> 如果这些 run event 被误删、误改、重排，或者在事故复盘/审计前被静默篡改，DeerFlow 当前缺少一种事后验证机制来判断 run history 是否仍可信。

## 2. 最早方案：完整 governance / guardrail 扩展

一开始讨论的方向较大：把安全治理扩展到 tool、MCP、skill install/update、memory write、final report/export 等关键动作。

这个方向的问题是：

- 范围太广，已经接近一个治理平台。
- 需要定义大量 action type、策略模型、hook 点和审计语义。
- MCP、tool、memory、export 等动作的风险模型不同，很难用一个生命周期机制统一解释。
- 对普通开发者来说，PR 面太大，不适合一次性提交。

因此这个方向被否掉。

结论：

> 不应该把 DeerFlow 当成通用安全治理框架来改。需要先找一个窄而稳定的落点。

## 3. 第二个方向：面向多用户和 MCP 权限治理

随后讨论过从 DeerFlow roadmap 中的多用户、SSO、RBAC、MCP/tools auditing 等方向出发，提出多用户场景下的 agent/tool/MCP 权限管控。

这个方向的价值是存在的：

- 多用户模式下，不同用户可能能访问不同 agent。
- 不同 agent 不应默认拥有所有 MCP tool。
- 单个 MCP tool 看似安全，但组合起来可能出现数据读取 + 外发风险。

但它的问题也明显：

- 仍然偏运行时治理，而不是 run event 完整性。
- 需要策略配置、权限模型、agent/MCP 绑定关系。
- 新增 agent 或 MCP 后如何减少策略维护成本，需要单独设计。

因此它更适合另一个 issue，而不是当前 run event 完整性问题。

结论：

> MCP / 多用户权限是运行时治理问题；run event 完整性是事后审计可信问题。两者应该拆开。

## 4. 第三个方向：可插拔完整性 provider

之后讨论过引入可插拔 provider，例如：

- 本地 hash-chain provider
- 签名 provider
- KMS/HSM provider
- timestamp provider
- immuDB provider

这个方向理论上灵活，但被质疑为“把 DeerFlow 当成框架来搞”。

主要问题：

- 第一版只想解决 run event 完整性，却引入 provider、配置、密钥、外部服务等大量扩展点。
- KMS/HSM/时间戳服务的授权、密钥保护、调用审计都需要解释。
- 维护者会很难判断这个 PR 的具体收益和边界。
- 用户也会被迫理解一套安全插件架构。

因此这个方向被收缩。

结论：

> 可以在代码内部保留很薄的 signer 接口，但不应该在 issue 里主打公开插件体系。

## 5. 第四个方向：默认 hash-chain

接着讨论过最轻量的 hash-chain：

```text
event_hash = sha256(canonical_event)
chain_hash = sha256(thread_id + seq + event_hash + prev_chain_hash)
```

这个方案优点是简单：

- 不需要外部服务。
- 不需要新表。
- 可以发现 event 内容被改、删除、重排、链断裂。

但它被质疑为“不引入签名的话，就是假的完整性保护”。

这个质疑成立。纯 hash-chain 只能防：

- 误操作导致的数据损坏。
- 只改 event 内容但没有同步重算链的篡改。
- 部分篡改、删除、重排。

但防不了：

- 有数据库写权限且知道算法的人重算整条链。

因此，纯 hash-chain 不足以支撑“生产向完整性保护”的说法。

结论：

> hash-chain 是必要的，但不能单独作为生产完整性保护的全部。需要签名。

## 6. 第五个方向：HMAC 签名

随后讨论过最小签名实现使用 HMAC-SHA256：

```text
signature = HMAC(secret, signing_payload)
```

它的优点：

- Python 标准库即可实现。
- 性能很轻。
- 对“数据库被改但 secret 没泄露”的场景有效。

但它被进一步质疑：

> 如果日志导出给第三方审计，第三方拿什么验证？

这个质疑也成立。HMAC 是共享密钥模型，验证者也需要同一个 secret。把 secret 给第三方审计方是不合理的，因为第三方拿到 secret 后也能伪造签名。

因此 HMAC 只适合内部自检，不适合“导出后第三方独立验证”。

结论：

> 如果目标包含审计导出和第三方验证，应该使用非对称签名，而不是 HMAC。

## 7. 第六个方向：非对称签名

方案进一步收敛为：

> 每条 persisted run event 写入时，计算 hash-chain，并用非对称私钥对完整性 payload 签名。

建议算法：

- Ed25519 优先。
- 私钥由 DeerFlow 部署方保存。
- 公钥可以公开给审计方。
- 导出的 run event 可由第三方仅凭公钥验证。

签名 payload 不应只签裸 `chain_hash`，而应签带上下文的结构，例如：

```json
{
  "purpose": "deerflow.run_event.integrity",
  "version": 1,
  "thread_id": "...",
  "run_id": "...",
  "seq": 42,
  "event_hash": "sha256:...",
  "prev_chain_hash": "sha256:...",
  "chain_hash": "sha256:...",
  "key_id": "deerflow-run-event-2026-01"
}
```

这样签名不能被轻易挪用到其他 thread、run 或 event 上。

结论：

> 非对称签名能解决 HMAC 的第三方验证问题，是当前更合适的最小生产方案。

## 8. 第七个方向：是否只支持 DB backend

曾经一度收敛到“只支持生产向 DB backend”，理由是：

- `db` 是生产持久化路径。
- SQLite/PostgreSQL 都支持同 row 写入完整性字段。
- PostgreSQL 更适合多节点和生产环境。

但进一步讨论后发现，如果完整性字段直接写在 event 自身上，而不是写额外 receipt 表，那么 JSONL 也可以自然支持：

- JSONL 每一行就是一个完整 event。
- 开启完整性后，在同一行追加 integrity 字段即可。
- 不存在新表或双写问题。

需要注意的是，JSONL 的并发限制不会因为完整性保护而改变。源码中 `JsonlRunEventStore` 已说明它适合 lightweight single-node，多个进程共享同目录可能产生重复或非单调 seq。

因此最终支持范围可以是：

- 支持 `db`
- 支持 `jsonl`
- 不支持 `memory`

结论：

> 完整性字段内嵌到 event 后，DB 和 JSONL 都可以支持；memory 因为非持久化，不适合支持。

## 9. 第八个方向：新表、消息队列、immuDB

讨论中还考虑过几个替代方向。

### 9.1 新增 receipt 表

新增 `run_event_integrity` 表可以保持 `run_events` 主表干净，并在同一 DB transaction 内写入 receipt。

但问题是：

- 仍然是两份数据，虽然 receipt 不存 content/metadata。
- 查询和验证要 join 或额外查询。
- 当前更简单的做法是直接在 event row 上加字段。

因此否掉。

### 9.2 消息队列

曾考虑过：

```text
RunJournal -> RunEventStore -> MQ -> 外部审计消费端
```

优点：

- 解耦外部审计。
- 可以让消费端做签名、归档、SIEM、告警。

但问题是：

- 应用层双写，存在 DB 成功但 MQ publish 失败的问题。
- MQ 不满足 `RunEventStore` 的查询接口。
- 对生产 PostgreSQL 用户来说，数据库层 CDC / logical replication 更自然。

因此 MQ 不适合作为当前 DeerFlow 侧改动。

### 9.3 immuDB

也讨论过把 run event 对接到 immuDB。

问题是：

- immuDB 通常是独立服务，不是 SQLite 那种嵌入式本地文件库。
- 需要部署、账号、密码、网络、备份、权限。
- 作为 DeerFlow 默认或第一版方案太重。

因此否掉。

结论：

> 新表、MQ、immuDB 都能解决部分问题，但都扩大了改动面。当前方案应直接在 persisted event 上增加签名完整性字段。

## 10. 当前最终收敛方案

最终收敛为：

> 为持久化 run event 增加可选的、每条 event 内嵌的 signed integrity metadata。默认关闭。开启后只影响新写入事件。支持 `db` 和 `jsonl` 后端，不支持 `memory`。

### 10.1 配置

示例：

```yaml
run_events:
  backend: db # or jsonl
  integrity:
    enabled: true
    signature_algorithm: ed25519
    private_key_env: DEERFLOW_RUN_EVENT_SIGNING_PRIVATE_KEY
    key_id: deerflow-run-event-2026-01
```

默认：

```yaml
run_events:
  integrity:
    enabled: false
```

如果 `backend=memory` 且 `integrity.enabled=true`，应启动失败，而不是静默降级。

如果开启 integrity 但 signing key 缺失，也应启动失败。

### 10.2 数据字段

开启后，每条新 event 增加字段：

```text
integrity_version
hash_algorithm
event_hash
prev_chain_hash
chain_hash
signature_algorithm
signature_key_id
signature
```

对 DB：

- 字段需要加到 `run_events` 表。
- 字段 nullable，用于兼容历史数据和默认关闭。

对 JSONL：

- 不开启时不写这些字段。
- 开启后在 JSONL event 行中追加这些字段。
- 不建议不开启时写 `null` 字段，避免膨胀文件和混淆验证语义。

### 10.3 写入流程

对每条 event：

```text
1. 分配 seq。
2. 构造最终 persisted event。
3. canonicalize event，排除 integrity 字段。
4. event_hash = sha256(canonical_event)。
5. prev_chain_hash = 同 thread 上一条受保护 event 的 chain_hash。
6. chain_hash = sha256(thread_id + seq + event_hash + prev_chain_hash)。
7. signature = ed25519_sign(signing_payload)。
8. 写入 event，包含 integrity 字段。
```

对 `put_batch`：

```text
prev = latest_chain_hash(thread_id)
for event in batch sorted by assigned seq:
    compute event_hash
    compute chain_hash
    sign payload
    prev = chain_hash
persist batch
```

### 10.4 验证流程

第三方或运维方拿到：

- run event 导出数据
- public key

即可验证：

```text
1. 按 thread_id + seq 排序。
2. 重新 canonicalize event。
3. 重算 event_hash。
4. 重算 chain_hash。
5. 检查 prev_chain_hash 是否连接。
6. 用 public key 验证 signature。
```

验证结果应区分：

- `verified`
- `unprotected`
- `failed`

历史旧数据或未开启 integrity 时写入的数据应标记为 `unprotected`，不应直接视为验证失败。

## 11. 签名失败和写入失败语义

当前 `RunJournal` 的持久化语义是 best-effort with retry：

- 事件先进入内存 buffer。
- flush 时调用 `RunEventStore.put_batch()`。
- 如果 `put_batch()` 抛异常，`RunJournal` 会把 batch 放回 buffer，等待后续重试。
- run 结束时 `worker.py` 会最终调用 `journal.flush()`。
- 如果最终 flush 仍失败，worker 只记录 warning，不会让 agent run 失败。

因此完整性功能第一版应沿用这个语义：

- 开启 integrity 后，不允许写入 unsigned event。
- 签名失败应视为 store 写入失败。
- 当前 batch 不应部分落盘。
- `put` / `put_batch` 抛异常，交给现有 `RunJournal` retry。
- 如果最终仍失败，run event 可能缺失，但 agent run 不会因此失败。

这意味着第一版保证的是：

> 只要 event 被成功持久化，它就是可验证的。

它不保证：

> 每个 run event 一定会成功持久化。

这个边界需要在 issue 中明确写出。

## 12. 安全边界

该方案能防：

- 持久化后的 event 内容被改。
- event 被删除或重排。
- 只拥有数据库/文件写权限、但没有私钥的人伪造有效历史。
- 导出日志后，第三方仅凭公钥独立验证 run history。

该方案防不了：

- 私钥泄露。
- DeerFlow 进程被完全控制后，攻击者调用私钥签出伪造事件。
- 写入前 event 内容已经是错误或恶意的。
- `RunJournal` best-effort 语义导致的日志缺失。
- JSONL 多进程并发写入问题。

因此应避免宣称它是 tamper-proof audit ledger。更准确的定位是：

> Optional signed tamper-evident metadata for persisted run events.

## 13. 与长期记忆完整性保护的区别

Run event 完整性保护和长期记忆完整性保护不是同一个问题。

| 维度 | Run event 完整性 | 长期记忆完整性 |
|---|---|---|
| 保护对象 | 已发生 run 的历史记录 | Agent 后续会读取和使用的长期状态 |
| 主要目标 | 事后审计、复盘、证明历史未被静默篡改 | 防止记忆污染影响未来行为 |
| 校验时机 | 写入时签名，事后验证 | 写入时授权，读取时验证 |
| 失败动作 | 标记历史不可信或缺失 | 拒绝注入上下文、隔离或要求确认 |
| 是否阻断当前 run | 第一版不阻断 | 可能需要阻断 |

长期记忆完整性更接近零信任模型：

- memory 写入要经过授权路径。
- memory entry 要有来源、版本、receipt。
- memory 被 Agent 使用前要验证。
- 验证失败就不应注入上下文。

Run event 完整性则是：

- 不阻止运行时行为。
- 不治理 tool/MCP/memory。
- 只保护已持久化 run history 的证据价值。

## 14. 最终建议的 issue 标题

建议标题：

```text
[RFC] Add optional signed integrity metadata for persisted run events
```

核心表述：

> DeerFlow already persists run history through `RunEventStore`, but persisted run events currently have no built-in way to prove that stored or exported history has not been silently modified. This proposal adds optional signed hash-chain metadata to persisted run events for `db` and `jsonl` backends. The feature is disabled by default and only protects newly written events after it is enabled.

这个版本相比之前的方案更稳定：

- 不做治理平台。
- 不做 MCP 权限模型。
- 不做新表。
- 不做 MQ。
- 不引入 immuDB。
- 不做 KMS/HSM 框架。
- 不改变 `RunJournal` 当前 best-effort 语义。
- 只在持久化 event 上增加可选签名完整性字段。

