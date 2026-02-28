# actr 待解决问题清单

> **日期**: 2026-02-28  **状态**: 待确认

---

## 问题 1：态势感知模块：文档与代码实现不一致（已梳理）

- **设计文档**: [network-awareness-doc-vs-code.md](./network-awareness-doc-vs-code.md)
- **问题**: 多份文档描述了独立的 `Lifecycle` trait（含 `on_ready`、`on_error`、`on_signaling_connected` 等），但代码中从未实现。实际只有 `on_start`/`on_stop` 定义在 `Workload` trait 中。
- **设计方案**: 不引入独立 `Lifecycle` trait，在 `Workload` 中新增 `on_error`（运行时异常）和 `on_network_state`（网络状态变化）两个钩子，同步修改 6 处文档。

---

## 问题 2：完善 actr-runtime 错误处理（已梳理）

- **设计文档**:
  - [error-handling-analysis.md](./error-handling-analysis.md)（整体分析）
  - [transient-error-retry-send-design.md](./transient-error-retry-send-design.md)（自动重试设计）
  - [dlq-read-api-design.md](./dlq-read-api-design.md)（DLQ API 设计）

包含三个子问题：

### 2.1 错误映射不准确

- **问题**: `NetworkError` 中 4 个网络变体（`ConnectionClosed`/`ChannelClosed`/`SendError`/`Timeout`）被错误映射为 Permanent，应为 Transient。`OutprocOutGate` 直接使用 `ProtocolError`，绕过 `RuntimeError::is_retryable()` 判断。
- **设计方案**: 修正映射关系，统一走 `RuntimeError` 分类体系。
- **备注**: 这是 Transient 自动重试的前置工作。

### 2.2 Transient 错误自动重试

- **问题**: 框架只做了「自动重连」没做「自动重发」，网络短暂中断时正在发送的请求直接失败。
- **设计方案**: 在 `OutprocOutGate` 层对 `send_request()`/`send_message()` 加入指数退避重试（最多 3 次，deadline 内完成）。
- **需要决策**: 接收端去重——业务层自行去重（方案一）还是框架内置去重（方案二）？

### 2.3 DLQ 读取 API 缺失

- **问题**: DLQ 存储层已完整实现，但未暴露给应用层。使用者无法查询毒消息、Redrive 或获取监控指标。
- **设计方案**: 在 `ActrRef` 上新增 5 个委托方法（`dlq_stats`/`dlq_query`/`dlq_get`/`dlq_delete`/`dlq_redrive`），存储层无需修改。

---

## 问题 3：支持 Answer 端主动触发 ICE Restart（已梳理）

- **设计文档**: [ICE_RESTART_BIDIRECTIONAL.md](./ICE_RESTART_BIDIRECTIONAL.md)
- **问题**: ICE Restart 只允许 Offerer 端发起，Answerer 端网络恢复后无法主动触发，只能被动等待 Offerer 指数退避重试，恢复延迟可达 30~60s。
- **设计方案**:
  - 方案 A：Answerer 通过信令发送 `IceRestartRequest` 通知 Offerer 打断退避立即重试，恢复延迟 ~2~5s
  - 方案 B：将指数退避改为固定间隔（5s），恢复延迟 ~7~15s
  - 两个方案可组合使用

---

## 问题 4：提供大文件分片传输的高级 API（进行中）

- **问题**: 当前大文件传输需要应用层自行处理分片、重组、进度追踪等逻辑，使用复杂度高。
- **设计方案**: 框架封装 chunk 切分 + 进度回调 + 断点续传的高级 API。

---

## 问题 5：支持 Pub/Sub 消息模式（未开始）

- **问题**: 文档 2.2 提及 Pub/Sub 支持计划，但当前仅支持点对点通信，不支持发布/订阅模式。
- **设计方案**: 支持 Topic 发布/订阅 + 消息广播，与 Actor 模型自然融合。

---

## 问题 6：服务重启后旧实例残留（未开始）

- **问题**: 服务 kill 后重启，短时间内信令服务器中存在新旧两个实例，旧实例不可用但其他节点仍会尝试与其通信。
- **根因**: 心跳间隔过长，无法快速检测旧实例下线。
- **设计方案**: actr 端加入心跳配置（缩短间隔快速清理），或信令端实现同一服务 ID 新注册时主动清理旧实例。

---

## 问题 7：心跳超时后缺少服务重新注册逻辑（未开始）

- **问题**: 心跳超时后 actrix 会清理该服务注册信息，网络恢复后服务未重新注册，导致其他节点无法发现。
- **临时方案**: 从 SQLite 中恢复注册信息，SQLite 数据过期时间调整为 12h。
- **最终方案**: 检测到心跳恢复（或信令重连成功）后，自动重新注册服务到 actrix。

---

## 问题 8：连接管理代码优化及问题修复（进行中）

- **问题**: 连接生命周期管理有若干待优化项，包括清理流程统一、回调管理、任务生命周期跟踪等。

---
