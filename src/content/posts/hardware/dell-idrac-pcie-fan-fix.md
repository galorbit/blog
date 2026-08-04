---
title: "Dell服务器安装第三方PCIe卡风扇满载噪音解决"
published: 2026-08-04
description: "针对Dell服务器安装非认证PCIe卡引发风扇满载噪音的问题，提供通过iDRAC命令行关闭LFM模式的详细步骤。"
category: "硬件与BMC"
tags:
  - "Dell服务器"
  - "iDRAC"
  - "PCIe扩展卡"
  - "风扇控制"
  - "BMC管理"
---

# Dell服务器安装第三方PCIe卡风扇满载噪音解决

Dell服务器安装不在兼容列表的第三方显卡或其他PCIE卡后，风扇转速一直满载，噪音很大。

## 背景说明

iDRAC9 引入多矢量冷却（Multi-Vector Cooling）技术。iDRAC/Lifecycle会检测戴尔认证PCIe卡，并自动将正确的气流输送到插槽以冷却该卡；当检测到非戴尔认证PCIe卡时（如非认证的GPU，网卡，HBA卡等），客户可以选择输入PCIe卡制造商指定的气流LFM（Linear Feet per Minute）要求，iDRAC和风扇算法将“学习”此信息，并将非戴尔认证PCIe卡自动冷却，为其输送适当的气流。

如果不知道LFM值，可设定为禁用状态。

~~登陆 iDRAC图形界面 配置----系统配置---硬件配置---PCIe 通风口配置中 将非认证PCIe卡的LFM模式设置为禁用。（或自定义LFM）~~   
iDRAC图形界面自定义LFM功能已经被取消，只能通过命令行关闭。

LFM Mode 有三个模式：自定义、自动和禁用。

## 命令行配置

```bash
ssh root@ip  # SSH登录iDRAC
racadm get System.PCIESlotLFM  # 查看所有槽位信息
racadm get System.PCIESlotLFM.2  # 获取槽位2的具体信息
racadm set System.PCIESlotLFM.2.LFMMode 1  # 将Slot2的LFM mode修改为disabled
# 0: auto(默认) 1: disabled 2: custom
```

![idrac](https://pic.byt3.ro/pic/IDRAC-0.jpg)

确保关闭自动冷却后PCIE卡的温度不会受影响。

参考链接:

- [idrac风扇修改](https://www.dell.com/community/zh/conversations/poweredge%E6%9C%8D%E5%8A%A1%E5%99%A8/14g%E6%9C%8D%E5%8A%A1%E5%99%A8-%E6%B1%87%E6%80%BB8-%E5%8A%A0%E4%BA%86%E7%AC%AC%E4%B8%89%E6%96%B9%E7%9A%84pci%E5%8D%A1%E5%BC%95%E8%B5%B7fan%E5%8A%A0%E9%80%9F%E8%BD%AC%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/647f7c29f4ccf8a8dea796de)
