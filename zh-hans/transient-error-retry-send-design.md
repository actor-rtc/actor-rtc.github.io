# Transient 错误重发机制设计(待确认)

> **文档版本**: v2.1  
> **状态**: 设计阶段  
> **最后更新**: 2026-02-05

---

## 目录

1. [问题场景](#1-问题场景)
2. [当前框架的实现空白](#2-当前框架的实现空白)
3. [解决方案](#3-解决方案)
4. [实现方案](#4-实现方案)
5. [实施计划](#5-实施计划)
- [附录 A: 代码实现](#附录-a-代码实现)
- [附录 B: 测试用例](#附录-b-测试用例)

---

## 1. 问题场景

### 1.1 高频故障场景

**场景一：移动网络切换**

用户从 WiFi 区域走到室外，手机切换到 4G 网络，原连接断开。此时正在发送的请求失败，错误抛给调用方，但后台已自动重连成功。

**场景二：断网重连**

网络短暂中断（如信号波动、路由器重启），连接断开。用户发送请求时失败，错误抛给调用方，但后台正在自动重连。

### 1.2 核心问题

```
发送请求 → 连接刚好断开 → 错误抛给调用方 → 后台自动重连成功
                                ↑
                            问题在这里
```

框架自动重连成功了，但请求已经失败，调用方需要自己重试。

---

## 2. 当前框架的实现空白

### 2.1 处理现状

| 机制 | 状态 | 说明 |
|------|------|------|
| **网络重连** | ✅ 已实现 | WirePool 自动重连 |
| **数据重发** | ❌ 未实现 | 错误直接抛给上层 |

**问题**：自动重连 + 不自动重发 = 处理不对称

### 2.2 受影响的方法

| 方法 | 需要自动重发 |
|------|-------------|
| `send_request()` | ✅ 需要 |
| `send_message()` | ✅ 需要 |
| `send_data_stream(Reliable)` | ✅ 需要 |
| `send_data_stream(LatencyFirst)` | ❌ 不需要 |
| `send_media_sample()` | ❌ 不需要 |

---

## 3. 解决方案

**在 Outbound 层加入重试逻辑**，对上层透明。

```
改进前：调用方 → OutGate → 失败 → 返回错误 → 调用方自己重试
改进后：调用方 → OutGate → 失败 → OutGate 自动重试 → 成功/最终失败
```

---

## 4. 实现方案

### 4.1 需要解决的问题

| 问题 | 解决方案 |
|------|---------|
| **Dest 隔离** | 每个请求独立的重试状态，不共享 |
| **超时控制** | 使用 deadline 模式，重试不超过 RPC timeout |
| **幂等性** | 发送端：相同 request_id；接收端：去重缓存 |
| **资源保护** | 指数退避 + 抖动 |

### 4.2 Dest 隔离

每个请求有独立的 Backoff 实例，重试不影响其他请求。

```
请求 1 (→ Dest A): 重试中...
请求 2 (→ Dest B): 正常发送 ✅  ← 不受影响
请求 3 (→ Dest A): 正常发送 ✅  ← 不受影响
```

### 4.3 超时控制

所有重试必须在 deadline 前完成。

```
timeout_ms = 30000
deadline = now() + 30s

重试 1: 失败，等待 500ms  (剩余 29.5s)
重试 2: 失败，等待 1s    (剩余 28.5s)
重试 3: 成功 ✅
   或: deadline 到达 → 返回超时错误
```

### 4.4 幂等性

所有需要重发的方法都需要幂等保证：

| 方法 | 去重键 | 幂等策略 |
|------|--------|---------|
| `send_request()` | request_id | 缓存响应，重复请求返回缓存 |
| `send_message()` | request_id | 记录 ID，重复消息忽略 |
| `send_data_stream()` | `chunk_id` (metadata) | 记录 ID，重复数据块忽略 |

**发送端**：重试时自动注入/保持相同的标识符

**接收端**：基于标识符去重，去重后移除内部标识（对应用层透明）

```
RPC 请求：
  收到 request_id="abc" → 检查缓存
    ├─ 未见过 → 处理请求，缓存结果
    └─ 见过   → 直接返回缓存的响应

单向消息：
  收到 request_id="def" → 检查去重表
    ├─ 未见过 → 处理消息，记录 ID
    └─ 见过   → 忽略（已处理过）

数据流：
  收到 metadata.chunk_id="uuid-xxx" → 检查去重表
    ├─ 未见过 → 记录 ID，移除 chunk_id，分发数据
    └─ 见过   → 忽略（重复）
```

#### 4.4.1 去重存储接口

去重存储抽象为 `DedupStore` 接口，支持灵活替换：

```rust
#[async_trait]
pub trait DedupStore: Send + Sync {
    async fn contains(&self, key: &str) -> bool;
    async fn mark(&self, key: &str);
    async fn get_response(&self, key: &str) -> Option<Bytes>;
    async fn put_response(&self, key: &str, value: Bytes);
    async fn try_mark_inflight(&self, key: &str) -> bool;
    async fn clear_inflight(&self, key: &str);
}
```

| 方法 | 说明 |
|------|------|
| `contains` | 检查 key 是否已处理（消息/数据流去重） |
| `mark` | 标记 key 已处理 |
| `get_response` | 获取缓存的 RPC 响应 |
| `put_response` | 缓存 RPC 响应 |
| `try_mark_inflight` | 标记请求"处理中"，返回 true 表示成功，false 表示已有相同请求在处理 |
| `clear_inflight` | 清除"处理中"标记（handler 完成或失败时调用） |

| 实现 | 适用场景 | 说明 |
|------|---------|------|
| `LruDedupStore`（默认） | 小并发量 | 快速去重，高 QPS 时可能漏判 |
| `RedisDedupStore` | 分布式/高并发 | 持久化，支持跨进程共享 |
| 业务层 DB 约束 | 金融/交易场景 | 最可靠，由业务层自行保证 |

### 4.5 重试参数（内置默认值）

| 参数 | 默认值 | 说明 |
|------|--------|------|
| max_attempts | 3 | 最大重试次数 |
| initial_delay | 1000ms | 初始退避时间 |
| max_delay | 5000ms | 最大退避时间 |
| multiplier | 2.0 | 退避倍数 |
| jitter_factor | 0.2 | 抖动范围 ±20% |

---

## 5. 实施计划

### Phase 1: 发送端重试 ⭐ 优先

**目标**：所有可靠发送方法支持自动重试

**任务**：
1. 实现通用 `retry_with_backoff()` 函数
2. 实现带抖动的 `ExponentialBackoff`
3. `send_request()` / `send_message()` / `send_data_stream(Reliable)` 添加重试
4. 添加 `PayloadType::requires_reliable_send()`

**产出**：可靠发送在 Transient 错误时自动重试

### Phase 2: 接收端幂等处理

**目标**：保证重复消息不会被重复处理

**任务**：
1. 定义 `DedupStore` 去重接口
2. 实现默认的 `LruDedupStore`（LRU + TTL）
3. 在 `WebRtcGate` 中集成去重（支持注入自定义实现）
4. 在 `DataStreamRegistry` 中集成去重（基于 `chunk_id`，去重后移除）

**产出**：
- 接收端能正确处理重复的请求，对应用层透明
- 支持按需替换去重存储（Redis/DB）以应对高并发场景

---

## 附录 A: 代码实现

### A.1 Phase 1 代码

#### A.1.1 重试常量

```rust
// outbound/retry.rs

use std::time::Duration;

// 重试默认配置（内置，不暴露）
const MAX_ATTEMPTS: u32 = 3;
const INITIAL_DELAY: Duration = Duration::from_millis(1000);
const MAX_DELAY: Duration = Duration::from_secs(5);
const MULTIPLIER: f64 = 2.0;
const JITTER_FACTOR: f64 = 0.2;
const MESSAGE_DEFAULT_TIMEOUT: Duration = Duration::from_secs(30);
```

#### A.1.2 带抖动的指数退避

```rust
// outbound/backoff.rs

use std::time::Duration;
use rand::Rng;

pub struct ExponentialBackoff {
    current: Duration,
    max: Duration,
    multiplier: f64,
    jitter_factor: f64,
    retries: u32,
    max_retries: u32,
}

impl ExponentialBackoff {
    pub fn new() -> Self {
        Self {
            current: INITIAL_DELAY,
            max: MAX_DELAY,
            multiplier: MULTIPLIER,
            jitter_factor: JITTER_FACTOR,
            retries: 0,
            max_retries: MAX_ATTEMPTS,
        }
    }
    
    pub fn next_delay(&mut self) -> Option<Duration> {
        if self.retries >= self.max_retries {
            return None;
        }
        
        // 添加抖动
        let base_ms = self.current.as_millis() as f64;
        let jitter = rand::thread_rng()
            .gen_range(-self.jitter_factor..self.jitter_factor);
        let delay_ms = (base_ms * (1.0 + jitter)).max(0.0) as u64;
        let delay = Duration::from_millis(delay_ms);
        
        // 计算下次延迟
        let next_ms = (base_ms * self.multiplier) as u64;
        self.current = Duration::from_millis(next_ms).min(self.max);
        
        self.retries += 1;
        Some(delay)
    }
    
    pub fn retry_count(&self) -> u32 {
        self.retries
    }
}
```

#### A.1.3 通用重试函数

```rust
// outbound/retry.rs

use std::future::Future;
use std::time::Duration;
use tokio::time;

/// 带退避的重试执行器（严格遵守 deadline）
///
/// - 每次 attempt 都会被 `remaining` 约束，避免单次 operation 卡住导致总耗时超过 deadline
/// - deadline 到期时返回最后一次错误，或 timeout_error
pub async fn retry_with_backoff<T, E, F, Fut>(
    deadline: Instant,
    mut operation: F,
    is_retryable: fn(&E) -> bool,
    timeout_error: fn() -> E,  // 超时时返回的错误
) -> Result<T, E>
where
    F: FnMut(Duration) -> Fut,
    Fut: Future<Output = Result<T, E>>,
    E: std::fmt::Debug,
{
    let mut backoff = ExponentialBackoff::new();
    let mut last_error: Option<E> = None;

    loop {
        let remaining = deadline.saturating_duration_since(Instant::now());
        if remaining.is_zero() {
            break;
        }

        // 关键：给每次 attempt 套一层 remaining timeout，防止单次 operation 超时拖死 deadline
        let attempt = time::timeout(remaining, operation(remaining)).await;

        match attempt {
            Ok(Ok(result)) => return Ok(result),

            Ok(Err(e)) if is_retryable(&e) => {
                last_error = Some(e);

                let remaining = deadline.saturating_duration_since(Instant::now());
                if remaining.is_zero() {
                    break;
                }

                if let Some(delay) = backoff.next_delay() {
                    time::sleep(delay.min(remaining)).await;
                } else {
                    break;
                }
            }

            Ok(Err(e)) => return Err(e),

            // attempt 被 deadline 截断
            Err(_elapsed) => break,
        }
    }

    Err(last_error.unwrap_or_else(timeout_error))
}
```

#### A.1.4 OutprocOutGate 重试实现

```rust
// outbound/outproc_out_gate.rs

use std::pin::Pin;
use tokio::select;
use tokio::time::{self, Duration, Instant};

impl OutprocOutGate {
    pub async fn send_request(&self, target: &ActrId, envelope: RpcEnvelope) -> ActorResult<Bytes> {
        let deadline = Instant::now() + Duration::from_millis(envelope.timeout_ms as u64);

        // 关键：同一个 request_id 只注册一次 waiter，避免重试时覆盖 pending_requests 导致晚到响应丢失
        let (tx, mut rx) = oneshot::channel();
        self.pending_requests
            .write()
            .await
            .insert(envelope.request_id.clone(), tx);

        // 如果最终失败/退出，一定清理 pending
        let request_id = envelope.request_id.clone();

        let mut backoff = ExponentialBackoff::new();
        let mut last_error: Option<ProtocolError> = None;

        // 发送 + 等待响应：一个 loop 内完成，rx 不会被 timeout() 丢掉
        loop {
            let remaining = deadline.saturating_duration_since(Instant::now());
            if remaining.is_zero() {
                self.pending_requests.write().await.remove(&request_id);
                return Err(last_error.unwrap_or(ProtocolError::Timeout).into());
            }

            // 1) 单次发送（transport-level），这里的超时由 outer remaining 约束
            if let Err(e) = self.do_send(target, PayloadType::RpcReliable, &envelope).await {
                if Self::is_retryable_transport(&e) {
                    last_error = Some(e);
                } else {
                    self.pending_requests.write().await.remove(&request_id);
                    return Err(e.into());
                }
            }

            // 2) 等待响应（不使用 tokio::time::timeout，以免丢掉 rx）
            //    如果超时则进入 backoff 后重发；如果收到响应则成功返回
            let wait = time::sleep(remaining);
            tokio::pin!(wait);

            select! {
                biased;

                res = &mut rx => {
                    self.pending_requests.write().await.remove(&request_id);
                    return res
                        .map_err(|_| ProtocolError::TransportError("channel closed".into()))
                        .map(Into::into);
                }

                _ = &mut wait => {
                    // response-level timeout：默认不当作 retryable，除非你确认“连接刚断/正在重连”
                    last_error = Some(ProtocolError::Timeout);

                    let remaining = deadline.saturating_duration_since(Instant::now());
                    if remaining.is_zero() {
                        self.pending_requests.write().await.remove(&request_id);
                        return Err(ProtocolError::Timeout.into());
                    }

                    if let Some(delay) = backoff.next_delay() {
                        time::sleep(delay.min(remaining)).await;
                        continue;
                    } else {
                        self.pending_requests.write().await.remove(&request_id);
                        return Err(ProtocolError::Timeout.into());
                    }
                }
            }
        }
    }

    pub async fn send_message(&self, target: &ActrId, envelope: RpcEnvelope) -> ActorResult<()> {
        // 这里保留原逻辑，但将 Timeout 排除在 retryable 之外
        let deadline = Instant::now() + MESSAGE_DEFAULT_TIMEOUT;

        retry_with_backoff(
            deadline,
            |_remaining| self.send_message_once(target, envelope.clone()),
            Self::is_retryable_transport,
            || ProtocolError::Timeout,
        )
        .await
    }

    pub async fn send_data_stream(
        &self,
        target: &ActrId,
        payload_type: PayloadType,
        mut data: DataStream,
    ) -> ActorResult<()> {
        // LatencyFirst 不重试
        if !payload_type.requires_reliable_send() {
            return self.send_data_stream_once(target, payload_type, &data).await;
        }

        // 注入 chunk_id（仅首次，重试保持相同 ID）
        if !data.metadata.iter().any(|m| m.key == "chunk_id") {
            data.metadata.push(MetadataEntry {
                key: "chunk_id".into(),
                value: Uuid::new_v4().to_string(),
            });
        }

        let deadline = Instant::now() + MESSAGE_DEFAULT_TIMEOUT;

        retry_with_backoff(
            deadline,
            |_remaining| self.send_data_stream_once(target, payload_type, &data),
            Self::is_retryable_transport,
            || ProtocolError::Timeout,
        )
        .await
    }

    // --- 单次发送方法 ---

    async fn send_message_once(&self, target: &ActrId, envelope: RpcEnvelope) -> ActorResult<()> {
        self.do_send(target, PayloadType::RpcReliable, &envelope).await
    }

    async fn send_data_stream_once(
        &self,
        target: &ActrId,
        payload_type: PayloadType,
        data: &DataStream,
    ) -> ActorResult<()> {
        let dest = Self::to_dest(target);
        let bytes = data.encode_to_vec();
        self.transport_manager
            .send(&dest, payload_type, &bytes)
            .await
            .map_err(|e| ProtocolError::TransportError(e.to_string()))
    }

    async fn do_send(&self, target: &ActrId, payload_type: PayloadType, envelope: &RpcEnvelope) -> ActorResult<()> {
        let data = Self::serialize_envelope(envelope);
        let dest = Self::to_dest(target);
        self.transport_manager
            .send(&dest, payload_type, &data)
            .await
            .map_err(|e| ProtocolError::TransportError(e.to_string()))
    }

    // 只把“明确的 transport transient”当作 retryable（避免 timeout 放大）
    fn is_retryable_transport(e: &ProtocolError) -> bool {
        matches!(e, ProtocolError::TransportError(_))
    }
}
```

#### A.1.5 PayloadType 扩展

```rust
impl PayloadType {
    pub fn requires_reliable_send(&self) -> bool {
        matches!(self, Self::RpcReliable | Self::RpcSignal | Self::StreamReliable)
    }
}
```

---

### A.2 Phase 2 代码（接收端去重）

#### A.2.1 去重接口与默认实现

```rust
// utils/dedup.rs

use async_trait::async_trait;
use bytes::Bytes;
use lru::LruCache;
use std::num::NonZeroUsize;
use std::sync::Arc;
use std::time::{Duration, Instant};
use tokio::sync::Mutex;

// ============== 去重接口 ==============

/// 去重存储接口
/// 
/// 框架提供默认的 LRU+TTL 实现，适合小并发量场景。
/// 高并发或需要强幂等保证时，可注入 Redis/DB 实现。
#[async_trait]
pub trait DedupStore: Send + Sync {
    /// 检查 key 是否存在（未过期）
    async fn contains(&self, key: &str) -> bool;
    
    /// 标记 key 已处理
    async fn mark(&self, key: &str);
    
    /// 获取缓存的响应（用于 RPC）
    async fn get_response(&self, key: &str) -> Option<Bytes>;
    
    /// 缓存响应
    async fn put_response(&self, key: &str, value: Bytes);

    /// 尝试标记请求为"处理中"，返回 true 表示成功（首次），false 表示已有其他请求在处理
    /// 注意：如果 handler 崩溃/取消，需调用 clear_inflight 清理，否则后续请求会被误拒
    async fn try_mark_inflight(&self, key: &str) -> bool;
    
    /// 清除"处理中"标记（handler 完成或失败时调用）
    async fn clear_inflight(&self, key: &str);
}

// ============== 默认实现：LRU + TTL ==============

/// 内存去重存储（LRU + TTL）
/// 
/// 适用场景：
/// - 小并发量（< 1k QPS）

pub struct LruDedupStore {
    /// 消息/数据流 ID 去重
    ids: Mutex<LruCache<String, Instant>>,
    /// RPC 响应缓存
    responses: Mutex<LruCache<String, (Bytes, Instant)>>,
    /// 正在处理中的请求
    inflight: Mutex<HashSet<String>>,
    ttl: Duration,
}

impl LruDedupStore {
    pub fn new(capacity: usize, ttl: Duration) -> Self {
        Self {
            ids: Mutex::new(LruCache::new(NonZeroUsize::new(capacity).unwrap())),
            responses: Mutex::new(LruCache::new(NonZeroUsize::new(capacity).unwrap())),
            inflight: Mutex::new(HashSet::new()),
            ttl,
        }
    }
}

impl Default for LruDedupStore {
    fn default() -> Self {
        Self::new(10_000, Duration::from_secs(300))
    }
}

#[async_trait]
impl DedupStore for LruDedupStore {
    async fn contains(&self, key: &str) -> bool {
        let cache = self.ids.lock().await;
        cache.peek(key)
            .map(|ts| ts.elapsed() < self.ttl)
            .unwrap_or(false)
    }

    async fn mark(&self, key: &str) {
        self.ids.lock().await.put(key.to_string(), Instant::now());
    }

    async fn get_response(&self, key: &str) -> Option<Bytes> {
        let cache = self.responses.lock().await;
        cache.peek(key)
            .filter(|(_, ts)| ts.elapsed() < self.ttl)
            .map(|(v, _)| v.clone())
    }

    async fn put_response(&self, key: &str, value: Bytes) {
        self.responses.lock().await.put(key.to_string(), (value, Instant::now()));
    }

    async fn try_mark_inflight(&self, key: &str) -> bool {
        self.inflight.lock().await.insert(key.to_string())
    }

    async fn clear_inflight(&self, key: &str) {
        self.inflight.lock().await.remove(key);
    }
}
```

#### A.2.2 WebRtcGate 集成

```rust
// wire/webrtc/gate.rs

pub struct WebRtcGate {
    // ... 现有字段 ...
    dedup: Arc<dyn DedupStore>,  // 去重存储（可替换）
}

impl WebRtcGate {
    /// 使用默认 LRU 去重
    pub fn new(/* ... */) -> Self {
        Self::with_dedup(Arc::new(LruDedupStore::default()))
    }

    /// 使用自定义去重存储
    pub fn with_dedup(dedup: Arc<dyn DedupStore>) -> Self {
        Self {
            // ... 现有字段初始化 ...
            dedup,
        }
    }

    async fn handle_envelope(&self, envelope: RpcEnvelope, from: &ActrId, ...) {
        let id = &envelope.request_id;

        // Response → 原有逻辑
        if self.pending_requests.read().await.contains_key(id) {
            // ...
            return;
        }

        // Request → 查响应缓存，再查 inflight
        if envelope.is_request {
            if let Some(cached) = self.dedup.get_response(id).await {
                self.do_send_response(from, id, cached).await;
                return;
            }
            // 检查是否有其他请求正在处理
            if !self.dedup.try_mark_inflight(id).await {
                return; // 已有相同请求在处理中，忽略
            }
        } 
        // Message → 查去重
        else if self.dedup.contains(id).await {
            return;
        } else {
            self.dedup.mark(id).await;
        }

        mailbox.enqueue(...).await;
    }

    /// 发送 RPC 响应（自动缓存，确保幂等）
    pub async fn send_response(&self, to: &ActrId, request_id: &str, response: Bytes) {
        // 1. 清除 inflight 标记
        self.dedup.clear_inflight(request_id).await;
        
        // 2. 缓存响应（即使发送失败，重试时也能返回缓存）
        self.dedup.put_response(request_id, response.clone()).await;
        
        // 3. 发送
        self.do_send_response(to, request_id, response).await;
    }

    async fn do_send_response(&self, to: &ActrId, request_id: &str, response: Bytes) {
        // 实际发送逻辑...
    }
}
```

#### A.2.3 DataStreamRegistry 集成

```rust
// inbound/data_stream_registry.rs

pub struct DataStreamRegistry {
    callbacks: RwLock<HashMap<String, DataStreamCallback>>,
    dedup: Arc<dyn DedupStore>,  // 去重存储（可替换）
}

impl DataStreamRegistry {
    /// 使用默认 LRU 去重
    pub fn new() -> Self {
        Self::with_dedup(Arc::new(LruDedupStore::default()))
    }

    /// 使用自定义去重存储
    pub fn with_dedup(dedup: Arc<dyn DedupStore>) -> Self {
        Self {
            callbacks: RwLock::new(HashMap::new()),
            dedup,
        }
    }

    pub async fn dispatch(&self, mut stream: DataStream) {
        // 提取并移除 chunk_id
        let chunk_id = Self::extract_chunk_id(&mut stream.metadata);
        
        // 去重检查
        if let Some(id) = &chunk_id {
            if self.dedup.contains(id).await {
                return; // 重复，忽略
            }
            self.dedup.mark(id).await;
        }

        // 分发（metadata 中已无 chunk_id，对应用层透明）
        if let Some(cb) = self.callbacks.read().await.get(&stream.stream_id) {
            cb(stream).await;
        }
    }

    /// 提取并移除 chunk_id
    fn extract_chunk_id(metadata: &mut Vec<MetadataEntry>) -> Option<String> {
        if let Some(pos) = metadata.iter().position(|m| m.key == "chunk_id") {
            Some(metadata.remove(pos).value)
        } else {
            None
        }
    }
}
```

#### A.2.4 使用示例

```rust
// 场景 1：小并发量 - 使用默认 LRU
let registry = DataStreamRegistry::new();
let gate = WebRtcGate::new(/* ... */);

// 场景 2：高并发/分布式 - 注入 Redis 实现
let redis_dedup = Arc::new(RedisDedupStore::new("redis://localhost", Duration::from_secs(300)));
let registry = DataStreamRegistry::with_dedup(redis_dedup.clone());
let gate = WebRtcGate::with_dedup(redis_dedup);

```

---

## 附录 B: 测试用例

```rust
#[tokio::test]
async fn test_retry_on_transient_error() {
    // 模拟第一次失败，第二次成功
    // 验证：返回成功，重试次数为 1
}

#[tokio::test]
async fn test_no_retry_for_latency_first() {
    // 发送 LatencyFirst 消息失败
    // 验证：不重试，直接返回错误
}

#[tokio::test]
async fn test_retry_respects_deadline() {
    // 设置 1s 超时，模拟持续失败
    // 验证：在 deadline 内停止重试
}

#[tokio::test]
async fn test_request_idempotency() {
    // 发送相同 request_id 的 RPC 请求两次
    // 验证：第二次返回缓存的响应，不重复处理
}

#[tokio::test]
async fn test_message_deduplication() {
    // 发送相同 request_id 的单向消息两次
    // 验证：第二次被忽略，不重复处理
}

#[tokio::test]
async fn test_data_stream_deduplication() {
    // 发送带 chunk_id 的数据流两次
    // 验证：第二次被忽略，且应用层看不到 chunk_id
}

#[tokio::test]
async fn test_dest_isolation() {
    // 并发发送到 Dest A 和 Dest B
    // 验证：Dest A 的重试不影响 Dest B
}

#[tokio::test]
async fn test_custom_dedup_store() {
    // 注入自定义 DedupStore 实现
    // 验证：去重逻辑使用自定义实现
}

#[tokio::test]
async fn test_lru_eviction_behavior() {
    // 创建小容量 LruDedupStore
    // 填满后继续插入，验证旧条目被驱逐
    // 验证：被驱逐的 ID 重新提交会被当作新请求
}
```
