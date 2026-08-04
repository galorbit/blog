---
title: "CentOS 7 卸载 MariaDB 并安装配置 MySQL 8.0"
published: 2026-08-04
description: "详细记录在 CentOS 7 系统中卸载 MariaDB、创建专用用户、通过 RPM Bundle 安装 MySQL 8.0 并完成初始化与密码修改的完整步骤。"
category: "服务部署"
tags:
  - "CentOS 7"
  - "MariaDB"
  - "MySQL 8.0"
  - "RPM 安装"
  - "数据库配置"
---

# CentOS 7 卸载 MariaDB 并安装配置 MySQL 8.0

MariaDB 和 MySQL 冲突，需要先卸载 MariaDB。

## 卸载 MariaDB 相关

检查 MariaDB

`rpm -qa | grep mariadb`

卸载（CentOS 7 只有一个 `mariadb-libs`）

`yum remove mariadb-libs`

## 创建 MySQL 用户

创建 MySQL 用户并设置密码

```shell
useradd mysql
passwd mysql
```

## 安装 MySQL

从官网下载需要安装的版本，下载 bundle 版本。

此处用 `mysql-8.0.22-1.el7.x86_64.rpm-bundle.tar` 演示

解压到任意目录

`tar -xvf mysql-8.0.22-1.el7.x86_64.rpm-bundle.tar`

按顺序安装以下包

```shell
rpm -ivh mysql-community-common-8.0.22-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-plugins-8.0.22-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-8.0.22-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-8.0.22-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-8.0.22-1.el7.x86_64.rpm
# 更新版本的 MySQL 可能还需要其他包，根据提示缺少什么就安装什么包
```

### 初始化 MySQL

`mysqld --initialize --console`

修改 MySQL 所有者为 MySQL 用户和用户组

`chown -R mysql:mysql /var/lib/mysql`

设置开机自启

`systemctl enable mysqld`

启动 MySQL

`systemctl start mysqld`

### 修改 MySQL 密码

查看初始化 root 密码

`grep 'temporary password' /var/log/mysqld.log`

输出结果即为临时密码。

登录 MySQL

`mysql -u root -p`

修改密码

`ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourPasswd';`
