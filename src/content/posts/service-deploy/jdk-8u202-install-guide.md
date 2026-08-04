---
title: "JDK 8u202 免费版本下载与安装配置指南"
published: 2026-08-04
description: "本文介绍 Oracle JDK 8u202 免费版本的授权政策、下载方式及在 Linux 系统下的环境变量配置与验证步骤。"
category: "服务部署"
tags:
  - "JDK"
  - "Java"
  - "环境配置"
  - "Linux"
  - "软件安装"
---

# JDK 8u202 免费版本下载与安装配置指南

## 下载安装包

从 2019 年 1 月份开始，Oracle JDK 开始对 Java SE 8 之后的版本开始进行商用收费，确切的说是 8u201/202 之后的版本。如果你用 Java 开发的功能如果是用作商业用途的，如果还不想花钱购买的话，能免费使用的最新版本是 8u201/202。当然如果是个人客户端或者个人开发者可以免费试用 Oracle JDK 所有的版本。

具体如下：

- JDK 8 之前版本，仍然免费。
- JDK 8 8u202 之前免费，包括 8u202，从 8u211 版本开始收费。
- JDK 9、JDK 10，全版本免费。
- JDK 11，11.0.2 前免费，包括 11.0.2。从 11.0.3 版本开始商用收费。
- JDK 12、JDK 13、JDK 14、JDK 15、JDK 16，全版本商用收费。
- JDK 17、JDK 18、JDK 19、JDK 20，全版本（二进制版本）免费。

也就是说：

## 免费版本

Java 的免费版本包括以下几个版本：

4、5、6、7、8（update 211 以前）、9、10、17

这些版本都可以供用户自由下载和使用，无需支付任何费用。用户不仅可以使用 Java 的基本功能，还可以无限制地发布和分发自己的应用程序。

## 付费版本

Java 的付费版本包括以下几个版本：

8（update 211 以后）、11～16

## 安装 JDK 8

使用最后一个免费版本 8u202，下载安装包 `jdk-8u202-linux-x64.tar.gz`。

创建安装目录：`mkdir /usr/local/java8`

解压文件到创建的目录：`tar -xvf /home/jdk-8u202-linux-x64.tar.gz -C /usr/local/java8`

解压后的目录结构为：`/usr/local/java8/jdk1.8.0_202`

设置全局环境变量，编辑 `nano /etc/profile`：

在文件最后添加以下内容：

```bash
# jdk 1.8
export JAVA_HOME=/usr/local/java8/jdk1.8.0_202
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```

使全局变量生效：`source /etc/profile`

验证是否安装完成：`java -version`

如果输出 java 版本号和编译信息即安装完成。
