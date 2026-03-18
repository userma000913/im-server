## 多节点集群与路由设计（im-server）

本文档描述如何在现有 `im-server` 的 `gmicro/actorsystem` + `bases` 抽象之上，落地一个**可水平扩展的多节点集群**：包括节点注册发现、方法（method）分布、按 `targetId` 路由、广播/组播、以及 ConnectManager（长连接层）在多节点下如何把消息准确下发到“用户所在的那台机器”。

> 背景：当前开源仓库中 `commons/gmicro/cluster.go` 是**单节点简化实现**（`GetAllNodes()` 只返回 `currentNode`，`GetTargetNode()` 只判断本地是否注册 method）。本文档给出一套“保持接口不变、扩展实现”的多节点方案。

---

## 1. 术语与目标

- **Node**：一台服务实例（进程），具有 `name/ip/exts`，并注册一组 actor `method`。
- **Method**：Actor 的逻辑入口名（例如 `connect`、`brd_append` 等），用于路由到目标 actor。
- **targetId**：路由分片键（通常是 `userId`、`groupId`、`conversationId` 等）。
- **UnicastRoute**：把一个 RPC 投递到“负责该 targetId 的那台 Node”。
- **BroadcastRoute**：把一个 RPC 投递到所有（或一部分）Node。
- **GroupRpcCall**：把一批 targetIds 按 Node 分组后分别投递（减少跨节点次数）。

**目标：**

1. ConnectManager 可水平扩展：用户 WebSocket 连接分散到多个 connect 节点。
2. Message/Conversation/History 等服务可水平扩展：按 userId/groupId 等分片处理。
3. 任意服务都能通过 `bases.UnicastRoute / Broadcast / GroupRpcCall` 找到正确节点并投递。
4. 支持节点上下线、故障转移（尽可能平滑），并可观测。

---

## 2. 当前代码的“单节点现状”

在 `commons/gmicro/cluster.go` 中：

- `Cluster` 只保存 `currentNode *Node`。
- `GetAllNodes()` 返回 `[]*Node{currentNode}`。
- `GetTargetNode(method, targetId)` 只在 `currentNode.methodMap` 命中时返回本地，否则返回 nil。
- `BroadcastRoute` 的 `getNodeList(method)` 也只返回本地节点。

**这意味着：**

- 所有 `bases.UnicastRoute(...)` 实际都是本地投递到 `actorSystem.ActerOf(method)`。
- `bases.Broadcast(...)` 也是“广播给自己”。

该抽象很好，因为：**只要扩展 `Cluster` 的节点列表与路由选择逻辑，上层业务代码几乎不需要改动。**

---

## 3. 多节点落地：需要补齐的三个能力

### 3.1 节点注册发现（Node Registry）

要实现多节点，首先要让每个进程把自己“注册”出去，并能“发现”集群里其他节点。

#### 3.1.1 Node 元信息（建议字段）

现有 `Node` 已包含：

- `Name`：节点名（全局唯一）。
- `Ip`：节点 IP/域名。
- `Exts`：扩展信息（推荐至少包含对外端口）。
- `Methods`：该节点支持的 actor methods（由 `RegisterActor` 时 `AddMethod` 填充）。

建议在 Exts 里约定：

- connect 节点：`wsport`（对外 WebSocket 端口），可选 `httpport`。
- api/admin 节点：`httpport`。
- message/history 等内部节点：可选 `rpcport`（如果未来做跨进程 RPC 通道）。

#### 3.1.2 注册中心选择

可选方案（按复杂度递增）：

- **静态配置**：配置文件里写死 node 列表（适合小规模/测试）。
- **Redis**：用 `SETEX` 做心跳续租（简单但一致性较弱）。
- **etcd / ZooKeeper**：Watch + Lease（推荐，能正确感知上下线）。
- **K8s**：通过 Service/Endpoints API 感知实例（云原生）。

#### 3.1.3 心跳与租约

每个节点周期性刷新注册中心中的租约（例如 5s 一次，TTL 15s）：

- 当节点宕机/网络隔离时，租约过期，其他节点的 Watch 会收到“下线事件”。
- 本地 cluster 更新其 node 列表与 method 索引。

---

### 3.2 Method → Node 索引（能力发现）

在多节点下，路由选择不仅依赖 `targetId`，还依赖“哪些节点支持 method”。

建议在 cluster 内维护两个索引：

- **nodes**：`map[nodeName]*Node`（全量节点表）
- **methodNodes**：`map[method][]*Node`（某 method 可投递到哪些节点）

更新策略：

- 节点上线：加入 nodes，并将其 Methods 合并进 methodNodes。
- 节点下线：从 nodes 删除，并从 methodNodes 的各个列表移除。
- Methods 变化（滚动升级可能发生）：按节点为单位替换其 Methods 集合，再重建 methodNodes。

这样 `getNodeList(method)` 就能返回真实列表，而不只是本地节点。

---

### 3.3 目标路由（targetId → Node）策略

`bases.UnicastRoute(method, targetId, ...)` 需要一个确定性策略：

- 输入：`method` + `targetId` + `methodNodes[method]`
- 输出：一个目标节点（或者一个“备选节点列表”用于失败重试）

常见策略：

#### 3.3.1 简单 hash（取模）

- `idx := hash(targetId) % len(methodNodes[method])`
- 优点：实现简单、性能好。
- 缺点：节点数变化时大量 key 迁移（缓存命中下降，热迁移问题明显）。

#### 3.3.2 一致性哈希（推荐）

- 为每个节点在环上放多个虚拟节点（vnodes）。
- `targetId` hash 后在环上顺时针找第一个节点。
- 优点：扩缩容时迁移量小，稳定性好。
- 缺点：实现略复杂，需要维护 ring。

#### 3.3.3 Rendezvous Hash（HRW，推荐）

- 为每个节点计算一个得分：`score(node, targetId)`，取最高者。
- 优点：实现简单、不需要维护环；扩缩容迁移也相对小。
- 缺点：每次选择要遍历节点列表（节点很大时会有开销，但通常可接受）。

> 建议：节点规模 < 200 时，HRW 很实用；规模更大或极端追求性能时用一致性哈希环。

---

## 4. “跨节点投递”的通信方式（现实问题）

当前 `cluster.baseRoute(...)` 是本地 `actorSystem.ActerOf(method).Tell(...)`。

要做到跨进程/跨机器投递，必须补一层“传输”：

### 4.1 方式 A：直接 gRPC（最直观）

- 每个节点暴露一个 gRPC Server：`Route(method, rpcMessageWraper)`。
- `cluster.UnicastRoute`：
  - 若目标 node 是本地：走本地 actor Tell。
  - 若目标 node 是远端：用 gRPC 调远端的 Route，再由远端把消息投递到本地 actor。
- 优点：工程直观、易调试。
- 缺点：需要维护连接池、超时、重试、限流等。

### 4.2 方式 B：消息中间件（NATS/Kafka/Redis Stream）

- 将 `RpcMessageWraper` 写入某个 topic：
  - 例如 topic 名包含 method + shard。
- 每个 node 订阅自己负责的 shard，消费后投递到本地 actor。
- 优点：天然解耦、削峰填谷。
- 缺点：实时性/顺序性/幂等等需要更完整设计。

### 4.3 方式 C：保持“服务内多进程”的假集群（仅单机多实例）

如果部署形态是“同一台机器多进程”，可以先用本地 IPC（unix socket）做过渡，但不推荐长期使用。

> 无论选哪种，核心原则是：**Cluster 的路由决策层和传输层解耦**，以便未来替换通信实现。

---

## 5. ConnectManager 多节点下：如何找到“用户在哪台节点在线”

这部分是 IM 的关键点：群聊/单聊下发时，必须知道某个 `userId` 当前连到哪台 connect 节点。

### 5.1 单节点下的做法（现状）

`services/connectmanager/services/connectmanager.go` 里维护：

- `OnlineUserConnectMap`：`(appkey_userid) -> map[session]WsHandleContext`
- `OnlineSessionConnectMap`：`session -> WsHandleContext`

这两个 map 都是**本进程内存**，单节点下足够。

### 5.2 多节点下的问题

当 connect 有 N 台机器时：

- 某个 userId 的连接只存在于“它连接到的那台 connect 节点”的内存里。
- 其他节点不知道它在线与否，更不知道 session/ctx。

因此必须引入一个**全局在线目录（presence directory）**。

### 5.3 推荐方案：presence 目录 + 本地连接表

每个 connect 节点仍保留本地 `OnlineUserConnectMap`（用于真正写 WS）。

同时在“全局目录”里记录：

- Key：`appkey_userid`
- Value：该 user 当前在线在哪些 connect 节点、有哪些 session：
  - 例如：`[{nodeName, wsHost, wsPort, session, deviceId, platform, instanceId, lastSeen}]`

这个全局目录可以存在哪里：

- etcd（推荐）：
  - 用 lease 做在线心跳。
  - Watch 可用于订阅用户上下线事件。
- Redis：
  - Hash + TTL 也能做，但一致性与过期语义要小心。

### 5.4 下发时的两阶段路由

当 Message 服务要下发给某个 `userId` 时：

1. **查 presence 目录**：
   - 得到该 user 的在线 nodeName/session 列表。
2. **按 nodeName 分组**，对每个 nodeName 发起一次 RPC：
   - RPC 内容包含：`targetUserId(s)` + `payload`（或完整 `DownMsg`）。
3. **在目标 connect 节点本地**：
   - 根据 session/用户从本地 `OnlineUserConnectMap` 找到 `WsHandleContext`。
   - 写 WebSocket（`ctx.Write(...)`）。

这就是“多节点下仍可做到 per-user 下发”的关键：**连接始终只在本地写，跨节点只传递业务消息和目标用户标识。**

---

## 6. 群消息（万人群）在多节点下如何 fanout

群消息本质是：

- 计算目标集合：群成员列表（可能 10 万）。
- 对在线成员做实时 fanout，对离线成员做历史/推送。

多节点下的优化建议：

### 6.1 在线成员过滤：先按节点聚合再下发

不要在 Message 节点上“对每个 userId 都单独 RPC 一次”。

推荐：

1. 从 presence 目录取回在线用户的 `(userId -> nodeName)`（可批量）。
2. 按 nodeName 聚合成：`nodeName -> []userId`
3. 对每个 nodeName 做一次批量下发 RPC（或复用 `GroupRpcCall` 的思想）：
   - 把同一个下行 payload + userId 列表发给该 connect 节点。
4. connect 节点本地遍历 `[]userId`，为每个 user 的 session 写 WS。

这样跨节点 RPC 次数从 \(O(nOnline)\) 降到 \(O(nConnectNodes)\)。

### 6.2 消息体复用：避免重复序列化

同一条群消息下发给 1 万人，payload 相同：

- 远端 RPC 传 payload 一份 + userId 列表。
- connect 节点在本地构造 `ImWebsocketMsg` 时尽量复用已序列化的 payload（能做的话），减少 CPU 与 GC 压力。

### 6.3 背压与限流

大群 fanout 时一定要有：

- connect 节点写 WS 的并发上限（goroutine pool）。
- 单连接写超时与断开策略（当前 connect 已有 write deadline）。
- 对慢客户端的丢弃/踢下线策略（防止拖垮节点）。

---

## 7. Broadcast（全员广播）在多节点下的语义

广播消息（`bases.Broadcast`）的本质是“广播到所有节点”，典型用途：

- 系统公告、配置刷新、全局缓存失效通知。

多节点实现中：

- `BroadcastRoute(method, msg)` 应遍历 `methodNodes[method]`：
  - 对每个节点发起一次 Route RPC。
  - 节点收到后投递到本地 actor。

> 注意：这不是 WebSocket 协议广播，而是“集群内 RPC 广播”。

---

## 8. 故障与一致性：必须提前想清楚

### 8.1 节点下线

connect 节点下线时：

- presence 目录中的该节点 session 会因 lease 过期自动消失。
- Message 服务后续下发会发现 user 不在线（或在线列表变更）。
- 终端会重连到其他 connect 节点并重新 Connect。

### 8.2 路由变化（扩缩容）

当 methodNodes 发生变化时（加节点/减节点）：

- 采用一致性哈希/HRW 可减少迁移。
- 对“强绑定状态”的 actor（例如 user actor）：
  - 需要考虑状态迁移或把状态外置到 Redis/DB。

### 8.3 幂等与重试

跨节点 RPC 可能超时/失败：

- 对于“下发消息”这种动作：
  - 建议在客户端/服务端用 msgId + seq 做幂等。
  - 允许重试，避免丢消息。

---

## 9. 可观测性与排错（建议埋点）

多节点下排错要有“链路可追踪”：

- **路由层指标**
  - `route_unicast_total{method,node}`
  - `route_unicast_fail_total{method,reason}`
  - `route_broadcast_total{method}`
- **connect 下发指标**
  - 在线用户数、每节点连接数、每连接写失败率
  - 大群 fanout 的耗时、每批次耗时、丢弃/踢下线数量
- **presence 目录指标**
  - 在线 key 数量、watch 延迟、租约刷新失败率

---

## 10. 落地改造建议（最小可行路径）

如果你要在当前仓库上逐步落地多节点，建议按顺序推进：

1. **先做 Node Registry（静态配置/etcd）**
   - 把 `cluster.getNodeList(method)` 改成返回真实节点列表。
2. **补齐路由选择（HRW/一致性哈希）**
   - 完成 `GetTargetNode(method, targetId)` 的真实实现。
3. **加上跨节点 Route 传输（gRPC）**
   - 让 `cluster.UnicastRoute` 能真正把消息送到远端 node。
4. **引入 presence 目录**
   - connect 节点上线时写入目录，下线自动过期。
   - message 下发改成“按 nodeName 批量下发”。
5. **最后做大群 fanout 优化**
   - 批量、并发控制、背压与观测。

---

如果你希望我把这份文档进一步“可执行化”，我可以补充两份附录：

- **附录 A**：Route gRPC 的 proto 设计（method、targetId、RpcMessageWraper 的传输格式）
- **附录 B**：presence schema（key/value 结构、租约与 watch 事件设计）
