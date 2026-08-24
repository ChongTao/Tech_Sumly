# 1 PostgreSQL 操作

以下示例以 PostgreSQL 16 为参考。`psql` 中以 `\\` 开头的是客户端元命令，不属于 SQL。

## 1.1 连接与数据库操作

```bash
psql -h 127.0.0.1 -p 5432 -U app_user -d postgres
```

```sql
CREATE DATABASE shop;
-- 切换到 shop 数据库后执行：
CREATE SCHEMA app;
SET search_path TO app;
```

常用 `psql` 命令：

```text
\l       查看数据库
\dn      查看 schema
\dt app.* 查看 app schema 中的表
\d app.users 查看表结构
```

## 1.2 建表与约束

```sql
CREATE TABLE app.users (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email varchar(128) NOT NULL UNIQUE,
  name varchar(64) NOT NULL,
  profile jsonb NOT NULL DEFAULT '{}'::jsonb,
  status smallint NOT NULL DEFAULT 1,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_status_created_at
  ON app.users (status, created_at DESC);
```

## 1.3 CRUD 与 UPSERT

```sql
INSERT INTO app.users (email, name, profile)
VALUES ('tom@example.com', 'Tom', '{"city":"Shanghai"}');

SELECT id, email, profile->>'city' AS city
FROM app.users
WHERE status = 1
ORDER BY id DESC
LIMIT 20;

UPDATE app.users SET status = 0 WHERE id = 1;
DELETE FROM app.users WHERE id = 1;

INSERT INTO app.users (email, name)
VALUES ('tom@example.com', 'Tom')
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;
```

## 1.4 聚合、关联与分页

```sql
SELECT status, count(*) AS user_count
FROM app.users
GROUP BY status;

SELECT o.id, u.name, o.amount
FROM app.orders AS o
JOIN app.users AS u ON u.id = o.user_id
WHERE o.status = 'PAID';

-- 游标分页
SELECT id, email, name
FROM app.users
WHERE id > 1000
ORDER BY id
LIMIT 20;
```

## 1.5 事务与锁

```sql
BEGIN;

SELECT balance FROM app.accounts WHERE id = 1 FOR UPDATE;
UPDATE app.accounts SET balance = balance - 100
WHERE id = 1 AND balance >= 100;
UPDATE app.accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
-- 发生错误时使用 ROLLBACK;
```

事务应尽量短。需要并发领取任务时，可使用 `FOR UPDATE SKIP LOCKED` 跳过已被其他事务锁定的行。

```sql
SELECT id
FROM app.jobs
WHERE status = 'READY'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

## 1.6 执行计划、维护与备份

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, amount
FROM app.orders
WHERE user_id = 1001
  AND created_at >= timestamptz '2026-01-01 00:00:00+00'
ORDER BY created_at DESC
LIMIT 20;

VACUUM (ANALYZE) app.orders;
```

```bash
pg_dump -h 127.0.0.1 -U app_user -Fc -d shop -f shop.dump
pg_restore -h 127.0.0.1 -U app_user -d shop_restore shop.dump
```

不要在高并发生产库上随意执行 `VACUUM FULL`，它会重写表并需要更强的锁；常规维护通常交给 autovacuum，必要时执行普通 `VACUUM (ANALYZE)`。
