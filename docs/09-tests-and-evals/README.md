# 09 - Tests 与 Evals：从历史 Bug 反推 Agent 工程问题

> 🎯 **本章目标**：学会用测试目录理解一个成熟 Agent 项目“真实坏过什么”，并区分传统软件测试和 LLM Evaluation。读完以后，你应该知道：为什么只看 Happy Path 远远不够，以及怎么给自己的二次开发建立可信 Benchmark。

---

## 目录

- [9.1 为什么读 Tests 比继续读实现更重要](#91-为什么读-tests-比继续读实现更重要)
- [9.2 LLM Structured Output 测什么](#92-llm-structured-output-测什么)
- [9.3 Retriever 为什么有大量 malformed tests](#93-retriever-为什么有大量-malformed-tests)
- [9.4 Scraper 与 URL Security](#94-scraper-与-url-security)
- [9.5 Multi-Agent 为什么要测 Graph Route](#95-multi-agent-为什么要测-graph-route)
- [9.6 Test 和 Eval 有什么区别](#96-test-和-eval-有什么区别)
- [9.7 Context Filter 怎么评](#97-context-filter-怎么评)
- [9.8 Planning 怎么评](#98-planning-怎么评)
- [9.9 Deep Research 怎么评](#99-deep-research-怎么评)
- [9.10 给自己的项目建立 Benchmark](#910-给自己的项目建立-benchmark)
- [9.11 面试高频题](#911-面试高频题)
- [9.12 本章总结](#912-本章总结)

---

## 9.1 为什么读 Tests 比继续读实现更重要

源码告诉你：

> “现在代码怎么写。”

Tests 经常告诉你：

> “过去哪里出过问题，所以现在为什么多了这层 guard。”

在 GPT Researcher 的测试目录里，可以看到很多非常“工程”的名字，例如：

~~~text
test_sub_query_normalization.py
test_source_curator_json_parsing.py
test_tavily_malformed.py
test_tavily_non_dict_response.py
test_serpapi_malformed_results.py
test_url_security.py
test_websocket_manager.py
test_multi_agents_route_bindings.py
test_vector_store_doc_guards.py
~~~

这些名字本身已经在告诉你：

> 真正的 Research Agent，经常坏在“边界数据不符合假设”，而不只坏在模型回答错。

---

## 9.2 LLM Structured Output 测什么

前面看过：

`generate_sub_queries()` 希望模型返回 Query 列表。

理想：

~~~json
["q1", "q2"]
~~~

但测试需要覆盖：

~~~json
{"queries": ["q1", "q2"]}
~~~

~~~json
{"query": "q1"}
~~~

~~~text
q1
~~~

以及：

~~~text
null
empty
wrong item type
broken json
~~~

### 为什么？

因为一个 Agent 的中间输出如果会控制下一步执行，就已经不是普通自然语言了。

它相当于：

> **模型生成的内部 API Payload。**

所以必须像普通 API 一样测：

~~~text
schema variance
empty value
unexpected nesting
invalid type
fallback
~~~

### 读测试时应该问

看到 `test_sub_query_normalization.py`，不要只记：

> “这里有测试。”

而要反推：

~~~text
如果没有 normalization
→ 下游会对 dict 调 append？
→ iteration 得到 key？
→ 整个 research flow 会在哪里崩？
~~~

这才是源码学习。

---

## 9.3 Retriever 为什么有大量 malformed tests

第三方搜索 API 不一定永远返回你期待的结构。

可能发生：

~~~json
null
~~~

或者：

~~~json
{
  "results": "not-a-list"
}
~~~

甚至某条 Result：

~~~json
{
  "title": null,
  "url": null
}
~~~

所以你会看到针对不同 Provider 的 malformed / null / non-dict tests。

### 这里的通用原则

> **Tool / Provider Output 也属于不可信输入。**

Agent 领域很容易只强调：

~~~text
用户输入不可信
LLM 输出不可信
~~~

其实还包括：

~~~text
Search API 输出不可信
Scraper 输出不可信
MCP Tool 输出不可信
Vector Store document 不可信
~~~

### 更理想的边界

二次开发时，可以在 Retriever Adapter 层尽早统一：

~~~python
class SearchResult(BaseModel):
    url: str
    title: str | None
    snippet: str | None
    raw_content: str | None
    provider: str
    requires_scraping: bool
~~~

这样主流程就不用不断写：

~~~python
item.get("href") or item.get("url")
~~~

---

## 9.4 Scraper 与 URL Security

Research Agent 最大的特殊性之一是：

> 它会主动访问用户或搜索引擎给出的 URL。

这会带来 SSRF 风险。

测试应该覆盖：

~~~text
localhost
127.0.0.1
private IP
weird scheme
redirect
encoded host
~~~

### 为什么这不是“安全团队以后再做”的事？

因为 Browser/Scraper 就是 Agent 的执行器之一。

如果 Agent 能：

~~~text
搜索 → 访问 URL
~~~

那么外部内容已经进入系统边界。

这和 Tool Calling 的权限控制是同一类问题。

---

## 9.5 Multi-Agent 为什么要测 Graph Route

LangGraph 工作流不仅要测“某个 Agent 函数返回什么”，还要测：

> **图本身有没有连对。**

例如：

~~~text
writer
→ fact_checker
~~~

如果某次 refactor 改了函数名或 route，单个 Node 的 unit test 全过，整张图仍可能坏。

所以类似 `test_multi_agents_route_bindings.py` 的测试会验证：

- Node 是否存在；
- conditional edge 是否绑定；
- route function 是否能返回合法目标；
- revise / accept 是否指向正确节点。

### Graph Agent 的测试多一层

普通函数：

~~~text
Input → Output
~~~

Graph Workflow：

~~~text
State
→ Node
→ Route
→ Next Node
→ State Update
~~~

因此 Graph Topology 本身就是被测对象。

---

## 9.6 Test 和 Eval 有什么区别

这是 Agent 求职里很常被混淆的问题。

### Test：验证确定性约束

例如：

> “无论模型返回 dict 还是 string，`_normalize_sub_queries` 最终必须给我 `list[str]`。”

这是：

~~~text
input
→ invariant
~~~

理想上可以稳定 Pass / Fail。

### Eval：评估统计性质量

例如：

> “keyword Context Filter 和 embeddings 相比，哪一个更容易找回真正支持答案的 chunk？”

这不是简单 bool。

需要：

~~~text
dataset
→ run system
→ metrics
→ compare
~~~

所以：

~~~text
Unit Test
不能替代
Research Quality Eval
~~~

反过来也一样。

---

## 9.7 Context Filter 怎么评

这是最容易做出可信实验的一层，因为可以固定 Search / Scrape 结果。

准备：

~~~text
Query
+
固定 Candidate Pages / Chunks
+
Gold Relevant Evidence
~~~

分别跑：

~~~text
keyword
embeddings
hybrid/reranker（如果你新增）
~~~

指标：

| 指标 | 解释 |
|---|---|
| Recall@K | Gold Evidence 有多少被找回 |
| MRR | 第一个正确 Evidence 排多前 |
| Context Tokens | 给 Writer 的上下文有多大 |
| Latency | 筛选耗时 |
| Cost | 是否调用外部模型/API |

### 为什么要固定 Candidate Pages？

如果每次连 Search 都重新跑：

~~~text
Search 波动
+
Context Filter 差异
~~~

混在一起，就不知道提升来自哪一层。

这是 Eval 设计中的“控制变量”。

---

## 9.8 Planning 怎么评

Query Decomposition 也可以评。

假设一个题目应该至少覆盖：

~~~text
orchestration
state
observability
use cases
~~~

可以定义：

### Coverage

生成的 Sub Queries 是否覆盖这些维度？

### Redundancy

不同 Query 是否其实在搜同一件事？

### Searchability

Sub Query 能不能搜到有效来源？

### Cost

是不是生成了很多低价值 Query？

### 一个简单的 Eval Record

~~~json
{
  "query": "...",
  "required_dimensions": [
    "orchestration",
    "state",
    "observability"
  ]
}
~~~

然后让 Planner 输出，再用规则 + Judge 做覆盖率判断。

---

## 9.9 Deep Research 怎么评

Deep Research 最容易产生一个错觉：

> “报告更长，所以研究更深。”

这不够。

应该比较：

~~~text
depth = 1
depth = 2
adaptive（如果你实现）
~~~

并记录：

- 新 Source 比例；
- Evidence Novelty；
- 最终质量提升；
- Cost Multiplier；
- Latency Multiplier。

例如：

~~~text
Depth 2:
成本 +180%
新来源 +35%
Citation Coverage +9%
最终质量 +4%
~~~

这时你才有资格讨论：

> Depth 2 值不值得。

---

## 9.10 给自己的项目建立 Benchmark

建议先做 30～50 条 Query，不要一开始追求几百条。

可以分五类：

~~~text
10 条：近期事实 / 最新技术
10 条：框架比较
10 条：长尾技术问题
10 条：学术/论文型
10 条：多维度开放研究
~~~

每条记录：

~~~json
{
  "query": "...",
  "required_dimensions": ["...", "..."],
  "freshness_sensitive": true,
  "reference_sources": ["..."],
  "key_facts": ["..."]
}
~~~

### 至少记录这些运行指标

~~~text
E2E latency
LLM calls
Search calls
URLs visited
Context tokens
Cost
Source count
Report length
Failure / retry count
~~~

### 质量指标

可以逐步加入：

~~~text
Query Coverage
Evidence Recall
Citation Precision
Citation Coverage
Unsupported Claim Rate
Source Diversity
~~~

---

## 9.11 面试高频题

### Q：为什么 Agent 既需要 Test 又需要 Eval？

> “Test 验证确定性软件约束，例如 parser normalization、route binding 和 security guard；Eval 评估模型与检索产生的统计性质量，例如 coverage、evidence recall 和 citation support。二者解决的问题不同。”

### Q：怎么评 Context Filter？

> “固定 Candidate Pages，准备 Gold Evidence，对 keyword/embedding/reranker 做 Recall@K、MRR、Token、Latency 和 Cost 对比，避免 Search 波动污染实验。”

### Q：怎么证明 Deep Research 更好？

> “不能只比较报告长度。应该看新增来源、Evidence Novelty、Citation Coverage、质量指标相对成本和延迟的增益，并做不同 depth/breadth 的消融。”

---

## 9.12 本章总结

~~~text
Tests 告诉你：
系统有没有违反确定性约束

Evals 告诉你：
系统在概率意义上有没有变好
~~~

学习开源 Agent 时，一个非常有效的方法就是：

~~~text
实现看不懂为什么有这个 guard
→ 搜对应 test
→ 找历史失败模式
→ 再回来看实现
~~~

下一章进入实战：不再只读上游，而是把它变成你自己的求职项目。

➡️ [10 - 动手改造路线](../10-hands-on-roadmap/README.md)
