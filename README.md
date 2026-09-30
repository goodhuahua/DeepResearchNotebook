# DeepResearchNotebook

<p align="center">
  <strong>面向 Agent 开发求职的 GPT Researcher 源码学习指南</strong>
</p>

> 🎯 **这份仓库写给谁？**  
> 写给“知道 LLM / RAG 是什么，但第一次读完整 Agent 项目”的开发者。你不需要提前熟悉 GPT Researcher，也不需要先懂 LangGraph、MCP 或复杂的 Agent 框架。

---

## 为什么重写这份笔记

第一次版本的问题很明显：**知道项目的人看得懂，不知道项目的人看不进去。**

所以这一版不再从“Facade、Provider、Context Filter”这些架构名词直接开讲，而是采用一条更适合学习源码的路线：

```text
先知道它解决什么问题
        ↓
先跑通一次完整研究
        ↓
知道每一步的数据长什么样
        ↓
再看整体架构
        ↓
再逐文件进入源码
        ↓
最后讨论并发、容错、成本、评测和二次开发
```

文档写法参考了 [learn-nanobot](https://github.com/bcefghj/learn-nanobot) 的教学方法：**目标明确、渐进披露、图示优先、关键源码逐段解释、每节最后总结设计价值和面试要点**。内容本身全部围绕 GPT Researcher 源码重新分析。

---

## 先用一句话理解 GPT Researcher

假设用户问：

> **“帮我研究 AI Coding Agent 的技术路线、主要产品和未来趋势，并给出有来源的报告。”**

普通聊天模型会倾向于直接生成答案。

GPT Researcher 的思路是：

```text
1. 先理解任务
2. 决定应该以什么“研究员角色”处理
3. 先搜一轮，了解当前世界里有哪些信息
4. 把大问题拆成多个小问题
5. 并行搜索这些小问题
6. 打开网页、读取正文
7. 从大量正文里筛选真正相关的证据
8. 汇总证据
9. 最后让 Writer 基于证据写报告
```

也就是说：

> **它不是“让 LLM 多想几步”，而是把研究过程工程化。**

---

## 一张图看懂普通 Research

```text
用户问题
  │
  ▼
GPTResearcher
  │
  ├─ 生成研究角色
  │
  ▼
ResearchConductor
  │
  ├─ Initial Search
  │      └─ 先获取少量外部信息
  │
  ├─ Planning
  │      └─ Strategic LLM 生成 Sub Queries
  │
  ├─ 并行研究多个 Sub Query
  │      ├─ Retriever 找 URL / 全文
  │      ├─ Browser/Scraper 读取网页
  │      └─ ContextManager 过滤证据
  │
  └─ 汇总 Context
         │
         ▼
ReportGenerator
  │
  └─ Smart LLM 基于证据写最终报告
```

这条链路是整个仓库的主线。

---

## 为什么它值得 Agent 岗位学习

这个项目同时覆盖了 Agent 开发中非常典型的工程问题：

| 能力 | GPT Researcher 中的实现 |
|---|---|
| Planning | Query Decomposition |
| Tool / 外部世界 | Search、Scrape、MCP |
| RAG / Context Engineering | keyword / Jev / embeddings |
| 并发 | `asyncio.gather`、Semaphore、WorkerPool |
| 子 Agent | Detailed Report / Deep Research |
| 多 Agent | LangGraph Chief Editor workflow |
| Provider 抽象 | 多 LLM、多 Retriever |
| 结构化输出 | JSON repair / normalization |
| 容错 | retry、fallback、partial failure |
| 成本 | step cost tracking |
| 可观测性 | WebSocket events、logs |
| Evaluation | context filter eval、quality eval |

因此它适合用来学习“**Agent 怎么从 Demo 走向工程系统**”。

---

## 学习路线

### Phase 1：先建立直觉

| 章节 | 你会解决什么问题 | 建议时间 |
|---|---|---:|
| [01 - 什么是 Research Agent](docs/01-what-is-research-agent/README.md) | Research Agent 和普通 LLM / RAG / ReAct 有什么区别？ | 1h |
| [02 - GPT Researcher 项目概览](docs/02-gpt-researcher-overview/README.md) | 项目有哪些模式？目录怎么看？先读哪些文件？ | 1h |

### Phase 2：真正理解主链路

| 章节 | 你会解决什么问题 | 建议时间 |
|---|---|---:|
| [03 - 架构深入解析](docs/03-architecture-deep-dive/README.md) | 核心对象如何协作？数据是怎么流动的？ | 2h |
| [04 - 源码逐行走读](docs/04-source-code-walkthrough/README.md) | 从入口一路追到 Search、Scrape、Context、Writer | 4h |
| [05 - Planning、Retrieval 与 MCP](docs/05-planning-retrieval-mcp/README.md) | Query 怎么拆？Retriever 怎么统一？MCP 怎么接入？ | 2h |
| [06 - Scraping、Context 与报告生成](docs/06-context-and-writing/README.md) | 搜索结果怎么变成证据？证据怎么变成报告？ | 2h |

### Phase 3：高级 Agent 工作流

| 章节 | 你会解决什么问题 | 建议时间 |
|---|---|---:|
| [07 - Deep / Detailed / Multi-Agent](docs/07-advanced-workflows/README.md) | 三种高级模式到底有什么区别？ | 2.5h |
| [08 - 工程化：并发、容错、成本、可观测性](docs/08-engineering/README.md) | 为什么一个真实 Agent 远不止 Prompt？ | 2h |
| [09 - Tests 与 Evals](docs/09-tests-and-evals/README.md) | 怎么从测试理解历史 Bug？怎么评测 Agent？ | 1.5h |

### Phase 4：从“读源码”变成“自己的项目”

| 章节 | 你会解决什么问题 | 建议时间 |
|---|---|---:|
| [10 - 动手改造路线](docs/10-hands-on-roadmap/README.md) | 怎么调试、改造、做 Benchmark？ | 3h+ |
| [11 - 面试与简历](docs/11-interview-and-resume/README.md) | 面试怎么讲？哪些内容能真实写进简历？ | 1.5h |

源码版本见 [99 - Source Snapshot](docs/99-source-snapshot/README.md)。

---

## 全书贯穿的示例

为了避免每一章换一个问题，我们会一直使用这个示例：

> **研究问题：比较 LangGraph、AutoGen、CrewAI 在 Agent 编排、状态管理、可观测性方面的设计差异，并给出适用场景。**

你会一路看到它如何变成：

```text
原始 Query
↓
研究角色
↓
Initial Search Results
↓
Sub Queries
↓
Search Results
↓
URLs / raw_content
↓
Scraped Pages
↓
Filtered Context
↓
Final Report
```

这能帮助你真正理解“数据怎么流”，而不只是记类名。

---

## 你不需要一开始就懂这些词

第一次读时，下面这些词只需要有模糊印象：

- Retriever：负责“找候选资料”
- Scraper：负责“真正读取网页正文”
- Context：最终准备给 LLM 的研究证据
- Provider：OpenAI、Anthropic 等模型供应商适配层
- MCP：一种连接外部工具/数据源的标准协议
- Sub Query：从大研究问题拆出来的小研究问题

后面的章节会第一次出现时重新解释。

---

## 四种运行模式先记住名字

GPT Researcher 仓库里容易混淆的是四条路径：

```text
Basic Research
  → 普通研究主链路

Detailed Report
  → 按章节创建子 Researcher，适合长报告

Deep Research
  → 根据研究结果继续生成 Follow-up Questions，递归下钻

Multi-Agent
  → LangGraph 编排 Editor / Researcher / Writer / Fact Checker 等角色
```

**先学 Basic，再学另外三种。**

如果一开始就从 Multi-Agent 或 Deep Research 读，很容易迷失。

---

## 源码阅读原则

本仓库不要求你“把全部代码逐行背下来”。

真正应该掌握的是：

```text
这个函数为什么存在？
输入是什么？
输出是什么？
它修改了哪些状态？
下一步调用谁？
哪里可能失败？
失败后怎么处理？
为什么不用更简单的实现？
```

源码学习的目标不是“看过”，而是**能解释设计决策**。

---

## 最终你应该能回答

完成前 9 章后，你应该可以不看源码解释：

1. GPT Researcher 为什么不是一个经典的 ReAct while-loop？
2. `GPTResearcher` 和 `ResearchConductor` 分别负责什么？
3. 为什么 Planning 之前还要做一次 Initial Search？
4. Retriever 和 Scraper 为什么必须分开？
5. 搜到 20 个网页以后，为什么不能全部塞给 LLM？
6. keyword / Jev / embeddings Context Filter 有什么区别？
7. Deep Research 的 breadth / depth / concurrency 分别控制什么？
8. Detailed Report 为什么默认顺序生成子章节？
9. MCP fast / deep 为什么是成本与覆盖率的权衡？
10. 一次研究失败可能发生在哪些层？
11. 怎么评价一次优化到底是真的变好，还是只是“感觉更好”？

如果这些问题能追问 20 分钟，你才真正适合把这个项目放到 Agent 岗位简历里。

---

## 版本说明

本仓库分析基于 GPT Researcher：

- Repository: [assafelovic/gpt-researcher](https://github.com/assafelovic/gpt-researcher)
- Commit: `0957c301ed06c2a5857b834358c7227c739041d4`
- Date: 2026-09-26

> 上游变化很快，所以文档中涉及实现细节时，以固定提交为准，而不是假设未来 `main` 永远相同。

---

## 声明

本仓库是个人学习、源码分析和求职准备材料，不是 GPT Researcher 官方文档。源码版权和许可证归上游项目所有；文档中的代码片段以解释调用链为目的，并尽量使用裁剪后的关键逻辑或伪代码。
