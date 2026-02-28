# 态势感知：文档与代码不一致问题

## 问题

多份文档（`appendix-glossary.zh.md`、`2.2-actor-cookbook.zh.md`、`1.2-framework-internal-protocols.zh.md` 等）描述了一个**独立的 `Lifecycle` trait**，但代码中**未实现**这个 trait。实际的生命周期钩子直接定义在 `Workload` trait 中。

### 文档描述（不存在的设计）

```rust
// ❌ 文档中描述的 Lifecycle trait，代码中不存在
pub trait Lifecycle {
    async fn on_start(&self, ctx: &Context);
    async fn on_ready(&self, ctx: &Context);    // 不存在
    async fn on_error(&self, ctx: &Context);    // 不存在，待新增到 Workload
    async fn on_stop(&self, ctx: &Context);
    async fn on_signaling_connected(&self, ctx: Arc<Context>);    // 不存在
    async fn on_signaling_disconnected(&self, ctx: Arc<Context>); // 不存在
}
```

### 代码实现（真实情况）

来源：`actr/crates/framework/src/workload.rs`

```rust
// ✅ 这是代码中实际存在的唯一实现
#[async_trait]
pub trait Workload: Send + Sync + 'static {
    type Dispatcher: MessageDispatcher<Workload = Self>;

    async fn on_start<C: Context>(&self, _ctx: &C) -> ActorResult<()> { Ok(()) }
    async fn on_stop<C: Context>(&self, _ctx: &C) -> ActorResult<()> { Ok(()) }

    // 没有 on_ready
    // 没有 on_error（待新增）
    // 没有 on_signaling_connected
    // 没有 on_signaling_disconnected
    // 没有 on_network_state
}
```

### 差异汇总

| 功能                        | 文档描述         | 代码实现        | 差异                             |
| --------------------------- | ---------------- | --------------- | -------------------------------- |
| `Lifecycle` trait           | 独立 trait       | **不存在**      | 生命周期钩子直接在 `Workload` 中 |
| `on_start`                  | `Lifecycle` 方法 | `Workload` 方法 | ⚠️ 位置不同                       |
| `on_stop`                   | `Lifecycle` 方法 | `Workload` 方法 | ⚠️ 位置不同                       |
| `on_ready`                  | `Lifecycle` 方法 | 不存在          | ❌                                |
| `on_error`                  | `Lifecycle` 方法 | 不存在          | ❌                                |
| `on_signaling_connected`    | `Lifecycle` 方法 | 不存在          | ❌                                |
| `on_signaling_disconnected` | `Lifecycle` 方法 | 不存在          | ❌                                |
| `ctx.log_info()`            | Context API      | 不存在          | ❌                                |
| `ctx.log_warn()`            | Context API      | 不存在          | ❌                                |
| `ctx.schedule_tell()`       | Context API      | 不存在          | ❌                                |

---

## 解决方案（待确认）

以代码为准，所有生命周期钩子直接加在 `Workload` trait 中。文档中描述的多个独立方法合并/精简为以下 4 个钩子。

### Workload 钩子清单

```rust
#[async_trait]
pub trait Workload: Send + Sync + 'static {
    type Dispatcher: MessageDispatcher<Workload = Self>;

    async fn on_start<C: Context>(&self, _ctx: &C) -> ActorResult<()> { Ok(()) }
    async fn on_stop<C: Context>(&self, _ctx: &C) -> ActorResult<()> { Ok(()) }
    async fn on_error<C: Context>(&self, _ctx: &C, _error: RuntimeError) -> ActorResult<()> { Ok(()) }
    async fn on_network_state<C: Context>(&self, _ctx: &C, _state: NetworkState) -> ActorResult<()> { Ok(()) }
}
```

| 钩子               | 触发时机                                 | 参数                  | 使用场景                                                                     |
| ------------------ | ---------------------------------------- | --------------------- | ---------------------------------------------------------------------------- |
| `on_start`         | ActrNode 启动完成，WebRTC 后台循环已就绪 | `ctx`                 | 初始化业务资源、向协调服务注册自己在线、启动定时任务                         |
| `on_stop`          | ActrNode 收到关闭信号，即将退出          | `ctx`                 | 释放业务资源、向协调服务注销、持久化内存状态                                 |
| `on_error`         | 框架捕获到运行时异常                     | `ctx`, `RuntimeError` | handler panic 后通知业务层                                                   |
| `on_network_state` | 网络连接状态发生变化                     | `ctx`, `NetworkState` | 信令断开时 UI 显示"重连中"、对端 WebRTC 断开时暂停数据流、连接恢复后同步状态 |

### 合并方法

文档中描述的独立方法合并到 `on_network_state`，通过 `NetworkState` 枚举区分：

| 文档中的方法                | 合并到             | 枚举值                      |
| --------------------------- | ------------------ | --------------------------- |
| `on_signaling_connected`    | `on_network_state` | `Signaling(Connected)`      |
| `on_signaling_disconnected` | `on_network_state` | `Signaling(Disconnected)`   |
| `on_ready`（信令就绪）      | `on_network_state` | `Signaling(Connected)`      |
| WebRTC 连接/断开            | `on_network_state` | `Webrtc { actr_id, state }` |

### NetworkState 定义

```rust
pub enum NetworkState {
    /// 信令状态（WebSocket → 信令服务器）
    Signaling(ConnectionState),
    /// WebRTC 状态（WebRTC → 对端）
    Webrtc { actr_id: ActrId, state: WebrtcState },
}

pub enum ConnectionState {
    Connecting,
    Connected,
    Disconnected,
}

pub enum WebrtcState {
    Connecting,
    Connected(TransportMode),
    Disconnected,
}

pub enum TransportMode {
    Direct,   // 直连
    Relayed,  // 中继
}
```

### 使用示例

```rust
// 网络状态感知
async fn on_network_state<C: Context>(&self, ctx: &C, state: NetworkState) -> ActorResult<()> {
    match state {
        NetworkState::Signaling(ConnectionState::Connected) => {
            tracing::info!("信令已连接，Actor 已就绪");
            self.is_ready.store(true, Ordering::Relaxed);
        }
        NetworkState::Signaling(ConnectionState::Disconnected) => {
            tracing::warn!("信令断开，暂停对外服务");
            self.is_ready.store(false, Ordering::Relaxed);
        }
        NetworkState::Webrtc { actr_id, state: WebrtcState::Disconnected } => {
            tracing::warn!("对端 {} 断开，近期发送可能丢失", actr_id);
            self.pause_stream_to(&actr_id).await;
        }
        NetworkState::Webrtc { actr_id, state: WebrtcState::Connected(mode) } => {
            tracing::info!("对端 {} 已连接 ({:?})", actr_id, mode);
            self.resume_stream_to(&actr_id).await;
        }
        _ => {}
    }
    Ok(())
}
```

### 关于 `ctx.log_info()` 等方法

不实现。Rust 生态已有成熟的 `tracing` crate，文档中移除 `ctx.log_info()`、`ctx.log_warn()` 等描述。

---

## 连接异常感知

WebRTC 回调返回 `()`，异常无法通过 `Result` 传播。业界做法（Pion、libwebrtc、mediasoup）是：**应用层关心"连接能不能用"，不关心"哪个回调出了什么错"**。框架内部消化回调异常，只把连接状态变化暴露给应用层。

```
回调异常（ICE/DC/SCTP 等）
    ↓
框架内部处理（日志 + 重试 + 重建通道）
    ↓
状态收敛：Connected / Disconnected / Reconnecting
    ↓
on_network_state 通知应用层
```

### 各回调当前处理与改进

| 回调               | 当前处理                          | 改进方案                                                           | 触发 `on_network_state`？      |
| ------------------ | --------------------------------- | ------------------------------------------------------------------ | ------------------------------ |
| `on_ice_candidate` | 信令发送失败只 log                | 日志记录即可，已有 ICE restart 和重连机制覆盖                      | 否（不影响连接状态）           |
| `on_data_channel`  | 注册失败只 log，未通知 `ready_tx` | 通知 `ready_tx` 失败信号，框架重试连接                             | 否（连接建立阶段）             |
| `dc.on_message`    | channel send 失败，消息静默丢弃   | `break` + 触发 `DataChannelClosed`，重建通道；**失败消息写入 DLQ** | **是**（通道关闭 → 状态变化）  |
| `dc.on_error`      | 只 log，不触发 lane 失效          | 与 `on_close` 统一：`invalidate_lane` + 重建通道                   | **是**（lane 失效 → 状态变化） |
| `on_track`         | RTP 读取失败 `break`              | 音频通道暂未实现，后续补充                                         | —                              |

> **注意**：不在回调中直接执行清理（可能死锁），通过内部事件驱动。

---

## 需通知应用层的态势感知事件

除了文档中描述的信令连接/断开等状态变化外，以下 WebRTC 事件也需要通过 `on_network_state` 通知应用层：

| 事件                                   | 通知原因                             |
| -------------------------------------- | ------------------------------------ |
| SCTP 断开（dc.on_close）               | 近期发送可能丢失，应用层需标记待同步 |
| DataChannel 关闭（dc.on_message 失败） | 通道不可用，框架重建中               |
| Lane 失效（dc.on_error）               | 通道异常，框架重建中                 |


### SCTP 断开时数据丢失（边界条件）

这是一个竞态窗口问题，仅在以下条件同时满足时触发：

1. **应用层正在持续发送数据**（如文件传输、流式推送）
2. **网络突然中断**（如 Wi-Fi 断开、基站切换、对端进程崩溃）
3. **发送操作恰好落在"写入缓冲区"到"SCTP 检测到断开"之间的窗口内** 

**根本原因**：`dc.send()` 是异步写入——数据交给 SCTP 发送缓冲区就返回 `Ok(())`，不等对端 SACK 确认。SCTP 本身有 SACK 机制保证可靠传输，但 WebRTC DataChannel API 不暴露 ACK 结果。当网络在这个窗口内断开，缓冲区中未 ACK 的数据丢失，而调用方已收到 `Ok(())`，以为发送成功了。

框架层面连接恢复没有问题（`dc.on_close` → `invalidate_lane` → 自动重连），但**调用方不知道断开前的消息丢了**。

以下日志展示了这个竞态窗口：

```
06:08:30.762  send_data_stream seq=11 → Ok(())     ← 调用方以为成功
06:08:30.783  SCTP: Alert is Fatal or Close Notify  ← 底层发现断了（20ms 后）
06:08:30.783  DataChannel closed × 4                ← on_close 触发清理
06:08:31.763  send_data_stream seq=12               ← 重连后继续发送（seq=11 已丢失）
```

框架通过 `on_network_state` 通知应用层"连接断了，近期发送可能丢失"，由应用层决定是否重新同步。

**通知链路**：

```
SCTP Fatal Alert
    │
    ▼
dc.on_close (× N 个 DataChannel)          ← connection.rs
    ├── invalidate_lane                    ← 已有
    └── notify_data_channel_closed         ← 已有
            │
            ▼
    Coordinator 处理                       ← coordinator.rs
    ├── 通知 ConnectionEvent::ConnectionClosed    ← 已有
    └── 🆕 通知 ConnectionEvent::PeerConnectionLost
            │
            ▼
    OutprocOutGate 事件监听                ← outproc_out_gate.rs
    ├── 清理 pending_requests，返回 Err    ← 已有
    ├── close_transport                    ← 已有
    └── 🆕 通知 Workload::on_network_state(Webrtc { Disconnected })
```

应用层实现示例：

```rust
async fn on_network_state<C: Context>(&self, ctx: &C, state: NetworkState) -> ActorResult<()> {
    match state {
        NetworkState::Webrtc { actr_id, state: WebrtcState::Disconnected } => {
            tracing::warn!("Peer {} disconnected, recent sends may be lost", actr_id);
            self.mark_stream_needs_resync(&actr_id).await;
        }
        _ => {}
    }
    Ok(())
}
```

> 本质上 SCTP 断开不算错误（`dc.send()` 返回 Ok，协议层没有违约），而是**网络状态变化引发的数据完整性风险**，所以通过 `on_network_state` 而非 `on_error` 通知。竞态窗口无法完全消除，最终依赖**应用层 ACK + 断点续传**。

## 需要修改的文档

| 文档                                     | 修改内容                                                                                     |
| ---------------------------------------- | -------------------------------------------------------------------------------------------- |
| `appendix-glossary.zh.md`                | Lifecycle 词条：改为"生命周期钩子定义在 `Workload` trait 中"，列出 4 个钩子                  |
| `2.2-actor-cookbook.zh.md`               | 模式六"网络状态感知"：`impl Lifecycle` → `impl Workload`，用 `on_network_state` 替代独立方法 |
| `1.2-framework-internal-protocols.zh.md` | Lifecycle trait 定义段落：与代码对齐                                                         |
| `appendix-runtime-design.zh.md`          | 移除对独立 `Lifecycle` trait 的 `downcast_ref` 等描述                                        |
| `4.1-overall.zh.md`                      | actr-framework 核心接口列表中移除独立 `Lifecycle`                                            |
| Context 相关描述                         | 移除 `ctx.log_info()`、`ctx.schedule_tell()` 等不存在的 API                                  |
