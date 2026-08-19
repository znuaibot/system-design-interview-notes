# System Design Patterns

## Constraint-Driven Design
**When to use**：任何设计开始时。
**How**：先列功能、规模、SLO、一致性、持久性、合规和成本，再选最简单满足约束的方案。
**Trade-offs**：减少过度设计，但依赖高质量需求澄清。
**组合**：与 Back-of-the-envelope 估算（ch02）配套，约束里的规模数字必须估出来。

## Partition by Stable Business Key
**When to use**：数据或吞吐无法由单节点承担，且局部顺序/事务可按键定义。
**How**：选择高基数稳定键；估算热点；建立路由、再平衡和迁移机制。
**Trade-offs**：扩展吞吐，但跨分区查询和事务更复杂。
**组合**：MQ 分区保序（ch19）、hash(hotel_id) 分片（ch22）、user_id 分区（ch23）都是实例。

## Replicate Across Failure Domains
**When to use**：需要容忍机器、可用区或区域故障。
**How**：定义复制因子、确认规则、读修复、追赶和灾备演练。
**Trade-offs**：提高可用性与持久性，增加延迟、成本和一致性复杂度。
**组合**：副本对账用 Merkle tree（ch06）；持久性量化见三副本 vs 纠删码（ch24）。

## Cache-Aside with Rebuildable State
**When to use**：读多、可接受短暂陈旧且数据有权威来源。
**How**：读未命中回源并填充；写时删除/更新；设置 TTL、抖动和热点保护。
**Trade-offs**：降低延迟和数据库压力，但引入失效、击穿和陈旧数据。
**组合**：缓存键选 geohash 而非原始坐标（ch16）；CDC 异步同步缓存（ch22）。

## Durable Log + Materialized Views
**When to use**：需要审计、重放、多种派生视图或流式处理。
**How**：事件先写有序持久日志；消费者幂等构建索引、聚合或状态。
**Trade-offs**：可恢复且解耦，但处理重复、schema 演进和积压更困难。
**组合**：Kappa 架构（ch21）；可靠性只给不可再生的日志（ch27）。

## At-Least-Once + Idempotent Effect
**When to use**：跨网络调用、队列消费、支付或预订。
**How**：稳定幂等键；原子记录请求与结果；重试返回已有结果；副作用状态机化。
**Trade-offs**：比端到端 exactly-once 实际，但需要去重存储和保留策略。
**组合**：支付 exactly-once 拆解（ch26）；聚合计费去重（ch21）。

## Idempotency Key + Unique Constraint
**When to use**：用户可能重复提交、客户端可能重试的写接口（下单、支付、预订）。
**How**：客户端生成全局唯一键随请求携带；服务端对该列加 unique 约束；冲突时返回已有结果而非报错。
**Trade-offs**：实现简单可靠；但键的生命周期与清理策略要定义，前端防重只是辅助。
**组合**：reservationID（ch22）、payment_order_id 兼 PSP nonce（ch26）。

## Outbox / Transactional Event Publication
**When to use**：数据库更新与事件发布必须一致。
**How**：业务记录和 outbox 在同一事务写入；relay 重试发布；消费者幂等。
**Trade-offs**：避免双写丢失，但增加 relay 延迟和清理工作。
**组合**：TC/C 的 phase status table（ch27）是同类"状态表兜底"思想。

## Fanout Hybrid
**When to use**：社交 feed 或通知中用户度数差异巨大。
**How**：普通来源写扩散；高粉来源读时合并；查询去重与排序。
**Trade-offs**：缓解名人写放大，但读路径更复杂。
**组合**：feed 缓存只存 ID 列表（ch11）；presence 扇出对大 V 需另策（ch12）。

## Spatial Cell Candidate + Exact Filter
**When to use**：附近搜索、地图或位置服务。
**How**：按 geohash/S2/grid 找当前及邻接 cell，再用真实球面距离过滤。
**Trade-offs**：索引高效，但需处理边界、密度倾斜和精度层级。
**组合**：索引选型矩阵 geohash/quadtree/S2（ch16）；geohash 频道池（ch17）。

## Time-Windowed Aggregation
**When to use**：指标、点击、趋势统计。
**How**：定义 event time、window、watermark、状态 checkpoint、去重和迟到校正。
**Trade-offs**：低延迟结果可能暂时不完整，最终正确性需补算。
**组合**：日终 reconciliation 兜底（ch21）；快照容错（ch21）。

## Star Schema Pre-aggregation
**When to use**：聚合结果需按多个维度（地区/IP/用户）过滤查询。
**How**：把过滤字段定义为 dimension，按 (实体, 时间, 维度) 预聚合存储；查询直读。
**Trade-offs**：查询极快、实现简单；但维度组合多时桶数与记录数膨胀。
**组合**：与 filter_id 编号策略配套（ch21）。

## Double-Entry Ledger + Reconciliation
**When to use**：任何资金或价值转移。
**How**：借贷条目原子写入；余额是派生值；与外部 PSP/银行定期对账。
**Trade-offs**：可审计且可校验，但 schema 和纠错流程要求严格。
**组合**：资金预扣（ch28）、对账差异分级（ch26）。

## Snapshot + Deterministic Replay
**When to use**：事件溯源、撮合引擎和需要快速恢复的状态机。
**How**：输入有严格序号；处理确定性；周期 snapshot；从 snapshot 后重放。
**Trade-offs**：恢复与审计强，但事件兼容和确定性约束高。
**组合**：sequencer 定序（ch28）；状态机禁外部 IO（ch27）。

## Sequencer Global Ordering
**When to use**：多输入并发的处理结果必须确定、可重放、公平（撮合、账务）。
**How**：单写者给每个入站请求与出站结果打全局递增序号；处理器严格校验序号连续，乱序即拒。
**Trade-offs**：获得确定性与快速恢复；但定序点是吞吐瓶颈与单点，需热备。
**组合**：为延迟自研而非 Kafka（ch28）；与 snapshot+replay 配套。

## Offline Precompute + Online Lookup
**When to use**：写多但可延迟、读要求极快的查询（自动补全、地图 tile、行情 K 线）。
**How**：原始数据落日志 → 周期聚合/构建结构（trie/tile/图表）→ 在线侧纯内存查表；结构存缓存+持久层双份。
**Trade-offs**：读延迟极低、在线侧简单；但数据有更新延迟，重建管道成为关键路径。
**组合**：Cached Trie（ch13）、地图 tile CDN（ch18）、K 线 ring buffer（ch28）。

## TTL as State
**When to use**：需要表达"活跃/在线/临时存在"这类会自然消失的状态。
**How**：状态写入带 TTL 的缓存，周期性续期；过期即状态消失，无需显式下线协议。
**Trade-offs**：省去上下线协调，实现极简；但 TTL 阈值即业务语义，心跳间隔必须小于 TTL。
**组合**：附近好友活跃性（ch17）；缓存历史日自动清理（ch22）。

## Delta Transfer with Content Blocks
**When to use**：大文件/大对象的同步与存储，改动通常局部。
**How**：内容切成固定大小块哈希寻址；只传/存变化块；按元数据顺序重组；块级去重。
**Trade-offs**：带宽与存储大幅节省；但块索引与重组逻辑成为新的复杂度。
**组合**：Google Drive delta sync（ch15）、对象存储 WAL 合并与映射表（ch24）。

## Critical Path Subtraction
**When to use**：延迟敏感的链路（撮合、交易、实时游戏）。
**How**：识别关键路径，删非必要职责（日志/风控外移）、删网络跳数（单机/mmap）、删锁（绑核单循环）；非关键路径走派生流。
**Trade-offs**：微秒级延迟与延迟稳定性；但牺牲水平扩展与常规运维手段，需用热备补偿。
**组合**：交易所三流分离（ch28）；上传与播放双流（ch14）。

## Three-Tier Failure Handling
**When to use**：分布式存储与有状态服务的故障体系设计。
**How**：检测（gossip 多源心跳）→ 临时故障（sloppy quorum + hinted handoff 代存回送）→ 永久故障（Merkle tree 对账修复）。
**Trade-offs**：故障分级处理避免一刀切；但三层机制各有状态要维护。
**组合**：KV 存储完整实例（ch06）；broker 故障选主再分配（ch19）。

## Notification as Hint, Not Data
**When to use**：多端同步、变更提醒（文件变更、新消息、新邮件）。
**How**：推送只告知"有变化"，不携带内容；客户端收到提示后走权威路径拉元数据与数据；离线变更入队列上线补拉。
**Trade-offs**：消息小、乱序无害、通知丢失可自愈；但多一次拉取往返。
**组合**：Google Drive 同步（ch15）、邮件实时推送（ch23）、导航 WebSocket（ch18）。
