# Day 1：项目骨架与企业级服务入口

## 今天学习的 WeKnora 思想

WeKnora 使用 Handler、Service、Repository 和基础设施分层组织后端。FoodLens 今天只建立骨架，后续会将采集、检索、Agent 和评估分别放入对应层，避免 Controller 直接处理业务和数据库。

WeKnora 的服务由多个基础设施协作运行。FoodLens 先用 Docker Compose 启动 PostgreSQL 和 Redis，为后续向量检索、全文检索和异步任务提供稳定环境。

## 今天的实现

- Spring Boot 3 + Java 21 项目配置。
- `api`、`application`、`domain`、`infrastructure` 基础包。
- `/api/v1/health` 应用健康接口。
- Actuator `/actuator/health`。
- PostgreSQL pgvector 和 Redis Compose 服务。
- HealthController 的 MVC 测试。

## Java 环境

本机默认 Java 是 17，但系统已安装 Java 21。项目使用 Java 21 执行：

```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 21)
./mvnw test
```

## 设计取舍

健康检查暂时只确认应用进程可响应；数据库和 Redis 的连接检查会在 Day 2/3 接入真实组件后加入。这样避免在基础骨架阶段让未配置的外部服务阻塞应用启动。

## 验收

- `./mvnw test` 通过。
- `docker compose config` 通过。
- PostgreSQL 和 Redis 能通过 Compose 启动。
- `/api/v1/health` 返回 `status=UP`。
