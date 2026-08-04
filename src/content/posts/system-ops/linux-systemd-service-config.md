---
title: "Linux手动创建系统服务配置文件详解"
published: 2026-08-04
description: "介绍Linux手动创建systemd服务配置文件的目录规范、核心参数说明及openGauss完整配置示例。"
category: "系统运维"
tags:
  - "systemd"
  - "Linux服务"
  - "配置文件"
  - "openGauss"
  - "系统运维"
---

# Linux手动创建系统服务配置文件详解

参考链接：[https://uixor.com/article/1479051730476994561](https://uixor.com/article/1479051730476994561)

## 单元文件存放目录

脚本文件一般存在于如下目录中：

- `/lib/systemd/system` ：本地配置的系统单元，开机不需要登录即可运行，优先级高；
- `/run/systemd/system` ：运行时配置的系统单元，运行时的配置服务，优先级中；
- `/usr/lib/systemd/system` ：软件包安装的系统单元，优先级低。

## 常用参数说明

- `[Unit]` ：服务的说明部分
- `Description` ：描述服务
- `After` ：指定服务启动的先后顺序依赖
- `[Service]` ：服务运行参数的设置部分
  > 注意：`ExecStart` 、`ExecReload` 、`ExecStop` 等命令全部要求使用绝对路径
- `Type=forking` ：表示以后台守护进程形式运行
- `ExecStart` ：服务的具体启动命令
- `ExecReload` ：服务重启命令
- `ExecStop` ：服务停止命令
- `PrivateTmp=True` ：表示给服务分配独立的临时空间
- `[Install]` ：服务安装的相关设置，可设置为多用户模式

## 配置示例

下面为一个系统服务的示例：

```ini
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
