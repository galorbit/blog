---
title: "QNAP NAS 安装配置 Entware 与 Zsh 指南"
published: 2026-08-04
description: "详细介绍在QNAP NAS设备上安装Entware环境，并配置Zsh与Oh-My-Zsh以实现持久化Shell体验。"
category: "NAS"
tags:
  - "QNAP"
  - "Entware"
  - "Zsh"
  - "Oh-My-Zsh"
  - "NAS配置"
---

# QNAP NAS 安装配置 Entware 与 Zsh 指南

## 引言

本文详细介绍了在 QNAP NAS 设备上安装和使用 Entware 的过程，包括如何获取适合版本、安装步骤以及配置必要的工具如 zsh 和 oh-my-zsh。通过这些步骤，用户可以在 QNAP 上获得类似 Linux 的 Shell 环境，并确保配置在重启后仍然有效。

## 安装过程

### 下载 Entware

1. 访问 [Qnapclub Store](https://www.qnapclub.eu/en/qpkg/) 找到适合您设备架构的 Entware 版本。

   - 例如，TS-551 和 TS-453Bmini 用户应选择 **x86_64** 版本。

2. 下载链接示例：

```text
https://www.qnapclub.eu/en/qpkg/model/download/11369/Entware-ng_0.97.qpkg
```

### 安装 Entware

1. 通过 SSH 连接到您的 QNAP NAS。
2. 执行以下命令进行安装：

```bash
cd /tmp
wget https://www.qnapclub.eu/en/qpkg/model/download/11369/Entware-ng_0.97.qpkg
sh Entware-ng_0.97.qpkg
rm Entware-ng_0.97.qpkg
```

## 使用配置

### 更新 OPKG 源

```bash
opkg update
```

### 安装 zsh 和 oh-my-zsh

1. 安装 zsh：

```bash
opkg install zsh
```

2. 安装 git（用于 oh-my-zsh）：

```bash
opkg install git-http
```

3. 安装并配置 oh-my-zsh：

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

## 配置持久化

为了避免重启后配置丢失，需进行以下操作：

### 创建目录并移动文件

```bash
mkdir -p /share/CACHEDEV1_DATA/.zsh
cd ~
mv .zsh_history .zshrc .oh-my-zsh /share/CACHEDEV1_DATA/.zsh
```

### 修改启动脚本

编辑 `/share/CACHEDEV1_DATA/.qpkg/Entware/Entware.sh` 文件，添加以下内容到启动命令中：

```bash
/share/CACHEDEV1_DATA/.zsh/.oh-my-zsh/zsh.sh
```

确保在 QNAP 重启时自动执行此脚本。

## 补充内容

### 安装 sudo 和其他工具

- 安装 sudo：

```bash
opkg install sudo
```

- 安装 coreutils（如 `grep`、`sed`）：

```bash
opkg install coreutils
```
