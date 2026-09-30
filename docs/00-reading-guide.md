# 00 - 阅读指南：怎样读懂一个真实 Research Agent

> 本章不是介绍 GPT Researcher 的功能，而是给出一条**源码阅读路线**。目标是避免一上来陷进 Retriever、Prompt 或前端细节，最后只记住一堆类名，却说不清 Agent 的控制流。

## 0.1 先建立正确的问题框架

读 Agent 项目时，建议始终追踪 6 个问题：

| 问题 | GPT Researcher 中对应位置 |
|---|---|
| 谁拥有全局状态？ | `GPTResearcher` |
| 谁决定下一步做什么？ | 普通模式：显式 Workflow + Planning LLM；Deep 模式：递归探索 |
| 外部信息怎样进入系统？ | Retriever → Scraper → ContextManager |
| LLM 在哪些节点被调用？ | choose_agent、query decomposition、context/curation、writer、deep research |
| 哪些地方有循环或递归？ | Sub-query fan-out、Detailed Report 子主题、Deep Research recursion |
| 怎么停止、降级和恢复？ | 固定 workflow 结束、depth/breadth、fallback、retry、empty-context abstain |

如果这 6 个问题还没回答，就不建议先背 Prompt。

## 0.2 三层阅读法

### 第一层：只看控制流

先只读下面 6 个文件：

```text
backend/report_type/basic_report/basic_report.py
gpt_researcher/agent.py
gpt_researcher/skills/researcher.py
gpt_researcher/actions/query_processing.py
gpt_researcher/skills/context_manager.py
gpt_researcher/skills/writer.py
```

把它压缩成一句话：

> BasicReport 创建 GPTResearcher；GPTResearcher 选择角色并委托 ResearchConductor；ResearchConductor 规划 Sub-Queries、检索和构造 Context；ReportGenerator 再把 Context 写成 Report。

此时先不要深入具体搜索引擎、LangChain 类、前端。

### 第二层：追踪数据形态

真正理解 Agent，需要知道数据经过每层时长什么样：

```text
query: str

↓ search

search_results:
[
  {"title": "...", "href": "...", "body": "..."},
  ...
]

↓ scrape / prefetched full content

pages:
[
  {"url": "...", "raw_content": "...", ...},
  ...
]

↓ context selection

sub_context: str

↓ aggregate

researcher.context: str | list

↓ report prompt

report: str (Markdown)
```

很多 Agent bug 本质不是“模型不聪明”，而是**层与层之间的数据契约不稳定**。GPT Researcher 代码里大量 guard 都是在修复这一类问题：字段缺失、非 dict、LLM JSON 结构漂移、空响应、重复 URL 等。

### 第三层：研究工程化细节

最后再读：

- `context/select.py`：context filter 路由与 fallback
- `utils/llm.py`：LLM retry、provider 参数、cost callback
- `skills/deep_research.py`：递归探索
- `backend/report_type/detailed_report/*`：子 Researcher
- `actions/retriever.py`：Retriever 插件化
- `config/config.py`：配置优先级
- tests：看真实项目曾经踩过什么坑

## 0.3 为什么不要把它简单叫 ReAct Agent

经典 ReAct：

```text
LLM Thought
  → Tool Call
  → Observation
  → LLM Thought
  → Tool Call
  → ...
  → Final Answer
```

GPT Researcher 普通模式更接近：

```text
Role Selection
  → Initial Search
  → LLM Planning
  → Parallel Deterministic Research Steps
  → Context Aggregation
  → LLM Writing
```

LLM 有“决策权”的节点是有限的，而不是每一步都自由选择工具。

这是一种重要的 Agent 设计思路：

> **把高熵决策交给模型，把可确定的执行流程交给代码。**

对于 research 任务，这通常比开放式 tool loop 更可控。

## 0.4 建议亲自做的断点

如果你在本地运行上游项目，建议至少在这些函数打断点：

1. `GPTResearcher.conduct_research`
2. `choose_agent`
3. `ResearchConductor.plan_research`
4. `generate_sub_queries`
5. `ResearchConductor._process_sub_query`
6. `ResearchConductor._search_relevant_source_urls`
7. `BrowserManager.browse_urls`
8. `ContextManager.get_similar_content_by_query`
9. `select_context`
10. `ReportGenerator.write_report`
11. `generate_report`

每个断点只记四件事：

```text
input
state before
output
state after
```

这比“逐行抄注释”有效得多。

## 0.5 一次调试建议打印什么

建议构造一个统一 trace：

```python
{
    "research_id": "...",
    "stage": "planning | retrieval | scrape | context | write",
    "query": "...",
    "sub_query": "...",
    "provider": "...",
    "input_count": 0,
    "output_count": 0,
    "elapsed_ms": 0,
    "cost": 0.0,
}
```

上游已经有 log handler、WebSocket event 和 step cost 的基础，你做二次开发时可以继续往结构化 Trace 演进。

## 0.6 阅读时需要区分四条路径

### Path A：Basic Report

```text
conduct_research()
→ write_report()
```

最适合第一次学习。

### Path B：Detailed Report

```text
initial research
→ generate subtopics
→ child researcher per subtopic
→ avoid duplicated headers/content
→ introduction + TOC + body + conclusion + references
```

这里开始出现“子 Agent/子 Researcher”协作。

### Path C：Deep Research

```text
generate queries
→ nested GPTResearcher
→ extract learnings
→ follow-up questions
→ recurse by depth
```

这里重点是搜索树、递归终止与并发控制。

### Path D：Multi-Agent

`multi_agents/` 是独立的 LangGraph / AG2 路径，不应和 `GPTResearcher` 主类强行混成一套实现。学习时先把 A/B/C 吃透，再读 Multi-Agent。

## 0.7 面试学习法

每读完一个模块，用下面模板复述：

> **职责**：它解决什么问题？  
> **输入/输出**：数据契约是什么？  
> **关键决策**：为什么不直接用另一种实现？  
> **失败模式**：哪里可能失败？  
> **降级**：失败后怎么继续？  
> **可扩展点**：怎样换 Provider / Retriever / Context Filter？  
> **我会怎么改**：如果你维护生产版本，会做什么重构？

如果只能回答“这个类调用那个类”，说明还没有形成工程理解。

## 0.8 本仓库的源码可信度原则

本文档分析优先级：

```text
当前固定提交源码
  > 同提交 tests
  > 同提交官方 README / docs
  > 仓库内部 reference 文档
  > 历史文章 / 二手解读
```

原因很简单：上游代码变化很快，文档可能晚于实现。例如 MCP 配置的并发污染问题已经在当前 `GPTResearcher._process_mcp_configs` 中通过“不写进 os.environ”修复，所以不能机械照搬旧流程图。

## 0.9 读完本仓库后的自测

不看源码回答：

1. 为什么 Planning 前要做 initial search？
2. 为什么 Retriever 和 Scraper 是两层？
3. `requires_scraping` 这个契约解决什么问题？
4. 为什么 Context Filter 默认不一定需要 Embedding？
5. MCP fast 与 deep 的成本差异是什么？
6. `visited_urls` 为什么会被父子 Researcher 共享？
7. Detailed Report 怎么降低章节重复？
8. Deep Research 的 breadth、depth、concurrency 分别控制什么？
9. LLM JSON 不合法时有哪些恢复手段？
10. 为什么空 context 时 Writer 选择 abstain 而不是照样生成？

能解释清楚这 10 题，再继续做自己的改造项目。
