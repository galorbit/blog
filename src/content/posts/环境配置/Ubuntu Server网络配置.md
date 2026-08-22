---
title: Ubuntu Server网络配置
published: 2026-08-22
description: 解决Ubuntu Server开机网络等待卡顿与静态IP被重置,并给出netplan多场景与bond配置模板。
category: 环境配置
tags:
  - Ubuntu
  - Netplan
  - 网络配置
slug: ubuntu-server-network-config
---

# Ubuntu Server网络配置

---

## 一、开机卡顿问题：A start job is running for wait for network to be Configured

### 问题原因

systemd-networkd-wait-online.service 默认等待所有网络接口上线，超时时间长达120秒，若接口未连接或配置错误会导致开机卡顿。

### 解决方案（三选一）

#### 方案1：缩短等待超时时间（推荐）

```bash
# 创建systemd覆盖配置（比直接修改原文件更安全）
sudo mkdir -p /etc/systemd/system/systemd-networkd-wait-online.service.d/

sudo tee /etc/systemd/system/systemd-networkd-wait-online.service.d/timeout.conf << 'EOF'
[Service]
ExecStart=
ExecStart=/lib/systemd/systemd-networkd-wait-online --timeout=30
EOF

# 重载配置
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
```

#### 方案2：标记网卡为可选（Netplan配置）

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      # 关键配置：标记为可选，不阻塞启动
      optional: true
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 114.114.114.114
```

#### 方案3：完全禁用等待服务（不推荐生产环境）

```bash
sudo systemctl mask systemd-networkd-wait-online.service
```

---

## 二、静态IP重启丢失问题（cloud-init冲突）

### 问题原因

cloud-init 在每次启动时会重新应用网络配置，覆盖手动修改的Netplan配置，常见于：

- 虚拟机（VMware/VirtualBox/Proxmox）
- 云服务器（AWS/阿里云/腾讯云）
- 使用cloud-image安装的系统

### 解决方案

#### 步骤1：禁用cloud-init的网络管理

```bash
# 创建cloud-init网络禁用配置
sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg << 'EOF'
network: {config: disabled}
EOF
```

#### 步骤2：清理cloud-init网络缓存

```bash
# 停止cloud-init相关服务
sudo systemctl stop cloud-init-local.service cloud-init.service cloud-config.service cloud-final.service

# 清理网络相关缓存文件
sudo rm -f /var/lib/cloud/instances/*/network-config

# 删除cloud-init生成的netplan配置文件
sudo rm -f /etc/netplan/50-cloud-init.yaml

# 禁用cloud-init网络服务（可选）
sudo systemctl disable cloud-init-local.service cloud-init.service
```

#### 步骤3：确保Netplan配置优先级最高

```bash
# 查看配置文件加载顺序（数字越小优先级越高）
ls -la /etc/netplan/

# 重命名你的配置文件为 01- 开头，确保优先加载
sudo mv /etc/netplan/00-installer-config.yaml /etc/netplan/01-netcfg.yaml
```

---

## 三、完整配置文件模板

### 场景1：单网卡静态IP

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      # 标记为可选，避免开机等待超时
      optional: true
      # 禁用DHCP
      dhcp4: false
      dhcp6: false
      # 配置静态IP地址（CIDR格式）
      addresses:
        - 192.168.1.100/24
      # 配置默认网关
      routes:
        - to: default
          via: 192.168.1.1
          metric: 100
      # 配置DNS服务器
      nameservers:
        addresses:
          - 8.8.8.8
          - 114.114.114.114
        search:
          - local
```

### 场景2：多网卡多IP（内网+外网）

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    # 内网网卡（业务网络）
    enp0s3:
      optional: true
      dhcp4: false
      addresses:
        - 10.0.0.10/24
      nameservers:
        addresses:
          - 10.0.0.1
          - 8.8.8.8
    # 外网网卡（默认网关只在此配置）
    enp0s8:
      optional: true
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
          metric: 100
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

### 场景3：单网卡多IP

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      optional: true
      dhcp4: false
      addresses:
        # 主业务IP
        - 192.168.1.100/24
        # 备用IP
        - 192.168.1.101/24
        # 管理IP（/32点对点）
        - 192.168.1.102/32
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
```

---

## 四、端口聚合（Bonding）配置

### Bond模式速查表

|模式|名称|说明|适用场景|
| ---------------| ------| --------------------| -----------------------------|
|balance-rr|0|轮询负载均衡|高吞吐，需交换机支持|
|active-backup|1|主备模式|最常用，高可用|
|balance-xor|2|XOR负载均衡|特定流量分发|
|broadcast|3|广播模式|特殊容错场景|
|802.3ad|4|LACP动态聚合|高性能+高可用，需交换机配置|
|balance-tlb|5|自适应发送负载均衡|无需交换机支持|
|balance-alb|6|自适应负载均衡|无需交换机支持|

---

### 方案A：主备模式（mode=1，无需交换机配置）

```yaml
# /etc/netplan/02-bond-active-backup.yaml
network:
  version: 2
  renderer: networkd
  bonds:
    bond0:
      # 绑定物理网卡
      interfaces:
        - enp0s3
        - enp0s8
      parameters:
        # 主备模式
        mode: active-backup
        # 指定主网卡（可选）
        primary: enp0s3
        # 链路检测间隔（毫秒）
        mii-monitor-interval: 100
        # 链路恢复等待时间（毫秒）
        up-delay: 2000
        # 链路故障判定时间（毫秒）
        down-delay: 2000
      # Bond接口IP配置
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 114.114.114.114
  # 物理网卡只需声明，不要配置IP
  ethernets:
    enp0s3:
      optional: true
    enp0s8:
      optional: true
```

---

### 方案B：LACP动态聚合（mode=4，需交换机配置）

```yaml
# /etc/netplan/02-bond-lacp.yaml
network:
  version: 2
  renderer: networkd
  bonds:
    bond0:
      interfaces:
        - enp0s3
        - enp0s8
      parameters:
        # LACP动态聚合模式
        mode: 802.3ad
        # LACP报文频率：fast(1秒)/slow(30秒)
        lacp-rate: fast
        # 负载均衡算法：基于源目IP+端口
        transmit-hash-policy: layer3+4
        # 链路检测间隔（毫秒）
        mii-monitor-interval: 100
        # 链路恢复等待时间（毫秒）
        up-delay: 2000
        # 链路故障判定时间（毫秒）
        down-delay: 2000
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
  ethernets:
    enp0s3:
      optional: true
    enp0s8:
      optional: true
```

交换机侧配置参考（以华为为例）：

```text
# 创建聚合组并配置为LACP模式
interface Eth-Trunk1
 mode lacp-static
 # 添加成员端口
 trunkport GigabitEthernet 0/0/1 0/0/2
 # 配置LACP超时时间为快速模式
 lacp timeout fast
```

---

### 方案C：Bond + VLAN（高级场景）

```yaml
# /etc/netplan/02-bond-vlan.yaml
network:
  version: 2
  renderer: networkd
  bonds:
    bond0:
      interfaces:
        - enp0s3
        - enp0s8
      parameters:
        mode: 802.3ad
        mii-monitor-interval: 100
  vlans:
    # 业务VLAN（ID 100）
    bond0.100:
      id: 100
      link: bond0
      addresses:
        - 10.10.100.10/24
    # 管理VLAN（ID 200）
    bond0.200:
      id: 200
      link: bond0
      addresses:
        - 192.168.200.10/24
      routes:
        - to: default
          via: 192.168.200.1
      nameservers:
        addresses:
          - 8.8.8.8
  ethernets:
    enp0s3:
      optional: true
    enp0s8:
      optional: true
```

---

## 五、应用配置和验证命令（一键执行）

```bash
# 备份原有配置
sudo cp -r /etc/netplan /root/netplan.backup.$(date +%Y%m%d)

# 禁用cloud-init网络管理（关键步骤）
sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg << 'EOF'
network: {config: disabled}
EOF

# 清理cloud-init缓存
sudo rm -f /etc/netplan/50-cloud-init.yaml 2>/dev/null

# 创建/更新Netplan配置（以单网卡为例）
sudo tee /etc/netplan/01-netcfg.yaml << 'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      optional: true
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
          metric: 100
      nameservers:
        addresses:
          - 8.8.8.8
          - 114.114.114.114
EOF

# 设置等待超时（解决开机卡顿）
sudo mkdir -p /etc/systemd/system/systemd-networkd-wait-online.service.d/
sudo tee /etc/systemd/system/systemd-networkd-wait-online.service.d/timeout.conf << 'EOF'
[Service]
ExecStart=
ExecStart=/lib/systemd/systemd-networkd-wait-online --timeout=30
EOF
sudo systemctl daemon-reload

# 验证并应用配置
sudo netplan try && sudo netplan apply

# 验证IP地址配置
echo "=== IP地址 ===" && ip -br a

# 验证路由表
echo "=== 路由表 ===" && ip route

# 验证Bond状态（如已配置）
echo "=== Bond状态 ===" && cat /proc/net/bonding/bond0 2>/dev/null || echo "未配置bond"

# 连通性测试
echo "=== 连通性测试 ===" && ping -c 2 192.168.1.1 && ping -c 2 8.8.8.8
```

---

## 六、故障排查速查表

```bash
# 查看Netplan配置是否生效
sudo netplan status

# 查看bond接口状态
sudo networkctl status bond0

# 查看物理接口状态
sudo networkctl status enp0s3

# 查看bonding详细状态（内核级信息）
cat /proc/net/bonding/bond0

# 查看systemd-networkd日志
sudo journalctl -u systemd-networkd -b --no-pager -n 50

# 检查cloud-init是否干扰网络配置
cloud-init status --long
cat /var/log/cloud-init.log | grep -i network

# 临时测试网络（不重启，谨慎使用）
sudo ip link set bond0 up
sudo ip addr add 192.168.1.100/24 dev bond0
sudo ip route add default via 192.168.1.1

# 重置网络服务（谨慎使用）
sudo systemctl restart systemd-networkd
sudo netplan apply
```

---

## 七、检查清单

```text
[ ] 1. 确认网卡名称：ip link show
[ ] 2. 禁用cloud-init网络：/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
[ ] 3. 删除cloud-init生成的netplan文件：50-cloud-init.yaml
[ ] 4. Netplan文件名以 01- 开头确保优先级
[ ] 5. YAML格式验证：sudo netplan try（120秒超时保护）
[ ] 6. 配置开机等待超时：systemd-networkd-wait-online --timeout=30
[ ] 7. Bond配置时：物理网卡只声明不配IP，所有配置放在bond接口
[ ] 8. 多网卡场景：仅一个接口配置 default 路由
[ ] 9. 远程操作：保留一个未退出的SSH会话作为"救命通道"
[ ] 10. 变更后测试：ping网关 + ping外网IP + ping域名
```
