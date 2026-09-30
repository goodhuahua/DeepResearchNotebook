# 04 - 源码逐行走读

> 🎯 **本章目标**：从最小入口开始，沿着真实调用链一直追到最终 Report。看完以后，你应该能够打开 IDE，从 `BasicReport.run()` 一路跳转到 Planning、Search、Scrape、Context 和 Writer，而不会迷路。

> 📌 **阅读说明**：下面的代码片段会删去日志、少量参数和边缘分支，只保留理解主链路需要的核心逻辑。需要核对完整实现时，点击每节给出的源码文件。

---

## 目录

- [4.1 先看阅读顺序](#41-先看阅读顺序)
- [4.2 BasicReport：最干净的入口](#42-basicreport最干净的入口)
- [4.3 GPTResearcher.__init__：一次研究 Session 是怎么组装的](#43-gptresearcher__init__一次研究-session-是怎么组装的)
- [4.4 GPTResearcher.conduct_research：总入口](#44-gptresearcherconduct_research总入口)
- [4.5 ResearchConductor.conduct_research：真正的普通研究主流程](#45-researchconductorconduct_research真正的普通研究主流程)
- [4.6 _get_context_by_web_search：普通 Research 的心脏](#46-_get_context_by_web_search普通-research-的心脏)
- [4.7 plan_research：Query 是怎么被拆开的](#47-plan_researchquery-是怎么被拆开的)
- [4.8 _process_sub_query：一个研究分支内部发生什么](#48-_process_sub_query一个研究分支内部发生什么)
- [4.9 Search → URL → Scrape：资料是怎么读回来的](#49-search--url--scrape资料是怎么读回来的)
- [4.10 ContextManager：为什么搜到的内容不能直接给 LLM](#410-contextmanager为什么搜到的内容不能直接给-llm)
- [4.11 write_report：研究结果如何变成报告](#411-write_report研究结果如何变成报告)
- [4.12 完整调用链回放](#412-完整调用链回放)
- [4.13 本章面试题](#413-本章面试题)
- [4.14 本章总结](#414-本章总结)

---

## 4.1 先看阅读顺序

不要从 `agent.py` 第一行一路向下硬读。

推荐顺序：

```text
1. backend/report_type/basic_report/basic_report.py
2. gpt_researcher/agent.py
3. gpt_researcher/skills/researcher.py
4. gpt_researcher/actions/query_processing.py
5. gpt_researcher/skills/context_manager.py
6. gpt_researcher/context/select.py
7. gpt_researcher/skills/writer.py
8. gpt_researcher/actions/report_generation.py
```

为什么？

因为这是一次请求真实经过的顺序。

---

## 4.2 BasicReport：最干净的入口

### 文件定位

`backend/report_type/basic_report/basic_report.py`

这个 wrapper 的价值是：**没有复杂 UI，也没有 Multi-Agent，只展示最核心的两步。**

核心逻辑可以理解为：

```python
async def run(self):
    await self.gpt_researcher.conduct_research()
    report = await self.gpt_researcher.write_report()
    return report
```

### 逐行理解

第一行：

```python
await self.gpt_researcher.conduct_research()
```

意思是：

> “先去世界上收集足够的研究证据。”

这一步结束后，最重要的状态是：

```python
researcher.context
```

第二行：

```python
report = await self.gpt_researcher.write_report()
```

意思是：

> “现在证据准备好了，开始写报告。”

### 第一个重要设计

```text
Research
≠
Writing
```

很多简单 Demo 会：

```text
search → 直接 prompt → answer
```

GPT Researcher 则明确建立两个生命周期。

---

## 4.3 GPTResearcher.__init__：一次研究 Session 是怎么组装的

### 文件定位

`gpt_researcher/agent.py`

类：

```python
class GPTResearcher:
```

### 不要被参数数量吓到

第一次看构造函数时，只找 4 类东西。

---

### 第一类：请求信息

```python
self.query = query
self.report_type = report_type
self.report_source = report_source
self.tone = tone
self.query_domains = query_domains or []
```

对应：

> 用户到底要研究什么、用什么来源、生成什么报告。

---

### 第二类：执行状态

```python
self.research_sources = []
self.research_images = []
self.visited_urls = visited_urls if visited_urls is not None else set()
self.context = context or []

self.research_costs = 0.0
self.step_costs = {}
```

对应：

> 研究到哪了、读过什么、已经花了多少钱。

---

### 第三类：配置与外部能力

```python
self.cfg = Config(config_path)
self.retrievers = get_retrievers(self.headers, self.cfg)
```

配置决定：

- 用哪个 LLM；
- 用哪个 Search；
- Context Filter；
- Deep Research breadth/depth；
- Scraper worker 数等。

---

### 第四类：核心组件

源码中最关键的一组初始化：

```python
self.research_conductor = ResearchConductor(self)
self.report_generator = ReportGenerator(self)
self.context_manager = ContextManager(self)
self.scraper_manager = BrowserManager(self)
self.source_curator = SourceCurator(self)
```

把它翻译成人话：

```text
GPTResearcher
├─ ResearchConductor：负责“研究怎么跑”
├─ ContextManager：负责“什么信息值得留下”
├─ BrowserManager：负责“网页怎么读”
├─ ReportGenerator：负责“报告怎么写”
└─ SourceCurator：负责“来源怎么进一步筛”
```

### 关键点解读

为什么每个对象都传 `self`？

因为它们都需要访问同一个 Research Session。

例如 ContextManager 会访问：

```text
researcher.cfg
researcher.memory
researcher.prompt_family
researcher.add_costs
```

这种写法简单，但耦合也比较强。

---

## 4.4 GPTResearcher.conduct_research：总入口

### 文件定位

`gpt_researcher/agent.py::conduct_research`

去掉日志后，可以简化成：

```python
async def conduct_research(self, on_progress=None):

    if self.report_type == ReportType.DeepResearch.value:
        return await self._handle_deep_research(on_progress)

    if not (self.agent and self.role):
        self.agent, self.role = await choose_agent(...)

    self.context = await self.research_conductor.conduct_research()

    if image_generation_enabled:
        self.available_images = await self.image_generator.plan_and_generate_images(...)

    return self.context
```

### 第一步：先判断 Deep Research

```python
if self.report_type == ReportType.DeepResearch.value:
```

说明：

> Deep Research 不是普通链路里多加几个循环，而是在入口处直接分叉。

这点后面第 7 章再展开。

---

### 第二步：生成 Agent Role

```python
self.agent, self.role = await choose_agent(...)
```

注意：

> 这里不是创建一个新的 Python Agent 实例。

更准确地说，它生成：

- 一个研究员身份；
- 一个 role prompt。

例如当前任务可能更偏：

```text
“Senior AI Systems Researcher”
```

后续 Planning / Writing 会使用这个角色。

---

### 第三步：委托给 ResearchConductor

```python
self.context = await self.research_conductor.conduct_research()
```

这里是最重要的一跳。

从此以后：

> `GPTResearcher` 不再自己处理 Web Search 细节，而把普通研究流程交给 `ResearchConductor`。

所以如果面试官问：

> “真正的普通 Research 主循环在哪？”

应该优先回答：

```text
gpt_researcher/skills/researcher.py
ResearchConductor
```

---

## 4.5 ResearchConductor.conduct_research：真正的普通研究主流程

### 文件定位

`gpt_researcher/skills/researcher.py`

### 它首先做的不是 Search，而是 Source Routing

项目支持不同数据来源：

```text
source_urls
web
local
hybrid
azure
LangChain documents
vector store
```

核心结构：

```python
if researcher.source_urls:
    research_data = await self._get_context_by_urls(...)

elif researcher.report_source == "web":
    research_data = await self._get_context_by_web_search(...)

elif researcher.report_source == "local":
    document_data = await DocumentLoader(...).load()
    research_data = await self._get_context_by_web_search(
        researcher.query,
        document_data,
        ...
    )

elif researcher.report_source == "hybrid":
    docs_context, web_context = await asyncio.gather(
        local_pass,
        web_pass,
    )
```

### 这一段的设计意义

无论来源是：

- Web；
- Local；
- Hybrid；

最后都尽量转成：

> **Context**

也就是说，下游 Writer 不需要关心资料到底从哪来。

---

### Hybrid 为什么用 gather？

```python
docs_context, web_context = await asyncio.gather(...)
```

因为：

```text
本地文档研究
和
Web 研究
```

彼此独立。

所以可以同时做。

这是项目中第一类自然并发。

---

## 4.6 _get_context_by_web_search：普通 Research 的心脏

### 文件定位

`gpt_researcher/skills/researcher.py::_get_context_by_web_search`

核心代码可以压缩为：

```python
async def _get_context_by_web_search(self, query, scraped_data=None, query_domains=None):

    initial_results = await self._get_initial_search_results(query, query_domains)

    sub_queries = await self.plan_research(
        query,
        query_domains,
        search_results=initial_results,
    )

    if self.researcher.report_type != "subtopic_report":
        sub_queries.append(query)

    initial_context = await self._get_context_from_initial_results(...)

    context = await asyncio.gather(
        *[
            self._process_sub_query(sub_query, scraped_data, query_domains)
            for sub_query in sub_queries
        ]
    )

    context = [c for c in [initial_context, *context] if c]

    return " ".join(context) if context else []
```

这段代码值得反复看。

---

### Step 1：Initial Search

```python
initial_results = await self._get_initial_search_results(...)
```

我们的示例 Query：

```text
比较 LangGraph、AutoGen、CrewAI...
```

先获得搜索结果。

这批结果有两个作用：

#### 作用 A：帮助 Planner

Planner 不必完全依赖模型记忆。

#### 作用 B：本身也可能是有价值的证据

所以后面还有：

```python
_get_context_from_initial_results(...)
```

避免“只拿来规划，然后丢掉”。

---

### Step 2：Planning

```python
sub_queries = await self.plan_research(...)
```

从一个问题生成多个研究方向。

例如：

```text
Q1 LangGraph state model
Q2 AutoGen multi-agent orchestration
Q3 CrewAI flows and crews
Q4 observability comparison
```

---

### Step 3：为什么还把原始 Query 加回去？

```python
sub_queries.append(query)
```

因为 Query Decomposition 可能漏掉：

- overview；
- 高权威总览文档；
- 原问题整体相关的来源。

所以原始 Query 是一个低成本的 recall insurance。

---

### Step 4：并行研究

```python
context = await asyncio.gather(...)
```

假设有 5 个 Query：

```text
q1 ─┐
q2 ─┤
q3 ─┼─ 同时进行
q4 ─┤
q5 ─┘
```

这就是普通 Research 最重要的并行边界。

---

## 4.7 plan_research：Query 是怎么被拆开的

### 文件定位

`gpt_researcher/skills/researcher.py::plan_research`  
`gpt_researcher/actions/query_processing.py`

`plan_research` 会把：

```text
query
+ initial search results
+ agent role
+ parent query
+ report type
```

交给：

```python
generate_sub_queries(...)
```

### Strategic LLM

Query Decomposition 使用的是：

```text
STRATEGIC_LLM
```

而不是把所有任务都混成同一个“LLM”。

项目逻辑上区分：

| LLM 角色 | 主要用途 |
|---|---|
| Fast | 轻量任务 |
| Strategic | Planning / reasoning |
| Smart | 最终写作 / 高质量生成 |

即使默认配置可能指向同一模型，这种职责分离仍然有架构价值。

---

### LLM 输出不可靠怎么办？

假设 Prompt 要求：

```json
["query 1", "query 2"]
```

模型却返回：

```json
{
  "queries": ["query 1", "query 2"]
}
```

或者：

```json
{
  "query": "query 1"
}
```

甚至纯字符串。

源码中的 `_normalize_sub_queries` 会统一成：

```python
list[str]
```

如果一条都解析不出来：

```text
fallback → original query
```

### 这里体现的工程原则

> **Prompt 不是数据契约。**

真正的工程系统必须对模型输出做 normalization。

---

### Strategic LLM 失败怎么办？

大致策略：

```text
Strategic LLM
    ↓ fail
Strategic LLM + explicit token limit
    ↓ fail
Smart LLM
```

这叫业务级 fallback。

---

## 4.8 _process_sub_query：一个研究分支内部发生什么

### 文件定位

`gpt_researcher/skills/researcher.py::_process_sub_query`

这是每个并行 Worker 真正执行的内容。

简化后：

```python
async def _process_sub_query(self, sub_query, scraped_data, query_domains):

    mcp_context = ...
    web_context = ""

    if not scraped_data:
        scraped_data = await self._scrape_data_by_urls(
            sub_query,
            query_domains,
        )

    if scraped_data:
        web_context = await self.researcher.context_manager \
            .get_similar_content_by_query(
                sub_query,
                scraped_data,
            )

    return self._combine_mcp_and_web_context(
        mcp_context,
        web_context,
        sub_query,
    )
```

翻译成人话：

```text
一个 Sub Query
   ↓
拿 MCP 证据（如果有）
   ↓
搜索并读取 Web 页面
   ↓
筛选相关内容
   ↓
把 MCP + Web 合并
   ↓
返回这个研究分支的 Context
```

---

## 4.9 Search → URL → Scrape：资料是怎么读回来的

### Step 1：找来源

`_search_relevant_source_urls` 会遍历 Retriever。

Retriever 可能返回：

```json
{
  "href": "https://...",
  "raw_content": null
}
```

也可能直接返回全文：

```json
{
  "href": "https://...",
  "raw_content": "full text..."
}
```

---

### Step 2：`requires_scraping`

当前代码非常值得学习的一点：

```python
requires_scraping = getattr(retriever, "requires_scraping", None)
```

三种情况。

#### True

```text
Search Provider 只给 preview
→ 一定继续抓网页
```

#### False

```text
Retriever 已经给全文
→ 不重复抓
```

#### None

兼容旧插件：

```text
用历史 heuristic 判断
```

### 为什么要这样做？

因为：

> “有 raw_content”不等于“它真的是全文”。

搜索摘要也可能很长。

---

### Step 3：visited_urls 去重

```text
Q1 → A, B, C
Q2 → A, D, E
Q3 → B, F
```

如果不去重：

```text
A 抓两次
B 抓两次
```

所以系统维护：

```python
researcher.visited_urls
```

---

### Step 4：真正 Scrape

```python
scraped_content = await researcher.scraper_manager.browse_urls(
    new_search_urls
)
```

然后把 Retriever 已经提供的全文合并进去：

```python
scraped_content.extend(prefetched_content)
```

最终得到统一的 pages。

---

## 4.10 ContextManager：为什么搜到的内容不能直接给 LLM

### 文件定位

`gpt_researcher/skills/context_manager.py`

核心方法：

```python
async def get_similar_content_by_query(self, query, pages):
    return await select_context(
        query,
        pages,
        self.researcher.cfg,
        max_results=10,
        embeddings=lambda: self.researcher.memory.get_embeddings(),
    )
```

它本身很薄。

真正策略在：

```text
gpt_researcher/context/select.py
```

---

### `select_context` 决策树

```text
CONTEXT_FILTER
    │
    ├─ none
    │    └─ 全部返回
    │
    ├─ small content
    │    └─ 内容很少，不压缩
    │
    ├─ jev
    │    └─ Jev relevance scoring
    │         ↓ fail
    │       keyword
    │
    ├─ embeddings
    │    └─ embedding similarity
    │         ↓ fail
    │       keyword
    │
    └─ keyword
         └─ BM25 / lexical ranking
```

默认 `auto`：

```text
有 TYPESAFE_API_KEY
→ jev

没有
→ keyword
```

### 一个很重要的细节：Embedding 是 Lazy 的

传进去的是：

```python
embeddings=lambda: ...
```

不是一开始就创建 embedding model。

所以如果实际走 keyword：

> 根本不需要 Embedding API。

这是非常实用的依赖优化。

---

## 4.11 write_report：研究结果如何变成报告

研究完成以后：

```python
report = await researcher.write_report()
```

进入：

```text
GPTResearcher.write_report
  ↓
ReportGenerator.write_report
  ↓
generate_report
  ↓
Smart LLM
```

### 最重要的安全行为：空 Context 不写

Writer 会先检查：

```text
有没有 research content？
```

如果完全没有证据：

> 不应该让模型凭参数记忆伪装成 research report。

所以系统会 abstain。

### 这为什么重要？

Research Agent 的核心承诺是：

```text
“我基于外部证据回答”
```

如果证据为空还继续生成，就已经退化成普通聊天模型了。

---

## 4.12 完整调用链回放

现在把所有东西连起来。

```text
BasicReport.run()
│
├─ GPTResearcher.conduct_research()
│  │
│  ├─ choose_agent()
│  │
│  └─ ResearchConductor.conduct_research()
│     │
│     └─ _get_context_by_web_search()
│        │
│        ├─ _get_initial_search_results()
│        │
│        ├─ plan_research()
│        │  └─ generate_sub_queries()
│        │
│        ├─ append(original_query)
│        │
│        └─ asyncio.gather(...)
│           │
│           └─ _process_sub_query()
│              │
│              ├─ _scrape_data_by_urls()
│              │  ├─ Retriever.search()
│              │  ├─ URL dedup
│              │  └─ BrowserManager.browse_urls()
│              │
│              └─ ContextManager.get_similar_content_by_query()
│                 └─ select_context()
│
└─ GPTResearcher.write_report()
   │
   └─ ReportGenerator.write_report()
      └─ generate_report()
         └─ Smart LLM
```

如果你能不看文档画出这张图，Basic Research 就算真正读懂了。

---

## 4.13 本章面试题

### Q1：普通 Research 的核心函数是哪一个？

推荐回答：

> “从业务入口看是 `GPTResearcher.conduct_research`，但真正 Web Research 的核心流程在 `ResearchConductor._get_context_by_web_search`。它先 Initial Search，再 Planning 生成 Sub Queries，然后用 `asyncio.gather` 并发执行 `_process_sub_query`，最后聚合 Context。”

### Q2：为什么 Planning 前要先搜一次？

> “这是 grounded planning。Planner 不完全依赖模型参数知识，而是先接触当前外部搜索结果。对近期项目、新术语和版本变化尤其重要。”

### Q3：为什么还要追加 original query？

> “Query decomposition 可能遗漏整体性来源，原始 Query 可以补充 overview 类结果，相当于提高 recall 的保险。”

### Q4：为什么 `requires_scraping` 很重要？

> “因为 Search Result 的 raw_content 可能只是很长的 snippet，不等于全文。显式 content contract 可以避免把 preview 错当正文，也能避免已经有全文的 Retriever 被重复抓取。”

---

## 4.14 本章总结

```text
┌─────────────────────────────────────────────────────────┐
│                   源码阅读只记 8 个跳转                  │
│                                                         │
│ BasicReport                                             │
│   ↓                                                     │
│ GPTResearcher                                           │
│   ↓                                                     │
│ ResearchConductor                                       │
│   ↓                                                     │
│ _get_context_by_web_search                              │
│   ↓                                                     │
│ plan_research                                           │
│   ↓                                                     │
│ _process_sub_query                                      │
│   ↓                                                     │
│ ContextManager                                          │
│   ↓                                                     │
│ ReportGenerator                                         │
└─────────────────────────────────────────────────────────┘
```

下一章开始把最复杂的 Planning、Retriever 和 MCP 单独拆开。

➡️ [05 - Planning、Retrieval 与 MCP](../05-planning-retrieval-mcp/README.md)
