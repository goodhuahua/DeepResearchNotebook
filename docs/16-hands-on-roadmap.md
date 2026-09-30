# 16 - 二次开发路线：把学习仓库升级成你的 Agent 项目

> **本章目标**：给出一条可落地的改造路线。建议选一条主线做完整：设计 → 实现 → 测试 → Eval → Benchmark → 文档，而不是到处改两行代码。

## 16.1 推荐主线：Adaptive + Budget-aware Deep Research

这是最适合 Agent 求职的综合改造。

目标：

```text
固定 breadth/depth
→ 动态 research frontier

实例独立成本
→ shared request budget

字符串 Context
→ structured evidence

事后看日志
→ structured trace
```

## 16.2 Phase 1：先建立可复现 Baseline

准备 20–50 个 Query，至少覆盖：

- 最新事件；
- 技术对比；
- 公司研究；
- 学术问题；
- 多维度综述。

记录：

```text
query
report
latency
cost
number of sources
number of search calls
context size
```

没有 baseline，后面无法证明优化有效。

## 16.3 Phase 2：ResearchState

先不要动算法，先把隐式状态显式化。

```python
@dataclass
class ResearchState:
    query: str
    visited_urls: set[str]
    evidence: list["Evidence"]
    costs: "CostState"
    trace_id: str
```

逐步替换散落 state。

收益：

- child 可共享；
- test 更容易；
- 数据边界清晰。

## 16.4 Phase 3：Evidence Model

```python
class Evidence(BaseModel):
    id: str
    query: str
    url: str
    title: str | None
    text: str
    source_provider: str
    relevance_score: float | None
    content_hash: str
```

Retriever/Scraper/Context 最终都产 Evidence。

不要过早 serialize 成 string。

## 16.5 Phase 4：Shared Budget

```python
class ResearchBudget:
    max_cost_usd: float
    max_search_calls: int
    max_llm_calls: int
    max_urls: int
    deadline: float

    async def reserve_llm(...): ...
    async def reserve_search(...): ...
```

每个 child 共享同一个 instance。

## 16.6 Phase 5：Stage Trace

定义：

```python
@asynccontextmanager
async def trace_stage(name, **attrs):
    ...
```

包：

- choose_agent；
- plan；
- search；
- scrape；
- context；
- writer；
- deep branch。

输出 JSONL 或 OpenTelemetry。

## 16.7 Phase 6：Coverage Evaluator

首轮 Research 后，输入：

```text
original query
planned dimensions
evidence summaries
```

输出：

```json
{
  "covered": ["..."],
  "missing": ["..."],
  "uncertain": ["..."],
  "repair_queries": ["..."]
}
```

只有有缺口才继续搜索。

## 16.8 Phase 7：Priority Frontier

替代递归函数：

```text
PriorityQueue
  item:
    query
    goal
    depth
    expected_value
```

Worker：

```text
pop best task
→ research
→ evaluate learnings
→ generate children
→ score
→ push
```

停止：

```text
queue empty
OR budget exhausted
OR deadline
OR information gain too low
```

这样比递归更容易做全局调度。

## 16.9 Branch Score

简单版本：

```text
score =
  novelty
+ uncertainty
+ expected coverage gain
- estimated cost
```

不必一开始训练模型，用 LLM structured evaluator 也可以。

## 16.10 Information Gain

用已有 Evidence 做 semantic similarity：

```text
new_evidence
vs
existing_evidence
```

如果高度重复：

```text
branch gain low
→ stop expansion
```

这能减少 Deep Research 反复搜同一观点。

## 16.11 Phase 8：Citation Verifier

Writer 输出后：

1. Extract claims；
2. 对每个 factual claim 找 supporting evidence；
3. Verify entailment；
4. unsupported → repair/rewrite；
5. 加 citation。

结构：

```json
{
  "claim": "...",
  "supported": true,
  "evidence_ids": ["e12", "e31"]
}
```

## 16.12 Phase 9：Eval

至少四类指标：

### Quality

- query coverage；
- factuality；
- citation precision；
- source diversity。

### Efficiency

- latency；
- cost；
- LLM calls；
- Search calls。

### Reliability

- failed request；
- empty context；
- retry count；
- timeout。

### Planning

- unnecessary branch ratio；
- repair query success；
- evidence novelty。

## 16.13 Context Filter A/B

直接比较：

```text
keyword
embeddings
hybrid
```

固定同一批 scraped pages，避免 Search 波动影响。

评：

- gold evidence recall@k；
- final answer support；
- context tokens；
- runtime；
- API cost。

这会让你的项目从“主观觉得”升级成“有数据”。

## 16.14 建议 PR / Commit 结构

```text
feat: add typed evidence model
feat: add shared research budget
feat: add structured trace spans
feat: add coverage evaluator
feat: add adaptive frontier scheduler
feat: add citation verifier
test: add budget and cancellation tests
eval: add research quality benchmark
docs: document architecture and benchmark
```

面试官看 commit history 就能理解演进。

## 16.15 Test 设计

### Unit

- Budget reserve / exhausted；
- Evidence dedup；
- Query normalization；
- branch scoring；
- parser fallback。

### Async Concurrency

- 多 branch 不超 semaphore；
- cancellation；
- MCP cache only filled once；
- shared budget atomic。

### Integration

mock：

- LLM；
- Search；
- Scraper。

跑完整：

```text
Query → Report
```

### Regression

针对每个修复 bug 留 test。

## 16.16 不建议作为第一改造的功能

### “换一个模型”

太浅。

### “加一个 Search Provider”

除非你同时设计插件、schema 和 eval，否则面试价值有限。

### “加 UI”

如果应聘 Agent 后端/算法，优先级低。

### “Prompt 调一调”

难以证明工程能力。

## 16.17 第二条可选路线：Enterprise Research

如果你偏后端：

- tenant config；
- auth；
- async job queue；
- persistence；
- distributed worker；
- Redis dedup；
- retry/circuit breaker；
- OpenTelemetry；
- S3 artifacts；
- quota/billing。

可以展示生产系统能力。

## 16.18 第三条可选路线：Local Knowledge Research

如果偏 RAG：

- local docs；
- vector store；
- BM25 + vector hybrid；
- reranker；
- metadata filter；
- evidence graph；
- citation verifier。

## 16.19 第四条可选路线：Multi-Agent Review

如果偏 Agent framework：

```text
Researcher
→ Writer
→ Fact Checker
→ Reviewer
→ Reviser
```

但要比较：

> 多 Agent 是否真的比单 Workflow 提升质量？

做 ablation：

```text
single
single + verifier
multi-agent
```

比较 cost/quality，才有价值。

## 16.20 最推荐的最终 Demo

输入：

> “比较三种主流 Agent Framework 在长期任务、Tool Use、Context Management、Observability 上的设计，并给出处。”

UI/CLI 显示：

```text
Plan
  4 dimensions

Research
  12 sources
  3 repair queries

Budget
  $0.43 / $0.60
  68% time remaining

Evidence
  31 chunks
  9 domains

Verification
  24 factual claims
  23 supported
  1 rewritten

Final report
```

这比“输出一篇 Markdown”更能体现 Agent 工程。

## 16.21 项目验收标准

只有满足下面这些，我才建议把“二次开发”作为简历重点：

- [ ] 能画自己的 architecture；
- [ ] 有至少一个非 trivial 核心改造；
- [ ] 有 tests；
- [ ] 有 eval dataset；
- [ ] 有 baseline vs improved 数据；
- [ ] 有失败案例；
- [ ] 能解释 trade-off；
- [ ] 有清晰 commit history；
- [ ] README 可复现；
- [ ] 能现场从源码找到关键函数。

完成以后，这个仓库才从“学习笔记”变成“求职作品”。
