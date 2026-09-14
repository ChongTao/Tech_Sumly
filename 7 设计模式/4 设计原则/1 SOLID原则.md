# SOLID 原则

SOLID 是面向对象设计中五项常用原则的首字母缩写。它们的目标不是让代码“看起来更抽象”，而是降低模块之间的耦合，使系统更容易理解、测试、修改和扩展。

需要注意，SOLID 是指导设计的启发式原则，不是必须机械执行的语法规则。一个只有几十行、变化很少的模块不需要为了套用原则而拆成很多接口；真正应该优先隔离的是经常变化、难以测试或会影响多个业务方向的职责。

## 1. 五项原则概览

| 缩写 | 英文名称 | 常见中文名称 | 主要解决的问题 |
| --- | --- | --- | --- |
| S | Single Responsibility Principle | 单一职责原则 | 一个模块承担太多互不相关的变化 |
| O | Open/Closed Principle | 开闭原则 | 新需求总要修改稳定的旧代码 |
| L | Liskov Substitution Principle | 里氏替换原则 | 子类型无法真正替代父类型 |
| I | Interface Segregation Principle | 接口隔离原则 | 使用者被迫依赖不需要的方法 |
| D | Dependency Inversion Principle | 依赖倒置原则 | 核心业务被具体技术细节绑死 |

## 2. S：单一职责原则

### 2.1 定义

单一职责原则（Single Responsibility Principle，SRP）要求一个类、模块或函数应该只有一个引起变化的原因。这里的“职责”不是简单地数方法数量，而是判断这些行为是否服务于同一个变化方向。

例如，订单服务同时负责计算价格、保存数据库、发送邮件和生成 PDF 报表，就包含了多个变化来源：计价规则、数据存储、通知渠道和报表格式都可能独立变化。

### 2.2 违反示例

```go
type OrderService struct {
    db Database
}

func (s OrderService) Submit(order Order) error {
    order.Total = calculatePrice(order.Items) // 业务规则
    if err := s.db.Save(order); err != nil { // 持久化
        return err
    }
    return sendEmail(order) // 通知渠道
}
```

数据库字段变化、计价规则变化或邮件服务变化，都会迫使 `OrderService` 修改，也让单元测试需要同时准备数据库和邮件依赖。

### 2.3 改进方式

按变化原因拆分职责，并让应用服务只负责组织用例流程：

```go
type PriceCalculator interface {
    Calculate(items []Item) int64
}

type OrderRepository interface {
    Save(order Order) error
}

type Notifier interface {
    Notify(order Order) error
}

type OrderService struct {
    pricing PriceCalculator
    orders  OrderRepository
    notify  Notifier
}

func (s OrderService) Submit(order Order) error {
    order.Total = s.pricing.Calculate(order.Items)
    if err := s.orders.Save(order); err != nil {
        return err
    }
    return s.notify.Notify(order)
}
```

拆分后的重点不是接口数量，而是计价、持久化和通知可以独立替换、测试和演进。拆分时仍应避免把一个简单函数拆成大量只有一行代码的类或接口。

### 2.4 识别信号

- 一个文件同时包含业务规则、SQL、HTTP 调用和日志格式化。
- 类名中出现 `And`、`Manager`、`Util` 等含义宽泛的词。
- 修改一个需求时，需要触碰多个无关功能，或者频繁产生回归问题。
- 单元测试必须启动数据库、消息队列等大量外部依赖。

## 3. O：开闭原则

### 3.1 定义

开闭原则（Open/Closed Principle，OCP）要求软件实体对扩展开放，对修改关闭。它并不是说旧代码永远不能修改，而是说面对可预见的新变化，应尽量通过新增实现、配置或组合来完成，而不是反复修改稳定的核心逻辑。

### 3.2 违反示例

```go
func CalculateShipping(kind string, weight int64) int64 {
    switch kind {
    case "standard":
        return weight * 2
    case "express":
        return weight * 5
    case "same_day":
        return weight * 10
    default:
        return 0
    }
}
```

每增加一种配送方式，就要修改这个函数。分支还可能散落在下单、退款和运费展示等多个地方。

### 3.3 改进方式

将变化点抽象为策略，稳定代码依赖抽象：

```go
type ShippingStrategy interface {
    Cost(weight int64) int64
}

type ShippingService struct {
    strategies map[string]ShippingStrategy
}

func (s ShippingService) Calculate(kind string, weight int64) (int64, error) {
    strategy, ok := s.strategies[kind]
    if !ok {
        return 0, fmt.Errorf("unsupported shipping type: %s", kind)
    }
    return strategy.Cost(weight), nil
}
```

新增配送方式时注册一个新的 `ShippingStrategy` 实现即可。策略模式、工厂、插件注册表和配置驱动通常都是落实 OCP 的方式。

### 3.4 使用边界

不要为了假设中的变化提前抽象。若变化类型只有两个、逻辑很短且没有扩展迹象，直接分支往往更清晰。应根据真实的变化频率、分支复杂度和扩展成本选择抽象位置。

## 4. L：里氏替换原则

### 4.1 定义

里氏替换原则（Liskov Substitution Principle，LSP）要求：如果 `B` 是 `A` 的子类型，那么使用 `A` 的代码应该能够透明地使用 `B`，而不改变程序的正确性。

替换不仅是“方法签名相同”，还包括契约一致：子类型不能加强调用者必须满足的前置条件，不能削弱方法承诺的后置条件，也不能破坏父类型声明的不变量。

### 4.2 常见反例

经典的“正方形继承矩形”问题中，矩形允许独立设置宽和高，但正方形要求两者始终相等。调用方按矩形契约设置宽度后，正方形却偷偷改变了高度，导致替换失效。

在 Go 中，另一种常见反例是接口实现遇到本来应该支持的操作却直接返回“不支持”：

```go
type ReadWriter interface {
    Read() ([]byte, error)
    Write([]byte) error
}

type ReadOnlyFile struct{}

func (ReadOnlyFile) Read() ([]byte, error) { return nil, nil }
func (ReadOnlyFile) Write([]byte) error {
    return errors.New("write is not supported")
}
```

如果调用方拿到 `ReadWriter` 后合理地认为 `Write` 可用，`ReadOnlyFile` 就不能安全替换它。

### 4.3 改进方式

让抽象准确表达能力，把接口拆成更小的契约：

```go
type Reader interface {
    Read() ([]byte, error)
}

type Writer interface {
    Write([]byte) error
}
```

这样只需要读取的调用方依赖 `Reader`，可读写实现也能分别满足对应契约。LSP 的核心检查问题是：调用方依赖的行为约定，所有实现是否都能满足？

### 4.4 识别信号

- 子类大量重写方法并抛出 `NotImplemented` 或“不支持”异常。
- 调用方必须先判断具体类型才能安全调用父类型方法。
- 子类方法改变了父类的返回含义、异常约定或状态不变量。
- 继承关系只是为了复用代码，而不是表达真正的“是一种”关系。

## 5. I：接口隔离原则

### 5.1 定义

接口隔离原则（Interface Segregation Principle，ISP）要求客户端不应该被迫依赖它不使用的方法。接口应围绕使用者的需求设计，而不是把某个实现拥有的所有能力一次性暴露出来。

### 5.2 违反示例

```go
type UserService interface {
    Get(id string) (User, error)
    Create(User) error
    Update(User) error
    Delete(id string) error
    ExportCSV() ([]byte, error)
}
```

只需要查询用户的页面、只需要导出的后台任务和完整管理后台都依赖同一个大接口。接口新增方法时，所有实现和测试替身都可能被迫修改。

### 5.3 改进方式

按客户端的实际使用方式拆分：

```go
type UserReader interface {
    Get(id string) (User, error)
}

type UserWriter interface {
    Create(User) error
    Update(User) error
    Delete(id string) error
}

type UserExporter interface {
    ExportCSV() ([]byte, error)
}
```

Go 的接口通常由使用方定义，这有助于让依赖保持最小。接口隔离也能减少 mock 的方法数量，让测试更聚焦。

## 6. D：依赖倒置原则

### 6.1 定义

依赖倒置原则（Dependency Inversion Principle，DIP）包含两层含义：

1. 高层模块不应依赖低层模块，二者都应依赖抽象。
2. 抽象不应依赖细节，细节应依赖抽象。

这里的“高层模块”是包含业务规则和用例的代码，“低层模块”是数据库、消息队列、HTTP 客户端、文件系统等技术实现。

### 6.2 违反示例

```go
type OrderService struct{}

func (OrderService) Find(id string) (Order, error) {
    db := sqlx.MustConnect("postgres", os.Getenv("DSN"))
    return loadOrder(db, id)
}
```

业务服务直接创建数据库连接并依赖 PostgreSQL。结果是业务代码难以测试，切换数据库或连接管理方式也会牵动核心逻辑。

### 6.3 改进方式

由高层模块定义自己需要的最小接口，由启动代码把具体实现注入进去：

```go
type OrderFinder interface {
    Find(id string) (Order, error)
}

type OrderService struct {
    orders OrderFinder
}

func NewOrderService(orders OrderFinder) OrderService {
    return OrderService{orders: orders}
}

func (s OrderService) Get(id string) (Order, error) {
    return s.orders.Find(id)
}
```

生产环境可以注入 PostgreSQL 实现，测试中则注入内存实现或 stub。依赖注入可以通过构造函数、参数或框架完成；原则本身不要求使用某个 DI 框架。

### 6.4 与依赖注入的区别

依赖倒置是设计原则，说明依赖关系应该指向哪里；依赖注入是实现手段，说明具体对象如何传入。使用构造函数注入并不自动代表设计合理，注入的接口仍应足够小、稳定且属于正确的边界。