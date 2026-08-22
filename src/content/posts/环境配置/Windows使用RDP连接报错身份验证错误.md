---
title: Windows使用RDP连接报错身份验证错误
published: 2026-08-22
description: 解决新版Windows用RDP连接旧版Server报身份验证错误,组策略中把加密数据库修正设为易受攻击。
category: 环境配置
tags:
  - Windows
  - RDP
  - 远程桌面
slug: windows-rdp-auth-error-fix
---

# Windows使用RDP连接报错身份验证错误

在新版windows10或者windows11使用RDP连接windows server 2016或2012时出现:

发生身份验证错误，要求的函数不支持
远程计算机192.168.48.210.
这可能是因为在远程计算机上阻止NTLM身份验证，这也可能是由于Credssp加密oracle修正所导致的。

## 解决方法

在本地计算机WIN+R运行gpedit.msc打开本地组策略编辑器。

导航到：计算机配置-管理模板-系统-凭据分配-加密数据库修正

双击"加密数据库修正"，将状态改为"已启用"。修改选项为"易受攻击"。

![rdp](https://pic.byt3.ro/pic/rdp-error-2.png)

确定保存后重新尝试连接远程服务器。
