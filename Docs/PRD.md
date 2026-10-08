# FoodSafe Agent

## 食品安全法规与召回分析 Agentic RAG

**版本**：v0.1

**实现语言**：Java 21 + Spring Boot 3

**产品定位**：参考 WeKnora 的企业级 RAG、Agent、记忆、异步任务、引用和评估设计，独立实现面向食品法规、召回公告和抽检信息的企业级 Agentic RAG 系统。

---

## 1. 背景与目标

食品安全信息分散在法规、标准、召回公告、抽检结果和风险提示中。普通关键词搜索难以处理自然语言问题，普通 RAG 又难以完成统计、时效核验和多来源交叉验证。

### 产品目标

1. 建立食品安全领域的持续更新知识库。
2. 支持法规问答、召回查询、抽检分析和风险研判。
3. 让 Agent 自主选择知识库检索、结构化查询和网页核验工具。
4. 所有事实结论都能追溯到来源、条款、日期和 URL。
5. 通过离线评估和线上观测持续衡量 RAG 与 Agent 质量。

### 非目标

- 不提供法律或医疗性质的最终决策。
- 不把用户记忆当作食品安全事实来源。
- 第一阶段不覆盖所有国家、语言和食品类别。
- 第一阶段不实现复杂企业审批流。

---

## 2. WeKnora 设计对照原则

本项目使用 Java 独立实现，不依赖 WeKnora 运行时或源码；但以下设计必须有对应学习和实现：

| 能力 | WeKnora 参考思想 | Java 项目实现 |
|---|---|---|
| 分层架构 | Handler → Service → Repository | Controller → Application Service → Repository |
| 文档流水线 | 解析、分块、Embedding、索引、后处理 | Ingestion Pipeline + 状态机 |
| 异步任务 | Redis/Asynq、重试、幂等 | Redis Streams/Job Worker、重试和幂等键 |
| RAG 流程 | Event/Plugin Pipeline | `PipelineStage` 接口和动态 Stage 组装 |
| 混合检索 | 向量 + 关键词 + RRF + Rerank | pgvector + PostgreSQL FTS + RRF + Rerank |
| Agent | ReAct：think → act → observe | Agent Runtime 状态机 |
| 工具治理 | 动态注册、能力检查、权限过滤 | Tool Registry + Capability Guard |
| 长期记忆 | scope、抽取、召回、确认、预算 | Memory Service + 异步抽取 + 词法/向量召回 |
| 上下文管理 | token 预算、摘要、checkpoint | Context Manager + Compaction Checkpoint |
| 引用 | Chunk 来源、原文定位、Citation | Evidence Ledger + Source Reference |
| 评估 | Recall、MRR、NDCG、生成指标 | RAG Eval + Agent Trajectory Eval |

任何新增业务能力都必须说明对应的 WeKnora 设计原则和为什么适用于食品场景。

---

## 3. 用户与使用场景

### 用户角色

- **普通员工**：查询法规、产品和召回信息。
- **食品安全专员**：分析风险、比较企业和生成报告。
- **空间管理员**：管理数据源、模型、成员、权限和评估任务。
- **系统管理员**：管理租户、系统配置、审计和任务运行状态。

### 核心场景

#### S1：法规问答

“苯甲酸钠是否允许用于某类食品？依据哪条标准？”

结果必须包含结论、法规名称、条款、发布日期和原文链接。

#### S2：召回查询

“某品牌最近是否有召回，涉及哪些批次，原因是什么？”

需要组合企业过滤、日期排序、召回检索和来源读取。

#### S3：抽检统计

“2025 年某地区不合格食品最多的企业有哪些？”

Agent 应调用结构化查询或分析工具，不能只依赖向量检索。

#### S4：多步风险分析

“分析某产品的风险，并给出法规依据和最新官方信息。”

需要召回检索、法规检索、网页核验、冲突检测和最终报告生成。

#### S5：个性化查询

“最近有什么需要关注的？”

根据用户已确认的地区、食品类别和长期关注主题改写查询，但不得限制用户可访问的知识范围。

---

## 4. 功能需求

### 4.1 数据源与采集

支持监管网站、法规网站和公告页面的定时采集。

功能要求：

- 定时任务和手动同步。
- URL、内容 hash、标题和版本去重。
- 增量更新和删除同步。
- 抓取失败重试、退避和死信记录。
- SSRF 防护、域名白名单、请求速率限制。
- 保存原始 HTML、清洗后的正文和来源 URL。

### 4.2 文档处理

支持法规、公告、HTML、PDF、Word、Excel 和 Markdown。

处理状态：

```text
pending → processing → finalizing → completed
                         ↘ failed/cancelled
```

食品领域抽取字段：

```text
document_type, title, publisher, source_url,
published_at, effective_from, effective_to,
region, food_category, company_name, product_name,
batch_number, risk_type, recall_reason, regulation_number,
status, content_hash
```

### 4.3 领域分块与索引

分块策略：

- 法规按章节、条、款、项切分。
- 召回公告按产品、批次、原因和措施切分。
- 抽检公告按样品、企业、不合格项目切分。
- 保留标题路径、条款编号、父块关系和原文定位。
- 支持 chunk size、overlap、parent-child 配置。

索引：

- pgvector 语义索引。
- PostgreSQL FTS 关键词索引。
- 元数据 B-tree 索引。
- 文档和 Chunk 版本索引。

### 4.4 混合检索

支持 `semantic`、`keyword` 和 `hybrid` 三种模式。

默认流程：

```text
查询改写 → 多路召回 → RRF 融合 → 元数据过滤 → Rerank → 去重 → Top-K
```

RRF：

```text
score = vector_weight / (k + vector_rank)
      + keyword_weight / (k + keyword_rank)
```

必须记录每个结果的：召回来源、原始排名、融合分数、rerank 分数和最终排名。

### 4.5 Agent Runtime

Agent 状态：

```text
CLASSIFY → PLAN → TOOL_CALL → OBSERVE → VERIFY → ANSWER
```

安全限制：

- 单次任务最多 8 个 Agent 回合。
- 工具调用总数和单工具调用次数可配置。
- 工具只能访问当前租户和授权知识库。
- 结构化查询仅允许 SELECT，并自动注入租户过滤。
- 失败工具不得被描述为成功。
- 证据不足时必须说明限制或继续核验。

### 4.6 食品领域工具

第一阶段实现：

| 工具 | 作用 |
|---|---|
| `search_food_regulation` | 检索法规、标准和条款 |
| `search_food_recall` | 查询召回公告和批次 |
| `query_food_records` | 按企业、地区、日期、类别统计 |
| `read_food_document` | 读取原文和上下文 |
| `verify_food_source` | 核对官方 URL、日期和最新版本 |
| `compare_food_risks` | 比较产品、企业或风险项目 |
| `search_food_memory` | 查询用户已确认的长期记忆 |

工具结果统一返回：

```json
{
  "success": true,
  "data": {},
  "evidence": [],
  "source_refs": [],
  "next_cursor": null,
  "error_code": null
}
```

### 4.7 记忆模块

记忆类型：`profile`、`preference`、`fact`、`task`、`interest`。

要求：

- 记忆按 `tenant_id + principal_id` 隔离。
- 支持 `explicit_only` 和 `auto` 两种写入模式。
- 自动抽取必须异步执行。
- 自动推断的内容先进入 `pending`，确认后才进入 `active`。
- 新事实替换旧事实时保留 superseded 状态。
- 常驻记忆和情境记忆使用不同预算。
- 记忆只辅助查询理解，不覆盖知识库权限和事实证据。

### 4.8 引用与证据

最终回答必须区分：

- 用户输入；
- 记忆背景；
- 知识库证据；
- 外部网页证据；
- Agent 推断。

引用至少包含：

```text
source_id, title, url, publisher,
published_at, document_version,
article_number, chunk_id, quote
```

### 4.9 评估与 Eval

支持通过 API 或命令行运行评估任务。

评估数据集包含：

- `questions`：问题。
- `corpus`：法规和公告语料。
- `qrels`：问题与正确证据的关联。
- `answers`：参考答案。
- `expected_tools`：期望使用的工具。
- `expected_sources`：期望引用来源。
- `requires_verification`：是否必须网页核验。

指标：

#### 检索指标

- Recall@5、Recall@10。
- MRR。
- NDCG@3、NDCG@10。
- MAP。

#### 生成指标

- ROUGE-L、BLEU 作为辅助指标。
- 引用正确率。
- 事实支持率。
- 过期法规误用率。

#### Agent 指标

- 工具选择准确率。
- 工具参数准确率。
- 多步任务完成率。
- 无效工具调用率。
- 证据充分时的平均调用次数。
- 必须核验问题的核验覆盖率。

评估必须保存：模型、检索参数、Prompt 版本、工具轨迹、证据列表和时间戳，保证可复现。

---

## 5. API 草案

```text
POST /api/v1/documents/sync
GET  /api/v1/documents/{id}
POST /api/v1/knowledge/search
POST /api/v1/agent/sessions
POST /api/v1/agent/sessions/{id}/messages
GET  /api/v1/agent/sessions/{id}/events
POST /api/v1/memories
GET  /api/v1/memories
POST /api/v1/evaluations
GET  /api/v1/evaluations/{id}
GET  /api/v1/health
```

问答接口支持 SSE，事件包括：

```text
plan_created
tool_started
tool_finished
evidence_found
verification_started
answer_delta
answer_completed
context_compacted
error
```

---

## 6. 数据模型草案

核心表：

```text
tenants
users
data_sources
documents
document_versions
chunks
chunk_embeddings
food_records
agent_sessions
agent_messages
agent_steps
tool_calls
evidence_records
memories
memory_embeddings
evaluation_tasks
evaluation_cases
evaluation_results
audit_logs
```

所有业务表必须包含：

```text
tenant_id
created_at
updated_at
deleted_at（适用时）
```

---

## 7. 非功能需求

### 安全

- 租户隔离和 RBAC。
- API Key / JWT 鉴权。
- 工具能力白名单。
- SQL 只读和表白名单。
- 网页抓取 SSRF 防护。
- 敏感字段日志脱敏。
- 来源和 Agent 操作审计。

### 可靠性

- 文档任务幂等。
- 失败重试和死信队列。
- 外部模型超时、限流和降级。
- Embedding 不可用时关键词检索降级。
- 记忆不可用时不影响正常回答。
- Agent 工具失败不得伪造成功结果。

### 性能目标

第一阶段目标：

- 普通检索 P95 小于 2 秒，不包含模型生成。
- Agent 首个事件 P95 小于 3 秒。
- 单次任务最多 8 个 Agent 回合。
- 单次回答默认最多返回 10 个证据片段。
- 数据采集任务支持至少 10 万篇文档的增量索引设计。

### 可观测性

每次请求记录：

- request_id、tenant_id、session_id。
- 检索参数和召回结果。
- Agent 状态转换。
- 工具调用、耗时和结果状态。
- 模型、Token、成本和错误。
- 引用和最终证据。

---

## 8. 分阶段交付

### M1：基础 RAG

- Spring Boot 项目骨架。
- 食品文档导入。
- 清洗、分块、Embedding。
- pgvector 检索。
- 基础回答和引用。

### M2：企业级检索

- 关键词检索。
- RRF 融合。
- 元数据过滤。
- Rerank。
- 异步任务、重试和幂等。

### M3：Agentic RAG

- Agent Runtime。
- Tool Registry。
- 食品法规、召回、结构化查询和来源核验工具。
- 多步工具轨迹和 SSE 事件。

### M4：记忆与上下文

- 显式记忆。
- 异步自动抽取。
- 记忆确认和替换。
- 词法/向量记忆召回。
- Token 预算和上下文压缩。

### M5：评估和生产化

- 离线评估集。
- RAG、Agent、引用三类指标。
- Langfuse 或 OpenTelemetry 观测。
- RBAC、审计、限流、部署文档。

---

## 9. 验收标准

项目完成必须满足：

1. 能从食品网页完成采集、清洗、分块和索引。
2. 能回答法规和召回问题，并返回可打开的官方来源。
3. Agent 能根据问题选择至少两类不同工具。
4. 对需要核验的问题，Agent 能执行来源核验并记录轨迹。
5. 用户记忆能影响查询改写，但不能替代事实证据。
6. 文档和工具任务具备重试、幂等和错误降级。
7. 能运行固定数据集并输出检索、引用和 Agent 指标。
8. 能展示一个前后对比实验，例如：纯向量 vs 混合检索，或无核验 Agent vs 核验 Agent。
9. 能通过租户隔离、工具权限和 SQL 只读测试。
10. 能用 5 分钟演示完整链路：提问 → 规划 → 工具调用 → 证据 → 引用回答 → 评估结果。

---

## 10. 简历项目表述

> 参考 WeKnora 的企业级 RAG、Agent Pipeline、长期记忆、上下文管理和评估设计，独立使用 Java 21 与 Spring Boot 实现食品安全法规与召回分析 Agentic RAG 系统。系统支持食品网页采集、领域元数据抽取、条款级分块、向量/关键词混合检索、RRF、Rerank、Agent 多工具调用、来源核验、长期记忆和可复现 Eval。

> 构建食品法规、召回公告和抽检数据评估集，从 Recall@K、MRR、NDCG、引用正确率、工具选择准确率和过期信息误用率等维度评估系统质量。

简历中应明确写“参考 WeKnora 设计思想，独立 Java 实现”，不要将 WeKnora 的原有代码成果写成自己的底层实现。
