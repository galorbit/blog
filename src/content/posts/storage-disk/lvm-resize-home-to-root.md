---
title: "CentOS不重启将/home空间分配给根目录"
published: 2026-08-04
description: "介绍在CentOS系统中不重启的情况下，通过LVM工具将/home分区空间重新分配给根目录/的操作步骤。"
category: "存储与磁盘"
tags:
  - "LVM"
  - "CentOS"
  - "磁盘扩容"
  - "XFS文件系统"
  - "逻辑卷管理"
---

# CentOS不重启将/home空间分配给根目录

## 备份 /home 目录

使用 LVM 扩展 `/` 分区，需要先将 `/home` 的逻辑卷删除，首先备份 `/home` 下的文件。

```bash
mkdir /backup
# 创建备份目录
tar -cvf /backup/home.tar /home
# 打包/home目录到备份目录
```

等待打包完成。

## 扩展根目录

手动停止使用 `/home` 目录的进程，或使用 `fuser -km /home` 强制停止。如果提示没有 `fuser`，手动安装 `yum install psmisc`。

### 卸载并删除 /home 逻辑卷

卸载 `/home` 分区：
```bash
umount /dev/mapper/centos-home
```

删除 `/home` 的 LV 卷：
```bash
lvremove /dev/mapper/centos-home
```

### 扩展 /root 逻辑卷与文件系统

扩展 `/root` 的 LV 卷（500G 为扩展的容量，按需填写）：
```bash
lvextend -L +500G /dev/mapper/centos-root
```

扩展 `/root` 的文件系统：
```bash
xfs_growfs /dev/mapper/centos-root
```

完成后可以使用 `df -hT` 查看 root 卷容量是否已扩展成功。

### 重建 /home 逻辑卷并恢复数据

重新创建 `/home` 的 LV 卷（10G 为创建 home 卷的容量，按需填写）：
```bash
lvcreate -L 10G -n home /dev/mapper/centos
```

创建文件系统：
```bash
mkfs.xfs /dev/mapper/centos-home
```

挂载 `/home`：
```bash
mount /dev/mapper/centos-home /home
```

恢复备份：
```bash
tar -xvf /backup/home.tar -C /home
```

解压完成后如果目录是 `/home/home/*` 这种路径，请手动移动一下。
