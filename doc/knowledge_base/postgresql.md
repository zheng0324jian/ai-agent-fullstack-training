# PostgreSQL

> 分类：Database
> 相关课程：预课 数据库与缓存速览
> 整理日期：2026-10-31

## 定义

PostgreSQL 是一个开源、功能完整的关系型数据库，以 SQL 标准兼容度高、数据类型丰富、扩展性强、稳定可靠著称。它既能承担传统业务库，也能通过扩展支持 JSON、全文检索、地理信息、向量检索等场景。

## 核心能力

| 能力 | 说明 |
| --- | --- |
| 事务与并发 | 完整 ACID；MVCC 多版本并发控制，读写互不阻塞；支持 Read Committed、Repeatable Read、Serializable |
| 数据类型 | 数值、文本、UUID、数组、范围、枚举、JSONB、几何类型、向量（需扩展）等 |
| SQL 能力 | CTE、窗口函数、递归查询、UPSERT（`ON CONFLICT`）、全文检索、`LATERAL` 等 |
| 扩展机制 | `CREATE EXTENSION` 即可加载 pgvector、PostGIS、pg_trgm、TimescaleDB、Citus 等 |
| 索引 | B-tree、Hash、GIN、GiST、BRIN、SP-GiST；支持部分索引、表达式索引、覆盖索引 |
| 高可用 | WAL、流复制、逻辑复制、热备、基于时间点恢复（PITR） |
| 安全 | 角色与权限、行级安全（RLS）、SSL、SCRAM 认证 |
| 运维生态 | `pg_dump`、`pg_basebackup`、`pg_stat_statements`、`EXPLAIN`、逻辑/物理复制 |

## 架构速览

- **进程模型**：`postmaster` 主进程 + 每个连接一个 backend 进程。连接开销较大，生产环境通常配 PgBouncer 之类的连接池。
- **层级**：数据库集簇（cluster）→ database → schema → table。注意这里的 cluster 是“一个 PostgreSQL 实例管理的多个数据库”，不是分布式集群。
- **MVCC**：更新一行不会原地覆盖，而是产生新版本，旧版本等待清理。因此需要 `autovacuum` 回收死元组。
- **WAL**：先写日志再写数据文件；用于崩溃恢复、复制和 PITR。

## 快速上手

用 Docker 启动（以 PostgreSQL 17 为例）：

```bash
docker run --name pg \
  -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 \
  -d postgres:17
```

连接：

```bash
psql -h localhost -U postgres
```

一个包含 JSONB 和 UPSERT 的例子：

```sql
CREATE TABLE users (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email      text UNIQUE NOT NULL,
  profile    jsonb NOT NULL DEFAULT '{}'::jsonb,
  created_at timestamptz NOT NULL DEFAULT now()
);

INSERT INTO users (email, profile)
VALUES ('alice@example.com', '{"name":"Alice","city":"Beijing"}')
ON CONFLICT (email) DO NOTHING;

SELECT id, profile->>'name' AS name
FROM users
WHERE profile @> '{"city":"Beijing"}';

CREATE INDEX idx_users_profile ON users USING gin (profile);
```

常用 `psql` 命令：`\l` 列出数据库、`\c dbname` 切换、`\dt` 列出表、`\d+ table` 看表结构、`\du` 看角色、`\x` 展开显示。

## 适用场景

### 适合

- Web 后端、SaaS、多租户系统（RLS 很实用）
- 金融、订单等强事务场景
- 半结构化数据（JSONB）
- 地理信息（PostGIS）
- AI 应用的业务数据与会话存储
- 中等规模分析、全文检索

### 不太适合

- 纯缓存/键值场景 → Redis
- 海量事件分析 → ClickHouse、BigQuery
- 嵌入式/单机小应用 → SQLite
- 极高写入吞吐且不愿分片 → 需要专门架构评估

## 与 MySQL 对比

| 维度 | PostgreSQL | MySQL |
| --- | --- | --- |
| SQL 与类型 | 更完整、更严格 | 常用功能足够 |
| JSON | JSONB，可索引、可查询 | JSON，能力相对有限 |
| 扩展性 | 很强 | 较弱 |
| 复杂查询/分析 | 更有优势 | 一般 |
| 并发模型 | MVCC 成熟 | InnoDB MVCC |
| 运维 | 稍复杂 | 托管产品多、上手简单 |
| 典型场景 | 复杂业务、GIS、AI、多租户 | 传统 Web、读多写少、广泛托管 |

两者都能覆盖大多数 Web 业务。选型更多看团队经验、托管服务和现有基础设施，不必迷信“谁更快”。

## 在 AI Agent 项目中的用法

- **业务数据**：用户、会话、消息、工具调用记录、任务状态
- **JSONB**：存模型返回的灵活结构、工具参数、元数据
- **pgvector**：存 embedding，做 RAG 语义检索

```sql
CREATE EXTENSION IF NOT EXISTS vector;

ALTER TABLE documents ADD COLUMN embedding vector(1536);

CREATE INDEX ON documents
  USING hnsw (embedding vector_cosine_ops);

SELECT id, content
FROM documents
ORDER BY embedding <=> :query_embedding
LIMIT 5;
```

常见组合是 **PostgreSQL 做主库 + Redis 做缓存/限流 + 对象存储放大文件**。

## 常见坑

1. **连接数不是越多越好**：默认 `max_connections=100`，每个连接占内存；生产用连接池。
2. **长事务会阻塞 vacuum**：导致表膨胀，监控 `pg_stat_activity` 和 `pg_stat_user_tables`。
3. **`jsonb` 和 `json` 不一样**：`jsonb` 是二进制、可索引、不保留键顺序；`json` 保留原始文本。
4. **时间字段用 `timestamptz`**：统一存 UTC，避免时区问题。
5. **`text` 和 `varchar(n)` 性能无差别**：需要长度限制时用 `CHECK` 或 `varchar(n)` 都可以。
6. **大表 `COUNT(*)` 可能很慢**：MVCC 需要扫描可见行；大屏统计更适合估算或计数表。
7. **副本不是备份**：误删会同步到副本；要配置 PITR 和定期备份。
8. **大版本升级要测试**：通常需要 `pg_upgrade` 或逻辑迁移。

## 选型建议

要**强事务 + 复杂查询 + 可扩展**，优先 PostgreSQL；要简单键值缓存用 Redis；要海量分析用 ClickHouse；要嵌入式用 SQLite。
