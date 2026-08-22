---
title: Linux使用gpu-burn-master压力测试
published: 2026-08-22
description: 用gpu-burn对GPU做压力测试,通过Gflop/s、错误数和温度判断显卡是否故障,支持双精度与多卡测试。
category: 环境配置
tags:
  - GPU
  - 压力测试
slug: linux-gpu-burn-master-stress-test
---

# Linux使用gpu-burn-master压力测试

## Linux GPU压力测试

首先安装好驱动和cuda，设置好cuda环境变量，使用nvcc -V查看cuda版本

1. 下载工具

`wget https://codeload.github.com/wilicc/gpu-burn/zip/master`

[【附件】gpu-burn-master.zip](/media/attachment/2025/04/gpu-burn-master.zip)

2. 解压

`unzip gpu-burn-master.zip`

3. 编译

```text
cd gpu-burn-master
make
ll

-rw-r--r--. 1 root root   2681 Apr  9  2024 compare.cu
-rw-r--r--. 1 root root   6104 Apr 14 16:33 compare.ptx
-rw-r--r--. 1 root root    341 Apr  9  2024 Dockerfile
-rwxr-xr-x. 1 root root  83232 Apr 14 16:33 gpu_burn
-rw-r--r--. 1 root root    962 Apr  9  2024 gpu-burn.8
-rw-r--r--. 1 root root  31595 Apr  9  2024 gpu_burn-drv.cpp
-rw-r--r--. 1 root root 105408 Apr 14 16:33 gpu_burn-drv.o
-rw-r--r--. 1 root root   1318 Apr  9  2024 LICENSE
-rw-r--r--. 1 root root   1469 Apr  9  2024 Makefile
-rw-r--r--. 1 root root   1801 Apr  9  2024 README.md

```

编译成功后生成了gpu_burn这个文件

4. 测试

`./gpu_burn 100`

默认测试所有GPU，测试时间为100S。可以使用-d参数来进行双精度浮点计算

`./gpu_burn -d 100`

5. 多卡指定GPU测试

`CUDA_VISIBLE_DEVICES=1 ./gpu_burn 100`

Gflop/s:GPU计算速度

errors：显示GPU计算有没有问题

temps：温度

测试结果有问题：

```text
[root@localhost gpu-burn-master]# ./gpu_burn 500
Using compare file: compare.ptx
Burning for 500 seconds.
GPU 0: Tesla T4 (UUID: GPU-899fdfea-f386-7371-8495-9a98620768fb)
Initialized device 0 with 14913 MB of memory (14798 MB available, using 13318 MB of it), using FLOATS
Results are 268435456 bytes each, thus performing 50 iterations
11.0%  proc'd: 150 (3392 Gflop/s)   errors: 30472096  (WARNING!)  temps: 63 C
        Summary at:   Mon Apr 14 04:37:04 PM CST 2025

22.0%  proc'd: 300 (3298 Gflop/s)   errors: 30537676  (WARNING!)  temps: 66 C
        Summary at:   Mon Apr 14 04:37:59 PM CST 2025

32.2%  proc'd: 450 (3194 Gflop/s)   errors: 30482853  (WARNING!)  temps: 67 C
        Summary at:   Mon Apr 14 04:38:50 PM CST 2025

43.2%  proc'd: 600 (3138 Gflop/s)   errors: 30498686  (WARNING!)  temps: 69 C
        Summary at:   Mon Apr 14 04:39:45 PM CST 2025

54.2%  proc'd: 750 (3093 Gflop/s)   errors: 30404967  (WARNING!)  temps: 69 C
        Summary at:   Mon Apr 14 04:40:40 PM CST 2025

64.2%  proc'd: 900 (3090 Gflop/s)   errors: 30511521  (WARNING!)  temps: 69 C
        Summary at:   Mon Apr 14 04:41:30 PM CST 2025

74.4%  proc'd: 1050 (3077 Gflop/s)   errors: 30539726  (WARNING!)  temps: 70 C
        Summary at:   Mon Apr 14 04:42:21 PM CST 2025

85.4%  proc'd: 1200 (3067 Gflop/s)   errors: 30461343  (WARNING!)  temps: 70 C
        Summary at:   Mon Apr 14 04:43:16 PM CST 2025

95.4%  proc'd: 1350 (3065 Gflop/s)   errors: 30454599  (WARNING!)  temps: 70 C
        Summary at:   Mon Apr 14 04:44:06 PM CST 2025

100.0%  proc'd: 1400 (3064 Gflop/s)   errors: 10159088  (WARNING!)  temps: 70 C
Killing processes with SIGTERM (soft kill)
Freed memory for dev 0
Uninitted cublas
done

Tested 1 GPUs:
        GPU 0: FAULTY
```

测试结果没问题

```text
[root@localhost gpu-burn-master]# ./gpu_burn 500
Using compare file: compare.ptx
Burning for 500 seconds.
GPU 0: NVIDIA RTX A400 (UUID: GPU-227259f4-b4d3-4638-69d1-4c5ceb2e21b0)
Initialized device 0 with 3777 MB of memory (3715 MB available, using 3343 MB of it), using FLOATS
Results are 268435456 bytes each, thus performing 11 iterations
10.8%  proc'd: 88 (1784 Gflop/s)   errors: 0   temps: 70 C
        Summary at:   Mon Apr 14 04:50:50 PM CST 2025

21.0%  proc'd: 165 (1774 Gflop/s)   errors: 0   temps: 74 C
        Summary at:   Mon Apr 14 04:51:41 PM CST 2025

31.2%  proc'd: 253 (1770 Gflop/s)   errors: 0   temps: 74 C
        Summary at:   Mon Apr 14 04:52:32 PM CST 2025

42.0%  proc'd: 330 (1772 Gflop/s)   errors: 0   temps: 74 C
        Summary at:   Mon Apr 14 04:53:26 PM CST 2025

53.0%  proc'd: 418 (1772 Gflop/s)   errors: 0   temps: 74 C
        Summary at:   Mon Apr 14 04:54:21 PM CST 2025

63.0%  proc'd: 506 (1771 Gflop/s)   errors: 0   temps: 74 C
        Summary at:   Mon Apr 14 04:55:11 PM CST 2025

73.6%  proc'd: 594 (1772 Gflop/s)   errors: 0   temps: 74 C
        Summary at:   Mon Apr 14 04:56:04 PM CST 2025

84.0%  proc'd: 671 (1772 Gflop/s)   errors: 0   temps: 74 C
        Summary at:   Mon Apr 14 04:56:56 PM CST 2025

94.2%  proc'd: 759 (1773 Gflop/s)   errors: 0   temps: 74 C
        Summary at:   Mon Apr 14 04:57:47 PM CST 2025

100.0%  proc'd: 814 (1772 Gflop/s)   errors: 0   temps: 74 C
Killing processes with SIGTERM (soft kill)
Freed memory for dev 0
Uninitted cublas
done

Tested 1 GPUs:
        GPU 0: OK

```
