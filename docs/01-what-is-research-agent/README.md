# 01 - 什么是 Research Agent

> 🎯 **本章目标**：在看 GPT Researcher 源码之前，先把“Research Agent 到底在解决什么问题”讲清楚。读完后，你应该能区分普通 LLM、搜索、RAG、ReAct Agent 和 Deep Research Agent。

---

## 目录

- [1.1 从一个真实任务开始](#11-从一个真实任务开始)
- [1.2 普通 LLM 为什么不够](#12-普通-llm-为什么不够)
- [1.3 Search、RAG、Agent 的区别](#13-searchragagent-的区别)
- [1.4 Research Agent 的完整工作流](#14-research-agent-的完整工作流)
- [1.5 GPT Researcher 属于哪种 Agent](#15-gpt-researcher-属于哪种-agent)
- [1.6 最重要的几个概念](#16-最重要的几个概念)
- [1.7 面试怎么说](#17-面试怎么说)
- [1.8 本章总结](#18-本章总结)

---

## 1.1 从一个真实任务开始

假设用户问：

> “比较 LangGraph、AutoGen、CrewAI 在 Agent 编排、状态管理、可观测性方面的设计差异，并给出适用场景。”

如果只是问：

> “Python 里的 list 和 tuple 有什么区别？”

模型参数里的知识通常就足够。

但上面的研究任务不一样，它至少包含四个难点：

```text
难点 1：信息分散
  → 三个项目的资料分布在文档、博客、源码、更新日志中

难点 2：信息会变化
  → 当前版本和一年前可能完全不同

难点 3：问题有多个维度
  → 编排、状态、可观测性不是一个搜索词能覆盖

难点 4：最后需要综合判断
  → 不只是“找到网页”，而是组织成有结构的报告
```

这就是 Research Agent 要解决的问题。

---

## 1.2 普通 LLM 为什么不够

### 普通 LLM 的典型模式

```text
用户问题
   ↓
LLM 根据参数知识 + 当前 Prompt
   ↓
生成答案
```

它最大的限制不是“不会写”，而是：

> **它无法保证自己掌握了完成研究任务所需的最新证据。**

例如模型可能：

- 知道 LangGraph 的旧架构；
- 不知道最近新增的能力；
- 对一个项目记得很多，对另一个项目记得很少；
- 把博客里的二手描述当成事实；
- 给出一篇结构漂亮但证据薄弱的报告。

### Research Agent 多出来什么？

```text
用户问题
   ↓
规划：我应该研究哪些方面？
   ↓
搜索：哪里可能有答案？
   ↓
阅读：网页里真正写了什么？
   ↓
筛选：哪些内容和当前问题有关？
   ↓
综合：多个来源之间怎么组织？
   ↓
写作：基于证据生成最终报告
```

因此：

> **Research Agent 的本质，是把“获取证据”和“生成答案”分开。**

---

## 1.3 Search、RAG、Agent 的区别

这是第一次读 Research Agent 最容易混淆的地方。

### Search：帮你找到候选资料

输入：

```text
"LangGraph state management"
```

输出：

```json
[
  {
    "title": "...",
    "url": "...",
    "snippet": "..."
  }
]
```

Search 解决：

> **去哪里找？**

它不负责真正读完整网页，也不负责写报告。

---

### RAG：从资料中找到相关内容，再让模型回答

典型 RAG：

```text
Query
  ↓
Vector / Keyword Retrieval
  ↓
Relevant Chunks
  ↓
LLM
  ↓
Answer
```

它解决：

> **已有资料里，哪些片段和问题相关？**

---

### Research Agent：主动组织一整个研究过程

```text
复杂 Query
  ↓
拆成多个研究方向
  ↓
分别搜索
  ↓
读取大量来源
  ↓
筛选证据
  ↓
必要时继续追问
  ↓
汇总报告
```

区别在于：

> RAG 通常默认“问题和知识库都已经准备好”；Research Agent 还要自己决定**搜什么、读什么、是否继续研究**。

---

## 1.4 Research Agent 的完整工作流

我们继续用贯穿全书的例子。

### 用户输入

```text
比较 LangGraph、AutoGen、CrewAI 在
Agent 编排、状态管理、可观测性方面的设计差异，
并给出适用场景。
```

### 第一步：理解任务

系统需要判断：

- 这是技术调研；
- 需要比较多个框架；
- 需要最新外部资料；
- 最终输出应是结构化分析报告。

### 第二步：先获得一点外部知识

GPT Researcher 会先做 Initial Search。

想象得到：

```text
Result 1: LangGraph overview
Result 2: AutoGen architecture
Result 3: CrewAI docs
...
```

为什么不是先拆问题？

因为模型可能对当前版本不熟悉。

**先看一眼现实世界，再规划。**

---

### 第三步：拆成 Sub Queries

可能得到：

```text
Q1: LangGraph orchestration and state management architecture
Q2: AutoGen multi-agent orchestration and state handling
Q3: CrewAI crew/flow orchestration architecture
Q4: observability and tracing comparison
Q5: suitable use cases and trade-offs
```

这一步叫：

> **Query Decomposition**

---

### 第四步：并行研究

```text
                    原始问题
                       │
                 Query Planner
          ┌────────────┼────────────┐
          ▼            ▼            ▼
         Q1           Q2           Q3
          │            │            │
       Search       Search       Search
          │            │            │
        Read         Read         Read
          │            │            │
       Context      Context      Context
          └────────────┼────────────┘
                       ▼
                    Aggregate
```

这一步既提升信息覆盖，也降低总等待时间。

---

### 第五步：Context Engineering

假设最终抓到了 20 篇网页，每篇几千到几万字。

不能全部塞给模型。

所以系统还需要：

```text
大量网页正文
   ↓
相关性判断
   ↓
只保留对当前 Sub Query 真正有帮助的片段
```

这就是 Context Engineering。

---

### 第六步：Synthesis

最终 Writer 得到的是：

```text
“已经筛过的一组研究证据”
```

而不是：

```text
“用户原问题”
```

于是它做的事情从“凭记忆回答”变成：

> **基于已获取证据进行综合写作。**

---

## 1.5 GPT Researcher 属于哪种 Agent

### 它不是经典 ReAct Loop

经典 ReAct：

```text
LLM:
  我下一步要搜索
    ↓
Search Tool
    ↓
Observation
    ↓
LLM:
  我再决定下一步
    ↓
...
```

特点：

> 每一轮“接下来做什么”都主要由模型决定。

---

### GPT Researcher 普通模式更像 Agentic Workflow

```text
代码固定：
Role → Initial Search → Planning → Parallel Research → Context → Writer

模型负责：
  角色生成
  Query Decomposition
  最终综合写作
```

也就是说：

> **高不确定性的决策交给 LLM，稳定的研究步骤交给代码。**

这是 GPT Researcher 最重要的架构思想之一。

---

### 为什么 Research 适合这种方式？

因为研究任务的基本过程非常稳定：

```text
找问题
→ 找资料
→ 读资料
→ 筛资料
→ 写报告
```

没有必要让模型每一轮都重新决定：

> “我现在是不是应该调用 Scraper？”

这些动作完全可以由代码确定。

这样更：

- 可预测；
- 容易并行；
- 容易测试；
- 容易控制成本；
- 容易定位失败在哪一步。

---

## 1.6 最重要的几个概念

### 1. Query

用户最初的问题。

```text
“比较三个 Agent 框架……”
```

### 2. Sub Query

Planner 为了研究而拆出来的小问题。

```text
“LangGraph state management architecture”
```

### 3. Retriever

负责寻找候选来源。

可以把它理解成：

> **搜索员。**

它可能调用 Tavily、Google、Exa、学术搜索或 MCP。

### 4. Scraper

负责打开 URL，拿到真正正文。

可以理解成：

> **阅读员。**

### 5. Context

经过筛选后，最终准备给模型看的研究证据。

不是“整个网页”，而是：

> **真正与当前问题相关的内容。**

### 6. Writer

基于 Context 生成报告。

可以理解成：

> **主笔。**

### 7. Deep Research

普通研究结束后，根据结果再生成新的 Follow-up Questions，继续向下查。

可以理解成：

> **发现新线索以后继续追查。**

---

## 1.7 面试怎么说

### Q：Research Agent 和普通 RAG 有什么区别？

可以回答：

> “RAG 更关注从已有知识库中检索相关内容并增强生成，而 Research Agent 还承担任务规划和信息获取本身。以 GPT Researcher 为例，它会先根据初始搜索结果生成 Sub Queries，再并发搜索和抓取网页，通过 Context Filter 选择证据，最后由 Writer 综合生成报告。Deep Research 还会根据已有结果产生 Follow-up Questions 继续研究，因此它是一个完整的 research workflow，而不只是单次 retrieval。”

### Q：GPT Researcher 是 ReAct Agent 吗？

可以回答：

> “普通模式不属于典型的开放式 ReAct Loop。它更像 domain-specific agentic workflow：控制流由代码固定为 Role Selection、Planning、Retrieval、Scraping、Context Selection 和 Writing，LLM 只负责部分高熵决策。Deep Research 会增加递归自治，但仍受 breadth、depth 和 concurrency 约束。”

---

## 1.8 本章总结

```text
┌─────────────────────────────────────────────────┐
│              本章只需要记住 4 件事                │
│                                                 │
│  1. Research Agent = 研究流程，而不只是 LLM       │
│                                                 │
│  2. Search 负责找资料，Scraper 负责读资料          │
│                                                 │
│  3. Context Engineering 决定哪些证据进入 LLM      │
│                                                 │
│  4. GPT Researcher 普通模式是强流程约束 Workflow  │
│     不是开放式 ReAct while-loop                  │
└─────────────────────────────────────────────────┘
```

下一章不看复杂源码，我们先整体认识 GPT Researcher 这个项目。

➡️ [02 - GPT Researcher 项目概览](../02-gpt-researcher-overview/README.md)
