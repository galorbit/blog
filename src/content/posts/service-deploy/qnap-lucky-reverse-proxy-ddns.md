---
title: "威联通NAS部署Lucky反向代理与DDNS配置"
published: 2026-08-04
description: "本文介绍在威联通NAS上通过Docker或虚拟机部署Lucky工具，并配置动态域名解析、SSL证书及HTTPS反向代理的详细步骤。"
category: "服务部署"
tags:
  - "Lucky"
  - "反向代理"
  - "DDNS"
  - "威联通"
  - "Docker"
  - "SSL证书"
---

# 威联通NAS部署Lucky反向代理与DDNS配置

## 部署

我使用的 Docker 可视化工具是 Portainer，威联通自带的容器工作站也是一样的。

Compose 文件内容如下：

```yaml
version: "3"
services:
  lucky:
    container_name: lucky
    restart: always
    network_mode: host # 网络模式使用host
    volumes:
      - /root/luckyconf:/goodluck # 配置文件目录随意更改
    image: gdy666/lucky
```

![lucky](https://pic.byt3.ro/pic/lucky-1.jpg)

如果你比较折腾的话，可能会遇到需要经常重启 Docker 这个应用的情况。在使用 Lucky 容器作为反向代理的情况下，重启 Docker 应用后 Lucky 的容器也会重启。需要等 Lucky 容器再次启动后才能使用域名访问 NAS 管理页面。

我更推荐另一种部署方式，就是在威联通的虚拟机上新建一个 Linux 的系统，将 Lucky 部署在虚拟机上。这样只要虚拟机不关机，反向代理总是可用的。

Linux 上 Lucky 的安装官网有一键脚本，可以直接使用一键脚本安装。

部署完成后打开 `ip:16601` 登录网页客户端，默认用户名和密码都为 `666`，登录后记得修改密码。

## DDNS 配置

打开动态域名菜单，新建任务。

```text
托管服务商: 你域名所在的服务商，我这里是 Cloudflare
token: 需要去域名服务商设置，每个服务商的设置方法都不一样，自行搜索。
我只有公网 IPv6，所以选择 IPv6。如果有公网 IPv4，勾选 IPv4。
获取方式默认
域名列表设置两条，一行一个
xx.com 
*.xx.com
```

其他选项默认，确定后任务列表会显示状态。等几分钟后可以 ping 一下域名看看是否同步完成。

![lucky](https://pic.byt3.ro/pic/lucky-2.jpg)

## 申请 SSL 证书

打开 SSL/TLS 证书菜单。

```text
添加证书
添加方式: ACME
颁发机构: Let's Encrypt
验证方式: 选择你的域名服务商
token 和域名列表设置和刚刚 DDNS 设置一样
其他默认
```

点击确定会自动申请证书。

![lucky](https://pic.byt3.ro/pic/lucky-3.jpg)

## 反向代理配置

打开 WEB 服务菜单。

### 添加 WEB 服务规则

选择定制模式。

```text
端口选择: 如果 443 端口没有被封可以选择 443 端口，被封了选择其他端口。
勾选防火墙自动放行和 TLS。
修改默认规则，服务类型选择反向代理，勾选忽略后端 TLS 证书验证，其他默认。
```

![lucky](https://pic.byt3.ro/pic/lucky-4.jpg)

### 添加子规则

子规则就是设置你需要代理的具体应用。

```text
操作模式选简易模式
服务类型: 反向代理
前端地址: 你的域名如 docker.xxx.com（不需要去域名服务商添加 DNS 解析，因为 DDNS 那边设置了泛域名解析）
后端地址: 你本地服务的局域网地址如 192.168.31.100:1234
勾选万事大吉
勾选忽略后端 TLS 证书验证。
```

![lucky](https://pic.byt3.ro/pic/lucky-6.jpg)

然后就可以使用域名来访问刚刚反向代理的服务了：`docker.xxx.com`，如果反向代理的端口不是 443 还需要添加端口。

需要设置别的应用反向代理，如上再添加一个子规则，设置不一样的二级域名就行。如 `aaa.xxx.com`。

### 设置重定向（HTTP 自动跳转 HTTPS）

新建 WEB 服务规则，选择定制模式。

监听端口设置 80，TLS 选择禁用。

然后修改默认规则，服务类型：重定向。默认目标地址：`https://{host}:{port}`，勾选万事大吉。

这样浏览器访问 HTTP 会自动跳转到 HTTPS。

![lucky](https://pic.byt3.ro/pic/lucky-5.jpg)
