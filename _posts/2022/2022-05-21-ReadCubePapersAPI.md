---
Toc: true
comments: true
title: ReadCube Papers API 记录
Date: 2022-05-21 15:30:07 +0800
categories: [开发]
---

本文用于记录在开发 Joplin 的 ReadCube Papers 插件以实现笔记软件与文献管理软件的联动时，使用到的一些 web 版 papers 的接口。插件地址：[joplin-plugin-enhancement](https://github.com/SeptemberHX/joplin-plugin-enhancement)。

## 登录及 cookie 获取

1. URL：https://services.readcube.com/authentication/login
2. 方法：POST
3. 请求数据类型为 formData，包含：
	1. client: webapp
	2. api: // 可为空
	3. client_version: // 可为空
	4. email: 用户邮箱
	5. password: 用户密码
4. 返回值 set-cookie 有三条，分别为：
	1. _readcube-login_token
	2. _readcube_session
	3. user_web_token

	三个一起即为 cookie。

## WebSocket

用于接收来自服务器端关于数据变更的消息，进而实现实时同步。

地址：wss://push.readcube.com/bayeux

1. `id`：每一条客户端发往服务端的消息均会有一个 `id` 为 36 进制。
2. 需要定期向服务端发送心跳包，否则一段时间后服务器将主动断开 ws 连接
3. 发送/接收的所有数据均为 Json 格式
4. `clientId` 很关键，不然无法成功握手，需要先获取 `clientId`

### WebSocket 握手

* URL：`https://push.readcube.com/bayeux?message=[{"channel":"/meta/handshake","version":"1.0","supportedConnectionTypes":["websocket","eventsource","long-polling","cross-origin-long-polling","callback-polling"],"id":"1"}]&jsonp=__jsonp1__`
* 方法：GET
* 返回：内含有 `clientId` 及是否成功，且该次交互 `id` 为 1。
* **需要设置 Cookie**

### WebSocket 连接

> 示例见：[ReadCube Papers WebSocket.ts](https://github.com/SeptemberHX/joplin-plugin-enhancement/blob/master/src/lib/papers/papersWS.ts)
{: .prompt-info}

1. 订阅：
	1. users：`[{"channel":"/meta/subscribe","clientId":"client-6774640","subscription":"/production/users/user-id","id":"3"}]`。不知道为啥官方连续请求了 2 次
	2. collections: `[{"channel":"/meta/subscribe","clientId":"client-6774640","subscription":"/production/collections/collection-id/changes","id":"4"}]`。不知道为啥官方连续请求了 3 次
2. 心跳包：`[{"channel":"/meta/connect","clientId":"client-6774640","connectionType":"websocket","id":"b"}]`。发送后，服务端约 25s 后返回成功结果，此时需要立刻发送下一个心跳包。<ins>服务端返回的结果中包含响应的延时长度（现在是 25000），可以以其为依据判断 WebSocket 的存活。</ins>

## 信息获取

> 均需要设置 cookie。示例见：[ReadCube Papers.ts](https://github.com/SeptemberHX/joplin-plugin-enhancement/blob/master/src/lib/papers/papersLib.ts)
{: .prompt-info}

可能是多论文库的原因，每个用户有至少一个 collections，每个 collection 中有若干 item，每个item 即为一篇论文，每个 collection 具备唯一 id，item 也具备唯一 id。


| 功能                                   | URL                                                                                     | 方法 | 说明                                                                                               |
| -------------------------------------- | --------------------------------------------------------------------------------------- | :--: | -------------------------------------------------------------------------------------------------- |
| 获取所有 collections                   | https://sync.readcube.com/collections/                                                  | GET  |                                                                                                    |
| 获取指定 collection 中的文章 item 列表 | https://sync.readcube.com/collections/{collectionId}/items?sort%5B%5D=title,asc&size=50 | GET  | 一次最多拉取的 size 为50，返回值中有 scroll_id，想要获取后面的需要增加参数 scoll_id 继续调用该接口 |
| 获取指定 item 的信息                   | https://sync.readcube.com/collections/{collectionId}/items/{itemId}                     | GET  | 上一个 item 列表中也包含具体 item 的信息，该部分应该主要用于同步单一 item 的信息                   |
| 获取指定 item 的所有 annotation        | https://sync.readcube.com/collections/{collectionId}/items/{itemId}/annotations         | GET  | PDF 里做的标记、笔记等                                                                             |

还有许多其它接口，个人需求原因不需要其他接口，因此需要自行研究。

## 更新

### item 信息更新

统一通过 PATCH 类型请求进行。

* URL：https://sync.readcube.com/collections/${collectionId}/items/${itemId}?client=webapp&client_id=${clientId}&client_version=4.12.6
* 类型：PATCH
* 参数：{"item":{"user_data":{"notes":"cas"}}}
* 返回值：item 的全部信息

其它的类似。