# 1 SQL 介绍

SQL（Structured Query Language，结构化查询语言）是关系型数据库的标准语言，用于定义、查询、修改和管理结构化数据。MySQL、PostgreSQL、Oracle、SQL Server 等数据库都以 SQL 作为核心接口，具体语法各有方言差异，但核心概念和写法相通。

## 1.1 核心特点

- 声明式语言：只描述“要什么数据”，由数据库优化器决定访问路径和执行方式。
- 面向集合操作：一条语句作用于行集合，而不是逐行循环处理。
- 标准化程度高：SQL 标准（SQL-92、SQL:1999、SQL:2016 等）被主流数据库广泛实现。
- 功能覆盖完整：建表、查询、修改、事务控制、权限管理都可以用 SQL 表达。
- 与具体数据库解耦：掌握标准 SQL 后，切换数据库主要关注方言差异即可。

## 1.2 基本概念

| 概念 | 说明 |
| :--- | :--- |
| relation | 关系，在数据库中表现为二维表 |
| schema | 模式，数据库对象的结构定义与命名空间 |
| row / tuple | 行（元组），表中的一条记录 |
| column / attribute | 列（属性），记录的一个字段 |
| data type | 数据类型，决定列的取值范围和可执行的操作 |
| constraint | 约束，如主键、唯一、非空、检查、外键 |
| view | 视图，基于查询定义的虚拟表 |
| transaction | 事务，一组要么全部成功、要么全部回滚的操作 |

## 1.3 语句分类

| 类别 | 全称 | 常用语句 | 用途 |
| :--- | :--- | :--- | :--- |
| DDL | Data Definition Language | `CREATE`、`ALTER`、`DROP`、`TRUNCATE` | 定义和管理表、索引等结构 |
| DML | Data Manipulation Language | `INSERT`、`UPDATE`、`DELETE` | 修改表中的数据 |
| DQL | Data Query Language | `SELECT` | 查询数据 |
| TCL | Transaction Control Language | `BEGIN`、`COMMIT`、`ROLLBACK`、`SAVEPOINT` | 管理事务 |
| DCL | Data Control Language | `GRANT`、`REVOKE` | 管理账号权限 |

## 1.4 查询语句要点

`SELECT` 是最常用的语句。它的书写顺序是 `SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT`，但逻辑处理顺序是 FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT。理解这一点，能避免大多数字段引用和别名使用上的错误。

### 1.4.1 多表连接

| 连接类型 | 结果 | 典型场景 |
| :--- | :--- | :--- |
| `INNER JOIN` | 只保留两侧都匹配的行 | 查询有订单的用户 |
| `LEFT JOIN` | 保留左表全部行，右表无匹配则补 `NULL` | 统计包含未下单用户的数据 |
| `RIGHT JOIN` | 保留右表全部行，左表无匹配则补 `NULL` | 语义上可用交换表顺序的 `LEFT JOIN` 代替 |
| `FULL JOIN` | 保留两侧全部行，无匹配一侧补 `NULL` | 对账、比对两个数据源的差异 |
| `CROSS JOIN` | 笛卡尔积 | 生成组合，如日期 × 商品 |

连接时应显式写出 `ON` 条件，避免依赖隐式连接；连接列的类型和字符集保持一致，否则可能无法使用索引。

### 1.4.2 聚合与分组

- `COUNT`、`SUM`、`AVG`、`MIN`、`MAX` 等聚合函数作用于行集合；`COUNT(*)` 统计总行数，`COUNT(col)` 不统计该列为 `NULL` 的行。
- 使用 `GROUP BY` 后，`SELECT` 中未参与聚合的列都应出现在分组键中，否则结果不确定或会被数据库拒绝。
- 对分组结果过滤用 `HAVING`，对原始行过滤用 `WHERE`；能在 `WHERE` 中提前过滤的条件不要放到 `HAVING`。

### 1.4.3 子查询与窗口函数

- 子查询可以出现在 `FROM`、`WHERE`、`SELECT` 等位置；能用连接直接表达的简单子查询优先用连接，便于优化器处理。
- `IN` 适合小集合；大集合或需要关联条件时改用 `JOIN` 或 `EXISTS`。`NOT IN` 遇到 `NULL` 容易得到意外结果，建议改用 `NOT EXISTS`。
- 窗口函数（如 `ROW_NUMBER()`、`RANK()`、`LAG()`、`SUM() OVER (...)`）在不折叠行的前提下完成排序、排名和分组统计，适合 Top N、环比、移动平均等分析场景。

## 1.5 常见方言差异

标准 SQL 之外，各数据库存在方言差异，跨库编写 SQL 时需要确认：

| 差异点 | MySQL / PostgreSQL | 其他数据库示例 |
| :--- | :--- | :--- |
| 分页 | `LIMIT n OFFSET m` | SQL Server 用 `OFFSET ... FETCH`，Oracle 旧版本用 `ROWNUM` |
| 标识符引用 | MySQL 用反引号，PostgreSQL 用双引号 | SQL Server 用方括号 |
| 自增主键 | MySQL `AUTO_INCREMENT`，PostgreSQL `IDENTITY` 或 `SERIAL` | SQL Server `IDENTITY`，Oracle 序列或 `IDENTITY` |
| 字符串拼接 | MySQL 用 `CONCAT()`，PostgreSQL 用 `||` 或 `CONCAT()` | Oracle 用 `||` |
| 布尔类型 | PostgreSQL 原生 `BOOLEAN`，MySQL 常用 `TINYINT(1)` | SQL Server 用 `BIT` |
| 当前时间 | 通用写法 `CURRENT_TIMESTAMP` | 各有函数，如 `NOW()`、`SYSDATE`、`GETDATE()` |

## 1.6 使用建议

- 明确指定需要的列，避免 `SELECT *`：减少多余传输，也避免表结构变化引发隐患。
- 执行 `UPDATE`、`DELETE` 前，先用相同条件的 `SELECT` 确认影响范围；生产变更放在事务中执行并保留回滚手段。
- 应用代码一律使用参数化查询或预编译语句，不拼接 SQL，防止 SQL 注入。
- 大表查询依赖索引并使用高效分页；深分页改用基于排序键的游标分页（`WHERE key > :last_key ORDER BY key LIMIT n`）。
- 保持书写规范：关键字大写、表名列名小写、统一缩进，便于评审和排查问题。