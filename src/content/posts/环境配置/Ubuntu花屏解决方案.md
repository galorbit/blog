---
title: Ubuntu花屏解决方案
published: 2026-08-22
description: 解决Ubuntu桌面版安装、进入系统和登录界面花屏,靠grub加nomodeset参数并禁用Wayland。
category: 环境配置
tags:
  - Ubuntu
  - 显卡
  - 花屏
slug: ubuntu-screen-distortion-fix
---

# Ubuntu花屏解决方案

服务器安装Ubuntu桌面版时会有各种花屏或者显示问题，记录一下解决方法。

## 安装系统时花屏

在选择安装页面的时候选中`install Ubuntu`，不要点击，按e进入编辑页面，在`quiet splash`后面删除“---”，添加`nomodeset`。

按CTRL+X即可继续安装。

## 安装后进入系统花屏

进入Ubuntu系统前按Esc键或者Shift进入GRUB引导页面，BIOS引导按Shift键，UEFI引导按ESC键（进入系统的一瞬间按一下就行，多按会进入grub命令行）。

安装了多内核的可能不需要手动按，开机自动会到GRUB引导界面，根据自己情况选择。

选择`advansced options for ubuntu` , 进入后选中当前使用的系统内核（一般是最上面一个）按下e键进入编辑界面，在`quiet splash $vt_handoff`这里加入`nomodeset`变成`quiet splash nomodeset $vt_handoff`，然后按CTRL+X继续引导进入系统。

**注：花屏时候输入密码是可以直接进入系统的，进入系统后就正常了。所以可以省略第一步直接进入系统执行第二步就行。**

进入系统后修改GRUB文件，不然每一次开机都需要编辑grub选项。

编辑grub文件：

`sudo vim /etc/default/grub`

将

`GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"`

改为

`GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nomodeset"`

如果提示没有vim这个命令，就是没有安装VIM编辑器，请安装vim或者使用nano、vi或者图形界面的文本编辑器都可以，只要你能编辑这个文件就行。

更新grub

`sudo update-grub`

## Ubuntu用户登陆界面花屏解决方法

禁用掉wayland图形界面

编辑

`vim /etc/gdm3/custom.conf`

将 `#WaylandEnable = false`

改成 `WaylandEnable = false`

保存重启后即可正常。
