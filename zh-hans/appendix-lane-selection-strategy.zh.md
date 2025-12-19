# 车道选择策略：基于 Protobuf Option 的声明式路由

> 本文档描述如何通过 protobuf 扩展 option 实现声明式的车道选择

---

## 1. 核心设计理念

### 1.1 双向口岸模型

```
                    Actor System

  发送方向 ──┐                      ┌── 接收方向
            │                      │
            ▼                      ▼
     ┌──────────────┐      ┌──────────────┐
     │ OutboundGate │      │ Inbound Gates│
     │  (出境口岸)   │      │  (入境口岸)   │
     └──────────────┘      └──────────────┘
            │                      │
            │                      ├─ Mailbox
            │                      ├─ FLG1
            │                      └─ FLG2
            ▼                      ▲
        4 条 Lane                 4 条 Lane
   (每类一条 + MediaTrack 多条)
            │                      │
            └──────── WebRTC ──────┘
```

### 1.2 两层路由

```
┌─────────────────────────────────────────────────────────┐
│  第一层：RPC → Lane 路由（声明式，protobuf option）      │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  • 在 .proto 文件中声明                                  │
│  • 通过 option (actr.lane_type) 标注                     │
│  • 代码生成器读取并生成路由逻辑                          │
│  • 默认策略：未标注 = RELIABLE                           │
│  • OutboundGate 根据 lane_type 选择正确的 Lane          │
└─────────────────────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  第二层：Lane → Gate 路由（静态配置）                    │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  • Lane 实例创建时配置 target_gate                       │
│  • InputHandler 直接读取 Lane.config().target_gate      │
│  • 同类型 Lane 连接到对应的入境 Gate                     │
└─────────────────────────────────────────────────────────┘
```

### 1.2 声明式优于命令式

```protobuf
service ChatService {
  // ✅ 声明式：通过 option 声明语义
  rpc SendMessage(MessageRequest) returns (MessageResponse) {
    option (actr.lane_type) = SIGNAL;  // 紧急消息，走 Signal 通道
  }

  // ❌ 命令式：在代码中手动选择通道（我们不采用这种方式）
  // 用户代码：ctx.send_via_signal_lane(msg)
}
```

**优势**：
- **关注点分离**：业务逻辑与传输层解耦
- **可维护性**：修改路由策略只需改 proto 文件
- **类型安全**：编译时确定，避免运行时错误
- **文档即代码**：proto 文件即路由策略文档

---

## 2. Protobuf 扩展定义

### 2.1 Option 扩展

```protobuf
// actr-protocol/proto/actr/options.proto

syntax = "proto2";
package actr;

import "google/protobuf/descriptor.proto";

// Payload types for traffic segregation (message type + transport characteristics)
// Format: <MessageType>_<TransportMode>
enum PayloadType {
    // RpcEnvelope types
    RPC_RELIABLE = 0;           // RpcEnvelope with reliable ordered transmission - DEFAULT
    RPC_SIGNAL = 1;             // RpcEnvelope with high-priority signaling channel

    // StreamChunk types
    STREAM_RELIABLE = 2;        // StreamChunk with reliable ordered transmission
    STREAM_LATENCY_FIRST = 3;   // StreamChunk with low latency partial-reliable transmission

    // MediaFrame types
    MEDIA_RTP = 4;              // MediaFrame over WebRTC RTP MediaTrack
}

// Method-level option for lane selection
extend google.protobuf.MethodOptions {
    optional PayloadType payload_type = 50001;
}
```

**说明**：
- **RPC_*** 用于 RpcEnvelope（State Path，经过 Mailbox）
- **STREAM_*** 用于 StreamChunk（Fast Path，直接回调）
- **MEDIA_RTP** 用于 MediaFrame（Fast Path，RTP 传输）
- 默认值为 `RPC_RELIABLE`（可靠有序的 RPC 调用）

### 2.2 使用示例

```protobuf
// example.proto

syntax = "proto3";
package myapp;

import "actr/options.proto";

/// 聊天服务
service ChatService {
  // 默认：RPC_RELIABLE 通道
  rpc GetHistory(HistoryRequest) returns (HistoryResponse);

  // 标注：RPC_SIGNAL 通道（高优先级）
  rpc SendUrgentMessage(UrgentMessageRequest) returns (MessageResponse) {
    option (actr.payload_type) = RPC_SIGNAL;
  }

  // 标注：STREAM_LATENCY_FIRST 通道（低延迟流式）
  rpc StreamMessages(StreamRequest) returns (stream Message) {
    option (actr.payload_type) = STREAM_LATENCY_FIRST;
  }
}

/// 媒体服务（service 级别默认）
service MediaService {
  option (actr.default_payload_type) = STREAM_LATENCY_FIRST;

  // 继承 service 默认：STREAM_LATENCY_FIRST
  rpc StreamVideo(VideoRequest) returns (stream VideoFrame);

  // 方法级别覆盖：RPC_SIGNAL
  rpc StopStream(StopRequest) returns (StopResponse) {
    option (actr.payload_type) = RPC_SIGNAL;
  }
}
```

---

## 3. 默认通道策略

### 3.1 默认规则

| RPC 类型 | Option 标注 | PayloadType | 说明 |
|---------|-----------|------------|------|
| Unary RPC | 未标注 | RPC_RELIABLE | 默认走可靠通道 |
| Unary RPC | `RPC_SIGNAL` | RPC_SIGNAL | 高优先级紧急消息 |
| Unary RPC | `RPC_RELIABLE` | RPC_RELIABLE | 显式声明（同默认） |
| Streaming RPC | 未标注 | STREAM_RELIABLE | 默认走可靠流式通道 |
| Streaming RPC | `STREAM_LATENCY_FIRST` | STREAM_LATENCY_FIRST | 低延迟流式传输 |
| MediaTrack API | 自动识别 | MEDIA_RTP | 每轨独立 RTP 通道 |

### 3.2 PayloadType 与 Lane 映射

根据 Transport 层的路由表 (`route_table.rs`)，每个 PayloadType 对应一组可选的 LaneType:

| PayloadType | 可用 LaneType | 说明 |
|------------|--------------|------|
| RPC_RELIABLE | WebRtcDataChannel(Reliable), WebSocket | 可靠有序，经过 Mailbox |
| RPC_SIGNAL | WebRtcDataChannel(Signal), WebSocket | 高优先级，经过 Mailbox |
| STREAM_RELIABLE | WebRtcDataChannel(Reliable), WebSocket | 流式可靠传输 |
| STREAM_LATENCY_FIRST | WebRtcDataChannel(LatencyFirst), WebSocket | 低延迟流式传输 |
| MEDIA_RTP | WebRtcMediaTrack | 独占 RTP 媒体轨道 |

**关键设计**：
- **静态路由表**：编译时确定，零运行时开销
- **降级支持**：WebRTC 失败自动降级到 WebSocket
- **MediaTrack 独占**：仅支持 WebRTC RTP MediaTrack

---

## 4. 代码生成器实现

### 4.1 读取 Option

```rust
// actr-codegen/src/payload_type_selector.rs

use prost_types::{DescriptorProto, MethodDescriptorProto};

/// 从方法描述符中提取 PayloadType
pub fn extract_payload_type(
    method: &MethodDescriptorProto,
    service_default: Option<PayloadType>,
) -> PayloadType {
    // 1. 检查方法级别 option
    if let Some(options) = &method.options {
        if let Some(payload_type) = options.get_extension(actr::PAYLOAD_TYPE_FIELD_NUMBER) {
            return payload_type;
        }
    }

    // 2. 回退到 service 级别默认值
    if let Some(default) = service_default {
        return default;
    }

    // 3. 最终默认：RPC_RELIABLE
    PayloadType::RpcReliable
}
```

### 4.2 生成代码

```rust
// 生成的 Rust 代码示例

// actr-codegen 为每个 RPC 生成带有 payload_type 信息的代码

impl ChatServiceClient {
    /// 生成的 RPC 方法：SendUrgentMessage
    pub async fn send_urgent_message(
        &self,
        request: UrgentMessageRequest,
    ) -> Result<MessageResponse> {
        // 代码生成器注入：payload_type = RPC_SIGNAL
        self.context.request_with_payload_type(
            request,
            PayloadType::RpcSignal,  // 从 proto option 读取
            "myapp.ChatService.SendUrgentMessage",
        ).await
    }

    /// 生成的 RPC 方法：GetHistory（默认）
    pub async fn get_history(
        &self,
        request: HistoryRequest,
    ) -> Result<HistoryResponse> {
        // 未标注，使用默认 RPC_RELIABLE
        self.context.request_with_payload_type(
            request,
            PayloadType::RpcReliable,  // 默认值
            "myapp.ChatService.GetHistory",
        ).await
    }
}
```

---

## 5. Runtime 实现

### 5.1 Context API 扩展

```rust
// actr-framework/src/context.rs

impl Context {
    /// 带 PayloadType 的 RPC 调用（由代码生成器调用）
    pub async fn request_with_payload_type<Req, Resp>(
        &self,
        request: Req,
        payload_type: PayloadType,
        route_key: &str,
    ) -> Result<Resp>
    where
        Req: Message,
        Resp: Message,
    {
        // 1. 从 DestTransport 获取对应的 Lane
        let lane = self.dest_transport.get_lane(payload_type).await?;

        // 2. 编码消息为 RpcEnvelope
        // 注意：traceparent 和 tracestate 会通过 OpenTelemetry 自动注入
        let envelope = RpcEnvelope {
            route_key: route_key.to_string(),
            payload: request.encode_to_vec().into(),
            traceparent: None,  // 自动注入
            tracestate: None,   // 自动注入
            request_id: Uuid::new_v4().to_string(),
            metadata: HashMap::new(),
        };

        // 3. 通过选中的 Lane 发送
        lane.send(envelope.encode()?).await?;

        // 4. 等待响应...
        todo!()
    }

    /// 用户友好的 API（向后兼容）
    /// 默认使用 RPC_RELIABLE 通道
    pub async fn request<Req, Resp>(&self, request: Req) -> Result<Resp>
    where
        Req: Message,
        Resp: Message,
    {
        self.request_with_payload_type(request, PayloadType::RpcReliable, "").await
    }
}
```

### 5.2 DestTransport（目标传输层）

```rust
// actr-runtime/src/transport/dest_transport.rs

use actr_protocol::PayloadType;
use crate::transport::Lane;

/// 单个目标的传输控制器
pub struct DestTransport {
    /// 目标地址
    dest: Dest,

    /// Connection (WebRTC or WebSocket)
    connection: Arc<RwLock<Option<Connection>>>,

    /// 内部缓存 Lane (Connection 内部实现)
}

impl DestTransport {
    /// 根据 PayloadType 获取对应的 Lane
    pub async fn get_lane(&self, payload_type: PayloadType) -> NetworkResult<Lane> {
        // 1. 等待连接就绪（事件驱动，零轮询）
        let conn = self.wait_connection_ready().await?;

        // 2. 从 Connection 获取或创建 Lane（内部缓存）
        conn.get_lane(payload_type).await
    }

    /// 等待连接就绪（使用 tokio::sync::watch）
    async fn wait_connection_ready(&self) -> NetworkResult<Arc<Connection>> {
        // 事件驱动实现，延迟 < 1ms
        // ...
    }
}
```

**关键设计**：
- **DestTransport 按需创建**：每个目标延迟初始化
- **Connection 内部缓存 Lane**：避免重复创建
- **事件驱动连接管理**：零轮询，延迟 < 1ms

### 5.3 Connection 层 Lane 缓存

```rust
// actr-runtime/src/wire/webrtc/connection.rs

pub struct WebRtcConnection {
    peer_connection: Arc<RTCPeerConnection>,

    /// DataChannel Cache：PayloadType → DataChannel（4 种 DataChannel 类型）
    data_channels: Arc<RwLock<[Option<Arc<RTCDataChannel>>; 4]>>,

    /// Lane Cache：PayloadType → Lane（4 种 DataChannel 类型）
    lane_cache: Arc<RwLock<[Option<Lane>; 4]>>,

    /// MediaTrack Cache：stream_id → Track（TODO: 支持多媒体流）
    media_tracks: Arc<RwLock<HashMap<String, Arc<String>>>>,
}

impl WebRtcConnection {
    /// 获取或创建 Lane（带缓存）
    pub async fn get_lane(&self, payload_type: PayloadType) -> NetworkResult<Lane> {
        if payload_type == PayloadType::MediaRtp {
            return Err(NetworkError::NotImplemented("MediaTrack Lane not implemented yet"));
        }

        let idx = payload_type as usize;

        // 1. 检查缓存
        if let Some(lane) = &self.lane_cache.read().await[idx] {
            return Ok(lane.clone());
        }

        // 2. 创建新 Lane
        let lane = self.create_lane_internal(payload_type).await?;

        // 3. 缓存
        self.lane_cache.write().await[idx] = Some(lane.clone());

        Ok(lane)
    }
}
```

**关键设计**：
- **Connection 内部缓存**：避免重复创建 Lane
- **数组索引优化**：PayloadType 值直接作为索引
- **MediaRtp 独立处理**：需要 track_id，暂未实现

---

## 6. 配置与初始化

### 6.1 Transport 层初始化

```rust
// actr-runtime/src/transport/manager.rs

impl TransportManager {
    pub async fn new() -> Self {
        Self {
            dests: Arc::new(RwLock::new(HashMap::new())),
        }
    }

    /// 获取目标传输控制器（延迟创建）
    pub async fn get_dest_transport(&self, dest: &Dest) -> NetworkResult<Arc<DestTransport>> {
        // 1. 检查缓存
        if let Some(dt) = self.dests.read().await.get(dest) {
            return Ok(dt.clone());
        }

        // 2. 双重检查锁创建
        let mut dests = self.dests.write().await;
        if let Some(dt) = dests.get(dest) {
            return Ok(dt.clone());
        }

        // 3. 创建新的 DestTransport
        let dt = Arc::new(DestTransport::new(dest.clone()).await?);
        dests.insert(dest.clone(), dt.clone());

        Ok(dt)
    }
}
```

**关键设计**：
- **完全延迟创建**：DestTransport 按需初始化
- **Connection 自动建立**：支持 WebRTC P2P 和 WebSocket C/S
- **Lane 自动创建**：Connection 内部按 PayloadType 缓存

### 6.2 配置文件示例

```toml
# Actr.toml

[actor]
# Actor 实例配置
actor_id = "manufacturer:service_name@instance_001:tenant_default"

[network]
# 网络配置
signaling_server = "wss://signaling.example.com"
ice_servers = ["stun:stun.l.google.com:19302"]

[mailbox]
# Mailbox 配置
capacity = 1000
db_path = "./data/mailbox.db"
```

**说明**：
- ✅ **零 Lane 配置**：Lane 由框架根据 PayloadType 自动创建
- ✅ **用户通过 proto option 声明**：在 .proto 文件中标注 `option (actr.payload_type) = RPC_SIGNAL`
- ✅ **完全声明式**：用户只关注业务语义，传输层细节完全透明
- ✅ **配置文件简洁**：只包含必要的网络和业务配置

---

## 7. 完整数据流

### 7.1 RPC 调用流程

```
┌─────────────────────────────────────────────────────────┐
│  1. 用户代码调用                                          │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  client.send_urgent_message(req).await?                 │
└─────────────────────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  2. 生成的 Stub 代码                                      │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  ctx.request_with_payload_type(                          │
│      req,                                                │
│      PayloadType::RpcSignal,  // 从 proto option 读取    │
│      route_key                                           │
│  )                                                       │
└─────────────────────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  3. Context 调用 DestTransport                           │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  lane = dest_transport.get_lane(PayloadType::RpcSignal) │
│  // 等待连接就绪（事件驱动，< 1ms）                       │
└─────────────────────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  4. Connection 返回缓存的 Lane                            │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  lane = connection.get_lane(PayloadType::RpcSignal)     │
│  // 使用 WebRtcDataChannel(Signal) 或 WebSocket         │
└─────────────────────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  5. 通过 Lane 发送数据                                    │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  lane.send(envelope.encode()?).await?                    │
│  // WebRTC DataChannel 或 WebSocket binary frame        │
└─────────────────────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  6. 对端 WebRtcGate 路由                                 │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  match payload_type {                                    │
│      RpcReliable | RpcSignal => mailbox.enqueue(),       │
│      ...                                                 │
│  }                                                       │
└─────────────────────────────────────────────────────────┘
```

### 7.2 StreamChunk 流程（Fast Path）

```
┌─────────────────────────────────────────────────────────┐
│  1. 用户代码发送流式数据                                  │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  stream.send_chunk(data, channel_id).await?              │
└─────────────────────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  2. 封装为 StreamChunk                                    │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  chunk = StreamChunk {                                   │
│      channel_id, sequence, data, ...                     │
│  }                                                       │
│  // PayloadType: STREAM_LATENCY_FIRST 或 STREAM_RELIABLE│
└─────────────────────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  3. 通过对应 Lane 发送                                    │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  lane = get_lane(PayloadType::StreamLatencyFirst)       │
│  lane.send(chunk.encode()?).await?                       │
└─────────────────────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  4. 对端 WebRtcGate 路由                                 │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  match payload_type {                                    │
│      StreamReliable | StreamLatencyFirst =>              │
│          DataStreamRegistry.dispatch(),  // 直接回调     │
│      ...                                                 │
│  }                                                       │
└─────────────────────────────────────────────────────────┘
```

---

## 8. 实现优先级

### 阶段 1: 基础设施 ✅
1. ✅ Protobuf option 扩展定义
2. ✅ LaneType/GateId enum
3. ✅ Lane/BorderGate trait

### 阶段 2: 代码生成器 ⏳
1. ⏳ 读取 protobuf option
2. ⏳ 生成带 lane_type 的 stub 代码
3. ⏳ 单元测试：验证 option 正确读取

### 阶段 3: Runtime 实现 ⏳
1. ⏳ OutboundGate 实现
2. ⏳ MediaTrackManager 实现
3. ⏳ Context::request_with_lane() 实现
4. ⏳ InputHandler 实现（接收方向）

### 阶段 4: 集成测试 ⏳
1. ⏳ 端到端测试：RPC → Lane → Gate
2. ⏳ MediaTrack 动态创建测试
3. ⏳ 压力测试：高并发 RPC 调用

---

## 9. 关键设计决策

### 9.1 为什么使用 Protobuf Option？

| 方案 | 优势 | 劣势 |
|------|------|------|
| **Protobuf Option**（✅ 采用） | 声明式、类型安全、文档即代码 | 需要代码生成器支持 |
| 配置文件映射 | 灵活 | 容易出错、与 proto 分离 |
| 代码手动指定 | 简单 | 侵入业务代码、难以维护 |

### 9.2 为什么 PayloadType 按需创建 Lane？

- **延迟创建**：未使用的 PayloadType 不占用资源
- **缓存复用**：Connection 内部缓存 Lane，避免重复创建
- **零配置**：无需用户配置，框架自动管理
- **类型安全**：PayloadType 枚举值直接作为数组索引

### 9.3 为什么不需要 LanePool？

- **每 PayloadType 一条**：5 种 PayloadType，最多 5 条 Lane
- **Connection 内部复用**：多次 get_lane(相同 PayloadType) 返回同一实例
- **WebRTC 自身优化**：单条 DataChannel 足够，内部已做流控和拥塞控制
- **避免过度设计**：YAGNI 原则，当前场景不需要池化

### 9.4 为什么 MediaRtp 暂未实现？

- **复杂性较高**：需要 WebRTC MediaStreamTrack API
- **需求不明确**：优先实现 RPC 和 StreamChunk
- **可扩展性**：预留 PayloadType 枚举值，未来按需实现

### 9.5 为什么不需要配置文件配置 Lane？

- **固定 QoS 参数**：WebRTC DataChannel 的 QoS 由 PayloadType 决定
  - RPC_SIGNAL / RPC_RELIABLE: ordered=true, max_retransmits=∞
  - STREAM_RELIABLE: ordered=true, max_retransmits=∞
  - STREAM_LATENCY_FIRST: ordered=false, max_retransmits=3, maxLife=100ms

- **声明式路由**：用户通过 protobuf option 声明即可
  ```protobuf
  rpc KickUser(...) returns (...) {
    option (actr.payload_type) = RPC_SIGNAL;  // 声明式
  }
  ```
  不需要在配置文件中再配置一次

- **关注点分离**：
  - **框架负责**：Lane 的创建和 QoS 参数
  - **用户负责**：通过 proto option 声明业务语义
  - **配置文件负责**：WebRTC 连接、Mailbox 等业务配置

- **简化用户体验**：配置文件只包含必要的业务配置，传输层细节对用户透明

---

## 10. 示例：完整应用

```protobuf
// chat_service.proto

syntax = "proto3";
package chat;

import "actr/options.proto";

service ChatService {
  // 默认通道：获取历史消息 (RPC_RELIABLE)
  rpc GetHistory(HistoryRequest) returns (HistoryResponse);

  // Signal 通道：紧急踢人指令
  rpc KickUser(KickRequest) returns (KickResponse) {
    option (actr.payload_type) = RPC_SIGNAL;
  }

  // Latency-First 通道：实时消息流
  rpc StreamMessages(StreamRequest) returns (stream Message) {
    option (actr.payload_type) = STREAM_LATENCY_FIRST;
  }

  // Latency-First 通道：在线状态订阅
  rpc SubscribePresence(PresenceRequest) returns (stream PresenceUpdate) {
    option (actr.payload_type) = STREAM_LATENCY_FIRST;
  }
}
```

```rust
// 生成的代码使用

#[async_trait]
impl WorkloadHandler<ChatMessage> for ChatActor {
    async fn handle(&mut self, msg: ChatMessage, ctx: &mut Context) -> Result<()> {
        // 场景 1: 普通消息 → 默认通道
        let history = ctx.request(GetHistoryRequest { limit: 100 }).await?;

        // 场景 2: 紧急指令 → Signal 通道（自动选择）
        ctx.request(KickRequest { user_id: "spam_bot" }).await?;

        // 场景 3: 实时订阅 → Latency-First 通道（自动选择）
        let mut stream = ctx.stream(StreamRequest { room_id: "lobby" }).await?;
        while let Some(msg) = stream.next().await {
            // 处理实时消息
        }

        Ok(())
    }
}
```

---

**总结**：

通过 **protobuf option 扩展** + **Transport 层静态路由** 实现声明式车道选择：

1. **声明式路由**：在 proto 文件中标注 `option (actr.payload_type) = RPC_SIGNAL`
2. **按需创建**：5 种 PayloadType，Connection 内部缓存 Lane
3. **静态路由表**：PayloadType → LaneType 映射编译时确定，零运行时开销
4. **事件驱动**：DestTransport 使用 tokio::sync::watch，零轮询，延迟 < 1ms
5. **自动降级**：WebRTC 失败自动降级到 WebSocket

达到**配置即代码、类型安全、零配置、性能高效**的目标。
