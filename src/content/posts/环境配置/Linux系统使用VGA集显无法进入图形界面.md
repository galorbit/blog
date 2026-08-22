---
title: Linux系统使用VGA集显无法进入图形界面
published: 2026-08-22
description: 解决Linux装NVIDIA显卡后VGA集显无法进图形界面,grub加nomodeset或注释nvidia配置即可输出。
category: 环境配置
tags:
  - 显卡
  - 图形界面
  - GRUB
slug: linux-vga-integrated-graphics-fix
---

# Linux系统使用VGA集显无法进入图形界面

服务器无法使用集显输出画面的一些解决方法。

## 安装系统后开机无法进入图形界面

超聚变机器安装NVIDIA显卡，安装Ubuntu后集显无法进入图形界面，需要添加`nomodeset`参数使开机不加载驱动。

`nano /etc/default/grub`

在`quiet splash`后添加`nomodeset`

`update-grub`更新grub，重启后就可以使用集显进入图形界面了。

## 安装NVIDA显卡驱动后集显无法进入图形界面

安装显卡驱动后开机默认会加载NVIDA驱动模块，ubuntu默认使用NVIDA模块输出，导致集显无法输出。

如果使用独显输出画面就不需要修改，使用集显输出就需要注释掉开机加载NVIDA模块的配置文件。

`nano /usr/share/X11/xorg.conf.d/*nvidia.conf`

注释掉所有行让其不生效，重启后就可以使用集显输出了。
