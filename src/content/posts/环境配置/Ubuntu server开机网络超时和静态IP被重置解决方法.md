---
title: Ubuntu server开机网络超时和静态IP被重置解决方法
image: "api"
published: 2026-08-22
description: 解决Ubuntu server开机卡在网络等待超时,以及静态IP重启后被cloud-init覆盖重置的问题。
category: 环境配置
tags:
  - Ubuntu
  - 网络配置
  - Netplan
slug: ubuntu-server-network-timeout-static-ip-fix
---

# Ubuntu server开机网络超时和静态IP被重置解决方法

> ubuntu server在配置静态重启后会被重置，并且开机会卡在"A start job is running for wait for network to be Configured"这里很久。

## 开机卡很久

开机卡2分钟是因为有网口没有配置网络单并没有禁用，系统一直在尝试获取地址导致超时。

解决方法：

编辑`systemd-networkd-wait-online.service`文件

`nano /etc/systemd/system/network-online.target.wants/systemd-networkd-wait-online.service`

添加以下内容：

```text
[Service]
TimeoutStartSec=2sec
```

## 配置静态ip重启后被重置

原因：网络配置文件`/etc/netplan/xxxx.yaml`的配置会在每次系统重启时被cloud云文件配置的内容所进行覆盖，可以通过禁用 cloud-init 的网络配置来防止这种情况发生。

解决方法：

编辑文件 `nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`

添加内容到最后一行 `network: {config: disabled}`
