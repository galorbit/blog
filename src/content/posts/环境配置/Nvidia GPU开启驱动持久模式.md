---
title: Nvidia GPU开启驱动持久模式
published: 2026-08-22
description: 开启NVIDIA驱动持久模式防止GPU掉卡,可用nvidia-smi临时开启或配置服务实现开机常驻。
category: 环境配置
tags:
  - NVIDIA
  - GPU
  - 驱动
slug: nvidia-gpu-persistence-mode
---

# NVIDIA Persistence Mode（持久模式）

## 什么是 Persistence Mode

NVIDIA GPU 驱动加载后，默认行为是**惰性初始化**：只有当某个进程（如 CUDA 应用）访问 GPU 时，
内核驱动才会被加载并保持活跃；当最后一个 GPU 进程退出后，驱动会在几秒内卸载，
GPU 回到低功耗空闲状态。

**Persistence Mode（持久模式）**  开启后：

- GPU 驱动在内核中**常驻加载**，即使没有任何进程在使用 GPU 也不会被卸载。
- 避免了驱动反复加载/卸载带来的延迟和状态丢失。
- 是运行 **NVIDIA MIG（多实例 GPU）** 、**vGPU** 和 **CUDA 持久运行环境** 的前提条件。

## 为什么需要它

- **防止"掉卡"** ：部分 GPU（尤其是数据中心卡如 A100/H100/L40S 等）在驱动卸载后可能无法被
  操作系统重新识别，表现为 `nvidia-smi` 看不到该卡，即常见的"掉卡"问题。
- **降低延迟**：CUDA 程序启动时无需等待驱动加载，减少首帧/首次计算的初始化时间。
- **保证 MIG/vGPU 配置不丢失**：这些特性依赖驱动常驻，持久模式关闭时配置可能在驱动卸载后重置。
- **维持 NVLink/NVSwitch 状态**：多 GPU 拓扑对驱动状态敏感，频繁加载/卸载可能导致 link 状态异常。

> 注：消费级显卡（GeForce 系列）通常对此不敏感，但也有依赖驱动常驻的场景。
> 数据中心/服务器显卡（Tesla/A-series/H-series/L-series）**强烈建议开启**。

## 查看当前状态

```shell
# 方式 1：通过 nvidia-smi 查看（推荐）
nvidia-smi -q -d PERSISTENCE
# 或快速查看
nvidia-smi -q | grep -i persistence

# 方式 2：直接看 nvidia-smi 顶部输出
nvidia-smi
# 输出示例（注意 Persistence-M 行）：
# +-----------------------------------------------------------------------------+
# | NVIDIA-SMI 550.54.15    Driver Version: 550.54.15    CUDA Version: 12.4     |
# |-------------------------------+----------------------+----------------------+
# | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
# | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
# |===============================+======================+======================|
# |   0  NVIDIA L40S        Off  | 00000000:01:00.0 Off |                    0 |
# ...
# Persistence-M 列为 Off 即未开启，On 即已开启。
```

## 临时开启（重启后失效）

```shell
# 开启 Persistence Mode（对所有 GPU）
sudo nvidia-smi -pm 1

# 对特定 GPU 开启（如 GPU 0 和 GPU 1）
sudo nvidia-smi -i 0 -pm 1
sudo nvidia-smi -i 1 -pm 1

# 关闭 Persistence Mode
sudo nvidia-smi -pm 0
```

执行后建议运行 `nvidia-smi` 确认 `Persistence-M` 列显示 **On**。

> 这种方式**仅在当前运行周期内有效**，系统重启后恢复为默认的 Off 状态。

## 永久生效：使用 nvidia-persistenced 服务

### 方法 1：systemd 服务（推荐，适用于 systemd 发行版）

NVIDIA 驱动自带的 `.run` 安装包会在 `/usr/bin/` 下安装 `nvidia-persistenced`，
并通过 `nvidia-persistenced.service` 注册系统服务。

```shell
# 1. 确认 nvidia-persistenced 二进制存在
which nvidia-persistenced
# 通常位于 /usr/bin/nvidia-persistenced

# 2. 如果二进制不存在（部分包管理器安装方式不包含此文件），
#    可以从 NVIDIA 驱动包中手动提取安装。

# 3. 创建 systemd 服务文件（如果不存在）
sudo tee /usr/lib/systemd/system/nvidia-persistenced.service << 'EOF'
[Unit]
Description=NVIDIA Persistence Daemon
Wants=syslog.target

[Service]
Type=forking
ExecStart=/usr/bin/nvidia-persistenced --user nvidia-persistenced
ExecStopPost=/bin/rm -rf /var/run/nvidia-persistenced

[Install]
WantedBy=multi-user.target
EOF

# 4. 创建运行服务的专用用户（如果不存在）
sudo useradd -r -s /sbin/nologin nvidia-persistenced 2>/dev/null || true

# 5. 启动服务并设为开机自启
sudo systemctl daemon-reload
sudo systemctl enable nvidia-persistenced
sudo systemctl start nvidia-persistenced

# 6. 检查服务状态
sudo systemctl status nvidia-persistenced
```

### 方法 2：rc.local / crontab（非 systemd 或简易方案）

```shell
# 通过 /etc/rc.local（需确保该文件有执行权限且 rc-local 服务已启用）
sudo tee -a /etc/rc.local << 'EOF'
#!/bin/bash
/usr/bin/nvidia-smi -pm 1
EOF
sudo chmod +x /etc/rc.local

# 或者通过 crontab @reboot
(sudo crontab -l 2>/dev/null; echo "@reboot /usr/bin/nvidia-smi -pm 1") | sudo crontab -
```

> 注意：`rc.local` 和 crontab 方式本质上是开机执行 `nvidia-smi -pm 1`，
> 与 `nvidia-persistenced` 守护进程略有差异。后者具备故障恢复能力和更完善的生命周期管理，
> 推荐优先使用 `nvidia-persistenced`。

## 从驱动 .run 包提取 nvidia-persistenced

如果你的驱动是通过包管理器（如 apt、yum）安装的，可能不包含 `nvidia-persistenced`。
可以从 NVIDIA 官网下载 `.run` 驱动包手动提取：

```shell
# 1. 下载 .run 包（以 550.54.15 为例）
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/550.54.15/NVIDIA-Linux-x86_64-550.54.15.run

# 2. 解压（不安装）
chmod +x NVIDIA-Linux-x86_64-550.54.15.run
./NVIDIA-Linux-x86_64-550.54.15.run -x

# 3. 提取 nvidia-persistenced
# 解压后文件位于同名目录下
cp NVIDIA-Linux-x86_64-550.54.15/nvidia-persistenced /usr/bin/
chmod +x /usr/bin/nvidia-persistenced
```

## Windows 下的等效设置

在 Windows 系统中，Persistence Mode 同样通过 `nvidia-smi` 控制，并可通过任务计划程序
实现开机自启：

```powershell
# 临时开启（管理员 PowerShell）
nvidia-smi -pm 1

# 永久开启：创建计划任务，开机时以 SYSTEM 权限执行
schtasks /create /tn "NVIDIA Persistence Mode" /tr "nvidia-smi -pm 1" /sc ONSTART /ru SYSTEM /rl HIGHEST
```

## 验证与监控

```shell
# 确认 Persistence Mode 已开启
nvidia-smi -q -d PERSISTENCE | head -10
# 预期输出：
#     Persistence Mode
#         Status          : Enabled

# 确认 nvidia-persistenced 进程在运行
ps aux | grep nvidia-persistenced
# 预期应有类似一行：
# nvidia-persistenced ... /usr/bin/nvidia-persistenced ...
```

## 常见问题

|问题|可能原因|解决|
| -------------------------------| ------------------------------| ----------------------------------------|
|`nvidia-smi -pm 1` 提示权限不足|非 root 执行|加 `sudo`|
|重启后 Persistence-M 又变回 Off|未配置开机自启|参照上文配置 systemd 服务|
|`nvidia-persistenced` 命令不存在|包管理器安装的驱动不包含此工具|从 .run 包提取或改用 rc.local 方案|
|服务启动失败|用户或目录权限不正确|检查 `/var/run/nvidia-persistenced` 是否存在且可写|
|开启后仍然掉卡|硬件问题或驱动 bug|检查 `dmesg`，更新驱动版本，检查 GPU 供电和散热|

## 参考资料

- [NVIDIA Persistence Daemon — README](https://download.nvidia.com/XFree86/nvidia-persistenced/)
- [NVIDIA Driver Documentation — Persistence Mode](https://docs.nvidia.com/deploy/driver-persistence/index.html)
- [nvidia-smi 手册](https://developer.download.nvidia.com/compute/DCGM/docs/nvidia-smi-367.38.pdf)
