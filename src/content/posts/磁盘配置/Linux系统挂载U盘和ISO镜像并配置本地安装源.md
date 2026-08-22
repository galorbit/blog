---
title: Linux系统挂载U盘和ISO镜像并配置本地安装源
image: "api"
published: 2026-08-22
description: 在Linux上挂载U盘和ISO镜像,配置本地yum或apt源,实现在无网络环境下安装软件包。
category: 磁盘配置
tags:
  - 挂载
  - ISO镜像
  - 本地软件源
slug: linux-mount-usb-iso-local-repo
---

# Linux系统挂载U盘和ISO镜像并配置本地安装源

挂载U盘复制系统安装镜像到服务器，挂载系统镜像并配置本地yum或apt源

## 挂载U盘

U盘格式为NTFS

FAT32不支持单文件大于4G，如系统镜像大于4G，就需要将U盘格式化为NTFS文件系统。

Linux 5.15以上内核才原生支持NTFS文件系统挂载，如内核版本低于5.15，安装NTFS-3G以支持NTFS文件系统。

---

U盘格式为FAT32

创建挂载文件夹

`mkdir /mnt/usb`

使用 `fdisk -l`查看U盘设备

如我的U盘为路径为`/dev/sdb1`

将U盘挂载到新建的usb目录

`mount -t vfat /dev/sdb1 /mnt/usb`

此时可以在/mnt/usb这个目录下访问U盘了

取消挂载U盘

`umount /dev/sdb1 /mnt/usb`

然后删除刚刚创建的文件夹

`rm -rf /mnt/usb`

## 挂载系统镜像

镜像所在目录：`/home/data/centos-xxxxx.iso`

创建挂载目录：`mkdir /mnt/cdrom`

挂载镜像：`mount -t iso9660 /home/data/centos-xxxxx.iso /mnt/cdrom`

检查挂载：`df -h`

取消挂载：`umount /mnt/cdrom`

## 配置yum源

首先将**CentOS-Base.repo**和**CentOS-Debuginfo.repo**改名，绕过网络安装。[^1]

`mv CentOS-Base.repo CentOS-Base.repo.bak`

`mv CentOS-Debuginfo.repo CentOS-Debuginfo.repo.bak`

创建本地源

`nano /etc/yum.repos.d/localyum.repo`

添加以下内容

```shell
[localyum]

name=localyum
baseurl=file:///mnt/cdrom #此路径为挂载的系统镜像路径
gpgcheck=0 #gpg校验，0是关闭，1是开启。不需要校验所以为0
enabled=1 #1是启用，0是关闭。
```

清除yum缓存

`yum clean all`

重建缓存

`yum makecache`

操作没有问题的话此时应该可以用刚刚创建的源来安装软件了，但仅仅可安装**Centos镜像本身所包含的软件包和依赖**

通常适用于新机器最小化安装老版本的Centos，没有网卡驱动，最小化安装又没有Gcc make这些必要工具去编译驱动。手动安装依赖太麻烦的选择。

## 配置apt源

ubuntu系统镜像挂载到 `/media/cdrom`(一定要挂载到这个目录，或者如果知道怎么解决源和目录问题的话挂载到哪里都可以)

```shell
mkdir /media/cdrom
mount ubuntu-18.04.6-desktop-amd64.iso /media/cdrom
```

备份并修改为本地软件源

```shell
cp /etc/apt/sources.list sources.list.bak
rm /etc/apt/sources.list ##直接删掉，后续会自动创建
apt-cdrom -m -d /media/cdrom add ##会自动创建源
```

## 其他

最小化安装的系统如果没有nano,使用vi来编辑文件。

如果后续需要使用网络源，重命名的文件改回去。

---

此方法是临时挂载，重启会失效。需要重启不失效，设置开机自动挂载。

`vi /etc/fstab`

加入下面内容

`/home/data/centos-xxxxxx.iso /mnt/cdrom/ iso9600 defaults,loop,ro 0 0`

保存即可

[^1]: 这里将网络源改名了，如果是没法访问网络，临时需要使用本地源安装软件，后续需要改回来，不然无法使用网络安装。
