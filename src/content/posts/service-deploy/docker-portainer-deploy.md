---
title: "使用 Docker 部署 Portainer CE 容器"
published: 2026-08-04
description: "记录通过 Docker 运行 Portainer CE 容器的完整命令及关键参数说明。"
category: "服务部署"
tags:
  - "Docker"
  - "Portainer"
  - "容器管理"
  - "服务部署"
---

# 使用 Docker 部署 Portainer CE 容器

SSH 登录：`ssh user@192.168.xx.xx`

执行：

```bash
docker run -d --restart=always --name="portainer" -p 20100:9000 -v /var/run/docker.sock:/var/run/docker.sock 6053537/portainer-ce

# 网络模式为bridge
# 20100管理端口,可随意更换
# 6053537/portainer-ce中文镜像
```
