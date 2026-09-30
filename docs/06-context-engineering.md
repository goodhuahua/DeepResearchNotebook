# 06 - Context Engineering：从海量网页里挑什么给 LLM

> **本章目标**：理解 Research Agent 真正决定答案质量的中间层——Context Selection。Retriever 找到“候选信息”，Context Engine 决定“哪些证据值得占用上下文窗口”。

## 6.1 Context 是 Research Agent 的瓶颈

假设 5 个 Sub-query，每个 5 个结果，每页 10k 字：

```text
5 × 5 × 10k = 250k characters+
```

直接塞给 LLM：

- 超上下文；
- 成本高；
- 相关信息被噪声淹没；
- 模型可能忽略中间内容；
- 重复证据很多。

所以必须做 Context Engineering。

## 6.2 统一入口

`ContextManager.get_similar_content_by_query`：

```text
query + pages
→ select_context(...)
→ compressed/relevant context string
```

`select_context` 是当前版本非常关键的策略路由。

## 6.3 五种 Context Filter

```text
auto
jev
keyword
embeddings
none
```

## 6.4 auto

逻辑：

```text
if TYPESAFE_API_KEY:
    jev
else:
    keyword
```

这意味着当前默认运行**并不强制依赖 Embedding**。

这是一个很重要的工程变化：过去很多 RAG 系统默认“必须 embedding”，但对一次性 Web Research，BM25/lexical 可能已经很有竞争力，而且零模型依赖。

## 6.5 none

```text
all pages
→ pretty_print_docs
```

适合：

- 内容很少；
- Debug；
- Benchmark baseline。

风险是 token 失控。

## 6.6 Small-content fast path

不管配置什么 mode，如果：

```text
total_chars < COMPRESSION_THRESHOLD
and
len(pages) <= max_results
```

就可能直接返回，不做昂贵 filtering。

当前默认 threshold 是 8000 字符。

这是一条典型成本优化：

> **不要为本来就很小的数据启动 embedding / relevance service。**

## 6.7 keyword 路径

使用 `LexicalContextCompressor`，核心思想接近 BM25/关键词相关性排序。

当前默认相关参数：

```text
KEYWORD_RELATIVE_THRESHOLD = 0.5
KEYWORD_MAX_RESULTS = 25
```

“relative threshold”不是固定 score，而是相对最佳 chunk 的比例。

优点：

- 本地；
- 快；
- 无 API key；
- 无 embedding model；
- 对实体、专业术语、关键词精确匹配很强。

缺点：

- 同义语义能力较弱；
- Query 与正文措辞差异大时可能漏掉。

## 6.8 embeddings 路径

`ContextCompressor`：

```text
Documents
→ RecursiveCharacterTextSplitter
   chunk_size = 1000
   overlap = 100
→ EmbeddingsFilter
   similarity_threshold
→ ContextualCompressionRetriever
→ relevant chunks
```

当前 `DEFAULT_CONFIG` 中 `SIMILARITY_THRESHOLD=0.42`；`ContextCompressor` 自己还有 env fallback，实际通过 cfg 传入时以配置为准。

## 6.9 为什么 chunk overlap=100

Chunk 边界可能把一句话或一个论证切断。

Overlap 能让：

```text
chunk A ending
<shared context>
chunk B beginning
```

减少边界信息损失。

代价是重复 token 和 embedding。

## 6.10 Jev 路径

当配置 `jev` 时，使用 TypeSafe Jev 做 chunk usefulness scoring。

如果 Jev 不可用：

```text
catch JevError
→ keyword
```

这体现了系统的 fallback-first 思维。

## 6.11 为什么所有高级路径都降级 keyword

因为 keyword：

- 无外部 API；
- 无 embedding credential；
- 本地可执行；
- 至少能返回“相关一些”的 Context。

因此它适合作为 availability baseline。

一个生产系统应该明确：

> **最低能力路径必须尽量无外部依赖。**

## 6.12 Lazy Embedding

调用 `select_context` 时传入的是：

```python
embeddings=lambda: self.researcher.memory.get_embeddings()
```

不是提前计算好的 embedding object。

只有真的选择 `embeddings` mode 时才调用。

这样：

- keyword 用户不需要 embedding API key；
- 启动更快；
- 少加载包；
- 少出 credential error。

这叫 dependency lazy materialization。

## 6.13 Vector Store 路径

如果报告源是现有 Vector Store：

```text
query
→ VectorStoreWrapper.asimilarity_search
→ docs
→ pretty_print
```

这和“先 Web search 再临时压缩”是不同 RAG 场景：

- Web Research：动态 corpus；
- Vector Store：已有 corpus。

## 6.14 Written Content Compression

Detailed Report 写子章节时还有另一种 context：

> 已经写过的章节。

它不是拿网页，而是：

```text
current subtopic
+ draft section titles
→ search previous written sections
→ relevant_written_contents
```

目的是告诉 Writer：

- 哪些观点已经写过；
- 避免重复；
- 保持跨章节一致。

如果 embedding 不可用，还会 fallback keyword ranking。

## 6.15 Context 是字符串还是结构化对象？

当前系统最终大量把 Context 格式化为字符串：

```text
Title: ...
Content: ...
Source: ...
```

好处：

- Prompt 构造简单；
- Provider 无关。

问题：

- provenance 难以严格维护；
- 去重只能靠文本/URL；
- 引用与 claim 的对应关系弱；
- 后续结构化 verification 较难。

## 6.16 更理想的 Evidence Object

二次开发可以保持到 Writer 前都使用：

```python
class Evidence:
    source_url: str
    title: str
    chunk: str
    query: str
    relevance_score: float
    source_type: str
    retrieved_at: datetime
```

Writer Prompt 最后一步才 serialize。

这样更适合做：

- Claim-Citation mapping；
- Dedup；
- Reranking；
- Source trust score；
- Evidence audit。

## 6.17 Context Selection 与 Summarization 的区别

Selection：

> 从原文中选相关片段。

Summarization：

> 用 LLM 重新生成压缩内容。

前者更保真、更便宜；后者压缩率高但可能丢细节或引入 hallucination。

当前主要 Context path 强调 selection/compression，而不是每页先做一次 LLM 摘要，这是成本上更健康的设计。

## 6.18 Context Window Budget

当前 Context Filter 有阈值和 top-k，但如果要生产化，可以显式做 Token Budget：

```text
planner budget
research context budget
writer reserved output budget
system prompt budget
citation metadata budget
```

然后：

```text
rank chunks
→ pack until token budget
→ preserve source diversity
```

而不是只按 max_results。

## 6.19 Source Diversity

只按 relevance 排名可能 top-10 都来自同一个网站。

改进：

```text
score = relevance
        + authority
        + recency
        + source diversity bonus
```

并设置：

```text
max_chunks_per_domain
max_chunks_per_document
```

这对 Deep Research 很重要。

## 6.20 Context Poisoning

网页可以故意堆 Query 关键词，提高 BM25/embedding 相似度。

因此 Context Filter 的“相关”不等于“可信”。

需要区分：

```text
relevance
credibility
freshness
independence
```

`SourceCurator` 是一个后续可选层，但默认并非全程强制。

## 6.21 Context Filter 为什么值得写进简历

因为它能体现你理解：

- RAG 不只是 Vector DB；
- context quality 比单纯 model size 更关键；
- embedding 不是唯一检索方式；
- latency / cost / availability 要一起设计；
- fallback 是生产系统的一等公民。

## 6.22 面试题

**Q：为什么默认 auto 不直接用 embeddings？**

A：当前 auto 有 Jev key 时用 Jev，否则用 keyword。本地 lexical ranking 没有模型和 credential 依赖，成本和可用性更好；只有显式选择 embeddings 才 lazy 初始化 embedding。

**Q：keyword 会不会比 embedding 差？**

A：不是绝对。实体名、术语、精确关键词场景 BM25 很强。语义改写场景 embedding 更有优势。正确做法是基于 eval，而不是把 embedding 当默认真理。

**Q：Context Engineering 与 RAG 的区别？**

A：RAG 强调 retrieval augmented generation；Context Engineering 更广，包括 retrieval、filtering、compression、budgeting、formatting、provenance、history selection 等所有进入模型上下文的控制。
