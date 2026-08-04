---
title: "修改IPMI风扇转速阈值解决报错问题"
published: 2026-08-04
description: "介绍如何通过 ipmitool 或 InspurDiagLogCollect 工具修改 IPMI 风扇转速阈值，解决因转速过低导致的持续报错及风扇满速运行问题。"
category: "硬件与BMC"
tags:
  - "IPMI"
  - "风扇阈值"
  - "服务器硬件"
  - "浪潮服务器"
---

# 修改IPMI风扇转速阈值解决报错问题

> 风扇转速低于主板警告范围，会持续报错，并且风扇间歇性满速运行。

## 修改IPMI阈值

Linux 下可以安装 `ipmitool`，Windows 下没有好用的工具，可以使用浪潮的 IPMI 工具 `InspurDiagLogCollect`。

`ipmitool -I lan -U ADMIN -H 192.168.48.20 sensor thresh FAN1 lower 150 225 300`

命令介绍

```bash
ADMIN
# IPMI用户名
192.168.48.20
# IPMI 地址
FAN1
# 要修改阈值的风扇 FANA FANB FAN1 FAN2 FAN3 FAN4
150
# 致命故障阈值 Non Recoverable
225
# 严重故障阈值 Critical
300
# 需要警惕的阈值 Non Critical
```

建议 `Non Critical` 设置为正常最低速度运行值的 -25%（风扇测速误差），`Critical` 设为 -45%，`Non Recoverable` 设为 -60%。例如低负载为 300 时，则可设置为 120、165、225。

使用 `InspurDiagLogCollect` 工具时，命令可以简化为 `sensor thresh FAN1 lower 150 225 300`。
