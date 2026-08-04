---
title: "Linux配置CUDA环境变量操作指南"
published: 2026-08-04
description: "介绍在Linux系统中手动安装CUDA后，如何为单用户或多用户配置、生效环境变量并验证版本的完整步骤。"
category: "GPU与驱动"
tags:
  - "CUDA"
  - "环境变量"
  - "Linux"
  - "驱动配置"
---

# Linux配置CUDA环境变量操作指南

通过 `.run` 安装包手动安装 CUDA 后，需要配置环境变量。请根据实际安装的版本号修改以下路径。

```shell
# CUDA 环境变量配置
export PATH=$PATH:/usr/local/cuda-12.8/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda-12.8/lib64
```

## 为当前用户配置

编辑当前用户的 shell 配置文件：
`nano /home/当前用户/.bashrc`

将上述环境变量添加到文件末尾。

## 为所有用户配置

新建配置文件：
`nano /etc/profile.d/cuda128.sh`

将上述环境变量写入文件并保存，随后赋予可执行权限：
`chmod +x /etc/profile.d/cuda128.sh`

## 使配置生效

根据配置范围选择对应命令执行：
- 当前用户：`source /home/当前用户/.bashrc`
- 所有用户：`source /etc/profile.d/cuda128.sh`

最后使用以下命令验证当前 CUDA 版本：
`nvcc -V`
