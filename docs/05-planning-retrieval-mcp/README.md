# 05 - Planning、Retrieval 与 MCP

> 🎯 **本章目标**：把“Agent 是怎么决定搜什么、去哪里搜、MCP 为什么也能当 Retriever”讲清楚。读完后，你应该能解释 Query Decomposition、Retriever Factory、`requires_scraping` 和 MCP fast/deep 的设计价值。

---

## 目录

- [5.1 为什么先 Planning 再 Search 不够](#51-为什么先-planning-再-search-不够)
- [5.2 Grounded Planning：为什么先搜一轮](#52-grounded-planning为什么先搜一轮)
- [5.3 generate_sub_queries：模型如何拆问题](#53-generate_sub_queries模型如何拆问题)
- [5.4 LLM 输出为什么必须 Normalize](#54-llm-输出为什么必须-normalize)
- [5.5 Retriever 到底是什么](#55-retriever-到底是什么)
- [5.6 为什么 Search Result 不能直接当正文](#56-为什么-search-result-不能直接当正文)
- [5.7 requires_scraping：一个很小但很重要的契约](#57-requires_scraping一个很小但很重要的契约)
- [5.8 MCP 如何接入 Research Workflow](#58-mcp-如何接入-research-workflow)
- [5.9 fast / deep / disabled](#59-fast--deep--disabled)
- [5.10 面试高频题](#510-面试高频题)
- [5.11 本章总结](#511-本章总结)

---

## 5.1 为什么先 Planning 再 Search 不够

很多人第一次设计 Research Agent，会写成：

```text
用户问题
  ↓
LLM 拆成 5 个 Query
  ↓
Search
```

这看起来合理，但有个问题：

> Planner 自己可能不知道“当前世界”发生了什么。

例如用户问的是一个最近刚更新的框架。

如果 Planner 的参数知识比较旧，它可能：

- 用旧项目名；
- 拆出过时问题；
- 漏掉刚新增的关键能力；
- 错过正确搜索词。

所以 GPT Researcher 采用：

```text
Query
  ↓
Initial Search
  ↓
Search Results
  ↓
Strategic LLM
  ↓
Sub Queries
```

这叫 **Grounded Planning**。

---

## 5.2 Grounded Planning：为什么先搜一轮

### 源码位置

`gpt_researcher/skills/researcher.py`

核心顺序：

```python
initial_results = await self._get_initial_search_results(
    query,
    query_domains,
)

sub_queries = await self.plan_research(
    query,
    query_domains,
    search_results=initial_results,
)
```

### 示例

原问题：

```text
比较 LangGraph、AutoGen、CrewAI...
```

Initial Search 可能先暴露：

```text
LangGraph Platform
AutoGen AgentChat / Core
CrewAI Crews / Flows
observability docs
```

于是 Planner 得到更准确的词汇，再拆 Query。

### 设计价值

这和人类研究很像：

> 你不会在完全没看资料前，就非常自信地确定研究提纲。

你通常先扫一眼：

- 目录；
- 最新文档；
- 搜索结果；

然后再决定深挖什么。

---

## 5.3 generate_sub_queries：模型如何拆问题

### 文件定位

`gpt_researcher/actions/query_processing.py`

核心函数：

```python
async def generate_sub_queries(
    query,
    parent_query,
    report_type,
    context,
    cfg,
    ...
):
```

Prompt 会包含：

- 原始 Query；
- Parent Query；
- Report Type；
- Initial Search Context；
- `MAX_ITERATIONS`。

然后调用：

```text
Strategic LLM
```

### 为什么叫 Strategic？

因为这一节点做的是：

> “接下来研究什么？”

不是：

> “最终文章怎么写？”

这属于规划问题。

---

### 一个可能的输出

```json
[
  "LangGraph orchestration and state management architecture",
  "AutoGen agent runtime and multi-agent coordination",
  "CrewAI crews and flows state management",
  "observability and tracing comparison"
]
```

现在大问题变成了多个独立 Worker 可执行的任务。

---

## 5.4 LLM 输出为什么必须 Normalize

### 理想世界

Prompt 要求：

```json
["a", "b", "c"]
```

### 真实世界

模型可能返回：

```json
{"queries": ["a", "b", "c"]}
```

或者：

```json
{"query": "a"}
```

或者：

```text
a
```

所以源码有：

```python
_normalize_sub_queries(...)
```

它会统一变成：

```python
list[str]
```

最后如果一条都没有：

```text
fallback → original query
```

### 工程原则

> **LLM 的输出格式必须被当成不可信输入。**

和处理第三方 API 一样：

- parse；
- validate；
- normalize；
- fallback。

---

### Planning 还有模型级 Fallback

当前逻辑大致是：

```text
Strategic LLM
  ↓ fail
Strategic LLM + explicit token limit
  ↓ fail
Smart LLM
```

这里有两个不同层次：

#### Transport Retry

网络失败、空响应。

#### Semantic Fallback

换一种模型角色继续完成业务。

---

## 5.5 Retriever 到底是什么

Retriever 不是：

> “一个网页爬虫。”

它解决的问题是：

> **针对这个 Query，哪些来源可能值得看？**

例如：

```text
TavilySearch
GoogleSearch
Exa
Semantic Scholar
PubMed
OpenAlex
MCPRetriever
```

它们都尽量输出相似的数据：

```json
{
  "title": "...",
  "href": "...",
  "body": "...",
  "raw_content": "..."
}
```

不同 Provider 字段还不完全统一，所以源码里经常看到：

```python
url = result.get("href") or result.get("url")
```

这也是后续可以重构的地方。

---

## 5.6 为什么 Search Result 不能直接当正文

假设 Search Provider 返回：

```json
{
  "title": "LangGraph",
  "href": "https://...",
  "body": "LangGraph is a framework for..."
}
```

这里的 `body` 可能只是 Search Snippet。

如果 Writer 直接拿这个写：

- 信息太浅；
- 细节缺失；
- 引用不完整；
- 容易把搜索摘要当权威原文。

因此：

```text
Retriever
→ Candidate Source
→ Scraper
→ Real Page Content
```

---

## 5.7 requires_scraping：一个很小但很重要的契约

### 源码位置

`ResearchConductor._search_relevant_source_urls`

关键逻辑：

```python
requires_scraping = getattr(
    retriever,
    "requires_scraping",
    None,
)
```

### 情况一：`True`

意思是：

> 我给你的只是预览，必须继续读网页。

即使 `raw_content` 很长，也不要误判成全文。

### 情况二：`False`

意思是：

> 我已经帮你拿到全文。

例如某些学术 API。

这时可以：

```text
直接加入 prefetched_content
→ 不再重复访问网页
```

### 情况三：`None`

为了兼容老插件，源码保留旧 heuristic。

### 为什么这个设计值得学习

因为它解决的是：

> **数据的语义，不只是数据的类型。**

两个字段都叫 `raw_content`：

- 一个可能是 snippet；
- 一个可能是真全文。

“字符串很长”并不能说明它是什么。

---

## 5.8 MCP 如何接入 Research Workflow

在 GPT Researcher 的主流程里，可以先把 MCP 理解成：

> **另一个外部信息源。**

所以它被包装成：

```text
MCPRetriever
```

于是主流程仍然是：

```text
Sub Query
  ↓
Retriever
  ↓
Context
```

只是 Retriever 内部可能：

```text
选择 MCP Tool
→ 调用 Tool
→ 返回数据
```

### 设计好处

ResearchConductor 不需要知道：

- 这个信息来自搜索引擎；
- 还是来自数据库 MCP；
- 还是文件系统 MCP。

它只需要拿到研究上下文。

---

## 5.9 fast / deep / disabled

MCP 往往比普通 Search 更贵，可能包含：

- LLM Tool Selection；
- Remote Tool Call；
- MCP Server I/O。

所以项目引入三种策略。

### disabled

```text
不调用 MCP
```

适合：

- 当前只需要 Web；
- MCP 不稳定；
- 成本敏感。

### fast

默认思路：

```text
Original Query
   ↓
MCP 调一次
   ↓
Cache
   ↓
所有 Sub Query 复用
```

优点：

- 少调用；
- 延迟低；
- 成本低。

缺点：

- 不够针对每个 Sub Query。

### deep

```text
q1 → MCP
q2 → MCP
q3 → MCP
```

优点：

- 每个研究方向更针对；
- 覆盖更全面。

缺点：

- 成本更高；
- 延迟更高。

---

### 为什么 MCP Cache 要 Lock

Hybrid 模式可能同时开始：

```text
Local pass
Web pass
```

两边都看到：

```text
cache == None
```

如果不加锁，就可能重复执行同一次昂贵 MCP 请求。

当前源码：

```python
async with self._mcp_cache_lock:
    if mcp_retrievers and self._mcp_results_cache is None:
        ...
```

这是普通异步程序中的共享状态并发问题。

---

### 为什么还要防 Tavily 双路径

如果同时：

```text
Direct Tavily Retriever
+
Tavily MCP
```

两个路径最终可能打的是同一个数据源。

结果：

- 数据重复；
- 多一次 MCP Tool Selection；
- 成本更高。

所以代码会识别这种冗余路径。

---

## 5.10 面试高频题

### Q：为什么 Planning 前要 Initial Search？

> “为了 grounded planning。先用外部结果给 Planner 提供当前世界信息，降低模型因参数知识过时导致 Query Decomposition 偏离的问题。”

### Q：Retriever 和 Scraper 有什么区别？

> “Retriever 负责 candidate discovery，输出候选来源；Scraper 负责 content acquisition，真正读取页面。搜索摘要不等于正文，所以两层必须分开。”

### Q：`requires_scraping` 为什么重要？

> “它是内容语义契约。Provider 可以明确声明自己的 raw_content 是全文还是 preview，比用字符串长度猜测更可靠。”

### Q：MCP fast 和 deep 怎么选？

> “fast 是主 Query 执行一次 MCP 后缓存复用，优化成本和延迟；deep 对每个 Sub Query 单独调用，优化覆盖率。它本质上是 quality-cost trade-off。”

---

## 5.11 本章总结

```text
┌────────────────────────────────────────────────┐
│                  Planning / Retrieval           │
│                                                │
│ Query                                          │
│   ↓                                            │
│ Initial Search                                 │
│   ↓                                            │
│ Strategic LLM → Sub Queries                    │
│   ↓                                            │
│ Retriever                                      │
│   ├─ URL → Scrape                              │
│   └─ Full Content → Direct                     │
│                                                │
│ MCP 是一种特殊 Retriever                        │
│   ├─ fast → cache                              │
│   ├─ deep → per query                          │
│   └─ disabled                                  │
└────────────────────────────────────────────────┘
```

下一章继续追：资料拿回来以后，怎么从几十万字符压缩成真正可用的 Context。

➡️ [06 - Scraping、Context 与报告生成](../06-context-and-writing/README.md)
