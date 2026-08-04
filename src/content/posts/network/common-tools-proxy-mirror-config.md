---
title: "常用工具镜像源与网络代理配置记录"
published: 2026-08-04
description: "记录 Docker、Pip、Git 及终端的国内镜像源与 HTTP/HTTPS 代理配置方法。"
category: "网络配置"
tags:
  - "镜像源"
  - "网络代理"
  - "Docker"
  - "Pip"
  - "Git"
---

# 常用工具镜像源与网络代理配置记录

## Docker 设置国内源

`nano /etc/docker/daemon.json`

添加以下内容：
```json
{
    "registry-mirrors": ["https://dockerpull.cn"]
}
```

## Docker Build 设置代理

```bash
docker build -f docker/Dockerfile.rocm -t vllm-rocm . \
        --network host \
 --build-arg HTTP_PROXY=http://192.168.48.101:7897 \
 --build-arg HTTPS_PROXY=http://192.168.48.101:7897
```

## Pip 配置

临时使用清华源

`pip install evalscope -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple some-package`

Pip 设置代理

```bash
pip config set global.http_proxy http://192.168.48.101:7897
pip config set global.https_proxy http://192.168.48.101:7897
```

## Git 设置代理

```bash
git config --global http.proxy http://192.168.48.101:7897
git config --global https.proxy https://192.168.48.101:7897
```

## 终端临时设置代理

```bash
export HTTP_PROXY="http://192.168.48.101:7897"
export HTTPS_PROXY="http://192.168.48.101:7897"
```
