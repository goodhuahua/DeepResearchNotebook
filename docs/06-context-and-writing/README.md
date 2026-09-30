# 06 - Scraping、Context 与报告生成

> 🎯 **本章目标**：理解“从搜索结果到最终报告”中最容易被低估的中间层。你会看到网页是如何被读取、去重、过滤，以及为什么 Context Engineering 往往比“换更大的模型”更重要。

---

## 目录

- [6.1 从 URL 到正文](#61-从-url-到正文)
- [6.2 为什么要 visited_urls](#62-为什么要-visited_urls)
- [6.3 Scraper 并发](#63-scraper-并发)
- [6.4 为什么 Context 是真正的瓶颈](#64-为什么-context-是真正的瓶颈)
- [6.5 Context Filter 五种模式](#65-context-filter-五种模式)
- [6.6 keyword 为什么不是“低级方案”](#66-keyword-为什么不是低级方案)
- [6.7 embeddings 路径](#67-embeddings-路径)
- [6.8 为什么失败统一降级 keyword](#68-为什么失败统一降级-keyword)
- [6.9 ReportGenerator 如何写报告](#69-reportgenerator-如何写报告)
- [6.10 空证据为什么必须拒绝生成](#610-空证据为什么必须拒绝生成)
- [6.11 Citation 的现状与改进方向](#611-citation-的现状与改进方向)
- [6.12 面试高频题](#612-面试高频题)
- [6.13 本章总结](#613-本章总结)

---

## 6.1 从 URL 到正文

前一章结束时，我们有：

~~~text
new_search_urls
+
prefetched_content
~~~

然后：

~~~python
scraped_content = await (
    self.researcher.scraper_manager
    .browse_urls(new_search_urls)
)

scraped_content.extend(prefetched_content)
~~~

### 这一步做什么？

例如 Search 得到：

~~~text
https://langchain-ai.github.io/langgraph/...
~~~

Browser/Scraper 真正访问页面，拿回来：

~~~json
{
  "url": "...",
  "raw_content": "完整页面正文..."
}
~~~

之后才进入 Context Filter。

你可以把这三层记成：

~~~text
Retriever  = 找“书目”
Scraper    = 把“书”拿回来
Context    = 从书里划重点
~~~

---

## 6.2 为什么要 visited_urls

假设 Planner 生成：

~~~text
Q1: LangGraph state
Q2: LangGraph orchestration
Q3: LangGraph observability
~~~

三个 Search 都可能搜到同一个 Overview 页面。如果每次都抓，就会发生：

~~~text
同一个 URL
→ 下载 3 次
→ 解析 3 次
→ Context 里出现 3 份重复内容
~~~

所以 Researcher 保存：

~~~python
self.visited_urls = set()
~~~

它同时解决三个问题：**性能**上减少网络请求，**成本**上减少 Context token，**质量**上减少重复证据挤占真正不同的来源。

### 父子 Researcher 还能共享它

Detailed / Deep Research 创建 child 时，会把已有 `visited_urls` 传下去：

~~~text
父任务已经读过的来源
→ 子任务尽量不要再抓
~~~

这是一个非常轻量的“研究记忆”。

---

## 6.3 Scraper 并发

`BrowserManager` 内部持有 WorkerPool。当前配置里有：

~~~text
MAX_SCRAPER_WORKERS
SCRAPER_RATE_LIMIT_DELAY
~~~

这意味着系统至少有两层并发：

~~~text
Sub Query 并发
   ↓
每个 Sub Query 内
Scraper Worker 并发
~~~

为什么不能无限提高并发？因为外部网站和 API 都有 rate limit、connection limit、目标站资源压力和第三方 Scraper 配额。

所以 Agent 工程里的并发要问：

> “这个系统整体会发出多少外部请求？”

而不是只问：

> “asyncio.gather 能不能再多开几个？”

---

## 6.4 为什么 Context 是真正的瓶颈

做一个粗略计算：

~~~text
5 个 Sub Query
× 每个 5 个 Search Result
× 每页 10,000 字符
= 250,000 字符
~~~

你不应该把这 25 万字符全部塞给 Writer，否则会出现 token 成本增加、无关内容增加、模型注意力被稀释、重复证据增加，以及更明显的 “Lost in the Middle”。

所以必须做：

~~~text
Pages
  ↓
Context Selection
  ↓
少量高价值 Evidence
~~~

这也是为什么 Research Agent 的质量，很多时候并不是由“最终 Writer 用了多大的模型”单独决定，而是由**前面给它准备了什么 Context**决定。

---

## 6.5 Context Filter 五种模式

### 文件定位

`gpt_researcher/context/select.py`

支持：

~~~text
auto
jev
keyword
embeddings
none
~~~

### none

全部返回。适合内容本来就少、Debug 或做 baseline。

### Small-content fast path

即使配置了高级过滤，如果：

~~~text
total_chars < COMPRESSION_THRESHOLD
and
pages <= max_results
~~~

也会直接返回。

背后的思想非常实用：

> 小问题不要强行走复杂流程。

如果只有几页短文，再启动 Embedding 或外部 relevance service，收益可能小于开销。

### auto

当前固定源码快照里：

~~~text
有 TYPESAFE_API_KEY
→ jev

否则
→ keyword
~~~

一个很容易被忽略的事实是：**GPT Researcher 的默认运行并不一定需要 Embedding。**

### jev

使用外部 relevance scoring。失败时降级到 keyword。

### embeddings

使用 embedding similarity。Embedding 初始化、API 或相关依赖失败时，同样降级到 keyword。

---

## 6.6 keyword 为什么不是“低级方案”

很多初学者会下意识认为：

~~~text
Embedding 一定比 BM25 高级
~~~

但这不是工程结论。例如 Query：

~~~text
"LangGraph StateGraph checkpoint"
~~~

这里包含专有名词、API 名和类名，BM25 / lexical ranking 往往非常有效。

| 维度 | keyword/BM25 |
|---|---|
| 外部 API | 不需要 |
| Embedding Key | 不需要 |
| 延迟 | 低 |
| 成本 | 低 |
| 可解释性 | 高 |
| 专有名词匹配 | 很强 |

当前 `select.py` 的注释还明确提到：keyword threshold 是结合 `evals/context_filter` 选择的。

这说明一个重要原则：

> **Context 策略应该由 Eval 决定，而不是由“哪个技术听起来更高级”决定。**

---

## 6.7 embeddings 路径

如果选择 `embeddings`，大体流程是：

~~~text
Page
  ↓
Chunk
  ↓
Embedding
  ↓
Query Embedding
  ↓
Similarity Filter
  ↓
Relevant Chunks
~~~

典型参数包括 `chunk_size`、`chunk_overlap` 和 `similarity_threshold`。

### 为什么需要 overlap？

假设一句关键话刚好跨边界：

~~~text
Chunk A: "...LangGraph persists state through"
Chunk B: "checkpoints..."
~~~

没有 overlap 时，语义容易被切断。

### Lazy Embedding

`ContextManager` 传给 `select_context` 的不是立即构造好的 Embedding 对象，而是：

~~~python
embeddings=lambda:
    self.researcher.memory.get_embeddings()
~~~

只有真的走到 `embeddings` 分支时，才会调用它。

这避免了 keyword 用户被迫配置 Embedding Key、无谓模型初始化，以及启动阶段多一处依赖失败点。

---

## 6.8 为什么失败统一降级 keyword

考虑生产场景：

~~~text
Jev API down
Embedding key missing
Embedding package missing
~~~

如果这些错误让整次 Research 失败，一个“优化模块”就变成了单点故障。

keyword 的价值在于：

~~~text
local
cheap
few dependencies
good enough
~~~

因此它是一个很好的 availability baseline。

这和普通后端很像：

~~~text
Redis cache 失败
→ 回数据库

高级 reranker 失败
→ 回基础检索
~~~

Agent 系统一样需要“最低可用路径”。

---

## 6.9 ReportGenerator 如何写报告

### 文件定位

`gpt_researcher/skills/writer.py`

初始化时会准备：

~~~python
self.research_params = {
    "query": ...,
    "agent_role_prompt": ...,
    "report_type": ...,
    "report_source": ...,
    "tone": ...,
    "cfg": ...,
}
~~~

真正写作时再加入 Context、Custom Prompt、可用图片，以及 Subtopic 模式下的 existing headers 和 relevant written contents，然后：

~~~python
report = await generate_report(...)
~~~

### 为什么 Writer 单独存在？

因为“找证据”和“写文章”是两类能力：

~~~text
Research 阶段：
追求 coverage / evidence quality

Writing 阶段：
追求 structure / synthesis / readability
~~~

### 最终 Writer 用什么模型？

逻辑上使用 `SMART_LLM`。最终报告是用户真正看到的产物，这一阶段更值得使用高质量模型。

### Prompt Family

`generate_report()` 会根据 `report_type` 选择不同 Prompt builder，因此普通 Report 和 Subtopic Report 不是同一套 Prompt。

Subtopic Report 还会看到 main topic、已有 headers 和已写相关内容，这是 Detailed Report 降低章节重复的关键。

---

## 6.10 空证据为什么必须拒绝生成

`ReportGenerator.write_report()` 里有一个非常重要的 guard：

~~~text
if context is empty:
    do not write a confident report
~~~

假设 Search API 挂了、网站全部被 block、Retriever 全部空结果。如果 Writer 仍然收到空 Context，LLM 依然可能写出一篇流畅文章，但那已经不是 Research。

所以正确行为是：

> **Abstain：明确告诉用户没有拿到可靠 source material。**

这是比“永远给答案”更成熟的 Agent 行为。

---

## 6.11 Citation 的现状与改进方向

当前项目会保留 Source URL，并把 Context 以带 Source 的文本形式交给 Writer。这已经比“让模型自由生成参考文献”更可靠。

但仍有一个结构性问题：

~~~text
Context 最终大量被序列化成 string
~~~

于是很难严格保证：

~~~text
Claim A
→ Evidence Chunk 17
→ Source URL X
~~~

### 更理想的 Evidence 数据结构

二次开发时可以保持：

~~~python
class Evidence:
    source_url: str
    title: str
    text: str
    query: str
    relevance_score: float | None
    source_provider: str
~~~

一直到 Writer 之后，再做：

~~~text
Claim Extraction
→ Evidence Matching
→ Citation Verification
→ Unsupported Claim Repair
~~~

这会比“纯 Prompt 约束引用”更强。

---

## 6.12 面试高频题

### Q：GPT Researcher 是标准向量 RAG 吗？

> “不是。Web Research 先经过 Search 和 Scrape，再用可配置 Context Filter 选择证据。固定源码快照中的 auto 模式在没有 TypeSafe key 时默认走 keyword/BM25，Embedding 是可选策略，而不是系统必需前提。”

### Q：为什么 keyword 可以作为 fallback？

> “因为它不依赖外部模型和 API，availability 很高，而且对 API 名、实体名、专业术语等 lexical query 很有效，适合作为最低能力路径。”

### Q：为什么空 Context 要 abstain？

> “Research Agent 的承诺是 evidence-grounded generation。没有证据时继续生成会退化成参数记忆回答，甚至形成有引用外观的幻觉。”

---

## 6.13 本章总结

~~~text
Search Result
   ↓
URL / Prefetched Content
   ↓
Scraper
   ↓
Pages
   ↓
Context Filter
   ├─ keyword
   ├─ Jev
   ├─ embeddings
   └─ none
   ↓
Filtered Evidence
   ↓
ReportGenerator
   ↓
Final Report
~~~

下一章我们把三个“高级模式”放在一起比较：Deep、Detailed 和 Multi-Agent。

➡️ [07 - Deep / Detailed / Multi-Agent](../07-advanced-workflows/README.md)
