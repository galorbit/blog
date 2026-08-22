---
title: QNAP安装与配置Entware指南
image: "api"
published: 2026-08-22
description: 在QNAP NAS上安装Entware获得类Linux软件环境,配置opkg源并安装zsh与oh-my-zsh,重启后配置依然生效。
category: NAS配置
tags:
  - QNAP
  - Entware
slug: qnap-install-configure-entware
---

# QNAP 安装与配置 Entware 的指南

## 引言

本文详细介绍了在QNAP NAS设备上安装和使用Entware的过程，包括如何获取适合版本、安装步骤以及配置必要的工具如zsh和oh-my-zsh。通过这些步骤，用户可以在QNAP上获得类似Linux的Shell环境，并确保配置在重启后仍然有效。

---

## 安装过程

### 下载 Entware

1. 访问 [Qnapclub Store](https://www.qnapclub.eu/en/qpkg/) 找到适合您设备架构的Entware版本。

- 例如，TS-551和TS-453Bmini用户应选择 **x86_64** 版本。

2. 下载链接示例：

```text
https://www.qnapclub.eu/en/qpkg/model/download/11369/Entware-ng_0.97.qpkg
```

### 安装 Entware

1. 通过SSH连接到您的QNAP NAS。
2. 执行以下命令进行安装：

```bash
cd /tmp
wget https://www.qnapclub.eu/en/qpkg/model/download/11369/Entware-ng_0.97.qpkg
sh Entware-ng_0.97.qpkg
rm Entware-ng_0.97.qpkg
```

---

## 使用配置

### 更新 OPKG 源

```bash
opkg update
```

### 安装 zsh 和 oh-my-zsh

1. 安装zsh：

```bash
opkg install zsh
```

2. 安装git（用于oh-my-zsh）：

```bash
opkg install git-http
```

3. 安装并配置oh-my-zsh：

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

---

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

确保在QNAP重启时自动执行此脚本。

---

## 补充内容

### 安装 sudo 和其他工具

- 安装sudo：

```bash
opkg install sudo
```

- 安装coreutils（如`grep`、`sed`）：

```bash
opkg install coreutils
```

---
