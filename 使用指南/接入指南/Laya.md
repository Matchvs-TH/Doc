**游戏交互时序图**

![](http://imgs.matchvs.com/static/时序图.jpg)

## 初始化

调用Matchvs.MatchvsEngine.getInstance方法获取Matchvs引擎对象：

```javascript
var engine = Matchvs.MatchvsEngine.getInstance();
```

另外创建一个回调对象，在进行注册、登录、发送消息等操作之后，该对象的方法会被异步调用：
```javascript
var response = {/* 按照文档规范定义一些回调方法 */};
```

接入下可以调用初始化方法init：
```javascript
engine.init(response, channel, platform, gameId);
```

Matchvs 提供了两个环境，alpha 调试环境和 release 正式环境。

游戏开发调试阶段请使用 alpha 环境，即 platform 传参"alpha"。如下：

```javascript
engine.init(response, "MatchVS", "alpha", 201016);
```

参数说明:

| 参数       | 含义                              |
| -------- | ------------------------------- |
| response | 回调对象                            |
| channel  | 渠道，填“MatchVS”即可                 |
| platform | 平台，调试环境填“alpha” ，正式环境填“release” |
| gameId   | 游戏ID，来自官网控制台游戏信息                |


**注意** 在整个应用全局，开发者只需要对引擎做一次初始化。

## 注册

Matchvs提供的用户ID被用于在各个服务中校验连接的有效性，调试前开发者需要先获取到一个合法的用户ID。
```javascript
engine.registerUser();
```

调用成功后会收到注册成功的回调 ：
```javascript
response.registerUserResponse = function(userInfo) {
	// 用户ID
	console.log("userId: ", userInfo.id);
	// token
	console.log("token: ", userInfo.token);
}
```


**注意** 用户ID和token有需要的可以缓存起来，在之后的应用启动中不必重复获取。如果你有自己的用户系统，可以将Matchvs 提供的 userID 和用户系统进行映射。


## 登录

成功获取用户ID后即可连接Matchvs服务：

```javascript
engine.login(userId, token, gameId, gameVersion, appkey, secret, deviceId, gatewayId);
```

参数说明:

| 参数          | 含义                          |
| ----------- | --------------------------- |
| userId      | 用户ID，调用注册接口后获取              |
| token       | 用户token，调用注册接口后获取           |
| gameId      | 游戏ID，来自Matchvs官网控制台游戏信息     |
| gameVersion | 游戏版本，自定义，用于隔离匹配空间           |
| appkey      | 游戏Appkey，来自Matchvs控制台游戏信息   |
| serect      | secret key，来自Matchvs控制台游戏信息 |
| deviceId    | 设备ID，用于多端登录检测，请保证是唯一ID      |
| gatewayId   | 服务器节点ID，默认为0                |

- 其中，appKey，secret，gameId是你创建游戏后从官网获取的信息，可以[前往控制台](http://www.matchvs.com/manage/gameContentList)查看。appkey和secret是校验游戏合法性的关键信息，请妥善保管secret信息。  
- userId 和 token 是第二步 **注册成功** 的回调信息。  
- deviceId 用于检测是否存在多个设备同时登录同一个用户的情况，如果一个账号在两台设备上登录，则后登录的设备会连接失败。
- Matchvs默认将相同游戏版本的用户匹配到一起。如果开发者对游戏进行了版本升级，不希望两个版本的用户匹配到一起，此时可以在登录的时候通过`gameVersion`区分游戏版本。 

登录成功会收到回调 ：

```javascript
response.loginResponse = function(loginRsp) {
	// 返回值
	var status = loginRsp.status;
	// 房间号
	var roomId = loginRsp.roomId;
}
```

SDK支持房间断线重连，掉线重新登录后可以选择加入原来的房间，loginResponse里的`roomId` 即为上次异常退出的房间ID。如果登录时没有异常退出的房间，则`roomid`为0。

## 加入房间

登录成功后，可以调用Matchvs加入房间逻辑将用户匹配至一个房间开始一局游戏（如：《荒野行动》的开始匹配、《球球大作战》的开始比赛等）

Matchvs默认提供了随机加入房间的模式，调用加入房间逻辑后，Matchvs服务器会自动帮助用户寻找当前可用房间，只有在同一个房间里的用户才可以互相通信。

随机加入房间的模式下，Matchvs服务器能够快速找到合适的房间，开发者只需要自定义房间人数上限，Matchvs服务端会根据当前房间人数判断是否可继续加入。  

**注意**  随机匹配不能匹配到客户端主动创建的房间里，即通过`createRoom()`（见联网扩展）创建的房间。

随机加入一个房间：

```javascript
engine.joinRandomRoom(maxPlayer, userProfile);
```


参数说明:

| 参数         | 含义                |
| ---------- | ----------------- |
| maxPlayer  | 最大玩家数，不超过20       |
| uerProfile | 玩家简介，可以填写昵称、段位等信息 |

加入房间的回调：

```javascript
response.joinRoomResponse = function(status, roomUserInfoList, roomInfo) {
	console.log("加入房间结果：", status);
	console.log("房间用户列表：", roomUserInfoList);
	console.log("房间信息：", roomInfo);
}
```

如果当前没有可用房间，Matchvs会自动创建一个房间并将该用户加入到服务端创建的房间。当其他用户加入时，Matchvs会通知开发者新加入的用户信息。

其他玩家加入房间的回调：

```javascript
reponse.joinRoomNotify = function(userId, roomUserInfo) {
	console.log("玩家：", userId);
	console.log("房间新加的用户的信息：", roomUserInfo);
}
```

**注意** 如果开发者想用户匹配成功后可查看对方信息，可以通过填充`userProfile`的方式，将当前用户的头像昵称信息填充至`userProfile`，Matchvs会在匹配成功时将`userProfile`广播给所有用户。 

如果用户已经在房间里，此时再次调用加入房间：如果房间未JoinOver，则玩家会退出房间然后随机加入房间；如果房间已经JoinOver，则SDK会返回重复加入的错误提示。

## 停止加入

如果房间内游戏人数已经满足开始条件，此时客户端需要通知Matchvs无需再向房间里加人。（如原本设置的房间人数上限为6，而开发者在房间人数满足4个即可开始游戏，开发者就需调用停止加入接口。） 

停止加入 ：

```javascript
engine.joinOver(proto);
```


参数说明:

| 参数    | 含义   |
| ----- | ---- |
| proto | 负载数据 |

停止加入的回调 ：

```javascript
response.joinOverResponse = function(joinOverRsp) {
	// 返回值
	console.log("加入房间结果：", joinOverRsp.status);
	// 负载数据
	console.log("负载信息：", joinOverRsp.cpProto);
}
```

**注意**  Matchvs服务器会判断房间是人满状态或者已停止加入状态，根据状态判断房间是否还可加人。为避免房间人满后开始游戏，在游戏过程中有人退出后，Matchvs判断人不满可继续向房间加人，建议在任何不希望中途加入的游戏里，只要满足开始游戏条件则向Matchvs服务端发送停止加入。

`proto` 为开发者自定义的协议内容，如果没有自定义协议可填`''`。proto的内容会伴随消息的广播以Notify的方式发给房间所有成员。其他接口里的`proto`机制均是如此。

## 游戏数据传输

当玩家在同一个房间时，即可互相通信。开发者可用该接口将数据发送给其他玩家，Matchvs默认将数据广播给当前房间内除自己以外的所有用户。

默认广播数据：

```javascript
engine.sendEvent(msg)
```
参数说明:

| 参数   | 含义   |
| ---- | ---- |
| msg  | 消息内容 |

返回结果为一个对象，该对象的属性为：

| 参数       | 含义               |
| -------- | ---------------- |
| result   | 错误码，0表示成功，其他表示失败 |
| sequence | 事件序号，作为事件的唯一标识   |

数据传输回调 ：

```javascript
response.sendEventResponse = function(sendEventRsp) {
	console.log("返回状态：", sendEventRsp.status);
	console.log("事件序号：", sendEventRsp.sequence);
}
```


收到其他人发的数据：

```javascript
response.sendEventNotify = function(eventInfo) {
    console.log("推送方用户ID：", eventInfo.srcUserId);
    console.log("消息内容：", eventInfo.cpProto);
}
```

## 离开房间

在成功加入房间后，开发者可调用离开房间使得用户退出当前房间，退出房间后将不能再与房间内的成员进行通信。  

- 当房间内有用户离开时，剩余用户也会收到离开的消息。
- 如果用户已经离开房间，此时可以随时再次加入其他房间。


离开房间 ：

```javascript
engine.leaveRoom(cpProto);
```

参数说明:

| 参数      | 含义   |
| ------- | ---- |
| cpProto | 负载信息 |

自己离开房间回调 ：

```javascript
response.leaveRoomResponse = function(leaveRoomRsp) {
	console.log("状态返回：", leaveRoomRsp.status);
	console.log("房间ID：", leaveRoomRsp.roomId);
	console.log("用户ID：", leaveRoomRsp.userId);
	console.log("负载信息：", leaveRoomRsp.cpProto);
}
```


其他成员离开房间回调 ：

```javascript
response.leaveRoomNotify = function(roomId, roomUserInfo) {;
	console.log("房间号：", roomId);
	console.log("离开房间的用户的信息：", roomUserInfo);
}
```


## 游戏登出

如果用户不会再加入游戏，此时可以调用登出与Matchvs服务端断开连接。  

**注意** 游戏退出时，务必要调用登出。

```javascript
engine.logout();
```

登出成功的回调 ：

```javascript
response.logoutResponse = function(status) {
	console.log("状态返回值：", status);
}
```


## 反初始化

在登出后，调用反初始化对资源进行回收。  

```javascript
engine.uninit();
```



## 数据存取

**注意** Matchvs 环境分为测试环境（alpha）和 正式环境（release），所以在使用http接口时，需要通过域名进行区分。使用正式环境需要先在[官网控制台](http://www.matchvs.com/manage/gameContentList)将您的游戏发布上线。

**alpha环境域名：alphavsopen.matchvs.com**

**release环境域名：releasevsopen.matchvs.com**

存储接口 ： **wc5/hashSet.do**

开发者可以通过调用该接口将自定义的数据存储至服务器。

```
http://alphavsopen.matchvs.com/wc5/hashSet.do?gameID=102003&userID=21023&key=1&value=a&sign=68c592733f19f6c5ae7e8b7ae8e5002f 
```

**注意：** value的长度上限为255字符，如果长度超过255，Matchvs 在存储时会忽略255后的字符内容。存储上限为每个玩家1000条，如果超过1000条，会返回对应错误。

可以调用hashSet实现增量存储。为避免特殊字符影响，存储前，建议开发者最好将字符串解码成二进制再用UrlEndcode编码后存储。

| 参数名    | 说明          |
| ------ | ----------- |
| gameID | 游戏ID        |
| userID | 用户ID        |
| key    | 自定义存储字段编号   |
| value  | 自定义存储字段的值   |
| sign   | 见下方sign获取方法 |

返回数据示例如下：

```
    {
        "code": 0,
        "data": "success",
        "status": 0
    }
```





取接口：**wc5/hashGet.do**

开发者可以通过调用该接口获取存储在服务器的自定义数据。

```
http://vsopen.matchvs.com/wc5/hashGet.do?gameID=102003&userID=21023&key=1&sign=b0244f7ed1d433975512a8f6c2ba4517 
```

**注意** 存储前，如果将字符串解码成二进制再用UrlEndcode编码后存储，对应的取出时应用UrlDecode进行解码后显示

| 参数名    | 说明          |
| ------ | ----------- |
| gameID | 游戏ID        |
| userID | 用户ID        |
| key    | 自定义存储字段键值   |
| sign   | 见下方sign获取方法 |

返回数据示例如下：

```
{
    "code": 0,
    "data": "this is my data",
    "status": 0
}
```



**sign值获取方法**

##### 1. 按照如下格式拼接出字符串:

```
appKey&param1=value1&param2=value2&param3=value3&token
```

- `appKey`为您在官网配置游戏所得

- `param1、param2、param3`等所有参数，按照数字`0-9`、英文字母`a~z`的顺序排列

  例 ： 有三个参数`gameID`、`userID`、`key`，则按照`appkey&gameID=xxx&key=xxx&userID=xxx&token` 的顺序拼出字符串。

- `token`通过用户注册请求获取

##### 2. 计算第一步拼接好的字符串的`MD5`值，即为`sign`的值。



## 错误码

```
response.errorResponse = function(error) {
	console.log("错误信息：", error);
}
```
**注意** Matchvs相关的异常信息可通过该接口获取

| 错误消息                              | 含义         |
| --------------------------------- | ---------- |
| fail                              | 失败         |
| network error or exception        | 网络异常       |
| server closed                     | 服务器关闭      |
| unkown message from server        | 消息无法识别     |
| room request failed               | 房间请求失败     |
| sdk not inited                    | sdk未初始化    |
| not connected to server           | 无法与服务器建立连接 |
| bad request parameters            | 请求参数错误     |
| user not exists                   | 用户不存在      |
| user already registed             | 用户已经注册     |
| connect to server failed          | 连接服务器失败    |
| matchvs not support the protocol  | 协议错误       |
| network recv timeout              | 接收超时       |
| network send timeout              | 发送超时       |
| cannot find available gate server | gateway不存在 |
| reconnect failed                  | 重连失败       |
| heart beat timeout                | 心跳超时       |
