---
title: 昇腾300I-Duo部署Qwen3-30B-A3B
image: "api"
published: 2026-08-22
description: 在昇腾300I Duo上部署Qwen3-30B-A3B模型，先用docker run测试，再用docker compose实现生产部署与开机自启。
category: 大模型
tags:
  - 昇腾300I Duo
  - Qwen3-30B
  - Docker Compose
slug: ascend-300i-duo-deploy-qwen3-30b-a3b
---

# 昇腾 300I Duo 部署 Qwen3-30B-A3B(先 docker run 测试,再 docker compose 生产部署)

## 背景与目标

- 在昇腾 300I Duo 服务器上部署 Qwen3-30B-A3B-w8a8-QuaRot-310 模型,提供 OpenAI 兼容接口,监听 `8080` 端口。
- **先测试,后上生产**:第一阶段用 `docker run` 方式启动容器,手动执行环境变量和 `vllm serve`,快速验证模型与参数没问题;验证通过后,第二阶段把容器创建、环境变量、服务启动命令整合为一个 `docker-compose.yml`,并用 `restart: always` 实现**开机自启**,用于生产环境。
- 整个流程里,`docker run` 方式中手动执行的命令,与 `docker-compose.yml` 里的配置是**一一对应**的,测试时改哪几个参数,生产配置里就改哪几个。

## 整体流程

1. 准备模型文件
2. 方式一(docker run,先测试):启动容器 → 进入容器 → 设置环境变量 → 启动 vllm serve → 验证
3. 方式二(docker compose,测试通过后上生产):编写 docker-compose.yml → 启动 → 验证 → 维护 → 开机自启

---

## 1. 准备模型文件

在宿主机上确认模型已放到 `/models/Qwen3-30B-A3B-w8a8-QuaRot-310`:

```bash
ls /models/Qwen3-30B-A3B-w8a8-QuaRot-310
```

确认目录下包含 `config.json` 等模型文件。两种方式都会把这个目录挂载进容器,容器内路径同样是 `/models`。

---

# 方式一:用 docker run 测试

先不要考虑生产配置,用最简单的方式把服务跑起来,确认模型和参数没问题。

## 2.1 启动容器

`docker run` 一次性完成:拉取镜像、创建容器、映射 NPU 设备与驱动目录。`-d` 表示后台运行,`-it` 保留交互终端,命令末尾的 `bash` 让容器以 bash 启动,方便随后进入操作。

```bash
docker run -it -d \
  --name qwen3-30b \
  --net=host \
  --shm-size=64g \
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
  quay.io/ascend/vllm-ascend:v0.23.0rc1-310p bash
```

参数说明:

| 参数 | 作用 |
| --- | --- |
| `--name qwen3-30b` | 容器名称,后续管理用 |
| `--net=host` | 与宿主机共享网络,直接监听 8080 |
| `--shm-size=64g` | 共享内存 64GB,防止多卡推理内存不足 |
| `--device /dev/davinci*` 等 | 映射 NPU 设备文件 |
| `-v 宿主机路径:容器路径` | 挂载昇腾驱动目录与模型目录 |

## 2.2 进入容器

容器已在后台运行,进入容器拿到一个交互 shell:

```bash
docker exec -it qwen3-30b bash
```

## 2.3 设置环境变量

在容器内的 shell 中执行,指定本次服务使用 NPU 卡 `0,1`(共 2 卡):

```bash
export ASCEND_RT_VISIBLE_DEVICES=0,1
```

## 2.4 启动 vllm serve

- `--quantization ascend`:使用昇腾平台量化,模型为 w8a8 QuaRot 版本。
- `--tensor-parallel-size 2`:使用 2 张 NPU 并行。
- `--additional-config`、`--compilation-config`:昇腾编译与算子融合相关配置。

```bash
vllm serve /models/Qwen3-30B-A3B-w8a8-QuaRot-310 \
    --host 0.0.0.0 \
    --port 8080 \
    --tensor-parallel-size 2 \
    --gpu-memory-utilization 0.92 \
    --max-num-seqs 32 \
    --served-model-name qwen3-30b \
    --dtype float16 \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_coder \
    --additional-config '{"ascend_compilation_config": {"fuse_norm_quant": false,"enable_npu_graph_ex":false}}' \
    --compilation-config '{"cudagraph_mode": "FULL_DECODE_ONLY", "cudagraph_capture_sizes": [1,32]}' \
    --quantization ascend \
    --max-model-len 40960 \
    --enable-prefix-caching
```

命令会前台运行,日志直接打印在终端。看到类似 `Application startup complete` 的输出,说明模型加载成功。

## 2.5 验证服务

在**宿主机**上(另开一个终端)调用接口:

```bash
curl http://localhost:8080/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
    "model": "qwen3-30b",
    "messages": [
      {"role": "system", "content": "你是个友善的AI助手。"},
      {"role": "user", "content": "你好,你是谁?可以做什么?"}
    ]
  }'
```

返回包含回答内容的 JSON,说明 `docker run` 方式测试通过,可以进入方式二部署生产。

## 2.6 测试结束后清理

回到容器终端按 `Ctrl + C` 停止 `vllm serve`,然后在宿主机上删除测试容器:

```bash
docker rm -f qwen3-30b
```

---

# 方式二:测试通过后,用 docker compose 部署生产

把方式一里手动执行的「容器创建 + 环境变量 + 启动命令」整合为一个 `docker-compose.yml`,容器起来后自动运行服务,并保证开机自启。

## 3.1 创建目录与 docker-compose.yml

```bash
mkdir -p /opt/qwen3-30b
```

```bash
cd /opt/qwen3-30b
```

用编辑器创建 `docker-compose.yml`(例如 `vim docker-compose.yml`),内容如下:

```yaml
services:
  qwen3-30b:
    image: quay.io/ascend/vllm-ascend:v0.23.0rc1-310p
    container_name: qwen3-30b
    restart: always
    network_mode: host
    shm_size: 64g
    environment:
      ASCEND_RT_VISIBLE_DEVICES: "0,1"
    devices:
      - /dev/davinci0
      - /dev/davinci1
      - /dev/davinci_manager
      - /dev/devmm_svm
      - /dev/hisi_hdc
    volumes:
      - /usr/local/dcmi:/usr/local/dcmi
      - /usr/local/bin/npu-smi:/usr/local/bin/npu-smi
      - /usr/local/Ascend/driver/lib64/:/usr/local/Ascend/driver/lib64/
      - /usr/local/Ascend/driver/version.info:/usr/local/Ascend/driver/version.info
      - /etc/ascend_install.info:/etc/ascend_install.info
      - /models:/models
    command:
      - vllm
      - serve
      - /models/Qwen3-30B-A3B-w8a8-QuaRot-310
      - --host
      - 0.0.0.0
      - --port
      - "8080"
      - --tensor-parallel-size
      - "2"
      - --gpu-memory-utilization
      - "0.92"
      - --max-num-seqs
      - "32"
      - --served-model-name
      - qwen3-30b
      - --dtype
      - float16
      - --enable-auto-tool-choice
      - --tool-call-parser
      - qwen3_coder
      - --additional-config
      - '{"ascend_compilation_config": {"fuse_norm_quant": false,"enable_npu_graph_ex":false}}'
      - --compilation-config
      - '{"cudagraph_mode": "FULL_DECODE_ONLY", "cudagraph_capture_sizes": [1,32]}'
      - --quantization
      - ascend
      - --max-model-len
      - "40960"
      - --enable-prefix-caching
```

与方式一(测试)的对应关系:

| docker compose 配置 | 对应的 docker run / 手动命令 |
| --- | --- |
| `image` | `docker run ... quay.io/ascend/vllm-ascend:v0.23.0rc1-310p` |
| `container_name` | `--name qwen3-30b` |
| `restart: always` | 生产新增:开机自启,测试时无 |
| `network_mode: host` | `--net=host` |
| `shm_size: 64g` | `--shm-size=64g` |
| `environment` | 容器内 `export ASCEND_RT_VISIBLE_DEVICES=0,1` |
| `devices` | 各 `--device /dev/...` 参数 |
| `volumes` | 各 `-v 宿主机路径:容器路径` 参数 |
| `command` | 容器内手动执行的 `vllm serve` 命令(含全部参数) |

> 说明:`command` 采用 YAML 列表逐项写法,JSON 参数用单引号包裹,可避免转义错误,也不需要再手动进入容器敲命令。

## 3.2 启动服务

在 `/opt/qwen3-30b` 目录下执行:

```bash
docker compose up -d
```

- `-d`:后台运行。
- 首次启动会拉取镜像,需要能访问网络(quay.io)。

查看容器状态:

```bash
docker ps
```

看到 `qwen3-30b` 的状态为 `Up` 即启动成功。

## 3.3 查看日志

```bash
docker compose logs -f
```

也可以指定容器名查看:

```bash
docker logs -f qwen3-30b
```

看到类似 `Application startup complete` 的输出,说明模型加载完成。按 `Ctrl + C` 退出日志查看。

## 3.4 验证服务

与测试阶段相同的验证命令:

```bash
curl http://localhost:8080/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
    "model": "qwen3-30b",
    "messages": [
      {"role": "system", "content": "你是个友善的AI助手。"},
      {"role": "user", "content": "你好,你是谁?可以做什么?"}
    ]
  }'
```

返回包含回答内容的 JSON,即生产服务正常。

## 3.5 停止与维护

停止并删除容器(volumes 挂载的数据不受影响):

```bash
docker compose down
```

修改了 `docker-compose.yml` 后重新应用:

```bash
docker compose up -d
```

只暂停(不删除容器):

```bash
docker compose stop
```

恢复运行:

```bash
docker compose start
```

## 3.6 开机自启说明

- 容器自身通过 `restart: always` 实现异常退出后自动重启。
- 还需要保证 **Docker 服务**随系统开机启动,否则容器无法被拉起:

```bash
systemctl enable docker
```

验证 Docker 已设置开机自启:

```bash
systemctl is-enabled docker
```

输出 `enabled` 即正常。之后宿主机重启,`qwen3-30b` 容器会自动恢复运行,无需手动操作。

---

## 常见问题

- **容器一直重启(Restarting)**:用 `docker logs qwen3-30b` 查看报错,常见原因是设备映射或模型路径不对。
- **报错 `no such device`**:确认 `/dev/davinci0`、`/dev/davinci1` 等设备文件存在。
- **端口被占用**:修改 `command` 中 `--port` 后的端口号,确保 `network_mode: host` 下该端口没有被其他进程占用。
- **拉取镜像失败**:确认宿主机能访问 `quay.io`。
- **生产与测试参数不一致**:改动参数时应先在方式一(docker run)里验证,再同步到 `docker-compose.yml` 的 `command` 中。
