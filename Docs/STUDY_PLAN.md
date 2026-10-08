# 7 天食品 Agentic RAG 学习与开发计划

## 总目标

7 天、每天 4 小时，完成一个使用 Java 独立实现的食品法规与召回分析 Agentic RAG MVP：

```text
食品数据导入 → 清洗分块 → 向量/关键词检索 → Agent 工具调用
→ 来源引用 → 基础记忆 → Eval → Docker Compose 部署
```

参考 WeKnora 的设计思想，但不依赖 WeKnora 运行时或源码。

## 固定技术栈

- Java 21
- Spring Boot 3
- PostgreSQL + pgvector
- Redis
- Docker Compose
- Jsoup
- Spring AI 或 OpenAI 兼容模型接口
- JUnit 5 + Testcontainers（能使用时）

## 每天固定工作方式

- 15 分钟：`git status`、查看昨天改动、确定今天目标。
- 60 分钟：阅读 WeKnora 文档和关键代码。
- 120 分钟：完成 Java 实现。
- 30 分钟：运行测试、手动验证或做实验。
- 15 分钟：提交 Git、记录设计取舍和明日问题。

每天至少形成一次小提交，提交信息使用：

```text
feat: add food document ingestion
fix: handle empty retrieval result
docs: record hybrid retrieval decision
test: add citation evaluation cases
```

---

## Day 1：架构、环境和 Java 项目骨架

### WeKnora 学习点

- 三进程架构：Go app、docreader、frontend。
- Handler → Service → Repository 分层。
- Chat Pipeline 的插件化思想。
- Agent 与固定 RAG Pipeline 的区别。

阅读：

- `website-docs/02-architecture/01-overview.md`
- `website-docs/02-architecture/02-backend-design.md`
- `website-docs/02-architecture/04-rag-pipeline.md`

### Java 实现

- 创建 `food-agentic-rag` Spring Boot 项目。
- 创建模块包：`api`、`application`、`domain`、`infrastructure`。
- 添加 `/api/v1/health`。
- 添加统一错误响应、配置文件和日志格式。
- 编写项目 README 和架构图。

### Git / 中间件

- 创建 Git 仓库和 `main` 分支。
- 学习 `branch`、`add`、`commit`、`log`、`diff`、`merge`。
- 用 Docker Compose 启动 PostgreSQL 和 Redis。
- 学习 `.env`、端口、volume 和 healthcheck。

### 当天验收

```text
./mvnw test 通过
docker compose up -d 成功
GET /api/v1/health 返回 UP
```

---

## Day 2：文档采集、清洗、分块和异步任务

### WeKnora 学习点

- 文档处理状态机：pending → processing → finalizing → completed。
- Asynq/Redis 异步任务、重试和幂等。
- docreader 与主服务解耦的原因。
- 分块为什么要保留标题路径和父子关系。

阅读：

- `website-docs/02-architecture/03-document-pipeline.md`
- `internal/application/service/knowledge_process.go`
- `internal/infrastructure/chunker/strategy.go`

### Java 实现

- `FoodDocument`、`FoodChunk`、`DocumentStatus`。
- 实现 HTML 抓取和正文清洗。
- 实现食品公告的标题级、条款级分块。
- 保存 `sourceUrl`、`publishedAt`、`region`、`companyName`、`foodCategory`。
- 实现一个 Redis Stream 文档处理任务。
- 加入幂等键：`sourceUrl + contentHash`。

### Git / 中间件

- 学习 Redis Stream、consumer group、ack 和重试。
- 学习数据库 migration，不直接手改生产表。
- 提交一次 `feat` 和一次 `test`。

### 当天验收

- 导入 5–10 篇食品公告。
- 重复导入不会产生重复文档。
- 任务失败可以重试。
- 能查询文档和 Chunk。

---

## Day 3：向量检索、关键词检索和 RRF

### WeKnora 学习点

- Vector Search、Keyword Search、Hybrid Search。
- RRF 为什么按排名融合，而不是直接相加分数。
- Rerank 在召回之后的作用。
- 元数据过滤和多知识库检索。

阅读：

- `website-docs/02-architecture/04-rag-pipeline.md`
- `internal/application/service/knowledgebase_search.go`
- `internal/application/service/knowledgebase_search_fusion.go`
- `internal/types/retrieval_config.go`

### Java 实现

- PostgreSQL 建立 pgvector 表和全文索引。
- 实现 Embedding Service 接口。
- 实现语义检索和 PostgreSQL FTS 关键词检索。
- 实现 RRF 融合。
- 实现地区、企业、食品类别和日期过滤。
- 增加 `POST /api/v1/knowledge/search`。

### Git / 中间件

- 学习 PostgreSQL volume、migration、索引和 EXPLAIN 基础。
- 学习 Redis 在任务队列中的职责。
- 用 feature branch 完成检索功能后合并回 main。

### 当天验收

- 同一个问题可以返回关键词结果和语义结果。
- 混合检索结果包含来源和排名信息。
- 能对比 `semantic`、`keyword`、`hybrid` 三种模式。

---

## Day 4：Agent Runtime 和食品工具

### WeKnora 学习点

- ReAct：Think → Act → Observe。
- Agent 工具动态注册和能力过滤。
- 工具失败、重试、权限和输出预算。
- `search_knowledge`、`read_document`、`web_search`、`web_fetch` 的组合关系。

阅读：

- `internal/agent/engine.go`
- `internal/agent/think.go`
- `internal/agent/act.go`
- `internal/agent/tools/`
- `website-docs/03-features/07-agent.md`

### Java 实现

- 定义 `AgentState`：CLASSIFY、PLAN、TOOL_CALL、OBSERVE、VERIFY、ANSWER。
- 实现 `AgentContext`、`AgentRuntime`、`ToolRegistry`。
- 实现 4 个工具：
  - `search_food_regulation`
  - `search_food_recall`
  - `query_food_records`
  - `verify_food_source`
- 加入最大回合数和工具调用日志。
- 实现 SSE 事件：plan、tool_started、tool_finished、answer_delta。

### Git / 中间件

- 学习 Spring Bean 自动注册和接口实现发现。
- 学习 SSE 和请求超时。
- 使用独立 feature branch 做 Agent 功能。

### 当天验收

输入：

```text
某品牌婴幼儿食品最近是否被召回？
```

Agent 至少完成：

```text
规划 → 召回检索 → 来源读取/核验 → 带来源回答
```

---

## Day 5：记忆、上下文管理和引用

### WeKnora 学习点

- profile、preference、fact、task、interest 的区别。
- 显式记忆和异步自动抽取。
- Memory Recall 如何影响 Query Understand。
- 文档 affinity 为什么只能轻量加权。
- Token 预算和上下文压缩与长期记忆的区别。

阅读：

- `website-docs/03-features/23-memory.md`
- `internal/application/service/memory/service.go`
- `internal/application/service/memory/extract.go`
- `internal/application/service/chat_pipeline/memory_recall.go`
- `internal/agent/compaction/`

### Java 实现

- 实现 `MemoryItem` 和 `MemoryScope`。
- 支持显式保存用户的地区、食品类别和偏好。
- 实现问题相关记忆召回。
- 把记忆作为查询背景传给 Agent，而不是作为事实证据。
- 实现 `Evidence`、`SourceReference` 和 Citation 输出。
- 增加回答 Prompt 长度限制。

### Git / 中间件

- 学习 Redis 延迟任务或 Spring Scheduler。
- 学习配置分层：application、profile、environment。
- 加入 memory 和 citation 单元测试。

### 当天验收

问题：

```text
最近有什么需要关注的？
```

记忆能够帮助改写为食品领域问题，但最终结论必须引用官方文档。

---

## Day 6：Eval、质量、错误处理和可观测性

### WeKnora 学习点

- Evaluation Dataset、Qrels、标准答案。
- Recall、MRR、NDCG、引用正确率。
- Agent 工具选择和多步轨迹评估。
- Langfuse/OpenTelemetry 的调用级观测。
- 工具结果压缩和错误降级。

阅读：

- `website-docs/03-features/15-evaluation.md`
- `internal/application/service/evaluation.go`
- `internal/application/service/metric_hook.go`
- `internal/tracing/`

### Java 实现

- 准备 20 个食品问题、标准来源和标准工具。
- 实现评估任务和结果持久化。
- 实现：
  - Recall@5
  - MRR
  - NDCG@10
  - Citation Accuracy
  - Tool Selection Accuracy
  - Verification Coverage
  - Stale Information Error Rate
- 记录 Agent 每一步工具调用和证据。
- 增加无结果、模型超时、工具失败和来源冲突测试。

### Git / 中间件

- 学习 Git tag 和 release。
- 学习 Docker 日志、健康检查和配置注入。
- 为 v0.1 打 tag。

### 当天验收

完成一组对比实验：

```text
纯向量检索 vs 混合检索
```

输出指标表和结论，不能只展示主观示例。

---

## Day 7：部署、演示和项目总结

### WeKnora 学习点

- 标准多服务部署和 Lite 模式的差异。
- 配置、健康检查、日志和资源隔离。
- 长任务、异步任务和模型服务的生产注意事项。

阅读：

- `website-docs/01-getting-started/02-installation.md`
- `website-docs/06-development/01-dev-guide.md`
- `docs/LITE.md`

### Java 实现

- 完善 Dockerfile 和 Docker Compose。
- 服务包括：Java app、PostgreSQL、Redis。
- 加入数据库初始化和健康检查。
- 增加 demo 数据导入脚本。
- 增加 README、API 示例、架构图和故障排查说明。
- 完成一次从空环境启动到问答的演示。

### Git / 中间件

- 学习 `tag`、`revert`、`stash`、`reset` 的区别。
- 查看 `git diff` 和最终提交历史。
- 建立 v1.0.0 release tag。

### 当天验收

从零执行：

```bash
cp .env.example .env
docker compose up -d --build
./scripts/load-demo-data.sh
curl /api/v1/health
```

完成 5 分钟演示：

```text
提问 → Agent 规划 → 工具调用 → 证据 → 引用回答 → Eval 结果
```

---

## 7 天最终交付物

1. Java Spring Boot 源码。
2. Docker Compose 一键启动环境。
3. 食品法规、召回和抽检 demo 数据。
4. 混合检索和 Agent 工具调用。
5. 长期记忆基础能力。
6. 引用和来源核验。
7. Eval 数据集、指标和对比实验。
8. 架构图、时序图和 API 文档。
9. Git 提交历史和 v1.0.0 tag。
10. 可放入简历的项目说明和 5 分钟演示脚本。

## 范围控制

7 天内不做：完整前端、多租户后台、所有模型供应商、知识图谱、MCP 全功能、浏览器自动化、复杂企业审批流。先完成可运行、可评估、可解释、可部署的核心闭环。
