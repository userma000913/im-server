## 技术架构设计说明（im-server）

本文档从技术视角概述 `im-server` 项目的整体架构设计，包括整体形态、服务划分、核心链路、数据与存储设计，以及横向扩展与可靠性思路，帮助你在阅读代码前先形成“全局脑图”。

---

## 1. 整体架构形态

- **系统定位**：面向多终端（移动 App / Web / 桌面）的即时通讯（IM）后端，支持单聊、群聊、历史消息、离线推送、多端同步，以及音视频房间等扩展能力。
- **架构风格**：多服务拆分 + 内部 Actor 模型：
  - 对外以 HTTP（REST）与 WebSocket 协议提供服务。
  - 服务内部通过自研/封装的 `gmicro` 与 Actor 系统进行解耦与消息分发。
- **主要接入协议**：
  - HTTP / HTTPS：登录、发送消息、管理会话、管理群、运营管理等。
  - WebSocket：IM 长连接、心跳、实时消息收发。

整体上可以抽象为四层：

1. **接入层（Gateway / Connect Manager）**
2. **业务服务层（Message / Conversation / Group / History / User / Push 等）**
3. **基础服务层（日志、配置、敏感词、文件存储等）**
4. **存储与基础设施层（MySQL / Redis / Mongo / HBase / 对象存储 / 消息中间件等）**

---

## 2. 服务划分与职责

### 2.1 接入层

- **API Gateway（`services/apigateway`）**
  - 使用 `gin-gonic/gin` 作为 Web 框架。
  - 职责：
    - 对外暴露 REST API：登录、发送消息、拉取会话和历史消息、设置用户/群属性等。
    - 进行参数校验、鉴权（例如 JWT）、基础限流与安全检查。
    - 将请求转发到内部业务服务（通过 `gmicro` / Actor / RPC 调用等）。

- **Connect Manager（`services/connectmanager`）**
  - 核心文件示例：`imwebsocketmsghandler.go`、`codec`、`imcontext` 等。
  - 职责：
    - 维护 WebSocket 连接：握手、鉴权、心跳（Ping/Pong）、断线重连。
    - 解析自定义二进制/文本协议（`codec` 层），根据 `Cmd` 决定业务操作：
      - `Connect` / `Disconnect`
      - `Ping`
      - `Publish` / `PublishAck`
      - `Query` / `QueryConfirm`
    - 将解析后的业务消息转交给内部监听器（`ImListener`），由业务服务处理。

### 2.2 业务服务层

- **Message Service（`services/message`）**
  - 职责：
    - 处理消息发送、转发、存储。
    - 管理消息状态（发送中、成功、失败）、已读 / 未读状态变更。
    - 处理屏蔽用户、消息撤回、消息删除等业务。
  - 与其他模块关系：
    - 与 `Conversation` 联动更新最新会话与未读计数。
    - 与 `HistoryMsg` 协作，保证历史可查询。
    - 与 `PushManager` 联动做离线消息推送。

- **Conversation Service（`services/conversation`）**
  - 职责：
    - 维护用户维度的会话列表（单聊 / 群聊）。
    - 会话置顶、会话免打扰、未读计数、@ 状态等。
    - 会话的创建、更新、删除（例如用户清理会话）。

- **History Message Service（`services/historymsg`）**
  - 职责：
    - 历史消息的持久化与查询（按用户、会话、时间等维度）。
    - 消息归档、合并（如长会话分段存储）。
    - 收藏消息、消息扩展信息（如 extra 字段）维护。

- **Group Service（`services/group`）**
  - 职责：
    - 群的创建/解散、群信息维护（名称、头像、公告等）。
    - 群成员增删改查，入群/退群逻辑（包括权限校验）。
    - 群管理功能：禁言、踢人、群设置（如仅管理员发言等）。

- **User Manager（`services/usermanager`）**
  - 职责：
    - 用户资料：昵称、头像、状态（在线/离线/隐身）等。
    - 用户关系：可能与好友关系管理、黑名单等模块协作。
    - 用户状态订阅：配合 `userstatussub` / `subscriptions` 服务广播上下线事件。

- **Push Manager（`services/pushmanager`）**
  - 职责：
    - 负责离线推送与通知：
      - 集成 Apple APNs（`apns2`）
      - 集成国内推送渠道（极光、华为、小米、Vivo 等 SDK）
    - 根据平台、设备 token 选择合适推送通道。
    - 支持多种推送策略（在线不推、仅离线推、重要消息强推等）。

- **RTC Room（`services/rtcroom`）**
  - 职责：
    - 管理音视频房间：创建、加入、挂断等。
    - 与 LiveKit / Agora / Zego 等第三方 SDK 协作，实现信令与房间管理。

- **Sensitive Manager（`services/sensitivemanager`）**
  - 职责：
    - 对消息文本进行敏感词检测与处理。
    - 可能使用 trie 树、正则或第三方接口做关键词过滤。

- **File Plugin（`services/fileplugin`）**
  - 职责：
    - 统一封装文件/日志等二进制数据的上传与访问。
    - 适配多种对象存储（阿里云 OSS、七牛、MinIO、S3 等）。

- **Admin Gateway（`services/admingateway`）**
  - 职责：
    - 提供运维/运营相关的管理接口（如封禁用户、手动推送、数据统计等）。

### 2.3 公共与基础服务

- **gmicro / Actor System（`commons/gmicro`, `commons/bases` 等）**
  - 通过 Actor 模型管理业务单元：
    - 每个 actor 负责一类业务上下文（如某个用户、某个会话、某个群）。
    - 通过消息投递而非直接方法调用实现解耦和并发控制。
  - 常见好处：
    - 简化并发问题（同一 actor 内串行处理）。
    - 便于水平扩展，将 actor 分布在不同节点上。

- **配置与工具（`commons/configures`, `commons/tools`）**
  - 配置加载：集中管理所有服务的端口、数据库、缓存、中间件配置。
  - 通用工具：UUID、加解密、时间、日志封装等。

---

## 3. 典型请求与消息流转

### 3.1 通过 HTTP 发送文本消息（示例链路）

1. **客户端**调用 API Gateway：
   - `POST /messages/send`（具体路径以实际代码为准）
   - 携带用户身份（如 JWT / token）与消息内容（接收方 ID、内容、扩展字段等）。

2. **API Gateway**：
   - 校验参数与鉴权。
   - 构造内部请求对象，调用 Message Service 对应的 actor / service。

3. **Message Service**：
   - 校验发送方/接收方状态（黑名单、禁言等）。
   - 写入消息存储（MySQL / Mongo / HBase 等）。
   - 更新 Conversation Service 中的会话记录与未读数。
   - 通知 HistoryMsg Service 进行历史记录管理（如异步归档）。
   - 通知 Push Manager 判断是否需要离线推送。

4. **Connect Manager**：
   - 如果接收方在线，Message Service 会通过内部通道下发实时消息到 Connect Manager，对应连接上的 WebSocket 会收到一条 `Publish` 类型的消息。

5. **客户端**：
   - 在线端实时收到消息（WebSocket）。
   - 离线端收到系统推送通知（APNs / 极光等）。

### 3.2 WebSocket 实时消息交互（简化）

1. 客户端通过 WebSocket 连接 Connect Manager，并发送 `Connect` 命令携带鉴权信息。
2. Connect Manager 校验并建立用户会话，将连接与用户 ID 绑定。
3. 客户端定期发送 `Ping`，服务端返回 `Pong` 或维护心跳逻辑。
4. 当客户端发送 `Publish`：
   - Connect Manager 解码、校验后，将消息转交 Message Service。
5. 当服务端要推送消息给客户端：
   - 由 Message Service / 其他服务通过内部通道将消息转发到 Connect Manager，由后者通过 WebSocket 向客户端发送 `Publish`。

---

## 4. 存储与数据模型设计（高层视角）

> 具体表结构可参考 `docs/jim.sql` 以及各模块下的 `storages/dbs`、`storages/models` 代码，这里只给出逻辑层级。

- **关系型数据库（MySQL / 兼容）**
  - 用户表：用户基础信息、状态字段。
  - 会话表：每个用户的会话列表、未读数、置顶/免打扰等。
  - 群表 + 群成员表：群元数据与成员关系。
  - 消息索引/映射表：用于快速按照会话+时间范围定位消息。

- **文档/大数据存储（MongoDB / HBase / LevelDB 等）**
  - 大量历史消息正文、扩展字段。
  - 广播消息、系统消息等特殊类型消息。

- **缓存（Redis 等）**
  - 在线状态、最近活跃会话、未读数缓存。
  - 分布式锁、频控计数器等。

- **对象存储（OSS / S3 / MinIO / 七牛）**
  - 图片、语音、文件、日志等二进制资源。

---

## 5. 横向扩展与高可用设计（思路层面）

- **接入层水平扩展**
  - API Gateway 与 Connect Manager 都可以多实例部署：
    - 前面通过负载均衡（Nginx / SLB / LB）分流。
    - WebSocket 连接按哈希或 IP 分配到不同节点。

- **业务服务横向拆分**
  - 基于用户 ID / 群 ID 按范围或哈希切分到不同服务实例或分片。
  - 使用 Actor 模型保证同一用户/会话在同一 actor 内串行处理，降低并发复杂度。

- **存储层扩展**
  - 消息和历史记录可按用户、会话、时间分片存放在不同库或不同集群。
  - 使用缓存减轻数据库的热点访问压力。

- **可靠性保证**
  - 消息发送流程中设计 Ack/重试机制（比如 WebSocket 的 `PublishAck`）。
  - 写入存储与推送动作解耦（例如通过异步任务 / 队列），避免单点阻塞。
  - 使用监控与日志系统（`services/logmanager` 等）收集指标与错误。

---

## 6. 如何结合架构阅读代码

阅读建议：

1. 先对照本架构文档，画出你自己的系统“方块图”（服务间关系图）。
2. 按照以下顺序阅读：
   - 接入层：`apigateway` + `connectmanager`
   - 消息核心：`message` + `conversation` + `historymsg`
   - 群与用户：`group` + `usermanager`
   - 推送与敏感词：`pushmanager` + `sensitivemanager`
3. 对每条典型链路（例如“HTTP 发消息”或 “WebSocket 收消息”），尝试在代码中找出完整调用链，确认与你在本文中的理解是否一致。

如果你想，我可以基于这份架构文档，下一步帮你画出一两条**完整时序图**（如“单聊消息发送+到达+已读”），再结合具体文件，一起深入分析实现细节。

