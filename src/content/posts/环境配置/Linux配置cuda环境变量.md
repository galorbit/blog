---
title: Linux配置cuda环境变量
published: 2026-08-22
description: 为手动安装的CUDA配置PATH和LD_LIBRARY_PATH环境变量,支持当前用户或全局生效。
category: 环境配置
tags:
  - CUDA
  - 环境变量
slug: linux-config-cuda-env
---

# Linux配置cuda环境变量

## Linux配置cuda环境变量

.run安装包手动安装cuda需要设置环境变量

按自己安装的版本设置

```shell
# cuda
export PATH=$PATH:/usr/local/cuda-12.8/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda-12.8/lib64
```

为当前用户设置环境变量：

`nano /home/当前用户/.bashrc`
添加在文件最后。

为所有用户添加:

新建文件`nano /etc/profile.d/cuda128.sh`
添加上面的环境变量，保存，并给文件添加可执行权限：`chmod +x /etc/profile.d/cuda128.sh`

使环境变量生效：

`source /home/当前用户/.bashrc`
或
`source /etc/profile.d/cuda128.sh`

使用`nvcc -V`验证当前cuda版本
