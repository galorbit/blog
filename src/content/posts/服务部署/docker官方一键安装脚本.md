---
title: docker官方一键安装脚本
image: "api"
published: 2026-08-22
description: 使用docker官方get-docker.sh一键脚本安装docker,通过DOWNLOAD_URL指定国内镜像源加速下载。
category: 服务部署
tags:
  - Docker
  - 软件安装
slug: docker-official-install-script
---

# docker官方一键安装脚本

> 仅适用于 Redhat/Centos Debian/Ubuntu

1. 下载脚本
   `curl -fsSL https://get.docker.com -o get-docker.sh`
   如果下载脚本报错多尝试几次
2. 使用清华源一键安装

`sudo DOWNLOAD_URL=https://mirrors.ustc.edu.cn/docker-ce sh get-docker.sh`
