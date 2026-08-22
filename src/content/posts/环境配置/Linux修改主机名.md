---
title: Linux修改主机名
published: 2026-08-22
description: 用hostnamectl永久或临时修改Linux主机名,并解决修改后sudo报unable to resolve host的报错问题。
category: 环境配置
tags:
  - 主机名
  - 系统配置
slug: linux-change-hostname
---

# Linux修改主机名

`hostnamectl` #查看当前主机名

## 永久修改主机名

1. `hostnamectl set-hostname greetingsyi` #将主机名修改为greetingsyi
2. 编辑hostname文件，将主机名写入即可。

```shell
vim /etc/hostname
debian
```

以上两种方式都可永久修改主机名

## 临时修改主机名

`hostname debian` #修改主机名为debian,重新登陆后恢复原来的主机名

## 修改主机名后报错

修改主机名后使用sudo提示`sudo unable to resolve host test : Name or service not known`

使用`su - root`切换到root用户

`vim /etc/hosts`

添加或修改掉原来的主机名。

```shell
127.0.0.1 localhost
127.0.0.1 debian
```

保存，切换到普通用户,使用`sudo`命令验证就正常了。
