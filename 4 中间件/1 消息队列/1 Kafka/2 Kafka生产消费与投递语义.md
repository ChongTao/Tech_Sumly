# 2 Kafka生产消费与投递语义

## 2.1 生产者写入流程

生产者发送一条消息时，大致会经历：

1. 序列化 Key 和 Value
2. 根据 Topic 与分区策略选择 Partition
3. 先写入本地发送缓冲区
4. 由后台 I/O 线程批量发送到 Broker
5. Broker 根据 `acks` 返回确认结果

Kafka Producer 默认就是异步发送模型，应用线程调用 `send()` 后通常不会阻塞到消息真正落盘。

## 2.2 分区策略

常见分区策略：

- 指定 Partition：完全由业务决定目标分区
- 指定 Key：相同 Key 通常会路由到同一 Partition，便于保证局部有序
- 不指定 Key：由分区器做均衡分发

实践上：

- 需要同一订单、同一用户有序时，通常使用业务主键作为 Key
- 如果 Key 分布不均，会造成热点 Partition

## 2.3 重要生产者参数

### acks

- `acks=0`：发送后不等待确认，吞吐高，但最不可靠
- `acks=1`：Leader 写入成功即返回
- `acks=all`：等待满足条件的副本确认，最可靠

生产环境更常见的是 `acks=all`。

### linger.ms

为了提高批量效果，Producer 可以等待一小段时间再把多条消息合并成一个批次发送。`linger.ms` 越大，请求数通常越少，但延迟会增加。

### batch.size

控制单批次缓冲大小。更大的批次通常带来更好的吞吐，但会增加内存占用。

### retries 与 delivery.timeout.ms

发送失败后 Producer 可以自动重试。现在更推荐使用 `delivery.timeout.ms` 约束整体投递时限，而不是单独依赖 `retries`。

## 2.4 幂等写入

Kafka Producer 支持幂等写入，用于避免重试带来的重复消息。现代版本中，幂等性已经是非常重要的默认能力。

启用幂等后，通常会同时具备：

- `acks=all`
- 自动重试
- 更强的分区内顺序保障

如果显式关闭幂等，并且允许多个 in-flight 请求与重试并存，就可能在失败重试后出现乱序。

## 2.5 事务消息

当业务是“消费 Kafka 后再写回 Kafka”这类链路时，可以使用事务 Producer，把以下动作绑定在同一事务里：

- 发送输出消息
- 提交消费位点

这样可以降低“重复消费后重复产出”的风险，是 Kafka exactly-once 语义的重要基础。

## 2.6 消费者读取流程

Kafka Consumer 采用拉模型，消费者需要持续调用 `poll()` 获取数据。

核心过程：

1. 订阅 Topic
2. 加入 Consumer Group
3. 被分配若干 Partition
4. 按当前位置拉取消息
5. 处理业务
6. 提交 Offset

## 2.7 Position 与 Committed Offset

Kafka 消费时有两个容易混淆的位置：

- Position：下一条要拉取的消息位置
- Committed Offset：已经持久化保存的消费进度

如果进程崩溃，消费者恢复时通常从 Committed Offset 继续，而不是从内存中的 Position 继续。

## 2.8 自动提交与手动提交

### 自动提交

优点：

- 配置简单
- 适合容忍少量重复或丢失的低风险场景

风险：

- 业务尚未真正处理完成时，Offset 可能已经提交

### 手动提交

优点：

- 可以把“业务处理完成”与“提交位点”放到更合理的位置
- 更适合需要控制重复消费风险的场景

风险：

- 应用代码更复杂
- 提交过晚会增加重复消费范围

## 2.9 auto.offset.reset

当消费者没有已提交位点，或者位点失效时，会用到 `auto.offset.reset`：

- `earliest`：从最早可读位置开始
- `latest`：从最新位置开始
- `none`：直接报错

新版本还支持按时间跨度回退的策略，但业务中最常见的仍然是 `earliest` 与 `latest`。

## 2.10 Consumer Group 与重平衡

当以下情况发生时，Consumer Group 可能触发 Rebalance：

- 新消费者加入
- 消费者离开或宕机
- 订阅 Topic 或 Partition 数发生变化

重平衡期间，分区归属会重新计算。如果消费代码没有正确处理暂停、提交和恢复，就容易出现重复消费或处理抖动。

## 2.11 新一代重平衡协议

Kafka 4.0 起，新一代 Consumer Rebalance Protocol 已可用于生产。它的特点是：

- 增量式重平衡
- 更短的暂停时间
- 由服务端控制更多组管理细节

客户端如果要启用，需要显式配置 `group.protocol=consumer`。如果继续使用默认值，则仍是经典协议。

## 2.12 三种常见投递语义

### At most once

先提交 Offset，再处理消息。

特点：

- 可能丢消息
- 不会重复处理

### At least once

先处理消息，再提交 Offset。

特点：

- 可能重复处理
- 一般不会丢消息

这是最常见、最实用的语义。

### Exactly once

Kafka 的 exactly-once 主要适用于 Kafka 到 Kafka 的处理链路，通常依赖：

- 幂等 Producer
- 事务
- 消费位点与输出结果原子提交

如果链路末端落到外部数据库、缓存、HTTP 接口，是否真正做到端到端 exactly-once，还取决于下游系统是否支持幂等或事务。

## 2.13 实战建议

- 需要局部顺序时，用稳定业务主键作为 Key
- 生产环境优先使用 `acks=all`
- 重要链路不要只依赖自动提交
- 消费逻辑必须保证幂等
- 把“重复消费可接受但可去重”作为默认设计前提
- 只有在确实需要时再引入事务，避免过早增加复杂度
