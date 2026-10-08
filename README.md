# FoodLens Agent

食品安全法规与召回分析 Agentic RAG。

FoodLens 参考 WeKnora 的企业级 RAG、Agent、长期记忆、上下文管理、引用和评估设计，使用 Java 独立实现，不依赖 WeKnora 的运行时或源码。

## 当前状态

Day 1：项目骨架、健康检查和 PostgreSQL/Redis 本地环境。

## 本地运行

```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 21)
cp .env.example .env
./mvnw test
docker compose up -d
./mvnw spring-boot:run
```

验证：

```bash
curl http://localhost:8080/api/v1/health
curl http://localhost:8080/actuator/health
```

## 文档

- [项目协作约定](Agent.md)
- [PRD](docs/PRD.md)
- [7 天学习计划](docs/STUDY_PLAN.md)
- [Day 1 学习记录](docs/learning/day-1-foundation.md)
