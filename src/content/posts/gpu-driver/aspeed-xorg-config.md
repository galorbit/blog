---
title: "配置 ASPEED 显卡 Xorg 显示驱动"
published: 2026-08-04
description: "记录在 Linux 系统中通过修改 xorg.conf 配置文件，为 ASPEED 显卡配置 modesetting 驱动及显示参数的方法。"
category: "GPU与驱动"
tags:
  - "ASPEED"
  - "Xorg"
  - "modesetting"
  - "显卡驱动"
  - "Linux"
---

# 配置 ASPEED 显卡 Xorg 显示驱动

## 编辑文件

`sudo nano /etc/X11/xorg.conf.d/10-aspeed.conf`

```conf
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
