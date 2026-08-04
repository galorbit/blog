---
title: "安装 NVIDIA Container Toolkit 运行时"
published: 2026-08-04
description: "本文记录了在 Linux 系统中添加 NVIDIA 官方及中科大镜像源，并通过 apt 安装 NVIDIA Container Toolkit 运行时的具体步骤。"
category: "服务部署"
tags:
  - "NVIDIA"
  - "Docker"
  - "容器运行时"
  - "软件源配置"
  - "环境部署"
---

# 安装 NVIDIA Container Toolkit 运行时

## 添加安装源和 GPG 密钥

```bash
# 1. 下载并转换 GPG 密钥，安全存储到 keyrings 目录
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

# 2. 下载仓库列表文件，将原始地址替换为中科大镜像，并指定签名密钥
curl -s -L https://mirrors.ustc.edu.cn/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://nvidia.github.io#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://mirrors.ustc.edu.cn#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

## 执行安装

运行以下命令更新软件源并安装工具包：

`sudo apt update && sudo apt install nvidia-container-toolkit`
