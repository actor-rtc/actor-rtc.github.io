# actr-runtime 错误处理分析

> **更新**: 2026-02-27 | **范围**: `actr/crates/runtime/` | **目的**: 梳理文档中已设计但代码未实现的错误处理部分，明确实现计划

actr 采用**微服务视角**的错误处理哲学：**框架负责传输，应用负责业务**。

- 设计哲学详见 [3.11-production-readiness.zh.md §4](file:///Users/zhj/RustProject/actr-project/actor-rtc.github.io/zh-hans/3.11-production-readiness.zh.md)
- 应用层错误模式详见 [2.2-actor-cookbook.zh.md 模式六](file:///Users/zhj/RustProject/actr-project/actor-rtc.github.io/zh-hans/2.2-actor-cookbook.zh.md)

---

## 1. 框架错误分类（已实现）

源文件: [error.rs](../../actr/crates/runtime/src/error.rs)，`RuntimeError` 16 个变体，`classification()` 分三类。



### 错误分类

| 分类          | 含义           | 变体                                                                    | 处理策略                                                                   |
| ------------- | -------------- | ----------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| **Transient** | 临时性故障     | `Unavailable`, `DeadlineExceeded`                                       | 自动重试（[设计文档](./transient-error-retry-send-design.md)）             |
| **Permanent** | 需修复系统状态 | `NotFound`, `InvalidArgument`, `FailedPrecondition`, `PermissionDenied` | 直接返回调用方                                                             |
| **Internal**  | 框架内部错误   | `Internal`（handler panic）                                             | 直接返回调用方，不重试（当前代码 `is_retryable()` 将其归为可重试，需调整） |
| **Poison**    | 消息损坏       | `DecodeFailure`                                                         | 写入 DLQ + 人工排查                                                        |

---

## 2. 错误映射问题（需调整）

当前错误类型使用有些混乱，主要体现在两处映射不准确，导致本应可重试的错误被当作不可重试处理。

### 2.1 NetworkError 映射不准确

源文件: [error.rs L140](file:///Users/zhj/RustProject/actr-project/actr/crates/runtime/src/error.rs#L140)

`From<NetworkError> for RuntimeError` 中 4 个变体被错误映射为 Permanent，应为 Transient：

| NetworkError 变体  | 当前映射          | 应映射为                |
| ------------------ | ----------------- | ----------------------- |
| `ConnectionClosed` | Other (Permanent) | Unavailable (Transient) |
| `ChannelClosed`    | Other (Permanent) | Unavailable (Transient) |
| `SendError`        | Other (Permanent) | Unavailable (Transient) |

**超时拆分**：当前 `TimeoutError` 和 `Timeout` 两个变体语义不清，需合并拆分为：

| 当前变体       | 使用位置                            | 拆分后              | 映射             | 可重试？                       |
| -------------- | ----------------------------------- | ------------------- | ---------------- | ------------------------------ |
| `TimeoutError` | inproc_manager L329（RPC 请求超时） | → `RequestTimeout`  | DeadlineExceeded | **否**（业务超时，调用方设的） |
| `TimeoutError` | manager L234（等待连接超时）        | → `InternalTimeout` | DeadlineExceeded | **是**（框架内部，可重试）     |
| `Timeout`      | 从未被构造                          | 删除                | —                | —                              |

### 2.2 ProtocolError 绕过 RuntimeError

`OutprocOutGate` 直接使用 `ProtocolError::TransportError`，未经 `RuntimeError::is_retryable()` 判断。需补齐映射：`TransportError → Unavailable`，`Timeout → DeadlineExceeded`。

### 2.3 Handler Panic 返回错误类型不对

`handle_incoming` 捕获 panic 后返回 `ProtocolError::Actr(ActrError::DecodeFailure)`，但 `DecodeFailure` 语义是"消息损坏"，不是"代码崩溃"。`RuntimeError` 中已定义了 `Internal` 变体（含 `panic_info` 字段），应使用 `Internal` 而非 `DecodeFailure`。

| 维度     | 文档描述 (3.11 §4.3.2 场景 E) | 代码实现 (actr_node.rs)                |
| -------- | ----------------------------- | -------------------------------------- |
| 返回类型 | `INTERNAL`                    | `DecodeFailure`（注释写 "as a proxy"） |  |

### 2.4 NetworkError 变体使用不恰当

代码中构造 `NetworkError` 时存在变体选择不当，影响上层映射和重试判断。
主要问题：

| 位置                                      | 当前                 | 问题                       | 应使用                |
| ----------------------------------------- | -------------------- | -------------------------- | --------------------- |
| lane.rs L105 DC closed                    | `DataChannelError`   | 连接断开不是 DC 错误       | `ChannelClosed`       |
| lane.rs L110 DC open 超时                 | `DataChannelError`   | 超时应独立区分             | `TimeoutError`        |
| lane.rs L164 WS sink 为 None              | `ConnectionError`    | 从未连接，不是断开         | `ConnectionNotFound`  |
| dest_transport.rs L179 WireBuilder 返回空 | `ConfigurationError` | 不是配置问题               | `ConnectionError`     |
| manager.rs L259 同上                      | `ConfigurationError` | 同上                       | `ConnectionError`     |
| signaling.rs L734 等待超时                | `ConnectionError`    | 超时被淹没在连接错误中     | `TimeoutError`        |
| signaling.rs L867 注册响应格式不对        | `ConnectionError`    | 不是连接问题，是协议不匹配 | `ProtocolError`       |
| signaling.rs L979 心跳响应非 Pong         | `ConnectionError`    | 同上                       | `ProtocolError`       |
| signaling.rs L1016 路由候选响应格式不对   | `ConnectionError`    | 同上                       | `ProtocolError`       |
| signaling.rs L1050 凭证更新被拒           | `ConnectionError`    | 服务端拒绝，不是连接问题   | `AuthenticationError` |
| signaling.rs L1059 凭证更新响应格式不对   | `ConnectionError`    | 同上                       | `ProtocolError`       |
| signaling.rs L1097 WS 未连接时发送        | `ConnectionError`    | 已断开，不是连接失败       | `ConnectionClosed`    |
| signaling.rs L1107 inbound channel 关闭   | `ConnectionError`    | 内部 channel 关闭          | `ChannelClosed`       |
| wire_builder.rs L111 创建时被取消         | `ConnectionClosed`   | 还没有连接，不是关闭       | `ConnectionError`     |

> 以上四项是 Transient 自动重试的**前置工作**。



## 5. 讨论：SCTP 断开数据丢失算错误吗？

严格来说**不算错误**。从各层视角看：

| 视角        | 判断     | 原因                                              |
| ----------- | -------- | ------------------------------------------------- |
| **SCTP 层** | 不是错误 | `send()` 成功把数据交给了缓冲区，协议层面没有违约 |
| **框架层**  | 不是错误 | `dc.send()` 返回 Ok，RuntimeError 管道没有触发    |
| **应用层**  | 是问题   | 调用方以为 seq=11 发送成功，但对端没收到          |

本质是 **SCTP 没有提供端到端 ACK 语义**（或者说 ACK 不暴露给上层），导致"已交给缓冲区"和"对端已收到"之间存在语义鸿沟。

这个问题更接近于**网络状态变化引发的数据完整性风险**，而不是某个函数返回了错误。框架能做的是：
1. 尽快通知应用层"连接断了"（`on_network_state`）
2. 发送前检查连接状态（`is_connected()`）减小窗口
3. 最终的端到端保障留给应用层 ACK 机制

---

## 6. 待办优先级

| 优先级 | 事项                                                                            | 涉及章节 | 前置依赖 |
| ------ | ------------------------------------------------------------------------------- | -------- | -------- |
| 🔴 高   | NetworkError → RuntimeError 映射修正（3 个变体）                                | §2.1     | 无       |
| 🔴 高   | 超时拆分：`TimeoutError` → `RequestTimeout` + `InternalTimeout`，删除 `Timeout` | §2.1     | 无       |
| 🔴 高   | NetworkError 变体使用不当修正（14 处）                                          | §2.4     | 无       |
| 🔴 高   | Handler panic 返回 `Internal` 而非 `DecodeFailure`                              | §2.3     | 无       |
| 🔴 高   | `Internal.is_retryable()` → false                                               | §2.3     | 无       |
| � 高   | ProtocolError → RuntimeError 统一                                               | §2.2     | 无       |
| 🟠 高   | Transient 自动重试实现                                                          | —        | 上述所有 |
| 🟡 中   | DLQ 读取 API                                                                    | —        | 无       |
| 🟢 低   | 清理未使用的 NetworkError 变体（18 个）                                         | §1       | 无       |

---

## 7. 后续需修改的文件

### 代码

| 文件                                       | 修改内容                                                                                   |
| ------------------------------------------ | ------------------------------------------------------------------------------------------ |
| `runtime/src/transport/error.rs`           | 拆分 `TimeoutError` → `RequestTimeout` + `InternalTimeout`；删除 `Timeout`；清理未使用变体 |
| `runtime/src/error.rs`                     | 修正 `From<NetworkError>` 映射（§2.1）；`Internal.is_retryable()` → false                  |
| `runtime/src/transport/lane.rs`            | 修正 `DataChannelError` → `ChannelClosed` / `TimeoutError`（§2.4）                         |
| `runtime/src/transport/dest_transport.rs`  | `ConfigurationError` → `ConnectionError`（§2.4）                                           |
| `runtime/src/transport/manager.rs`         | 同上 + 超时变体改为 `InternalTimeout`                                                      |
| `runtime/src/transport/inproc_manager.rs`  | 超时变体改为 `RequestTimeout`                                                              |
| `runtime/src/wire/webrtc/signaling.rs`     | 修正 `ConnectionError` 滥用（§2.4，8 处）                                                  |
| `runtime/src/lifecycle/actr_node.rs`       | Handler panic → `RuntimeError::Internal`（§2.3）                                           |
| `runtime/src/outbound/outproc_out_gate.rs` | `ProtocolError` → `RuntimeError` 统一（§2.2）+ Transient 重试                              |

### 文档

| 文档                                                                           | 修改内容                   |
| ------------------------------------------------------------------------------ | -------------------------- |
| [transient-error-retry-send-design.md](./transient-error-retry-send-design.md) | 重试实现后标记为"已实现"   |
| [dlq-read-api-design.md](./dlq-read-api-design.md)                             | DLQ API 实现后更新状态     |
| [3.11-production-readiness.zh.md](./3.11-production-readiness.zh.md)           | 更新错误分类和超时拆分描述 |
