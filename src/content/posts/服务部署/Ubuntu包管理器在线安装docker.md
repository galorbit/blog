---
title: Ubuntu包管理器在线安装docker
image: "api"
published: 2026-08-22
description: Ubuntu系统通过apt在线安装docker,添加GPG密钥与docker-ce软件源,安装docker-ce及compose插件。
category: 服务部署
tags:
  - Docker
  - 软件安装
  - Ubuntu
slug: ubuntu-package-manager-install-docker
---

# Ubuntu包管理器在线安装docker

安装依赖

`apt install curl vim wget gnupg dpkg apt-transport-https lsb-release ca-certificates`

添加验证密钥

```text
install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

添加并更换安装源

```text
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
```

更新源
`apt update`

安装
`apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin`
