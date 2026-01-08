# Network Event Handling

## 概述

网络事件处理模块提供了一套完整的基础设施，用于处理平台层（Android/iOS）的网络状态变化事件，并自动触发相应的恢复机制（信令重连、ICE 重启等）。

## 架构设计

### 核心架构图

```text
        (FFI Path - Implemented)  (Actor Path - TODO)
                ↓                           ↓
    ┌─ NetworkEventHandle ─┐    ┌─ Direct Proto Message ─┐
    │ • Platform FFI calls │    │ • Actor call/tell       │
    │ • Send via channel   │    │ • Send to mailbox       │
    │ • Await result       │    │ • No handle needed      │
    └──────────┬───────────┘    └──────────┬─────────────┘
               │                           │
               └──────────┬────────────────┘
                          │ Both trigger
                          ↓
              ActrNode::network_event_loop()
                  • Receive from channel
                  • Delegate to processor
                  • Send result back
                          ↓
              NetworkEventProcessor
                  • process_network_available()
                  • process_network_lost()
                  • process_network_type_changed()
```

### 设计原则

1. **解耦设计**：通过 channel 实现调用方与处理方的完全解耦（目前使用FFI，后续可以支持Proto Message）
2. **异步处理**：网络事件在后台异步处理，不阻塞平台层
3. **结果反馈**：处理结果（成功/失败/耗时）反馈给调用方
4. **可扩展性**：通过 Trait 支持自定义处理逻辑

## 核心组件

### 1. NetworkEvent（网络事件）

定义了三种网络事件类型：

```rust
pub enum NetworkEvent {
    /// 网络可用（从断网恢复）
    Available,
    
    /// 网络丢失（断网）
    Lost,
    
    /// 网络类型变化（WiFi ↔ Cellular）
    TypeChanged { is_wifi: bool, is_cellular: bool },
}
```

### 2. NetworkEventResult（处理结果）

封装了事件处理的结果信息：

```rust
pub struct NetworkEventResult {
    /// 事件类型
    pub event: NetworkEvent,
    
    /// 处理是否成功
    pub success: bool,
    
    /// 错误信息（如果失败）
    pub error: Option<String>,
    
    /// 处理耗时（毫秒）
    pub duration_ms: u64,
}
```

**辅助方法**：
- `NetworkEventResult::success(event, duration_ms)` - 创建成功结果
- `NetworkEventResult::failure(event, error, duration_ms)` - 创建失败结果

### 3. NetworkEventProcessor（处理器 Trait）

定义了网络事件的处理接口：

```rust
#[async_trait::async_trait]
pub trait NetworkEventProcessor: Send + Sync {
    /// 处理网络可用事件
    async fn process_network_available(&self) -> Result<(), String>;
    
    /// 处理网络丢失事件
    async fn process_network_lost(&self) -> Result<(), String>;
    
    /// 处理网络类型变化事件
    async fn process_network_type_changed(
        &self,
        is_wifi: bool,
        is_cellular: bool,
    ) -> Result<(), String>;
}
```

### 4. DefaultNetworkEventProcessor（默认实现）

提供了默认的网络事件处理逻辑：

**依赖**：
- `SignalingClient` - 信令客户端（用于重连）
- `WebRtcCoordinator` - WebRTC 协调器（用于 ICE 重启）

**处理逻辑**：

#### 网络可用（Available）
1. 强制断开现有连接（避免"僵尸连接"）
2. 短暂延迟（100ms），让资源清理
3. 重新连接 WebSocket
4. 触发 ICE 重启（如果 WebRTC 已初始化）

#### 网络丢失（Lost）
1. 清理待处理的 ICE 重启尝试
2. 主动断开 WebSocket

#### 网络类型变化（TypeChanged）
1. 作为网络丢失处理
2. 等待网络稳定（500ms）
3. 作为网络恢复处理

### 5. NetworkEventHandle（事件句柄）

轻量级句柄，用于发送网络事件并接收处理结果：

```rust
pub struct NetworkEventHandle {
    event_tx: mpsc::Sender<NetworkEvent>,
    result_rx: Arc<Mutex<mpsc::Receiver<NetworkEventResult>>>,
}
```

**方法**：
- `handle_network_available()` - 处理网络可用事件
- `handle_network_lost()` - 处理网络丢失事件
- `handle_network_type_changed(is_wifi, is_cellular)` - 处理网络类型变化

**特点**：
- 可克隆（Clone）
- 线程安全（Send + Sync）
- 异步等待结果

## 使用指南

### 1. 创建 ActrSystem

```rust
use actr_runtime::lifecycle::ActrSystem;

// 创建系统（不创建网络事件基础设施）
let system = ActrSystem::new(config).await?;
```

### 2. 按需创建 NetworkEventHandle

```rust
// 只在需要网络事件处理时才创建
let network_handle = system.create_network_event_handle();
```

**注意**：
- `create_network_event_handle()` 只能调用一次
- 如果不调用，网络事件功能将不可用（零开销）

### 3. Attach Workload

```rust
// Attach workload（自动获取 channels）
let node = system.attach(workload);
let actr_ref = node.start().await?;
```

**内部机制**：
- `attach()` 会检查是否有网络事件 channels
- 如果存在，传递给 ActrNode
- 如果不存在，网络事件功能不启用

### 4. 处理网络事件（平台层）

#### Android 示例

```kotlin
// NetworkCallback 实现
override fun onAvailable(network: Network) {
    // 调用 FFI
    val result = network_handle.handle_network_available()
    if (result.success) {
        Log.i(TAG, "Network available processed in ${result.durationMs}ms")
    } else {
        Log.e(TAG, "Failed: ${result.error}")
    }
}

override fun onLost(network: Network) {
    val result = network_handle.handle_network_lost()
    // 处理结果...
}

override fun onCapabilitiesChanged(
    network: Network,
    capabilities: NetworkCapabilities
) {
    val isWifi = capabilities.hasTransport(TRANSPORT_WIFI)
    val isCellular = capabilities.hasTransport(TRANSPORT_CELLULAR)
    val result = network_handle.handle_network_type_changed(isWifi, isCellular)
    // 处理结果...
}
```

#### Rust FFI 层

```rust
#[no_mangle]
pub extern "C" fn handle_network_available(
    handle: *const NetworkEventHandle,
) -> NetworkEventResultFFI {
    let handle = unsafe { &*handle };
    
    // 异步调用（在 tokio runtime 中）
    let result = tokio::runtime::Handle::current()
        .block_on(handle.handle_network_available())
        .unwrap_or_else(|e| NetworkEventResult::failure(
            NetworkEvent::Available,
            e,
            0,
        ));
    
    // 转换为 FFI 结构
    result.into()
}
```

### 5. 自定义处理逻辑（可选）

如果需要自定义网络事件处理逻辑：

```rust
use actr_runtime::lifecycle::NetworkEventProcessor;

struct CustomNetworkEventProcessor {
    metrics: Arc<Metrics>,
    // 其他依赖...
}

#[async_trait::async_trait]
impl NetworkEventProcessor for CustomNetworkEventProcessor {
    async fn process_network_available(&self) -> Result<(), String> {
        // 记录指标
        self.metrics.record_network_available();
        
        // 自定义处理逻辑
        // ...
        
        Ok(())
    }
    
    async fn process_network_lost(&self) -> Result<(), String> {
        self.metrics.record_network_lost();
        // ...
        Ok(())
    }
    
    async fn process_network_type_changed(
        &self,
        is_wifi: bool,
        is_cellular: bool,
    ) -> Result<(), String> {
        self.metrics.record_network_type_change(is_wifi, is_cellular);
        // ...
        Ok(())
    }
}
```

**注入自定义处理器**：

目前需要修改 `ActrNode::start()` 中的代码：

```rust
// 替换默认处理器
let event_processor = Arc::new(CustomNetworkEventProcessor::new(
    // 依赖...
));
```

## 内部实现细节

### Channel 创建时机

```rust
impl ActrSystem {
    pub async fn new(config: Config) -> ActorResult<Self> {
        // 不创建任何网络事件 channels
        Ok(Self {
            // ...
            network_event_channels: Mutex::new(None),
        })
    }
    
    pub fn create_network_event_handle(&self) -> NetworkEventHandle {
        // 延迟创建 channels
        let (event_tx, event_rx) = mpsc::channel(100);
        let (result_tx, result_rx) = mpsc::channel(100);
        
        // 存储 channels（供 attach 使用）
        *self.network_event_channels.lock().unwrap() = Some((event_rx, result_tx));
        
        // 返回 handle
        NetworkEventHandle::new(event_tx, result_rx)
    }
}
```

### 后台事件循环

```rust
impl ActrNode {
    async fn network_event_loop(
        mut event_rx: mpsc::Receiver<NetworkEvent>,
        result_tx: mpsc::Sender<NetworkEventResult>,
        event_processor: Arc<dyn NetworkEventProcessor>,
        shutdown_token: CancellationToken,
    ) {
        loop {
            tokio::select! {
                Some(event) = event_rx.recv() => {
                    let start = Instant::now();
                    
                    // 处理事件
                    let result = match &event {
                        NetworkEvent::Available => {
                            event_processor.process_network_available().await
                        }
                        NetworkEvent::Lost => {
                            event_processor.process_network_lost().await
                        }
                        NetworkEvent::TypeChanged { is_wifi, is_cellular } => {
                            event_processor.process_network_type_changed(
                                *is_wifi, *is_cellular
                            ).await
                        }
                    };
                    
                    let duration_ms = start.elapsed().as_millis() as u64;
                    
                    // 构造结果
                    let event_result = match result {
                        Ok(_) => NetworkEventResult::success(event, duration_ms),
                        Err(e) => NetworkEventResult::failure(event, e, duration_ms),
                    };
                    
                    // 发送结果
                    let _ = result_tx.send(event_result).await;
                }
                
                _ = shutdown_token.cancelled() => {
                    break;
                }
            }
        }
    }
}
```

## 性能特性

### 零开销（可选功能）

- 如果不调用 `create_network_event_handle()`，完全零开销
- 不创建 channels，不启动后台任务
- `ActrSystem` 只存储 1 个 `Mutex<Option<...>>` 字段

### Channel 容量

- Event channel: 100
- Result channel: 100

可根据实际需求调整。

### 处理耗时

典型处理耗时（参考）：
- 网络可用：100-500ms（包括重连和 ICE 重启）
- 网络丢失：10-50ms（清理操作）
- 网络类型变化：600-1000ms（断开 + 等待 + 重连）

## 错误处理

### 处理失败

如果事件处理失败，`NetworkEventResult` 会包含：
- `success: false`
- `error: Some(error_message)`
- `duration_ms`: 失败前的耗时

### Channel 错误

- **发送失败**：返回 `Err("Failed to send network event")`
- **接收失败**：返回 `Err("Failed to receive network event result")`

### 恢复策略

平台层可以根据返回的 `NetworkEventResult` 决定是否重试：

```rust
let result = network_handle.handle_network_available().await?;
if !result.success {
    // 记录错误
    log::error!("Network recovery failed: {:?}", result.error);
    
    // 可选：延迟后重试
    tokio::time::sleep(Duration::from_secs(5)).await;
    let retry_result = network_handle.handle_network_available().await?;
    // ...
}
```

## 未来扩展

### Actor Proto Message 支持（TODO）

未来可以支持通过 Actor Message 触发网络事件：

```rust
// 定义 proto message
message NetworkAvailableMessage {}
message NetworkLostMessage {}
message NetworkTypeChangedMessage {
    bool is_wifi = 1;
    bool is_cellular = 2;
}

// Actor 发送消息
actor_ref.call(NetworkAvailableMessage {}).await?;
```

**实现要点**：
- 在 Workload 中定义消息处理方法
- 直接调用 `NetworkEventProcessor`
- 不经过 `NetworkEventHandle`

### 自定义 Processor 注入

未来可以支持在创建时注入自定义 Processor：

```rust
let system = ActrSystem::new(config).await?;
let custom_processor = Arc::new(CustomNetworkEventProcessor::new());
let network_handle = system.create_network_event_handle_with_processor(custom_processor);
```

## 测试建议

### 单元测试

```rust
#[tokio::test]
async fn test_network_event_handle() {
    let (event_tx, mut event_rx) = mpsc::channel(10);
    let (result_tx, result_rx) = mpsc::channel(10);
    
    let handle = NetworkEventHandle::new(event_tx, result_rx);
    
    // 模拟后台处理
    tokio::spawn(async move {
        if let Some(event) = event_rx.recv().await {
            let result = NetworkEventResult::success(event, 100);
            result_tx.send(result).await.unwrap();
        }
    });
    
    // 测试
    let result = handle.handle_network_available().await.unwrap();
    assert!(result.success);
    assert_eq!(result.duration_ms, 100);
}
```

### 集成测试

```rust
#[tokio::test]
async fn test_network_recovery_flow() {
    let config = Config::default();
    let system = ActrSystem::new(config).await.unwrap();
    let network_handle = system.create_network_event_handle();
    
    let workload = TestWorkload::new();
    let node = system.attach(workload);
    let _actr_ref = node.start().await.unwrap();
    
    // 触发网络事件
    let result = network_handle.handle_network_available().await.unwrap();
    assert!(result.success);
    
    // 验证信令已重连
    // 验证 ICE 重启已触发
    // ...
}
```

## 相关文档

- [Network Recovery](./network-recovery.md) - 网络恢复机制详解
- [ICE Restart](./ICE_RESTART_FIX_SUMMARY.md) - ICE 重启实现
- [Architecture Refactor](./network-event-handle-refactor.md) - 架构重构方案

## 总结

网络事件处理模块提供了一套完整、灵活、高性能的解决方案：

✅ **解耦设计** - 通过 Channel 实现调用方与处理方的完全解耦
✅ **异步处理** - 不阻塞平台层  
✅ **结果反馈** - 完整的处理结果信息  
✅ **可扩展** - 支持自定义处理逻辑  
✅ **零开销** - 可选功能，按需启用  
✅ **类型安全** - 编译期保证正确性  

通过这套机制，可以优雅地处理各种网络状态变化，确保系统的健壮性和可靠性。
