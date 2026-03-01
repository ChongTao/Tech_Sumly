# 一 MongoDB

[MongoDB](https://www.runoob.com/mongodb/mongodb-intro.html) 是一个**面向文档（Document-Oriented）的 NoSQL 数据库**，采用 BSON（Binary JSON）格式存储数据，适合高并发、高扩展、海量数据场景。核心特点是：

- 文档模型（JSON 风格）
- 动态 Schema（无固定表结构）
- 高可扩展（分片）
- 高可用（副本集）
- 丰富索引能力
- 强大的聚合框架

MongoDB使用集合（Collections）来组织文档（Documents），每个文档都是由键值对组成的。

- **数据库（Database）**：存储数据的容器，类似于关系型数据库中的数据库。
- **集合（Collection）**：数据库中的一个集合，类似于关系型数据库中的表。
- **文档（Document）**：集合中的一个数据记录，类似于关系型数据库中的行（row），以 BSON 格式存储

## 1.1 MongoDB概念

| SQL 术语/概念 | MongoDB 术语/概念 | 解释/说明                           |
| :------------ | :---------------- | :---------------------------------- |
| database      | database          | 数据库                              |
| table         | collection        | 数据库表/集合                       |
| row           | document          | 数据记录行/文档                     |
| column        | field             | 数据字段/域                         |
| index         | index             | 索引                                |
| table joins   |                   | 表连接,MongoDB不支持                |
| primary key   | primary key       | 主键,MongoDB自动将_id字段设置为主键 |
