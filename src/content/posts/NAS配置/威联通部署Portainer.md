---
title: 威联通部署Portainer
published: 2026-08-22
description: 用docker命令在威联通NAS部署Portainer容器管理工具,映射20100端口,通过Web界面管理Docker容器。
category: NAS配置
tags:
  - 威联通
  - Portainer
  - Docker
slug: qnap-deploy-portainer
---

# 威联通部署Portainer

ssh登录 `ssh user@192.168.xx.xx`

执行:

```bash
docker run -d --restart=always --name="portainer" -p 20100:9000 -v /var/run/docker.sock:/var/run/docker.sock 6053537/portainer-ce

# 网络模式为bridge
# 20100管理端口,可随意更换
# 6053537/portainer-ce中文镜像

```
