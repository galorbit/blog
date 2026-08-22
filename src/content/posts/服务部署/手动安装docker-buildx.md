---
title: 手动安装docker-buildx
image: "api"
published: 2026-08-22
description: 手动安装docker-buildx插件,下载对应架构的release文件放入cli-plugins目录,重启docker验证版本。
category: 服务部署
tags:
  - Docker
  - buildx
slug: manual-install-docker-buildx
---

# 手动安装docker-buildx

## 手动安装docker buildx

从[这里下载](https://github.com/docker/buildx/releases)对应系统架构的buildx文件

添加可执行权限:

`chmod +x buildx-v0.25.0-rc2.linux-arm64`

创建文件夹:

`mkdir -p /usr/lib/docker/cli-plugins`

复制buildx到创建好的文件夹:

`cp buildx-v0.25.0-rc2.linux-arm64 /usr/lib/docker/cli-plugins/docker-buildx`

重启docker：

`systemctl restart docker`

查看是否安装成功:

`docker buildx version`

输出：`github.com/docker/buildx v0.25.0-rc2 01e731b5b2617efdfc239be5429d2f4118afbc1b`安装成功
