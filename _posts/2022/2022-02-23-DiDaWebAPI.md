---
Toc: true
comments: true
title: 滴答清单网页版 API 记录
Date: 2022-02-23 17:52:57 +0800
---

> 官方已更新 API 文档，直接查询官方即可：[TickTick Developer](https://developer.dida365.com/api#/openapi?id=authorization)
{: .prompt-tip}

因为计算机是我每天学习工作的主力，有时候想在清理滴答清单任务的时候计时，但又发现滴答清单没有计时功能，所以想着后续有时间用滴答清单的 API 写个简单的计时功能，这样可以直接集成不用造 todolist 的轮子了。以下是在研究滴答清单网页版时发现的一些 API。目前官网有意愿放出 API 文档，但是目前只有一个简陋的旧版本文档，所以只能自己动手了。

## <font color=DodgerBlue>基本说明</font>

> 非官方，随时可能会失效！
{: .prompt-warning}

### <font color=DodgerBlue>业务逻辑</font>

<img src="/assets/img/滴答清单网页版API记录/didaAPI.png" alt="业务逻辑关系" style="zoom:50%;" />

### <font color=DodgerBlue>cookie 设置</font>

1. 请求头中 cookie 需要包含 t=xxxxx 字段，否则部分请求会出现 500 错误等
2. xxxxx 为**登录**API返回的`token`字段的值

## <font color=DodgerBlue>API 列表</font>

### <font color=DodgerBlue>SignOn</font>

* 地址：https://api.dida365.com/api/v2/user/signon?wc=true&remember=true

* 类型：POST

* 关键 Headers：

  * x-device：通过调试工具在网页版滴答清单上获取

* body：

  ```json
  {
    "password": "密码",
    "username": "邮箱"
  }
  ```

* 返回值：

  ```json
  {
      "token": "cookie需要这个东西",
      "userId": "xxxxx",
      "username": "xxxx",
      "proStartDate": "2018-12-13T11:24:50.000+0000",
      "proEndDate": "2022-12-13T12:41:14.000+0000",
      "subscribeType": "order",
      "needSubscribe": true,
      "inboxId": "xxxx",
      "teamUser": false,
      "activeTeamUser": false,
      "freeTrial": false,
      "pro": true,
      "ds": false
  }
  ```

### <font color=DodgerBlue>WebSocket</font>

* 地址：wss://wss.dida365.com/web
* 说明：接收来自服务器的信息，比如任务更新等
* 消息：
  1. 刚连接时会返回一个 token，用于 `PushRegister` 中与 websocket 建立消息连接
  2. 需要定期向服务端发送心跳包：web版本是每9分钟发送 hello 字符串
  3. 服务端发生更新时，会主动发送一个 `checkPointID` 用于`BatchCheck`获取变化

### <font color=DodgerBlue>PushRegister</font>

* 地址：https://api.dida365.com/api/v2/push/register

* 类型：POST

* body：

  ```json
  {
      "pushToken": "WebSocket返回的token",
      "osType": 41
  }
  ```

### <font color=DodgerBlue>BatchCheck</font>

* 地址：https://api.dida365.com/api/v2/batch/check/{checkPointID}

* 类型：GET

* 参数：

  * `checkPointID`：
    * 0：获取全部
    * WebSocket 中提供的 ID：获取相较于上一个 checkPoint 的更新内容

* 返回值：

  ```json
  {
    "checkPoint": 1646973127626,  // 当前的 checkPointID
    "syncTaskBean": {
    	"update": [
    		{
    			"id": "任务ID",
    			"projectId": "projectId",
          ...
  			}
      ],
  		"delete": [],
  		"add": []
    },
  	"projectProfiles": null,
  	"projectGroups": [  // 项目描述
      ...
    ],
    "filters": null,
    "tags": [],     // tag 描述
  	"inboxId": null // checkPointId 为 0 时值为非 null，表示收件箱 Id，与其他的 Project 区分
  }
  ```

  