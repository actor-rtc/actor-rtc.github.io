# Actr WebRTC 配置指南

Actr 提供三种 WebRTC 策略：`no_config`（默认）、`ephemeral_ports`（推荐）、`muxed_port`（复用）。

> **术语**：`host`（内网直连）、`srflx`（公网映射）、`relay`（TURN 中继）

---

## 1. 策略模式说明 (Strategy Overview)

### 1.1 No Config (默认模式)

这是**最简单**的模式，适用于**无需固定端口** 由系统随机分配端口的场景。

*   **配置示例**：
    ```toml
    [system.webrtc]
    stun_urls = ["stun:stun.example.com:3478"]
    turn_urls = ["turn:turn.example.com:3478"]
    ```
*   **参数说明**：
    无。

*   **原理**：
    Actr 使用webrtc库的默认实现，只需要填入stun和turn服务器地址即可。
*   **优点**：
    *   只需要基础配置即可
*   **缺点**：
    *   可能无法直连，通常需要依赖中继服务。
*   **适用场景**：
    *   移动端 App。
    *   桌面客户端。
    *   浏览器

### 1.2 Ephemeral Ports (临时端口段模式)

这是 WebRTC 的**标准工作模式**，也是**连接质量最好**的模式。

*   **配置示例**：
    ```toml
    [system.webrtc.ephemeral_ports]
    udp_ports = "50000-50100"
    public_ips = ["203.0.113.10"] # NAT后必填
    
    [system.webrtc]
    turn_urls = ["turn:turn.example.com:3478"]
    stun_urls = ["stun:stun.example.com:3478"]
    ice_relay_acceptance_min_wait = 2000 # 服务端直连优先
    ```
*   **参数说明**：
    *   `udp_ports`: (必填) 指定 Actr 可用于分配的 UDP 端口范围，格式为 `"MIN-MAX"`。范围大小建议至少 100 个。
    *   `public_ips`: (可选) 如果服务器在 NAT 后，必须填入公网 IP，格式 `["PUBLIC"]`。如果服务器直接拥有公网 IP，可不填。
*   **原理**：
    Actr 为每个客户端连接分配一个**独立**的 UDP 端口（从配置范围中选择）。
*   **优点**：
    *   **连接质量最好**：独立端口无干扰，传输效率最高。
    *   **灵活的部署模式**：既支持 **Host 内网连接**（局域网高速互通），也支持 **公网连接**（通过映射公网 IP 供外网访问）。
*   **缺点**：
    *   **防火墙配置繁琐**：需要在防火墙/安全组开放较大的端口范围（如 100 个端口）。
*   **适用场景**：
    *   拥有公网 IP 的服务端。
    *   可以配置 NAT 端口段映射的网关。

### 1.3 Muxed Port (单端口复用模式)

这是为了应对**严格防火墙**环境而设计的特殊模式。

*   **配置示例**：
    ```toml
    [system.webrtc.muxed_port]
    udp_port = "50000"
    public_ips = ["203.0.113.10"] 
    
    [system.webrtc]
    turn_urls = ["turn:turn.example.com:3478"] 
    ice_relay_acceptance_min_wait = 100
    ```
*   **参数说明**：
    *   `udp_port`: (必填) 指定唯一的 UDP 端口号，格式为字符串 `"PORT"`。
    *   `public_ips`: (可选) 单端口模式不支持 STUN，因此**强烈建议**配置此项以直接告知对方公网地址。如不配置，流量将完全通过 TURN 中继。
*   **原理**：
    所有客户端连接**共用同一个** UDP 端口。
*   **优点**：
    *   **端口管理简单**：只需在防火墙开放 1 个端口。
*   **缺点**：
    *   **不支持 STUN**：无法自动探测公网 IP（必须手动配置 `public_ips` 或走中继）。
    *   **内网通信可能绕路**：由于通常只配置公网 IP 且不支持 STUN，局域网内的设备可能被迫经过外网接口或中继进行通信。
*   **适用场景**：
    *   严格限制开放端口的企业内网。
    *   Docker/K8s 容器环境（减少端口映射数量）。

---

## 2. 快速选择 (Quick Select)

| 你的场景                     | 推荐策略          | 关键配置                     |
| ---------------------------- | ----------------- | ---------------------------- |
| **移动端 / 客户端**          | `no_config`       | 无需配置                     |
| **服务端（有公网 IP）**      | `ephemeral_ports` | 端口段 + STUN/TURN           |
| **服务端（NAT 后）**         | `ephemeral_ports` | 端口段 + `public_ips` + TURN |
| **严格防火墙（单端口）**     | `muxed_port`      | 单端口 + TURN                |
| **严格防火墙+NAT（单端口）** | `muxed_port`      | 单端口 + `public_ips` + TURN |

---

## 3. 配置模板

根据你的场景，直接复制以下配置。

### 场景 A：移动端 / 客户端

无需配置特定的端口策略，但**建议配置** STUN/TURN 以确保连通性。

```toml
[system.webrtc]
stun_urls = ["stun:stun.example.com:3478"]
turn_urls = ["turn:turn.example.com:3478"]
ice_relay_acceptance_min_wait = 100         # 客户端推荐速度优先
```

---

### 场景 B：服务端（有公网 IP）

机器直接拥有公网 IP，无 NAT。

```toml
[system.webrtc.ephemeral_ports]
udp_ports = "50000-50100"

[system.webrtc]
stun_urls = ["stun:stun.example.com:3478"]  # 帮助对方发现公网 IP
turn_urls = ["turn:turn.example.com:3478"]
ice_relay_acceptance_min_wait = 2000        # 直连优先
```

---

### 场景 C：服务端（NAT 后）

机器使用内网 IP，通过路由器的公网 IP 访问。

```toml
[system.webrtc.ephemeral_ports]
udp_ports = "50000-50100"
public_ips = ["203.0.113.10"]  # 必须配置：["公网IP"]

[system.webrtc]
stun_urls = ["stun:stun.example.com:3478"]
turn_urls = ["turn:turn.example.com:3478"]  # TURN 兜底
ice_relay_acceptance_min_wait = 2000        # 直连优先
```

> **要求**：必须在路由器/防火墙上配置 **静态端口映射**（映射整个端口段）。

---

### 场景 D：严格防火墙（只能单端口）

只能开放单个 UDP 端口，无法开放端口段。

```toml
[system.webrtc.muxed_port]
udp_port = "50000"
# 如果在 NAT 后，取消注释下行并配置 IP：
# public_ips = ["203.0.113.10/192.168.1.10"]

[system.webrtc]
turn_urls = ["turn:turn.example.com:3478"]  # 必须配置 TURN
ice_relay_acceptance_min_wait = 100         # 速度优先
```

---

## 4. 常见问题 (FAQ)

### Q: NAT 后配置了 `public_ips` 还需要 STUN 吗？
**不需要，也不能要**。`public_ips` 显式指定了公网 IP，STUN 会产生冲突（webrtc-rs 限制）。

### Q: 为什么 `muxed_port` 必须配置 TURN？
因为单端口复用模式不支持 STUN，无法自动发现公网地址。如果没有 TURN，一旦直连失败就无法通信。

### Q: ICE 等待时间怎么设？
- **直连优先**（服务端）：`2000` ms。等待更长时间以建立直连，节省 TURN 流量。
- **速度优先**（客户端/单端口）：`100` ms。快速建立连接，不惜使用 TURN。

### Q: 怎么验证配置是否成功？
1. **检查 IP**：`curl ifconfig.me`（公网）和 `ip addr`（内网）确保填写的 IP 正确。
2. **检查端口**：确保防火墙/云厂商安全组已开放对应 UDP 端口（段）。

---

## 5. 故障排查

| 现象              | 可能原因                                    | 解决                                  |
| ----------------- | ------------------------------------------- | ------------------------------------- |
| **无法直连**      | 端口未开放、`public_ips` 写错、未做端口映射 | 检查防火墙和映射配置                  |
| **连接很慢**      | ICE 等待时间太长、直连超时                  | 改为 `100` ms，或检查直连配置         |
| **流量全走 TURN** | 直连一直失败                                | 使用 `ephemeral_ports` 并检查端口映射 |

---

## 总结

- 能用 **场景 B/C** (`ephemeral_ports`) 就尽量用，效果最好。
- 实在受限再用 **场景 D** (`muxed_port`)。
- 移动端直接用 **场景 A**。