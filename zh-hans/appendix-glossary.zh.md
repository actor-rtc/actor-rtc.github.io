# 附录 A：名词解释

本附录旨在为 `actr` 生态系统中的核心名词提供一份统一、权威的定义。术语按概念分组和依赖关系组织，**按学习路径排序**，尽量做到先定义后使用。

---

## 1. 框架定位

### Actor
`Actor` 是一种并发计算模型，以独立的 "Actor" 为基本单元，通过异步消息传递实现通信与状态管理，互不共享内存。

经典 Actor 模型有三个核心要点：
- 每个 Actor 拥有其私有状态
- 异步无阻塞的消息传递是 Actor 间唯一交互方式
- Actor 接收消息后，可自主决定回应、创建新 Actor 或转发消息

### Actor-rtc
`Actor-rtc` 是我们的框架名称。`Actor-rtc` 是围绕 Actor 模型和 rtc (real-time communication) 所构建的分布式计算框架。rtc 也是 `WebRTC` 中的 rtc，其是框架的核心通信协议。

### Actr
`Actr` 在不同语境下有多重含义：
- **框架简称**: `Actor-rtc` 的缩写形式
- **宏观 Actor**: 区别于经典 Actor 模型（Akka/Erlang），`Actr` 侧重"宏观"、Service-Oriented 层面的 Actor 模型。框架不提供 Actor 自主创建 Actor 的机制，所有 Actor 实例的创建都是由开发者定义的，也可以视需要由 Kubernetes 等现代容器编排系统综合管理。Actor 的私有状态封装、只能由消息触发变更的机制等则是完全一致的
- **运行时进程**: 指代一个正在运行的 Actr 进程。比如我们说 `ActrNode = ActrSystem + Workload`，此时 `Actr` 即代表一个典型的 Actr 进程

**推荐用法**:
- 框架名称用 `Actor-rtc` 或 `actr`
- 进程实例用 `ActrNode`
- 架构模式用"宏观 Actor"

---

## 2. 核心运行时抽象

### ActrSystem
`ActrSystem` 是 `Actr` 的运行时环境，负责管理 Actr 的生命周期、网络连接（如 WebRTC 和信令）、消息调度，并通过 `Context` 对象向 Actor 提供核心服务等基础设施能力。

**核心职责**:
- 生命周期管理（启动、运行、停止）
- 网络连接管理（WebRTC、WebSocket、信令）
- 消息路由和调度
- 为 Workload 提供 Context 对象

### Workload（Load）
指 Actr 节点中由用户实现的业务逻辑与协议适配层，既可作为服务端提供能力，也可作为客户端消费能力。中性命名，避免将所有实例都称为"服务"。简称 **Load**。

**实现要求**:
- 实现 `.proto` 生成的 `{ServiceName}Handler` trait
- 通过代码生成器自动获得 `MessageHandler` trait
- 通过 `attach()` 方法与 ActrSystem 结合

### ActrNode
一个正在运行的进程节点的**完整描述**。它是框架运行时与业务逻辑的组合体：

```
ActrNode = ActrSystem + Workload
```

**关键设计**:
- 不是"容器包含内容"的关系，而是**两个平行组件的组合**
- ActrSystem 提供基础设施，Workload 提供业务逻辑
- 通过 `attach()` 操作建立连接，完成组装

### ActrType
Actr 进程的类型标识符，由 `<manufacturer>` 和 `<name>` 两个部分组成，用于唯一标识一个 Actr 的类型。

**格式**: `<manufacturer>:<name>`（如 `acme:echo-service`）

**用途**:
- 服务发现：其他 Actor 通过 ActrType 查找服务
- 权限控制：ACL 规则基于 ActrType 授权
- 日志追踪：识别服务来源

**配置位置**: `Actr.toml` 中的 `[package]` 部分：
```toml
[package]
manufacturer = "acme"
type = "echo-service"  # 生成 ActrType: "acme:echo-service"
```

### ActrId
Actr 进程的全局唯一标识符，用于标识运行时的具体实例。

**格式**: `<manufacturer>:<name>:<serial_number>`

**示例**: `acme:echo-service:550e8400-e29b-41d4-a716-446655440000`

**组成部分**:
- `<manufacturer>:<name>`: ActrType（服务类型）
- `<serial_number>`: 运行时生成的 UUID（实例编号）

**用途**:
- 消息路由：框架通过 ActrId 路由消息到具体实例
- 追踪调试：日志中区分同一服务的不同实例
- 点对点通信：直接向特定实例发送消息

**对比表**:

| 术语         | 作用域         | 用途               | 示例                             |
| ------------ | -------------- | ------------------ | -------------------------------- |
| **ActrType** | 服务类型级别   | 服务发现、权限控制 | `acme:echo-service`              |
| **ActrId**   | 运行时实例级别 | 消息路由、实例追踪 | `acme:echo-service:550e8400-...` |

### Realm（领域）
框架的多租户与安全隔离机制。每个 Actor 属于一个 `Realm`（通过 `realm_id` 标识），默认情况下，不同 `Realm` 之间的 Actor 完全隔离。

**典型应用场景**:
- **SaaS 多租户**: 每个租户拥有独立的 Realm（如 `tenant-acme`, `tenant-corp`）
- **环境隔离**: 开发、测试、生产环境使用不同的 Realm（如 `dev`, `staging`, `prod`）
- **权限分组**: 同一组织内的不同部门或项目使用独立 Realm

**配置方式**: 在 `Actr.toml` 中填入配置：
```toml
[package]
realm = "tenant-acme"  # 指定所属 Realm
```

**运行时行为**:
- 服务发现仅返回同 Realm 的服务列表
- 不同 Realm 的 ActrNode 尝试连接时会被拒绝
- 可按 Realm 过滤日志和监控数据

**高级用法**: 通过 `[acl]` 配置可以实现有限的、可控的跨 `Realm` 通信

### Context
这里的 `Context` 特指 Actr 的上下文对象，由 ActrSystem 实现，并由 Actr 在调用方法时传入，包含了当前 Actr 的唯一标识符、调用者标识符、分布式追踪 ID、请求 ID 等上下文信息。

**核心理念**: "请求，而非命令" (Ask, Don't Command)

**三大职责**:

1. **副作用的桥梁**: Actr 不直接执行日志、网络、定时器等操作，而是通过 `context` 发出请求
   ```rust
   ctx.log_info("...");
   ctx.schedule_tell(...);
   ```

2. **请求上下文的载体**: 封装调用环境信息（caller_id, trace_id）

3. **Actr 间通信的媒介**: 提供类型安全的 `tell` 和 `call` 方法

**关键优势**:
- 可测试性：可用模拟的 Context 验证 Actr 行为
- 状态纯粹性：业务逻辑无副作用
- 类型安全：编译时检查消息类型

**参考文档**: [3.2 Context 的设计哲学](./3.2-the-context-philosophy.zh.md)

### ActrRef
在 `Actr` 进程启动后，会返回一个 `ActrRef` 句柄，基于该句柄，应用进程可以与 `Actr` 进程进行通信，以完成应用 Shell 与 Workload 之间的互动需求。

**类比**: 类似于浏览器主进程与 Service Worker 之间的关系

**提供方法**:
- `call()`: 请求-响应式调用
- `tell()`: 单向消息发送

**特点**: 支持进程内零序列化消息传递

### Shell
`Shell`（有时也会使用 `Application Shell`）引申于操作系统中的 `Shell` 概念（提供用户与操作系统交互的接口）。

**定义**: 在 `actr` 网络世界的边缘，那些负责与外部硬件设备、用户等进行交互的进程，我们称之为 `Shell`。

**典型示例**:
- GUI 应用的 `main ui` 线程
- CLI 应用的命令行解析器
- 物联网设备的传感器接口

**与 Workload 的关系**: Shell 通过 ActrRef 与 Workload 通信

### Lifecycle
一个由开发者为 `Workload` 实现的原生 `trait`。它定义了一系列生命周期钩子方法，`ActrSystem` 会在 Actor 生命周期的关键状态转换点调用这些方法。

**核心方法**:
- `on_start`: Actor 启动时调用
- `on_ready`: Actor 准备就绪时调用
- `on_error`: 发生错误时调用
- `on_stop`: Actor 停止时调用

### RemoteActor
一个本地的、类型安全的代理对象，它代表了网络中另一个进程里运行的对等端 Actor。

**用途**: 通过 RemoteActor 可以像调用本地方法一样调用远程 Actor 的服务

---

## 3. 消息通信

### RpcRequest
RPC 请求消息必须实现的 trait，用于类型安全的 RPC 调用。定义了 Request 与 Response 的关联类型以及路由键（route_key），使编译器能够自动推导响应类型。

**核心方法**:
```rust
trait RpcRequest {
    type Response;
    fn route_key() -> &'static str;
}
```

**实现方式**：
- **自动生成**（推荐）：由 actr-cli 代码生成器为所有 RPC 请求类型自动实现
- **手动实现**：Client-only 应用可手动实现以避免生成不必要的 server 代码

**注意**：流式数据（DataStream、MediaFrame）不实现此 trait，它们通过 Fast-Path 直接派发。

### RpcEnvelope（RPC 消息信封）
RPC 调用的消息容器类型，封装了 RPC 消息的完整元数据和 Payload。

**核心字段**:
- `route_key`: 路由键（如 `"/echo.v1.EchoService/Echo"`）
- `payload`: 序列化的 Request/Response（Bytes）
- `trace_id`: 分布式追踪 ID
- `request_id`: 请求 ID（用于响应匹配）
- `metadata`: 元数据（键值对）
- `timeout_ms`: 超时时间

**传输路径**: 通过 State Path（Mailbox）传输，保证可靠、有序、持久化

### {ServiceName}Handler
由 protobuf 服务定义生成的业务逻辑接口 trait。

**命名规则**: `{ServiceName}Handler`（如 `EchoServiceHandler`）

**作用**: 用户 Workload 需要实现此 trait 的各个方法（对应 `.proto` 文件中的 rpc 方法）

**示例**:
```rust
#[async_trait]
pub trait EchoServiceHandler {
    async fn echo(
        &self,
        request: EchoRequest,
        ctx: Arc<dyn Context>
    ) -> ActorResult<EchoResponse>;
}
```

### MessageHandler（消息处理器 Trait）
框架定义的核心 trait，所有 Workload 必须实现。它定义了统一的消息处理入口 `handle_message()`，接收 `RpcEnvelope` 并通过 `MessageDispatcher` 将消息路由到具体的业务方法。

**核心方法**：
```rust
async fn handle_message(
    &self,
    envelope: RpcEnvelope,
    ctx: Arc<dyn Context>
) -> ActorResult<Option<RpcEnvelope>>;
```

**作用**：
- 提供统一的消息处理接口，供 Scheduler 调用
- 内部委托给 `MessageDispatcher::dispatch()` 进行类型安全路由
- 实现编译时的零开销消息分发

**调用链**：
```
Scheduler → MessageHandler::handle_message()
          → MessageDispatcher::dispatch()
          → {ServiceName}Handler::{method}()
```

**实现方式**：
- **自动生成**（推荐）：`actr gen` 为 Workload 自动实现此 trait
- **手动实现**：高级场景可手动实现以自定义消息拦截逻辑

**代码定义**：`actr_framework::MessageHandler` trait

### MessageDispatcher（消息派发器）
由代码生成器为每个服务生成的静态路由 trait，负责根据消息类型将消息派发到对应的 Handler 方法。

**核心方法**：
```rust
async fn dispatch<W: {ServiceName}Handler>(
    workload: &W,
    envelope: RpcEnvelope,
    ctx: Arc<dyn Context>
) -> ActorResult<Option<RpcEnvelope>>;
```

**工作原理**：
1. 解析 `envelope.route_key`（如 `"/echo.v1.EchoService/Echo"`）
2. 使用编译时生成的 `match` 语句进行静态分发
3. 调用 Workload 的对应方法（如 `workload.echo()`）
4. 将返回值序列化为响应 `RpcEnvelope`

**关键设计**:
- 替代旧版的 `MessageRouter`
- 通过 `dispatch()` 方法实现零 dyn 的静态分发
- 编译时生成，零运行时开销

**代码定义**：`{service_name}_actr::MessageDispatcher` trait（代码生成）

### WorkloadRouting（Workload 路由配置）
一个零尺寸类型 (Zero-Sized Type, ZST)，用于在编译时配置 Workload 的路由策略。通过实现 `RoutingConfig` trait，定义 Workload 对应的 `MessageDispatcher` 类型。

**典型实现**：
```rust
pub struct MyWorkloadRouting;

impl RoutingConfig for MyWorkloadRouting {
    type Dispatcher = echo_v1_actr::MessageDispatcher;
}
```

**作用**：
- **编译时类型关联**：将 Workload 与其 MessageDispatcher 绑定
- **零运行时开销**：ZST 不占用内存，仅用于编译时类型推导
- **模块化设计**：允许 Workload 实现多个服务（通过多个 RoutingConfig）

**使用位置**：
```rust
impl MessageHandler for MyWorkload {
    type Routing = MyWorkloadRouting;  // 关联路由配置

    async fn handle_message(...) -> ... {
        <Self::Routing as RoutingConfig>::Dispatcher::dispatch(...)
    }
}
```

**代码定义**：用户定义的 ZST，实现 `actr_framework::RoutingConfig` trait

### Dest
即 `Destination`，表示消息发送的目的地。

**类型定义**: `actr_framework::Dest` enum

**枚举变体**:

- **`Dest::Local`**：本地 Workload（同进程内的业务逻辑）
  - 使用场景：`ctx.call()` 时目标是自己
  - 传输方式：直接函数调用，无序列化
  - 延迟：< 1μs

- **`Dest::Shell`**：应用外壳（Shell 进程）
  - 使用场景：Workload 向 Shell 发送通知或响应
  - 传输方式：Inproc (mpsc 通道)，零序列化
  - 延迟：~10μs

- **`Dest::Remote(ActrId)`**：远程 Actor（网络中其他节点）
  - 使用场景：跨节点 RPC 调用
  - 传输方式：Outproc (WebRTC DataChannel 或 WebSocket)，需 Protobuf 序列化
  - 延迟：1-50ms（取决于网络）

---

## 4. 传输层抽象

### PayloadType
消息负载类型枚举，定义消息的业务类型与传输特性。

**枚举值**:
- `RpcReliable`：可靠有序的 RPC 消息（业务默认）
- `RpcSignal`：高优先级的 RPC 消息（信令、紧急指令）
- `StreamReliable`：可靠有序的流式数据（文件传输等）
- `StreamLatencyFirst`：低延迟优先的流式数据（实时数据，可容忍丢包）
- `MediaRtp`：媒体流（音视频轨道，RTP 传输）

**关键作用**: 每种类型映射到不同的 DataLaneType 组合，由 `PayloadTypeExt` trait 提供静态路由策略

### DataLaneType
DataLane 的具体实现类型枚举，代表实际的物理/逻辑传输方式。

**枚举值**:
- `Mpsc`: 进程内 tokio mpsc 通道（零序列化，~10μs 延迟）
- `WebRtcDataChannel(qos)`: WebRTC 数据通道（支持三种 QoS 配置）
- `WebSocket`: WebSocket 连接（用于信令或降级传输）

**代码定义**：`actr_runtime::transport::route_table::DataLaneType` enum

**注意**：MediaTrack 使用 WebRTC 原生 RTP 通道，不通过 DataLane 传输。

### DataChannelQoS
WebRTC DataChannel 的服务质量 (Quality of Service) 配置枚举，定义数据通道的可靠性和顺序性特征。

**枚举值**:

- **`Signal`**：信令通道
  - ordered=true, max_retransmits=∞
  - 用于高优先级 RPC（如紧急指令）
  - 对应 PayloadType::RpcSignal

- **`Reliable`**：可靠传输
  - ordered=true, max_retransmits=∞
  - 用于常规 RPC 和可靠流式数据
  - 对应 PayloadType::RpcReliable, StreamReliable

- **`LatencyFirst`**：延迟优先
  - ordered=false, max_retransmits=3, maxPacketLifeTime=100ms
  - 用于低延迟流式数据（可容忍丢包）
  - 对应 PayloadType::StreamLatencyFirst

**代码定义**：`actr_runtime::transport::route_table::DataChannelQoS` enum

**映射关系**：PayloadType → DataLaneType(DataChannelQoS) 由 `PayloadTypeExt::data_lane_types()` 定义

### DataLane（数据通道）
统一传输通道抽象，位于具体传输实现（Inproc/Outproc）之上，屏蔽底层差异。

**术语说明**：在口语和设计讨论中，"Lane" 是 "DataLane" 的简称，但在代码和 API 文档中，请使用完整的 "DataLane" 名称。

**类型定义**: `actr_runtime::transport::lane::DataLane` enum

**枚举变体**:
- **Inproc（进程内）**：使用 tokio::sync::mpsc，直接传递 RpcEnvelope 对象（零序列化）。接口：`send_envelope()`/`recv_envelope()`
- **Outproc（跨进程）**：使用 WebRTC DataChannel / WebSocket（共享物理连接），传递 Bytes（需序列化）。接口：`send()`/`recv()`

**关键特性**：
- 统一抽象：上层对 DataLane 编程，而非对具体协议编程
- 零拷贝/零序列化（Inproc）；Bytes 高效传输（Outproc）
- PayloadType 感知：Mpsc 与 WebSocket 显式携带类型信息
- 与 TransportManager/WirePool 配合：出站由 TransportManager 路由到具体 DataLane；Outproc 连接由 WirePool 管理

### PayloadTypeExt
为 `PayloadType` 枚举提供扩展方法的 trait，核心功能是 `data_lane_types()` 方法，返回该 PayloadType 对应的 DataLaneType 静态路由表。

**核心方法**:
```rust
fn data_lane_types(&self) -> &'static [DataLaneType];
```

**关键设计**：
- **编译时路由**：路由表在编译时确定，无运行时查表开销
- **优先级顺序**：返回的数组按优先级排序（如 WebRTC DataChannel 优先于 WebSocket）
- **自动降级**：当首选 DataLane 不可用时，自动选择备选项

**映射示例**:
```rust
// PayloadType → DataLaneType 的路由策略
RpcReliable → [Mpsc, WebRtcDataChannel(Reliable), WebSocket]
RpcSignal → [Mpsc, WebRtcDataChannel(Signal), WebSocket]
StreamReliable → [Mpsc, WebRtcDataChannel(Reliable)]
StreamLatencyFirst → [Mpsc, WebRtcDataChannel(LatencyFirst)]
MediaRtp → [WebRtcMediaTrack]  // 原生 RTP 通道
```

**代码定义**：`actr_runtime::transport::route_table::PayloadTypeExt` trait
**实现位置**：`route_table.rs:36-68`

### Inproc / Outproc
框架中的两种传输模式。

**Inproc（进程内传输）**:
- 用于 Shell 与 Workload 之间的零序列化通信
- 使用 `tokio::sync::mpsc` 通道
- 直接传递 Rust 对象，零序列化
- 延迟：~10μs

**Outproc（跨进程传输）**:
- 用于不同 ActrNode 之间通过 WebRTC 或 WebSocket 进行的网络通信
- 需要 Protobuf 序列化
- 延迟：1-50ms（取决于网络）

### TransportManager
传输层的顶层抽象，管理多个目标（Dest）的消息发送和接收。

**实现类型**:
- `InprocTransportManager`（进程内）
- `OutprocTransportManager`（跨进程）

**职责**:
- 按需创建 `DestTransport` 实例
- 路由消息到正确的 Lane

### InprocTransportManager
进程内传输管理器，负责 Shell（应用外壳）与 Workload 之间的零序列化消息传递。

**实现方式**: 使用 `tokio::sync::mpsc` 通道，直接传递 `RpcEnvelope` 对象

### OutprocTransportManager
跨进程传输管理器，管理多个 `DestTransport` 实例（每个目标 Actor 一个）。

**职责**:
- 按需创建连接
- 路由消息到正确的 Lane
- 协调 WebRTC 和 WebSocket 的切换与降级

### DestTransport
目标传输控制器，管理到单个目标 Actr 的所有连接（WebRTC + WebSocket）。

**功能**:
- 事件驱动的连接状态监控
- 智能重连功能

---

## 5. 连接层抽象

### Wire
底层物理连接的抽象，表示一个具体的传输协议连接（WebRTC PeerConnection 或 WebSocket）。

**职责**:
- 连接的生命周期管理
- 消息收发接口
- 连接状态监控

### WirePool
连接池管理器，管理到特定目标 Actor 的所有 Wire 实例（WebRTC + WebSocket）。

**职责**:
- 连接的创建、复用
- 健康检查
- 智能重连

**实现细节**: 使用事件驱动模型（`tokio::sync::watch`）通知上层连接状态变化

### WireBuilder
Wire 构建器，封装连接建立的异步流程。

**提供方法**:
- `build_webrtc()`: 构建 WebRTC 连接
- `build_websocket()`: 构建 WebSocket 连接

**处理逻辑**: 信令交换、ICE 协商、连接超时等

### Signaling Server（信令服务器）
一个独立于 `actr` 框架的外部组件，其作用是充当"介绍人"。

**职责**:
- 帮助网络中的 Actor 互相发现
- 交换建立直接 WebRTC P2P 连接所需的初始协商信息（如 SDP 和 ICE Candidates）

**推荐实现**: `actrix` - Actor-RTC 官方信令服务器

---

## 6. 双路径模型

### Dual-Path（双路径）
`Actr` 框架的核心逻辑之一，包含**状态路径 (State Path)** 和**快车道 (Fast Path)** 两条并行路径。

**设计动机**:
- 状态路径用于处理需要可靠、有序处理的状态变更消息
- 快车道用于处理高吞吐量、低延迟的流式数据

**关键理念**:
- 区分消息类型和优先级，对不同通道做针对性的优化
- 开发者需要准确理解两条通道的语义和背后的逻辑，正确利用

### State Path（状态路径）
双路径模型中的可靠路径。所有影响 Actor 核心状态的消息都经由该路径，进入 `Mailbox` 并由 `Scheduler` 串行化处理。

**保证**:
- 顺序性：消息按FIFO顺序处理
- 原子性：每条消息的处理是原子的
- 持久化：消息持久化到 SQLite，崩溃后可恢复

**处理的消息类型**:
- `PayloadType::RpcReliable`
- `PayloadType::RpcSignal`

### Fast Path（快车道）
双路径模型中的高性能路径。数据在此路径上会绕过 Mailbox 和调度器，被直接派发给相应的回调函数。

**特点**:
- 极低延迟：数据直接从网络层推送到回调
- 高吞吐量：无中间队列
- 无持久化：数据是易失的，崩溃后即丢失
- 无状态保证：绕过 Mailbox，无法保证顺序性和原子性

**处理的消息类型**:
- `PayloadType::StreamReliable`
- `PayloadType::StreamLatencyFirst`
- `PayloadType::MediaRtp`

**安全警告**:
- 永远不要在 Fast Path 回调中直接修改 `self.state`
- 永远不要使用 `sleep()` 来协调 Fast Path 与 State Path 的同步
- 如需修改状态，使用"并发句柄"模式通过 `ctx.tell()` 发送到 State Path

**参考文档**: [3.8 Fast Path 内部机制](./3.8-fast-path-internals.zh.md), [3.10 Fast Path 生命周期](./3.10-fast-path-lifecycle.zh.md)

### Mailbox（邮箱）
基于 SQLite 实现的持久化消息队列，用于缓存进入状态路径的所有消息。

**核心接口**:
- `enqueue()`: 入队（持久化写入）
- `dequeue()`: 出队（获取待处理消息）
- `ack()`: 确认（标记消息已处理）

**实现细节**:
- 双优先级队列系统（高优 + 普通）
- 基于 SQLite 事务保证消息不丢失
- 每条通道内部严格保证 FIFO 顺序

**参考文档**: [3.6 持久化 Mailbox](./3.6-the-persistent-mailbox.zh.md)

### Scheduler（调度器）
在 actr 框架中，不存在独立的 Scheduler 组件。调度功能由 `ActrNode::start()` 方法中的 Mailbox 轮询循环实现。

**实现方式**: 通过 `tokio::select!` 事件循环串行化处理状态路径消息

**参考文档**: [3.7 状态路径调度](./3.7-state-path-scheduling.zh.md)

---

## 7. 配置与工具链

### IDL (Interface Definition Language)
接口定义语言，用于以语言无关的方式定义服务接口和数据结构。在 Actor-RTC 框架中，特指 **Protocol Buffers (Protobuf)** 语言。

**作用**：
- **跨语言契约**: `.proto` 文件是服务提供方和消费方之间的契约
- **类型安全**: 编译时确保消息结构正确性
- **向后兼容**: Protobuf 的演进规则保证版本兼容性

**示例**：
```protobuf
syntax = "proto3";

package echo.v1;

service EchoService {
  rpc Echo(EchoRequest) returns (EchoResponse);
}

message EchoRequest {
  string message = 1;
}

message EchoResponse {
  string message = 1;
}
```

**与其他术语的关系**：
- **IDL → Contract**: `.proto` 文件定义 Contract（契约）
- **IDL → Codegen**: `actr gen` 基于 IDL 生成代码
- **IDL → Fingerprint**: 通过分析 IDL 语义计算 Fingerprint

### Contract (服务契约)
服务提供方和消费方之间的正式协议，定义了服务的接口、方法签名、数据结构和行为语义。在 Actor-RTC 中，Contract 以 `.proto` 文件形式表达。

**核心理念 - 契约优先开发 (Contract-First)**：

1. **设计阶段**: 先编写 `.proto` 文件，明确服务边界
2. **生成阶段**: 运行 `actr gen` 自动生成代码
3. **实现阶段**: 开发者实现生成的 Handler trait
4. **验证阶段**: 编译器确保实现符合契约

**契约的生命周期**：
```
定义 (.proto)
  ↓
生成 (MessageHandler trait + 消息类型)
  ↓
实现 (Workload 实现 trait)
  ↓
部署 (Fingerprint 锁定版本)
  ↓
运行时验证 (兼容性协商)
```

**Contract vs ServiceSpec**：
- **Contract**: 开发时的 `.proto` 文件（源代码）
- **ServiceSpec**: 运行时的服务元信息（包含 Fingerprint、ACL 等）

**参考文档**: [3.3 应用层 Proto 契约](./3.3-application-proto-contract.zh.md)

### Codegen (代码生成)
基于 `.proto` 契约自动生成 Rust 代码的过程。由 `actr gen` 命令触发，调用 `protoc` + `actr-protoc-plugin` 生成类型安全的 trait 和消息类型。

**生成的核心组件**：

1. **消息类型 (Message Types)**:
   ```rust
   pub struct EchoRequest {
       pub message: String,
   }
   pub struct EchoResponse {
       pub message: String,
   }
   ```

2. **Handler Trait**:
   ```rust
   #[async_trait]
   pub trait EchoServiceHandler {
       async fn echo(
           &self,
           request: EchoRequest,
           ctx: Arc<dyn Context>
       ) -> ActorResult<EchoResponse>;
   }
   ```

3. **MessageDispatcher Trait**:
   ```rust
   pub trait MessageDispatcher {
       async fn dispatch<W: EchoServiceHandler>(
           workload: &W,
           envelope: RpcEnvelope,
           ctx: Arc<dyn Context>
       ) -> ActorResult<Option<RpcEnvelope>>;
   }
   ```

4. **RpcRequest Impl**（为每个 Request 消息类型）:
   ```rust
   impl RpcRequest for EchoRequest {
       type Response = EchoResponse;
       fn route_key() -> &'static str {
           "/echo.v1.EchoService/Echo"
       }
   }
   ```

**关键优势**：
- **零模板代码**: 开发者只需实现业务逻辑
- **编译时类型安全**: 所有消息路由在编译时检查
- **零运行时开销**: 使用静态分发，无 dyn trait object

**工具链**:
- `actr gen`: 主命令
- `protoc`: Protobuf 编译器
- `actr-protoc-plugin`: 框架定制的 protoc 插件

**参考文档**: [3.1 Attach 机制与代码生成](./3.1-attach-and-codegen.zh.md)

### Fingerprint（服务指纹）
基于 `.proto` 文件内容计算的**语义哈希**，用于精确锁定服务版本。

**两种类型**:

1. **Proto 级指纹** (`semantic:abc123...`)
   - 单个 `.proto` 文件的内容哈希
   - 只要文件内容相同，指纹就相同（忽略空格、注释）

2. **服务级指纹** (`service_semantic:xyz789...`)
   - 所有 `exports` 文件组合的哈希
   - 代表完整服务契约的版本

**关键优势**:
- **防止版本漂移**: 即使服务提供者更新了 `.proto`，你的应用仍使用锁定的版本
- **显式升级**: 必须手动运行 `actr install --upgrade` 才能更新依赖
- **协商透明**: 运行时自动进行版本协商，无需手动检查

**计算方式**: 由 `actr-version` 模块通过语义分析（AST 哈希）计算

**参考文档**: [2.4 项目清单与 CLI](./2.4-project-manifest-and-cli.zh.md)

### ServiceSpec
服务规格说明，包含服务的完整元信息。

**包含内容**:
- 服务名
- 版本
- 指纹 (Fingerprint)
- 方法签名
- ACL 规则

**用途**:
- 服务发现
- 兼容性检查
- 运行时验证

### actr.toml
`actr` 项目的清单文件（类似 `package.json` 或 `Cargo.toml`）。

**核心配置**:
```toml
[package]
name = "webrtc-echo-actor"
manufacturer = "acme"
type = "echo-service"
realm = "default"

exports = ["proto/echo.v1.proto"]

[scripts]
run = "cargo run --release"
test = "cargo test"

[system.signaling]
url = "ws://localhost:8081"
```

**配置部分**:
- `[package]`: Actor 身份和类型
- `exports`: 提供的服务契约
- `[dependencies]`: 依赖的外部服务
- `[scripts]`: 构建和运行脚本
- `[system.signaling]`: 信令服务器配置
- `[acl]`: 访问控制列表

### actr.lock.toml
由 `actr install` 后生成的文件，用于锁定项目所依赖的外部服务版本指纹。

**作用**:
- 确保构建的可重现性
- 锁定依赖的 Fingerprint

**示例**:
```toml
[[dependencies]]
actr_type = "acme:storage-service"
fingerprint = "semantic:def456"
proto_files = ["storage.v1.proto"]
```

**版本控制**: 应该被提交到版本控制中

### compat.lock.toml
一个在运行时由系统自动生成的文件，用于记录并缓存"运行时兼容性协商"成功的结果。

**存储位置**: 默认存储到 `/tmp/actr/<project_hash>` 目录

**用途**:
- 记录兼容性协商结果
- 加速后续启动
- 当依赖服务发生变更后，系统会判断新版本是否与当前所使用的存根兼容

### actr
Actor-RTC 框架的命令行工具，提供项目管理、依赖安装、代码生成、构建和运行等功能。

**核心命令**：
- `actr init <name>`: 创建新项目骨架
- `actr install`: 安装外部服务依赖（下载 `.proto`，生成 `actr.lock.toml`）
- `actr gen`: 基于 `.proto` 生成代码
- `actr build`: 编译项目
- `actr run [script]`: 运行脚本（如 `test`, `dev`）

**配置文件**:
- `Actr.toml`: 项目清单
- `actr.lock.toml`: 依赖锁定文件
- `compat.lock.toml`: 兼容性缓存文件

**参考文档**: [2.4 项目清单与 CLI](./2.4-project-manifest-and-cli.zh.md), [4.7 actr 命令行](./4.7-actr.zh.md)

### Runtime Compatibility Negotiation（运行时兼容性协商）
当消费者 Actor 在网络中找不到其 `actr.lock.toml` 锁定的、指纹完全匹配的服务实例时的协商机制。

**协商流程**:
1. SDK 首先尝试"精确指纹匹配"（快速路径）
2. 若失败，则委托信令服务器执行集中式的兼容性筛选（基于 actr-version）
3. 复用全局缓存与请求合并
4. 客户端仅在得到"已过滤的兼容实例列表"后做可用性/负载排序与选择

**参考文档**: [3.12 服务发现与兼容性](./3.12-service-discovery-and-compatibility.zh.md)

---

## 8. 安全机制

### ACL (Access Control List)
访问控制列表，定义在 `actr.toml` 的 `[acl]` 部分。

**作用**: 控制哪些 Actr 可以调用本服务的特定方法

**授权维度**:
- 基于 `ActrType`（如 `acme:client-app`）
- 基于 `<manufacturer>`（如 `acme:*`）
- 基于 `<realm_id>`（如 `tenant-acme`）

**细粒度控制**: 支持到方法级别的授权

**参考文档**: [3.13 动态发现与 ACL](./3.13-dynamic-discovery-and-acls.zh.md)

---

## 核心概念辨析

| 术语 (Term)            | 角色               | 描述                                                                                                                                                                                                                                                                |
| :--------------------- | :----------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Actor / 宏观 Actor** | **概念 / 进程**    | 指代 Actor 模型中的**并发单元**这一抽象概念，在本框架的语境下，特指一个独立的**操作系统进程**。它是拥有独立状态、通过消息与网络中其他 Actor 隔离的"宏观"实体。                                                                                                      |
| **ActrSystem**         | **框架运行时**     | 框架提供的**运行时环境**，是 Actor 赖以生存的"操作系统"。它负责管理底层网络、消息调度、生命周期事件等，为上层业务逻辑提供动力。                                                                                                                                     |
| **Workload（Load）**   | **业务逻辑实现**   | 指**开发者编写的、实现了具体业务逻辑的 `struct`**。它实现由 `.proto` 文件生成的服务 Handler trait（如 `EchoServiceHandler`），通过代码生成器的 blanket 实现自动获得 `Workload` trait 和 `MessageHandler` trait，是整个进程中真正执行业务计算的部分。简称 **Load**。 |
| **ActrNode**           | **进程的完整形态** | 一个正在运行的进程节点的**完整描述**。它是框架运行时与业务逻辑的组合体，即 **`ActrNode = ActrSystem + Workload`**。                                                                                                                                                 |

---

**版本**: v0.9.x
**最后更新**: 2025-11-07
