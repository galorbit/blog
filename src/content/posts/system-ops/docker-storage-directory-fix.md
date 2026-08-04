---
title: "Docker默认存储目录空间不足解决方法"
published: 2026-08-04
description: "解决Docker默认存储目录空间不足问题，通过修改daemon.json配置迁移数据根目录的操作指南。"
category: "系统运维"
tags:
  - "Docker"
  - "存储目录"
  - "空间不足"
  - "镜像导入"
---

# Docker默认存储目录空间不足解决方法

Docker 默认存储目录 `/var/lib/docker` 被占满，本地导入 Docker 镜像提示空间不足解决方法。

停止 Docker 服务
`systemctl stop docker`

创建新的存储目录
`mkdir -p /data/docker`

移动原有内容
`mv /var/lib/docker /data/docker`

创建并修改 Docker 存储目录配置文件
`vim /etc/docker/daemon.json`

添加以下内容：
```json
{
  "registry-mirrors": ["https://dockerpull.com"],
  "data-root": "/data/docker"
}
```

重启 Docker 服务
`systemctl restart docker`

查看是否修改成功
`docker info | grep Dir`

如果列出的目录即为当前 Docker 的默认目录。
