# iii Engine 概述

[iii](https://github.com/iii-hq/iii)（读作 Triple-I Engine）是一个用于实时组合、扩展和观测服务的运行时。它将 HTTP、队列、定时任务、状态、流、沙箱和 Agent 等能力统一为可发现、可调用、可追踪的系统组件。

它不是大模型推理引擎，也不负责训练或托管模型；它更适合用作 Agent 或业务服务的运行时与集成层。Agent 可以按需加入 Worker、发现其函数、调用函数并查看调用链路。

## 业务如何使用

业务团队通常不需要把现有系统整体迁移到 iii。更常见的做法是保留订单、库存、支付等已有服务，将需要被复用或自动化的能力封装成 Function，再由 iii 为它们接入不同的 Trigger。这样，同一段业务逻辑既能被现有系统通过 HTTP 调用，也能由队列、定时任务或 Agent 驱动。

例如，在电商售后场景中，可以将“校验订单”“查询物流”“创建退款单”“通知用户”分别注册为 Function：

```text
用户/客服 Agent 提交退款请求
            │
            ▼
退款编排 Function：校验订单 → 查询物流 → 创建退款单 → 发送通知
            │                    │             │             │
            └──── 调用已有订单、物流、支付、消息服务的 Worker ────┘
            │
            ▼
返回处理结果，并保留完整 Trace
```

同一编排还可声明多个入口：客服后台用 HTTP 同步发起，订单事件通过队列异步触发，未处理退款由 Cron 定时补偿；Agent 则只调用被授权的 Function，而不是直接持有各个系统的连接方式和调用细节。

### 对业务的价值

- **更快交付自动化流程**：将跨系统步骤组合为 Function 和 Trigger，新增入口或流程时少写胶水代码，避免每个团队重复接入 HTTP、消息队列和定时调度。
- **复用已有能力，降低改造风险**：以 Worker 包装存量服务即可逐步接入，无须把核心订单、支付或数据服务重写为新的技术栈。
- **适配同步、异步和 Agent 场景**：同一业务能力可按需要以接口、事件、定时任务或 Agent 工具的方式提供，减少逻辑分叉和行为不一致。
- **问题定位更直接**：一次业务请求跨越多个 Worker 时，可通过 Trace、日志和运行状态查看在哪一步失败、耗时多少，便于客服排障和运营追踪。
- **更易治理 Agent 调用**：将 Agent 可用能力收敛为目录中的 Function，可在业务侧设置权限、审批、幂等、限流和人工兜底，而不是让 Agent 直接操作底层系统。

iii 更适合“需要编排多个服务、持续接入新工具，并希望看清执行过程”的业务，例如智能客服、运营自动化、订单履约、内容审核和数据处理。若只是单个稳定的同步接口，直接维护普通服务通常更简单；iii 的价值会随跨系统协作和触发方式的增加而提升。

## 核心模型

iii 用三个原语描述系统：

- **Worker**：连接到 iii Engine 的进程，可用 TypeScript、Python、Rust、Go 等语言编写；负责注册函数和触发器。
- **Function**：具有稳定标识的工作单元，例如 `orders::validate`；接收输入并可返回结果。
- **Trigger**：触发函数执行的声明式条件，例如直接调用、HTTP 请求、Cron、队列消息、状态变化或流事件。

```text
HTTP / Queue / Cron / State / Agent
                │
                ▼
            Trigger
                │
                ▼
   Function（由某个 Worker 注册）
                │
                ▼
     调用其他 Function / 返回结果
                │
                ▼
       Trace、日志和运行时状态
```

Worker 注册到引擎后会进入共享目录；其他 Worker 可以立即发现并调用其 Function。这样的设计将跨语言调用、触发路由和观测收敛到同一运行时接口。

## 与 Agent 的关系

对于 Agent，iii 将“工具”表示为目录中可发现的 Function：

1. Agent 发现当前系统已注册的 Worker 与 Function。
2. Agent 调用 Function 完成数据访问、队列投递、执行脚本等任务。
3. 当能力缺失时，可通过 `iii worker add <worker>` 加入新的 Worker。
4. 调用过程可在 Console 中查看触发、日志、Trace 与状态。

因此，iii 适合构建需要动态工具扩展、跨语言协作和可观测执行过程的 Agent 系统。具体权限、审批和隔离策略仍应由应用与部署环境负责，不能仅依赖运行时目录。

## 快速开始

先安装 CLI：

```bash
curl -fsSL https://install.iii.dev/iii/main/install.sh | sh
```

安装脚本会直接在本机执行；生产或受控环境应先审阅脚本内容，并使用团队认可的软件分发方式。

初始化项目并启动引擎：

```bash
iii project init myapp
cd myapp
iii
```

默认示例中，Engine 监听 `ws://localhost:49134`。随后可以增量添加本地或目录中的 Worker：

```bash
iii worker add ./workers/math-worker
iii worker add state
iii worker add http
```

调用已注册函数：

```bash
iii trigger math::add a=2 b=3
```

## 跨语言调用示例

典型场景是 Python Worker 注册计算函数，TypeScript Worker 调用它并暴露 HTTP 接口：

```text
POST /math/add-two-numbers
        │
        ▼
TypeScript Worker: math::add_two_numbers
        │  Function 调用
        ▼
Python Worker: math::add
        │
        ▼
返回 JSON 结果
```

函数处理逻辑不必与 HTTP、队列等入口强绑定；通过为同一 Function 声明不同 Trigger，可以复用业务逻辑并由引擎处理路由与交付。

## 内置与配套能力

项目提供或可添加的能力包括：

- `http`：将 Function 暴露为 HTTP 端点。
- `queue`：队列相关触发与处理。
- `cron`：定时任务。
- `state`：持久化键值状态。
- `pubsub`、`stream`：事件发布订阅和流式事件。
- `sandbox`、`exec`：受控执行类能力。
- `observability`：追踪与运行信息。

使用 `iii console` 可打开控制台，检查 Worker、Function、Trigger、队列、Trace、日志和实时状态。

## SDK 与 Agent Skills

官方 SDK 覆盖 Node.js、Python、Rust 与 Go：

```bash
npm install iii-sdk
pip install iii-sdk
go get github.com/iii-hq/iii/sdk/packages/go/iii
```

仓库还提供面向编码 Agent 的 Skills，覆盖 HTTP、队列、Cron、状态、流和自定义触发器等原语：

```bash
npx skills add iii-hq/iii/skills
```

## 运行与采用注意事项

- Worker 只需通过 WebSocket 接入 Engine，因此可部署在本地、云环境或 Kubernetes 中。
- 官方快速开始中的 Worker 使用 microVM；Linux 上需确保 `/dev/kvm` 可访问。
- `worker add` 会把新能力接入运行中的系统。应审查 Worker 来源、依赖、所需凭据和网络权限，尤其是允许 Agent 动态添加 Worker 时。
- 引擎运行时采用 Elastic License 2.0（ELv2）；SDK、CLI、Console、文档和网站采用 Apache-2.0。商用或再分发前应分别核对许可证要求。

## 参考

- [iii GitHub 仓库](https://github.com/iii-hq/iii)
- [iii 官方文档](https://iii.dev/docs)
- [Quickstart](https://iii.dev/docs/quickstart)
- [Worker 目录](https://workers.iii.dev/)
