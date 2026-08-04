---
title: "QNAP磁盘休眠与SWAP配置指南"
published: 2026-08-04
description: "介绍QNAP NAS环境下配置硬盘自动休眠与SWAP内存交换阈值的操作步骤及注意事项。"
category: "NAS"
tags:
  - "QNAP"
  - "硬盘休眠"
  - "SWAP"
  - "系统运维"
  - "命令行"
---

# QNAP磁盘休眠与SWAP配置指南

## 硬盘休眠

首先开启磁盘休眠。

### 创建自动断开和重连脚本

```bash
touch /share/SSD/sh/disconnect_internal_raid.sh # 断开
touch /share/SSD/sh/rebuild_internal_raid.sh # 重连
chmod +x /share/SSD/sh/disconnect_internal_raid.sh
chmod +x /share/SSD/sh/rebuild_internal_raid.sh
```

填充内容：

`` `vi /share/SSD/sh/disconnect_internal_raid.sh` ``

```bash
echo "Disconnecting md9"
mdadm /dev/md9 --fail /dev/sda1
mdadm /dev/md9 --fail /dev/sdb1

echo "Disconnecting md13"
mdadm /dev/md13 --fail /dev/sda4
mdadm /dev/md13 --fail /dev/sdb4
```

`` `vi /share/SSD/sh/rebuild_internal_raid.sh` ``

```bash
echo "Re-adding md9"
mdadm /dev/md9 --re-add /dev/sda1
mdadm /dev/md9 --re-add /dev/sdb1

echo "Re-adding md13"
mdadm /dev/md13 --re-add /dev/sda4
mdadm /dev/md13 --re-add /dev/sdb4
```

### 添加定时任务

`` `echo "00 01 * * * /share/SSD/sh/rebuild_internal_raid.sh" >> /etc/config/crontab`  # 每天01:00执行重新连接
`` `echo "15 01 * * * /share/SSD/sh/disconnect_internal_raid.sh" >> /etc/config/crontab`  # 每天01:15自动断开

### 重启crontab

`` `crontab /etc/config/crontab && /etc/init.d/crond.sh restart`  # 添加任务并重启使其生效

### NAS重启后自动运行

```bash
mount $(/sbin/hal_app --get_boot_pd port_id=0)6 /tmp/config   # 型号不同，这条命令也不同
touch /tmp/config/autorun.sh
chmod +x /tmp/config/autorun.sh
# 在autorun.sh下添加自动断开脚本路径
vi /tmp/config/autorun.sh
/share/SSD/sh/disconnect_internal_raid.sh
umount /tmp/config
```

### 检测硬盘状态

`` `hdparm -C /dev/sda` ``

- `idle/active`：硬盘没有休眠
- `standby`：说明硬盘处于休眠状态

## SWAP

```bash
# 输出数值为内存占用百分比多少后使用SWAP的阈值
cat /proc/sys/vm/swappiness
25
# 修改阈值为5%
echo 5 > /proc/sys/vm/swappiness
```
