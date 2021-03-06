---
title: 'Jellyfin: 个人媒体中心搭建'
date: 2021-06-11 14:40:09
tags:
- Linux
- 娱乐
toc: true
comments: true
---

> Jellyfin是一个开源免费的媒体解决方案：让你能够完美控制你的媒体文件。它能够实现从个人服务器，没有任何附加条件地串流至任何设备。**Your media，your server，your way**。

可以把Jellyfin视作你个人的爱奇艺等视频网站！和传统文件夹保存视频、影片等相比，有着以下优点：

* **更好的组织你的媒体文件**：能够从 IMDB 等影片信息网站匹配视频相关信息并展现出来，包括但不限于：字幕下载，海报横幅下载，演员信息拉取，电影简介、电视剧季集信息等获取等。
* **任意设备浏览器即可观看**：无需集中在一台设备上观看，在能够访问到个人 Jellyfin 服务器的任何地方都能够通过浏览器观看视频。
* **多用户**：每个用户有着不同的资料库访问权限，各自的观看进度均独立保存。
* **支持不同类型的影音文件**：电影、电视剧、音乐以及在线电视节目。

> 官方在线 demo：[Jellyfin](https://demo.jellyfin.org/stable/web/index.html)

截图：

<center>
	<div style="display:inline-block;">
    <img src="/assets/img/Jellyfin-个人媒体中心搭建/image-20210611145838353.png" alt="主页" width=280px/>
  </div>
	<div style="display:inline-block;margin-left:10px;">
    <img src="/assets/img/Jellyfin-个人媒体中心搭建/image-20210611145937768.png" alt="电影" width=280px/>
  </div>
  <div style="display:inline-block;margin-left:10px;">
        <img src="/assets/img/Jellyfin-个人媒体中心搭建/image-20210611150016110.png" alt="老友记剧集" width=280px/>
  </div>
</center>


## Jellyfin 部署

Jellyfin 能够直接通过包管理器直接安装，官方也提供了docker容器镜像，因此 Jellyfin 可以运行在任何具备 docker 环境的地方，比如一台 VPS 或者群辉设备等。受限于设备限制，这里只展示在 VPS 上借助 docker 部署 Jellyfin 的方法。

### VPS 选取

具体的配置要求取决于 Jellyfin 的设置：服务端解码还是客户端解码？视频质量（分辨率等）高低？同时在线观看的人数？具体情况还是需要根据自己的视频资源测试后才能有个大概，这里仅供参考，提醒一下 VPS 的配置对于视频观看的观感有着很大影响。

这里没有做过具体的实验，从个人使用体验来看，如果在服务端解码，最好能够由 GPU 进行硬件加速，单靠 CPU 解码会消耗大量资源，尤其是 4k 视频下，CPU 解码时拖动视频进度条无法及时响应，会卡顿几秒，甚至无法流畅播放，GPU 解码就好多了。网络方面，Jellyfin 几档默认配置：1080p - 10\~60 M，4k - 80\~120 M（取决于视频的最高质量）。

### Jellyfin 容器部署

这里请直接参考官方文档：[Installing Jellyfin | Documentation - Jellyfin Project](https://jellyfin.org/docs/general/administration/installing.html) 。总体流程分为以下几步：

1. 安装 docker 环境：[Install Docker Engine | Docker Documentation](https://docs.docker.com/engine/install/)
2. 配置 docker 环境下的硬件加速【可选】：[Hardware Acceleration | Documentation - Jellyfin Project](https://jellyfin.org/docs/general/administration/hardware-acceleration.html)
3. 配置并部署 Jellyfin 容器

这里给出个人使用的 docker 部署命令，其中：

* 3-7 行为 NVIDIA 硬件加速所需的参数，见上面第 2 点
* 10-13 行将 VPS 的存储空间挂载到容器中以实现数据存储，防止每次重启容器的数据丢失。其中：
  * `/config` 为 Jellyfin 的配置相关目录，将 `/data/jellyfin/.jellyfin/config` 修改为任意你想使用的目录
  * `/cache` 为 Jellyfin 的缓存目录，将 `/data/jellyfin/.jellyfin/cache` 修改为任意你想使用的目录
  * `/media` 为 Jellyfin 的媒体目录，里面应该存放你的视频、音乐等文件，将 `/data/jellyfin` 修改为你想使用的目录
  * 默认端口为 8096，或者通过 `-p port:8096` 参数自定义

```shell
docker run -d \
 --name jellyfin \
 -e NVIDIA_DRIVER_CAPABILITIES=all \
 -e NVIDIA_VISIBLE_DEVICES=all \
 --gpus all \
 --device /dev/dri/renderD128:/dev/dri/renderD128 \
 --device /dev/dri/card0:/dev/dri/card0 \
 --user 0:0 \
 --net=host \
 --volume /data/jellyfin/.jellyfin/config:/config \
 --volume /data/jellyfin/.jellyfin/cache:/cache \
 --mount type=bind,source=/data/jellyfin,target=/media \
 --restart=unless-stopped \
 jellyfin/jellyfin
```

### Jellyfin 可能存在的问题

1. **无法刮削视频的元数据**：一般都是因为网络原因，导致无法访问 IMDB 等元数据网站，或者访问超时。虽然可以通过合理设置代理的方式解决，但是鉴于 Jellyfin 本身在元数据管理、修改方面较弱，不建议使用代理方法。请参考下一节 TinyMediaManager 方法。
2. **无法及时扫描文件夹的变化**：在人为修改元数据后，主动触发重新扫描，有时发现修改后的元数据并没有起效。该问题亦是由于网络原因导致，无法联网获取元数据导致了无法及时刷新（即便人为修改了）。这种情况需要在媒体库设置中，将自动刮削元数据选项取消勾选。
3. **字幕不显示**：字幕加载了，但是不显示。一般是由于字幕的字体缺失。通过 `vim` 之类的看一下字幕里面是不是指定了字体，然后在 Jellyfin 设置里，勾选候补字体，然后选择候补字体文件夹，再把缺失字体放进去即可。

## 视频元数据刮削：TinyMediaManager

TinyMediaManager 是一个基于 Java 的媒体管理工具，能够为 Kodi，MediaPortal 以及 Plex 等从 TheMovieDB，Imdb，Ofdb 等网站刮削媒体元数据。相较于 Jellyfin 自带的功能，该工具提供了更丰富的刮削、修改功能，且操作界面更加直观。官网见：[tinyMediaManager](https://www.tinymediamanager.org/) ，截图如下：

<img src="Jellyfin-个人媒体中心搭建/image-20210611181739987.png" alt="TinyMediaManager 界面" style="zoom: 33%;" />

感谢社区，让 TinyMediaManager 能够以容器的方式跑在服务器上，通过浏览器就能够使用。这里使用 [dzhuang/tinymediamanager-docker: A repository for creating a docker container including TinyMediaManager with GUI interface. (github.com)](https://github.com/dzhuang/tinymediamanager-docker) 版本，主要加入了中文字体支持。部署命令如下，主要参数：

* 端口为 8095，将 `-p 8095:5800` 修改为想使用的端口即可。
* 配置文件夹 `/config` ，将 `/data/jellyfin/.tmm/config` 修改为想使用的目录即可。**该目录与 Jellyfin 没有关联，可以任意选取可用目录**。
* 媒体文件夹 `/media` ，将 `/data/jellyfin` 修改为想使用的目录即可。**该目录与 Jellyfin 中的媒体目录需要保持一致**！

```shell
docker run -p 8095:5800 --user 0:0 -v /data/jellyfin/.tmm/config:/config -v /data/jellyfin/:/media dzhuang/tinymediamanager
```

之后在设置中配置好 电影、剧 各自的目录，然后点击`更新源` 按钮就可看到自己的视频文件，右键任意资源即可进行一系列操作。**网络问题可以在设置中配置代理解决。**

> TinyMediaManager 需要注意媒体文件夹的读写权限！否则会失败。

## 视频下载组合：Frostwire + Transmission

Frostwire 是一个提供了 BT 搜索的下载客户端，这里借用其搜索功能寻找视频资源，官网：[FrostWire - BitTorrent Client, Cloud Downloader, Media Player. 100% Free Download, No subscriptions required.](https://www.frostwire.com/)

Transmission 是一个能够在服务器上运行的 BT 下载客户端，通过网页即可管理，见教程：[How to set up transmission-daemon on a Raspberry Pi and control it via web interface - LinuxConfig.org](https://linuxconfig.org/how-to-set-up-transmission-daemon-on-a-raspberry-pi-and-control-it-via-web-interface)

<center>
	<div style="display:inline-block;">
    <img src="/assets/img/Jellyfin-个人媒体中心搭建/image-20210611183625148.png" alt="Frostwire" width=400px />
  </div>
  <div style="display:inline-block;margin-left:10px">
		<img src="/assets/img/Jellyfin-个人媒体中心搭建/image-20210611183751495.png" alt="Transmission" width=400px />
  </div>
</center>

基本使用流程：

1. Frostwire 上搜索你想要的视频
2. 右键复制 magent 信息
3. 在 Transmission 中下载
4. TinyMediaManager 刮削
5. Jellyfin 观看

## 突破内网限制：Zerotier/n2n/SoftEther

这里不再介绍这三款软件。考虑到 Zerotier 能够以最高成功率实现跨 Nat 直连而无需自己的中转服务器，建议使用 Zerotier 以节省流量。

请参考以下博客：

* **Zerotier**：[用ZeroTier搭建属于自己的虚拟局域网(VLAN) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/83849371)

* **SoftEther**：[使用 SoftEther VPN 搭建虚拟局域网 - 米V米 (mivm.cn)](https://www.mivm.cn/softether-vpn-lan/)
* **n2n**：[n2n实现内网穿透 - 简书 (jianshu.com)](https://www.jianshu.com/p/5021b70c3ff9)

### SoftEther 自启动

新建 `/etc/init.d/vpnclient` ，内容如下：

```bash
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
  $CMD localhost /client /cmd accountconnect 【你的连接名称。如果该连接设置了自启动，那么就不需要了】
  ifconfig 【你的vpn的网络适配器名称】 【你的ip】 netmask 255.0.0.0
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

注意还需要给它执行权限：`sudo chmod +x /etc/init.d/vpnclient`

然后交由 `systemd` 管理：

```shell
sudo ln -s /etc/init.d/vpnclient /etc/rc5.d/S99vpnclient
sudo systemctl enable vpnclient
sudo systemctl start vpnclient
```

之后即可开机自启。
