---
title: ESXi_VGPU驱动安装和授权服务器搭建
published: 2026-08-23
description: 在ESXi主机上安装NVIDIA vGPU驱动，配置图形共享，并部署FastAPI-DLS授权服务器供虚拟机使用vGPU。
image: "api"
category: 环境配置
tags:
  - vGPU驱动
  - 授权服务器
  - ESXi
slug: esxi-vgpu-driver-installation-and-license-server-setup
---

# ESXi 安装 vGPU驱动与授权服务器部署文档

## 1\. 文档说明

主要包含以下内容：

* 环境 IP 规划
* ESXi 主机安装 NVIDIA vGPU 驱动
* 驱动验证与卸载
* ESXi 图形设备共享配置
* FastAPI-DLS 授权服务器部署（Docker / Docker Compose）
* 虚拟机使用 vGPU 的简要配置
* 常用命令与排障建议

\---

## 2\. 环境规划

|用途|IP 地址|说明|
|-|-|-|
|vCenter|192.168.48.90|管理 ESXi 主机和虚拟机|
|ESXi|192.168.48.91|安装 NVIDIA vGPU 驱动并承载 vGPU 虚拟机|
|OE-Server|192.168.48.92|其他业务服务器（文档中不做详细部署说明）|
|VGPU-License|192.168.48.95|FastAPI-DLS 授权服务器，提供 NVIDIA vGPU 授权|

> 注意：实际部署时请根据现场网段、DNS、防火墙规则调整以上地址。

\---

## 3\. ESXi 安装 NVIDIA vGPU 驱动

### 3.1 准备驱动包

1. 从 NVIDIA 官网或硬件厂商获取对应 ESXi 版本的 NVIDIA vGPU 驱动离线包。
2. 将驱动包上传到 ESXi 主机，例如保存到 `/tmp` 目录：

   * 开启 ESXi SSH 服务后使用 `scp`、`sftp` 或 WinSCP 上传；
   * 也可以上传到数据存储后再复制到 `/tmp`。

### 3.2 安装驱动

在 ESXi Shell 或 SSH 中执行：

```bash
esxcli software vib install -d /tmp/\*\*\*\*.zip
```

其中 `\*\*\*\*.zip` 为实际的驱动离线包文件名。

建议在安装前将主机置于维护模式：

```bash
esxcli system maintenanceMode set -e true
```

安装完成后：

```bash
esxcli system maintenanceMode set -e false
```

然后重启 ESXi 主机：

```bash
reboot
```

### 3.3 验证驱动是否正常装载

重启完成后，重新通过 SSH/ESXi Shell 登录，查看 NVIDIA 相关内核模块：

```bash
vmkload\_mod -l | grep nvidia
```

执行 `nvidia-smi` 查看 GPU 是否正常识别：

```bash
nvidia-smi
```

如果能看到 NVIDIA GPU 信息且没有报错，说明 vGPU 驱动已经正常安装。

\---

## 4\. ESXi 图形共享配置

vGPU 需要将 ESXi 主机的 GPU 设置为“直接共享（Shared Direct）”模式，操作如下：

1. 登录 vCenter 或 vSphere Client。
2. 选择对应的 ESXi 主机（192.168.48.91）。
3. 进入 **配置（Configure）** > **图形（Graphics）**。
4. 选择要配置的 GPU 设备，点击 **编辑（Edit）**。
5. 共享类型选择 **直接共享（Shared Direct）**。
6. 勾选 **重新启动 X.org 服务器（Restart X.org server）**。
7. 点击 **确定（OK）**，按界面提示完成配置。

配置完成后，主机上的 GPU 就会以 vGPU 方式共享给虚拟机使用。

\---

## 5\. 部署 vGPU 授权服务器（FastAPI-DLS）

### 5.1 环境要求

* 一台 Linux 服务器，示例 IP：`192.168.48.95`
* 已安装 Docker 和 Docker Compose
* 开放 TCP `443` 端口
* 确保该服务器能被 ESXi / 虚拟机访问

### 5.2 方式一：Docker 命令行方式

```bash
docker run -d --restart=always \\
  -e DLS\_URL=192.168.48.95 \\
  -e DLS\_PORT=443 \\
  -p 443:443 \\
  makedie/fastapi-dls:latest
```

### 5.3 方式二：Docker Compose 方式

创建目录和编排文件：

```bash
mkdir -p /opt/docker/fastapi-dls/cert
```

在 `/opt/docker/fastapi-dls/docker-compose.yml` 中写入以下内容：

```yaml
version: '3.9'

x-dls-variables: \&dls-variables
  TZ: Asia/Shanghai
  DLS\_URL: 192.168.48.95
  DLS\_PORT: 443
  LEASE\_EXPIRE\_DAYS: 90
  DATABASE: sqlite:////app/database/db.sqlite
  DEBUG: false

services:
  dls:
    image: collinwebdesigns/fastapi-dls:latest
    restart: always
    environment:
      <<: \*dls-variables
    ports:
      - "443:443"
    volumes:
      - /opt/docker/fastapi-dls/cert:/app/cert
      - dls-db:/app/database
    logging:  # optional, for those who do not need logs
      driver: "json-file"
      options:
        max-file: 5
        max-size: 10m

volumes:
  dls-db:
```

启动服务：

```bash
cd /opt/docker/fastapi-dls
docker compose up -d
```

### 5.4 说明

* 上述两种方式使用了不同的镜像名：`makedie/fastapi-dls:latest` 和 `collinwebdesigns/fastapi-dls:latest`，请根据实际能拉取的镜像选择一种即可，参数含义相同。
* `DLS\_URL` 必须填写授权服务器的实际访问地址，即 `192.168.48.95`。
* `DLS\_PORT` 为服务端口，默认 `443`。
* `LEASE\_EXPIRE\_DAYS` 为授权租期，示例为 `90` 天。
* 证书目录 `/app/cert` 用于存放 TLS/HTTPS 证书，首次启动可自动生成自签名证书。

### 5.5 验证授权服务

服务启动后，在浏览器或命令行访问：

```bash
curl -k https://192.168.48.95/-/client-token
```

如果能获取到客户端配置 Token（`client\_configuration\_token` 相关内容），说明授权服务器工作正常。

也可以通过 Docker 查看容器状态：

```bash
docker ps
```

确保 `fastapi-dls` 容器处于 `Up` 状态。

\---

## 6\. 虚拟机使用 vGPU 的简要配置

完成以上步骤后，可在 vCenter 中为虚拟机添加 vGPU 设备：

1. 关闭虚拟机。
2. 编辑虚拟机设置。
3. 添加设备：**PCI 设备**（或其他设备中对应 NVIDIA vGPU 的设备）。
4. 选择需要的 vGPU Profile（例如 `GRID P4-1Q`、`A16-1Q` 等，以实际型号支持为准）。
5. 开机并在虚拟机内安装对应的 NVIDIA vGPU Guest 驱动。
6. 根据 NVIDIA vGPU 版本要求配置授权：

   * Linux 虚拟机通常需要从授权服务器获取 Client Configuration Token，放入 `/etc/nvidia/ClientConfigToken/` 目录，并重启 `nvidia-gridd` 服务。
   * Windows 虚拟机的配置方式以 NVIDIA vGPU 驱动官方说明为准。
7. 在虚拟机内执行 `nvidia-smi` 查看 vGPU 状态，确认授权和显存信息正常。

\---

## 7\. 驱动卸载

如果需要卸载 ESXi 上的 NVIDIA vGPU 驱动：

### 7.1 查看已安装的 VIB

```bash
esxcli software vib list
```

也可以过滤 NVIDIA 相关 VIB：

```bash
esxcli software vib list | grep -i nvidia
```

### 7.2 卸载指定 VIB

```bash
esxcli software vib remove -n <VIB名称>
```

将 `<VIB名称>` 替换为实际安装的 NVIDIA VIB 名称。卸载完成后建议重启 ESXi 主机。

\---

## 8\. 常用排障命令

|检查项|命令|
|-|-|
|查看 NVIDIA 内核模块|`vmkload\_mod -l \| grep nvidia`|
|查看 GPU 信息|`nvidia-smi`|
|查看已安装 VIB|`esxcli software vib list \| grep -i nvidia`|
|查看授权服务容器状态|`docker ps`|
|查看授权服务日志|`docker logs -f <容器名或ID>`|
|验证授权服务 Token 接口|`curl -k https://192.168.48.95/-/client-token`|

\---

