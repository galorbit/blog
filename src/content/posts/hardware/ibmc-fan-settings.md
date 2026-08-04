---
title: "服务器IBMC手动设置风扇转速与断电恢复配置"
published: 2026-08-04
description: "记录通过SSH登录IBMC控制台，使用ipmcset命令手动设置风扇模式与转速的方法，并提示断电后需重新配置。"
category: "硬件与BMC"
tags:
  - "IBMC"
  - "风扇控制"
  - "服务器硬件"
  - "命令行配置"
---

# 服务器IBMC手动设置风扇转速与断电恢复配置

> 手动设置风扇服务器断电后会失效，需要再次设置

SSH 连接 IBMC：
`ssh Administrator@192.168.2.100`

登录后执行以下命令：

```bash
ipmcset -d fanmode -v 1 0
# 设置风扇模式为手动并且不超时
ipmcset -d fanlevel -v 80
# 设置风扇转速为80%
```
