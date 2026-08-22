---
title: systemd配置系统服务
published: 2026-08-22
description: 手把手配置systemd系统服务,详解[Unit][Service][Install]各参数含义,并附上openGauss服务示例。
category: 环境配置
tags:
  - Systemd
  - 系统服务
slug: systemd-service-config
---

# systemd配置系统服务

Linux手动创建系统服务配置文件的介绍

## 配置软件为系统服务参数详解

参考链接：[https://uixor.com/article/1479051730476994561](https://uixor.com/article/1479051730476994561)

脚本文件一般存在于如下目录中

`/lib/systemd/system`：本地配置的系统单元，开机不需要登录即可运行，优先级高；

`/run/systemd/system`：运行时配置的系统单元，运行时的配置服务，优先级中；

`/usr/lib/systemd/system`：软件包安装的系统单元，优先级低。

常用的配置

`[Unit]` 服务的说明

`Description` 描述服务

`After` 描述服务类别

`[Service]` 服务运行参数的设置

注意：service的启动、重启、停止命令全部要求使用绝对路径

`Type=forking` 是后台运行的形式

`ExecStart` 为服务的具体运行命令

`ExecReload` 为重启命令

`ExecStop` 为停止命令

`PrivateTmp=True` 表示给服务分配独立的临时空间

`[Install]` 服务安装的相关设置，可设置为多用户

下面为一个系统服务的示例

```shell
[Unit]

Description=openGauss
Documentation=openGauss
After=syslog.target
After=network.target

[Service]

Type=forking
User=opengauss
Group=opengauss
Environment=PGDATA=/usr/local/opengauss/data
Environment=GAUSSHOME=/usr/local/opengauss/install
Environment=LD_LIBRARY_PATH=/usr/local/opengauss/install/lib
ExecStart=/usr/local/opengauss/install/bin/gs_ctl start
ExecReload=/usr/local/opengauss/install/bin/gs_ctl restart
ExecStop=/usr/local/opengauss/install/bin/gs_ctl stop
KillMode=mixed
KillSignal=SIGINT
TimeoutSec=0

[Install]

WantedBy=multi-user.target
```
