
---
title: ESXi 8.0 All-in-One 避坑实录：从 128G 单盘安装到 OpenWrt 直通配置
date: 2025-11-24 20:00:00
tags: [ESXi, OpenWrt, HomeLab, 虚拟化, OpenClash]
categories: 运维
---

最近入手了一台小主机，安装两块X340T2万兆网卡，计划用一块 128G 的固态硬盘搭建 ESXi 8.0 U2 并运行 OpenWrt 作为主路由。虽然硬件配置简单，但 ESXi 8.0 的默认分区策略和直通配置还是有不少坑。本文记录了从安装到网卡直通、以及 OpenClash 配置的全过程。

## 1. ESXi 8.0 U2 安装与空间拯救

ESXi 8.0 引入了新的分区机制，默认会预留约 128G 的 `OSData` 分区。对于只有 128G 硬盘的设备来说，这意味着安装完系统后将**没有任何空间创建 Datastore（存储）** 来存放虚拟机。

### 准备工作
* **镜像下载**：[H3C VMware ESXi 镜像下载](https://www.h3c.com/cn/Service/Document_Software/Software_Download/Server/Catalog/system/system/VMware/)
* **制作启动盘**：使用 [Rufus](https://rufus.ie/) 将 ISO 写入 U 盘。

### 关键步骤：修改分区大小
1.  插入 U 盘启动，在出现 ESXi 倒计时加载画面（黑底白字）时，**立刻按下 `Shift + O`**。
2.  在命令行末尾输入空格，追加以下参数：
    ```bash
    systemMediaSize=min
    ```
    > **注意**：`min` 模式会将系统占用限制在 33GB 左右，从而为虚拟机留出约 90GB 的可用空间。如果不加此参数，128G 硬盘将无法创建存储。

3.  按回车继续安装，后续按照提示设置 Root 密码并完成安装。

---

## 2. ESXi 初始化配置

### BIOS 设置
进入主板 BIOS，确保开启以下虚拟化选项，否则无法通过硬件直通：
* **VT-x / VT-d** (Intel) 或 **SVM / IOMMU** (AMD)
* **SR-IOV** (如果有)

### ESXi 后台配置
1.  **激活与存储**：登录 Web 后台，输入许可证，并确认 `datastore1` 是否已自动创建（约 80-90GB）。
2.  **开启直通 (Passthrough)**：
    * 进入 `管理` -> `硬件` -> `PCI 设备`。
    * 找到用于 WAN 口的物理网卡，点击 `切换直通`。
    * **避坑指南**：**千万不要直通管理网口（连接电脑的那个口）**，否则后台会失联。

### 疑难杂症：直通报错修复
如果切换直通时报错 `Cannot configure PCI-Passthrough on incapable device`（常见于消费级网卡或多功能设备），需 SSH 登录 ESXi 修改配置文件：

```bash
vi /etc/vmware/passthru.map
````

在文件末尾根据你的设备 ID（在 PCI 设备列表中查看 Vendor ID 和 Device ID）添加白名单，例如：

```text
# <标记>  <Vendor ID>  <Device ID>  <重置方法>  <默认操作>
d3d0     8086         15b8         d3d0        default
```

保存后重启 ESXi 即可生效。

-----

## 3\. OpenWrt 虚拟机安装

### 镜像转换

1.  下载 OpenWrt 固件（推荐 ext4 格式）。
2.  下载 [StarWind V2V Converter](https://www.starwindsoftware.com/starwind-v2v-converter)。
3.  将 `.img.gz` 解压后的 `.img` 文件转换为 ESXi 专用的 `.vmdk` 文件。

### 虚拟机创建

1.  **创建虚拟机**：客户机操作系统选择 `Linux` -\> `其他 Linux (64位)`。
2.  **删除默认硬盘**：删除自带的硬盘，重新添加“现有硬盘”，上传转换好的 `.vmdk` 文件。
3.  **调整容量**：
      * 在 ESXi 中将硬盘大小修改为 1GB（或其他你想要的大小）。
      * 为了让 OpenWrt 识别到扩容的空间，需要挂载一个 GParted ISO 镜像引导启动，在图形化界面中将分区拉大（Resize），应用更改后关机。
4.  **硬件直通挂载**：
      * **内存预留（必须）**：在 `编辑设置` -\> `内存` 中，勾选 **“预留所有客户机内存”**。否则直通设备会导致开机失败。
      * **添加 PCI 设备**：选择之前直通好的物理网卡。

-----

## 4\. OpenWrt 网络配置

https://kvcb.me/2019/04/04/OpenWrt/  ; https://github.com/wangyu-/tinyfecVPN/wiki/%E7%94%A8openwrt%E8%B7%AF%E7%94%B1%E5%99%A8%E6%90%AD%E5%BB%BA%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86%EF%BC%8C%E5%8A%A0%E9%80%9F%E5%B1%80%E5%9F%9F%E7%BD%91%E5%86%85%E6%89%80%E6%9C%89%E8%AE%BE%E5%A4%87 ;
https://flandre-scarlet.moe/blog/2016/

此时 OpenWrt 拥有两个网口：

  * **eth0** (虚拟口)：连接 ESXi 的 vSwitch，作为 LAN 口管理。
  * **eth1** (直通口)：直接连接物理网线，作为 WAN 口拨号。

### 修改 IP 地址

通过 ESXi 的控制台（VNC）进入 OpenWrt 命令行：

```bash
vi /etc/config/network
```

配置 LAN 口（根据自家局域网规划）：

```bash
config interface 'lan'
        option device 'eth0'
        option proto 'static'
        option ipaddr '192.168.0.138'    # 设置为你的管理IP
        option netmask '255.255.255.0'
        option gateway '192.168.0.1'     # 上级网关（如果是旁路模式）
        list dns '223.5.5.5'             # 保证能解析域名下载插件
```

重启网络服务：

```bash
service network restart
```

### 绑定 WAN 口

登录 OpenWrt Web 后台：

1.  进入 `网络` -\> `接口` -\> `WAN`。
2.  在 `物理设置` 中选择直通的 **eth1** 接口。
3.  协议选择 `PPPoE`（拨号）或 `DHCP`（光猫路由模式）。

-----

## 5\. 配置 OpenClash

网络通畅后，安装 OpenClash 实现网络分流。

1.  **依赖安装**：先更新软件包列表（如果使用的是非全功能版固件）。
    ```bash
    opkg update
    opkg install libcap-bin ruby-yaml
    ```
2.  **上传插件**：通过 SFTP 将 OpenClash 的 `.ipk` 包上传至 `/tmp` 目录并安装：
    ```bash
    opkg install /tmp/luci-app-openclash*.ipk
    ```
3.  **内核下载**：
      * 进入 OpenClash 插件设置，由于网络环境可能无法直接下载内核，建议手动下载 `Dev` 和 `Tun` 内核，上传至 `/etc/openclash/core/` 目录并赋予执行权限 `chmod +x`。
4.  **订阅配置**：导入订阅链接，开启 `Fake-IP` 模式以获得最佳性能。

-----

## 总结

通过 `systemMediaSize=min` 参数，我们成功在 128G 硬盘上“抠”出了足够的空间；通过修改 `passthru.map` 和内存预留，解决了网卡直通的各种疑难杂症。这套方案非常适合利用闲置小主机搭建高性能的家庭网络中心。


