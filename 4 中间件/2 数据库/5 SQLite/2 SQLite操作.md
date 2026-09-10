# 2 SQLite 操作

## 2.1 创建数据库与查看结构

使用 SQLite 命令行时，数据库文件不存在会在首次打开时创建：

```bash
sqlite3 app.db
```

常用命令以点号开头，不属于 SQL：

```sql
.databases
.tables
.schema users
.headers on
.mode column
```

## 2.2 建表与增删改查

```sql
CREATE TABLE IF NOT EXISTS notes (
  id INTEGER PRIMARY KEY,
  title TEXT NOT NULL,
  content TEXT NOT NULL DEFAULT '',
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO notes (title, content)
VALUES ('SQLite 笔记', '事务与索引');

SELECT id, title, created_at
FROM notes
WHERE title LIKE 'SQLite%'
ORDER BY id DESC
LIMIT 20;

UPDATE notes SET content = '事务、索引与 WAL' WHERE id = 1;
DELETE FROM notes WHERE id = 1;
```

生产代码应使用参数绑定，避免 SQL 注入并复用预编译语句：

```sql
SELECT * FROM notes WHERE id = ?;
```

## 2.3 事务与批量写入

```sql
BEGIN IMMEDIATE;
INSERT INTO notes (title, content) VALUES ('A', '内容 A');
INSERT INTO notes (title, content) VALUES ('B', '内容 B');
COMMIT;
```

发生异常时执行 `ROLLBACK`。批量写入应放在一个合理大小的事务中；逐条自动提交会产生大量日志刷盘和锁竞争。

## 2.4 查询计划与维护

```sql
EXPLAIN QUERY PLAN
SELECT * FROM notes WHERE title = 'SQLite 笔记';

CREATE INDEX IF NOT EXISTS idx_notes_title ON notes(title);
ANALYZE;
PRAGMA integrity_check;
VACUUM;
```

`VACUUM` 会重建数据库文件，可能需要额外磁盘空间并持有较强锁，适合在维护窗口执行；WAL 模式下还应关注 checkpoint 和 WAL 文件大小。

## 2.5 导出、导入与备份

SQLite 命令行可以导出可执行 SQL：

```sql
.output backup.sql
.dump
.output stdout
```

也可以使用在线备份命令：

```sql
.backup 'backup.db'
```

恢复导出的 SQL：

```bash
sqlite3 restored.db < backup.sql
```

在应用中应优先调用驱动提供的参数绑定、事务和在线备份 API，并对备份执行可用性校验。
