---
title: "Ubuntu服务器集显自定义分辨率设置与持久化"
published: 2026-08-04
description: "记录Ubuntu桌面版服务器集显低分辨率问题的临时调整与通过配置文件实现永久生效的方法。"
category: "系统运维"
tags:
  - "Ubuntu"
  - "xrandr"
  - "分辨率设置"
  - "系统配置"
  - "集显"
---

# Ubuntu服务器集显自定义分辨率设置与持久化

很多使用集显的服务器，安装 Ubuntu 桌面版默认分辨率特别低且无法修改。记录一下。

## 使用 xrandr 命令新增显示模式

如果不知道自己显示器的最佳分辨率，可直接输入 `xrandr` 命令，输出结果会显示可用模式。

键入命令 `xrandr`：

```bash
Screen 0: minimum 320 x 200, current 1440 x 900, maximum 1920 x 1080
VGA-1 connected 1440x900+0+0 (normal left inverted right x axis y axis) 0mm x 0mm 1360x768 59.8
1024x768 60.0
800x600 60.3
848x480 60.0
640x480 59.9
```

我的显示器最佳分辨率为 1920x1080，使用 `cvt` 获取显示模式的格式。

执行 `cvt 1920 1080`：

```bash
# 1920x1080 59.96 Hz (CVT 2.07M9) hsync: 67.16 kHz; pclk: 173.00 MHz Modeline "1920x1080_60.00" 173.00 1920 2048 2248 2576 1080 1083 1088 1120 -hsync +vsync
```

将得到的显示格式用 `xrandr` 添加：

```bash
sudo xrandr --newmode "1920x1080" 173.00 1920 2048 2248 2576 1080 1083 1088 1120 -hsync +vsync
sudo xrandr --addmode VGA-1 1920x1080 #注：这里的VGA-1和下一条命令的VGA-1是你的显示设备，可以使用xrandr查看你的显示设备，注意修改。
sudo xrandr --output VGA-1 --mode 1920x1080
```

使用以上命令添加显示模式以后，不出意外的话分辨率会自动修改为你设置的分辨率。如果没有自动修改，在设置里面手动改一下就可以了。

## 永久设置此分辨率

上面添加的分辨率会在重启后失效，重启系统后又无法设置了。要一直使用此分辨率需要修改当前用户的配置文件。

执行 `cd ~` 进入当前用户目录。

运行 `ls -a` 可以看到有个 `.profile` 的隐藏文件。

编辑 `vim ~/.profile`，在文件末尾添加以下刚刚设置的配置：

```bash
cvt 1920 1080
sudo xrandr --newmode "1920x1080" 173.00 1920 2048 2248 2576 1080 1083 1088 1120 -hsync +vsync
sudo xrandr --addmode VGA-1 1920x1080
```

保存，重启系统后就可以在设置里修改为刚才配置的分辨率了。

如果需要登陆其他用户，还是需要重新配置一遍。
