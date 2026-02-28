# DLQ 读取与管理 API 缺失

> **日期**: 2026-02-27  
> **影响范围**: `actr-runtime`, `actr-runtime-mailbox`

---

## 1. 问题描述

框架已实现 DLQ 的写入路径和存储层读取能力，但**没有将读取能力暴露给使用者**。

使用者目前无法通过框架 API：
- 查询 DLQ 中有哪些毒消息
- 获取毒消息的详细信息（raw_bytes、错误原因等）
- 将修复后的消息重新投递（Redrive）
- 获取 DLQ 统计用于监控告警

唯一的读取方式是直接打开 SQLite 文件 `{mailbox_path}.dlq` 手动查询。

### 1.1 文档与实现的差距

| 文档描述                               | 实际代码                   |
| -------------------------------------- | -------------------------- |
| `mailbox.redrive_from_dlq(message_id)` | ❌ 方法不存在               |
| `dlq_message_count` 指标               | ❌ 未暴露                   |
| "定期检查 DLQ，修复毒消息"             | ❌ 没有提供检查入口         |
| "应用可按需实现 dead-letter 队列"      | ❌ 只是一句原则，无示例代码 |

### 1.2 已实现的基础

- ✅ `DeadLetterQueue` trait：`enqueue`, `query`, `get`, `delete`, `record_redrive_attempt`, `stats`
- ✅ `SqliteDeadLetterQueue`：完整 SQLite 实现，含单元测试
- ✅ `ActrNode.dlq`：运行时持有 `Arc<dyn DeadLetterQueue>`
- ✅ 写入路径：`actr_node.rs` `handle_incoming` 中 `RpcEnvelope::decode()` 失败时自动 enqueue

**结论：存储层已就绪，缺的只是应用层到存储层的桥梁。**

---

## 2. 修复方案（待确定）

### 2.1 在 ActrRef 上新增 DLQ 管理方法



```rust
impl<W: Workload> ActrRef<W> {

    pub async fn dlq_stats(&self) -> ActorResult<DlqStats> {
        self.node.dlq.stats().await.map_err(into_transport_error)
    }

    pub async fn dlq_query(&self, query: DlqQuery) -> ActorResult<Vec<DlqRecord>> {
        self.node.dlq.query(query).await.map_err(into_transport_error)
    }

    pub async fn dlq_get(&self, id: Uuid) -> ActorResult<Option<DlqRecord>> {
        self.node.dlq.get(id).await.map_err(into_transport_error)
    }

    pub async fn dlq_delete(&self, id: Uuid) -> ActorResult<()> {
        self.node.dlq.delete(id).await.map_err(into_transport_error)
    }

    pub async fn dlq_redrive(&self, id: Uuid) -> ActorResult<()> {
        // 1. 读取 DLQ 记录
        let record = self.node.dlq.get(id).await?
            .ok_or(NotFound)?;

        // 2. 原样放回 Mailbox
        let from = record.from.unwrap_or_default();
        self.node.mailbox.enqueue(from, record.raw_bytes, Normal).await?;

        // 3. 从 DLQ 删除
        self.node.dlq.delete(id).await?;

        Ok(())
    }
}
```

### 2.2 Redrive 逻辑


```
DLQ.get(id) → Mailbox.enqueue(from, raw_bytes) → DLQ.delete(id)
```

- **不修改消息内容**（业界惯例：AWS SQS、RabbitMQ、Azure Service Bus 都是原样搬回）
- 如果用户需要修改内容后再重投递，自行组合 `dlq_get` + 业务处理 + `dlq_delete`

### 2.3 错误处理

| 场景                 | 处理                                    |
| -------------------- | --------------------------------------- |
| DLQ 记录不存在       | 返回 NotFound                           |
| Mailbox enqueue 失败 | 保留 DLQ 记录，返回错误                 |
| 重投递后仍解码失败   | 消息再次进入 DLQ（redrive_attempts +1） |
| 并发 redrive 同一条  | 第二次返回 NotFound（安全）             |

---

## 3. 使用示例

```rust
let actr = node.start().await?;

// 业务调用（原有能力，不变）
actr.echo(EchoRequest { message: "hello".into() }).await?;

// DLQ 管理（新增能力）
let stats = actr.dlq_stats().await?;
if stats.total_messages > 0 {
    println!("⚠️ DLQ 中有 {} 条毒消息", stats.total_messages);

    let records = actr.dlq_query(DlqQuery {
        error_category: Some("protobuf_decode".into()),
        limit: Some(10),
        ..Default::default()
    }).await?;

    for r in &records {
        println!("[{}] {} (已重试{}次)", r.id, r.error_message, r.redrive_attempts);
    }

    // 修复 schema 兼容性问题后，原样重投递
    for r in &records {
        match actr.dlq_redrive(r.id).await {
            Ok(()) => println!("✅ {}", r.id),
            Err(e) => println!("❌ {}: {:?}", r.id, e),
        }
    }
}

actr.wait_for_ctrl_c_and_shutdown().await?;
```

---

## 4. 改动范围

| 文件                                   | 改动                          | 复杂度 |
| -------------------------------------- | ----------------------------- | :----: |
| `runtime/src/actr_ref.rs`              | 新增 5 个委托方法             | **低** |
| `runtime/src/lifecycle/actr_node.rs`   | 新增 `dlq_redrive()` 协调逻辑 | **中** |
| `runtime-mailbox/src/dlq.rs`           | 不改                          |   -    |
| `runtime-mailbox/src/sqlite_dlq.rs`    | 不改                          |   -    |
| `runtime/src/lifecycle/actr_system.rs` | 不改                          |   -    |

> 存储层 Trait 和实现完全满足需求，无需修改。

---

## 5. 文档同步

实现完成后需同步更新以下文档：

- [ ] `3.11-production-readiness.zh.md` — 补充 DLQ 读取 API 说明，移除 `mailbox.redrive_from_dlq()` 伪代码
- [ ] `3.6-the-persistent-mailbox.zh.md` — 补充"应用可按需实现 dead-letter 队列"的具体使用方式
- [ ] `4.6-actr-runtime.zh.md` — 补充 `ActrRef` 新增方法文档

---
