## AI 机器人 / 大模型接入指南（im-server）

本文说明 `im-server` 如何集成 AI 机器人（大模型），以及如何扩展为对接你自己的 LLM 服务。核心目录为：

- `services/botmsg/`：机器人消息服务（Bot Message Service）
- `services/botmsg/services/botengines/`：对接具体大模型/机器人平台的“引擎”实现
- `services/botmsg/storages/`：机器人会话、上下文等的存储

---

## 1. 整体架构概览

### 1.1 角色划分

- **客户端 / 业务方**
  - 像对普通用户一样向某个“机器人用户”或“机器人群”发送 IM 消息。

- **im-server（本仓库）**
  - 负责接收消息（Message Service）。
  - 当目标是机器人时，将消息路由到 `botmsg` 模块。
  - 调用对应的 Bot Engine（Dify / Coze / SiliconFlow / 自定义等）。
  - 将 AI 回复以 IM 消息的形式写回会话，广播给客户端。

- **外部大模型 / 机器人平台**
  - Dify / Coze / SiliconFlow / 你自建的 LLM 服务。
  - 以 HTTP(S) 接口方式暴露聊天、流式输出等能力。

### 1.2 模块结构

- `services/botmsg/starter.go`
  - 启动入口，将机器人服务注册到内部 Actor / gmicro 系统。

- `services/botmsg/services/botservice.go`
  - 核心对外服务层，封装“与机器人交互”的主逻辑。

- `services/botmsg/services/syncmsgbotservice.go`
  - 与消息系统的同步逻辑：在收到消息时触发机器人回复。

- `services/botmsg/actors/botmsgactor.go`
  - Actor 封装，将单个会话/用户与机器人交互串行化处理。

- `services/botmsg/services/botengines/`
  - `botengine.go`：统一接口 `IBotEngine`。
  - `difyengine.go`：对接 Dify。
  - `cozeengine.go`：对接 Coze。
  - `siliconflowengine.go`：对接 SiliconFlow（支持 tools）。
  - `custombotengine.go`：自定义扩展引擎。

- `services/botmsg/storages/`
  - `storages/models/botconver.go`：机器人会话模型。
  - `storages/dbs/botconverdao.go`：会话持久化 DAO。
  - `storages/storage.go`：Storage 封装。

---

## 2. IBotEngine 接口与内置实现

### 2.1 IBotEngine 定义

`services/botmsg/services/botengines/botengine.go`：

- 接口：
  - `StreamChat(ctx, senderId, targetId string, channelType pbobjs.ChannelType, question string, f func(answerPart string, sectionStart, sectionEnd, isEnd bool))`
    - **流式对话接口**，适合 SSE/流式 LLM 输出。
    - `f` 回调每收到一段内容就被调用一次。
      - `answerPart`：当前片段内容。
      - `sectionStart`：是否是当前“回答片段”的开始。
      - `sectionEnd`：是否是当前“回答片段”的结束。
      - `isEnd`：整个对话是否结束（true 表示本轮完成）。
  - `Chat(ctx, senderId, targetId string, channelType pbobjs.ChannelType, question string) string`
    - **非流式对话接口**，一次性返回完整回答。

- 默认实现：
  - `NilBotEngine`：空实现，返回空字符串或不做任何事，用于未配置引擎时的兜底。

### 2.2 DifyBotEngine（difyengine.go）

- 配置字段：
  - `ApiKey string`：Dify API 鉴权凭证。
  - `Url string`：Dify 对话接口地址。

- 实现要点：
  - `StreamChat`：
    - 构造 `DifyChatMsgReq` 请求体，并设置 `ResponseMode = "streaming"`。
    - 使用 `tools.CreateStream` 发起 HTTP POST，拿到流式响应。
    - 接收每一行 `data:...`，解析为 `DifyStreamRespItem`。
    - 当 `item.Event == "message"` 时，调用回调 `f(item.Answer, sectionStart, false, false)`。
    - 当 `item.Event == "message_end"` 时，调用 `f(item.Answer, false, false, true)` 并结束。

### 2.3 CozeBotEngine（cozeengine.go）

- 配置字段：
  - `Token string`：Coze 鉴权 token。
  - `Url string`：Coze 对话接口地址。
  - `BotId string`：Coze 机器人 ID。

- 会话管理：
  - 根据 `senderId + targetId + channelType` 生成 `converKey`。
  - 调用 `GetCozeConverId` 获取/创建对应的 Coze conversation id：
    - 使用 `commons/caches.LruCache` 做本地缓存。
    - 使用 `services/botmsg/storages` 将 converId 持久化到 DB。
    - 若 DB 无记录，则调用 `createCozeConver` 创建 Coze 会话。

- 流式对话：
  - 通过 `tools.CreateStream` 建立连接。
  - 解析 `event:` / `data:` 前缀，区别 `conversation.message.delta` 和 `conversation.message.completed` 等事件。
  - 按事件类型分段调用回调 `f`，控制 `sectionStart` / `sectionEnd` 标志。

### 2.4 SiliconFlowEngine（siliconflowengine.go）

- 配置字段：
  - `ApiKey string`
  - `Url string`
  - `Model string`

- 流式接口 `StreamChat`：
  - 构造 `SiliconFlowChatReq`，设置 `Stream = true`。
  - 调用 `tools.CreateStream` 发起请求。
  - 对每行 `data:...` 解析为 `SiliconFlowChatResp`。
  - 遍历 `Choices`，读取 `choice.Delta.Content`，调用回调 `f` 输出增量。

- 非流式接口 `Chat`：
  - 设置 `Stream = false`，可选配 `Tools`（函数调用）。
  - 使用 `tools.HttpDoBytesWithTimeout` 发起 HTTP 请求。
  - 直接返回原始响应字符串（调用方可自行解析）。

---

## 3. 消息是如何路由到 AI 机器人的？

> 说明：这一节描述“典型链路”，具体细节可以在 `services/botmsg/services` 和 `actors` 中查看实际代码实现。

### 3.1 用户发消息

1. 客户端向某个“机器人账号”或“机器人会话”发送消息（HTTP 或 WebSocket 入口）。
2. `apigateway` 或 `connectmanager` 收到后，将消息交给 Message Service。
3. Message Service 判断目标（例如根据用户配置、会话类型、特殊前缀等）是否为 Bot：
   - 如果是普通用户 → 按常规 IM 流转。
   - 如果是机器人 → 将这条消息路由到 `botmsg` 服务。

### 3.2 botmsg 服务处理

1. `botmsg` Actor 收到消息事件：
   - 从上下文中拿到 `senderId`、`targetId`、`channelType`、消息内容（`question`）。
2. 根据应用/配置选择具体的 Bot Engine：
   - 例如：某个 appkey 配置为使用 Coze，则取 `CozeBotEngine`。
   - 未配置时使用 `NilBotEngine`（不会返回内容）。
3. 调用 `engine.StreamChat` 或 `engine.Chat`：
   - 对于流式模式：每收到一段回答，就把这一段作为“IM 消息内容的一部分”写回（可以是单条多段，也可以拆成多条）。
   - 对于非流式模式：拿到完整回答后一次性写入会话。
4. 调用 Message / Conversation / History 服务：
   - 将机器人回答作为普通消息写入 DB。
   - 更新会话列表、未读数。
   - 推送给在线客户端。

> 对客户端来说，这些都是“普通消息”，只是 `fromUser` / `from` 字段是一个机器人账号而已。

---

## 4. 如何接入你自己的大模型服务？

下面给出一个最小化的“自定义引擎”接入步骤，假设你有一个 HTTP Chat 接口：

- URL：`https://your-llm.example.com/chat`
- 请求体：`{"query": "用户问题"}`
- 响应体：`{"answer": "大模型回答"}`（非流式）

### 4.1 实现自定义 Engine 结构体

在 `services/botmsg/services/botengines/custombotengine.go`（若已存在则复用）中添加类似实现：

```go
type MyLLMBotEngine struct {
    ApiKey string `json:"api_key"`
    Url    string `json:"url"`
}

func (engine *MyLLMBotEngine) Chat(ctx context.Context, senderId, targetId string, channelType pbobjs.ChannelType, question string) string {
    // 构造请求
    req := map[string]string{"query": question}
    body := tools.ToJson(req)

    headers := map[string]string{
        "Authorization": fmt.Sprintf("Bearer %s", engine.ApiKey),
        "Content-Type":  "application/json",
    }

    resp, code, err := tools.HttpDoBytesWithTimeout(http.MethodPost, engine.Url, headers, body, 0)
    if err != nil || code != http.StatusOK {
        logs.WithContext(ctx).Errorf("call my-llm api failed. http_code:%d,err:%v", code, err)
        return ""
    }

    // 简单示例：假设直接返回纯文本
    return string(resp)
}

func (engine *MyLLMBotEngine) StreamChat(ctx context.Context, senderId, targetId string, channelType pbobjs.ChannelType, question string, f func(part string, sectionStart, sectionEnd, isEnd bool)) {
    // 如果你的接口暂不支持流式，可以简单地复用 Chat：
    answer := engine.Chat(ctx, senderId, targetId, channelType, question)
    if answer != "" {
        f(answer, true, true, true)
    } else {
        f("", false, false, true)
    }
}
```

> 注意：上面示例直接使用了 `tools.ToJson`、`tools.HttpDoBytesWithTimeout`、`logs.WithContext` 等通用封装，保持与现有代码风格一致。

### 4.2 在配置或初始化处挂载你的 Engine

通常会有一个地方按照配置创建对应的 BotEngine（具体可在 `botservice.go` 或相关初始化代码中查找），你可以：

1. 在配置文件中增加一段：
   - 指定当前 appkey 的 bot 类型为 `"my-llm"`。
   - 配置 `api_key` / `url` 等。
2. 在代码中根据配置分支：
   - 当类型为 `"my-llm"` 时，创建 `&MyLLMBotEngine{ApiKey: ..., Url: ...}`。
   - 注册为当前应用的 `IBotEngine` 实例。

这样，当机器人会话被触发时，系统就会自动走到你的 `MyLLMBotEngine` 里。

---

## 5. 与独立 bot-connector 项目的关系

在 `README.md` 中提到：

- `bot-connector` 仓库：**机器人对接服务，用于打通 im-server 和 三方机器人**。

结合当前仓库的 `services/botmsg` 模块，可以理解为：

- `im-server` 负责 IM 核心：连接、消息、多端同步等。
- `services/botmsg` + 外部 `bot-connector` 可以：
  - 将消息从 IM 路由到多个不同的机器人源（LLM、规则引擎等）。
  - 在外部集中管理机器人编排、缓存、限流等。

如果你的 AI 业务比较复杂（多模型编排、多轮任务、工具调用等），推荐：

1. 在 `bot-connector` 中实现复杂逻辑，对外暴露一个统一 HTTP 接口。
2. 在本仓库使用一个简单的自定义 `IBotEngine`，仅负责把用户问题转发给 `bot-connector` 并接收回答。

---

## 6. 调试与排错建议

- **从 Message 日志开始看起**
  - 确认消息是否已被识别为“机器人消息”，并成功路由到 `botmsg`。

- **观察 botmsg 日志**
  - 重点看 `call xxx api failed` 相关日志（Dify / Coze / SiliconFlow / 自定义）。
  - 确认 HTTP 状态码、错误信息。

- **抓外部 HTTP 请求**
  - 如果方便，可以在本地/测试环境用代理（Charles/Fiddler）或网关日志查看请求/响应内容。

- **检查会话持久化（尤其是 Coze）**
  - 若发现每次提问都像“新对话”，可能是 converId 缓存或 DB 持久化存在问题。

通过理解本指南内容，你可以：

- 理解 `im-server` 内部与 AI 交互的完整路径。
- 快速对接现有的大模型平台（Dify / Coze / SiliconFlow 等）。
- 在不破坏现有架构的前提下，安全地扩展出你自己的 LLM 接入方案。

