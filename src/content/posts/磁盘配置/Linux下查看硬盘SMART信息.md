---
title: Linux下查看硬盘SMART信息
published: 2026-08-22
description: 用smartctl查看硬盘SMART信息,解读SSD与HDD关键参数、运行自检并判断磁盘健康状况。
category: 磁盘配置
tags:
  - 硬盘健康
  - SMART
  - smartctl
slug: linux-smart-disk-check
---

# 硬盘信息查看与 SMART 诊断指南

## 目录

1. [工具安装](#工具安装)
2. [smartctl 常用命令](#smartctl-常用命令)
3. [SMART 参数解读](#smart-参数解读)
4. [SSD 重点关注参数](#ssd-重点关注参数)
5. [HDD 重点关注参数](#hdd-重点关注参数)
6. [SMART 自检操作](#smart-自检操作)
7. [其他磁盘信息工具](#其他磁盘信息工具)
8. [附：示例盘 SMART 数据分析](#附示例盘-smart-数据分析)

---

## 工具安装

```bash
# RHEL / CentOS / openEuler
yum install -y smartmontools

# Debian / Ubuntu
apt install -y smartmontools

# 验证安装
smartctl --version
```

---

## smartctl 常用命令

|命令|用途|
| ----| --------------------------------------------|
|`smartctl -i /dev/sdb`|查看磁盘基本信息（型号、序列号、固件、容量）|
|`smartctl -H /dev/sdb`|健康状态快速检查（PASSED / FAILED）|
|`smartctl -A /dev/sdb`|仅查看 SMART 属性表|
|`smartctl -a /dev/sdb`|查看全部 SMART 信息（最常用）|
|`smartctl -x /dev/sdb`|查看全部信息，包括非 SMART 扩展数据|
|`smartctl -l error /dev/sdb`|查看 SMART 错误日志|
|`smartctl -l selftest /dev/sdb`|查看自检历史记录|
|`smartctl -l devstat /dev/sdb`|查看设备统计（读写量、通电时间等）|
|`smartctl -g all /dev/sdb`|查看所有可配置的 SMART 参数|

---

## SMART 参数解读

### VALUE / WORST / THRESH 字段含义

|字段|含义|
| ----| ----------------------------------------------------------|
|**VALUE**|当前归一化值（通常 100 为最佳，越低越差）|
|**WORST**|历史最差值（该属性曾经达到的最差记录）|
|**THRESH**|告警阈值（VALUE 低于此值时，SMART 判定为 FAILED）|
|**RAW_VALUE**|原始值（厂商自定义，通常直接反映物理计数而非归一化）|
|**TYPE**|`Pre-fail` = 预警型（触发即表示即将故障）；`Old_age` = 老化型（正常使用损耗）|

> **判断准则**：当 `VALUE <= THRESH` 时，`WHEN_FAILED` 列会显示 `FAILING_NOW`。这是 SMART 判定硬盘即将故障的信号。

---

## SSD 重点关注参数

### 一、寿命与磨损

|ID|属性名|含义|正常值|告警信号|
| --| ----------------------------| ----------------------| -----------| --------------------------|
|**233**|Media_Wearout_Indicator|NAND 寿命指示器|100（全新）|持续下降至接近 THRESH|
|**232**|Available_Reservd_Space|可用预留空间（备用块）|100|持续下降 → 备用块耗尽风险|
|**230**|(SSD 厂商) Life_Curve_Status|SSD 寿命曲线状态|100|下降|
|**241**|Total_LBAs_Written|累计写入量（主机）|随使用增长|配合 DWPD 规格评估剩余寿命|
|**242**|Total_LBAs_Read|累计读取量（主机）|随使用增长|—|

> **重要**：企业级 SSD（如 Solidigm/Intel）的 `Total_LBAs_Written` / `Total_LBAs_Read`
> RAW_VALUE 使用的单位与消费级 SSD 不同，**不能按 ****`RAW × 512B`** ** 计算**。
>
> 以 Solidigm（原 Intel）企业级 SATA SSD 为例，根据官方文档：
>
> - **Total_LBAs_Written (ID 241)** ：RAW_VALUE 每增加 1 表示主机写入了 **65,536 个扇区 (32 MiB)**
> - **Total_LBAs_Read (ID 242)** ：RAW_VALUE 每增加 1 表示主机读取了 **65,536 个扇区 (32 MiB)**
> - **Total NAND Writes**：RAW_VALUE 每增加 1 表示 **1 GB** 的 NAND 写入量
>
> 换算公式：
>
> ```
> 主机写入量 = RAW_VALUE(241) × 65,536 × 512 B = RAW_VALUE(241) × 32 MiB
> 主机读取量 = RAW_VALUE(242) × 65,536 × 512 B = RAW_VALUE(242) × 32 MiB
> NAND写入量 = RAW_VALUE(NAND属性) × 1 GB
> ```
>
> **以本示例盘为例**：
>
> |属性|RAW_VALUE|换算过程|结果|
> | ---------------------------| ---------| ---------------| ------------|
> |Total_LBAs_Read (ID 242)|70319|70319 × 32 MiB| **~2197 GiB** (~2.15 TiB)|
> |Total_LBAs_Written (ID 241)|51925|51925 × 32 MiB| **~1623 GiB** (~1.58 TiB)|

### 二、坏块与错误

|ID|属性名|含义|正常值|告警信号|
| --| -----------------------| ----------------------------------| --------| --------------------------------------|
|**5**|Reallocated_Sector_Ct|重映射扇区数（坏块已被备用块替换）|0| **&gt; 0 且持续增长**|
|**183**|Runtime_Bad_Block|运行时检测到的坏块|0|> 0|
|**187**|Reported_Uncorrect|不可纠正的错误数|0| **&gt; 0（严重）**|
|**197**|Current_Pending_Sector|待重映射的疑似坏扇区|0| **&gt; 0（可能即将变为坏块）**|
|**184**|End-to-End_Error|端到端数据完整性错误|0|> 0|
|**199**|UDMA_CRC_Error_Count|SATA 接口 CRC 校验错误|0|> 0（通常为线缆/接口问题，非盘体故障）|
|**175**|Program_Fail_Count_Chip|NAND 编程失败次数|0 或极低|快速增长|

### 三、温度与环境

|ID|属性名|含义|正常范围|告警信号|
| --| -----------------------| ------------------------------| --------| ---------------------|
|**194**|Temperature_Celsius|盘体温度 (°C)|25–45| **&gt; 60°C**（高温降速/损坏风险）|
|**190**|Airflow_Temperature_Cel|气流温度 (°C)，通常同盘体温度|25–45|> 60°C|

### 四、通电与使用

|ID|属性名|含义|备注|
| --| -----------------| -----------------| ----------------------|
|**9**|Power_On_Hours|通电时间（小时）|RAW_VALUE 为实际小时数|
|**12**|Power_Cycle_Count|通电/断电循环次数|频繁上下电可关注|

---

## HDD 重点关注参数

除与 SSD 共通的坏块/错误/温度参数外，机械盘还需额外关注：

|ID|属性名|含义|告警信号|
| --| -----------------------| ------------------------------| ----------------------------|
|**1**|Raw_Read_Error_Rate|原始读取错误率|持续上升|
|**3**|Spin_Up_Time|主轴起转时间|明显增加 → 电机老化|
|**4**|Start_Stop_Count|主轴起停次数|磁盘设计有寿命上限|
|**7**|Seek_Error_Rate|寻道错误率|持续上升|
|**10**|Spin_Retry_Count|主轴起转重试次数|> 0|
|**188**|Command_Timeout|命令超时次数|> 0|
|**191**|G-Sense_Error_Rate|震动/冲击感应错误率|> 0（物理冲击）|
|**193**|Load_Cycle_Count|磁头加载/卸载次数|笔记本硬盘关注是否超设计寿命|
|**196**|Reallocated_Event_Count|重映射事件次数（含成功和失败）|> 0 且增长|
|**198**|Offline_Uncorrectable|离线扫描发现的不可纠正扇区| **&gt; 0（严重）**|

---

## SMART 自检操作

### 运行自检

```bash
# 短自检（约 2 分钟，快速扫描部分区域）
smartctl -t short /dev/sdb

# 长自检（全盘扫描，时间取决于容量，可能数小时）
smartctl -t long /dev/sdb

# 传输自检（检测运输过程中是否受损，主要用于 HDD）
smartctl -t conveyance /dev/sdb

# 中止正在运行的自检
smartctl -X /dev/sdb
```

### 查看自检结果

```bash
# 查看自检进度和结果
smartctl -l selftest /dev/sdb

# 实时查看自检状态
smartctl -c /dev/sdb | grep -i "self-test"
```

### 自检结果解读

```text
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Short offline       Completed without error       00%        20         -
# 2  Short offline       Completed without error       00%         8         -
```

|Status 字段|含义|
| -----------| -------------------------------------------|
|`Completed without error`|自检通过，无错误|
|`Completed: read failure`|自检发现读取错误，需关注 LBA_of_first_error|
|`Self-test routine in progress`|自检进行中（查看 Remaining 百分比）|
|`Aborted by host`|被用户中止|

---

## 其他磁盘信息工具

### lsblk — 列出块设备

```bash
# 树形展示所有块设备
lsblk

# 带型号、序列号、容量、挂载点
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL,SERIAL,ROTA,TRAN

# 仅显示磁盘（排除分区）
lsblk -d -o NAME,SIZE,ROTA,MODEL,SERIAL
```

其中 `ROTA=0` 表示 SSD，`ROTA=1` 表示 HDD。

### hdparm — ATA 磁盘信息与性能

```bash
# 查看详细磁盘信息（型号、固件、支持特性、速度）
hdparm -I /dev/sdb

# 简单测速（仅参考，不要用于精确基准测试）
hdparm -Tt /dev/sdb
```

### NVMe 专用

```bash
# 列出所有 NVMe 设备
nvme list

# 查看 NVMe 控制器详细信息
nvme id-ctrl /dev/nvme0

# NVMe 设备的 SMART 信息
nvme smart-log /dev/nvme0
```

### dmesg — 内核日志

```bash
# 查看磁盘相关的内核消息（连接、错误、重置等）
dmesg | grep -i sdb

# 实时监控（插入磁盘时观察识别情况）
dmesg -w | grep -i sdb
```

### lsscsi — SCSI 设备列表

```bash
lsscsi
lsscsi -s    # 带容量
```

### blkid — 查看文件系统与 UUID

```bash
blkid /dev/sdb*
```

### fdisk / parted — 分区信息

```bash
fdisk -l /dev/sdb
parted /dev/sdb print
```

---

## 附：示例盘 SMART 数据分析

以下基于 `smartctl -a /dev/sdb` 输出对一块 **SOLIDIGM SSDSC2KB038TZ（3.84TB 企业级 SATA SSD）**  的评估：

### SMART 属性关键值

```text
ID 241 Total_LBAs_Written  RAW_VALUE = 51925
ID 242 Total_LBAs_Read     RAW_VALUE = 70319
```

### 读写量计算（Solidigm 官方公式）

根据 Solidigm 官方文档：

- **Total_LBAs_Written**：RAW 每 +1 = 65,536 扇区 (32 MiB) 主机写入
- **Total_LBAs_Read**：RAW 每 +1 = 65,536 扇区 (32 MiB) 主机读取

|项目|计算过程|结果|
| ----------| ------------------------| ----|
|累计读取量|70319 × 65,536 × 512 B|**2,359,514,103,808 B**|
||= 70319 × 32 MiB|**2,197.5 GiB ≈ 2.15 TiB**|
|累计写入量|51925 × 65,536 × 512 B|**1,742,313,881,600 B**|
||= 51925 × 32 MiB|**1,622.7 GiB ≈ 1.58 TiB**|

### 综合评估

|检查项|数值|判定|
| ------------------------| -----------------| ---------------|
|SMART 整体健康|`PASSED`|正常|
|通电时间|33 小时|极新（~1.4 天）|
|通电循环|40 次|正常|
|重映射扇区 (ID 5)|0|无坏块|
|待处理扇区 (ID 197)|0|无|
|不可纠正错误 (ID 187)|0|无|
|端到端错误 (ID 184)|0|数据完整性良好|
|运行时坏块 (ID 183)|0|无|
|UDMA CRC 错误 (ID 199)|0|接口/线缆正常|
|可用预留空间 (ID 232)|100 (VALUE)|备用块充足|
|NAND 寿命指示器 (ID 233)|100 (VALUE)|满寿命|
|累计写入量 (ID 241)|<sub>1623 GiB (</sub>1.58 TiB)|使用量低|
|累计读取量 (ID 242)|<sub>2197 GiB (</sub>2.15 TiB)|使用量低|
|温度 (ID 194)|33°C|正常|
|错误日志|No Errors Logged|无历史错误|
|自检记录|2 次 Short 均通过|自检正常|

**综合评估**：该盘状态健康，所有关键指标正常。写入量约 1.58 TiB，对于企业级 3.84TB SSD（典型 DWPD=1，即每日可写入 3.84TB，持续 5 年 ≈ 总计 7PB 寿命），当前写入仅占极少比例，剩余寿命充裕。

> **关于 ID 175 ****`Program_Fail_Count_Chip`**：RAW_VALUE=180388563398 数值看似很大，但 VALUE=100、THRESH=10，VALUE 远高于 THRESH。此为厂商特定编码格式（可能是 NAND 擦除/编程总次数），非实际失败计数。判断应以 VALUE/WORST/THRESH 的关系为准，而非只看 RAW_VALUE。

### 常见错误换算对比

|换算方法|写入量|读取量|正确？|
| ------------------------------| ---------| ---------| ---------------------|
|RAW × 512B（消费级 SSD 常用）|~25.4 MiB|~34.3 MiB|错误（低估约 5 万倍）|
|RAW × 32 MiB（**Solidigm 官方公式**）| **~1623 GiB**| **~2197 GiB**|正确|

> **不同厂商、不同型号的计量单位可能不同**。消费级 SSD 常以 512B 扇区计数，企业级 SSD 使用更大的单位块。获取准确读写量应查阅对应厂商的官方 SMART 属性规范。

---

## 快速检查清单

拿到一块盘后按以下顺序检查：

```text
1. smartctl -H /dev/sdX         --> 看 PASSED / FAILED
2. smartctl -A /dev/sdX         --> 聚焦以下属性：
   +-- ID 5    Reallocated_Sector_Ct    必须为 0
   +-- ID 197  Current_Pending_Sector   必须为 0
   +-- ID 187  Reported_Uncorrect       必须为 0
   +-- ID 233  Media_Wearout_Indicator  看剩余寿命（SSD）
   +-- ID 232  Available_Reservd_Space  看备用块（SSD）
   +-- ID 194  Temperature_Celsius      不超 55°C
   +-- ID 199  UDMA_CRC_Error_Count     必须为 0（>0 换线缆）
3. smartctl -l error /dev/sdX    --> 确认无历史错误
4. smartctl -l selftest /dev/sdX --> 确认自检通过（或运行一次 short）
5. dmesg | grep -i sdX           --> 确认无内核报错
```

---

## 参数速查

|参数|含义|
| ----| --------------------------------------|
|`-i`|基本信息（型号/序列号/固件/容量/接口）|
|`-H`|健康状态（PASSED/FAILED）|
|`-A`|仅 SMART 属性表|
|`-a`|全部 SMART 信息（常用）|
|`-x`|全部信息含扩展（最全）|
|`-t short`|运行短自检|
|`-t long`|运行长自检|
|`-t conveyance`|运行运输自检|
|`-X`|中止自检|
|`-l error`|错误日志|
|`-l selftest`|自检日志|
|`-l devstat`|设备统计信息|
|`--scan`|扫描所有支持 SMART 的设备|
|`--scan-open`|扫描并尝试打开所有设备|
