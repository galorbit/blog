---
title: Ubuntu禁用挂起和休眠
image: "api"
published: 2026-08-22
description: 修改/etc/systemd/sleep.conf配置文件,禁用Ubuntu的挂起、休眠和混合睡眠等电源状态。
category: 环境配置
tags:
  - Ubuntu
  - 电源管理
  - 休眠
slug: ubuntu-disable-suspend-hibernate
---

# Ubuntu禁用挂起和休眠

挂起休眠和其他睡眠功能由 systemd管理，适用于所有启用了systemd的发行版

编辑配置文件`nano /etc/systemd/sleep.conf`，将以下行取消注释，值修改为no。

```shell
AllowSuspend=no
AllowHibernation=no
AllowSuspendThenHibernate=no
AllowHybridSleep=no
```

`sudo systemctl status suspend.target suspend.target hibernate.target hybrid-sleep.target`查看状态

![状态](https://pic.byt3.ro/pic/suspend.jpg)

要重新启用休眠，将这4行重新添加注释。

参考链接：

- [如何在 Ubuntu 24.04或22.04 中彻底禁用挂起/休眠功能](https://www.ufans.top/index.php/archives/1105/)
