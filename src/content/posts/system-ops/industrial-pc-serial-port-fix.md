---
title: "工控机多串口仅显示4个的解决方法"
published: 2026-08-04
description: "针对工控机Linux系统下多串口仅显示4个的问题，提供内核参数、模块配置及BIOS检查等完整排查与修复方案。"
category: "系统运维"
tags:
  - "工控机"
  - "串口驱动"
  - "Linux内核"
  - "GRUB配置"
  - "硬件枚举"
---

# 工控机多串口仅显示4个的解决方法

## 问题现象

工控机（通常搭载 Intel/AMD 平台，主板集成多个串口芯片）安装了 Linux 系统后，`/dev/ttyS*` 设备仅出现 `ttyS0` ~ `ttyS3`（共 4 个），而实际硬件有 6/8/10 个串口。

**根本原因**：Linux 内核 8250 串口驱动默认的 `nr_uarts` 值为 4，超出部分不会被枚举。

## 解决方法（Linux 系统）

### 方法一：内核启动参数（推荐）

1. **编辑 GRUB 配置**：

```bash
sudo vi /etc/default/grub
```

找到 `GRUB_CMDLINE_LINUX` 行，在内核参数引号内追加：

```text
8250.nr_uarts=16
```

例如：

```bash
GRUB_CMDLINE_LINUX="quiet splash 8250.nr_uarts=16"
```

> 数值 `16` 可根据实际串口数量调整（如 `8`、`12`、`20`），建议按需设置即可。

2. **更新 GRUB 引导配置**：

   - **Debian / Ubuntu 系列**：

   ```bash
   sudo update-grub
   ```

   - **RHEL / CentOS / Rocky / AlmaLinux 系列**：

   ```bash
   sudo grub2-mkconfig -o /boot/grub2/grub.cfg
   ```

3. **重启验证**：

```bash
sudo reboot
```

重启后检查串口设备：

```bash
dmesg | grep ttyS
ls -l /dev/ttyS*
```

### 方法二：模块参数（无需重启）

如果系统已将 8250 编译为模块，可通过模块参数动态调整：

```bash
# 临时调整（立即生效，重启失效）
echo 16 | sudo tee /sys/module/8250/parameters/nr_uarts
```

如需持久化，创建模块配置文件：

```bash
echo "options 8250 nr_uarts=16" | sudo tee /etc/modprobe.d/8250.conf
```

然后执行 `sudo update-initramfs -u`（Debian/Ubuntu）或 `sudo dracut -f`（RHEL 系列）并重启。

### 方法三：内核编译时固定（自编译内核场景）

如果自行编译内核，在 `.config` 中设置：

```text
CONFIG_SERIAL_8250_NR_UARTS=16
CONFIG_SERIAL_8250_RUNTIME_UARTS=16
```

## 故障排查补充

### 查看当前 nr_uarts 值

```bash
cat /sys/module/8250/parameters/nr_uarts
```

### 查看内核检测到的所有串口

```bash
dmesg | grep -i "serial\|ttyS\|8250"
sudo lspci -v | grep -i serial
```

### 验证串口是否可访问（硬件层面）

```bash
# 安装 setserial
sudo apt install setserial      # Debian/Ubuntu
sudo yum install setserial      # RHEL/CentOS

# 列出串口及 IRQ/I/O 地址
sudo setserial -g /dev/ttyS*
```

## Windows 系统的对应问题

如果工控机安装的是 **Windows** 系统：

1. Windows 对 COM 端口默认限制为 4 个（COM1~COM4）。超出部分需要在 **设备管理器** 中手动更改端口号：
   - 右键串口设备 → **端口设置** → **高级** → **COM 端口号** → 选择未占用的编号（如 COM5~COM256）。
2. 部分工控机需在 BIOS 中开启 **Serial Port** 对应的通道（Enabled）并分配正确资源（IO=3F8/2F8/3E8/2E8, IRQ=4/3/5/6...）。

## BIOS 检查

进入 BIOS 确认以下设置：

- 各串口通道状态是否为 **Enabled**
- 串口模式是否设为 **RS232**（如果实际接的是 RS232 设备）
- I/O 地址与 IRQ 是否存在冲突（手动分配或设为 Auto）

## 参考

- Linux 内核文档：`Documentation/admin-guide/serial-console.rst`
- 8250 驱动源码：`drivers/tty/serial/8250/8250_core.c`
