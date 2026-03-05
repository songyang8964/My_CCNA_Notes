# Linux（CCNA 中的 Linux 基础）

## 1. 知识点定位

CCNA 200-301 不会专门考"Linux 操作系统"本身，但 Linux 相关的知识散布在多个考试域中，主要出现在：

- **Network Automation and Programmability（10%）** — 自动化脚本运行环境、REST API 调用
- **IP Connectivity** — 从主机视角排障（路由表、DNS、ping、traceroute）
- **Security Fundamentals** — SSH 访问网络设备

**重要程度：中等偏次要。** 每次考试基本会涉及 1-3 道与主机网络配置/排障命令相关的题目。

---

## 2. 核心概念

CCNA 关心的 Linux 知识只有一个核心主题：**从 Linux 主机的视角理解网络配置和排障。**

需要掌握的内容分三块：网络配置查看、网络排障命令、以及 Linux 与 Windows 命令的对比。

---

## 3. 工作原理（Linux 网络排障的逻辑链）

排障思路自下而上逐层检查：

### 第一层：接口有没有 UP，IP 地址对不对？
```
ifconfig
ip address show
```

关注字段：

| 字段 | 含义 |
|------|------|
| UP / DOWN | 接口是否启用 |
| inet x.x.x.x | IPv4 地址 |
| netmask | 子网掩码 |
| inet6 | IPv6 地址 |
| ether xx:xx:xx:xx:xx:xx | MAC 地址 |
| RX/TX packets | 收发包计数，看有没有 errors |

### 第二层：默认网关配不配，路由表对不对？
```
ip route show
route -n
netstat -r
```

典型输出：
```
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.100
```

- 第一行 = 默认路由，所有非本地流量发往 192.168.1.1（网关）
- 第二行 = 直连网段

### 第三层：DNS 能不能解析？
```
nslookup www.google.com
dig www.google.com
cat /etc/resolv.conf
```

`/etc/resolv.conf` 存的是 DNS 服务器地址。DNS 不对 → 能 ping IP 但不能 ping 域名。

### 第四层：连通性和路径
```
ping 8.8.8.8
traceroute 8.8.8.8
```

### 第五层：端口和连接状态
```
ss -tuln
netstat -tuln
```

`-t` TCP, `-u` UDP, `-l` LISTEN, `-n` 数字显示

---

## 4. CCNA 必须掌握的 Linux 命令

### 4.1 网络配置查看

| 命令 | 作用 | 考试注意点 |
|------|------|-----------|
| `ifconfig` | 查看接口 IP、MAC、状态 | 老命令，很多考题还在用 |
| `ip address show` / `ip addr` | 同上，新命令 | 和 ifconfig 输出格式不同，信息一样 |
| `ip route show` / `ip route` | 查看路由表 | 重点看 default 那一行 |
| `route -n` | 查看路由表（老命令） | `-n` 不解析主机名，直接显示 IP |
| `cat /etc/resolv.conf` | 查看 DNS 配置 | 考题可能给文件内容让你判断 |

### 4.2 网络排障

| 命令 | 作用 | 考试注意点 |
|------|------|-----------|
| `ping` | 测试连通性（ICMP） | Linux **不停**，Ctrl+C 停。Windows 发 4 个停 |
| `traceroute` | 追踪路径 | Linux 默认 **UDP**。Windows `tracert` 默认 **ICMP**。经典考点 |
| `nslookup` | DNS 查询 | Linux 和 Windows 都有 |
| `dig` | DNS 查询（更详细） | **Linux 独有** |
| `arp -a` | 查看 ARP 表 | Linux 和 Windows 命令相同 |
| `ss -tuln` / `netstat -tuln` | 查看端口和连接 | `-t` TCP `-u` UDP `-l` LISTEN `-n` 数字 |

### 4.3 其他

| 命令 | 作用 |
|------|------|
| `sudo` | 以 root 权限执行命令 |
| `ssh user@host` | SSH 远程连接 |
| `cat` | 查看文件内容 |

---

## 5. 常见考点与陷阱

**考点 1：Linux `traceroute` vs Windows `tracert`**
Linux = **UDP**，Windows = **ICMP**。按 Windows 习惯选 ICMP 就错了。

**考点 2：Linux `ping` 不会自动停止**
Linux 不停（`-c N` 限制次数），Windows 发 4 个停。

**考点 3：给 `ifconfig` / `ip addr` 输出，判断问题**
检查：IP 是否正确子网、掩码是否对、接口是否 UP。

**考点 4：默认网关缺失或错误**
能 ping 本地但不能 ping 远端 → 没有 default 路由。

**考点 5：`/etc/resolv.conf`**
能 ping IP 但不能 ping 域名 → DNS 配置问题。

---

## 6. 对比与总结

### Linux vs Windows 网络命令对比（高频考点）

| 功能     | Linux                   | Windows             | 关键区别            |
| ------ | ----------------------- | ------------------- | --------------- |
| 查看 IP  | `ifconfig` / `ip addr`  | `ipconfig`          | 拼写不同            |
| 详细 IP  | 默认就是详细                  | `ipconfig /all`     | Windows 需要 /all |
| 路由表    | `ip route` / `route -n` | `route print`       | 格式不同            |
| Ping   | `ping`（不停）              | `ping`（4次停）         | `-c N` 限制次数     |
| 路径追踪   | `traceroute`（**UDP**）   | `tracert`（**ICMP**） | **协议不同**        |
| DNS 查询 | `nslookup` / `dig`      | `nslookup`          | `dig` Linux 独有  |
| ARP    | `arp -a`                | `arp -a`            | 相同              |
| DNS 配置 | `/etc/resolv.conf`      | `ipconfig /all`     | 文件 vs 命令        |
| SSH    | `ssh`（内置）               | `ssh`（Win10+）       | 早期需 PuTTY       |

### 快速记忆清单

- `ifconfig` / `ip addr` → 看 IP、MAC、接口状态
- `ip route` → 看路由表，找 **default** 网关
- `/etc/resolv.conf` → DNS 服务器配置
- `traceroute` = **UDP**（Linux），`tracert` = **ICMP**（Windows）
- `ping` Linux 不停，Windows 发 4 个停
- 排障顺序：接口状态 → IP/掩码 → 默认网关 → DNS → ping/traceroute

### 可以不深挖的内容

- Linux 文件系统结构、用户权限管理
- iptables / nftables 防火墙
- Linux 内核网络参数调优
- systemd / 服务管理
- Linux 发行版区别