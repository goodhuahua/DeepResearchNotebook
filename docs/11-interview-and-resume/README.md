# 11 - 面试与简历：把源码理解变成可验证的项目能力

> 🎯 **本章目标**：把前 10 章学到的内容转换成面试表达和简历项目。重点不是背“标准答案”，而是形成一条可追问、可画图、可落到源码、可拿实验数据证明的项目叙事。

---

## 目录

- [11.1 面试官真正想确认什么](#111-面试官真正想确认什么)
- [11.2 60 秒项目介绍怎么组织](#112-60-秒项目介绍怎么组织)
- [11.3 5 分钟架构讲解](#113-5-分钟架构讲解)
- [11.4 最值得准备的 12 个问题](#114-最值得准备的-12-个问题)
- [11.5 源码追问题怎么回答](#115-源码追问题怎么回答)
- [11.6 二次开发项目怎么讲](#116-二次开发项目怎么讲)
- [11.7 简历应该写什么、不应该写什么](#117-简历应该写什么不应该写什么)
- [11.8 可使用的简历 Bullet 模板](#118-可使用的简历-bullet-模板)
- [11.9 一周复习计划](#119-一周复习计划)
- [11.10 最终自测](#1110-最终自测)
- [11.11 本章总结](#1111-本章总结)

---

## 11.1 面试官真正想确认什么

如果你把 GPT Researcher 放在 Agent 岗位简历里，面试官通常不只是想知道：

> “你有没有看过这个 GitHub 项目？”

更可能会逐层确认：

~~~text
第一层：你知道它做什么吗？
   ↓
第二层：你能画出主链路吗？
   ↓
第三层：你能落到具体源码吗？
   ↓
第四层：你理解为什么这么设计吗？
   ↓
第五层：你能指出 trade-off 和技术债吗？
   ↓
第六层：你自己改过什么？
   ↓
第七层：你怎么证明改造有效？
~~~

真正有区分度的是后四层。

所以这份笔记的目标从来不应该是：

~~~text
“我记住了 20 个类名”
~~~

而应该是：

> **“我能从架构、源码、工程约束和实验结果四个层次解释一个完整 Agent 系统。”**

---

## 11.2 60 秒项目介绍怎么组织

不要从 Feature List 开始：

> “GPT Researcher 是一个开源项目，支持 Tavily、MCP、LangGraph……”

更好的结构是：

~~~text
问题
→ 核心架构
→ 一个关键设计
→ 你做的改造
→ 数据结果
~~~

例如你的表达骨架可以是：

> “我主要研究并二次开发了 GPT Researcher 这类 Deep Research Agent。它解决的是开放式研究任务：先对用户问题做 grounded planning，把问题拆成多个 sub-query，再并发检索和抓取网页，通过 context filter 压缩为 evidence，最后由 writer 做 evidence-grounded synthesis。它和经典 ReAct 不同，普通路径是一个显式的 Plan–Fan-out–Gather–Synthesize workflow。我的二次开发重点放在共享预算和自适应研究深度，用 Coverage Evaluator 判断 evidence gap，只在必要时继续扩展 research frontier，并通过 benchmark 比较质量、成本和延迟。”

这段话里已经埋了很多可追问点：grounded planning、sub-query、context filter、ReAct vs workflow、budget、adaptive research、benchmark。

---

## 11.3 5 分钟架构讲解

白板上先画这个：

~~~text
User Query
   ↓
GPTResearcher
   ↓
ResearchConductor
   ├─ Initial Search
   ├─ Planning
   │    └─ Strategic LLM
   ├─ Parallel Sub Queries
   │    ├─ Retriever
   │    ├─ Scraper
   │    └─ Context Filter
   └─ Context Aggregation
           ↓
      ReportGenerator
           ↓
        Smart LLM
           ↓
         Report
~~~

然后只讲四件事。

### 第一件：控制面在哪里

~~~text
GPTResearcher
= Session / Facade

ResearchConductor
= 普通 Research Workflow
~~~

### 第二件：LLM 和确定性代码如何分工

LLM 负责：

~~~text
角色
Planning
Deep Follow-up
Synthesis
~~~

代码负责：

~~~text
并发
检索
抓取
去重
过滤
Fallback
成本
~~~

### 第三件：数据如何变化

~~~text
Query
→ Search Results
→ Sub Queries
→ URLs / Full Text
→ Pages
→ Filtered Context
→ Report
~~~

### 第四件：主要工程约束

~~~text
Quality
Cost
Latency
Reliability
Security
~~~

这样讲比从目录树开始更容易让面试官形成心智模型。

---

## 11.4 最值得准备的 12 个问题

### Q1：GPT Researcher 普通模式是 ReAct 吗？

答题核心：

> 不是经典开放式 ReAct while-loop，更像 domain-specific agentic workflow。控制路径由代码显式规定，LLM 负责高不确定性节点。

追问时可以画：

~~~text
Role
→ Initial Search
→ Plan
→ Parallel Research
→ Context
→ Writer
~~~

### Q2：为什么 Planning 前先做 Initial Search？

答题核心：

> Grounded Planning。避免 Planner 只依赖参数知识，对近期信息、版本变化和陌生实体规划错误。

### Q3：为什么还要把 original query 加回 sub-query？

答题核心：

> 提升 recall。Decomposition 可能漏掉整体性或 overview 来源，原始 Query 是低成本补充。

### Q4：Retriever 和 Scraper 为什么分开？

答题核心：

~~~text
Retriever = candidate discovery
Scraper   = content acquisition
~~~

Search snippet 不能等价为完整网页。

### Q5：`requires_scraping` 解决什么问题？

答题核心：

> 它把“raw_content 到底是 Preview 还是 Full Content”变成显式 Provider 契约，避免用字符串长度猜语义。

### Q6：GPT Researcher 是标准向量 RAG 吗？

答题核心：

> 不是。Web Research 是 Search → Scrape → Context Selection。Context Filter 可以是 keyword、Jev、embeddings 或 none；Embedding 不是必需路径。

### Q7：为什么 keyword 是一个合理 fallback？

答题核心：

> 本地、低成本、依赖少、对实体/API 名有效，是高 availability baseline；并且应该通过 Eval 而不是技术偏好判断质量。

### Q8：Deep Research 的 breadth / depth / concurrency 分别是什么？

~~~text
breadth     = 每层展开多少方向
depth       = 允许继续下钻多少层
concurrency = 同层最多同时执行多少 branch
~~~

重点补一句：

> 三者都影响成本和延迟，不应该只看“研究越多越好”。

### Q9：Deep 和 Detailed 有什么区别？

~~~text
Deep
→ Evidence Exploration

Detailed
→ Long-form Report Coordination
~~~

Detailed 用 previous written contents 降低章节重复，Deep 用 follow-up questions 继续探索。

### Q10：Detailed 为什么默认顺序写章节？

答题核心：

> 后面的章节需要看到前面已经写过的 headers 和相关内容。顺序执行牺牲 latency，换取 cross-section consistency。

### Q11：Agent 如何处理模型输出不稳定？

答题核心：

~~~text
json_repair
normalization
schema guard
fallback
default path
~~~

不要只说“Prompt 要求 JSON”。

### Q12：怎么证明你做的 Agent 优化有效？

一定要回答：

~~~text
Baseline
+
固定 Dataset
+
Metrics
+
Ablation
+
Quality/Cost/Latency 对比
~~~

如果回答只剩：

> “我感觉效果更好了。”

项目可信度会明显下降。

---

## 11.5 源码追问题怎么回答

面试官可能直接问：

> “你说 Planning 在这里，具体函数是什么？”

不要慌着背行号。用“路径 + 函数 + 作用”回答。

例如：

~~~text
gpt_researcher/skills/researcher.py
ResearchConductor._get_context_by_web_search

这里先调用 _get_initial_search_results，
再调用 plan_research。
plan_research 最后进入
actions/query_processing.py::generate_sub_queries。
~~~

如果继续问：

> “Sub Query 怎么并发？”

回答：

~~~text
同一个 _get_context_by_web_search 里
对 _process_sub_query(...) 使用 asyncio.gather。
~~~

如果再问：

> “每个 Worker 里面做什么？”

继续：

~~~text
_process_sub_query
→ _scrape_data_by_urls
→ ContextManager.get_similar_content_by_query
→ _combine_mcp_and_web_context
~~~

最好的准备方式不是背答案，而是在 IDE 里练习：给自己 30 秒，从一个函数跳到下一个函数。

做到：

> 面试官说一个阶段，你知道去哪一个文件找。

---

## 11.6 二次开发项目怎么讲

建议按“问题—证据—设计—实现—结果”讲。

### Problem

例如：

> “固定 breadth/depth 会让简单问题也产生不必要的递归研究，而且 Nested Researcher 缺少统一 request budget。”

### Evidence

先做 Baseline，记录：

~~~text
平均 Search Calls
P95 Latency
Average Cost
Citation Coverage
~~~

例如发现某些简单 Query 在 depth=2 时成本显著增加，但新增有效 Evidence 很少。

### Design

引入：

~~~text
Typed Evidence
Shared ResearchBudget
Coverage Evaluator
Priority Frontier
~~~

### Implementation

具体落到：

~~~text
ResearchState
DeepResearch scheduler
Nested GPTResearcher creation
Cost callback
Trace
~~~

### Result

必须给真实数据，例如：

~~~text
成本下降 X%
P95 延迟下降 Y%
Citation Coverage 保持 / 提升 Z%
~~~

如果尚未真的跑实验，就不要虚构数字。

---

## 11.7 简历应该写什么、不应该写什么

### 当前只读过源码时

可以写成 Open-source Study / Source Analysis，例如：

- 深入分析 GPT Researcher Research Workflow；
- 梳理 Planning / Retrieval / Context / Deep Research；
- 编写源码学习文档。

但不要写：

> “独立设计并实现 GPT Researcher。”

这不符合事实。

### 真正做完二次开发后

重点切到：

~~~text
你新增的问题解决方案
+
你负责的核心模块
+
Benchmark 结果
~~~

而不是继续把上游原有能力写成你的成果。

### 一个简单判断标准

每个简历 Bullet 都问：

> “面试官让我打开 GitHub 指给他看，我能指出哪些 Commit 是我写的吗？”

如果答案是否定的，就应该收敛表述。

---

## 11.8 可使用的简历 Bullet 模板

下面是“完成对应工作以后”才可以使用的模板。

### 模板 A：Agent Workflow

~~~text
基于 GPT Researcher 二次开发自适应 Deep Research Agent，将固定 breadth/depth 的递归策略改造成 Coverage-driven Research Frontier，根据证据覆盖度与剩余预算动态生成后续研究任务，降低低价值搜索分支。
~~~

### 模板 B：Budget / Reliability

~~~text
设计请求级 Shared ResearchBudget，在主/子 Researcher 间统一约束 LLM、Search、URL、Deadline 与成本预算，并实现 MCP deep→fast、降低 breadth 等渐进式降级策略，避免递归 Research 出现成本与并发失控。
~~~

### 模板 C：Evidence / Citation

~~~text
重构 Research Context 为带 provenance 的 Typed Evidence 数据模型，保留 Query、Source URL、Provider、Relevance 与 Evidence ID；实现 Claim–Evidence Verification 流程，降低最终报告 unsupported claims。
~~~

### 模板 D：Eval

~~~text
搭建 Research Agent 离线评测集与 Benchmark Pipeline，从 Query Coverage、Evidence Recall、Citation Coverage、Unsupported Claim Rate、Latency、Cost 等维度对 Basic / Fixed Deep / Adaptive Deep 进行消融实验。
~~~

一定要替换成真实数据。没有数据时就不要写“提升 40%”。

---

## 11.9 一周复习计划

### Day 1：只画主链路

不看文档，画：

~~~text
Query
→ GPTResearcher
→ ResearchConductor
→ Plan
→ Retrieve
→ Scrape
→ Context
→ Writer
~~~

然后打开源码核对。

### Day 2：Planning + Retrieval

必须能解释：

~~~text
Initial Search
generate_sub_queries
normalize
requires_scraping
visited_urls
MCP fast/deep
~~~

### Day 3：Context + Writer

解释：

~~~text
auto
keyword
Jev
embeddings
fallback
abstain
~~~

### Day 4：Deep + Detailed

手画两个流程，对比：

~~~text
递归研究
vs
顺序章节写作
~~~

### Day 5：Multi-Agent + Engineering

重点：

~~~text
LangGraph
bounded revision loop
concurrency
retry
cost
trace
security
~~~

### Day 6：你的二次开发

把自己的：

~~~text
Problem
Design
Code
Tests
Eval
Result
~~~

讲 3 遍。

### Day 7：模拟追问

要求自己不使用：

~~~text
“差不多”
“应该是”
“我记得”
~~~

遇到记不清的实现，要能快速说出：

> “这个逻辑我会从哪个文件/函数定位。”

---

## 11.10 最终自测

如果下面问题有 3 个以上回答不清楚，建议不要急着面试。

- [ ] 能在 60 秒解释 GPT Researcher 吗？
- [ ] 能在白板画普通 Research 主链路吗？
- [ ] 能解释为什么它不是经典 ReAct 吗？
- [ ] 能说出 `GPTResearcher` 和 `ResearchConductor` 的职责差异吗？
- [ ] 能解释 Initial Search 的价值吗？
- [ ] 能解释 Retriever / Scraper / Context 的边界吗？
- [ ] 能解释 `requires_scraping` 吗？
- [ ] 能解释 Context Filter 的 fallback 吗？
- [ ] 能解释 Deep 和 Detailed 的差异吗？
- [ ] 能解释 Multi-Agent 为什么仍复用 GPTResearcher 吗？
- [ ] 能指出系统至少 5 个失败点吗？
- [ ] 能解释 Structured Output 为什么要 Normalize 吗？
- [ ] 能设计一个 Agent Eval 吗？
- [ ] 能指出至少两个架构技术债吗？
- [ ] 能明确说出自己的二次开发改了哪些文件和逻辑吗？
- [ ] 能展示真实 Benchmark，而不是只有 Demo 吗？

---

## 11.11 本章总结

~~~text
源码项目能不能写进简历
不取决于：
“你看了多少文件”

而取决于：

理解主链路
   +
能落到源码
   +
能分析 Trade-off
   +
做了自己的核心改造
   +
有 Tests
   +
有 Eval
   +
有可复现数据
~~~

如果你现在只是完成了源码学习，这已经是第一阶段；下一步应该优先按照第 10 章完成一个真正属于自己的改造闭环。

源码版本与文档参考范围见：

➡️ [99 - Source Snapshot](../99-source-snapshot/README.md)
