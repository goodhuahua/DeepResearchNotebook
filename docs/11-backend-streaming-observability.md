# 11 - Backend、Streaming 与可观测性

> **本章目标**：从“库”走向“产品”：研究过程怎样通过 FastAPI / WebSocket 暴露，为什么系统大量发送 stage event，以及如何从现有基础继续构建 Agent Trace。

## 11.1 两种使用形态

GPT Researcher 可以当：

### Python Library

```python
researcher = GPTResearcher(...)
await researcher.conduct_research()
await researcher.write_report()
```

### Application Backend

```text
REST / WebSocket
→ run_agent
→ report type routing
→ stream progress/report
```

这两种模式共享核心 Agent。

## 11.2 Report Type Routing

Backend `run_agent` 会根据 report type：

```text
multi_agents
→ multi-agent workflow

detailed_report
→ DetailedReport

else
→ BasicReport
```

因此 Backend 是 composition root，而不是把所有模式塞进 `GPTResearcher`。

## 11.3 WebSocket 为什么重要

Research 是长任务。

如果只用普通 HTTP：

```text
request
...
30-120s
...
response
```

用户不知道：

- 是否卡住；
- 正在搜什么；
- 是否抓到来源；
- 报告写到哪里。

WebSocket 可以不断发送：

```text
logs
images
report tokens
progress
completion
```

改善 perceived latency。

## 11.4 stream_output

核心模块大量调用：

```text
stream_output(
    category,
    event_name,
    message,
    websocket,
    ...
)
```

例如：

- subqueries
- running_subquery_research
- scraping_urls
- scraping_complete
- context_combined
- writing_report
- report_written
- deep research progress

这其实已经形成一个弱事件模型。

## 11.5 Streaming 不只是 UI 功能

它同时提供：

### Execution Visibility

观察卡在哪个 stage。

### Progress Feedback

让前端知道任务还活着。

### Debug Evidence

可以复盘 Sub-query 和来源数量。

### Cost UX

研究结束时可以显示成本。

## 11.6 LLM Streaming

Writer 调：

```text
create_chat_completion(stream=True, websocket=...)
```

Provider 逐 token/逐 chunk 输出。

对 OpenAI adapter 当前还开启 stream usage，以便 streaming 时也拿 usage metadata 做成本统计。

## 11.7 Streaming Retry 的限制

统一 LLM utility：

```text
stream + websocket → max_attempts = 1
```

原因：

如果已经发送：

```text
"The main finding is ..."
```

然后 provider 断线，自动从头 retry：

```text
"The main finding is ..."
```

前端会看到重复。

所以 retry 必须知道“输出是否已经 commit 给用户”。

这是所有 streaming agent 的通用难题。

## 11.8 Logging

`GPTResearcher` 可以接收 `log_handler`。

执行阶段会记录：

- research start；
- chosen agent；
- context size；
- report length；
- errors；
- cost。

Deep Research 还会记录执行时间和 research cost。

## 11.9 Step Cost

Researcher 有：

```text
research_costs
step_costs
_current_step
```

典型 step：

- agent_selection
- research
- report_writing

`add_costs`：

```text
total += cost
step_costs[current_step] += cost
```

这是最小可用成本归因。

## 11.10 为什么成本必须是 Trace 的一部分

Research Agent 的一次请求可能调用：

- Planner LLM 2 次；
- 5 × Search API；
- 20 × web fetch；
- Context API；
- Writer LLM；
- Deep 模式若干 nested LLM。

只看 total token 无法优化。

理想 Trace：

```text
research_id
└── agent_selection  $...
└── planning         $...
└── q1
    ├── search       ...
    ├── scrape       ...
    └── context      ...
└── q2 ...
└── writing          $...
```

## 11.11 Multi-Agent 的额外 Trace

`multi_agents/main.py` 支持：

- LangSmith：存在 `LANGCHAIN_API_KEY` 时打开 tracing；
- 可选 Monocle tracing，通过 `MONOCLE_TRACING` 和 exporter 配置。

这条路径比单 Agent 核心更依赖图工作流，因此现成 tracing 更重要。

## 11.12 Research ID

不同 wrapper 会生成 research ID，用于：

- 日志；
- 输出；
- 图片；
- 状态隔离。

理想情况下所有：

```text
logs
LLM calls
searches
scrapes
cost
artifacts
```

都应该关联同一个 research_id。

## 11.13 当前 event 的不足

大量 event 是：

```text
event_name + human-readable message
```

对 UI 很好，但对机器分析不够。

建议同时发送结构化 payload：

```json
{
  "trace_id": "...",
  "span_id": "...",
  "stage": "retrieval",
  "event": "search_completed",
  "query": "...",
  "provider": "tavily",
  "result_count": 5,
  "latency_ms": 832,
  "cost_usd": 0.001
}
```

## 11.14 Trace 应该围绕 Stage，而不是 print

推荐 Span：

```text
research
├── choose_agent
├── plan
├── subquery[q1]
│   ├── retrieve[tavily]
│   ├── retrieve[exa]
│   ├── scrape
│   └── context_filter
├── subquery[q2]
└── write_report
```

这样能回答：

- 哪个 Provider 慢？
- 哪个 Sub-query 没有证据？
- 哪次 retry 最多？
- 哪一层花钱最多？
- 哪个 source 最终进入 report？

## 11.15 Streaming Event 与 Trace Event 应分离

UI event：

> “🌐 Scraping 5 URLs...”

Trace event：

```json
{
  "operation": "scrape",
  "url_count": 5
}
```

不要让监控系统解析 emoji 字符串。

## 11.16 Cancellation

长 research 应支持：

```text
user cancels
→ cancel research task
→ cancel child tasks
→ close HTTP sessions
→ stop streaming
→ persist partial trace
```

如果做自己的版本，这是后端非常值得补的一项。

## 11.17 Persistence / Replay

Agent debug 很依赖“可重放”。

建议记录：

- request config snapshot；
- Planner raw output；
- normalized subqueries；
- Search Result IDs；
- selected context chunks；
- Writer prompt hash；
- model/provider metadata；
- output。

之后可以做 offline replay，不用每次重新搜全网。

## 11.18 Privacy

Trace 很容易泄露：

- API keys（不应该）；
- 用户私有文档；
- Search Query；
-网页正文；
- Prompt；
- 模型输出。

所以日志必须：

- secret redaction；
- data classification；
- retention policy；
- tenant isolation。

## 11.19 面试题

**Q：为什么 Research Agent 特别需要可观测性？**

A：一次回答跨多个外部系统和异步分支，最终质量问题可能来自 Planner、Search、Scraper、Context 或 Writer。没有 stage-level trace，只看到最终坏答案很难定位。

**Q：Streaming 为什么会影响 retry 设计？**

A：partial output 已经对用户可见，重试不是幂等的。要么不自动 retry，要么实现 resume/dedup 协议。

**Q：你会重点监控哪些指标？**

A：end-to-end latency、各 stage latency、search success rate、scrape success rate、context size、source count、LLM retries、cost per stage、unsupported claim rate。
