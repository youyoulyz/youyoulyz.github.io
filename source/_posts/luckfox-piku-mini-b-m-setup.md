---
title: Luckfox Pico Mini B/M开发板SD卡刷写与入门指南
date: 2026-03-11 18:00:00
tags: [LuckFox, RV1103, Embedded]
categories: 嵌入式
---

Luckfox Pico Mini B/M是一款基于瑞芯微RV1103芯片的迷你Linux开发板，售价约$9，集成64MB DDR2内存、单核ARM Cortex-A7处理器和0.5TOPS NPU。由于其仅有128MB SPI Flash，日常开发推荐使用SD卡启动。本文记录从SD卡刷写到USB调试连接的完整过程。

## 硬件规格概览

| 项目 | 规格 |
|------|------|
| 处理器 | ARM Cortex-A7 @ 1.2GHz + RISC-V |
| NPU | 0.5TOPS，支持int4/int8/int16 |
| 内存 | 64MB DDR2 |
| 存储 | 128MB SPI Flash（板载）/ SD卡槽 |
| USB | USB 2.0 Host/Device |
| 相机接口 | MIPI CSI 2-lane |
| GPIO | 17路GPIO |

## 系统镜像获取

### 官方预编译镜像

Luckfox官方提供Ubuntu和Buildroot的预编译SD卡镜像：

**下载链接：** https://drive.google.com/drive/folders/14kFWY93MZ4Zga4ke2PVQgUs1y9xcMG0S

常用镜像包括：
- `Ubuntu_Luckfox_Pico_Mini_B_MicroSD_250313.zip` - Ubuntu 22.04系统
- Buildroot镜像（需自行从SDK编译或使用社区提供版本）

### 其他可用系统

| 系统 | 内存占用 | 适用场景 |
|------|---------|----------|
| Buildroot | 低 | 嵌入式最小化系统 |
| Ubuntu 22.04 | 中等至高 | 全功能开发环境 |
| Foxbuntu | 中等 | 低功耗网络应用（Meshtastic、LoRa）|
| Alpine Linux | 极低 | 安全导向极简系统 |

## SD卡刷写方法

### 准备工作

1. **下载镜像压缩包**并解压到工作目录
2. **确认SD卡设备路径**，在Linux中运行：
   ```bash
   $ lsblk
   ```
   插入SD卡后找到对应设备，通常为/dev/sdb或/dev/sdc

3. **获取刷写工具** `blkenvflash.py`（官方镜像压缩包内已包含）

### 使用blkenvflash.py工具刷写

此工具专用于将多个.img分区文件按正确偏移量写入SD卡：

```bash
$ cd Ubuntu_Luckfox_Pico_Mini_B_MicroSD_250313
$ sudo python3 blkenvflash.py /dev/sdb
```

输出示例：
```
== blkenvflash.py 0.0.1 ==
writing to /dev/sdb
   mmcblk1: env.img size:32,768/32K (offset:0/0B) imgsize:32,768 (32K)
   mmcblk1: idblock.img size:524,288/512K (offset:32,768/32K) imgsize:188,416 (184K)
   mmcblk1: uboot.img size:262,144/256K (offset:0/0B) imgsize:262,144 (256K)
   mmcblk1: boot.img size:33,554,432/32M (offset:0/0B) imgsize:3,150,336 (3,150,336B)
   mmcblk1: oem.img size:536,870,912/512M (offset:0/0B) imgsize:113,909,760 (111,240K)
   mmcblk1: userdata.img size:268,435,456/256M (offset:0/0B) imgsize:9,999,360 (9,765K)
   mmcblk1: rootfs.img size:6,442,450,944/6G (offset:0/0B) imgsize:1,136,914,432 (1,110,268K)
done.
```

> ⚠️ **重要提示：**
> - 参数必须是SD卡设备路径（如/dev/sdb）**而非分区**（如/dev/sdb1）
> - 大文件刷写耗时较长，请耐心等待，进程卡住是正常的
> - 必须从包含所有.img文件的目录运行此脚本
> - SDK编译生成的镜像同样适用此工具刷写

### SD卡分区结构说明

| 文件 | 大小 | 描述 |
|------|------|------|
| idblock.img | 184KB | Rockchip Bootloader识别块 |
| download.bin | 263KB | Rockchip专用底层引导程序 |
| uboot.img | 256KB | U-Boot主引导加载程序 |
| env.img | 32KB | U-Boot环境配置 |
| boot.img | 3.4MB | Linux内核+设备树 |
| rootfs.img | 195MB | 根文件系统 |
| oem.img | 43MB | OEM厂商数据分区 |
| userdata.img | 9.6MB | 用户数据分区 |

## 首次启动与连接

### 启动优先級

- **有SD卡插入**：从SD卡启动（推荐）
- **无SD卡**：从板载128MB SPI Flash启动

> 💡 实测性能：SD卡读取速度约21.87 MB/s，SPI Flash仅4.40 MB/s，强烈建议使用SD卡

### USB RNDIS网络连接

Pico通过USB提供RNDIS网络接口实现调试连接：

```bash
# 插入USB后查看新接口
$ ifconfig
enxae935ce313b2: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.32.0.100  netmask 255.255.0.0 ...
```

**配置步骤：**
1. 将USB RNDIS接口IP设置为`172.32.0.100/16`（与Pico同一网段）
2. Pico默认IP为`172.32.0.70`（Ubuntu镜像）或`172.32.0.93`
3. 如需上网，在主机上开启**网络共享**给该接口
4. 测试连接：
   ```bash
   $ ping 172.32.0.70
   ```

### SSH登录与文件传输

```bash
# SSH登录（Ubuntu镜像默认用户 pico）
$ ssh pico@172.32.0.70

# SFTP方式传输文件
$ scp file.txt pico@172.32.0.70:/home/pico/
```

### ADB调试连接

```bash
$ adb device
$ adb shell
```
配置好了权限自动会连接。





**参考链接：**
- [Luckfox Pico Mini B 原始文档](https://github.com/themrleon/luckfox-pico-mini-b)
- [官方SDK](https://github.com/LuckfoxTECH/luckfox-pico)
