---
title: "Linux 磁盘分区、格式化与自动挂载指南"
published: 2026-08-04
description: "本文详细记录在 Linux 系统中使用 parted 分区、mkfs 格式化及通过 UUID 配置开机自动挂载的完整操作流程。"
category: "存储与磁盘"
tags:
  - "Linux"
  - "磁盘分区"
  - "ext4"
  - "fstab"
  - "自动挂载"
---

# Linux 磁盘分区、格式化与自动挂载指南

## 查看硬盘

使用 `lsblk` 命令，列出所有设备，查看需要挂载硬盘的物理路径。

本例中需要挂载的硬盘为 `/dev/sdb`。

## 硬盘分区

选择需要分区的硬盘，这里使用 `parted` 工具。

```bash
parted /dev/sdb
```

可以输入 `?` 来查看 parted 命令帮助。

创建磁盘分区表：

```bash
mklabel gpt
```
输入 `yes` 确认。

创建分区，并分配大小：

```text
mkpart
分区名（回车）
文件系统（ext4 或其他）
分区起始点 1MiB
分区结束点 100%
```

直接使用 `mkpart` 命令不加参数进行分区时，起始点和结束点默认单位为 MiB，可以使用 `unit GB` 命令来修改默认单位为 GB。起始点位置前建议预留一些空间，不要从 0 开始。

如果设置错误，可以使用 `rm <分区编号>` 命令删除，分区编号可以使用 `print` 查看。

完成后使用 `quit` 退出。

## 格式化

```bash
mkfs.ext4 /dev/sdb1
```

此处选择格式化为 ext4 文件系统，根据实际需求选择其他文件系统亦可。若提示文件系统已存在，可添加 `-F` 选项强制覆盖。

## 挂载

- **创建挂载点**

  ```bash
  mkdir /data
  ```

- **临时挂载**

  ```bash
  mount /dev/sdb1 /data
  ```

  使用 `df -h` 命令可以查看到 `/dev/sdb1` 分区已经挂载到了 `/data` 目录。

- **取消挂载**

  `umount /dev/sdb1` 或 `umount /data` 均可。临时挂载在重启后会失效。

- **设置开机自动挂载硬盘**

  自动挂载建议使用对应分区的 UUID，以防止设备名变更。UUID 唯一标识每个分区，可确保不会发生错误挂载。

  使用 `blkid /dev/sdb1` 查看分区 UUID：

  ```text
  /dev/sdb1: UUID="8efdf108-963f-476d-8992-581d37b13909" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="0b455b11-df18-47b6-a2ab-441848562ae8"
  ```

  复制其中的 UUID 值（例如 `8efdf108-963f-476d-8992-581d37b13909`）。

  编辑 `/etc/fstab` 文件，将参数写入以实现自动挂载：

  ```bash
  echo 'UUID=8efdf108-963f-476d-8992-581d37b13909 /data ext4 defaults 0 0' | sudo tee -a /etc/fstab
  ```

  上述命令会将参数追加至 `/etc/fstab` 文件末尾。写入后请注意检查是否有误，也可直接使用 `vim /etc/fstab` 手动编辑。

  `/etc/fstab` 文件包含以下六项参数：

  | 列序 | 参数说明 |
  | :--- | :--- |
  | 第一列 | 设备文件或 UUID / Label |
  | 第二列 | 设备的挂载点（挂载目录） |
  | 第三列 | 文件系统格式（可使用 `auto` 自动识别） |
  | 第四列 | 文件系统挂载参数 |
  | 第五列 | dump 备份设置（`0` 表示不备份，`1` 表示每天备份，`2` 表示不定期备份，通常设为 `0`） |
  | 第六列 | 磁盘检查设置（`0` 表示不检查） |

## 验证

执行 `df -h` 可查看挂载状态。

若此前未执行 `mount /dev/sdb1 /data`，此时应无法看到挂载信息，因为内核尚未读取 `/etc/fstab` 文件。此时可执行 `mount -a`，随后再次执行 `df -h`，即可看到 `/dev/sdb1` 已成功挂载至 `/data` 目录。

系统重启后，同样会观察到 `/dev/sdb1` 分区自动挂载至 `/data`。

## 扩容

相关操作请参考官方文档：[华为云 DSS 扩容指南](https://support.huaweicloud.com/usermanual-dss/dss_01_2311.html)
