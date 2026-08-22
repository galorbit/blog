---
title: Xorg重新生成配置文件
published: 2026-08-22
description: 用Xorg -configure重新生成xorg.conf文件,调整glx与modesetting驱动配置,解决X服务无法启动的问题。
category: 环境配置
tags:
  - Xorg
  - 配置文件
  - 显示
slug: xorg-regenerate-config
---

# Xorg重新生成配置文件

/etc/X11/路径下缺少xorg.conf配置和X服务无法启动需要重新生成配置文件

执行`Xorg -configure`会在当前用户目录下自动生成`xorg.conf.new`文件

执行`mv ~./xorg.conf.new /etc/X11/xorg.conf` 将文件移动到需要的目录并改名。

此时可以执行`startx`看能否启动服务，如果无法启动再做下面修改.

备份源文件`cp /etc/X11/xorg.conf /etc/X11/xorg.conf.bak`

修改配置文件`/etc/X11/xorg.conf`

需要修改的有两个地方:

1. 将`Load "glx"`替换成`Disable "glx"`，并在此行尾部增加文本`Disable "glamoregl"`
2. 将`Driver "modesetting"`替换成文本`Driver "fbdev"`

修改完后保存文件`/etc/X11/xorg.conf`
