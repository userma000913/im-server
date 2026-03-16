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
 
---

## 7. 1～2 周系统学习计划（面向 3 年 Go 后端）

> 说明：假设你工作日每天 2～3 小时投入，如果时间更紧/更宽裕，可以按天往后顺延或合并。

### 第 1 周：跑通全局 & 核心链路

- **第 1 天：项目启动与配置体系**
  - 阅读：`go.mod`、`commons/configures/configure.go`，理解整体依赖和配置结构。
  - 实操：
    - 准备本地配置文件（参考 `commons/configures` 以及 `docs/jim.sql` 中的数据库信息）。
    - 启动至少一个对外服务：`apigateway` + `connectmanager`（可通过 `main.go` / `cmd` 或各服务 `starter.go`）。
    - 用 curl/Postman 访问 API 网关根路径 `/`，确认返回 `"ok-ok"` 或类似健康检查结果。

- **第 2 天：API Gateway 结构与路由**
  - 阅读：
    - `services/apigateway/starter.go`：关注 gin 初始化、全局中间件。
    - `services/apigateway/routers/router.go`：整体路由结构。
    - 挑选 2～3 个典型接口阅读 `services/apigateway/apis/*.go`（如消息、历史消息相关）。
  - 输出：
    - 画一个简单“API 层结构图”：标出路由 -> handler -> 内部 service/actor 的调用关系。

- **第 3 天：ConnectManager 与 WebSocket 协议**
  - 阅读：
    - `services/connectmanager/starter.go`
    - `services/connectmanager/server/imwebsocketmsghandler.go`
    - `services/connectmanager/server/codec/*.go`（重点看消息结构、Cmd 枚举）。
  - 实操：
    - 用一个简单 WebSocket 客户端（可以是浏览器插件 / 小脚本）连接本地 `connectmanager`。
    - 手动发送 `Connect` / `Ping` / 简单 `Publish` 命令，观察服务端日志和响应。

- **第 4 天：消息服务 Message Service（发送路径）**
  - 阅读：
    - `services/message/services/msgservice.go`
    - 核心 `actors/` 中与发送相关的 actor（如 `addmsgactor`、`msgackactor`，具体以目录为准）。
    - `services/message/storages/dbs/*.go`，理解消息表/收件箱/发件箱结构。
  - 输出：
    - 选一个“发送消息”的入口（HTTP 或 WebSocket），从 handler/MsgHandler 一路追到 DB DAO，画出完整调用链和时序图。

- **第 5 天：会话 & 历史消息**
  - 阅读：
    - `services/conversation/services/conversationservice.go`、`mentionmsgservice.go`
    - `services/conversation/storages/dbs/*.go`
    - `services/historymsg/services/*.go`、`storages/models/hismsg.go`、`storages/mongodbs/*.go`
  - 思考：
    - 一条消息写入后，会话未读数是如何更新的？
    - 历史消息如何分表/分集合存储？查询接口支持哪些维度（按会话、时间等）。
  - 输出：
    - 画出“发送消息后，会话与历史如何联动更新”的流程图。

### 第 2 周：高级模块 + 性能 & 演练

- **第 6 天：群聊与用户管理**
  - 阅读：
    - `services/group/services/*.go`、`actors/*.go`，关注建群、加群、退群、踢人等流程。
    - `services/group/dbs/*.go` 与相关 models，了解群/成员表结构。
    - `services/usermanager/starter.go`、`actors/*.go`，理解注册、资料变更、免打扰等逻辑。
  - 输出：
    - 选一个“加群/退群”流程，写出从 API/WS 入口到 DB 的完整调用链说明（文字+简单时序图）。

- **第 7 天：推送与敏感词**
  - 阅读：
    - `services/pushmanager/services/*.go`、`storages/dbs/*.go`，理解推送配置、token 存储、推送通道抽象。
    - `services/sensitivemanager/sensitive/*.go` 与 `sensitivecall/*.go`，看敏感词过滤的核心实现。
  - 思考：
    - 当前项目如何区分在线/离线推送？哪些地方触发 PushManager？
    - 敏感词过滤在发送链路的哪个阶段被调用？失败/命中时如何反馈？

- **第 8 天：音视频房间与 FilePlugin**
  - 阅读：
    - `services/rtcroom/services/roomservice.go`、`actors/*.go`，对照依赖（LiveKit/Agora/Zego）理解房间生命周期。
    - `services/fileplugin/services/*.go`，看文件上传/客户端日志上报的处理路径。
  - 输出：
    - 写一段文字总结：IM 主业务和 RTC 房间的边界在哪里？哪些是强耦合，哪些是松耦合（例如只负责信令）？

- **第 9 天：存储层与分片思路**
  - 阅读：
    - `docs/jim.sql` + `commons/dbcommons/*.go`，理解全局 DB 管理与迁移。
    - `commons/mongocommons/mongomanager.go`、`commons/kvdbcommons/*.go`，熟悉 Mongo/LevelDB/HBase 封装。
  - 思考：
    - 哪些表/集合是典型热点？现在的 schema 是否已经为分库分表/分片预留了空间？
    - 结合 `architecture-design.md` 中的“横向扩展”章节，对比当前实现有哪些已经落地，哪些仍是思路级别。

- **第 10 天：综合小练习（推荐至少完成 1 个）**
  - 任选/组合以下练习（至少 1 个完整做完）：
    - **练习 A：按你理解的真实代码，重画一版“HTTP 发消息 + WebSocket 下发 + 离线推送”的全链路时序图**（以实际函数/actor 名称为准，而不是文档示例）。
    - **练习 B：增加一个简单扩展字段（例如消息的“importance”等级）**：
      - 从 API 请求结构 -> 内部模型 -> DB 字段 -> 返回结构，全链路过一遍（可以只在本地分支实现，不必提交）。
    - **练习 C：为敏感词模块加一个简单的“白名单/跳过逻辑”**，阅读现有过滤流程后，在合适位置插入判断。
  - 输出：
    - 为你完成的练习写一份 300～500 字的小结，说明你在这个项目中看到的“架构优点/潜在坑点”，加深理解。

你可以直接把这个学习计划当作 checklist，按天推进；如果某一天内容过多，可以把一项拆到第二天继续。后续如果你在某个阶段（比如 Message/Conversation）卡住，我也可以针对那几块再帮你写更细的“文件级”阅读顺序。

