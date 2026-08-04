---
title: "威联通部署qBittorrentEE应用指南"
published: 2026-08-04
description: "本文介绍在威联通NAS上通过Docker Compose部署qBittorrentEE下载工具的完整步骤与配置说明。"
category: "服务部署"
tags:
  - "qBittorrentEE"
  - "Docker Compose"
  - "威联通"
  - "NAS"
  - "下载工具"
---

# 威联通部署qBittorrentEE应用指南

创建新应用程序：

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

完成后使用 `ip:30120` 地址打开。

![qbittorrentee](https://pic.byt3.ro/pic/qbit-0.jpg)

首次部署默认用户名和密码需要查看容器日志。

![qbittorrentee](https://pic.byt3.ro/pic/qbit-1.jpg)
