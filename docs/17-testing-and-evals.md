# 17 - Tests 与 Evals：从历史 Bug 反推 Agent 工程问题

> **本章目标**：通过测试目录理解一个成熟 Agent 项目最常坏在哪里，并区分传统软件测试和 LLM Evaluation。

## 17.1 为什么读 tests 很重要

源码告诉你：

> 现在怎么实现。

测试经常告诉你：

> 以前哪里坏过，作者为什么加这段奇怪的 guard。

GPT Researcher 的 tests 中大量覆盖：

- malformed Provider payload；
- null 字段；
- JSON parser；
- Web loader；
- URL security；
- Config；
- MCP；
- Multi-Agent routing；
- token/cost guard。

这比只读 happy path 更接近生产现实。

## 17.2 LLM Parser Tests

典型：

```text
test_sub_query_normalization.py
test_source_curator_json_parsing.py
```

说明核心风险：

```text
Prompt asks for schema
≠
Model always returns schema
```

测试应该覆盖：

- list；
- wrapper dict；
- string；
- fenced JSON；
- malformed JSON；
- empty；
- wrong item type。

## 17.3 Search Provider Malformed Tests

可看到类似：

```text
test_tavily_malformed
test_tavily_non_dict_response
test_serpapi_malformed_results
test_serper_non_dict_payload
test_xquik_null_fields
```

意义：

> 第三方 API 即使 HTTP 成功，payload 也可能和你假设的不一样。

Retriever boundary 应该做 schema normalization。

## 17.4 Scraper / Web Loader Tests

例如：

```text
test_web_base_loader_docs_guard
test_web_base_loader_enrichment_guard
```

需要覆盖：

- document None；
- metadata 缺失；
- extraction partial；
- encoding；
- unsupported content。

## 17.5 URL Security

`test_url_security.py` 提醒：

Browser Agent 不只是功能问题，还有 SSRF / network policy。

测试至少包括：

- localhost；
- private IP；
- weird scheme；
- redirect；
- encoded host。

## 17.6 Config Tests

多层环境变量配置很容易出问题：

- bool 字符串；
- Optional；
- list JSON；
- malformed list；
- deprecated var；
- override precedence。

Config parser 应该被当核心模块测试。

## 17.7 Multi-Agent Route Tests

`test_multi_agents_route_bindings.py` 说明图工作流最容易出现：

- node 没绑定；
- conditional edge 错；
- method rename 后 graph 仍引用旧名字。

图 Agent 应该测试“graph topology”，而不只是单节点函数。

## 17.8 Traditional Test vs LLM Eval

### Test

确定性 contract：

```text
input
→ expected invariant
```

例如：

> invalid sub-query response 最终必须返回 list[str]。

### Eval

统计性质量：

```text
dataset
→ run system
→ quality metrics
```

例如：

> keyword context filter 的 evidence recall 是否优于 baseline。

两者不能互相替代。

## 17.9 Context Filter Eval

上游 `context/select.py` 注释直接指向：

```text
evals/context_filter
```

并说明 keyword threshold 是由 eval 选择的。

这是很好的工程习惯：

> 算法参数由 benchmark 选择，而不是凭感觉。

## 17.10 Research Quality Eval

上游还有 quality eval / benchmark 脚本。

Research Report 的质量可能评价：

- factuality；
- completeness；
- citation；
- style；
- judge model score。

需要警惕 judge model bias，因此最好混合：

- LLM judge；
- deterministic metrics；
- human spot-check。

## 17.11 你自己的 Eval Dataset

不要只测 3 个 Query。

建议 50 条，分层：

```text
10 factual recent
10 comparison
10 long-tail technical
10 academic
10 ambiguous/multi-hop
```

保存：

- expected dimensions；
- gold sources（如果可做）；
- freshness requirement；
- reference answer key facts。

## 17.12 Context Eval

最容易做出可信数字：

给定固定 pages：

```text
query
+ candidate chunks
+ gold relevant chunks
```

比较：

- BM25；
- embeddings；
- hybrid；
- reranker。

指标：

```text
Recall@5
Recall@10
MRR
context token count
latency
cost
```

## 17.13 Citation Eval

提取最终 claims：

```text
claim
→ cited evidence
```

人工或 verifier 判断：

- citation exists；
- citation supports claim；
- source accessible；
- source trustworthy enough。

指标：

```text
Citation Precision
Citation Coverage
Unsupported Claim Rate
```

## 17.14 Planning Eval

Query decomposition 也能测：

### Coverage

是否覆盖 gold dimensions。

### Redundancy

不同 Sub-query 是否重复。

### Searchability

Query 是否能返回 relevant sources。

### Cost

是否生成太多无效 Query。

## 17.15 Deep Research Eval

不要只看 final report 更长。

测：

```text
depth 1 vs depth 2 vs adaptive
```

比较：

- new source rate；
- evidence novelty；
- quality gain；
- cost multiplier；
- latency multiplier。

如果 depth 2 多花 3 倍成本只提升 2% quality，就不划算。

## 17.16 Regression Corpus

每次发现失败案例都保存：

```json
{
  "query": "...",
  "failure_type": "bad_json | no_source | duplicate | ...",
  "expected_invariant": "...",
  "fixture": "..."
}
```

Agent 产品非常需要“失败案例库”。

## 17.17 Mock 的边界

Unit test 不应该真的打 LLM/Search。

Mock：

- Provider response；
- Retriever；
- Scraper。

但 Integration/Eval 必须定期打真实系统，否则 mock 只能证明你的假世界正确。

## 17.18 Async Tests

特别测试：

- semaphore max active count；
- gather partial failure；
- cancellation；
- lock only-one initialization；
- no session leak；
- shared budget race。

Agent 的并发 bug 往往不在单元 happy path 出现。

## 17.19 适合简历的 Eval 成果

比“准确率提升”更可信的表达：

> 建立 50 条 Research Query benchmark，分别评估 Query Coverage、Evidence Recall@10、Citation Precision、平均成本与 P95 延迟；对 keyword / embedding / hybrid Context Filter 做消融，并基于结果调整 Context packing 策略。

前提：你真的做了并能给 repo 链接。

## 17.20 最后一个原则

> **Agent 的质量不是 Prompt 的感觉，而是可以分 stage 测量的系统性质。**

当你能同时写出 unit test、integration test 和 evaluation，才真正进入 Agent 工程，而不是 Prompt Demo。
