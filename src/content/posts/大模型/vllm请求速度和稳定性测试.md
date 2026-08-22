---
title: vllm请求速度和稳定性测试
image: "api"
published: 2026-08-22
description: 使用evalscope perf对vllm推理服务做并发请求与单请求速度测试,评估服务吞吐与稳定性。
category: 大模型
tags:
  - vLLM
  - 性能测试
slug: vllm-speed-stability-test
---

# vllm请求速度和稳定性测试

## 并发请求测试

```bash
evalscope perf \
    --url "http://127.0.0.1:8080/v1/chat/completions" \
    --parallel 1 \
    --model deepseek \
    --number 1 \
    --api openai \
    --dataset openqa \
    --stream

#parallel 并发数
#number 请求数量
```

## 单个请求速度测试

```bash
evalscope perf \
 --parallel 1 \
 --url http://127.0.0.1:8080/v1/completions \
 --model deepseek \
 --log-every-n-query 5 \
 --connect-timeout 6000 \
 --read-timeout 6000 \
 --max-tokens 2048 \
 --min-tokens 2048 \
 --api openai \
 --dataset speed_benchmark \
 --debug
```
