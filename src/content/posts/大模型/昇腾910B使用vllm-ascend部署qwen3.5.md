---
title: 昇腾910B使用vllm-ascend部署qwen3.5
image: "api"
published: 2026-08-22
description: 在两张昇腾910B NPU上使用vllm-ascend容器部署Qwen3.5-35B,直通davinci设备并配置共享内存等参数。
category: 大模型
tags:
  - 昇腾
  - vLLM
  - 大模型部署
slug: ascend-910b-vllm-ascend-deploy-qwen35
---

# 昇腾910B 使用 vllm-ascend 部署 Qwen3.5-35B

> 基于 2 张 Ascend 910B NPU，通过 vLLM-Ascend 容器化部署 Qwen3.5-35B 推理服务。
>
> 镜像：`quay.io/ascend/vllm-ascend:v0.19.1rc1-310p-openeuler`

---

## 一、环境检查

```bash
# 检查 NPU 驱动版本和芯片状态
npu-smi info
# 预期输出：能看到 2 张 Ascend 910B，状态均为 OK

# 检查驱动设备节点
ls /dev/davinci*
# 输出示例：/dev/davinci0  /dev/davinci1  /dev/davinci_manager  /dev/devmm_svm
```

---

## 二、拉取镜像

```bash
docker pull quay.io/ascend/vllm-ascend:v0.19.1rc1-310p-openeuler
```

---

## 三、方式一：docker run（手动启动）

适合调试和临时使用。

### 3.1 创建并进入容器

```bash
docker run -it \                                    # 交互式终端
  --name qwen3.5-35b \                              # 容器名称
  --restart always \                                # 宕机后自动重启
  --shm-size 1g \                                   # 共享内存（建议调大为 32g~64g）
  --network host \                                  # 使用宿主机网络
  --device /dev/davinci0:/dev/davinci0 \            # NPU 设备 0
  --device /dev/davinci1:/dev/davinci1 \            # NPU 设备 1
  --device /dev/davinci_manager:/dev/davinci_manager \
  --device /dev/devmm_svm:/dev/devmm_svm \
  --device /dev/hisi_hdc:/dev/hisi_hdc \
  -v /usr/local/dcmi:/usr/local/dcmi \
  -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi \
  -v /usr/local/Ascend/driver/lib64/:/usr/local/Ascend/driver/lib64/ \
  -v /usr/local/Ascend/driver/version.info:/usr/local/Ascend/driver/version.info \
  -v /etc/ascend_install.info:/etc/ascend_install.info \
  -v /data/qwen3.5-35b:/model \                    # 模型文件挂载
  quay.io/ascend/vllm-ascend:v0.19.1rc1-310p-openeuler \
  /bin/bash
```

### 3.2 在容器内手动启动 vLLM 服务

```bash
vllm serve /model \
  --tensor-parallel-size 2 \      # 2 卡张量并行
  --enforce-eager \               # 强制 eager 模式（降低显存占用）
  --dtype float16 \               # float16 精度
  --served-model-name qwen3.5-35b \   # API 模型名称
  --host 0.0.0.0 \                # 监听所有地址
  --port 8000 \                   # API 端口
  --max-model-len 12000           # 最大上下文长度
```

> 退出容器不停止服务：`Ctrl+P+Q`（detach），或另开终端用 `docker exec` 进入。

---

## 四、方式二：docker compose（全自动部署）

适合生产环境，容器创建、开机自启、服务启动全部自动化，无需人工介入。

### 4.1 编写 docker-compose.yml

在宿主机任意目录创建 `docker-compose.yml`：

```yaml
version: "3.9"

services:
  vllm-ascend:
    image: quay.io/ascend/vllm-ascend:v0.19.1rc1-310p-openeuler
    container_name: qwen3.5-35b
    restart: always                              # 开机自启 + 异常重启
    network_mode: host                           # 使用宿主机网络
    shm_size: "1g"                               # 共享内存（建议 32g~64g）
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
      - /data/qwen3.5-35b:/model                 # 模型文件挂载
    command: >
      vllm serve /model
      --tensor-parallel-size 2
      --enforce-eager
      --dtype float16
      --served-model-name qwen3.5-35b
      --host 0.0.0.0
      --port 8000
      --max-model-len 12000
```

### 4.2 启动服务

```bash
# 后台启动（自动完成：拉取镜像 → 创建容器 → 启动服务）
docker compose up -d

# 查看启动日志
docker compose logs -f
```

### 4.3 管理服务

```bash
docker compose up -d       # 创建并后台启动
docker compose down        # 停止并删除容器
docker compose stop        # 停止容器（不删除）
docker compose start       # 启动已停止的容器
docker compose restart     # 重启容器
docker compose logs -f     # 实时查看日志
```

> **开机自启说明：**  `restart: always` 确保容器随 Docker 守护进程自动启动，配合宿主机侧 Docker 服务开机自启即可实现断电恢复后自动拉起推理服务。宿主机配置：
>
> ```bash
> systemctl enable docker      # 适用于 CentOS / openEuler
> ```

---

## 五、测试 API 服务

```bash
curl http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
    "model": "qwen3.5-35b",                        # 必须与 --served-model-name 一致
    "messages": [
      {"role": "system", "content": "你是个友善的AI助手。"},
      {"role": "user", "content": "详细解释一下大模型的eager模式和graph模式是什么？有什么区别？"}
    ]
  }'
```

### 流式测试

```bash
curl -N http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
    "model": "qwen3.5-35b",
    "stream": true,
    "messages": [
      {"role": "user", "content": "用一句话介绍你自己。"}
    ]
  }'
```

---

## 六、容器管理命令速查

```bash
docker ps -a | grep qwen3.5-35b    # 查看容器状态
docker logs -f qwen3.5-35b         # 实时查看日志（docker run 方式）
docker exec -it qwen3.5-35b bash   # 进入运行中的容器
docker stop qwen3.5-35b            # 停止容器
docker start qwen3.5-35b           # 启动已停止的容器
docker compose up -d               # compose 方式启动
docker compose down                # compose 方式停止并删除
```

---

## 七、常见问题

### eager 与 graph 模式的区别

- **Eager 模式：**  按算子逐条执行，不编译计算图。显存占用低，适合快速验证和调试，推理延迟略高。
- **Graph 模式：**  编译优化整个计算图后执行。推理速度快，但编译阶段额外消耗显存，首次调用有预热时间。

### 共享内存不足

启动时报 `Bus error` 或 `unable to open shared memory file`，将 `shm_size` 增大到 32g~64g。

### NPU 实时监控

```bash
watch -n 1 npu-smi info
```

### tokenizer 兼容性

如果容器内 tokenizer 报错，升级 transformers：

```bash
pip install --upgrade transformers
```
