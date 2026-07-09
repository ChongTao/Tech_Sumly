# 5 Kafka安装与启动

本文以 Kafka 4.x 为主，默认使用当前主流的 KRaft 模式，不再展开 ZooKeeper 时代的安装方式。

## 5.1 安装前准备

安装 Kafka 前，建议先确认以下环境：

- 操作系统：Linux、macOS、Windows 均可，本地学习更常见的是 Linux 或 macOS
- Java：Kafka 4.x 的 Broker、Connect 和命令行工具要求 Java 17 及以上
- 端口：默认常见端口为 `9092`
- 磁盘：日志目录需要有足够可用空间

如果只是本地学习，单机单节点即可。如果是生产环境，则需要提前规划：

- Broker 数量
- Controller 数量
- 数据目录
- 副本数
- 网络与监听地址

## 5.2 下载安装包

从 Kafka 官方下载二进制安装包，解压后进入目录：

```bash
tar -xzf kafka_2.13-4.3.1.tgz
cd kafka_2.13-4.3.1
```

目录中常见内容包括：

- `bin/`：命令行脚本
- `config/`：配置文件
- `libs/`：依赖库
- `logs/`：运行日志

## 5.3 单机安装流程

### 5.3.1 生成集群 ID

Kafka 在 KRaft 模式下启动前，需要先生成一个 Cluster ID：

```bash
bin/kafka-storage.sh random-uuid
```

也可以保存到环境变量中：

```bash
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
echo $KAFKA_CLUSTER_ID
```

### 5.3.2 检查配置文件

本地开发环境可以直接使用 `config/server.properties`。它通常已经配置为单机可运行的 combined 模式，即一个进程同时承担 Broker 和 Controller 角色。

重点关注以下配置：

```properties
process.roles=broker,controller
node.id=1
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
controller.listener.names=CONTROLLER
log.dirs=/tmp/kraft-combined-logs
```

如果你是在云主机、容器或多网卡环境中部署，还需要重点检查：

- `listeners`
- `advertised.listeners`
- `controller.quorum.bootstrap.servers`

## 5.3.3 格式化日志目录

生成 Cluster ID 后，需要对日志目录执行格式化：

```bash
bin/kafka-storage.sh format --standalone -t $KAFKA_CLUSTER_ID -c config/server.properties
```

这一步会初始化 KRaft 元数据和日志目录。若日志目录已经存在旧数据，不要随意重复格式化，避免覆盖已有集群状态。

### 5.3.4 启动 Kafka

启动单机 Kafka：

```bash
bin/kafka-server-start.sh config/server.properties
```

如果想后台运行，可以使用：

```bash
bin/kafka-server-start.sh -daemon config/server.properties
```

### 5.3.5 验证服务是否正常

启动成功后，先创建一个测试 Topic：

```bash
bin/kafka-topics.sh --create --topic quickstart-events --bootstrap-server localhost:9092
```

查看 Topic：

```bash
bin/kafka-topics.sh --describe --topic quickstart-events --bootstrap-server localhost:9092
```

发送测试消息：

```bash
bin/kafka-console-producer.sh --topic quickstart-events --bootstrap-server localhost:9092
```

消费测试消息：

```bash
bin/kafka-console-consumer.sh --topic quickstart-events --from-beginning --bootstrap-server localhost:9092
```

如果生产和消费都正常，说明本地安装已经完成。

## 5.4 Docker 安装方式

如果只是想快速体验 Kafka，也可以直接使用官方 Docker 镜像。

拉取镜像：

```bash
docker pull apache/kafka:4.3.1
```

启动容器：

```bash
docker run -p 9092:9092 apache/kafka:4.3.1
```

如果你偏好原生镜像，也可以使用：

```bash
docker pull apache/kafka-native:4.3.1
docker run -p 9092:9092 apache/kafka-native:4.3.1
```

Docker 方式适合本地试验，但生产环境通常还是需要单独规划磁盘目录、端口映射、监听地址和持久化卷。

## 5.5 Windows 安装要点

Kafka 在 Windows 上也能运行，但更适合学习和测试。

常见步骤：

1. 安装 Java 17+
2. 下载并解压 Kafka 二进制包
3. 打开 PowerShell 或 CMD，进入 Kafka 根目录
4. 执行 `kafka-storage` 初始化
5. 执行 `kafka-server-start` 启动

需要注意：

- Windows 下路径分隔符和日志目录路径要检查清楚
- 如果端口被占用，需要修改 `listeners`
- 某些脚本在 PowerShell 与 Git Bash 中表现略有差异

## 5.6 生产环境安装建议

生产环境不要直接照抄单机示例，至少要额外考虑：

- Controller 与 Broker 分离部署
- 3 个或 5 个 Controller 组成多数派
- 数据目录放在独立磁盘
- 正确配置 `advertised.listeners`
- 为 Topic 规划好副本数与 Partition 数
- 配置监控、日志采集和告警
- 避免误操作格式化已有日志目录

## 5.7 常见问题

### 5.7.1 启动报 Java 版本不匹配

通常是因为 Kafka 4.x 需要 Java 17 及以上，而本机 Java 版本过低。

### 5.7.2 启动成功但客户端连不上

优先检查：

- `listeners`
- `advertised.listeners`
- 防火墙或安全组
- 端口是否被占用

### 5.7.3 重启后数据丢失

常见原因：

- 使用了临时日志目录
- 容器没有挂载持久化卷
- 重复执行了格式化命令
