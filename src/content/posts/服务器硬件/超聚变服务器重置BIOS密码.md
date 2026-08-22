---
title: 超聚变服务器重置BIOS密码
image: "api"
published: 2026-08-22
description: 超聚变服务器重置BIOS密码,通过IBMC关闭弱口令策略、重设管理员密码并升级BIOS固件完成。
category: 服务器硬件
tags:
  - 超聚变
  - BIOS
  - 密码重置
slug: xfusion-server-reset-bios-password
---

# 超聚变服务器重置BIOS密码

关闭IBMC弱口令和密码复杂度

```text
ipmcset -t user -d weakpwddic -v disabled
ipmcset -d passwordcomplexity -v disabled
```

---

设置Administrator用户的密码

```text
ipmcset -d password -v Administrator
```

---

```text
ipmcset -d clearcmos
```

---

上传BIOS固件重置BIOS密码

```text
ipmcset -t maintenance -d upgradebios -v /tmp/1288HV7-2288HV7-5288V7-G5200V7-BIOS_01.01.00.05.hpm
```
