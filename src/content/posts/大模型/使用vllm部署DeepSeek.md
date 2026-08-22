---
title: 使用vllm部署DeepSeek
published: 2026-08-22
description: 使用vllm在Nvidia GPU上容器化部署DeepSeek,详解nvidia运行时切换、镜像拉取与tensor parallel等启动参数。
category: 大模型
tags:
  - vLLM
  - DeepSeek
  - 大模型部署
slug: deploy-deepseek-with-vllm
---

# 使用vllm部署DeepSeek

## Nvidia GPU

需要提前安装好：驱动，CUDA，Docker,NVidia-docker

1.1 切换nvidia runtime
`sudo nvidia-ctk runtime configure --runtime=docker`

1.2 重启docker
`sudo systemctl restart docker`

1.3 拉取镜像

`docker pull dockerpull.cn/vllm/vllm-openai`

1.4 创建容器

```bash
docker run -d \
  --gpus all \
  --ipc=host \
  --restart always \
  --name deepseek \
  --network host \
  -v /model/DeepSeek-R1-Distill-Qwen-32B:/model \
  dockerpull.cn/vllm/vllm-openai:v0.8.4 
```

1.5 启动服务

```bash
vllm serve /model/ \
  --host 0.0.0.0 \
  --port 8080 \
  --tensor-parallel-size 4 \
  --enable-chunked-prefill \
  --enforce-eager \
  --served-model-name deepseek \
  --trust-remote-code \
  --gpu-memory-utilization 0.9 \
  --cpu-offload-gb 0 \
  --max-model-len 16384 \
  --max-num-batched-tokens 2048 \
  --dtype bfloat16
```

1.6 常用启动服务参数介绍

```text
tensor-parallel-size 
并行数量，使用几块GPU就写几

enforce-eager 
是否强制开启 Pytorch 的 eager 模式，默认关闭，此时会额外使用 CUDA graph 做进一步加速，但会占用额外显存，并增加一些服务启动耗时。若启动报显存不足，可以尝试启动命令中添加--enforce-eager参数来节约显存占用，但是推理性能会稍有下降

served-model-name
启动的服务名称

gpu-memory-utilization
GPU使用显存比例，值越大占用显存越高

cpu-offload-gb
将模型加载到内存，显存足够就设置为0

max-model-len
模型最大输入长度，显存不够可适当调低

max-num-batched-tokens
每次迭代最大处理的 token 数，默认取2048和模型支持的上下文长度的较大值；若显存富余较多，且输入文本较长，可以适当调大此参数，以获得更好的吞吐性能。

api-key "1!Deshine"
兼容OPEN-API的密钥，有特殊符号需要加""

dtype
模型精度类型，可选值 ["auto", "float16", "bfloat16", "float32"]
```

2.1 创建自启动vllm服务容器

```bash
docker run -d \
  --gpus all \
  --ipc=host \
  --restart always \
  --name deepseek \
  --network host \
  -v /model/DeepSeek-R1-Distill-Qwen-32B:/model \
  dockerpull.cn/vllm/vllm-openai:latest \
  --model /model \
  --host 0.0.0.0 \
  --port 8080 \
  --tensor-parallel-size 4 \
  --enable-chunked-prefill \
  --enforce-eager \
  --served-model-name deepseek \
  --trust-remote-code \
  --gpu-memory-utilization 0.96 \
  --cpu-offload-gb 0 \
  --max-model-len 6000 \
  --max-num-batched-tokens 4096 \
  --dtype auto
```

3.1 请求验证

```bash
curl http://localhost:8080/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
    "model": "deepseek",   
    "messages": [
    {"role": "system", "content": "你是个友善的AI助手。"},
    {"role": "user", "content": "霸王龙是什么物种？" }
    ]}'
```

## AMD GPU

需要提前安装好：驱动、Rocm、docker。

**注意：MI210在Ubuntu 22.04下需要在 `/etc/default/grub` 下添加 `GRUB_CMDLINE_LINUX_DEFAULT="amd_iommu=off"`,否则只能使用单卡**

1.1 拉取镜像

`docker pull dockerpull.cn/rocm/vllm-dev:nightly`

1.2 创建容器

```bash
docker run -it \
    --ipc=host \
    --network=host \
    --privileged \
    --cap-add=CAP_SYS_ADMIN \
    --device=/dev/kfd \
    --device=/dev/dri \
    --device=/dev/mem \
    --group-add render \
    --cap-add=SYS_PTRACE \
    --security-opt seccomp=unconfined \
    -v /model/DeepSeek-R1-Distill-Llama-70B:/model \
   dockerpull.cn/rocm/vllm-dev:nightly
```

1.3 启动服务

```bash
vllm serve /model/ \
  --host 0.0.0.0 \
  --port 8080 \
  --tensor-parallel-size 8 \
  --served-model-name deepseek \
  --trust-remote-code \
  --gpu-memory-utilization 0.9 \
  --cpu-offload-gb 0 \
  --max-model-len 4096 \
  --max-num-batched-tokens 4096 \
  --dtype bfloat16
```

## HuaWei 300i Duo

前置条件：安装好NPU驱动，固件，Docker

1.1 修改模型配置文件：
`vim config.json`

将`bfloat16`修改为`float16`

1.2 修改模型目录权限：

`chmod -R 750 /model/DeepSeek-R1-Distill-Qwen-32B/`

1.3 创建容器

```bash
docker run -it -d --net=host --shm-size=1g \
    --name mindie \
    --device=/dev/davinci_manager:rwm \
    --device=/dev/hisi_hdc:rwm \
    --device=/dev/devmm_svm:rwm \
    --device=/dev/davinci0:rwm \
    --device=/dev/davinci1:rwm \
    -v /usr/local/Ascend/driver:/usr/local/Ascend/driver:ro \
    -v /usr/local/Ascend/firmware/:/usr/local/Ascend/firmware:ro \
    -v /usr/local/sbin:/usr/local/sbin:ro \
    -v /Model:/Model:ro \
    swr.cn-south-1.myhuaweicloud.com/ascendhub/mindie:2.3.0-300I-Duo-py311-openeuler24.03-lts bash
```

1.4 进入容器

`docker exec -it deepseek bash`

1.5 修改mindie服务配置文件
`vim /usr/local/Ascend/mindie/latest/mindie-service/conf/config.json`

要修改以下配置：

```text
"ipAddress" : "0.0.0.0",
#全0配置，方便局域网访问
"allowAllZeroIpListening" : true,
#允许全0监听
"httpsEnabled" : true,
#禁用https 

"npuDeviceIds" : [[0,1,2,3]],
#npu数量，按实际情况修改

"modelName" : "deepseek",
#模型名称
"modelWeightPath" : "/model",
#模型目录
"worldSize" : 4,
#和上面npu数量配置对应

"maxIterTimes" : 512,
#对话输出不完整，按实际情况调整数值
```

1.6 启动推理服务:

`cd /usr/local/Ascend/mindie/latest/mindie-service/bin`
`./mindieservice_daemon`

如果启动服务有报错，开启详细日志方便定位问题。

```bash
export MINDIE_LOG_TO_STDOUT="true"
export MINDIE_LOG_TO_STDOUT=1
```

1.7 验证推理服务：

以下模板所有兼容openai的推理服务都可以用

```bash
curl http://localhost:1025/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek",
    "messages": [
      {"role": "system", "content": "你是个友善的AI助手。"},
      {"role": "user", "content": "地球是什么时候诞生的？诞生以来到现在经历了哪些时期？发生过几次生物大灭绝？"}
    ],
    "max_tokens": 4096,
    "temperature": 0.8,
    "stream": false
  }'
```

注：2.0以上版本的镜像可能需要设置依赖库的环境变量，MINDIE的docker镜像自带了环境变量一键脚本

```bash
cd /usr/local/Ascend/mindie

source set_env.sh
```

## vllm纯cpu推理容器

1.1 拉取镜像
`docker pull public.ecr.aws/q9t5s3a7/vllm-cpu-release-repo:v0.8.5.post1`

1.2 创建自启动vllm服务容器

```bash
docker run -d \
  --privileged=true \
  --shm-size=4g \
  --ipc=host \
  --restart always \
  --name bge-m3 \
  --network host \
  -e VLLM_CPU_KVCACHE_SPACE=40 \
  -e VLLM_CPU_OMP_THREADS_BIND=21-25 \
  -v /model/bge-m3:/model/bge-m3 \
  public.ecr.aws/q9t5s3a7/vllm-cpu-release-repo:v0.8.5.post1 \
  --model /model/bge-m3 \
  --host 0.0.0.0 \
  --port 8082 \
  --served-model-name bge-m3 \
  --api-key 'abc123'

```

1.3 纯CPU容器参数解释

```text
VLLM_CPU_KVCACHE_SPACE=40
#设置KV缓存大小为40G,更多的缓存可以允许更多的并发。
VLLM_CPU_OMP_THREADS_BIND=0-12
#指定CPU 内核为0-12核
```
