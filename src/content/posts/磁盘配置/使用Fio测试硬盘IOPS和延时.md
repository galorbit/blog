---
title: 使用Fio测试硬盘IOPS和延时
image: "api"
published: 2026-08-22
description: 用Fio对硬盘做延迟、IOPS、带宽和混合读写测试,解读结果指标并附一键批量测试脚本。
category: 磁盘配置
tags:
  - 硬盘性能测试
  - FIO
  - IOPS
slug: fio-disk-iops-latency-test
---

# FIO 硬盘性能测试指南

## 目录

1. [测试前准备](#测试前准备)
2. [延迟测试（Latency）](#延迟测试latency)
3. [IOPS 测试](#iops-测试)
4. [带宽/吞吐量测试（Bandwidth）](#带宽吞吐量测试bandwidth)
5. [混合读写测试](#混合读写测试)
6. [结果解读](#结果解读)
7. [附录：JSON 输出模式](#附录json-输出模式)

---

## 测试前准备

```bash
# 创建测试输出目录
mkdir -p /root/disk-test

# （可选）对测试盘进行预处理/预填充，消除新盘首次写入的性能偏差
# ⚠️  此操作会覆盖数据，请确认 /dev/sdb 为目标盘
fio --name=prefill --filename=/dev/sdb --direct=1 --ioengine=libaio \
    --bs=128k --rw=write --iodepth=32 --size=100% --numjobs=1
```

> **参数说明**：`--size=100%` 表示写满整盘（FIO 会根据块设备大小自动计算）。如果只想写一定量数据，可改为 `--size=10G`。

---

## 延迟测试（Latency）

> **目的**：测量单次 I/O 操作的响应时间。
> **关键参数**：`--iodepth=1 --numjobs=1`（队列深度=1，单并发）
> **关注指标**：`lat (usec/msec)` 中的 `avg`、`p99`（即 `clat percentiles` 下的 `99.00th`）

### 4K 顺序读（延迟）

```bash
fio --name=seq_4k_read_lat \
    --filename=/dev/sdb \
    --direct=1 \
    --ioengine=libaio \
    --bs=4k \
    --rw=read \
    --iodepth=1 \
    --numjobs=1 \
    --runtime=60 \
    --time_based \
    --group_reporting \
    --output=/root/disk-test/4k_seq_read_lat.log \
    --output-format=normal
```

### 4K 顺序写（延迟）

```bash
fio --name=seq_4k_write_lat \
    --filename=/dev/sdb \
    --direct=1 \
    --ioengine=libaio \
    --bs=4k \
    --rw=write \
    --iodepth=1 \
    --numjobs=1 \
    --runtime=60 \
    --time_based \
    --group_reporting \
    --output=/root/disk-test/4k_seq_write_lat.log \
    --output-format=normal
```

### 4K 随机读（延迟）

```bash
fio --name=rand_4k_read_lat \
    --filename=/dev/sdb \
    --direct=1 \
    --ioengine=libaio \
    --bs=4k \
    --rw=randread \
    --iodepth=1 \
    --numjobs=1 \
    --runtime=60 \
    --time_based \
    --group_reporting \
    --output=/root/disk-test/4k_rand_read_lat.log \
    --output-format=normal
```

### 4K 随机写（延迟）

```bash
fio --name=rand_4k_write_lat \
    --filename=/dev/sdb \
    --direct=1 \
    --ioengine=libaio \
    --bs=4k \
    --rw=randwrite \
    --iodepth=1 \
    --numjobs=1 \
    --runtime=60 \
    --time_based \
    --group_reporting \
    --output=/root/disk-test/4k_rand_write_lat.log \
    --output-format=normal
```

> **修正说明**（对比原笔记）：
>
> - 统一了 `--name` 命名风格（原笔记中读测试名为 `seq_4k_lat`，写测试名为 `seq_4k_write`，不一致）
> - 添加了 `--numjobs=1`（原笔记未显式指定，但 FIO 默认即为 1，显式写出更清晰）
> - 添加了 `--time_based`：确保 `--runtime=60` 是基于时间的运行，而非基于文件大小
> - 添加了 `--group_reporting`：将多个 job 的统计合并输出（此处单 job 虽然无实际影响，但作为模板保留）

---

## IOPS 测试

> **目的**：测量磁盘每秒能完成的 I/O 操作数。
> **关键参数**：小数据块（4K）+ 高队列深度（64）+ 多并发（4 或更多）
> **关注指标**：`IOPS=` 后面的数值（读 IOPS / 写 IOPS）

### 4K 随机读（IOPS）

```bash
fio --name=rand_4k_read_iops \
    --filename=/dev/sdb \
    --direct=1 \
    --ioengine=libaio \
    --bs=4k \
    --rw=randread \
    --iodepth=64 \
    --numjobs=4 \
    --runtime=60 \
    --time_based \
    --group_reporting \
    --output=/root/disk-test/4k_rand_read_iops.log \
    --output-format=normal
```

### 4K 随机写（IOPS）

```bash
fio --name=rand_4k_write_iops \
    --filename=/dev/sdb \
    --direct=1 \
    --ioengine=libaio \
    --bs=4k \
    --rw=randwrite \
    --iodepth=64 \
    --numjobs=4 \
    --runtime=60 \
    --time_based \
    --group_reporting \
    --output=/root/disk-test/4k_rand_write_iops.log \
    --output-format=normal
```

### 4K 随机混合读写 70%读/30%写（IOPS）

```bash
fio --name=rand_4k_rw70_iops \
    --filename=/dev/sdb \
    --direct=1 \
    --ioengine=libaio \
    --bs=4k \
    --rw=randrw \
    --rwmixread=70 \
    --iodepth=64 \
    --numjobs=4 \
    --runtime=60 \
    --time_based \
    --group_reporting \
    --output=/root/disk-test/4k_rand_rw70_iops.log \
    --output-format=normal
```

> **并发与队列深度说明**：
>
> - `iodepth=64`：每个 job 的异步 I/O 队列深度为 64
> - `numjobs=4`：同时运行 4 个 job 线程
> - **有效队列深度** = `iodepth × numjobs` = 64 × 4 = 256
> - 如果目标是对标纯单线程高队列场景，可将 `numjobs` 降为 1

---

## 带宽/吞吐量测试（Bandwidth）

> **目的**：测量磁盘的最大连续读写速率。
> **关键参数**：大块（128K 或 1M）+ 高队列深度（32 或 64）
> **关注指标**：`BW=` 后面的数值（单位 `MiB/s` 或 `GB/s`）

### 128K 顺序读（带宽）

```bash
fio --name=seq_128k_read_bw \
    --filename=/dev/sdb \
    --direct=1 \
    --ioengine=libaio \
    --bs=128k \
    --rw=read \
    --iodepth=32 \
    --numjobs=1 \
    --runtime=60 \
    --time_based \
    --group_reporting \
    --output=/root/disk-test/128k_seq_read_bw.log \
    --output-format=normal
```

### 128K 顺序写（带宽）

```bash
fio --name=seq_128k_write_bw \
    --filename=/dev/sdb \
    --direct=1 \
    --ioengine=libaio \
    --bs=128k \
    --rw=write \
    --iodepth=32 \
    --numjobs=1 \
    --runtime=60 \
    --time_based \
    --group_reporting \
    --output=/root/disk-test/128k_seq_write_bw.log \
    --output-format=normal
```

### 1M 顺序读（带宽 — 大块极限）

```bash
fio --name=seq_1m_read_bw \
    --filename=/dev/sdb \
    --direct=1 \
    --ioengine=libaio \
    --bs=1m \
    --rw=read \
    --iodepth=32 \
    --numjobs=1 \
    --runtime=60 \
    --time_based \
    --group_reporting \
    --output=/root/disk-test/1m_seq_read_bw.log \
    --output-format=normal
```

### 1M 顺序写（带宽 — 大块极限）

```bash
fio --name=seq_1m_write_bw \
    --filename=/dev/sdb \
    --direct=1 \
    --ioengine=libaio \
    --bs=1m \
    --rw=write \
    --iodepth=32 \
    --numjobs=1 \
    --runtime=60 \
    --time_based \
    --group_reporting \
    --output=/root/disk-test/1m_seq_write_bw.log \
    --output-format=normal
```

---

## 混合读写测试

> **目的**：模拟真实混合负载场景。

### 4K 随机混合读写 — 50%读 / 50%写

```bash
fio --name=rand_4k_rw50 \
    --filename=/dev/sdb \
    --direct=1 \
    --ioengine=libaio \
    --bs=4k \
    --rw=randrw \
    --rwmixread=50 \
    --iodepth=32 \
    --numjobs=4 \
    --runtime=120 \
    --time_based \
    --group_reporting \
    --output=/root/disk-test/4k_rand_rw50.log \
    --output-format=normal
```

### 128K 顺序混合读写 — 70%读 / 30%写

```bash
fio --name=seq_128k_rw70 \
    --filename=/dev/sdb \
    --direct=1 \
    --ioengine=libaio \
    --bs=128k \
    --rw=readwrite \
    --rwmixread=70 \
    --iodepth=16 \
    --numjobs=1 \
    --runtime=120 \
    --time_based \
    --group_reporting \
    --output=/root/disk-test/128k_seq_rw70.log \
    --output-format=normal
```

---

## 结果解读

以下以一个 FIO 输出片段为例，说明关键指标：

```text
read: IOPS=125k, BW=488MiB/s (512MB/s)(28.6GiB/60001msec)
    slat (usec): min=2, max=158, avg= 3.45, stdev= 1.23
    clat (usec): min=105, max=1582, avg=498.32, stdev=87.65
     lat (usec): min=109, max=1586, avg=502.10, stdev=88.02
    clat percentiles (usec):
     |  1.00th=[  392],  5.00th=[  412], 10.00th=[  420],
     | 20.00th=[  433], 30.00th=[  445], 40.00th=[  461],
     | 50.00th=[  482], 60.00th=[  502], 70.00th=[  529],
     | 80.00th=[  562], 90.00th=[  603], 95.00th=[  635],
     | 99.00th=[  701], 99.50th=[  734], 99.90th=[  873],
     | 99.95th=[  930], 99.99th=[ 1303]
```

|指标|含义|
| ----| ----------------------------------------------------------------|
|`IOPS`|每秒 I/O 操作数，越大越好|
|`BW`|带宽（吞吐量），括号内为 MB/s 十进制，括号外为 MiB/s 二进制|
|`slat`|提交延迟（Submission Latency）：从 FIO 发起 I/O 到内核接收的时间|
|`clat`|完成延迟（Completion Latency）：从内核接收到 I/O 完成的时间|
|`lat`|总延迟 = `slat + clat`|
|`clat percentiles`|完成延迟的百分位分布，`99.00th` 即 P99 延迟|

**性能判定参考**：

- **NVMe SSD**：4K 随机读 IOPS 通常 200K–1M+，延迟 P99 < 500µs
- **SATA SSD**：4K 随机读 IOPS 通常 60K–100K，延迟 P99 < 1ms
- **HDD**：4K 随机读 IOPS 通常 100–200，延迟 P99 可达 10–20ms

---

## 附录：JSON 输出模式

当 `--output-format=json` 时，FIO 输出为结构化 JSON，方便脚本解析：

```bash
# 示例：4K 随机读 IOPS 测试，JSON 输出
fio --name=rand_4k_read_iops \
    --filename=/dev/sdb \
    --direct=1 \
    --ioengine=libaio \
    --bs=4k \
    --rw=randread \
    --iodepth=64 \
    --numjobs=4 \
    --runtime=60 \
    --time_based \
    --group_reporting \
    --output=/root/disk-test/4k_rand_read_iops.json \
    --output-format=json
```

### JSON 结果关键字段提取示例（Python）

```python
import json

with open("/root/disk-test/4k_rand_read_iops.json") as f:
    data = json.load(f)

# 提取读 IOPS
read_iops = data["jobs"][0]["read"]["iops"]
print(f"Read IOPS: {read_iops}")

# 提取 P99 延迟 (单位: 纳秒，需转换为微秒)
clat_p99_ns = data["jobs"][0]["read"]["clat_ns"]["percentile"]["99.000000"]
clat_p99_us = clat_p99_ns / 1000
print(f"P99 clat: {clat_p99_us:.2f} us")

# 提取带宽 (单位: KiB/s)
read_bw_kib = data["jobs"][0]["read"]["bw"]
read_bw_mib = read_bw_kib / 1024
print(f"Read BW: {read_bw_mib:.2f} MiB/s")
```

> **JSON vs Normal**：
>
> - `normal` 格式：人类可读，适合直接查看和归档
> - `json` 格式：机器可解析，适合批量测试和自动化报表
> - 建议：调试阶段用 `normal`，自动化测试用 `json`（或同时生成，使用不同 `--output` 路径）

---

## 一键批量测试脚本

将以下内容保存为 `fio_benchmark.sh`，执行 `bash fio_benchmark.sh` 即可运行全部测试。

```bash
#!/bin/bash
# fio_benchmark.sh — 硬盘性能全项测试脚本
# 用法: bash fio_benchmark.sh /dev/sdb

DEVICE="${1:-/dev/sdb}"
OUTDIR="/root/disk-test"
RUNTIME=60

mkdir -p "$OUTDIR"

echo "=== Step 1/4: 延迟测试 (4K, iodepth=1, numjobs=1) ==="

fio --name=seq_4k_read_lat   --filename="$DEVICE" --direct=1 --ioengine=libaio --bs=4k --rw=read      --iodepth=1  --numjobs=1 --runtime=$RUNTIME --time_based --group_reporting --output="$OUTDIR/4k_seq_read_lat.log"   --output-format=normal
fio --name=seq_4k_write_lat  --filename="$DEVICE" --direct=1 --ioengine=libaio --bs=4k --rw=write     --iodepth=1  --numjobs=1 --runtime=$RUNTIME --time_based --group_reporting --output="$OUTDIR/4k_seq_write_lat.log"  --output-format=normal
fio --name=rand_4k_read_lat  --filename="$DEVICE" --direct=1 --ioengine=libaio --bs=4k --rw=randread  --iodepth=1  --numjobs=1 --runtime=$RUNTIME --time_based --group_reporting --output="$OUTDIR/4k_rand_read_lat.log"  --output-format=normal
fio --name=rand_4k_write_lat --filename="$DEVICE" --direct=1 --ioengine=libaio --bs=4k --rw=randwrite --iodepth=1  --numjobs=1 --runtime=$RUNTIME --time_based --group_reporting --output="$OUTDIR/4k_rand_write_lat.log" --output-format=normal

echo "=== Step 2/4: IOPS 测试 (4K, iodepth=64, numjobs=4) ==="

fio --name=rand_4k_read_iops  --filename="$DEVICE" --direct=1 --ioengine=libaio --bs=4k --rw=randread  --iodepth=64 --numjobs=4 --runtime=$RUNTIME --time_based --group_reporting --output="$OUTDIR/4k_rand_read_iops.log"  --output-format=normal
fio --name=rand_4k_write_iops --filename="$DEVICE" --direct=1 --ioengine=libaio --bs=4k --rw=randwrite --iodepth=64 --numjobs=4 --runtime=$RUNTIME --time_based --group_reporting --output="$OUTDIR/4k_rand_write_iops.log" --output-format=normal

echo "=== Step 3/4: 带宽测试 (128K, iodepth=32, numjobs=1) ==="

fio --name=seq_128k_read_bw  --filename="$DEVICE" --direct=1 --ioengine=libaio --bs=128k --rw=read  --iodepth=32 --numjobs=1 --runtime=$RUNTIME --time_based --group_reporting --output="$OUTDIR/128k_seq_read_bw.log"  --output-format=normal
fio --name=seq_128k_write_bw --filename="$DEVICE" --direct=1 --ioengine=libaio --bs=128k --rw=write --iodepth=32 --numjobs=1 --runtime=$RUNTIME --time_based --group_reporting --output="$OUTDIR/128k_seq_write_bw.log" --output-format=normal

echo "=== Step 4/4: 混合读写测试 ==="

fio --name=rand_4k_rw70  --filename="$DEVICE" --direct=1 --ioengine=libaio --bs=4k   --rw=randrw     --rwmixread=70 --iodepth=64 --numjobs=4 --runtime=$RUNTIME --time_based --group_reporting --output="$OUTDIR/4k_rand_rw70.log"  --output-format=normal
fio --name=seq_128k_rw70 --filename="$DEVICE" --direct=1 --ioengine=libaio --bs=128k --rw=readwrite  --rwmixread=70 --iodepth=16 --numjobs=1 --runtime=$RUNTIME --time_based --group_reporting --output="$OUTDIR/128k_seq_rw70.log" --output-format=normal

echo "=== 测试完成，结果保存在 $OUTDIR ==="
```

---

## 参数速查表

|参数|含义|常用值|
| ----| ------------------------------------------| -----------------------|
|`--direct=1`|绕过系统缓存（O_DIRECT），测试真实磁盘性能|`1`|
|`--ioengine=libaio`|Linux 异步 I/O 引擎|`libaio`|
|`--bs`|块大小|`4k`, `128k`, `1m`|
|`--rw`|读写模式|`read`, `write`, `randread`, `randwrite`, `randrw`, `readwrite`|
|`--rwmixread`|混合读写中读的比例（百分比）|`50`, `70`|
|`--iodepth`|队列深度|延迟测试 `1`，IOPS 测试 `32`–`128`|
|`--numjobs`|并发 job 数|延迟测试 `1`，IOPS 测试 `1`–`8`|
|`--runtime`|每项测试运行时间（秒）|`60`, `120`|
|`--time_based`|按时间运行（而非按文件大小）|—|
|`--group_reporting`|合并所有 job 的统计输出|—|
|`--output-format`|输出格式|`normal`, `json`, `json+`|
