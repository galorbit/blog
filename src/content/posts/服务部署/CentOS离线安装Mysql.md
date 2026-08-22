---
title: CentOS离线安装Mysql
image: "api"
published: 2026-08-22
description: CentOS离线安装MySQL 8.0,先卸载冲突的mariadb,再按顺序rpm安装bundle包并完成初始化。
category: 服务部署
tags:
  - MySQL
  - 数据库
  - 离线安装
slug: centos-offline-install-mysql
---

# CentOS离线安装Mysql

## 卸载mariadb相关

mariadb和mysql冲突，需要先卸载mariadb。

检查mariadb

`rpm -qa|grep mariadb`

卸载
Centos 7只有一个`mariadb-libs`

`yum remove mariadb-libs`

## 创建mysql用户

创建mysql用户并设置密码

```shell
useradd mysql
passwd mysql
```

## 安装Mysql

从官网下载需要安装的版本，下载bundle版本。

此处用`mysql-8.0.22-1.el7.x86_64.rpm-bundle.tar`演示

解压到任意目录

`tar -xvf mysql-8.0.22-1.el7.x86_64.rpm-bundle.tar`

按顺序安装以下包

```shell
rpm -ivh mysql-community-common-8.0.22-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-plugins-8.0.22-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-8.0.22-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-8.0.22-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-8.0.22-1.el7.x86_64.rpm
# 更新版本的mysql可能还需要其他包，根据提示缺少什么就安装什么包
```

### 初始化mysql

`mysqld --initialize --console`

修改mysql所有者为mysql用户和用户组

`chown -R mysql:mysql /var/lib/mysql`

设置开机自启

`systemctl enable mysqld`

启动mysql

`systemctl start mysqld`

### 修改mysql密码

查看初始化root密码

`grep 'temporary password' /var/log/mysqld.log`输出则为默认的密码

登录mysql

`mysql -u root -p`

修改密码

`ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourPasswd';`
