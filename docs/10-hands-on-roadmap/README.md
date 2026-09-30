# 10 - 动手改造路线：把“读过源码”变成自己的 Agent 项目

> 🎯 **本章目标**：给出一条可以真正落地的实战路线。重点不是“Fork 后改几个 Prompt”，而是选择一个工程问题，完成 **Baseline → 设计 → 实现 → Tests → Eval → Benchmark → 文档** 的闭环。

---

## 目录

- [10.1 先明确：什么改造有简历价值](#101-先明确什么改造有简历价值)
- [10.2 第一件事：本地跑通并打断点](#102-第一件事本地跑通并打断点)
- [10.3 建立 Baseline](#103-建立-baseline)
- [10.4 推荐主线：Budget-aware Adaptive Deep Research](#104-推荐主线budget-aware-adaptive-deep-research)
- [10.5 Phase 1：Typed Evidence](#105-phase-1typed-evidence)
- [10.6 Phase 2：Shared Budget](#106-phase-2shared-budget)
- [10.7 Phase 3：Structured Trace](#107-phase-3structured-trace)
- [10.8 Phase 4：Coverage Evaluator 与 Replanning](#108-phase-4coverage-evaluator-与-replanning)
- [10.9 Phase 5：Citation Verification](#109-phase-5citation-verification)
- [10.10 怎么写 Tests](#1010-怎么写-tests)
- [10.11 怎么做 Benchmark](#1011-怎么做-benchmark)
- [10.12 不建议优先做的改造](#1012-不建议优先做的改造)
- [10.13 项目验收标准](#1013-项目验收标准)
- [10.14 本章总结](#1014-本章总结)

---

## 10.1 先明确：什么改造有简历价值

### 价值较低

~~~text
换一个 LLM
改几个 Prompt
加一个简单 Retriever
改 UI 样式
把默认参数调大
~~~

这些工作可以帮助熟悉项目，但很难证明你具备 Agent 系统设计能力。

### 价值更高

~~~text
发现一个真实系统问题
  ↓
定义可测量指标
  ↓
设计机制
  ↓
修改核心链路
  ↓
补测试
  ↓
做 Eval
  ↓
拿出前后对比
~~~

例如：

> “固定 breadth/depth 的 Deep Research 成本波动大，而且 nested researcher 的成本没有统一预算约束。”

这是一个很好的问题。

因为它同时涉及：

- Agent Planning；
- 异步调度；
- Shared State；
- Cost；
- Stop Condition；
- Evaluation。

---

## 10.2 第一件事：本地跑通并打断点

不要马上改代码。

先选一个固定 Query：

~~~text
比较 LangGraph、AutoGen、CrewAI 在
Agent 编排、状态管理、可观测性方面的设计差异，
并给出适用场景。
~~~

至少在这些位置打断点：

~~~text
GPTResearcher.__init__
GPTResearcher.conduct_research
ResearchConductor._get_context_by_web_search
ResearchConductor.plan_research
generate_sub_queries
ResearchConductor._process_sub_query
ResearchConductor._search_relevant_source_urls
ResearchConductor._scrape_data_by_urls
ContextManager.get_similar_content_by_query
select_context
ReportGenerator.write_report
generate_report
~~~

每个断点只记录四件事：

~~~text
Input
State Before
Output
State After
~~~

例如：

~~~text
函数：
_process_sub_query

Input:
sub_query = "LangGraph state management"

State Before:
visited_urls = 12
context = ...

Output:
"Title: ... Content: ... Source: ..."

State After:
visited_urls = 16
research_sources += 4
~~~

这会比抄几百行源码注释更有效。

---

## 10.3 建立 Baseline

如果没有 Baseline，后面任何“优化”都只是主观感觉。

准备 20～50 条 Query。

每次记录：

~~~json
{
  "query": "...",
  "latency_s": 42.3,
  "llm_calls": 8,
  "search_calls": 5,
  "urls_visited": 17,
  "context_chars": 42000,
  "cost_usd": 0.31,
  "source_count": 13
}
~~~

再记录质量指标，例如：

~~~text
Coverage
Citation Precision
Citation Coverage
Unsupported Claims
Source Diversity
~~~

### 为什么 Baseline 要先做？

假设改造后：

~~~text
报告看起来更详细
~~~

但其实：

~~~text
Cost +250%
Latency +180%
Citation Precision -5%
~~~

这不一定是优化。

---

## 10.4 推荐主线：Budget-aware Adaptive Deep Research

我更推荐你做这一条，而不是同时改十个功能。

目标：

~~~text
当前：
固定 breadth + depth
→ 不管问题难不难都按配置展开

改造后：
根据 Coverage / Evidence Gap / Budget
动态决定还要不要继续研究
~~~

最终架构可以是：

~~~text
User Query
   ↓
Initial Plan
   ↓
Research Frontier
   ↓
Worker Pool
   ↓
Evidence Store
   ↓
Coverage Evaluator
   ├─ 信息够了 → Writer
   └─ 有缺口 → Generate Repair Tasks
                    ↓
                 Frontier
~~~

再加一个全局：

~~~text
Shared Budget
~~~

控制：

- 成本；
- LLM Call 数；
- Search Call 数；
- URL 数；
- Deadline；
- Max Depth。

---

## 10.5 Phase 1：Typed Evidence

当前项目里 Context 经常在不同阶段变成：

~~~text
str
list
dict
~~~

还要兼容：

~~~text
href / url
body / content
~~~

第一步可以先不改算法，只把关键证据结构标准化。

例如：

~~~python
class Evidence(BaseModel):
    id: str
    query: str
    source_url: str
    title: str | None
    text: str
    source_provider: str
    relevance_score: float | None = None
    content_hash: str
~~~

### 为什么先做这个？

后面的：

~~~text
Dedup
Budget
Citation
Trace
Eval
~~~

都需要稳定 ID 和 provenance。

如果一开始就把 Evidence 拼成字符串，后面很难追踪：

> 最终这句话到底来自哪个 Chunk？

---

## 10.6 Phase 2：Shared Budget

设计一个请求级预算对象：

~~~python
class ResearchBudget:
    max_cost_usd: float
    max_llm_calls: int
    max_search_calls: int
    max_urls: int
    deadline_ts: float

    async def reserve_llm(...):
        ...

    async def reserve_search(...):
        ...

    def remaining_cost(self) -> float:
        ...
~~~

### 关键点：所有 Child 共享同一个对象

~~~text
Root Researcher
├─ Child A
├─ Child B
└─ Child C

全部：
→ SharedBudget
~~~

这样 Deep Research 才能真正做到：

~~~text
已经快没预算
→ 不再展开低价值 Branch
~~~

而不是：

~~~text
每个 Child 都觉得自己还有完整预算
~~~

---

### Budget 不足时怎么降级？

不要只做：

~~~text
raise BudgetExceeded
~~~

可以设计策略：

~~~text
优先级 1：停止低价值新 Branch
优先级 2：breadth 降低
优先级 3：MCP deep → fast
优先级 4：减少 max search results
优先级 5：Smart Model → cheaper model
优先级 6：停止并输出当前最佳结果
~~~

这才是 Agent 的“资源感知”。

---

## 10.7 Phase 3：Structured Trace

当前项目已经有 logs 和 events。

你可以在核心 Stage 上加：

~~~python
async with trace.span(
    "subquery_research",
    query=sub_query,
):
    ...
~~~

每个 Span 记录：

~~~json
{
  "trace_id": "...",
  "span_id": "...",
  "stage": "context_filter",
  "query": "...",
  "input_count": 17,
  "output_count": 8,
  "latency_ms": 142,
  "cost_usd": 0.0
}
~~~

### 最值得做的 Dashboard 指标

~~~text
P50 / P95 E2E latency
Cost per research
LLM calls per research
Search calls per research
Sources per report
Retry rate
Empty-context rate
Context size before/after filtering
Deep branch count
~~~

---

## 10.8 Phase 4：Coverage Evaluator 与 Replanning

普通模式最大局限之一：

~~~text
只 Plan 一次
→ 执行
→ 直接写
~~~

可以加入：

~~~text
Initial Plan
  ↓
First Research Pass
  ↓
Coverage Evaluator
  ↓
问题的关键维度都覆盖了吗？
  ├─ Yes → Writer
  └─ No  → Repair Queries
             ↓
          Second Pass
~~~

### Evaluator 输入

~~~text
Original Query
Planned Dimensions
Current Evidence Summary
Already Visited Sources
Remaining Budget
~~~

### 输出

~~~json
{
  "covered": [
    "orchestration",
    "state management"
  ],
  "missing": [
    "observability"
  ],
  "repair_queries": [
    "AutoGen observability tracing...",
    "CrewAI tracing telemetry..."
  ]
}
~~~

### 为什么比固定 Deep 更好？

固定 Deep：

~~~text
不管是否已经够了
→ 继续展开
~~~

Adaptive：

~~~text
有 Evidence Gap
→ 才展开
~~~

---

## 10.9 Phase 5：Citation Verification

最终报告生成后，再加一层：

~~~text
Report
  ↓
Extract Factual Claims
  ↓
Find Supporting Evidence
  ↓
Verifier
  ↓
Supported?
  ├─ Yes → attach citation
  └─ No  → rewrite / remove / research again
~~~

数据可以是：

~~~json
{
  "claim": "LangGraph uses checkpointing for durable state...",
  "supported": true,
  "evidence_ids": ["ev_17", "ev_22"]
}
~~~

这会让项目从：

~~~text
Research Agent
~~~

升级成更明确的：

~~~text
Evidence-grounded Research Agent
~~~

简历价值会明显更高。

---

## 10.10 怎么写 Tests

### Unit Tests

至少覆盖：

~~~text
Evidence schema normalization
Budget reserve / release
Budget exceeded
Branch score
Coverage parser
Query normalization
Citation verifier parser
~~~

### Async Tests

重点覆盖：

~~~text
Semaphore 最大活跃任务数
多个 Child 共享 Budget 的原子性
Cancellation
MCP cache 只初始化一次
超时后 Child 是否停止
~~~

### Integration Test

Mock：

~~~text
LLM
Retriever
Scraper
~~~

跑完整：

~~~text
Query
→ Plan
→ Search
→ Context
→ Writer
~~~

验证状态有没有正确传播。

---

## 10.11 怎么做 Benchmark

最终最好做一张表：

| 方案 | Quality | Citation Coverage | Avg Cost | P95 Latency | Search Calls |
|---|---:|---:|---:|---:|---:|
| Basic Baseline | ... | ... | ... | ... | ... |
| Fixed Deep | ... | ... | ... | ... | ... |
| Adaptive Deep | ... | ... | ... | ... | ... |

### 你真正要证明的是

不是：

> “我的代码更多。”

而是：

~~~text
在相近质量下：
成本更低 / 延迟更低

或者：

在相近成本下：
Coverage / Citation 更高
~~~

这才是系统优化。

---

## 10.12 不建议优先做的改造

### 只换模型

学习价值有，但项目深度不够。

### 只加一个 Search Provider

除非你同时做：

- Schema；
- Adapter；
- Fusion；
- Eval。

否则只是在接 API。

### 只调 Prompt

很难证明改进来自哪里，也难建立稳定 Benchmark。

### 一上来改 Multi-Agent

如果 Basic 主链路都没完全吃透，Multi-Agent 只会让 Debug 更困难。

---

## 10.13 项目验收标准

做到这些以后，才建议把“二次开发”作为简历核心项目：

- [ ] 能不看文档画出 Basic 主链路
- [ ] 有至少一个非 trivial 核心改造
- [ ] 有 Unit / Async / Integration Tests
- [ ] 有 Eval Dataset
- [ ] 有 Baseline vs Improved 数据
- [ ] 有失败案例分析
- [ ] 能解释 Quality / Cost / Latency Trade-off
- [ ] 有清晰 Commit History
- [ ] README 能让别人复现
- [ ] 能现场从源码找到关键实现
- [ ] 能明确说明哪些代码来自上游、哪些是你新增的

---

## 10.14 本章总结

~~~text
学习项目
   ↓
跑通源码
   ↓
建立 Baseline
   ↓
发现真实问题
   ↓
做核心改造
   ↓
补 Tests
   ↓
做 Eval
   ↓
拿出数字
   ↓
求职作品
~~~

最后一章讨论怎么把这些技术内容转化成面试表达和真实简历 Bullet。

➡️ [11 - 面试与简历](../11-interview-and-resume/README.md)
