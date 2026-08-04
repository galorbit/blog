---
title: "Ubuntu 系统 GCC 多版本安装与切换指南"
published: 2026-08-04
description: "介绍在 Ubuntu 系统中通过官方源及 PPA 安装 GCC 多版本，并使用 update-alternatives 进行优先级配置与切换的完整指南。"
category: "系统运维"
tags:
  - "GCC"
  - "Ubuntu"
  - "多版本管理"
  - "update-alternatives"
  - "编译环境"
---

# Ubuntu 系统 GCC 多版本安装与切换指南

## 从官方仓库安装

安装 GCC：

`sudo apt install gcc g++`

或安装开发工具包：

`sudo apt install build-essential`

## 从 Ubuntu Toolchain PPA 安装

添加 PPA 源：

`sudo add-apt-repository ppa:ubuntu-toolchain-r/ppa -y`

更新软件源：

`sudo apt update`

Ubuntu Toolchain PPA 提供了多个版本的 GCC，可以选择安装需要的 GCC 版本。

```bash
sudo apt install g++-12 gcc-12
sudo apt install g++-11 gcc-11
sudo apt install g++-10 gcc-10
sudo apt install g++-9 gcc-9
```

## 多版本切换和设置优先级

```bash
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 100 --slave /usr/bin/g++ g++ /usr/bin/g++-12 --slave /usr/bin/gcov gcov /usr/bin/gcov-12
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 80 --slave /usr/bin/g++ g++ /usr/bin/g++-11 --slave /usr/bin/gcov gcov /usr/bin/gcov-11
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-10 60 --slave /usr/bin/g++ g++ /usr/bin/g++-10 --slave /usr/bin/gcov gcov /usr/bin/gcov-10
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9 40 --slave /usr/bin/g++ g++ /usr/bin/g++-9 --slave /usr/bin/gcov gcov /usr/bin/gcov-9
```

系统默认使用优先级最高的版本，要使用哪个版本，就把哪个版本的优先级设置到最高。

如果设置了都 update-alternatives 手动管理，可以使用 `sudo update-alternatives --config gcc` 命令来直接切换优先级。
