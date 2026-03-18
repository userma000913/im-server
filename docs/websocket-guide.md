## WebSocket 接入与协议详解（im-server）

本文档聚焦 `im-server` 中的 WebSocket 接入层与自定义协议实现，结合 `services/connectmanager` 与 `services/connectmanager/server/codec` 目录的实际代码，帮助你从“握手建立连接”一路理解到“消息收发与 Ack / 查询确认”的完整链路。

---

## 1. 总体结构与职责分层（实际代码视角）

**整个 WebSocket 接入链路由三块组成：**

- **ConnectManager 服务（`services/connectmanager`）**
  - `server/imwebsocketserver.go`：真正监听 WebSocket 端口的 HTTP 服务器。
    - 对外暴露：
      - `GET /im`：WebSocket 连接入口（gorilla/websocket 升级）。
      - `POST /im/publish`：HTTP 版“上行消息”入口（走同一套内部路由，见第 6 节）。
      - `GET /health`：健康检查。
  - `server/imwebsocketmsghandler.go`：对单帧 `ImWebsocketMsg` 进行 Cmd 分发。
  - `server/imlistener.go`：`ImListener` 的具体实现，将 Connect / Publish / Query 等转成内部 RPC 调用。
  - `services/connectmanager.go`：维护在线连接缓存，多端登录踢人等。

- **Codec 模块（`services/connectmanager/server/codec`）**
  - `imwebsocketmsg.go`：定义 protobuf `ImWebsocketMsg`，以及 Encrypt/Decrypt 逻辑。
    - 当前 WebSocket 路径上，**实际的“帧格式”就是一个 protobuf 编码的 `ImWebsocketMsg`**，而不是手写字节流。
  - `message.go` 等：定义通用的 `MsgHeader`/`IMessage`，主要用于服务端内部构造下行消息时的抽象（不是当前 WebSocket 读循环里的主入口）。
  - 其它 `*_message.go`：不同 Cmd 类型的消息体结构（ConnectMsgBody / PublishMsgBody / QueryMsgBody 等）。

- **业务监听器（`ImListener` 实现）**
  - `IMWebsocketMsgHandler` 解码出 `ImWebsocketMsg` 后，会根据 `Cmd` 调用不同的监听方法：
    - `Connected` / `Diconnected`
    - `PingArrived`
    - `PublishArrived` / `PubAckArrived`
    - `QueryArrived` / `QueryConfirmArrived`
  - 这些监听方法中再去调用内部 Message / Conversation / User 等服务。

这三层共同组成了 WebSocket 接入“前门”：**HTTP 升级 -> protobuf 帧（`ImWebsocketMsg`）解密/解码 -> 业务回调（`ImListener`）**。

---

## 2. 协议常量与命令字（Cmd）

命令字与 QoS 常量统一定义在 `services/connectmanager/server/codec/message.go` / `connect.proto` 对应的生成代码中，`ImWebsocketMsg` 上就是这些字段：

- **QoS（消息质量）**
  - `QoS_NoAck = 0`：不需要 Ack，适合心跳或弱一致性通知。
  - `QoS_NeedAck = 1`：需要 Ack，用于重要业务消息。

- **命令字 Cmd**
  - `Cmd_Connect = 0`：客户端请求建立会话（携带 token/appkey/deviceId 等）。
  - `Cmd_ConnectAck = 1`：服务端返回连接结果、userId、session 等。
  - `Cmd_Disconnect = 2`：主动断开。
  - `Cmd_Publish = 3`：上行/下行业务消息主通道。
  - `Cmd_PublishAck = 4`：Publish 的 ack（包含 msgId、时间戳、群成员数等）。
  - `Cmd_Query = 5` / `Cmd_QueryAck = 6` / `Cmd_QueryConfirm = 7`：查询 + 应答 + 客户端确认。
  - `Cmd_Ping = 8` / `Cmd_Pong = 9`：心跳。

> 注意：`MsgHeader` 中的 bit 编码是更底层的抽象，这一层通常在构造 `IMessage` 下行时使用；**WebSocket 读循环本身直接把整帧作为 protobuf 的 `ImWebsocketMsg` 来处理**（见第 3 节）。

---

## 3. WebSocket 端点与帧格式（/im）

### 3.1 端点定义

在 `ImWebsocketServer.AsyncStart` 中：

- `GET /im`：WebSocket 升级入口。
- `POST /im/publish`：HTTP 直连入口，使用同样的 `ImWebsocketMsg`/`PublishMsgBody` 协议（见第 6 节）。
- `GET /health`：简单 JSON 健康检查。

### 3.2 WebSocket 读写循环

`ImWebsocketChild.startWsListener` 是核心读循环：

1. 使用 gorilla/websocket 从连接读取一帧二进制数据。
2. `tools.PbUnMarshal(message, wsMsg)` 直接将整个 payload 反序列化为 `codec.ImWebsocketMsg`。
3. 调用 `wsMsg.Decrypt(ctx)` 做负载解密/反混淆（第 4 节详述）。
4. 将 `wsMsg` 交给 `IMWebsocketMsgHandler.HandleRead` 做 Cmd 分发。

下行方向则通过 `WsHandleContextImpl.Write`：

1. 业务层构造实现了 `codec.IMessage` 的下行消息对象（例如 ConnectAckMessage、UserPublishAckMessage 等）。
2. 在 `Write` 中调用 `imMsg.ToImWebsocketMsg()` 得到一个 `ImWebsocketMsg`。
3. 调用 `Encrypt(ctx)` 对 `Payload` 混淆/加密。
4. 使用 `tools.PbMarshal(wsImMsg)` 序列化为 protobuf 二进制。
5. 通过 gorilla/websocket 写入 `BinaryMessage`。

> 结论：**对客户端来说，WebSocket 上的每一帧都是一个 protobuf 编码的 `ImWebsocketMsg`，其中 `Cmd/QoS/Payload` 等字段用常规 proto 方式组织。**

---

## 4. 加解密与混淆（Obfuscation）机制

### 4.1 Connect 握手与 obfuscation code

`ImWebsocketMsg.Decrypt` 对 `Cmd_Connect` 有特殊逻辑：

- 客户端在 `Cmd_Connect` 帧中将 `ConnectMsgBody` protobuf 序列化后放入 `Payload`，但未混淆。
- 服务端收到后：
  1. 读取原始 `x.Payload`。
  2. 计算 `obfCode := CalObfuscationCode(x.Payload)`。
  3. 将 `obfCode` 写入 `imcontext.StateKey_ObfuscationCode`。
  4. 调用 `DoObfuscation(obfCode, x.Payload)` 对 payload 做一次异或混淆。
  5. 再将被混淆过的 `Payload` 反序列化为 `ConnectMsgBody`。

这一步等于在**第一次 Connect 请求**时，基于 payload 派生出每个连接独有的混淆码。

### 4.2 后续消息的加解密

对于除 `Cmd_Connect` 以外的所有命令：

- `Decrypt(ctx)`：
  - 从上下文中读取 `obfCode`（`GetObfuscationCodeFromCtx`）。
  - 使用 `DoObfuscation` 对 `Payload` 做反混淆。
  - 再根据 `Cmd` 反序列化为对应的具体消息体（PublishMsgBody / QueryMsgBody 等）。

- `Encrypt(ctx)`：
  - 根据 `Cmd` 将消息体重新序列化为 `Payload`。
  - 从上下文取出 `obfCode` 后，调用 `DoObfuscation` 混淆。
  - 将 `Payload` 设置到 `ImWebsocketMsg`，并清空原来的 oneof（`Testof`）。

> 从设计上看，这套机制主要提供轻量级的“混淆”和一定程度的重放防护，而不是强加密；客户端 SDK 需要用同样逻辑实现，否则无法正确收发。

---

## 5. WebSocket 消息处理流程（IMWebsocketMsgHandler）

核心入口在 `services/connectmanager/server/imwebsocketmsghandler.go` 中的：

- `IMWebsocketMsgHandler.HandleRead(ctx imcontext.WsHandleContext, message interface{})`
- `IMWebsocketMsgHandler.HandleException(ctx, code, ex)`

### 4.1 HandleRead 主逻辑

简化后的逻辑为：

1. 将 `message` 转为 `*codec.ImWebsocketMsg`。
2. 根据 `wsMsg.Cmd` 分支处理：
   - `Cmd_Connect`：
     - 调用 `listener.Connected(wsMsg.GetConnectMsgBody(), ctx)`。
   - `Cmd_Disconnect`：
     - 若 `!imcontext.CheckConnected(ctx)`：直接关闭连接。
     - 构造/补全 `DisconnectMsgBody`，调用 `listener.Diconnected(disconnectMsg, ctx)`。
   - `Cmd_Ping`：
     - 若未连接：关闭连接。
     - 否则调用 `listener.PingArrived(ctx)`（通常会在 listener 中进行 Pong 或心跳更新）。
   - `Cmd_Publish`：
     - 若未连接：关闭连接。
     - 否则调用 `listener.PublishArrived(wsMsg.GetPublishMsgBody(), int(wsMsg.GetQos()), ctx)`，带上 QoS。
   - `Cmd_PublishAck`：
     - 若未连接：关闭连接。
     - 否则调用 `listener.PubAckArrived(wsMsg.GetPubAckMsgBody(), ctx)`。
   - `Cmd_Query`：
     - 若未连接：关闭连接。
     - 否则调用 `listener.QueryArrived(wsMsg.GetQryMsgBody(), ctx)`。
   - `Cmd_QueryConfirm`：
     - 若未连接：关闭连接。
     - 否则调用 `listener.QueryConfirmArrived(wsMsg.GetQryConfirmMsgBody(), ctx)`。
   - 其它未知 Cmd：
     - 关闭连接。
     - 若 `imcontext.CheckConnected(ctx)` 为真，再调用 `listener.ExceptionCaught` 上报错误。

> 可以看到，**除 Connect 外的所有命令在处理前都会做“是否已连接”的检查**，这是保护服务端的一层安全措施，防止非法/半握手状态下的业务调用。

### 4.2 异常处理（HandleException）

- `HandleException` 只是简单地将异常转发给 `listener.ExceptionCaught`。
- 通常 listener 的实现会：
  - 记录错误日志。
  - 决定是否关闭连接、是否返回错误包给客户端。

---

## 6. 典型交互时序（客户端视角）

以下是一个典型的 WebSocket 会话流程（伪代码级别）：

1. **建立连接**
   - 客户端通过 `ws://host:port/path` 建立 WebSocket 物理连接。
   - 建立成功后，立即发送 `Cmd_Connect` 帧，携带：
     - 用户标识（userId / deviceId）。
     - 鉴权凭证（token / 签名）。
     - 客户端版本、平台信息等。
   - 服务端校验成功后，返回 `Cmd_ConnectAck`。

2. **维持心跳 & 服务器空闲检测**
   - 客户端每隔固定时间发送 `Cmd_Ping`。
   - 服务端在 `PingArrived` 中直接 `ctx.Write(codec.NewPongMessage())` 返回 `Cmd_Pong`。
   - 同时，`ImWebsocketChild.startTicker` 会每 5 秒检查一次 `latestActiveTime`：
     - 若 5 分钟内无任何读到的消息，则触发 `HandleException`，以“心跳超时”关闭连接。

3. **发送业务消息**
   - 客户端构造 `Cmd_Publish` 帧，设置：
     - `QoS`：如果希望确认送达，设为 `QoS_NeedAck`。
     - 消息体：例如单聊/群聊消息的 protobuf/JSON 序列化结果。
   - 服务端在 `PublishArrived` 中：
     - 做鉴权/限流/敏感词过滤等校验。
     - 将消息投递给 Message / Conversation / History 等服务。
     - 若 `QoS_NeedAck`，在完成关键动作后发送 `Cmd_PublishAck`。

4. **查询与确认（历史消息 / RTC / 其它查询）**
   - 客户端发送 `Cmd_Query`，请求补拉历史消息或查询状态。
   - 服务端处理后返回 `Cmd_QueryAck` 携带结果列表。
   - 客户端消费完一批后，发送 `Cmd_QueryConfirm` 告知服务端“本批已处理”，服务端可据此更新游标或清理缓存。

5. **断开连接 & 多端互踢**
   - 客户端主动发送 `Cmd_Disconnect`：
     - 服务端在 `Diconnected` 逻辑中释放资源、更新用户在线状态。
     - 然后关闭底层 WebSocket。
   - 或服务端在错误/超时/多端登录互踢等场景下主动关闭连接：
     - 通过 `services.PutInContextCache` / `RemoveFromContextCache` 维护 `OnlineUserConnectMap` 和 `OnlineSessionConnectMap`。
     - 当同一用户的新连接建立时，根据 `KickMode` 和 `Platform` 决定是否对旧连接发送 `Disconnect` + 关闭。

---

## 7. HTTP /im/publish 与 WebSocket 的关系

除了 WebSocket 上行外，ConnectManager 还提供了一个 HTTP 入口来发送“等价的 Publish”：

- 端点：`POST /im/publish`
- 处理函数：`imhttpmsghandlers.ImHttpPubHandler`
- 请求要求：
  - Header：
    - `x-token`：业务侧的用户 token（会通过 `tokens.ParseTokenString` + appSecret 校验）。
    - `x-appkey`、`x-deviceid`、`x-instanceid`、`x-platform`：补充上下文。
  - Body：
    - 一个 protobuf 序列化后的 `ImWebsocketMsg`，其中必须是 `Cmd_Publish`，并带有合法的 `PublishMsgBody`。

处理流程：

1. 解析 body 为 `ImWebsocketMsg`，取出 `PublishMsgBody`。
2. 校验 token、topic、targetId 等。
3. 构造 `pbobjs.RpcMessageWraper`，调用 `bases.SyncUnicastRoute` 将消息投递到下游服务。
4. 将下游返回的结果封装为一个 `PublishAckMsgBody`，再构造 `UserPublishAckMessage`。
5. 使用 `tools.PbMarshal(ack.ToImWebsocketMsg())` 直接回写给 HTTP 客户端。

> 这意味着：**/im/publish 是一个“无需长连接的 HTTP 上行通道”，协议层面与 WebSocket 的 Publish + PublishAck 完全兼容（都是 `ImWebsocketMsg` + `PublishMsgBody`/`PublishAckMsgBody`）。**

---

## 8. 如何在项目中扩展 WebSocket 协议

如果你需要在现有协议上扩展新的命令或字段，可以遵循以下思路：

1. **在 codec 层增加 Cmd 常量**
   - 在 `message.go` 中添加新的 `Cmd_Xxx` 常量，并确保不会与现有值冲突（0～15 共 16 种）。
   - 如有必要，为新命令设计对应的消息体结构（通常在 codec 或 pb 中定义）。

2. **在 `ImWebsocketMsg` 中增加访问方法**
   - 为新命令增加 `GetXxxMsgBody()` / `SetXxxMsgBody()` 等方法（具体按现有风格实现）。

3. **在 `IMWebsocketMsgHandler` 中增加分支**
   - 在 `switch wsMsg.Cmd` 中加入新 Cmd case。
   - 调用 `listener` 上新的回调方法（需要在 `ImListener` 接口中先定义）。

4. **在具体业务 listener 中实现逻辑**
   - 找到 ConnectManager 对应的 `ImListener` 实现类，为新回调实现业务处理。
   - 里面可以调用内部 actor / service 完成实际业务。

5. **更新客户端 SDK**
   - 在 Go / 移动端 / Web SDK 中同步更新命令常量、编码/解码逻辑。
   - 保证客户端和服务端的协议版本匹配。

扩展时建议先在测试环境新增命令，并通过 WebSocket 调试工具验证收发流程，再逐步接入正式客户端。

---

## 9. 端到端完整流程梳理

本节把前面零散的内容串成一个“从连接建立到消息收发”的完整流程，按照真实代码路径来写，方便你对照调试。

### 9.1 建立连接（/im）

1. **客户端发起 WebSocket 连接**
   - 请求：`GET /im`，由 `ImWebsocketServer.ImWsServer` 处理。
   - 通过 gorilla/websocket 完成协议升级，拿到 `*websocket.Conn`。
2. **创建连接 child 与上下文**
   - `ImWebsocketServer` 为每个连接创建一个 `ImWebsocketChild`：
     - 保存 `wsConn`、`messageListener`（通常是 `ImListenerImpl`）、`latestActiveTime` 等。
   - 调用 `child.startWsListener(referer, clientIp)`：
     - 创建 `WsHandleContextImpl`，作为后续所有处理的 `imcontext.WsHandleContext` 实例。
     - 在 `imcontext` 中写入：
       - `StateKey_ConnectSession`：随机 session id。
       - `StateKey_ConnectCreateTime`：连接建立时间戳。
       - `StateKey_Limiter`：限流器 `rate.Limiter`。
       - `StateKey_Referer`、`StateKey_ClientIp` 等。
   - 启动心跳检测 ticker（每 5 秒检查一次 `latestActiveTime`，超 5 分钟则关闭连接）。

### 9.2 Connect 握手与登录

3. **客户端发送 `Cmd_Connect` 帧**
   - WebSocket 上发出一个 protobuf 编码的 `ImWebsocketMsg`：
     - `Cmd = Cmd_Connect`
     - `Payload` 中是序列化后的 `ConnectMsgBody`（未混淆）。
4. **服务端读取并解码帧**
   - `ReadMessage` 读到二进制帧 → `tools.PbUnMarshal` 反序列化为 `ImWebsocketMsg`。
   - 调用 `wsMsg.Decrypt(ctx)`：
     - 对 `Cmd_Connect`：
       - 根据 `Payload` 计算 obfuscation code，写入 `StateKey_ObfuscationCode`。
       - 对 `Payload` 做一次 `DoObfuscation`，再反序列化为 `ConnectMsgBody`。
5. **Cmd 分发到 `ImListener.Connected`**
   - `IMWebsocketMsgHandler.HandleRead` 根据 `Cmd_Connect` 调用 `ImListenerImpl.Connected`。
6. **登录校验与上下文初始化**
   - `Connected` 中：
     - 记录 connect log（`UserConnectLog`）。
     - 通过 `services.CheckLogin` 校验 token/appkey 等：
       - 失败：写入 `ConnectAck`（失败 code + ext），短延时后 `ctx.Close`。
       - 成功：
         - 标记 `StateKey_Connected = "1"`。
         - 在 `imcontext` 中写入：
           - `StateKey_Appkey`、`StateKey_UserID`、`StateKey_Platform`、`StateKey_Version`、`StateKey_DeviceID` 等。
         - 调用 `services.PutInContextCache`：
           - 将 `session -> ctx`，`(appkey, userid) -> { session -> ctx }` 记录到全局 map。
           - 根据 app 配置进行多端互踢（必要时给旧连接发送 `Disconnect`，再关闭）。
         - 注册 PushToken、上报 Online 状态。
         - 写 connect 成功日志和 `ConnectAck`（成功 code + session + userId）。

### 9.3 心跳与超时

7. **客户端定期发送 `Cmd_Ping`**
   - 帧格式仍是 `ImWebsocketMsg`，只不过 `Cmd = Cmd_Ping`，通常无复杂 payload。
8. **服务端响应 `Cmd_Pong`**
   - `IMWebsocketMsgHandler` 将 Cmd 路由到 `ImListenerImpl.PingArrived`。
   - `PingArrived` 中直接 `ctx.Write(codec.NewPongMessage())` 发送 `Cmd_Pong`。
   - `ImWebsocketChild` 的 `latestActiveTime` 在每次 `ReadMessage` 后都会更新。
9. **空闲超时断开**
   - ticker 每 5 秒检查一次 `current - latestActiveTime`。
   - 若超过 5 分钟：
     - 调用 `HandleException(..., IMErrorCode_CONNECT_CLOSE_HEARTBEAT_TIMEOUT, ...)`。
     - `ExceptionCaught` 里会写日志、调用 `services.Offline`，并从 `ContextCache` 移除。
     - 最终通过 `ctx.Close` 关闭连接。

### 9.4 上行业务消息（Publish）

10. **客户端发送 `Cmd_Publish` 帧**
    - `ImWebsocketMsg` 中：
      - `Cmd = Cmd_Publish`
      - `Qos` 通常为 `QoS_NoAck` 或 `QoS_NeedAck`。
      - `Payload` 是混淆后的 `PublishMsgBody`（客户端使用 obfCode 加密）。
11. **服务端解密并解析 Publish**
    - 读帧、`PbUnMarshal` 得到 `ImWebsocketMsg`。
    - `Decrypt(ctx)` 使用 obfCode 对 `Payload` 解混淆，并反序列化到 `PublishMsgBody`。
12. **路由到 `ImListenerImpl.PublishArrived`**
    - 记录 connection log。
    - 校验必要参数（topic/targetId）：
      - 不合法：立即 `NewUserPublishAckMessage` + 写回 ack（带错误码）。
    - 调用限流器（`StateKey_Limiter`）：
      - 超限：返回 `IMErrorCode_CONNECT_EXCEEDLIMITED` 的 ack。
13. **转发到下游业务服务**
    - 通过 `bases.UnicastRoute` 构造 `pbobjs.RpcMessageWraper`：
      - `RpcMsgType = UserPub`
      - `Method = "upstream"`，真实业务方法名放在 `ExtParams[RpcExtKey_RealMethod]`。
      - `RequesterId`、`TargetId`、`AppDataBytes` 等。
    - 如果路由失败（无对应业务 handler）：
      - 返回 `IMErrorCode_CONNECT_UNSUPPORTEDTOPIC` 的 ack。
14. **业务服务处理成功后（下游流程）**
    - 业务侧会生成下行消息，并在需要时通过 ConnectManager 的 server-pub 路径，把消息推送到目标用户的在线连接上（略，属于“服务端推送”路径，不在本节展开）。

### 9.5 查询与确认（Query / QueryAck / QueryConfirm）

15. **客户端发送 `Cmd_Query` 帧**
    - 用于补拉历史消息、查询状态、创建 RTC 房间等。
16. **服务端在 `QueryArrived` 中处理**
    - 记录日志。
    - 校验 topic/targetId、限流。
    - 调用 `services.PreProcessRtcCreate` 做部分 RTC 特殊处理。
    - 如果是历史消息查询，可能经过 `services.HisMsgRedirect` 调整 targetId。
    - 通过 `bases.UnicastRoute` 将查询转发给对应业务服务，`Qos = QoS_NeedAck`。
17. **业务服务返回 `Cmd_QueryAck`**
    - Response 通过 ConnectManager 反向写入 WebSocket 连接，承载结果集。
18. **客户端处理完一批后发送 `Cmd_QueryConfirm`**
    - 由 `QueryConfirmArrived` 处理：
      - 根据 index 在 `QueryAckCallback` map 中查找并执行注册的回调。
      - 写日志，便于追踪“客户端已经确认收到/处理完哪一批数据”。

### 9.6 断开与多端登录

19. **客户端主动断开（Cmd_Disconnect）**
    - 带上 `DisconnectMsgBody` 的 code 等信息。
    - `ImListenerImpl.Diconnected`：
      - 根据 code 决定是否清理 push token。
      - 调用 `services.Offline`，从 ContextCache 移除。
      - `ctx.Close` 关闭连接。
20. **服务端被动断开**
    - 网络错误：读写异常时在 `startWsListener` 中触发 `HandleException`。
    - 心跳超时：ticker 触发异常，`IMErrorCode_CONNECT_CLOSE_HEARTBEAT_TIMEOUT`。
    - 多端互踢：
      - 在 `PutInContextCache` 检测到同一用户在不允许并存的平台重复登录。
      - 为旧连接构造 `Disconnect` 消息（包含合适的 code），写入后短延时关闭。

---

## 10. 大群（如 10 万人）消息下发流程

这一节从“有用户在大群里发了一条消息”出发，说明消息如何通过 WebSocket 链路触达到其他成员。

### 10.1 上行：发送群消息

1. **群成员 A 客户端发送上行 Publish**
   - 通过 WebSocket 向 `/im` 建立的长连接发送一帧：
     - `Cmd = Cmd_Publish`
     - `PublishMsgBody` 中：
       - `Topic`：表示“发送群消息”的上行方法（具体值由 SDK 约定）。
       - `TargetId`：群 ID。
       - `Data`：业务层封装的消息内容（文本/图片/@ 信息等）。
   - 这帧数据在 ConnectManager 入口的处理与 9.4 节完全一致。
2. **ConnectManager 接收并路由**
   - `ImWebsocketChild.startWsListener` 读到二进制帧，反序列化为 `ImWebsocketMsg`。
   - `Decrypt(ctx)` 使用 obfCode 解混淆 payload，得到 `PublishMsgBody`。
   - `IMWebsocketMsgHandler.HandleRead` 根据 `Cmd_Publish` 调用 `ImListenerImpl.PublishArrived`。
   - `PublishArrived` 中：
     - 记录 connection log。
     - 校验 `Topic`、`TargetId` 是否为空。
     - 应用限流（`StateKey_Limiter`）。
     - 构造 `pbobjs.RpcMessageWraper{ RpcMsgType: UserPub, Method: "upstream", ExtParams[RpcExtKey_RealMethod] = Topic }`。
     - 通过 `bases.UnicastRoute(..., "connect")` 投递到下游 **消息/群业务服务**。

> 到这一步，**只是把“发送群消息”的意图从 WebSocket 层路由到了消息业务层**。

### 10.2 中间：消息持久化与会话/历史更新

在消息业务层（Message / Group / Conversation / History 服务组合）中，典型流程是：

1. **校验与权限**
   - 检查群是否存在，发送者是否在群内，是否被禁言，群是否处于全员禁言等（会调用 Group 服务）。
2. **生成并写入消息**
   - 生成消息 ID、消息 seq 等。
   - 将该群消息写入消息存储（MySQL / HBase / Mongo 等，具体实现见各服务 `storages`）。
3. **更新会话信息**
   - 通知 Conversation 服务：
     - 更新每个群成员对应的群会话记录（最新消息、未读数等）。
     - 大群场景下通常会做批量处理或惰性更新，避免逐个操作 10 万行。
4. **记录历史消息**
   - 通知 HistoryMsg 服务，将消息归档到群的历史消息集合中，供后续补拉。

> 这一阶段主要是“算清楚这条消息是什么、写到哪儿”，还没真正对 10 万个终端推送。

### 10.3 下行：在线成员的实时 WebSocket 推送

当消息业务层确认写入成功后，会向 ConnectManager 发起下行投递（服务间 RPC）：

1. **业务层发起下行请求**
   - 对于这条群消息涉及到的每个用户：
     - 调用 ConnectManager 暴露的下行 RPC（形如“给某个 userId 下发一条群消息”）。
2. **ConnectManager 查找在线连接**
   - 在下行实现中（位于 ConnectManager 的服务层）：
     - 根据 `appkey + userId` 查询 `services.OnlineUserConnectMap`：
       - 得到该用户当前所有在线连接的 `WsHandleContext`（手机、Web、PC、多实例等）。
3. **构造并写入下行帧**
   - 为每个在线连接：
     - 构造实现了 `codec.IMessage` 的下行消息对象（例如 Server 端下行的 Publish/DownMsg）。
     - 调用 `ToImWebsocketMsg()` 得到一个 `ImWebsocketMsg`，`Cmd` 通常仍为 `Cmd_Publish`。
     - 使用连接对应的 obfCode 执行 `Encrypt(ctx)`。
     - 调用 `WsHandleContextImpl.Write`，最终通过 gorilla/websocket 写出一帧二进制。
4. **客户端 SDK 处理下行 Publish**
   - 在客户端 SDK 内：
     - 接收到 `Cmd_Publish` 的下行帧。
     - 解混淆 payload，解析 `PublishMsgBody`。
     - 根据下行 `Topic`（例如 `message/group/down`）分发到本地回调，展示在聊天 UI 中。

> 对 10 万人大群来说，ConnectManager 会采用 goroutine 池、批量调度等方式并发下发给当前在线的所有终端，**协议上仍然是“一人一帧的下行 Publish”**，只是发送过程是高并发、分批完成的。

### 10.4 离线成员的推送与补拉

对于当前不在线的群成员，流程会分两部分完成：

1. **离线推送（PushManager）**
   - 在消息业务层/ConnectManager 路由过程中，发现某个 userId 没有在线连接时：
     - 将该消息标记为“离线消息”，并交给 PushManager。
   - PushManager：
     - 查找该用户的设备 token、推送通道（APNs/厂商推送等）。
     - 构造系统推送（标题、摘要、badge 未读数等）。
     - 调用对应厂商 SDK 发送通知。
2. **历史补拉（HistoryMsg + Query）**
   - 离线用户在稍后重新上线后：
     - 通过 WebSocket 上的 `Cmd_Query` 或 HTTP API 调用“拉取历史消息”接口。
     - ConnectManager 将 Query 路由到 HistoryMsg 服务。
     - HistoryMsg 从存储中查出该群最近的未读消息列表，封装为 `QueryAck` 返回。
     - 客户端收到 `Cmd_QueryAck` 后，补齐 UI 上遗漏的群消息。

> 总结起来，一条群消息对 10 万人的传播，**被拆成“实时在线 WebSocket 下发 + 离线推送通知 + 登录后补拉历史”三部分完成**，既保证实时性，又避免对存储和网络造成瞬时极端压力。

---

## 11. 调试与排错建议

- **抓包与协议校验**
  - 使用 WebSocket 调试工具或代理（如 Charles、Fiddler）查看原始帧。
  - 对照 `Version`、`HeaderCode`、`Checksum`、长度字段确认编码是否正确。

- **日志与错误码**
  - ConnectManager 层的日志通常会输出连接 id、用户 id、Cmd 类型和错误码。
  - 若频繁出现“数据非法”导致的断连，重点检查：
    - Cmd 值是否在 0～9 之间。
    - HeaderCode 与 body 一致性（Checksum 校验）。

- **本地模拟客户端**
  - 推荐写一个最小化的本地 CLI 客户端：
    - 使用 `gorilla/websocket` 建立连接。
    - 手工构造 `MsgHeader` + body，测试不同 Cmd 与 QoS 组合。
  - 方便你在不依赖完整 App 的前提下快速验证协议行为。

通过理解和掌握本篇所述内容，你基本可以把 `im-server` 的 WebSocket 层当作一套“可扩展的二进制命令协议接入网关”来使用和修改，无论是排查线上问题还是扩展新特性都会更加游刃有余。

