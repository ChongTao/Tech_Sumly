## find() 和 findOne() 有什么区别

| 方法 | 返回结果 | 没有匹配文档时 |
| :--- | :--- | :--- |
| `find(filter)` | 返回游标（Cursor），可遍历多条匹配文档，并可链式调用 `sort()`、`limit()`、`skip()` 等方法 | 空游标 |
| `findOne(filter)` | 返回一条匹配文档，不返回游标 | `null` |

示例：

```javascript
db.users.find({ status: "active" })
// 返回游标，可逐条读取匹配文档

db.users.findOne({ status: "active" })
// 返回一条文档；没有匹配项时返回 null

db.users.find({ status: "active" }).sort({ createdAt: -1 }).limit(1)
// 按创建时间倒序，返回至多一条文档的游标
```

需要确定返回哪一条文档时，应明确指定排序条件；不排序时，不能依赖 `findOne()` 返回某个固定的匹配文档。`mongosh` 展示 `find()` 的结果时使用游标，当前屏幕显示的条数不代表查询只匹配了这些文档。

## MongoDB 中的 _id 字段

`_id` 是文档的唯一标识，相当于集合内的主键。每个文档都必须有 `_id`；默认情况下，MongoDB 会为集合建立 `_id_` 唯一索引。插入时若未提供 `_id`，MongoDB 会自动生成 `ObjectId`，常见驱动会在发送插入请求前生成它。

`_id` 也可以使用字符串、UUID 等合适的 BSON 值，但必须在集合内保持唯一，并且创建后不能修改。`ObjectId` 长 12 字节，包含秒级时间、随机值和递增计数器；它大致按生成时间排序，但不保证严格的全局先后顺序。需要准确按业务时间排序时，应保存并查询单独的时间字段。

```javascript
// 未指定 _id，MongoDB 会生成 ObjectId
db.users.insertOne({ name: "张三" })

// 也可以自行指定稳定且唯一的值
db.orders.insertOne({ _id: "order-20261013-001", total: 99.5 })
```

分片集群中，如果 `_id` 不是分片键，唯一性约束可能只在单个分片内生效；需要由应用确保跨分片的 `_id` 不重复。

## MongoDB 如何存放图片和视频

根据文件大小和访问方式选择存储方案：

| 方案 | 适用场景 | 说明 |
| :--- | :--- | :--- |
| BSON `Binary` 字段 | 较小文件，且希望文件与业务文档一起读写 | 文件内容以二进制值存入文档；单个 BSON 文档不能超过 16 MiB。 |
| GridFS | 文件超过 16 MiB，或需要分块读写、流式处理 | GridFS 将文件拆分为多个 chunk，并保存文件元数据，适合通过 MongoDB 管理较大文件。 |
| 对象存储 | 大型图片/视频、访问量较高或需要 CDN | 文件放在 S3、MinIO 等对象存储中，MongoDB 保存对象 key、文件大小、类型等元数据。 |

生产环境的大型媒体文件通常放在对象存储中，由 MongoDB 保存元数据和对象 key；需要访问时再生成受控下载地址。只有小文件且确有原子读写需求时，才考虑直接放入文档。不要把文件转成 Base64 字符串存储，Base64 通常会使数据体积增加约三分之一。

## MongoDB 如何确保高可用性

MongoDB 通常通过副本集实现高可用：副本集保存相同的数据集，由 Primary 接收写入，Secondary 异步复制操作日志并应用数据。Primary 故障后，剩余有投票权的成员会发起选举，选出新的 Primary；选举期间可能出现短暂的写入中断。

生产环境通常将至少三个承载数据的节点部署在不同主机或可用区，客户端使用副本集连接串，以便驱动发现新 Primary 并进行故障转移。重要写入可使用 `writeConcern: { w: "majority" }`，让多数节点确认后再返回。仲裁节点只参与投票、不保存数据，不能代替数据副本；副本集也不能代替备份。

## 介绍 MongoDB 中的分片

分片（Sharding）把一个集合的数据分布到多个 Shard 上，用于扩展存储容量和读写吞吐。典型分片集群由以下部分组成：

- **Shard**：保存集合数据；生产环境中每个 Shard 通常是一个副本集。
- **mongos**：查询路由器，接收客户端请求并将其转发到相关 Shard。
- **Config Server**：以副本集形式保存分片集群的元数据和分片范围信息。

分片键决定文档如何分布。选择时应关注基数、数据分布是否均匀，以及常用查询是否包含分片键；分布不均可能造成热点，缺少分片键的查询则可能广播到多个 Shard。分片解决横向扩展问题，副本集负责单个 Shard 的数据冗余和故障切换，两者通常配合使用。

## 介绍 MongoDB 中的副本集

副本集（Replica Set）是一组维护同一数据集的 `mongod` 实例。通常由一个 Primary 和多个 Secondary 组成：Primary 接收写入并记录操作日志（oplog），Secondary 持续复制并应用这些操作。副本集成员通过心跳检测状态；Primary 不可用时，具备投票权的成员会选举新的 Primary。

副本集提供数据冗余和自动故障切换。Secondary 读可分担部分查询，但复制存在延迟，读取可能不是最新数据。仲裁节点只投票、不保存数据。副本集本身不负责把数据拆分到多台机器；需要水平拆分时使用分片集群。

## 介绍 MongoDB 的聚合

聚合管道（Aggregation Pipeline）将文档依次传入多个阶段，在数据库端完成过滤、字段投影、分组、排序、关联等处理。常用阶段包括 `$match`、`$project`、`$group`、`$sort`、`$limit`、`$unwind` 和 `$lookup`。通常应尽早使用 `$match` 减少后续阶段处理的数据量。

示例：统计已支付订单中每个用户的订单总额，并取总额最高的前 10 名：

```javascript
db.orders.aggregate([
  { $match: { status: "PAID" } },
  { $group: { _id: "$userId", totalAmount: { $sum: "$amount" }, orderCount: { $sum: 1 } } },
  { $sort: { totalAmount: -1 } },
  { $limit: 10 }
])
```

## MongoDB 是否支持 ACID 事务

支持。单文档写入具有原子性；MongoDB 也支持跨多个文档和集合的 ACID 事务。多文档事务在副本集上从 MongoDB 4.0 起支持，在分片集群上从 4.2 起支持。事务通过 Session 开启、提交或回滚。

事务会增加资源和延迟开销。建模时优先让需要原子更新的数据位于同一文档；确实需要跨文档保持一致时再使用事务，并尽量缩短事务持续时间。

## 介绍 MongoDB 的 Map-Reduce

Map-Reduce 是一种批量处理模型：`map` 函数把文档映射为键值对，`reduce` 函数按键汇总值，旧接口还支持 `finalize` 阶段。MongoDB 的 Map-Reduce 功能从 5.0 起已弃用；新开发通常应使用聚合管道，后者更容易被数据库优化，也能覆盖多数分组和转换需求。

## MongoDB 的 TTL 索引以及使用

TTL（Time To Live）索引会在后台定期删除达到过期时间的文档，适合会话、临时令牌和短期日志等数据。TTL 索引只能建在单个日期字段上，删除不是精确到期执行，过期文档可能短暂保留；因此不适合要求精确时刻触发的业务任务。

```javascript
// expireAt 到达后过期
db.sessions.createIndex({ expireAt: 1 }, { expireAfterSeconds: 0 })

// createdAt 之后保留 7 天
db.logs.createIndex({ createdAt: 1 }, { expireAfterSeconds: 7 * 24 * 60 * 60 })
```

## 如何优化 MongoDB 的查询

- 先定位慢查询，再用 `explain("executionStats")` 检查执行计划。关注 `COLLSCAN`（集合扫描）、`IXSCAN`（索引扫描）、`totalDocsExamined` 和 `totalKeysExamined`。
- 为高频过滤和排序条件设计索引；复合索引的字段顺序会影响能否命中。索引过多会增加写入和存储成本。
- 用投影只返回需要的字段，并用 `limit()` 控制结果量。大偏移量分页避免反复使用 `skip()`，可改用上次结果的排序键继续查询。
- 聚合时尽早过滤数据；避免无法利用索引的查询条件，并结合实际数据量验证优化效果。

```javascript
db.orders.find({ userId: 1001, status: "PAID" }, { userId: 1, total: 1, createdAt: 1 })
  .sort({ createdAt: -1 })
  .limit(20)
  .explain("executionStats")
```
