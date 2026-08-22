---
title: Linux挂载硬盘
image: "api"
published: 2026-08-22
description: Linux挂载新硬盘全流程:parted分区、格式化、临时挂载,再用UUID写入fstab开机自动挂载。
category: 磁盘配置
tags:
  - Linux硬盘
  - 挂载
  - fstab
slug: linux-mount-hard-disk
---

# Linux挂载硬盘

## 查看硬盘

使用`lsblk`命令，列出所有设备，查看需要挂载硬盘的物理路径

我这边需要挂载的硬盘为 `/dev/sdb`

## 硬盘分区

选择需要分区的硬盘，这里使用parted工具。

`parted /dev/sdb`

可以输入“？”来查看parted命令介绍。

创建磁盘分区表：

`mklabel gpt` 输入yes

创建分区，并分配大小：

```text
mkpart
分区名 回车
文件系统 ext4或其他
分区起始点 1Mib
分区结束点 100%

```

---

直接使用`mkpart`命令不加参数来分区，起始点和结束点默认单位为M，可以使用`unit GB`命令来修改默认单位为GB。起始点位置前建议预留一些空间，不要从0开始。

如果设置错误，可以使用`rm 分区编号`命令删除，分区编号可以使用`print`查看。

使用`quit`退出。

## 格式化

`mkfs.ext4 /dev/sdb1` 这里选择格式化为ext4文件系统，根据需求选择其他文件系统也可以。如果提示已有文件格式，添加 -F 选项强制覆盖。

## 挂载

- 创建挂载点

`mkdir /data`

- 临时挂载：

`mount /dev/sdb1 /data`

使用`df -h`命令可以查看到/dev/sdb分区已经挂载到了/data目录了。

- 取消挂载：

`umount /dev/sdb1`或`umount /data`都可以

临时挂载重启后会取消挂载。

- 设置开机自动挂载硬盘：

自动挂载硬盘选择对应分区的UUID来挂载，防止设备号会变。UUID唯一标识每一个分区，确保不会出现错误的挂载。

使用`blkid /dev/sdb1`查看分区UUID

输出示例：

`/dev/sdb1: UUID="8efdf108-963f-476d-8992-581d37b13909" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="0b455b11-df18-47b6-a2ab-441848562ae8"`

复制`UUID="8efdf108-963f-476d-8992-581d37b13909`

编辑 `/etc/fstab`文件，将参数写入，实现自动挂载。

`echo 'UUID=8efdf108-963f-476d-8992-581d37b13909 /data ext4 defaults 0 0' | sudo tee -a /etc/fstab`

将参数写入/etc/fstab文件的最后一行，注意检查是否有错误，也可以直接使用`vim /etc/fstab`命令手动编辑。

```text
/etc/fstab文件里有六项参数

第一列：设备文件或UUID或者label

第二列：设备的挂载点（挂载目录）

第三列：该文件系统的格式（可以使用auto参数自动识别）

第四列：文件系统的参数

第五列：dump备份的设置（0表示不进行dump备份，1表示每天进行dump备份，2表示不定日期进行dump备份。通常设置为0）

第六列：磁盘检查设置（0为不检查）
```

## 验证

执行`df -h`可以看到是否挂载成功

如果前面没有执行`mount /dev/sdb1 /data`，此时应该是看不到挂载的，内核还没有读取`/etc/fstab`这个文件，执行`mount -a`,然后再执行`df -h`，就可以看到/dev/sdb1已经成功挂载到了/data目录下。

重启后也可以看到/dev/sdb1分区自动挂载到了/data

## 扩容

查看[链接](https://support.huaweicloud.com/usermanual-dss/dss_01_2311.html)
