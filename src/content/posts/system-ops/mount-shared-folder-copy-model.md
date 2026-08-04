---
title: "挂载网络共享文件夹并复制大模型文件"
published: 2026-08-04
description: "本文介绍如何在Linux环境下挂载Windows或Linux共享文件夹，并使用rsync命令高效复制大模型文件。"
category: "系统运维"
tags:
  - "CIFS挂载"
  - "rsync同步"
  - "Linux命令"
  - "文件传输"
  - "大模型部署"
---

# 挂载网络共享文件夹并复制大模型文件

> 模型下载到了文件服务器，需要挂载到本地目录，并复制到需要部署的服务器

## 挂载

安装 `cifs-utils`：

```bash
apt-get install cifs-utils
# Ubuntu/Debian
yum install cifs-utils
# CentOS
```

创建挂载目录：
` mkdir /pc `

执行挂载命令：

```bash
# Windows共享挂载示例
mount -t cifs -o username=Administrator,password='1!Deshine' //192.168.48.201/fileserver/deepseek /pc
# username: Windows用户名
# password: Windows密码
# //192.168.48.201: Windows主机IP
# fileserver: 共享名（右键文件夹-属性-共享可查看）
# /deepseek: Windows共享文件夹路径
# /pc: 挂载到本地的目录

# Linux SMB共享挂载示例
mount -t cifs -o username=greetingsyi,password='Yiminghaopwd@',vers=3.0 //192.168.48.250/FileServer /FileServer
```

## 复制

创建模型存放目录：
` mkdir /model `

进入共享文件夹挂载目录：
` cd /pc `

查看需要复制的模型目录：

```bash
[root@local pc]# cd gguf/DeepSeek-R1-Distill-Llama-70B/
[root@local DeepSeek-R1-Distill-Llama-70B]# ll
total 41523825
-rwxr-xr-x. 1 root root 42520395616 Feb 14 18:59 DeepSeek-R1-Distill-Llama-70B-Q4_K_M.gguf
-rwxr-xr-x. 1 root root         626 Feb 20 18:12 Modelfile
[root@local DeepSeek-R1-Distill-Llama-70B]# pwd
/pc/gguf/DeepSeek-R1-Distill-Llama-70B
[root@local DeepSeek-R1-Distill-Llama-70B]#
```

复制模型到 `/model`：
` rsync -vr /pc/gguf/DeepSeek-R1-Distill-Llama-70B /model --progress `

使用 `rsync` 命令来复制大文件，如果没有 `rsync`，可使用 `apt` 或 `dnf` 进行安装。
