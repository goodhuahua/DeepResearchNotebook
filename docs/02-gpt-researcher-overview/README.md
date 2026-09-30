# 02 - GPT Researcher 项目概览

> 🎯 **本章目标**：在进入源码前，先知道项目有哪些模块、有哪些运行模式、最小使用方式是什么，以及读源码应该从哪里开始。

---

## 目录

- [2.1 项目到底做什么](#21-项目到底做什么)
- [2.2 最小使用方式](#22-最小使用方式)
- [2.3 四种核心运行模式](#23-四种核心运行模式)
- [2.4 源码目录怎么看](#24-源码目录怎么看)
- [2.5 最重要的 8 个文件](#25-最重要的-8-个文件)
- [2.6 第一次读源码不要看什么](#26-第一次读源码不要看什么)
- [2.7 一次请求的宏观流程](#27-一次请求的宏观流程)
- [2.8 面试要点](#28-面试要点)
- [2.9 本章总结](#29-本章总结)

---

## 2.1 项目到底做什么

GPT Researcher 的核心目标可以概括为：

> **输入一个研究问题，自动完成“找资料 → 阅读 → 筛选 → 综合 → 输出报告”。**

它不是搜索引擎。

也不是一个简单的 ChatGPT Wrapper。

它更像：

```text
研究助理团队
├─ Planner：决定要查哪些问题
├─ Searcher：找资料
├─ Reader：读网页
├─ Curator：筛选相关证据
└─ Writer：写最终报告
```

只不过这些角色并不一定对应 5 个独立 Agent 对象，而是由不同模块和 LLM 节点共同完成。

---

## 2.2 最小使用方式

上游最核心的 SDK 用法可以压缩为：

```python
from gpt_researcher import GPTResearcher

researcher = GPTResearcher(
    query="比较 LangGraph、AutoGen、CrewAI 的架构差异"
)

context = await researcher.conduct_research()
report = await researcher.write_report()
```

只看这三行，你已经能发现一个关键设计：

```text
conduct_research()
和
write_report()
是分开的
```

这意味着项目把任务拆成：

### Phase A：Research

```text
世界上有哪些证据？
```

### Phase B：Writing

```text
如何基于证据写报告？
```

这也是后面所有源码分析的主轴。

---

## 2.3 四种核心运行模式

这是整个项目最容易让新读者混乱的地方。

### 模式 1：Basic Research

最应该先学。

```text
Query
→ Planning
→ 多个 Sub Query
→ Search / Scrape / Context
→ Writer
```

适合普通研究报告。

---

### 模式 2：Detailed Report

目标不是“搜得更深”，而是：

> **写更长、更有章节结构的报告。**

流程：

```text
先做一次全局研究
   ↓
生成 Subtopics
   ↓
Subtopic A 创建一个 GPTResearcher
   ↓
写 A
   ↓
Subtopic B 创建一个 GPTResearcher
   ↓
B 会看到 A 已经写过什么
   ↓
...
```

它重点解决：

- 长报告章节拆分；
- 章节之间重复；
- 已写内容协调。

---

### 模式 3：Deep Research

目标是：

> **发现新线索以后继续追查。**

```text
Query
  ↓
生成 breadth 个搜索方向
  ↓
每个方向创建 Nested GPTResearcher
  ↓
提取 Learnings + Follow-up Questions
  ↓
depth > 1 ?
  ├─ 是 → 继续向下研究
  └─ 否 → 汇总
```

它重点解决“研究深度”。

---

### 模式 4：Multi-Agent

这是独立的 LangGraph 工作流。

角色包括：

```text
Chief Editor
Editor
Researcher
Writer
Reviewer / Reviser
Fact Checker
Visualizer
Publisher
Human
```

它重点解决：

> **角色分工 + 审核闭环。**

---

### 四种模式对比

| 模式 | 主要目标 | 控制结构 | 新手阅读优先级 |
|---|---|---|---|
| Basic | 完成普通研究 | Plan → Fan-out → Gather | ★★★★★ |
| Detailed | 写长报告 | 顺序 Subtopic Worker | ★★★★☆ |
| Deep | 深挖证据 | 递归搜索树 | ★★★★☆ |
| Multi-Agent | 多角色审核 | LangGraph 状态图 | ★★☆☆☆ |

---

## 2.4 源码目录怎么看

第一次打开仓库，会看到很多目录。

先只看这一小部分：

```text
gpt_researcher/
├── agent.py
├── actions/
│   ├── agent_creator.py
│   ├── query_processing.py
│   ├── retriever.py
│   └── report_generation.py
├── skills/
│   ├── researcher.py
│   ├── browser.py
│   ├── context_manager.py
│   ├── writer.py
│   └── deep_research.py
├── context/
│   ├── select.py
│   ├── lexical.py
│   └── compression.py
├── retrievers/
├── scraper/
└── llm_provider/

backend/
└── report_type/
    ├── basic_report/
    └── detailed_report/

multi_agents/
└── agents/
```

先不要试图看完所有目录。

---

## 2.5 最重要的 8 个文件

### 1. `gpt_researcher/agent.py`

核心类：

```text
GPTResearcher
```

它保存一次研究任务的状态，并组装各模块。

可以类比：

> **项目经理。**

---

### 2. `gpt_researcher/skills/researcher.py`

核心类：

```text
ResearchConductor
```

普通 Research 真正的主流程在这里。

可以类比：

> **研究执行负责人。**

---

### 3. `actions/query_processing.py`

负责：

- Initial Search helper；
- Query Decomposition；
- 结构化输出修复。

可以类比：

> **研究计划制定器。**

---

### 4. `skills/browser.py`

负责：

- 把 URL 交给 Scraper；
- 记录来源；
- 处理图片。

可以类比：

> **网页阅读调度器。**

---

### 5. `skills/context_manager.py`

负责：

> 从抓回来的大量内容中，找真正与问题相关的部分。

可以类比：

> **资料整理员。**

---

### 6. `context/select.py`

真正决定：

```text
keyword?
Jev?
embeddings?
none?
```

这是 Context Strategy 的核心。

---

### 7. `skills/writer.py`

核心类：

```text
ReportGenerator
```

负责准备写作参数，并调用最终报告生成 Action。

---

### 8. `skills/deep_research.py`

只在 Deep Research 模式重点阅读。

负责：

- breadth；
- depth；
- Nested GPTResearcher；
- Learnings；
- Follow-up Questions。

---

## 2.6 第一次读源码不要看什么

先不要深入：

- 所有 Retriever 实现；
- 所有 Scraper 实现；
- 前端 Next.js；
- 图片生成；
- 每一个 LLM Provider；
- AG2 版本 Multi-Agent；
- 所有测试。

原因不是它们不重要，而是：

> **你还没有主干，分支越看越乱。**

正确顺序：

```text
先知道“谁调用谁”
再看“每个谁内部怎么做”
```

---

## 2.7 一次请求的宏观流程

我们继续使用：

> 比较 LangGraph、AutoGen、CrewAI……

### 入口

```python
researcher = GPTResearcher(query=...)
```

此时主要是：

- 保存 Query；
- 加载 Config；
- 创建 Retriever；
- 创建 ResearchConductor；
- 创建 ContextManager；
- 创建 BrowserManager；
- 创建 ReportGenerator。

还没有真正研究。

---

### 执行研究

```python
await researcher.conduct_research()
```

进入：

```text
GPTResearcher.conduct_research()
   ↓
choose_agent()
   ↓
ResearchConductor.conduct_research()
   ↓
_get_context_by_web_search()
```

然后：

```text
Initial Search
   ↓
plan_research()
   ↓
Sub Queries
   ↓
asyncio.gather(...)
```

每个 Sub Query：

```text
Search
→ URL
→ Scrape
→ Context Filter
```

---

### 写报告

```python
await researcher.write_report()
```

进入：

```text
ReportGenerator.write_report()
   ↓
generate_report()
   ↓
Smart LLM
```

到这里一次 Basic Research 才结束。

---

## 2.8 面试要点

### Q：`GPTResearcher` 是不是核心 Research 算法？

更准确的说法：

> “GPTResearcher 是一次 research session 的 Facade 和状态容器。它保存 query、context、visited_urls、cost 等状态，并组装 ResearchConductor、ContextManager、BrowserManager、ReportGenerator。普通研究真正的流程控制主要在 ResearchConductor。”

### Q：为什么 `conduct_research` 和 `write_report` 分开？

> “因为检索证据和内容生成是两类问题。分开以后 Research 可以独立复用，Writer 可以使用外部 context，调试时也能判断问题到底来自 retrieval 还是 generation。”

---

## 2.9 本章总结

```text
┌──────────────────────────────────────────────┐
│              第一次读项目只记这张图            │
│                                              │
│ GPTResearcher                                │
│      │                                       │
│      ▼                                       │
│ ResearchConductor                            │
│      │                                       │
│      ├─ Planning                             │
│      ├─ Retriever                            │
│      ├─ Browser/Scraper                      │
│      └─ ContextManager                       │
│             │                                │
│             ▼                                │
│      Aggregated Context                      │
│             │                                │
│             ▼                                │
│      ReportGenerator                         │
│                                              │
└──────────────────────────────────────────────┘
```

下一章开始把这张图展开成完整架构。

➡️ [03 - 架构深入解析](../03-architecture-deep-dive/README.md)
