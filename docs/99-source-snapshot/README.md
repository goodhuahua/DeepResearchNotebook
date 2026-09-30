# 99 - Source Snapshot

> 本章记录这套学习笔记分析的**固定源码版本**。GPT Researcher 更新很快；为了让文档中的函数、配置和行为可以复现，正文不以“未来 main 分支可能是什么样”为依据。

---

## 1. GPT Researcher

- Repository: https://github.com/assafelovic/gpt-researcher
- Branch analyzed: `main`
- Commit: `0957c301ed06c2a5857b834358c7227c739041d4`
- Commit date: 2026-09-26
- Commit message: `Merge pull request #2173 from assafelovic/docs/homepage-restore-hero`

文档中的核心实现判断，均以这个提交的**当前源码**为优先依据。

---

## 2. 主要源码锚点

### 普通 Research

~~~text
gpt_researcher/agent.py
  GPTResearcher

gpt_researcher/skills/researcher.py
  ResearchConductor

gpt_researcher/actions/query_processing.py
  generate_sub_queries
  plan_research_outline

gpt_researcher/skills/browser.py
  BrowserManager

gpt_researcher/skills/context_manager.py
  ContextManager

gpt_researcher/context/select.py
  select_context

gpt_researcher/skills/writer.py
  ReportGenerator

gpt_researcher/actions/report_generation.py
  generate_report
~~~

### Deep Research

~~~text
gpt_researcher/skills/deep_research.py
  DeepResearchSkill
~~~

### Detailed Report

~~~text
backend/report_type/detailed_report/detailed_report.py
  DetailedReport
~~~

### Multi-Agent

~~~text
multi_agents/agents/orchestrator.py
  ChiefEditorAgent

multi_agents/agents/editor.py
multi_agents/agents/researcher.py
multi_agents/agents/writer.py
multi_agents/agents/fact_checker.py
~~~

### Provider / Config

~~~text
gpt_researcher/config/
gpt_researcher/llm_provider/
gpt_researcher/utils/llm.py
gpt_researcher/retrievers/
gpt_researcher/scraper/
~~~

---

## 3. 本文档使用的关键配置快照

在本次分析版本中，值得建立直觉的默认配置包括：

~~~text
RETRIEVER=tavily

EMBEDDING=openai:text-embedding-3-small
SIMILARITY_THRESHOLD=0.42
CONTEXT_FILTER=auto

FAST_LLM=openai:gpt-5.4-mini
SMART_LLM=openai:gpt-5.4
STRATEGIC_LLM=openai:gpt-5.4

FAST_TOKEN_LIMIT=6000
SMART_TOKEN_LIMIT=12000
STRATEGIC_TOKEN_LIMIT=8000

MAX_SEARCH_RESULTS_PER_QUERY=5
MAX_ITERATIONS=3
MAX_SCRAPER_WORKERS=15
MAX_SUBTOPICS=3
TOTAL_WORDS=1200

DEEP_RESEARCH_BREADTH=3
DEEP_RESEARCH_DEPTH=2
DEEP_RESEARCH_CONCURRENCY=4

MCP_STRATEGY=fast
REASONING_EFFORT=medium
~~~

> 默认配置会随上游变化。理解源码时应该关注“这个参数控制哪一层”，而不是死记数值。

---

## 4. 参考的文档写作方法

本仓库在**文档组织方式**上参考了两个优秀的源码学习项目。

### learn-nanobot

- Repository: https://github.com/bcefghj/learn-nanobot
- Reference commit used during rewrite: `740d041a52e1d50499aabc70ec5bd6ae592737d0`

主要借鉴：

~~~text
面向初学者的学习顺序
章节目标
概念 → 类比 → 源码
关键代码片段逐段解释
设计价值
面试问题
章节总结
下一章导航
~~~

没有复制其 Nanobot 技术内容。

### how-claude-code-works

- Repository: https://github.com/Windy3f3f3f3f/how-claude-code-works
- Reference commit: `f4d6505ed9162a0ee6be089190f74c419ecacb19`

主要借鉴：

~~~text
机制导向的源码解释
围绕主循环建立心智模型
不仅写“是什么”，也解释“为什么这样设计”
关注 recovery / context / tool 等工程机制
~~~

---

## 5. 为什么固定 Snapshot 很重要

假设未来上游把：

~~~text
ResearchConductor
~~~

改名、拆分或迁移到 Graph。

如果本笔记只写：

> “当前 main 分支里是这样。”

几个月后读者就无法判断：

- 是文档错了；
- 还是上游变了。

固定 Commit 后，结论就变成：

> “在 `0957c3...` 这个版本，普通 Web Research 的核心路径是这样。”

这是源码分析应该具备的可复现性。

---

## 6. 源码与内部参考文档冲突时怎么办

GPT Researcher 仓库内部有一些面向开发工具的 reference / documentation 文件。

这些材料可以用于：

- 快速找到模块；
- 理解维护者预期；
- 确认术语。

但如果出现：

~~~text
Reference 文档描述 A
当前源码实现 B
~~~

本仓库采用：

> **当前固定 Commit 的可执行源码优先。**

例如 MCP 配置处理的实现细节，就应以当前 `agent.py` / `researcher.py` 为准，而不是沿用旧说明。

---

## 7. 推荐源码验证方式

读到任何关键结论时，都建议做三步：

~~~text
1. 找入口函数
2. 向下追实际调用
3. 搜对应 Tests
~~~

例如想验证：

> “Initial Search 结果会不会被后续搜索覆盖而丢失？”

不要只看注释。

应该同时看：

~~~text
ResearchConductor._get_context_by_web_search

_get_context_from_initial_results

tests/test_planning_sources.py
~~~

这样才能理解一个行为为什么存在。

---

## 8. 文档维护规则

未来如果更新到新的 GPT Researcher Commit，建议：

1. 先更新本文件的 Commit；
2. 跑一遍第 04 章主调用链；
3. 检查第 05～08 章的机制是否变化；
4. 搜 Tests 中是否新增相关回归用例；
5. 最后再修改正文。

不要只根据上游 README 改本仓库的源码分析。

---

返回学习入口：

➡️ [DeepResearchNotebook README](../../README.md)
