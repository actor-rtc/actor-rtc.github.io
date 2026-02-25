# ICE Restart 双向触发方案设计

## 1. 问题背景

### 1.1 问题成因

当前实现中，为了避免双方同时创建 Offer 导致的 Glare 冲突，以及简化 restart 生命周期（inflight 标记、重试计数、退避计时）的状态管理，ICE restart 只允许 **Offerer** 端发起，Answerer 端直接跳过。如果 Answerer 端具有平台级网络感知能力（如 iOS/Android能精确感知网络恢复、WiFi/4G 切换等事件），这些信息无法被利用来加速 ICE 恢复：

```rust
// coordinator.rs
if !state.is_offerer {
    tracing::warn!(
        "🚫 Skip ICE restart to serial={}: we are not the offerer",
        target.serial_number
    );
    return Ok(());
}
```

这导致 **Answerer 端网络恢复后无法主动触发 ICE restart**，只能被动等待 Offerer 端的退避重试。

### 1.2 实际影响 — Answerer 网络切换的恢复延迟问题

典型场景：**Answerer 端切换网络（WiFi → 4G），Offerer 端网络正常**。

```text
时序问题：

T0:  Answerer 网络切换 → iOS/Android 平台触发 NetworkTypeChanged
T1:  Answerer process_network_type_changed() → 重连信令 → retry_failed_connections()
     └─ restart_ice() → 因为 !is_offerer → 直接跳过 ❌

T0:  Offerer 检测到 PeerConnection → Disconnected
T2:  install_restart_handler 触发 restart_ice() → do_ice_restart_inner 开始
T3:  ICE restart offer 发送，但 Answerer 信令可能还没恢复 → 超时
T4:  指数退避 sleep(5s) → 重试
T5:  sleep(10s) → 重试
...
     可能等到 30-60s 才成功
```

**核心矛盾**：如果 Answerer 端拥有**平台级别的网络感知能力**（精确知道网络何时恢复），但这个信息无法传递给 Offerer，导致 Offerer 只能在指数退避循环中"盲等"。

---

## 2. 方案 A：信令通知 + 唤醒退避（推荐）

### 2.1 核心思路

Answerer 在**网络恢复时通知 Offerer**：「我回来了，你可以立刻重试了」。Offerer 收到通知后，通过 `tokio::sync::Notify` **打断当前的退避等待**，立刻进入下一次 ICE restart 尝试。

```text
Answerer                         Signaling                       Offerer
  |                                |                               |
  |-- 网络切换 (WiFi→4G)          |                               |
  |                                |                               |
  |                                |              [检测到 Disconnected]
  |                                |              [restart_ice() 开始]
  |                                |              [ICE offer 发送失败]
  |                                |              [进入退避 sleep(5s)...]
  |                                |                               |
  |-- process_network_available()  |                               |
  |-- 重连信令 WebSocket ✅        |                               |
  |-- retry_failed_connections()   |                               |
  |   └─ restart_ice()             |                               |
  |      └─ 发送 IceRestartRequest |                               |
  |       ─────────────────────────►                               |
  |                                |──IceRestartRequest──────────► |
  |                                |                    [收到通知!] |
  |                                |           restart_wake.notify_one()
  |                                |                 [退避被打断!]  |
  |                                |              [立刻创建新 offer]|
  |                                |◄──IceRestartOffer──────────── |
  |   ◄────────────────────────────|                               |
  |-- handle_ice_restart_offer()   |                               |
  |-- 创建 Answer                  |                               |
  |   ─────Answer─────────────────►|                               |
  |                                |──────Answer──────────────────►|
  |                                |                               |
  |◄═══════════ ICE restart 成功,  DataChannel 保留 ══════════════►|
```

### 2.2 与现有 NetworkEvent 机制的协同

```text
┌─────────────────────────────────────────────────────────────────────┐
│ Answerer 端                                                         │
│                                                                     │
│  iOS/Android 平台层                                                  │
│       │                                                             │
│       ▼                                                             │
│  NetworkEventHandle.handle_network_available()                      │
│       │                                                             │
│       ▼                                                             │
│  DefaultNetworkEventProcessor.process_network_available()           │
│       │                                                             │
│       ├─ Step 1: 重连信令 WebSocket                                  │
│       │                                                             │
│       └─ Step 2: retry_failed_connections()                         │
│              │                                                      │
│              ▼                                                      │
│         restart_ice() ← 修改：Answerer 走新分支                      │
│              │                                                      │
│              ▼                                                      │
│         request_ice_restart_from_peer()                             │
│              │                                                      │
│              ▼                                                      │
│         send_actr_relay(IceRestartRequest) ────────► Offerer        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ Offerer 端                                                          │
│                                                                     │
│  do_ice_restart_inner (退避循环中)                                    │
│       │                                                             │
│       ▼                                                             │
│   tokio::select! {                                                  │
│       _ = sleep(delay) => { /* 正常退避 */ }                         │
│       _ = restart_wake.notified() => { /* ⚡ 被唤醒，立刻重试 */ }    │
│   }                                                                 │
│                                                                     │
│  handle_ice_restart_request()                                       │
│       │                                                             │
│       ├─ 已有 restart 进行中? → restart_wake.notify_one() (唤醒退避)  │
│       │                                                             │
│       └─ 没有 restart 进行中? → restart_ice(from) (发起新 restart)    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 实现步骤

#### 步骤 1：添加 Protocol 定义

```protobuf
// actr-protocol/proto/signaling.proto

message IceRestartRequest {
  required ActrId from = 1;     // 请求方（answerer）
  required ActrId to = 2;       // 目标方（offerer）
  optional string reason = 3;   // 请求原因（可选，如 "network_recovered"）
}

message ActrRelay {
  // ... 现有字段 ...
  oneof payload {
    // ... 现有类型 ...
    IceRestartRequest ice_restart_request = 14;  // 新增
  }
}
```

#### 步骤 2：PeerState 添加唤醒通知

```rust
// coordinator.rs

struct PeerState {
    // ... 现有字段 ...

    /// 用于外部唤醒退避等待（如对端网络恢复通知）
    restart_wake: Arc<tokio::sync::Notify>,
}

// 初始化时：
PeerState {
    // ...
    restart_wake: Arc::new(tokio::sync::Notify::new()),
}
```

#### 步骤 3：修改 do_ice_restart_inner 的退避等待

将 `tokio::time::sleep(delay).await` 替换为可被唤醒的等待：

```rust
// coordinator.rs - do_ice_restart_inner

async fn do_ice_restart_inner(
    // ... 现有参数 ...
    restart_wake: Arc<tokio::sync::Notify>,  // 新增参数
) -> RuntimeResult<bool> {

    for delay in backoff {
        // ========== Guard 1: Check signaling state ==========
        if !signaling_client.is_connected() {
            tokio::select! {
                _ = tokio::time::sleep(delay) => {}
                _ = restart_wake.notified() => {
                    tracing::info!(
                        "⚡ Backoff interrupted by peer notification (signaling guard), serial={}",
                        target.serial_number
                    );
                }
            }
            continue;
        }

        // ========== Guard 2: Check ICE gathering state ==========
        // ... 现有逻辑 ...

        // ========== Both guards passed, create offer and send ==========
        // ... 现有 offer 创建/发送逻辑 ...

        // Wait for restart completion
        let success = Self::wait_for_restart_completion_static(
            peers, target, ICE_RESTART_TIMEOUT
        ).await;

        if success {
            restart_ok = true;
            break;
        }

        // ====== 关键修改：退避等待可被唤醒 ======
        tokio::select! {
            _ = tokio::time::sleep(delay) => {
                // 正常退避到期
            }
            _ = restart_wake.notified() => {
                tracing::info!(
                    "⚡ ICE restart backoff interrupted by peer notification for serial={}",
                    target.serial_number
                );
            }
        }
    }

    // ... 现有结果处理 ...
}
```

#### 步骤 4：修改 restart_ice 支持 Answerer 分支

```rust
// coordinator.rs

pub async fn restart_ice(
    self: &Arc<Self>,
    target: &actr_protocol::ActrId,
) -> RuntimeResult<()> {
    // ... 现有锁和检查逻辑 ...

    let mut peers = self.peers.write().await;
    if let Some(state) = peers.get_mut(target) {
        // ... 现有去重检查（restart_task_handle、ice_restart_inflight）...

        if state.is_offerer {
            // Offerer：直接发起 ICE restart（现有逻辑）
            let restart_wake = state.restart_wake.clone();  // 传入 Notify
            // ... spawn do_ice_restart_inner(..., restart_wake) ...
        } else {
            // Answerer：通知 Offerer 立刻重试
            drop(peers); // 释放锁
            self.request_ice_restart_from_peer(target).await?;
        }
    }

    Ok(())
}
```

#### 步骤 5：实现 Answerer 端通知方法

```rust
// coordinator.rs

/// Answerer 通知 Offerer 网络已恢复，请立刻重试 ICE restart
async fn request_ice_restart_from_peer(
    self: &Arc<Self>,
    target: &ActrId,
) -> RuntimeResult<()> {
    tracing::info!(
        "📤 Sending IceRestartRequest to offerer serial={} (network recovered)",
        target.serial_number
    );

    let payload = actr_relay::Payload::IceRestartRequest(IceRestartRequest {
        from: self.local_id.clone(),
        to: target.clone(),
        reason: Some("network_recovered".to_string()),
    });

    // 注意：不设置 ice_restart_inflight
    // Answerer 端不需要管理 restart 状态，它只是发送通知
    // Offerer 端在 handle_ice_restart_request 中判断当前状态并决定如何处理
    if let Err(e) = self.send_actr_relay(target, payload).await {
        tracing::warn!(
            "⚠️ Failed to send IceRestartRequest to serial={}: {}",
            target.serial_number, e
        );
        return Err(e);
    }

    Ok(())
}
```

#### 步骤 6：Offerer 端处理通知

在 `handle_envelope` 方法中添加新的 payload 处理分支：

```rust
// coordinator.rs - handle_envelope 方法中添加

Some(actr_relay::Payload::IceRestartRequest(req)) => {
    tracing::info!(
        "📥 Received IceRestartRequest from serial={}, reason={:?}",
        req.from.serial_number,
        req.reason
    );
    if let Err(e) = self.handle_ice_restart_request(&req.from).await {
        tracing::error!("❌ Failed to handle IceRestartRequest: {}", e);
    }
}
```

新增处理方法：

```rust
// coordinator.rs

/// 处理来自 Answerer 的 ICE restart 通知
///
/// 三种情况：
/// 1. 已有 restart，且正在退避等待中 → 唤醒退避，让 Offerer 立刻重试
/// 2. 已有 restart，且 offer 已发出正在等 answer → 唤醒退避作为保险
///    （如果当前 wait 超时了，下一轮会立刻重试而非再等退避）
/// 3. 没有 restart 进行中 → 立即发起新的 restart
async fn handle_ice_restart_request(
    self: &Arc<Self>,
    from: &ActrId,
) -> RuntimeResult<()> {
    let (is_offerer, has_inflight_restart, is_waiting_answer) = {
        let peers = self.peers.read().await;
        match peers.get(from) {
            Some(state) => {
                let has_inflight = state.restart_task_handle
                    .as_ref()
                    .map(|h| !h.is_finished())
                    .unwrap_or(false);
                let waiting_answer = state.ice_restart_inflight;
                (state.is_offerer, has_inflight, waiting_answer)
            }
            None => {
                tracing::warn!(
                    "⚠️ IceRestartRequest from unknown peer serial={}",
                    from.serial_number
                );
                return Ok(());
            }
        }
    };
    
    if !is_offerer {
        tracing::warn!(
            "⚠️ Received IceRestartRequest but not offerer for serial={}",
            from.serial_number
        );
        return Ok(());
    }
    
    if has_inflight_restart {
        if is_waiting_answer {
            // Offer 已发出，正在 wait_for_restart_completion 中等待 answer
            // Answerer 网络刚恢复，可能还没收到我们之前发的 offer
            // 两种可能：
            //   a) Answerer 马上就会收到 offer 并回复 → 不需要干预
            //   b) Offer 已丢失（Answerer 断网期间发的）→ 等当前超时后自动重试
            // 无论哪种情况，唤醒退避都是安全的：
            //   - 如果当前 wait 成功了，唤醒 notify 会被后续 notified() 消费
            //   - 如果当前 wait 超时了，退避 sleep 会被立刻唤醒
            tracing::info!(
                "⚡ ICE restart offer already sent for serial={}, \
                 waking backoff in case current attempt times out",
                from.serial_number
            );
        } else {
            // restart task 在运行但 inflight=false，说明正在退避等待中
            tracing::info!(
                "⚡ Waking up ICE restart backoff for serial={} (peer network recovered)",
                from.serial_number
            );
        }
        let peers = self.peers.read().await;
        if let Some(state) = peers.get(from) {
            state.restart_wake.notify_one();
        }
    } else {
        // 没有 restart 在进行中，立刻发起
        tracing::info!(
            "♻️ Initiating ICE restart for serial={} (upon peer request)",
            from.serial_number
        );
        self.restart_ice(from).await?;
    }
    
    Ok(())
}
```

#### 步骤 7：信令服务器转发

`ActrRelay` 通常是透明转发，不需要信令服务器做额外处理。只需确保 proto 定义中 `IceRestartRequest` 作为 `ActrRelay.payload` 的一个变体即可。

---

## 3. 方案 B：固定间隔重试

### 3.1 核心思路

不引入新信令消息，将 Offerer 端现有的指数退避改为固定间隔（如每 5s 重试一次），缩短平均恢复延迟。

### 3.2 实现

仅需修改 `do_ice_restart_inner` 中的退避策略：

```rust
// coordinator.rs - do_ice_restart_inner

// 改动：将 ExponentialBackoff 替换为固定间隔
const ICE_RESTART_RETRY_INTERVAL: Duration = Duration::from_secs(5);

for _ in 0..ICE_RESTART_MAX_RETRIES {
    // ... 现有 guard 检查 ...
    // ... 创建并发送 offer ...
    // ... wait_for_restart_completion(5s) ...

    // 固定间隔等待（替代指数退避）
    tokio::time::sleep(ICE_RESTART_RETRY_INTERVAL).await;
}
```

### 3.3 特点

- ✅ **改动极小**：只改退避策略，一行代码
- ✅ **不需要 Proto 变更**：不引入新信令消息
- ✅ **不依赖 NetworkEvent**：纯 Offerer 端改动，服务端场景也适用
- ⚠️ **恢复延迟仍为秒级**：恢复延迟 = 0 ~ `(TIMEOUT + INTERVAL)`，平均 `(TIMEOUT + INTERVAL) / 2`（以当前配置 TIMEOUT=5s, INTERVAL=5s 为例：平均 ~5s，最坏 10s）
- ⚠️ **断网期间重试更频繁**：消耗更多信令流量和重试次数
- ⚠️ **长时间断网更快触发 cleanup**：固定频率消耗重试预算更快

---

## 4. 方案对比

| 维度                  | 方案 A：信令通知 + 唤醒退避                | 方案 B：固定间隔重试                                            |
| --------------------- | ------------------------------------------ | --------------------------------------------------------------- |
| **端到端恢复** ¹      | **~2 ~ 5s**（含 ICE 协商）                 | 平均 `(TIMEOUT+INTERVAL)/2` + ICE 协商时间（当前配置 ~7 ~ 15s） |
| **实现改动**          | 新增 Notify + 一条信令消息 + ~80 行代码    | **改一行（去掉指数）**                                          |
| **Proto 变更**        | 需要新增 `IceRestartRequest`               | **不需要**                                                      |
| **断网期间开销**      | **零额外开销**                             | 固定频率重试，信令流量较多                                      |
| **60s 内可尝试次数**  | **不消耗额外次数**                         | `60s / (TIMEOUT + INTERVAL)` 次（当前配置 ~6 次）               |
| **长时间断网**        | **不影响重试预算**                         | 消耗重试次数更快，更早触发 cleanup                              |
| **依赖 NetworkEvent** | 是（需要平台网络感知）                     | **否（纯 Offerer 端改动）**                                     |
| **适用场景**          | 对恢复延迟敏感、已有 NetworkEvent 基础设施 | 简单场景、无平台网络感知、可接受秒级延迟                        |

> ¹ **端到端恢复**：从 Answerer 网络恢复到 ICE restart 完全完成（连接状态变为 Connected）。**方案 A 耗时构成**：信令重试等待/RTT (~1s) + Offer/Answer 交换 + ICE candidate gathering + connectivity check (1~4s)。**方案 B 耗时构成**：盲等当前一轮剩余时间 `0 ~ (TIMEOUT + INTERVAL)` (当前配置平均 ~5s) + 相同的 ICE 协商开销。

### 建议

- 如果**已有 `NetworkEvent` 基础设施**（如当前项目的 iOS/Android 平台层），推荐 **方案 A**，充分利用平台网络感知能力
- 如果是**纯服务端场景**或不具备平台网络感知能力，**方案 B** 足够且更简单
- 两个方案**可以组合**：将指数退避改为固定间隔（方案 B），同时加上通知唤醒（方案 A），双保险

---

## 5. 边界场景分析（方案 A）

| 场景                                    | 行为                                                                                                         | 结果                               |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------ | ---------------------------------- |
| Offerer 没有 restart 进行中             | 收到通知 → 立刻 `restart_ice()`                                                                              | ✅ 直接发起 restart                 |
| Offerer 正在退避等待中                  | 收到通知 → `notify_one()` 唤醒                                                                               | ✅ 立刻进入下一次尝试               |
| **Offerer 已发 offer，等待 answer**     | 收到通知 → `notify_one()` 存储；Answerer 网络恢复后收到 offer → 回复 answer                                  | ✅ 当前尝试正常完成，通知被后续消费 |
| **Offerer 已发 offer，但 offer 已丢失** | 收到通知 → `notify_one()` 存储；当前 `wait_for_restart_completion` 超时 → 退避 sleep 被唤醒 → 立刻重发 offer | ✅ 自动恢复                         |
| Offerer 正在创建/发送 offer             | 收到通知 → `notify_one()` 无人等待                                                                           | ✅ 无影响，当前尝试继续             |
| Offerer restart 已完成                  | 收到通知 → peer 已 Connected                                                                                 | ✅ `restart_ice()` 去重跳过         |
| 信令未连通                              | `send_actr_relay` 失败                                                                                       | ✅ Offerer 退避循环自行重试         |
| 多次通知                                | 多次 `notify_one()`                                                                                          | ✅ Notify 幂等，无副作用            |
| Peer 已被 cleanup                       | `peers.get(from)` 返回 None                                                                                  | ✅ 直接返回，无错误                 |
