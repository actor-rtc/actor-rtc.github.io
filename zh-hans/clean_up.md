# 资源清理

## 问题总结

### 1. 缺乏统一的资源清理机制

**当前状态：没有主动的、协调的资源清理机制**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          当前资源管理现状                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  连接断开时：                                                                │
│                                                                             │
│  ┌──────────────────┐                                                       │
│  │ RTCPeerConnection│  ← 底层连接断开                                       │
│  │    (Closed)      │                                                       │
│  └────────┬─────────┘                                                       │
│           │                                                                 │
│           │  ❌ 无事件通知                                                   │
│           │                                                                 │
│           ▼                                                                 │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐          │
│  │  WirePool        │  │  DestTransport   │  │  OutprocOutGate  │          │
│  │  (仍持有旧连接)   │  │  (缓存未失效)    │  │  (请求仍在等待)  │          │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘          │
│           │                    │                    │                      │
│           │  ❌ 被动发现       │  ❌ 被动发现       │  ❌ 等待超时          │
│           │  (下次发送失败)    │  (下次使用报错)    │  (30s 后才失败)       │
│           ▼                    ▼                    ▼                      │
│       发送失败              使用报错             请求超时                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

| 层级 | 当前状态 | 问题 |
|------|---------|------|
| **OutprocOutGate** | 无清理逻辑 | `pending_requests` 等待超时才失败，用户需要等待 |
| **OutprocTransportManager** | 无清理逻辑 | 继续向已断开的连接发送消息 |
| **DestTransport** | 无清理逻辑 | 缓存的连接已失效但仍在使用 |
| **WirePool** | 被动检测 | 仅在下次发送时才发现连接已断开 |
| **WebRtcConnection** | AtomicBool 通知 | 仅触发一次，后续订阅者收不到 |
| **WebRtcCoordinator** | 部分清理 | 清理触发点不完整，正在创建的连接无法取消 |

### 3. 清理触发点不完整

| 场景 | 当前行为 | 问题 |
|------|---------|------|
| **收到新 Offer** | 直接建立新连接 | 旧连接资源泄漏 |
| **RoleAssignment 角色变更** | 直接按新角色行动 | 旧连接资源泄漏 |
| **ICE restart 失败** | 仅关闭 PeerConnection | 上层缓存未清理 |
| **DataChannel 关闭** | 仅标记状态 | 上层不感知 |
| **正在创建连接时收到清理信号** | 无法取消 | 资源浪费，可能产生竞态 |

### 4. 缺乏事件广播机制

| 当前实现 | 问题 |
|---------|------|
| `AtomicBool` 单次触发 | 只能通知一个订阅者，不支持多层协调清理 |
| 各层独立检测状态 | 无法同步，清理时机不一致 |
| 无统一事件类型 | 缺少 `ConnectionClosed`、`IceRestartCompleted` 等事件 |

---

## 解决方案：统一事件广播架构

为了解决上述问题，我们引入了**统一的事件广播机制**和**自上而下的清理链路**：

### 核心设计原则

1. **统一事件源**：使用 `ConnectionEventBroadcaster` (tokio broadcast channel) 作为唯一的事件分发中心
2. **单一外部监听器**：`OutprocOutGate` 作为唯一的外部事件订阅者，避免多层重复监听
3. **自上而下清理**：从应用层到底层的清理链路：OutprocOutGate → TransportManager → DestTransport → WirePool → Connection
4. **精确资源绑定**：`pending_requests` 与 `ActorId` 绑定，支持按 peer 精确清理
5. **主动通知机制**：连接状态变化实时广播，无需轮询或被动检测

### 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                    统一事件广播架构                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  事件源 (WebRtcConnection)                                   │
│         │                                                   │
│         ▼                                                   │
│  ConnectionEventBroadcaster (broadcast channel)             │
│         │                                                   │
│         ├─────────────────────────┬─────────────────────┐   │
│         ▼                         ▼                     ▼   │
│  OutprocOutGate          WebRtcCoordinator         (预留)  │
│  (外部监听器)             (内部监听器)                       │
│         │                         │                         │
│         ▼                         ▼                         │
│  自上而下清理链              内部状态清理                     │
│  (Layer 1-7)                (peers, candidates...)          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 事件类型定义

```rust
pub enum ConnectionEvent {
    /// PeerConnection 状态变化
    StateChanged { peer_id: ActrId, state: ConnectionState },
    
    /// DataChannel 关闭
    DataChannelClosed { peer_id: ActrId, payload_type: PayloadType },
    
    /// 连接完全关闭
    ConnectionClosed { peer_id: ActrId },
    
    /// ICE Restart 开始
    IceRestartStarted { peer_id: ActrId },
    
    /// ICE Restart 完成
    IceRestartCompleted { peer_id: ActrId, success: bool },
}

pub enum ConnectionState {
    New, Connecting, Connected, Disconnected, Failed, Closed,
}
```

---

## 每层清理机制详解

### 清理触发全景流程图

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           清理触发与传播流程                                              │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                            触发源 (Trigger Sources)                              │   │
│  ├─────────────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                                  │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │   │
│  │  │ RTCPeerConn    │  │ DataChannel    │  │ SignalingClient│  │ ICE Restart    │ │   │
│  │  │ 状态变化       │  │ 关闭           │  │ 收到消息       │  │ 失败           │ │   │
│  │  └───────┬────────┘  └───────┬────────┘  └───────┬────────┘  └───────┬────────┘ │   │
│  │          │                   │                   │                   │          │   │
│  │          │ state=Closed      │ on_close          │ Offer/            │ 重试耗尽  │   │
│  │          │                   │                   │ RoleAssignment    │          │   │
│  │          ▼                   ▼                   ▼                   ▼          │   │
│  │  ┌───────────────────────────────────────────────────────────────────────────┐ │   │
│  │  │                    WebRtcCoordinator.drop_peer_connection()               │ │   │
│  │  │                                                                           │ │   │
│  │  │  // 收到新 Offer 时                  // 收到 RoleAssignment 角色变更时     │ │   │
│  │  │  handle_offer() {                   handle_role_assignment() {            │ │   │
│  │  │    if peers.contains(from) {          if role_changed {                   │ │   │
│  │  │      drop_peer_connection(from)         drop_peer_connection(peer)        │ │   │
│  │  │    }                                  }                                   │ │   │
│  │  │  }                                  }                                     │ │   │
│  │  │                                                                           │ │   │
│  │  │  // ICE restart 失败时              // PeerConnection 状态变化时          │ │   │
│  │  │  do_ice_restart_inner() {           on_state_change(Closed) {             │ │   │
│  │  │    if retries_exhausted {             drop_peer_connection(peer)          │ │   │
│  │  │      drop_peer_connection(target)   }                                     │ │   │
│  │  │    }                                                                      │ │   │
│  │  │  }                                                                        │ │   │
│  │  └───────────────────────────────────────┬───────────────────────────────────┘ │   │
│  │                                          │                                     │   │
│  └──────────────────────────────────────────┼─────────────────────────────────────┘   │
│                                             │                                         │
│                                             │ event_tx.send(ConnectionClosed)         │
│                                             ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │                     broadcast::Sender<ConnectionEvent>                          │  │
│  │                                    │                                            │  │
│  │                                    │ (唯一订阅者)                                │  │
│  │                                    ▼                                            │  │
│  │                              [event_rx]                                         │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                     │                                                 │
│                                     ▼                                                 │
│                          ┌─────────────────────┐                                      │
│                          │ Layer 1:            │                                      │
│                          │ OutprocOutGate      │                                      │
│                          │ (唯一事件监听器)    │                                      │
│                          │                     │                                      │
│                          │ ConnectionClosed => │                                      │
│                          │ 1. closing_peers    │                                      │
│                          │    .insert(id)      │                                      │
│                          │ 2. 按ActorId清理    │                                      │
│                          │    pending_requests │                                      │
│                          │    (精确匹配)       │                                      │
│                          │ 3. 调用下层清理:    │                                      │
│                          │    transport_manager│                                      │
│                          │    .close_transport │                                      │
│                          └──────────┬──────────┘                                      │
│                                     │ close_transport()                               │
│                                     ▼                                                 │
│                          ┌─────────────────────┐                                      │
│                          │ Layer 2:            │                                      │
│                          │ OutprocTransport    │                                      │
│                          │ Manager             │                                      │
│                          │ (被动清理)          │                                      │
│                          │                     │                                      │
│                          │ close_transport =>  │                                      │
│                          │ - token.cancel()    │                                      │
│                          │ - transports.remove │                                      │
│                          │ - transport.close() │                                      │
│                          └──────────┬──────────┘                                      │
│                                     │ close()                                         │
│                                     ▼                                                 │
│                          ┌─────────────────────┐                                      │
│                          │ Layer 3:            │                                      │
│                          │ DestTransport       │                                      │
│                          │ (被动清理)          │                                      │
│                          │                     │                                      │
│                          │ close() =>          │                                      │
│                          │ - mark_closed       │                                      │
│                          │ - conn_mgr.close()  │                                      │
│                          └──────────┬──────────┘                                      │
│                                     │ close_all()                                     │
│                                     ▼                                                 │
│                          ┌─────────────────────┐                                      │
│                          │ Layer 4:            │                                      │
│                          │ WirePool            │                                      │
│                          │ (被动清理)          │                                      │
│                          │                     │                                      │
│                          │ close_all() =>      │                                      │
│                          │ - closed = true     │                                      │
│                          │ - conn.close()      │                                      │
│                          └──────────┬──────────┘                                      │
│                                     │ close()                                         │
│                                     ▼                                                 │
│                          ┌─────────────────────┐                                      │
│                          │ Layer 5:            │                                      │
│                          │ WireHandle          │                                      │
│                          │ (被动清理)          │                                      │
│                          │                     │                                      │
│                          │ close() =>          │                                      │
│                          │ - WebRTC.close()    │                                      │
│                          │ - WebSocket.close() │                                      │
│                          └──────────┬──────────┘                                      │
│                                     │ close()                                         │
│                                     ▼                                                 │
│                          ┌─────────────────────┐                                      │
│                          │ Layer 6:            │                                      │
│                          │ WebRtcConnection    │                                      │
│                          │ (被动清理)          │                                      │
│                          │                     │                                      │
│                          │ close() =>          │                                      │
│                          │ - pc.close()        │                                      │
│                          │ - clear channels    │                                      │
│                          └─────────────────────┘                                      │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### 触发源汇总

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              所有清理触发源                                              │
├────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                        │
│  1. RTCPeerConnection 状态变化                                                          │
│     └── state = Closed → drop_peer_connection()                                        │
│                                                                                        │
│  2. DataChannel 关闭                                                                    │
│     └── on_close → event_tx.send(DataChannelClosed) → WirePool 标记 Failed             │
│                                                                                        │
│  3. 收到新 Offer (handle_offer)                                                         │
│     └── 检测到旧连接存在 → 清理旧连接 → 建立新连接                                        │
│                                                                                        │
│  4. 收到新 RoleAssignment 且角色变更 (handle_role_assignment)                            │
│     └── old_role != new_role → drop_peer_connection() → 按新角色行动                    │
│                                                                                        │
│  5. ICE restart 失败 (do_ice_restart_inner)                                             │
│     └── 重试次数耗尽 → drop_peer_connection() → 尝试新建连接                             │
│                                                                                        │
│  6. 外部调用 close_all_peers                                                            │
│     └── 关闭所有 PeerConnection → 清理所有状态                                           │
│                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 取消正在进行的连接创建

### 取消机制架构图

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           CancellationToken 取消机制                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│                         创建连接请求                                                     │
│                              │                                                          │
│                              ▼                                                          │
│  ┌───────────────────────────────────────────────────────────────────────────────────┐  │
│  │                    OutprocTransportManager                                         │  │
│  │                                                                                    │  │
│  │  async fn get_or_create_transport(dest) {                                         │  │
│  │      // 检查是否正在关闭                                                            │  │
│  │      if closing_peers.contains(dest) {                                            │  │
│  │          return Err(ConnectionClosed)  ◄───── 快速失败                             │  │
│  │      }                                                                             │  │
│  │                                                                                    │  │
│  │      // 创建 CancellationToken                                                     │  │
│  │      let token = CancellationToken::new();                                        │  │
│  │      tokens.insert(dest, token.clone());                                          │  │
│  │                                                                                    │  │
│  │      // 调用 WireBuilder 创建连接                                                   │  │
│  │      wire_builder.create_connections(dest, Some(token)).await                     │  │
│  │  }                                                                                 │  │
│  │                                                                                    │  │
│  │  // 收到关闭事件时                                                                  │  │
│  │  fn on_connection_closed(peer_id) {                                               │  │
│  │      if let Some(token) = tokens.remove(peer_id) {                                │  │
│  │          token.cancel()  ◄───── 取消进行中的连接创建                                │  │
│  │      }                                                                             │  │
│  │  }                                                                                 │  │
│  └───────────────────────────────────────────────────────────────────────────────────┘  │
│                              │                                                          │
│                              │ create_connections(dest, token)                          │
│                              ▼                                                          │
│  ┌───────────────────────────────────────────────────────────────────────────────────┐  │
│  │                      DefaultWireBuilder                                            │  │
│  │                                                                                    │  │
│  │  async fn create_connections(dest, token) {                                       │  │
│  │      // WebRTC 连接                                                                │  │
│  │      if token.is_cancelled() { return Err(ConnectionClosed) }  ◄── 检查点1        │  │
│  │                                                                                    │  │
│  │      let webrtc_conn = coordinator.create_connection(dest, token.clone()).await;  │  │
│  │                                                                                    │  │
│  │      // WebSocket 连接                                                             │  │
│  │      if token.is_cancelled() { return Err(ConnectionClosed) }  ◄── 检查点2        │  │
│  │                                                                                    │  │
│  │      let ws_conn = create_websocket(dest).await;                                  │  │
│  │  }                                                                                 │  │
│  └───────────────────────────────────────────────────────────────────────────────────┘  │
│                              │                                                          │
│                              │ create_connection(dest, token)                           │
│                              ▼                                                          │
│  ┌───────────────────────────────────────────────────────────────────────────────────┐  │
│  │                      WebRtcCoordinator                                             │  │
│  │                                                                                    │  │
│  │  async fn create_connection(dest, token) {                                        │  │
│  │      if token.is_cancelled() { return Err(Cancelled) }  ◄── 检查点3               │  │
│  │                                                                                    │  │
│  │      let pc = negotiator.create_peer_connection().await;                          │  │
│  │                                                                                    │  │
│  │      if token.is_cancelled() {                           ◄── 检查点4              │  │
│  │          pc.close().await;  // 清理已创建的资源                                     │  │
│  │          return Err(Cancelled);                                                    │  │
│  │      }                                                                             │  │
│  │                                                                                    │  │
│  │      // 等待连接就绪（可能被取消）                                                   │  │
│  │      tokio::select! {                                                              │  │
│  │          _ = token.cancelled() => {                      ◄── 取消等待              │  │
│  │              pc.close().await;                                                     │  │
│  │              return Err(Cancelled);                                                │  │
│  │          }                                                                         │  │
│  │          result = ready_rx => { ... }                                              │  │
│  │      }                                                                             │  │
│  │  }                                                                                 │  │
│  └───────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 取消检查点汇总

| 层级 | 检查点位置 | 取消后动作 |
|------|-----------|-----------|
| OutprocTransportManager | `get_or_create_transport` 入口 | 直接返回 `ConnectionClosed` 错误 |
| DefaultWireBuilder | `create_connections` 每个连接创建前 | 返回 `ConnectionClosed` 错误 |
| WebRtcCoordinator | `create_connection` 入口 | 返回 `Cancelled` 错误 |
| WebRtcCoordinator | `create_peer_connection` 后 | 关闭 PC 后返回 `Cancelled` |
| WebRtcCoordinator | 等待 `ready_rx` 时 | 使用 `select!` 监听取消信号 |

---

## 完整清理链路

### 统一清理流程（事件触发 → 自上而下清理链）

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                            完整清理链路（自上而下）                                           │
├─────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 事件源：WebRtcConnection / Coordinator                                                 │  │
│  │ 发送: ConnectionClosed, DataChannelClosed, StateChanged                               │  │
│  └─────────────────────────────────────────┬───────────────────────────────────────────────┘  │
│                                            │                                                 │
│                                            ▼ (broadcast)                                     │
│  ┌───────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                    ConnectionEventBroadcaster                                          │  │
│  └───────────────────────┬───────────────────────────┬───────────────────────────────────┘  │
│                          │                           │                                       │
│                          ▼ (订阅1: 外部)              ▼ (订阅2: 内部)                         │
│              ┌──────────────────────┐    ┌──────────────────────────────┐                   │
│              │  OutprocOutGate      │    │  WebRtcCoordinator           │                   │
│              │  (唯一外部事件监听器) │    │  (内部事件监听器)            │                   │
│              └──────────┬───────────┘    └───────────┬──────────────────┘                   │
│                         │                            │                                       │
│                         ▼                            ▼                                       │
│  ┌─────────────────────────────────┐   ┌────────────────────────────────────────────────┐   │
│  │ Layer 1: OutprocOutGate         │   │ WebRtcCoordinator                              │   │
│  │ ─────────────────────────────   │   │ ──────────────────                             │   │
│  │ 1. closing_peers.insert(id)     │   │ cleanup_cancelled_connection(peer_id) {        │   │
│  │ 2. 按 ActorId 清理               │   │   - in_flight_restarts.remove().abort()        │   │
│  │    pending_requests:             │   │   - peers.remove(peer_id)                      │   │
│  │    HashMap<String, (ActorId, tx)>│   │   - peer_connection.close()                    │   │
│  │ 3. 触发下层清理链:               │   │   - webrtc_conn.close()                        │   │
│  │    transport_manager             │   │   - pending_candidates.remove()                │   │
│  │    .close_transport(dest)        │   │   - negotiated_role.remove()                   │   │
│  └──────────┬──────────────────────┘   │   - pending_role.remove()                      │   │
│             │                           │   - pending_ready.remove()                     │   │
│             ▼                           │ }                                              │   │
│  ┌─────────────────────────────────┐   └────────────────────────────────────────────────┘   │
│  │ Layer 2: OutprocTransportManager│                                                         │
│  │ ─────────────────────────────   │                                                         │
│  │ close_transport(dest) {         │                                                         │
│  │   - token.cancel()              │   // 取消正在创建的连接                                  │
│  │   - transports.remove(dest)     │                                                         │
│  │   - transport.close()           │   // 调用下层                                            │
│  │ }                               │                                                         │
│  └──────────┬──────────────────────┘                                                         │
│             ▼                                                                                 │
│  ┌─────────────────────────────────┐                                                         │
│  │ Layer 3: DestTransport          │                                                         │
│  │ ─────────────────────────────   │                                                         │
│  │ close() {                       │                                                         │
│  │   for conn in connections {     │                                                         │
│  │     conn.close()                │                                                         │
│  │     conn_mgr.mark_closed(type)  │   // 标记连接为关闭                                      │
│  │   }                             │                                                         │
│  │   conn_mgr.close_all()          │   // 清理 WirePool                                      │
│  │ }                               │                                                         │
│  └──────────┬──────────────────────┘                                                         │
│             ▼                                                                                 │
│  ┌─────────────────────────────────┐                                                         │
│  │ Layer 4: WirePool               │                                                         │
│  │ ─────────────────────────────   │                                                         │
│  │ close_all() {                   │                                                         │
│  │   - closed.store(true)          │   // 终止后台连接任务                                    │
│  │   - connections.clear()         │                                                         │
│  │   for handle in connections {   │                                                         │
│  │     handle.close()              │   // 调用下层                                            │
│  │   }                             │                                                         │
│  │   - ready_tx.send(empty_set)    │   // 通知连接不可用                                      │
│  │ }                               │                                                         │
│  └──────────┬──────────────────────┘                                                         │
│             ▼                                                                                 │
│  ┌─────────────────────────────────┐                                                         │
│  │ Layer 5: WireHandle             │                                                         │
│  │ ─────────────────────────────   │                                                         │
│  │ close() {                       │                                                         │
│  │   match self {                  │                                                         │
│  │     WebRTC(conn) =>             │                                                         │
│  │       conn.close()              │   // WebRTC 连接清理                                     │
│  │     WebSocket(ws) =>            │                                                         │
│  │       ws.close()                │   // WebSocket 连接清理                                  │
│  │   }                             │                                                         │
│  │ }                               │                                                         │
│  └──────────┬──────────────────────┘                                                         │
│             ▼                                                                                 │
│  ┌─────────────────────────────────┐                                                         │
│  │ Layer 6: WebRtcConnection       │                                                         │
│  │ ─────────────────────────────   │                                                         │
│  │ close() {                       │                                                         │
│  │   - connected = false           │                                                         │
│  │   - peer_connection.close()     │   // RTCPeerConnection 底层关闭                          │
│  │   - data_channels.clear()       │   // 清理 DataChannel                                   │
│  │   - media_tracks.clear()        │   // 清理 MediaTrack                                    │
│  │   - lane_cache.clear()          │   // 清理缓存                                            │
│  │ }                               │                                                         │
│  └──────────┬──────────────────────┘                                                         │
│             ▼                                                                                 │
│  ┌─────────────────────────────────┐                                                         │
│  │ Layer 7: RTCPeerConnection      │                                                         │
│  │ (webrtc-rs 底层)                │                                                         │
│  └─────────────────────────────────┘                                                         │
│                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 清理资源汇总表

| 层级 | 组件 | 清理资源 | 触发方式 |
|------|------|---------|----------|
| **0** | WebRtcCoordinator (内部监听器) | `peers`, `pending_candidates`, `in_flight_restarts`, `negotiated_role`, `pending_role`, `pending_ready` | 订阅 ConnectionEvent (内部清理) |
| **1** | OutprocOutGate | `pending_requests: HashMap<String, (ActorId, Sender)>`, `closing_peers` | 订阅 ConnectionEvent (唯一外部监听器) |
| **2** | OutprocTransportManager | `transports: HashMap<Dest, DestTransport>`, `pending_tokens` | 被动（close_transport 调用） |
| **3** | DestTransport | 连接实例，`conn_mgr: WirePool` | 被动（close 调用） |
| **4** | WirePool | `connections: [Option<WireStatus>; 2]`, `closed` 标志 | 被动（close_all 调用） |
| **5** | WireHandle | WebRTC/WebSocket 连接句柄 | 被动（close 调用） |
| **6** | WebRtcConnection | `peer_connection`, `data_channels`, `media_tracks`, `lane_cache` | 被动（close 调用） |
| **7** | RTCPeerConnection | 底层 WebRTC 连接资源 | 被动（close 调用） |

---

## 触发清理资源的完整列表

| 触发事件 | 事件源 | 清理范围 | 取消进行中连接 | 说明 |
|---------|-------|---------|---------------|------|
| `StateChanged(Closed)` | WebRtcConnection | 完整清理 | ✅ | PeerConnection 已关闭 |
| `ConnectionClosed` | WebRtcConnection.close() | 完整清理 | ✅ | 连接完全断开 |
| `DataChannelClosed` | DataChannel.on_close | WirePool 标记 Failed | ✅ | datachannel close  |
| `StateChanged(Disconnected/Failed)` | RTCPeerConnection | 触发 ICE restart | ❌ | Offerer 发起重连 |
| ICE restart 失败 | Coordinator | 完整清理 | ✅ | 重试耗尽后清理 |
| **收到新 Offer** | SignalingClient | 旧连接清理 | ✅ | 清理后建立新连接 |
| **收到新 RoleAssignment** | SignalingClient | 旧连接清理 | ✅ | 角色变更需重新建链 |
| 外部调用 close_all_peers | 应用层 | 所有连接清理 | ✅ | 关闭时调用 |

### 收到新 Offer / RoleAssignment 的清理逻辑

```rust
// handle_offer 中
async fn handle_offer(&self, from: &ActrId, offer_sdp: String) -> RuntimeResult<()> {
    // 检查是否有旧连接
    if self.peers.read().await.contains_key(from) {
        tracing::info!("🔄 Existing connection found, cleaning up for new Offer");
        
        // 1. 取消 ICE restart 任务
        if let Some(handle) = self.in_flight_restarts.lock().await.remove(from) {
            handle.abort();
        }
        
        // 2. 关闭旧连接
        if let Some(old_state) = self.peers.write().await.remove(from) {
            old_state.peer_connection.close().await;
            old_state.webrtc_conn.close().await;
        }
        
        // 3. 清理缓存状态
        self.pending_candidates.write().await.remove(from);
        self.negotiated_role.lock().await.remove(from);
        self.pending_role.lock().await.remove(from);
        self.pending_ready.lock().await.remove(from);
    }
    
    // 建立新连接...
}

// handle_role_assignment 中
async fn handle_role_assignment(&self, assign: RoleAssignment, peer: ActrId) {
    // 检查角色是否变更
    let old_role = self.negotiated_role.lock().await.get(&peer).cloned();
    
    if let Some(old_is_offerer) = old_role {
        if old_is_offerer != assign.is_offerer {
            tracing::info!("🔄 Role changed for {}, cleaning up old connection", peer.serial_number);
            
            // 角色变更，需要清理旧连接
            self.drop_peer_connection(&peer).await;
        }
    }
    
    // 处理新角色...
}
```


## 需要修改的文件汇总

| 文件 | 操作 | 说明 |
|------|------|------|
| `src/transport/connection_event.rs` | **新建** ✅ | 定义 ConnectionEvent 和 ConnectionEventBroadcaster |
| `src/wire/webrtc/mod.rs` | 添加模块导出 ✅ | 导出 connection_event 模块 |
| `src/transport/mod.rs` | 添加模块导出 ✅ | 导出 connection_event 模块 |
| `src/wire/webrtc/connection.rs` | 重构通知机制 ✅ | 发送 ConnectionEvent（状态变化、关闭等） |
| `src/wire/webrtc/coordinator.rs` | 内部事件监听 ✅ | spawn_internal_event_listener 清理内部状态 |
| `src/transport/manager.rs` | 添加取消机制 ✅ | close_transport() 被动清理 |
| `src/transport/wire_pool.rs` | 添加 close_all() ✅ | closed 标志 + 被动清理 |
| `src/transport/wire_builder.rs` | 添加 cancel_token ✅ | 支持取消正在创建的连接 |
| `src/transport/dest_transport.rs` | 显式调用清理 ✅ | close() 中调用 conn_mgr.close_all() |
| `src/transport/error.rs` | 添加错误类型 ✅ | ConnectionClosed 错误 |
| `src/outbound/outproc_out_gate.rs` | **唯一外部事件监听器** ✅ | pending_requests 与 ActorId 绑定 + 自上而下清理 |
| `src/wire/webrtc/gate.rs` | pending_requests 更新 ✅ | 类型改为 HashMap<String, (ActorId, Sender)> |
