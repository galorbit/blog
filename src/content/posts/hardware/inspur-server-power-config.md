---
title: "浪潮服务器单电源安装触发BMC报错的处理方法"
published: 2026-08-04
description: "记录浪潮服务器单电源安装引发BMC告警的排查步骤，提供使用官方工具修改电源模式的原始指令。"
category: "硬件与BMC"
tags:
  - "浪潮服务器"
  - "BMC"
  - "电源配置"
  - "IPMI"
  - "InspurDiagLogCollect"
---

# 浪潮服务器单电源安装触发BMC报错的处理方法

在浪潮日志收集工具（InspurDiagLogCollect）连接服务器，输入以下命令：

## 改为2电源
```bash
raw 0x3c 0x2a 0x0 0x2
```

## 改为1电源
```bash
raw 0x3c 0x2a 0x0 0x1
```
