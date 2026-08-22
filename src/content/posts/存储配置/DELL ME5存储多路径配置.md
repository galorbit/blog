---
title: DELL ME5存储多路径配置
published: 2026-08-22
description: 为DELL PowerVault ME5存储编写multipath.conf多路径配置,启用ALUA优先级与路径选择策略,提升链路冗余。
category: 存储配置
tags:
  - 多路径
  - multipath
  - ALUA
slug: dell-me5-storage-multipath-config
---

# DELL PowerVault ME5 多路径配置文件（multipath.conf）

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

---

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

---

## 参数参考

### DELL ME5 推荐参数速查

|参数|推荐值|说明|
| --- | --- | --- |
|`path_grouping_policy`|`group_by_prio`|按 ALUA 优先级分组路径|
|`prio`|`alua`|DELL ME5 原生支持 ALUA，使用 ALUA 优先级判断主动/被动路径|
|`path_checker`|`tur`|Test Unit Ready，对 ME5 兼容性最好|
|`failback`|`immediate`|故障路径恢复后立即切回|
|`rr_min_io_rq`|`1`|每个路径发 1 个 I/O 即轮换（blk-mq），实现均匀负载|
|`no_path_retry`|`5` 或 `queue`|根据业务容忍度设置|
|`fast_io_fail_tmo`|`10`|10 秒无响应即标记路径失败|
|`dev_loss_tmo`|`60`|60 秒后移除 SCSI 设备|

### 内核版本对应参数差异

|内核 / 发行版|应使用的参数|
| --- | --- |
|RHEL 7 / CentOS 7 (blk-mq 未启用)|`rr_min_io 100` + `getuid_callout`|
|RHEL 8+ / Rocky 8+ / openEuler 22.03+|`rr_min_io_rq 1` + `getuid`|
|Ubuntu 20.04+ / Debian 11+|`rr_min_io_rq 1` + `getuid`|

---

## 参考

- DELL PowerVault ME5 部署指南
- `man multipath.conf`
- `man multipathd`
- 内核文档：`Documentation/admin-guide/device-mapper/multipath.rst`
