---
title: "解决Ubuntu服务器安装N卡后集显无法输出画面的方法"
published: 2026-08-04
description: "解决Ubuntu服务器安装NVIDIA显卡后集显无法输出画面的问题，含GRUB参数调整与Xorg配置注释方法。"
category: "GPU与驱动"
tags:
  - "集成显卡"
  - "NVIDIA驱动"
  - "Ubuntu"
  - "图形界面"
  - "GRUB配置"
---

# 解决Ubuntu服务器安装N卡后集显无法输出画面的方法

## 安装系统后开机无法进入图形界面

超聚变机器安装 NVIDIA 显卡，安装 Ubuntu 后集显无法进入图形界面，需要添加 `nomodeset` 参数使开机不加载驱动。

编辑 GRUB 配置文件：`nano /etc/default/grub`

在 `quiet splash` 后添加 `nomodeset`。

执行 `update-grub` 更新 GRUB，重启后即可使用集显进入图形界面。

## 安装 NVIDIA 显卡驱动后集显无法进入图形界面

安装显卡驱动后开机默认会加载 NVIDIA 驱动模块，Ubuntu 默认使用 NVIDIA 模块输出，导致集显无法输出。

如果使用独显输出画面就不需要修改；若需使用集显输出，则需要注释掉开机加载 NVIDIA 模块的配置文件。

编辑 Xorg 配置目录下的 NVIDIA 配置文件：`nano /usr/share/X11/xorg.conf.d/*nvidia.conf`

注释掉文件内的所有行使其不生效，重启后即可恢复集显输出。
