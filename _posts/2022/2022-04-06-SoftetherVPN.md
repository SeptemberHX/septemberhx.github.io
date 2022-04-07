---
title: 🔐 Softether VPN 搭建及使用
date: 2022-04-06 21:10:30 +08:00
tags:
- Linux
- 软件
categories: [手册, Linux]
toc: true
comments: true
---

在一些诸如公司、学校实验室、部分家用网络等内网环境下，部分服务器、NAS、Emby等由于没有公网 IP 导致离开内网环境下无法访问，此时需要搭建 VPN、端口转发等方式来实现内网外访问。考虑到一些 frp 等端口转发工具不关注用户权限、zerotier 及 n2n 等又不能细粒度的控制用户请求，这里选用功能较为全面的 Softether VPN。

> 如果只是个人使用，仅想访问内网对应服务且不需要权限管理等额外功能，那么推荐使用 frp、zerotier 等工具，配置简易，使用简单，容易上手。
{: .prompt-info }

## <font color=DodgerBlue>准备</font>

一台具备公网 IP 的服务器：Softether 服务端需要能被所有客户端访问到

* CPU、RAM 配置无需太高
* 网络环境要好，和 VPN 的使用体验直接挂钩

## <font color=DodgerBlue>Softether 服务端</font>

### <font color=DodgerBlue>安装</font>

1. 现有很多较为完善的教程，直接参考即可。例如：[https://www.mivm.cn/softether-vpn-lan/](https://www.mivm.cn/softether-vpn-lan/)
2. 官方有图形客户端，无需全程命令行

### <font color=DodgerBlue>重点功能</font>

1. VPN 流量等权限管控：通过为不同用户建立不同用户组，并限制不同用户组使用 VPN 的上行、下行等实现权限控制
2. 用户行为监控：有丰富的日志功能记录流量使用情况，并可以记录每个用户具体的数据量、活动连接、来源 IP 等等
3. 安全：支持多账号，支持白名单、黑名单等

## <font color=DodgerBlue>Softether 客户端连接</font>

### <font color=DodgerBlue>通用方法</font>

使用系统自带 VPN 功能，选择 `L2TP/IPsec` 类型：

> 不推荐 Windows 使用此方法
{: .prompt-warning}

* Windows：系统设置 -> 网络和 Internet -> VPN -> 添加 VPN
* Linux：系统设置 -> 网络 -> VPN
* MacOS：系统设置 -> 网络 -> 新建 -> VPN
* 移动设备：系统设置 -> 搜索 VPN

### <font color=DodgerBlue>Windows</font>

> 推荐直接使用官方提供的<font color=DodgerBlue>图形客户端</font>
{: .prompt-info }

> 使用<font color=OrangeRed>通用方法</font>之后，如果发现无法正常上网，需要： 控制面板 -> 网络和 Internet -> 网络连接 里找到刚刚的网络适配器，双击，然后选择 属性 -> 网络 -> TCP/IPv4 -> 高级 -> <font color=green>取消勾选</font>“在远程网络上使用默认网关”
> ![适配器设置](/assets/img/SoftetherVPN/适配器设置.png)
{: .prompt-warning}

### <font color=DodgerBlue>Linux</font>

#### <font color=DodgerBlue>命令行</font>

##### <font color=DodgerBlue>连接</font>

1. 官网下载 Linux 客户端（似乎只有源码包）
2. 解压后，进入 ./vpnclient 目录，执行 `make`，确保执行完成后，生成 `vpnclient` 及 `vpncmd` 两个可执行文件
3. 启动 vpn 服务：`sudo ./vpnclient start`，强烈建议直接使用 root 权限执行
4. 配置 vpn：`./vpncmd`
    1. 依次选择 client -> localhost
    2. `help` 可查看所有命令列表
    3. `AccountCreate` 创建新连接，参数和附件指南中一致即可，注意这一步需要输入虚拟网卡名称，没有的话一般会自动创建，如果没有自动创建，那么执行 `NicXXXXX` 命令手动创建一个网卡
    4. `AccountPasswordSet` 设置连接的密码
    5. `AccountConnect` 连接即可
5. 配置 IP 地址
    1. 4.3 中创建的虚拟网卡名加上 `vpn_` 前缀才是真正的虚拟网卡名称
    2. `sudo ifconfig 虚拟网卡名 10.1.1.XXX netmask 255.0.0.0`  <- 注意修改 IP 地址
    3. `sudo dhclient 虚拟网卡名` <- 可能要执行几十秒甚至一分钟，等着就好

##### <font color=DodgerBlue>自启动</font>

> 注意修改自启脚本里的连接名：CONNECTION、虚拟网卡名 及 固定 IP 地址！
{: .prompt-warning }

1. 新建 `/etc/init.d/vpnclient`:
    ```shell
    #!/bin/sh
    # Start/stop the vpnclien daemon
    #
    ### BEGIN INIT INFO
    # Provides:           vpnclient
    # Required-Start:     $local_fs $syslog $time
    # Required-Stop:      $local_fs $syslog $time
    # Should-Start:       $network
    # Should-Stop:        $network
    # Default-Start:      2 3 4 5
    # Default-Stop:
    # Short-Description:  vpn client service
    # Description:        Softether vpn client service.
    ### END INIT INFO
    
    EXE_DIR=/opt/vpnclient/
    SER="$EXE_DIR"vpnclient
    CMD="$EXE_DIR"vpncmd
    
    start()
    {
      echo start
      $SER start
      $CMD localhost /client /cmd accountconnect CONNECTION  # <======= 连接名
      ifconfig vpn_xxx 10.1.1.XXX netmask 255.0.0.0  # <=============== 注意修改 IP 地址和虚拟网卡名称
      echo end
    }
    
    stop()
    {
      $SER stop
    }
    
    case "$1" in
    start)
    start
    ;;
    stop)
    stop
    ;;
    esac
    exit 0
    ```
    {: file='/etc/init.d/vpnclient'}
2. `sudo chmod 755 /etc/init.d/vpnclient`
3. `sudo systemctl daemon-reload`
4. `sudo systemctl enable vpnclient.service`
5. `sudo systemctl start vpnclient.service`

#### <font color=DodgerBlue>GUI</font>

参考“通用方法”。

### <font color=DodgerBlue>固定 IP 设置</font>

1. Windows：在网络适配器中修改 IP4 地址
2. Linux：命令行修改对应适配器的 IP4 地址
3. MacOS：vpn 高级设置中直接指定 IP4 地址
