---
Toc: true
comments: true
title: Homelab AIO 及网络配置
Date: 2024-08-03 11:15:10
categories: [生活, Linux]
image:
  path: /assets/img/homelab/new_homelab.png
  alt: homepage 示意图
---

经历了将近一年的时间，我不断对 Homelab 进行修改，最终变成了像图片里这个样子的 All in One/Boom！

## 配置清单

1. NUC9 i7：成品机，可插 3 块 PCIE 固态硬盘，另有两个 PCIE 插槽
    * CPU：i7-9850H
    * 内存【加装】：64GB
    * 硬盘【加装】：3 x 4T 固态硬盘
    * 网卡【加装】：EB-LINK intel 82599芯片PCI-E X4 10G万兆单光口光纤网卡X520-DA1
2. AIO 主力机，组装而成
    * CPU：AMD 7950x3D
    * 内存：64GB
    * 主板：X670E
    * 散热：AXP90
    * 硬盘：2T 固态 + 之前买的不同容量的机械硬盘
    * 网卡：10G + 2.5G
3. ArmKVM：成品
4. 交换机：4x2.5G + 2x10G 光口

## 网络

入户宽带是 1G 的，且家里现有网络环境均是 1G 的，已经足够日常使用了，但是考虑到 AIO 与 Nas 的数据访问备份速度，最终还是在 AIO 与 Nas 之间通过万兆交换机相连，尽可能小范围的调整了网络环境。

但是还有一个比较重要的问题，这边移动家庭宽带虽然给 IPV6 公网，但是一个入户只能给共计 8 个 IP 地址，超过 8 个就会面临着不可控的断网等问题，因此没有办法简单粗暴的直连所有设备。

考虑到 PVE 中并不是所有的虚拟机都需要公网 IPV6，配置 NAT 成为了必要。

### PVE 配置 NAT

> 更详细可参考 [Proxmox VE 配置 NAT IPv4+IPv6、分发独立 IPv6 之网络配置模板和理解](https://blog.skyju.cc/post/proxmox-ipv4-nat-ipv6/)

总体可以分为以下步骤：

#### 1. PVE Host 添加用于 NAT 的网卡

默认情况下会有 vmbr0 与 Host 的网卡绑定，我们只需要添加新的 vmbr1 网卡，并将 vmbr1 的网络流量全部通过 vmbr0 对外即可：

```conf
auto vmbr1

# 下面是 IPV4 配置，10.10.0.1/24 替换成自己的即可
iface vmbr1 inet static
    address 10.10.0.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
    post-up   iptables -t nat -A POSTROUTING -s '10.10.0.0/24' -o vmbr0 -j MASQUERADE
    post-down iptables -t nat -D POSTROUTING -s '10.10.0.0/24' -o vmbr0 -j MASQUERADE

# 下面是 IPV6 配置，2001:db8:1::1/64 替换成自己的即可
iface vmbr1 inet6 static
    address 2001:db8:1::1/64
    post-up sysctl -w net.ipv6.conf.all.forwarding=1
    post-up ip6tables -t nat -A POSTROUTING -s 2001:db8:1::/64 -o vmbr0 -j MASQUERADE
    post-down sysctl -w net.ipv6.conf.all.forwarding=0
    post-down ip6tables -t nat -D POSTROUTING -s 2001:db8:1::/64 -o vmbr0 -j MASQUERADE
```
{: file='/etc/network/interfaces'}

#### 2. PVE 虚拟机配置网络

以 Ubuntu 为例。首先需要将 vmbr1 网络添加到虚拟机中，之后修改配置文件：

```yaml
# 网卡 ens19 替换成对应的名称
network:
    ethernets:
        ens19:
            addresses:
            - 10.10.0.202/24
            - 2001:db8:1::2/64
            nameservers:
              addresses: [2400:3200::1]
            routes:
                - to: default
                  via: 10.10.0.1
    version: 2
```
{: file='/etc/netplan/50-cloud-init.yaml'}

由于没有配置 DHCP，每台虚拟机的 ipv4 和 ipv6 地址均需要手动配置才可以。

#### 3. 宿主机配置 DHCP

首先安装 dhcp-server：

```shell
sudo apt install isc-dhcp-server
```

其次编辑配置文件`/etc/dhcp/dhcpd.conf`：

```conf
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option routers 192.168.1.1;
  option domain-name-servers 8.8.8.8, 8.8.4.4;
  default-lease-time 600;
  max-lease-time 7200;
}
```
{: file='/etc/dhcp/dhcpd.conf'}

同时修改配置文件`/etc/default/isc-dhcp-server`：
```conf
INTERFACESv4="vmbr1"
```
{: file='/etc/default/isc-dhcp-server'}

#### 4. 简单一点的方法

1. PVE 宿主机确保有两个网口，没有就加装一个
2. 网口一直接接入光猫，自动获取 IPV6，所有需要 IPV6 公网的虚拟机全部添加该网卡
3. 网口二直接接入家中的二级路由器/WIFI等，不需要公网的虚拟机添加该网卡即可

## 家庭服务器

* Nas：主要存储、备份重要数据
    1. 所有跑在 AIO 中的、有价值的 docker 数据等
        * 目前绝大部分的服务我都是用 docker + 映射存储目录运行的，可以很方便的直接将映射的存储目录打包压缩并备份到 Nas 上
        * PVE 虚拟机备份，为了防止某一天 PVE 系统无法启动等问题，每天凌晨定时备份虚拟机到 Nas 上
        * 重要文件单独备份，包括 KodBox 的网盘文件、immich 的相册文件等
        * MacBook 的时间机器备份盘
    2. 运行这一些比较重要且不需要经常调整的服务
        * AdGuard-Home，家里的 WIFI 目前都是接入它了
        * tailscale，异地组网
* AIO：各种虚拟机，硬盘是单独穿透到每个虚拟机的（硬盘可以选择是否被备份）
    1. Windows 11 + 4060 passthrough：应对日常需要 Windows 的场景，还能玩一玩植物大战僵尸
    2. Ubuntu + 4060 passthrough：应对需要显卡的服务，如 emby、immich 等；还有其他大部分服务都在这里
    3. Ubuntu：跑了一些关键的、需要稳定一点的服务，如 JumpServer 等
* ArmKVM：直连 AIO，专门负责控制 AIO 服务器，防止机器故障需要人工介入、调整 BIOS 等情况

## 现在还存在的一些问题

1. Truenas 部分：
    1. 重启后经常无法一次性启动成功，需要再启动一次才行，后续还需要想办法解决一下
2. AIO 部分：
    1. 功耗实在有些高，日常低负载情况下 120w 左右，算上 Truenas 部分，全部功耗到 150w 了，但是目前也没什么好办法降低一下功耗，尽管电费也不要多少就是了
    2. 积热问题有些严重，装机器的时候失误用了 AXP90，但是实际上可以上 AXP120 的，导致日常低负载情况下 CPU 温度 50 度，PVE 用 stress 跑满 CPU 长时间顶温度墙，且全核仅能维持在 4.0 GHZ，用 GeekBench 6 跑分单核/多核和一般跑分比低了至少 8% 左右，而且主板上的各个散热装甲完全无法得到有效散热，每次停机拆机维护的时候装甲都很烫手，还有中间那块显卡，温度甚至会隔着空气传到机箱外壳 =.=
    3. 正中间的那个 PCI 挡板，由于下面主板的散热装甲，实际上啥也不能装，本来是想装一个抽风风扇的，结果装不上，只能换到两块显卡之间
    4. 机箱：
        1. 前方三个 8cm 风扇受限于大小风量有限，CPU 附近的热量基本吹不开
        2. 机箱正面的 USB 及 typec 的线，c 口线与 USB 线是一根，且两个接口之间的线长大概也就 10cm左右，主板上 USB 接口及 c 接口需要在一起才能插上，否则只能舍弃其中一个
        3. 机箱外壳在底部还有两个螺丝，每次拆外壳都需要把机器翻过来才能拆开

## 未来

1. 显卡虚拟化：需要一个能够虚拟化的显卡同时穿透到多个虚拟机里。目前两块 4060 说实话有点用不太上
2. 散热改进：准备换 12cm 撒热风扇，看看能不能好一些
3. 备份 NAS：单有一个 NAS 不太够，准备再整一个配置低一些的 NAS 做异地备份