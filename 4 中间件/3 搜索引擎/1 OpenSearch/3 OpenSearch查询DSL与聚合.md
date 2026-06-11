# 一 OpenSearch 查询 DSL 与聚合

OpenSearch 的查询通常由 `query`、`filter`、`sort`、`_source`、`aggs` 这几部分组成。实际使用时，建议先区分“相关度查询”和“过滤条件”，再决定是否叠加聚合统计。

## 1.1 查询与过滤的区别

- `query`：参与相关度评分，适合全文检索，例如 `match`、`multi_match`。
- `filter`：只做条件过滤，不参与评分，适合精确匹配、范围筛选、状态过滤，缓存命中率更高。

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "title": "wireless mouse"
          }
        }
      ],
      "filter": [
        {
          "term": {
            "status": "online"
          }
        },
        {
          "range": {
            "price": {
              "lte": 200
            }
          }
        }
      ]
    }
  }
}
```

## 1.2 常见查询类型

### `match`

适合 `text` 字段，会先分词再检索。

```json
GET /articles/_search
{
  "query": {
    "match": {
      "content": "OpenSearch 查询优化"
    }
  }
}
```

### `term`

适合 `keyword`、布尔值、枚举值等精确匹配字段。

```json
GET /users/_search
{
  "query": {
    "term": {
      "status": "active"
    }
  }
}
```

### `multi_match`

在多个字段上做全文检索。

```json
GET /articles/_search
{
  "query": {
    "multi_match": {
      "query": "OpenSearch",
      "fields": ["title^2", "content"]
    }
  }
}
```

### `bool`

组合多个条件时最常用：

- `must`：必须满足，参与评分。
- `filter`：必须满足，不参与评分。
- `should`：可选条件，命中后提高分数。
- `must_not`：必须不满足。

```json
GET /users/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "alice" } }
      ],
      "filter": [
        { "term": { "status": "active" } }
      ],
      "must_not": [
        { "term": { "deleted": true } }
      ]
    }
  }
}
```

## 1.3 排序、分页与返回字段

### 排序

```json
GET /orders/_search
{
  "sort": [
    { "created_at": "desc" },
    { "_score": "desc" }
  ]
}
```

### 分页

浅分页可以用 `from + size`：

```json
GET /orders/_search
{
  "from": 0,
  "size": 20,
  "query": {
    "match_all": {}
  }
}
```

深分页更适合使用 `search_after`，避免高偏移量带来的性能问题：

```json
GET /orders/_search
{
  "size": 20,
  "sort": [
    { "created_at": "desc" },
    { "_id": "asc" }
  ],
  "search_after": ["2026-06-01T10:00:00", "10001"]
}
```

### 限制返回字段

```json
GET /users/_search
{
  "_source": ["name", "email", "status"],
  "query": {
    "match_all": {}
  }
}
```

## 1.4 高亮

高亮适合搜索结果页直接展示命中片段。

```json
GET /articles/_search
{
  "query": {
    "match": {
      "content": "OpenSearch"
    }
  },
  "highlight": {
    "fields": {
      "content": {}
    }
  }
}
```

## 1.5 聚合

聚合用于统计分析，常见用途包括计数、分组、求平均值、区间统计等。

### `terms` 聚合

按字段分组统计。

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "status_count": {
      "terms": {
        "field": "status"
      }
    }
  }
}
```

### `avg` 聚合

计算平均值。

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "avg_amount": {
      "avg": {
        "field": "amount"
      }
    }
  }
}
```

### `date_histogram` 聚合

按时间桶统计趋势。

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "daily_orders": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "day"
      }
    }
  }
}
```

### 聚合 + 过滤组合

```json
GET /orders/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "paid" } },
        {
          "range": {
            "created_at": {
              "gte": "now-30d/d"
            }
          }
        }
      ]
    }
  },
  "aggs": {
    "by_channel": {
      "terms": {
        "field": "channel"
      }
    }
  }
}
```

## 1.6 实践建议

- 全文搜索优先用 `match`，精确过滤优先用 `term` 和 `filter`。
- 聚合字段通常应为 `keyword`、数值、日期类型，不建议直接对 `text` 字段做聚合。
- 深分页不要长期依赖 `from`，优先考虑 `search_after` 或滚动导出。
- 如果查询慢，先看是否把大量精确条件写进了 `must`，这类条件通常更适合放进 `filter`。
