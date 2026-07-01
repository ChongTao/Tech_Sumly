# 4 Kafka常用命令

下面的命令以 Kafka 4.x 常见的本地脚本方式为例，默认脚本位于 `bin/` 目录下。

## 4.1 初始化与启动

生成集群 ID：

```bash
bin/kafka-storage.sh random-uuid
```

格式化日志目录：

```bash
bin/kafka-storage.sh format --standalone -t <cluster-id> -c config/server.properties
```

启动 Kafka：

```bash
bin/kafka-server-start.sh config/server.properties
```

## 4.2 Topic 管理

创建 Topic：

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 --create \
  --topic orders \
  --partitions 3 \
  --replication-factor 3
```

查看 Topic 列表：

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

查看 Topic 详情：

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic orders
```

增加 Partition：

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 --alter \
  --topic orders \
  --partitions 6
```

> Kafka 目前不支持减少 Partition 数。

删除 Topic：

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic orders
```

## 4.3 生产与消费测试

命令行生产消息：

```bash
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic orders
```

命令行消费消息：

```bash
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic orders --from-beginning
```

## 4.4 Topic 配置查看与修改

查看 Topic 配置：

```bash
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name orders \
  --describe
```

修改 Topic 配置：

```bash
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name orders \
  --alter \
  --add-config retention.ms=604800000
```

删除 Topic 覆盖配置：

```bash
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name orders \
  --alter \
  --delete-config retention.ms
```

## 4.5 Consumer Group 查看

查看所有消费组：

```bash
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
```

查看消费组详情：

```bash
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe \
  --group order-service
```