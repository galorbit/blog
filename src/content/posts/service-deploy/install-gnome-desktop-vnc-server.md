---
title: "安装 GNOME 桌面与 VNC 远程服务配置指南"
published: 2026-08-04
description: "详细记录在 Linux 系统中安装 GNOME 桌面环境及配置 VNC 远程服务的完整步骤，涵盖 root 与普通用户的服务模板、防火墙放行及服务启停操作。"
category: "服务部署"
tags:
  - "GNOME桌面"
  - "VNC远程桌面"
  - "systemd服务"
  - "firewall-cmd"
  - "Linux运维"
---

# 安装 GNOME 桌面与 VNC 远程服务配置指南

如果有安装桌面可以跳过下面安装桌面服务的步骤。

## 安装 GNOME Desktop 桌面服务

`yum groupinstall "GNOME Desktop" -y`

安装完成后建议重启。

查看系统运行模式

`systemctl get-default`

切换到桌面运行模式

`systemctl set-default graphical.target`

进入图形界面

`init 5`

将图形模式设置为默认启动模式

`ln -sf /lib/systemd/system/graphical.target /etc/systemd/system/default.target`

以上已安装好桌面并设置为默认启动模式，可以重启测试一下是否默认进入桌面。

## 安装 VNC

`yum install tigervnc-server -y`

### 将 VNC 用户设置为服务

#### 登录后的用户为 root

复制一个服务设置的模板。

`cp /lib/systemd/system/vncserver@.service /etc/systemd/system/vncserver@:1.service`

编辑这个服务的配置

`vim /etc/systemd/system/vncserver@:1.service`

此配置为 root 用户，每个用户都需要一个单独的服务配置。需要几个，就复制几个。

```bash
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target

[Service]
Type=forking

ExecStartPre=-/usr/bin/vncserver -kill %i
ExecStart=/sbin/runuser -l root -c "/usr/bin/vncserver %i"
PIDFile=/root/.vnc/%H%i.pid
ExecStop=-/usr/bin/vncserver -kill %i
WantedBy=multi-user.target
```

以上配置文件 VNC 登录后为 root 用户的配置文件。

#### 登录后的用户为普通用户

重新复制一个配置模板。

`cp /lib/systemd/system/vncserver@.service /etc/systemd/system/vncserver@:2.service`

服务名为 vncserver@:2.service，后续添加以此类推。

编辑此配置文件。

`vim /etc/systemd/system/vncserver@:2.service`

```bash
[Unit]

Description=Remote desktop service (VNC)
After=syslog.target network.target

[Service]

Type=simple
ExecStartPre=-/usr/bin/vncserver -kill %i
#这里的<user>为你系统的非管理员用户，如greetingsyi,不带括号。
ExecStart=/sbin/runuser -l <user> -c "/usr/bin/vncserver %i"
#这里的<user>为你系统的非管理员用户，如greetingsyi,不带括号。PIDFile=/home/<user>/.vnc/%H%i.pid
ExecStop=-/usr/bin/vncserver -kill %i
[Install]

WantedBy=multi-user.target
```

以上配置为 VNC 登录后为普通用户的模板。

### 重载 systemd 配置

`systemctl daemon-reload`

每次添加配置后都需要使用上面这个命令。

### 设置 VNC 密码

`vncpasswd`

重复输入两次密码，提示 Would you like to enter a view-only password (y/n)?，默认回车就行。

备注：此处设置的 VNC 密码为你当前登录用户的 VNC 密码，如果是在 root 用户下设置密码，则此密码为 root 用户的 VNC 密码。其他普通用户的 VNC 密码，切换到其他用户下设置即可。

### 防火墙放行服务与端口

vncserver@:1.service 的端口为 5901，@：2 的为 5902，以此类推，创建了多少个用户，就放行多少端口。

`firewall-cmd --zone=public --add-port=5901/tcp --permanent`

放行 vnc 服务。

`firewall-cmd --add-service vnc-server`

重新载入防火墙配置生效。

`firewall-cmd --reload`

查看端口是否放行成功。

`firewall-cmd --list-port`

### 启动 VNC 服务

开机自启服务。

`systemctl enable vncserver@:1.service`

启动

`systemctl start vncserver@:1.service`

关闭

`systemctl stop vncserver@:1.service`

到这里服务端就配置完成了，在 Windows 下可以使用 VNC-Viewer 来连接。

备注：在启动普通用户服务时，如上面配置的 vncserver@:2.service，使用 `systemctl status vncserver@:2.service` 命令查看服务状态时，可能会报错，显示未启动，不过不影响正常使用。
