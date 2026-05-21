# Draft: Kepler — AI Agent 技术知识图谱

## 项目概念
构建一个网站，以知识图谱形式追踪 AI Agent 技术的演进：
- 自动从网络收集历史技术演进数据并填充
- 每天定时监控多个网站，收集最新知识
- 用新知识替换/更新旧知识
- 核心引擎考虑使用 GitHub 上的 Graphiti

## 用户技术背景
熟悉：LLM API、KV Cache、Agent Loop、Tool Use、Reasoning、Planning、Skills、MCP、Memory、Subagent、Multi-Agent
深入理解：Prompt Engineering、Context Engineering、Harness Engineering

## 调研发现
- Graphiti: 26K⭐，Apache 2.0，Python，双时态知识图谱框架，非常适合追踪技术演变
- Graphiti 需要外部 Neo4j/FalkorDB 作为图数据库
- Graphiti 没有内置爬虫——需要自己搭爬虫+调度层
- Sentinel 是更完整的替代方案（内置 Firecrawl + 自我修复循环）
- 知识图谱可视化：可用 D3.js、Force Graph、Cytoscape 等

## 已确认决策
- **知识域**: AI Agent 大领域，具体子域由 LLM 自主判定和扩展
- **爬虫策略**: 由我推荐
- **前端技术栈**: 由我选择
- **LLM 提供商**: DeepSeek（Graphiti 默认 OpenAI，需要适配）
- **部署方式**: 快速打包部署到服务器，浏览器直接访问
- **规模预期**: 初期小规模，后续可扩展

## 已确认决策（续）
- **数据源（种子）**: Hacker News、arxiv、Hugging Face blog → 自动发现更多
- **核心引擎**: GitHub Graphiti（Neo4j 后端）
- **数据检索**: Horizon 评估完成（见下方）

## Horizon 调研结论
- 4.3k⭐, Python, MIT，AI 新闻聚合+摘要工具
- 支持 8 类数据源爬取（HN、RSS、Reddit、Telegram、Twitter、GitHub 等）
- 有 MCP Server，双语输出（中英文）
- **问题**: 是 CLI/Newsletter 工具，不是数据管道库。无持久化 DB、无 JSON 输出、无变更追踪
- **结论**: 不建议直接用作 Graphiti 摄入层。更好的方案是用 `feedparser` + `httpx` + 各平台 API 自建轻量爬虫层，Horizon 的 scraper 实现可作为参考

## 全部决策汇总

| 决策项 | 结论 |
|---|---|
| 知识域 | AI Agent（LLM 自主判定子域） |
| 核心引擎 | Graphiti + Neo4j / Kuzu |
| 数据检索 | 自建轻量爬虫层（feedparser + httpx） |
| 种子数据源 | HN, arxiv, Hugging Face blog → 自动发现 |
| 历史数据 | C 混合（先上线跑新，后台回溯） |
| 更新频率 | 每日一次 |
| LLM 提供商 | DeepSeek |
| 部署方式 | 快速打包，浏览器访问，小规模 |
| 前端技术栈 | 我来选 |
| 用户系统 | 无，公开访问 |
