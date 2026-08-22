---
title: H3C交换机配置笔记
published: 2026-08-22
description: H3C Comware 7交换机配置全记录,含网络拓扑与VLAN规划、核心与POE交换机配置详解、常用命令速查与MAC定位实战。
category: 环境配置
tags:
  - 交换机
  - VLAN
  - 网络
slug: h3c-switch-config-notes
---

# H3C交换机配置笔记

>
> **设备型号**: H3C Comware 7 系列
> **涉及设备**: Core-Switch（核心交换机）、POE-Switch（POE交换机）

---

## 一、网络拓扑

```text
                        ┌──────────────┐
                        │    路由器     │
                        │ 192.168.43.253│
                        └──────┬───────┘
                               │ VLAN 43 (Trunk, PVID 43)
                               │
                   ┌───────────┴───────────┐
                   │ 核心交换机 Core-Switch │
                   │ 管理IP: 192.168.43.254 │
                   │ 版本: 7.1.070 R3507P35│
                   └───────────┬───────────┘
                               │ 1/0/22 (Trunk, VLAN all)
                               │
                   ┌───────────┴───────────┐
                   │ POE交换机  POE-Switch  │
                   │ 管理IP: 192.168.48.252 │
                   │ 版本: 7.1.070 R3507P28│
                   └───┬───┬───┬───┬───┬───┘
                       │   │   │   │   │
          ┌────────────┘   │   │   │   └────────────┐
          │ 1/0/1          │   │   │   1/0/19-20     │
          │ (VLAN 45)      │   │   │   (Trunk,AP)    │
     ┌────┴────┐    1/0/2~18  │   │                 │
     │  录像机  │   摄像头×17  │   │                 │
     └─────────┘   (VLAN 45)  │   │                 │
                              │   │             ┌───┴───┐
                    1/0/21~23 │   │             │ AP-01 │
                    (VLAN 43) │   │             │ AP-02 │
                              │   │             └───────┘
                              │   │
                    核心1/0/1~20: 业务端口 (VLAN 48, Access)
                    核心1/0/21:   预留 (Trunk)
                    核心1/0/23:   预留 (VLAN 48)
```

**VLAN 规划表：**

|VLAN ID|网段|网关|用途|DHCP|
| ---------| -----------------| ----------------| ------------------------------| ------------------|
|1|-|-|禁用（默认VLAN，已shutdown）|无|
|43|192.168.43.0/24|192.168.43.254|AP和路由器互联网络|192.168.43.5-119|
|45|192.168.45.0/24|192.168.45.254|监控网络（摄像头+NVR）|192.168.45.5-119|
|48|192.168.48.0/24|192.168.48.254|业务网络（办公）|192.168.48.5-119|

**管理地址池**: 192.168.43.120-127 和 192.168.48.120-127（保留不纳入DHCP，且享有ACL通行特权）

---

## 二、H3C Web管理初始化配置（修正版）

> **原始笔记错误**: 原文第11行缺少 `class manage` 关键字，在 Comware 7 中是必选参数，不指定会导致用户创建失败。

```text
<H3C> system-view
[H3C] interface M-GigabitEthernet 0/0/0                     # 进入专用管理接口（MGMT口）
[H3C-M-GigabitEthernet0/0/0] ip address 192.168.1.10 24     # 设置管理IP（掩码也可用CIDR）
[H3C-M-GigabitEthernet0/0/0] undo shutdown                   # 确保接口开启
[H3C-M-GigabitEthernet0/0/0] quit

[H3C] ip http enable                                          # 开启HTTP服务
[H3C] ip https enable                                         # 开启HTTPS服务（强烈推荐）

[H3C] local-user admin class manage                           # ⚠️ 原文遗漏 class manage，已修正
[H3C-luser-manage-admin] password simple YourStrongPassword   # 设置密码（simple明文/cipher密文）
[H3C-luser-manage-admin] service-type http https ssh telnet terminal  # 授予全部服务权限
[H3C-luser-manage-admin] authorization-attribute user-role network-admin  # 赋予最高管理权限
[H3C-luser-manage-admin] quit
```

**修正说明：**

|行号|原文|问题|修正|
| ------| ------| -------------------------------------------------| ------|
|11|`local-user admin`|Comware 7 必须指定 `class manage`，否则无法创建本地管理用户|`local-user admin class manage`|
|13|`service-type http https`|功能不完整，生产环境建议同时开启 SSH|`service-type http https ssh telnet terminal`|

> **提示**: 如果设备已接入云平台（如 H3C Oasis），可通过 `cloud-management server domain oasis.h3c.com` 实现云端管理，无需单独配置管理IP。

---

## 三、核心交换机配置详解

### 3.1 完整运行配置

```text
<Core-Switch>dis cu
#
 version 7.1.070, Release 3507P35
#
 sysname Core-Switch
#
 irf mac-address persistent timer
 irf auto-update enable
 undo irf link-delay
 irf member 1 priority 1
#
 dhcp enable
#
 dns server 223.5.5.5
#
 lldp global enable
#
 password-recovery enable
 pex working-mode switch slot 1
#
vlan 1
 description Default-VLAN-Disabled
#
vlan 43
 description 路由器互联和AP网络
#
vlan 45
 description 监控网络
#
vlan 48
 description 业务网络
#
 stp global enable
#
dhcp server ip-pool vlan43
 gateway-list 192.168.43.254
 network 192.168.43.0 mask 255.255.255.0
 address range 192.168.43.5 192.168.43.119
 dns-list 223.5.5.5 119.29.29.29
#
dhcp server ip-pool vlan45
 gateway-list 192.168.45.254
 network 192.168.45.0 mask 255.255.255.0
 address range 192.168.45.5 192.168.45.119
 dns-list 223.5.5.5 119.29.29.29
#
dhcp server ip-pool vlan48
 gateway-list 192.168.48.254
 network 192.168.48.0 mask 255.255.255.0
 address range 192.168.48.5 192.168.48.119
 dns-list 223.5.5.5 119.29.29.29
#
interface NULL0
#
interface Vlan-interface1
 shutdown
 undo dhcp select server
 dhcp client identifier ascii 90742e9456f0-VLAN0001
 ipv6 address dhcp-alloc
#
interface Vlan-interface43
 description AP网络网关
 ip address 192.168.43.254 255.255.255.0
 packet-filter 3000 inbound
#
interface Vlan-interface45
 description 监控网络网关
 ip address 192.168.45.254 255.255.255.0
 packet-filter 3000 inbound
#
interface Vlan-interface48
 description 业务网络网关
 ip address 192.168.48.254 255.255.255.0
 packet-filter 3000 inbound
#
interface GigabitEthernet1/0/1  to 1/0/20    # 业务端口（共20个）
 port access vlan 48
#
interface GigabitEthernet1/0/21               # 预留Trunk口
 port link-type trunk
 port trunk permit vlan all
#
interface GigabitEthernet1/0/22               # ↓ 下联POE交换机
 port link-type trunk
 port trunk permit vlan all
#
interface GigabitEthernet1/0/23               # 预留Access口
 port access vlan 48
#
interface GigabitEthernet1/0/24               # ↑ 上联路由器
 port link-type trunk
 port trunk permit vlan all
 port trunk pvid vlan 43                      # 未打标签的帧归入VLAN 43
#
interface Ten-GigabitEthernet1/0/25 to 1/0/28 # 万兆口（未使用）
#
 scheduler logfile size 16
#
line class aux
 user-role network-admin
#
line class vty
 user-role network-operator
#
line aux 0
 user-role network-admin
#
line vty 0 63
 authentication-mode scheme
 user-role network-operator
 protocol inbound ssh
#
 ip route-static 0.0.0.0 0 192.168.43.253     # 默认路由指向路由器
#
 ssh server enable
 ssh server acl 2000                           # SSH仅允许ACL 2000中的源IP
#
 arp ip-conflict log prompt
 arp user-ip-conflict record enable
#
 ntp-service enable
 ntp-service source Vlan-interface43
 ntp-service unicast-server ntp.aliyun.com
 ntp-service unicast-server ntp.tencent.com
#
acl basic 2000
 description ssh限制
 rule 5 permit source 192.168.48.120 0.0.0.7    # 允许 192.168.48.120~127
 rule 10 permit source 192.168.43.120 0.0.0.7   # 允许 192.168.43.120~127
 rule 15 deny                                    # 拒绝其他所有IP
#
acl advanced 3000
 description 隔离配置
 rule 5 permit ip source 192.168.43.120 0.0.0.7
 rule 10 permit ip source 192.168.48.120 0.0.0.7
 rule 15 deny ip source 192.168.43.0 0.0.0.255 destination 192.168.45.0 0.0.0.255
 rule 20 deny ip source 192.168.45.0 0.0.0.255 destination 192.168.43.0 0.0.0.255
 rule 25 deny ip source 192.168.48.0 0.0.0.255 destination 192.168.45.0 0.0.0.255
 rule 30 deny ip source 192.168.45.0 0.0.0.255 destination 192.168.48.0 0.0.0.255
 rule 100 permit ip                              # 放行其余所有流量
#
 netconf soap http enable
 ip http enable
 ip https enable
 smartmc tc enable
 cloud-management server domain oasis.h3c.com    # H3C绿洲云平台
#
return
```

### 3.2 配置逐段解释

#### 3.2.1 系统基础配置

|配置项|命令|作用说明|
| ----------| ------| -------------------------------------------------------------------------------------|
|设备名|`sysname Core-Switch`|设置设备主机名，便于识别和日志区分|
|IRF堆叠|`irf member 1 priority 1`|配置IRF（智能弹性架构）成员优先级为1。IRF可实现多台交换机虚拟化为1台，当前仅1台成员|
|DHCP服务|`dhcp enable`|全局开启DHCP服务，为VLAN 43/45/48提供IP自动分配|
|DNS|`dns server 223.5.5.5`|指定阿里DNS为设备自身的DNS解析服务器|
|LLDP|`lldp global enable`|开启链路层发现协议，可自动发现邻居设备（用于拓扑识别）|
|密码恢复|`password-recovery enable`|允许在忘记密码时通过Console口进行密码恢复|

#### 3.2.2 VLAN 规划

|VLAN|描述|关键操作|设计意图|
| ---------| --------------------| -------------------------------| --------------------------------------------|
|VLAN 1|`Default-VLAN-Disabled`|`interface Vlan-interface1` → `shutdown`|**安全加固**：禁用默认VLAN 1的三层接口，防止未授权访问|
|VLAN 43|路由器互联和AP网络|网关 192.168.43.254，提供DHCP|承载AP管理流量、与上行路由器互联|
|VLAN 45|监控网络|网关 192.168.45.254，提供DHCP|承载摄像头和NVR录像机流量|
|VLAN 48|业务网络|网关 192.168.48.254，提供DHCP|承载办公电脑等业务终端|

#### 3.2.3 DHCP 地址池

三个DHCP地址池设计一致：

- **地址范围**: xxx.xxx.xxx.5 ~ xxx.xxx.xxx.119（共115个可用地址）
- **保留地址段 .1~.4**: 预留给静态设备（服务器、打印机等）
- **保留地址段 .120~.254**: 预留给管理地址和特殊设备
- **DNS**: 阿里DNS 223.5.5.5 + 腾讯DNS 119.29.29.29（双DNS冗余）

> ⚠️ **注意**: DHCP地址池和静态管理IP的地址段不重叠，.120~.127 的管理地址未被纳入DHCP范围，避免冲突。

#### 3.2.4 端口分配

|端口|类型|VLAN|用途|
| -----------------| --------| --------------| ----------------------------------|
|1/0/1 ~ 1/0/20|Access|VLAN 48|业务端口（办公电脑等）|
|1/0/21|Trunk|All|**预留口**（可用于级联扩展）|
|1/0/22|Trunk|All|**下联POE交换机**（核心级联链路）|
|1/0/23|Access|VLAN 48|**预留口**|
|1/0/24|Trunk|All, PVID 43|**上联路由器**（PVID 43使无标签帧进入VLAN 43）|
|1/0/25 ~ 1/0/28|-|-|万兆光口（未使用）|

#### 3.2.5 ACL 安全策略

**ACL 2000（SSH访问控制）** ：应用在 `ssh server acl 2000`

```text
rule 5  permit 192.168.48.120~127   → 业务网管理主机可SSH
rule 10 permit 192.168.43.120~127   → AP网管理主机可SSH
rule 15 deny   all                   → 其他IP一律拒绝
```

> 只有指定的管理网段（.120~.127）才能SSH登录交换机，防止未授权访问。

**ACL 3000（跨VLAN隔离）** ：应用在所有VLAN三层接口的 `packet-filter 3000 inbound`

```text
rule 5   permit 管理段(43) → 任意        # 管理IP不受限制
rule 10  permit 管理段(48) → 任意        # 管理IP不受限制
rule 15  deny   VLAN43  → VLAN45        # AP网段不能访问监控网段
rule 20  deny   VLAN45  → VLAN43        # 监控网段不能访问AP网段
rule 25  deny   VLAN48  → VLAN45        # 业务网段不能访问监控网段
rule 30  deny   VLAN45  → VLAN48        # 监控网段不能访问业务网段
rule 100 permit 任意     → 任意         # 放行其余流量（含出Internet流量）
```

**隔离效果总结：**

|源 \ 目标|VLAN43 (AP)|VLAN45 (监控)|VLAN48 (业务)|Internet|
| ---------------| -------------| ---------------| ---------------| ----------|
|VLAN43 (AP)|✅ 通|❌ 隔离|✅ 通|✅ 通|
|VLAN45 (监控)|❌ 隔离|✅ 通|❌ 隔离|✅ 通|
|VLAN48 (业务)|✅ 通|❌ 隔离|✅ 通|✅ 通|
|管理主机|✅ 通|✅ 通|✅ 通|✅ 通|

> **设计意图**: 监控网络（VLAN 45）完全隔离，摄像头和NVR只能内网通信及访问外网，防止监控视频流被办公网络窃取。AP网络和业务网络之间可以互通（如无线办公场景）。

#### 3.2.6 安全加固

|配置|作用|
| ---------------------| -----------------------------------------|
|VLAN 1 关闭三层接口|消除默认VLAN的潜在攻击面|
|SSH 仅限 ACL 2000|只有管理网段可远程登录|
|`protocol inbound ssh`|VTY线路仅允许SSH协议，禁用Telnet|
|`password-control length 4`|密码最小长度限制（建议生产环境改为≥8）|
|`arp ip-conflict log prompt`|ARP IP冲突时记录日志并告警|

#### 3.2.7 路由

```text
ip route-static 0.0.0.0 0 192.168.43.253
```

- 默认路由指向路由器的 VLAN 43 接口地址 192.168.43.253
- 所有访问外网的流量经此路由转发到路由器

#### 3.2.8 NTP 时间同步

```text
ntp-service enable
ntp-service source Vlan-interface43
ntp-service unicast-server ntp.aliyun.com
ntp-service unicast-server ntp.tencent.com
```

- 双NTP服务器冗余（阿里云 + 腾讯云）
- NTP源接口指定为Vlan-interface43，确保NTP请求从该接口出去
- 时间同步对日志审计和安全认证（如证书验证）至关重要

#### 3.2.9 云管理

```text
cloud-management server domain oasis.h3c.com
```

- 设备注册到 H3C 绿洲云平台，可远程管理和监控

---

## 四、POE交换机配置详解

### 4.1 完整运行配置

```text
<POE-Switch>dis cu
#
 version 7.1.070, Release 3507P28
#
 sysname POE-Switch
#
 clock timezone Beijing add 08:00:00
#
 irf mac-address persistent timer
 irf auto-update enable
 undo irf link-delay
 irf member 1 priority 1
#
 save current-configuration binary-only interval 1    # 每1分钟自动保存配置
#
 dhcp enable
#
 dns server 223.5.5.5
 dns server 119.29.29.29
#
 lldp global enable
#
 password-recovery enable
#
vlan 1
 description 禁用默认VLAN1
#
vlan 43
 description AP和路由器网络
#
vlan 45
 description 监控网络
#
vlan 48
 description 业务网络
#
 stp global enable
#
interface NULL0
#
interface Vlan-interface1
 shutdown
 dhcp client identifier ascii 74adcbfc987c-VLAN0001
 ipv6 address dhcp-alloc
#
interface Vlan-interface48
 description 管理接口
 ip address 192.168.48.252 255.255.255.0
#
interface GigabitEthernet1/0/1
 description 录像机
 port access vlan 45
 poe enable
#
interface GigabitEthernet1/0/2  to 1/0/18              # 摄像头 ×17
 description 摄像头
 port access vlan 45
 poe enable
#
interface GigabitEthernet1/0/19                        # AP-01
 port link-type trunk
 port trunk permit vlan all
 port trunk pvid vlan 43
 poe enable
#
interface GigabitEthernet1/0/20                        # AP-02
 port link-type trunk
 port trunk permit vlan all
 port trunk pvid vlan 43
 poe enable
#
interface GigabitEthernet1/0/21  to 1/0/23            # AP预留口 (VLAN 43)
 port access vlan 43
 poe enable
#
interface GigabitEthernet1/0/24                        # ↑ 上联核心交换机
 description 上联核心交换机
 port link-type trunk
 port trunk permit vlan all
 poe enable                                             # ⚠️ 此处POE可能多余
#
interface GigabitEthernet1/0/25  to 1/0/28            # 未使用
#
 scheduler logfile size 16
#
line vty 0 63
 authentication-mode scheme
 user-role network-operator
 protocol inbound ssh
#
 ip route-static 0.0.0.0 0 192.168.48.254              # 默认路由指向核心交换机
#
 ssh server enable
 ssh server acl 2000
#
 ntp-service enable
 ntp-service unicast-server 192.168.43.254             # NTP从核心交换机同步
#
acl basic 2000
 description ssh限制
 rule 5 permit source 192.168.48.120 0.0.0.7
 rule 10 permit source 192.168.43.120 0.0.0.7
 rule 15 deny
#
 netconf soap http enable
 ip http enable
 ip https enable
 smartmc tc enable
 cloud-management server domain oasis.h3c.com
#
return
```

### 4.2 配置逐段解释

#### 4.2.1 系统基础配置

|配置项|命令|作用说明|
| ----------| -------------| --------------------------------------------|
|设备名|`sysname POE-Switch`|设备名称|
|时区|`clock timezone Beijing add 08:00:00`|设置北京时区（UTC+8），确保日志时间准确|
|自动保存|`save current-configuration binary-only interval 1`|**每1分钟自动保存一次二进制配置**，防止断电丢配置|
|DNS|双DNS服务器|阿里 223.5.5.5 + 腾讯 119.29.29.29（冗余）|
|LLDP|`lldp global enable`|开启邻居发现协议|

#### 4.2.2 VLAN 规划

> POE交换机上的VLAN定义与核心交换机**必须一致**，否则Trunk链路无法正确转发。

|VLAN|POE上描述|用途|
| ---------| ----------------| ---------------------------------------------|
|VLAN 1|禁用默认VLAN1|三层接口已shutdown，安全加固|
|VLAN 43|AP和路由器网络|AP管理SSID、无线客户端流量|
|VLAN 45|监控网络|摄像头、NVR录像机|
|VLAN 48|业务网络|POE交换机自身管理（仅配管理IP，不参与DHCP）|

> ⚠️ **关键注意**: POE交换机没有配置DHCP Server，所有VLAN的DHCP由核心交换机统一提供。Trunk链路传输DHCP广播，在核心交换机上完成地址分配。

#### 4.2.3 管理方式

```text
interface Vlan-interface48
 description 管理接口
 ip address 192.168.48.252 255.255.255.0
```

- POE交换机的管理IP为 **192.168.48.252**（位于业务VLAN 48）
- 由于VLAN 1已关闭，必须为交换机配置一个可路由的管理VLAN接口
- 默认路由 `0.0.0.0/0 → 192.168.48.254`（核心交换机VLAN 48网关）实现远程管理可达

#### 4.2.4 端口分配

|端口|类型|VLAN|POE|用途|
| -----------------| --------| --------------| -----| ---------------------------|
|1/0/1|Access|45|✅|录像机（NVR）|
|1/0/2 ~ 1/0/18|Access|45|✅|摄像头 ×17|
|1/0/19|Trunk|All, PVID 43|✅|AP-01（支持多SSID多VLAN）|
|1/0/20|Trunk|All, PVID 43|✅|AP-02（支持多SSID多VLAN）|
|1/0/21 ~ 1/0/23|Access|43|✅|AP预留口|
|1/0/24|Trunk|All|✅|**上联核心交换机**|

#### 4.2.5 POE 供电分析

POE交换机在此拓扑中的供电角色：

|设备类型|数量|端口|预估功率（每口）|总预估|
| -----------| ------| -----------| ------------------| ---------------|
|摄像头|17台|1/0/2~18|~7-15W|~120-255W|
|NVR录像机|1台|1/0/1|POE供电？|取决于NVR型号|
|AP|2台|1/0/19-20|~10-15W|~20-30W|
|**合计**|**20台POE设备**||| **~140-285W**|

> ⚠️ **需要注意**:
>
> 1. 确认POE交换机整机POE功率预算（通常 370W 或 740W），确保能满足所有设备供电
> 2. 端口 1/0/24（上联核心）也开启了POE，如果核心交换机不支持POE受电，建议执行 `undo poe enable` 关闭此口的POE功能
> 3. 端口 1/0/21~23 开启了POE但未使用，可按需关闭

#### 4.2.6 NTP 时间同步

```text
ntp-service enable
ntp-service unicast-server 192.168.43.254
```

- NTP服务器指向核心交换机的VLAN 43接口（192.168.43.254）
- 核心交换机从公网NTP同步，POE交换机从核心同步，形成**层级时间同步**架构
- 注意：NTP请求需要通过Trunk链路跨越VLAN 43，确保路由可达

#### 4.2.7 安全配置

POE交换机的安全配置与核心交换机保持一致：

- ACL 2000 限制SSH登录源IP
- SSH协议替代Telnet
- VLAN 1 三层接口关闭
- 云平台注册

---

## 五、配置文件错误与修复建议

### 5.1 核心交换机

|问题|严重程度|说明|修复建议|
| -----------------------------------| ----------| -------------------------------------------------------------------------------------------------------------------------| ------------------------|
|VLAN 48 无法访问 VLAN 43 的管理IP|⚠️ 中|ACL 3000 规则顺序：rule 10 放通 .48.120~.127，rule 25 阻断 .48.0/24→.45.0/24。并未阻断48和43的互通，但需要验证实际通信|`ping 192.168.43.254 source 192.168.48.x` 验证|
|`pex working-mode switch slot 1`|🔵 低|PEX模式配置，如果是独立交换机此配置无意义但无害|如非PEX环境可忽略|
|password-control length 4|🔴 高|密码最小长度仅4位，不符合安全基线（建议≥8位）|`password-control length 8`|
|`undo password-control complexity user-name check`|⚠️ 中|关闭了密码中不能包含用户名的检查|按安全策略决定是否开启|

### 5.2 POE交换机

|问题|严重程度|说明|修复建议|
| ---------------------------| ----------| ----------------------------------------------------------------------------------| ---------------|
|端口 1/0/24 `poe enable`|⚠️ 中|上联核心交换机的端口开启了POE供电，核心交换机一般不支持POE受电，存在损坏端口风险|**强烈建议执行** `undo poe enable`|
|端口 1/0/21~23 无描述|🔵 低|端口未配置 `description`，不利于后期维护|添加描述如 `description AP-预留`|
|password-control length 4|🔴 高|同核心交换机，密码长度不足|`password-control length 8`|

---

## 六、H3C 交换机常用命令速查

### 6.1 系统与基本信息

```bash
display version                          # 查看软件/硬件版本信息
display device                           # 查看设备板卡/模块状态
display device manuinfo                  # 查看设备序列号、制造信息
display cpu-usage                        # 查看CPU使用率
display memory                           # 查看内存使用情况
display fan                              # 查看风扇状态
display power                            # 查看电源状态
display temperature                      # 查看温度传感器信息
display environment                      # 查看环境（温度/风扇/电源）综合信息
display clock                            # 查看系统时间
display current-configuration            # 查看当前运行配置（等同于 dis cu）
display saved-configuration              # 查看已保存的启动配置
display startup                          # 查看下次启动加载的文件
display logbuffer                        # 查看日志缓冲区
display diagnostic-information           # 一键收集诊断信息（相当于 show tech-support）
reset logbuffer                          # 清除日志缓冲区
save force                               # 强制保存配置
reboot                                   # 重启设备
```

### 6.2 VLAN 相关

```bash
display vlan                             # 查看所有VLAN
display vlan brief                       # 查看VLAN简要信息
display vlan 43                          # 查看指定VLAN详细信息
display interface vlan-interface 43      # 查看VLAN三层接口状态
vlan 100                                 # 创建VLAN 100
undo vlan 100                            # 删除VLAN 100
port access vlan 48                      # 端口加入Access VLAN
port link-type trunk                     # 设置端口为Trunk模式
port trunk permit vlan 43 45 48          # Trunk允许指定VLAN通过
port trunk permit vlan all               # Trunk允许所有VLAN通过
port trunk pvid vlan 43                  # 设置Trunk端口的PVID（本征VLAN）
display port vlan                        # 查看端口VLAN信息
```

### 6.3 接口与端口

```bash
display interface brief                  # 查看所有接口状态（UP/DOWN）
display interface GigabitEthernet 1/0/1  # 查看指定接口详细信息
display interface description            # 查看所有接口描述
interface GigabitEthernet 1/0/1          # 进入接口视图
shutdown                                 # 关闭接口
undo shutdown                            # 开启接口
description 上联路由器                    # 设置接口描述
duplex full                              # 设置双工模式
speed 1000                               # 设置端口速率
display counters                         # 查看接口统计（收发字节/包数）
display counters rate                    # 查看接口实时速率
reset counters interface                 # 清除接口统计计数
display packet-drop                      # 查看接口丢包信息
```

### 6.4 MAC 地址表

```bash
display mac-address                      # 查看所有MAC地址表（所有VLAN）
display mac-address vlan 48              # 查看指定VLAN的MAC地址表
display mac-address count                # 查看MAC地址表总数
display mac-address aging-time           # 查看MAC地址老化时间
display mac-address mac-move             # 查看MAC漂移记录（排查环路）
mac-address timer aging 300              # 设置MAC老化时间（秒）
```

### 6.5 MAC 地址查询（重点）

```bash
# === 按MAC查端口 ===
display mac-address 74ad-cbfc-987c       # 查指定MAC在哪个端口上（H3C格式）
display mac-address 74ad-cbfc-987c vlan 48  # 限定VLAN查询

# === 按端口查MAC ===
display mac-address interface GigabitEthernet 1/0/22  # 查看某端口下所有MAC
display mac-address interface GigabitEthernet 1/0/22 count  # 只显示数量

# === 按VLAN查MAC ===
display mac-address vlan 45              # 查监控VLAN下所有设备MAC

# === MAC漂移检测（排查网络环路） ===
display mac-address mac-move             # 查看是否有MAC在端口间漂移

# === ARP表（IP ↔ MAC 映射） ===
display arp                              # 查看所有ARP表项
display arp vlan 48                      # 查指定VLAN的ARP
display arp | include 192.168.48.100     # 查特定IP的ARP记录（使用正则过滤）
display arp interface Vlan-interface48   # 查指定三层接口的ARP
```

### 6.6 IP 与路由

```bash
display ip interface brief               # 查看三层接口IP摘要
display ip routing-table                 # 查看路由表
display ip routing-table 192.168.48.0    # 查到达特定网段的路由
ip route-static 0.0.0.0 0 192.168.43.253 # 添加默认路由
undo ip route-static 0.0.0.0 0           # 删除默认路由
display ip routing-table statistics      # 路由表统计
```

### 6.7 DHCP

```bash
display dhcp server ip-pool              # 查看所有DHCP地址池
display dhcp server ip-pool vlan48       # 查看指定地址池详情
display dhcp server ip-in-use            # 查看已分配的IP地址
display dhcp server ip-in-use pool vlan48  # 查指定地址池已分配IP
display dhcp server expired              # 查看过期未释放的租约
display dhcp server statistics           # DHCP统计（请求/分配/拒绝次数）
display dhcp server conflict             # 查看IP地址冲突记录
reset dhcp server conflict               # 清除冲突记录
reset dhcp server ip-in-use              # 强制释放所有租约（慎用）
```

### 6.8 STP（生成树）

```bash
display stp                              # 查看STP全局状态
display stp brief                        # 查看STP端口角色和状态
display stp region-configuration         # 查看MSTP域配置
display stp abnormal-port                # 查看被STP阻塞的异常端口
display stp tc                           # 查看TC（拓扑变更）报文统计
display stp bpdu-protection              # 查看BPDU保护状态
stp global enable                        # 全局开启STP
stp mode mstp                            # 设置STP模式为MSTP
stp edged-port                           # 配置边缘端口（连接终端，快速进入转发）
```

### 6.9 LLDP（链路层发现）

```bash
display lldp neighbor-information list   # 查看LLDP邻居列表
display lldp neighbor-information interface GigabitEthernet 1/0/24  # 查特定端口邻居
display lldp neighbor-information verbose # 查看邻居详细信息（含设备名、管理IP）
display lldp local-information           # 查看本设备通告的LLDP信息
```

### 6.10 POE（以太网供电）

```bash
display poe device                       # 查看POE模块信息
display poe interface                    # 查看所有接口POE状态
display poe interface GigabitEthernet 1/0/1  # 查看指定端口POE详情
display poe interface power              # 查看各端口POE供电功率
display poe power-usage                  # 查看POE总功率使用情况
display poe temperature                  # 查看POE模块温度（部分型号支持）
poe enable                               # 接口下开启POE
undo poe enable                          # 接口下关闭POE
poe max-power 15000                      # 设置端口最大POE功率（毫瓦）
poe priority high                        # 设置POE供电优先级
```

### 6.11 ACL 与安全

```bash
display acl all                          # 查看所有ACL
display acl 3000                         # 查看ACL 3000详情
display packet-filter interface          # 查看接口上应用的包过滤策略
display packet-filter statistics         # 查看包过滤匹配统计
display ssh server-info                  # 查看SSH服务状态
display ssh server session               # 查看当前SSH会话
display ip https                         # 查看HTTPS服务状态
display password-control                 # 查看密码策略配置
display password-control blacklist       # 查看密码失败黑名单
reset password-control blacklist         # 清除密码黑名单
```

### 6.12 用户与会话

```bash
display users                            # 查看当前在线用户
display line                             # 查看线路状态
display local-user                       # 查看所有本地用户
display current-configuration | include local-user  # 从配置中过滤本地用户
free user-interface vty 1                # 强制断开VTY 1 线路上的用户
display telnet server                    # 查看Telnet服务状态
display ssh server status                # 查看SSH服务状态
```

### 6.13 日志与诊断

```bash
display logbuffer                        # 查看日志缓冲区
display logbuffer | include error        # 日志过滤（包含error的行）
display logbuffer | exclude informational # 排除info级别日志
display diagnostic-information           # 收集完整诊断信息（供H3C技术支持分析）
display device manuinfo                  # 查看设备序列号和制造信息（用于报修）
```

### 6.14 IRF（堆叠）

```bash
display irf                              # 查看IRF配置和成员状态
display irf topology                     # 查看IRF拓扑
display irf configuration                # 查看IRF端口配置
switchto irf member 2                    # 切换到成员2的控制台
```

### 6.15 配置管理

```bash
save                                    # 保存当前配置
save force                              # 强制保存（跳过确认提示）
save safely                             # 安全保存（先写临时文件再替换）
display saved-configuration             # 查看启动配置
display current-configuration           # 查看当前运行配置
display current-configuration | include vlan  # 用 | 过滤配置内容
display current-configuration | begin interface  # 从interface行开始显示
reset saved-configuration               # 清除启动配置（慎用）
startup saved-configuration flash:/startup.cfg  # 指定启动配置文件
```

### 6.16 |（管道符）过滤技巧

```bash
# H3C Comware 7 支持强大的管道符过滤
display current-configuration | include vlan          # 包含vlan的行
display current-configuration | exclude vlan          # 排除包含vlan的行
display current-configuration | begin interface       # 从匹配行开始显示
display current-configuration | section interface     # 按段落显示
display version | include "7.1"                       # 精确匹配
display mac-address | include GigabitEthernet1/0/22   # 过滤端口
display logbuffer | include "Down" | exclude "Up"     # 管道符组合
display arp | count                                   # 统计行数
```

---

## 七、华为交换机/路由器常用命令速查

> 华为和华三命令体系**高度相似**（源自同一厂商体系，后分立），90%以上的命令直接通用，以下仅列出差异和华为特有命令。

### 7.1 华为与H3C命令差异对照

|功能|H3C Comware|华为 VRP|说明|
| --------------| -------------| ---------------------| ----------------|
|查看运行配置|`dis cu`|`dis cu` 或 `display current-configuration`|完全相同|
|进入系统视图|`system-view`|`system-view`|完全相同|
|接口命名|`GigabitEthernet 1/0/1`|`GigabitEthernet 0/0/1`|槽位号起始不同|
|查看版本|`display version`|`display version`|完全相同|
|保存配置|`save`|`save`|完全相同|
|VLAN创建|`vlan 100`|`vlan 100` 或 VLAN视图 `vlan batch 100 200`|华为支持批量|
|Trunk配置|`port link-type trunk`|`port link-type trunk`|完全相同|
|MAC地址格式|`74ad-cbfc-987c`|`74ad-cbfc-987c`|完全相同|
|Web管理|`ip http enable`|`http server enable`|⚠️ **命令不同**|
|SSH服务|`ssh server enable`|`stelnet server enable`|⚠️ **命令不同**|
|查看ARP|`display arp`|`display arp`|完全相同|
|NTP配置|`ntp-service`|`ntp-service` 或 `ntp`|部分版本不同|

### 7.2 华为特有/差异命令

```bash
# === Web/HTTP 服务 ===
http server enable                       # 华为开启HTTP（H3C是 ip http enable）
http secure-server enable                # 华为开启HTTPS
display http server                      # 查看HTTP服务状态

# === SSH ===
stelnet server enable                    # 华为开启SSH（H3C是 ssh server enable）
display ssh server status                # 查看SSH状态

# === VLAN批量操作 ===
vlan batch 10 20 30                      # 批量创建VLAN
vlan batch 100 to 200                    # 批量创建100-200
display vlan summary                     # VLAN汇总

# === 接口统计 ===
display interface counters               # 华为接口统计命令相同
display interface description            # 查看接口描述（相同）

# === 配置回滚 ===
display configuration rollback           # 查看配置回滚点
rollback configuration to commit-id 1    # 回滚到指定提交点

# === 诊断 ===
display diagnostic-information           # 收集诊断信息（相同命令）
display device manufacture-info          # 查看制造信息（华为用 manufacture-info）

# === 补丁管理 ===
display patch-information                # 查看补丁信息（华为特有）
```

### 7.3 华为常用命令（通用类）

```bash
display device                           # 查看设备硬件信息
display cpu-usage                        # CPU使用率
display memory-usage                     # 内存使用率（华为是 memory-usage）
display temperature all                  # 查看所有温度传感器
display alarm all                        # 查看所有告警
display interface brief                  # 接口状态摘要
display ip interface brief               # 三层IP接口摘要
display ip routing-table                 # 路由表
display vlan                             # VLAN信息
display mac-address                      # MAC地址表
display arp all                          # ARP表
display stp brief                        # 生成树状态
display lldp neighbor brief              # LLDP邻居
display logbuffer                        # 日志缓冲区
display trapbuffer                       # 告警缓冲区（华为特有）
display current-configuration | section bgp  # 按段落过滤配置
```

---

## 八、查询端口 MAC 地址 — 实战场景命令大全

### 8.1 场景一：已知IP，查MAC和端口

```bash
# 方法1：先查ARP获取MAC，再查MAC表获取端口
display arp | include 192.168.48.100     # 第1步：从IP找到MAC
# 假设输出: 192.168.48.100  74ad-cbfc-987c  ...
display mac-address 74ad-cbfc-987c       # 第2步：从MAC找到端口

# 方法2：华为等效命令
display arp | include 192.168.48.100
display mac-address 74ad-cbfc-987c
```

### 8.2 场景二：已知端口，查下面接了哪些设备

```bash
# H3C/华为通用
display mac-address interface GigabitEthernet 1/0/22
# 再逐个查ARP看对应IP
display arp | include <MAC后半段>

# 一次性脚本思路（H3C Comware 7）
display mac-address interface GigabitEthernet 1/0/22
display arp | include <从上面输出的MAC>
```

### 8.3 场景三：排查MAC地址漂移（环路检测）

```bash
display mac-address mac-move             # 查看MAC漂移记录
# 如果输出非空，说明有MAC在多个端口间震荡 → 可能存在环路

# 详细信息
display mac-address mac-move verbose
```

### 8.4 场景四：统计某个VLAN下的设备数量

```bash
display mac-address vlan 48 count        # MAC地址数量 ≈ 设备数量
display arp vlan 48 | count              # ARP表项数量
```

### 8.5 场景五：定位ARP攻击源

```bash
# 1. 检查是否有IP冲突
display arp ip-conflict                  # H3C

# 2. 检查是否有异常数量的MAC对应同一IP
display arp | include 192.168.48.1       # 网关IP的ARP应只有1条

# 3. 检查ARP表是否被快速填满
display arp | count                      # 与正常时期对比数量

# 4. 开启ARP DAI或防攻击功能
arp anti-attack check user-bind enable   # H3C
```

---

## 九、运维建议与检查清单

### 9.1 日常巡检命令（建议保存为脚本）

```bash
# 登录后一键巡检
display device                           # 硬件状态
display environment                      # 环境（风扇/温度/电源）
display interface brief                  # 端口状态
display cpu-usage                        # CPU
display memory                           # 内存
display logbuffer | include error        # 近期错误日志
display stp brief                        # STP状态
display mac-address mac-move             # MAC漂移（环路排查）
display dhcp server ip-in-use            # DHCP使用情况
display poe power-usage                  # POE功率使用（POE交换机）
```

### 9.2 安全加固检查清单

- [ ] VLAN 1 三层接口是否已关闭
- [ ] 是否禁用 Telnet，仅使用 SSH
- [ ] SSH 是否配置了 ACL 源IP限制
- [ ] 密码最小长度是否 ≥ 8 位
- [ ] 是否配置了 `undo password-control complexity user-name check` 是否必要
- [ ] 是否开启 `ip https enable`（优于HTTP）
- [ ] 是否配置了 NTP 时间同步（日志审计需要）
- [ ] Console 口是否配置密码
- [ ] 未使用端口是否 shutdown 或划入隔离VLAN

### 9.3 备份建议

```bash
# 1. 定期备份配置到TFTP/FTP
tftp 192.168.48.120 put startup.cfg

# 2. 或在PC端通过SCP下载
# (在PC上执行) scp admin@192.168.48.252:/startup.cfg ./backup/

# 3. 导出配置到终端直接复制
display current-configuration
```

---

## 十、附录：同厂商命令差异速记表

|场景|H3C Comware 7|华为 VRP 8|Cisco IOS|
| -----------| ---------------| ------------| -----------|
|查看配置|`dis cu`|`dis cu`|`show run`|
|进入配置|`system-view`|`system-view`|`conf t`|
|保存|`save`|`save`|`wr` / `copy run start`|
|查看接口|`dis int brief`|`dis int brief`|`show ip int brief`|
|查看VLAN|`dis vlan`|`dis vlan`|`show vlan brief`|
|查看MAC表|`dis mac-address`|`dis mac-address`|`show mac address-table`|
|查看ARP|`dis arp`|`dis arp`|`show arp`|
|查看路由|`dis ip routing-table`|`dis ip routing-table`|`show ip route`|
|查看日志|`dis logbuffer`|`dis logbuffer`|`show log`|
|ping|`ping -c 4 1.1.1.1`|`ping -c 4 1.1.1.1`|`ping 1.1.1.1`|
|查看STP|`dis stp brief`|`dis stp brief`|`show spanning-tree`|
|查看LLDP|`dis lldp neighbor list`|`dis lldp neighbor brief`|`show lldp neighbors`|
|Web管理|`ip http enable`|`http server enable`|`ip http server`|
|开启SSH|`ssh server enable`|`stelnet server enable`|`ip ssh version 2`|

---

> **文档维护说明**: 本文档基于实际运行配置生成，后续如有拓扑变更或配置调整，请同步更新本文档。如有不明之处，请结合 `display current-configuration` 实际输出核对。
