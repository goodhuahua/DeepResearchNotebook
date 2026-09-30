# 03 - Planning：角色选择与 Query Decomposition

> **本章目标**：理解 GPT Researcher 中“模型真正负责决策”的第一个核心节点——它怎样把一个开放问题变成可并行执行的研究计划。

## 3.1 Planning 解决什么问题

用户问题可能是：

> “分析 2026 年 Agent coding 产品的技术路线、商业模式和竞争格局。”

直接拿这个 Query 搜一次会遇到：

- 搜索覆盖面不足；
- 结果集中在最热门关键词；
- 不同维度混在一起；
- Writer 很难得到结构均衡的证据。

因此先拆成：

```text
Q1 技术架构和 Agent loop
Q2 主流产品与市场玩家
Q3 商业模式和定价
Q4 近期趋势 / 新发布能力
Q5 风险与限制
```

然后并发研究。

## 3.2 角色选择：choose_agent

路径：

```text
GPTResearcher.conduct_research
→ choose_agent
→ smart LLM
```

输入：

- query
- parent_query（如果是子主题）
- PromptFamily 的 auto-agent instruction

输出期望 JSON：

```json
{
  "server": "...",
  "agent_role_prompt": "..."
}
```

真正重要的是 `agent_role_prompt`。

后续它会作为 system message 参与 Planning / Writing。

## 3.3 为什么角色生成使用 Smart LLM

角色一旦选错，会污染后续所有 stage：

```text
bad role
→ bad query decomposition
→ bad source priorities
→ bad writing criteria
```

所以它不是简单分类器，而属于全局高杠杆决策。

当前实现使用 smart model，温度较低（`0.15`），目标是稳定。

## 3.4 LLM JSON 不可信：三层恢复

`choose_agent` 不直接相信 `json.loads`。

大致顺序：

```text
json_repair.loads
   ↓ fail / wrong shape
json.loads
   ↓ fail
handle_json_error
   ├── json_repair
   ├── regex extract {...}
   └── default agent
```

最后 fallback：

```text
Default Agent
+ generic critical-thinking researcher role
```

这是一个非常实用的工程经验：

> **如果 LLM 的结构化输出控制系统流程，就必须把 parser recovery 当成业务逻辑，而不是异常边角。**

## 3.5 Planning 入口

`ResearchConductor.plan_research`：

```text
query
+ search_results
+ role
+ cfg
+ parent_query
+ report_type
+ retriever_names
→ plan_research_outline
```

如果没传 search_results，它会先执行 initial search。

## 3.6 Grounded Planning

为什么先搜一次？

传统：

```text
Query
→ LLM
→ Sub Queries
```

GPT Researcher：

```text
Query
→ Initial Retriever
→ Search Results
→ Strategic LLM
→ Sub Queries
```

这让规划获得“当前世界”的信号。

例如 Query 中出现一个模型刚发布两天，LLM 参数记忆可能不知道；Search Result 至少可以把名称、公司、发布时间等信息暴露给 Planner。

## 3.7 Strategic LLM

`generate_sub_queries` 使用：

```text
cfg.strategic_llm_provider
cfg.strategic_llm_model
ReasoningEffort = medium
```

而不是统一全用 Smart LLM。

这体现了**按认知任务分模型**：

| 类型 | 典型职责 |
|---|---|
| Fast | 低成本轻任务 |
| Smart | 高质量生成 |
| Strategic | 规划、推理、Query decomposition |

当前默认 Smart 和 Strategic 可能是同一模型，但逻辑角色仍然分开，方便后续独立配置。

## 3.8 Query Prompt 的关键输入

`generate_search_queries_prompt` 获得：

- current query
- parent query
- report type
- `MAX_ITERATIONS`
- initial search context

其中 `MAX_ITERATIONS` 在这里更接近“希望生成多少条研究 Query 的约束”，并不是一个通用 Agent while-loop 的最大 turn。

这点面试时要避免说错。

## 3.9 Strategic LLM 的重试和降级

第一尝试：

```text
strategic model
reasoning effort = medium
max_tokens = None
```

失败后：

```text
same strategic model
+ strategic_token_limit
```

再次失败：

```text
fallback to smart model
```

这是一条典型的：

```text
primary strategy
→ constrained retry
→ fallback provider/model role
```

## 3.10 Sub-query 输出归一化

LLM 可能返回：

```json
["a", "b"]
```

也可能：

```json
{"queries": ["a", "b"]}
```

甚至：

```json
{"query": "a"}
```

或者一个纯字符串。

`_normalize_sub_queries` 会统一为：

```python
list[str]
```

如果最后一条都没有，就 fallback 到 original query。

意义：

> 下游应该依赖稳定的数据契约，而不是依赖“Prompt 说了请返回 JSON 数组”。

## 3.11 MCP-only 的特殊规划

`plan_research_outline` 有一个优化：

如果当前只有 MCP Retriever：

```text
skip sub-query generation
→ return [original query]
```

原因是某些 MCP 工具内部本身已经带有工具选择 / 复杂检索能力，再额外做 N 个 sub-query 可能只增加调用成本。

如果 MCP 和普通 Retriever 混用，则仍给普通 Retriever 生成 Sub-query。

## 3.12 为什么最后还要追加 original query

普通 report（非 `subtopic_report`）会：

```python
sub_queries.append(query)
```

这相当于保留一个“总问题”检索通道。

原因：

- decomposition 可能遗漏整体性 source；
- 原始 Query 可能能找到权威 overview；
- 子问题只覆盖局部视角。

它是一种低成本 recall insurance。

## 3.13 Planning 的并行价值

规划不是为了“看起来更 Agent”，而是把后续 workload 转换成适合并行的 DAG：

```text
                 Query
                   │
                Planner
          ┌────────┼────────┐
          ▼        ▼        ▼
         Q1       Q2       Q3
          │        │        │
       search   search   search
          │        │        │
       context  context  context
          └────────┼────────┘
                   ▼
                aggregate
```

比一个 Query 反复搜索更容易做：

- concurrency；
- per-branch timeout；
- per-branch retry；
- source diversity；
- progress tracking。

## 3.14 Planning 的局限

### 1. 没有显式 coverage evaluator

生成 3 个 Sub-query 后，系统不会先评估“这些 Query 是否覆盖了所有研究维度”。

### 2. 普通模式不是自适应 Planning

Sub-query 执行后，不会根据证据缺口再次补 Query；这是 Deep Research 才更接近的能力。

### 3. Initial Search 主要来自第一个 Retriever

这意味着 Planner 的第一印象可能有 provider bias。

### 4. 规划和报告结构不是完全同一层

Sub-query 是“怎么找资料”，Report section 是“怎么组织输出”，两者不是一回事。

## 3.15 如果我来改：Coverage-aware Planning

可以增加：

```text
Query
→ draft research dimensions
→ generate candidate subqueries
→ coverage judge
→ repair missing dimensions
→ execute
```

输出结构：

```json
{
  "dimensions": [
    {
      "name": "technical architecture",
      "queries": ["..."],
      "success_criteria": "..."
    }
  ]
}
```

这样更适合复杂企业研究。

## 3.16 如果我来改：Evidence-driven Replanning

普通模式可以加入轻量二阶段：

```text
Plan
→ Execute
→ Evidence Gap Check
→ if gaps:
      Generate Repair Queries
      Execute
→ Write
```

这会介于当前普通模式和 Deep Research 之间。

## 3.17 面试题

**Q：为什么不是让 LLM 每次自己选 Search Tool？**

A：Research 任务的高层结构比较稳定。把“并发搜索、抓取、context filtering”固化成代码可以降低 tool selection error 和 token 开销；LLM 只负责高熵的 Query decomposition。

**Q：为什么 Planning 之前要先 Search？**

A：让 Planner grounded 到当前搜索环境，尤其对近期事件、新实体和用户可能写错的术语更可靠。

**Q：怎么处理 LLM 不按 JSON 输出？**

A：parser recovery + schema normalization + original query fallback。上游在 agent selection、sub-query、deep research parser 都大量采用这种模式。

**Q：`MAX_ITERATIONS=3` 是不是 Agent 最多跑 3 轮？**

A：不是通用 ReAct turn。普通 research 中它主要进入 search query generation prompt，影响研究 query 的规划规模；具体执行是对这些 sub-query fan-out。
