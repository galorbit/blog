---
title: "Linux 软件 RAID 创建与管理指南（mdadm）"
published: 2026-08-04
description: "本文详细讲解基于 mdadm 工具在 Linux 系统中创建、配置、监控及维护软件 RAID 阵列的完整流程与常见问题排查。"
category: "存储与磁盘"
tags:
  - "mdadm"
  - "Linux"
  - "RAID"
  - "存储配置"
  - "系统运维"
---

# Linux 软件 RAID 创建与管理指南（mdadm）

## 环境准备

### 安装 mdadm

```bash
# Debian / Ubuntu
sudo apt install mdadm

# RHEL / CentOS / openEuler / Rocky
sudo dnf install mdadm
```

### 确认磁盘设备

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
```

确保目标磁盘无重要数据（操作会清空数据）。

## 创建 RAID

### 方案一：使用整块磁盘（直接使用 /dev/nvmeXnY）

```bash
sudo mdadm --create --verbose /dev/md0 \
  --level=5 \
  --raid-devices=4 \
  /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1
```

> **建议**：最好先创建分区（见方案二），避免内核自动扫描干扰。

### 方案二：先分区再创建 RAID（推荐）

使用 `gdisk` 或 `fdisk` 创建类型为 `Linux RAID` 的分区：

```bash
# 以 /dev/nvme0n1 为例，其余磁盘相同操作
sudo gdisk /dev/nvme0n1
```

gdisk 操作步骤：

1. `o` — 创建新的 GPT 分区表
2. `n` — 新建分区
3. 分区号默认、起始扇区默认、结束扇区默认（使用整盘）
4. Hex code 输入 `fd00`（Linux RAID 自动检测）
5. `w` — 写入并退出

其余磁盘重复上述步骤。完成后确认：

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE
```

输出应类似：

```
nvme0n1      xxxG  disk
├─nvme0n1p1  xxxG  part  linux_raid_member
nvme1n1      xxxG  disk
├─nvme1n1p1  xxxG  part  linux_raid_member
...
```

创建 RAID：

```bash
sudo mdadm --create --verbose /dev/md0 \
  --level=5 \
  --raid-devices=4 \
  /dev/nvme0n1p1 /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
```

## 监控创建进度

```bash
watch -n 1 cat /proc/mdstat
```

> RAID 5 创建时会进行后台同步（resync），同步期间阵列可用但性能较低。大容量磁盘（数 TB）同步可能需要数小时。

查看阵列详细信息：

```bash
sudo mdadm --detail /dev/md0
```

## 创建文件系统

```bash
# XFS（推荐用于大容量、大文件场景）
sudo mkfs.xfs /dev/md0

# 或 ext4（通用场景）
sudo mkfs.ext4 /dev/md0
```

> **注意**：创建时选择的文件系统应与挂载配置一致。

## 挂载并持久化配置

### 挂载

```bash
sudo mkdir -p /data
sudo mount /dev/md0 /data
df -h /data
```

### 保存 RAID 配置

```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf
```

查看保存结果：

```bash
cat /etc/mdadm.conf
```

### 写入 fstab（开机自动挂载）

获取 UUID：

```bash
sudo blkid /dev/md0
```

示例输出：

```
/dev/md0: UUID="8d398eb1-64e3-47a1-9066-9f8f4fdd6d12" TYPE="xfs"
```

添加自动挂载条目：

```bash
echo 'UUID=8d398eb1-64e3-47a1-9066-9f8f4fdd6d12 /data xfs defaults 0 0' | sudo tee -a /etc/fstab
```

验证 fstab 配置正确：

```bash
sudo mount -a
```

> **注意**：fsck 检查顺序（最后一列）对于 RAID 设备通常设为 `0`，由 mdadm 自身处理一致性检查。

## 验证与测试

```bash
# 查看阵列状态
cat /proc/mdstat

# 详细信息
sudo mdadm --detail /dev/md0

# 模拟磁盘故障（测试容错，不要在生产环境随意操作）
sudo mdadm /dev/md0 --fail /dev/nvme0n1p1
sudo mdadm --detail /dev/md0   # 查看降级状态

# 移除故障盘
sudo mdadm /dev/md0 --remove /dev/nvme0n1p1

# 添加热备盘
sudo mdadm /dev/md0 --add /dev/nvme4n1p1

# 恢复故障盘重新加入（需先 wipefs）
sudo mdadm /dev/md0 --add /dev/nvme0n1p1
```

## 阵列管理

### 停止阵列

```bash
sudo umount /data
sudo mdadm --stop /dev/md0
```

### 重新组装（重启后自动）

如果配置已写入 `/etc/mdadm.conf`，重启后会自动组装。手动组装：

```bash
sudo mdadm --assemble --scan
```

或指定设备：

```bash
sudo mdadm --assemble /dev/md0 /dev/nvme0n1p1 /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
```

### 删除阵列（彻底清除数据）

```bash
sudo umount /data
sudo mdadm --stop /dev/md0
sudo mdadm --zero-superblock /dev/nvme0n1p1
sudo mdadm --zero-superblock /dev/nvme1n1p1
sudo mdadm --zero-superblock /dev/nvme2n1p1
sudo mdadm --zero-superblock /dev/nvme3n1p1
```

## RAID 级别选择参考

| 级别   | 最少磁盘 | 可用容量 | 容错   | 读写性能     | 适用场景                     |
|--------|----------|----------|--------|--------------|------------------------------|
| RAID 0 | 2        | 100%     | 无     | 读写↑        | 临时/缓存，不重要的高速存储  |
| RAID 1 | 2        | 50%      | 1 盘   | 读↑ 写=      | 系统盘/关键数据              |
| RAID 5 | 3        | (N-1)/N  | 1 盘   | 读↑ 写↓      | 通用存储，性价比高           |
| RAID 6 | 4        | (N-2)/N  | 2 盘   | 读↑ 写↓↓     | 大容量数据安全               |
| RAID 10| 4        | 50%      | 每组1盘| 读写↑        | 数据库/高IO场景              |

## 常见问题

### 重启后阵列未自动组装

- 确认 `/etc/mdadm.conf` 配置正确
- 检查 `mdadm` 服务是否启用：
  ```bash
  sudo systemctl enable mdmonitor
  sudo systemctl start mdmonitor
  ```
- 手动组装：`sudo mdadm --assemble --scan`

### 磁盘故障后系统无法启动

如果系统根目录在 RAID 上，需在 initramfs 中包含 mdadm 配置：

```bash
# Debian/Ubuntu
sudo update-initramfs -u

# RHEL/openEuler
sudo dracut -f
```

### 创建 RAID 时提示设备正在使用

```bash
# 检查是否已被其他进程占用
lsblk
# 清除 superblock
sudo wipefs -a /dev/nvme0n1p1
```

## 参考命令速查

```bash
# 查看 RAID 状态
cat /proc/mdstat
sudo mdadm --detail /dev/md0

# 查看磁盘 UUID 和文件系统
sudo blkid

# 查看所有块设备
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT

# 查看 mdadm 配置
cat /etc/mdadm.conf
```
