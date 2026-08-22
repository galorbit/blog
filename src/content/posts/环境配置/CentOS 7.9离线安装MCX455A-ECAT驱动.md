---
title: CentOS 7.9离线安装MCX455A-ECAT驱动
image: "api"
published: 2026-08-22
description: 在CentOS 7.9上通过本地YUM仓库离线安装Mellanox MCX455A-ECAT驱动，并配置InfiniBand设备与服务。
category: 环境配置
tags:
  - 离线安装
  - Mellanox驱动
  - InfiniBand
slug: centos-7-9-offline-install-mcx455a-ecat-driver
---

## CentOS 7.9离线安装MCX455A-ECAT驱动

### 操作系统平台

RHEL/CentOS 7.9

### 准备本地 YUM 仓库

#### 1. 挂载系统安装光盘

> [!NOTE]
> 确保系统已插入操作系统安装光盘，且系统识别到光驱设备。大多数情况下，光驱设备为 /dev/sr0，但某些系统可能为 /dev/cdrom 或其他设备名。如遇问题，可使用 `lsblk` 命令查看可用设备。

```bash
# 检查光驱设备
lsblk

# 创建挂载点
mkdir -p /media/cdrom

# 挂载光盘（指定文件系统类型为iso9660）
mount -t iso9660 /dev/sr0 /media/cdrom
```

#### 2. 备份原有 YUM 仓库配置

```bash
# 创建备份目录
mkdir -p /etc/yum.repos.d/backup

# 备份现有仓库配置
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup/
```

#### 3. 创建本地仓库配置文件

```bash
vi /etc/yum.repos.d/local.repo
```

```ini
[local-centos]
name=Local CentOS Repository
baseurl=file:///media/cdrom
enabled=1
gpgcheck=0
```

> [!WARNING]
> 将 gpgcheck 设置为 0 仅建议在测试环境中使用。在生产环境中，应保留 GPG 检查功能并导入适当的 GPG 密钥以保证软件包的完整性。

### 安装必要依赖

```bash
# 清理并重建 YUM 缓存
yum clean all
yum makecache

# 安装基础开发工具和 IB 卡相关依赖
yum install -y nano vim python-devel pciutils lsof libusbx tcl fuse-libs tcsh tk usbutils
```

### 安装 Mellanox OFED

#### 1. 解压安装包

```bash
unzip MLNX_OFED_LINUX-5.6-2.0.9.0-rhel7.9-x86_64.zip
cd MLNX_OFED_LINUX-5.6-2.0.9.0-rhel7.9-x86_64
```

#### 2. 授予执行权限并安装

```bash
# 授予安装脚本执行权限
chmod +x mlnx_add_kernel_support.sh mlnxofedinstall

# 执行安装（添加内核支持并跳过仓库配置）
./mlnxofedinstall --add-kernel-support --skip-repo
```

> [!NOTE]
> `--add-kernel-support` 参数确保 OFED 驱动与当前内核版本兼容；`--skip-repo` 参数避免覆盖现有仓库配置。安装过程会自动检测并安装必要的内核模块和用户空间库。

#### 3. 更新 initramfs 并重启服务

```bash
# 更新 initramfs 以包含新的驱动模块
dracut -f

# 重启 InfiniBand 服务
/etc/init.d/openibd restart

# 设置服务开机自启
systemctl enable openibd
systemctl enable mst
```

### 配置 InfiniBand 设备

#### 1. 检查设备状态

```bash
# 检查 MST 驱动状态
mst status

# 查看 IB 设备信息
ibdev2netdev
ibstatus
```

#### 2. 配置 IB 卡为InfiniBand模式

```bash
# 以 Mellanox ConnectX-4/5 卡为例设置链路类型
# 请替换 [DEVICE] 为实际设备名，可通过 mst status 获取
mlxconfig -d /dev/mst/[DEVICE] set LINK_TYPE_P1=1
```

> [!WARNING]
> 实际设备路径应通过 `mst status` 命令确认，如 /dev/mst/mt4115_pciconf0。请勿直接使用笔记中的示例路径。

### 服务管理说明

#### OpenIBD 服务

> [!IMPORTANT]
> openibd 是系统初始化脚本，负责在 Linux 系统启动时自动加载所有必要的 RDMA/InfiniBand 内核模块，必须设置为开机自启。

#### MST 服务

> [!NOTE]
> MST (Mellanox Software Tools) 用于设备发现与状态检查：
> -`mst status`: 检查设备状态
> -`mlxconfig`: 配置和管理 IB/HCA 设备
> -`ibqueryerrors`, `perfquery`: 查询端口信息和性能计数器
>
> 应设置 MST 服务开机自启以便持续管理设备。

#### OpenSM 服务（子网管理器）

> [!WARNING]
> 在 IB 网络中，OpenSM 负责：
>
> - 拓扑发现：扫描整个 IB Fabric，识别所有节点和交换机
> - LID 分配：给每个 IB 端口分配唯一的本地标识符
> - 路由计算：生成路由表
> - 网络状态维护：监控链路健康状况
>
> **重要提示**：整个子网内只需一台服务器启动 OpenSM 并设置开机自启，其他机器必须禁用 OpenSM 服务，否则会导致服务冲突。

### 验证安装

```bash
# 检查 IB 端口状态
ibstatus

# 查看 HCA 设备信息
ibdev2ibdev

# 检查网络设备映射
ibdev2netdev

# 验证子网管理器（若本机为 OpenSM 主机）
opensm --version
systemctl status opensm
```

> [!TIP]
> 完成配置后，建议通过以下命令测试 RDMA 功能：
>
> ```bash
> # 在一台服务器上
> ib_send_lat -R
> 
> # 在另一台服务器上
> ib_send_lat <server-ip>
> ```
