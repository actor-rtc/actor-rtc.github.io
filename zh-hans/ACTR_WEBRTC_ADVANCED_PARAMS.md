# Actr WebRTC 高级参数：提升连接速度与直连率

本文档面向 actr 使用者与开发者，介绍 Actr 新增的 WebRTC 高级参数配置，旨在：

- **提升连接建立速度**：通过 ICE 候选等待时间优化，更快选中可用候选，减少无效等待
- **提升直连率**：优先使用 Host/Srflx 候选，减少 TURN 中继依赖，节省带宽成本
- **适配多种部署形态**：移动端（Android/iOS/浏览器）、公网节点（云服务器/NAT）、集群内互联（内网直连）

> 术语说明：本文将 WebRTC/ICE 的候选类型简称为 `host`（内网直连）、`srflx`（STUN/公网映射）、`relay`（TURN 中继）。

---

## 目录

- [Actr WebRTC 高级参数：提升连接速度与直连率](#actr-webrtc-高级参数提升连接速度与直连率)
  - [目录](#目录)
  - [1. 设计要点（为什么需要这些参数）](#1-设计要点为什么需要这些参数)
  - [2. 高级参数配置](#2-高级参数配置)
    - [2.1 完整配置示例](#21-完整配置示例)
    - [2.2 `udp_ports`：端口策略](#22-udp_ports端口策略)
      - [端口范围（推荐）](#端口范围推荐)
      - [单端口（可选）](#单端口可选)
    - [2.3 `public_ips`：NAT 1:1 映射](#23-public_ipsnat-11-映射)
    - [2.4 ICE 候选等待时间](#24-ice-候选等待时间)
  - [3. 推荐组合与部署建议](#3-推荐组合与部署建议)
    - [3.1 移动端（Android/iOS/浏览器）](#31-移动端androidios浏览器)
    - [3.2 按网络拓扑选择配置](#32-按网络拓扑选择配置)
      - [3.2.1 节点直接拥有公网 IP（无 NAT）](#321-节点直接拥有公网-ip无-nat)
      - [3.2.2 节点在 NAT 后（推荐：静态端口映射）](#322-节点在-nat-后推荐静态端口映射)
      - [3.2.3 节点在 NAT 后（备选：STUN 动态发现）](#323-节点在-nat-后备选stun-动态发现)
    - [3.3 配置选择决策树](#33-配置选择决策树)
  - [4. 实现方案](#4-实现方案)
    - [4.1 配置层实现](#41-配置层实现)
    - [4.2 运行时实现](#42-运行时实现)
    - [4.3 角色协商优化](#43-角色协商优化)
  - [5. 当前限制](#5-当前限制)
    - [5.1 单端口模式（UDPMux）限制](#51-单端口模式udpmux限制)

---

## 1. 设计要点（为什么需要这些参数）

WebRTC 建链的关键在于 ICE（候选收集 + 连通性检测）。默认策略往往“更通用但更慢”，在以下场景容易表现不佳：

- **移动端网络频繁切换**：需要更快收敛到可用候选
- **公网节点在 NAT/LB 后**：对端需要看到"公网可达"候选，否则只能走 TURN
- **集群内部互联**：应优先使用内网 host 候选，避免绕公网

因此我们引入一组高级参数：
- `public_ips`：用于 NAT 1:1 映射（让对端看到公网可达 IP）
- `udp_ports`：端口策略（单端口或端口范围）
- ICE 候选等待时间：控制 host/srflx/prflx/relay 的接受时机

---

## 2. 高级参数配置

配置位置：`[system.webrtc.advanced]`

### 2.1 完整配置示例

```toml
[system.webrtc.advanced]
# NAT 1:1 公网 IP 映射
public_ips = ["203.0.113.10/192.168.1.10"]

# 端口策略（二选一）
udp_ports = "50000-50100"  # 端口范围（推荐）
# udp_ports = "50000"      # 单端口

# ICE 候选等待时间（单位：毫秒）
ice_host_acceptance_min_wait = 0
ice_srflx_acceptance_min_wait = 20
ice_prflx_acceptance_min_wait = 40
ice_relay_acceptance_min_wait = 100
```

---

### 2.2 `udp_ports`：端口策略

#### 端口范围（推荐）

```toml
udp_ports = "50000-50100"
```

**特点**：
- ✅ 支持完整的候选类型（host/srflx/relay）
- ✅ 自动适应有/无内网场景
- ⚠️ 需要开放端口段

**适用场景**：推荐用于大多数部署场景。

#### 单端口（可选）

```toml
udp_ports = "50000"
```

**特点**：
- ✅ 运维最简单，只需一个 UDP 端口
- ⚠️ **UDPMux 模式禁用 `srflx` 候选收集**（webrtc-ice 限制）
- ⚠️ 会替换内网 host 候选（不适合混合网络场景）

**适用场景**：节点独占公网 IP，无内网互联需求。

---

### 2.3 `public_ips`：NAT 1:1 映射

**作用**：为 NAT 后的节点配置公网 IP 映射，生成公网可达候选。

**配置格式**：

```toml
# 显式映射（推荐，适用于多网卡）
public_ips = ["203.0.113.10/192.168.1.10"]

# 自动匹配本地 IP
public_ips = ["203.0.113.10"]
```

**关键点**：
- 需要配合 NAT 静态端口映射（公网端口 = 内网端口）
- 支持对称型 NAT 直连
- 不能与 STUN 同时配置（webrtc-rs 限制）

---

### 2.4 ICE 候选等待时间

**作用**：控制 ICE 在连接建立过程中，多快接受某类候选。

**参数**：
- `ice_host_acceptance_min_wait`：内网直连等待时间（默认 0ms）
- `ice_srflx_acceptance_min_wait`：公网映射等待时间（推荐 20ms）
- `ice_prflx_acceptance_min_wait`：对端反射等待时间（推荐 40ms）
- `ice_relay_acceptance_min_wait`：TURN 中继等待时间（推荐 100ms）

**调优策略**：

```toml
# 直连优先（节省 TURN 带宽）
ice_relay_acceptance_min_wait = 2000

# 速度优先（更快建立连接）
ice_relay_acceptance_min_wait = 100
```

---

## 3. 推荐组合与部署建议

> 本节按**网络拓扑**和**NAT类型**组织，提供清晰的配置方案。

### 3.1 移动端（Android/iOS/浏览器）

移动端通常处于运营商 NAT 后，不承担被动连入职责，**无需配置高级参数**，使用默认值即可。

---

### 3.2 按网络拓扑选择配置

#### 3.2.1 节点直接拥有公网 IP（无 NAT）

**适用场景**：独立服务器、裸金属服务器，网卡直接绑定公网 IP。

**配置**：

```toml
[system.webrtc.advanced]
# 端口策略（二选一）
udp_ports = "50000-50100"  # 端口范围（推荐）
# udp_ports = "50000"      # 单端口（运维更简单）

# 无需配置 public_ips
```

**候选类型**：
- ✅ `typ host 203.0.113.10:<port>`（公网 IP，直接可达）
- ✅ `typ relay`（TURN 兜底，可选）

**端口选择**：
- **端口范围**：功能完整，支持更多候选类型
- **单端口**：运维最简单，只需开放一个端口

**优势**：配置最简单，直连成功率最高。

---

#### 3.2.2 节点在 NAT 后（推荐：静态端口映射）

**适用场景**：
- ✅ 云主机（NAT 后）
- ✅ 支持对称型 NAT 直连
- ✅ **自动适应有/无内网场景**（通用方案）

**配置**：

```toml
[system.webrtc.advanced]
# 端口策略（二选一）
udp_ports = "50000-50100"  # 端口范围（推荐，支持 typ srflx）
# udp_ports = "50000"      # 单端口（运维更简单，不支持 typ srflx）

public_ips = ["203.0.113.10/192.168.1.10"]  # 公网IP/内网IP

ice_host_acceptance_min_wait = 0
ice_srflx_acceptance_min_wait = 20
ice_prflx_acceptance_min_wait = 40
ice_relay_acceptance_min_wait = 100
```

**网络配置**：
- 公网 IP：`203.0.113.10`（可多节点共享，通过端口区分）
- 防火墙：开放对应的 UDP 端口（入站 + 出站）
- **NAT 静态端口映射**（关键）：
  - 端口范围：`203.0.113.10:50000-50100` → `192.168.1.10:50000-50100`
  - 单端口：`203.0.113.10:50000` → `192.168.1.10:50000`
  - 必须同端口转发（公网端口 = 内网端口）

**对称型 NAT 直连原理**：
- **问题**：STUN 发现的端口只对 STUN 服务器有效，对端连接时 NAT 会分配新端口
- **解决**：静态端口映射 + `public_ips`，WebRTC 生成固定的公网候选，NAT 直接转发

**候选类型（自动生成）**：
- ✅ `typ host 192.168.1.10:<port>`（内网，**仅当节点有内网 IP 时生成**）
- ✅ `typ srflx 203.0.113.10:<port>`（公网，对称型 NAT 可直连，**仅端口范围模式**）
- ✅ `typ relay`（TURN 兜底）

**连接行为（自动选择）**：
- **有内网场景**：节点间优先 `typ host`（内网直连），外部客户端用 `typ srflx`
- **无内网场景**：所有连接用 `typ srflx`（公网）

**端口选择对比**：
- **端口范围**：
  - ✅ 支持 `typ srflx` 候选（更高直连成功率）
  - ✅ 自动适应有/无内网场景
  - ⚠️ 需要开放端口段
- **单端口**：
  - ✅ 运维最简单（只需一个端口）
  - ⚠️ 不支持 `typ srflx`（UDPMux 限制）
  - ⚠️ 会替换内网 host 候选（不适合混合网络场景）
  - ✅ 适用于节点独占公网 IP 且无内网互联需求

**为什么配置相同但行为不同？**
- WebRTC 自动检测本地网卡 IP
- 有内网 IP 自动生成 `typ host` 候选
- ICE 自动优先选择内网候选（延迟更低）

---

#### 3.2.3 节点在 NAT 后（备选：STUN 动态发现）

**适用场景**：
- ⚠️ 仅适用于**非对称型 NAT**（Full Cone / Restricted Cone / Port Restricted Cone）
- 无法配置静态端口映射

**配置**：

```toml
[system.webrtc]
stun_urls = ["stun:stun.l.google.com:19302"]

[system.webrtc.advanced]
udp_ports = "50000-50100"  # 必须使用端口范围（STUN 不支持单端口）
```

**候选类型**：
- ✅ `typ host`（内网）
- ✅ `typ srflx`（STUN 动态发现，**对称型 NAT 下无法直连**）
- ✅ `typ relay`（TURN 兜底）

**限制**：
- ❌ 不支持对称型 NAT 直连（必须回退到 TURN）
- ❌ 不支持单端口（STUN 需要端口范围）
- 如果是对称型 NAT，请使用 3.2.2

---

### 3.3 配置选择决策树

```
第一步：选择端口策略（根据防火墙/运维需求）
├─ 端口范围（如 50000-50100）→ 功能更完整，支持 typ srflx
└─ 单端口（如 50000）→ 运维更简单，仅一个端口

第二步：根据网络环境选择配置
├─ 节点直接拥有公网 IP（无 NAT）
│  └─ 3.2.1（端口范围或单端口都可用）
│
└─ 节点在 NAT 后面
   ├─ 能否配置静态 NAT 端口映射？
   │  ├─ 是 → 3.2.2（端口范围或单端口都可用）✅ 推荐
   │  │     • 端口范围：支持 typ srflx，自动适应有/无内网
   │  │     • 单端口：运维简单，仅适用于独占公网 IP
   │  │
   │  └─ 否（无法配置静态端口映射）
   │     ├─ NAT 是非对称型？
   │     │  ├─ 是 → 3.2.3（STUN，必须端口范围）
   │     │  └─ 否（对称型）→ 必须依赖 TURN 中继 ⚠️
   │     └─
   └─

关键点：
• 端口选择：端口范围功能更完整，单端口运维更简单
• 无 NAT 场景：端口范围和单端口都可用（3.2.1）
• 有 NAT + 静态映射：端口范围和单端口都可用（3.2.2）
• 对称型 NAT 直连 → 必须 public_ips + 静态端口映射（3.2.2）
• 非对称型 NAT + STUN → 必须使用端口范围（3.2.3）
• 有内网连通性 → WebRTC 自动优先走内网（typ host）

NAT 类型检测：
• STUN 工具：stunclient、nattest
• 在线测试：https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/
• 简单判断：STUN 能发现公网地址但对端无法连接 → 可能是对称型 NAT
```

---

## 4. 实现方案

本节描述如何在代码中应用这些高级参数。

**⚠️ 重要说明：参数应用策略**

高级参数根据其用途分为两类：

1. **ICE 候选等待时间**：对所有连接生效（Offerer + Answerer）
   - 控制 ICE 候选选择优先级
   - 影响连接建立速度和直连率
   - 双方都需要配置

2. **UDP 端口策略 + NAT 1:1 映射**：仅对 Answerer 生效
   - 公网部署的服务端节点
   - 需要让客户端看到"公网可达"候选
   - 客户端（Offerer）不需要配置

**角色判定优化**：

如果配置了 `udp_ports` 或 `public_ips`，在角色协商时会**优先判定为 Answerer**，确保高级参数能够生效。

---

### 4.1 配置层实现

**配置结构**：

```rust
// actr/crates/config/src/config.rs
pub struct WebRtcAdvancedConfig {
    pub udp_ports: UdpPorts,           // 端口策略
    pub public_ips: Vec<String>,       // NAT 1:1 映射
    pub ice_host_acceptance_min_wait: u64,
    pub ice_srflx_acceptance_min_wait: u64,
    pub ice_prflx_acceptance_min_wait: u64,
    pub ice_relay_acceptance_min_wait: u64,
}

pub enum UdpPorts {
    Single(u16),              // 单端口："50000"
    Range(u16, u16),          // 端口范围："50000-50100"
}
```

**配置解析**：

```toml
[system.webrtc.advanced]
udp_ports = "50000-50100"  # 或 "50000"
public_ips = ["203.0.113.10/192.168.1.10"]
ice_host_acceptance_min_wait = 0
ice_srflx_acceptance_min_wait = 20
ice_prflx_acceptance_min_wait = 40
ice_relay_acceptance_min_wait = 2000
```

---

### 4.2 运行时实现

**核心位置**：`actr/crates/runtime/src/wire/webrtc/negotiator.rs`

**实现架构**：

```rust
impl WebRtcNegotiator {
    /// 创建 PeerConnection（根据角色应用配置）
    pub async fn create_peer_connection(
        &self,
        is_answerer: bool,
    ) -> NetworkResult<RTCPeerConnection> {
        // 1. 创建 MediaEngine（注册编解码器）
        let mut media_engine = MediaEngine::default();
        // ... 注册 VP8, H264, OPUS ...
        
        // 2. 创建 SettingEngine
        let mut setting_engine = SettingEngine::default();
        
        // 3. 应用 ICE 等待时间（所有连接）
        self.apply_ice_wait_times(&mut setting_engine);
        
        // 4. 应用高级参数（仅 Answerer）
        if is_answerer {
            self.apply_answerer_config(&mut setting_engine)?;
        }
        
        // 5. 创建 API 和 PeerConnection
        let api = APIBuilder::new()
            .with_media_engine(media_engine)
            .with_setting_engine(setting_engine)
            .build();
            
        api.new_peer_connection(rtc_config).await
    }
    
    /// 应用 ICE 候选等待时间（所有连接）
    fn apply_ice_wait_times(&self, setting_engine: &mut SettingEngine) {
        let advanced = &self.config.advanced;
        
        setting_engine.set_host_acceptance_min_wait(
            Some(Duration::from_millis(advanced.ice_host_acceptance_min_wait))
        );
        setting_engine.set_srflx_acceptance_min_wait(
            Some(Duration::from_millis(advanced.ice_srflx_acceptance_min_wait))
        );
        setting_engine.set_prflx_acceptance_min_wait(
            Some(Duration::from_millis(advanced.ice_prflx_acceptance_min_wait))
        );
        setting_engine.set_relay_acceptance_min_wait(
            Some(Duration::from_millis(advanced.ice_relay_acceptance_min_wait))
        );
    }
    
    /// 应用 Answerer 专用配置（UDP 端口 + NAT 1:1）
    fn apply_answerer_config(&self, setting_engine: &mut SettingEngine) -> NetworkResult<()> {
        let advanced = &self.config.advanced;
        
        // 应用 UDP 端口策略
        match &advanced.udp_ports {
            UdpPorts::Single(port) => {
                // 单端口模式（需要 UDPMux 缓存，暂未实现）
                tracing::warn!("Single port mode requires UDPMux caching");
            }
            UdpPorts::Range(min, max) => {
                let ephemeral = EphemeralUDP::new(*min, *max)?;
                setting_engine.set_udp_network(UDPNetwork::Ephemeral(ephemeral));
            }
        }
        
        // 应用 NAT 1:1 映射（仅端口范围模式）
        if matches!(&advanced.udp_ports, UdpPorts::Range(..)) && !advanced.public_ips.is_empty() {
            setting_engine.set_nat_1to1_ips(
                advanced.public_ips.clone(),
                RTCIceCandidateType::Srflx,  // Server Reflexive
            );
        }
        
        Ok(())
    }
}
```

**调用示例**：

```rust
// Offerer（主动发起方）
let peer_connection = negotiator.create_peer_connection(false).await?;
// ✅ 应用 ICE 等待时间
// ❌ 不应用 UDP 端口和 NAT 1:1

// Answerer（被动接入方）
let peer_connection = negotiator.create_peer_connection(true).await?;
// ✅ 应用 ICE 等待时间
// ✅ 应用 UDP 端口和 NAT 1:1
```

---

### 4.3 角色协商优化

**优先判定为 Answerer**：

如果节点配置了 `udp_ports` 或 `public_ips`，在角色协商时会优先判定为 Answerer，确保高级参数能够生效。

```rust
// 角色协商逻辑（伪代码）
fn negotiate_role(&self, peer: &ActrId) -> bool {
    // 如果配置了高级参数，优先成为 Answerer
    if self.has_advanced_config() {
        return false;  // is_offerer = false
    }
    
    // 否则使用默认协商逻辑（如 serial number 比较）
    self.local_id.serial_number > peer.serial_number
}

fn has_advanced_config(&self) -> bool {
    let advanced = &self.config.advanced;
    
    // 检查是否配置了 UDP 端口或 public_ips
    !matches!(advanced.udp_ports, UdpPorts::Range(50000, 50100))  // 非默认值
        || !advanced.public_ips.is_empty()
}
```

**优势**：
- 🎯 确保配置了高级参数的节点成为 Answerer
- 🚀 高级参数能够正确应用
- 💡 无需手动指定角色

---

## 5. 当前限制

### 5.1 单端口模式（UDPMux）限制

**技术限制**（webrtc-rs 0.14.0）：

1. **无法使用 srflx 候选**：
   - 单端口模式下，webrtc-rs 会**禁用 STUN 收集**
   - 无法生成 `typ srflx` 候选（Server Reflexive）
   - 只能生成 `typ host` 候选

2. **NAT 1:1 映射的影响**：
   - 如果配置了 `public_ips` 且使用 `Host NAT1:1` 模式
   - 会**替换** host 候选的 IP 地址为公网 IP
   - 可能导致集群内互联失败（丢失内网 IP）

**推荐方案**：

对于需要公网可达的节点，建议使用**端口范围模式**：

```toml
[system.webrtc.advanced]
# ✅ 推荐：端口范围模式
udp_ports = "50000-50100"
public_ips = ["203.0.113.10/192.168.1.10"]  # 生成 srflx 候选

# ❌ 不推荐：单端口模式（无 srflx 候选）
# udp_ports = "50000"
# public_ips = ["203.0.113.10"]  # 会替换 host IP，可能影响内网连接
```

**单端口模式适用场景**：

仅在以下情况考虑单端口模式：
- 防火墙严格限制，只能开放单个端口
- 不需要公网可达性（仅内网使用）
- 可以依赖 TURN 中继进行外部连接