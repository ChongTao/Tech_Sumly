# 1 MySQL 操作

以下示例以 MySQL 8.0、InnoDB 为准。执行 `DROP`、`DELETE` 和 `UPDATE` 前，应先用 `SELECT` 核对影响范围。

## 1.1 连接与数据库操作

```bash
mysql -h 127.0.0.1 -P 3306 -u app_user -p
```

```sql
SHOW DATABASES;
CREATE DATABASE shop DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
USE shop;
DROP DATABASE shop;
```

## 1.2 建表与约束

```sql
CREATE TABLE users (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  email VARCHAR(128) NOT NULL,
  name VARCHAR(64) NOT NULL,
  status TINYINT NOT NULL DEFAULT 1,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_users_email (email),
  KEY idx_users_status_created_at (status, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

SHOW CREATE TABLE users;
DESC users;
ALTER TABLE users ADD COLUMN phone VARCHAR(32) NULL;
```

### 1.2.1 外键定义与维护

以下示例用订单关联用户。父表 `users.id` 是主键；`orders.user_id` 建立索引后既能满足外键要求，也能加速按用户查询订单。删除用户被禁止，删除订单时其订单项会一并删除。

```sql
CREATE TABLE orders (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  user_id BIGINT UNSIGNED NOT NULL,
  status VARCHAR(16) NOT NULL,
  amount DECIMAL(12, 2) NOT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_orders_user_id (user_id),
  CONSTRAINT fk_orders_user
    FOREIGN KEY (user_id) REFERENCES users (id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE order_items (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  order_id BIGINT UNSIGNED NOT NULL,
  product_name VARCHAR(128) NOT NULL,
  quantity INT UNSIGNED NOT NULL,
  PRIMARY KEY (id),
  KEY idx_order_items_order_id (order_id),
  CONSTRAINT fk_order_items_order
    FOREIGN KEY (order_id) REFERENCES orders (id)
    ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

用 `SHOW CREATE TABLE orders;` 查看约束定义；`INFORMATION_SCHEMA.KEY_COLUMN_USAGE` 可查询库内的外键关系。添加或删除外键时需使用约束名，且应先确认现有数据没有孤儿记录：

```sql
SELECT o.id, o.user_id
FROM orders AS o
LEFT JOIN users AS u ON u.id = o.user_id
WHERE u.id IS NULL;

ALTER TABLE orders DROP FOREIGN KEY fk_orders_user;

ALTER TABLE orders
  ADD CONSTRAINT fk_orders_user
  FOREIGN KEY (user_id) REFERENCES users (id)
  ON DELETE RESTRICT ON UPDATE RESTRICT;
```

`DROP FOREIGN KEY` 不会删除为外键单独创建的索引；执行 `SHOW INDEX FROM orders;` 确认后，再按索引名执行 `DROP INDEX`。对已有大表增加外键会校验存量数据并可能影响 DDL 和写入，应在演练后选择合适的变更窗口。

## 1.3 CRUD

```sql
INSERT INTO users (email, name) VALUES ('tom@example.com', 'Tom');
INSERT INTO users (email, name) VALUES
  ('alice@example.com', 'Alice'),
  ('bob@example.com', 'Bob');

SELECT id, email, name FROM users WHERE status = 1 ORDER BY id DESC LIMIT 20;
UPDATE users SET status = 0 WHERE id = 1;
DELETE FROM users WHERE id = 1;
```

使用参数化查询传递用户输入，避免通过字符串拼接构造 SQL，从而防止 SQL 注入。

## 1.4 聚合、关联与分页

```sql
SELECT status, COUNT(*) AS user_count
FROM users
GROUP BY status;

SELECT o.id, u.name, o.amount
FROM orders AS o
JOIN users AS u ON u.id = o.user_id
WHERE o.status = 'PAID';

-- 游标分页：记录上一页最后一条 id
SELECT id, email, name
FROM users
WHERE id > 1000
ORDER BY id
LIMIT 20;
```

## 1.5 事务与锁

```sql
START TRANSACTION;

SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
UPDATE accounts SET balance = balance - 100 WHERE id = 1 AND balance >= 100;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
-- 发生错误时使用 ROLLBACK;
```

`SELECT ... FOR UPDATE` 会对读到的记录加排他锁，应在事务内尽快完成业务操作并提交。扣减余额、库存等操作还应通过条件更新检查结果行数，避免出现负数。

## 1.6 索引与执行计划

```sql
CREATE INDEX idx_orders_user_created_at ON orders (user_id, created_at);
SHOW INDEX FROM orders;
EXPLAIN ANALYZE
SELECT id, amount
FROM orders
WHERE user_id = 1001
  AND created_at >= '2026-01-01'
ORDER BY created_at DESC
LIMIT 20;
DROP INDEX idx_orders_user_created_at ON orders;
```

### 1.6.1 常见索引操作与排查

```sql
-- 联合索引：支持 user_id = ?、user_id = ? AND status = ? 等最左前缀查询
CREATE INDEX idx_orders_user_status_created_at
  ON orders (user_id, status, created_at);

-- 唯一约束也会创建唯一索引
ALTER TABLE users ADD CONSTRAINT uk_users_phone UNIQUE (phone);

-- 检查索引与建表定义
SHOW INDEX FROM orders;
SHOW CREATE TABLE orders;

-- MySQL 8.0：先把索引设为不可见，观察是否存在性能回退，再决定是否删除
ALTER TABLE orders ALTER INDEX idx_orders_user_status_created_at INVISIBLE;
ALTER TABLE orders ALTER INDEX idx_orders_user_status_created_at VISIBLE;

DROP INDEX idx_orders_user_status_created_at ON orders;
```

索引顺序应服务于实际谓词，而不是字段的书写顺序。例如 `(user_id, status, created_at)` 可以支持按 `user_id` 查询，也可以支持按 `user_id` 和 `status` 查询；仅按 `status` 或仅按 `created_at` 查询通常不能有效使用该索引。删除索引前先用 `EXPLAIN ANALYZE` 对比典型 SQL，并检查它是否也被外键、唯一约束或其他查询依赖。

重点关注扫描行数、是否命中预期索引、是否出现临时表或额外排序。分析后再调整索引或 SQL。
