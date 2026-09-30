# 07 - Report Generation：研究证据怎样变成最终报告

> **本章目标**：理解 Writer 链路、Prompt Family、Report Type、图片注入、streaming、空 Context abstention，以及为什么“写报告”应该和“做研究”解耦。

## 7.1 Writer 的位置

主链路：

```text
ResearchConductor
→ researcher.context
→ GPTResearcher.write_report
→ ReportGenerator.write_report
→ generate_report
→ smart LLM
→ Markdown
```

Writer 不负责 Search。

## 7.2 为什么分离 Research 和 Writing

如果一个 Prompt 同时要求：

> 搜资料 + 判断来源 + 组织大纲 + 写 2000 字报告

模型承担太多职责。

拆开之后：

### Research 阶段优化 Recall

```text
find enough evidence
```

### Context 阶段优化 Precision

```text
keep relevant evidence
```

### Writing 阶段优化 Synthesis

```text
structure and communicate
```

每层可以独立测试。

## 7.3 GPTResearcher.write_report

它主要做：

- 标记当前 cost step = `report_writing`；
- 选择 internal/external context；
- 把预生成图片传给 Writer；
- 记录 report length；
- 返回字符串。

`ext_context` 让调用方可以绕过 internal context，复用 Writer。

## 7.4 ReportGenerator

`ReportGenerator` 初始化时缓存：

```text
query
agent_role_prompt
report_type
report_source
tone
websocket
cfg
headers
```

调用 `write_report` 时再加入：

- context
- custom_prompt
- available_images
- subtopic-specific state

这是一种“稳定参数 + 每次调用参数”的分离。

## 7.5 Empty Context Abstention

这是 Writer 最值得学习的 guard：

```text
if no research content:
    return "I could not gather any source material..."
```

为什么？

如果让 LLM 在 context 为空时继续生成，模型会依赖参数记忆，可能仍然写：

- 很流畅；
- 有看似可信结构；
- 甚至有“引用式表达”；

但它不再是 research report。

所以可靠系统需要允许“没有证据就不回答”。

## 7.6 Prompt by Report Type

`generate_report`：

```text
get_prompt_by_report_type(report_type, prompt_family)
```

不同报告类型不是只改一个标题，而是选择不同 Prompt 构造逻辑。

普通报告：

```text
query
+ context
+ report_source
+ report_format
+ tone
+ total_words
+ language
```

Subtopic：

```text
query
+ existing_headers
+ relevant_written_contents
+ main_topic
+ context
```

Subtopic Prompt 的额外信息是 Detailed Report 避免重复的关键。

## 7.7 Prompt Family

`prompt_family` 不是硬编码单个 Prompt 字符串。

它统一提供：

- auto agent instructions
- search query prompt
- report prompt
- introduction
- conclusion
- draft titles
- source curation
- pretty print docs
- local/web join

这相当于一个 Prompt Strategy / Theme。

好处：

- 可以整体替换一套 Prompt 体系；
- 不用在 Action 中散落巨大字符串；
- 便于 domain customization。

## 7.8 Agent Role 作为 System Message

最终写作：

```python
messages = [
    {"role": "system", "content": agent_role_prompt},
    {"role": "user", "content": report_prompt},
]
```

所以 role 决定“你是谁”，report prompt 决定“现在具体写什么”。

这比把所有规则揉进一个 user prompt 更清晰。

## 7.9 Smart LLM

Writer 用 `SMART_LLM`。

当前默认：

```text
openai:gpt-5.4
```

这是高 token、长输出、高质量综合节点。

这里比 Fast LLM 更值得花成本，因为它是最终用户可见结果。

## 7.10 Streaming

`generate_report` 调 `create_chat_completion` 时：

```text
stream=True
websocket=websocket
```

因此前端可以边生成边看到报告。

Streaming 带来的工程差异：

- 不宜轻易做自动 retry；
- 一旦用户已经看到部分 token，retry 可能重复内容；
- 成本统计要支持 streamed usage；
- 中断需要处理 partial output。

上游 `create_chat_completion` 也因此规定：

```text
stream + websocket
→ max_attempts = 1
```

而普通非流式调用最多可重试多次。

这是非常合理的语义区别。

## 7.11 Writer Retry

`generate_report` 第一种消息结构失败后，会 fallback：

```text
system: role
user: content
```

→

```text
user: role + content
```

这是兼容某些 Provider 对 system message 支持差异的一种后备路径。

## 7.12 图片注入

如果 `available_images` 存在，Writer Prompt 会追加：

```text
AVAILABLE IMAGES:
- Image 1: ![alt](url) - section hint
...
```

并要求模型在相关段落插入准确 Markdown。

注意它先过滤：

- 非 dict；
- 没有 URL；
- 缺少 metadata 的行。

避免一条坏图片记录中断报告。

## 7.13 Report Format 与输出控制

配置里有：

```text
TOTAL_WORDS
REPORT_FORMAT
LANGUAGE
TONE
```

这些都进入 Prompt，而不是后处理。

这属于 “generation-time format control”。

生产中可以再加：

- JSON schema；
- AST post-processing；
- Markdown validator；
- Citation checker。

## 7.14 Introduction / Conclusion 是独立节点

Detailed Report 会分别调用：

```text
write_introduction()
write_report_conclusion()
```

而不是让一个超长 Prompt 一次性生成整个 Detailed Report。

这样：

- 可并行/分阶段；
- 可独立重试；
- 内容更容易控制；
- Main body 已有后再写 conclusion，信息更完整。

## 7.15 Reference 不是完全由 LLM 自由生成

Detailed Report 最后：

```text
add_references(conclusion, visited_urls)
```

意味着系统保留自己的 visited URL state，用于构造 Reference。

这是比“让 LLM 凭上下文列参考文献”更可靠的方向。

不过 claim-to-source 精确绑定仍有改进空间。

## 7.16 Citation 的难点

当前 Context 最终常变成 string。

这使得 Writer 看到：

```text
Content...
Source...
```

但没有强制：

```text
claim_id -> evidence_chunk_id
```

因此可能出现：

- 引用存在但并不完全支持 claim；
- 多 source 混合后 provenance 弱化。

更强实现应做 Claim Verification。

## 7.17 可加入的 Citation Pipeline

```text
Draft report
→ extract factual claims
→ retrieve supporting evidence per claim
→ verifier model / NLI
→ attach citation
→ flag unsupported claims
→ final report
```

这是一个非常好的简历二次开发方向。

## 7.18 Writer 的成本归因

`GPTResearcher` 有：

```text
_current_step
step_costs
research_costs
```

报告写作前会：

```text
_current_step = "report_writing"
```

LLM cost callback 最终归到这个 step。

因此可以区分：

- agent selection 成本；
- research 成本；
- report writing 成本；
- deep research 成本。

这比只给一个 total cost 更适合优化。

## 7.19 一个更完整的生产 Writer

建议：

```text
Evidence Set
→ Outline Planner
→ Section Writers
→ Cross-section Dedup
→ Claim Verifier
→ Citation Resolver
→ Style Editor
→ Markdown/JSON Validator
→ Final
```

GPT Researcher Detailed Report 已经有其中一些雏形：

- subtopics；
- section generation；
- previous-written-content retrieval；
- conclusion；
- references。

## 7.20 面试题

**Q：为什么 ReportGenerator 不自己搜索？**

A：分离 retrieval 和 generation 能让两阶段独立优化、测试和复用。Writer 的 contract 是“给定 evidence context，合成报告”。

**Q：为什么空 Context 时不继续生成？**

A：研究报告的可靠性依赖证据。没有 source material 时继续生成会退化成参数记忆回答，还可能伪造可信感，所以应 abstain。

**Q：Streaming 为什么只尝试一次 LLM 请求？**

A：已经向用户发送部分 token 后自动 retry 会重复或冲突，无法像非流式调用一样无感重试。

**Q：Detailed Report 怎样减少重复？**

A：每个 Subtopic Writer 会拿到 existing headers 和通过 draft section titles 检索出的 relevant written contents，在 Prompt 层告诉模型哪些结构和内容已出现。
