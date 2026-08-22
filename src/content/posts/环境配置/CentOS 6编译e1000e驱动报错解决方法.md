---
title: CentOS 6编译e1000e驱动报错解决方法
published: 2026-08-22
description: 解决CentOS 6编译e1000e网卡驱动的'dev' undeclared报错,通过降级驱动版本、修改源码与安装内核开发包。
category: 环境配置
tags:
  - 网卡驱动
  - 编译
  - CentOS
slug: centos6-compile-e1000e-driver-fix
---

# CentOS 6编译e1000e驱动报错解决方法

## Centos编译e1000e驱动报错'dev' undeclared

内核版本不兼容

CentOS 6 默认内核版本为 2.6.32，而较新版本的 e1000e 驱动（如 3.8.4）可能依赖更高版本内核的 API，导致变量名（如 dev）或函数接口不兼容 。

驱动源码与内核头文件冲突

部分驱动源码中变量名（如 dev）可能与当前内核头文件中的定义冲突，或未正确声明相关结构体 。

选择兼容的驱动版本
降级驱动版本：下载适用于 CentOS 6 内核（2.6.32）的 e1000e 驱动版本（如 3.2.6），避免使用过新的驱动包 。

## 查看当前驱动版本

`modinfo -F version e1000e`

手动修复源码中的变量名

### 修改netdev.c文件：

在报错位置（如 netdev.c:7788）将 dev 替换为 pci_dev 或其他符合内核头文件定义的变量名。例如：

```text
#原始代码可能为：
struct net_device *netdev = pci_get_drvdata(to_pci_dev(dev));
#修改为：
struct net_device *netdev = pci_get_drvdata(pci_dev);
#需结合具体报错行和内核头文件检查变量声明 。
```

### 安装内核开发包：

确认已安装与当前内核版本匹配的 kernel-devel 和 kernel-headers：

`yum install kernel-devel-$(uname -r) gcc`

指定内核源码路径：

在驱动源码的 common.mk 文件中，添加正确的内核源码路径 ：

`KSP := /usr/src/kernels/2.6.32-573.el6.x86_64`

清理并重新编译：
修改源码后执行 make clean 再重新编译：
`make clean && make && make install`
