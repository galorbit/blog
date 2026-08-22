---
title: 威联通部署lucky并设置反向代理
published: 2026-08-22
description: 在威联通部署lucky容器作为反向代理并配置DDNS动态域名,推荐部署到虚拟机以保证代理稳定可用。
category: NAS配置
tags:
  - 威联通
  - Lucky
  - 反向代理
slug: qnap-deploy-lucky-reverse-proxy
---

# 威联通部署lucky并设置反向代理

## 部署

我使用的docker可视化工具是portainer,威联通自带的容器工作站也是一样的.
compose文件内容:

```yaml
version: "3"
services:
    lucky:
        container_name: lucky
        restart: always
        network_mode: host #网络模式使用host
        volumes:
            - /root/luckyconf:/goodluck #配置文件目录随意更改
        image: gdy666/lucky
```

![lucky](https://pic.byt3.ro/pic/lucky-1.jpg)

如果你比较折腾的话,可能会遇到需要经常重启docker这个应用的情况.在使用lucky容器作为反向代理的情况下,重启docker应用后lucky的容器也会重启.需要等lucky容器再次启动后才能使用域名访问NAS管理页面.

我更推荐另一种部署方式,就是在威联通的虚拟机上新建一个linux的系统,将lucky部署在虚拟机上.这样只要虚拟机不关机,反向代理总是可用的.

linux上lucky的安装官网有一键脚本,可以直接使用一键脚本安装.

部署完成后打开`ip:16601`登录网页客户端,默认用户名和密码都为`666`,登录后记得修改密码

## DDNS

打开动态域名菜单,新建任务.

```text

托管服务商:你域名所在的服务商,我这里是cloudflare

token:需要去域名服务商设置,每个服务商的设置方法都不一样,自行搜索.

我只有公网IPV6,所以选择IPV6.如果有公网IPV4,勾选IPV4.

获取方式默认

域名列表设置两条,一行一个
xx.com
*.xx.com
```

其他选项默认,确定后任务列表会显示状态.等几分钟后可以ping一下域名看看是否同步完成.

![lucky](https://pic.byt3.ro/pic/lucky-2.jpg)

## 申请SSL证书

打开SSL/TLS证书

```text
添加证书

添加方式ACME

颁发机构Let's Encrypt

验证方式选择你的域名服务商

token和域名列表设置和刚刚DDNS设置一样

其他默认
```

点击确定会自动申请证书.

![lucky](https://pic.byt3.ro/pic/lucky-3.jpg)

## 反向代理

打开WEB服务菜单.

### 添加WEB服务规则

选择定制模式.

```text
端口选择:如果443端口没有被封可以选择443端口,被封了选择其他端口.

勾选防火墙自动放行和TLS.

修改默认规则,服务类型选择反向代理,勾选忽略后端TLS证书验证,其他默认.
```

![lucky](https://pic.byt3.ro/pic/lucky-4.jpg)

### 添加子规则

子规则就是设置你需要代理的具体应用.

```text
操作模式选简易模式

服务类型反向代理

前端地址:你的域名如docker.xxx.com(不需要去域名服务商添加DNS解析,因为DDNS那边设置了泛域名解析)

后端地址:你本地服务的局域网地址如192.168.31.100:1234

勾选万事大吉

勾选忽略后端TLS证书验证.
```

![lucky](https://pic.byt3.ro/pic/lucky-6.jpg)

然后就可以使用域名来访问刚刚反向代理的服务了:`docker.xxx.com`,如果反向代理的端口不是443还需要添加端口.

需要设置别的应用反向代理,如上再添加一个子规则,设置不一样的二级域名就行.如`aaa.xxx.com`.

### 设置重定向(http自动跳转https)

新建web服务规则,选择定制模式

监听端口设置80,TLS选择禁用

然后修改默认规则,服务类型:重定向.默认目标地址:`https://{host}:{port}`,勾选万事大吉.

这样浏览器访问http会自动跳转到https

![lucky](https://pic.byt3.ro/pic/lucky-5.jpg)
