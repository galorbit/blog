---
title: 服务器添加独显后VGA无法启动桌面解决方案
image: "api"
published: 2026-08-22
description: 服务器加装独显后VGA无法启动桌面,通过xorg.conf.d配置ASPEED集显使用modesetting驱动。
category: 环境配置
tags:
  - 显卡
  - Xorg
  - 桌面显示
slug: server-vga-gui-fail-after-gpu-fix
---

# 服务器添加独显后VGA无法启动桌面解决方案

## 编辑文件

`sudo nano /etc/X11/xorg.conf.d/10-aspeed.conf`

```text
Section "Device"
    Identifier "DeviceASPEED"
    Driver "modesetting"
    BusID "PCI:4:0:0"
EndSection

Section "Screen"
    Identifier "ScreenASPEED"
    Device "DeviceASPEED"
    DefaultDepth 24
EndSection

Section "ServerFlags"
    Option "AllowIndirectGL" "on"
    Option "AIGLX" "off"
EndSection

```
