## 广播消息流程详解（im-server）

本文专门说明 `im-server` 里“广播消息”的实现方式，即如何把一条“系统级广播/全员通知”分发到所有节点、所有用户的收件箱，并最终通过 WebSocket/HTTP 下发给在线终端。

---

## 1. 整体结构与参与模块

- **broadcast 服务（`services/broadcast`）**
  - 负责接收“广播上行”的指令，将其转换为一条通用的下行消息 `DownMsg`。
  - 将这条 `DownMsg`：
    - 写入“广播专用历史/收件箱”。
    - 通过集群广播给所有相关节点。

- **message 服务（广播 inbox）**
  - `services/message/actors/broadcastinboxactor.go`
  - 将广播消息存入“广播收件箱”，供每个用户/终端在需要时读取/补拉。

- **公共集群/路由层（`commons/bases` + `commons/gmicro`）**
  - `bases.Broadcast` / `BroadcastRouteWithNoSender` / `cluster.BroadcastWithNoSender`：
    - 负责把一条 RPC（`pbobjs.RpcMessageWraper`）广播到所有目标节点。

- **ConnectManager / PushManager / HistoryMsg 等**
  - 广播消息最终在这些模块的参与下，变成：
    - 在线用户的 WebSocket 下行消息。
    - 不在线用户的离线推送 + 历史补拉。

> 关键点：**“广播”主要是在服务器内部/集群层面做 fanout，WebSocket 协议本身依然是一人一帧的 `Cmd_Publish`。**

---

## 2. 广播上行入口（谁来触发广播）

### 2.1 API 网关入口

在 `services/apigateway/routers/router.go` 中，有一个用于发送广播的 HTTP API（示例）：

- `POST /messages/broadcast/send` → `apis.SendBroadCastMsg`
  - 业务服务器或管理后台可以调用这个接口，发起一条“系统广播消息”（比如全站公告、系统通知）。
  - API 层会组装一个通用的上行结构 `pbobjs.UpMsg`，包含：
    - `MsgType`：消息类型（文本、通知等）。
    - `MsgContent`：实际内容（序列化后的业务 JSON/protobuf）。
    - `Flags`：标志位（是否状态消息、是否只写不推等）。

### 2.2 交给 broadcast 服务

API 层收到请求后，会通过内部 RPC 调用 **broadcast 服务** 的 actor（见第 3 节），让它来执行“真正的广播逻辑”：

- `services/broadcast/actors/BroadcastMsgActor` 接收 `UpMsg`，调用 `services.BroadcastMsg`。

---

## 3. broadcast 服务核心逻辑：BroadcastMsg

### 3.1 BroadcastMsg 函数（关键代码）

`services/broadcast/services/broadcastservice.go`：

- 函数签名：

```go
func BroadcastMsg(ctx context.Context, msg *pbobjs.UpMsg) (errs.IMErrorCode, string, int64, int64)
```

- 主要步骤：
  1. **确定发送者与会话标识**
     - `senderId := bases.GetRequesterIdFromCtx(ctx)`
     - `converId := commonservices.GetConversationId(senderId, senderId, ChannelType_BroadCast)`
  2. **生成消息 ID / 时间戳 / seq**
     - 使用 `convercache.GetMsgConverCache(...).GenerateMsgId(...)`：
       - 得到 `msgId`、`sendTime`、`msgSeq`。
  3. **构造下行消息 `DownMsg`**
     - `DownMsg` 字段：
       - `SenderId = senderId`
       - `TargetId = senderId`（在广播语义下，target 更偏向“广播会话”而非具体用户）
       - `ChannelType = ChannelType_BroadCast`
       - `MsgType`、`MsgContent`、`Flags` 等来自上行 `UpMsg`。
  4. **写入广播历史**
     - 如果不是“状态消息”（`!msgdefines.IsStateMsg(msg.Flags)`）：
       - 调用 `commonservices.SaveHistoryMsg(ctx, senderId, "", ChannelType_BroadCast, downMsg, 0)`。
  5. **保存到“广播 inbox”**
     - `bases.AsyncRpcCall(ctx, "brd_inbox", senderId, downMsg)`：
       - 异步调用 message 服务中的 `BrdcastInboxActor`，把广播消息写入广播收件箱。
  6. **集群广播到所有 message 节点**
     - `bases.Broadcast(ctx, "brd_append", downMsg)`：
       - 通过公共集群接口，把一条 `RpcMessageWraper` 广播到所有需要处理广播的节点。

> 到这里，一条广播消息已经被“标准化”为一个 `DownMsg`，写入历史、收件箱，并通过集群广播传递到所有 message 节点上，方便每个节点对自己管理的用户做后续 fanout。

---

## 4. Broadcast 底层：bases.Broadcast / Cluster.BroadcastWithNoSender

在 `commons/bases/base.go` 中，`Broadcast` 的实现：

```go
func Broadcast(ctx context.Context, method string, req proto.Message, opts ...BaseActorOption) {
    if len(opts) > 0 {
        for _, opt := range opts {
            ctx = opt.HandleCtx(ctx)
        }
    }
    dataBytes, _ := tools.PbMarshal(req)
    BroadcastRouteWithNoSender(&pbobjs.RpcMessageWraper{
        RpcMsgType:   pbobjs.RpcMsgType_ServerPub,
        AppKey:       GetAppKeyFromCtx(ctx),
        Session:      GetSessionFromCtx(ctx),
        Method:       method,
        RequesterId:  GetRequesterIdFromCtx(ctx),
        ReqIndex:     GetSeqIndexFromCtx(ctx),
        Qos:          GetQosFromCtx(ctx),
        AppDataBytes: dataBytes,
        OnlySendbox:  GetOnlySendboxFromCtx(ctx),
        NoSendbox:    GetNoSendboxFromCtx(ctx),
        IsFromApi:    GetIsFromApiFromCtx(ctx),
        IsFromApp:    GetIsFromAppFromCtx(ctx),
        TargetIds:    GetTargetIdsFromCtx(ctx),
        ExtParams:    GetExtsFromCtx(ctx),
        MsgId:        GetMsgIdFromCtx(ctx),
        DelMsgId:     GetDelMsgIdFromCtx(ctx),
    })
}
```

- 关键点：
  - **广播的“单位”是 `RpcMessageWraper`**，不是具体用户 ID。
  - `Method = "brd_append"`（在 BroadcastMsg 中传入），表示要在目标服务上调用哪个 actor 逻辑。
  - `cluster.BroadcastWithNoSender` 会将这条 RPC 发送到所有符合路由规则的节点。

在 `commons/gmicro/cluster.go` 中，`BroadcastWithNoSender` 会：

- 遍历所有注册了该 `method` 的节点。
- 将 `RpcMessageWraper` 投递给每个节点上的对应 actor。

> 换句话说，**Broadcast 是“多节点多 actor 级别的广播”，不是“WebSocket 帧级”的广播**。

---

## 5. 广播 Inbox：BrdcastInboxActor

`services/message/actors/broadcastinboxactor.go`：

- Actor：`BrdcastInboxActor`
  - 接收的消息类型：`*pbobjs.DownMsg`（广播下行消息）。
  - 处理逻辑：
    - 写日志，标记 `msg_type`、`msg_id`、`msg_time`。
    - 调用 `services.SaveBroadcastMsg(ctx, msg)`：
      - 把广播消息写入“广播专用”的存储（`IBroadcastMsgStorage` 实现）。
    - 回发 `QueryAckWraper` 给发送方（通常是 broadcast 服务）。

> 这一步是将广播消息落地到 **message 服务的“广播收件箱/历史表”**，方便后续按用户、按终端、按时间窗口查询。

---

## 6. 广播如何真正“送到用户终端”

广播本身只是“把一条系统消息复制到所有人都能看到的位置”，真正“送到用户设备”会结合以下路径：

1. **在线用户的实时下发**
   - 当某个用户在广播消息写入时已经在线：
     - message / connectmanager 会通过正常的下行路径（ServerPub → ImWebsocketMsg(Cmd_Publish)）将这条广播以“普通下行消息”的形式发给在线连接。
   - 对终端来说，这只是一条“来自系统/某个特殊 senderId 的 DownMsg”，展示在一个“系统通知/广播频道”里。

2. **离线用户的补拉**
   - 离线用户在稍后登录时：
     - 通过 WebSocket 的 `Cmd_Query` 或 HTTP API 调用“拉取广播消息”的接口。
     - HistoryMsg/Message 基于广播收件箱表，为该用户返回最近的未读广播消息列表。

3. **可选的离线推送**
   - 若业务需要，可以为广播消息配置 Push 策略：
     - 在写入广播收件箱后，由 PushManager 根据规则推送通知（例如“系统公告”、“运营活动”等）。

> 总体上，广播消息对客户端协议层是“普通消息”的一种特例：区别在于 `ChannelType = BroadCast`、展示入口可能是“系统消息列表”，以及服务端在存储与 fanout 策略上更偏向“写一次、多地读”。

---

## 7. 广播与普通群聊的区别总结

- **目标范围：**
  - 群聊：一条消息的目标集是“某个群的成员”（可能是 N 人）。
  - 广播：目标是“所有用户”或符合某个条件的一大批用户。

- **分发机制：**
  - 群聊：
    - Message 服务根据群成员列表，为每个成员生成会话/未读，并通过 ConnectManager 做 per-user 下行。
  - 广播：
    - Broadcast 服务生成一条 DownMsg。
    - 通过 `bases.Broadcast` 把这条消息广播到所有 message 节点。
    - 每个节点将其写入广播收件箱/历史，后续按需下发/补拉。

- **协议层（WebSocket）的表现：**
  - 二者对终端来说，最终都表现为：
    - `Cmd_Publish` 下行。
    - 只是 `ChannelType`、`MsgType` 和展示逻辑不同。

---

通过理解本指南，你可以：

- 清楚地区分“业务层的广播（广播到所有用户）”和“协议层的一人一帧下行”。
- 了解广播消息在 `im-server` 中的落点：历史、广播收件箱，以及如何通过 ConnectManager/HistoryMsg 到达终端。
- 在需要时，安全地扩展自己的广播逻辑（例如按标签、按分组广播），只需要在 broadcast 服务里定制目标集合和存储策略，再复用现有的集群广播与下行通道。

