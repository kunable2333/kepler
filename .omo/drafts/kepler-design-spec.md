# Kepler — AI Agent 技术演进知识图谱：设计文档

> 2026-05-21 | 版本 1.0

## TL;DR

构建一个自动追踪 AI Agent 技术演进的知识图谱网站。系统每日自动爬取 Hacker News、arxiv、Hugging Face Blog，由 DeepSeek 驱动 Graphiti 自动构建时序知识图谱，浏览器访问即可交互式探索技术演变脉络。

---

## 1. 项目目标

### 核心目标

让用户通过可视化知识图谱，清晰了解 AI Agent 领域的技术演进、前沿发展和知识边界。

### 定义完成

- [ ] 网站可公开访问，浏览器打开即可浏览知识图谱
- [ ] 每日自动爬取 HN、arxiv、HF Blog 内容并摄入图谱
- [ ] 图谱支持搜索、时间线、子图探索三种交互
- [ ] 历史数据后台异步回溯（C 混合策略）
- [ ] Docker 一键部署，单容器运行

### Must Have

- Graphiti 时序知识图谱（实体提取、关系解析、双时态追踪）
- 三个种子数据源（HN、arxiv、HF Blog）的自动爬取
- 图谱可视化（力导向图 + 时间线 + 搜索）
- DeepSeek 作为 LLM 和 embedding 提供商
- 每日定时更新
- Docker 单容器部署

### Must NOT Have

- 用户注册/登录系统（公开访问）
- 评论/社交功能
- 个性化推荐
- 多语言接口（初期仅中文）
- 实时更新（每日批次即可）
- 过度抽象的微服务架构（单进程运行）

---

## 2. 架构设计

### 整体架构

```
┌─────────────────── 单 Docker 容器 ───────────────────┐
│                                                        │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────┐  │
│  │ 爬虫层    │   │ Graphiti │   │   FastAPI Web    │  │
│  │          │   │          │   │  ┌────────────┐  │  │
│  │ HN API   │──▶│ 实体提取  │──▶│  │ REST API   │  │  │
│  │ arxiv RSS│   │ 关系解析  │   │  │ /api/search │  │  │
│  │ HF RSS   │   │ 时序管理  │   │  │ /api/timeline│ │  │
│  │          │   │          │   │  │ /api/explore│  │  │
│  │ APScheduler│  │ Kuzu 嵌入│   │  └────────────┘  │  │
│  │ (每日触发)│   │ 式图DB   │   │                  │  │
│  └──────────┘   └────┬─────┘   │  静态前端         │  │
│                      │         │  (vis-network     │  │
│                 DeepSeek API   │   图谱可视化)      │  │
│                 (实体+嵌入)    └──────────────────┘  │
│                                                        │
│  /data → Kuzu 持久化 (docker volume)                   │
└────────────────────────────────────────────────────────┘
```

### 组件职责

| 组件 | 职责 | 依赖 |
|---|---|---|
| **爬虫层** `kepler/ingest/` | 从 HN/arxiv/HF 获取内容，清洗去重，输出 `RawItem` | 无 |
| **内容提取器** `kepler/ingest/extractor.py` | HTML→Markdown，URL 去重（SHA-256），内容长度过滤 | 爬虫层 |
| **Graphiti 引擎** `kepler/graph/` | 实体提取、关系解析、时序管理、语义嵌入 | DeepSeek API |
| **Kuzu 存储** | 嵌入式图数据库，存节点/边/嵌入向量 | 无外部依赖 |
| **Web API** `kepler/web/api.py` | REST 接口：搜索、时间线、子图探索 | Graphiti |
| **前端** `kepler/web/static/` | 知识图谱可视化 + 时间线 + 搜索页 | Web API |
| **调度器** `kepler/scheduler.py` | APScheduler，每日定时触发爬取→摄入流程 | 全部 |

### 数据流

```
[每日 08:00 触发]
       │
       ▼
┌─────────────────┐
│ fetch_all_sources│  并发爬取三个源
│ (asyncio.gather) │
└────────┬────────┘
         ▼
┌─────────────────┐
│  deduplicate()   │  URL SHA-256 对比已摄入记录
└────────┬────────┘
         ▼
┌─────────────────┐
│ graphiti.add_    │  DeepSeek 提取实体+关系
│ episode_bulk()   │  + 嵌入向量计算
└────────┬────────┘
         ▼
┌─────────────────┐
│   Kuzu 持久化    │  写入磁盘 /data/kuzu/
└─────────────────┘
```

---

## 3. 组件详细设计

### 3.1 爬虫层

**数据模型**：

```python
@dataclass
class RawItem:
    source: Literal["hackernews", "arxiv", "huggingface"]
    url: str
    title: str
    content: str              # 清洗后的 Markdown
    published_at: datetime
    source_tags: list[str]    # arxiv 分类 / HN 标签 / HF 主题
```

**三个适配器**：

| 适配器 | 接口 | 过滤策略 |
|---|---|---|
| `HackerNewsScraper` | HN Firebase API (`/v0/topstories`) | 取当天 top 100，关键词匹配 "agent/LLM/AI" |
| `ArxivScraper` | `feedparser` 解析 arxiv RSS | 按 `cs.AI` `cs.CL` `cs.MA` 分类过滤 |
| `HFBlogScraper` | `feedparser` 解析 HF blog RSS | 全量获取，博客数量少 |

**去重策略**：URL SHA-256 存 Kuzu 节点，每次摄入前先查是否存在。

**错误处理**：单个源失败不阻塞其他源；全部失败记录日志，第二天重试。

### 3.2 Graphiti 配置

**DeepSeek 适配**（关键）：DeepSeek 提供 OpenAI 兼容接口，配置如下：

```python
graphiti = Graphiti(
    config=GraphitiConfig(
        llm_client=OpenAIClient(
            base_url="https://api.deepseek.com/v1",
            api_key=os.environ["DEEPSEEK_API_KEY"],
            model="deepseek-chat",
            temperature=0.0,
        ),
        embedder=SentenceTransformerEmbedder(
            model_name="all-MiniLM-L6-v2",  # 本地 embedding，不调 API
        ),
        store=KuzuStore(
            db_path="./data/kuzu",
            # Kuzu 是嵌入式数据库，零配置
        ),
    )
)
```

**⚠️ 注意事项**：
- `deepseek-chat` 可能需要较大的 max_tokens（实体提取输出较长）
- **Embedding 使用本地模型** `all-MiniLM-L6-v2`（通过 `sentence-transformers`），因为 DeepSeek 没有独立的 Embeddings API。本地运行，无额外 API 成本
- Graphiti 默认 `SEMAPHORE_LIMIT=10`（LLM 并发限制），建议调低到 3-5 适配 DeepSeek 速率限制

### 3.3 图谱 Schema

**实体类型（节点）**：

| 类型 | 字段 | 示例 |
|---|---|---|
| `Technology` | category, maturity, first_appeared | RAG, Function Calling, Chain-of-Thought |
| `Organization` | org_type | OpenAI, Anthropic, LangChain |
| `Person` | affiliation | 论文作者、项目维护者 |
| `Paper` | arxiv_id, citations, published_date | "Attention Is All You Need" |
| `Product` | product_type | ChatGPT, Claude, Copilot, Cursor |

**关系类型（边）**：

| 关系 | 含义 | 时序意义 |
|---|---|---|
| `BuiltWith` | X 基于 Y 构建 | 技术栈依赖链 |
| `Implements` | X 实现了 Y 技术 | 理论→实践落地 |
| `ProposedBy` | 技术由论文/人提出 | 创新溯源 |
| `CompetesWith` | 竞争关系 | 技术路线分化 |
| `Deprecates` | X 替代 Y | **核心**：追踪技术淘汰 |
| `DependsOn` | 上游依赖 | 供应链/生态依赖 |
| `PublishedBy` | 组织发布论文/产品 | 成果归属 |

**自动发现**：Graphiti 遇到无法归类的新实体时，DeepSeek 自动建议新类型，无需手动扩展。

### 3.4 Web API

**端点设计**：

| 方法 | 路径 | 功能 | 返回 |
|---|---|---|---|
| `GET` | `/api/graph/search?q=xxx` | 语义搜索节点 | `{ nodes: [...], total: N }` |
| `GET` | `/api/graph/node/{id}` | 节点详情 | `{ node, edges, timeline }` |
| `GET` | `/api/graph/timeline?entity=xxx` | 实体时间线 | `{ events: [{date, change_type, evidence}] }` |
| `GET` | `/api/graph/explore?center=xxx&depth=2` | 子图探索 | `{ nodes, edges }` 图数据 |
| `GET` | `/api/stats` | 统计信息 | `{ node_count, edge_count, last_update }` |
| `GET` | `/api/health` | 健康检查 | `{ status: "ok" }` |

### 3.5 前端设计

单页应用，三个视图通过 Tab 切换：

| 视图 | 内容 | 技术 |
|---|---|---|
| **图谱视图** | 力导向图，节点=技术/产品/组织，边=关系，点击展开子图 | `vis-network` |
| **时间线视图** | 选中实体后，展示技术演变时间轴（新→旧），每节点标注来源 | 自定义 (轻量 CSS) |
| **搜索视图** | 输入框 + 搜索结果列表，关键词高亮，点击跳转节点详情 | 纯 HTML/CSS |

**交互细节**：
- 图谱初始加载核心节点（50-100 个），按重要性排序
- 点击节点 → 展开与该节点关联的子图
- 悬停节点 → 显示摘要卡片
- 时间线按年份组织，标注关键转折点（新框架发布、技术替代）
- 搜索支持中英文混合关键词

### 3.6 历史数据回溯

采用 C 混合策略：

1. **第一阶段（上线后立即）**：系统只爬取新内容，图谱从当天开始积累
2. **第二阶段（异步后台）**：运行回溯脚本，批量拉取 arxiv 历年 Agent 相关论文 + HN 历史高赞帖，一次性灌入

回溯脚本独立于主流程，通过 `--backfill` 参数触发，可中断可续。

---

## 4. 部署

### Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PYTHONUNBUFFERED=1
ENV KEPLER_DATA_DIR=/data

RUN mkdir -p /data

VOLUME ["/data"]
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s \
    CMD curl -f http://localhost:8000/api/health || exit 1

CMD ["sh", "-c", "uvicorn kepler.web:app --host 0.0.0.0 --port 8000"]
```

### 部署命令

```bash
# 构建
docker build -t kepler .

# 运行
docker run -d \
  --name kepler \
  -p 8000:8000 \
  -v kepler_data:/data \
  -e DEEPSEEK_API_KEY=sk-xxx \
  --restart unless-stopped \
  kepler

# 访问
# http://<server-ip>:8000
```

### 依赖清单

```
graphiti>=0.29.0
kuzu>=0.6.0
fastapi>=0.115.0
uvicorn[standard]
feedparser>=6.0.0
httpx>=0.28.0
apscheduler>=3.10.0
openai>=1.50.0
beautifulsoup4>=4.12.0
markdownify>=0.3.0
sentence-transformers>=2.7.0
python-dotenv>=1.0.0
```

---

## 5. 项目结构

```
kepler/
├── Dockerfile
├── requirements.txt
├── .env.example              # DEEPSEEK_API_KEY=sk-xxx
├── kepler/
│   ├── __init__.py
│   ├── config.py             # 全局配置（API keys, 路径, 调度时间）
│   ├── scheduler.py           # APScheduler，每日触发
│   ├── ingest/
│   │   ├── __init__.py
│   │   ├── base.py            # 抽象基类
│   │   ├── hackernews.py      # HN Firebase API
│   │   ├── arxiv.py           # arxiv RSS + feedparser
│   │   ├── huggingface.py     # HF blog RSS
│   │   └── extractor.py       # HTML 清洗 + Markdown 转换 + 去重
│   ├── graph/
│   │   ├── __init__.py
│   │   ├── client.py          # Graphiti 初始化（DeepSeek 配置）
│   │   ├── schema.py          # Pydantic 实体/关系类型定义
│   │   └── ingest.py          # 摄入流程（add_episode）
│   └── web/
│       ├── __init__.py
│       ├── api.py             # FastAPI 路由 + REST 接口
│       ├── app.py             # FastAPI app 工厂（启动调度器）
│       └── static/
│           ├── index.html     # SPA 入口
│           ├── graph.js       # vis-network 图谱可视化
│           ├── timeline.js    # 时间线视图
│           └── style.css
└── data/                      # (gitignore) 运行时数据
    └── kuzu/                  # Kuzu 数据库文件
```

---

## 6. 风险 & 已知限制

| 风险 | 缓解措施 |
|---|---|
| DeepSeek 没有 Embedding API | ✅ 已用本地 `sentence-transformers` 替代 |
| DeepSeek API 速率限制 | `SEMAPHORE_LIMIT` 调低到 3-5；重试 + 退避 |
| Graphiti v0.x API 变更 | 锁定版本号，升级前测试 |
| Kuzu 嵌入式 DB 规模上限 | 单文件 10TB 理论上限；实际百万节点内无忧 |
| 爬虫源不稳定（API 变更） | 失败不影响整体，记录日志次日重试 |
| 中文内容 arxiv/HN 少 | 初期英文为主，后续扩展中文源 |

---

## 7. 成功标准

- [ ] `docker run` 后浏览器访问 8000 端口可看到知识图谱页面
- [ ] 首次运行后 24h 内图谱有 ≥20 个节点
- [ ] 每日定时任务成功执行（日志验证）
- [ ] 搜索功能返回语义匹配结果
- [ ] 时间线展示某实体的历史演变
- [ ] 30 天后图谱节点数 ≥200
