
## 第零章：在学 VPN 之前，你必须先理解的背景知识

### 0.1 企业网络通信的基本问题

假设你在一家公司上班，这家公司有三个办公室：

- 总部在 Dublin
- 分部在 Berlin
- 分部在 Amsterdam

每个办公室内部都有自己的局域网（LAN），员工之间可以互相通信。但是三个办公室之间怎么通信？

**方案一：租专线（Leased Line / MPLS）**

直接找运营商拉一条专用线路，Dublin ↔ Berlin，Dublin ↔ Amsterdam。

- 优点：安全、稳定、带宽有保障
- 缺点：**极其昂贵**。如果你有 50 个分支机构，两两互联需要 50×49/2 = 1225 条专线，成本不可承受

**方案二：直接通过 Internet 通信**

三个办公室都有 Internet 接入，直接通过 Internet 互相发送内部数据。

- 优点：便宜，Internet 接入成本低
- 缺点：**完全不安全**。数据在 Internet 上以明文传输，任何中间节点（ISP 路由器、攻击者）都可以看到、篡改、伪造你的数据

**方案三：VPN — 在 Internet 上建立安全隧道**

这就是 VPN 要解决的问题：**用 Internet 的低成本，实现接近专线的安全性**。

### 0.2 什么是"隧道"（Tunnel）

在网络中，"隧道"不是物理概念，而是一种**逻辑封装技术**。

正常情况下，一个 IP 数据包从 A 发到 B，中间经过的所有路由器都能看到这个包的源 IP、目的 IP、甚至 payload 内容。

"隧道"做的事情是：**把整个原始数据包当作 payload，外面再套一层新的 IP 头**。

这样做的效果：

- 中间路由器只能看到外层 IP 头（隧道两端的公网 IP）
- 原始数据包被完整包裹在里面，中间路由器看不到、也不关心里面的内容
- 就像你把一封信（原始数据）放进另一个信封（隧道封装）里寄出去，邮递员只看外面信封的地址

### 0.3 什么是"加密"（Encryption）

隧道封装解决了"包裹"的问题，但**包裹不等于加密**。

如果你只是把信放进另一个信封，有人拆开外层信封还是能看到里面的内容。

加密做的事情是：**把原始数据变成一串看不懂的乱码，只有拥有正确密钥的接收方才能还原**。

加密需要以下要素：

| 要素 | 说明 |
|------|------|
| **加密算法** | 用什么数学方法加密，例如 AES、3DES、DES |
| **密钥（Key）** | 加密和解密用的"钥匙"，没有密钥就无法还原数据 |
| **密钥交换机制** | 双方如何安全地协商出一个共同的密钥（不能明文发送密钥，否则密钥也会被截获） |

### 0.4 数据安全的三大核心需求（CIA）

在网络安全领域，保护数据需要满足三个需求，缩写为 **CIA**（这个缩写 CCNA 考试会考）：

| 需求 | 英文 | 含义 | 威胁场景 |
|------|------|------|---------|
| **机密性** | Confidentiality | 数据不被未授权的人看到 | 攻击者在 Internet 上截获数据包并读取内容 |
| **完整性** | Integrity | 数据在传输中没有被篡改 | 攻击者截获数据包，修改金额从 100 改成 10000，再发给接收方 |
| **认证** | Authentication | 确认通信对方的真实身份 | 攻击者伪装成你的分公司路由器，骗你发送机密数据 |

**这三个需求就是 VPN 技术要解决的核心问题。** 不同的 VPN 技术解决其中的不同子集。

### 0.5 数据包封装的基础回顾

在理解 GRE 和 IPSec 的封装之前，你必须清楚一个正常 IP 数据包的结构：

```
[ IP Header (20 bytes) | TCP/UDP Header | Application Data ]
```

IP Header 中最关键的字段（对理解 VPN 有关的）：

| 字段 | 说明 |
|------|------|
| **Source IP** | 发送方 IP 地址 |
| **Destination IP** | 接收方 IP 地址 |
| **Protocol** | 上层协议编号：TCP=6, UDP=17, GRE=47, AH=51, ESP=50 |
| **TTL** | 生存时间，每经过一个路由器减 1 |

中间路由器根据 **Destination IP** 查路由表做转发决策，根据 **Protocol** 字段知道上层是什么协议。

---

## 第一章：VPN 总览

### 1.1 VPN 的定义

VPN（Virtual Private Network，虚拟专用网络）是一种技术，**在公共网络（通常是 Internet）上建立逻辑隧道，使两个私有网络之间的通信具备安全性（加密、认证、完整性）**。

"Virtual"：不是真正的物理专线，而是逻辑上的
"Private"：通信内容对外部不可见（加密保护）
"Network"：连接的是网络或终端

### 1.2 VPN 的两大部署场景

这是 CCNA 必须清楚区分的两种 VPN 类型：

#### Site-to-Site VPN（站点到站点）

**场景：** 两个办公室（站点）之间建立永久的 VPN 隧道。

**工作方式：**

1. 每个站点的出口设备（路由器或防火墙）负责 VPN 隧道的建立和维护
2. 站点内部的终端用户完全不知道 VPN 的存在
3. 用户 A 在 Dublin 办公室发一个数据包给 Berlin 办公室的用户 B
4. Dublin 的出口路由器发现目标是 Berlin 的私有网段，自动将数据包加密封装后通过 Internet 发出
5. Berlin 的出口路由器收到后解密解封装，将原始数据包转发给用户 B

**关键特征：**

- 隧道建立在 **网关设备之间**（router-to-router 或 firewall-to-firewall）
- 终端用户 **不需要安装任何 VPN 软件**
- 隧道通常是 **长期存在** 的
- 典型协议：**IPSec**、GRE over IPSec

```
Dublin 办公室                               Berlin 办公室
[PC-A] --→ [Router-A] ====Internet隧道==== [Router-B] --→ [PC-B]
  10.1.1.0/24         203.0.113.1   198.51.100.1        10.2.2.0/24
                          ↑                 ↑
                      VPN 隧道的两个端点
                     （用户完全无感知）
```

#### Remote-Access VPN（远程接入）

**场景：** 单个用户（比如在家办公的员工）通过 Internet 连接到公司内部网络。

**工作方式：**

1. 用户在自己的电脑/手机上安装 VPN 客户端软件（例如 Cisco AnyConnect）
2. 用户启动 VPN 客户端，输入公司 VPN 服务器地址、用户名、密码
3. 客户端与公司 VPN 服务器建立加密隧道
4. 建立后，用户的流量被加密后通过隧道发送到公司网络，就像用户坐在公司办公室一样

**关键特征：**

- 隧道建立在 **用户终端 与 企业 VPN 网关之间**
- 用户 **必须安装 VPN 客户端软件**
- 连接通常是 **临时的**（用完就断）
- 典型协议：**SSL/TLS VPN**（如 Cisco AnyConnect）、IPSec VPN Client

```
员工家中                                    公司总部
[Laptop + VPN Client] ===Internet隧道=== [VPN Server/Firewall] → [内部网络]
                                                                    10.0.0.0/8
```

### 1.3 VPN 协议分类总览

现在你知道了 VPN 的两种部署场景，接下来看实现 VPN 的具体协议/技术：

| 协议/技术 | 工作层级 | 加密 | 认证 | 完整性 | 组播支持 | 典型用途 |
|-----------|---------|------|------|--------|---------|---------|
| **GRE** | L3 | ❌ | ❌ | ❌ | ✅ | 简单隧道封装 |
| **IPSec** | L3 | ✅ | ✅ | ✅ | ❌ | Site-to-Site 加密 VPN |
| **GRE over IPSec** | L3 | ✅ | ✅ | ✅ | ✅ | 需要跑路由协议的加密隧道 |
| **SSL/TLS VPN** | L4-L7 | ✅ | ✅ | ✅ | ❌ | Remote-Access VPN |

接下来我们逐个深入讲解 GRE、IPSec、以及它们的组合。

---

## 第二章：GRE（Generic Routing Encapsulation）

### 2.1 GRE 是什么

GRE（通用路由封装）是一种 **隧道协议**，由 Cisco 最初开发。

它的 IP Protocol Number 是 **47**（就像 TCP 是 6，UDP 是 17，GRE 就是 47）。

GRE 做的事情非常简单：**把一个完整的数据包（称为 Passenger Packet）当作 payload，外面套一个 GRE Header 和一个新的 IP Header（称为 Delivery Header），然后通过正常的 IP 路由发送出去**。

**GRE 只负责封装，不负责加密。**

### 2.2 GRE 封装结构

一个正常的数据包：

```
[ Original IP Header | TCP Header | Data ]
     src: 10.1.1.10
     dst: 10.2.2.20
     protocol: 6 (TCP)
```

经过 GRE 封装后：

```
[ New IP Header | GRE Header | Original IP Header | TCP Header | Data ]
     ↑                ↑              ↑
     Delivery      (4 bytes+)     Passenger（原始包完整保留）
     Header
  src: 203.0.113.1
  dst: 198.51.100.1
  protocol: 47 (GRE)
```

逐层解释：

| 层 | 内容 | 说明 |
|----|------|------|
| **New IP Header** | 20 bytes | 新的 IP 头，Source = 隧道本端公网 IP，Dest = 隧道对端公网 IP，Protocol = 47 |
| **GRE Header** | 4 bytes（基础） | GRE 协议头，包含 Protocol Type 字段（指明被封装的是什么协议，如 0x0800 = IPv4） |
| **Original IP Header** | 20 bytes | 原始数据包的 IP 头，完整保留不修改 |
| **原始 Payload** | 变长 | TCP/UDP Header + Application Data，完整保留 |

**关键认知：中间路由器只看 New IP Header，只关心 203.0.113.1 → 198.51.100.1 这条路径，完全不知道里面还包着一个 10.1.1.10 → 10.2.2.20 的原始包。**

### 2.3 GRE 的工作流程（逐步）

以下是一个完整的 GRE 通信过程：

**网络拓扑：**

```
Dublin LAN                                           Berlin LAN
10.1.1.0/24                                         10.2.2.0/24
    |                                                    |
[PC-A: 10.1.1.10]                              [PC-B: 10.2.2.20]
    |                                                    |
[R1: GigE0/0 = 203.0.113.1]  ← Internet →  [R2: GigE0/0 = 198.51.100.1]
     Tunnel0: 172.16.0.1/30                   Tunnel0: 172.16.0.2/30
```

**Step 1：PC-A 发送原始数据包**

PC-A（10.1.1.10）要发数据给 PC-B（10.2.2.20）。PC-A 发出一个正常数据包：

```
[ IP Header: src=10.1.1.10, dst=10.2.2.20 | TCP | Data ]
```

PC-A 不知道 VPN 的存在，它只知道把去往 10.2.2.0/24 的包发给自己的默认网关 R1。

**Step 2：R1 收到数据包，查路由表**

R1 查路由表，发现去往 10.2.2.0/24 的路由下一跳指向 Tunnel0 接口（这条路由可以是静态路由，也可以是通过隧道跑的动态路由协议学到的）。

**Step 3：R1 执行 GRE 封装**

R1 发现数据包需要从 Tunnel0 接口出去，于是执行 GRE 封装：

1. 保留原始数据包完整不动
2. 在前面加上 GRE Header（4 bytes）
3. 再在最前面加上新的 IP Header：
   - Source IP = 203.0.113.1（Tunnel source，即 R1 的公网 IP）
   - Destination IP = 198.51.100.1（Tunnel destination，即 R2 的公网 IP）
   - Protocol = 47（GRE）

封装后的数据包：

```
[ New IP: src=203.0.113.1, dst=198.51.100.1, proto=47 | GRE | Original IP: src=10.1.1.10, dst=10.2.2.20 | TCP | Data ]
```

**Step 4：封装后的包通过 Internet 正常路由转发**

中间的 Internet 路由器只看外层 IP Header，一路按 198.51.100.1 做路由转发，最终送达 R2。

**Step 5：R2 收到数据包，识别 GRE**

R2 收到一个目的 IP 是自己（198.51.100.1）的数据包，检查 Protocol 字段 = 47，知道这是一个 GRE 封装的包。

**Step 6：R2 执行 GRE 解封装**

R2 剥离：
1. 外层 IP Header
2. GRE Header

还原出原始数据包：

```
[ IP Header: src=10.1.1.10, dst=10.2.2.20 | TCP | Data ]
```

**Step 7：R2 正常转发原始数据包**

R2 查路由表，发现 10.2.2.0/24 是自己直连的网段，将原始数据包转发给 PC-B。

PC-B 收到的数据包和 PC-A 发出的完全一样，整个 GRE 隧道过程对两端 PC 完全透明。

### 2.4 GRE 的关键特性

| 特性 | 详细说明 |
|------|---------|
| **支持组播（Multicast）** | 这是 GRE 最重要的特性。OSPF 用 224.0.0.5/6，EIGRP 用 224.0.0.10，这些组播包可以在 GRE 隧道中正常传输。这意味着你可以在 GRE 隧道两端运行动态路由协议 |
| **支持多协议封装** | GRE 可以封装 IPv4、IPv6、甚至非 IP 协议（如旧的 IPX、AppleTalk）。GRE Header 中的 Protocol Type 字段指明被封装的是什么协议 |
| **不提供任何安全保护** | GRE 不加密、不认证、不做完整性校验。数据在隧道中是明文的。如果有人截获了 GRE 包，他可以剥离外层 Header 看到里面的原始数据 |
| **增加封装开销** | 每个数据包增加至少 24 bytes（20 bytes New IP Header + 4 bytes GRE Header）。如果底层物理链路 MTU 是 1500，那 GRE 隧道内有效 MTU 只有 1500 - 24 = **1476** |
| **点对点隧道** | 一个 GRE Tunnel 接口只能连接一个对端（一个 source 对一个 destination）。如果要连 5 个站点，需要分别建 5 个隧道 |

### 2.5 MTU 问题详解

这个点需要单独讲清楚，因为它经常在考试和实际中造成问题。

**MTU（Maximum Transmission Unit）**：一个网络接口能发送的最大帧的 payload 大小。以太网默认 MTU = 1500 bytes。

**问题：** PC-A 发出一个 1500 bytes 的数据包（已经是最大了），R1 要给它加 GRE 封装，加完后变成 1524 bytes。但 R1 的物理接口 MTU 还是 1500，发不出去。

**解决方法：**

1. **调低 Tunnel 接口的 MTU**：设置 `ip mtu 1476`，这样通过隧道的数据包最大只能是 1476 bytes，加上 24 bytes GRE 开销后正好 1500
2. **调整 TCP MSS**：设置 `ip tcp adjust-mss 1436`，告诉 TCP 连接的最大分段大小更小（1436 = 1476 - 40，其中 40 是 IP Header 20 + TCP Header 20），从源头避免产生过大的包
3. **分片（Fragmentation）**：R1 把封装后的大包拆成小片发送，对端重组。但分片影响性能，不推荐

### 2.6 GRE 配置命令

```cisco
! ====== Router R1 配置 ======

! 进入 Tunnel 接口
R1(config)# interface Tunnel 0

! 给隧道接口分配 IP 地址（这是隧道的逻辑 IP，双方必须同一子网）
R1(config-if)# ip address 172.16.0.1 255.255.255.252

! 指定隧道的源（本地物理接口或 IP）
R1(config-if)# tunnel source GigabitEthernet0/0
! 也可以写成：tunnel source 203.0.113.1（直接写 IP 地址）

! 指定隧道的目的（对端公网 IP）
R1(config-if)# tunnel destination 198.51.100.1

! 隧道模式，默认就是 gre ip，可以不写
R1(config-if)# tunnel mode gre ip

! （可选）调整 MTU
R1(config-if)# ip mtu 1476

! （可选）调整 TCP MSS
R1(config-if)# ip tcp adjust-mss 1436

! 需要配路由，让目标网段走隧道
R1(config)# ip route 10.2.2.0 255.255.255.0 172.16.0.2
! 或者在隧道上跑 OSPF/EIGRP 动态学习路由


! ====== Router R2 配置（镜像配置）======

R2(config)# interface Tunnel 0
R2(config-if)# ip address 172.16.0.2 255.255.255.252
R2(config-if)# tunnel source GigabitEthernet0/0
R2(config-if)# tunnel destination 203.0.113.1
R2(config-if)# ip mtu 1476
R2(config-if)# ip tcp adjust-mss 1436

R2(config)# ip route 10.1.1.0 255.255.255.0 172.16.0.1
```

**配置要点：**

| 要点 | 说明 |
|------|------|
| tunnel source | 必须是本地有效的接口或 IP 地址，通常是连接 Internet 的物理接口 |
| tunnel destination | 对端的公网 IP 地址，必须 **路由可达**（通过底层物理路由） |
| 隧道接口 IP | 两端必须在同一子网（如 172.16.0.1/30 和 172.16.0.2/30） |
| 路由 | 必须有路由指向隧道，否则流量不会走隧道 |

### 2.7 GRE 验证命令

```cisco
R1# show interface Tunnel 0
! 查看隧道接口状态，确认 up/up
! 注意：GRE 隧道接口可能显示 up/up 即使对端不可达
! （因为 GRE 本身没有 keepalive 检测机制，除非手动配置）

R1# show ip route
! 确认去往对端网段的路由是否存在且指向 Tunnel 接口

R1# ping 172.16.0.2 source 172.16.0.1
! 测试隧道连通性
```

---

## 第三章：IPSec（Internet Protocol Security）

### 3.1 IPSec 是什么

IPSec 不是一个单独的协议，而是一个 **安全框架（Framework）**，它是一组协议和算法的集合，共同协作来提供安全的 IP 通信。

IPSec 工作在 **L3（网络层）**。

**IPSec 提供的安全服务：**

| 安全服务 | 英文 | 做什么 | 怎么做 |
|---------|------|--------|--------|
| 机密性 | Confidentiality | 加密数据，防止被窃听 | 使用加密算法（AES, 3DES, DES） |
| 完整性 | Integrity | 保证数据没被篡改 | 使用哈希算法（SHA, MD5）生成数据摘要 |
| 认证 | Authentication | 验证通信对方的身份 | 使用 PSK（预共享密钥）或数字证书 |
| 防重放 | Anti-Replay | 防止攻击者截获数据包后重复发送 | 使用序列号（Sequence Number） |

### 3.2 IPSec 的组件拆解

IPSec 框架由以下组件构成，每个组件负责不同的功能。理解这些组件是理解 IPSec 的基础：

#### 3.2.1 IKE（Internet Key Exchange）— 密钥交换协议

**问题：** 加密需要双方有一把共同的密钥。但你不能把密钥直接通过 Internet 发给对方，因为 Internet 不安全，密钥会被截获。

**IKE 解决的问题：** 在不安全的网络上，安全地协商出一把共同的加密密钥。

IKE 使用 **Diffie-Hellman（DH）算法** 来实现这一点。DH 算法的原理你不需要深入理解（CCNA 不考），你只需要知道：

- DH 允许双方各自生成一部分数据，通过 Internet 交换这些数据后，双方可以各自计算出一把相同的密钥
- 即使攻击者截获了交换的数据，他也无法算出最终的密钥
- DH Group 编号越大，密钥越长，越安全：DH1 (768-bit)、DH2 (1024-bit)、DH5 (1536-bit)、DH14 (2048-bit)、...

IKE 有两个版本：IKEv1 和 IKEv2。IKEv2 更高效更安全。CCNA 层面知道两者都存在即可，不需要深入差异。

#### 3.2.2 加密算法 — 提供机密性

| 算法 | 密钥长度 | 安全性 | 说明 |
|------|---------|--------|------|
| **DES** | 56-bit | ❌ 不安全，已淘汰 | Data Encryption Standard，历史上第一个广泛使用的加密标准 |
| **3DES** | 168-bit | ⚠️ 勉强可用 | 对数据做三次 DES 加密，比 DES 安全但性能差 |
| **AES-128** | 128-bit | ✅ 安全 | Advanced Encryption Standard，目前主流 |
| **AES-256** | 256-bit | ✅ 非常安全 | AES 的更强版本，推荐使用 |

**CCNA 考试要记住的：AES 是当前推荐的加密算法，DES 已被淘汰。**

#### 3.2.3 哈希算法 — 提供完整性

哈希算法的作用：对数据计算出一个固定长度的"指纹"（称为 Hash / Digest / HMAC）。发送方把数据和指纹一起发送，接收方重新计算指纹并对比，如果不一致说明数据被篡改了。

| 算法 | 输出长度 | 安全性 | 说明 |
|------|---------|--------|------|
| **MD5** | 128-bit | ⚠️ 有已知漏洞 | 速度快但安全性不足 |
| **SHA-1** | 160-bit | ⚠️ 逐渐淘汰 | 比 MD5 安全 |
| **SHA-256** | 256-bit | ✅ 安全 | SHA-2 家族，推荐使用 |

**实际在 IPSec 中使用的是 HMAC（Hash-based Message Authentication Code），即 HMAC-SHA-256、HMAC-MD5 等。HMAC 在哈希过程中加入了密钥，比纯哈希更安全。**

#### 3.2.4 认证方式 — 验证对端身份

| 方式 | 说明 | 优缺点 |
|------|------|--------|
| **PSK（Pre-Shared Key）** | 双方提前约定一个相同的密码字符串 | 简单易配置，但密钥管理困难（站点多时每对都要配不同的 PSK） |
| **Digital Certificate（数字证书）** | 使用 PKI（公钥基础设施）和 CA（证书颁发机构）签发的证书来验证身份 | 更安全、可扩展，但需要 CA 基础设施 |

**CCNA 层面主要考 PSK。**

#### 3.2.5 IPSec 协议：AH 和 ESP

IPSec 定义了两个协议来实际保护数据包，这是 **CCNA 重点考点**：

**AH（Authentication Header）— IP Protocol 51**

- 提供：认证 + 完整性 + 防重放
- **不提供加密（不提供机密性）**
- AH 对整个数据包（包括外层 IP Header 中不变的字段）做完整性校验
- **致命问题：NAT 会修改 IP Header 中的 IP 地址，导致 AH 的完整性校验失败。所以 AH 和 NAT 不兼容。**

**ESP（Encapsulating Security Payload）— IP Protocol 50**

- 提供：**加密** + 认证 + 完整性 + 防重放
- ESP 只对 IP payload 做完整性校验（不包括外层 IP Header）
- 所以 NAT 修改外层 IP Header 不影响 ESP 的完整性校验
- **ESP 可以与 NAT 配合使用**（通过 NAT-T，即 NAT Traversal，将 ESP 封装进 UDP 4500）

**关键对比（必记）：**

| 特性 | AH (Protocol 51) | ESP (Protocol 50) |
|------|-------------------|---------------------|
| 机密性（加密） | ❌ | ✅ |
| 完整性 | ✅ | ✅ |
| 认证 | ✅ | ✅ |
| 防重放 | ✅ | ✅ |
| 保护外层 IP Header | ✅（所以与 NAT 冲突） | ❌（所以与 NAT 兼容） |
| 实际使用频率 | 极低 | **绝大多数场景都用 ESP** |

**总结一句话：ESP 能做 AH 能做的所有事，还额外提供加密，而且与 NAT 兼容。所以实际中几乎只用 ESP。**

### 3.3 IPSec 的两种传输模式

IPSec 对数据包有两种不同的处理方式，这决定了数据包的封装结构：

#### Transport Mode（传输模式）

**做法：** 只加密/保护原始数据包的 **payload 部分**（TCP/UDP Header + Data），**保留原始 IP Header 不变**。

**封装结构（以 ESP 为例）：**

```
[ Original IP Header | ESP Header | TCP Header | Data | ESP Trailer | ESP Auth ]
                       ↑_________________________________↑
                              这部分被加密
```

**使用场景：** 端到端通信，即两台主机（host）之间直接用 IPSec 保护通信。因为保留了原始 IP Header，所以路由不受影响。

**典型场景：** 两台服务器之间的直接加密通信。

#### Tunnel Mode（隧道模式）

**做法：** 加密/保护 **整个原始数据包**（包括原始 IP Header），然后在最外面加一个新的 IP Header。

**封装结构（以 ESP 为例）：**

```
[ New IP Header | ESP Header | Original IP Header | TCP Header | Data | ESP Trailer | ESP Auth ]
                  ↑__________________________________________________________↑
                                       这部分被加密
```

**使用场景：** 网关到网关通信（Site-to-Site VPN）。新 IP Header 的 src/dst 是两端网关的公网 IP，中间路由器只看新 IP Header 做转发。

**CCNA 核心结论：Site-to-Site VPN 使用 Tunnel Mode，这是默认且最常见的模式。**

**对比表：**

| 特性 | Transport Mode | Tunnel Mode |
|------|---------------|-------------|
| 加密范围 | 只加密 payload | 加密整个原始 IP 包 |
| 原始 IP Header | 保留 | 被加密包裹在内部 |
| 新 IP Header | 不添加 | 添加（网关公网 IP） |
| 使用场景 | Host-to-Host | **Gateway-to-Gateway（Site-to-Site）** |
| 开销 | 较小 | 较大（多了一个 IP Header） |

### 3.4 IPSec 的完整工作流程（两个阶段）

IPSec 隧道的建立分为 **两个阶段（Phase）**，这是 CCNA 的核心考点：

#### Phase 1：IKE SA（也叫 ISAKMP SA）— 建立"管理隧道"

**目的：** 双方互相认证身份，协商加密参数，建立一条 **安全的管理通道**。这条管理通道不传用户数据，它的唯一用途是保护 Phase 2 的协商过程。

**详细步骤：**

**Step 1：发起方发送协商请求**

R1 向 R2 发送自己支持的安全参数组合（可以发多个供对方选择）：
- 加密算法：我支持 AES-256、AES-128、3DES
- 哈希算法：我支持 SHA-256、SHA-1、MD5
- 认证方式：我支持 PSK
- DH Group：我支持 DH14、DH5、DH2
- SA Lifetime：我想要 86400 秒（24 小时）

**Step 2：响应方选择匹配的参数**

R2 检查自己的配置，找出一组与 R1 匹配的参数（双方必须至少有一组完全匹配才能继续），然后回复确认。

**Step 3：DH 密钥交换**

双方使用选定的 DH Group 交换各自的公开值，然后各自独立计算出一把相同的 Shared Secret Key。这把密钥之后用来加密 Phase 1 后续通信和 Phase 2 的协商。

**Step 4：身份认证**

双方使用 PSK（或证书）互相验证身份。如果用 PSK，双方必须配置了相同的 PSK 字符串。

**Phase 1 完成：** 双方建立了 IKE SA（一条安全的管理隧道），后续 Phase 2 的协商将在这条隧道保护下进行。

**Phase 1 的两种协商模式（了解即可）：**

| 模式 | 消息数 | 安全性 | 说明 |
|------|--------|--------|------|
| Main Mode | 6 条消息 | 更安全（身份信息加密后传输） | 默认使用 |
| Aggressive Mode | 3 条消息 | 较不安全（身份信息明文传输） | 更快，但安全性打折 |

#### Phase 2：IPSec SA — 建立"数据隧道"

**目的：** 在 Phase 1 建立的安全管理通道保护下，协商实际加密用户数据的参数，建立 **IPSec SA**。

**详细步骤：**

**Step 1：协商 IPSec 参数（Transform Set）**

双方在 Phase 1 建立的安全通道中协商：
- 使用 AH 还是 ESP（通常是 ESP）
- 加密算法（如 AES-256）
- 哈希算法（如 SHA-256）
- 传输模式：Transport 还是 Tunnel（Site-to-Site 用 Tunnel）
- IPSec SA 的 Lifetime

**Step 2：定义感兴趣流量（Interesting Traffic）**

双方协商哪些流量需要被加密。这是通过 **ACL** 定义的：

```cisco
access-list 100 permit ip 10.1.1.0 0.0.0.255 10.2.2.0 0.0.0.255
```

意思是：从 10.1.1.0/24 到 10.2.2.0/24 的流量需要被 IPSec 保护。

**Step 3：建立 IPSec SA**

协商成功后，双方各建立 **一对单向的 IPSec SA**：
- 一条用于发送（outbound SA）
- 一条用于接收（inbound SA）

每条 SA 由一个 **SPI（Security Parameter Index）** 标识。

**Phase 2 完成：** 数据隧道建立，匹配感兴趣流量的数据包开始被加密传输。

#### 完整时间线总结

```
时间 →

[Phase 1: IKE]                              [Phase 2: IPSec]
R1 ←→ R2                                    R1 ←→ R2
┌─────────────────────────┐                 ┌──────────────────────┐
│ 1. 协商 IKE 参数         │                 │ 1. 协商 IPSec 参数    │
│ 2. DH 密钥交换           │                 │ 2. 定义感兴趣流量     │
│ 3. 互相认证身份           │                 │ 3. 建立 IPSec SA      │
│                          │                 │                       │
│ 结果：IKE SA（管理隧道）  │  ── 保护 ──→   │ 结果：IPSec SA（数据隧道）│
└─────────────────────────┘                 └──────────────────────┘
                                                     │
                                                     ↓
                                            用户数据开始加密传输
```

### 3.5 一个完整的 IPSec 数据传输过程

假设 Phase 1 和 Phase 2 都已完成，现在 PC-A 发数据给 PC-B：

**Step 1：** PC-A 发出原始数据包：`[IP: 10.1.1.10 → 10.2.2.20 | TCP | Data]`

**Step 2：** R1 收到数据包，检查是否匹配感兴趣流量的 ACL。匹配。

**Step 3：** R1 使用 ESP 协议对数据包进行加密和封装（Tunnel Mode）：
- 加密整个原始 IP 包
- 添加 ESP Header（含 SPI 和序列号）和 ESP Trailer
- 计算完整性摘要（ESP Auth）
- 添加新的 IP Header（src=R1 公网 IP, dst=R2 公网 IP, Protocol=50）

```
[ New IP: 203.0.113.1 → 198.51.100.1, Proto=50 | ESP Header | *加密的*{Original IP | TCP | Data} | ESP Trailer | ESP Auth ]
```

**Step 4：** 加密后的数据包通过 Internet 正常路由转发到 R2

**Step 5：** R2 收到数据包，识别 Protocol=50（ESP），使用对应的 IPSec SA 进行解密

**Step 6：** R2 验证完整性（ESP Auth），验证序列号（防重放），解密获得原始数据包

**Step 7：** R2 将原始数据包转发给 PC-B

### 3.6 IPSec 配置逻辑（CCNA 层面）

CCNA 不要求你完整手写 IPSec 配置，但你需要理解配置的逻辑流程：

```
Step 1: 配置 IKE Phase 1 Policy
   ↓
Step 2: 配置 PSK（Pre-Shared Key）
   ↓
Step 3: 配置 IPSec Transform Set（Phase 2 参数）
   ↓
Step 4: 定义感兴趣流量（ACL）
   ↓
Step 5: 创建 Crypto Map（把以上全部绑定在一起）
   ↓
Step 6: 将 Crypto Map 应用到出接口
```

**示例配置（了解级别，不需要背诵）：**

```cisco
! Step 1: Phase 1 Policy
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

! Step 2: PSK
crypto isakmp key MYSECRETKEY address 198.51.100.1

! Step 3: Transform Set
crypto ipsec transform-set MY-SET esp-aes 256 esp-sha256-hmac
 mode tunnel

! Step 4: 感兴趣流量
access-list 100 permit ip 10.1.1.0 0.0.0.255 10.2.2.0 0.0.0.255

! Step 5: Crypto Map
crypto map MY-MAP 10 ipsec-isakmp
 match address 100
 set peer 198.51.100.1
 set transform-set MY-SET

! Step 6: 应用到接口
interface GigabitEthernet0/0
 crypto map MY-MAP
```

### 3.7 IPSec 验证命令

```cisco
! 查看 Phase 1 SA 状态
R1# show crypto isakmp sa
! 状态 QM_IDLE = Phase 1 建立成功

! 查看 Phase 2 SA 状态
R1# show crypto ipsec sa
! 显示：加密/解密包计数、SA lifetime、SPI 值等
! 如果 encaps/decaps 计数在增长，说明数据在正常加密传输

! 查看 IKE Policy 配置
R1# show crypto isakmp policy
```

---

## 第四章：GRE over IPSec — 组合使用

### 4.1 为什么需要组合

| 技术 | 加密 | 组播支持 |
|------|------|---------|
| 纯 IPSec | ✅ | ❌ |
| 纯 GRE | ❌ | ✅ |
| GRE over IPSec | ✅ | ✅ |

**核心问题：** 你有两个站点通过 Internet 连接，你需要在它们之间运行 OSPF 动态路由协议。

- 你不能只用 IPSec：因为 OSPF 使用组播地址 224.0.0.5 和 224.0.0.6，纯 IPSec 不支持组播
- 你不能只用 GRE：因为 GRE 不加密，数据在 Internet 上是明文的

**解决方案：GRE over IPSec**

- 先用 GRE 封装（解决组播问题）
- 再用 IPSec 加密 GRE 包（解决安全问题）

### 4.2 封装过程（逐步）

**原始数据包：**
```
[ IP: 10.1.1.10 → 10.2.2.20 | OSPF Hello | ... ]
```

**Step 1：GRE 封装**
```
[ GRE IP: 203.0.113.1 → 198.51.100.1, Proto=47 | GRE Header | IP: 10.1.1.10 → 10.2.2.20 | OSPF | ... ]
```

**Step 2：IPSec 加密（ESP Tunnel Mode）**
```
[ New IP: 203.0.113.1 → 198.51.100.1, Proto=50 | ESP Header | *加密*{GRE IP Header | GRE Header | Original IP | OSPF | ...} | ESP Trailer | ESP Auth ]
```

**最终数据包从外到内：**
```
New IP Header → ESP Header → GRE IP Header → GRE Header → Original IP Header → Original Payload
    (明文)        (明文)      ←─────────────── 这些全被加密 ──────────────────→
```

### 4.3 对端的解封装过程

**Step 1：** R2 收到数据包，识别 Protocol=50（ESP）

**Step 2：** R2 用 IPSec SA 解密，得到 GRE 封装的数据包

**Step 3：** R2 识别内层 Protocol=47（GRE），剥离 GRE Header

**Step 4：** 得到原始数据包（OSPF Hello），正常处理

### 4.4 MTU 开销计算

GRE over IPSec 的封装开销比单独的 GRE 或 IPSec 都大：

```
原始数据包
  + GRE 开销：20 (new IP) + 4 (GRE) = 24 bytes
  + ESP 开销：20 (new IP) + 8 (ESP header) + 2 (ESP trailer) + 12 (ESP auth) ≈ 42 bytes
  + 可能的填充（padding）
---
总开销大约：60-70 bytes
```

所以 GRE over IPSec 隧道的有效 MTU 大约是 **1400 bytes 左右**（从 1500 减去开销）。

实际配置中通常设置：
```cisco
interface Tunnel 0
 ip mtu 1400
 ip tcp adjust-mss 1360
```

---

## 第五章：SSL/TLS VPN（简要）

### 5.1 是什么

SSL/TLS VPN 使用 **SSL（Secure Sockets Layer）或 TLS（Transport Layer Security）** 协议来建立加密隧道。

它工作在 **L4-L7**，与 IPSec（L3）不同。

**最典型的产品：Cisco AnyConnect Secure Mobility Client**

### 5.2 两种接入模式

| 模式 | 说明 | 客户端需求 |
|------|------|-----------|
| **Clientless SSL VPN** | 用户通过浏览器访问 VPN 网关的 HTTPS 页面，在网页上访问内部资源 | 不需要安装任何软件，只需要浏览器 |
| **Full Tunnel SSL VPN** | 用户安装 AnyConnect 客户端，建立完整隧道，所有流量或指定流量走隧道 | 需要安装 AnyConnect |

### 5.3 SSL/TLS VPN vs IPSec VPN

| 特性 | IPSec VPN | SSL/TLS VPN |
|------|-----------|-------------|
| 工作层级 | L3 | L4-L7 |
| 典型部署 | Site-to-Site | **Remote-Access** |
| 客户端需求 | IPSec client 或网关设备 | 浏览器或 AnyConnect |
| NAT 穿越 | 需要 NAT-T（UDP 4500） | 天然支持（走 HTTPS/TCP 443） |
| 灵活性 | 保护所有 IP 层流量 | 可以精细控制到应用层 |
| 防火墙友好 | 可能被防火墙阻止（ESP protocol） | 非常友好（TCP 443 几乎不会被阻） |

**CCNA 考试要知道的核心点：SSL/TLS VPN 主要用于 Remote-Access 场景，IPSec 主要用于 Site-to-Site 场景。**

---

## 第六章：CCNA 常见考点与陷阱

### 考点 1：GRE 不加密

**题目示例：** "A network engineer needs to create a tunnel between two sites. Which protocol provides encapsulation but does NOT provide encryption?"

**答案：GRE**

**陷阱：** 选项中可能有 "IPSec in Transport Mode" — 这是错的，即使是 Transport Mode 的 ESP 也提供加密。

### 考点 2：AH vs ESP 的区别

**题目示例：** "Which IPSec protocol provides confidentiality?"

**答案：ESP（只有 ESP 提供加密/机密性，AH 不提供）**

**陷阱：** 题目问 "Which provides authentication?" — **AH 和 ESP 都提供认证**，不要看到名字叫 "Authentication Header" 就以为只有 AH 提供认证。

### 考点 3：AH 与 NAT 不兼容

**题目示例：** "Why does AH fail when used with NAT?"

**答案：** AH 对整个数据包（包括 IP Header 中不变的字段如 Source IP、Dest IP）做完整性校验。NAT 设备会修改 IP Header 中的 IP 地址，导致接收方重新计算的完整性摘要与发送方附带的不一致，校验失败。

### 考点 4：Tunnel Mode vs Transport Mode

**题目示例：** "Which IPSec mode is used for Site-to-Site VPN?"

**答案：Tunnel Mode**

**陷阱：** 题目可能描述一个场景但不直接说是 Site-to-Site，而是说 "between two gateways" 或 "protecting traffic between two networks" — 这就是 Tunnel Mode。

### 考点 5：为什么用 GRE over IPSec

**题目示例：** "An engineer needs to run OSPF between two remote sites over an encrypted VPN. Which solution should be used?"

**答案：GRE over IPSec**

**推理过程：**
1. OSPF 需要组播（224.0.0.5/6）
2. IPSec 不支持组播 → 不能只用 IPSec
3. GRE 支持组播但不加密 → 不能只用 GRE
4. GRE over IPSec = 组播 + 加密 → 正确答案

### 考点 6：Site-to-Site vs Remote-Access

**题目示例：** "Which type of VPN requires the end user to install client software?"

**答案：Remote-Access VPN**

**关键区分：**
- Site-to-Site：网关设备之间建隧道，用户无感知，不需要装软件
- Remote-Access：用户终端到网关，用户必须装 VPN 客户端

### 考点 7：IPSec Phase 1 vs Phase 2

**题目示例：** "What is the purpose of IKE Phase 1?"

**答案：** 建立安全的管理通道（IKE SA / ISAKMP SA），用于保护 Phase 2 的协商。Phase 1 不传输用户数据。

**另一种考法：** "During which phase is the actual data encryption parameters negotiated?"

**答案：Phase 2**

### 考点 8：IPSec 使用的协议和端口

| 协议/端口 | 用途 |
|-----------|------|
| **UDP 500** | IKE 协商（Phase 1 和 Phase 2） |
| **UDP 4500** | NAT-T（NAT Traversal），当检测到 NAT 时，ESP 被封装进 UDP 4500 |
| **IP Protocol 50** | ESP 数据传输 |
| **IP Protocol 51** | AH 数据传输 |

**陷阱：** ESP 和 AH 不是 TCP 或 UDP 协议，它们是直接在 IP 层上的协议（和 TCP/UDP 平级）。Protocol 字段值分别是 50 和 51。

---

## 第七章：完整对比总结

### 7.1 核心技术对比

| 特性 | GRE | IPSec (ESP) | GRE over IPSec | SSL/TLS VPN |
|------|-----|-------------|-----------------|-------------|
| 工作层级 | L3 | L3 | L3 | L4-L7 |
| 加密（机密性） | ❌ | ✅ | ✅ | ✅ |
| 认证 | ❌ | ✅ | ✅ | ✅ |
| 完整性 | ❌ | ✅ | ✅ | ✅ |
| 防重放 | ❌ | ✅ | ✅ | ✅ |
| 组播支持 | ✅ | ❌ | ✅ | ❌ |
| 多协议封装 | ✅ | ❌ | ✅ | ❌ |
| NAT 兼容性 | 需处理 | ESP 可以 (NAT-T) | 需处理 | 天然支持 |
| IP Protocol Number | 47 | 50 (ESP) / 51 (AH) | 50 (外层) | N/A (TCP 443) |
| 典型场景 | 简单隧道 | Site-to-Site 加密 | 加密隧道 + 路由协议 | Remote Access |

### 7.2 IPSec 两个阶段对比

| 特性 | Phase 1 (IKE SA) | Phase 2 (IPSec SA) |
|------|-------------------|---------------------|
| 目的 | 建立安全管理通道 | 建立数据加密通道 |
| 协商内容 | 加密/哈希/认证/DH/Lifetime | ESP or AH / 加密 / 哈希 / Mode / Lifetime |
| 使用端口 | UDP 500 | 在 Phase 1 通道内协商 |
| SA 数量 | 一条双向 SA | 一对单向 SA（一进一出） |
| 传输用户数据 | ❌ | ✅ |

### 7.3 AH vs ESP 对比

| 特性 | AH (Protocol 51) | ESP (Protocol 50) |
|------|-------------------|---------------------|
| 加密 | ❌ | ✅ |
| 认证 | ✅ | ✅ |
| 完整性 | ✅ | ✅ |
| 防重放 | ✅ | ✅ |
| 保护 IP Header | ✅ | ❌ |
| NAT 兼容 | ❌ | ✅ |
| 使用频率 | 极低 | 主流 |

### 7.4 Transport Mode vs Tunnel Mode

| 特性 | Transport Mode | Tunnel Mode |
|------|---------------|-------------|
| 加密范围 | 只加密 payload | 加密整个原始 IP 包 |
| 新 IP Header | 不添加 | 添加 |
| 使用场景 | Host-to-Host | Gateway-to-Gateway |
| 开销 | 较小 | 较大 |
| VPN 类型 | 很少用 | Site-to-Site VPN 默认 |

---

## 第八章：可以不深挖的内容

以下内容在 CCNA 200-301 中不会考到细节：

- IKE Phase 1 Main Mode 的 6 条消息的具体内容
- Aggressive Mode 的 3 条消息的具体内容
- DH 算法的数学原理
- IPSec 的完整 CLI 配置（不需要手写，理解逻辑即可）
- IKEv1 vs IKEv2 的详细技术差异（知道 v2 更好即可）
- DMVPN（Dynamic Multipoint VPN）— 这是 CCNP 内容
- FlexVPN — CCNP 内容
- GET VPN、GDOI — 高级内容
- SSL/TLS 的握手过程细节

---

## 快速记忆卡片

**CIA = Confidentiality + Integrity + Authentication**

**ESP = Everything Security Protocol（加密 + 认证 + 完整性 + 防重放）**

**AH = Almost Has it（认证 + 完整性 + 防重放，就是没加密）**

**GRE = 隧道但不安全；IPSec = 安全但不支持组播；GRE over IPSec = 两者结合**

**Phase 1 = 建管理通道；Phase 2 = 建数据通道**

**Site-to-Site = 网关之间，用户无感知；Remote-Access = 用户装客户端**

**ESP = Protocol 50, AH = Protocol 51, GRE = Protocol 47**

**IKE = UDP 500; NAT-T = UDP 4500**

---

*CCNA 200-301 VPN/GRE/IPSec 完全笔记 — 完*
