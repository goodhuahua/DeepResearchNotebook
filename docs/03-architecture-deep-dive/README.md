# 03 - 架构深入解析

> 🎯 **本章目标**：把 GPT Researcher 从“几个类名”变成一张能在脑中运行的系统图。重点不是背目录，而是理解：**谁拥有状态、谁控制流程、数据在每层变成什么形状。**

---

## 目录

- [3.1 从最重要的问题开始：状态放在哪里](#31-从最重要的问题开始状态放在哪里)
- [3.2 六层架构](#32-六层架构)
- [3.3 核心对象关系](#33-核心对象关系)
- [3.4 一条 Query 的完整数据流](#34-一条-query-的完整数据流)
- [3.5 Search、Scrape、Context 为什么必须分三层](#35-searchscrapecontext-为什么必须分三层)
- [3.6 Agent 中 LLM 到底在哪些地方出现](#36-agent-中-llm-到底在哪些地方出现)
- [3.7 关键设计模式](#37-关键设计模式)
- [3.8 架构上的优点与技术债](#38-架构上的优点与技术债)
- [3.9 面试高频题](#39-面试高频题)
- [3.10 本章总结](#310-本章总结)

---

## 3.1 从最重要的问题开始：状态放在哪里

很多人读 Agent 项目时先找：

> “Agent Loop 在哪？”

但工程上更应该先问：

> **一次任务执行过程中，状态到底放在哪里？**

在 GPT Researcher 中，核心状态主要在：

```python
class GPTResearcher:
    self.query
    self.context
    self.visited_urls
    self.research_sources
    self.agent
    self.role
    self.research_costs
    self.step_costs
```

所以你可以把 `GPTResearcher` 理解成：

```text
一次 Research Session
```

例如我们的示例：

```text
query
= "比较 LangGraph、AutoGen、CrewAI..."

visited_urls
= {
  "https://...",
  "https://..."
}

context
= "筛选后的研究证据..."

research_costs
= 0.0 → 不断累计
```

### 为什么理解这一点很重要？

因为后面的：

- ResearchConductor
- BrowserManager
- ContextManager
- ReportGenerator

都会拿到同一个 parent researcher。

它们并不是完全独立的小服务，而是：

> **围绕同一个 Research Session 协作。**

---

## 3.2 六层架构

可以把项目理解为六层。

```text
┌────────────────────────────────────────────────────┐
│ Layer 6  Delivery                                  │
│ FastAPI / WebSocket / CLI / BasicReport            │
├────────────────────────────────────────────────────┤
│ Layer 5  Orchestration                             │
│ GPTResearcher                                      │
├────────────────────────────────────────────────────┤
│ Layer 4  Domain Skills                             │
│ ResearchConductor / Browser / Context / Writer     │
├────────────────────────────────────────────────────┤
│ Layer 3  Actions                                   │
│ choose_agent / generate_sub_queries / generate...  │
├────────────────────────────────────────────────────┤
│ Layer 2  Adapters                                  │
│ Retriever / Scraper / LLM Provider / VectorStore   │
├────────────────────────────────────────────────────┤
│ Layer 1  External World                            │
│ Search API / Website / LLM API / MCP / Documents   │
└────────────────────────────────────────────────────┘
```

下面逐层解释。

---

### Layer 1：External World

真正的外部系统：

- 搜索引擎；
- 网页；
- OpenAI / Anthropic / Google；
- MCP Server；
- 本地文档；
- Vector Store。

这些系统都：

- 可能慢；
- 可能失败；
- 输出格式可能变化。

因此不能让业务逻辑直接依赖它们。

---

### Layer 2：Adapters

例如：

```text
TavilySearch
GoogleSearch
MCPRetriever

GenericLLMProvider

Scraper
```

它们负责：

> **把不同外部系统转换成项目内部可以使用的统一接口。**

这就是 Adapter 思想。

---

### Layer 3：Actions

Actions 是更细粒度的“动作”。

例如：

```text
choose_agent()
generate_sub_queries()
get_search_results()
generate_report()
```

一个 Action 通常只完成一件具体的事情。

---

### Layer 4：Domain Skills

Skills 是“为了一个业务目标，把多个动作组合起来”。

例如：

```text
ResearchConductor
  = Planning
  + Search
  + Scrape
  + Context
  + Aggregate
```

`ReportGenerator` 则负责报告写作流程。

---

### Layer 5：GPTResearcher

它负责：

- 保存一次请求的状态；
- 加载配置；
- 初始化组件；
- 选择 Basic / Deep；
- 暴露 `conduct_research()` / `write_report()`。

所以它更像：

> **Facade + Session State + Dependency Container**

而不是“所有研究算法都在这里”。

---

### Layer 6：Delivery

最外层负责：

> 用户怎么调用它？

可能是：

- Python SDK；
- CLI；
- FastAPI；
- WebSocket UI；
- DetailedReport wrapper；
- Multi-Agent route。

---

## 3.3 核心对象关系

`GPTResearcher.__init__` 中最值得看的不是几十个参数，而是下面这些对象：

```python
self.research_conductor = ResearchConductor(self)
self.report_generator = ReportGenerator(self)
self.context_manager = ContextManager(self)
self.scraper_manager = BrowserManager(self)
self.source_curator = SourceCurator(self)
```

### 这里发生了什么？

每个模块都拿到了：

```text
self = 当前 GPTResearcher
```

于是：

```text
ContextManager
可以访问
researcher.cfg
researcher.memory
researcher.add_costs
researcher.prompt_family
```

### 好处

- 组装简单；
- 新功能接入快；
- 所有模块共享状态方便。

### 问题

- 隐式依赖很多；
- 单元测试需要造一个“大号 fake researcher”；
- Skill 很容易依赖越来越多 parent 属性；
- `GPTResearcher` 有变成 God Object 的趋势。

这就是一个很好的面试 trade-off。

---

## 3.4 一条 Query 的完整数据流

现在不要看函数，只看“数据长什么样”。

### Stage 1：用户 Query

```text
"比较 LangGraph、AutoGen、CrewAI..."
```

类型：

```python
str
```

---

### Stage 2：Initial Search Results

大致类似：

```json
[
  {
    "title": "LangGraph overview",
    "href": "https://...",
    "body": "search snippet..."
  },
  {
    "title": "AutoGen docs",
    "href": "https://...",
    "body": "..."
  }
]
```

注意：

> 这里通常只是 Search Result，不一定是网页全文。

---

### Stage 3：Sub Queries

Planner 输出：

```python
[
    "LangGraph orchestration and state management",
    "AutoGen multi-agent architecture",
    "CrewAI crew and flow architecture",
    "observability comparison"
]
```

数据从：

```text
一个复杂问题
```

变成：

```text
多个可并行研究的问题
```

---

### Stage 4：URLs / Prefetched Content

Retriever 可能返回两类数据：

#### 类型 A：只有 URL

```json
{
  "url": "https://...",
  "snippet": "..."
}
```

需要 Scrape。

#### 类型 B：已经有全文

```json
{
  "url": "https://...",
  "raw_content": "full content..."
}
```

可能不需要再抓。

---

### Stage 5：Scraped Pages

```json
[
  {
    "url": "https://...",
    "raw_content": "真正的网页正文..."
  }
]
```

---

### Stage 6：Context

经过筛选后：

```text
Title: ...
Content: 与当前 Query 最相关的内容...
Source: https://...
```

此时才真正适合进入 Writer。

---

### Stage 7：Report

最终：

```text
# Comparison of Agent Frameworks

## Orchestration
...

## State Management
...
```

---

## 3.5 Search、Scrape、Context 为什么必须分三层

这是项目里最重要的分层之一。

### Search

问题：

> **哪些来源可能有答案？**

输出：

- URL；
- title；
- snippet；
- 有时包含 raw_content。

### Scrape

问题：

> **这个页面真正写了什么？**

输出：

- 网页全文；
- PDF 文本；
- 文档内容。

### Context

问题：

> **全文里哪些部分值得给 LLM？**

输出：

- 相关片段；
- 压缩后的证据。

---

### 如果把三层混在一起会怎样？

假设 Search API 返回：

```text
500 字 snippet
```

如果代码认为：

> “500 字挺长，应该就是全文了。”

可能导致：

- 关键信息没抓到；
- Citation 来源弱；
- Writer 只看到摘要；
- 报告深度不足。

因此当前源码甚至引入：

```python
requires_scraping
```

让 Retriever 明确告诉系统：

> “我的结果只是 preview”  
> 或  
> “我已经提供 full content”。

这是成熟系统才会遇到的细节。

---

## 3.6 Agent 中 LLM 到底在哪些地方出现

很多初学者会误以为：

> “整个 Agent 都是 LLM 控制的。”

其实不是。

### LLM 节点 1：Choose Agent

```text
Query
→ 生成任务相关的研究角色
```

### LLM 节点 2：Planning

```text
Query + Initial Search
→ Sub Queries
```

### LLM 节点 3：Deep Research

```text
Research Result
→ Learnings + Follow-up Questions
```

### LLM 节点 4：Writer

```text
Filtered Context
→ Report
```

而这些步骤主要是普通代码：

```text
并发
Search API 调用
URL 去重
网页抓取
Context 路由
Fallback
Cost 累计
```

因此这个项目最值得学的就是：

> **Agent = LLM 决策 + 普通软件工程。**

---

## 3.7 关键设计模式

### 模式 1：Facade

`GPTResearcher` 给外部一个简单接口：

```python
conduct_research()
write_report()
```

外部不用知道内部几十个模块。

---

### 模式 2：Strategy

Context Filter 可以切换：

```text
keyword
jev
embeddings
none
```

MCP 也有：

```text
fast
deep
disabled
```

同一个流程，不同策略。

---

### 模式 3：Adapter / Factory

```text
Retriever name
→ Retriever class

provider name
→ LLM adapter
```

核心 Workflow 不需要知道每个 SDK 怎么调用。

---

### 模式 4：Orchestrator-Workers

Planning 后：

```text
Planner
→ q1 worker
→ q2 worker
→ q3 worker
→ gather
```

Deep / Detailed 又会创建更多 Researcher worker。

---

### 模式 5：Graceful Degradation

例如 Context：

```text
Jev fail
→ keyword

Embedding fail
→ keyword
```

高阶能力失败，不代表整个任务失败。

---

## 3.8 架构上的优点与技术债

### 做得好的地方

#### 1. Research / Writing 分离

调试很清晰：

```text
报告不好
→ 是没搜到？
→ 还是写坏了？
```

#### 2. Domain Workflow 清晰

Research 的核心步骤稳定，所以没必要用完全开放式 Agent Loop。

#### 3. Provider 隔离

换 Search / LLM 不需要重写主流程。

#### 4. 并发边界自然

Sub Query 天生适合并发。

---

### 技术债

#### 1. GPTResearcher 偏重

状态 + 配置 + Services 混在同一个对象。

#### 2. 数据 Schema 不够统一

代码需要兼容：

```text
href / url
body / content
string / list / dict context
```

说明内部数据契约还可以更严格。

#### 3. Evidence Provenance 还不够结构化

大量 Context 最终还是字符串。

如果要做更严格 Citation Verification，最好维护 typed Evidence。

---

## 3.9 面试高频题

### Q：为什么说 GPT Researcher 更像 Workflow Agent？

> “普通模式的控制路径是显式的：Role Selection → Initial Search → Query Planning → Parallel Sub-query Research → Context Aggregation → Writing。模型不会每一步自由选择工具。这样做牺牲了一些开放性，但获得了更稳定的延迟、成本和可测试性。”

### Q：Search / Scrape / Context 分层有什么价值？

> “三层分别对应 candidate discovery、content acquisition 和 evidence selection。如果混在一起，很容易把 snippet 当全文、重复抓已经提供全文的 API，或者把大量无关正文直接送给 LLM。”

### Q：你觉得架构最大问题是什么？

> “Skills 都直接依赖整个 GPTResearcher parent，对开发速度很友好，但隐式依赖较多。我会逐步拆成 ResearchRequest、ResearchState、ResearchServices 和 SharedBudget，让 child researcher 的状态共享更明确。”

---

## 3.10 本章总结

```text
┌──────────────────────────────────────────────────────┐
│                  GPT Researcher 架构心智图             │
│                                                      │
│ User/API                                             │
│    ↓                                                 │
│ GPTResearcher  ← 一次 Research Session                │
│    ↓                                                 │
│ ResearchConductor ← 普通研究 Workflow                 │
│    ├─ Planning                                       │
│    ├─ Retriever                                      │
│    ├─ Browser/Scraper                                │
│    └─ ContextManager                                 │
│             ↓                                        │
│          Context                                     │
│             ↓                                        │
│       ReportGenerator                                │
│             ↓                                        │
│           Report                                     │
└──────────────────────────────────────────────────────┘
```

下一章开始真正沿源码走一遍。

➡️ [04 - 源码逐行走读](../04-source-code-walkthrough/README.md)
