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



```go
func ExampleUsage() error {
	ctx := context.Background()

	client, err := NewMongoClient("mongodb://localhost:27017")
	if err != nil {
		return err
	}

	col := client.
		Database("nlp_translation_engine").
		Collection("nlp_translation_engine")

	// 写入
	err = UpsertTranslation(ctx, col, "5d41402abc4b2a76", "zh:en:user",
		TranslationValue{
			SourceLang:   "zh",
			TargetLang:   "en",
			FromSource:   "user",
			Translation:  "Hello",
			TranslatedAt: time.Now(),
		},
	)
	if err != nil {
		return err
	}

	// 查询
	val, err := FindTranslation(ctx, col, "5d41402abc4b2a76", "zh:en:user")
	if err != nil {
		return err
	}

	if val != nil {
		println("translation:", val.Translation)
	}

	return nil
}

```





MongoDB 的 **TTL（Time To Live）索引**用于**自动删除过期文档**，非常适合做缓存、临时数据、日志、会话等场景。下面从**原理、用法、注意事项、实战示例**几个层面系统说明。

------

## 一、TTL 索引是什么

TTL 索引是一种 **特殊的单字段索引**，MongoDB 会后台启动一个 **TTL Monitor** 线程，**周期性扫描**集合中带 TTL 索引的字段：

- 当文档中的时间字段 **早于当前时间 - expireAfterSeconds**
- MongoDB 会**自动删除该文档**

👉 删除是**异步的、最终一致**，不是精确到秒。

------

## 二、TTL 的核心特性

| 特性             | 说明               |
| ---------------- | ------------------ |
| 自动删除         | 无需应用代码干预   |
| 后台执行         | 对业务影响小       |
| 最终一致         | 可能延迟 1~60 秒   |
| 只支持单字段     | 不能是复合索引     |
| 只能用于日期字段 | `Date` / `ISODate` |
| 不支持数组       | 数组字段无法 TTL   |

------

## 三、TTL 索引的基本用法

### 1️⃣ 基础示例（固定存活时间）

```
db.logs.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 } // 1 小时后自动删除
)
```

文档示例：

```
{
  "_id": ObjectId("..."),
  "message": "test log",
  "createdAt": ISODate("2026-01-14T10:00:00Z")
}
```

🕒 在 `createdAt + 3600s` 之后会被删除。

------

### 2️⃣ 绝对过期时间（推荐）

```
db.sessions.createIndex(
  { expireAt: 1 },
  { expireAfterSeconds: 0 }
)
```

文档示例：

```
{
  "_id": "token_123",
  "userId": 1001,
  "expireAt": ISODate("2026-01-14T12:00:00Z")
}
```

📌 到 `expireAt` 指定时间 **立即可被删除**

👉 **这是生产环境最常用的方式**

------

## 四、TTL Monitor 的工作机制（重要）

- MongoDB **每 60 秒左右**扫描一次 TTL 索引
- 每次扫描会：
  - 找出过期文档
  - **批量删除**
- 删除过程：
  - 普通 `delete`
  - 会写 oplog（影响复制集）

⚠️ 所以：

- **不是精确定时**
- 可能有 **分钟级延迟**

------

## 五、常见限制 & 坑点

### ❌ 1. 不能是复合索引

```
// ❌ 错误
db.test.createIndex(
  { userId: 1, createdAt: 1 },
  { expireAfterSeconds: 3600 }
)
```

TTL 索引 **只能包含一个字段**

------

### ❌ 2. 字段必须是 Date 类型

```
{
  "expireAt": "2026-01-14 12:00:00" // ❌ string
}
```

必须是：

```
{
  "expireAt": ISODate("2026-01-14T12:00:00Z")
}
```

------

### ❌ 3. 数组字段不生效

```
{
  "expireAt": [ ISODate("...") ] // ❌
}
```

------

### ❌ 4. 不适合强一致删除

TTL 不能用于：

- 金融数据
- 强事务语义
- 精确到秒的控制

------

## 六、TTL + 业务设计最佳实践

### ✅ 推荐模式：expireAt + expireAfterSeconds = 0

优点：

- 每条数据过期时间可控
- 易于调试
- 业务语义清晰

```
db.cache.createIndex(
  { expireAt: 1 },
  { expireAfterSeconds: 0 }
)
```

------

### ✅ 常见使用场景

| 场景            | 说明                         |
| --------------- | ---------------------------- |
| Session / Token | 登录态自动失效               |
| 缓存            | 自动淘汰                     |
| 临时任务        | 中间态清理                   |
| 日志            | 控制日志保留周期             |
| 翻译/中间结果   | 你之前提到的翻译缓存就很适合 |

------

## 七、Go（mongo-go-driver）示例

```
indexModel := mongo.IndexModel{
	Keys: bson.D{{Key: "expireAt", Value: 1}},
	Options: options.Index().
		SetExpireAfterSeconds(0),
}

_, err := collection.Indexes().CreateOne(ctx, indexModel)
if err != nil {
	log.Fatal(err)
}
```

插入数据：

```
doc := bson.M{
	"key":      "abc",
	"value":    "data",
	"expireAt": time.Now().Add(10 * time.Minute),
}
collection.InsertOne(ctx, doc)
```

------

## 八、TTL vs 手动删除对比

| 方式     | 优点                   | 缺点       |
| -------- | ---------------------- | ---------- |
| TTL      | 简单、稳定、无业务侵入 | 非实时     |
| 定时任务 | 精确、可控             | 维护成本高 |
| 逻辑删除 | 强一致                 | 数据膨胀   |
