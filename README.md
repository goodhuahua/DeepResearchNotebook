# DeepResearchNotebook

> 面向 **AI Agent / LLM 应用开发岗位** 的 GPT Researcher 源码学习笔记。  
> 目标不是“会调用一个 Deep Research API”，而是把一个真实开源 Research Agent 的核心链路拆开：**任务理解 → 角色生成 → 研究规划 → 多路检索 → 网页抓取 → Context Engineering → 报告生成 → Deep Research → 子 Agent → MCP → 并发/容错/成本/可观测性**。

> [!IMPORTANT]
> 本仓库是个人学习与源码分析笔记，不是 GPT Researcher 官方文档。分析基于 `assafelovic/gpt-researcher` 的 `main` 分支提交 **`0957c301ed06c2a5857b834358c7227c739041d4`（2026-09-26）**。上游持续演进，阅读时请优先结合对应源码快照。

## 为什么做这份笔记

如果目标是求职 Agent 开发岗位，只把 GPT Researcher 写进简历远远不够。面试真正容易被追问的是：

- 它到底是不是 ReAct Agent？它的“Agent Loop”在哪里？
- 一条用户 Query 是怎样被拆成 Sub-Queries 的？
- 搜索结果为什么不能直接塞进 Prompt？
- Retriever、Scraper、Context Filter、Writer 各自负责什么？
- Web / Local / Hybrid / Vector Store / MCP 五类来源怎么统一？
- 为什么项目要区分 `FAST_LLM`、`SMART_LLM`、`STRATEGIC_LLM`？
- Deep Research 与普通 Research 有什么本质差异？
- Detailed Report 为什么会创建子 Researcher？如何避免章节重复？
- 并发在哪里？哪些地方会有 race condition？项目怎么处理？
- LLM 输出 JSON 不稳定时怎么恢复？
- 搜索、抓取、Embedding、LLM 任一环节失败时怎么降级？
- 这个项目有哪些设计值得复用，又有哪些地方你会重构？

这份仓库围绕这些问题组织，而不是围绕“怎么安装、怎么点 UI”组织。

## 一句话理解 GPT Researcher

**GPT Researcher 不是一个通用的“LLM → Tool → Observation → 再调用 LLM”的无限 ReAct 循环，而是一个面向 Research 任务的、强流程约束的 Agent Workflow。**

它把研究行为拆成稳定流水线：

```text
User Query
    │
    ▼
GPTResearcher                 ← Facade / Orchestrator
    │
    ├── choose_agent()        ← 根据任务生成研究角色
    │
    ▼
ResearchConductor
    │
    ├── Initial Search
    ├── Plan Research
    │     └── Strategic LLM → Sub Queries
    │
    ├── asyncio.gather(Sub Queries)
    │      ├── Retriever(s)
    │      ├── MCP Retriever
    │      ├── URL Dedup
    │      ├── Browser / Scraper
    │      └── Context Selection / Compression
    │
    ├── Aggregate Context
    └── Optional Source Curation
           │
           ▼
ReportGenerator
    │
    ├── Prompt by Report Type
    ├── Smart LLM
    └── Markdown Report
```

而 `DeepResearchSkill` 会在这套常规 Research Pipeline 之上再加一层**递归式探索**：

```text
Query
  └── breadth 个 Search Queries
       ├── Nested GPTResearcher
       │    └── Learnings + Follow-up Questions
       ├── Nested GPTResearcher
       │    └── Learnings + Follow-up Questions
       └── ...
              │
              └── depth > 1 → 继续向下探索
```

这也是本项目最值得学习的地方：**Agent 不一定必须是一条开放式 ReAct Loop；对于垂直任务，显式 Workflow + 有限自治往往更稳定、更便宜、更容易测试。**

## 源码全景

核心代码位于：

```text
gpt_researcher/
├── agent.py                    # ★ GPTResearcher：总编排器 / Facade
├── prompts.py                  # ★ Prompt Family
├── actions/
│   ├── agent_creator.py        # ★ 自动角色生成
│   ├── query_processing.py     # ★ 搜索 + Sub-query 规划
│   ├── retriever.py            # ★ Retriever Factory
│   ├── web_scraping.py         # 抓取动作
│   ├── report_generation.py    # ★ 最终报告 LLM 调用
│   └── markdown_processing.py
├── skills/
│   ├── researcher.py           # ★ ResearchConductor：普通研究主链路
│   ├── context_manager.py      # ★ Context 入口
│   ├── browser.py              # ★ 抓取编排
│   ├── writer.py               # ★ ReportGenerator
│   ├── deep_research.py        # ★ 递归 Deep Research
│   ├── curator.py              # Source Curation
│   └── image_generator.py
├── context/
│   ├── select.py               # ★ Context Filter 策略路由
│   ├── lexical.py              # BM25/关键词路径
│   ├── jev_filter.py           # Jev 路径
│   └── compression.py          # Embedding/VectorStore 压缩
├── retrievers/                 # Tavily / Google / Bing / Exa / MCP / 学术搜索...
├── scraper/                    # BeautifulSoup / Browser / PDF 等抓取实现
├── llm_provider/               # 多 LLM Provider 适配
├── memory/                     # Embedding 抽象
├── vector_store/               # Vector Store Wrapper
├── mcp/                        # MCP 相关实现
└── config/                     # 配置系统

backend/
├── report_type/
│   ├── basic_report/           # conduct_research → write_report
│   └── detailed_report/        # Main Research + Subtopic Researchers
└── server/                     # FastAPI / WebSocket / Streaming

multi_agents/                   # 另一套 LangGraph / AG2 多 Agent 路径
```

## 学习路线

| 阶段 | 章节 | 你应该解决的问题 |
|---|---|---|
| Phase 0 | [阅读指南](docs/00-reading-guide.md) | 怎么读一个真实 Agent 项目，而不是迷失在目录里 |
| Phase 1 | [01 架构总览](docs/01-architecture-overview.md) | 系统有哪些层？核心对象怎么协作？ |
| Phase 1 | [02 主链路](docs/02-main-agent-lifecycle.md) | 一次 Research 从请求到 Report 完整经过什么？ |
| Phase 1 | [03 Planning](docs/03-planning-and-query-decomposition.md) | Agent 如何选角色、拆 Query、调用 Strategic LLM？ |
| Phase 2 | [04 Retrieval 与 MCP](docs/04-retrieval-and-mcp.md) | 多 Retriever 如何统一？MCP 为什么有 fast/deep/disabled？ |
| Phase 2 | [05 Browser 与 Scraping](docs/05-browsing-and-scraping.md) | Search Result 如何变成可信正文？如何去重与并发抓取？ |
| Phase 2 | [06 Context Engineering](docs/06-context-engineering.md) | keyword/Jev/embedding/none 怎么选？为什么要压缩？ |
| Phase 2 | [07 Report Generation](docs/07-report-generation.md) | Context 怎样变成带结构的最终报告？ |
| Phase 3 | [08 Deep Research](docs/08-deep-research.md) | breadth/depth/递归/并发/Follow-up Question 如何工作？ |
| Phase 3 | [09 Detailed Report 与子 Agent](docs/09-detailed-report-and-subagents.md) | 为什么要给每个子主题创建新 Researcher？ |
| Phase 3 | [10 配置与 Provider](docs/10-config-and-provider.md) | 多模型、多 Retriever、插件扩展是怎样解耦的？ |
| Phase 3 | [11 Backend 与可观测性](docs/11-backend-streaming-observability.md) | API、WebSocket、Streaming、日志与成本怎样串起来？ |
| Phase 4 | [12 并发、容错与工程化](docs/12-reliability-concurrency-cost.md) | 这套 Agent 如何避免“一处失败，全链路崩掉”？ |
| Phase 4 | [13 源码走读地图](docs/13-source-code-walkthrough.md) | 面试前如何从关键函数反向复述源码？ |
| Phase 4 | [14 架构评价与对比](docs/14-design-analysis.md) | 与 Claude Code / Nanobot / ReAct 的异同是什么？ |
| Phase 5 | [15 面试与简历](docs/15-interview-and-resume.md) | 怎么把“读过源码”转化成可验证的项目能力？ |
| Phase 5 | [16 二次开发路线](docs/16-hands-on-roadmap.md) | 如何把学习项目升级成真正属于自己的项目？ |

## 建议的阅读顺序

**只有 30 分钟**：README → 01 → 02 → 06 → 14。

**准备 Agent 面试**：01 → 02 → 03 → 04 → 06 → 08 → 09 → 12 → 15。

**准备二次开发**：02 → 04 → 05 → 06 → 10 → 11 → 12 → 16。

**想从源码验证每个结论**：直接打开 [13 源码走读地图](docs/13-source-code-walkthrough.md)，按“入口函数 → 子函数 → 数据结构”的顺序跳转。

## 这份笔记采用的分析方法

参考：

- [how-claude-code-works](https://github.com/Windy3f3f3f3f/how-claude-code-works)：按“主循环、上下文工程、工具系统、多 Agent、可观测性”等**机制**拆章，强调源码定位、调用链和“为什么这样设计”。
- [learn-nanobot](https://github.com/bcefghj/learn-nanobot)：按**求职学习路线**组织，加入源码走读、架构图、面试问答和简历表达。

但不会复制两者内容。对 GPT Researcher 的结论直接来自上游源码，并尽量遵循：

1. **先画数据流，再讲类。**
2. **先讲 Why，再讲 How。**
3. **关键结论给出源码文件与函数。**
4. **明确区分普通 Research、Deep Research、Detailed Report、Multi-Agent 四条路径。**
5. **不仅描述 happy path，也看异常处理、重试、fallback、并发和成本。**
6. **指出实现中的 trade-off，而不是把所有代码都解释成“优秀设计”。**

## 读源码时最重要的五个结论

### 1. `GPTResearcher` 不是核心算法本体，而是 Facade

`gpt_researcher/agent.py` 主要负责状态、依赖和生命周期编排。真正普通研究逻辑在 `skills/researcher.py::ResearchConductor`，写作在 `skills/writer.py::ReportGenerator`。

### 2. Planning 不是“凭空拆问题”

当前主链路会先拿一次初始 Search Result，再让 Strategic LLM 基于 Query + 搜索上下文生成 Sub-Queries。这样规划阶段拥有最低限度的外部世界信息，而不是只依赖模型参数记忆。

### 3. Search 和 Context 是两层问题

Retriever 解决“**去哪找**”，Scraper 解决“**把页面正文拿回来**”，Context Manager 解决“**哪些内容值得送进 LLM**”。把三者混成一个 RAG 步骤，会错过项目最关键的工程分层。

### 4. Context Filter 是当前版本的重要工程优化

当前支持：

```text
auto
├── TYPESAFE_API_KEY 存在 → jev
└── 否则                 → keyword

jev         → 外部 relevance scoring
keyword     → 本地 BM25 / lexical ranking
embeddings  → chunk + embedding similarity
none        → 不过滤
```

失败会尽量降级到 keyword，而不是让研究任务直接失败。

### 5. Deep Research 是“递归 Researcher”，不是单次 Prompt 加长

每个探索 Query 都会创建一个新的 `GPTResearcher` 执行正常研究，再从结果中抽取 learnings / follow-up questions，并在剩余 depth 内继续下钻。它把“深度”显式映射成搜索树，而不是简单把 `MAX_ITERATIONS` 调大。

## 默认配置快照

上游当前默认值（以本仓库分析提交为准）：

| 配置 | 默认值 | 作用 |
|---|---:|---|
| `RETRIEVER` | `tavily` | 默认搜索 Provider |
| `FAST_LLM` | `openai:gpt-5.4-mini` | 轻量模型 |
| `SMART_LLM` | `openai:gpt-5.4` | 写作/高质量任务 |
| `STRATEGIC_LLM` | `openai:gpt-5.4` | Planning / Reasoning |
| `MAX_ITERATIONS` | `3` | 普通 Research 的 Query 规划数量相关上限 |
| `MAX_SEARCH_RESULTS_PER_QUERY` | `5` | 每 Query 搜索结果 |
| `CONTEXT_FILTER` | `auto` | Context 路由 |
| `MAX_SCRAPER_WORKERS` | `15` | 抓取 Worker 数 |
| `DEEP_RESEARCH_BREADTH` | `3` | Deep Research 每层宽度 |
| `DEEP_RESEARCH_DEPTH` | `2` | Deep Research 深度 |
| `DEEP_RESEARCH_CONCURRENCY` | `4` | Deep Research 并发 |
| `MCP_STRATEGY` | `fast` | MCP 执行策略 |
| `TOTAL_WORDS` | `1200` | 报告目标长度 |

> 注意：上游更新频繁，默认模型和参数尤其容易变化。面试中不要死背数值，要能解释它们分别控制哪一层。

## 求职导向：最终要能讲清楚什么

学完这份笔记，你至少应该能不看代码回答：

> “GPT Researcher 采用一个面向 research domain 的显式 workflow。入口 `GPTResearcher` 负责 orchestration，常规研究由 `ResearchConductor` 执行。它先基于初始搜索结果使用 strategic model 做 query decomposition，再并发执行 sub-query；每个 sub-query 经过多 Retriever、URL 去重、抓取和 context filtering，最后聚合为研究上下文。Writer 再根据 report type 构建 Prompt，由 smart model 生成报告。Deep Research 在这条链路上递归创建 nested researcher，用 breadth/depth 控制搜索树。工程上它通过 asyncio 并发、MCP cache/lock、visited URL 去重、LLM retry、JSON repair、context fallback 和 step cost accounting 提升可靠性与成本可控性。”

如果这段话你能继续向下追问 20 分钟仍然说得清楚，这个项目才真正适合写到简历里。

## 上游源码锚点

分析快照：`0957c301ed06c2a5857b834358c7227c739041d4`

- [GPTResearcher](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/agent.py)
- [ResearchConductor](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/skills/researcher.py)
- [ContextManager](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/skills/context_manager.py)
- [Context selection](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/context/select.py)
- [ReportGenerator](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/skills/writer.py)
- [DeepResearchSkill](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/skills/deep_research.py)
- [DetailedReport](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/report_type/detailed_report/detailed_report.py)
- [Config defaults](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/config/variables/default.py)

## 声明

本仓库用于学习、源码阅读和求职准备。GPT Researcher 及参考项目的版权和许可归各自作者与项目所有；本文档中的源码链接用于定位与讨论，不将上游源码复制为本仓库实现。
