---
title: CentOS安装JDK
image: "api"
published: 2026-08-22
description: CentOS安装JDK,梳理Oracle JDK各版本商用收费规则,选用免费版8u202解压安装并配置环境变量。
category: 服务部署
tags:
  - JDK
  - Java
  - 软件安装
slug: centos-install-jdk
---

# CentOS安装JDK

## 下载安装包

从2019年1月份开始，Oracle JDK 开始对 Java SE 8 之后的版本开始进行商用收费，确切的说是 8u201/202 之后的版本。如果你用 Java 开发的功能如果是用作商业用途的，如果还不想花钱购买的话，能免费使用的最新版本是 8u201/202。当然如果是个人客户端或者个人开发者可以免费试用 Oracle JDK 所有的版本。

具体如下：

JDK8 之前版本，仍然免费。

JDK8 8u202之前免费，包括8u202，从 8u211版本开始收费。

JDK9、JDK10，全版本免费。

JDK11，11.0.2前免费，包括11.0.2. 从 11.0.3 版本开始商用收费。

JDK12、JDK13、JDK14、JDK15、JDK16，全版本商用收费。

JDK17、JDK18、JDK19、JDK20，全版本(二进制版本)免费。

也就是说

一、免费版本

Java的免费版本包括以下几个版本：

4 5 6 7 8(update 211以前) 9 10 17

这些版本都可以供用户自由下载和使用，无需支付任何费用。用户不仅可以使用Java的基本功能，还可以无限制地发布和分发自己的应用程序。

二、付费版本

Java的付费版本包括以下几个版本：

8(update 211以后)

11～16

## 安装JDK 8

使用最后一个免费版本8u202,下载安装包jdk-8u202-linux-x64.tar.gz

创建安装目录

`mkdir /usr/local/java8`

解压文件到创建的目录

`tar -xvf /home/jdk-8u202-linux-x64.tar.gz -C /usr/local/java8`

解压后的目录结构为

`/usr/local/java8/jdk1.8.0_202`

设置全局环境变量，编辑`nano /etc/profile`

在文件最后添加以下内容:

```shell
# jdk 1.8
export JAVA_HOME=/usr/local/java8/jdk1.8.0_202
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```

使全局变量生效`source /etc/profile`

验证是否安装完成

`java -version`

如果输出java 版本号和编译信息即安装完成。
