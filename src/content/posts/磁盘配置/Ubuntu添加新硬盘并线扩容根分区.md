---
title: Ubuntu添加新硬盘并线扩容根分区
image: "api"
published: 2026-08-22
description: Ubuntu不停机给LVM根分区扩容:新硬盘分区后加入卷组,扩容逻辑卷并刷新文件系统。
category: 磁盘配置
tags:
  - LVM
  - 磁盘扩容
  - 在线扩容
slug: ubuntu-add-disk-extend-root-partition
---

# Ubuntu添加新硬盘并线扩容根分区

ubuntu添加新硬盘，并在不停机的情况下扩容现有的lvm根分区

## ubuntu添加新硬盘在线扩容根分区

### 查看新加入的硬盘

```text
root@node1:~# lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0  63.9M  1 loop /snap/core20/2318
loop1                       7:1    0    87M  1 loop /snap/lxd/29351
loop2                       7:2    0  38.8M  1 loop /snap/snapd/21759
sda                         8:0    0   931G  0 disk
├─sda1                      8:1    0     1G  0 part /boot/efi
├─sda2                      8:2    0     2G  0 part /boot
└─sda3                      8:3    0 927.9G  0 part
└─ubuntu--vg-ubuntu--lv 252:0    0 927.9G  0 lvm  /
sdb                         8:16   0   1.8T  0 disk
```

新硬盘为 /dev/sdb

### 分区

`fdisk /dev/sdb`

```text
按顺序输入
查看设备 --> p
新建分区--> n
主分区---> p
分区号---> 回车
起始地址---> 回车
分区大小---> 回车
分区类型---> t
选择分区---> 1
lvm---> 8e
查看---> p
保存---> w
```

```text
root@node1:~# fdisk /dev/sdb

Welcome to fdisk (util-linux 2.37.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x0f86b880.

Command (m for help): p
Disk /dev/sdb: 1.82 TiB, 2000398934016 bytes, 3907029168 sectors
Disk model: ST2000LM007-1R81
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x0f86b880

Command (m for help): n
Partition type
p   primary (0 primary, 0 extended, 4 free)
e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-3907029167, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-3907029167, default 39070291                   67):

Created a new partition 1 of type 'Linux' and of size 1.8 TiB.

Command (m for help): t
Selected partition 1
Hex code or alias (type L to list all): L

00 Empty            24 NEC DOS          81 Minix / old Lin  bf Solaris
01 FAT12            27 Hidden NTFS Win  82 Linux swap / So  c1 DRDOS/sec (FAT-
02 XENIX root       39 Plan 9           83 Linux            c4 DRDOS/sec (FAT-
03 XENIX usr        3c PartitionMagic   84 OS/2 hidden or   c6 DRDOS/sec (FAT-
04 FAT16 <32M       40 Venix 80286      85 Linux extended   c7 Syrinx
05 Extended         41 PPC PReP Boot    86 NTFS volume set  da Non-FS data
06 FAT16            42 SFS              87 NTFS volume set  db CP/M / CTOS / .
07 HPFS/NTFS/exFAT  4d QNX4.x           88 Linux plaintext  de Dell Utility
08 AIX              4e QNX4.x 2nd part  8e Linux LVM        df BootIt
09 AIX bootable     4f QNX4.x 3rd part  93 Amoeba           e1 DOS access
0a OS/2 Boot Manag  50 OnTrack DM       94 Amoeba BBT       e3 DOS R/O
0b W95 FAT32        51 OnTrack DM6 Aux  9f BSD/OS           e4 SpeedStor
0c W95 FAT32 (LBA)  52 CP/M             a0 IBM Thinkpad hi  ea Linux extended
0e W95 FAT16 (LBA)  53 OnTrack DM6 Aux  a5 FreeBSD          eb BeOS fs
0f W95 Ext'd (LBA)  54 OnTrackDM6       a6 OpenBSD          ee GPT
10 OPUS             55 EZ-Drive         a7 NeXTSTEP         ef EFI (FAT-12/16/
11 Hidden FAT12     56 Golden Bow       a8 Darwin UFS       f0 Linux/PA-RISC b
12 Compaq diagnost  5c Priam Edisk      a9 NetBSD           f1 SpeedStor
14 Hidden FAT16 <3  61 SpeedStor        ab Darwin boot      f4 SpeedStor
16 Hidden FAT16     63 GNU HURD or Sys  af HFS / HFS+       f2 DOS secondary
17 Hidden HPFS/NTF  64 Novell Netware   b7 BSDI fs          fb VMware VMFS
18 AST SmartSleep   65 Novell Netware   b8 BSDI swap        fc VMware VMKCORE
1b Hidden W95 FAT3  70 DiskSecure Mult  bb Boot Wizard hid  fd Linux raid auto
1c Hidden W95 FAT3  75 PC/IX            bc Acronis FAT32 L  fe LANstep
1e Hidden W95 FAT1  80 Old Minix        be Solaris boot     ff BBT

Aliases:
linux          - 83
swap           - 82
extended       - 05
uefi           - EF
raid           - FD
lvm            - 8E
linuxex        - 85
Hex code or alias (type L to list all): 8e
Changed type of partition 'Linux' to 'Linux LVM'.

Command (m for help): p
Disk /dev/sdb: 1.82 TiB, 2000398934016 bytes, 3907029168 sectors
Disk model: ST2000LM007-1R81
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x0f86b880

Device     Boot Start        End    Sectors  Size Id Type
/dev/sdb1        2048 3907029167 3907027120  1.8T 8e Linux LVM

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

### 扩容

`pvcreate /dev/sdb1` 创建物理卷PV

```text
root@node1:~# pvcreate /dev/sdb1
Physical volume "/dev/sdb1" successfully created.
```

`pvdisplay` 查看物理卷

```text
root@node1:~# pvdisplay
--- Physical volume ---
PV Name               /dev/sda3
VG Name               ubuntu-vg
PV Size               <927.95 GiB / not usable 3.00 MiB
Allocatable           yes (but full)
PE Size               4.00 MiB
Total PE              237554
Free PE               0
Allocated PE          237554
PV UUID               Xwor6X-qxh9-P05z-GzxV-GZoo-RXFE-XeuXF8

"/dev/sdb1" is a new physical volume of "<1.82 TiB"
--- NEW Physical volume ---
PV Name               /dev/sdb1
VG Name
PV Size               <1.82 TiB
Allocatable           NO
PE Size               0
Total PE              0
Free PE               0
Allocated PE          0
PV UUID               JHGh9m-l919-fxKT-RuhP-0UOH-pEAB-7tP0fN
```

`vgdisplay` 查看卷组

```text
root@node1:~# vgdisplay
--- Volume group ---
VG Name               ubuntu-vg
System ID
Format                lvm2
Metadata Areas        1
Metadata Sequence No  3
VG Access             read/write
VG Status             resizable
MAX LV                0
Cur LV                1
Open LV               1
Max PV                0
Cur PV                1
Act PV                1
VG Size               <927.95 GiB
PE Size               4.00 MiB
Total PE              237554
Alloc PE / Size       237554 / <927.95 GiB
Free  PE / Size       0 / 0
VG UUID               cLFpMk-poC2-kyVl-GhZl-Ghi5-6bZE-nnv3SM
```

要扩容的vg卷组名为 ubuntu-vg

`vgextend ubuntu-vg /dev/sdb1` 将新硬盘添加到卷组

```text
root@node1:~# vgextend ubuntu-vg /dev/sdb1
Volume group "ubuntu-vg" successfully extended
```

`vgdisplay` 查看卷组

```text
root@node1:~# vgdisplay
--- Volume group ---
VG Name               ubuntu-vg
System ID
Format                lvm2
Metadata Areas        2
Metadata Sequence No  4
VG Access             read/write
VG Status             resizable
MAX LV                0
Cur LV                1
Open LV               1
Max PV                0
Cur PV                2
Act PV                2
VG Size               <2.73 TiB
PE Size               4.00 MiB
Total PE              714485
Alloc PE / Size       237554 / <927.95 GiB
Free  PE / Size       476931 / <1.82 TiB
VG UUID               cLFpMk-poC2-kyVl-GhZl-Ghi5-6bZE-nnv3SM
```

新加入的硬盘已经变ubuntu-vg的可用空间

`lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv` #扩容逻辑卷

```text
root@node1:~# lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
Size of logical volume ubuntu-vg/ubuntu-lv changed from <927.95 GiB (237554 ex                   tents) to <2.73 TiB (714485 extents).
Logical volume ubuntu-vg/ubuntu-lv successfully resized.
```

扩容完成后空间还是用不了，需要刷新逻辑卷容量

`resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv` #刷新逻辑卷

```text
使用df -hT 查看扩容的分区文件系统
ext4文件系统用resize2fs命令
xfs文件系统用xfs_growfs命令
```

```text
root@node1:~# resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mapper/ubuntu--vg-ubuntu--lv is mounted on /; on-line resizin                   g required
old_desc_blocks = 116, new_desc_blocks = 349
The filesystem on /dev/mapper/ubuntu--vg-ubuntu--lv is now 731632640 (4k) blocks                    long.
```

扩容完成
