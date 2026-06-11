# 一 OpenSearch Mapping 与分词

Mapping 决定字段如何存储、索引和查询；分词决定文本会被拆成什么词。很多查询不符合预期，根因并不是 DSL 写错，而是字段类型或分词策略设计错了。

## 1.1 常见字段类型

- `text`：用于全文检索，会分词，适合标题、正文、描述。
- `keyword`：用于精确匹配、排序、聚合，适合状态、标签、编号、邮箱。
- `integer`、`long`、`float`：用于数值计算和范围查询。
- `date`：用于时间过滤、排序、时间聚合。
- `boolean`：用于布尔条件。
- `object`：对象结构。
- `nested`：数组对象需要独立匹配时使用。
- `knn_vector`：向量检索使用。

## 1.2 `text` 与 `keyword` 的区别

最容易混淆的是这两个类型：

- `text`：写入后会被分词，适合 `match`。
- `keyword`：不分词，保留原值，适合 `term`、聚合、排序。

```json
PUT /users
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "email": {
        "type": "keyword"
      },
      "status": {
        "type": "keyword"
      },
      "age": {
        "type": "integer"
      },
      "created_at": {
        "type": "date"
      }
    }
  }
}
```

如果一个字段既要全文检索，又要排序/聚合，通常使用多字段：

```json
PUT /articles
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "fields": {
          "raw": {
            "type": "keyword"
          }
        }
      }
    }
  }
}
```

这样：

- `title` 用于 `match`
- `title.raw` 用于 `term`、`sort`、`aggs`

## 1.3 动态映射与显式映射

- 动态映射：写入新字段时自动推断类型，开发期方便，但容易产生脏字段和类型误判。
- 显式映射：预先定义字段类型，更稳定，适合生产环境。

查看 mapping：

```bash
GET /users/_mapping
```

生产环境建议：

- 核心索引尽量显式定义 mapping。
- 避免让业务方随意写入新字段。
- 对日志类、半结构化数据，再考虑动态映射或动态模板。

## 1.4 分词器

分词器决定文本如何拆词，常见场景：

- 英文：`standard`、`simple`
- 大小写归一化：常配合 `lowercase`
- 中文：通常需要额外中文分词插件

自定义分词器示例：

```json
PUT /articles
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "my_analyzer"
      }
    }
  }
}
```

测试分词效果：

```json
GET /articles/_analyze
{
  "analyzer": "my_analyzer",
  "text": "OpenSearch Quick Start"
}
```

## 1.5 `object` 与 `nested`

数组对象如果只是普通对象数组，OpenSearch 会把它们拍平成字段集合，可能导致“跨对象误匹配”。

例如订单中有多个商品：

```json
{
  "items": [
    { "sku": "A", "color": "red" },
    { "sku": "B", "color": "blue" }
  ]
}
```

如果 `items` 是普通 `object`，查询 `sku=A and color=blue` 可能错误命中。此时应改用 `nested`。

```json
PUT /orders
{
  "mappings": {
    "properties": {
      "items": {
        "type": "nested",
        "properties": {
          "sku": { "type": "keyword" },
          "color": { "type": "keyword" }
        }
      }
    }
  }
}
```

## 1.6 修改 mapping 的限制

- 大部分字段类型创建后不能直接修改。
- 已存在字段如果类型设计错误，通常需要新建索引再重建数据。
- 分词器通常也需要在建索引时设计好，后改成本较高。

常见做法：

1. 创建新索引，例如 `users_v2`
2. 写入新 mapping
3. 用 `reindex` 迁移数据
4. 切换别名到新索引

## 1.7 实践建议

- 标识类字段优先用 `keyword`，不要误建成 `text`。
- 要聚合和排序的字段，提前确认是否需要 `keyword` 子字段。
- 不确定字段结构时，先设计数据样例，再设计 mapping。
- 中文搜索不要默认依赖 `standard`，应结合实际语言环境选分词方案。
