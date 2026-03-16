## 项目学习指南（im-server）

本指南帮助你系统性地阅读和学习 `im-server` 这个即时通讯服务端项目，从整体架构到各个业务模块，循序渐进。

---

## 1. 入门准备

- **1.1 环境准备**
  - 安装 Go（版本参考 `go.mod` 中的 `go` 字段）。
  - 安装并启动依赖服务：MySQL、Redis、MongoDB（如果有）、本地或测试对象存储（如 MinIO）。
  - 根据 `commons/configures` 下的配置示例文件，准备本地配置（如 `config.yaml` / `config.local.yaml` 等，具体以项目实际为准）。

- **1.2 快速跑通**
  - 找到项目入口（`main.go` 或 `cmd` 目录下的启动文件），了解如何启动各个服务。
  - 本地起服务后，用简单方式验证：
    - 访问 API 网关根路径 `/` 是否返回 `"ok-ok"`（见 `services/apigateway/starter.go`）。
    - 如果有 WebSocket 接入，确认能建立连接并收发简单心跳消息。

---

## 2. 整体架构概览

- **2.1 模块划分**
  - `services/apigateway`：对外 HTTP 接口，处理登录、发送消息、查询会话等 REST 请求。
  - `services/connectmanager`：长连接与 WebSocket 管理，解析自定义协议，维护在线连接。
  - `services/message`：核心消息服务，负责消息发送、存储、屏蔽用户、已读状态等。
  - `services/historymsg`：历史消息服务，负责查询、归档、合并、收藏等历史记录。
  - `services/conversation`：会话列表服务，维护最近会话、未读数、置顶、@ 信息等。
  - `services/group`：群聊服务，处理建群、加群、退群、群设置、群成员管理等。
  - `services/pushmanager`：推送服务，封装 APNs、极光、厂商推送，将离线用户消息推送到终端。
  - `services/usermanager`：用户管理服务，维护用户资料、状态、在线/离线信息等。
  - `services/rtcroom`：实时音视频相关房间管理（结合 LiveKit / Agora / Zego 等 SDK）。
  - `services/sensitivemanager`：敏感词过滤服务，对消息内容进行检测与处理。
  - `services/fileplugin`：文件、日志等上传服务，封装 OSS / MinIO / 七牛 / S3 等对象存储。
  - `services/admingateway`：管理后台网关，提供运营 / 运维相关的管理 API。

- **2.2 公共基础库**
  - `commons/gmicro`：内部微服务 / Actor 系统框架，封装服务生命周期、Actor 模型等。
  - `commons/bases`：通用 Option / Context 封装，例如 `BaseActorOption` 等。
  - `commons/configures`：配置加载与全局配置结构体。
  - `commons/tools`：通用工具，例如 `UUID` 生成、加解密、时间工具等。
  - `commons/pbdefines`：协议相关的 protobuf 定义与生成代码。

- **2.3 关键依赖（从 go.mod 推断）**
  - Web 框架：`gin-gonic/gin`
  - ORM & 数据库：`gorm`、`go-sql-driver/mysql`
  - 存储：MySQL、MongoDB、LevelDB、HBase 等（不同模块使用）。
  - 消息推送：APNs (`apns2`)、极光、厂商 SDK（华为、小米、Vivo 等）。
  - 对象存储：阿里云 OSS、七牛、MinIO、AWS S3。
  - 实时音视频：LiveKit、Agora、Zego 等协议 SDK。

---

## 3. 推荐学习路径（由浅入深）

- **阶段一：整体运行 & 配置**
  1. 阅读 `go.mod`，了解主要依赖和项目类型（IM 服务端）。
  2. 阅读 `commons/configures` 目录，弄清楚：
     - 配置文件结构（数据库、Redis、各服务端口等）。
     - `configures.Config` 的主要字段含义。
  3. 找到 `main.go` 或启动脚本，理解如何启动多个服务（API 网关、连接管理、消息服务等）。

- **阶段二：对外接口（API 层）**
  1. 阅读 `services/apigateway/starter.go`：
     - 了解如何初始化 `gin.Engine`。
     - 看 `routers.Route` 如何注册路由。
  2. 阅读 `services/apigateway/routers` 下的路由定义：
     - 列出系统对外开放的 HTTP 接口（发送消息、拉历史、会话列表等）。
  3. 对每个核心 API，追踪到对应的 `service` / `actor` 调用路径。

- **阶段三：连接层（WebSocket / 长连接）**
  1. 阅读 `services/connectmanager/server/imwebsocketmsghandler.go`：
     - 理解 `IMWebsocketMsgHandler` 如何根据 `Cmd` 分发消息（Connect / Disconnect / Ping / Publish / Query 等）。
     - 了解 `ImListener` 接口的职责。
  2. 阅读 `services/connectmanager/server/codec`：
     - 理解自定义 WebSocket 协议格式、消息体结构。
  3. 阅读 `imcontext` 相关代码：
     - 看如何检查连接状态、如何在 context 中存储用户信息。

- **阶段四：核心业务（消息 + 会话 + 历史）**
  1. `services/message`
     - 关注发送流程：从 API / WebSocket 输入到消息落库、推送、会话更新的完整链路。
     - 阅读消息存储 DAO，理解数据库表结构设计。
  2. `services/conversation`
     - 理解会话是如何维护的（最近联系人、未读数、置顶、@ 消息等）。
  3. `services/historymsg`
     - 看历史消息的查询接口、分页策略、归档/合并逻辑。

- **阶段五：高级功能模块**
  1. `services/group`：群聊创建、成员管理、群设置。
  2. `services/pushmanager`：离线推送、各平台推送通道适配。
  3. `services/sensitivemanager`：敏感词过滤算法实现（例如 trie 树等）。
  4. `services/rtcroom`：音视频房间创建/加入/挂断的业务流程。
  5. `services/fileplugin`：文件上传、日志上报的存储适配。

---

## 4. 阅读代码时的建议方法

- **4.1 从“入口函数”向内追踪**
  - 先找出 HTTP 路由或 WebSocket 处理函数作为起点。
  - 再往下追踪到 service 层、DAO 层、存储层，画出简单的调用链。

- **4.2 善用日志和错误码**
  - 观察各模块的日志与错误码（例如 `commons/errs`），
  - 通过日志信息反推出关键流程和边界条件。

- **4.3 对照数据库表结构**
  - 找到 SQL 脚本（如 `docs/jim.sql`）和 DAO 层代码一起看，
  - 理解每张表在整个 IM 业务中的作用（消息表、会话表、群表、用户表等）。

---

## 5. 实战练习建议

- **练习 1：写一个“发送消息”的完整流程图**
  - 选择一个发送消息的 API 或 WebSocket 命令，
  - 从入口（路由 / handler）一路跟到数据库写入、会话更新、推送发送，
  - 用流程图或时序图画出来，加深对架构的理解。

- **练习 2：本地实现一个简单命令行客户端**
  - 使用 Go / 任意语言，实现一个简单的 CLI：
    - 通过 HTTP 登录获取 token（如果有）。
    - 通过 WebSocket 连接 IM 服务。
    - 发送一条文本消息给某个用户或群，观察服务器的处理逻辑。

- **练习 3：添加一个简单的扩展字段**
  - 在消息或会话上增加一个简单扩展字段（例如“是否重要”、“客户端标签”），
  - 贯穿 API 入参、内部模型、数据库字段、返回结果，
  - 体验一次“小需求”的全链路改动。

- **练习 4：阅读并修改敏感词逻辑**
  - 理解敏感词过滤的实现方式，
  - 尝试添加/修改一些敏感词规则或过滤策略。

---

## 6. 后续可以深入的方向

- **性能与扩展性**
  - 了解如何横向扩展连接管理服务、消息服务，如何做分库分表或分片。
  - 研究 `go-zero`、内部 `gmicro` / Actor 模型的使用方式。

- **可靠性与一致性**
  - 消息的去重、幂等、确认（Ack）机制。
  - 多端消息同步、一致性如何保证。

- **安全与风控**
  - 鉴权（token / JWT）、权限控制。
  - 防刷、防滥用策略（限流、黑名单等）。

你可以先按上面的“推荐学习路径”从 1～3 阶段开始，如果你希望，我可以根据你当前的水平，帮你制定一个更细的“每日学习计划”，或者陪你一起按这个大纲逐个模块阅读和讲解代码。

