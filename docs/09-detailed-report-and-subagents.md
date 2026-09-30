# 09 - Detailed Report：子 Researcher 怎样协作写长报告

> **本章目标**：理解另一种“多 Agent”形态——不是递归探索，而是按报告子主题拆分多个 Researcher，并利用全局已写内容减少重复。

## 9.1 为什么普通 Report 不够

Basic Report：

```text
all context
→ one writer call
→ report
```

长报告会遇到：

- 单 Prompt 太长；
- 章节覆盖不均；
- 不易控制结构；
- 某些子领域证据不充分；
- 章节重复。

Detailed Report 将“研究 + 写作”按子主题拆开。

## 9.2 主流程

`DetailedReport.run`：

```text
_initial_research()
→ _get_all_subtopics()
→ write_introduction()
→ _generate_subtopic_reports()
→ merge visited URLs
→ _construct_detailed_report()
```

最终：

```text
Introduction
+ Table of Contents
+ Subtopic Reports
+ Conclusion
+ References
```

## 9.3 第一步：Global Research

初始化一个 main `GPTResearcher`：

```text
report_type = research_report
```

先：

```python
await self.gpt_researcher.conduct_research()
```

得到：

- global_context；
- global_urls；
- main agent/role。

这个 Global Research 为后续生成 subtopics 和章节提供整体视角。

## 9.4 生成 Subtopics

```text
main query
+ global context
→ ReportGenerator.get_subtopics
→ construct_subtopics
→ PydanticOutputParser(Subtopics)
```

这里使用结构化 parser，输出每个：

```text
subtopic.task
```

数量受 `MAX_SUBTOPICS` 控制，当前默认 3。

## 9.5 每个 Subtopic 创建一个新 GPTResearcher

`_get_subtopic_report`：

```python
subtopic_assistant = GPTResearcher(
    query=current_subtopic_task,
    report_type="subtopic_report",
    parent_query=self.query,
    visited_urls=self.global_urls,
    agent=self.gpt_researcher.agent,
    role=self.gpt_researcher.role,
    ...
)
```

关键共享状态：

- parent query；
- main agent；
- main role；
- visited URLs；
- MCP config。

也就是说 child 不需要重新定义整体身份。

## 9.6 Child 的初始 Context

创建后：

```text
child.context = dedup(global_context)
```

然后 child 还会：

```python
await subtopic_assistant.conduct_research()
```

所以每个章节不是只从 Global Context 切一块出来，而是：

> **先继承全局知识，再针对自己的主题补做专项研究。**

## 9.7 为什么 Subtopic Report 有 parent_query

如果 child query 只是：

> “商业模式”

它不知道整篇报告主题。

传入：

```text
parent_query = "AI Agent coding products..."
```

Planner/role 可以理解：

> 我研究的是“该总主题下的商业模式”，不是泛泛研究商业模式。

这是 hierarchical context。

## 9.8 章节草稿标题

Child 完成 research 后：

```text
current subtopic
+ child context
→ get_draft_section_titles
→ draft titles
```

再解析成 header text。

为什么先生成标题？

因为下面需要用这些标题去检索“以前写过的相关内容”。

## 9.9 Written Content Retrieval

全局维护：

```text
global_written_sections
```

当前 child：

```text
current_subtopic
+ draft_section_titles
→ ContextManager
→ search previous written sections
→ relevant_written_contents
```

这是一种非常有意思的“写作记忆”。

不是聊天 memory，而是：

> **跨章节的内容去重与一致性 context。**

## 9.10 Existing Headers

还维护：

```text
existing_headers
```

每写完一个 subtopic：

```text
{
  "subtopic task": ...,
  "headers": ...
}
```

下一个 Writer 会看到已有结构。

于是它同时知道：

- 哪些 heading 已经用过；
- 哪些相关内容已经写过。

## 9.11 Subtopic Writer 输入

```text
query = current subtopic
main_topic = parent query
context = fresh evidence
existing_headers = previous report structure
relevant_written_contents = previous related sections
```

这比独立并行写 N 篇文章后硬拼起来更成熟。

## 9.12 为什么 Subtopic 当前是顺序执行

`_generate_subtopic_reports`：

```python
for subtopic in subtopics:
    result = await self._get_subtopic_report(subtopic)
```

没有 `asyncio.gather`。

乍看性能差，但这是有状态依赖：

```text
chapter 1
→ update global_written_sections / existing_headers
→ chapter 2 uses them
→ update
→ chapter 3 uses all previous
```

如果全部并发：

- 章节互相不知道对方写了什么；
- 重复内容增加；
- heading 冲突增加。

这是一个典型 **质量 vs 延迟** trade-off。

## 9.13 全局状态更新

每个 child 完成后：

```text
global_written_sections += extracted sections
global_context = child context
global_urls += child visited URLs
existing_headers += child headers
```

注意 `global_context` 会被更新成 child 的 context，而不是无限 append 所有历史。

这可以控制 context 膨胀，但也可能让早期 context 在后续可见性下降。

## 9.14 Context Hashability 兼容

`_hashable_context` 会把 dict context 转成 string：

```text
Title: ...
Content: ...
```

原因：MCP 或其它路径可能让 context item 是 dict，而 set 去重要求 hashable。

这又一次说明当前 Context type 还不够严格统一。

## 9.15 最终组装

```text
introduction
+
table_of_contents(report_body)
+
report_body
+
conclusion
+
references
```

Conclusion 是在 body 完成后生成，因此能总结真实成稿，而不是先猜结论。

## 9.16 Detailed vs Deep

| 维度 | Detailed Report | Deep Research |
|---|---|---|
| 目标 | 长文结构 | 研究深度 |
| 拆分依据 | subtopics / sections | evidence-driven follow-ups |
| child | subtopic researcher | branch researcher |
| 递归 | 否 | 是 |
| 执行 | 子主题顺序 | 同层并发 + 递归 |
| 共享重点 | written sections / headers | learnings / visited / citations |
| 输出 | 直接组装长报告 | 最终增强 context，再由 main writer 写 |

这两套机制不要混淆。

## 9.17 Detailed Report 是多 Agent 吗

可以叫 hierarchical multi-worker：

```text
main researcher
→ subtopic researcher 1
→ subtopic researcher 2
→ ...
```

但 child 之间没有直接通信，通信媒介是 parent 维护的共享状态：

- headers；
- written sections；
- URLs；
- context。

## 9.18 如果要并发怎么做

不能简单 `gather`。

可以两阶段：

### Phase A 并发 Research

```text
all subtopics research concurrently
→ evidence per subtopic
```

### Phase B 顺序 Writing

```text
section 1 write
→ section 2 sees section 1
→ section 3 sees 1+2
```

这样保留写作一致性，又降低搜索延迟。

这是很好的二次开发任务。

## 9.19 更强的去重

当前：

- existing header；
- embedding/keyword 检索 written content；
- Prompt 告诉模型避免重复。

可以加：

```text
new draft
→ semantic overlap checker
→ duplicate paragraph detector
→ rewrite/merge
```

## 9.20 面试题

**Q：为什么 Detailed Report 不直接让一个大模型一次写完？**

A：长报告需要分主题专项研究、分段生成和跨章节去重。拆分后每个子 Researcher 可以补充自己的证据，同时通过 existing headers 和 relevant written contents 保持整体一致性。

**Q：为什么子主题顺序执行？**

A：后一个章节依赖前面章节已经生成的 headers 和 written sections；并发写会丢失这种 coordination state。

**Q：Detailed 与 Deep Research 有什么差别？**

A：Detailed 优化输出结构；Deep 优化证据探索。前者按 subtopic 分工写长文，后者由 learnings/followups 驱动递归搜索。
