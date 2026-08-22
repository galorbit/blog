---
title: Lvm将home容量扩容到根目录
image: "api"
published: 2026-08-22
description: 通过LVM把/home容量在不关机情况下分给根目录,删home卷扩容root后再重建并恢复备份。
category: 磁盘配置
tags:
  - LVM
  - 磁盘扩容
  - 逻辑卷
slug: lvm-home-to-root-expand
---

# Lvm将home容量扩容到根目录

> centos安装不手动分配容量会自动分配到/home,在不关机的情况下将/home的容量重新分配到/目录

## 备份/home目录

使用lvm扩展/分区，需要先将/home的逻辑卷删除，首先备份/home下的文件。

```text
mkdir /backuup
创建备份目录
tar -cvf /backup/home.tar /home
打包/home目录到备份目录
```

等待打包完成

## 扩展根目录

手动停止使用home目录的进程，或使用`fuser -km /home`强制停止。
如果提示没有fuser，手动安装`yum install psmisc`。

卸载home分区
`umount /dev/mapper/centos-home`

删除/home的lv卷
`lvremove /dev/mapper/centos-home`

扩展/root的lv卷
`lvextend -L +500G /dev/mapper/centos-root`
500G为扩展的容量，按需填写

扩展/root的文件系统
`xfs_growfs /dev/mapper/centos-root`
完成后可以使用`df -hT`看到root卷容量已经扩展成功。

重新创建/home的lv卷
`lvcreate -L 10G -n /dev/mapper/centos-home`
10G为创建home卷的容量，按需填写

创建文件系统
`mkfs.xfs /dev/mapper/centos-home`

挂载/home
`mount /dev/mapper/centos-home`

恢复备份
`tar -xvf /backup/home.tar -C /home`
解压完成后如果目录是`/home/home/*`这种路径，手动移动一下。
