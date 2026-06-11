# 一 OpenSearch 索引管理

OpenSearch 真正常见的问题，不在 CRUD，而在索引如何演进。索引模板、别名、重建索引、生命周期管理，决定了系统能不能稳定迭代。

## 1.1 别名

别名是一个逻辑名称，可以指向一个或多个真实索引，常用于无停机切换。

创建带别名的索引：

```json
PUT /users_v1
{
  "aliases": {
    "users": {}
  }
}
```

查询别名：

```bash
GET /_cat/aliases?v
```

切换别名：

```json
POST /_aliases
{
  "actions": [
    { "remove": { "index": "users_v1", "alias": "users" } },
    { "add": { "index": "users_v2", "alias": "users" } }
  ]
}
```

## 1.2 重建索引 `reindex`

当 mapping 设计错误、需要拆字段或迁移新索引时，通常使用 `reindex`。

```json
POST /_reindex
{
  "source": {
    "index": "users_v1"
  },
  "dest": {
    "index": "users_v2"
  }
}
```

常见流程：

1. 创建新索引并定义新 mapping。
2. 执行 `reindex` 迁移历史数据。
3. 校验文档量、查询结果、聚合结果。
4. 切换别名。
5. 观察稳定后再删除旧索引。

## 1.3 索引模板

当索引命名具有规律时，可以使用模板统一设置 `settings`、`mappings`、`aliases`。

```json
PUT /_index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "level": { "type": "keyword" },
        "message": { "type": "text" }
      }
    }
  }
}
```

适用场景：

- 日志索引，如 `logs-2026.06.10`
- 订单分月索引
- 多租户按规则创建索引

## 1.4 生命周期管理

数据量持续增长时，需要考虑冷热分层、保留周期和自动删除。OpenSearch 中常见做法是使用 ISM（Index State Management）策略。

一个简化的 ISM 思路：

- 热阶段：允许写入，副本数正常。
- 温阶段：查询为主，减少写入。
- 删除阶段：超过保留期后自动删除。

示例：

```json
PUT _plugins/_ism/policies/logs_policy
{
  "policy": {
    "description": "logs retention policy",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [],
        "transitions": [
          {
            "state_name": "delete",
            "conditions": {
              "min_index_age": "30d"
            }
          }
        ]
      },
      {
        "name": "delete",
        "actions": [
          {
            "delete": {}
          }
        ],
        "transitions": []
      }
    ]
  }
}
```

将策略绑定到索引：

```json
PUT /logs-2026.06/_settings
{
  "index.plugins.index_state_management.policy_id": "logs_policy"
}
```

## 1.5 刷新、刷新间隔与写入性能

- `refresh` 后新文档才能立即被搜索到。
- 默认会周期性刷新。
- 写入量很大时，不要频繁手动刷新，否则会影响吞吐。

查看设置：

```bash
GET /users/_settings
```

手动刷新：

```bash
POST /users/_refresh
```

## 1.6 快照与恢复

生产环境只靠副本不等于备份。索引误删、误更新、集群故障，通常需要快照恢复。

常见流程：

1. 注册快照仓库
2. 创建快照
3. 验证快照可恢复
4. 在需要时恢复指定索引

如果知识库后续继续扩展 OpenSearch，建议单独补一篇“快照与恢复”，把仓库类型、权限要求、恢复粒度、常见故障单独展开。
