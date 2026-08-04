---
title: "挂载U盘与系统镜像并配置本地软件源"
published: 2026-08-04
description: "本文介绍在Linux中挂载U盘与系统ISO镜像，并配置本地YUM与APT软件源的详细步骤。"
category: "存储与磁盘"
tags:
  - "U盘挂载"
  - "系统镜像"
  - "本地YUM源"
  - "本地APT源"
  - "存储与磁盘"
---

# 挂载U盘与系统镜像并配置本地软件源

## 挂载U盘

U盘格式为 NTFS。FAT32 不支持单文件大于 4G，如系统镜像大于 4G，就需要将 U盘格式化为 NTFS 文件系统。Linux 5.15 以上内核才原生支持 NTFS 文件系统挂载，如内核版本低于 5.15，安装 ntfs-3g 以支持 NTFS 文件系统。

U盘格式为 FAT32。

创建挂载文件夹：

```bash
mkdir /mnt/usb
```

使用 `fdisk -l` 查看 U盘设备。如我的 U盘路径为 `/dev/sdb1`。

将 U盘挂载到新建的 usb 目录：

```bash
mount -t vfat /dev/sdb1 /mnt/usb
```

此时可以在 `/mnt/usb` 目录下访问 U盘了。

取消挂载 U盘：

```bash
umount /mnt/usb
```

然后删除刚刚创建的文件夹：

```bash
rm -rf /mnt/usb
```

## 挂载系统镜像

镜像所在目录：`/home/data/centos-xxxxx.iso`

创建挂载目录：

```bash
mkdir /mnt/cdrom
```

挂载镜像：

```bash
mount -t iso9660 /home/data/centos-xxxxx.iso /mnt/cdrom
```

检查挂载：

```bash
df -h
```

取消挂载：

```bash
umount /mnt/cdrom
```

## 配置 YUM 源

首先将 `CentOS-Base.repo` 和 `CentOS-Debuginfo.repo` 改名，绕过网络安装。[^1]

```bash
mv CentOS-Base.repo CentOS-Base.repo.bak
mv CentOS-Debuginfo.repo CentOS-Debuginfo.repo.bak
```

创建本地源：

```bash
nano /etc/yum.repos.d/localyum.repo
```

添加以下内容：

```ini
[localyum]
name=localyum
baseurl=file:///mnt/cdrom # 此路径为挂载的系统镜像路径
gpgcheck=0 # gpg校验，0是关闭，1是开启。不需要校验所以为0
enabled=1 # 1是启用，0是关闭。
```

清除 YUM 缓存：

```bash
yum clean all
```

重建缓存：

```bash
yum makecache
```

操作没有问题的话此时应该可以用刚刚创建的源来安装软件了，但仅仅可安装 **CentOS 镜像本身所包含的软件包和依赖**。通常适用于新机器最小化安装老版本的 CentOS，没有网卡驱动，最小化安装又没有 gcc、make 这些必要工具去编译驱动。手动安装依赖太麻烦的选择。

## 配置 APT 源

Ubuntu 系统镜像挂载到 `/media/cdrom`（一定要挂载到这个目录，或者如果知道怎么解决源和目录问题的话挂载到哪里都可以）。

```bash
mkdir /media/cdrom
mount -t iso9660 -o loop ubuntu-18.04.6-desktop-amd64.iso /media/cdrom
```

备份并修改为本地软件源：

```bash
cp /etc/apt/sources.list sources.list.bak
rm /etc/apt/sources.list # 直接删掉，后续会自动创建
apt-cdrom -m -d /media/cdrom add # 会自动创建源
```

## 其他说明

最小化安装的系统如果没有 `nano`，使用 `vi` 来编辑文件。

如果后续需要使用网络源，重命名的文件改回去。

此方法是临时挂载，重启会失效。需要重启不失效，设置开机自动挂载：

```bash
vi /etc/fstab
```

加入下面内容：

```text
/home/data/centos-xxxxxx.iso /mnt/cdrom iso9660 defaults,loop,ro 0 0
```

保存即可。

[^1]: 这里将网络源改名了，如果是没法访问网络，临时需要使用本地源安装软件，后续需要改回来，不然无法使用网络安装。
