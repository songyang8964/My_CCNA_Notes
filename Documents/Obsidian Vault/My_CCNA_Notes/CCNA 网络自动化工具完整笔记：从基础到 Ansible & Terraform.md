

---

# 第一章：为什么需要网络自动化

## 1.1 传统网络管理的痛点

传统网络管理方式：管理员通过 Console 线缆或 SSH 逐台登录设备，用 CLI 手动敲命令配置。

这种方式在设备数量少的时候没问题，但规模一大就暴露出严重缺陷：
#
**效率低下。** 你要给 200 台交换机都创建 VLAN 10。传统方式就是 SSH 登录第 1 台，敲命令，退出；SSH 登录第 2 台，敲同样的命令，退出……重复 200 次。一台花 2 分钟，200 台就是接近 7 个小时的纯体力劳动。

**容易出错。** 在第 150 台交换机上你手滑敲错了一个数字，VLAN ID 写成了 100 而不是 10。这台交换机的配置就和其他 199 台不一致了。而且你可能当时根本没发现，直到用户报障才知道出了问题。

**没有记录。** 你三个月前在某台交换机上改了一条 ACL，现在已经忘了改过什么、为什么改。running-config 里能看到当前配置，但看不到变更历史和变更原因。

**没法复现。** 你精心配好了一台交换机，现在要配另外 199 台一模一样的。你只能靠记忆或者笔记，一台一台重复操作。

**没法回滚。** 你改了一个配置导致网络故障，想恢复到之前的状态。但你不记得之前是什么配置了，也没有保存过历史版本。

**没法审计。** 领导问"谁在什么时候改了什么"，你答不上来。

## 1.2 网络自动化的目标

网络自动化就是用工具和代码来替代上面这些手动操作：

- **批量执行：** 一次操作同时推送到 200 台设备，不用逐台登录
- **一致性：** 所有设备执行完全相同的配置，不会因为手误出现差异
- **可追溯：** 配置文件存在 Git 里，每次变更都有记录
- **可复现：** 同一份配置文件可以反复使用，结果完全一致
- **可回滚：** Git 版本控制让你随时回退到任何历史版本

---

# 第二章：基础概念

## 2.1 Procedural vs Declarative（过程式 vs 声明式）

这两个概念是理解所有自动化工具的基础。

### Procedural（过程式）— 告诉系统"怎么做"

你写出一步一步的指令，系统按顺序执行。你必须自己想清楚每一步该做什么、先做什么后做什么。系统不会帮你判断，你写什么它就执行什么。

用你最熟悉的场景举例——在交换机上创建 VLAN 10：

```
system-view
vlan 10
 name Finance
 quit
interface GigabitEthernet 0/0/1
 port link-type access
 port default vlan 10
 quit
```

CCNA 网络自动化工具完整笔记：从基础到 Ansible & Terraform必须自己决定：先进 system-view，再创建 VLAN，再进接口，再改 link-type，再分配 VLAN。顺序、步骤、逻辑全部由你控制。如果 VLAN 10 已经存在了，你还是会执行 `vlan 10` 这条命令——系统不会帮你判断"已经存在了就跳过"。

### Declarative（声明式）— 告诉系统"要什么结果"

你只描述最终状态应该是什么样子，不写具体步骤。系统自己去判断当前状态和目标状态的差距，然后自动执行需要的操作。

同样的需求，用 Declarative 的方式表达：

```yaml
vlans:
  - id: 10
    name: Finance

interfaces:
  - name: GigabitEthernet0/0/1
    mode: access
    vlan: 10
```

你只是描述了"我要 VLAN 10 存在，名字叫 Finance，接口 GE0/0/1 是 access 模式属于 VLAN 10"。你没有写任何步骤。系统收到这个声明后自己去检查：VLAN 10 存不存在？不存在就创建；已经存在就跳过。接口当前是什么模式？不对就改；已经对了就什么都不做。

### 执行逻辑对比

**Procedural 的执行逻辑：**

```
你写的脚本:
  第1步: 创建 VLAN 10         → 系统执行: 创建 VLAN 10
  第2步: 命名为 Finance        → 系统执行: 命名为 Finance
  第3步: 进入接口              → 系统执行: 进入接口
  第4步: 设置 access 模式      → 系统执行: 设置 access 模式
  第5步: 分配 VLAN 10         → 系统执行: 分配 VLAN 10
```

你写了 5 步，系统就执行 5 步。不管当前状态是什么，每一步都会执行。

**Declarative 的执行逻辑：**

```
你声明的目标状态:
  VLAN 10 存在，名字 Finance
  GE0/0/1 是 access 模式，属于 VLAN 10

系统自动执行:
  检查 VLAN 10 → 已经存在 → 跳过
  检查 VLAN 名字 → 是 Finance → 跳过
  检查 GE0/0/1 → 当前是 trunk → 需要改 → 执行修改
  检查 VLAN 分配 → 当前是 VLAN 20 → 需要改 → 执行修改
```

系统比较"当前状态"和"目标状态"的差距，只执行必要的变更。已经符合的部分不会重复操作。

### 核心对比表

|维度|Procedural（过程式）|Declarative（声明式）|
|---|---|---|
|你告诉系统什么|**怎么做（How）**|**要什么结果（What）**|
|执行逻辑|按你写的步骤顺序执行|系统自动比较差异，只执行必要变更|
|幂等性|默认**不具备**，多次执行可能出问题|天然**具备**，多次执行结果一样|
|对当前状态的感知|不感知（除非你写代码去检查）|自动感知并计算差异|
|复杂度控制|你自己负责所有逻辑|系统替你处理大部分逻辑|
|代表工具|**Python, Bash, 传统 CLI**|**Puppet, Terraform, Ansible**|
|类似的 Cisco 概念|逐台设备 SSH 写命令|IBN / DNA Center 的 Intent|

---

## 2.2 幂等性（Idempotent）

### 定义

**一个操作执行一次和执行多次，产生的结果完全相同，这个操作就具备幂等性。**

### 具体例子

**有幂等性的操作：**

`interface GigabitEthernet 0/0/1` 下执行 `port default vlan 10`。执行一次，接口属于 VLAN 10。再执行一次，接口还是属于 VLAN 10。执行十次，结果都一样。这条命令是幂等的——它描述的是一个目标状态，不管你执行多少次，最终状态不变。

**没有幂等性的操作：**

假设你写了一个脚本，逻辑是"在 ACL 末尾追加一条 permit 规则"。执行一次，ACL 多了一条规则。执行第二次，ACL 又多了一条一模一样的规则。执行十次，ACL 里出现了十条重复的规则。每次执行都改变了系统状态，结果不同。这个操作就不是幂等的。

### 更直观的理解

- **幂等：** 把房间灯的开关设置到"开"的位置。不管你拨多少次到"开"，灯都是亮的，状态不变。
- **非幂等：** 按一下灯的按钮。按一下亮，再按一下灭，再按一下又亮。每次操作的结果取决于当前状态。

### 为什么 Declarative 天然幂等

Declarative 声明的是目标状态，系统每次执行时都会对比当前状态和目标状态。如果已经一致了，就什么都不做。所以不管执行多少次，最终状态永远是你声明的那个状态。

Procedural 声明的是步骤，系统不管当前状态是什么，每次都会把步骤执行一遍。如果步骤里有"追加""创建"这类操作，多次执行就会产生重复或冲突。

### "幂等"这个词的来源

来自数学术语。数学中幂等的定义是：一个运算对自身施加多次，结果和施加一次相同。数学表达式是 **f(f(x)) = f(x)**。

- "幂"= 乘方/次方（重复操作）
- "等"= 相等
- "幂等"= 做多少次重复操作，结果都相等

英文 Idempotent：**idem**（拉丁语"相同"）+ **potent**（拉丁语"操作"）= 相同的操作效果。

---

## 2.3 Infrastructure as Code（IaC，基础设施即代码）

### 核心思想

**把网络设备（或服务器、云资源等基础设施）的配置，用代码文件的形式来定义、存储、管理。**

"代码"不一定是 Python 脚本，更常见的是配置描述文件，比如 YAML、JSON、HCL 这类格式。这些文件描述了"我的基础设施应该是什么状态"。

举个例子，你把 200 台交换机的 VLAN 配置写成一个 YAML 文件：

```yaml
switches:
  - hostname: SW-Floor1
    vlans:
      - id: 10
        name: Finance
      - id: 20
        name: Engineering
    interfaces:
      - name: GE0/0/1
        mode: access
        vlan: 10

  - hostname: SW-Floor2
    vlans:
      - id: 10
        name: Finance
      - id: 20
        name: Engineering
    interfaces:
      - name: GE0/0/1
        mode: access
        vlan: 20
```

然后用自动化工具（Ansible、Terraform、Puppet 等）读取这个文件，自动把配置推送到设备上。

**这个 YAML 文件就是你的"基础设施"。你像管理代码一样管理它。**

### 为什么叫"即代码"

因为你对这些配置文件做的事情，和程序员对代码做的事情完全一样：

**版本控制。** 把配置文件放进 Git 仓库。每一次修改都有 commit 记录——谁改的、什么时候改的、改了什么、为什么改。想看三个月前的配置？`git log` 查一下就行。想回滚到上周的版本？`git revert` 一条命令。

**代码审查。** 有人要改网络配置，先提一个 Pull Request。团队成员 review 这个改动，确认没问题后再合并，然后自动下发到设备。不再是某个人半夜偷偷 SSH 上去改了一条 ACL。

**自动化测试。** 配置文件修改后，可以先在测试环境中自动验证，通过了再推到生产环境。

**可复现。** 要新建一个和现有环境一模一样的网络？把同一份配置文件对着新设备跑一遍就行，结果完全一致。

### IaC 的核心优势总结

|传统手动方式|Infrastructure as Code|
|---|---|
|配置存在设备本地 running-config 里|配置存在 Git 仓库的代码文件里|
|没有变更历史|每次变更都有 Git commit 记录|
|改错了靠记忆回滚|`git revert` 一键回滚|
|逐台手动操作|自动化批量下发|
|不同人配出来不一样|同一份代码文件保证一致性|
|无法审计|Git log 完整审计轨迹|

### 和 Declarative / Procedural 的关系

IaC 通常和 Declarative 结合使用，但也可以是 Procedural 的。两个概念描述的是不同维度：

- **Declarative / Procedural** = 表达方式（怎么告诉系统做事）
- **IaC** = 管理方式（用代码文件来管理基础设施，而不是手动操作）

两者不冲突，可以组合。

---

## 2.4 Push vs Pull 模型

自动化工具把配置送到目标设备有两种方式：

### Push（推送模式）

管理员在控制机上主动运行工具，工具把配置推送到目标设备。你不运行工具，它就什么都不做。

```
管理员手动触发
      ↓
[控制机] ——推送配置——→ [设备1]
         ——推送配置——→ [设备2]
         ——推送配置——→ [设备3]
```

代表工具：**Ansible、Terraform**

### Pull（拉取模式）

目标设备上安装了 Agent 软件，Agent 定期（比如每 30 分钟）主动去服务器上拉取最新配置。如果发现当前配置和服务器上定义的目标状态不一致，Agent 自动纠正。

```
[设备1上的Agent] ——定期拉取——→ [服务器]
[设备2上的Agent] ——定期拉取——→ [服务器]
[设备3上的Agent] ——定期拉取——→ [服务器]
```

代表工具：**Puppet、Chef**

### 对比

|维度|Push|Pull|
|---|---|---|
|触发方式|管理员手动触发|Agent 自动定期执行|
|是否需要 Agent|不需要|需要|
|持续合规|不运行就不检查|Agent 持续确保设备符合目标状态|
|代表工具|Ansible, Terraform|Puppet, Chef|

---

## 2.5 Agent vs Agentless

### Agent（有代理）

需要在每台被管理的设备上安装一个客户端软件（Agent）。Agent 负责与管理服务器通信、接收配置、执行变更。

优势：Agent 可以持续运行，定期检查设备状态，自动纠偏。 劣势：每台设备都要装 Agent，部署和维护成本高；有些网络设备（老旧交换机）不支持安装 Agent。

代表工具：**Puppet（需要 Puppet Agent）、Chef（需要 Chef Client）**

### Agentless（无代理）

被管理的设备上不需要安装任何额外软件。管理工具通过设备已有的管理接口（SSH、NETCONF、API）直接连接并操作。

优势：部署简单，不用在目标设备上装任何东西；对网络设备特别友好，因为大多数网络设备都支持 SSH。 劣势：不运行工具就无法持续监控设备状态。

代表工具：**Ansible（通过 SSH/NETCONF）、Terraform（通过 API）**

---

## 2.6 YAML 基础

Ansible 的 Playbook 用 YAML 格式编写，所以你需要能看懂 YAML。YAML 是一种人类可读的数据格式，用缩进表示层级关系。

### 基本语法

```yaml
# 这是注释

# 键值对（Key: Value）
hostname: SW-Floor1
vlan_id: 10
enabled: true

# 列表（用 - 开头）
vlans:
  - 10
  - 20
  - 30

# 嵌套结构（用缩进表示层级）
switch:
  hostname: SW-Floor1
  vlans:
    - id: 10
      name: Finance
    - id: 20
      name: Engineering
  interfaces:
    - name: GE0/0/1
      mode: access
      vlan: 10
```

### YAML 的关键规则

- **用空格缩进**，不能用 Tab（这是 YAML 最常见的错误）
- 缩进的空格数量必须一致（通常用 2 个空格）
- 列表项用 `-` 开头
- 键和值之间用 `:` （冒号加空格）分隔
- 字符串通常不需要加引号
- `true` / `false` 表示布尔值

### 为什么 Ansible 选择 YAML

YAML 的最大优势是**人类可读性**。即使你没有编程基础，看一个 YAML 文件也能大致理解它在描述什么。相比之下，Puppet 用的 Ruby DSL 和 Terraform 用的 HCL 都有更多的编程语法，学习门槛更高。

---

# 第三章：Ansible

## 3.1 知识点定位

- 属于 CCNA 200-301 "Network Automation and Programmability"（10%）域
- **高频必考**，在所有自动化工具中 CCNA 对 Ansible 覆盖最详细
- 考试通常出 2-4 道相关题目
- 典型考法：归类题（Declarative/Agentless/Push）、组件识别题（Playbook/Play/Task/Module 的层级）、给 Playbook 片段问做什么

## 3.2 Ansible 是什么

**Ansible 是 Red Hat 公司维护的一个开源自动化工具。** 核心用途是：批量管理设备的配置，实现自动化部署和编排。

### 核心特征一览

|特征|说明|
|---|---|
|开发公司|**Red Hat**|
|方式|**Declarative**（CCNA 标准答案）|
|Agent|**Agentless（无代理）**|
|执行模式|**Push（推送模式）**|
|Playbook 编写语言|**YAML**|
|Ansible 本身的开发语言|**Python**|
|通信协议|**SSH / NETCONF**|
|幂等性|具备|

### 关于 Declarative 的补充说明

Ansible 同时有 Procedural 和 Declarative 的特征。Playbook 中的 Task 按照 YAML 中写的顺序依次执行（这是 Procedural 特征）。但每个 Task 内部，Module 的行为是 Declarative 的（声明目标状态，自动判断是否需要变更）。

**CCNA 考试通常将 Ansible 归为 Declarative，直接选 Declarative 就对了。**

## 3.3 Ansible 的架构

Ansible 的架构非常简单，只有两个角色：

**Control Node（控制节点）** — 运行 Ansible 的机器，通常是管理员的 Linux 工作站或一台专用管理服务器。Ansible 只能安装在 Linux/macOS 上，不能安装在 Windows 上（Windows 可以作为被管理的目标，但不能作为 Control Node）。

**Managed Nodes（被管节点）** — 被 Ansible 管理的设备。可以是 Linux 服务器、Windows 服务器、网络交换机、路由器等。它们上面不需要安装任何 Ansible 软件。

```
                          SSH / NETCONF
[Control Node] ────────────────────────→ [Managed Node 1: 交换机]
  (Ansible)    ────────────────────────→ [Managed Node 2: 交换机]
               ────────────────────────→ [Managed Node 3: 路由器]
               ────────────────────────→ [Managed Node 4: Linux服务器]
```

没有中间服务器，没有数据库，没有 Agent。就是一台控制机通过 SSH 直接连到目标设备执行操作。

## 3.4 核心组件详解（考试重点）

Ansible 有五个核心组件，考试非常喜欢考它们各自的定义和层级关系。

### Inventory（清单）— "管谁？"

定义了 Ansible 要管理哪些设备。就是一个列表，记录了设备的 IP/主机名、分组信息、连接方式等。

```ini
[switches]
sw1 ansible_host=192.168.1.1
sw2 ansible_host=192.168.1.2
sw3 ansible_host=192.168.1.3

[routers]
r1 ansible_host=10.0.0.1
r2 ansible_host=10.0.0.2
```

- `[switches]` 和 `[routers]` 是组名，把设备按角色分组
- `ansible_host` 指定设备的实际 IP 地址
- 之后在 Playbook 中可以用组名来指定"对哪些设备执行操作"

Inventory 可以是 INI 格式（如上）或 YAML 格式。

### Module（模块）— "做什么具体操作？"

Module 是 Ansible 实际执行操作的最小单元。每个 Module 负责一个具体的功能。Ansible 内置了几千个 Module，也可以自己开发。

网络相关的常见 Module：

|Module|作用|
|---|---|
|`ios_config`|向 Cisco IOS 设备推送配置命令|
|`ios_command`|在 Cisco IOS 设备上执行 show 命令并返回结果|
|`ios_vlan`|管理 Cisco IOS 设备上的 VLAN（创建/删除/修改）|
|`nxos_config`|向 Cisco NX-OS 设备推送配置|
|`eos_config`|向 Arista EOS 设备推送配置|

Module 是 Python 写的，但你作为使用者不需要关心 Module 的内部代码，只需要知道调用哪个 Module、传什么参数。

### Task（任务）— "做一件什么事？"

一个 Task 就是"调用一个 Module 去执行一个具体操作"。Task 包含一个 Module 名称和传给这个 Module 的参数。

```yaml
- name: Create VLAN 10
  ios_vlan:
    vlan_id: 10
    name: Finance
    state: present
```

逐行解释：

- `name: Create VLAN 10` — 这个 Task 的描述性名称，方便你看输出日志时知道在执行什么
- `ios_vlan:` — 调用的 Module 名称
- `vlan_id: 10` — 传给 Module 的参数：VLAN ID 是 10
- `name: Finance` — 传给 Module 的参数：VLAN 名称是 Finance
- `state: present` — 传给 Module 的参数：确保这个 VLAN 存在（如果不存在就创建，已存在就跳过）

注意 `state: present` 这个参数——它体现了 Declarative 的理念。你说的是"VLAN 10 应该存在"，不是"创建 VLAN 10"。如果 VLAN 10 已经存在了，Module 不会重复创建。这就是幂等性。

`state` 的另一个常见值是 `absent`，表示"确保这个东西不存在"——如果存在就删除，不存在就跳过。

### Play（剧本段落）— "在谁身上做哪些事？"

一个 Play 把一组 Task 和一组目标设备关联起来。它定义了"在哪些设备上执行哪些任务"。

```yaml
- name: Configure switches
  hosts: switches
  tasks:
    - name: Create VLAN 10
      ios_vlan:
        vlan_id: 10
        name: Finance
        state: present

    - name: Create VLAN 20
      ios_vlan:
        vlan_id: 20
        name: Engineering
        state: present
```

逐行解释：

- `name: Configure switches` — 这个 Play 的描述性名称
- `hosts: switches` — 目标设备是 Inventory 中 `switches` 组的所有设备
- `tasks:` — 下面列出要执行的 Task 列表
- 然后是两个 Task，分别创建 VLAN 10 和 VLAN 20

这个 Play 的含义是：在 Inventory 中 `switches` 组的所有设备（sw1, sw2, sw3）上，执行两个 Task。

### Playbook（剧本）— "完整的自动化任务定义"

Playbook 是一个 YAML 文件，包含一个或多个 Play。它是你实际运行的文件。

```yaml
# playbook.yml — 这就是一个 Playbook

- name: Configure switches        # 第一个 Play
  hosts: switches
  tasks:
    - name: Create VLAN 10
      ios_vlan:
        vlan_id: 10
        name: Finance
        state: present

    - name: Create VLAN 20
      ios_vlan:
        vlan_id: 20
        name: Engineering
        state: present

- name: Configure routers          # 第二个 Play
  hosts: routers
  tasks:
    - name: Set hostname
      ios_config:
        lines:
          - hostname CORE-RTR
```

这个 Playbook 包含两个 Play：第一个在 switches 组上创建 VLAN，第二个在 routers 组上设置 hostname。

运行方式：`ansible-playbook playbook.yml`

### 层级关系总结（必须记清楚）

```
Playbook（一个 YAML 文件 — 你实际运行的东西）
  └── Play 1（在 switches 组上执行）
  │     ├── Task 1（调用 ios_vlan Module → 创建 VLAN 10）
  │     └── Task 2（调用 ios_vlan Module → 创建 VLAN 20）
  └── Play 2（在 routers 组上执行）
        └── Task 1（调用 ios_config Module → 设置 hostname）
```

**Playbook 包含多个 Play，Play 包含多个 Task，Task 调用一个 Module。**

各组件速记：

|组件|回答的问题|说明|
|---|---|---|
|**Inventory**|管谁？|设备清单，记录 IP/主机名、分组|
|**Module**|做什么具体操作？|最小执行单元，Python 编写|
|**Task**|做一件什么事？|调用一个 Module + 传参数|
|**Play**|在谁身上做哪些事？|关联目标设备和 Task 列表|
|**Playbook**|完整的自动化任务定义|包含一个或多个 Play 的 YAML 文件|

## 3.5 Ansible 与网络设备的通信方式

Ansible 作为 Agentless 工具，通过网络设备已有的管理接口来通信：

|目标设备类型|通信协议|`ansible_connection` 参数值|
|---|---|---|
|Linux 服务器|SSH|`ssh`|
|Cisco IOS/IOS-XE|SSH（CLI）|`network_cli`|
|Cisco NX-OS/IOS-XR|SSH 或 NETCONF|`network_cli` 或 `netconf`|
|Juniper|NETCONF|`netconf`|
|REST API 设备|HTTPS|`httpapi`|

`ansible_connection` 是在 Inventory 或 Playbook 中指定的参数，告诉 Ansible 用什么方式连接目标设备。

对于网络设备，最常用的是 `network_cli`——Ansible 通过 SSH 连接到设备，然后发送 CLI 命令，和你手动 SSH 登录敲命令是同一个通道，但由 Ansible 自动完成。

## 3.6 关键文件和命令

### 文件

|文件|作用|格式|
|---|---|---|
|`playbook.yml`|定义自动化任务（Task、Play）|YAML|
|`inventory` 或 `hosts`|定义被管设备列表和分组|INI 或 YAML|
|`ansible.cfg`|Ansible 的全局配置文件|INI|

### 命令

|命令|作用|
|---|---|
|`ansible-playbook playbook.yml`|运行一个 Playbook|
|`ansible all -m ping`|对所有设备执行连通性测试（不是 ICMP ping）|
|`ansible-inventory --list`|查看 Inventory 中的设备列表|

## 3.7 执行流程

当你运行 `ansible-playbook playbook.yml` 时，完整流程如下：

1. **读取 Inventory** — Ansible 确定目标设备列表
2. **执行第一个 Play** — 对 Play 中指定的每台设备，通过 SSH/NETCONF 建立连接
3. **按顺序执行 Task** — 每个 Task 调用对应的 Module。Module 判断当前状态是否需要变更（Declarative），需要就执行，已符合就跳过
4. **执行下一个 Play** — 第一个 Play 完成后开始第二个 Play
5. **输出汇总报告** — 每台设备每个 Task 的结果：
    - `ok` = 已符合目标状态，未做变更
    - `changed` = 执行了变更
    - `failed` = 执行失败

## 3.8 Ansible 常见考点与陷阱

**考点 1：是否需要 Agent？** 不需要（Agentless）。通过 SSH 直接连接。Puppet 和 Chef 需要 Agent，Ansible 和 Terraform 不需要。

**考点 2：Playbook 用什么语言写？** **YAML。** 不是 Python。Ansible 本身是用 Python 开发的，但 Playbook 用 YAML 编写。考题经常把 Python 和 YAML 混在选项里。**Ansible 开发语言 = Python，Playbook 编写语言 = YAML。**

**考点 3：组件层级关系？** Playbook > Play > Task > Module。考题可能给一段 YAML 片段问某部分是什么。"定义目标设备和任务关联的是什么？"→ Play。"执行具体操作的最小单元是什么？"→ Module。

**考点 4：Push 还是 Pull？** Ansible 是 Push。管理员主动运行 Playbook 推送配置。Puppet 是 Pull，Agent 定期拉取。

**考点 5：Declarative 还是 Procedural？** CCNA 标准答案 Declarative。不要因为 Playbook 中 Task 按顺序执行就选 Procedural。

## 3.9 可以不深挖的内容

- Ansible Galaxy（社区 Module 仓库）
- Ansible Vault（加密敏感数据）
- Ansible Tower / AWX（企业级 Web GUI）
- Jinja2 模板语法
- Ansible Role 的目录结构
- Handler 和 Notify 机制
- 自定义 Module 开发
- ansible.cfg 详细配置项

---

# 第四章：Terraform

## 4.1 知识点定位

- 属于 CCNA "Network Automation and Programmability"（10%）域
- **重要程度：次要**，但需要知道它是什么以及和 Ansible/Puppet 的区别
- 考试中主要作为工具归类题的选项出现，1-2 道题

## 4.2 Terraform 是什么

**Terraform 是 HashiCorp 公司开发的一个开源 Infrastructure as Code（IaC）工具。** 核心用途是：用代码文件定义基础设施资源（服务器、网络、存储、云服务等），然后自动创建、修改、删除这些资源。

### 核心特征一览

|特征|说明|
|---|---|
|开发公司|**HashiCorp**|
|方式|**Declarative**|
|Agent|**Agentless**|
|执行模式|**Push（按需执行）**|
|配置语言|**HCL（HashiCorp Configuration Language）**|
|开发语言|**Go**|
|通信方式|**API 调用**|
|幂等性|具备（通过 State File 实现）|
|核心强项|**云基础设施的创建和生命周期管理**|

### Terraform 和 Ansible 的本质区别

两者都是 IaC 工具，都是 Agentless，都是 Declarative。但它们解决的问题不同：

- **Terraform** 解决的是"**这个东西存不存在**"——在 AWS 上创建一个 VPC、在 Azure 上开一台虚拟机、在 GCP 上建一个存储桶。它管的是基础设施资源的生命周期（创建 → 修改 → 销毁）。
- **Ansible** 解决的是"**这个东西怎么配置**"——给已经存在的交换机推送 VLAN 配置、给已经存在的服务器安装软件、给已经存在的路由器修改 BGP 参数。

简单说：**Terraform 管"有没有"，Ansible 管"怎么配"。**

## 4.3 核心工作流程

Terraform 的使用分为三个清晰的阶段：

### 第一步：Write（编写配置文件）

用 HCL 编写 `.tf` 文件，声明你想要的基础设施目标状态。

```hcl
resource "aws_vpc" "my_vpc" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "my_subnet" {
  vpc_id     = aws_vpc.my_vpc.id
  cidr_block = "10.0.1.0/24"
}
```

这个文件只是在说"我要一个 VPC，地址段是 10.0.0.0/16；我要一个子网，属于这个 VPC，地址段是 10.0.1.0/24"。没有任何步骤，纯粹的声明。

### 第二步：Plan（生成执行计划）

运行 `terraform plan`，Terraform 会：

1. 读取 `.tf` 配置文件（目标状态）
2. 查询当前的真实状态（通过 API 去云平台查）
3. 比较两者的差异
4. 生成执行计划，告诉你"我准备做什么"

```
+ aws_vpc.my_vpc will be created
+ aws_subnet.my_subnet will be created

Plan: 2 to add, 0 to change, 0 to destroy.
```

**这一步不会做任何实际改动。** 只是预览，让你确认 Terraform 打算做的事情是不是你想要的。这是一个安全机制——先看再做。

### 第三步：Apply（执行变更）

确认计划没问题后，运行 `terraform apply`。Terraform 按照计划通过 API 调用云平台，实际创建/修改/删除资源。

### 流程总结

```
Write（写 .tf 文件声明目标状态）
  ↓
Plan（对比目标状态 vs 当前状态，生成变更计划，不实际执行）
  ↓
Apply（确认后执行变更）
```

## 4.4 核心概念详解

### State File（状态文件）

每次 Terraform 执行完操作后，会把当前基础设施的状态保存到一个 `terraform.tfstate` 文件中。

下次执行 `terraform plan` 时，Terraform 用这个 State File 来了解"当前状态是什么"，然后和你的 `.tf` 配置文件（目标状态）做对比，计算差异。

```
.tf 配置文件       →  你想要的目标状态
terraform.tfstate  →  当前的实际状态
terraform plan     →  比较两者差异，生成变更计划
```

**这是 Terraform 实现幂等性的关键机制。** 如果你不改 `.tf` 文件，反复执行 `terraform plan`，Terraform 发现目标状态和当前状态一致，就会告诉你"没有变更需要执行"。

### Provider（提供者）

Terraform 本身是一个通用框架，它不知道怎么和 AWS 通信，也不知道怎么和 Azure 通信。具体的通信能力由 Provider 插件提供。

Provider 告诉 Terraform"怎么和某个平台的 API 交互"。你要管理什么平台，就安装对应的 Provider：

|Provider|管理什么|
|---|---|
|aws|AWS 云资源（EC2、VPC、S3 等）|
|azurerm|Azure 云资源|
|google|GCP 云资源|
|cisco_ios|Cisco IOS 设备|
|cisco_aci|Cisco ACI 数据中心|
|meraki|Cisco Meraki 设备|

Terraform 理论上可以管理任何有 API 的平台，只要有对应的 Provider。

## 4.5 关键命令

|命令|作用|对应阶段|
|---|---|---|
|`terraform init`|初始化项目，下载 Provider 插件|准备|
|`terraform plan`|预览变更（不实际执行）|Plan|
|`terraform apply`|执行变更，实际创建/修改/删除资源|Apply|
|`terraform destroy`|删除所有由 Terraform 管理的资源|销毁|

## 4.6 Terraform 常见考点与陷阱

**考点 1：Declarative 还是 Procedural？** Declarative。`.tf` 文件描述的是目标状态，不是步骤。

**考点 2：是否需要 Agent？** 不需要（Agentless）。通过 API 与目标平台通信。

**考点 3：和 Ansible 的区别？** Terraform 管基础设施的创建和销毁（"有没有"），Ansible 管已有设备的配置（"怎么配"）。如果题目问"哪个工具更适合管理云基础设施的生命周期？"答案是 Terraform。

**考点 4：幂等性？** 具备。通过 State File 跟踪当前状态，只执行必要变更。

**考点 5：配置语言？** HCL。不是 YAML（Ansible），不是 Ruby DSL（Puppet）。

## 4.7 可以不深挖的内容

- HCL 具体语法和写法
- Module、Variable、Output 等高级概念
- State File 远程存储（Remote Backend）和锁机制
- Terraform Cloud / Terraform Enterprise
- Provider 开发
- `terraform import` 等高级命令

---

# 第五章：终极对比（考试核心）

## 5.1 四大工具全面对比

|维度|Ansible|Puppet|Terraform|Chef|
|---|---|---|---|---|
|开发公司|**Red Hat**|Puppet Inc.|**HashiCorp**|Progress|
|方式|**Declarative**|Declarative|**Declarative**|Procedural|
|Agent|**Agentless**|**需要 Agent**|**Agentless**|**需要 Agent**|
|配置语言|**YAML**|Puppet DSL (Ruby)|**HCL**|Ruby DSL|
|开发语言|**Python**|Ruby|**Go**|Ruby|
|通信方式|**SSH / NETCONF**|Agent → Server (HTTPS)|**API**|Agent → Server (HTTPS)|
|执行模式|**Push**|**Pull**|**Push**|**Pull**|
|核心强项|**网络/服务器配置管理**|持续配置合规|**云基础设施生命周期**|服务器配置管理|
|网络设备支持|**强**|一般|有但非主战场|弱|
|幂等性|具备|具备|具备|具备|
|学习曲线|**低（YAML 易读）**|中|中|高|

## 5.2 考试必记分类

### 需要 Agent vs 不需要 Agent

|需要 Agent|不需要 Agent（Agentless）|
|---|---|
|**Puppet**|**Ansible**|
|**Chef**|**Terraform**|

### Push vs Pull

|Push（推送，手动触发）|Pull（拉取，Agent 自动）|
|---|---|
|**Ansible**|**Puppet**|
|**Terraform**|**Chef**|

### 配置语言对应

|工具|配置语言|
|---|---|
|Ansible|**YAML**|
|Puppet|**Puppet DSL (Ruby)**|
|Terraform|**HCL**|
|Chef|**Ruby DSL**|

## 5.3 Ansible vs Terraform 核心区分

|维度|Ansible|Terraform|
|---|---|---|
|核心职责|**配置管理**（在已有设备上推送配置）|**基础设施生命周期**（创建/修改/销毁资源）|
|简单理解|管"**怎么配**"|管"**有没有**"|
|状态管理|无 State File|**有 State File**|
|配置语言|YAML|HCL|
|通信方式|SSH / NETCONF|API|
|网络设备支持|强（大量网络 Module）|有但非主战场|

## 5.4 Ansible vs Puppet 核心区分

|维度|Ansible|Puppet|
|---|---|---|
|Agent|**不需要**|**需要安装 Puppet Agent**|
|执行模式|**Push（你手动触发）**|**Pull（Agent 每30分钟自动拉取）**|
|配置语言|**YAML**|**Puppet DSL (Ruby)**|
|持续合规|不运行就不检查|Agent 持续运行，自动纠偏|
|学习曲线|低|中|

---

# 第六章：快速记忆总结

## 核心概念口诀

```
Procedural = How（怎么做）→ 写步骤 → 代表：Python, CLI
Declarative = What（要什么）→ 写结果 → 代表：Puppet, Terraform, Ansible

幂等 = 执行多少次结果都一样 → Declarative 天然幂等

IaC = 用代码文件管理基础设施 → 版本控制 + 自动化 + 可复现
```

## 工具核心口诀

```
Ansible:   无Agent  推Push  YAML写  SSH连   管配置   Red Hat出品
Terraform: 无Agent  推Push  HCL写   API连   管生死   HashiCorp出品
Puppet:    有Agent  拉Pull  Ruby写  HTTPS连  管合规   Puppet Inc出品
Chef:      有Agent  拉Pull  Ruby写  HTTPS连  管配置   Progress出品
```

## Ansible 组件口诀

```
Inventory = 管谁（设备清单）
Module    = 做什么（最小执行单元）
Task      = 用哪个 Module 做一件事
Play      = 在谁身上做哪些 Task
Playbook  = 包含所有 Play 的 YAML 文件（你实际运行的东西）

层级：Playbook > Play > Task > Module
```

## Terraform 流程口诀

```
Write（写 .tf 文件）→ Plan（预览变更）→ Apply（执行变更）
State File 记录当前状态 → 和 .tf 文件对比 → 只执行差异部分 → 幂等
Provider 插件 → 对接不同平台的 API
```

## Ansible 两个"语言"不要搞混

```
Ansible 开发语言 = Python（你不需要写）
Playbook 编写语言 = YAML（你需要会读）
```