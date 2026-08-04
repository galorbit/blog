---
title: "X服务无法启动时重新生成xorg.conf配置指南"
published: 2026-08-04
description: "解决X服务无法启动问题，通过Xorg命令重新生成配置文件并修改关键参数以恢复图形环境。"
category: "系统运维"
tags:
  - "Xorg"
  - "xorg.conf"
  - "Linux"
  - "图形界面"
  - "驱动配置"
---

# X服务无法启动时重新生成xorg.conf配置指南

`/etc/X11/` 路径下缺少 `xorg.conf` 配置文件会导致 X 服务无法启动，此时需要重新生成配置文件。

执行 `Xorg -configure` 会在当前用户目录下自动生成 `xorg.conf.new` 文件。

执行 `mv ~/xorg.conf.new /etc/X11/xorg.conf` 将文件移动到需要的目录并改名。

此时可以执行 `startx` 看能否启动服务，如果无法启动再做下面修改。

备份源文件 `cp /etc/X11/xorg.conf /etc/X11/xorg.conf.bak`。

修改配置文件 `/etc/X11/xorg.conf`。

需要修改的有两个地方：

1. 将 `Load "glx"` 替换成 `Disable "glx"`，并在此行尾部增加文本 `Disable "glamoregl"`。
2. 将 `Driver "modesetting"` 替换成文本 `Driver "fbdev"`。

修改完后保存文件 `/etc/X11/xorg.conf`。
