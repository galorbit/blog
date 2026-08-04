---
title: "CentOS 与 Ubuntu Docker 安装指南"
published: 2026-08-04
description: "详细记录在 CentOS 和 Ubuntu 系统中安装、配置及启动 Docker 服务的完整步骤与注意事项。"
category: "服务部署"
tags:
  - "Docker"
  - "CentOS"
  - "Ubuntu"
  - "软件源"
  - "容器引擎"
---

# CentOS 与 Ubuntu Docker 安装指南

## CentOS

### 卸载旧版本 Docker

```bash
sudo yum remove docker \
docker-client \
docker-client-latest \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-engine
```

### 设置储存库

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

### 从储存库安装

**首次安装会提示接受 GPG 密钥，接受即可。**

1. 安装最新版

```bash
sudo yum install docker-ce \
docker-ce-cli \
containerd.io \
docker-buildx-plugin \
docker-compose-plugin
```

2. 安装指定版本

```bash
yum list docker-ce --showduplicates | sort -r
docker-ce.x86_64 3:24.0.0-1.el8 docker-ce-stable
docker-ce.x86_64 3:23.0.6-1.el8 docker-ce-stable
<...>
```

此处列出的版本为 el8 的，CentOS 7 或者其他版本显示不一样，安装方法一样的。

此处安装 23.0.6 版本，安装其他版本也一样。都是 `docker-ce-版本号` 和 `docker-ce-cli-版本号`。

```bash
sudo yum install docker-ce-3:23.0.6-1.el8 \
docker-ce-cli-3:23.0.6-1.el8 \
containerd.io \
docker-buildx-plugin \
docker-compose-plugin
```

### 从 RPM 软件包安装

在[这个链接](https://download.docker.com/linux/centos/)，选择你的系统版本和需要安装的 Docker 版本，下载 rpm 包。

`sudo yum install /路径/包名.rpm`

### 启动和开机自启

自启

`sudo systemctl enable docker`

启动

`sudo systemctl start docker`

## Ubuntu

### 卸载有冲突的软件包

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

### 安装必要软件

```bash
sudo apt update
sudo apt install apt-transport-https \
ca-certificates \
curl gnupg-agent \
software-properties-common
```

### 添加源和 GPG KEY

导入 GPG

`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`

添加 apt 源

`sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"`

### 安装

安装最新版

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
```

安装指定版本

列出可用版本

```bash
sudo apt update
apt list -a docker-ce
docker-ce/focal 5:24.0.6-1~ubuntu.20.04~focal amd64
docker-ce/focal 5:24.0.5-1~ubuntu.20.04~focal amd64
docker-ce/focal 5:24.0.4-1~ubuntu.20.04~focal amd64
docker-ce/focal 5:24.0.3-1~ubuntu.20.04~focal amd64
docker-ce/focal 5:24.0.2-1~ubuntu.20.04~focal amd64
...
```

安装 24.0.6 版本

`sudo apt install docker-ce=5:24.0.6-1~ubuntu.20.04~focal`

其他软件包也一样，`sudo apt install 包名=版本号`
