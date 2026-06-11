# 常见的运维命令

## 集群相关

- 查看集群是否健康

  ```bash
  GET /_cluster/health
  ```

  状态说明如下：

  | 状态   | 含义               |
  | :----- | :----------------- |
  | green  | 一切正常           |
  | yellow | 副本未分配         |
  | red    | 主分片异常（严重） |

## 节点相关

- 查看节点列表

  ```bash
  GET /_cat/nodes?v
  ```

- 查看节点资源使用情况

  ```bash
  GET /_cat/nodes?v&h=name,ip,heap.percent,ram.percent,cpu,load_1m,node.role
  ```

- 查看节点详细信息

  ```bash
  GET /_nodes/stats
  ```

## 索引相关

- 查看所有索引

  ```bash
  GET /_cat/indices?v
  ```

- 查看索引大小、文档数

  ```bash
  GET /_cat/indices?v&h=index,docs.count,store.size
  ```

-  查看索引 mapping

  ```bash
  GET /users/_mapping
  ```

- 查看索引 settings

  ```bash
  GET /users/_settings
  ```

- 刷新索引

  ```bash
  POST /users/_refresh
  ```

## 分片相关

- 查看分片分布

  ```bash
  GET /_cat/shards?v
  ```

- 查看未分配分片原因

  ```bash
  GET /_cluster/allocation/explain
  ```

## 磁盘相关

- 查看磁盘使用情况

  ```bash
  GET /_cat/allocation?v
  ```

## 监控

- 查看当前任务

  ```bash
  GET /_tasks
  ```

- 查看具体任务

  ```bash
  GET /_tasks/<task_id>
  ```

- 取消任务

  ```bash
  POST /_tasks/<task_id>/_cancel
  ```

## 别名

- 查看别名

  ```bash
  GET /_cat/aliases?v
  ```

  

- 切换索引别名（零停机）

  ```bash
  POST /_aliases
  {
    "actions": [
      { "remove": { "index": "users_v1", "alias": "users" } },
      { "add":    { "index": "users_v2", "alias": "users" } }
    ]
  }
  ```
