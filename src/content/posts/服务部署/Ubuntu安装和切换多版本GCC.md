---
title: Ubuntu安装和切换多版本GCC
image: "api"
published: 2026-08-22
description: Ubuntu安装GCC 9-12等多版本编译器,用update-alternatives配置切换与优先级,满足不同编译需求。
category: 服务部署
tags:
  - GCC
  - 编译工具
slug: ubuntu-install-switch-gcc-versions
---

# Ubuntu安装和切换多版本GCC

## 从官方仓库安装

安装GCC

`sudo apt install gcc g++`

或安装开发工具包

`sudo apt install build-essential`

## 从Ubuntu Toolchain PPA安装

添加PPA源

`sudo add-apt-repository ppa:ubuntu-toolchain-r/ppa -y`

更新软件源

`sudo apt update`

Ubuntu Toolchain PPA 提供了多个版本的 GCC，可以选择安装需要的 GCC 版本。

```shell
sudo apt install g++-12 gcc-12
sudo apt install g++-11 gcc-11
sudo apt install g++-10 gcc-10
sudo apt install g++-9 gcc-9
```

## 多版本切换和设置优先级

```shell
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 100 --slave /usr/bin/g++ g++ /usr/bin/g++-12 --slave /usr/bin/gcov gcov /usr/bin/gcov-12
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 80 --slave /usr/bin/g++ g++ /usr/bin/g++-11 --slave /usr/bin/gcov gcov /usr/bin/gcov-11
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-10 60 --slave /usr/bin/g++ g++ /usr/bin/g++-10 --slave /usr/bin/gcov gcov /usr/bin/gcov-10
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9 40 --slave /usr/bin/g++ g++ /usr/bin/g++-9 --slave /usr/bin/gcov gcov /usr/bin/gcov-9
```

系统默认使用优先级最高的版本，要使用哪个版本，就把哪个版本的优先级设置到最高。

如果设置了都update-alternatives手动管理，可以使用`sudo update-alternatives --config gcc`命令来直接切换优先级。
