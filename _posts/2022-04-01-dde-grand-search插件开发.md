---
title: dde-grand-search 插件开发
date: 2022-04-01 14:44:05 +08:00
tags:
- Linux
- Deepin
categories: [开发, Deepin]
toc: true
comments: true
image:
  src: /assets/img/dde-grand-search插件开发/example.png
  alt: dde-top-panel 集成 dde-grand-search 效果图
---

截止到目前 2022-04-01 为止，github 上没有放出 `dde-grand-search `的代码仓库，但是可以通过 `apt source` 的方式下载到全部代码。因此目前没有官方文档展示其插件开发过程，但是当前 Deepin V20.5 版本下，`dde-control-center` 的搜索效果是以插件形式集成到 `dde-grand-search` 中的，因此可以参照 `dde-control-center` 相关部分进行模仿开发。

> `dde-grand-search` 的插件机制可能会随着版本号变化而变动，使得本篇内容不再适用，此时需要按照本篇思路重新梳理插件开发方法。
{: .prompt-info }

## 插件文件目录

按照软件设计的基本逻辑推断，再结合 `dde-dock` 中的插件机制，可以推测得到 `dde-grand-search` 有很大概率也是通过插件的方式进行后续功能增加的，尤其是目前 `dde-grand-search` 的搜索项目太少，参考 MacOS 的聚焦功能，能够想象到的后续拓展功能就包括：字典、剪贴板、邮件以及其他各种 Deepin 全家桶功能的集成。

因此只需要确定插件目录下有哪些以及存在的插件，然后去看对应部分的源码即可。插件目录在没有官方文档的情况下，最简单可靠的方法就是阅读源码，不过首先需要确保 `/etc/apt/sources.list` 中 `deb-src` 一行没有被注释掉：

```shell
apt source dde-grand-search
```

解压代码，Clion 打开项目目录，通过文件名判断插件相关的主要类为 `pluginmanager.cpp` 中的 `PluginManagerPrivate`，其中的 `defaultPath` 即为插件路径。

然后利用 Clion 的全局搜索功能，可以查到路径定义在 `CMakeLists.txt` 中。最终得到路径：`/usr/lib/x86_64-linux-gnu/dde-grand-search-daemon/plugins/searcher`，同时发现已存在一个插件文件：`com.deepin.dde-grand-search.dde-control-center-setting.conf`，名称可知是设置中心的插件，内容如下：

```conf
[Grand Search]
Name=com.deepin.dde-grand-search.dde-control-center-setting
Mode=Trigger
DBusService=com.deepin.dde.ControlCenter
DBusAddress=/com/deepin/dde/ControlCenter
DBusInterface=com.deepin.dde.ControlCenter.GrandSearch
InterfaceVersion=1.0
```
{: file='com.deepin.dde-grand-search.dde-control-center-setting.conf'}

显然，是通过配置文件告诉 `dde-grand-search` 对应插件的 `DBus Interface` 地址，然后统一调用的。`Mode` 还有其它模式，现在是搜索时触发模式，其它可以直接搜索源码，对应文件中有着注释。

## 接口及数据格式

由上一节可知，实现插件需要两步：
1. 配置文件告诉 `dde-grand-search` 插件的 `DBus` 接口
2. 实现配置文件中的接口

但是需要实现哪些具体接口，在没有官方文档的情况下，直接参照 `dde-control-center` 的源码，在官方 github 仓库里简单的搜索 `GrandSearch` 关键词，即可快速定位到相关内容，主要关键文件如下：
1. `src/frame/dbuscontrolcenterservice.h`
2. `src/frame/dbuscontrolcenterservice.cpp`
3. `src/frame/window/mainwindow.cpp`

阅读源码后，可以发现几个关键的 `DBus` 接口：
```c++
class DBusControlCenterGrandSearchService: public QDBusAbstractAdaptor
{
/* ... */
public Q_SLOTS: // METHODS
    // dde-grand-search 输入时，调用该接口获取搜索结果
    QString Search(const QString json);
    
    bool Stop(const QString json);
    
    // 搜索结果被点击时，调用该接口进行响应
    bool Action(const QString json);
/* ... */
```
{: file='dbuscontrolcenterservice.h'}

接着只需明确输入的 `json` 和返回的 `QString` 具体格式即可，参见 `MainWindow::GrandSearchSearch()`。

> 以下列出的数据结构不全，仅列出了关键的部分，更为详细的需要翻阅 `dde-grand-search` 源码
{: .prompt-info }

### Search

```json
{
  "ver": "应该是版本号，具体类型需要看代码",
  "mID": "推测用来标记插件搜索请求的，具体类型看代码",
  "cont": "搜索关键词"
}
```
{: file='Search 接口输入'}

```json
{
  "ver": "直接使用输入的 ver",
  "mID": "直接使用输入的 mID",
  "cont": [
    {
      "group": "dde-grand-search 显示结果时的组名称",
      "items": [
        {
          "item": "搜索结果 QString，触发时将作为传递给 Action 接口的参数",
          "name": "搜索结果显示出的字符串，可以和 item 不一样",
          "icon": "搜索结果条目图标路径",
          "type": "类型，目前来看好像没有什么作用"
        },
        /* 可以有多个搜索结果 */
      ]
    }
  ]
}
```
{: file='Search 接口输出'}

### Stop

该部分此处不涉及，跳过。

### Action

```json
{
  "action": "动作类型，我们主要关注 openitem 类型，表示双击了搜索结果",
  "item": "选中搜索结果的 item 属性，和 Search 的返回结果中的 item 对应"
}
```
{: file='Action 接口输入'}

## 例子：`dde-top-panel`支持搜索菜单

> 项目地址：[https://github.com/SeptemberHX/dde-top-panel/tree/dde-grand-search](https://github.com/SeptemberHX/dde-top-panel/tree/dde-grand-search)
{: .prompt-tip}

### 插件配置文件
```conf
[Grand Search]
Name=com.deepin.dde-grand-search.dde-top-panel-setting
Mode=Trigger
DBusService=com.septemberhx.dde.TopPanel
DBusAddress=/com/septemberhx/dde/TopPanel
DBusInterface=com.septemberhx.dde.TopPanel.GrandSearch
InterfaceVersion=1.0
```
{: file='com.deepin.dde-grand-search.dde-top-panel-setting.conf'}

### `DBus` 接口实现

```cpp
QString DBusTopPanelService::Search(const QString json) {
    //解析输入的json值
    QJsonDocument jsonDocument = QJsonDocument::fromJson(json.toLocal8Bit().data());
    if(!jsonDocument.isNull()) {
        QJsonObject jsonObject = jsonDocument.object();

        QJsonObject jsonResults;
        QJsonArray items;
        
        // 调用具体方法获取菜单搜索结果
        QStringList searchResult = this->parent()->GrandSearchSearch(jsonObject.value("cont").toString());
        
        // 将搜索结果拼接成 json 格式
        for (int i = 0; i < searchResult.length(); i++) {
            QJsonObject jsonObj;
            jsonObj.insert("item", searchResult[i]);
            // 去掉 & 符号，否则影响搜索结果显示的观感
            jsonObj.insert("name", searchResult[i].remove('&'));
            jsonObj.insert("icon", "menu");
            jsonObj.insert("type", "application/x-dde-top-panel-xx");
            items.insert(i, jsonObj);
        }

        QJsonObject objCont;
        objCont.insert("group", "TopPanel");
        objCont.insert("items", items);

        QJsonArray arrConts;
        arrConts.insert(0, objCont);

        jsonResults.insert("ver", jsonObject.value("ver"));
        jsonResults.insert("mID", jsonObject.value("mID"));
        jsonResults.insert("cont", arrConts);

        QJsonDocument document;
        document.setObject(jsonResults);

        return document.toJson(QJsonDocument::Compact);
    }

    return QString();
}

bool DBusTopPanelService::Stop(const QString json) {
    return true;
}

bool DBusTopPanelService::Action(const QString json) {
    QString searchName;
    QJsonDocument jsonDocument = QJsonDocument::fromJson(json.toLocal8Bit().data());
    if(!jsonDocument.isNull()) {
        QJsonObject jsonObject = jsonDocument.object();
        if (jsonObject.value("action") == "openitem") {
            // 打开 item 的操作
            searchName = jsonObject.value("item").toString();
            
            // 调用结构执行
            return this->parent()->GrandSearchAction(searchName);
        }
    }
    return false;
}
```
{: file='DBusTopPanelService.cpp'}

### 实现效果

![dde-grand-search 搜索结果](/assets/img/dde-grand-search插件开发/example.png)