---
title: "Ubuntu 系统安装 Docker CE 指南"
published: 2026-08-04
description: "本文记录了在 Ubuntu 系统上通过清华大学镜像源安装 Docker CE 及相关组件的完整步骤。"
category: "服务部署"
tags:
  - "Docker"
  - "Ubuntu"
  - "软件源"
  - "容器化"
  - "命令行"
---

# Ubuntu 系统安装 Docker CE 指南

## 安装依赖

`apt install curl vim wget gnupg dpkg apt-transport-https lsb-release ca-certificates`

## 添加验证密钥

```bash
install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

## 添加并更换安装源

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## 更新源

`apt update`

## 安装

`apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin`
