---
title: 昇腾910B部署Qwen3.6-35B-A3B并开启MTP
image: "api"
published: 2026-08-22
description: 在昇腾910B八卡服务器上使用vllm-ascend部署Qwen3.6-35B-A3B，开启MTP投机解码，提供OpenAI兼容接口。
category: 大模型
tags:
  - 昇腾910B
  - vLLM
  - MTP
slug: ascend-910b-deploy-qwen3-6-35b-a3b-with-mtp
---

# 昇腾 910B 部署 Qwen3.6-35B-A3B(开启 MTP)

## 背景与目标

- 在昇腾 910B(8 卡)服务器上,使用 vLLM(Ascend 版)部署 Qwen3.6-35B-A3B 模型。
- 开启 **MTP(Multi-Token Prediction,多 token 预测)** 投机解码,加快推理速度。
- 服务对外提供 OpenAI 兼容的 HTTP 接口,监听 `8011` 端口。
- 以下所有命令都在**目标服务器**的 shell 中执行,全程同一台机器。

## 整体流程

1. 检查环境(NPU、Python)
2. 安装 vllm-ascend
3. 准备模型文件
4. 设置环境变量
5. 启动服务(开启 MTP)
6. 查看日志并验证服务
7. 停止服务

---

## 1. 检查环境

先确认 8 张 910B 卡都能被识别:

```bash
npu-smi info
```

如果提示命令不存在,说明昇腾驱动或 `npu-smi` 未安装,需要先搭建昇腾基础环境。

再确认 Python 版本(vLLM 需要 Python 3.8 及以上):

```bash
python3 --version
```

## 2. 安装 vllm-ascend

如果目标机器还没安装 vLLM 的昇腾适配版,先安装:

```bash
pip install vllm-ascend
```

> 提示:如果系统默认 `pip` 不对应 `python3`,请改用 `pip3 install vllm-ascend`,或在 conda 对应环境中安装。

## 3. 准备模型文件

把 Qwen3.6-35B-A3B 模型文件放到 `/data/modles/Qwen3.6-35B-A3B`(此路径与启动命令保持一致):

```bash
ls /data/modles/Qwen3.6-35B-A3B
```

确认目录下包含 `config.json`、`tokenizer.json` 等模型文件后再继续。若目录不存在,需先创建并放入模型:

```bash
mkdir -p /data/modles/Qwen3.6-35B-A3B
```

## 4. 设置环境变量

启动前先执行以下命令设置环境变量。这些变量只在**当前 shell** 生效,换终端或重启后需要重新执行。

```bash
export PYTORCH_NPU_ALLOC_CONF="expandable_segments:True"
export HCCL_OP_EXPANSION_MODE="AIV"
export HCCL_BUFFSIZE=4096
export OMP_NUM_THREADS=2
export TASK_QUEUE_ENABLE=1
export HCCL_P2P_LEVEL=5
export VLLM_NPU_USE_FLASH_ATTN=1
```

各变量的作用:

| 变量 | 作用 |
| --- | --- |
| `PYTORCH_NPU_ALLOC_CONF` | 启用 expandable_segments 内存分配,减少显存碎片 |
| `HCCL_OP_EXPANSION_MODE` | HCCL 算子扩展模式设为 AIV |
| `HCCL_BUFFSIZE` | HCCL 通信缓冲区大小(MB) |
| `OMP_NUM_THREADS` | OpenMP 线程数 |
| `TASK_QUEUE_ENABLE` | 开启任务队列 |
| `HCCL_P2P_LEVEL` | 设置 P2P 通信级别 |
| `VLLM_NPU_USE_FLASH_ATTN` | 使用昇腾 Flash Attention 加速 |

## 5. 启动服务(开启 MTP)

关键参数说明:

- `--tensor-parallel-size 8`:使用全部 8 张卡做张量并行。
- `--enable-expert-parallel`:开启专家并行(该模型是 MoE 架构)。
- `--max-model-len 131072`:最大上下文长度 128K。
- `--speculative_config`:值为 `{"method": "mtp", ...}`,即启用 MTP 投机解码。
- `--served-model-name "qwen3.6"`:对外暴露的模型名,后续调用接口时使用。

在**同一个 shell** 中,先进入专门的工作目录再启动(日志会写到当前目录):

```bash
mkdir -p /opt/qwen36
```

```bash
cd /opt/qwen36
```

执行启动命令:

```bash
nohup vllm serve /data/modles/Qwen3.6-35B-A3B \
    --served-model-name "qwen3.6" \
    --host 0.0.0.0 \
    --port 8011 \
    --tensor-parallel-size 8 \
    --data-parallel-size 1 \
    --enable-expert-parallel \
    --trust-remote-code \
    --max-model-len 131072 \
    --max-num-batched-tokens 2048 \
    --max-num-seqs 64 \
    --gpu-memory-utilization 0.94 \
    --enable-prefix-caching \
    --disable-log-stats \
    --async-scheduling \
    --allowed-local-media-path / \
    --mm-processor-cache-gb 0 \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_coder \
    --default-chat-template-kwargs '{"enable_thinking":false}' \
    --speculative_config '{"method": "mtp", "num_speculative_tokens": 1, "enforce_eager": true}' \
    --compilation-config '{"cudagraph_mode":"FULL_DECODE_ONLY"}' \
    --additional-config '{"enable_cpu_binding":true, "multistream_overlap_shared_expert": true, "pipeline_parallel_size": 1}' \
    >> vllm.log 2>&1 &
```

命令行为说明:

- `nohup ... &`:让服务在后台运行,关闭终端也不会退出。
- `>> vllm.log 2>&1`:把标准输出和错误输出都写入当前目录的 `vllm.log`,方便排查问题。

## 6. 查看日志并验证服务

实时查看启动日志:

```bash
tail -f vllm.log
```

看到类似 `Application startup complete`(或 `Uvicorn running on ...`)的输出,说明启动成功。按 `Ctrl + C` 退出日志查看。

查询已加载的模型列表:

```bash
curl http://localhost:8011/v1/models
```

返回包含 `qwen3.6` 的 JSON,说明接口正常。

再做一次真实对话测试:

```bash
curl http://localhost:8011/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
      "model": "qwen3.6",
      "messages": [{"role": "user", "content": "你好,请介绍一下你自己。"}]
    }'
```

## 7. 停止服务

后台启动的服务用进程名终止:

```bash
pkill -f "vllm serve"
```

如果上面命令没有生效,先查找进程号再终止:

```bash
ps -ef | grep vllm
```

```bash
kill -9 <PID>
```

## 常见问题

- **端口 8011 被占用**:用 `netstat -tlnp | grep 8011` 查看占用进程,停掉后再启动。
- **模型路径不存在**:检查 `/data/modles/Qwen3.6-35B-A3B` 是否存在、目录内容是否完整。
- **看不到 8 张卡**:检查 `npu-smi info` 输出,确认驱动正常,且环境变量 `ASCEND_RT_VISIBLE_DEVICES` 未被设置为更小的值。
- **日志没有输出**:确认是否在同一个 shell 中先执行了第 4 步的环境变量。
