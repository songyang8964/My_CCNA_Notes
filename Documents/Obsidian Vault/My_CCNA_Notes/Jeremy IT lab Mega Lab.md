## Cisco IOS 的模式层级

Cisco 的 CLI 是分层的，每一层能做的事不同，提示符也不同：

```
开机进入
   ↓
User EXEC Mode          Router>              只能看，不能改
   ↓  enable (en)
Privileged EXEC Mode    Router#              能看所有东西，能执行操作命令
   ↓  configure terminal (conf t)
Global Config Mode      Router(config)#      改全局配置
   ↓  interface / router ospf / line 等
Sub-config Modes        Router(config-if)#   改具体子项配置
                        Router(config-router)#
                        Router(config-line)#
```

## 每层能干什么

**User EXEC `Router>`** — 权限最低，刚登录就在这里。只能执行基础查看命令，比如 `ping`、`show version`、`show ip interface brief`（部分 show 命令在这层受限）。**不能改任何配置。**

**Privileged EXEC `Router#`** — 敲 `enable` 进入。可以执行所有 show 命令、`copy`、`reload`、`write`、`debug` 等操作命令。**还是不能改配置，但能看所有东西、能保存配置、能重启设备。**

**Global Configuration `Router(config)#`** — 敲 `configure terminal` 进入。**从这里开始才能改配置**：设 hostname、配 ACL、配路由协议、进入子模式等。

**Sub-configuration Modes** — 从 Global Config 进入具体子项：

|命令|进入的子模式|提示符|用途|
|---|---|---|---|
|`interface GigabitEthernet0/0`|接口配置|`(config-if)#`|配 IP、shutdown、speed 等|
|`router ospf 1`|路由协议配置|`(config-router)#`|配 network、area 等|
|`line console 0`|线路配置|`(config-line)#`|配 password、logging 等|
|`vlan 10`|VLAN 配置|`(config-vlan)#`|配 VLAN name 等|

## 简单判断规则

**要"看"东西 → `Router#`（Privileged EXEC）：**

```
show running-config
show ip route
show interfaces
show crypto ipsec sa
copy running-config startup-config
ping 8.8.8.8
reload
```

**要"改"东西 → `Router(config)#` 或更深的子模式：**

```
hostname R1
ip route 0.0.0.0 0.0.0.0 203.0.113.1
interface GigabitEthernet0/0
 ip address 10.1.1.1 255.255.255.0
 no shutdown
```

## 怎么跳转

|从哪|到哪|敲什么|
|---|---|---|
|`>` → `#`|User → Privileged|`enable`|
|`#` → `(config)#`|Privileged → Global Config|`configure terminal`|
|`(config)#` → `#`|Global Config → Privileged|`end` 或 Ctrl+Z|
|`(config-if)#` → `(config)#`|子模式 → Global Config|`exit`|
|`(config-if)#` → `#`|子模式 → 直接回 Privileged|`end` 或 Ctrl+Z|
|`#` → `>`|Privileged → User|`disable`|

**一个常见操作习惯：** 在 config 模式下想执行 show 命令，可以加 `do` 前缀：

```
Router(config)# do show ip route
Router(config-if)# do show running-config
```

这样不用退出配置模式就能查看信息，实际工作中非常常用。