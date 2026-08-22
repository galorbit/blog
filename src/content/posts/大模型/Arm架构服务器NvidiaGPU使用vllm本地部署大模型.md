---
title: Arm架构服务器NvidiaGPU使用vllm本地部署大模型
published: 2026-08-22
description: 在ARM64架构服务器上配合Nvidia GPU使用vllm本地部署大模型,涵盖依赖安装、Conda环境与PyTorch预编译包。
category: 大模型
tags:
  - vLLM
  - ARM架构
  - 源码编译
slug: arm-server-nvidia-gpu-vllm-deploy
---

# Arm架构服务器NvidiaGPU使用vllm本地部署大模型

## 系统信息

- 操作系统: **openEuler 24.03 SP2**
- 处理器架构: **ARM64 / aarch64**
- GPU: **NVIDIA GPU**
- 驱动: NVIDIA Driver **560.28.03**
- CUDA 版本: **12.6**
- glibc 版本: **2.38**

---

## 一、基础环境准备

### 1. 安装系统依赖库

```bash
sudo yum install -y gcc g++ cmake ninja-build.aarch64 numactl-devel.aarch64 git zlib-devel
```

> [!WARNING]
> 确保系统中的 glibc 版本为 **2.38**，否则可能与预构建的 PyTorch Wheel 不兼容。

### 2. 安装 Python 虚拟环境工具 Conda

下载并安装 Miniconda：

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-aarch64.sh
bash Miniconda3-latest-Linux-aarch64.sh -b -p /root/miniconda3
export PATH=/root/miniconda3/bin:$PATH
source /root/miniconda3/bin/activate
```

### 3. 创建隔离的 Python 环境

```bash
conda create -n vllm python=3.9
conda activate vllm
```

---

## 二、安装 PyTorch（CUDA 12.6）

推荐使用国内镜像加快下载速度（如华为云）

```bash
pip install torch-2.6.0.dev20250104+cu126-cp39-cp39-linux_aarch64.whl \
     -i https://mirrors.huaweicloud.com/repository/pypi/simple
```

验证安装是否成功：

```bash
python -c "import torch; print(torch.__version__); print(torch.cuda.is_available())"
```

---

## 三、源码构建 Triton 编译器

### 1. 获取指定版本代码

```bash
git clone https://github.com/triton-lang/triton.git
cd triton
git checkout v3.2.0
```

设置代理防止 GitHub 下载失败：

```bash
export HTTP_PROXY="http://192.168.48.101:7897"
export HTTPS_PROXY="http://192.168.48.101:7897"
```

### 2. 安装构建所需的依赖项

```bash
cd python
pip install -r requirements.txt
pip install wheel setuptools
```

### 3. 编译为 Wheel 文件并安装

```bash
python setup.py bdist_wheel
pip install dist/triton-3.2.0-cp39-cp39-linux_aarch64.whl
```

---

## 四、构建 VLLM 推理框架

### 1. 获取源代码并切换目录

```bash
git clone https://github.com/vllm-project/vllm.git
cd vllm
```

### 2. 使用现有 Torch（可选）

```bash
python use_existing_torch.py
```

### 3. 安装构建依赖

```bash
pip install numpy -i https://mirrors.huaweicloud.com/repository/pypi/simple
pip install -r requirements/build.txt -i https://mirrors.huaweicloud.com/repository/pypi/simple
```

### 4. 控制并行编译参数并打包

```bash
export MAX_JOBS=60
python setup.py bdist_wheel
```

### 5. 安装 VLLM 打包文件

```bash
pip install dist/vllm-0.9.2rc2.dev6+g25950dca9.d20250705.cu126-cp39-cp39-linux_aarch64.whl
```

---

## 五、启动模型 API 服务配置

### 1. 创建启动脚本目录

```bash
mkdir -p /opt/vllm
nano /opt/vllm/start_vllm.sh
```

内容如下，请根据具体路径和服务需求修改：

```bash
#!/bin/bash

# 设置 conda 环境路径（请根据你的实际路径修改）
export PATH=/root/miniconda3/bin:$PATH
source /root/miniconda3/bin/activate

# 替换为你实际使用的环境名称
conda activate vllm

# 启动服务
vllm serve /home/model/DeepSeek-R1-Distill-Qwen-7B \
  --host 0.0.0.0 \
  --port 8080 \
  --tensor-parallel-size 1 \
  --enable-chunked-prefill \
  --enforce-eager \
  --served-model-name deepseek \
  --trust-remote-code \
  --gpu-memory-utilization 0.9 \
  --cpu-offload-gb 0 \
  --max-model-len 16384 \
  --max-num-batched-tokens 2048 \
  --dtype bfloat16 \
  --api-key '1!Deshine'
```

保存后赋予执行权限：

```bash
chmod +x /opt/vllm/start_vllm.sh
```

### 2. 配置 systemd 管理方式

创建 systemd 配置文件 `/etc/systemd/system/vllm.service`：

```bash
nano /etc/systemd/system/vllm.service
```

写入如下内容：

```ini
[Service]
User=root
WorkingDirectory=/opt/vllm
Environment="PATH=/root/miniconda3/bin:/root/miniconda3/envs/vllm/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin"
ExecStart=/bin/bash /opt/vllm/start_vllm.sh
ExecStop=/bin/kill -SIGTERM $MAINPID
Restart=always
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=vllm
```

### 3. 启动后台服务

```bash
systemctl daemon-reload
systemctl start vllm
systemctl status vllm
```

查看日志可使用 syslog 或 journalctl：

```bash
journalctl -u vllm -f
```

---

> [!TIP]
> 如需开机自启动，运行：
>
> ```bash
> systemctl enable vllm
> ```
