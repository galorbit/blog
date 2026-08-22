---
title: Linux启动进入grub命令行界面解决方法
published: 2026-08-22
description: 修复卡在grub rescue无法启动的系统:手动引导内核、chroot重装GRUB、修复UEFI启动项。
category: 环境配置
tags:
  - GRUB
  - 启动修复
  - 系统引导
slug: linux-grub-command-line-fix
---

# Linux 系统 GRUB 引导修复

> 适用场景：系统无法启动，卡在 `grub rescue>` 或 `grub>` 提示符。

---

## 一、GRUB Rescue 手动引导进入系统

### 1.1 查看可用分区

```text
grub rescue> ls
```

输出示例：

```text
(hd0) (hd0,msdos1) (hd0,msdos2) (lvm/openeuler-root)
```

### 1.2 定位 /boot 所在分区

寻找包含 `grub2/` 或 `vmlinuz-*` 文件的分区：

```text
grub rescue> ls (hd0,msdos1)/
```

确认看到 `grub2/`、`vmlinuz-*` 等文件。

> **注意**：如果 `/boot` 是独立分区，内核文件在分区根目录，不需要 `/boot/` 前缀。如果不是独立分区，路径为 `/boot/vmlinuz-xxx`。

### 1.3 设置 GRUB 前缀及加载模块

```text
grub rescue> set root=(hd0,msdos1)
grub rescue> set prefix=(hd0,msdos1)/grub2
grub rescue> insmod normal
grub rescue> normal
```

如果根目录位于 **LVM** 卷上，在 `insmod normal` 之前先加载 LVM 模块：

```text
grub rescue> set root=(hd0,msdos1)
grub rescue> set prefix=(hd0,msdos1)/grub2
grub rescue> insmod lvm
grub rescue> insmod normal
grub rescue> normal
```

- 执行 `normal` 后 GRUB 会加载完整菜单，选择对应内核即可启动。
- 如果菜单未正确生成，则继续手动引导步骤。

### 1.4 手动引导内核（跳过 GRUB 菜单）

```text
grub> insmod lvm
grub> set root=(hd0,msdos1)
grub> linux /vmlinuz-<版本号> root=/dev/mapper/openeuler-root
grub> initrd /initramfs-<版本号>.img
grub> boot
```

> - 按 `Tab` 键可补全 vmlinuz 和 initramfs 文件名。
> - `root=` 指定实际根分区，也可用 `root=UUID=<uuid>`（`ls (lvm/openeuler-root)` 查看 UUID）。

### 1.5 进入系统后重装 GRUB

**Legacy BIOS 引导：**

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo grub2-install /dev/sda     # 注意：安装到整块磁盘，不是分区（如 /dev/sda，不是 /dev/sda1）
```

**UEFI 引导（见第三节）。**

---

## 二、通过 Live CD/USB chroot 修复（推荐）

当系统损坏较严重时，chroot 方式更彻底。

### 2.1 启动 Live 系统，挂载分区

```bash
# 挂载根分区
sudo mount /dev/mapper/openeuler-root /mnt

# 挂载 /boot（独立分区则挂载，非独立跳过）
sudo mount /dev/sda1 /mnt/boot

# 挂载 EFI 分区（UEFI 系统）
sudo mount /dev/sda1 /mnt/boot/efi

# 挂载虚拟文件系统
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
```

### 2.2 chroot 进入系统

```bash
sudo chroot /mnt
```

### 2.3 重装 GRUB

**Legacy BIOS：**

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
grub2-install /dev/sda
```

**UEFI：**

```bash
grub2-mkconfig -o /boot/efi/EFI/openEuler/grub.cfg
dnf reinstall grub2-efi-x64 shim-x64 grub2-common
```

> **注意**：发行版不同，EFI 路径有所区别：
>
> - openEuler：`/boot/efi/EFI/openEuler/`
> - CentOS/RHEL：`/boot/efi/EFI/centos/` 或 `/boot/efi/EFI/redhat/`
> - Ubuntu：`/boot/efi/EFI/ubuntu/`

### 2.4 退出 chroot 并重启

```bash
exit
sudo umount -R /mnt
sudo reboot
```

---

## 三、UEFI 引导修复

### 3.1 确认 EFI 分区中的引导文件

```bash
ls /boot/efi/EFI/openeuler/
```

应包含 `grubx64.efi` 和 `shimx64.efi` 文件。

### 3.2 重新生成 EFI GRUB 配置

```bash
sudo grub2-mkconfig -o /boot/efi/EFI/openEuler/grub.cfg
```

### 3.3 修复 NVRAM 启动项

如果引导文件已恢复但 BIOS 启动项中找不到系统：

```bash
sudo efibootmgr -c -d /dev/sda -p 1 -L "openEuler" -l /EFI/openeuler/shimx64.efi
```

参数说明：

|参数|含义|
| ----| ----------------------------|
|`-c`|创建新启动项|
|`-d /dev/sda`|磁盘设备（按实际调整）|
|`-p 1`|EFI 分区号（按实际调整）|
|`-L "openEuler"`|启动菜单显示名称|
|`-l /EFI/openeuler/shimx64.efi`|EFI 引导文件路径（以 `/` 分隔）|

---

## 四、设置本地 DNF/YUM 源（无网络环境重装 GRUB 包）

如果系统无法联网，需挂载 ISO/DVD 作为本地源来安装 `grub2-efi-x64`、`shim-x64` 等包。

### 4.1 挂载 ISO

```bash
sudo mount -o loop /path/to/openEuler-xxx.iso /mnt
```

### 4.2 配置本地 repo

```bash
cat << 'EOF' | sudo tee /etc/yum.repos.d/local.repo
[local]
name=local
baseurl=file:///mnt
enabled=1
gpgcheck=0
EOF
```

### 4.3 重新安装 GRUB EFI 包

```bash
sudo dnf clean all
sudo dnf reinstall grub2-efi-x64 shim-x64 grub2-common
```

---

## 五、常见错误排查

### 5.1 `no such partition`

GRUB 分区编号不正确。用 `ls` 列出所有分区，逐个检查。

### 5.2 `file not found`

内核或 initramfs 路径不对。确认 `/boot` 位置（是否独立分区）和文件名。

### 5.3 `unknown filesystem`

GRUB 模块未加载。如 LVM 需先 `insmod lvm`，btrfs/xfs 同理。

### 5.4 EFI 启动项修复后重启仍进 BIOS

- 检查 BIOS 启动模式是否为 **UEFI**（非 Legacy/CSM）
- 检查安全启动（Secure Boot）是否关闭或已正确配置 `shimx64.efi`
- 用 `efibootmgr -v` 查看当前 NVRAM 启动项确认引导路径是否正确

---

## 六、参考命令速查

```bash
# 查看当前 NVRAM 启动项
sudo efibootmgr -v

# 删除启动项
sudo efibootmgr -b <BootXXXX> -B

# 调整启动顺序
sudo efibootmgr -o <XXXX>,<YYYY>,<ZZZZ>

# 查看内核当前 cmdline
cat /proc/cmdline
```
