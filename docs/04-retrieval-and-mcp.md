# 04 - Retrieval 与 MCP：Agent 怎样从外部世界找信息

> **本章目标**：理解 Retriever 层的抽象、多个 Search Provider 如何统一、MCP 怎样接入，以及为什么当前版本专门设计了 `fast / deep / disabled` 三种 MCP 策略。

## 4.1 Retrieval 不是 Scraping

先区分两个动作：

```text
Retrieval / Search
→ 找到“哪些来源可能相关”
→ 输出 title / url / snippet / raw_content

Scraping / Browsing
→ 真正读取 URL
→ 输出 page content
```

很多 Research Agent 把两者混在一起，导致：

- 搜索摘要被误当正文；
- 已经返回全文的学术 API 被重复抓取；
- Citation URL 丢失；
- 对每个 Provider 写大量 if/else。

GPT Researcher 当前实现显式区分这两层。

## 4.2 Retriever Factory

入口：

```text
GPTResearcher.__init__
→ get_retrievers(headers, cfg)
→ get_retriever(name)
```

内置支持包括：

- Tavily
- Google
- Bing
- Brave
- DuckDuckGo
- SearX
- SerpAPI / Serper / SearchAPI
- Exa
- arXiv
- Semantic Scholar
- PubMed Central
- OpenAlex
- MCP
- Custom
- 社交搜索等

默认 Retriever 当前是 `tavily`。

## 4.3 Retriever 配置优先级

`get_retrievers` 大致按：

```text
headers["retrievers"]
  ↓
headers["retriever"]
  ↓
cfg.retrievers
  ↓
cfg.retriever
  ↓
default Tavily
```

因此 API 请求可以覆盖全局配置。

## 4.4 多 Retriever

配置可以是：

```text
tavily,exa,openalex
```

普通研究中 `_search_relevant_source_urls` 会迭代当前 Retriever。

这可以提高：

- recall；
- source diversity；
- 学术 / Web 混合覆盖。

代价：

- 搜索成本线性增加；
- 重复来源增加；
- rate limit 更复杂。

因此 visited URL 去重和 content contract 很重要。

## 4.5 第三方 Retriever 插件

未知名称不是直接报错，而会尝试 Python entry point：

```text
group = "gpt_researcher.retrievers"
```

这意味着第三方包可以注册 Retriever，而不用修改核心仓库。

架构意义：

> Retriever 是一个稳定 extension point，而不是一个不断膨胀的 switch statement。

虽然当前内置 Provider 仍通过 match 显式注册，但 entry point 给外部扩展留了空间。

## 4.6 Search 为什么放到 asyncio.to_thread

Retriever 的 `search()` 往往是同步阻塞 HTTP 调用。

Action 中：

```python
return await asyncio.to_thread(
    search_retriever.search,
    **search_kwargs,
)
```

否则同步 requests 会阻塞整个 event loop，导致其它 Sub-query 无法真正并发。

这是 Python async 项目里非常典型的兼容策略：

```text
async orchestration
+ legacy sync SDK
→ asyncio.to_thread
```

## 4.7 Search Result 数据契约

常见字段：

```text
title
href / url
body / content
raw_content
```

不同 Retriever 不完全一致，因此代码里经常出现：

```python
url = item.get("url") or item.get("href")
body = item.get("body") or item.get("content")
```

这暴露出当前系统的一点技术债：

> Retriever 输出没有完全归一化成严格统一的 Pydantic model。

如果你做二次开发，值得先引入 `SearchResult` schema。

## 4.8 requires_scraping：很关键的内容契约

在 `_search_relevant_source_urls` 中，系统会判断 Retriever 是否声明：

```python
requires_scraping = getattr(retriever, "requires_scraping", None)
```

三种情况：

### `True`

Retriever 返回的只是搜索摘要，不管 snippet 多长，都继续抓 URL。

### `False`

Retriever 自己已经拿到全文。

如果 `raw_content` 存在：

- 直接作为 prefetched content；
- 加入 research_sources；
- 把 URL 加到 visited_urls；
- 不再抓网页。

### `None`

为了兼容旧 Retriever，继续使用历史 heuristic：

> raw_content 长度大于 100 时，认为可能已经是全文。

这是一段非常有“真实系统演进痕迹”的代码：先有 heuristic，后来新增明确能力声明，但保留向后兼容。

## 4.9 为什么这比统一“全部抓一遍”更好

因为某些来源：

- API 已经返回结构化全文；
- URL 可能禁止 crawler；
- 二次抓取会丢掉 API 提供的干净文本；
- 二次抓取浪费延迟和带宽。

因此 Retriever 不只是“Search API Wrapper”，它还需要声明自己的 content semantics。

## 4.10 MCP 在系统里的位置

MCP 被做成一种 Retriever：

```text
MCP Server
→ MCPRetriever
→ search-like results
→ merge into research context
```

这让 MCP 获得了一个很自然的接入点：从核心 Research Workflow 看，它只是另一个信息源。

## 4.11 MCP 配置的一个重要并发修复

当前 `GPTResearcher._process_mcp_configs` 明确避免通过 `os.environ` 注入 MCP Retriever，而是修改当前实例的 `cfg.retrievers`。

原因：

```text
process-wide environment
+ concurrent requests
= session A can pollute session B
```

因此 MCP 配置尽量被限制到 Researcher session。

这是一个值得在面试里讲的真实 concurrency bug 类别：**共享进程级配置污染**。

## 4.12 MCP Strategy

当前有三种：

### disabled

```text
never call MCP
```

适合：

- MCP 不稳定；
- 只想用 Web；
- 控成本。

### fast（默认）

```text
main query
→ MCP once
→ cache
→ every sub-query reuses same MCP context
```

优点：

- MCP 调用数量低；
- Tool selection / LLM selection 成本低；
- 延迟低。

缺点：

- 各 Sub-query 的 MCP context 不够针对。

### deep

```text
q1 → MCP
q2 → MCP
q3 → MCP
...
```

更全面，但成本更高。

## 4.13 为什么 MCP Cache 需要 Lock

Hybrid 模式可能同时启动两个独立 research pass。

如果都发现：

```text
_mcp_results_cache is None
```

且同时开始填 cache，就会重复执行同一个 MCP research。

因此：

```python
async with self._mcp_cache_lock:
    if cache is None:
        ...
```

这是标准 double-check / critical section 思路。

它说明 Agent 中不仅有“模型并发”，还有普通异步程序的共享状态并发问题。

## 4.14 Tavily MCP 重复检测

当前代码还专门处理：

```text
direct Tavily Retriever
+
Tavily MCP
```

如果两者同时启用，可能对同一个后端 API 重复查询，只是 MCP 路径还多一层工具选择成本。

`_tavily_mcp_redundant_with_direct` 会识别配置并跳过 MCP Tavily。

这是很典型的 **semantic deduplication**：不仅对 URL 去重，也对“数据源能力”去重。

## 4.15 MCP 两阶段思想

`_execute_mcp_research` 的日志中有：

```text
Stage 1: Selecting optimal MCP tools
```

MCP Retriever 内部可以自己完成：

1. 根据 Query 选择合适 Tool；
2. 调用工具获取结果。

因此在外部 ResearchConductor 看来是一次 Retriever search，但内部可能已经是一个 Agentic tool-selection workflow。

这也是为什么 MCP-only 时可以跳过额外 Sub-query Planning。

## 4.16 Web + MCP Context 怎样合并

顺序大致是：

```text
web context
+
formatted MCP context with source
→ final_context
```

每个 MCP item 会保留：

- content
- title
- url（如果有）
- source marker

这样 Writer 至少还能看到来源边界。

## 4.17 Retrieval 失败策略

多个 Retriever 搜索时，单个 Provider 异常通常被捕获并继续。

`quick_search(all_retrievers=True)` 更明确使用：

```python
asyncio.gather(..., return_exceptions=True)
```

然后：

- Exception 跳过；
- 空结果跳过；
- URL 去重；
- 合并。

这是“部分可用”的设计，而不是 all-or-nothing。

## 4.18 生产化可以怎么加强

### 统一 SearchResult Schema

```python
class SearchResult(BaseModel):
    url: str
    title: str | None
    snippet: str | None
    raw_content: str | None
    source_provider: str
    requires_scraping: bool
    score: float | None
```

### Provider-level Circuit Breaker

连续失败后短时间禁用 Provider，而不是每个 Sub-query 都继续撞失败 API。

### Per-provider Timeout / Budget

不同 Provider 配：

```text
timeout
max_results
max_cost
priority
```

### Retrieval fusion

当前更多是 merge + URL dedup。可以加入 RRF / weighted fusion。

## 4.19 面试题

**Q：MCP 为什么实现成 Retriever 而不是通用 Tool Loop？**

A：GPT Researcher 的主任务是获取研究证据，MCP 在这里主要承担外部信息源角色。把它适配成 Retriever 可以直接复用已有 planning、context 和 report pipeline。

**Q：fast 与 deep 怎么选？**

A：fast 用主 Query 调一次 MCP 后复用，适合高延迟或工具选择昂贵的 MCP；deep 对每个 Sub-query 调用，覆盖更好但成本和延迟更高。

**Q：Retriever 返回了 raw_content，为什么还不能直接用？**

A：要看 Retriever 的语义契约。snippet 可能很长但仍不是全文，所以当前版本引入 `requires_scraping` 显式声明，并保留旧 heuristic 兼容。
