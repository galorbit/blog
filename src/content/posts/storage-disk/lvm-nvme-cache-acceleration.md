---
title: "使用LVM为机械硬盘添加NVMe缓存加速读写"
published: 2026-08-04
description: "本文介绍在Ubuntu 20.04环境下，使用LVM技术为机械硬盘挂载NVMe缓存盘以提升读写性能的完整操作流程与验证方法。"
category: "存储与磁盘"
tags:
  - "LVM"
  - "NVMe缓存"
  - "机械硬盘"
  - "存储加速"
  - "系统运维"
---

# 使用LVM为机械硬盘添加NVMe缓存加速读写

## 操作系统平台

Ubuntu 20.04

## 前提条件

- 确保系统已安装 `lvm2` 软件包
- 本例已有数据卷组 `vg_data`，对应物理卷为 `/dev/sdb`。按照自己实际情况修改。
- 了解LVM缓存模式差异：
  - `writethrough`：数据同时写入缓存和原始卷，适合读密集型场景
  - `writeback`：数据先写入缓存，稍后同步到原始卷，适合读写均衡场景（本次示例使用）

## 操作步骤

1. **确认现有LVM结构**

   ```bash
   vgdisplay vg_data
   lvdisplay vg_data/data
   ```

2. **创建缓存盘分区（/dev/sdc）**

   > [!NOTE]
   > 缓存盘建议使用SSD/NVMe等高性能存储设备

   ```bash
   parted /dev/sdc
   (parted) mklabel gpt
   (parted) mkpart primary 1% 100%
   (parted) quit
   ```

3. **创建物理卷并添加到卷组**

   ```bash
   pvcreate /dev/sdc1
   vgextend vg_data /dev/sdc1
   ```

4. **创建缓存元数据逻辑卷**

   > [!WARNING]
   > 元数据卷大小建议为缓存池的1-2%，通常1-2GB足够

   ```bash
   lvcreate -n cache_meta -L 1G vg_data /dev/sdc1
   ```

5. **创建缓存池逻辑卷**

   ```bash
   lvcreate -n cache_pool -L 400G vg_data /dev/sdc1
   ```

6. **将缓存池转换为LVM缓存池类型**

   ```bash
   lvconvert --type cache-pool --poolmetadata vg_data/cache_meta vg_data/cache_pool
   ```

7. **将缓存应用到目标逻辑卷**

   > [!WARNING]
   > 操作前建议备份重要数据，此操作会重启I/O服务

   ```bash
   lvconvert --type cache --cachepool vg_data/cache_pool --cachemode writeback vg_data/data
   ```

8. **验证缓存配置状态**

   ```bash
   lvs -a -o name,attr,cache_mode,cache_pool,cache_size
   ```

## 验证要点

1. 检查目标逻辑卷属性是否包含 `Cwi-ao-C-` 标识
2. 确认 `cache_size` 显示正确缓存容量
3. 监控I/O性能变化以确认缓存生效

## 注意事项

> [!NOTE]
> - LVM缓存不支持跨越不同卷组操作，必须使用同一卷组内的逻辑卷
> - 建议元数据卷与缓存池使用同一物理卷，避免单点故障
> - 缓存失效处理：系统崩溃可能导致缓存数据不一致，需配置监控机制
> - 缓存移除命令：`lvconvert --splitcache vg_data/data`
