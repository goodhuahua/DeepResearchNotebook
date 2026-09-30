# 13 - 源码走读地图：从入口函数一路追到外部世界

> **本章目标**：给出一张可以直接用于 IDE 跳转的源码地图。不是逐行翻译代码，而是标明“哪个文件最值得读、从哪个函数进入、下一跳是什么”。

## 13.1 第一优先级：必须能背出职责的文件

| 优先级 | 文件 | 核心对象/函数 | 一句话职责 |
|---|---|---|---|
| ★★★★★ | `gpt_researcher/agent.py` | `GPTResearcher` | Session 状态 + 总编排 |
| ★★★★★ | `skills/researcher.py` | `ResearchConductor` | 普通 Research 主流程 |
| ★★★★★ | `actions/query_processing.py` | `generate_sub_queries` | Planning / Query decomposition |
| ★★★★★ | `skills/context_manager.py` | `ContextManager` | Context 统一入口 |
| ★★★★★ | `context/select.py` | `select_context` | Context 策略路由 |
| ★★★★★ | `skills/writer.py` | `ReportGenerator` | Writer 业务层 |
| ★★★★★ | `actions/report_generation.py` | `generate_report` | 最终 LLM 写作 |
| ★★★★★ | `skills/deep_research.py` | `DeepResearchSkill` | 递归研究 |
| ★★★★☆ | `actions/retriever.py` | `get_retriever(s)` | Search Adapter Factory |
| ★★★★☆ | `skills/browser.py` | `BrowserManager` | Scraping 编排 |
| ★★★★☆ | `utils/llm.py` | `create_chat_completion` | LLM 统一调用/重试/成本 |
| ★★★★☆ | `config/config.py` | `Config` | 配置 Overlay |
| ★★★★☆ | `backend/report_type/detailed_report/*` | `DetailedReport` | 子主题协作 |
| ★★★☆☆ | `multi_agents/agents/*` | `ChiefEditorAgent` 等 | 独立 Multi-Agent 图 |

## 13.2 路线 A：最小 Basic Report

从：

```text
backend/report_type/basic_report/basic_report.py
```

看：

```python
async def run(self):
    await self.gpt_researcher.conduct_research()
    report = await self.gpt_researcher.write_report()
    return report
```

这是最干净的入口。

下一跳：

```text
gpt_researcher/agent.py
```

## 13.3 GPTResearcher.__init__

阅读时不要逐参数陷进去，先找 5 件事：

### 1. Config

```text
Config(config_path)
```

### 2. Retriever

```text
get_retrievers(...)
```

### 3. Skills

```text
ResearchConductor
ReportGenerator
ContextManager
BrowserManager
SourceCurator
DeepResearchSkill?
ImageGenerator
```

### 4. Shared State

```text
context
visited_urls
research_sources
research_images
cost
```

### 5. MCP

```text
_process_mcp_configs
_resolve_mcp_strategy
```

读完后应该能回答：一个 Researcher instance 里“状态”和“服务”分别有哪些。

## 13.4 GPTResearcher.conduct_research

按这张 call tree：

```text
conduct_research
├── _log_event(start)
├── deep?
│   └── _handle_deep_research
├── choose_agent
├── research_conductor.conduct_research
├── optional image_generator.plan_and_generate_images
└── return context
```

重点看：

- Deep 在哪分叉；
- role 什么时候生成；
- context 在哪被赋值；
- images 为什么先于 write report。

## 13.5 choose_agent

文件：

```text
actions/agent_creator.py
```

阅读顺序：

```text
choose_agent
→ create_chat_completion
→ json_repair / json
→ handle_json_error
→ _agent_pair_from_payload
→ extract_json_with_regex
→ default
```

这里是“LLM structured output engineering”的典型案例。

## 13.6 ResearchConductor.conduct_research

文件很长，先只搜：

```text
async def conduct_research
```

画出 source routing。

然后 Web 路径只跟：

```text
_get_context_by_web_search
```

不要先读下面所有辅助函数。

## 13.7 _get_context_by_web_search

这是普通模式最重要函数之一。

依次找：

```text
mcp strategy
initial search
plan_research
sub_queries
original query append
asyncio.gather
initial context preserve
join
```

读完后回答：

- Planning 的 search result 从哪来？
- Sub-query 是否并行？
- original query 为什么再跑？
- initial search 会不会完全浪费？

## 13.8 plan_research

跳：

```text
ResearchConductor.plan_research
→ plan_research_outline
→ generate_sub_queries
```

再进：

```text
actions/query_processing.py
```

## 13.9 query_processing.py

建议按：

```text
get_search_results
→ generate_sub_queries
→ _normalize_sub_queries
→ plan_research_outline
```

重点：

- `asyncio.to_thread`；
- strategic model；
- retry / smart fallback；
- MCP-only shortcut；
- normalize data contract。

## 13.10 _process_sub_query

回到 `skills/researcher.py`。

这是单个研究分支的核心：

```text
MCP path
+
_scrape_data_by_urls
+
ContextManager
+
_combine_mcp_and_web_context
```

把它画成一个独立函数盒子，你就理解了 fan-out 的 worker 做什么。

## 13.11 _search_relevant_source_urls

重点找：

```text
requires_scraping
prefetched_content
new_search_urls
visited_urls
random.shuffle
```

这是 Retriever 与 Browser 的数据契约边界。

## 13.12 _scrape_data_by_urls

非常简单：

```text
search URL/prefetched
→ BrowserManager.browse_urls
→ extend(prefetched)
→ optional vector_store.load
```

越简单的 orchestration function 越能看清边界。

## 13.13 BrowserManager

文件：

```text
skills/browser.py
```

读：

```text
__init__
browse_urls
select_top_images
```

下一跳：

```text
actions/web_scraping.py::scrape_urls
```

注意 BrowserManager 是 Skill，不是具体 HTML parser。

## 13.14 ContextManager

文件：

```text
skills/context_manager.py
```

主方法：

```text
get_similar_content_by_query
→ context.select.select_context
```

其它方法分别服务：

- vector store；
- previous written sections。

## 13.15 select_context

这一个函数建议逐行看。

决策树：

```text
resolve mode
→ none?
→ small?
→ jev?
→ embeddings?
→ keyword fallback
```

这段代码非常适合面试时白板讲。

## 13.16 compression.py

如果选择 embeddings，再读：

```text
ContextCompressor
→ RecursiveCharacterTextSplitter
→ EmbeddingsFilter
→ ContextualCompressionRetriever
```

如果你熟悉 RAG，这一层非常直观。

## 13.17 ReportGenerator

```text
skills/writer.py
```

读：

```text
write_report
→ empty context guard
→ report params
→ generate_report
```

Detailed 相关再看：

- get_subtopics
- get_draft_section_titles
- write_introduction
- write_report_conclusion

## 13.18 generate_report

`actions/report_generation.py`。

重点：

```text
get_prompt_by_report_type
→ subtopic/custom/default branch
→ image prompt injection
→ create_chat_completion(stream=True)
→ fallback message form
```

## 13.19 LLM 统一入口

`utils/llm.py::create_chat_completion`。

必须看：

- model None guard；
- max token guard；
- reasoning effort capability；
- temperature capability；
- LLM_KWARGS override；
- provider factory；
- retry；
- cost callback。

然后再追：

```text
llm_provider/generic/base.py
```

只需要理解 `from_provider` factory，不必把所有 Provider SDK 都背完。

## 13.20 Deep Research 源码路线

```text
GPTResearcher.conduct_research
→ _handle_deep_research
→ DeepResearchSkill.run
→ generate_research_plan
→ deep_research
→ generate_search_queries
→ process_query
    → nested GPTResearcher.conduct_research
    → process_research_results
→ recurse
→ trim context
```

一定要观察：

- nested GPTResearcher；
- semaphore；
- breadth shrink；
- depth - 1；
- all-failure stop。

## 13.21 Detailed Report 路线

```text
DetailedReport.run
→ _initial_research
→ _get_all_subtopics
→ write_introduction
→ _generate_subtopic_reports
    → _get_subtopic_report
        → child GPTResearcher
        → conduct_research
        → draft titles
        → similar written content
        → write_report
        → update global state
→ _construct_detailed_report
```

这里最重要的是“为什么顺序”。

## 13.22 Multi-Agent 路线

单独看：

```text
multi_agents/main.py
→ ChiefEditorAgent
→ init_research_team
→ StateGraph
```

Agents 目录包括：

- editor
- researcher
- writer
- reviewer
- reviser
- fact_checker
- publisher
- human
- visualizer

它是另一套显式角色图，不是 `GPTResearcher` Skills 的简单扩展。

## 13.23 测试反向读法

当一个实现看不懂为什么这么写，搜对应 test。

例如：

```text
test_sub_query_normalization
test_source_curator_json_parsing
test_url_security
test_websocket_manager
test_multi_agents_route_bindings
test_tavily_malformed
```

测试往往直接告诉你历史 bug 是什么。

## 13.24 IDE 阅读建议

第一轮只使用：

- Go to Definition
- Find References
- Call Hierarchy
- Search function name

不要一开始全局搜关键词“agent”。

## 13.25 最终源码心智图

```text
                  GPTResearcher
                       │
     ┌─────────────────┼────────────────────┐
     │                 │                    │
 ResearchConductor  ContextManager      ReportGenerator
     │                 │                    │
 query_processing   select_context       generate_report
     │                 │                    │
 Retrievers         lexical / jev /      LLM Provider
     │              embeddings
 BrowserManager
     │
 Scraper

DeepResearchSkill ── creates ──> GPTResearcher
DetailedReport    ── creates ──> GPTResearcher
```

如果这张图能在脑中稳定存在，剩下文件都只是细节填充。
