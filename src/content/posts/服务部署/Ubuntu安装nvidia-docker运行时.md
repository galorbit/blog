---
title: Ubuntu安装nvidia-docker运行时
published: 2026-08-22
description: Ubuntu安装nvidia-docker运行时,使用中科大镜像安装nvidia-container-toolkit,使容器支持GPU调用。
category: 服务部署
tags:
  - Docker
  - NVIDIA
  - GPU
slug: ubuntu-install-nvidia-docker-runtime
---

# Ubuntu安装nvidia-docker运行时

> 安装nvidia docker 运行时

1. 添加安装源和gpgkey

```bash
# 1. 下载并转换 GPG 密钥，安全存储到 keyrings 目录
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

# 2. 下载仓库列表文件，将原始地址替换为中科大镜像，并指定签名密钥
curl -s -L https://mirrors.ustc.edu.cn/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://nvidia.github.io#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://mirrors.ustc.edu.cn#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

2. 安装

`sudo apt update && sudo apt install nvidia-container-toolkit`
