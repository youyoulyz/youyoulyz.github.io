---
title: 上海科技大学校园网 OpenWrt 配置指南
date: 2026-05-08 15:00:00
tags:
  - OpenWrt
  - 校园网
  - IPv6
  - 上海科技大学
categories:
  - 网络
---

## 前言

上海科技大学（ShanghaiTech）的校园网采用 Web Portal 认证方式，设备需要先通过浏览器登录才能上网。对于希望使用 OpenWrt 路由器的用户来说，认证流程和 IPv6 配置都有不少坑。本文记录了完整的配置过程，包括：

- 校园网 Web 认证的解决方案（MAC 地址克隆）
- IPv6 配置（ISP 不下发前缀委派的处理）
- OpenClash Fake-IP 模式下的 DNS 问题

<!-- more -->

## 网络环境说明

| 项目 | 值 |
|------|-----|
| 认证方式 | Web Portal（浏览器登录） |
| WAN 口获取方式 | DHCP（IPv4）+ DHCPv6（IPv6） |
| IPv6 前缀 | ISP 不下发前缀委派，仅分配单个 /64 地址 |
| 路由方式 | 源地址限定路由（source-specific routing） |
| 内网 DNS | 10.15.44.11 |
| 内网网段 | 10.x.x.x / 192.168.x.x |

## 一、基础网络配置

### 1.1 WAN 口配置

编辑 `/etc/config/network`：

```config
config interface 'wan'
    option device 'eth0'
    option proto 'dhcp'

config interface 'WAN6'
    option proto 'dhcpv6'
    option device 'eth0'
    option force_link '1'
    option reqaddress 'try'
    option reqprefix 'auto'
    option norelease '1'
```

### 1.2 LAN 口配置

```config
config interface 'lan'
    option proto 'static'
    option ipaddr '192.168.1.1'
    option netmask '255.255.255.0'
    option device 'br-lan'
    option ip6assign '64'
    option ip6ifaceid 'eui64'
```

## 二、校园网 Web 认证：MAC 地址克隆

上海科技大学的校园网采用 Web Portal 认证。路由器直接连接时，由于无法弹出浏览器登录页面，会导致无法上网。

### 2.1 认证原理

校园网交换机通过识别设备的 MAC 地址来判断是否已认证。如果我们把路由器 WAN 口的 MAC 地址改成一台已经认证过的 PC 的 MAC 地址，就能"继承"该 PC 的认证状态。

### 2.2 操作步骤

**第一步：用 PC 正常登录校园网**

1. 将 PC 直接连入校园网（有线或无线）
2. 打开浏览器，访问任意网页
3. 按照重定向页面完成 Web 认证登录
4. 确认 PC 可以正常上网

**第二步：获取 PC 的 MAC 地址**

在 PC 上执行：

```bash
# Linux
ip link show | grep ether

# Windows (CMD)
ipconfig /all | findstr "Physical Address"

# macOS
ifconfig | grep ether
```

记录下 PC 有线网卡的 MAC 地址，例如：`fc:34:97:0d:94:35`

**第三步：在 OpenWrt 上克隆 MAC 地址**

编辑 `/etc/config/network`，在 WAN 口配置中添加 `macaddr` 选项：

```config
config interface 'wan'
    option device 'eth0'
    option proto 'dhcp'
    option macaddr 'fc:34:97:0d:94:35'
```

**第四步：断开 PC 的校园网连接**

> ⚠️ **重要**：PC 和路由器不能同时使用同一个 MAC 地址连接校园网，否则会导致 MAC 冲突。

- 断开 PC 的校园网有线连接（或关闭 PC 的校园网 Wi-Fi）
- 将校园网网线插入路由器 WAN 口

**第五步：重启网络**

```bash
/etc/init.d/network restart
```

此时路由器应该可以正常获取 IP 地址并上网。

### 2.3 注意事项

- **认证过期**：校园网认证通常有时效性，过期后需要重新认证。可以编写脚本定期调用认证接口，或者在断网时重复上述步骤。
- **MAC 冲突**：确保 PC 不再使用同一 MAC 连接校园网。如果 PC 需要同时上网，可以使用路由器的 LAN 口或 Wi-Fi。
- **换设备**：如果更换了 PC，需要重新获取新 PC 的 MAC 地址并更新配置。

## 三、IPv6 配置

上海科技大学的校园网支持 IPv6，但 ISP（学校网络中心）**不会下发 IPv6 前缀委派**。这意味着：

- WAN 口可以获取到一个公网 IPv6 地址（`2001:da8:xxxx::/64`）
- 但 LAN 口无法获得公网 IPv6 前缀，子网设备无法直接获得公网 IPv6 地址

### 3.1 问题分析

通过 `ifstatus WAN6` 可以看到：

```json
"ipv6-prefix": [],
"ipv6-prefix-assignment": []
```

`ipv6-prefix` 为空，说明 ISP 没有下发前缀委派。

此外，学校的 IPv6 路由使用了**源地址限定路由**（source-specific routing）：

```
default from 2001:da8:801d:70a3::/64 via fe80::200:5eff:fe00:101 dev eth0
```

这条默认路由只对源地址在 `2001:da8:801d:70a3::/64` 网段的流量生效。如果我们给 LAN 分配 ULA 地址（`fd33::`），这些流量将没有默认路由。

### 3.2 解决方案：NAT66 + 通用默认路由

#### 步骤一：删除 WAN 口的 `delegate` 限制

编辑 `/etc/config/network`，确保 WAN 口没有 `delegate '0'`：

```config
config interface 'wan'
    option device 'eth0'
    option proto 'dhcp'
    option macaddr 'fc:34:97:0d:94:35'
    # 不要设置 option delegate '0'
```

#### 步骤二：给 LAN 口添加静态 IPv6 地址

在 `/etc/config/network` 的 LAN 口配置中添加：

```config
config interface 'lan'
    option proto 'static'
    option ipaddr '192.168.1.1'
    option netmask '255.255.255.0'
    option device 'br-lan'
    option ip6assign '64'
    option ip6ifaceid 'eui64'
    option ip6addr 'fd33:5b9c:a935::1/64'
```

#### 步骤三：添加通用默认 IPv6 路由

由于 ISP 的默认路由是源地址限定的，需要添加一条通用默认路由。

临时添加（重启后失效）：

```bash
ip -6 route add default via fe80::200:5eff:fe00:101 dev eth0 metric 512
```

持久化配置（推荐），在 `/etc/config/network` 中添加：

```config
config route6
    option interface 'wan'
    option target '::/0'
    option gateway 'fe80::200:5eff:fe00:101'
    option metric '512'
```

> ⚠️ **注意**：网关地址 `fe80::200:5eff:fe00:101` 是学校交换机的 link-local 地址，可能因接入交换机不同而变化。可以通过 `ifstatus WAN6` 查看当前的默认网关。

#### 步骤四：启用 IPv6 伪装（NAT66）

编辑 `/etc/config/firewall`，在 WAN zone 中添加 `masq6`：

```config
config zone
    option name 'wan'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'
    option masq '1'
    option masq6 '1'
    option mtu_fix '1'
    list network 'wan'
    list network 'wan6'
```

#### 步骤五：配置 DHCP RA

编辑 `/etc/config/dhcp`，确保 LAN 的 RA 配置正确：

```config
config dhcp 'lan'
    option interface 'lan'
    option start '100'
    option limit '150'
    option leasetime '12h'
    option dhcpv4 'server'
    option ra 'server'
    list ra_flags 'managed'
    option ra_default '1'
    option dhcpv6 'server'
```

关键选项说明：
- `ra 'server'`：开启 Router Advertisement 服务
- `ra_flags 'managed'`：设置 M 标志，告知客户端通过 DHCPv6 获取地址
- `ra_default '1'`：强制通告默认路由（即使没有公网前缀）

#### 步骤六：重启服务

```bash
/etc/init.d/network restart
/etc/init.d/odhcpd restart
/etc/init.d/firewall restart
```

### 3.3 验证 IPv6 连通性

在 LAN 内的设备上：

```bash
# 检查是否获得 ULA 地址
ip -6 addr show | grep "fd33:"

# 测试 IPv6 连通性
ping6 shanghaitech.edu.cn

# 测试 HTTP
curl -6 http://ipv6.test-ipv6.com/
```

### 3.4 odhcpd 的 RFC 9096 问题

odhcpd 实现了 RFC 9096 的逻辑：当 WAN 有默认路由但 LAN 没有公网前缀时，会将 RA 中的前缀 Valid/Pref lifetime 设为 0，导致客户端不会生成 IPv6 地址。

解决方法是给 LAN 口配置静态 IPv6 地址（如上文的 `ip6addr 'fd33:5b9c:a935::1/64'`），这样 odhcpd 会认为 LAN 有合法前缀，正常通告。

## 四、OpenClash 配置

### 4.1 基本配置

推荐使用 **Fake-IP** 模式，配合 **Enhanced Mode** 实现全局代理。

在 OpenClash 的配置文件 `/etc/config/openclash` 中：

```config
config openclash
    option enable '1'
    option en_mode 'fake-ip'
    option proxy_port '7892'
    option tproxy_port '7895'
    option mixed_port '7893'
    option dns_port '7874'
    option ipv6_enable '1'
```

### 4.2 DNS 配置

推荐在 OpenClash 的 nameserver 组中加入校园网内部 DNS：

```config
config dns_servers
    option enabled '1'
    option group 'nameserver'
    option type 'udp'
    option ip '10.15.44.11'
    option interface 'Disable'
    option direct_nameserver '0'
    option node_resolve '0'
```

这样可以确保校园内网域名（如 `shanghaitech.edu.cn`）能正确解析到内网地址。

### 4.3 Fake-IP 过滤

对于需要直连的校园内网域名，建议添加到 Fake-IP 过滤列表，避免走代理：

```config
config openclash
    option custom_fakeip_filter '1'
    list fakeip_filter 'shanghaitech.edu.cn'
    list fakeip_filter '*.shanghaitech.edu.cn'
```

## 五、常见问题

### 5.1 DNS 重绑定攻击告警

**症状**：`logread` 中出现大量 `possible DNS-rebind attack detected`，部分内网域名无法解析。

**原因**：dnsmasq 的 DNS 重绑定保护功能会拦截解析到内网 IP 地址的域名。对于校园内网服务（如 `egate-new.shanghaitech.edu.cn` → `10.15.44.75`），这会导致解析失败。

**解决**：在 `/etc/config/dhcp` 中添加重绑定例外：

```bash
uci add_list dhcp.@dnsmasq[0].rebind_domain='/shanghaitech.edu.cn/'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

### 5.2 IPv6 地址过期

**症状**：设备上的公网 IPv6 地址 `preferred_lft 0sec`（已过期）。

**原因**：ISP 的 DHCPv6 租约更新问题，或者前缀委派丢失。

**解决**：
1. 清除过期地址：`ip -6 addr flush dev enp7s0 scope global`
2. 重启网络连接让设备重新获取地址
3. 确保路由器的 NAT66 配置正确

### 5.3 OpenClash 代理后 IPv6 不通

**症状**：关闭 OpenClash 后 IPv6 正常，开启后 IPv6 失败。

**原因**：OpenClash 的 `openclash_mangle_v6` 链会将 IPv6 流量重定向到代理端口。如果代理服务器不支持 IPv6 或配置有误，会导致 IPv6 连接失败。

**解决**：
1. 检查 OpenClash 的 IPv6 代理配置
2. 确认代理节点支持 IPv6
3. 或者在 OpenClash 中禁用 IPv6 代理，让 IPv6 流量直连

### 5.4 认证过期后无法上网

**症状**：路由器突然无法上网，但设备仍连接正常。

**原因**：校园网 Web 认证过期。

**解决**：
1. 重新用 PC 登录校园网认证
2. 更新路由器 WAN 口 MAC 地址
3. 或者编写自动认证脚本（后续文章介绍）

## 六、最终配置文件参考

### /etc/config/network（关键部分）

```config
config interface 'wan'
    option device 'eth0'
    option proto 'dhcp'
    option macaddr 'fc:34:97:0d:94:35'

config interface 'lan'
    option proto 'static'
    option ipaddr '192.168.1.1'
    option netmask '255.255.255.0'
    option device 'br-lan'
    option ip6assign '64'
    option ip6ifaceid 'eui64'
    option ip6addr 'fd33:5b9c:a935::1/64'

config interface 'WAN6'
    option proto 'dhcpv6'
    option device 'eth0'
    option force_link '1'
    option reqaddress 'try'
    option reqprefix 'auto'
    option norelease '1'

config route6
    option interface 'wan'
    option target '::/0'
    option gateway 'fe80::200:5eff:fe00:101'
    option metric '512'
```

### /etc/config/dhcp（关键部分）

```config
config dnsmasq
    option domainneeded '1'
    option boguspriv '1'
    option rebind_protection '1'
    option rebind_localhost '1'
    option local '/lan/'
    option domain 'lan'
    list rebind_domain '/shanghaitech.edu.cn/'

config dhcp 'lan'
    option interface 'lan'
    option start '100'
    option limit '150'
    option leasetime '12h'
    option dhcpv4 'server'
    option ra 'server'
    list ra_flags 'managed'
    option ra_default '1'
    option dhcpv6 'server'
```

### /etc/config/firewall（关键部分）

```config
config zone
    option name 'wan'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'
    option masq '1'
    option masq6 '1'
    option mtu_fix '1'
    list network 'wan'
    list network 'wan6'
```

## 七、总结

在上海科技大学使用 OpenWrt 的核心挑战在于：

1. **Web 认证**：通过 MAC 地址克隆绕过
2. **无前缀委派**：使用 NAT66 + ULA 地址解决
3. **源地址限定路由**：添加通用默认路由解决
4. **RFC 9096**：给 LAN 配置静态 IPv6 地址解决
5. **DNS 重绑定**：添加域例外解决

希望这篇文章能帮助同样在上海科技大学使用 OpenWrt 的同学们少走弯路。

---

*本文基于 OpenWrt 24.10.4 (x86_64) 编写，其他版本可能存在差异。*
