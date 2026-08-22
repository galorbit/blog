---
title: 修改docker默认存储目录
published: 2026-08-22
description: docker默认目录空间不足时,迁移数据并在daemon.json中设置data-root更换存储位置。
category: 环境配置
tags:
  - Docker
  - 存储
slug: docker-default-storage-dir-change
---

# 修改docker默认存储目录

docker默认存储目录`/var/lib/docker`被占满，本地导入docker镜像提示空间不足解决方法。

停止docker服务
`systemctl stop docker`

创建新的存储目录
`mkdir -p /data/docker`

移动原有内容
`mv /var/lib/docker /data/docker`

创建并修改docker存储目录配置文件
`vim /etc/docker/daemon.json`

添加以下内容

```shell
{
  "registry-mirrors": ["https://dockerpull.com"],
  "data-root": "/data/docker"
}
```

重启docker服务
`systemctl restart docker`

查看是否修改成功
`docker info|grep Dir`

如果列出的目录即为当前docker的默认目录。
