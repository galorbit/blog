---
title: "解决Ubuntu静态IP重启失效及开机网络等待卡顿"
published: 2026-08-04
description: "解决Ubuntu Server配置静态IP重启失效问题，以及优化开机网络等待超时卡顿现象。"
category: "网络配置"
tags:
  - "Ubuntu"
  - "静态IP"
  - "systemd-networkd"
  - "cloud-init"
  - "网络配置"
---

# 解决Ubuntu静态IP重启失效及开机网络等待卡顿

Ubuntu Server 在配置静态 IP 重启后会被重置，并且开机会卡在 "A start job is running for wait for network to be Configured" 这里很久。

## 开机卡很久

开机卡 2 分钟是因为有网口没有配置网络但并没有禁用，系统一直在尝试获取地址导致超时。

解决方法：

编辑 `systemd-networkd-wait-online.service` 文件：

`nano /etc/systemd/system/network-online.target.wants/systemd-networkd-wait-online.service`

添加以下内容：

```ini
[Service]
TimeoutStartSec=2sec
```

## 配置静态 IP 重启后被重置

原因：网络配置文件 `/etc/netplan/xxxx.yaml` 的配置会在每次系统重启时被 cloud 云文件配置的内容所进行覆盖，可以通过禁用 cloud-init 的网络配置来防止这种情况发生。

解决方法：

编辑文件：

`nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`

添加内容到最后一行：

```text
network: {config: disabled}
```
