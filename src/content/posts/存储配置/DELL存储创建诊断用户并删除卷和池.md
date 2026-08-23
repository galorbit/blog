---
title: DELL存储创建诊断用户并删除卷和池
published: 2026-08-23
description: 在DELL存储上创建诊断账户，启用虚拟池删除覆盖，按顺序删除卷、磁盘组和存储池。
image: "api"
category: 存储配置
tags:
  - DELL存储
  - 诊断用户
  - 卷删除
slug: dell-storage-create-diagnostic-user-delete-volumes-pools
---

# DELL存储创建诊断账户并删除卷和池

本文档详细说明如何在DELL存储系统上创建诊断用户账户，并按照正确顺序删除卷、磁盘组和存储池的完整操作流程。

> **警告**：以下操作涉及数据删除，执行前请确认已备份重要数据。删除操作不可逆，所有数据将永久丢失。

---

## 操作步骤总览

1. [创建诊断用户账户](#步骤1-创建诊断用户账户)
2. [启用虚拟池删除覆盖功能](#步骤2-启用虚拟池删除覆盖功能)
3. [查看并删除卷](#步骤3-查看并删除卷)
4. [查看磁盘组状态](#步骤4-查看磁盘组状态)
5. [删除磁盘组和存储池](#步骤5-删除磁盘组和存储池)

---

## 步骤1：创建诊断用户账户

使用 `create user` 命令创建具有诊断角色的用户账户。该账户拥有执行系统维护和故障排除所需的权限。

**命令：**
```bash
create user dell roles diagnostic
```

执行后系统会提示输入密码并确认：
```
Enter new password: **********
Re-enter new password: **********
Success: Command completed successfully. (dell) - The new user was created. (2025-06-10 16:09:30)
```

![创建诊断用户](https://pic.byt3.ro/pic/01-create-diagnostic-user.png)

**说明：**
- 用户名设为 `dell`，可根据实际需求修改
- 角色指定为 `diagnostic`（诊断），该角色具备系统诊断和维护权限
- 创建完成后建议立即退出当前会话，使用新账户重新登录验证

---

## 步骤2：启用虚拟池删除覆盖功能

在删除被隔离或存在异常的磁盘组之前，必须启用高级设置中的虚拟池删除覆盖功能。此设置允许绕过系统的保护检查，强制删除虚拟池和磁盘组。

**命令：**
```bash
set advanced-settings virtual-pool-delete-override enabled
```

执行后系统会显示警告信息并要求确认：
```
Virtual pools and disk groups must be removed in a specific order to maintain data integrity.
The virtual-pool-delete-override will bypass any system checks generally made to preserve data.
Using this setting enabled may cause irreparable damage to the pool and all volumes in it.
Are you sure you want to continue? (y/n) y

Info: The virtual-pool-delete-override setting will remain enabled for approximately 1 hour.
After that time, the setting will automatically be disabled. When the system has been properly
restarted (individually, to avoid data unavailability) using the command restart controllers,
the setting will be disabled immediately.

Success: Command completed successfully. (2025-06-10 16:09:55)
```

![启用虚拟池删除覆盖](https://pic.byt3.ro/pic/02-enable-virtual-pool-delete-override.png)

**重要注意事项：**
- 此设置会在约 **1小时** 后自动禁用
- 重启控制器后也会立即禁用此设置
-  必须在设置有效期内完成后续删除操作
- 仅在对问题磁盘组进行紧急维护时使用，正常操作中应保持禁用

---

## 步骤3：查看并删除卷

### 3.1 列出所有卷

首先查看系统中存在的所有卷及其状态：

**命令：**
```bash
show volumes
```

输出示例：
```
Pool Name      Total Size Alloc Size Class    Type Health Reason Action
-----------    ---------- ---------- -------- ---- ------ ------ ------
               Volume3    140.7TB    140.7TB  Virtual base OK
Success: Command completed successfully. (2025-06-10 16:10:37)
```

### 3.2 删除卷

尝试删除目标卷（本例中为 `Volume3`）：

**命令：**
```bash
delete volumes Volume3
```

系统会提示确认：
```
When the volumes are deleted all data in those volumes will be lost.
Do you want to continue? (y/n) y
```

**可能遇到的情况：**

如果磁盘组处于隔离（quarantined）状态，删除卷可能会失败：
```
Error: The operation failed because the disk group is quarantined. - Volume Volume3 was NOT deleted. (Volume3)
Error: Command failed. - One or more volumes could not be deleted. (2025-06-10 16:11:06)
```

![查看和删除卷](https://pic.byt3.ro/pic/03-list-and-delete-volumes.png)

**处理策略：**
- 如果卷删除成功：直接继续到步骤4
- 如果卷删除失败（磁盘组被隔离）：需要先处理磁盘组，见步骤4和步骤5
- 在启用了 `virtual-pool-delete-override` 的情况下，可以通过删除磁盘组来间接删除卷

---

## 步骤4：查看磁盘组状态

在执行删除操作前，需要确认磁盘组的当前状态和健康状况。

**命令：**
```bash
show disk-groups
```

输出示例：
```
Name   Size     Free Pool Tier % of Pool Own RAID       Disks
Planned Interl Vols Chk Status Current Job Job%   Sec Fmt  Health
--------------------------------------------------------------------------------------
dgA01  159.8TB  0B        N/A  100     A   RAID6      10
 0                    512k QTOF                     512e    Fault

Reason
Action
--------------------------------------------------------------
  The disk group is quarantined.
  - Look for events in the event log related to quarantine (172, 485, or 590), 
    and follow the recommended actions for those events.
Success: Command completed successfully. (2025-06-10 16:09:59)
```

**关键信息解读：**

| 字段 | 值 | 说明 |
|------|-----|------|
| Name | dgA01 | 磁盘组名称 |
| Size | 159.8TB | 磁盘组总容量 |
| Free | 0B | 可用空间为0 |
| Pool | A | 所属存储池 |
| RAID | RAID6 | RAID级别 |
| Disks | 10 | 包含10块磁盘 |
| Health | Fault | 健康状态为故障 |
| Status | QTOF | Quarantined - Temporarily Offline Fault（临时离线故障隔离） |

**磁盘组隔离原因：**
- 系统检测到磁盘组存在问题，已将其隔离以保护数据安全
- 常见隔离事件代码：172、485、590
- 需要查看事件日志确定具体原因

![查看磁盘组状态](https://pic.byt3.ro/pic/04-show-disk-groups.png)

---

## 步骤5：删除磁盘组和存储池

### 5.1 删除磁盘组

在启用了 `virtual-pool-delete-override` 后，可以强制删除被隔离的磁盘组：

**命令：**
```bash
remove disk-groups dgA01
```

系统会显示多重警告并要求确认：
```
When the disk groups are deleted, all volumes in those disk groups will be deleted. 
All data in those volumes will be lost.
Do you want to continue? (y/n) y

Disk group dgA01 has volumes present. Do you want to continue? (y/n) y

Info: Disk group dgA01 was deleted. (dgA01)
Success: Command completed successfully. (2025-06-10 16:12:53)
```

或者在某些情况下，删除磁盘组会同时删除关联的虚拟池：
```
Removing all disk groups from a pool and all volumes in that pool. All data in the pool will be lost.
Do you want to continue? (y/n) y

Info: The virtual pool was deleted. (A)
Success: Command completed successfully. (2025-06-10 16:10:22)
```

### 5.2 验证删除结果

删除后再次查看磁盘组列表确认：
```bash
show disk-groups
```

如果删除成功，输出应该为空或不再包含已删除的磁盘组。

### 5.3 删除存储池（如需要）

如果磁盘组删除后存储池仍然存在，可以手动删除：

**命令：**
```bash
show pools
delete pools A
```

![删除磁盘组和存储池](https://pic.byt3.ro/pic/05-remove-disk-groups-and-pools.png)

---

## 操作顺序总结

正确的删除顺序至关重要：

```
┌─────────────────────────────────┐
│  1. create user dell roles      │
│     diagnostic                  │
│  （创建诊断用户）                 │
└──────────────┬──────────────────┘

┌─────────────────────────────────┐
│  2. set advanced-settings       │
│     virtual-pool-delete-        │
│     override enabled            │
│  （启用删除覆盖，有效期~1小时）    │
└──────────────┬──────────────────┘

┌─────────────────────────────────┐
│  3. show volumes                │
│     delete volumes <name>       │
│  （尝试删除卷，可能因隔离而失败）  │
└──────────────┬──────────────────

┌─────────────────────────────────┐
│  4. show disk-groups            │
│  （确认磁盘组状态和隔离原因）      │
└────────────────────────────────┘

┌─────────────────────────────────┐
│  5. remove disk-groups <name>   │
│  （强制删除磁盘组及关联卷）        │
└──────────────┬──────────────────┘

┌─────────────────────────────────┐
│  6. show pools                  │
│     delete pools <name>         │
│  （如需要，删除空存储池）          │
└─────────────────────────────────┘
```

---

## 常见问题与注意事项

### Q1: 为什么删除卷时会报错 "disk group is quarantined"？
**A:** 当磁盘组因硬件故障或其他问题被系统隔离时，其中的卷无法被单独删除。必须先删除整个磁盘组才能释放这些卷。

### Q2: virtual-pool-delete-override 设置过期了怎么办？
**A:** 如果设置过期（约1小时后自动禁用），需要重新执行 `set advanced-settings virtual-pool-delete-override enabled` 命令再次启用。

### Q3: 删除磁盘组后如何恢复？
**A:** 删除操作不可逆。如需恢复数据，只能从备份中还原。建议在删除前确认已完成必要的数据迁移或备份。

### Q4: 如何查看磁盘组被隔离的具体原因？
**A:** 使用以下命令查看事件日志：
```bash
show events
```
查找与事件代码 172、485 或 590 相关的记录，根据推荐的行动方案进行处理。

### Q5: 删除操作会影响其他正常的磁盘组吗？
**A:** 不会。每个磁盘组是独立的存储单元，删除一个磁盘组不会影响其他健康的磁盘组及其中的数据。但请确保操作前已正确指定目标磁盘组名称。

---

## 相关命令速查表

| 操作 | 命令 |
|------|------|
| 创建诊断用户 | `create user <username> roles diagnostic` |
| 启用删除覆盖 | `set advanced-settings virtual-pool-delete-override enabled` |
| 查看所有卷 | `show volumes` |
| 删除卷 | `delete volumes <volume_name>` |
| 查看磁盘组 | `show disk-groups` |
| 删除磁盘组 | `remove disk-groups <disk_group_name>` |
| 查看所有池 | `show pools` |
| 删除池 | `delete pools <pool_name>` |
| 查看事件日志 | `show events` |
| 重启控制器 | `restart controllers` |

