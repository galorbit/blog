---
title: 昇腾NPU切分手册
image: "api"
published: 2026-08-22
description: 记录昇腾910B NPU的VNPU切分方法，涵盖模板选择、创建、验证、销毁及资源规划，用于多模型共享物理卡。
category: 环境配置
tags:
  - 昇腾NPU
  - VNPU切分
  - npu-smi
slug: ascend-npu-partition-guide
---

# 昇腾 NPU 切分手册（个人笔记）

> 适用环境：kunlun G5680 V2 算力服务器，8 × 昇腾 910B-B4 64G（NPU0~NPU7）
> 工具：npu-smi（本机版本 26.0.rc1）
> 整理自桌面 `npu切分.txt` 及日常使用经验，2026-08-17

---

## 目录

- [1. 概念速览](#1-概念速览)
- [2. 常用查看命令](#2-常用查看命令)
- [3. 切分模板明细](#3-切分模板明细)
- [4. VNPU ID 分配表](#4-vnpu-id-分配表)
- [5. 创建 VNPU（切分）](#5-创建-vnpu切分)
- [6. 验证切分](#6-验证切分)
- [7. 删除 VNPU（销毁）](#7-删除-vnpu销毁)
- [8. 资源规划与互斥规则](#8-资源规划与互斥规则)
- [9. 常见问题](#9-常见问题)

---

## 1. 概念速览

- **VNPU（虚拟 NPU）**：把一张 64G 物理 NPU 切分成多个虚拟卡，让多个小模型共享一张物理卡。
- 本机每张卡支持两种切法（**只能二选一**）：
  - **4 卡切分**：`vir05_1c_16g`（1 核 16G，每卡 4 个 VNPU）→ 小模型（bge / OCR / 语音等）
  - **2 卡切分**：`vir10_3c_32g`（3 核 32G，每卡 2 个 VNPU）→ 稍大模型（8B 重排 / 8B 向量等）
- 切分与销毁均通过 `npu-smi set -t create-vnpu / destroy-vnpu` 完成；
- 切分后容器以 `/dev/vdavinci<ID>:/dev/davinci0` 方式使用虚拟卡；
- **8 卡模型（如 Qwen3.5-122B、DeepSeek-V4-Flash）需要整卡，运行前必须销毁全部 VNPU。**

## 2. 常用查看命令

### 2.1 整机状态

```bash
npu-smi info
```

输出要点：8 张卡（910B4-1）健康状态 OK、温度/功耗、HBM 显存占用（65536MB = 64G）、以及各卡正在运行的进程。本机当前示例：全部 NPU 无运行进程。

### 2.2 查看切分模板

```bash
npu-smi info -t template-info
```

本机输出（两种模板及资源配额）：

```
+------------------------------------------------------------------------------------------+
|NPU instance template info is:                                                            |
|Name                AICORE    Memory    AICPU     VPC            VENC           JPEGD     |
|                               GB                 PNGD           VDEC           JPEGE     |
|==========================================================================================|
|vir10_3c_32g        10        32        3         4              0              12        |
|                                                  0              1              2         |
+------------------------------------------------------------------------------------------+
|vir05_1c_16g        5         16        1         2              0              6         |
|                                                  0              0              1         |
+------------------------------------------------------------------------------------------+
```

### 2.3 查看某张卡的切分状态

```bash
npu-smi info -t info-vnpu -i <卡号> -c 0
```

示例（NPU7 未切分时的输出）：

```
+-------------------------------------------------------------------------------+
| NPU resource static info as follow:                                           |
| Format:Free/Total                   NA: Currently, query is not supported.    |
| AICORE    Memory    AICPU    VPC    VENC    VDEC    JPEGD    JPEGE    PNGD    |
|            GB                                                                 |
|===============================================================================|
| 20/20     60/64     6/6      9/9    0/0     2/2     24/24    4/4      NA/NA   |
+-------------------------------------------------------------------------------+
| Total number of vnpu: 0                                                       |
+-------------------------------------------------------------------------------+
```

字段解读：

| 字段 | 含义 |
| --- | --- |
| Free/Total | 空闲/总数（AICORE 20 个、显存 64G、AICPU 6 个、VPC 9 个等；Memory 的 60/64 表示已被系统占用 4G） |
| Total number of vnpu | 当前该卡已创建的 VNPU 数量 |
| Vnpu ID | 已创建的 VNPU 编号（下方表格，未创建时为空） |
| Vgroup ID / Container ID / Status / Template Name | VNPU 所在组、绑定容器、状态、使用的模板名 |

## 3. 切分模板明细

| 模板 | AICORE | 内存(GB) | AICPU | VPC | VENC | VDEC | JPEGD | JPEGE | PNGD | 每卡切分数 | 适用 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| vir10_3c_32g | 10 | 32 | 3 | 4 | 0 | 1 | 12 | 2 | 0 | 2 个 | 8B 重排 / 8B 向量等稍大模型 |
| vir05_1c_16g | 5 | 16 | 1 | 2 | 0 | 0 | 6 | 1 | 0 | 4 个 | bge 系列 / OCR / 语音等小模型 |

> 资源核算示例：NPU7 切 2 个 `vir10_3c_32g` 后，AICORE 2×10=20 全部占用（Free/Total = 0/20），显存 2×32=64G 全部占用（0/64），与实测输出一致（见 6.1）。

## 4. VNPU ID 分配表

每张 NPU 的 VNPU ID 有固定范围（整机唯一，不可跨卡使用）：

| 物理卡 | VNPU ID 范围 | 物理卡 | VNPU ID 范围 |
| --- | --- | --- | --- |
| NPU0 | 100 ~ 115 | NPU4 | 164 ~ 179 |
| NPU1 | 116 ~ 131 | NPU5 | 180 ~ 195 |
| NPU2 | 132 ~ 147 | NPU6 | 196 ~ 211 |
| NPU3 | 148 ~ 163 | NPU7 | 212 ~ 227 |

> 本机常见分配：NPU6 → 196/197（2 切）、198~201（4 切）；NPU7 → 212/213（2 切）、214~217（4 切）。

## 5. 创建 VNPU（切分）

### 5.1 命令格式

```bash
npu-smi set -t create-vnpu -i <卡号> -c 0 -f <模板名> -v <VNPU ID>
```

| 参数 | 说明 |
| --- | --- |
| -t create-vnpu | 创建 VNPU |
| -i | 物理卡号（0~7） |
| -c 0 | 芯片编号（本机固定 0） |
| -f | 模板名：vir10_3c_32g（2 切）或 vir05_1c_16g（4 切） |
| -v | VNPU ID（必须在该卡的 ID 范围内，见第 4 章） |

### 5.2 单条创建示例（实测输出）

```bash
[root@localhost ~]# npu-smi set -t create-vnpu -i 7 -c 0 -f vir10_3c_32g -v 212
        Status                         : OK
        Message                        : Create vnpu success
[root@localhost ~]# npu-smi set -t create-vnpu -i 7 -c 0 -f vir10_3c_32g -v 213
        Status                         : OK
        Message                        : Create vnpu success
```

### 5.3 批量切分脚本

4 卡切分（NPU6 → 198~201，NPU7 → 214~217）：

```bash
# 卡6：ASR / TTS-Base / TTS-CustomVoice / TTS-VoiceDesign
for v in 198 199 200 201; do
  echo "卡6 创建 vNPU $v ..."
  npu-smi set -t create-vnpu -i 6 -c 0 -f vir05_1c_16g -v "$v"
done

# 卡7：bge-m3 / bge-reranker-v2-m3 / PaddleOCR-VL-1.6 / GLM-OCR
for v in 214 215 216 217; do
  echo "卡7 创建 vNPU $v ..."
  npu-smi set -t create-vnpu -i 7 -c 0 -f vir05_1c_16g -v "$v"
done
```

2 卡切分（NPU6 → 196/197，NPU7 → 212/213）：

```bash
npu-smi set -t create-vnpu -i 7 -c 0 -f vir10_3c_32g -v 212
npu-smi set -t create-vnpu -i 7 -c 0 -f vir10_3c_32g -v 213
npu-smi set -t create-vnpu -i 6 -c 0 -f vir10_3c_32g -v 196
npu-smi set -t create-vnpu -i 6 -c 0 -f vir10_3c_32g -v 197
```

### 5.4 注意事项

1. **先切分，再启动模型容器**（`docker compose up -d` 前必须完成）；
2. VNPU ID 必须落在该卡范围内（见第 4 章），且整机唯一，重复会失败；
3. 一卡只能处于一种切分状态（2 切或 4 切），切分前若该卡已有 VNPU，先销毁干净（见第 7 章）；
4. 资源不足（如已切满）时创建会失败，先 `info-vnpu` 确认剩余资源。

## 6. 验证切分

### 6.1 查看切分结果（实测输出：NPU7 切完 212/213 后）

```bash
npu-smi info -t info-vnpu -i 7 -c 0
```

```
+-------------------------------------------------------------------------------+
| NPU resource static info as follow:                                           |
| Format:Free/Total                   NA: Currently, query is not supported.    |
| AICORE    Memory    AICPU    VPC    VENC    VDEC    JPEGD    JPEGE    PNGD    |
|            GB                                                                 |
|===============================================================================|
| 0/20      0/64      1/7      1/9    0/0     0/2     0/24     0/4      NA/NA   |
+-------------------------------------------------------------------------------+
| Total number of vnpu: 2                                                       |
+-------------------------------------------------------------------------------+
|  Vnpu ID  |  Vgroup ID     |  Container ID  |  Status  |  Template Name       |
+-------------------------------------------------------------------------------+
|  212      |  0             |  000000000000  |  0       |  vir10_3c_32g        |
+-------------------------------------------------------------------------------+
|  213      |  1             |  000000000000  |  0       |  vir10_3c_32g        |
+-------------------------------------------------------------------------------+
```

解读：AICORE 0/20、显存 0/64（2 个 32G 模板已占满）；Total number of vnpu: 2；列表显示 VNPU 212/213，Status 0 表示正常，Container ID 为 0 表示尚未被容器绑定。

### 6.2 检查设备节点

```bash
ls /dev/vdavinci*
# 应能看到 vdavinci196/197/198/... / vdavinci212/213/... 等节点
```

### 6.3 整机状态复核

```bash
npu-smi info
```

切分后整机信息中对应卡会显示 VNPU 占用（HBM 显存被虚拟卡占用），同时可确认无残留进程。

## 7. 删除 VNPU（销毁）

### 7.1 单条删除

```bash
npu-smi set -t destroy-vnpu -i <卡号> -c 0 -v <VNPU ID>
```

示例：

```bash
npu-smi set -t destroy-vnpu -i 7 -c 0 -v 212
```

### 7.2 批量删除脚本

批量删除 NPU6/NPU7 上全部 4 卡切分 VNPU（自动判断物理卡号）：

```bash
for v in 198 199 200 201 214 215 216 217; do
  if [ "$v" -ge 212 ]; then phy=7; else phy=6; fi
  echo "销毁卡${phy} vNPU ${v} ..."
  npu-smi set -t destroy-vnpu -i "$phy" -c 0 -v "$v"
done
```

按卡单独批量删除：

```bash
# 删除 NPU6 上所有 VNPU（196/197 或 198~201 视当前切法）
npu-smi set -t destroy-vnpu -i 6 -c 0 -v 196
npu-smi set -t destroy-vnpu -i 6 -c 0 -v 197
npu-smi set -t destroy-vnpu -i 6 -c 0 -v 198
npu-smi set -t destroy-vnpu -i 6 -c 0 -v 199
npu-smi set -t destroy-vnpu -i 6 -c 0 -v 200
npu-smi set -t destroy-vnpu -i 6 -c 0 -v 201
```

### 7.3 注意事项

1. **先停止使用该 VNPU 的容器（docker compose down），再销毁 VNPU**，否则正在运行的推理会异常；
2. 销毁后可执行 `npu-smi info -t info-vnpu -i <卡号> -c 0` 确认 `Total number of vnpu` 归零；
3. 释放全部 VNPU 后该卡才可用于 8 卡大模型。

## 8. 资源规划与互斥规则

| 规则 | 说明 |
| --- | --- |
| 一卡一法 | 每张物理卡只能选 2 切（vir10_3c_32g）或 4 切（vir05_1c_16g）之一 |
| 切换切法 | 先把该卡现有 VNPU 全部销毁，再按新模板创建 |
| 8 卡模型互斥 | Qwen3.5-122B / DeepSeek-V4-Flash 等 8 卡模型运行前，必须停止所有 VNPU 容器并销毁全部 VNPU |
| 本机常见布局 | NPU0-3 / NPU4-5 给多卡 LLM；NPU6、NPU7 切分给检索 / OCR / 语音小模型 |
| ID 规划 | 按第 4 章 ID 范围固定分配（NPU6 用 196-211，NPU7 用 212-227），避免混乱 |

## 9. 常见问题

| 现象 | 处理 |
| --- | --- |
| create-vnpu 失败 | 确认 VNPU ID 在对应卡范围内且未被占用；确认该卡当前切法与模板一致（先销毁再切） |
| 切分成功但容器起不来 | 确认 `/dev/vdavinci<ID>` 节点存在；容器 devices 映射写 `/dev/vdavinci<ID>:/dev/davinci0` |
| info-vnpu 显示 Free 为 0 | 该卡 VNPU 已切满（2 个 32G 或 4 个 16G），需先销毁再规划 |
| 8 卡模型报设备占用 | 还有 VNPU 未销毁，按第 7 章批量删除后重试 |
| destroy-vnpu 失败 | 先确认对应容器已停止（docker compose down），再销毁 |
| 忘记某卡切了哪些 VNPU | `npu-smi info -t info-vnpu -i <卡号> -c 0` 查看列表 |

---

*手册完。命令均在本机 26.0.rc1 版本实测过；切分/销毁操作请避开业务高峰，先停容器再动 VNPU。*
