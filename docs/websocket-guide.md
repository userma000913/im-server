## WebSocket 接入与协议详解（im-server）

本文档聚焦 `im-server` 中的 WebSocket 接入层与自定义协议实现，结合 `services/connectmanager` 与 `services/connectmanager/server/codec` 目录的实际代码，帮助你从“握手建立连接”一路理解到“消息收发与 Ack / 查询确认”的完整链路。

---

## 1. 总体结构与职责分层

- **ConnectManager 服务（`services/connectmanager`）**
  - 负责 WebSocket 长连接的建立、鉴权、心跳、断线处理。
  - 负责解析自定义二进制协议，将其转换为内部 `ImWebsocketMsg` 结构。
  - 按 `Cmd` 类型分发到不同的业务处理钩子（`ImListener` 接口）。

- **Codec 模块（`services/connectmanager/server/codec`）**
  - 定义协议常量：版本号、协议标识、命令字（Cmd）、QoS 等。
  - 定义统一消息头 `MsgHeader`，实现编码/解码、校验和验证。
  - 提供接口 `IMessage` 与 `ImWebsocketMsg` 等结构，用于在底层字节流与上层业务结构之间转换。

- **业务监听器（`ImListener` 实现）**
  - `IMWebsocketMsgHandler` 解码出 `ImWebsocketMsg` 后，会根据 `Cmd` 调用不同的监听方法：
    - `Connected` / `Diconnected`
    - `PingArrived`
    - `PublishArrived` / `PubAckArrived`
    - `QueryArrived` / `QueryConfirmArrived`
  - 这些监听方法中再去调用内部 Message / Conversation / User 等服务。

这三层共同组成了 WebSocket 接入“前门”：**网络连接 -> 自定义帧解析 -> 业务回调**。

---

## 2. 协议常量与命令字（Cmd）

在 `services/connectmanager/server/codec/message.go` 中可以看到：

- **协议版本与标识**
  - `Version_1 byte = 1`
  - `ProtoId string = "jug9le1m"`（一般用于握手阶段协议确认）

- **QoS（消息质量）**
  - `QoS_NoAck = 0`：不需要 Ack，适合心跳或对可靠性要求不高的通知。
  - `QoS_NeedAck = 1`：需要 Ack，适合必须确认送达的消息。

- **命令字 Cmd（4 bit 编码）**
  - `Cmd_Connect = 0`：客户端 -> 服务端，请求建立会话（携带 token / userId 等）。
  - `Cmd_ConnectAck = 1`：服务端 -> 客户端，连接确认。
  - `Cmd_Disconnect = 2`：客户端主动断开连接。
  - `Cmd_Publish = 3`：双向消息发送命令（文本/图片等业务消息通过这里承载）。
  - `Cmd_PublishAck = 4`：对于 `QoS_NeedAck` 的 Publish，服务端或客户端返回 Ack。
  - `Cmd_Query = 5`：查询类命令（如补拉消息、查询状态等，具体取决于上层定义）。
  - `Cmd_QueryAck = 6`：Query 的应答。
  - `Cmd_QueryConfirm = 7`：对 Query 结果的确认（例如“客户端已处理完本批数据”）。
  - `Cmd_Ping = 8`：心跳 Ping。
  - `Cmd_Pong = 9`：心跳 Pong。

从实现可以看到，**Cmd 和 QoS 都被压缩编码在 `HeaderCode` 这个 1 byte 字段里**：

- 高 4 bit：Cmd
- 低 2 bit：QoS

这意味着协议在带宽上较为节省，适合高频 IM 消息场景。

---

## 3. 消息头与体的编码格式

### 3.1 消息头结构（`MsgHeader`）

`MsgHeader` 在 `message.go` 中定义：

- 字段：
  - `Version byte`：协议版本。
  - `HeaderCode byte`：复用字段，编码了 Cmd 和 QoS。
  - `Checksum byte`：简单 XOR 校验和，用于快速检测包是否损坏。
  - `MsgBodySize int`：消息体长度（变长编码）。

- 编码流程（`EncodeHeader`）：
  1. 写入 `Version`。
  2. 写入 `HeaderCode`（其中已经包含 Cmd 与 QoS）。
  3. 计算并写入 `Checksum = calChecksum(HeaderCode, bodyBytes)`。
  4. 如果命令不是 `Ping`/`Pong`，再写入 `MsgBodySize`（变长整数字节序列）。

- 解码流程（`DecodeHeader`）：
  1. 读取 1 byte 作为 `HeaderCode`。
  2. 再读取 1 byte 作为 `Checksum`。
  3. 若 Cmd 不是 `Ping`/`Pong`，调用 `Bytes2MsgBodySize` 读取 body 长度。

> 这里可以看到：Ping/Pong 被特殊对待，**不携带 `MsgBodySize` 字段**，从而减少心跳包大小。

### 3.2 长度字段编码（变长 int）

- `MsgBodySize2Bytes` 将整型长度编码为一串字节：
  - 每个字节的低 7 bit 存放数据，高 1 bit 作为“是否还有后续字节”的标记。
  - 直到最后一个字节的最高位为 0。
- `Bytes2MsgBodySize` 反向解码。

> 这种模式类似 MQTT / Protobuf 的变长整型编码，适合消息体长度分布不均匀的场景。

### 3.3 校验和验证

- `ValidateChecksum` 会基于 `HeaderCode` 与 body 再算一次 checksum，与包里携带的值对比：
  - 一致：认为包合法。
  - 不一致：包被视为损坏，一般应关闭连接或丢弃。

---

## 4. WebSocket 消息处理流程（IMWebsocketMsgHandler）

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

## 5. 典型交互时序（客户端视角）

以下是一个典型的 WebSocket 会话流程（伪代码级别）：

1. **建立连接**
   - 客户端通过 `ws://host:port/path` 建立 WebSocket 物理连接。
   - 建立成功后，立即发送 `Cmd_Connect` 帧，携带：
     - 用户标识（userId / deviceId）。
     - 鉴权凭证（token / 签名）。
     - 客户端版本、平台信息等。
   - 服务端校验成功后，返回 `Cmd_ConnectAck`。

2. **维持心跳**
   - 客户端每隔固定时间发送 `Cmd_Ping`。
   - 服务端在 `PingArrived` 中更新连接状态，并按需返回 Pong（实现上可以是发送 `Cmd_Pong` 或内部心跳标记）。
   - 若多次心跳丢失，服务端会在其他地方主动关闭连接。

3. **发送业务消息**
   - 客户端构造 `Cmd_Publish` 帧，设置：
     - `QoS`：如果希望确认送达，设为 `QoS_NeedAck`。
     - 消息体：例如单聊/群聊消息的 protobuf/JSON 序列化结果。
   - 服务端在 `PublishArrived` 中：
     - 做鉴权/限流/敏感词过滤等校验。
     - 将消息投递给 Message / Conversation / History 等服务。
     - 若 `QoS_NeedAck`，在完成关键动作后发送 `Cmd_PublishAck`。

4. **查询与确认**
   - 客户端发送 `Cmd_Query`，请求补拉历史消息或查询状态。
   - 服务端处理后返回 `Cmd_QueryAck` 携带结果列表。
   - 客户端消费完一批后，发送 `Cmd_QueryConfirm` 告知服务端“本批已处理”，服务端可据此更新游标或清理缓存。

5. **断开连接**
   - 客户端主动发送 `Cmd_Disconnect`：
     - 服务端在 `Diconnected` 逻辑中释放资源、更新用户在线状态。
     - 然后关闭底层 WebSocket。
   - 或服务端在错误/超时场景下主动关闭连接，并通过 `ExceptionCaught` 记录原因。

---

## 6. 如何在项目中扩展 WebSocket 协议

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

## 7. 调试与排错建议

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

