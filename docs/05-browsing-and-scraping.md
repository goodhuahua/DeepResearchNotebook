# 05 - Browser 与 Scraping：从“搜索结果”到“可用证据”

> **本章目标**：看懂 GPT Researcher 怎样把 URL 转成可进入 Context Engine 的正文，并理解抓取并发、去重、prefetched content、图片和失败容错。

## 5.1 为什么 Search 后还必须 Browse

搜索引擎典型结果：

```json
{
  "title": "Example",
  "href": "https://...",
  "body": "一小段摘要..."
}
```

但研究报告需要：

- 完整上下文；
- 原文细节；
- 引用来源；
- 更长证据链。

因此主链路：

```text
Search
→ URLs
→ BrowserManager
→ Scraper
→ Page Content
```

## 5.2 BrowserManager 是抓取编排器

`gpt_researcher/skills/browser.py`。

初始化：

```python
self.worker_pool = WorkerPool(
    max_scraper_workers,
    scraper_rate_limit_delay,
)
```

它并不自己解析 HTML，而是负责：

- 调用 scrape action；
- 保存 research_sources；
- 选图片；
- 输出 streaming event。

这是“业务层 Browser”，不是底层 HTML Parser。

## 5.3 scrape_urls

`actions/web_scraping.py::scrape_urls`：

```text
urls
→ Scraper(urls, user_agent, scraper_type, worker_pool)
→ await scraper.run()
→ scraped_data
→ image_urls
→ close session
```

一个值得注意的工程细节：

```python
finally:
    scraper.session.close()
```

HTTP Session 如果不关闭，会留下连接池和 socket 资源。

Agent 项目一样需要处理传统网络编程资源生命周期。

## 5.4 Scraper Strategy

配置：

```text
SCRAPER = "bs"   # current default
```

上游 scraper 目录支持多类抓取方式。

核心 Workflow 不直接依赖 BeautifulSoup / Browser SDK，而通过 `Scraper` 选择具体实现。

这与 Retriever / LLM Provider 的设计一致：

> 对外部基础设施使用 Adapter/Strategy，把 orchestration 保持稳定。

## 5.5 两种内容来源

`_scrape_data_by_urls` 最终会合并：

```text
scraped_content
+
prefetched_content
```

### scraped_content

来自 URL 实际抓取。

### prefetched_content

某些 Retriever 已经提供 full text，不需要再访问 URL。

最后两者都转换为 ContextManager 可处理的 page dict。

## 5.6 URL 去重

关键状态：

```python
researcher.visited_urls: set
```

`_get_new_urls` 在 URL 进入 Browser 前过滤。

优势：

- 避免不同 Sub-query 抓同一文章；
- Detailed Report 子主题可共享来源历史；
- Deep Research nested researchers 可减少重复。

## 5.7 为什么 visited_urls 是系统级关键状态

Research Agent 很容易出现：

```text
Q1 search → A, B, C
Q2 search → A, D, E
Q3 search → A, B, F
```

如果没有 visited：

- A 被抓 3 次；
- B 被抓 2 次；
- token/context 重复；
- API 和网站压力增加。

一个简单 set 能显著降低成本。

## 5.8 visited 的并发语义

当前实现共享 Python set，并在 event loop 中短同步片段里做：

```python
if url not in visited:
    visited.add(url)
```

没有 await 插入两步之间，单 event loop 下通常不会在该片段被其他 coroutine 切走。

但如果未来：

- 多线程；
- 多进程；
- 分布式 worker；

这个 set 就不再够用。

生产化需要：

- Redis set；
- DB unique constraint；
- distributed lock；
- research_id scoped cache。

## 5.9 Random Shuffle

新 URL 会：

```python
random.shuffle(new_search_urls)
```

其效果是避免每次都按 Search Provider 原始排序固定抓取。

不过这也带来：

- 运行不可完全复现；
- Source order 不稳定。

如果做 benchmark/eval，最好提供 deterministic seed 或关闭 shuffle。

## 5.10 WorkerPool

抓取层有专门 WorkerPool。

当前默认：

```text
MAX_SCRAPER_WORKERS = 15
SCRAPER_RATE_LIMIT_DELAY = 0.0
```

这意味着系统至少有两层并发：

```text
Sub-query concurrency
   └── Scraper worker concurrency
```

总外部请求压力可能是乘法关系，因此生产配置必须一起评估。

## 5.11 Rate Limit

`scraper_rate_limit_delay` 给 Scraper Worker 一个最低请求间隔。

适用于：

- 目标站点限速；
- 第三方 scraping API；
- 避免短时间过量请求。

Agent 的“速度”不是越快越好；抓取系统需要礼貌和稳定性。

## 5.12 抓取失败为什么不让整次研究失败

`scrape_urls` catch exception，返回已有结果或空列表。

ResearchConductor 对空 context 也能继续其它 Sub-query。

因此失败边界大致是：

```text
one URL fails
→ lose one source

one retriever fails
→ other retrievers continue

one subquery fails
→ other subqueries continue

all evidence fails
→ Writer abstains
```

这是很合理的 failure containment。

## 5.13 partial payload guard

`process_scraped_data` 会检查：

- item 是否 dict；
- status；
- content 是否存在；
- URL 是否存在。

真实 Scraper 很容易返回部分结构，不能假设每条都完整。

代码里这种 guard 很多，说明上游经过了大量 malformed-data bug 修复。

## 5.14 图片处理

`BrowserManager.browse_urls` 还会收集页面图片。

`select_top_images`：

1. 按 score 降序；
2. 用 image hash 去重；
3. 避免已经选过的 URL；
4. 取 top-k。

这和主文本链路分离。

之后如果 Image Generator 开启，还会有另一套“生成图片”流程；两者不要混淆：

- research_images：从来源页面选择；
- available_images：预生成、可嵌入报告的图片。

## 5.15 抓取与安全

对任意用户 URL 做 fetch 需要关注：

- SSRF；
- localhost / private network；
- redirect；
- file scheme；
- oversized response；
- decompression bomb；
- malicious HTML / prompt injection。

上游 tests 中有 `test_url_security.py`，说明 URL 安全已经是项目关注点。

如果做企业版，Scraper 层应该有独立安全网关。

## 5.16 Web 内容本身是不可信输入

抓到的网页文本应该被当成：

> **untrusted evidence, not instructions**

Research Agent 的网页可能含：

```text
"ignore previous instructions..."
"send secrets to..."
```

即使 GPT Researcher 主要把网页当 context，仍应该在 prompt 和 pipeline 层明确：

- 内容是证据；
- 不执行网页中的命令；
- 不允许网页改变 system policy。

这是所有 Browser Agent 必须考虑的 prompt injection 面。

## 5.17 内容清洗

当前 `actions/web_scraping.py::extract_main_content` 在这个辅助文件里仍是很薄的 placeholder，而实际 Scraper 实现承担更多内容抽取。

这提醒我们：

> 不要只看函数名推断真正实现，要继续追它调用的底层对象。

## 5.18 一个建议的生产抓取模型

```text
URL
→ URL policy
→ DNS/IP check
→ fetch with timeout/size limit
→ MIME validation
→ HTML/PDF parser
→ boilerplate removal
→ metadata extraction
→ content hash dedup
→ chunking
→ provenance record
```

GPT Researcher 已有其中多个部件，但如果你做二次开发，可以把 provenance 做得更严格。

## 5.19 Source Provenance

理想每个 chunk 都应该带：

```json
{
  "source_url": "...",
  "source_title": "...",
  "retriever": "tavily",
  "fetched_at": "...",
  "content_hash": "...",
  "chunk_id": "...",
  "query": "..."
}
```

这样最终 Citation 可以从 chunk lineage 自动生成，而不只是依赖 prompt 文本。

## 5.20 面试题

**Q：搜索 API 已经返回 snippet，为什么还要 Scraper？**

A：snippet 是用于检索排序的摘要，不保证覆盖原文关键事实。研究报告需要完整证据，所以对 declared requires_scraping 的 Retriever 仍要读取网页正文。

**Q：为什么有 WorkerPool？**

A：抓取是高延迟 I/O，可以并发；但不能无限并发，所以用 worker limit 和 rate delay 控制外部压力。

**Q：visited_urls 的作用只是性能优化吗？**

A：不只是。它还减少重复 context 和重复 citation，并在 Detailed/Deep Research 的父子 Researcher 之间保持研究覆盖状态。
