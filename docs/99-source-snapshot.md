# 99 - Source Snapshot 与参考范围

本文档用于固定本仓库分析的代码版本，防止上游更新后“文档描述”和“main 最新代码”混在一起。

## GPT Researcher

- Repository: [assafelovic/gpt-researcher](https://github.com/assafelovic/gpt-researcher)
- Branch: `main`
- Analyzed commit: [0957c301ed06c2a5857b834358c7227c739041d4](https://github.com/assafelovic/gpt-researcher/commit/0957c301ed06c2a5857b834358c7227c739041d4)
- Commit date: 2026-09-26

核心源码锚点：

- [gpt_researcher/agent.py](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/agent.py)
- [gpt_researcher/skills/researcher.py](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/skills/researcher.py)
- [gpt_researcher/skills/deep_research.py](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/skills/deep_research.py)
- [gpt_researcher/skills/context_manager.py](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/skills/context_manager.py)
- [gpt_researcher/context/select.py](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/context/select.py)
- [gpt_researcher/context/compression.py](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/context/compression.py)
- [gpt_researcher/skills/writer.py](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/skills/writer.py)
- [gpt_researcher/actions/query_processing.py](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/actions/query_processing.py)
- [gpt_researcher/actions/retriever.py](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/actions/retriever.py)
- [gpt_researcher/actions/report_generation.py](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/actions/report_generation.py)
- [gpt_researcher/utils/llm.py](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/utils/llm.py)
- [backend/report_type/detailed_report/detailed_report.py](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/report_type/detailed_report/detailed_report.py)
- [multi_agents/agents/orchestrator.py](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/multi_agents/agents/orchestrator.py)

## 文档写法参考 1：how-claude-code-works

- Repository: [Windy3f3f3f3f/how-claude-code-works](https://github.com/Windy3f3f3f3f/how-claude-code-works)
- 本次参考时最新提交：[f4d6505ed9162a0ee6be089190f74c419ecacb19](https://github.com/Windy3f3f3f3f/how-claude-code-works/commit/f4d6505ed9162a0ee6be089190f74c419ecacb19)
- 参考的是文档组织方法：机制拆章、Agent Loop 全景、源码函数定位、设计理由与恢复路径。

没有复制其 Claude Code 分析内容。

## 文档写法参考 2：learn-nanobot

- Repository: [bcefghj/learn-nanobot](https://github.com/bcefghj/learn-nanobot)
- 本次参考时最新提交：[740d041a52e1d50499aabc70ec5bd6ae592737d0](https://github.com/bcefghj/learn-nanobot/commit/740d041a52e1d50499aabc70ec5bd6ae592737d0)
- 参考的是求职导向结构：学习路线、架构深入、源码走读、面试问题、简历表达与实战路线。

没有复制其 Nanobot 分析内容。

## 关于上游内部文档

GPT Researcher 自身包含 `.claude/references/architecture.md` 和 `flows.md`。本仓库将其作为辅助索引，但**以固定提交的真实源码优先**。

原因是内部 reference 可能略有滞后。例如当前源码中的 MCP 处理刻意避免通过进程级 `os.environ` 修改 Retriever，以减少并发请求间配置污染；如果旧文档描述与实现不同，应以源码为准。

## 使用本仓库时的版本原则

如果未来上游发生重大改动，建议新建：

```text
snapshots/
  2026-09-26.md
  YYYY-MM-DD.md
```

并记录：

- upstream commit；
- changed core files；
- architecture delta；
- docs that need update。

不要直接把旧结论改成“永远正确”，因为 Agent 框架迭代很快。
