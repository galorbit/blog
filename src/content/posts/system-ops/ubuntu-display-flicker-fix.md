---
title: "Ubuntu服务器桌面版安装及登录后花屏修复指南"
published: 2026-08-04
description: "记录服务器安装Ubuntu桌面版时遇到的花屏或显示异常问题，提供安装前、安装后及登录界面的详细修复步骤。"
category: "系统运维"
tags:
  - "Ubuntu"
  - "GRUB"
  - "nomodeset"
  - "Wayland"
  - "花屏修复"
---

# Ubuntu服务器桌面版安装及登录后花屏修复指南

服务器安装Ubuntu桌面版时会有各种花屏或者显示问题，记录一下解决方法。

## 安装系统时花屏

在选择安装页面的时候选中 `install Ubuntu`，不要点击，按 `e` 进入编辑页面，在 `quiet splash` 后面删除 `---`，添加 `nomodeset`。

按 `Ctrl+X` 即可继续安装。

## 安装后进入系统花屏

进入Ubuntu系统前按 `Esc` 键或者 `Shift` 进入GRUB引导页面，BIOS引导按 `Shift` 键，UEFI引导按 `Esc` 键（进入系统的一瞬间按一下就行，多按会进入grub命令行）。

安装了多内核的可能不需要手动按，开机自动会到GRUB引导界面，根据自己情况选择。

选择 `Advanced options for Ubuntu`，进入后选中当前使用的系统内核（一般是最上面一个）按下 `e` 键进入编辑界面，在 `quiet splash $vt_handoff` 这里加入 `nomodeset` 变成 `quiet splash nomodeset $vt_handoff`，然后按 `Ctrl+X` 继续引导进入系统。

**注：** 花屏时候输入密码是可以直接进入系统的，进入系统后就正常了。所以可以省略第一步直接进入系统执行第二步就行。

进入系统后修改GRUB文件，不然每一次开机都需要编辑grub选项。

编辑grub文件：
`sudo vim /etc/default/grub`

将
`GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"`
改为
`GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nomodeset"`

如果提示没有vim这个命令，就是没有安装VIM编辑器，请安装vim或者使用nano、vi或者图形界面的文本编辑器都可以，只要你能编辑这个文件就行。

更新grub：
`sudo update-grub`

## Ubuntu用户登录界面花屏解决方法

禁用掉Wayland图形界面。

编辑配置文件：
`sudo vim /etc/gdm3/custom.conf`

将
`#WaylandEnable = false`
改成
`WaylandEnable = false`

保存重启后即可正常。
