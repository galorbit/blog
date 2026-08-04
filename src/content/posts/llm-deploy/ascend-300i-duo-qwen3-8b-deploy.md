---
title: "基于Ascend 300I Duo的Qwen3-8B部署指南"
published: 2026-08-04
description: "基于Ascend 300I Duo硬件，使用vllm-ascend镜像通过Docker或Docker Compose部署Qwen3-8B模型的完整指南与参数说明。"
category: "大模型部署"
tags:
  - "Ascend 300I Duo"
  - "vllm-ascend"
  - "Qwen3-8B"
  - "Docker部署"
  - "NPU推理"
---

# 基于Ascend 300I Duo的Qwen3-8B部署指南

## 硬件环境

- **推理卡**: 华为 Ascend 300I Duo（双芯 310P，每卡 2 个 NPU）
- **NPU 设备**: `/dev/davinci0`, `/dev/davinci1`
- **显存**: 共享 96GB HBM（通过 devmm_svm 统一编址）
- **驱动**: Ascend Driver（CANN 对应版本）

## 前置条件

```bash
# 确认 NPU 设备存在
ls -la /dev/davinci*

# 确认驱动已加载
npu-smi info
lsmod | grep drv

# 确认 Docker 已安装
docker --version
```

## 1. Docker Run（交互式调试 / 首次验证）

### 1.1 进入容器 Bash 手动调试

```bash
docker run -it \
  --name vllm-ascend \
  --shm-size=96g \
  --network host \
  --device /dev/davinci0 \
  --device /dev/davinci1 \
  --device /dev/davinci_manager \
  --device /dev/devmm_svm \
  --device /dev/hisi_hdc \
  -v /usr/local/dcmi:/usr/local/dcmi \
  -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi \
  -v /usr/local/Ascend/driver/lib64/:/usr/local/Ascend/driver/lib64/ \
  -v /usr/local/Ascend/driver/version.info:/usr/local/Ascend/driver/version.info \
  -v /etc/ascend_install.info:/etc/ascend_install.info \
  -v /models:/models \
  quay.io/ascend/vllm-ascend:v0.21.0rc1-310p \
  /bin/bash
```

进入容器后可手动执行：

```bash
# 查看 NPU 可用性
npu-smi info

# 手动启动服务（用于临时调试）
vllm serve /models/Qwen3-8B-w8a8sc-310-vllm-tp2 \
  --host 0.0.0.0 \
  --port 8080 \
  --tensor-parallel-size 2 \
  --gpu-memory-utilization 0.90 \
  --max-num-seqs 32 \
  --served-model-name qwen3-8b \
  --dtype float16 \
  --additional-config '{"ascend_compilation_config": {"fuse_norm_quant": false}}' \
  --compilation-config '{"cudagraph_mode": "FULL_DECODE_ONLY", "cudagraph_capture_sizes": [1,4,8,16]}' \
  --quantization ascend \
  --max-model-len 16384 \
  --no-enable-prefix-caching \
  --load-format sharded-state
```

### 1.2 直接启动服务（生产 / 守护模式）

```bash
docker run -d \
  --name vllm-ascend \
  --restart always \
  --shm-size=64g \
  --network host \
  --device /dev/davinci0 \
  --device /dev/davinci1 \
  --device /dev/davinci_manager \
  --device /dev/devmm_svm \
  --device /dev/hisi_hdc \
  -v /usr/local/dcmi:/usr/local/dcmi \
  -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi \
  -v /usr/local/Ascend/driver/lib64/:/usr/local/Ascend/driver/lib64/ \
  -v /usr/local/Ascend/driver/version.info:/usr/local/Ascend/driver/version.info \
  -v /etc/ascend_install.info:/etc/ascend_install.info \
  -v /models:/models \
  quay.io/ascend/vllm-ascend:v0.21.0rc1-310p \
  vllm serve /models/Qwen3-8B-w8a8sc-310-vllm-tp2 \
    --host 0.0.0.0 \
    --port 8080 \
    --tensor-parallel-size 2 \
    --gpu-memory-utilization 0.90 \
    --max-num-seqs 32 \
    --served-model-name qwen3-8b \
    --dtype float16 \
    --additional-config '{"ascend_compilation_config": {"fuse_norm_quant": false}}' \
    --compilation-config '{"cudagraph_mode": "FULL_DECODE_ONLY", "cudagraph_capture_sizes": [1,4,8,16]}' \
    --quantization ascend \
    --max-model-len 16384 \
    --no-enable-prefix-caching \
    --load-format sharded-state
```

## 2. Docker Compose（推荐生产部署）

```yaml
version: '3.8'

services:
  vllm-ascend:
    image: quay.io/ascend/vllm-ascend:v0.21.0rc1-310p
    container_name: vllm-ascend
    restart: always
    network_mode: host
    shm_size: '96g'
    devices:
      - /dev/davinci0:/dev/davinci0
      - /dev/davinci1:/dev/davinci1
      - /dev/davinci_manager:/dev/davinci_manager
      - /dev/devmm_svm:/dev/devmm_svm
      - /dev/hisi_hdc:/dev/hisi_hdc
    volumes:
      - /usr/local/dcmi:/usr/local/dcmi
      - /usr/local/bin/npu-smi:/usr/local/bin/npu-smi
      - /usr/local/Ascend/driver/lib64/:/usr/local/Ascend/driver/lib64/
      - /usr/local/Ascend/driver/version.info:/usr/local/Ascend/driver/version.info
      - /etc/ascend_install.info:/etc/ascend_install.info
      - /models:/models
    environment:
      - ASCEND_RT_VISIBLE_DEVICES=0,1
    command: >
      vllm serve /models/Qwen3-8B-w8a8sc-310-vllm-tp2
      --host 0.0.0.0
      --port 8080
      --tensor-parallel-size 2
      --gpu-memory-utilization 0.90
      --max-num-seqs 32
      --served-model-name qwen3-8b
      --dtype float16
      --additional-config '{"ascend_compilation_config": {"fuse_norm_quant": false}}'
      --compilation-config '{"cudagraph_mode": "FULL_DECODE_ONLY", "cudagraph_capture_sizes": [1,4,8,16]}'
      --quantization ascend
      --max-model-len 16384
      --no-enable-prefix-caching
      --load-format sharded-state
```

**部署命令：**

```bash
# 启动
docker compose up -d

# 查看日志
docker logs -f vllm-ascend

# 停止
docker compose down
```

## 3. 常用操作

### 查看 NPU 状态

```bash
# 宿主机
npu-smi info

# 容器内（已挂载 npu-smi）
docker exec -it vllm-ascend npu-smi info
```

### 手动拉取 / 更新镜像

```bash
docker pull quay.io/ascend/vllm-ascend:v0.21.0rc1-310p
```

### 测试推理接口

```bash
# OpenAI 兼容 API
curl http://localhost:8080/v1/models

curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-8b",
    "messages": [{"role": "user", "content": "你好"}],
    "max_tokens": 256
  }'
```

### 查看显存占用

```bash
docker exec vllm-ascend npu-smi info -m
```

### 容器资源监控

```bash
docker stats vllm-ascend
```

## 4. 参数说明

| 参数 | 值 | 说明 |
| :--- | :--- | :--- |
| `--tensor-parallel-size` | 2 | 对应 300I Duo 双 NPU，张量并行度为 2 |
| `--gpu-memory-utilization` | 0.90 | HBM 利用率 90%，预留 10% 作为缓冲 |
| `--max-num-seqs` | 32 | 最大并发序列数，根据实际负载调整 |
| `--quantization` | ascend | Ascend 量化格式（w8a8 对称量化） |
| `--max-model-len` | 16384 | 最大上下文长度（Qwen3-8B 支持到 32K，16K 保守配置） |
| `--dtype` | float16 | 推理精度，Ascend 310P 推荐 fp16 |
| `--no-enable-prefix-caching` | - | 禁用前缀缓存，Ascend 上 prefix caching 可能不稳定 |
| `--load-format` | sharded-state | 分片权重加载，与 tensor-parallel 配合使用 |
| `--compilation-config` | cudagraph_mode=FULL_DECODE_ONLY | vllm-ascend 官方推荐编译优化配置，用于预热捕获解码图 |
| `--shm-size` | 64g | 共享内存大小，需 >= 模型权重体积以避免加载失败 |

## 5. 版本兼容矩阵

| 组件 | 版本 |
| :--- | :--- |
| vllm-ascend 镜像 | v0.21.0rc1-310p |
| CANN / Ascend Driver | 镜像内已预装，需与宿主机驱动兼容 |
| 模型 | Qwen3-8B-w8a8sc-310-vllm-tp2 |
| Docker | >= 20.10 |
| 宿主机 OS | openEuler 22.03 LTS / Ubuntu 22.04 / CentOS 7 |
