---
title: 威联通部署emby开心版
image: "api"
published: 2026-08-22
description: 用docker compose在威联通部署emby开心版,配置特权模式与集显直通实现硬解,挂载媒体目录与代理。
category: NAS配置
tags:
  - 威联通
  - Emby
  - 硬解
slug: qnap-deploy-emby
---

# 威联通部署emby开心版

## 容器部署

compose文件内容

```yaml
version: "3.8"
services:
    embyserver:
        privileged: true #开启特权模式
        image: amilys/embyserver #开心版的镜像
        container_name: emby
        network_mode: bridge # 网络模式选择bridge
        environment:
            - uid=1000 #如果你创建的用户加入了root组也可以设置为你用户的uid.不知道什么意思的直接设置为0
            - gid=0 #gid必须设置为0 不然容器无法使用集显,开启特权模式也没用
            - HTTP_PROXY=http://192.168.31.200:40105
            - HTTPS_PROXY=http://192.168.31.200:40105 #代理没需求可以不设置
            - TZ=Asia/Shanghai # 设置容器的时区为亚洲/上海
        volumes:
            - /share/hotdata/docker/emby:/config # 配置保存目录
            - /share/media:/media #媒体目录,可以设置多个
        ports:
            - 40130:8096 # 40130端口随意设置
        devices:
            - /dev/dri:/dev/dri #集显设备目录
        restart: unless-stopped

```

完成后使用`你的ip:40130`打开emby的管理页面,创建管理用户后就可以到控制台页面了.

## 硬解

### 设置

首先确定的你CPU支持的解码类型.可以通过[这个表格](https://www.kdocs.cn/l/cqUOg8FWNI8A)来查看你的CPU支持的解码类型,感谢大佬辛苦整理的表格!

点击右上角齿轮进入控制台,选择转码菜单.
![emby](https://pic.byt3.ro/pic/emby-1.jpg)

启用硬件加速 选择高级
如果首选硬件解码器那边没有像我一样显示,就是容器不能访问集显设备.检查创建容器时的uid gid设置和集显设备挂载是否有问题.

根据上面表格上你CPU支持的解码类型来设置,建议优先使用QuickSync
![emby](https://pic.byt3.ro/pic/emby-2.jpg)

配置完成后创建媒体库.

### 测试硬解

点击播放媒体文件,播放页面点击齿轮,设置一个低质量来启用转码.
![emby](https://pic.byt3.ro/pic/emby-3.jpg)

点击播放统计,如果硬解成功了会有个绿色的小方块,再查看CPU占用查看硬解效果.
