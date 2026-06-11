# 一 OpenSearch 常见的命令

## 1.1 创建索引

- 创建一个空索引（使用默认分片数）

  ```sh
  PUT /users
  ```

- 创建指定分片和副本

  ```json
  PUT /users
  {
    "settings": {
      "number_of_shards": 3,       // 主分片数量
      "number_of_replicas": 1      // 每个主分片的副本数
    }
  }
  ```

- 创建带有分词器（分词器必须在创建索引时指定，后续不可更改）

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
        },
        "content": {
          "type": "text"
        }
      }
    }
  }
  
  ```

- 创建索引并设置别名

  ```json
  PUT /users_v1
  {
    "aliases": {
      "users": {}
    }
  }
  ```

- 检查索引是否创建成功

  ```sh
  GET /_cat/indices?v
  ```

- 查看索引的详情

  ```json
  GET /users
  ```

## 1.2 创建文档

- 创建一条文档（自动生成ID）

  ```json
  POST /users/_doc
  {
    "name": "Alice",
    "age": 30,
    "email": "alice@example.com",
    "created_at": "2026-02-11T10:00:00"
  }
  
  ```

- 创建文档指定ID（如果ID已存在，则覆盖）

  ```json
  PUT /users/_doc/1
  {
    "name": "Bob",
    "age": 25,
    "email": "bob@example.com",
    "created_at": "2026-02-11T11:00:00"
  }
  ```

- 批量创建文档

  ```json
  POST /_bulk
  { "index": { "_index": "users", "_id": "3" } }
  { "name": "David", "age": 35 }
  { "index": { "_index": "users", "_id": "4" } }
  { "name": "Eva", "age": 22 }
  ```

- 响应结果

  ```json
  {
    "_index": "users",
    "_id": "1",
    "result": "created",   // 是否创建成功
    "_version": 1
  }
  ```

## 1.3 查询文档

- 查询所有文档

  ```json
  GET /users/_search
  ```

- 通过ID查询

  ```sh
  GET /users/_doc/1001
  ```

- 全文搜索（会分词）

  ```bash
  GET /users/_search
  {
    "query": {
      "match": {
        "name": "Frank"   // 字段类型是text
      }
    }
  }
  ```

- 多词搜索

  ```bash
  GET /articles/_search
  {
    "query": {
      "match": {
        "content": "OpenSearch powerful"
      }
    }
  }
  ```

- 精确匹配-term

  ```bash
  GET /users/_search
  {
    "query": {
      "term": {
        "email": "frank@example.com"   // 字段类型是keyword
      }
    }
  }
  
  ```

- 组合条件查询

  ```bash
  GET /users/_search
  {
    "query": {
      "bool": {
        "must": [
          { "match": { "name": "Frank" } },
          { "term": { "status": "active" } }
        ]
      }
    }
  }
  ```

- 范围查询

  ```bash
  {
    "range": {
      "age": {
        "gte": 20,
        "lte": 40
      }
    }
  }
  ```

- 分页查询

  ```bash
  GET /users/_search
  {
    "from": 0,
    "size": 10,
    "query": {
      "match_all": {}
    }
  }
  ```

- 排序

  ```bash
  GET /users/_search
  {
    "sort": [
      { "age": "desc" }
    ]
  }
  ```


## 1.4 更新数据

- 全量更新

  ```bash
  PUT /users/_doc/1001
  {
    "name": "Frank",
    "age": 41,
    "email": "frank@example.com",
    "status": "active",
    "created_at": "2026-02-11T14:30:00"
  }
  
  ```

## 1.5 删除数据

- 按照ID删除数据

  ```bash
  DELETE /users/_doc/1001
  ```

  返回结果:

  ```json
  {
    "_index": "users",
    "_id": "1001",
    "result": "deleted"  // 表示删除成功
  }
  ```

- 批量删除

  ```bash
  POST /_bulk
  { "delete": { "_index": "users", "_id": "1002" } }
  { "delete": { "_index": "users", "_id": "1003" } }
  ```

- 删除索引

  ```bash
  DELETE /users
  ```

