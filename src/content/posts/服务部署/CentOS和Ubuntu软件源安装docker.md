---
title: CentOS和Ubuntu软件源安装docker
published: 2026-08-22
description: CentOS与Ubuntu通过官方软件源安装docker,配置yum或apt仓库,安装docker-ce及compose插件并支持指定版本。
category: 服务部署
tags:
  - Docker
  - 软件安装
  - CentOS
slug: centos-ubuntu-install-docker-from-repo
---

# CentOS和Ubuntu软件源安装docker

## Centos

### 卸载旧版本Docker

```shell
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

```shell
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

### 从储存库安装

**首次安装会提示接受GPG秘钥，接受就行.**

1.安装最新版

```shell
sudo yum install docker-ce \
docker-ce-cli \
containerd.io \
docker-buildx-plugin \
docker-compose-plugin
```

2.安装指定版本

```shell
yum list docker-ce --showduplicates | sort -r
docker-ce.x86_64 3:24.0.0-1.el8 docker-ce-stable
docker-ce.x86_64 3:23.0.6-1.el8 docker-ce-stable
<...>
```

此处列出的版本为el8的，Centos 7或者其他版本显示不一样，安装方法一样的。

此处安装23.0.6版本，安装其他版本也一样。都是`docker-ce-版本号`和`docker-ce-cli-版本号`

```shell
sudo yum install docker-ce-3:23.0.6-1.el8 \
docker-ce-cli-3:23.0.6-1.el8 \
containerd.io \
docker-buildx-plugin \
docker-compose-plugin
```

### 从RPM软件包安装

在[这个链接](https://download.docker.com/linux/centos/)，选择你的系统版本和需要安装的docker版本，下载rpm包。

`sudo yum install /路径/包名.rpm`

### 启动和开机自启

自启

`sudo systemctl enable docker`

启动

`sudo systemctl start docker`

## Ubuntu

### 卸载有冲突的软件包

```shell
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

### 安装必要软件

```shell
sudo apt update
sudo apt install apt-transport-https \
ca-certificates \
curl gnupg-agent \
software-properties-common
```

### 添加源和GPG KEY

导入GPG

`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`

添加apt源

`sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"`

### 安装

安装最新版

```shell
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
```

安装指定版本

列出可用版本

```shell
sudo apt update
apt list -a docker-ce
docker-ce/focal 5:24.0.6-1~ubuntu.20.04~focal amd64
docker-ce/focal 5:24.0.5-1~ubuntu.20.04~focal amd64
docker-ce/focal 5:24.0.4-1~ubuntu.20.04~focal amd64
docker-ce/focal 5:24.0.3-1~ubuntu.20.04~focal amd64
docker-ce/focal 5:24.0.2-1~ubuntu.20.04~focal amd64
...
```

安装24.0.6版本

`sudo apt install docker-ce=5:24.0.6-1~ubuntu.20.04~focal`

其他软件包也一样，`sudo apt install 包名=版本号`
