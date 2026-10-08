# FoodLens 项目协作与开发约定

## 1. 项目定位

FoodLens 是一个使用 Java 独立实现的食品安全法规与召回分析 Agentic RAG 系统。

项目参考 WeKnora 的企业级 RAG、Agent、长期记忆、上下文管理、异步任务、来源引用和评估设计，但不依赖 WeKnora 的运行时或源码。目标是通过独立实现掌握这些通用思想，并将它们落地到食品信息场景。

项目标题：

> FoodLens Agent：食品安全法规与召回分析 Agentic RAG

## 2. 核心目标

系统需要逐步支持：

```text
食品网站数据采集
→ 清洗与领域元数据抽取
→ 条款级/公告级分块
→ 向量 + 关键词混合检索
→ RRF 融合与 Rerank
→ Agent 规划和工具调用
→ 多来源核验与冲突检测
→ 带来源、日期和条款的回答
→ 长期记忆与个性化查询
→ RAG/Agent/Evidence Eval
→ Docker Compose 部署
```

第一阶段只实现可运行、可评估、可解释、可部署的 MVP，不扩展到所有企业能力。

## 3. 技术方向

默认技术栈：

- Java 21
- Spring Boot 3
- Maven
- Spring AI 或 OpenAI 兼容模型接口
- PostgreSQL + pgvector
- PostgreSQL Full Text Search
- Redis
- Docker Compose
- Jsoup
- Apache Tika
- JUnit 5
- Mockito
- Testcontainers
- GitHub Actions

如果技术选型发生变化，必须在 `docs/decisions/` 记录原因、替代方案和影响。

## 4. WeKnora 学习映射

每个核心功能都要同时完成“学习思想、独立实现、验证效果”三件事。

| FoodLens 模块 | 参考 WeKnora 思想 | 要掌握的知识 |
|---|---|---|
| Controller/Service/Repository | 后端分层 | 依赖方向、接口隔离、可测试性 |
| Ingestion Pipeline | 文档处理流水线 | 状态机、幂等、重试、增量更新 |
| Chunking | 领域分块 | 父子块、标题路径、元数据 |
| Hybrid Retriever | 多路检索 | 向量、关键词、RRF、Rerank |
| PipelineStage | Chat Pipeline 插件 | 动态组装、阶段职责、错误传播 |
| AgentRuntime | ReAct Agent | Think/Act/Observe、规划、工具循环 |
| ToolRegistry | Agent 工具治理 | 注册、能力过滤、权限、参数校验 |
| MemoryService | 长期记忆 | Scope、抽取、确认、召回、预算 |
| ContextManager | 上下文压缩 | Token 估算、摘要、Checkpoint、工具结果裁剪 |
| EvidenceLedger | Citation | 证据链、来源定位、引用完整性 |
| EvaluationRunner | Evaluation | 数据集、Qrels、检索指标、轨迹指标 |

不照搬 WeKnora 代码。实现中可以借鉴它的设计取舍，但 Java 代码、包结构、接口和测试必须由 FoodLens 独立建立。

## 5. 食品领域范围

第一阶段数据类型：

- 食品安全法规与标准
- 食品召回公告
- 食品抽检公告
- 食品安全风险提示

推荐元数据：

```text
document_type
title
publisher
source_url
published_at
effective_from
effective_to
region
food_category
company_name
product_name
batch_number
risk_type
recall_reason
regulation_number
status
content_hash
```

第一阶段 Agent 工具：

- `search_food_regulation`
- `search_food_recall`
- `query_food_records`
- `read_food_document`
- `verify_food_source`
- `compare_food_risks`
- `search_food_memory`

## 6. Agent 行为要求

Agent 状态：

```text
CLASSIFY → PLAN → TOOL_CALL → OBSERVE → VERIFY → ANSWER
```

必须遵守：

- 先判断问题需要知识库检索、结构化查询、网页核验还是组合工具。
- 工具调用有最大回合数和超时。
- 工具失败不能被描述为成功。
- 搜索摘要不能自动当作完整证据，需要时读取原文。
- 对最新法规、召回状态和来源冲突执行核验。
- 记忆只能影响个性化查询，不能替代官方来源。
- 最终答案区分用户输入、记忆背景、知识库证据、网页证据和 Agent 推断。

## 7. 测试要求

测试必须随功能一起实现，不能在项目末尾集中补测试。

### 单元测试

覆盖：

- 分块边界、标题路径、父子关系
- RRF 计算和排序
- 元数据过滤
- Agent 状态转换
- 工具参数校验
- 记忆预算和作用域隔离
- 引用来源合并
- Eval 指标计算

### 集成测试

使用 Testcontainers 或 Docker Compose 验证：

- PostgreSQL/pgvector 写入和检索
- Redis 任务入队、消费、重试和幂等
- 文档导入到索引的完整链路
- Agent 工具与数据库的集成

### 验收测试

每个核心场景至少有一个端到端案例：

- 法规问答
- 食品召回查询
- 抽检统计
- 多步风险分析
- 来源核验失败
- 无相关证据时的安全回答

## 8. 环境学习要求

每天都要学习并记录一个环境或中间件知识点：

- Java/Maven 构建和依赖管理
- Spring Profile 和环境变量
- Dockerfile、Compose、volume、healthcheck
- PostgreSQL migration、索引和 EXPLAIN
- pgvector 和全文检索
- Redis Stream、Consumer Group、ACK、重试
- 日志、配置和密钥管理
- Testcontainers
- GitHub Actions
- 部署、健康检查、日志和故障排查

敏感信息只能通过 `.env` 或 CI Secrets 注入，不能提交到 Git。

## 9. Git 约定

分支：

```text
main
codex/day-1-foundation
codex/day-2-ingestion
codex/day-3-retrieval
codex/day-4-agent
codex/day-5-memory
codex/day-6-eval
codex/day-7-deploy
```

提交信息：

```text
feat: add food document ingestion
fix: handle empty retrieval result
test: add rrf ranking tests
docs: record retrieval decision
chore: configure github actions
```

每天至少一次小提交。每个功能提交前运行：

```bash
./mvnw test
./mvnw verify
```

如果暂时没有 Maven Wrapper，先使用 `mvn`，然后尽快提交 Wrapper。

## 10. CI/CD 要求

GitHub Actions 至少包含：

1. Checkout 代码。
2. 设置 Java 21。
3. 缓存 Maven 依赖。
4. 执行格式检查或静态检查。
5. 执行单元测试。
6. 启动 Testcontainers 或服务容器并执行集成测试。
7. 构建 JAR 和 Docker 镜像。
8. 生成测试报告。
9. main 分支合并后构建可部署镜像。

CI 失败时先修复失败原因，再继续开发，不跳过测试。

## 11. 文档规则

所有学习笔记、架构图、实验结果和决策记录放在 `docs/`：

```text
docs/
├── architecture/       # 架构和时序图
├── learning/           # WeKnora 学习笔记
├── decisions/          # 技术决策记录
├── eval/               # 数据集、指标和实验报告
├── operations/         # 环境、部署和故障排查
└── api/                # API 示例
```

每一天至少更新一个文档，内容包括：

- 今天学习的 WeKnora 知识点。
- FoodLens 的对应实现。
- 遇到的问题和解决方式。
- 测试结果。
- Git 提交号。
- 下一步计划。

## 12. 每日工作节奏

每天 4 小时：

- 60 分钟：阅读 WeKnora 文档和关键代码。
- 120 分钟：实现 FoodLens 功能。
- 30 分钟：测试、环境或中间件实践。
- 30 分钟：写学习文档、提交 Git、复盘。

每天结束前必须有：

```text
可运行代码
至少一个测试或实验
一篇学习/决策记录
一次 Git 提交
```

## 13. MVP 验收标准

7 天结束时必须能够：

1. 通过 Docker Compose 启动 Java、PostgreSQL 和 Redis。
2. 导入食品法规或召回公告。
3. 完成清洗、分块和索引。
4. 运行向量、关键词和混合检索。
5. Agent 至少调用两个食品工具完成一次多步任务。
6. 返回来源 URL、日期、文档和 Chunk 引用。
7. 支持最基础的用户记忆和查询个性化。
8. 运行 RAG、引用和 Agent 评估。
9. GitHub Actions 能执行构建和测试。
10. README 能指导别人从零启动并完成一次演示。

## 14. 范围控制

7 天内不做：

- 完整前端后台
- 所有模型供应商
- 全量 MCP 能力
- 浏览器自动化
- 知识图谱完整产品化
- 复杂企业审批流
- 全面多租户管理后台

优先保证核心闭环：

```text
数据 → 检索 → Agent → 证据 → 回答 → Eval → 部署
```
