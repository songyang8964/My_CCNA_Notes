## 核心原则：TCP vs UDP 的选择逻辑

按传输层协议分组记忆比逐个背端口号有效得多。

- **TCP** = 可靠传输（三次握手、确认重传、有序）
- **UDP** = 快、轻、不可靠（发完不管）

一个协议选 TCP 还是 UDP，取决于它的业务需求。搞清楚需求就能推导出来，不用死背。

---

## 按功能分组的端口速查

### 第一组：远程管理（天天用的）

| 端口 | 协议 | 干什么 |
| --- | --- | --- |
| 22 | SSH | 加密远程登录（你每天登 OTC 设备用的） |
| 23 | Telnet | 明文远程登录（不安全，但考试爱考和 SSH 对比） |

**记法**：22 和 23 挨着，SSH 比 Telnet 安全，端口号也更小（先有安全意识的人先占了 22）。考试只要问"哪个安全"，答案永远是 SSH/22。

### 第二组：Web 访问

| 端口 | 协议 | 干什么 |
| --- | --- | --- |
| 80 | HTTP | 明文网页 |
| 443 | HTTPS | 加密网页（SSL/TLS VPN 也走这个端口） |

**记法**：80 是最基础的 Web 端口，所有人都知道。443 = HTTPS = 加密版 HTTP。

### 第三组：文件传输

| 端口 | 协议 | 干什么 |
| --- | --- | --- |
| 20 | FTP Data | FTP 实际传数据的通道 |
| 21 | FTP Control | FTP 控制命令通道（登录、ls、cd 等） |
| 69 | TFTP | 简化版 FTP，UDP，无认证。用来传 IOS 镜像、备份配置 |

**记法**：FTP 用两个端口（20 数据、21 控制），21 是你连接时先碰到的（控制），然后 20 开始传数据。TFTP 的 69 单独记，它用 UDP 而不是 TCP——因为 TFTP 设计目标是"极简"。

**考试陷阱**：题目问 "Which protocol uses UDP for file transfer?" — 答案是 TFTP (69/UDP)，不是 FTP。

### 第四组：邮件

| 端口 | 协议 | 干什么 |
| --- | --- | --- |
| 25 | SMTP | 发邮件 |
| 110 | POP3 | 收邮件（下载到本地，服务器删除） |
| 143 | IMAP | 收邮件（服务器保留副本，多设备同步） |

**记法**：SMTP = Send Mail，25 是最早的邮件端口。收邮件两个协议（POP3 和 IMAP），CCNA 偶尔考但不是重点。

### 第五组：DNS 和 DHCP（网络基础服务）

| 端口 | 协议 | 干什么 |
| --- | --- | --- |
| 53 | DNS | 域名解析，同时用 TCP 和 UDP（查询用 UDP，区域传输用 TCP） |
| 67 | DHCP Server | DHCP 服务器监听端口 |
| 68 | DHCP Client | DHCP 客户端监听端口 |

**记法**：DNS = 53，这个没有捷径，高频出现自然记住。DHCP 的 67/68 是一对，服务器 67 客户端 68。

**考试陷阱**："DNS uses which transport protocol?" — 答案是 both TCP and UDP。很多人只记得 UDP，但 DNS 区域传输（zone transfer）用 TCP。

### 第六组：网络管理

| 端口 | 协议 | 干什么 |
| --- | --- | --- |
| 161 | SNMP Agent | 设备上的 Agent 监听，接收 NMS 的 Get/Set |
| 162 | SNMP Trap/Inform | NMS 监听，接收设备发来的 Trap |
| 514 | Syslog | 设备把日志发到 Syslog 服务器，UDP |
| 49 | TACACS+ | 认证授权审计（AAA），Cisco 私有，TCP |
| 1812/1813 | RADIUS | 认证(1812)/计费(1813)，UDP |

**记法**：SNMP 161/162 是一对，161 在设备端，162 在管理端。Syslog 514 单独记。TACACS+ 和 RADIUS 的对比是高频考点。

**考试高频对比**：

| 特性 | TACACS+ | RADIUS |
| --- | --- | --- |
| 端口 | TCP 49 | UDP 1812/1813 |
| 开发者 | Cisco | 开放标准 |
| 加密范围 | 整个 payload 加密 | 只加密密码 |
| AAA 分离 | 认证、授权、审计 分开处理 | 认证和授权 合并处理 |

### 第七组：NTP 和 VPN 相关

| 端口 | 协议 | 干什么 |
| --- | --- | --- |
| 123 | NTP | 时间同步，UDP |
| 500 | IKE | IPSec 密钥协商 |
| 4500 | NAT-T | IPSec 穿越 NAT 时用 |

---

## 按传输层协议分类的规律

### 规律一：需要"可靠性"的用 TCP

这类协议如果数据丢了或乱了，业务就废了，所以必须用 TCP：

| 端口 | 协议 | 为什么必须可靠 |
| --- | --- | --- |
| 20/21 | FTP | 传文件，丢一个字节文件就损坏了 |
| 22 | SSH | 加密远程登录，丢包或乱序会导致加密流断裂 |
| 23 | Telnet | 远程登录，命令不能丢 |
| 25 | SMTP | 发邮件，邮件内容不能丢 |
| 80 | HTTP | 网页内容不能缺 |
| 110 | POP3 | 收邮件，同理 |
| 143 | IMAP | 收邮件，同理 |
| 443 | HTTPS | 加密网页，同理 |
| 49 | TACACS+ | 认证授权，安全相关的数据绝不能丢 |

**记忆口诀**：凡是涉及"登录、传文件、传网页、传邮件、安全认证"的，全部 TCP。

### 规律二：需要"速度快、轻量"的用 UDP

这类协议的特点是：数据量小、时效性强、丢了可以再发一次或者丢了也无所谓：

| 端口 | 协议 | 为什么用 UDP |
| --- | --- | --- |
| 69 | TFTP | 极简文件传输，协议本身内置了简单的确认机制 |
| 53 | DNS（查询） | 一问一答，查询包很小，丢了客户端重发就行 |
| 67/68 | DHCP | 客户端还没 IP 地址，无法建立 TCP 连接 |
| 123 | NTP | 时间同步，一个小包搞定，丢了下次再同步 |
| 161/162 | SNMP | 管理查询和 Trap，要求低延迟，丢一条 Trap 不致命 |
| 514 | Syslog | 日志发送，量大，丢几条日志可以接受 |
| 500 | IKE | IPSec 密钥协商，IKE 协议自身有重传机制 |
| 4500 | NAT-T | 同上 |
| 1812/1813 | RADIUS | 历史设计选择，RADIUS 自己实现了重传 |

**记忆口诀**：凡是"一问一答、小包快速、有没有 IP 都要能工作、丢了不致命或自带重传"的，全部 UDP。

### 规律三：两个都用的（极少，但 CCNA 必考）

| 端口 | 协议 | TCP 用途 | UDP 用途 |
| --- | --- | --- | --- |
| 53 | DNS | 区域传输（主备 DNS 服务器之间同步全量记录，数据量大） | 普通查询（客户端问 "google.com 的 IP 是什么"，一问一答） |

CCNA 范围内只有 DNS 需要记"两个都用"。考试非常爱考这个点。

### 规律四：不用 TCP 也不用 UDP 的（IP Protocol Number）

这些协议直接跑在 IP 层上，和 TCP/UDP 是平级关系，根本没有端口号的概念：

| IP Protocol Number | 协议 | 为什么不用 TCP/UDP |
| --- | --- | --- |
| 1 | ICMP | 网络层控制协议，ping/traceroute 本身就是底层诊断工具 |
| 6 | TCP | 它自己就是传输层协议 |
| 17 | UDP | 它自己就是传输层协议 |
| 47 | GRE | 隧道封装协议，直接在 IP 层操作 |
| 50 | ESP | IPSec 加密协议，直接在 IP 层操作 |
| 51 | AH | IPSec 认证协议，直接在 IP 层操作 |
| 89 | OSPF | 路由协议，直接在 IP 层操作，不需要传输层 |

**记忆口诀**：ICMP、GRE、ESP、AH、OSPF 这些协议都是"网络层自己人"，不需要借助 TCP/UDP。

---

## 最终分类总结图

```
                    ┌─ 登录类：SSH(22), Telnet(23)
                    ├─ 文件类：FTP(20/21)
         TCP ──────├─ Web类：HTTP(80), HTTPS(443)
        (可靠)      ├─ 邮件类：SMTP(25), POP3(110), IMAP(143)
                    └─ 安全类：TACACS+(49)

                    ┌─ 网络基础：DHCP(67/68), TFTP(69), NTP(123)
         UDP ──────├─ 管理监控：SNMP(161/162), Syslog(514)
        (快速)      ├─ VPN相关：IKE(500), NAT-T(4500)
                    └─ 认证类：RADIUS(1812/1813)

      TCP+UDP ───── DNS(53)

                    ┌─ ICMP(1)
    IP Protocol ───├─ GRE(47), ESP(50), AH(51)
   (无端口号)       └─ OSPF(89)
```

---

## 深入理解：IP Protocol Number vs 端口号

### 数据包结构

一个完整的数据包从外到内是这样的：

```
[ L2 Frame Header | IP Header | TCP/UDP Header | Application Data ]
                      ↑              ↑
                   网络层          传输层
```

IP Protocol Number 和端口号分别存在不同的 Header 里，作用完全不同。

### IP Header 里的 Protocol 字段

IP Header 有一个 8-bit 的字段叫 Protocol，它的作用是告诉接收方：我这个 IP 包的 payload 交给上层的哪个协议处理。

设备收到一个 IP 包后，看 Protocol 字段的值来决定下一步：

| Protocol 值 | 含义 | 设备的动作 |
| --- | --- | --- |
| 1 | ICMP | 把 payload 交给 ICMP 模块处理 |
| 6 | TCP | 把 payload 交给 TCP 模块处理 |
| 17 | UDP | 把 payload 交给 UDP 模块处理 |
| 47 | GRE | 把 payload 交给 GRE 模块处理 |
| 50 | ESP | 把 payload 交给 ESP 模块处理 |
| 89 | OSPF | 把 payload 交给 OSPF 进程处理 |

**关键点**：TCP 和 UDP 本身就是 Protocol 字段的两个值（6 和 17）。它们和 GRE、ESP、OSPF 是平级的。

### TCP/UDP Header 里的端口号

只有当 IP Header 的 Protocol = 6 (TCP) 或 17 (UDP) 时，IP payload 里面才会有一个 TCP 或 UDP Header。端口号存在于这个 TCP/UDP Header 里面。

端口号的作用是：在 TCP/UDP 内部，再区分把数据交给哪个应用程序。

比如一台服务器同时运行了 SSH、HTTP、DNS 三个服务，都用 TCP。TCP 模块收到数据后靠端口号区分：

| 端口号 | 交给哪个应用 |
| --- | --- |
| 22 | SSH 进程 |
| 80 | HTTP 进程 |
| 53 | DNS 进程 |

### 两者的层级关系（完整图示）

```
一个 SSH 数据包：

[ IP Header                    | TCP Header         | SSH Data    ]
  Protocol = 6 (TCP)             Dst Port = 22
  ↑                              ↑
  第一次分拣：                    第二次分拣：
  "这个包交给 TCP 模块"           "TCP 再交给 22 号端口的进程(SSH)"


一个 OSPF 数据包：

[ IP Header                    | OSPF Data                       ]
  Protocol = 89 (OSPF)
  ↑
  第一次分拣：
  "这个包直接交给 OSPF 进程"
  
  没有 TCP/UDP Header！
  没有端口号！
  OSPF 直接坐在 IP 上面。
```

### 快递类比

IP Protocol Number = 快递送到大楼后，前台看包裹标签决定送去哪个部门。标签写"6"送去 TCP 部门，写"17"送去 UDP 部门，写"89"直接送去 OSPF 部门。

端口号 = 快递送到 TCP 部门后，TCP 部门内部再看一个小标签，决定送给部门里的哪个人。标签写"22"给 SSH，写"80"给 HTTP。

OSPF 快递根本不经过 TCP 部门，直接从前台送到 OSPF 部门。所以 OSPF 没有"部门内部小标签"（端口号）。

### 对比总结

| | IP Protocol Number | 端口号 (Port Number) |
| --- | --- | --- |
| 存在哪里 | IP Header 的 Protocol 字段 | TCP 或 UDP Header 的 Port 字段 |
| 作用 | IP 层决定把 payload 交给哪个上层协议 | TCP/UDP 内部决定交给哪个应用程序 |
| 层级 | L3 → L4 的分拣 | L4 → L7 的分拣 |
| 谁有 | 所有 IP 包都有 | 只有 TCP/UDP 包才有 |
| OSPF | Protocol = 89 | 没有端口号 |
| SSH | Protocol = 6 (TCP) | Port = 22 |
| ESP | Protocol = 50 | 没有端口号 |
| SNMP | Protocol = 17 (UDP) | Port = 161/162 |

所以当考题说 "OSPF uses port 89" 的时候，这句话错在：OSPF 根本不经过 TCP/UDP，不存在端口号这个概念。89 是 IP Protocol Number，不是 port。

---

## CCNA 考试最爱挖的坑

- **坑 1**："RADIUS uses TCP" — 错。RADIUS 用 UDP。TACACS+ 才用 TCP。这两个经常反着考。
- **坑 2**："DNS uses UDP" — 不完整。DNS 查询用 UDP，区域传输用 TCP，正确答案是 both。
- **坑 3**："OSPF uses port 89" — 错。OSPF 用 IP Protocol 89，不是端口 89。OSPF 不经过 TCP/UDP。
- **坑 4**："SNMP Trap uses port 161" — 错。Agent 接收请求用 161，Trap 发到 NMS 的 162。

---

## 最终记忆策略

不要试图一次背完所有端口。按优先级来：

- **必须秒答的（考试出现率极高）**：22 SSH、23 Telnet、80 HTTP、443 HTTPS、53 DNS、67/68 DHCP、20/21 FTP、69 TFTP、161/162 SNMP、25 SMTP
- **需要熟悉的**：49 TACACS+、1812/1813 RADIUS、514 Syslog、123 NTP、500 IKE
- **不能和端口号混的**：Protocol 1 ICMP、6 TCP、17 UDP、47 GRE、50 ESP、51 AH、89 OSPF

刷题的时候每碰到一次就强化一次，比单独背诵有效得多。
