---
title: 威联通部署qbittorrentee
image: "api"
published: 2026-08-22
description: 用docker compose在威联通部署qbittorrentee下载工具,host网络并设置PUID/PGID与Web管理端口。
category: NAS配置
tags:
  - 威联通
  - qBittorrent
  - 下载工具
slug: qnap-deploy-qbittorrentee
---

# 威联通部署qbittorrentee

创建新应用程序:

compose文件内容

```yaml
version: "3"
services:
  qbittorrentee:
    image: superng6/qbittorrentee #容器名
    container_name: qbittorrentee #使用镜像
    network_mode: host #网络模式使用host
    restart: unless-stopped
    environment:
      - TZ=Asia/Shanghai
      - WEBUIPORT=30120 #WEB管理页面端口
      - PUID=1000
      - PGID=1000 #PUID和PGID建议设置为非ADMIN用户的ID,方便SMB管理
    volumes:
      - /share/hotdata/docker/qbittorrent:/config #配置目录
      - /share/media/downloads:/downloads
```

完成后使用`ip:30120`地址打开.

![qbittorrentee](https://pic.byt3.ro/pic/qbit-0.jpg)

首次部署默认用户名和密码需要查看容器日志.

![qbittorrentee](https://pic.byt3.ro/pic/qbit-1.jpg)
