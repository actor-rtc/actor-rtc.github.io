# Python Bindings 及部分实现思路（待确认版本）

本文档详细描述了 `actr-runtime-py` 中需要绑定的结构、方法、参数和返回值，描述了部分实现思路

## 目录

1. [Python 绑定层（Bindings Architecture）](#python-绑定层bindings-architecture)
   1. [枚举类型](#枚举类型)
   2. [异常类型](#异常类型)
   3. [ActrSystem](#actrsystem)
   4. [ActrNode](#actrnode)
   5. [ActrRef](#actrref)
   6. [Context](#context)
   7. [ActorResult（可选）](#actorresult)
2. [实现思路](#实现思路)
   1. [Python Workload 接口](#python-workload-接口)
   2. [错误处理](#错误处理)
   3. [完整使用示例](#完整使用示例)
3. [问题](#问题)

---

## Python 绑定层（Bindings Architecture）

### 枚举类型

### PayloadType

消息传输类型枚举，用于指定 RPC 和 DataStream 的传输特性。

| 值 | 说明 |
|---|---|
| `RpcReliable` | 可靠的 RPC 请求（默认） |
| `RpcSignal` | 信号式 RPC |
| `StreamReliable` | 可靠的数据流（默认） |
| `StreamLatencyFirst` | 低延迟优先的数据流 |

---

### Dest

目标地址枚举，用于指定 RPC 调用的目标。

**类型**: `Dest` wrapper 类

**静态方法**：
- `Dest.shell()` - 创建 Shell 目标（Workload → App，inproc 反向通道）
- `Dest.local()` - 创建 Local 目标（调用本地 Workload）
- `Dest.actor(actr_id: ActrId)` - 创建 Actor 目标（调用远程 Actor）。`actr_id` 是 Python `ActrId` protobuf 对象（`generated.actr_pb2.ActrId`）

**实例方法**：
- `is_shell() -> bool` - 检查是否为 Shell 目标
- `is_local() -> bool` - 检查是否为 Local 目标
- `is_actor() -> bool` - 检查是否为 Actor 目标
- `as_actor_id() -> Optional[ActrId]` - 如果是 Actor 目标，返回 ActrId；否则返回 None

**语义说明**：
- **`Dest::Shell`**: Workload → App（inproc 反向通道）
  - 用于 Workload 调用 App 侧
  - 通过 `InprocOutGate` 路由（零序列化）
  - 示例：Workload 推送通知到 App

- **`Dest::Local`**: 调用本地 Workload
  - 从 App：通过 `InprocOutGate` 路由（零序列化）
  - 从 Workload：通过 `OutprocOutGate` 路由（完整序列化，在传输层短接）
  - 示例：App 调用其本地 Workload，或 Workload 调用自身

- **`Dest::Actor(ActrId)`**: 远程 Actor（完整的 outproc）
  - 用于跨进程 Actor 通信
  - 通过 `OutprocOutGate` 路由（WebRTC/WebSocket）
  - 示例：ClientWorkload 调用 RemoteServer
  - 使用 `Dest.actor(actr_id)` 创建，其中 `actr_id` 是 Python `ActrId` protobuf 对象（`generated.actr_pb2.ActrId`）

---

### 异常类型

Python 异常体系，继承关系如下：

```
ActrRuntimeError (基类)
├── ActrTransportError (传输错误)
├── ActrDecodeError (编解码错误)
├── ActrUnknownRoute (未知路由)
└── ActrGateNotInitialized (Gate 未初始化)
```

所有异常都继承自 `PyException`，可以通过 `except actr_runtime_py.ActrRuntimeError` 捕获。

---

### ActrSystem

Actor 系统，用于创建和管理 Actor 节点。

### 方法

| 方法 | 参数 | 返回值 | 说明 |
|---|---|---|---|
| `from_toml(path: str)` | `path`: TOML 配置文件路径 | `ActrSystem` (async) | 从 TOML 文件加载配置并创建 ActrSystem |
| `from_env()` | 无 | `ActrSystem` (async) | 从环境变量加载配置并创建 ActrSystem（按需提供） |
| `default()` | 无 | `ActrSystem` (async) | 使用默认配置创建 ActrSystem（按需提供） |
| `attach(workload: PyObject)` | `workload`: Python Workload 对象 | `ActrNode` | 将 Workload 附加到系统，返回 ActrNode（消耗 ActrSystem）。**必须在异步上下文中调用**（例如在 `async def main()` 中），以便获取 Python 事件循环句柄 |

**注意**：
- `attach` 方法会消耗 `ActrSystem`，每个 `ActrSystem` 只能调用一次 `attach`
- `attach` 必须在异步上下文中调用（例如在 `async def main()` 中），以便获取当前线程的 Python 事件循环句柄并保存到 `PyWorkloadWrapper` 中
- 保存的事件循环句柄用于后续的 `dispatch` 调用，通过 `run_coroutine_threadsafe` 将协程投递到 Python 事件循环

---

### ActrNode

Actor 节点，表示一个已附加 Workload 但尚未启动的节点。

### 方法

| 方法 | 参数 | 返回值 | 说明 |
|---|---|---|---|
| `start()` | 无 | ``ActorResult[ActrRef]` (async) | 尝试启动节点，返回 ActorResult（消耗 ActrNode） |


**注意**：
- `start()` 消耗 `ActrNode`，每个 `ActrNode` 只能调用一次。
- `start()` 失败返回 `ActorResult`。

---

### ActrRef

Actor 引用，表示一个已启动的 Actor 节点。用于 Shell → Workload 的 RPC 调用（进程内通信）。

### 方法 (需要在Rust 充补充 call_raw, tell_raw 方法)

| 方法 | 参数 | 返回值 | 说明 |
|---|---|---|---|
| `actor_id()` | 无 | Python `ActrId` protobuf 对象 | 返回 Actor 的 ID（`generated.actr_pb2.ActrId`） |
| `call_raw(route_key, request, timeout_ms=30000, payload_type=PayloadType.RpcReliable, response_type=None)` | `route_key`: 路由键字符串<br>`request`: 请求 protobuf bytes<br>`timeout_ms`: 超时时间（毫秒，可选）<br>`payload_type`: 传输类型（可选）<br>`response_type`: 可选的 Python protobuf 类，用于自动反序列化 | `ActorResult[bytes \| ResponseObject]` (async) | 调用 Actor 方法（Shell → Workload RPC）。如果提供了 `response_type`，返回反序列化的 protobuf 对象；否则返回 bytes |
| `tell_raw(route_key, message, payload_type=PayloadType.RpcReliable)` | `route_key`: 路由键字符串<br>`message`: 消息 protobuf bytes<br>`payload_type`: 传输类型（可选） | `ActorResult[None]` (async) | 发送单向消息到 Actor（Shell → Workload，fire-and-forget） |
| `shutdown()` | 无 | `None` | 立即关闭 Actor（同步） |
| `wait_for_shutdown()` | 无 | `None` (async) | 等待 Actor 完全关闭 |
| `wait_for_ctrl_c_and_shutdown()` | 无 | `None` (async) | 等待 Ctrl+C 信号，然后关闭 Actor（消耗 ActrRef） |

**注意**：
- `wait_for_ctrl_c_and_shutdown()` 会消耗 `ActrRef`，通常用于主循环。
- `actor_id()` 返回的是 Python `ActrId` protobuf 对象（`generated.actr_pb2.ActrId`）。
- **`call_raw()` 和 `tell_raw()` 用于 Shell → Workload 的进程内 RPC 调用**，目标自动设置为当前 Actor，不需要指定 target。
- **对于 Workload → 其他 Actor 的 RPC 调用**，应使用 `Context.call_raw()` 和 `Context.tell_raw()`，需要指定 target Actor ID。
- `call_raw()` 和 `tell_raw()` 总是返回 `ActorResult`，不会抛出异常。
- **自动反序列化**：`call_raw()` 支持可选的 `response_type` 参数。如果提供，会自动调用 `FromString()` 方法反序列化响应，返回 Python protobuf 对象而不是 bytes。这提供了更友好的 API，同时保持向后兼容（不提供 `response_type` 时返回 bytes）。

**ActrRef vs Context 的区别**：

| 方面 | ActrRef (Shell → Workload) | Context (Workload → 其他 Actor) |
|------|---------------------------|--------------------------------|
| 调用者 | Shell（启动 Actor 的代码） | Workload（Actor 的业务逻辑） |
| 目标 | 当前 Actor | 需要指定 `Dest`（`Dest::Shell`、`Dest::Local` 或 `Dest::Actor(id)`） |
| 通信方式 | 进程内（in-process） | 进程内或进程间（取决于 `Dest`） |
| 方法 | `actr_ref.call(route_key, request_bytes)` | `ctx.call_raw(target, route_key, request_bytes)` |
| 性能 | ~10μs（零序列化） | 取决于目标类型（Shell/Local: ~10μs，Actor: 网络延迟） |

**使用示例**：

```python
from actr_runtime_py import ActrSystem, PayloadType
from generated import my_service_pb2

# 启动 Actor
system = await ActrSystem.from_toml("config/Actr.toml")
workload = MyWorkload()
node = system.attach(workload)
actr_ref = await node.start()

# 调用 Actor 方法（Shell → Workload）
req = my_service_pb2.EchoRequest(message="Hello")

# 方式 1：返回 bytes（需要手动反序列化）
result = await actr_ref.call_raw(
    "my_service.EchoService.Echo",
    req.SerializeToString(),
    timeout_ms=30_000,
    payload_type=PayloadType.RpcReliable
)
if result.is_ok():
    response_bytes = result.unwrap()
    response = my_service_pb2.EchoResponse.FromString(response_bytes)
    print(f"Response: {response.message}")
else:
    error = result.unwrap_err()
    print(f"Error: {error}")

# 方式 2：自动反序列化（更友好）
result = await actr_ref.call_raw(
    "my_service.EchoService.Echo",
    req.SerializeToString(),
    timeout_ms=30_000,
    payload_type=PayloadType.RpcReliable,
    response_type=my_service_pb2.EchoResponse  # 提供 response_type
)
if result.is_ok():
    response = result.unwrap()  # 直接是 EchoResponse 对象，无需手动反序列化
    print(f"Response: {response.message}")
else:
    error = result.unwrap_err()
    print(f"Error: {error}")

# 发送单向消息（Shell → Workload）
event = my_service_pb2.LogEvent(level="INFO", message="User logged in")
result = await actr_ref.tell_raw(
    "my_service.EchoService.LogEvent",
    event.SerializeToString(),
    payload_type=PayloadType.RpcReliable
)
if result.is_ok():
    print("Event sent successfully")
else:
    error = result.unwrap_err()
    print(f"Error: {error}")
```

---

### Context

Actor 上下文，提供 Actor 身份、服务发现、RPC 和 DataStream 能力。

### 数据访问方法

| 方法 | 参数 | 返回值 | 说明 |
|---|---|---|---|
| `self_id()` | 无 | Python `ActrId` protobuf 对象 | 返回当前 Actor 的 ID（`generated.actr_pb2.ActrId`） |
| `caller_id()` | 无 | `Optional[ActrId]` (protobuf 对象) | 返回调用者 Actor ID（RPC 调用时可用，`generated.actr_pb2.ActrId`） |
| `request_id()` | 无 | `str` | 返回当前请求的 ID |

### 服务发现方法

| 方法 | 参数 | 返回值 | 说明 |
|---|---|---|---|
| `discover_route_candidate(actr_type: ActrType)` | `actr_type`: Python `ActrType` protobuf 对象（`generated.actr_pb2.ActrType`） | Python `ActrId` protobuf 对象 (async) | 发现指定类型的 Actor 实例，返回其 ID（`generated.actr_pb2.ActrId`） 
**注意**：
- `actr_type` 参数是 Python protobuf 对象，需要使用 `generated.actr_pb2.ActrType` 创建。
- 返回值是 Python `ActrId` protobuf 对象（`generated.actr_pb2.ActrId`），不是字符串。

### RPC 方法 (Rust 需要补充call_raw,tell_raw 方法)

| 方法 | 参数 | 返回值 | 说明 |
|---|---|---|---|
| `call_raw(target, route_key, request, timeout_ms=30000, payload_type=PayloadType.RpcReliable, response_type=None)` | `target`: `Dest` wrapper 对象<br>`route_key`: 路由键字符串<br>`request`: 请求 protobuf bytes<br>`timeout_ms`: 超时时间（毫秒，可选）<br>`payload_type`: 传输类型（可选）<br>`response_type`: 可选的 Python protobuf 类，用于自动反序列化 | `ActorResult[bytes \| ResponseObject]` (async) | 执行请求/响应 RPC 调用。如果提供了 `response_type`，返回反序列化的 protobuf 对象；否则返回 bytes |
| `tell_raw(target, route_key, message, payload_type=PayloadType.RpcReliable)` | `target`: `Dest` wrapper 对象<br>`route_key`: 路由键字符串<br>`message`: 消息 protobuf bytes<br>`payload_type`: 传输类型（可选） | `ActorResult[None]` (async) | 执行单向消息 RPC 调用（fire-and-forget），返回 `ActorResult`。成功时包含 None，失败时包含错误 |

**使用示例**（请求/响应 RPC）：

```python
from actr_runtime_py import Dest, PayloadType
from generated import my_service_pb2

# 调用远程 Actor
target = Dest.actor(server_id)  # server_id 是 Python ActrId protobuf 对象（generated.actr_pb2.ActrId）
req = my_service_pb2.EchoRequest(message="Hello")
req_bytes = req.SerializeToString()

# 方式 1：返回 bytes（需要手动反序列化）
result = await ctx.call_raw(
    target,
    "my_service.EchoService.Echo",
    req_bytes,
    timeout_ms=30_000,
    payload_type=PayloadType.RpcReliable
)
if result.is_ok():
    response_bytes = result.unwrap()
    response = my_service_pb2.EchoResponse.FromString(response_bytes)
    print(f"Response: {response.message}")
else:
    error = result.unwrap_err()
    print(f"Error: {error}")

# 方式 2：自动反序列化（更友好）
result = await ctx.call_raw(
    target,
    "my_service.EchoService.Echo",
    req_bytes,
    timeout_ms=30_000,
    payload_type=PayloadType.RpcReliable,
    response_type=my_service_pb2.EchoResponse  # 提供 response_type
)
if result.is_ok():
    response = result.unwrap()  # 直接是 EchoResponse 对象，无需手动反序列化
    print(f"Response: {response.message}")
else:
    error = result.unwrap_err()
    print(f"Error: {error}")

# 调用 Shell（从 Workload）
shell_target = Dest.shell()
result = await ctx.call_raw(
    shell_target,
    "my_service.EchoService.NotifyApp",
    req_bytes
)
```

**使用示例**（单向消息 RPC）：

```python
from actr_runtime_py import Dest, PayloadType
from generated import my_service_pb2

# 调用远程 Actor
target = Dest.actor(server_id)  # server_id 是 Python ActrId protobuf 对象（generated.actr_pb2.ActrId）
event = my_service_pb2.LogEvent(level="INFO", message="User logged in")
event_bytes = event.SerializeToString()

result = await ctx.tell_raw(
    target,
    "my_service.EchoService.LogEvent",
    event_bytes,
    payload_type=PayloadType.RpcReliable
)
if result.is_ok():
    print("Event sent successfully")
else:
    error = result.unwrap_err()
    print(f"Error: {error}")

# 调用 Shell（从 Workload）
shell_target = Dest.shell()
result = await ctx.tell_raw(
    shell_target,
    "my_service.EchoService.NotifyApp",
    event_bytes
)
```

**注意**：
- `target` 参数是 `Dest` wrapper 对象（`actr_runtime_py.Dest`），可以通过 `Dest.shell()`、`Dest.local()` 或 `Dest.actor(actr_id)` 创建。
- `call_raw` 和 `tell_raw` 总是返回 `ActorResult`，不会抛出异常。需要使用 `result.is_ok()` 检查成功，或使用 `result.unwrap()` 获取值（失败时会抛出异常）。
- **语义区分**：`call_raw` 用于需要响应的 RPC 调用，`tell_raw` 用于不需要响应的单向消息。
- **自动反序列化**：`call_raw` 支持可选的 `response_type` 参数。如果提供，会自动调用 `FromString()` 方法反序列化响应，返回 Python protobuf 对象而不是 bytes。这提供了更友好的 API，同时保持向后兼容（不提供 `response_type` 时返回 bytes）。
- **Dest 选择指南**：
  - 调用远程 Actor：使用 `Dest.actor(actr_id)`
  - 从 Workload 调用 App：使用 `Dest.shell()`
  - 调用本地 Workload：使用 `Dest.local()`

### DataStream 方法

| 方法 | 参数 | 返回值 | 说明 |
|---|---|---|---|
| `register_stream(stream_id: str, callback: Callable)` | `stream_id`: 流标识符<br>`callback`: 回调函数 `async def callback(data_stream: DataStream, sender_id: ActrId) -> None` | `None` (async) | 注册一个 DataStream 回调函数。当收到匹配的 DataStream 时，会调用回调函数。回调函数接收 `DataStream` protobuf 对象和发送者 `ActrId` protobuf 对象 |
| `unregister_stream(stream_id: str)` | `stream_id`: 流标识符 | `None` (async) | 取消注册 DataStream |
| `send_data_stream(target: Dest, data_stream: DataStream)` | `target`: 目标地址（`Dest` wrapper 对象）<br>  - 使用 `Dest.shell()` 用于 Workload → App 调用<br>  - 使用 `Dest.local()` 用于调用本地 Workload<br>  - 使用 `Dest.actor(actr_id)` 用于调用远程 Actor<br>`data_stream`: DataStream protobuf 对象（`generated.package_pb2.DataStream`） | `ActorResult[None]` (async) | 发送 DataStream 数据块，返回 `ActorResult`。成功时包含 None，失败时包含错误。默认使用 `StreamReliable` 传输类型 |

**注意**：
- `register_stream()` 的回调函数签名：`async def callback(data_stream: DataStream, sender_id: ActrId) -> None`
  - `data_stream`: `generated.package_pb2.DataStream` 对象
  - `sender_id`: `generated.actr_pb2.ActrId` 对象
- `send_data_stream` 接受 `Dest` wrapper 对象和 `DataStream` protobuf 对象。
- `send_data_stream` 默认使用 `StreamReliable` 传输类型。
- `send_data_stream` 总是返回 `ActorResult`，不会抛出异常。需要使用 `result.is_ok()` 检查成功，或使用 `result.unwrap()` 获取值（失败时会抛出异常）。
- **最佳实践**：在发送 DataStream 请求之前先注册回调（使用 `register_stream()`），避免丢失首包。

---

### ActorResult

Rust `ActorResult<T>` 的 Python 包装，用于表示可能成功或失败的操作。

### 属性

| 属性 | 类型 | 说明 |
|---|---|---|
| `ok` | `bool` | 操作是否成功 |
| `value` | `Optional[T]` | 成功时的值（`ok == True` 时可用） |
| `error` | `Optional[Exception]` | 失败时的异常（`ok == False` 时可用） |

### 方法

| 方法 | 参数 | 返回值 | 说明 |
|---|---|---|---|
| `is_ok()` | 无 | `bool` | 检查操作是否成功 |
| `unwrap()` | 无 | `T` | 如果成功返回值，否则抛出异常 |
| `unwrap_err()` | 无 | `Exception` | 如果失败返回异常，否则抛出 `ValueError` |

**使用示例**：
```python
from actr_runtime_py import Dest, PayloadType
from generated import my_service_pb2, actr_pb2

target_id = actr_pb2.ActrId(...)  # Python ActrId protobuf 对象
target = Dest.actor(target_id)  # 转换为 Dest

# 请求/响应 RPC（自动反序列化）
req = my_service_pb2.EchoRequest(message="Hello")
result = await ctx.call_raw(
    target,
    "my_service.EchoService.Echo",
    req.SerializeToString(),
    response_type=my_service_pb2.EchoResponse
)
if result.is_ok():
    response = result.unwrap()  # EchoResponse 对象
else:
    error = result.unwrap_err()

# 单向消息 RPC
event = my_service_pb2.LogEvent(level="INFO", message="User logged in")
result = await ctx.tell_raw(
    target,
    "my_service.EchoService.LogEvent",
    event.SerializeToString()
)
if result.is_ok():
    # 消息发送成功
else:
    error = result.unwrap_err()
```

---

**使用示例**：

使用 `register_stream`（直接回调）：
```python
from generated import package_pb2, actr_pb2

async def my_callback(data_stream: package_pb2.DataStream, sender_id: actr_pb2.ActrId):
    # data_stream 和 sender_id 都是 protobuf 对象，直接使用
    print(f"Received from {sender_id}: seq={data_stream.sequence}, payload={data_stream.payload}")

await ctx.register_stream("my-stream-id", my_callback)
```

**注意**：
- `register_stream` 允许自定义回调逻辑，可以访问 `sender_id` 等信息
- 使用完毕后应调用 `unregister_stream()` 取消注册
- 回调函数接收的是 protobuf 对象，不需要手动反序列化

---


## 实现思路

### Python Workload 接口

Python Workload 采用Workload 对象返回 Dispatcher 对象，实现三层架构：

1. **Handler**：定义业务逻辑接口，实现具体的 RPC 处理方法
2. **Dispatcher**：负责消息路由，根据 `route_key` 调用 Handler 的相应方法
3. **Workload**：用户实现的业务逻辑与协议适配层，既可作为服务端提供能力，也可作为客户端消费能力;负责生命周期管理，组合 Handler 和 Dispatcher

### 架构概览

```
┌─────────────────────────────────────────────────────────┐
│                    Workload                              │
│  ┌──────────────┐         ┌──────────────┐             │
│  │   Handler    │◄────────│  Dispatcher  │             │
│  │ (业务逻辑)    │         │  (消息路由)   │             │
│  └──────────────┘         └──────────────┘             │
│         │                        ▲                       │
│         │                        │                       │
│    on_start()              get_dispatcher()              │
│    on_stop()                                            │
└─────────────────────────────────────────────────────────┘
```

### Binding 实现

Python 的 Workload、Dispatcher 和 Handler 都是纯 Python 类，通过 Rust Binding 层桥接到 Rust Runtime。

#### PyWorkloadWrapper

**作用**：包装 Python Workload 对象，实现 Rust 的 `Workload` trait。

**主要实现**：

1. **创建包装器**：
   - 将 Python Workload 对象转换为 `Py<PyAny>` 存储
   - 在 `attach()` 时获取并保存 Python 事件循环句柄

2. **获取 Dispatcher**：
   - 通过反射调用 Python 的 `get_dispatcher()` 方法
   - 返回 Python Dispatcher 对象

3. **生命周期方法**：
   - `on_start`/`on_stop`：通过反射调用 Python 方法
   - 使用 `pyo3_asyncio::tokio::into_future()` 将 Python 协程转换为 Rust Future

**示例代码**：
```rust
pub struct PyWorkloadWrapper {
    py_obj: Py<PyAny>,  // Python Workload 对象
    event_loop: Option<Py<PyAny>>,  // Python 事件循环句柄（在 attach() 时设置）
}

impl PyWorkloadWrapper {
    // 设置事件循环句柄（在 attach() 时调用）
    pub fn set_event_loop(&mut self, loop_obj: Py<PyAny>) {
        self.event_loop = Some(loop_obj);
    }
    
    // 获取 Python Dispatcher
    fn get_dispatcher(&self) -> Option<Py<PyAny>> {
        Python::with_gil(|py| -> PyResult<Option<Py<PyAny>>> {
            let obj = self.py_obj.as_ref(py);
            // 调用 Python 的 get_dispatcher() 方法
            if let Ok(dispatcher) = obj.call_method0("get_dispatcher") {
                dispatcher.extract::<Py<PyAny>>().map(Some)
            } else {
                Ok(None)
            }
        }).ok().flatten()
    }
}
```

#### PyDispatcher

**作用**：实现 Rust 的 `MessageDispatcher` trait，调用 Python Dispatcher。

**主要实现**：

1. **dispatch 方法**：
   - 调用 `workload.get_dispatcher()` 获取 Python Dispatcher
   - 调用 `dispatch()` 执行分发


**示例代码**：
```rust
pub struct PyDispatcher;

impl MessageDispatcher for PyDispatcher {
    async fn dispatch(
        workload: &PyWorkloadWrapper,
        dispatcher_obj: Py<PyAny>,
        runtime_ctx: &RuntimeContext,
        route_key: String,
        payload: Vec<u8>,
    ) -> ActorResult<Bytes> {
        // 使用保存的事件循环句柄
        let event_loop = workload.event_loop.as_ref()
            .ok_or_else(|| ProtocolError::TransportError("Event loop not set".to_string()))?;
        
        // 使用 run_coroutine_threadsafe + wrap_future 避免 spawn_blocking
        let fut = Python::with_gil(|py| -> PyResult<_> {
            // 创建协程
            let ctx_py = make_ctx_py(py, &runtime_ctx)?;
            let dispatcher = dispatcher_obj.as_ref(py);
            let workload_py = workload_obj.as_ref(py);
            let ctx_obj = ctx_py.to_object(py);
            let route = PyString::new(py, &route_key);
            let pay = PyBytes::new(py, &payload);
            
            let coro = dispatcher.call_method1("dispatch", (workload_py, route, pay, ctx_obj))?;
            
            // 使用 run_coroutine_threadsafe 将协程投递到 Python 事件循环
            let asyncio = py.import("asyncio")?;
            let run_coroutine_threadsafe = asyncio.getattr("run_coroutine_threadsafe")?;
            let concurrent_future = run_coroutine_threadsafe.call1((coro, event_loop.as_ref(py)))?;
            
            // 使用 wrap_future 将 concurrent.futures.Future 转换为 asyncio.Future
            let wrap_future = asyncio.getattr("wrap_future")?;
            let asyncio_future = wrap_future.call1((concurrent_future,))?;
            
            // 使用 into_future 将 asyncio.Future 转换为 Rust Future
            pyo3_asyncio::tokio::into_future(asyncio_future)
        })?;
        
        // 直接 await Rust Future，完全异步，不需要 spawn_blocking
        let result_obj = fut.await?;
        
        // 提取返回的 bytes
        Python::with_gil(|py| {
            result_obj.extract::<&PyBytes>(py)
                .map(|b| Bytes::from(b.as_bytes().to_vec()))
        })
    }
}
```

#### 调用流程

```
Rust Runtime 收到 RPC (Tokio worker thread)
  ↓
PyDispatcher::dispatch()
  ↓
workload.get_dispatcher() → 获取 Python Dispatcher
  ↓
Python Dispatcher 路由 → 调用 workload.handler.method(req, ctx)
  ↓
返回 Python protobuf 对象 → 序列化为 bytes
  ↓
返回 Rust Bytes
```

**关键点**：
- Python 类通过 PyO3 反射调用，无需暴露给 Rust
- 类型转换通过 protobuf 序列化/反序列化完成

---

### Handler 接口（业务逻辑）

Handler 定义业务逻辑接口，实现具体的 RPC 处理方法。

**接口要求**：
- Handler 是一个类，定义业务逻辑方法
- 每个 RPC 方法对应一个业务逻辑方法
- 方法签名：`async def method_name(self, req: RequestType, ctx: Context) -> ResponseType`

**使用示例**：
```python
from generated import data_stream_multi_pb2 as pb2

class MyServiceHandler:
    """业务逻辑 Handler"""
    
    async def start_stream(self, req: pb2.StartStreamRequest, ctx) -> pb2.StartStreamResponse:
        """处理 StartStream RPC 请求"""
        # 业务逻辑实现
        return pb2.StartStreamResponse(
            accepted=True,
            stream_id=req.stream_id,
            message="Stream started"
        )
    
    async def ack_completion(self, req: pb2.AckCompletionRequest, ctx) -> pb2.AckCompletionResponse:
        """处理 AckCompletion RPC 请求"""
        # 业务逻辑实现
        return pb2.AckCompletionResponse(
            acknowledged=True,
            message="Acknowledged"
        )
    
    # 可选：生命周期方法
    async def on_start(self, ctx):
        """Actor 启动时的初始化逻辑"""
        pass
    
    async def on_stop(self, ctx):
        """Actor 停止时的清理逻辑"""
        pass
```

**注意**：
- Handler 方法接收 protobuf 请求对象和 `Context` 对象
- Handler 方法返回 protobuf 响应对象
- Handler 可以可选地实现 `on_start` 和 `on_stop` 生命周期方法

### Dispatcher 接口（消息路由）

Dispatcher 负责消息路由，根据 `route_key` 调用 Handler 的相应方法,应该自动生成。

| 方法 | 参数 | 返回值 | 说明 |
|---|---|---|---|
| `dispatch(workload, route_key: str, payload: bytes, ctx: Context)` | `workload`: Workload 实例<br>`route_key`: 路由键字符串<br>`payload`: 请求 protobuf bytes<br>`ctx`: Context 对象 | `bytes` (async) | 根据 route_key 反序列化请求，调用 Handler 的相应方法，返回序列化后的响应 bytes |

**使用示例**：
```python
from generated import data_stream_multi_pb2 as pb2

ROUTE_KEY_START_STREAM = "data_stream_multi.DataStreamMultiService.StartStream"
ROUTE_KEY_ACK_COMPLETION = "data_stream_multi.DataStreamMultiService.AckCompletion"

class MyServiceDispatcher:
    """消息路由 Dispatcher"""
    
    async def dispatch(self, workload, route_key: str, payload: bytes, ctx) -> bytes:
        """
        Dispatcher 的 dispatch 方法
        
        参数：
        - workload: MyServiceWorkload 实例（包含 handler 属性）
        - route_key: 路由键字符串
        - payload: 请求 protobuf bytes
        - ctx: Context 对象
        
        返回：响应 protobuf bytes
        """
        if route_key == ROUTE_KEY_START_STREAM:
            # 反序列化请求
            req = pb2.StartStreamRequest.FromString(payload)
            # 调用 Handler 的业务逻辑方法
            resp = await workload.handler.start_stream(req, ctx)
            # 序列化响应
            return resp.SerializeToString()
        
        elif route_key == ROUTE_KEY_ACK_COMPLETION:
            req = pb2.AckCompletionRequest.FromString(payload)
            resp = await workload.handler.ack_completion(req, ctx)
            return resp.SerializeToString()
        
        else:
            raise RuntimeError(f"Unknown route_key: {route_key}")
```

**注意**：
- Dispatcher 的 `dispatch` 方法接收**四个参数**：`(workload, route_key, payload, ctx)`
- Dispatcher 通过 `workload.handler.method()` 调用 Handler 的业务逻辑方法
- Dispatcher 负责请求的反序列化和响应的序列化
- 必须返回 `bytes`（protobuf 序列化后的响应）

### Workload 接口

Workload 由用户实现的业务逻辑与协议适配层，既可作为服务端提供能力，也可作为客户端消费能力；负责生命周期管理，组合 Handler 和 Dispatcher。

| 方法 | 参数 | 返回值 | 说明 |
|---|---|---|---|
| `on_start(ctx: Context)` | `ctx`: Context 对象 | `None` (async) | Actor 启动时的初始化逻辑 |
| `on_stop(ctx: Context)` | `ctx`: Context 对象 | `None` (async) | Actor 停止时的清理逻辑 |
| `get_dispatcher()` | 无 | `Dispatcher` 对象 | 返回 Dispatcher 实例，用于处理消息路由 |

**使用示例**：
```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class MyServiceWorkload:
    """Workload：组合 Handler 和 Dispatcher"""
    handler: MyServiceHandler  # Handler 实例
    _dispatcher: Optional[MyServiceDispatcher] = None
    
    def __post_init__(self):
        """初始化 Dispatcher 实例"""
        if self._dispatcher is None:
            self._dispatcher = MyServiceDispatcher()
    
    def get_dispatcher(self):
        """返回与此 Workload 关联的 Dispatcher"""
        return self._dispatcher
    
    async def on_start(self, ctx) -> None:
        """生命周期钩子：Actor 启动时调用"""
        # 如果 Handler 实现了 on_start，则调用它
        if hasattr(self.handler, "on_start"):
            await self.handler.on_start(ctx)
    
    async def on_stop(self, ctx) -> None:
        """生命周期钩子：Actor 停止时调用"""
        # 如果 Handler 实现了 on_stop，则调用它
        if hasattr(self.handler, "on_stop"):
            await self.handler.on_stop(ctx)
```

**完整使用示例**：
```python
from actr_runtime_py import ActrSystem

# 1. 创建 Handler（实现业务逻辑）
handler = MyServiceHandler()

# 2. 创建 Workload（自动创建 Dispatcher）
workload = MyServiceWorkload(handler)

# 3. 附加到系统并启动
system = await ActrSystem.from_toml("Actr.toml")
node = system.attach(workload)
actr_ref = await node.start()
```

**注意**：
- Workload 必须包含一个 `handler` 属性（Handler 实例）
- Workload 必须实现 `get_dispatcher()` 方法，返回 Dispatcher 实例
- Workload 的 `on_start` 和 `on_stop` 可以委托给 Handler（如果 Handler 实现了这些方法）
- 所有方法都是异步的（`async def`）


---

### 错误处理

### 异常映射

Rust 的错误类型会映射到 Python 异常：

| Rust 错误 | Python 异常 |
|---|---|
| `ProtocolError::TransportError` | `ActrTransportError` |
| `ProtocolError::DecodeError` / `EncodeError` / `DeserializationError` | `ActrDecodeError` |
| `ProtocolError::UnknownRoute` | `ActrUnknownRoute` |
| `ActrError::GateNotInitialized` | `ActrGateNotInitialized` |
| 其他 | `ActrRuntimeError` |

### 使用 ActorResult

`call_raw` 和 `tell_raw` 方法总是返回 `ActorResult`，可以使用以下方式进行错误处理：

```python
from actr_runtime_py import Dest, PayloadType
from generated import my_service_pb2, actr_pb2

target_id = actr_pb2.ActrId(...)  # Python ActrId protobuf 对象
target = Dest.actor(target_id)  # 转换为 Dest

# 请求/响应 RPC（自动反序列化）
req = my_service_pb2.EchoRequest(message="Hello")
result = await ctx.call_raw(
    target,
    "my_service.EchoService.Echo",
    req.SerializeToString(),
    response_type=my_service_pb2.EchoResponse
)
if result.is_ok():
    response = result.unwrap()  # EchoResponse 对象
    # 处理成功情况
else:
    error = result.unwrap_err()
    # 处理错误情况

# 单向消息 RPC
event = my_service_pb2.LogEvent(level="INFO", message="User logged in")
result = await ctx.tell_raw(
    target,
    "my_service.EchoService.LogEvent",
    event.SerializeToString()
)
if result.is_ok():
    # 消息发送成功
else:
    error = result.unwrap_err()
    # 处理错误情况
```

---

### 完整使用示例

### 基础示例：RPC 服务（Handler + Dispatcher + Workload 三层架构）

```python
import asyncio
from dataclasses import dataclass
from typing import Optional
from actr_runtime_py import ActrSystem, PayloadType, log
from generated import my_service_pb2

# 路由键常量
ROUTE_KEY_HELLO = "my_service.EchoService.Hello"
ROUTE_KEY_PING = "my_service.EchoService.Ping"

# 1. 定义 Handler（业务逻辑接口）
class MyServiceHandler:
    """业务逻辑 Handler"""
    
    async def hello(self, req: my_service_pb2.HelloRequest, ctx) -> my_service_pb2.HelloResponse:
        """处理 Hello 请求"""
        log("info", f"Received Hello request: {req.name}")
        return my_service_pb2.HelloResponse(message=f"Hello, {req.name}!")
    
    async def ping(self, req: my_service_pb2.PingRequest, ctx) -> my_service_pb2.PingResponse:
        """处理 Ping 请求"""
        log("info", "Received Ping request")
        return my_service_pb2.PingResponse(pong=True)
    

# 2. 定义 Dispatcher（消息路由）
class MyServiceDispatcher:
    """消息路由 Dispatcher"""
    
    async def dispatch(self, workload, route_key: str, payload: bytes, ctx) -> bytes:
        """
        Dispatcher 的 dispatch 方法
        
        参数：
        - workload: MyServiceWorkload 实例（包含 handler 属性）
        - route_key: 路由键字符串
        - payload: 请求 protobuf bytes
        - ctx: Context 对象
        
        返回：响应 protobuf bytes
        """
        if route_key == ROUTE_KEY_HELLO:
            # 反序列化请求
            req = my_service_pb2.HelloRequest.FromString(payload)
            # 调用 Handler 的业务逻辑方法
            resp = await workload.handler.hello(req, ctx)
            # 序列化响应
            return resp.SerializeToString()
        
        elif route_key == ROUTE_KEY_PING:
            req = my_service_pb2.PingRequest.FromString(payload)
            resp = await workload.handler.ping(req, ctx)
            return resp.SerializeToString()
        
        else:
            raise RuntimeError(f"Unknown route_key: {route_key}")

# 3. 定义 Workload（生命周期管理）
@dataclass
class MyServiceWorkload:
    """Workload：组合 Handler 和 Dispatcher"""
    handler: MyServiceHandler  # Handler 实例
    _dispatcher: Optional[MyServiceDispatcher] = None
    
    def __post_init__(self):
        """初始化 Dispatcher 实例"""
        if self._dispatcher is None:
            self._dispatcher = MyServiceDispatcher()
    
    def get_dispatcher(self):
        """返回与此 Workload 关联的 Dispatcher"""
        return self._dispatcher
    
    async def on_start(self, ctx) -> None:
        """生命周期钩子：Actor 启动时调用"""
        # 如果 Handler 实现了 on_start，则调用它
        if hasattr(self.handler, "on_start"):
            await self.handler.on_start(ctx)
    
    async def on_stop(self, ctx) -> None:
        """生命周期钩子：Actor 停止时调用"""
        # 如果 Handler 实现了 on_stop，则调用它
        if hasattr(self.handler, "on_stop"):
            await self.handler.on_stop(ctx)

async def main():
    # 1. 加载配置
    system = await ActrSystem.from_toml("Actr.toml")
    
    # 2. 创建 Handler（实现业务逻辑）
    handler = MyServiceHandler()
    
    # 3. 创建 Workload（自动创建 Dispatcher）
    workload = MyServiceWorkload(handler)
    
    # 4. 附加 Workload 到系统（Dispatcher 通过 get_dispatcher() 自动获取）
    node = system.attach(workload)
    
    # 5. 启动节点
    actr_ref = await node.start()
    # actor_id() 返回 Python ActrId protobuf 对象（generated.actr_pb2.ActrId）
    actor_id = actr_ref.actor_id()
    log("info", f"Actor ID: {actor_id}")
    
    # 6. 等待 Ctrl+C 并关闭
    await actr_ref.wait_for_ctrl_c_and_shutdown()

if __name__ == "__main__":
    asyncio.run(main())
```

**说明**：
- **Handler** (`MyServiceHandler`) 负责业务逻辑实现，定义具体的 RPC 处理方法
- **Dispatcher** (`MyServiceDispatcher`) 负责消息路由，根据 `route_key` 调用 Handler 的相应方法（通过 `workload.handler.method()`）
- **Workload** (`MyServiceWorkload`) 负责生命周期管理，组合 Handler 和 Dispatcher
- Workload 通过 `get_dispatcher()` 方法返回 Dispatcher 对象
- Dispatcher 的 `dispatch` 方法接收四个参数：`(workload, route_key, payload, ctx)`
- 这种方式实现了职责分离，符合 Rust 的 trait 设计

### DataStream 示例：Client-Server 模式（Handler + Dispatcher + Workload 三层架构）

#### Server 端（发送 DataStream）

```python
import asyncio
from actr_runtime_py import ActrSystem, PayloadType, DataStream, log
from generated import actr_pb2
from generated import package_pb2, actr_pb2
from generated import data_stream_multi_pb2 as pb2
from generated.data_stream_multi_service_actor import (
    DataStreamMultiServiceHandler,
    DataStreamMultiServiceWorkload
)

# 1. 定义 Handler（业务逻辑）
class ServerHandler(DataStreamMultiServiceHandler):
    """Server Handler：实现业务逻辑"""
    
    def __init__(self) -> None:
        self._active = 0
        log("info", "ServerHandler initialized")

    async def start_stream(self, req: pb2.StartStreamRequest, ctx) -> pb2.StartStreamResponse:
        """处理 StartStream RPC 请求"""
        caller = ctx.caller_id()
        if caller is None:
            raise RuntimeError("No caller_id in ctx")
        # caller 是 Python ActrId protobuf 对象（generated.actr_pb2.ActrId）

        self._active += 1
        session_id = self._active
        log("info", f"start_stream: client_id={req.client_id}, stream_id={req.stream_id}, message_count={req.message_count}, session_id={session_id}, caller={caller}")

        async def _run() -> None:
            """后台任务：发送 DataStream 消息"""
            log("info", f"[server] Starting stream task: client_id={req.client_id}, stream_id={req.stream_id}, message_count={req.message_count}, caller={caller}")
            for i in range(1, int(req.message_count) + 1):
                msg = f"Message #{i} for client {req.client_id} (session {session_id})"
                log("info", f"[server] Sending message {i}/{req.message_count} to client {req.client_id}, caller={caller}, stream_id={req.stream_id}")
                time.sleep()
                # 创建 DataStream protobuf 对象
                data_stream_pb = package_pb2.DataStream(
                    stream_id=req.stream_id,
                    sequence=i,
                    payload=msg.encode("utf-8"),
                )
                # 转换为 wrapper 对象
                data_stream_wrapper = DataStream(data_stream_pb)
                
                # 发送 DataStream（需要将 caller 转换为 Dest）
                from actr_runtime_py import Dest
                target = Dest.actor(caller)  # caller 是 Python ActrId protobuf 对象（从 ctx.caller_id() 返回）
                result = await ctx.send_data_stream(
                    target,  # Dest wrapper 对象
                    data_stream_wrapper,  # DataStream wrapper 对象
                )
                if result.is_ok():
                    log("info", f"[server] ✅ Successfully sent message {i}/{req.message_count}")
                else:
                    error = result.unwrap_err()
                    log("error", f"[server] ❌ Failed to send message {i}/{req.message_count}: {error}")
                    raise RuntimeError(f"Failed to send message {i}/{req.message_count}: {error}")
                
                if i < req.message_count:
                    await asyncio.sleep(1.0)
            
            log("info", f"[server] Completed streaming {req.message_count} messages to client {req.client_id}")

        # 启动后台任务
        asyncio.create_task(_run())

        return pb2.StartStreamResponse(
            accepted=True,
            stream_id=req.stream_id,
            message=f"Streaming {req.message_count} messages to {req.client_id}",
        )

    async def ack_completion(self, req: pb2.AckCompletionRequest, ctx) -> pb2.AckCompletionResponse:
        """处理 AckCompletion RPC 请求"""
        caller = ctx.caller_id()
        log("info", f"ack_completion: client {caller} received {req.messages_received} messages")
        return pb2.AckCompletionResponse(
            acknowledged=True,
            message=f"Acknowledged: client received {req.messages_received} messages",
        )


async def main() -> int:
    # 2. 创建 Handler（实现业务逻辑）
    handler = ServerHandler()
    
    # 3. 创建 Workload（自动创建 Dispatcher）
    workload = DataStreamMultiServiceWorkload(handler)
    # DataStreamMultiServiceWorkload 会自动创建 DataStreamMultiServiceDispatcher 实例
    # Dispatcher 通过 get_dispatcher() 方法返回，负责消息路由
    
    # 4. 加载配置并附加 Workload
    system = await ActrSystem.from_toml("server/Actr.toml")
    node = system.attach(workload)
    
    # 5. 启动节点
    actr_ref = await node.start()
    log("info", f"✅ Python Server started! Actor ID: {actr_ref.actor_id()}")
    
    # 6. 等待 Ctrl+C 并关闭
    await actr_ref.wait_for_ctrl_c_and_shutdown()
    log("info", "Server shutting down...")
    return 0

if __name__ == "__main__":
    raise SystemExit(asyncio.run(main()))
```

**说明**：
- **Handler** (`ServerHandler`) 继承自 `DataStreamMultiServiceHandler`，实现业务逻辑方法（`start_stream`, `ack_completion`）
- **Workload** (`DataStreamMultiServiceWorkload`) 自动创建 `DataStreamMultiServiceDispatcher` 实例
- **Dispatcher** (`DataStreamMultiServiceDispatcher`) 通过 `get_dispatcher()` 方法返回，负责消息路由
- `caller` 是 Python `ActrId` protobuf 对象（从 `ctx.caller_id()` 返回），需要转换为 `Dest` 用于 `send_data_stream`（使用 `Dest.actor(caller)`）
- `DataStream` 需要从 protobuf 对象转换为 wrapper 对象：`DataStream(data_stream_pb)`

#### Client 端（接收 DataStream - 使用 register_stream）

```python
import asyncio
from actr_runtime_py import ActrSystem, PayloadType, log
from generated import package_pb2, actr_pb2
from generated import data_stream_multi_pb2 as pb2

class ClientWorkload:
    """Client Workload（不需要处理 RPC 请求，所以不需要 Dispatcher）"""
    def __init__(self, client_id: str, expected: int):
        self.client_id = client_id
        self.expected = expected
        self.received_count = 0

    async def on_start(self, ctx) -> None:
        # 1. 发现服务器（使用 Python ActrType protobuf 对象）
        actr_type = actr_pb2.ActrType(manufacturer="acme", name="DataStreamMultiService")
        server_id = await ctx.discover_route_candidate(actr_type)
        log("info", f"Discovered server: {server_id}")

        stream_id = f"{self.client_id}-stream"
        
        # 2. 注册 DataStream 回调
        async def stream_callback(data_stream: package_pb2.DataStream, sender_id: actr_pb2.ActrId) -> None:
            # data_stream 和 sender_id 都是 protobuf 对象，直接使用
            self.received_count += 1
            text = data_stream.payload.decode("utf-8", errors="replace")
            log("info", f"📥 Received message {self.received_count}/{self.expected} from {sender_id}: {text}")

        await ctx.register_stream(stream_id, stream_callback)

        # 3. 发送启动请求（使用 call_raw 和自动反序列化）
        req = pb2.StartStreamRequest(
            client_id=self.client_id,
            message_count=self.expected,
            stream_id=stream_id,
        )
        # server_id 是 Python ActrId protobuf 对象，需要转换为 Dest
        from actr_runtime_py import Dest
        target = Dest.actor(server_id)
        result = await ctx.call_raw(
            target,
            "data_stream_multi.DataStreamMultiService.StartStream",
            req.SerializeToString(),
            timeout_ms=30_000,
            payload_type=PayloadType.RpcReliable,
            response_type=pb2.StartStreamResponse  # 自动反序列化
        )
        if not result.is_ok():
            error = result.unwrap_err()
            log("error", f"Failed to send StartStreamRequest: {error}")
            raise RuntimeError(f"Failed to send StartStreamRequest: {error}")
        
        # result.unwrap() 返回 StartStreamResponse 对象（已自动反序列化）
        response = result.unwrap()
        log("info", f"StartStreamRequest sent successfully, response: {response.message}")

        # 4. 等待所有消息接收完成
        while self.received_count < self.expected:
            await asyncio.sleep(0.1)

        # 5. 清理
        await ctx.unregister_stream(stream_id)
        log("info", f"Completed! Received {self.received_count} messages")

    async def on_stop(self, ctx) -> None:
        log("info", f"Client {self.client_id} stopped")

async def main():
    system = await ActrSystem.from_toml("client/Actr.toml")
    workload = ClientWorkload("client-1", 5)
    node = system.attach(workload)
    actr_ref = await node.start()
    log("info", f"Client started: {actr_ref.actor_id()}")
    # on_start 会等待所有消息接收完成，然后退出
    await asyncio.sleep(0.1)  # 等待 on_start 完成

if __name__ == "__main__":
    asyncio.run(main())
```

**说明**：
- Client 端不需要处理 RPC 请求，所以不需要实现 `get_dispatcher()` 方法
- 如果 Workload 没有实现 `get_dispatcher()`，系统会使用默认行为（向后兼容）
- `discover_route_candidate` 返回 Python `ActrId` protobuf 对象，需要转换为 `Dest` 用于 `call_raw`
- `call_raw` 使用 `response_type` 参数自动反序列化响应
- 使用 `Dest.actor(server_id)` 创建远程 Actor 目标

---

## 问题

1. rust 没有检测断网通知事件；webrtc 通过 peer connection state change 事件触发ICE Restart ; websocket(信令) 通过心跳触发是否重连； 是否需要暴露ice restart 、ws_reconnect 方法 给到客户端，检查到断网后调用方法进行重连。
