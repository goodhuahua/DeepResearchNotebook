# 07 - Deep Research、Detailed Report 与 Multi-Agent

> 🎯 **本章目标**：一次性解决 GPT Researcher 最容易混淆的三个高级概念。读完以后，你应该能非常明确地回答：**哪个是在“搜得更深”，哪个是在“写得更长”，哪个是在“增加角色审核”。**

---

## 目录

- [7.1 先看结论](#71-先看结论)
- [7.2 Deep Research：沿着新线索继续追查](#72-deep-research沿着新线索继续追查)
- [7.3 Detailed Report：按章节创建子 Researcher](#73-detailed-report按章节创建子-researcher)
- [7.4 Multi-Agent：LangGraph 角色工作流](#74-multi-agentlanggraph-角色工作流)
- [7.5 三种模式的状态共享](#75-三种模式的状态共享)
- [7.6 并发方式为什么不同](#76-并发方式为什么不同)
- [7.7 什么时候应该用哪种](#77-什么时候应该用哪种)
- [7.8 面试高频题](#78-面试高频题)
- [7.9 本章总结](#79-本章总结)

---

## 7.1 先看结论

| 模式 | 解决的问题 | 核心机制 |
|---|---|---|
| Deep Research | 证据不够深 | Learnings → Follow-up → Recursion |
| Detailed Report | 报告太长、章节容易重复 | Subtopic Researcher + Previous Written Content |
| Multi-Agent | 需要角色审核闭环 | LangGraph + Editor/Writer/Fact Checker |

一句话记忆：

~~~text
Deep     = 搜得更深
Detailed = 写得更长
Multi    = 审得更多
~~~

理解这三个目标差异以后，源码就不容易混。

---

## 7.2 Deep Research：沿着新线索继续追查

### 普通 Research 的局限

普通模式：

~~~text
先 Plan 一次
→ 执行所有 Sub Query
→ 写报告
~~~

如果研究过程中发现：

> “AutoGen 最近把某些能力拆到了新的 AgentChat / Core 层次。”

普通模式不会自动说：

> “那我再专门研究一下这个新线索。”

Deep Research 会。

### 文件定位

`gpt_researcher/skills/deep_research.py`

核心类：

`DeepResearchSkill`

### 核心状态

初始化时保存：

~~~text
breadth
depth
concurrency_limit
learnings
visited_urls
context
sources
~~~

当前固定源码快照的默认配置是：

~~~text
breadth = 3
depth = 2
concurrency = 4
~~~

### breadth 是什么？

当前这一层要展开多少条 Research Query。

~~~text
Root
├─ Q1
├─ Q2
└─ Q3
~~~

### depth 是什么？

还允许向下追多少层。

~~~text
Root
├─ Q1
│  ├─ Q1.1
│  └─ Q1.2
├─ Q2
│  ├─ Q2.1
│  └─ Q2.2
└─ Q3
~~~

### concurrency 是什么？

同一层最多同时执行多少 branch。

源码使用：

~~~python
semaphore = asyncio.Semaphore(
    self.concurrency_limit
)
~~~

防止一次启动太多 Search、Scrape、LLM 和 MCP 调用。

---

### 最关键：每个 Branch 都创建一个新的 GPTResearcher

简化代码：

~~~python
researcher = GPTResearcher(
    query=serp_query["query"],
    report_type="research_report",
    report_source="web",
    visited_urls=self.visited_urls,
    mcp_configs=...,
    mcp_strategy=...,
)

context = await researcher.conduct_research()
~~~

这意味着：

> Deep Research 没有重新实现 Search / Scrape / Context。

它只是：

~~~text
递归地复用普通 Researcher
~~~

这是一个很典型的“通过组合复用成熟能力”的设计。

---

### Branch 完成后还要再做一次 LLM 分析

Nested Researcher 返回一大段 Context 后，并不是直接进入下一层。

还会：

~~~text
branch context
   ↓
Strategic LLM
   ↓
learnings
+
followUpQuestions
~~~

例如：

~~~json
{
  "learnings": [
    {
      "insight": "LangGraph 将状态持久化与 checkpoint 机制结合...",
      "sourceUrl": "https://..."
    }
  ],
  "followUpQuestions": [
    "LangGraph 的 checkpoint 与 AutoGen runtime state 有什么本质区别？"
  ]
}
~~~

下一层 Query 来自：

~~~text
Previous research goal
+
Follow-up questions
~~~

这就是“深”的真正来源。

---

### 为什么不会无限递归？

至少有五个刹车：

1. `depth` 每层减 1；
2. breadth 会向下缩小；
3. Semaphore 控制同层并发；
4. 当前层如果所有 branch 都失败，会直接停止；
5. Context 有总 word limit。

因此它不是：

~~~text
LLM 想搜多久就搜多久
~~~

而是：

> **在显式资源边界里的递归自治。**

---

## 7.3 Detailed Report：按章节创建子 Researcher

### 目标不同

Detailed Report 不关心“继续发现新问题”。

它主要关心：

> 一篇很长的报告，怎么避免一个 Prompt 一次写崩？

### 文件定位

`backend/report_type/detailed_report/detailed_report.py`

主流程：

~~~python
async def run(self):
    await self._initial_research()
    subtopics = await self._get_all_subtopics()
    introduction = await self.gpt_researcher.write_introduction()
    _, report_body = await self._generate_subtopic_reports(subtopics)
    return await self._construct_detailed_report(
        introduction,
        report_body,
    )
~~~

翻译成人话：

~~~text
先做一次全局研究
   ↓
生成 Subtopics
   ↓
一个章节一个章节专项研究
   ↓
拼成完整长报告
~~~

---

### 第一步：Global Research

~~~python
await self.gpt_researcher.conduct_research()

self.global_context = self.gpt_researcher.context
self.global_urls = self.gpt_researcher.visited_urls
~~~

为什么先全局研究？

因为要先知道：

- 整个主题有哪些主要维度；
- 章节怎么拆；
- 哪些来源已经看过。

---

### 第二步：生成 Subtopics

~~~text
main query
+
global context
→ LLM
→ subtopics
~~~

我们的贯穿示例可能得到：

~~~text
1. Agent Orchestration
2. State Management
3. Observability
4. Suitable Use Cases
~~~

---

### 第三步：每个 Subtopic 创建 Child GPTResearcher

源码核心：

~~~python
subtopic_assistant = GPTResearcher(
    query=current_subtopic_task,
    report_type="subtopic_report",
    parent_query=self.query,
    visited_urls=self.global_urls,
    agent=self.gpt_researcher.agent,
    role=self.gpt_researcher.role,
)
~~~

Child 继承：

- main topic；
- role；
- visited URLs；
- MCP 配置。

这样：

> “State Management”不会被当成一个完全孤立的问题研究，而是知道自己属于“比较三个 Agent Framework”这个总任务。

---

### 第四步：为什么默认顺序执行？

源码：

~~~python
for subtopic in subtopics:
    result = await self._get_subtopic_report(
        subtopic
    )
~~~

这里没有 `asyncio.gather`。

原因是后一个章节会使用：

~~~text
existing_headers
global_written_sections
~~~

例如：

~~~text
Chapter 1 已经写了
“LangGraph checkpoint”

Chapter 2 写 State Management 时
会看到这个内容已经出现
~~~

于是 Writer 可以少重复。

### 这是什么 Trade-off？

~~~text
顺序执行
→ 延迟更高
→ 但跨章节一致性更强

并行执行
→ 更快
→ 但每章不知道别人已经写了什么
~~~

Detailed Report 选择了前者。

---

### Previous Written Content：一种“写作记忆”

每个 Child Researcher 会先生成本章节的 draft section titles。

然后：

~~~text
current subtopic
+
draft section titles
→ 检索 previous written sections
~~~

把相关已写内容交给 Writer。

这不是聊天 Memory，而是：

> **防重复、保持长文一致性的局部记忆。**

---

## 7.4 Multi-Agent：LangGraph 角色工作流

### 文件定位

`multi_agents/agents/orchestrator.py`

核心类：

`ChiefEditorAgent`

它创建：

~~~text
WriterAgent
EditorAgent
ResearchAgent
PublisherAgent
HumanAgent
FactCheckerAgent
VisualizerAgent
~~~

### 顶层 StateGraph

简化后：

~~~text
Initial Research
   ↓
Planner
   ↓
Human Review
   ├─ revise → Planner
   └─ accept
        ↓
Parallel Section Research
        ↓
Writer
        ↓
Fact Checker
   ├─ revise → Writer
   └─ accept
        ↓
Visualizer
        ↓
Publisher
        ↓
END
~~~

这和 Basic Research 最大区别是：

> 显式把“编辑、审稿、事实检查”变成不同角色节点。

---

### Multi-Agent 仍然复用 GPTResearcher

这点非常重要。

`ResearchAgent` 内部并不是自己实现 Search。

它会：

~~~text
ResearchAgent
→ GPTResearcher
→ conduct_research
→ write_report
~~~

因此：

> Multi-Agent 是更高一层的 orchestration，GPTResearcher 仍然是底层 research worker。

---

### Human / Fact Check 为什么一定要有上限？

图里有循环：

~~~text
Planner
↔ Human

Writer
↔ Fact Checker
~~~

如果 Evaluator 永远返回：

~~~text
revise
~~~

系统就可能无限循环。

所以有：

~~~text
max_plan_revisions
max_fact_check_revisions
~~~

达到上限后强制进入下一阶段，而不是最终撞 LangGraph recursion limit。

这是所有 Evaluator-Optimizer Agent 都必须考虑的问题。

---

## 7.5 三种模式的状态共享

### Deep

共享重点：

~~~text
visited_urls
learnings
citations
context
MCP config
~~~

### Detailed

共享重点：

~~~text
global_context
global_urls
global_written_sections
existing_headers
role
~~~

### Multi-Agent

共享方式不同：

~~~text
LangGraph State
~~~

每个 Node：

~~~text
state
→ process
→ state update
~~~

这也是 OOP Session State 和 Graph State 两种不同 Agent 状态管理风格。

---

## 7.6 并发方式为什么不同

### Deep：同层并发

因为 Branch 相对独立：

~~~text
Q1
Q2
Q3
→ gather with semaphore
~~~

### Detailed：顺序

因为后面的章节依赖前面已写内容。

### Multi-Agent：Section Research 并行

它不等所有章节互相看成稿，而是提前把：

~~~text
sibling_sections
~~~

告诉每个 Worker：

> 这些主题由别的章节负责，不要重复。

这是一种用“静态边界”换取并发速度的做法。

---

## 7.7 什么时候应该用哪种

### 普通技术调研

优先：

~~~text
Basic
~~~

先别上复杂模式。

### 需要很长、有目录的报告

考虑：

~~~text
Detailed
~~~

### 问题探索性很强，首轮搜索很难覆盖

考虑：

~~~text
Deep
~~~

### 需要 Human Review / Fact Checker / Revision Loop

考虑：

~~~text
Multi-Agent
~~~

一个成熟的工程判断是：

> **不要因为“多 Agent 更酷”就默认使用 Multi-Agent。**

更多 Agent 意味着更多状态、更多 LLM call、更多失败点、更高成本和更难 Debug。

---

## 7.8 面试高频题

### Q：Deep 和 Detailed 有什么区别？

> “Deep 优化 evidence exploration，通过 learnings 和 follow-up questions 递归下钻；Detailed 优化 long-form report generation，通过 subtopic researcher、existing headers 和 previous written sections 协调章节。”

### Q：Detailed 为什么不并行？

> “因为章节写作有顺序依赖。后一个章节需要看到前面已经写过的 headers 和相关内容来减少重复，所以它选择质量一致性而不是最低延迟。”

### Q：Multi-Agent 为什么还需要 GPTResearcher？

> “LangGraph 负责角色级 orchestration，而具体 research ability 仍由 GPTResearcher 提供。ResearchAgent 只是把 GPTResearcher 封装成图节点里的 worker。”

### Q：所有 Review Loop 为什么都要有上限？

> “LLM-based evaluator 不是确定性的，可能持续要求 revise。没有最大 revision/budget，图会无限循环或撞 recursion limit。”

---

## 7.9 本章总结

~~~text
Basic
  Plan → Research → Write

Deep
  Research → Learn → Follow-up → Recurse

Detailed
  Global Research → Subtopics → Sequential Sections

Multi-Agent
  Planner → Human → Researchers → Writer
        → Fact Checker → Publisher
~~~

下一章把视角从“算法”切到“工程”：一个真实 Agent 为什么需要大量 retry、fallback、lock、cost 和 trace。

➡️ [08 - 工程化](../08-engineering/README.md)
