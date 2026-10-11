# db/

双图的 PostgreSQL 落地层：纯 PG（pgx/v5 stdlib 驱动，**没有 SQLite 双支持**——modernc.org/sqlite 只属于 traffic 包），无 ORM，手写参数化 SQL（`$1`）。

## 组织

- `DB` 内嵌 `*sql.DB`；按领域分 Store（`AssetStore` / `CompanyStore`…持有 `*DB`，经 `d.Assets()` 等工厂产出）；简单查询直接写在 `*DB` 方法上。
- `schema.sql`（单文件）随 `//go:embed` 内嵌，按分区注释组织：资产（companies/assets/company_scope）、探索（exploration_nodes/edges/anchors/activity）、LLM、任务（tasks/task_scope/archives）、agents、MCP、可见性、会话、触发器、拦截（intercept_rules/intercept_pending）、findings（含 retests/traffic 绑定/证据快照）、日志与通知。
- 资产图：`assets` 按 type CHECK 六类（root_domain/subdomain/ip/service/app/endpoint），每类独立部分唯一索引，`task_ids BIGINT[]` GIN；`task_scope` 是任务级范围（source: auto|agent|manual）。探索图：`exploration_nodes`（kind: begin/goal/intent/fact/finding/hint/digest，intent 软删）+ `edges`（rel 五种）+ `anchors`（node×asset）+ `activity` 流水（含 token 计数）。

## 幂等迁移（重启即迁移）

`Open()` → `ensureDatabase`（必要时自动 CREATE DATABASE）→ 连接池封顶 32 → `withSchemaMigrationLock` → 整份 schema.sql 幂等 Exec → `seedBuiltins`。因此**任何 schema 改动必须幂等**：

- 新列用 `ADD COLUMN IF NOT EXISTS`；改 CHECK 用 `DO $$` 先检测老库内联约束再重建；索引 `CREATE INDEX IF NOT EXISTS`。
- 一次性 seed 必须用 settings 标志门控 + `ON CONFLICT DO NOTHING`，**不得复活用户已修改/删除的内置数据**。
- 迁移 advisory lock（7337741001）钉在单条 `sql.Conn` 上（防连接池错锁）；长归档事务用 `pg_advisory_xact_lock` 与启动 DDL 互斥。
- DDL 死锁（40P01）有退避重试，保持该行为。

## 代码约定

- 事务：`BeginTx` + `defer tx.Rollback() //nolint:errcheck` + 显式 Commit。
- 错误：`fmt.Errorf("%w")` 包装；PG 错误用 `errors.As(*pgconn.PgError)` 判 code；`updated_at` 由 `set_updated_at()` 触发器维护；脏 IP 用 `try_inet()` 防 22P02。
- 测试为直连 PG 的集成测试：DSN 走 `DSN()`（env `ARTEX_PG_DSN` > config.json，无默认），不可用 `t.Skipf("postgres unavailable")`；TestMain 持 advisory lock 7337741002 与 server/agent/evidence 包在共享 dev 库上串行——改表结构会影响多包测试。
