# 08 - Deep Research：把一次研究扩展成递归搜索树

> **本章目标**：完整拆解 `DeepResearchSkill`。这是 GPT Researcher 里最接近“自主研究”的实现：模型不只生成一批 Sub-query，还会从已有结果中生成 Follow-up Questions，并继续向下探索。

## 8.1 普通 Research 与 Deep Research 的差异

普通：

```text
Query
→ one planning step
→ N sub-queries
→ parallel research
→ aggregate
→ write
```

Deep：

```text
Query
→ follow-up research plan
→ breadth queries
→ nested researchers
→ extract learnings + follow-up questions
→ if depth > 1:
      build next query
      recurse
→ aggregate
→ write
```

核心区别不是“搜索次数更多”，而是：

> **下一层研究问题由上一层证据动态产生。**

## 8.2 配置

当前默认：

```text
DEEP_RESEARCH_BREADTH = 3
DEEP_RESEARCH_DEPTH = 2
DEEP_RESEARCH_CONCURRENCY = 4
```

三者分别控制：

- breadth：当前层生成多少条 SERP query；
- depth：递归剩余层数；
- concurrency：同一层最多并行跑多少 nested researcher。

## 8.3 DeepResearchSkill 初始化

它拿到 parent `GPTResearcher`，继承：

- tone
- config path
- headers
- websocket
- visited_urls
- MCP config/strategy

并维护：

- learnings
- research_sources
- context

因此 Deep Research 不是一套完全独立引擎，而是**组合普通 GPTResearcher**。

## 8.4 run() 的第一步不是直接递归

首先：

```text
generate_research_plan(original query)
```

这个函数：

1. 用所有 Retriever 获取 initial web knowledge；
2. 加入当前时间；
3. 让 Strategic LLM 生成多个 follow-up questions。

然后当前实现并不等待用户回答，而是自动填：

```text
"Automatically proceeding with research"
```

最后组成：

```text
Initial Query: ...
Follow-up Questions and Answers:
Q: ...
A: Automatically proceeding with research
...
```

再进入真正 `deep_research`。

这说明代码支持“澄清问题式规划”的结构，但当前自动模式实际上是自问自答后继续。

## 8.5 每层第一步：generate_search_queries

输入当前 query，要求 Strategic LLM 返回：

```json
[
  {
    "query": "...",
    "researchGoal": "..."
  }
]
```

这里和普通 `generate_sub_queries` 不完全一样：

Deep Research 每个 Query 还显式携带 `researchGoal`。

后面递归会继续使用这个 goal。

## 8.6 LLM 输出解析

Deep Research 自己实现了一套比较强的 parser recovery：

- 先找 JSON；
- 支持 fenced JSON；
- `json_repair`；
- 支持 dict wrapper；
- 如果 JSON 仍失败，再按文本行模式识别：
  - `Query:`
  - `Goal:`
  - `Question:`
  - `Learning:`
- URL 正则抽 citation。

这说明深度研究链路非常依赖结构化中间状态，所以 parser 被当成核心模块。

## 8.7 同层并发

代码：

```python
semaphore = asyncio.Semaphore(self.concurrency_limit)

async def process_query(...):
    async with semaphore:
        ...

results = await asyncio.gather(*tasks)
```

即：

```text
q1 ─┐
q2 ─┼─ concurrency limit → gather
q3 ─┤
q4 ─┘
```

避免 breadth 很大时瞬间启动大量：

- Search；
- Scrape；
- LLM；
- MCP。

## 8.8 每个 Query 都创建 Nested GPTResearcher

这是 Deep Research 最关键的实现：

```python
researcher = GPTResearcher(
    query=serp_query["query"],
    report_type="research_report",
    report_source="web",
    visited_urls=self.visited_urls,
    mcp_configs=...,
    mcp_strategy=...,
)

context = await researcher.conduct_research()
```

也就是说，每个 Deep branch 内部又执行一遍完整普通 Research：

```text
deep query
→ choose role
→ initial search
→ subqueries
→ search/scrape/context
→ branch context
```

因此复杂度不是简单的 `breadth × depth`，而是“递归层 × 每个 nested researcher 内部的普通 fan-out”。

## 8.9 结果再分析：process_research_results

Nested Researcher 产生大量 context 后，Deep Skill 再用 Strategic LLM 提炼：

```json
{
  "learnings": [
    {
      "insight": "...",
      "sourceUrl": "..."
    }
  ],
  "followUpQuestions": [
    "..."
  ]
}
```

这一层完成两个任务：

### Compression

把 branch context 压缩成 learnings。

### Replanning Signal

把证据缺口或新方向变成 follow-up question。

这就是 Deep Research 的闭环。

## 8.10 递归 Query 怎样形成

如果 `depth > 1`：

```text
Previous research goal: ...
Follow-up questions: ...
```

拼成下一层 Query。

然后：

```text
new_depth = depth - 1
new_breadth = max(2, breadth // 2)
```

即越往下：

- 深度递减；
- 广度大致收缩；
- 但最低仍保留 2。

这是一种“先广后深”的启发式。

## 8.11 递归结构图

假设 breadth=3, depth=2：

```text
Root
├── Q1
│   └── ordinary GPTResearcher
│       └── learnings / followups
│           ├── Q1.1
│           └── Q1.2
├── Q2
│   └── ordinary GPTResearcher
│       └── learnings / followups
│           ├── Q2.1
│           └── Q2.2
└── Q3
    └── ordinary GPTResearcher
        └── learnings / followups
            ├── Q3.1
            └── Q3.2
```

注意：当前代码先并发处理一层 `Q1/Q2/Q3`，但对每个 result 的更深层递归是在后续 `for result in results` 中触发的，并不是把整棵树所有下一层节点一次性全并发出去。

## 8.12 为什么深层不无限扩张

至少有四个刹车：

1. `depth` 每层 -1；
2. breadth 向下缩；
3. semaphore 限同层并发；
4. 如果当前层所有 branch 都失败，直接停止 descent。

其中第 4 条是很重要的 bug-fix 型保护：

```text
all branches failed
→ do NOT generate endless follow-up work
→ return current accumulated result
```

## 8.13 空 Query 保护

如果 `generate_search_queries` 最终一条都没解析出来：

```text
stop descent
→ return current accumulated state
```

避免结构化输出失败导致后续逻辑异常。

## 8.14 Context Word Limit

Deep Research 容易爆炸。

代码设：

```text
MAX_CONTEXT_WORDS = 25000
```

`trim_context_to_word_limit` 从后往前保留内容，偏向较新的 item。

如果第一条单 item 就超限，会截到 max words。

这是一个非常直接的 context budget。

## 8.15 为什么从后往前保留

注释写的是：

> preserve most recent/relevant items

Deep Research 越往后通常是越深层的 follow-up evidence，因此作者倾向保留后续内容。

但“更近 = 更相关”不总成立。

可改为基于 relevance / information gain 排序，而不是时间顺序。

## 8.16 Learnings 去重

返回：

```python
list(set(all_learnings))
```

简单有效，但有两个问题：

- 顺序丢失；
- 语义重复但字符串不同仍无法去重。

可以改成 embedding clustering / semantic dedup。

## 8.17 Citation 状态

Deep parser 建立：

```text
learning -> citation URL
```

最后 `run` 会把 citation 拼到 learning 后面，再把 all context 也加入最终 context。

这比完全不保留 citation 更强，但仍是“文本级引用”，不是严格 claim graph。

## 8.18 visited_urls

Nested Researcher 收到共享 visited URL set。

目标：

- 深层不要反复读相同 source；
- 让探索尽量向新证据扩张。

但也有 trade-off：

> 某个 URL 可能对 Q1 只读出了一个角度，对 Q2 仍很关键；全局 visited 会阻止重复抓取，但 Context Manager 可以依赖此前保留的 context，是否足够取决于状态保存方式。

## 8.19 成本复杂度

粗略看：

```text
deep branch count
×
ordinary researcher cost
+
deep planner / extractor calls
```

而 nested ordinary researcher 内又有 N 个 sub-query。

所以 Deep Research 很容易比 Basic Report 贵一个数量级。

面试时不要只说“更深效果更好”，要说清 cost surface。

## 8.20 当前成本统计需要警惕

`DeepResearchSkill.run` 用 parent researcher 的 `get_costs()` 前后差计算 deep research cost。

但 nested `GPTResearcher` 是新实例，各自拥有自己的 cost state。阅读这段代码时应进一步验证 nested cost 是否被显式汇总回 parent；当前创建代码没有直观看到共享 cost callback。

因此如果做成本治理，我会把统一 `CostTracker` 作为显式共享依赖传给所有 child researcher，而不是依赖每个实例自己的 float。

## 8.21 Deep Research 是 Multi-Agent 吗

广义上可以说它创建多个 Agent instance。

但它不是“不同角色互相发消息”的群体 Agent。

更准确：

> **Recursive hierarchical research workers**。

每个 child 都是完整 GPTResearcher，但它们主要通过父流程汇总，而不是 peer-to-peer communication。

## 8.22 可以怎样改进

### Information Gain Stop

不是固定 depth，而是：

```text
if new evidence adds little information:
    stop branch
```

### Branch Scoring

对 follow-up branch 按：

- novelty；
- uncertainty；
- expected evidence value；

排序，只展开高价值分支。

### Shared Evidence Store

所有 nested researcher 写入同一个结构化 evidence store，而不是各自 string context 再 merge。

### Global Scheduler

把递归改成显式 frontier：

```text
priority queue of research tasks
→ bounded workers
→ dynamically add followups
```

更容易做预算和并发。

## 8.23 面试题

**Q：Deep Research 的“深”具体体现在哪？**

A：每一层 branch 完成普通研究后，Strategic LLM 从结果提取 learnings 和 follow-up questions，再把 research goal + followups 组成下一层 query 递归执行；不是简单增加一次搜索数量。

**Q：如何防止爆炸？**

A：固定 depth、收缩 breadth、Semaphore 并发限制、失败层终止、25k word context cap。

**Q：为什么 child 直接复用 GPTResearcher？**

A：复用稳定的普通 research pipeline，Deep Skill 只负责树形 orchestration，而不用重复实现 Retriever/Scraper/Context 等底层能力。

**Q：你会怎么重构？**

A：把递归改成 frontier scheduler + shared evidence store + shared cost budget，让 branch 可以按 information gain 动态扩展和停止。
