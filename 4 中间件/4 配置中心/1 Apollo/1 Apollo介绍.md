# 1 Apollo介绍

[Apollo](https://github.com/apolloconfig/apollo)是携程开源的**分布式配置中心**，主要用于集中管理微服务配置，并支持实时推送、灰度发布、多集群等特性。目前被大量企业使用，生态成熟，其目标是：

- **集中管理配置**，支持多环境、多集群
- **实时生效**（通过长轮询感知变更，通常可在秒级传播，实际受网络和客户端处理影响）
- **版本化管理**（发布历史可审计、可回滚）
- **发布可追溯**（服务端发布有版本和历史记录；客户端通过拉取和本地缓存实现最终一致）
- **高可用架构**（多实例部署，不依赖 ZK）

# 2 核心架构

Apollo 的核心运行组件包括 Portal、Admin Service、Config Service 和 Client；服务端通常还依赖 Eureka（服务注册与发现）以及 MySQL（持久化配置和发布记录）。

- **Portal**：配置管理界面，用于修改、发布配置
- **Admin Service**： Portal 背后服务，负责修改配置、存储数据库
- **Config Service**：客户端访问的服务，支持推送和长轮询
- **Client（如 agollo）**：本地缓存 + 长轮询获取最新版配置
- **Eureka / Meta Server**：Config Service、Admin Service 的服务发现入口；不同版本的部署方式可能有所差异
- **MySQL**：保存应用、命名空间、配置项、发布记录、权限等元数据

Portal 和 Admin Service 面向管理端，Config Service 面向客户端。生产环境应将三类服务分开扩容，并为 Portal、Admin Service、Config Service 配置各自的健康检查和监控。

# 3 配置模型

Apollo 配置模型包含：

- **应用（AppId）**
- **命名空间（Namespace）**
- **集群（Cluster）**
- **环境（DEV/FAT/UAT/PRO）**

一个配置通常由 `AppId + Cluster + Namespace` 定位。环境（Environment）决定访问哪套 Apollo 服务和数据库，Cluster 用于同一环境内的集群隔离，Namespace 用于组织配置文件或配置集合，例如 `application`、`application.yaml` 和自定义 Namespace。相同 Namespace 可以通过关联或继承复用公共配置，但应避免形成难以追踪的覆盖关系。

Apollo 中的配置修改不会直接对所有客户端生效，通常要经过“编辑 -> 发布 -> 客户端感知变更”流程。只有发布后的版本才会被 Config Service 作为当前版本提供。

# 4 配置下发机制

客户端获取配置的流程：

- 客户端启动时：拉取全量配置，通过 Config Service 拉取当前 Namespace 最新配置，写入本地文件缓存（保障服务重启或网络异常可恢复）
- 客户端维持长轮询：监听是否有变更，通过向Config Service 发送 watch 请求，如果服务端有变更，立即返回“namespace changed”通知
- 客户端收到通知后：主动去拉最新配置
- 通知注册的监听器：ChangeListener，Listener 得到 changeEvent（包含 changed key/value）

客户端获取配置通常遵循以下优先级：先读取内存配置，启动时从本地备份恢复，再向 Config Service 拉取最新配置。网络异常时可以继续使用最近一次成功获取的本地配置，但这也意味着配置可能暂时过期。应用应明确区分“必须拿到最新配置才能启动”和“允许使用旧配置降级”两种策略。

## 4.1 配置发布链路

1. 管理员在 Portal 编辑 Namespace 中的键值。
2. Portal 调用 Admin Service，Admin Service 校验权限并写入 MySQL。
3. 发布操作生成新的 release 版本，并记录发布人、时间和变更历史。
4. Config Service 感知发布版本变化，长轮询请求返回 Namespace 已变更。
5. 客户端重新拉取配置，更新内存和本地缓存，并触发监听器。

发布成功不等于业务已经安全使用新配置。对连接池、线程池、路由、开关等配置，应确认客户端监听器具备校验、原子替换和失败回滚逻辑。

# 5 客户端（agollo）功能

- `agollo.Start()`：启动客户端
- `agollo.StartWithConfig(loadAppConfig func() (*config.AppConfig, error))`: 带有配置启动客户端
- `GetStringValue(key, default)`: 获取单个 key
- `GetConfig(namespace)`: 获取 namespace 对象
- `AddChangeListener(listener)`: 监听配置变更

# 6 使用

初始化客户端

```go
client, err := agollo.StartWithConfig(func() (*config.AppConfig, error) {
    return &config.AppConfig{
        AppID:          "123",      // Apollo 应用的唯一 ID
		Cluster:        "dev",      // 要访问的 Apollo 分区（Cluster
		IP:             "http://10.0.0.1:8080",   // 服务端IP
		NamespaceName:  "application.yaml",       // 要拉取的配置文件 Namespace。
		IsBackupConfig: true,                     // 是否启用本地配置备份
        BackupConfigPath: "/opt/",                // 本地备份的路径
		Secret:         "123456789",              // 用于Apollo OpenAPI 安全签名 的密钥
        Label:           "dev",                   // 用于发布配置时的 label。
        SyncServerTimeout: 90,                    // 长轮询超时时间,默认值是 90 秒。
        MustStart: true,                          // 首次启动是否必须成功从 Apollo 获取配置
    }, nil
})

	//Use your apollo key to test
	cache := client.GetConfigCache(c.NamespaceName)
	value, _ := cache.Get("key")
	fmt.Println(value)
```

获取单个key

```
val, ok := cache.Get("key1")
```

事件监听

```go
// 监听配置变更
client.AddChangeListener(&MyListener{})

type MyListener struct{}

// 处理变化的字段，只有字段有变化时触发
func (l *MyListener) OnChange(event *config.ChangeEvent) {
    fmt.Println("变化的 Namespace:", event.Namespace)
    fmt.Println("变化详情:", event.Changes)
}

// 只要发布就会触发，无论是否有变化
func (l *MyListener) OnNewestChange(event *config.ChangeEvent) {}
```

> `event.Changes` 是 map[string]*Change：
> ```go
> type Change struct {
>     Key        string
>     OldValue   string
>     NewValue   string
>     ChangeType int  // ADD, MODIFY, DELETE
> }
> ```
>
> 



# 7 原理介绍

## 7.1 长轮询机制原理

agollo 客户端启动后，会为每个 Namespace 启动一个长轮询 goroutine。客户端向 Config Service 发起带超时时间的请求：如果 Namespace 没有变化，服务端在超时后返回，客户端重新发起请求；如果有发布，服务端提前返回变更通知，客户端再发起一次普通拉取获取完整配置。

长轮询只负责降低变更通知延迟，不传输完整配置，也不保证业务代码已经完成热更新。客户端应设置合理的超时、重试退避和日志，避免 Config Service 故障时产生重试风暴。

## 7.2 灰度发布与回滚

- 灰度发布可以按集群、IP 等维度让部分实例先使用新配置，验证无误后再扩大范围。
- 灰度规则、目标实例和正式发布版本应纳入审计，避免灰度配置长期残留。
- 发布历史支持查看和回滚；回滚前应确认旧配置仍与当前代码版本兼容。
- 数据库连接、密钥、线程池等高风险配置应优先小范围验证，并准备应用层兜底值。

## 7.3 高可用与故障降级

- Config Service 和 Admin Service 应部署多个实例；客户端不要只配置一个服务地址。
- Config Service 应保持无状态，实例间共享 MySQL 中的配置数据，并通过服务发现或负载均衡访问。
- 本地缓存可以应对短暂网络故障，但不能替代配置服务、数据库备份和恢复演练。
- 需要关注 Config Service 请求成功率、长轮询连接数、发布延迟、客户端拉取失败数和配置缓存更新时间。

## 7.4 权限与安全

- AppId 用于标识应用，不是身份认证凭据；管理端应通过账号、角色和 Namespace 权限控制发布范围。
- Portal、Admin Service、Config Service 和 MySQL 应限制网络访问范围，生产环境启用 HTTPS 和数据库访问控制。
- 密钥、令牌和数据库密码不应以明文写入普通配置或提交到代码仓库；必要时使用专门的密钥管理系统，并限制读取权限。
- 记录配置查看、修改、发布和回滚审计；敏感值在日志、界面和变更事件中应脱敏。



# 8 使用建议

- 配置键命名、类型和默认值应形成约定；字符串配置在客户端解析成结构体前先做格式校验。
- 不要在监听器中执行耗时网络调用或阻塞主线程；复杂变更应异步处理，并保证重复通知不会造成副作用。
- 配置变更应兼容旧版本和新版本实例，滚动发布期间避免只被单一版本理解的配置格式。
- 大段文本、二进制数据和高频动态状态不适合放入 Apollo；配置中心不是缓存、消息队列或业务数据库。
- 配置发布前后都应保留版本号，必要时通过自动化校验、审批和回滚流程降低误发布风险。

# 9 注意事项

## 9.1 Apollo 客户端获取的到底是什么

Apollo 客户端获取的不是“字节”，而是**字符串（string）**。

- Apollo 在服务端存储配置时就是按照“文本”存储：properties、yaml、json 本质都是文本。
- 客户端拉取配置时，Apollo SDK 通常返回字符串格式的原始内容。
- 使用时应在应用层完成类型转换和结构校验，例如将字符串解析为 YAML/JSON 或转换为整数、布尔值。
- 配置监听回调可能在独立线程中执行，更新共享配置对象时应使用不可变对象、锁或原子替换，避免读到半更新状态。

# 10 Namespace 类型与配置覆盖

Apollo 常见 Namespace 类型如下：

| 类型 | 特点 | 适用场景 |
| :--- | :--- | :--- |
| `properties` | 默认类型，按 key-value 管理，适合单项配置 | 开关、地址、超时时间、业务参数 |
| `yaml` / `yml` | 以完整文本保存层级结构 | Spring Boot、复杂结构配置 |
| `json` | 以完整 JSON 文本保存 | 前端配置、结构化参数 |
| `xml` | 以完整 XML 文本保存 | 需要 XML 格式的组件配置 |
| `txt` | 原始文本，不进行键值解析 | 模板、脚本片段、证书文本等 |

`application` 是常用的默认 Namespace。自定义 Namespace 适合隔离不同组件或业务域的配置，但不宜拆分得过细，否则会增加发布、监听和排查成本。

配置读取时应明确以下规则：

- 同一个 AppId、Cluster 和 Namespace 下，客户端只使用已发布的版本。
- 公共 Namespace、关联 Namespace 和应用自身配置可能存在覆盖关系，最终值应以实际客户端 SDK 的解析规则为准。
- 不要依赖配置项在多个 Namespace 中的隐式覆盖；对关键配置应在文档中记录来源和优先级。
- 配置删除也需要发布，客户端收到删除事件后才会移除对应值；应用应准备默认值或降级行为。

# 11 部署与容量规划

Apollo 的生产部署至少需要规划 Portal、Admin Service、Config Service、Eureka 和 MySQL。不同环境通常使用独立的 Apollo 服务和数据库，避免测试环境误读或修改生产配置。

- Config Service 是客户端流量入口，应按客户端连接数、长轮询连接数和配置拉取峰值扩容。
- Admin Service 和 Portal 主要承载管理与发布流量，实例数通常低于 Config Service，但应具备故障切换能力。
- MySQL 应配置备份、主从或高可用方案；Apollo 配置历史和发布记录不能只依赖应用本地缓存恢复。
- 不同环境的 Meta Server 地址、数据库连接和服务发现配置应通过部署配置注入，避免写死在业务代码中。
- 发布前评估配置大小和更新频率；超大配置或频繁变化的数据会增加网络、数据库和客户端解析压力。

# 12 客户端容错策略

客户端常见异常及建议处理方式：

| 异常 | 建议策略 |
| :--- | :--- |
| 首次无法连接 Config Service | 根据配置决定阻止启动或使用明确的本地默认值；核心安全配置不应静默使用空值 |
| 运行期间 Config Service 不可用 | 使用最后一次成功配置，记录告警并进行带退避的重试 |
| 配置格式解析失败 | 保留旧配置，拒绝半成品更新，并记录 Namespace、版本和错误位置 |
| 监听器执行失败 | 不影响长轮询主循环；将失败交给独立错误处理和告警机制 |
| 配置版本与代码不兼容 | 采用向后兼容字段、双读双写或分阶段发布，避免一次发布同时改变协议 |

本地缓存文件属于应用运行数据，应限制文件权限，避免把数据库密码、令牌等敏感配置以明文落盘。对敏感值可以只在专用密钥系统中保存，Apollo 只保存引用或非敏感配置。

# 13 发布与变更规范

- 配置项应包含用途、取值范围、默认值和生效方式说明；特别标明“重启生效”还是“动态生效”。
- 将高风险变更拆成小批次，先在测试环境和少量实例进行验证，再逐步扩大范围。
- 变更说明中记录关联代码版本、影响范围、回滚版本和验证指标。
- 对数据库地址、限流阈值、线程池大小等配置增加格式、范围和依赖检查，避免合法文本造成非法运行参数。
- 变更后观察错误率、延迟、连接池、线程池和业务指标，而不只观察 Apollo 发布是否成功。

# 14 常见问题排查

1. **Portal 发布成功但客户端没有变化**：确认修改已经发布，检查客户端使用的 AppId、Environment、Cluster、Namespace 和 Meta Server 地址是否正确。
2. **只有部分实例生效**：检查灰度规则、实例 IP、集群名称和客户端本地缓存时间，确认是否存在不同版本客户端。
3. **配置服务请求超时**：检查 Config Service 健康状态、长轮询连接数、负载均衡超时、网络策略和客户端重试是否形成风暴。
4. **客户端持续使用旧值**：确认本地备份路径和文件权限，查看拉取失败日志及配置版本；不要直接删除缓存文件作为首选修复方式。
5. **配置解析失败**：先在独立校验程序中验证 YAML、JSON 或 properties 格式，再发布；监听器更新失败时应保留旧版本并触发告警。

排查时应同时记录 `AppId`、Environment、Cluster、Namespace、release 版本、客户端版本和实例标识，避免只凭 Portal 页面状态判断问题。
