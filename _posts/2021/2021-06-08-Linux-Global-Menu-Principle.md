---
title: Linux 桌面上的全局菜单实现原理
date: 2021-06-08 22:09:18
tags: Linux
comments: true
---

> 全局菜单指的是将原本属于应用窗口内部的菜单栏（一般位于标题栏下面）脱离窗口独立各自的应用窗口显示，且没有导致原有的功能缺失。

macOS 上面就有相当完善的全局菜单机制，Linux上最近几年各大桌面环境诸如 KDE、Gnome 也陆续支持全局菜单（KDE是有官方支持的，Gnome则是第三方插件实现的）。

并没有考证过全局菜单在 Linux 上的发展历史，但是从 KDE 实现全局菜单的原理来看，使用到了 `com.canonical.AppMenu.Registrar.xml` 以及 `com.canonical.dbusmenu.xml` 两个 DBus 描述文件，从这两个文件名字来看，应该是基于 unity 的遗留财产了。

这里只介绍 KDE 上的全局菜单原理。

## 基本原理

简单来说，就是将应用程序内部的菜单栏，通过序列化的方式暴露到标准的 DBus 上，然后由全局菜单服务从对应的 DBus 中反序列化回菜单栏并显示出来。所以要做的事情有：
1. 不同类型的应用程序将自己的菜单栏序列化暴露到 DBus 上：这一步显然得是框架支持，Qt,GTK之类的
2. 通过 DBus 反序列化得到菜单，并根据当前窗口的 id 显示对应的菜单栏

先说序列化与反序列化菜单的情况：这个其实已经有现成实现得了，直接拿来用就好了，appmenu-qt,appmenu-gtk-module 这些，所以这一块就不用操心了。

其次是怎么样才能知道每个程序全局菜单的 DBus? 显然需要框架主动告诉系统，该程序的 DBus service name 是什么，path 是什么，这样才有可能成功获取到菜单。在这件事情上，Qt 和 Gtk 的处理方法是不同的。

1. Qt
首先 Qt 本身已经支持全局菜单了，参见 [Qt文档](https://doc.qt.io/qt-5/qmenubar.html#qmenubar-as-a-global-menu-bar)。当 `com.canonical.AppMenu.Registrar` DBus 存在的时候，程序就会自动向该 DBus 注册，那只需要在实现这个 DBus Service 的时候，记录下来 Qt 上报的内容就能够知道该 Qt 程序的 DBus 服务名称以及路径，再结合窗口 ID，就很容易定位到

2. Gtk
gtk 本身是不支持全局菜单的，需要借助 `GTK_MODULES=appmenu-gtk-module` 才能够实现。当使用了该 module 后，它也会检测 `com.canonical.AppMenu.Registrar` DBus 是否存在，如果存在就启动全局菜单。

但是不同的是，Gtk 并不会向这个 Dbus 注册程序的 DBus 服务名称和路径，而是将这些信息写在了窗口属性里，需要使用 `xlib` 之类的进行对应属性的获取。下面是几个比较关键的属性名称，可以参照 gtk wiki 查看。
* `_GTK_APPLICATION_OBJECT_PATH`
* `_GTK_WINDOW_OBJECT_PATH`
* `_GTK_MENUBAR_OBJECT_PATH`
* `_GTK_APP_MENU_OBJECT_PATH`

那么思路就很清楚了，Qt 程序的话，就通过 `com.canonical.AppMenu.Registrar` DBUS 读取程序的全局菜单的 dbus 服务名称和路径，GTK 的话，就通过窗口属性。

## KDE 原理

KDE 为 Qt 和 Gtk 做了统一，最终将程序的 dbus 名称和路径写在了窗口属性 `_KDE_NET_WM_APPMENU_SERVICE_NAME` 以及 `_KDE_NET_WM_APPMENU_OBJECT_PATH`。

首先是Qt，具体代码请参考 [plasma-workspace/appmenu](https://github.com/KDE/plasma-workspace/tree/52fbaf72ab9ef26a2795497eb4ea1b8418ecb11b/appmenu)。它实现了 `com.canonical.AppMenu.Registrar` DBus 服务，并在窗口注册实现里，将全局菜单的 dbus 服务名称和路径记录在了窗口属性中。序列化的方式是 `dbusmenu`

其次是 Gtk，由于 Gtk 采用的是另外一套序列化工具，所以 KDE 做了一个代理，见 [gmenu-dbusmenu-proxy](https://github.com/KDE/plasma-workspace/tree/52fbaf72ab9ef26a2795497eb4ea1b8418ecb11b/gmenu-dbusmenu-proxy)。这里面做了两件事，一是读取 Gtk 窗口的属性并记录到 `_KDE_NET_WM_APPMENU_SERVICE_NAME` 以及 `_KDE_NET_WM_APPMENU_OBJECT_PATH`；二是将 Gtk 序列化的方式转为了 `dbusmenu` 方式。

最后就是全局菜单的插件了，监控窗口变化事件，DBus 上的菜单更新事件，为每个窗口呈现各自的全局菜单，见 [plasma-active-window-control](https://github.com/KDE/plasma-active-window-control)

## Deepin OS 上的全局菜单实现

具体见：

* [SeptemberHX/dde-top-panel: dde top panel for Deepin V20 (github.com)](https://github.com/SeptemberHX/dde-top-panel) ：负责监控 DBus 并从 DBus 中读取并显示出当前活跃窗口的菜单
* [SeptemberHX/dde-globalmenu-service: DBus service registery for dde-globalmenu (github.com)](https://github.com/SeptemberHX/dde-globalmenu-service) ：负责启动 DBus 并将不同的全局菜单实现（Qt、Gtk等）做一层统一代理以方便统一调用

