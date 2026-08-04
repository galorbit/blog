---
title: "Emby 容器部署与硬件加速配置指南"
published: 2026-08-04
description: "本文介绍如何通过 Docker Compose 部署 Emby 媒体服务器，并详细讲解基于 CPU 集显的硬件加速配置与转码测试方法。"
category: "NAS"
tags:
  - "Emby"
  - "Docker Compose"
  - "硬件加速"
  - "媒体服务器"
  - "集显转码"
---

# Emby 容器部署与硬件加速配置指南

## 容器部署

compose 文件内容如下：

```yaml
version: "3.8"
services:
  embyserver:
    privileged: true # 开启特权模式
    image: amilys/embyserver # 社区版镜像
    container_name: emby
    network_mode: bridge # 网络模式选择 bridge
    environment:
      - uid=1000 # 若用户已加入 root 组可设为该用户 UID，不知道意思的直接设为 0
      - gid=0 # gid 必须设置为 0，否则容器无法使用集显，开启特权模式也没用
      - HTTP_PROXY=http://192.168.31.200:40105
      - HTTPS_PROXY=http://192.168.31.200:40105 # 代理无需求可不设置
      - TZ=Asia/Shanghai # 设置容器的时区为亚洲/上海
    volumes:
      - /share/hotdata/docker/emby:/config # 配置保存目录
      - /share/media:/media # 媒体目录，可设置多个
    ports:
      - 40130:8096 # 40130 端口可随意设置
    devices:
      - /dev/dri:/dev/dri # 集显设备目录
    restart: unless-stopped
```

完成后使用 `你的ip:40130` 打开 Emby 的管理页面，创建管理用户后即可进入控制台页面。

## 硬解

### 设置

首先确定你的 CPU 支持的解码类型。可以通过[这个表格](https://www.kdocs.cn/l/cqUOg8FWNI8A)来查看你的 CPU 支持的解码类型，感谢大佬辛苦整理的表格！

点击右上角齿轮进入控制台，选择转码菜单。
![emby](https://pic.byt3.ro/pic/emby-1.jpg)

启用硬件加速并选择高级选项。如果“首选硬件解码器”下拉框中没有显示对应选项，说明容器无法访问集显设备。请检查创建容器时的 `uid`、`gid` 设置以及集显设备挂载是否正确。

根据上述表格中你 CPU 支持的解码类型进行设置，建议优先使用 QuickSync。
![emby](https://pic.byt3.ro/pic/emby-2.jpg)

配置完成后即可创建媒体库。

### 测试硬解

点击播放媒体文件，在播放页面点击齿轮图标，选择一个较低画质以触发转码。
![emby](https://pic.byt3.ro/pic/emby-3.jpg)

点击播放统计，若硬解成功会出现绿色标识，此时可查看 CPU 占用率以验证硬解效果。
