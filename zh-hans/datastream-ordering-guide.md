# StreamReliable 顺序处理问题

## 问题背景

`PayloadType::StreamReliable` 类型的消息设计为**可靠有序的流式数据**，但在之前的实现中，消息被 `tokio::spawn` 并发执行，导致消息处理顺序无法保证。

### 问题代码（旧版本）

```rust
pub async fn dispatch(&self, chunk: DataStream, sender_id: ActrId) {
    if let Some(callback) = self.callbacks.get(&chunk.stream_id) {
        let callback = callback.clone();
        tokio::spawn(async move {  // ❌ 问题：所有消息都被并发处理
            if let Err(e) = callback(chunk, sender_id).await {
                tracing::error!("❌ Stream chunk callback error: {:?}", e);
            }
        });
    }
}
```

**问题**：
- Chunk 1 callback 耗时长 → 可能后完成
- Chunk 2 callback 耗时短 → 可能先完成
- 结果：消息乱序 ❌

---

## 当前解决方案：分层设计

我们采用了 **分层控制** 的设计哲学：
- **底层（DataStreamRegistry）**：提供顺序保证，不做并发假设
- **上层（业务代码）**：根据需求决定并发策略

### 架构设计

```
┌─────────────────────────────────────────────────────┐
│ DataStreamRegistry::dispatch (底层，顺序调用)        │
│   ↓ callback                                         │
└─────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────┐
│ 业务 callback (快速返回，非阻塞)                     │
│   → 发送到 per-stream channel                        │
└─────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────┐
│ Stream A Task  │ Stream B Task  │ Stream C Task    │
│ (串行处理)     │ (串行处理)     │ (串行处理)       │
│ msg1 → msg2    │ msg1 → msg2    │ msg1 → msg2      │
│   → msg3       │   → msg3       │   → msg3         │
└─────────────────────────────────────────────────────┘
     ↑ 不同 stream 并发，同一 stream 串行
```

### 底层实现（已完成）

**文件**: `actr/crates/runtime/src/inbound/data_stream_registry.rs`

```rust
pub async fn dispatch(&self, chunk: DataStream, sender_id: ActrId) {
    if let Some(callback) = self.callbacks.get(&chunk.stream_id) {
        let callback = callback.clone();
        // ✅ 直接 await，保证顺序调用
        if let Err(e) = callback(chunk, sender_id).await {
            tracing::error!("❌ Stream chunk callback error: {:?}", e);
        }
    }
}
```

**关键改动**：
- 移除 `tokio::spawn`
- 直接 `await` callback
- 保证按接收顺序调用

---

## 上层实现：Per-Stream Channel 模式

### 方案特点

✅ **不同 stream_id 并发处理**：充分利用多核  
✅ **相同 stream_id 串行处理**：保证消息顺序  
✅ **非阻塞 callback**：快速返回，不阻塞底层 dispatch  

### 使用示例（Rust）

#### 1. 数据结构

```rust
use dashmap::DashMap;
use tokio::sync::mpsc;

pub struct MyStreamService {
    /// Per-stream task spawner: 每个 stream_id 一个 channel
    stream_senders: Arc<DashMap<String, mpsc::UnboundedSender<(DataStream, ActrId)>>>,
}

impl MyStreamService {
    pub fn new() -> Self {
        Self {
            stream_senders: Arc::new(DashMap::new()),
        }
    }
}
```

#### 2. 注册 Stream 处理器

```rust
async fn prepare_stream<C: Context>(
    &self,
    stream_id: String,
    ctx: &C,
) -> ActorResult<()> {
    let stream_senders = self.stream_senders.clone();

    // 为这个 stream_id 创建 channel 和专属处理任务（如果还没有）
    if !stream_senders.contains_key(&stream_id) {
        let (tx, mut rx) = mpsc::unbounded_channel::<(DataStream, ActrId)>();
        stream_senders.insert(stream_id.clone(), tx);

        // 启动专属任务来串行处理这个 stream 的消息
        let stream_id_clone = stream_id.clone();
        tokio::spawn(async move {
            tracing::info!("🚀 Started dedicated task for stream: {}", stream_id_clone);
            
            while let Some((data_stream, sender_id)) = rx.recv().await {
                // 在这里串行处理消息
                let text = String::from_utf8_lossy(&data_stream.payload);
                tracing::info!(
                    "Received message {}: {} (from {})",
                    data_stream.sequence,
                    text,
                    sender_id.to_string_repr()
                );
                
                // 你的业务逻辑...
                process_data(&data_stream).await;
            }
            
            tracing::info!("🛑 Stream task finished: {}", stream_id_clone);
        });
    }

    // 注册 stream callback：快速返回，消息发送到 channel
    ctx.register_stream(
        stream_id.clone(),
        move |data_stream: DataStream, sender_id| {
            let stream_senders = stream_senders.clone();
            let stream_id = stream_id.clone();
            
            Box::pin(async move {
                // 发送到该 stream 的专属 channel（非阻塞）
                if let Some(tx) = stream_senders.get(&stream_id) {
                    if let Err(e) = tx.send((data_stream, sender_id)) {
                        tracing::error!("Failed to send to stream {} channel: {:?}", stream_id, e);
                    }
                } else {
                    tracing::warn!("No sender found for stream: {}", stream_id);
                }
                Ok(())
            })
        },
    )
    .await?;

    Ok(())
}
```

#### 3. 完整示例

```rust
use actr_protocol::{ActorResult, ActrId, DataStream};
use actr_runtime::prelude::*;
use dashmap::DashMap;
use std::sync::Arc;
use tokio::sync::mpsc;

pub struct StreamService {
    stream_senders: Arc<DashMap<String, mpsc::UnboundedSender<(DataStream, ActrId)>>>,
}

impl StreamService {
    pub fn new() -> Self {
        Self {
            stream_senders: Arc::new(DashMap::new()),
        }
    }

    pub async fn register_stream<C: Context>(
        &self,
        stream_id: String,
        ctx: &C,
    ) -> ActorResult<()> {
        let stream_senders = self.stream_senders.clone();

        if !stream_senders.contains_key(&stream_id) {
            let (tx, mut rx) = mpsc::unbounded_channel();
            stream_senders.insert(stream_id.clone(), tx);

            tokio::spawn(async move {
                while let Some((data_stream, sender_id)) = rx.recv().await {
                    // 业务逻辑：串行处理
                    tracing::info!(
                        "Processing msg {} from stream {}",
                        data_stream.sequence,
                        data_stream.stream_id
                    );
                }
            });
        }

        ctx.register_stream(
            stream_id.clone(),
            move |data_stream, sender_id| {
                let stream_senders = stream_senders.clone();
                let stream_id = stream_id.clone();
                Box::pin(async move {
                    if let Some(tx) = stream_senders.get(&stream_id) {
                        let _ = tx.send((data_stream, sender_id));
                    }
                    Ok(())
                })
            },
        )
        .await
    }
}
```

---

## 使用示例（Python）

### 数据结构

```python
import asyncio
from typing import Dict

class StreamService:
    def __init__(self):
        # 每个 stream_id 一个 Queue
        self.stream_queues: Dict[str, asyncio.Queue] = {}
```

### 注册 Stream 处理器

```python
async def prepare_stream(self, stream_id: str, ctx):
    # 为这个 stream_id 创建 Queue 和专属处理任务（如果还没有）
    if stream_id not in self.stream_queues:
        queue = asyncio.Queue()
        self.stream_queues[stream_id] = queue

        # 启动专属任务来串行处理这个 stream 的消息
        async def stream_task():
            logger.info(f"🚀 Started dedicated task for stream: {stream_id}")
            
            while True:
                try:
                    stream, sender_id = await queue.get()
                    text = stream.payload().decode("utf-8")
                    logger.info(
                        f"Received message {stream.sequence()}: {text} "
                        f"(from {sender_id})"
                    )
                    
                    # 你的业务逻辑...
                    await process_data(stream)
                    
                    queue.task_done()
                except asyncio.CancelledError:
                    logger.info(f"🛑 Stream task cancelled: {stream_id}")
                    break

        asyncio.create_task(stream_task())

    # 注册 stream callback：快速返回，消息发送到 Queue
    async def stream_callback(stream, sender_id):
        if stream_id in self.stream_queues:
            await self.stream_queues[stream_id].put((stream, sender_id))

    await ctx.register_stream(stream_id, stream_callback)
```

---

## 完整示例代码

### Rust 示例

参考：`actr-examples/data-stream-peer-concurrent/`
- **Client**: `client/src/stream_client_service.rs`
- **Server**: `server/src/stream_server_service.rs`

### Python 示例

参考：`actr-python/examples/data-stream-peer-concurrent/`
- **Client**: `client/client.py`
- **Server**: `server/server.py`

---

## 核心优势

| 特性 | 效果 |
|------|------|
| **顺序保证** | 同一 stream_id 的消息严格按序处理 ✅ |
| **并发性能** | 不同 stream_id 并发处理 ✅ |
| **非阻塞** | callback 快速返回，不阻塞 dispatch ✅ |
| **灵活性** | 上层自主决定并发策略 ✅ |

---

## 最佳实践

### 1. 选择合适的 Channel 类型

**Rust**:
```rust
// Unbounded: 适合低频消息
let (tx, rx) = mpsc::unbounded_channel();

// Bounded: 适合高频消息，避免内存堆积
let (tx, rx) = mpsc::channel(100);
```

**Python**:
```python
# Unbounded
queue = asyncio.Queue()

# Bounded
queue = asyncio.Queue(maxsize=100)
```

### 2. 优雅关闭

**Rust**:
```rust
impl Drop for StreamService {
    fn drop(&mut self) {
        // Channel 会自动关闭，导致 rx.recv() 返回 None
        self.stream_senders.clear();
    }
}
```

**Python**:
```python
async def cleanup(self):
    """取消所有 stream 任务"""
    for task in self.stream_tasks.values():
        task.cancel()
    await asyncio.gather(*self.stream_tasks.values(), return_exceptions=True)
```

### 3. 监控与告警

```rust
// 定期检查队列深度
if let Some(tx) = stream_senders.get(&stream_id) {
    let capacity = tx.capacity();
    if capacity.is_some() && capacity.unwrap() < 10 {
        tracing::warn!("Stream {} queue nearly full!", stream_id);
    }
}
```

---

## 总结

通过 **分层设计 + per-stream channel** 模式，我们实现了：

1. **底层保证**：顺序调用 callback
2. **上层灵活**：根据 stream_id 分组并发
3. **最佳实践**：不同 stream 并发 + 同一 stream 串行

这种设计模式在流式处理系统中非常常见（如 Kafka Partition、gRPC Streaming），是高性能和正确性的完美平衡点。🎯
