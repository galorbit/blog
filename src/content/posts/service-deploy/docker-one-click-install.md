---
title: "Docker 官方脚本一键安装指南"
published: 2026-08-04
description: "本文介绍在 RedHat/CentOS 及 Debian/Ubuntu 系统上，通过官方脚本配合国内镜像源一键安装 Docker 的方法。"
category: "服务部署"
tags:
  - "Docker"
  - "脚本安装"
  - "清华源"
  - "Linux"
---

# Docker 官方脚本一键安装指南

> 仅适用于 RedHat/CentOS、Debian/Ubuntu 系统。

1. 下载脚本
   ```bash
   curl -fsSL https://get.docker.com -o get-docker.sh
   ```
   如果下载脚本报错，请多尝试几次。

2. 使用清华源一键安装
   ```bash
   sudo DOWNLOAD_URL=https://mirrors.ustc.edu.cn/docker-ce sh get-docker.sh
   ```
