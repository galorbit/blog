---
title: "Ollama 服务部署与模型导入指南"
published: 2026-08-04
description: "记录在 Linux 环境下手动部署 Ollama 服务、配置 systemd 单元文件及通过 Modelfile 导入 DeepSeek 模型的完整操作流程。"
category: "大模型部署"
tags:
  - "Ollama"
  - "大模型部署"
  - "Linux"
  - "服务管理"
  - "模型导入"
---

# Ollama 服务部署与模型导入指南

> 需要先安装好 GPU 驱动和 CUDA，AMD 显卡同样需要安装驱动和 Rocm

## 安装 Ollama

### 解压和创建用户组

Ollama 下载地址：https://github.com/ollama/ollama/releases

下载 `ollama-linux-amd64.tgz` 并上传到服务器，本文档以 `/home/supervisor` 目录为例。

```bash
cd /home/supervisor

sudo tar -C /usr -xzf ollama-linux-amd64.tgz
# 解压到 /usr 目录下

sudo useradd -r -s /bin/false -U -m -d /usr/share/ollama ollama
# 创建 ollama 用户和组

sudo usermod -a -G ollama $(whoami)
# 将当前用户添加到 ollama 组

sudo usermod -a -G ollama supervisor
# 将 supervisor 用户添加到 ollama 组
```

### 添加 Ollama 服务

编辑服务文件：`nano /etc/systemd/system/ollama.service`

```ini
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
ExecStart=/usr/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=3
Environment="PATH=$PATH"
Environment="OLLAMA_HOST=0.0.0.0"
# 监听服务
Environment="CUDA_VISIBLE_DEVICES=0,1,2,3"
# 使用几卡就改几卡
Environment="OLLAMA_SCHED_SPREAD=1"
# 设置多卡同时运行
Environment="OLLAMA_LOAD_TIMEOUT=60m"
# 模型加载超时时间
Environment="OLLAMA_KEEP_ALIVE=-1"
# 模型保留在内存的时间
Environment="OLLAMA_DEBUG=1"
Environment="OLLAMA_MODELS=/data/ollama"
# 更改模型目录

[Install]
WantedBy=default.target
```

以上配置按需修改。

一些其他可用变量：

```text
"OLLAMA_DEBUG":             "显示额外的调试信息（例如 OLLAMA_DEBUG=1）"
"OLLAMA_FLASH_ATTENTION":   "启用闪电注意力"
"OLLAMA_GPU_OVERHEAD":      "为每个 GPU 保留一部分显存（字节）"
"OLLAMA_HOST":              "ollama 服务器的 IP 地址（默认 127.0.0.1:11434）"
"OLLAMA_KEEP_ALIVE":        "模型在内存中保持加载的持续时间（默认 \"5m\"）"
"OLLAMA_LLM_LIBRARY":       "设置 LLM 库以绕过自动检测"
"OLLAMA_LOAD_TIMEOUT":      "允许模型加载停滞的最长时间（默认 \"5m\"）"
"OLLAMA_MAX_LOADED_MODELS": "每个 GPU 加载的最大模型数量"
"OLLAMA_MAX_QUEUE":         "最大排队请求数量"
"OLLAMA_MODELS":            "模型目录的路径"
"OLLAMA_NOHISTORY":         "不保留读取历史"
"OLLAMA_NOPRUNE":           "启动时不修剪模型数据"
"OLLAMA_NUM_PARALLEL":      "最大并行请求数量"
"OLLAMA_ORIGINS":           "允许的来源列表（以逗号分隔）"
"OLLAMA_SCHED_SPREAD":      "始终在所有 GPU 上调度模型"
"OLLAMA_TMPDIR":            "临时文件的位置"
"OLLAMA_MULTIUSER_CACHE":   "为多用户场景优化提示缓存"
```

重新载入 daemon：`systemctl daemon-reload`

设置 Ollama 开机自启：`systemctl enable ollama`

启动 Ollama：`systemctl restart ollama`

查看 Ollama 状态：`systemctl status ollama`

## 导入模型

进入模型所在目录：`cd /model/DeepSeek-R1-Distill-Llama-70B`

创建 Modelfile 文件：`nano Modelfile`

DeepSeek 量化模型可直接复制以下模板，修改模型文件名即可。

```text
FROM ./DeepSeek-R1-Q4_K_M.gguf

template """
{{- if .System }}{{ .System }}{{ end }}
{{- range $i, $_ := .Messages }}
{{- $last := eq (len (slice $.Messages $i)) 1}}
{{- if eq .Role "user" }}<｜User｜>{{ .Content }}
{{- else if eq .Role "assistant" }}<｜Assistant｜>{{ .Content }}{{- if not $last }}<｜end▁of▁sentence｜>{{- end }}
{{- end }}
{{- if and $last (ne .Role "assistant") }}<｜Assistant｜>{{- end }}
{{- end }}
"""

#PARAMETER num_gpu 24
# 模型分层，非必要，按实际情况计算，需要预留 6G 的显存
PARAMETER num_ctx 2048
PARAMETER stop "<｜begin▁of▁sentence｜>"
PARAMETER stop "<｜end▁of▁sentence｜>"
PARAMETER stop "<｜User｜>"
PARAMETER stop "<｜Assistant｜>"
```

导入模型：`ollama create DeepSeek-R1-70B -f Modelfile`

进度条跑完后模型导入完成。

### 测试

运行模型：`ollama run DeepSeek-R1-70B --verbose`

添加 `--verbose` 参数，会显示模型输出 token 速度。

也可以使用 cherry studio 或者 open-webui 来测试。

Ollama 默认端口为 11434。

调用 `http://ip:11434` 测试。
