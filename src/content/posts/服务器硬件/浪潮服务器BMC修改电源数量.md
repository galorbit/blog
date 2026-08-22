---
title: 浪潮服务器BMC修改电源数量
published: 2026-08-22
description: 浪潮服务器只装一个电源时BMC报错,通过InspurDiagLogCollect工具执行raw命令修改电源数量配置消除告警。
category: 服务器硬件
tags:
  - 浪潮
  - BMC
  - 电源管理
slug: inspur-server-bmc-power-supply-count
---

# 浪潮服务器BMC修改电源数量

浪潮服务器出厂配置双电源,只安装一个电源BMC报错

在浪潮日志收集工具（InspurDiagLogCollect）连接服务器，输入以下命令：

## 改为2电源

`raw 0x3c 0x2a 0x0 0x2`

## 改为1电源

`raw 0x3c 0x2a 0x0 0x1`


