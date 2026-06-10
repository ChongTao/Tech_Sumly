# 1 MongoDB安装

MongoDB的下载路径：https://www.mongodb.com/try/download/community window平台安装指导见：[Windows 平台安装 MongoDB | 菜鸟教程](https://www.runoob.com/mongodb/mongodb-window-install.html)。安装完后配置完环境变量后，在data目录下创建db目录（安装版本是8.2.5）。

- 启动MongoDB服务器

  ```sh
  mongod --dbpath D:\devApp\MongoDB\data\db
  ```

- 连接MongoDB服务器（注意如果使用MongoDB Shell需要单独下载[MongoDB Shell Download | MongoDB](https://www.mongodb.com/try/download/shell)）

  ```sh
  mongosh
  ```

# 2 基础连接

- `mongosh`：本地默认连接MongoDB（端口号27017）, 示例`mongosh`

- `mongosh <连接字符串>`：远程连接

  ```sh
  mongosh "mongodb://user:password@ip:27017/mydb?authSource=admin"
  ```

- `exit/quit`：退出MongoDB Shell

- `mongosh --version`: 查看MongoDB Shell的版本。

# 3 数据库操作

- `show dbs / show databases`: 列出所有数据库。
- `use <数据库名>`： 切换/创建数据库。
- `db`: 查看当前所属数据库。
- `db.dropDatabase()`: 删除当前数据库（注意不会退出当前数据库）。
- `db.stats()`: 查看当前数据统计信息（大小、文档数）。

# 4 集合操作（类似表操作）

- `show collections / show tables`: 列出当前库所有集合。

- `db.createCollection(<集合名>,[选项])`：创建集合。

  ```sh
  db.createCollection("logs", {capped: true, size: 1024*1024})
  ```

- `db.<集合名>.drop()`: 删除指定集合。

  ```sh
  db.logs.drop()
  ```

- `db.<集合名>.renameCollection(<新名>)`：重命名集合

- `db.<集合名>.stats()`: 查看集合统计信息。

# 5 文档操作

## 5.1 插入文档

| 命令                                         | 功能说明                    | 示例                                                         |
| :------------------------------------------- | :-------------------------- | :----------------------------------------------------------- |
| `db.<集合名>.insertOne(<文档>)`              | 插入单个文档                | `db.users.insertOne({name: "张三", age: 25, email: "zhangsan@test.com"})` |
| `db.<集合名>.insertMany([<文档1>, <文档2>])` | 插入多个文档                | `db.users.insertMany([{name: "李四", age: 30}, {name: "王五", age: 28}])` |
| `db.<集合名>.insert(<文档/数组>)`            | 兼容旧版插入（单 / 多文档） | `db.users.insert({name: "赵六", age: 22})`                   |

## 5.2 查询文档

| 命令                                 | 功能说明                           | 示例                                                         |
| :----------------------------------- | :--------------------------------- | :----------------------------------------------------------- |
| `db.<集合名>.find()`                 | 查询集合所有文档（默认返回 20 条） | `db.users.find()`（格式化：`db.users.find().pretty()`）      |
| `db.<集合名>.find(<条件>)`           | 按条件查询                         | `db.users.find({age: 25})`（年龄 = 25）`db.users.find({age: {$gt: 25}})`（年龄 > 25） |
| `db.<集合名>.findOne(<条件>)`        | 查询符合条件的第一个文档           | `db.users.findOne({name: "张三"})`                           |
| `db.<集合名>.find(<条件>, <投影>)`   | 指定返回字段（1 显示，0 隐藏）     | `db.users.find({age: {$gt:25}}, {name:1, age:1, _id:0})`（仅返回姓名、年龄） |
| `db.<集合名>.countDocuments(<条件>)` | 统计符合条件的文档数               | `db.users.countDocuments({age: {$gte:20}})`                  |
| `db.<集合名>.find().sort(<排序>)`    | 排序（1 升序，-1 降序）            | `db.users.find().sort({age: -1})`（按年龄降序）              |
| `db.<集合名>.find().limit(<数量>)`   | 限制返回条数                       | `db.users.find().limit(5)`（仅返回前 5 条）                  |
| `db.<集合名>.find().skip(<数量>)`    | 跳过指定条数（分页用）             | `db.users.find().skip(10).limit(5)`（第 11-15 条）           |

## 5.3 更新文档

| 命令                                         | 功能说明                             | 示例                                                         |
| :------------------------------------------- | :----------------------------------- | :----------------------------------------------------------- |
| `db.<集合名>.updateOne(<条件>, <更新操作>)`  | 更新符合条件的第一个文档             | `db.users.updateOne({name: "张三"}, {$set: {age: 26, email: "zs@test.com"}})` |
| `db.<集合名>.updateMany(<条件>, <更新操作>)` | 更新所有符合条件的文档               | `db.users.updateMany({age: {$lt:30}}, {$inc: {age: 1}})`（年龄 < 30 的都 + 1） |
| `db.<集合名>.replaceOne(<条件>, <新文档>)`   | 替换符合条件的第一个文档（全量替换） | `db.users.replaceOne({name: "李四"}, {name: "李四", age: 31, email: "ls@test.com"})` |

## 5.4. 删除文档

| 命令                             | 功能说明                         | 示例                                   |
| :------------------------------- | :------------------------------- | :------------------------------------- |
| `db.<集合名>.deleteOne(<条件>)`  | 删除符合条件的第一个文档         | `db.users.deleteOne({name: "赵六"})`   |
| `db.<集合名>.deleteMany(<条件>)` | 删除所有符合条件的文档           | `db.users.deleteMany({age: {$lt:20}})` |
| `db.<集合名>.deleteMany({})`     | 删除集合所有文档（保留集合结构） | `db.users.deleteMany({})`              |

# 6 索引操作（Index，优化查询性能）

| 命令                                      | 功能说明                           | 示例                                                         |
| :---------------------------------------- | :--------------------------------- | :----------------------------------------------------------- |
| `db.<集合名>.createIndex(<字段>, <选项>)` | 创建索引（1 升序，-1 降序）        | `db.users.createIndex({name: 1})`（单字段升序索引）`db.users.createIndex({name:1, age:-1})`（复合索引）`db.users.createIndex({email:1}, {unique: true})`（唯一索引） |
| `db.<集合名>.getIndexes()`                | 查看集合所有索引                   | `db.users.getIndexes()`                                      |
| `db.<集合名>.dropIndex(<索引名>)`         | 删除指定索引                       | `db.users.dropIndex("name_1")`（删除 name 的单字段索引）     |
| `db.<集合名>.dropIndexes()`               | 删除所有索引（保留默认的_id 索引） | `db.users.dropIndexes()`                                     |

# 7 常用查询操作符（核心）

| 操作符                            | 功能               | 示例                                                         |
| :-------------------------------- | :----------------- | :----------------------------------------------------------- |
| `$eq`                             | 等于（默认可省略） | `db.users.find({age: {$eq:25}})` → 等价于 `db.users.find({age:25})` |
| `$ne`                             | 不等于             | `db.users.find({age: {$ne:25}})`                             |
| `$gt`/`$gte`                      | 大于 / 大于等于    | `db.users.find({age: {$gt:25}})`                             |
| `$lt`/`$lte`                      | 小于 / 小于等于    | `db.users.find({age: {$lt:30}})`                             |
| `$in`                             | 包含在指定数组中   | `db.users.find({age: {$in: [25,28,30]}})`                    |
| `$nin`                            | 不包含在指定数组中 | `db.users.find({age: {$nin: [20,21]}})`                      |
| `$and`                            | 多条件且           | `db.users.find({$and: [{age:{$gt:25}}, {name: "张三"}]})`    |
| `$or`                             | 多条件或           | `db.users.find({$or: [{age:25}, {name: "李四"}]})`           |
| `$like`（MongoDB 无原生，用正则） | 模糊查询           | `db.users.find({name: /张/})`（包含 “张”）`db.users.find({name: /^张/})`（以 “张” 开头） |

# 8 用户与权限管理（安全）

| 命令                                        | 功能说明                             | 示例                                                         |
| :------------------------------------------ | :----------------------------------- | :----------------------------------------------------------- |
| `use admin`                                 | 切换到管理员数据库（权限管理默认库） | `use admin`                                                  |
| `db.createUser(<用户配置>)`                 | 创建用户                             | `db.createUser({user: "admin", pwd: "123456", roles: [{role: "root", db: "admin"}]})`（超级管理员）`db.createUser({user: "mydb_user", pwd: "123456", roles: [{role: "readWrite", db: "mydb"}]})`（mydb 读写权限） |
| `db.auth(<用户名>, <密码>)`                 | 验证用户（登录）                     | `db.auth("admin", "123456")`（返回 1 表示成功）              |
| `db.dropUser(<用户名>)`                     | 删除指定用户                         | `db.dropUser("mydb_user")`                                   |
| `db.changeUserPassword(<用户名>, <新密码>)` | 修改用户密码                         | `db.changeUserPassword("admin", "654321")`                   |

# 9 注意事项

1. MongoDB 的`_id`字段：每个文档默认自动生成唯一`_id`（ObjectId 类型），可手动指定但需保证唯一。
2. 批量操作注意：`insertMany`/`updateMany`/`deleteMany`执行前建议先通过`find`验证条件，避免误操作。
3. 分页最佳实践：`skip()+limit()`适合小数据量，大数据量建议用`_id`或时间戳排序后分页（如`find({_id: {$gt: last_id}}).limit(10)`）。
4. 索引优化：高频查询字段建议创建索引，避免全集合扫描；唯一索引可防止重复数据（如用户邮箱）。

```
db.collection.find(query, projection)
```

# 10 聚合管道

MongoDB 的聚合管道适合做统计分析、字段转换、分组汇总和多阶段数据处理。相比在应用层拉全量数据再计算，聚合通常更高效，也更便于复用。

## 10.1 常用阶段

| 阶段 | 作用 | 示例 |
| :--- | :--- | :--- |
| `$match` | 过滤文档，尽量前置以减少扫描量 | `{ $match: { status: "PAID" } }` |
| `$project` | 选择或重命名字段 | `{ $project: { userId: 1, amount: 1 } }` |
| `$group` | 分组聚合 | `{ $group: { _id: "$userId", total: { $sum: "$amount" } } }` |
| `$sort` | 排序 | `{ $sort: { total: -1 } }` |
| `$limit` | 限制返回数量 | `{ $limit: 10 }` |
| `$unwind` | 展开数组字段 | `{ $unwind: "$tags" }` |
| `$lookup` | 关联其他集合 | `{ $lookup: { from: "users", localField: "userId", foreignField: "_id", as: "user" } }` |

## 10.2 示例：统计每个用户的订单总额

```javascript
db.orders.aggregate([
  { $match: { status: "PAID" } },
  { $group: { _id: "$userId", totalAmount: { $sum: "$amount" }, orderCount: { $sum: 1 } } },
  { $sort: { totalAmount: -1 } },
  { $limit: 10 }
])
```

# 11 事务

MongoDB 支持多文档事务，但事务会带来额外开销。设计时优先通过合理的数据模型降低跨文档事务需求，只有在强一致性要求明确时再使用事务。

## 11.1 使用原则

- 单文档操作天然原子，优先用单文档表达业务状态。
- 多文档事务适合余额扣减、订单与库存等必须同时成功或失败的场景。
- 事务内语句尽量少，避免长事务占用资源。
- 事务不是建模不当的补救措施，先优化数据模型，再考虑事务。

## 11.2 示例：Session + Transaction

```javascript
const session = db.getMongo().startSession()
const orderDb = session.getDatabase("mall")

try {
  session.startTransaction()

  orderDb.orders.insertOne(
    { orderNo: "O20260608", userId: 1001, amount: 199, status: "CREATED" }
  )

  orderDb.inventory.updateOne(
    { sku: "SKU-001", stock: { $gte: 1 } },
    { $inc: { stock: -1 } }
  )

  session.commitTransaction()
} catch (e) {
  session.abortTransaction()
  throw e
} finally {
  session.endSession()
}
```

# 12 Explain 与索引验证

索引创建后不要只看“有没有建”，要看查询是否真正命中索引。`explain()` 是判断执行计划的核心工具。

## 12.1 常见关注点

- `COLLSCAN`：全表扫描，通常说明索引缺失或条件未命中索引前缀。
- `IXSCAN`：索引扫描，说明查询已使用索引。
- `totalDocsExamined`：扫描文档数，越小越好。
- `totalKeysExamined`：扫描索引键数，通常应明显小于全量数据。

## 12.2 示例

```javascript
db.users.find({ email: "a@test.com" }).explain("executionStats")
```

# 13 常用更新操作符

| 操作符 | 作用 | 示例 |
| :--- | :--- | :--- |
| `$set` | 设置字段值 | `{ $set: { status: "DONE" } }` |
| `$unset` | 删除字段 | `{ $unset: { tempField: "" } }` |
| `$inc` | 数值递增/递减 | `{ $inc: { stock: -1 } }` |
| `$push` | 数组追加元素 | `{ $push: { tags: "mongodb" } }` |
| `$addToSet` | 数组去重追加 | `{ $addToSet: { roles: "admin" } }` |
| `$pull` | 删除数组中匹配的元素 | `{ $pull: { tags: "deprecated" } }` |
| `$currentDate` | 写入当前时间 | `{ $currentDate: { updateTime: true } }` |
