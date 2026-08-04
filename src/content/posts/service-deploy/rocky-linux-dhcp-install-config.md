---
title: "Rocky Linux DHCP 服务器安装与配置指南"
published: 2026-08-04
description: "记录在 Rocky Linux 系统上安装、配置及启动 DHCP 服务器的完整步骤与参数说明。"
category: "服务部署"
tags:
  - "DHCP"
  - "Rocky Linux"
  - "网络服务"
  - "服务器配置"
---

# Rocky Linux DHCP 服务器安装与配置指南

关闭局域网内其他 DHCP 服务器，避免 DHCP 服务器互相冲突。

## 安装 DHCP Server

rockylinux

```bash
dnf install dhcp-server
```

设置开机自启

```bash
systemctl enable dhcpd
```

## 编辑配置文件

配置文件目录 `/etc/dhcp/dhcpd.conf`，可用 `dhcpd` 命令查看相关目录。

编辑配置文件

```text
DHCP Server Configuration file.
see /usr/share/doc/dhcp-server/dhcpd.conf.example
see dhcpd.conf(5) man page
subnet 192.168.48.0 netmask 255.255.255.0 {
                range 192.168.48.5 192.168.48.79;
                #地址池的范围
                option domain-name-servers 192.168.48.1;
                #为客户端指明DNS服务器IP地址
                option domain-name "local";
                #为客户端指明DNS名字。
                option routers 192.168.48.1;
                #路由器ip，可以写网关ip
                option broadcast-address 192.168.48.255;
                #广播地址
                default-lease-time 1800;
                #指定缺省租赁时间的长度，单位是秒。
                max-lease-time 3600;
                #指定最大租赁时间长度，单位是秒。
                filename "iventoy_loader_16000";
                #开始启动文件的名称。
                next-server 192.168.48.220;
                #设置服务器从引导文件中装入主机名。
}
```

重启服务后会自动应用配置文件

```bash
systemctl restart dhcpd
```
