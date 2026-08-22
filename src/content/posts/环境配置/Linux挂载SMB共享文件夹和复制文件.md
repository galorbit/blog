---
title: Linux挂载SMB共享文件夹和复制文件
published: 2026-08-22
description: 用cifs-utils挂载Windows或Linux的SMB共享文件夹,再用rsync把模型等大文件复制到本地。
category: 环境配置
tags:
  - SMB
  - 文件共享
  - 挂载
slug: linux-mount-smb-share-copy-files
---

# 挂载共享文件夹和复制模型

> 模型下载到了文件服务器，需要挂载到本地目录，并复制到需要部署的服务器

## 挂载

安装cifs-utils

```text
apt-get install cifs-utils
#ubuntu debian
yum install cifs-utils
#Centos
```

创建挂载目录
`mkdir /pc`

挂载

```text
#windows共享挂载
mount -t cifs -o username=Administrator,password='password' //192.168.48.201/fileserver/deepseek /pc
# username windows用户名
# password windows密码
# //192.168.48.100 windows主机IP
# M-computer 共享名，右键文件夹-属性-共享 可以看到
# /Data windows 共享文件夹路径
# /pc 挂载到本地的目录

#Linux smb共享挂载
mount -t cifs -o username=user,password='password',vers=3.0 //192.168.48.250/FileServer /FileServer
```

## 复制

创建模型存放目录
`mkdir /model`

进入共享文件夹挂载目录

`cd /pc`

查看需要复制的模型目录

```text
[root@local pc]# cd gguf/DeepSeek-R1-Distill-Llama-70B/
[root@local DeepSeek-R1-Distill-Llama-70B]# ll
total 41523825
-rwxr-xr-x. 1 root root 42520395616 Feb 14 18:59 DeepSeek-R1-Distill-Llama-70B-Q4_K_M.gguf
-rwxr-xr-x. 1 root root         626 Feb 20 18:12 Modelfile
[root@local DeepSeek-R1-Distill-Llama-70B]# pwd
/pc/gguf/DeepSeek-R1-Distill-Llama-70B
[root@local DeepSeek-R1-Distill-Llama-70B]#

```

复制模型到/model

`rsync -vr /pc/gguf/DeepSeek-R1-Distill-Llama-70B /model --progress`

使用rsync命令来复制大文件，如果没有rsync，使用apt 或dnf来安装
