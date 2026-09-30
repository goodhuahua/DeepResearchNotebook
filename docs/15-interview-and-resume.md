# 15 - 面试与简历：怎样把“读过 GPT Researcher”变成可验证能力

> **本章目标**：避免把开源项目包装成自己开发。正确策略是：先能源码级讲解，再做可验证二次开发，最后把自己的改造写进简历。

## 15.1 不建议直接写

不建议：

> “独立开发 GPT Researcher，实现智能研究 Agent。”

这是开源项目，不是你的原创实现。

也不建议：

> “精通 GPT Researcher。”

面试官连续追 3 个函数就能验证。

## 15.2 可以真实表达

如果目前只做源码学习：

> **GPT Researcher 源码分析与 Agent 工程实践**：系统梳理其 Plan-Fanout-Gather-Synthesize 主链路，深入研究 Query Decomposition、多 Retriever、网页抓取、Context Filtering、MCP、Deep Research 递归与 Detailed Report 子 Agent 协作，并基于源码沉淀可复现技术文档。

如果你完成后面的二次开发，再把“分析”升级成“实现”。

## 15.3 最有含金量的不是“跑起来”

最低层：

```text
pip install
→ API key
→ run
```

简历价值很低。

高一层：

```text
read source
→ understand architecture
→ explain trade-offs
```

再高一层：

```text
identify limitation
→ implement modification
→ add tests
→ benchmark
→ document result
```

真正适合 Agent 岗位的是第三层。

## 15.4 推荐简历项目标题

完成二开后可用：

> **Deep Research Agent 工程化优化（基于 GPT Researcher）**

或者：

> **可观测、预算感知的 Deep Research Agent**

或者：

> **Evidence-grounded Research Agent：检索、上下文压缩与引用验证**

标题要突出你的新增工作，而不是只写上游项目名。

## 15.5 推荐做出的三类成果

### A. Shared Budget / Trace

实现：

- request-level CostBudget；
- nested researcher 共享；
- stage trace；
- deadline；
- cost breakdown。

能体现后端工程能力。

### B. Evidence / Citation Verification

实现：

- 结构化 Evidence；
- claim extraction；
- citation support verifier；
- unsupported claim 标记。

能体现 RAG/Research 质量能力。

### C. Adaptive Research Planning

实现：

- coverage check；
- evidence gap；
- repair query；
- information gain stop。

能体现 Agent planning 能力。

三选一做深，胜过同时改十个小功能。

## 15.6 简历 Bullet 模板：预算与可观测性

只有你真的完成后再使用：

> - 基于 GPT Researcher 重构 Research 执行链路，引入 request-scoped shared budget，将 LLM、Search、Scrape、Deep Research 子任务的成本、deadline 与并发配额统一治理，支持按 stage 统计 latency/cost 并在预算不足时动态降级 breadth、Retriever 与模型配置。
> - 设计结构化 Trace，覆盖 Planning → Retrieval → Scraping → Context Filtering → Writing 全链路，记录 Sub-query、Provider、来源数量、上下文规模、重试与成本指标，提升复杂 Agent 质量问题定位效率。

## 15.7 简历 Bullet 模板：Evidence

> - 将 string-based research context 重构为结构化 Evidence Store，保留 source URL、query、chunk、relevance 与 provenance，在 Writer 后增加 claim-citation verification，对无证据支持的事实声明进行检测与回查。
> - 对 lexical / embedding Context Filter 构建离线评测集，比较 Evidence Recall、Citation Precision、Token Cost 与延迟，基于评测结果设计混合 Rerank 策略。

## 15.8 简历 Bullet 模板：Adaptive Planning

> - 在单次 Query Decomposition 基础上实现 Evidence-Gap Replanning：首轮并发研究后由 Coverage Evaluator 识别信息缺口，生成 repair queries 并按 information gain / cost 进行优先级调度，减少固定 breadth/depth 带来的无效搜索。
> - 将递归 Deep Research 重构为 priority frontier scheduler，以 Semaphore + shared budget 控制全局并发和搜索树扩张，支持 branch-level early stop 与任务取消。

## 15.9 STAR 讲法

### Situation

Research Agent 对复杂 Query 采用固定 breadth/depth，成本波动大；nested researcher 的调用链长，定位瓶颈困难。

### Task

在不改变核心检索能力的前提下，让系统：

- 可观测；
- 可预算；
- 可提前终止。

### Action

讲清楚：

- 你读了哪些入口；
- 怎么画出 call graph；
- 状态放在哪里；
- 哪些 child 共享 budget；
- 怎么做 async cancellation；
- 怎么加 test/eval。

### Result

必须尽量有数字：

```text
P95 latency
average cost
source recall
citation precision
failed request rate
```

没有数字时不要虚构，至少给可复现 benchmark 脚本和样例结果。

## 15.10 高频面试题：Agent 架构

**Q1：这个项目的 Agent Loop 在哪里？**

答：普通模式不是经典 while ReAct，核心是 `ResearchConductor._get_context_by_web_search` 的 Plan → parallel sub-query processing → aggregate；Deep 模式再通过 `DeepResearchSkill.deep_research` 递归展开。

**Q2：GPTResearcher 类是不是核心算法？**

答：更多是 Facade/session state holder；普通 research algorithm 在 ResearchConductor。

**Q3：为什么 Query decomposition 能提速？**

答：除了提升 coverage，还把研究问题转换成天然可并行分支，通过 `asyncio.gather` 降低 wall-clock latency。

## 15.11 高频面试题：RAG / Context

**Q：它是不是标准向量 RAG？**

答：不是。Web Research 先 Search + Scrape，再通过可配置 Context Filter 做证据选择；当前 auto 没有 TypeSafe key 时甚至默认 lexical keyword，而 embedding 是可选策略。

**Q：为什么 keyword 可能比 embedding 更合适？**

答：实体和术语检索强、无模型依赖、成本低、可用性好。是否更优要根据 context filter eval 判断。

## 15.12 高频面试题：并发

**Q：并发在哪几层？**

答：

- ordinary sub-query gather；
- Hybrid local/web；
- scraper worker；
- Deep same-level semaphore；
- context written-section lookup。

要关注乘法压力和共享 state。

## 15.13 高频面试题：MCP

**Q：MCP fast/deep 的区别？**

答：fast 主 Query 调一次缓存复用，deep 每个 Sub-query 都执行 MCP。一个偏 cost/latency，一个偏 coverage。

**Q：为什么要 MCP cache lock？**

答：并发 research pass 可能都看到 cache miss，lock 防重复填充。

## 15.14 高频面试题：可靠性

**Q：LLM 返回坏 JSON 怎么办？**

答：`json_repair`、normalization、regex/text fallback、业务 default；不能只靠 prompt 保证。

**Q：所有来源都失败怎么办？**

答：Context 为空时 Writer abstain，不应该生成伪研究报告。

## 15.15 高频面试题：Deep Research

**Q：breadth、depth、concurrency 区别？**

答：

- breadth：每层生成多少研究 Query；
- depth：递归多少层；
- concurrency：同层同时执行多少 branch。

**Q：Deep Research 和 Detailed Report 有何不同？**

答：Deep 优化 evidence exploration；Detailed 优化 output decomposition 和 cross-section consistency。

## 15.16 高频面试题：设计评价

**Q：你认为项目最大的优点？**

可答：

> 把高熵 LLM 决策限制在 role/planning/synthesis，稳定执行部分代码化；同时把 Search、Scrape、Context 分层，让 Research domain workflow 很清晰。

**Q：最大的技术债？**

可答：

> Context / SearchResult 的结构类型不够严格，多个函数需要兼容 string/list/dict 与不同 Provider 字段；此外 child researcher 的 budget/trace 应显式共享。

## 15.17 不要背答案，要能追源码

面试官可能说：

> “你说 Sub-query 是并发的，源码在哪？”

你应该能答：

```text
gpt_researcher/skills/researcher.py
ResearchConductor._get_context_by_web_search
asyncio.gather(_process_sub_query...)
```

> “Context fallback 在哪？”

```text
gpt_researcher/context/select.py
select_context
```

> “Deep child 在哪创建？”

```text
gpt_researcher/skills/deep_research.py
DeepResearchSkill.deep_research → process_query
```

这才叫“读过源码”。

## 15.18 一个 7 天复习计划

| Day | 任务 |
|---|---|
| 1 | 画 Basic 主链路，不看文档复述 |
| 2 | 手写 Planning + Retrieval call graph |
| 3 | 跑 3 种 Context Filter，对比输出 |
| 4 | 跑 Deep Research，记录每层 Query |
| 5 | Debug Detailed Report 子章节状态 |
| 6 | 实现一个二次开发点 + test |
| 7 | 录 10 分钟项目讲解，自问 20 个追问 |

## 15.19 最终原则

简历真正应该表达：

> **我通过一个成熟 Agent 项目学到了什么，并用工程改造证明了我会什么。**

而不是：

> “我 fork 了一个 Star 很多的仓库。”
