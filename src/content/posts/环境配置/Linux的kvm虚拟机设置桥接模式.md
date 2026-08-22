---
title: Linux的kvm虚拟机设置桥接模式
image: "api"
published: 2026-08-22
description: 在Linux宿主机上创建br0、br1网桥绑定物理网卡,让KVM虚拟机通过桥接模式接入局域网。
category: 环境配置
tags:
  - KVM
  - 虚拟机
  - 网络
slug: linux-kvm-bridge-mode
---

# Linux的kvm虚拟机设置桥接模式

## Linux KVM 虚拟机桥接网络配置（Bridge Mode）

### 目标

在 Linux 宿主机上配置 KVM 桥接网络，使虚拟机能够与局域网通信。通过创建 `br0` 和 `br1` 网桥设备，分别绑定物理网口 `eno1np0` 和 `eno2np1`，实现应用和数据库虚拟机的网络隔离。

---

### 步骤 1：查看物理网卡信息

确认当前系统的物理网卡名称：

```bash
ip addr show
```

假设：

- `eno1np0` 用于应用虚拟机 → 创建网桥 `br0`
- `eno2np1` 用于数据库虚拟机 → 创建网桥 `br1`

---

### 步骤 2：停止 NetworkManager 服务

为避免服务冲突，在修改传统网络脚本前先停止 NetworkManager：

```bash
systemctl stop NetworkManager
```

---

### 步骤 3：备份原始网卡配置文件

```bash
mv /etc/sysconfig/network-scripts/ifcfg-eno1np0 /etc/sysconfig/network-scripts/ifcfg-eno1np0.bak
mv /etc/sysconfig/network-scripts/ifcfg-eno2np1 /etc/sysconfig/network-scripts/ifcfg-eno2np1.bak
```

---

### 步骤 4：创建网桥设备 `br0`

编辑网桥配置文件：

```bash
nano /etc/sysconfig/network-scripts/ifcfg-br0
```

输入以下内容：

```ini
DEVICE=br0
TYPE=Bridge
ONBOOT=yes
NM_CONTROLLED=yes
BOOTPROTO=none
STP=on
DELAY=0
IPV6INIT=no
```

> 补充 `IPV6INIT=no` 可选字段以明确禁用 IPv6（根据实际需求调整）。

---

### 步骤 5：配置物理网卡 `eno1np0` 绑定到 `br0`

编辑物理网卡配置文件：

```bash
nano /etc/sysconfig/network-scripts/ifcfg-eno1np0
```

输入以下内容：

```ini
DEVICE=eno1np0
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=yes
BRIDGE=br0
IPV6INIT=no
```

---

### 步骤 6：创建网桥设备 `br1`

编辑网桥配置文件：

```bash
nano /etc/sysconfig/network-scripts/ifcfg-br1
```

输入以下内容：

```ini
DEVICE=br1
TYPE=Bridge
ONBOOT=yes
NM_CONTROLLED=yes
BOOTPROTO=none
STP=on
DELAY=0
IPV6INIT=no
```

---

### 步骤 7：配置物理网卡 `eno2np1` 绑定到 `br1`

编辑物理网卡配置文件：

```bash
nano /etc/sysconfig/network-scripts/ifcfg-eno2np1
```

输入以下内容：

```ini
DEVICE=eno2np1
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=yes
BRIDGE=br1
IPV6INIT=no
```

---

### 步骤 8：启动网桥并启用网络服务

重启 NetworkManager 以应用更改：

```bash
systemctl restart NetworkManager
```

验证网桥是否成功创建并启用：

```bash
ip link show br0
ip link show br1
bridge link
```

确认 `eno1np0` 和 `eno2np1` 是否已正确附加到对应的网桥。

---

### 步骤 9：配置 KVM 虚拟机使用桥接网络

KVM图形界面管理器

在硬件配置界面，选择网络。

模式选择桥接模式，设备填br0或者br1，按实际需求更改。

---

### 可选：宿主机 IP 配置说明

如需为宿主机本身分配 IP 地址，**不应分配给物理网卡**（因其已作为网桥端口），而应直接配置给网桥接口。

例如，在 `ifcfg-br0` 中添加静态 IP：

```ini
BOOTPROTO=static
IPADDR=192.168.1.10
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=8.8.8.8
```

然后重新加载网络配置。

---
