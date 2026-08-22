---
title: PMC芯片存储控制器管理命令（arcconf）
published: 2026-08-22
description: 汇总PMC存储控制器工具arcconf的常用命令,涵盖RAID创建删除、固件更新与SMART查询。
category: 磁盘配置
tags:
  - 硬件RAID
  - RAID卡
  - arcconf
slug: pmc-arcconf-storage-controller
---

# PMC芯片存储控制器管理命令（arcconf）

常用命令示范：

`arcconf getconfig 1`获取控制卡配置

[华为Raid卡常用命令](support.huawei.com/enterprise/zh/doc/EDOC1000163568/f63b5f9b#ZH-...)

```text
ATAPASSWORD 物理盘设置密码
BACKUPUNIT 备份单元操作
CONSISTENCYCHECK 打开控制器后台一致性检测模式
COPYBACK 打开控制器后台copyback模式
CPLD CPLD相关操作
CREATE 创建逻辑盘
DELETE 删除逻辑盘
ERRORTUNABLE 设置控制器错误可调属性
EXPANDERUPGRADE 更新expander firmware
FAILOVER 打开控制器failover模式
GETCONFIG 打印控制器信息
GETEXCEPTION 获取控制器、逻辑盘、物理盘例外
GETLOGS 获取控制器log信息
GETPERFORM 获取performance mode参数
GETSMARTSTATS 获取控制器SMART信息
GETSTATUS 显示正在运行任务的状态
GETVERSION 打印所有控制器的版本信息
IDENTIFY 闪烁控制器上所接硬盘的LED灯
IMAGEUPDATE 更新物理盘Firmware
KEY 安装feature key到控制器内
LIST 列出系统连接的所有控制器
MODIFY 变更raid级别或在线扩展容量
PHYERRORLOG 显示控制器或设备或expander PHY的PHY errorr日志
PLAYCONFIG 将XML配置导入到控制器
PRESERVECACHE 变更控制器的cache保存设置
RESCAN 检查新增或移除的硬盘
RESETSTATISTICSCOUNTERS 重置控制器统计计数
ROMUPDATE 更新控制器Firmware
SAVECONFIG 保存控制器信息XML文件
SAVESUPPORTARCHIVE 保存配置存档
SEEPROM 更新控制器SEEPROM firmware
SETALARM 设置控制器报警
SETBIOSPARAMS 设置控制器BIOS参数
SETBOOT 设置可启动设备
SETCACHE 调整物理或逻辑设备的cache模式
SETCONFIG 恢复出厂设置
SETCONTROLLERMODE 控制器模式设置
SETCUSTOMMODE 设置用户自定义模式
SETMAXCACHE 调整物理或逻辑盘的maxcache设置
SETNAME 重命名逻辑设备的设备号
SETNCQ 开启控制器NCQ状态SETPERFORM 根据应用变更适配器设置
SETPHY 重配置PHY设置
SETPOWER 设置控制器或逻辑设备功耗
SETPRIORITY 改变特定的或全局的任务优先级
SETSTATE 人工设置物理设备或逻辑设备状态
SETSTATSDATACOLLECTION 打开控制器统计数据收集模式SLOTCONFIG 列出背板上的每个槽位的设备
SMP 发送SMP命令至expander
TASK 物理或逻辑设备上开启可应用任务
UARTLOG 改变控制器UART控制
UNINIT 人工停止状态为raw或ready的物理设备初始化
VERIFYWRITE 打开控制器的验证写特性
```
