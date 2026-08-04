---
title: "DELL PowerVault ME5 多路径 multipath.conf 配置说明"
published: 2026-08-04
description: "本文提供DELL PowerVault ME5存储的Linux多路径multipath.conf标准配置、部署步骤及常见问题排查指南。"
category: "存储与磁盘"
tags:
  - "multipath"
  - "Dell PowerVault ME5"
  - "SAN存储"
  - "Linux多路径"
  - "device-mapper"
---

# DELL PowerVault ME5 多路径 multipath.conf 配置说明

## 完整配置

```ini
defaults {
    user_friendly_names      yes
    find_multipaths          yes
    flush_on_last_del        no
    queue_without_daemon     fail
    dev_loss_tmo             60
    fast_io_fail_tmo         10
}

blacklist {
    devnode "^hd[a-z]"
}

blacklist_exceptions {
    vendor "DellEMC"
    product "ME5"
}

devices {
    device {
        vendor                  "DellEMC"
        product                 "ME5"
        getuid                  "/lib/udev/scsi_id --page=0x83 --whitelisted --device=/dev/%n"
        prio                    "alua"
        path_grouping_policy    group_by_prio
        path_selector           "service-time 0"
        path_checker            tur
        features                "0"
        hardware_handler        "0"
        rr_weight               uniform
        rr_min_io_rq            1
        no_path_retry           5
        failback                immediate
        dev_loss_tmo            60
        fast_io_fail_tmo        10
    }
}

multipaths {
    # 示例：
    # multipath {
    #     wwid    3600a098038303867635d4a48624e5465
    #     alias   oracle_data01
    # }
    # multipath {
    #     wwid    3600a098038303867635d4a48624e5466
    #     alias   backup_lun01
    # }
}
```

## 与原笔记的差异说明

| 项目 | 原笔记 | 修正后 | 原因 |
| --- | --- | --- | --- |
| `getuid_callout` | `getuid_callout` | `getuid` | `getuid_callout` 是旧版（multipath-tools < 0.8.0）参数，RHEL 8+/openEuler 22.03+ 已弃用，必须换为 `getuid` |
| `blacklist` sd 规则 | `^sd[a-z]` + `^sd[a-z][a-z]*` | 移除 sd 黑名单 | 原规则试图通过黑名单 + 例外过滤 ME5 设备。正确实践是 **只黑名单本地设备（hd*）** ，依赖 `vendor/product` 匹配来锁定 ME5。保留 sd 黑名单存在漏匹配风险（例如设备名恰好不匹配模式时会被误黑） |
| `flush_on_last_del` | `yes` | `no` | 设为 `yes` 会在最后一条路径断开时删除 multipath map，SAN 维护或瞬间链路抖动可能导致 map 意外删除。建议保持 `no` |
| `rr_min_io` / `rr_min_io_rq` | `rr_min_io 100` | `rr_min_io_rq 1` | `rr_min_io` 是 **bio** 层参数（旧内核），`rr_min_io_rq` 是 **blk-mq** 层参数（RHEL 8+/openEuler 22.03+ 默认使用 blk-mq）。在 blk-mq 内核上 `rr_min_io` 不生效，必须使用 `rr_min_io_rq` |
| `no_path_retry` | `fail` | `5` | `fail` 在路径全部断开时立即向上层返回 I/O 错误。对于数据库等应用，建议设为 `5`（重试 5 次）或 `queue`（无限排队），给 SAN 链路恢复留出缓冲时间 |
| `path_selector` | 未指定 | `service-time 0` | 显式指定路径选择器，避免依赖发行版默认值差异。`service-time 0` 在 ALUA 环境下能根据路径响应时间动态选择最优路径 |

## 配置部署步骤

### 1. 安装 multipath-tools

```bash
# RHEL / CentOS / Rocky / openEuler
sudo dnf install device-mapper-multipath

# Debian / Ubuntu
sudo apt install multipath-tools
```

### 2. 写入配置

```bash
sudo vi /etc/multipath.conf
```

粘贴上述配置内容。

### 3. 加载模块并启动服务

```bash
sudo modprobe dm-multipath
sudo systemctl enable multipathd
sudo systemctl restart multipathd
```

### 4. 验证多路径状态

```bash
# 查看所有 multipath 设备
sudo multipath -ll

# 查看路径拓扑
sudo multipath -v3

# 实时查看路径事件
sudo journalctl -u multipathd -f
```

## 常见问题

### 看不到 multipath 设备

```bash
# 检查是否已被黑名单拦截
sudo multipath -d -v3 | grep -i "blacklist\|DellEMC\|ME5"

# 强制扫描
sudo multipath -v2

# 检查 SCSI 设备是否能正确识别 vendor/product
cat /sys/block/sdX/device/vendor
cat /sys/block/sdX/device/model
```

### 重启后 multipath 设备不自动恢复

```bash
# 确认 WWIDs 已记录
sudo cat /etc/multipath/wwids

# 手动添加（如果缺少）
sudo multipath -a /dev/sdX

# 重新组装
sudo multipath -r
```

### 路径不均衡（Active/Active 预期但显示单路径）

检查 ALUA 状态：

```bash
# 确认每个路径的 ALUA 优先级
sudo cat /sys/block/sdX/device/alua_state

# 查看路径优先级
sudo multipath -ll | grep -A 3 "ALUA"

# 手动触发优先级重新探测
sudo multipathd reconfigure
```

## 参数参考

### DELL ME5 推荐参数速查

| 参数 | 推荐值 | 说明 |
| --- | --- | --- |
| `path_grouping_policy` | `group_by_prio` | 按 ALUA 优先级分组路径 |
| `prio` | `alua` | DELL ME5 原生支持 ALUA，使用 ALUA 优先级判断主动/被动路径 |
| `path_checker` | `tur` | Test Unit Ready，对 ME5 兼容性最好 |
| `failback` | `immediate` | 故障路径恢复后立即切回 |
| `rr_min_io_rq` | `1` | 每个路径发 1 个 I/O 即轮换（blk-mq），实现均匀负载 |
| `no_path_retry` | `5` 或 `queue` | 根据业务容忍度设置 |
| `fast_io_fail_tmo` | `10` | 10 秒无响应即标记路径失败 |
| `dev_loss_tmo` | `60` | 60 秒后移除 SCSI 设备 |

### 内核版本对应参数差异

| 内核 / 发行版 | 应使用的参数 |
| --- | --- |
| RHEL 7 / CentOS 7 (blk-mq 未启用) | `rr_min_io 100` + `getuid_callout` |
| RHEL 8+ / Rocky 8+ / openEuler 22.03+ | `rr_min_io_rq 1` + `getuid` |
| Ubuntu 20.04+ / Debian 11+ | `rr_min_io_rq 1` + `getuid` |

## 参考

- DELL PowerVault ME5 部署指南
- `man multipath.conf`
- `man multipathd`
- 内核文档：`Documentation/admin-guide/device-mapper/multipath.rst`
