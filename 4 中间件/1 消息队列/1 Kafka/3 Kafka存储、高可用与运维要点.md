# 3 Kafka存储、高可用与运维要点

## 3.1 存储模型

Kafka 的本质是分区日志。每个 Partition 都是一个只追加的日志文件集合，消息按 Offset 递增写入。

这带来几个直接结果：

- 顺序写性能高
- 天然支持消息回放
- 消费进度与数据生命周期分离

消费者消费完消息后，Broker 不会立刻删除数据，是否保留由 Topic 的日志策略决定。

## 3.2 日志保留策略

Kafka 常见的 Topic 保留策略有两类。

### delete

默认策略。日志达到时间或空间阈值后，旧日志段会被删除。

适合：

- 普通业务消息
- 日志采集
- 监控指标流

### compact

按 Key 保留“最新值”，旧值会在压缩过程中被清理。

适合：

- 用户画像最新状态
- 配置变更流
- 维表同步
- CDC 最终态主题

### delete,compact

可以同时启用删除和压缩：

- 既做按 Key 保留最新值
- 又受整体保留时间或空间限制

## 3.3 Partition 数如何理解

Partition 数影响三件事：

- Topic 的最大并行写入能力
- Consumer Group 的最大并行消费能力
- 单个 Partition 的数据量与热点程度

需要注意：

- Kafka 目前不支持减少 Partition 数
- 动态增加 Partition 后，基于 `hash(key) % partitionCount` 的分区映射会变化
- 如果消费者使用 `auto.offset.reset=latest`，扩容 Partition 的窗口期可能导致新分区上的早期消息没有被旧消费者读到

因此，Partition 数最好在建 Topic 时结合吞吐、顺序性、热点和扩容预期一起规划。

## 3.4 副本、Leader、ISR

每个 Partition 通常会配置多个副本：

- Leader 副本负责处理读写
- Follower 副本负责复制 Leader 数据
- ISR 是当前与 Leader 保持同步的副本集合

写入可靠性通常与下面两个配置一起理解：

- `acks=all`
- `min.insync.replicas`

如果 ISR 数量低于 `min.insync.replicas`，而生产者又要求 `acks=all`，写入会失败，这是一种“宁可失败也不写成不够安全的数据”的保护机制。

## 3.5 副本因子建议

副本因子越高，容错越强，但存储与复制开销也越大。

实践上通常这样选：

- 开发环境：1
- 一般生产环境：2 或 3
- 对可靠性要求更高的核心链路：3

Kafka 官方也建议常见场景使用 2 或 3 的副本因子。

## 3.6 KRaft 部署模型

Kafka 4.x 已经全面转向 KRaft，不再依赖 ZooKeeper。

在 KRaft 模式下，一个 Kafka 进程可以承担：

- `broker`
- `controller`
- `broker,controller`

其中：

- 小型开发环境可以使用 combined 模式
- 关键生产环境更建议把 Controller 与 Broker 分离部署

Controller 负责元数据仲裁与管理，通常会部署 3 个或 5 个节点，通过多数派维持可用性。

## 3.7 Broker 重要基础配置

### listeners / advertised.listeners

- `listeners` 表示 Broker 绑定的监听地址
- `advertised.listeners` 表示对客户端公布的可访问地址

在容器、云主机、NAT 场景下，两者经常不同。

### log.dirs

Broker 的日志目录。Kafka 把 Partition 数据直接存放在本地磁盘目录下。

### controller.quorum.bootstrap.servers

KRaft 模式下 Broker 和 Controller 都需要知道 Controller Quorum 的连接信息。

## 3.8 Topic 级别常见配置

- `retention.ms`：按时间保留
- `retention.bytes`：按大小保留
- `cleanup.policy`：删除或压缩
- `compression.type`：压缩算法
- `min.insync.replicas`：最小同步副本数
- `max.message.bytes`：单条消息大小上限

不要把所有 Topic 都套用同一组默认值。业务事件、审计日志、CDC 数据流、实时计算中间结果，它们对保留周期和可靠性的要求通常不同。

## 3.9 分层存储

Kafka 支持 Tiered Storage，把较老的日志段转移到远端存储，降低本地磁盘压力。

需要注意：

- Broker 侧需启用远端日志能力
- Topic 侧还要显式开启 `remote.storage.enable`
- 本地保留与整体保留可以分别配置
- Kafka 本身不提供开箱即用的 `RemoteStorageManager` 实现，需要用户自行提供

这意味着分层存储更像“可扩展能力”，不是默认拿来即用的单机开关。

## 3.10 运维关注点

日常运维至少要关注：

- Broker 是否在线
- Topic 是否存在无 Leader 分区
- ISR 是否频繁收缩
- 消费堆积是否持续扩大
- 生产失败率与重试率是否异常
- 磁盘使用率是否接近上限
- 重平衡是否频繁触发
