# 1 SQLite 介绍

SQLite 是嵌入式关系型数据库。它将整个数据库保存在一个文件中，通过应用进程直接读写，不需要独立的数据库服务器。SQLite 支持 SQL、事务、索引、视图和触发器，适合桌面应用、移动应用、边缘设备、测试环境、单机工具和低到中等并发的本地服务。

## 1.1 核心特点

- 零配置、单文件部署，数据库创建和迁移成本低。
- 支持 ACID 事务，提交记录通过 rollback journal 或 WAL 保证崩溃恢复。
- 关系模型和 SQL 能力完整，支持 CTE、窗口函数、JSON 函数和 FTS5 等扩展能力。
- 无独立服务进程和用户权限系统，应用通过文件权限控制数据库访问。
- 读并发较好，但同一数据库文件同一时刻通常只有一个写事务。

SQLite 不是把数据全部放在内存中的数据库；默认数据保存在磁盘文件中，页缓存由 SQLite 管理。它也不是为跨多节点水平扩展、复杂用户权限或大量并发写入设计的数据库。

## 1.2 基本概念

| 概念 | 说明 |
| :--- | :--- |
| database file | 数据库文件，包含表、索引、触发器和元数据 |
| page | SQLite 读写和加锁的基本存储单位，常见大小为 4096 字节 |
| rowid | 大多数表隐含的 64 位整数行标识；`INTEGER PRIMARY KEY` 可直接作为其别名 |
| type affinity | 列对存入值的推荐类型，不等同于强制类型约束 |
| journal | 事务日志；常见模式为 rollback journal 和 WAL |
| connection | 应用与数据库文件之间的连接，事务和锁状态与连接相关 |

## 1.3 存储与类型系统

SQLite 的存储类只有 `NULL`、`INTEGER`、`REAL`、`TEXT` 和 `BLOB`。列声明中的 `VARCHAR(255)`、`BOOLEAN`、`DATETIME` 等名称主要用于计算 type affinity，通常不会像 MySQL 那样严格限制值的类型。

```sql
CREATE TABLE users (
  id INTEGER PRIMARY KEY,
  name TEXT NOT NULL,
  age INTEGER CHECK (age >= 0),
  enabled INTEGER NOT NULL DEFAULT 1,
  profile TEXT
);
```

实践中应注意：

- 布尔值通常使用 `INTEGER` 存储 `0/1`，必要时用 `CHECK (enabled IN (0, 1))` 约束。
- 日期时间通常保存为 ISO-8601 文本、Unix 时间戳整数或儒略日实数，项目内应统一格式。
- 金额优先保存为最小货币单位的整数，避免浮点数精度问题。
- 需要强类型约束时，使用 `STRICT` 表（SQLite 3.37+）和 `CHECK` 约束。

## 1.4 主键、rowid 与索引

`INTEGER PRIMARY KEY` 是 rowid 表的特殊定义：插入时可自动分配整数，并且不需要再维护一棵独立的主键索引。`INT PRIMARY KEY`、`BIGINT PRIMARY KEY` 等写法则不具备同样的 rowid 别名语义。

```sql
CREATE TABLE orders (
  id INTEGER PRIMARY KEY,
  user_id INTEGER NOT NULL,
  created_at TEXT NOT NULL,
  status TEXT NOT NULL,
  UNIQUE (user_id, created_at)
);

CREATE INDEX idx_orders_user_time
ON orders (user_id, created_at DESC);
```

- 索引适合等值、范围、连接和排序查询，但会增加磁盘空间与写入成本。
- 联合索引要根据真实查询条件设计，通常将等值过滤列放在范围列之前。
- `EXPLAIN QUERY PLAN` 可检查 SQLite 是否使用了预期索引。
- 不要仅因为列出现在 `WHERE` 中就创建索引；应结合数据分布和查询频率验证。

## 1.5 事务、锁与 WAL

SQLite 的事务默认是串行化写入。常见事务模式如下：

| 模式 | 行为 | 适用场景 |
| :--- | :--- | :--- |
| `DEFERRED` | 首次读写时再获取锁 | 默认模式，读事务较多 |
| `IMMEDIATE` | 开始事务时尝试获取写锁 | 希望尽早发现写入冲突 |
| `EXCLUSIVE` | 获取更强的独占锁 | 特殊批处理或需要阻止其他连接访问 |

默认的 rollback journal 模式下，写入可能阻塞读取。启用 WAL 后，读者可以继续读取旧快照，写入先追加到 WAL 文件，通常更适合读多写少的应用：

```sql
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 5000;
```

WAL 不是多写并发方案，仍然只有一个写者；长时间不结束的读事务还可能阻碍 checkpoint，导致 WAL 文件持续增长。事务应尽量短小，并避免在事务中等待网络或用户操作。

## 1.6 外键与完整性

SQLite 支持外键，但每个连接默认可能未启用。应用建立连接后应显式执行：

```sql
PRAGMA foreign_keys = ON;
```

```sql
CREATE TABLE order_items (
  id INTEGER PRIMARY KEY,
  order_id INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  quantity INTEGER NOT NULL CHECK (quantity > 0)
);
```

外键保证引用完整性，不能替代索引。被引用的父表键应有主键或唯一索引，子表外键列也通常应建立索引，以降低级联操作和连接查询成本。

## 1.7 备份与适用边界

- 不要在数据库仍可能写入时只使用普通文件复制作为唯一备份方式；优先使用 SQLite 在线备份 API、`sqlite3 .backup`，或在受控停机后复制文件。
- 备份应包含数据库文件及必要的迁移版本信息，并定期验证恢复结果。
- 单文件数据库适合本地数据、缓存之外的轻量持久化和嵌入式业务数据。
- 多实例高并发写入、跨主机共享写入、复杂权限隔离、分库分表和高可用复制场景，应评估 MySQL 或 PostgreSQL 等服务型数据库。
