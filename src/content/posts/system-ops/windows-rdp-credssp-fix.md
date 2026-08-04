---
title: "修复 Windows RDP 连接时的 CredSSP 身份验证错误"
published: 2026-08-04
description: "解决新版 Windows 10/11 通过 RDP 连接旧版 Windows Server 时出现的 CredSSP 加密 Oracle 修正身份验证错误。"
category: "系统运维"
tags:
  - "Windows"
  - "RDP"
  - "CredSSP"
  - "组策略"
  - "身份验证"
---

# 修复 Windows RDP 连接时的 CredSSP 身份验证错误

在新版 Windows 10 或 Windows 11 使用 RDP 连接 Windows Server 2016 或 2012 时出现：

> 发生身份验证错误，要求的函数不支持  
> 远程计算机 192.168.48.210。  
> 这可能是因为在远程计算机上阻止 NTLM 身份验证，这也可能是由于 CredSSP 加密 Oracle 修正所导致的。

## 解决方法

在本地计算机按 `Win + R` 运行 `gpedit.msc` 打开本地组策略编辑器。

导航至：计算机配置 → 管理模板 → 系统 → 凭据分配 → 加密 Oracle 修正

双击“加密 Oracle 修正”，将状态改为“已启用”。修改选项为“易受攻击”。
![rdp](https://pic.byt3.ro/pic/rdp-error-2.png)

点击确定保存后，重新尝试连接远程服务器。
