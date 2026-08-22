---
title: Linux跨网段转发配置
image: "api"
published: 2026-08-22
description: 双网卡Linux服务器开启IP转发与masquerade伪装,让笔记本跨网段访问隔离网络设备。
category: 环境配置
tags:
  - 网络
  - 路由
  - NAT
slug: linux-cross-subnet-forwarding
---

# Linux 跨网段转发配置

> **适用系统**: Rocky Linux 9.x / RHEL 9.x / CentOS Stream 9
> **场景**: 双网卡服务器作为网桥，让笔记本通过服务器转发同时访问两个物理隔离的网段

---

## 一、场景描述

```text
┌─────────────────────────────────┐     ┌─────────────────────────────┐
│  192.168.48.0/24（办公网络）      │     │  10.0.0.0/24（隔离网络）       │
│                                 │     │                             │
│  ┌──────────┐                  │     │  ┌──────────┐               │
│  │  笔记本    │                  │     │  │ 设备 A    │               │
│  │ 48.x     │                  │     │  │ 10.0.0.x │               │
│  └────┬─────┘                  │     │  └──────────┘               │
│       │                        │     │                             │
│       │     ┌──────────────────┼─────┼──────────────────┐          │
│       │     │  服务器            │     │                  │          │
│       │     │                  │     │                  │          │
│       └─────► ens65f0          │     │  ens65f1 ◄────────┘          │
│             │ 192.168.48.95    │     │  10.0.0.200                 │
│             └──────────────────┼─────┼──────────────────┘          │
│                                 │     │                             │
│             网关: 192.168.48.254│     │  网关: 10.0.0.254           │
└─────────────────────────────────┘     └─────────────────────────────┘

两个网络物理隔离，服务器是唯一同时连接两边的节点。
目标：笔记本只需接在 192.168.48.0/24 中，就能访问 10.0.0.0/24 的所有设备。
```

## 二、核心原理

实现跨网段访问需要同时解决三个问题：

|层面|问题|解决方法|对应配置|
| ------| ------------------------------------------------------------------------------------------------------------| --------------------------------------------------| --------------------------------------|
|**① IP 转发**|服务器默认不转发非本机的数据包|开启内核 IP 转发|`net.ipv4.ip_forward = 1`|
|**② 反向路径过滤**|数据包从非最佳路径口进入时被内核丢弃（严格模式下）|关闭或设为松散模式|`rp_filter = 0`|
|**③ 回程路由（关键）**|10.0.0.x 收到源 IP 为 192.168.48.x 的包后，不知道怎么回包——因为 10.0.0.0/24 的网关到不了 192.168.48.0/24|SNAT/Masquerade：将源 IP 改为服务器的 10.0.0.200|firewalld masquerade 或 iptables NAT|
|**④ 客户端路由**|笔记本默认网关会把 10.0.0.0/24 的流量发给默认网关（路由器），而不是服务器|笔记本上添加静态路由|`route add 10.0.0.0/24 → 192.168.48.95`|

### rp_filter 详解

> **原文已提及但不够详细，此处补充：**

Linux 内核的 `rp_filter`（Reverse Path Filter）用于防止 IP 欺骗攻击。取值有三种：

|值|模式|行为|
| ----| ---------------------| ------------------------------------------------------------|
|0|关闭|不检查源地址合法性|
|1|严格模式 (RFC 3704)|数据包进入接口必须和去往源 IP 的最佳出接口相同，否则丢弃|
|2|松散模式|只要源 IP 在当前路由表中有可达路由即可，不要求是同一个接口|

**本例为什么必须关闭/松散 rp_filter：**

- 笔记本 (192.168.48.x) → 访问 10.0.0.50
- 数据包从 ens65f0 进入服务器（源 IP = 192.168.48.x，目标 IP = 10.0.0.50）
- 严格模式下，内核查路由表：去往 192.168.48.x 的最佳接口确实是 ens65f0 → 通过
- **但 MASQUERADE 之后**，在 POSTROUTING 链上源地址被改成了 10.0.0.200
- 另外，从 10.0.0.50 回包时：数据包从 ens65f1 进入，源 IP = 10.0.0.50，目标 IP = 10.0.0.200（经 NAT 反向转换成 192.168.48.x）
- 严格模式下去往 10.0.0.50 的最佳接口是 ens65f1 → 通过
- **实际上本例中 rp_filter 不一定会丢包**，但关闭它是最保险的做法，尤其在使用 SNAT 的双网卡场景中

> **结论**: 双网卡转发场景推荐设置为 `net.ipv4.conf.all.rp_filter = 2`（松散模式）或 `0`（关闭），设为 2 比 0 更安全，既允许非对称路由又保留基本的源地址校验。

---

## 三、方案选择

|方案|适用场景|优点|缺点|
| ----------| ----------------------------------| -----------------------------| ----------------------------------|
|**A. firewalld masquerade**（推荐）|一般场景，已有 firewalld 运行|配置简单，规则持久化|依赖 firewalld 服务|
|**B. iptables/nftables 手动**|需要精确控制规则，或无 firewalld|无需额外服务，规则精确|持久化需额外处理|
|**C. 纯路由 + 对端回程路由**|能控制 10.0.0.0/24 网关时|无 NAT 损耗，连接跟踪开销小|需修改对端网关路由表，通常不可行|

> 本文档以方案 A 为主进行展开，方案 B 在附录中提供参考。

---

## 四、服务器端配置（Rocky Linux 9.7）

### 4.1 步骤一：开启 IP 转发

```bash
# ===== 临时生效（立即测试用） =====
sysctl -w net.ipv4.ip_forward=1

# ===== 永久生效 =====
#  修正：使用 /etc/sysctl.d/ 而非直接追加 /etc/sysctl.conf
#          Rocky Linux 9 推荐 /etc/sysctl.d/ 中的独立配置文件
cat > /etc/sysctl.d/99-ipforward.conf << 'EOF'
# 开启 IPv4 包转发
net.ipv4.ip_forward = 1
EOF

sysctl -p /etc/sysctl.d/99-ipforward.conf

# 验证
sysctl net.ipv4.ip_forward
# 期望输出: net.ipv4.ip_forward = 1
```

### 4.2 步骤二：配置 rp_filter

```bash
# ===== 临时设置 =====
sysctl -w net.ipv4.conf.all.rp_filter=2        # 松散模式（推荐，比 0 更安全）
sysctl -w net.ipv4.conf.ens65f0.rp_filter=2
sysctl -w net.ipv4.conf.ens65f1.rp_filter=2

# ===== 永久生效 =====
cat > /etc/sysctl.d/99-rpfilter.conf << 'EOF'
# 双网卡转发场景需关闭严格反向路径过滤
# 2 = 松散模式：源 IP 路由可达即接受（推荐）
# 0 = 关闭：完全不检查
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.ens65f0.rp_filter = 2
net.ipv4.conf.ens65f1.rp_filter = 2
EOF

sysctl -p /etc/sysctl.d/99-rpfilter.conf

# 验证
sysctl -a | grep "\.rp_filter"
```

### 4.3 步骤三（新增）：检查 SELinux 状态

> **Rocky Linux 9 默认 SELinux 为** **`enforcing`** **模式，可能阻止转发。**

```bash
# 查看当前 SELinux 模式
getenforce

# 查看是否有被拒绝的转发操作（配置完成后如果不行，查日志）
ausearch -m avc -ts recent | grep forward
journalctl -xe | grep -i "SELinux.*denied"
```

如果 SELinux 确实阻止了转发，有两种处理方式：

```bash
# 方式一：临时关闭 SELinux（仅用于诊断，不推荐长期方案）
setenforce 0

# 方式二（推荐）：开启 SELinux 允许 IP 转发的布尔值
setsebool -P nis_enabled 1
# 注意：nis_enabled=1 实际上允许了多种非标转发行为
# 更精确的做法是创建自定义 SELinux 策略模块
```

> 如果 `getenforce` 返回 `Permissive` 或 `Disabled`，则无需处理。

### 4.4 步骤四：配置防火墙 — Masquerade（SNAT）

#### 方案 A（推荐）：trusted zone + masquerade

```bash
# 将两个网口放入 trusted zone（该 zone 默认允许所有流量）
firewall-cmd --permanent --zone=trusted --change-interface=ens65f0
firewall-cmd --permanent --zone=trusted --change-interface=ens65f1

# 开启 masquerade（SNAT —— 源地址转换）
firewall-cmd --permanent --add-masquerade

# 重载生效
firewall-cmd --reload

# 验证配置
firewall-cmd --list-all --zone=trusted
echo "---"
firewall-cmd --query-masquerade
# 期望输出: yes
```

#### 方案 B：精细化控制（保持原 zone 不变）

```bash
# 开启 masquerade
firewall-cmd --permanent --add-masquerade

# 添加 FORWARD 规则：允许两个方向的数据流转发
# ens65f0 → ens65f1（笔记本访问隔离网络）
firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 \
  -i ens65f0 -o ens65f1 \
  -s 192.168.48.0/24 -d 10.0.0.0/24 \
  -j ACCEPT

# ens65f1 → ens65f0（隔离网络回包）
firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 \
  -i ens65f1 -o ens65f0 \
  -s 10.0.0.0/24 -d 192.168.48.0/24 \
  -j ACCEPT

firewall-cmd --reload

# 验证
firewall-cmd --direct --get-all-rules
```

> **注意**：方案 B 的 FORWARD 规则仅用于精细控制场景。如果 `masquerade` 已开启且 zone 允许了转发，则方案 A 已足够。

### 4.5 步骤五（新增）：确认网口配置正确

```bash
# 查看网口IP配置
ip addr show ens65f0
ip addr show ens65f1

# 查看路由表
ip route show

# 确认两个网口都在正确的子网中
ip route | grep "192.168.48"
ip route | grep "10.0.0"
```

### 4.6 服务器端配置总验证

```bash
#!/bin/bash
# 一键诊断脚本（在服务器上执行）

echo "===== 1. IP 转发状态 ====="
sysctl net.ipv4.ip_forward

echo ""
echo "===== 2. rp_filter 状态 ====="
sysctl net.ipv4.conf.all.rp_filter
sysctl net.ipv4.conf.ens65f0.rp_filter
sysctl net.ipv4.conf.ens65f1.rp_filter

echo ""
echo "===== 3. firewall-cmd masquerade ====="
firewall-cmd --query-masquerade 2>/dev/null || echo "firewalld 未运行"

echo ""
echo "===== 4. NAT 规则（nftables） ====="
nft list ruleset 2>/dev/null | grep -A5 masquerade || echo "无 masquerade 规则"

echo ""
echo "===== 5. FORWARD 规则 ====="
nft list chain inet firewalld filter_FWDO_forward 2>/dev/null || \
nft list chain ip filter FORWARD 2>/dev/null || \
echo "未找到 firewalld forward 链"

echo ""
echo "===== 6. 路由表 ====="
ip route show

echo ""
echo "===== 7. SELinux 状态 ====="
getenforce
```

---

## 五、笔记本端配置（Windows 11）

### 5.1 添加静态路由

```powershell
# 必须以管理员身份运行 CMD 或 PowerShell

# 添加路由：访问 10.0.0.0/24 时不走默认网关，而是发给服务器 192.168.48.95
route -p add 10.0.0.0 mask 255.255.255.0 192.168.48.95

# 参数说明：
#   add 10.0.0.0          — 目标网络
#   mask 255.255.255.0    — 子网掩码（/24）
#   192.168.48.95          — 下一跳（服务器的 ens65f0 地址）
#   -p                    — Persistent，持久化，重启后保留
```

### 5.2 （新增）Metric 路由优先级

如果笔记本有多个网卡（如有线 + 无线），可能出现路由冲突。设置跃点数（Metric）来控制优先级：

```powershell
# 查看当前路由表
route print

# 如果你希望 10.0.0.0/24 的路由有更高优先级
route -p add 10.0.0.0 mask 255.255.255.0 192.168.48.95 metric 10
# metric 值越小，优先级越高

# 验证
route print | findstr "10.0.0.0"
```

### 5.3 （新增）删除/修改路由

```powershell
# 查看当前持久化路由
route print -4 | findstr "10.0.0"

# 删除持久化路由（不再需要时）
route delete 10.0.0.0

# 先删后改
route delete 10.0.0.0
route -p add 10.0.0.0 mask 255.255.255.0 192.168.48.96  # 改成新网关
```

---

## 六、测试验证

### 6.1 基础连通性测试

```powershell
# === 在笔记本（Windows 11）上执行 ===

# 1. 测试路由是否正确配置
tracert -d 10.0.0.200
# 期望：第一跳应该是 192.168.48.95

# 2. 测试服务器两个网口均可达
ping 192.168.48.95
ping 10.0.0.200

# 3. 测试 10.0.0.0/24 中的目标设备
ping 10.0.0.50

# 4. 确认 192.168.48.0/24 的原有通信不受影响
ping 192.168.48.254        # 网关
ping 192.168.48.1          # 路由器或其他主机

# 5. 外网通信不受影响
ping 223.5.5.5
```

### 6.2 （新增）TCP 端口测试

```powershell
# ICMP 通不代表 TCP 通，用 PowerShell 的 Test-NetConnection 测试具体服务
Test-NetConnection -ComputerName 10.0.0.50 -Port 22
Test-NetConnection -ComputerName 10.0.0.50 -Port 80
Test-NetConnection -ComputerName 10.0.0.50 -Port 3389
```

### 6.3 （新增）服务器端抓包诊断

```bash
# === 在服务器（Rocky Linux）上执行 ===

# 终端1：抓取 ens65f0 入口流量（来自笔记本的请求）
tcpdump -i ens65f0 -nn icmp and host 192.168.48.100
# 将 192.168.48.100 替换为笔记本的实际 IP

# 终端2：抓取 ens65f1 出口流量（经 SNAT 后的请求，源地址应为 10.0.0.200）
tcpdump -i ens65f1 -nn icmp

# 终端3：同时抓两个口，看数据包是否转发成功
tcpdump -i any -nn icmp and host 10.0.0.50

# 高级：检查 conntrack 连接跟踪表
conntrack -L | grep "10.0.0.50"
```

---

## 七、（新增）故障排查

### 问题 1：ping 超时，tracert 显示第一跳不到 192.168.48.95

**原因**: 笔记本路由未生效或配置错误

```powershell
# 检查路由表
route print | findstr "10.0.0"
# 应出现：10.0.0.0  255.255.255.0  192.168.48.95  ...

# 如果路由表中没有，重新添加
route -p add 10.0.0.0 mask 255.255.255.0 192.168.48.95
```

### 问题 2：tracert 第一跳正确，但服务器无响应

**原因**:

1. IP 转发未开启
2. firewalld 规则未生效
3. SELinux 阻止

```bash
# 在服务器上逐步排查：
# 1) 确认 IP 转发
sysctl net.ipv4.ip_forward           # 应为 1

# 2) 确认转发功能实际生效
cat /proc/sys/net/ipv4/ip_forward    # 应输出 1

# 3) 确认 fw 规则
firewall-cmd --query-masquerade      # 应为 yes
firewall-cmd --list-all

# 4) 确认 NAT 规则实际存在
nft list ruleset | grep masquerade

# 5) 检查 SELinux 阻断
ausearch -m avc -ts recent
```

### 问题 3：笔记本能 ping 通 10.0.0.200，但 ping 不通 10.0.0.50

**原因**: 目标设备 10.0.0.50 的防火墙/网关配置问题

**分析**: 数据包经过 SNAT 后，源地址已变为 10.0.0.200。10.0.0.50 应该能把回包发给 10.0.0.200（同一子网）。

```bash
# 在服务器上 tcpdump 确认数据包是否发出并收到回包
tcpdump -i ens65f1 -nn icmp and host 10.0.0.50

# 如果看到 "request" 但没有 "reply"：
# → 10.0.0.50 收到包了但没有回包（检查目标设备防火墙）
# 如果完全没有包发出：
# → 服务器路由问题（检查 ip route get 10.0.0.50）
```

### 问题 4：通信时断时续

**原因**: 可能是有多个网卡、路由冲突、或 conntrack 表满了

```bash
# 检查 conntrack 表使用情况
conntrack -C                        # 当前连接数
cat /proc/sys/net/netfilter/nf_conntrack_max  # 最大连接数

# 如果连接数接近上限，调大限制
echo 262144 > /proc/sys/net/netfilter/nf_conntrack_max

# 检查是否有连接跟踪冲突
conntrack -S
```

### 问题 5：Windows 路由重启后丢失

**原因**: 路由添加时未使用 `-p` 参数

```powershell
# 重新添加（注意 -p）
route delete 10.0.0.0
route -p add 10.0.0.0 mask 255.255.255.0 192.168.48.95
```

### 问题 6：服务器重启后转发失败

**原因**: 某些配置未持久化

```bash
# 逐一核对：
# ① IP 转发是否持久化
cat /etc/sysctl.d/99-ipforward.conf
# 应有: net.ipv4.ip_forward = 1

# ② rp_filter 是否持久化
cat /etc/sysctl.d/99-rpfilter.conf

# ③ firewalld 规则是否 persistent
firewall-cmd --list-all              # 当前运行
firewall-cmd --list-all --permanent  # 永久配置
# 二者应一致，若不一致执行：
firewall-cmd --runtime-to-permanent
```

---

## 八、（新增）数据流向详解

```text
┌─────────────────────────────────────────────────────────────────────┐
│  正向数据流（笔记本 → 10.0.0.50）                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  笔记本 (192.168.48.100)                                              │
│     │                                                                │
│     │  [原始包]                                                       │
│     │  SRC: 192.168.48.100  DST: 10.0.0.50                           │
│     │  查路由表 → 匹配 10.0.0.0/24 → 下一跳 192.168.48.95            │
│     ▼                                                                │
│  ens65f0 (192.168.48.95) ─── 服务器收到包                              │
│     │                                                                │
│     │  [PREROUTING] — 路由决策                                        │
│     │  内核发现目标 10.0.0.50 不在本机 → 需要转发                       │
│     │  ip_forward=1 → 允许转发                                        │
│     │  rp_filter=2 → 松散检查通过                                     │
│     │                                                                │
│     │  [FORWARD] — 经过 FORWARD 链                                    │
│     │  firewalld filter_FORWARD → ACCEPT                             │
│     │                                                                │
│     │  [POSTROUTING] — SNAT/Masquerade                               │
│     │  SRC: 192.168.48.100 → 10.0.0.200   ◄── MASQUERADE            │
│     │  conntrack 记录: orig=(48.100→10.0.50), reply=(10.0.50→10.0.200)│
│     ▼                                                                │
│  ens65f1 (10.0.0.200)                                                │
│     │                                                                │
│     │  [SNAT后的包]                                                    │
│     │  SRC: 10.0.0.200  DST: 10.0.0.50                               │
│     ▼                                                                │
│  10.0.0.50 收到，SRC 是 10.0.0.200 → 直接回包给 10.0.0.200             │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  反向数据流（10.0.0.50 → 笔记本）                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  10.0.0.50                                                           │
│     │                                                                │
│     │  [回包]                                                         │
│     │  SRC: 10.0.0.50  DST: 10.0.0.200                               │
│     ▼                                                                │
│  ens65f1 ─── 服务器收到回包                                            │
│     │                                                                │
│     │  [PREROUTING] — conntrack 命中                                  │
│     │  DST: 10.0.0.200 → 192.168.48.100  ◄── NAT 反向转换            │
│     │                                                                │
│     │  [FORWARD]                                                      │
│     │                                                                │
│     ▼                                                                │
│  ens65f0                                                              │
│     │                                                                │
│     │  [恢复后的包]                                                    │
│     │  SRC: 10.0.0.50  DST: 192.168.48.100                            │
│     ▼                                                                │
│  笔记本收到回包 ✓                                                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 九、（新增）安全加固建议

### 9.1 限制 Masquerade 范围

默认 `firewall-cmd --add-masquerade` 会对所有出口流量做 SNAT。如果只想针对特定子网做 masquerade，使用 rich rule 或 nftables 手动控制：

```bash
# 方式一：firewalld rich rule（仅对特定源/目的做 masquerade）
firewall-cmd --permanent --zone=trusted --add-rich-rule='rule family="ipv4" \
  source address="192.168.48.0/24" destination address="10.0.0.0/24" masquerade'
firewall-cmd --reload

# 方式二：使用 nftables 精确控制（见附录）
```

### 9.2 限制转发的源和目标

```bash
# 如果 default zone 的 target 不是 ACCEPT，添加精确的 FORWARD 规则
firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 \
  -i ens65f0 -o ens65f1 \
  -s 192.168.48.0/24 -d 10.0.0.0/24 \
  -j ACCEPT

firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 \
  -i ens65f1 -o ens65f0 \
  -s 10.0.0.0/24 -d 192.168.48.0/24 \
  -m state --state ESTABLISHED,RELATED \
  -j ACCEPT  # 仅允许已建立的连接回包

firewall-cmd --reload
```

### 9.3 安全基线检查

```bash
# 检查防火墙默认策略
firewall-cmd --get-default-zone
firewall-cmd --list-all

# 确保未被授权的外网口不在 trusted 等宽松 zone 中
firewall-cmd --get-active-zones

# 确保 SSH 等公共服务受限制
firewall-cmd --list-services --zone=public
```

---

## 十、（新增）清理与回滚

### 10.1 移除笔记本路由

```powershell
# Windows 上
route delete 10.0.0.0
# -p 持久化路由需单独删除，delete 会同时删除活跃和持久化条目
```

### 10.2 回滚服务器配置

```bash
# 1. 关闭 masquerade
firewall-cmd --permanent --remove-masquerade
firewall-cmd --reload

# 2. 恢复网卡到原始 zone（假设原来是 public）
firewall-cmd --permanent --zone=public --change-interface=ens65f0
firewall-cmd --permanent --zone=public --change-interface=ens65f1
firewall-cmd --reload

# 3. 关闭 IP 转发（如果不再需要）
sysctl -w net.ipv4.ip_forward=0
rm -f /etc/sysctl.d/99-ipforward.conf

# 4. 恢复 rp_filter
sysctl -w net.ipv4.conf.all.rp_filter=1
rm -f /etc/sysctl.d/99-rpfilter.conf

# 5. 恢复 SELinux（如果做过修改）
setsebool -P nis_enabled 0

# 6. 应用默认 sysctl
sysctl --system
```

---

## 十一、附录

### 附录 A：iptables/nftables 手动方案（不依赖 firewalld）

#### A.1 使用 iptables-nft（Rocky Linux 9 默认使用 nftables 作为后端）

```bash
# 安装 iptables 工具（如未安装）
dnf install -y iptables-nft

# 开启转发
sysctl -w net.ipv4.ip_forward=1

# 添加 NAT 规则
iptables -t nat -A POSTROUTING -s 192.168.48.0/24 -d 10.0.0.0/24 -o ens65f1 -j MASQUERADE

# 允许转发
iptables -A FORWARD -i ens65f0 -o ens65f1 -s 192.168.48.0/24 -d 10.0.0.0/24 -j ACCEPT
iptables -A FORWARD -i ens65f1 -o ens65f0 -s 10.0.0.0/24 -d 192.168.48.0/24 -m state --state ESTABLISHED,RELATED -j ACCEPT

# 持久化保存
iptables-save > /etc/sysconfig/iptables

# 确保 iptables 服务开机启动
systemctl enable --now iptables
```

#### A.2 使用原生 nftables

```bash
# 查看当前 nftables 规则（注意 firewalld 也使用 nftables）
nft list ruleset

# 手动添加规则（不建议与 firewalld 混用）
nft add table ip nat
nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }
nft add rule ip nat postrouting ip saddr 192.168.48.0/24 ip daddr 10.0.0.0/24 oif ens65f1 masquerade

# 持久化
nft list ruleset > /etc/sysconfig/nftables.conf
systemctl enable --now nftables  # ⚠️ 会与 firewalld 冲突，二选一
```

### 附录 B：纯路由方案（不使用 NAT）

> 只有在**能控制 10.0.0.0/24 网关路由表**的场景才可行。

1. 服务器开启 `ip_forward=1`
2. 笔记本添加路由 `route -p add 10.0.0.0 mask 255.255.255.0 192.168.48.95`
3. **在 10.0.0.254（对端网关）上**添加回程路由：`ip route 192.168.48.0/24 via 10.0.0.200`

优点：无 NAT 性能损耗，连接跟踪零开销。
缺点：需要修改对端网关路由，实际场景中**通常做不到**。

### 附录 C：NetworkManager 关联网络配置

如果网卡由 NetworkManager 管理（Rocky Linux 9 默认），建议确认配置：

```bash
# 查看连接信息
nmcli connection show

# 查看网卡所属连接
nmcli device status

# 如果两个网卡配置了同一个 zone，确保路由正确
nmcli connection show <connection-name> | grep ipv4.routes
```

### 附录 D：关键文件路径速查

|文件|作用|
| ------| -----------------------------|
|`/etc/sysctl.d/99-ipforward.conf`|IP 转发持久化配置|
|`/etc/sysctl.d/99-rpfilter.conf`|rp_filter 持久化配置|
|`/etc/firewalld/zones/trusted.xml`|firewalld trusted zone 定义|
|`/etc/sysconfig/iptables`|iptables 持久化规则|
|`/etc/sysconfig/nftables.conf`|nftables 持久化规则|
|`/proc/sys/net/ipv4/ip_forward`|内核 IP 转发开关（实时）|
|`/proc/sys/net/ipv4/conf/*/rp_filter`|内核 rp_filter 开关（实时）|

### 附录 E：H3C/华为交换机配置相关命令

如果跨网段转发涉及交换机配置（例如在交换机上划分 VLAN 或将服务器端口设为 Trunk），参考以下命令：

```bash
# H3C Comware 7
interface GigabitEthernet 1/0/22
  port link-type trunk
  port trunk permit vlan all

# 华为 VRP
interface GigabitEthernet 0/0/22
  port link-type trunk
  port trunk allow-pass vlan all
```

### 附录 F：Rocky Linux 9 特定注意事项

|项目|Rocky Linux 9 行为|需要注意|
| -----------------| ------------------------------------------| --------------------------------------------|
|防火墙后端|`firewalld` 使用 `nftables` 作为后端（不再是 iptables）|不要混用 `iptables` 和 `firewall-cmd` 命令|
|SELinux|默认 `enforcing`|可能阻止转发，需使用 `setsebool` 或 `audit2allow`|
|网络管理|默认 `NetworkManager`|手动修改 `/etc/sysconfig/network-scripts/` 可能被覆盖|
|sysctl 加载顺序|`sysctl --system` 按以下顺序：`/etc/sysctl.d/*.conf` → `/etc/sysctl.conf`|数字前缀越小越先加载，后加载的覆盖先加载的|
|`iptables` 命令|实际的 `iptables` 是 `iptables-nft`（基于 nftables）|使用 `iptables -V` 确认版本|

---

> **文档维护说明**: 本文档基于实际生产场景整理，请根据实际网络环境调整 IP 地址和接口名。如有拓扑变更或新增需求，请同步更新本文档。
