# 2 SQL 常用命令

以下示例使用 MySQL 与 PostgreSQL 通用的标准 SQL 语法，以 `users` 和 `orders` 两张表作为示例数据。执行 `DROP`、`TRUNCATE`、`DELETE`、`UPDATE` 前，应先用 `SELECT` 核对影响范围。

## 2.1 示例数据

```sql
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(64) NOT NULL,
  email VARCHAR(128) NOT NULL UNIQUE,
  status SMALLINT NOT NULL DEFAULT 1,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT NOT NULL,
  status VARCHAR(16) NOT NULL,
  amount DECIMAL(12, 2) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (id, name, email, status) VALUES
  (1, '张三', 'zhangsan@example.com', 1),
  (2, '李四', 'lisi@example.com', 1),
  (3, '王五', 'wangwu@example.com', 0);

INSERT INTO orders (id, user_id, status, amount) VALUES
  (101, 1, 'paid', 99.90),
  (102, 1, 'shipped', 45.00),
  (103, 2, 'paid', 120.00);
```

## 2.2 查询 SELECT

```sql
-- 查询全部列
SELECT * FROM users;

-- 查询指定列并起别名
SELECT name AS user_name, email FROM users;

-- 条件查询
SELECT id, name FROM users WHERE status = 1;

-- 模糊匹配与多条件组合
SELECT id, name FROM users
WHERE name LIKE '张%' AND status = 1;

-- 排序与分页（每页 10 条，取第 2 页）
SELECT id, name, created_at FROM users
ORDER BY created_at DESC
LIMIT 10 OFFSET 10;
```

## 2.3 聚合与分组

```sql
-- 总用户数与启用状态用户数
SELECT
  COUNT(*) AS total,
  COUNT(CASE WHEN status = 1 THEN 1 END) AS active
FROM users;

-- 每个用户的订单数与总金额
SELECT user_id, COUNT(*) AS order_count, SUM(amount) AS total_amount
FROM orders
GROUP BY user_id;

-- 只保留订单总金额大于 100 的用户
SELECT user_id, SUM(amount) AS total_amount
FROM orders
GROUP BY user_id
HAVING SUM(amount) > 100;
```

## 2.4 多表连接

```sql
-- 内连接：只查询有订单的用户及其订单
SELECT u.name, o.id AS order_id, o.amount
FROM users u
INNER JOIN orders o ON o.user_id = u.id;

-- 左连接：查询全部用户，无订单的用户订单数为 0
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.name;
```

## 2.5 插入与修改数据

```sql
-- 插入一行
INSERT INTO users (id, name, email) VALUES (4, '赵六', 'zhaoliu@example.com');

-- 更新指定行（先用 SELECT ... WHERE id = 4 核对范围）
UPDATE users SET status = 0 WHERE id = 4;

-- 删除指定行
DELETE FROM users WHERE id = 4;
```

## 2.6 表结构操作 DDL

DDL 示例使用单独的 `demo` 表，避免影响示例数据：

```sql
CREATE TABLE demo (
  id INT PRIMARY KEY,
  note VARCHAR(64)
);

-- 增加列
ALTER TABLE demo ADD COLUMN created_at TIMESTAMP;

-- 删除列
ALTER TABLE demo DROP COLUMN note;

-- 重命名表
ALTER TABLE demo RENAME TO demo2;

-- 清空数据（保留表结构；MySQL 中隐式提交，无法回滚）
TRUNCATE TABLE demo2;

-- 删除表
DROP TABLE demo2;
```

## 2.7 索引与视图

```sql
-- 创建索引
CREATE INDEX idx_orders_user_id ON orders (user_id);

-- 删除索引：MySQL 需要指定表名，PostgreSQL 直接按索引名删除
DROP INDEX idx_orders_user_id ON orders;  -- MySQL
DROP INDEX idx_orders_user_id;            -- PostgreSQL
```

```sql
-- 创建视图：保存常用查询
CREATE VIEW active_users AS
SELECT id, name, email FROM users WHERE status = 1;

SELECT * FROM active_users;

-- 删除视图
DROP VIEW active_users;
```

## 2.8 事务控制

```sql
-- 一组操作要么全部提交，要么全部回滚
BEGIN;
UPDATE users SET status = 0 WHERE id = 2;
INSERT INTO orders (id, user_id, status, amount) VALUES (104, 2, 'paid', 30.00);
COMMIT;

-- 发现误删时回滚
BEGIN;
DELETE FROM orders WHERE id = 103;
ROLLBACK;
```