# 校园网络设计与实现（eNSP）

基于华为 eNSP 仿真平台搭建的双校区校园网络，采用「接入 — 核心 — 出口」三层架构，覆盖教学楼、图书馆、宿舍楼、服务器区四类业务场景，实现链路冗余、网关冗余、动态路由、地址自动分配、跨校区加密互联与服务器区访问控制。

---

## 1. 项目概述

| 项目 | 说明 |
|---|---|
| 拓扑规模 | 主校区 + 分校区，两校区经 Internet 互联 |
| 核心层 | 主校区双核心（LSW1 / LSW2），互为备份 |
| 接入层 | 主校区接入交换机双上行至双核心；分校区接入交换机上行至 LSW3 |
| 出口 | 主校区 R1、分校区 R2，通过 GRE over IPsec 建立隧道 |
| 业务区 | 教学楼、图书馆、宿舍楼、服务器区 |
| 地址规划 | 主校区 10.0.0.0/16，分校区 10.1.0.0/16 |

设计目标：

- 接入层双上行，链路冗余的同时实现流量负载分担
- 网关冗余，单台核心设备故障时业务不中断
- 全网三层互通，跨校区路由收敛快、拓扑变化影响范围可控
- 终端地址自动分配，并防止私接 DHCP 服务器
- 服务器区按最小权限开放服务，业务可用、管理面收敛

---

## 2. 网络架构

### 2.1 出口与互联

```
Internet (123.45.67.0/24)
├── R1  AR2240  123.45.67.1/24   主校区出口
│   ├── GE0/0/1  10.0.193.10/30 ── LSW1 Vlanif202 10.0.193.9/30
│   └── GE0/0/2  10.0.193.6/30  ── LSW2 Vlanif201 10.0.193.5/30
└── R2  AR2240  123.45.67.2/24   分校区出口
    └── GE0/0/1  10.1.192.1/30  ── LSW3 Vlanif200 10.1.192.2/30
```

两校区之间通过 GRE 隧道（192.168.0.0/30）承载 OSPF 报文，隧道流量由 IPsec 加密。

### 2.2 主校区

```
LSW1 (S5700) ── GE0/0/24 Trunk ── LSW2 (S5700)
  │                                  │
  ├── VLAN 300 ── 教学楼接入 (S3700)
  ├── VLAN 301 ── 图书馆接入 (S3700)
  ├── VLAN 302 ── 宿舍楼接入 (S3700)
  └── VLAN 303 ── 服务器区接入 (S3700)
                    ├── DHCP Server (AR1220)  10.0.192.12
                    ├── DNS & FTP             10.0.192.10
                    └── 学工系统               10.0.192.11
```

接入交换机同时上联 LSW1 与 LSW2，两台核心之间以 GE0/0/24 作为 Trunk 互联。

### 2.3 分校区

```
LSW3 (S5700)
  ├── VLAN 300 ── 教学楼接入 (S3700)
  ├── VLAN 301 ── 图书馆接入 (S3700)
  └── VLAN 302 ── 宿舍楼接入 (S3700)
```

---

## 3. 设备清单

| 角色 | 设备 | 型号 |
|---|---|---|
| 主校区核心 | LSW1 / LSW2 | S5700 |
| 分校区核心 | LSW3 | S5700 |
| 接入交换机 | 教学楼、图书馆、宿舍楼、服务器区 | S3700 |
| 出口路由器 | R1 / R2 | AR2240 |
| 服务器 | DHCP Server | AR1220 |
| 服务器 | DNS & FTP、学工系统 | Server |
| 终端 | PC1 – PC6 | PC |

---

## 4. 地址与 VLAN 规划

### 4.1 VLAN 编号规则

| 范围 | 用途 |
|---|---|
| 1 – 199 | 保留 |
| 200 – 299 | 核心与出口互联 |
| 300 – 499 | 接入层业务 |

### 4.2 主校区（10.0.0.0/16）

| VLAN | 区域 | 网段 | 网关 | 说明 |
|---|---|---|---|---|
| 300 | 教学楼 | 10.0.0.0/18 | 10.0.0.1 | |
| 301 | 图书馆 | 10.0.64.0/18 | 10.0.64.1 | |
| 302 | 宿舍楼 | 10.0.128.0/18 | 10.0.128.1 | |
| 303 | 服务器区 | 10.0.192.0/24 | 10.0.192.1 | |
| 200 | 核心互联 | 10.0.193.0/30 | — | LSW1 .1 / LSW2 .2 |
| 201 | LSW2 – R1 | 10.0.193.4/30 | — | LSW2 .5 / R1 .6 |
| 202 | LSW1 – R1 | 10.0.193.8/30 | — | LSW1 .9 / R1 .10 |

### 4.3 分校区（10.1.0.0/16）

| VLAN | 区域 | 网段 | 网关 | 说明 |
|---|---|---|---|---|
| 300 | 教学楼 | 10.1.0.0/18 | 10.1.0.1 | |
| 301 | 图书馆 | 10.1.64.0/18 | 10.1.64.1 | |
| 302 | 宿舍楼 | 10.1.128.0/18 | 10.1.128.1 | |
| 200 | LSW3 – R2 | 10.1.192.0/30 | — | LSW3 .2 / R2 .1 |

### 4.4 公网与隧道

| 用途 | 地址 |
|---|---|
| R1 公网口 GE0/0/0 | 123.45.67.1/24 |
| R2 公网口 GE0/0/0 | 123.45.67.2/24 |
| GRE 隧道 | 192.168.0.0/30（R1 .1 / R2 .2） |
| DNS | 10.0.192.10 |

---

## 5. 关键技术实现

### 5.1 MSTP 多实例与链路负载分担

两台核心之间划分不同实例的主备根桥，使接入层两条上行链路分别转发不同 VLAN，在消除环路的同时利用全部带宽。

```
# LSW1
stp instance 0 priority 4096
stp instance 1 root primary
stp instance 2 root secondary

# LSW2
stp instance 0 priority 8192
stp instance 1 root secondary
stp instance 2 root primary

# 两台核心配置一致
stp region-configuration
 instance 1 vlan 300 to 301
 instance 2 vlan 302 to 303
 active region-configuration
```

| 实例 | 承载 VLAN | 主根 | 备份根 |
|---|---|---|---|
| 0 | 其余 VLAN | LSW1 | LSW2 |
| 1 | 300 – 301 | LSW1 | LSW2 |
| 2 | 302 – 303 | LSW2 | LSW1 |

效果：教学楼、图书馆流量经 LSW1 上行，宿舍楼、服务器区流量经 LSW2 上行。

### 5.2 VRRP 网关冗余

四个业务网段均配置 VRRP，虚拟网关固定为 10.0.x.1，LSW1 优先级 120 作为 Master，LSW2 默认优先级 100 作为 Backup。

```
interface Vlanif300
 ip address 10.0.0.2 255.255.192.0
 vrrp vrid 1 virtual-ip 10.0.0.1
 vrrp vrid 1 priority 120
 dhcp select relay
 dhcp relay server-ip 10.0.192.12
```

| 接口 | 虚拟网关 | LSW1 | LSW2 |
|---|---|---|---|
| Vlanif300 | 10.0.0.1 | Master（120） | Backup（100） |
| Vlanif301 | 10.0.64.1 | Master（120） | Backup（100） |
| Vlanif302 | 10.0.128.1 | Master（120） | Backup（100） |
| Vlanif303 | 10.0.192.1 | Master（120） | Backup（100） |

终端网关地址唯一，核心切换后无需重新配置。

### 5.3 OSPF 多区域

主校区运行在骨干区域，分校区独立成区域，由出口路由器承担 ABR，避免非骨干区域直接互引。

| 设备 | 区域 | 宣告网段 |
|---|---|---|
| LSW1 | area 0 | 10.0.0.0/18、10.0.64.0/18、10.0.128.0/18、10.0.192.0/24、10.0.193.0/24 |
| LSW2 | area 0 | 同上 |
| R1 | area 0 | 10.0.193.4/30、10.0.193.8/30、123.45.67.0/24、192.168.0.0/30 |
| R2 | area 0 / area 1 | 192.168.0.0/30（area 0）、10.1.192.0/30（area 1） |
| LSW3 | area 1 | 10.1.0.0/18、10.1.64.0/18、10.1.128.0/18、10.1.192.0/30 |

```
# R2（ABR）
ospf 1
 area 0.0.0.0
  network 192.168.0.0 0.0.0.3
 area 0.0.0.1
  network 10.1.192.0 0.0.0.3
```

### 5.4 DHCP 中继与 Snooping

主校区地址池集中部署在服务器区的 DHCP Server（10.0.192.12），各业务网关配置中继；分校区由 R2 就近分配。

```
# 主校区 DHCP Server
ip pool vlan300
 gateway-list 10.0.0.1
 network 10.0.0.0 mask 255.255.192.0
 dns-list 10.0.192.10

# 核心交换机（各业务 Vlanif）
 dhcp select relay
 dhcp relay server-ip 10.0.192.12
```

接入层开启 DHCP Snooping，仅将上联口设为 trusted，防止终端私接 DHCP 服务器抢占地址分配。

```
dhcp snooping enable
interface GigabitEthernet0/0/1
 dhcp snooping trusted
```

### 5.5 GRE over IPsec 跨校区互联

GRE 负责承载两校区之间的 OSPF 报文，IPsec 负责加密隧道流量。采用传输模式，仅加密 GRE 报文本身，开销更小。

```
interface Tunnel0/0/1
 ip address 192.168.0.1 255.255.255.252
 tunnel-protocol gre
 source 123.45.67.1
 destination 123.45.67.2

acl number 3000
 rule 0 permit ip source 123.45.67.1 0 destination 123.45.67.2 0

ipsec proposal rtb
 encapsulation-mode transport
 esp authentication-algorithm sha2-512
 esp encryption-algorithm aes-256

ike proposal 1
 encryption-algorithm aes-cbc-128
 dh group14

ike peer rtb v1
 ike-proposal 1
 pre-shared-key cipher ******
 remote-address 123.45.67.2

ipsec policy rtb 1 isakmp
 security acl 3000
 ike-peer rtb
 proposal rtb

interface GigabitEthernet0/0/0
 ipsec policy rtb
```

### 5.6 服务器区 ACL 白名单

在服务器区接入交换机的双上联口配置入方向流量过滤，仅放行对外提供的服务端口，其余流量默认拒绝。终端可正常访问业务系统，但无法连接数据库或远程登录服务器。

```
acl name servicesACL
 rule 0  permit tcp destination 10.0.192.11 0 destination-port eq www
 rule 5  permit tcp destination 10.0.192.11 0 destination-port eq 443
 rule 10 permit tcp destination 10.0.192.10 0 destination-port eq ftp
 rule 15 permit tcp destination 10.0.192.10 0 destination-port eq ftp-data
 rule 20 permit tcp destination 10.0.192.10 0 destination-port eq domain
 rule 25 permit udp destination 10.0.192.10 0 destination-port eq dns
 rule 30 permit udp destination 10.0.192.12 0 destination-port eq bootps
 rule 35 permit icmp

interface GigabitEthernet0/0/1
 traffic-filter inbound acl name servicesACL
```

| 服务 | 目标 | 端口 |
|---|---|---|
| Web | 学工系统 10.0.192.11 | 80 / 443 |
| FTP | DNS & FTP 10.0.192.10 | 21 / 20 |
| DNS | 10.0.192.10 | 53（TCP / UDP） |
| DHCP | DHCP Server 10.0.192.12 | 67 |
| ICMP | 全网 | — |

---

## 6. 验证方式

| 验证项 | 命令 | 预期结果 |
|---|---|---|
| 生成树实例 | `display stp instance 1 brief` | 各接入交换机上联口分别处于不同实例的 Root/Designated |
| 网关冗余 | `display vrrp brief` | 四个 Vlanif 均为 Master，LSW2 为 Backup |
| 路由邻居 | `display ospf peer brief` | 所有邻居状态为 Full |
| 路由表 | `display ip routing-table` | 两校区业务网段互达，隧道网段存在 |
| 地址分配 | DHCP Server 上 `display ip pool` | 各地址池已分配地址，网关与 DNS 正确 |
| 跨校区连通 | `ping 10.1.0.x`、`ping 10.0.0.x` | 双向可达 |
| 域名解析 | `ping xuegong.exampleu.edu.cn` | 两校区均能解析并连通 |
| 隧道加密 | `display ipsec sa` | SA 建立成功，收发报文计数增长 |



---

## 7. 设计取舍

**地址按 /18 划分**：单网段约 1.6 万个可用地址（其实还可以进行按楼层再次划分，只不过为了简化拓扑减轻硬件负担没有再次划分）；服务器区单独使用 /24，便于 ACL 精确控制。

**采用 MSTP 多实例而非单实例**：单实例下双上行必然有一条被阻塞，带宽浪费一半；多实例让不同 VLAN 走不同核心，同时保留故障时的自动切换能力。

**采用 GRE over IPsec 而非纯 IPsec**：纯 IPsec 只能保护点对点流量，无法承载组播/单播路由协议；叠加 GRE 后 OSPF 报文可以在隧道内正常泛洪，两校区不需要维护静态路由。

**ACL 部署在服务器区接入层而非核心**：在靠近被保护对象的侧做过滤，流量在进入服务器区之前就被丢弃，不占用核心与上联链路带宽，策略也更容易随业务变化调整。

---

## 8. 文件说明

| 文件 | 说明 |
|---|---|
| `校园网.topo` | eNSP 拓扑文件，包含全部设备与链路 |
| `flash.efz`（各设备目录） | 设备 flash 快照，内含运行配置 |
| `华为设备GRE Over IPSec配置.md` | GRE over IPsec 配置步骤备忘 |
