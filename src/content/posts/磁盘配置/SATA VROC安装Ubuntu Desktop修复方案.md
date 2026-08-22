---
title: SATA VROC安装Ubuntu Desktop修复方案
published: 2026-08-22
description: 修复Intel SATA VROC的RAID5磁盘安装Ubuntu桌面版后重启进入BusyBox的问题,在initramfs中集成dmraid与kpartx激活RAID卷。
category: 磁盘配置
tags:
  - VROC
  - dmraid
  - 引导修复
slug: sata-vroc-ubuntu-desktop-fix
---

# SATA VROC安装Ubuntu Desktop修复方案

---

## 一、问题概述

### 现象

- 服务器使用 Intel SATA VROC 创建 RAID5
- 安装 Ubuntu 22.04 Desktop 时能识别 RAID 磁盘，安装过程正常
- 安装完成重启后进入 **BusyBox**，提示：

```text
  Gave up waiting for root file system device.
```

### 根因

```text
BIOS/VROC RAID5（Intel imsm 元数据）
 ↓
Linux 内核通过 VMD 驱动暴露物理磁盘（/dev/sda、sdb、sdc）
 ↓
需 dmraid 工具将物理磁盘组装为逻辑 RAID 卷
 ↓
dmraid -ay 创建 /dev/mapper/isw_* 整盘设备
 ↓
kpartx -av 读取分区表创建分区设备（p1、p2）
 ↓
⚠ 安装程序生成的 initramfs 未包含上述工具 → 启动时找不到根分区
```

### 关键特征识别

|特征|含义|
| -------------------------------| ----------------------------------------|
|`lsblk` 显示 `isw_` 前缀|Intel Matrix Storage（imsm）元数据格式|
|单盘类型为 `disk`，RAID 类型为 `dmraid`|FakeRAID / dmraid 模式|
|`/dev/mapper/` 下只有整盘无分区|缺少 `kpartx` 分区映射|

---

## 二、完整修复流程

### 2.1 准备 Live 环境

```bash
# 1. 从 Ubuntu 22.04 Live USB 启动，选择 "Try Ubuntu"
# 2. 连接网络
# 3. 打开终端
```

### 2.2 安装工具并激活 RAID

```bash
# 更新软件源
sudo apt update

# 安装核心工具
sudo apt install dmraid kpartx -y

# 扫描 RAID 元数据（确认识别）
sudo dmraid -r

# 预期输出：
# /dev/sda: isw, "isw_bcehdedef_Volume0", GROUP, ok, ...
# /dev/sdb: isw, "isw_bcehdedef_Volume0", GROUP, ok, ...
# /dev/sdc: isw, "isw_bcehdedef_Volume0", GROUP, ok, ...

# 查看 RAID 集信息
sudo dmraid -s

# 激活所有 RAID 卷
sudo dmraid -ay

# 创建分区映射（关键步骤！）
sudo kpartx -av /dev/mapper/isw_bcehdedef_Volume0

# 确认设备已生成
ls -la /dev/mapper/isw_bcehdedef_Volume0*
```

### 2.3 挂载已安装系统

```bash
# 设定变量（根据实际设备名调整）
ROOT_DEV="/dev/mapper/isw_bcehdedef_Volume0p2"
EFI_DEV="/dev/mapper/isw_bcehdedef_Volume0p1"

# 挂载根分区
sudo mount $ROOT_DEV /mnt

# 验证挂载正确
ls /mnt
# 应显示：bin boot dev etc home lib ... usr var

# 挂载 EFI 分区
sudo mount $EFI_DEV /mnt/boot/efi

# 挂载虚拟文件系统
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
```

### 2.4 chroot 并执行修复

```bash
sudo chroot /mnt
```

进入 chroot 后，依次执行以下四组操作：

#### A. 确认软件包已安装

```bash
apt update
apt install dmraid kpartx -y
```

#### B. 配置 initramfs 内核模块

```bash
cat >> /etc/initramfs-tools/modules << 'EOF'
dm-mod
dm-mirror
dm-raid
dm-zero
EOF
```

#### C. 创建 dmraid 引导激活脚本

```bash
cat > /etc/initramfs-tools/scripts/local-top/dmraid << 'SCRIPT'
#!/bin/sh

PREREQ=""
prereqs()
{
 echo "$PREREQ"
}

case $1 in
prereqs)
 prereqs
 exit 0
 ;;
esac

echo "Loading device-mapper modules..."
modprobe -q dm-mod 2>/dev/null
modprobe -q dm-mirror 2>/dev/null
modprobe -q dm-raid 2>/dev/null

if [ -x /sbin/dmraid ]; then
 echo "Activating dmraid arrays..."
 /sbin/dmraid -ay
fi

if [ -x /sbin/kpartx ]; then
 echo "Creating partition mappings..."
 for dev in /dev/mapper/isw_*; do
 [ -b "$dev" ] && /sbin/kpartx -av "$dev"
 done
fi
SCRIPT

chmod +x /etc/initramfs-tools/scripts/local-top/dmraid
```

#### D. 修复 GRUB 和 fstab

```bash
# 添加 rootdelay
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="\(.*\)"/GRUB_CMDLINE_LINUX_DEFAULT="\1 rootdelay=10"/' /etc/default/grub

# 确认 fstab 使用 /dev/mapper/ 路径（非 /dev/sdX）
cat /etc/fstab
# 正确格式示例：
# /dev/mapper/isw_bcehdedef_Volume0p2 / ext4 errors=remount-ro 0 1
# /dev/mapper/isw_bcehdedef_Volume0p1 /boot/efi vfat umask=0077 0 1
```

#### E. 重建 initramfs 和 GRUB

```bash
# 重建所有内核的 initramfs
update-initramfs -u -k all

# 更新 GRUB 配置
update-grub

# 重新安装 GRUB 到 EFI（UEFI 系统）
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu

# 退出 chroot
exit
```

### 2.5 卸载并重启

```bash
# 清理挂载（顺序严格）
sudo umount /mnt/boot/efi
sudo umount /mnt/dev
sudo umount /mnt/proc
sudo umount /mnt/sys
sudo umount /mnt

# 重启，拔掉 Live USB
sudo reboot
```

---

## 三、验证清单

### 3.1 启动前验证（chroot 环境内）

```bash
# 验证 dmraid + kpartx 已打入 initramfs
lsinitramfs /boot/initrd.img-$(uname -r) | grep -E "dmraid|kpartx|dm-mod"

# 应至少看到：
# scripts/local-top/dmraid
# usr/sbin/dmraid
# usr/sbin/kpartx
# usr/lib/x86_64-linux-gnu/libdmraid.so.1.0.0.rc16

# 确认 rootdelay 已设置
grep CMDLINE /etc/default/grub
# GRUB_CMDLINE_LINUX_DEFAULT="quiet splash rootdelay=10"

# 确认 fstab 正确
cat /etc/fstab
```

### 3.2 启动后验证（正常进入系统后）

```bash
# 1. 确认 dmraid 状态
sudo dmraid -s
sudo dmraid -r

# 2. 确认分区来自 Mapper 设备
mount | grep mapper
lsblk

# 3. 固化 UEFI 引导项
sudo efibootmgr -v
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
```

---

## 四、故障子场景处理

### 场景 A：重启后仍然进入 BusyBox

在 BusyBox 命令行中执行：

```bash
# 手动加载并激活
modprobe dm-mod
modprobe dm-raid
dmraid -ay
kpartx -av /dev/mapper/isw_bcehdedef_Volume0

# 确认设备出现
ls /dev/mapper/

# 继续启动
exit
```

**如果 exit 后正常进入系统**，说明 initramfs 脚本**执行时机**有问题。修复方法：

```bash
# 将脚本从 local-top 移到 local-premount（更早执行）
sudo mv /etc/initramfs-tools/scripts/local-top/dmraid \
 /etc/initramfs-tools/scripts/local-premount/dmraid

# 增加等待逻辑
sudo sed -i '/\/sbin\/dmraid -ay/a\ sleep 3' \
 /etc/initramfs-tools/scripts/local-premount/dmraid

# 重建
sudo update-initramfs -u -k all
```

### 场景 B：进入 GRUB Rescue 模式（`grub>` 提示符）

```bash
# 列出所有磁盘和分区
ls
ls (hd0,gpt2)/
ls (hd1,gpt2)/

# 找到含 /boot 的分区后手动引导
set root=(hd0,gpt2)
linux /boot/vmlinuz-6.8.0-40-generic root=/dev/mapper/isw_bcehdedef_Volume0p2 rootdelay=15
initrd /boot/initrd.img-6.8.0-40-generic
boot

# 进入系统后立即重装 GRUB：
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
sudo update-grub
```

### 场景 C：内核更新后又出现同样问题

原因：内核更新时 initramfs 重建未包含 dmraid/kpartx。安装监控 hook：

```bash
sudo cat > /etc/kernel/postinst.d/zz-check-dmraid << 'EOF'
#!/bin/sh
set -e
echo "=== Checking dmraid in initramfs for kernel $1 ==="
if lsinitramfs /boot/initrd.img-$1 2>/dev/null | grep -q "dmraid"; then
 echo "dmraid: OK"
else
 echo "dmraid: MISSING — forcing rebuild"
 update-initramfs -u -k $1
fi
EOF

sudo chmod +x /etc/kernel/postinst.d/zz-check-dmraid
```

---

## 五、长期运维建议

### 5.1 内核更新后检查清单

```bash
# 每次 apt upgrade 升级内核后执行：
sudo update-initramfs -u -k all
sudo lsinitramfs /boot/initrd.img-$(uname -r) | grep -E "dmraid|kpartx"
sudo update-grub
```

### 5.2 备份关键配置

```bash
# 备份到安全位置
sudo cp /etc/initramfs-tools/modules /root/backup/
sudo cp /etc/initramfs-tools/scripts/local-top/dmraid /root/backup/
sudo cp /etc/default/grub /root/backup/
sudo cp /etc/fstab /root/backup/
sudo dmraid -s > /root/backup/dmraid-info.txt
```

### 5.3 RAID 健康监控

```bash
# 定期检查 RAID 成员状态
sudo dmraid -r

# 检查磁盘 SMART 信息
sudo smartctl -a /dev/sda
sudo smartctl -a /dev/sdb
sudo smartctl -a /dev/sdc

# 检查磁盘 I/O 错误
dmesg | grep -iE "error|fail|ata|sd[a-c]"
```

---

## 六、关键命令速查表

|操作|命令|
| ---------------------| ------|
|扫描 RAID 元数据|`dmraid -r`|
|查看 RAID 集|`dmraid -s`|
|激活 RAID|`dmraid -ay`|
|创建分区映射|`kpartx -av /dev/mapper/isw_*`|
|查看 Mapper 设备|`ls -la /dev/mapper/`|
|验证 initramfs 内容|`lsinitramfs /boot/initrd.img-* \| grep dmraid`|
|重建 initramfs|`update-initramfs -u -k all`|
|更新 GRUB|`update-grub`|
|重装 GRUB EFI|`grub-install --target=x86_64-efi --efi-directory=/boot/efi`|
|查看块设备树|`lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT`|

---

## 七、技术背景补充

### dmraid vs mdadm

||dmraid|mdadm|
| ------------| --------------------------------| ---------------------|
|适用场景|BIOS/Firmware RAID（FakeRAID）|Linux 原生软件 RAID|
|元数据格式|isw（Intel）、ddf、nvidia 等|Linux md superblock|
|RAID 发现|读取磁盘尾部元数据|扫描 superblock|
|设备路径|`/dev/mapper/isw_*`|`/dev/md*`|
|跨平台兼容|可与 Windows 双启动共享 RAID|仅 Linux|

### VROC 的两种模式

|模式|说明|Linux 路径|
| ------------------------| -----------------------------------| ------------------|
|VMD + NVMe RAID|VMD 直接暴露 RAID 卷为 /dev/nvme*|需要 `vmd` 内核模块|
|VMD + SATA RAID (imsm)|VMD 暴露物理盘，需 dmraid 组装|需要 `dmraid` + `kpartx`|

本文方案针对第二种模式。
