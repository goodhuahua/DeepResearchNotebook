# 12 - 并发、容错、成本：一个 Agent 怎样从 Demo 变成工程系统

> **本章目标**：把散落在源码里的工程策略统一起来。Agent 开发岗位真正区分“会调 API”和“能做系统”的，通常就是这一章。

## 12.1 先看失败面

一次 Research 依赖：

```text
LLM
Search Provider
MCP Server
Website
Scraper
Embedding / Context Service
Vector Store
WebSocket
Parser
```

任意一个都可能失败。

因此不能设计成：

```text
any error → entire request crash
```

而应该是：

```text
local failure
→ contain
→ fallback / skip
→ preserve useful partial work
```

## 12.2 LLM Transport Retry

统一 `create_chat_completion`：

非流式请求最多尝试多次，使用指数退避：

```text
1s → 2s → 4s → 8s → capped
```

并同时处理：

- Exception；
- empty response。

这属于 transient error recovery。

## 12.3 Streaming 不自动多次 retry

当 websocket 正在 stream：

```text
max_attempts = 1
```

因为输出已经产生外部副作用。

这是幂等性意识。

## 12.4 Semantic Fallback

仅 retry 同一个模型不够。

`generate_sub_queries`：

```text
strategic call
→ retry strategic with explicit token limit
→ fallback smart LLM
```

这是 semantic availability：

> 只要还有一个足够强的模型，就继续完成业务。

## 12.5 Structured Output Recovery

多个模块使用：

- `json_repair`；
- dict/list normalization；
- regex fallback；
- text-line parser；
- original query fallback；
- default agent fallback。

因为 LLM 最常见的失败不一定是 HTTP 500，而是：

> 200 OK，但输出形状不符合预期。

## 12.6 Malformed External Data Guards

源码和 tests 中大量防：

- result 不是 dict；
- URL 为 null；
- content 缺失；
- scraper partial payload；
- source curator JSON 异常；
- Tavily / SerpAPI 非预期结构；
- vector-store doc 异常。

这是现实 Agent 的常态：**外部工具输出也是不可信数据**。

## 12.7 Retrieval Partial Failure

多个 Retriever 时，一个失败并不阻止其它来源。

Deep Research 某 branch 失败返回 `None`，gather 后过滤。

如果整层都失败，停止递归。

这种层级 failure containment 很重要。

## 12.8 Context Fallback

`select_context`：

```text
Jev fails
→ keyword

Embeddings fails
→ keyword
```

因此高级 Context Service 不是 single point of failure。

## 12.9 Empty Evidence Failure

最坏情况：

```text
all retrieval failed
→ context empty
```

Writer 不编造，直接 abstain。

这叫 fail-safe，而不是 fail-open。

## 12.10 并发层级

至少有：

### Sub-query fan-out

`asyncio.gather`。

### Hybrid source fan-out

Local pass + Web pass concurrently。

### Scraper workers

`WorkerPool`。

### Deep Research branch concurrency

`Semaphore`。

### previous written content lookup

多个 title/query 可 gather。

所以 concurrency 并不是一个参数能控制全部。

## 12.11 Concurrency Budget

生产中应该有统一预算：

```text
global request concurrency
tenant concurrency
research concurrency
search-provider concurrency
scraper concurrency
LLM concurrency
```

否则：

```text
4 deep branches
× 4 ordinary subqueries
× 5 URLs
= potentially large burst
```

## 12.12 MCP Cache Lock

Hybrid 并发时两个 pass 可能同时填 MCP cache。

当前使用：

```python
asyncio.Lock()
```

避免重复昂贵计算。

这是共享 mutable state 的明确 critical section。

## 12.13 Process-global State Pollution

当前 MCP 配置避免修改全局 `os.environ["RETRIEVER"]`。

因为两个请求：

```text
Request A: enable MCP
Request B: web only
```

如果 A 临时改全局 env，B 可能也看到 MCP。

这种 bug 比模型 hallucination 更“工程”，但一样会导致错误行为。

## 12.14 visited_urls

这是 dedup state。

作用：

- 少抓重复 URL；
- 少花 token；
- source coverage 更广。

但在分布式环境里要升级为共享 store。

## 12.15 MCP Fast Cache

`fast`：

```text
one MCP call
→ reuse N subqueries
```

本质是成本/延迟优化。

`deep`：

```text
N MCP calls
```

是质量/覆盖优化。

这是一个非常直接的质量-成本 knob。

## 12.16 Small Context Fast Path

内容很少时跳过压缩/embedding。

减少：

- API；
- latency；
- embedding cost。

不要“为了架构完整”每次都走最复杂流程。

## 12.17 Cost Tracking

LLM utility 在成功后计算 cost，通过 callback 交给 Researcher。

Researcher：

```text
total research_costs
+
step_costs
```

因此可以定位昂贵 stage。

不过如果 child researcher 各自持有 cost tracker，父级是否完整聚合需要特别验证。这正是为什么生产系统最好共享统一 budget object。

## 12.18 预算应该在执行前检查

当前更多是“调用后统计”。

更强系统应该：

```text
estimated remaining cost
+
spent cost
>
request budget
→ stop / downgrade
```

例如：

- 深层 branch 降 breadth；
- Smart → Fast；
- MCP deep → fast；
- 少抓 URL；
- 提前停止。

## 12.19 Token Budget

需要分别管理：

- Planner input/output；
- Context budget；
- Writer output；
- Deep accumulated context。

Deep Research 已经有 25k words cap，但普通链路仍可以进一步做显式 token packing。

## 12.20 Timeout

外部调用应有：

```text
connect timeout
read timeout
total operation timeout
branch timeout
request deadline
```

并把剩余 deadline 向子任务传播。

否则某一个慢网站可以拖住整次 research。

## 12.21 Cancellation Propagation

理想：

```text
client disconnects
→ cancel root task
→ cancel gather children
→ cancel scraper workers
→ close sessions
→ do not continue paid LLM calls
```

这对成本治理很重要。

## 12.22 Retry 不是越多越好

需要分类：

### Retryable

- timeout；
- 429；
- transient 5xx；
- empty provider response。

### Non-retryable

- invalid API key；
- malformed config；
- permission denied；
- unsupported model parameter。

错误分类可以避免“错误 key 重试 10 次”。

## 12.23 Circuit Breaker

如果 Tavily 连续失败：

```text
open circuit
→ temporarily skip Tavily
→ use Exa / other provider
```

比每个 Sub-query 都等待失败更好。

## 12.24 Evaluation

可靠性不只是“不报错”。

Research Agent 至少要评：

- Source recall；
- Source precision；
- factuality；
- citation correctness；
- freshness；
- report coverage；
- latency；
- cost。

上游有 `evals/context_filter`、quality eval 等目录，说明项目也在向可测量优化演进。

## 12.25 一个生产级 Budget 对象

可以设计：

```python
class ResearchBudget:
    deadline: datetime
    max_cost_usd: float
    max_llm_calls: int
    max_search_calls: int
    max_urls: int
    max_context_tokens: int
    max_depth: int

    def reserve(...): ...
    def spent(...): ...
    def remaining(...): ...
```

父子 Researcher 共享同一个。

这样所有自治都在明确资源边界内。

## 12.26 面试题

**Q：Agent 的可靠性和普通后端有什么不同？**

A：它不仅有网络/数据库错误，还有 LLM 输出 shape 漂移、tool result malformed、context overflow、递归工作量爆炸、source hallucination 等概率性失败，所以需要 parser recovery、budget、fallback 和 evidence guard。

**Q：系统有哪些并发？**

A：普通 Sub-query gather、Hybrid source gather、Scraper worker pool、Deep Research semaphore，以及部分 Context lookup 并发。需要把这些乘起来看总体外部请求压力。

**Q：你认为当前最需要加强的成本治理是什么？**

A：共享的 request-level budget，尤其让 nested researcher 共享 cost/token/deadline，而不仅是每个实例独立统计调用后成本。
