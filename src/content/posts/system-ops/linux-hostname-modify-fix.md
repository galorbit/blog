---
title: "Linux 主机名修改与 sudo 报错修复"
published: 2026-08-04
description: "介绍 Linux 系统中永久与临时修改主机名的方法，以及修改后解决 sudo 解析报错的操作步骤。"
category: "系统运维"
tags:
  - "Linux"
  - "主机名"
  - "hostnamectl"
  - "sudo"
  - "网络配置"
---

# Linux 主机名修改与 sudo 报错修复

## 永久修改主机名

1. `hostnamectl` # 查看当前主机名
2. `hostnamectl set-hostname greetingsyi` # 将主机名修改为 greetingsyi
3. 编辑 `/etc/hostname` 文件，将主机名写入即可：
   ```bash
   vim /etc/hostname
   greetingsyi
   ```

以上两种方式都可永久修改主机名。

## 临时修改主机名

`hostname greetingsyi` # 修改主机名为 greetingsyi，重新登录后恢复原来的主机名

## 修改主机名后报错

修改主机名后使用 `sudo` 提示如下错误：
> `sudo: unable to resolve host test : Name or service not known`

使用 `su - root` 切换到 root 用户

执行 `vim /etc/hosts`

添加或修改掉原来的主机名：
```text
127.0.0.1 localhost
127.0.0.1 greetingsyi
```

保存，切换回普通用户，使用 `sudo` 命令验证就正常了。
