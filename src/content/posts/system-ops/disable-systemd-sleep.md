---
title: "在 Linux 系统中通过 systemd 禁用挂起与休眠功能"
published: 2026-08-04
description: "介绍如何通过修改 systemd 配置文件彻底禁用 Linux 系统的挂起、休眠及混合睡眠功能，并附带状态检查命令。"
category: "系统运维"
tags:
  - "systemd"
  - "挂起休眠"
  - "Linux配置"
  - "电源管理"
---

# 在 Linux 系统中通过 systemd 禁用挂起与休眠功能

挂起、休眠和其他睡眠功能由 systemd 管理，适用于所有启用了 systemd 的发行版。

编辑配置文件 ` nano /etc/systemd/sleep.conf `，将以下行取消注释，值修改为no。

```ini
AllowSuspend=no
AllowHibernation=no
AllowSuspendThenHibernate=no
AllowHybridSleep=no
```

` sudo systemctl status suspend.target hibernate.target hybrid-sleep.target ` 查看状态

![状态](https://pic.byt3.ro/pic/suspend.jpg)

要重新启用休眠，将这 4 行重新添加注释。

参考链接：

- [如何在 Ubuntu 24.04或22.04 中彻底禁用挂起/休眠功能](https://www.ufans.top/index.php/archives/1105/)
