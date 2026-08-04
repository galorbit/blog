---
title: "手动安装 Docker Buildx 插件指南"
published: 2026-08-04
description: "介绍在 Linux 环境中手动下载、赋予权限、配置路径并验证 Docker Buildx 插件的安装流程。"
category: "服务部署"
tags:
  - "Docker"
  - "buildx"
  - "Linux"
  - "命令行"
  - "插件管理"
---

# 手动安装 Docker Buildx 插件指南

从 [这里下载](https://github.com/docker/buildx/releases) 对应系统架构的 buildx 文件。

添加可执行权限：
`chmod +x buildx-v0.25.0-rc2.linux-arm64`

创建文件夹：
`mkdir -p /usr/lib/docker/cli-plugins`

复制 buildx 到创建好的文件夹：
`cp buildx-v0.25.0-rc2.linux-arm64 /usr/lib/docker/cli-plugins/docker-buildx`

重启 Docker：
`systemctl restart docker`

查看是否安装成功：
`docker buildx version`

输出示例：`github.com/docker/buildx v0.25.0-rc2 01e731b5b2617efdfc239be5429d2f4118afbc1b` 安装成功。
