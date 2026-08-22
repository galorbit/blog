---
title: yum安装对应内核版本的kernel-devel
image: "api"
published: 2026-08-22
description: 用yum安装与当前内核版本完全一致的kernel-devel,解决编译内核模块时版本不匹配导致的报错或加载失败。
category: 环境配置
tags:
  - 内核开发
  - yum安装
  - kernel-devel
slug: yum-install-kernel-devel-matching-kernel-version
---

# yum 安装与内核版本对应的 kernel-devel

## 背景说明

- 编译内核模块(例如某些驱动、需要 `modprobe` 加载的模块)时,必须安装 `kernel-devel`。
- `kernel-devel` 的版本必须与**当前运行的内核版本完全一致**,否则编译会报错或模块无法加载。
- 本文适用于 RHEL / CentOS 等使用 `yum` 的发行版。

## 整体流程

1. 查看当前内核版本
2. 查看 yum 源中可用的 kernel-devel 版本
3. 安装与内核一致版本
4. 验证安装结果
5. RHEL 8+ 的 `kernel-devel-matched` 方式

---

## 1. 查看当前内核版本

```bash
uname -r
```

例如输出:

```text
5.14.0-284.11.1.el9_2.x86_64
```

记下这个版本号,后面要安装的 kernel-devel 必须与它一致。

## 2. 查看 yum 源中可用的 kernel-devel 版本

```bash
yum --showduplicates list kernel-devel
```

在输出中确认存在与步骤 1 版本号一致的包(例如 `kernel-devel-5.14.0-284.11.1.el9_2.x86_64`)。

> 提示:如果列表里找不到对应版本,可能需要先 `yum update` 更新内核和仓库,或确认已配置正确的 yum 源。

## 3. 安装与当前内核版本对应的 devel

`$(uname -r)` 会自动替换为当前内核版本号,无需手工填写:

```bash
sudo yum install -y kernel-devel-$(uname -r)
```

如果同时需要内核头文件,可以一并安装:

```bash
sudo yum install -y kernel-devel-$(uname -r) kernel-headers-$(uname -r)
```

## 4. 验证安装结果

确认已安装的 kernel-devel 包:

```bash
rpm -qa | grep kernel-devel
```

确认头文件目录已生成:

```bash
ls /usr/src/kernels/
```

目录名与 `uname -r` 的输出一致,即安装成功。

## 5. RHEL 8+ 的 matched 包

RHEL 8 及以上版本的 devel 包带有内核类型后缀(`default`、`rt` 等),直接 `yum install kernel-devel` 可能无法正确匹配。有两种安装方式:

方式一:使用 `kernel-devel-matched`,自动匹配当前内核类型

```bash
yum install kernel-devel-matched
```

方式二:明确指定与当前内核一致的版本

```bash
yum install kernel-devel-$(uname -r)
```

## 常见问题

- **找不到对应版本**:先执行 `yum update` 更新内核和仓库,或确认已订阅所需仓库。
- **安装后 `/usr/src/kernels` 下没有目录**:说明安装的版本与当前内核不一致,改用 `kernel-devel-$(uname -r)` 重新安装。
- **提示权限不足**:命令前加 `sudo`。
