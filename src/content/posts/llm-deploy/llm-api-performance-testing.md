---
title: "大模型接口并发与单请求性能测试"
published: 2026-08-04
description: "使用 EvalScope 工具对大模型 API 进行并发请求与单请求速度的基准测试命令及参数说明。"
category: "大模型部署"
tags:
  - "EvalScope"
  - "大模型"
  - "性能测试"
  - "API接口"
  - "并发测试"
---

# 大模型接口并发与单请求性能测试

### 并发请求测试

```bash
evalscope perf \
    --url "http://127.0.0.1:8080/v1/chat/completions" \
    --parallel 1 \
    --model deepseek \
    --number 1 \
    --api openai \
    --dataset openqa \
    --stream

# parallel 并发数
# number 请求数量
```

### 单个请求速度测试

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
