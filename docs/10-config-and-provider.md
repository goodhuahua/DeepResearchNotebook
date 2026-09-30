# 10 - 配置、LLM Provider 与可扩展性

> **本章目标**：理解 GPT Researcher 怎样把模型、Embedding、Retriever、Prompt Family 和运行参数从核心业务逻辑中解耦。

## 10.1 配置不是一个 .env 文件

`Config` 会组合：

```text
DEFAULT_CONFIG
→ custom JSON config
→ environment variables
```

环境变量优先级最高。

此外实例参数、headers、backend wrapper 还能继续覆盖部分行为。

所以实际是多层 Configuration Overlay。

## 10.2 默认配置

分析快照的关键默认：

```text
RETRIEVER = tavily
EMBEDDING = openai:text-embedding-3-small
CONTEXT_FILTER = auto

FAST_LLM = openai:gpt-5.4-mini
SMART_LLM = openai:gpt-5.4
STRATEGIC_LLM = openai:gpt-5.4

FAST_TOKEN_LIMIT = 6000
SMART_TOKEN_LIMIT = 12000
STRATEGIC_TOKEN_LIMIT = 8000

MAX_SEARCH_RESULTS_PER_QUERY = 5
MAX_ITERATIONS = 3
MAX_SCRAPER_WORKERS = 15
MAX_SUBTOPICS = 3

DEEP_RESEARCH_BREADTH = 3
DEEP_RESEARCH_DEPTH = 2
DEEP_RESEARCH_CONCURRENCY = 4

MCP_STRATEGY = fast
REASONING_EFFORT = medium
```

数值会变化，真正要学的是分层。

## 10.3 为什么要三类 LLM

### Fast LLM

目标：

- 成本敏感；
- 低复杂度；
- 可以快速失败/重试的节点。

### Smart LLM

目标：

- 最终报告；
- 高质量 synthesis；
- 长输出。

### Strategic LLM

目标：

- Planning；
- Query generation；
- Deep research extraction；
- reasoning-heavy tasks。

即使当前默认 Smart 和 Strategic 是同一个 model，也值得保留两个 logical slot。

因为生产环境可能：

```text
smart = best writing model
strategic = best reasoning model
```

## 10.4 provider:model 解析

配置格式：

```text
openai:gpt-...
anthropic:claude-...
ollama:...
```

`Config.parse_llm` 拆成：

```python
(provider, model)
```

核心 workflow 只拿两个字段，不关心 SDK。

## 10.5 GenericLLMProvider

调用链：

```text
create_chat_completion
→ get_llm(provider)
→ GenericLLMProvider.from_provider
→ ChatOpenAI / ChatAnthropic / ...
```

当前 supported provider 很多，包括：

- OpenAI
- Anthropic
- Azure OpenAI
- Google
- Bedrock
- Cohere
- Groq
- DeepSeek
- Ollama
- OpenRouter
- vLLM
- Mistral
- LiteLLM
- xAI 等

它本质是 LangChain chat model adapter factory。

## 10.6 Model Capability Adaptation

`create_chat_completion` 不是机械传全部参数。

例如某些模型：

```text
NO_SUPPORT_TEMPERATURE_MODELS
```

就不传自定义 temperature。

某些模型：

```text
SUPPORT_REASONING_EFFORT_MODELS
```

会注入 reasoning effort。

这是 Provider 抽象里的重要细节：

> 统一接口不等于所有模型能力完全一样。

需要 capability adaptation。

## 10.7 LLM_KWARGS

调用方可以传：

```text
LLM_KWARGS
```

它最后覆盖框架计算出来的 provider kwargs。

这相当于 escape hatch。

好处是新 Provider 参数不用立刻修改核心 Config schema。

坏处是类型安全弱，配置错误可能运行时才发现。

## 10.8 OpenAI-compatible Base URL

如果设置：

```text
OPENAI_BASE_URL
```

会传给 OpenAI adapter。

因此可以支持兼容接口服务。

这是很多企业私有部署会用到的扩展点。

## 10.9 Retry 属于统一 LLM 层

`create_chat_completion`：

```text
non-stream request
→ up to 10 attempts
→ exponential backoff capped around 8s

stream + websocket
→ 1 attempt
```

把重试放统一 LLM utility 的好处：

- 各业务 Action 不必重复；
- Provider failure policy 一致。

但某些 Action 还会在更高层做 semantic fallback，比如 Strategic → Smart。

这形成两层恢复：

```text
transport/provider retry
+
business-level model fallback
```

## 10.10 Cost Callback

LLM 成功后：

```text
calculate_llm_cost(...)
→ cost_callback(cost)
```

成本由 utility 层捕获，再交给 Researcher state。

这让业务函数不用自己解析 usage。

## 10.11 Embedding 配置

类似：

```text
provider:model
```

但 embedding 并不会总初始化。

Context Filter 走 lexical 时可以完全不用它。

这是配置层和 lazy dependency 的配合。

## 10.12 Environment 类型转换

`Config.convert_env_value` 会按 `BaseConfig` 类型转换：

- bool
- int
- float
- str
- list
- dict
- Optional

list/dict 使用 `json_repair`，对手工配置的轻微格式问题更宽容。

## 10.13 Deprecated Config

当前代码还兼容：

- `LLM_PROVIDER`
- `FAST_LLM_MODEL`
- `SMART_LLM_MODEL`
- `EMBEDDING_PROVIDER`

并发 warning。

这体现成熟开源项目常见的 backward compatibility burden。

## 10.14 Prompt Family 也是配置

`PROMPT_FAMILY` 可以选择提示词体系。

`GPTResearcher` 支持直接传 class/object，也可以从 config 解析。

因此“换 prompt”不是去 Action 内改字符串。

## 10.15 Retriever 也是配置

`RETRIEVER` 可多选。

而 headers 又能 request-level override。

这使 SaaS 场景可以：

```text
User A → Tavily
User B → Exa + OpenAlex
User C → MCP
```

而不重启进程。

## 10.16 配置隔离为什么重要

MCP 的历史实现曾经容易通过 process env 修改 Retriever。

当前代码明确改成 instance config。

因为：

```text
request-local decision
must not mutate
process-global configuration
```

在 async server 中尤其重要。

## 10.17 一个更严格的生产配置架构

建议分：

```text
StaticAppConfig
TenantConfig
RequestConfig
ExperimentConfig
RuntimeState
```

并明确 merge order。

同时使用 immutable model，避免 request 中途修改 cfg 造成难追踪行为。

## 10.18 Secrets

API key 应该和普通 config 分离：

```text
config value
≠ secret value
```

生产中通过 secret manager 注入，不写入日志和 trace。

## 10.19 Feature Capability Matrix

多模型系统最好显式维护：

| Capability | OpenAI X | Claude Y | Local Z |
|---|---|---|---|
| streaming | ✓ | ✓ | ✓ |
| reasoning effort | ✓ | - | - |
| temperature | restricted | restricted | ✓ |
| tool call | ✓ | ✓ | depends |
| max output | ... | ... | ... |

而不是靠散落的 model-name list 长期维护。

当前代码已经有 capability list 雏形。

## 10.20 面试题

**Q：为什么 Smart 和 Strategic 分开，即使默认同模型？**

A：职责不同，未来可以独立优化成本和能力；logical abstraction 不应该被当前 provider 配置绑死。

**Q：统一 Provider 最大难点是什么？**

A：不同模型能力不等价。参数、streaming、reasoning、token 语义都不同，所以除了 Adapter 还需要 capability-aware normalization。

**Q：为什么 MCP 配置不能改全局 env？**

A：Web server 多请求并发时会产生 cross-request state pollution，实例级配置更安全。
