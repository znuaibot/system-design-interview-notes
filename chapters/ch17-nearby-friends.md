# Chapter 17: Design Nearby Friends

## Core Idea
附近好友 = 位置持续变化版的 proximity：WebSocket 有状态服务器接收位置更新，Redis 位置缓存（TTL 即活跃性）+ Redis pub/sub 按好友关系扇出；扩展靠分布式 Redis 集群 + 一致性哈希 + 服务发现。

## Frameworks Introduced
- **共享后端扇出模型**：P2P 对移动端不可行（连接不稳、耗电），改用后端收位置更新 → 找应接收的活跃好友 → 转发。
  - When to use：任何"一对多实时位置广播"。
  - How：每用户一个 pub/sub channel，WebSocket server 订阅自己连接的每个用户的好友 channel，收到更新后算距离再转发。
- **分布式 pub/sub 扩展法**：单机 Redis 撑不住 ~14M 更新/秒 → 分布式 Redis 集群 + zookeeper/etcd 记录 channel 分布 + 一致性哈希最小化迁移。
  - When to use：channel 数上亿、推送 QPS 超单机极限时。
  - How：channel→服务器映射存服务发现，WebSocket server 本地缓存哈希环数据；扩缩容选低峰期、灰度重订阅。

## Key Concepts
- **规模基线**：1B 用户、10% 用此功能、100M 日活、10% 并发（10M）、人均 400 好友、30 秒刷新 → 位置更新 QPS≈334k；每用户平均转发 40 个在线好友。
- **附近定义**：5 英里可配置；直线距离；10 分钟不活跃即消失；位置历史留存供 ML。
- **WebSocket Servers（有状态）**：持客户端长连接，收位置更新、算距离、转发；初始化时给客户端播种附近好友位置；下线需 drain 优雅关闭。
- **Redis Location Cache**：user_id→(lat, long, timestamp)，TTL 过期即视为不活跃自动移除——TTL 就是活跃性定义。
- **Redis Pub/Sub**：每用户一个 channel 的轻量消息总线；空闲 channel 不耗内存，可预分配所有功能用户的 channel。
- **Location History DB**：Cassandra 类写优化存储，存历史轨迹供分析，不参与在线查询。
- **Subscribe/Unsubscribe 语义**：好友上线时服务端通知客户端订阅其 channel，下线时退订。
- **cur 状态内存缓存**：WebSocket server 内存保存所连用户位置，避免每次距离计算都读 Redis。
- **Geohash channel 池（随机附近人扩展）**：按 geohash 建公共 channel，区域内用户订阅本格 + 邻格，收到随机陌生人位置。
- **Erlang 替代方案**：百万级轻量进程天然适合连接 + pub/sub，但人才稀缺。

## Decision Rules
1. 位置放带 TTL 的 Redis——TTL 即"活跃"的产品语义
2. 扇出走好友关系 pub/sub（每用户 channel），不做全局广播
3. 更新 QPS 超单机 → 分布式 Redis 集群 + 服务发现记 channel 分布
4. 集群扩缩容用一致性哈希减少 channel 迁移，选低峰期执行
5. 允许丢个别位置点（最终一致），换系统简单与低延迟
6. 好友数设上限（如 5000），保护"鲸鱼用户"所在服务器

## Mental Models
- **TTL 即状态机**：不需要显式的上线/下线协议，缓存过期就是下线——用存储特性表达业务语义。
- **扇出量 = 在线好友数**：容量规划的核心公式是"更新 QPS × 平均在线好友数"，不是用户数。
- **channel 是预分配的便宜资源**：空闲 channel 零内存，把"上线时建 channel"的复杂度提前消掉。

## Anti-patterns
- **P2P 直连广播位置**：移动网络不稳、耗电、NAT 穿透复杂。
- **全局位置广播**：与好友关系无关的用户也收到，扇出爆炸。
- **强一致位置存储**：为秒级延迟的位置数据付分布式事务成本，不值。
- **扩缩容高峰期做 channel 迁移**：大量重订阅 + 丢更新同时发生。

## Worked Example
用户 A（400 好友、10% 在线）每 30 秒上报：移动端 → LB → A 的 WebSocket server → 写 location history DB + 更新 Redis 位置缓存（TTL 10min）+ 内存缓存 → publish 到 A 的 channel → 订阅了 A 的 40 个在线好友所在的 WebSocket server 收到 → 各自算距离，≤5 英里的推给客户端带时间戳。客户端初始化：拉好友列表 → 订阅各 channel → 从缓存取位置播种地图。规模核算：10M 并发/30s≈334k 更新/秒，每次扇出 ~40 → ~14M 推送/秒，单机 Redis ~100k/s → 需 ~140 台 Redis；channel 内存 ~200GB 用 2 台 100GB 节点。

## Interview Lens
1. 开场先问清产品语义：附近多远（5mi 可配）、直线距离、不活跃阈值（10min）、历史留存——这些直接决定架构。
2. 报出 334k 更新/秒与 14M 推送/秒的推导，扩展论证全在数字里。
3. "为什么不用 P2P"与"TTL 即活跃性"是两个最亮的点。
4. Redis pub/sub 扩展深挖：140 台集群、服务发现、一致性哈希、低峰迁移——完整讲下来就是 senior 答案。
5. 主动给扩展题（随机附近人 → geohash channel 池、Erlang 替代）。

## Practitioner Checklist
- [ ] 附近半径、距离算法、不活跃阈值可配置
- [ ] 位置缓存 TTL = 活跃性定义
- [ ] pub/sub 按好友关系扇出
- [ ] WebSocket server 优雅 drain 下线
- [ ] 分布式 Redis + 服务发现 + 一致性哈希
- [ ] 扩缩容低峰执行、容忍少量丢更新
- [ ] 好友数上限与鲸鱼用户预案
- [ ] 位置历史与分析链路分离

## Key Takeaways
1. WebSocket + Redis（TTL 缓存 + pub/sub）是骨架
2. 扇出规模 = 更新 QPS × 在线好友数，决定需要 ~140 台 Redis
3. 一致性哈希 + 服务发现管理 channel 分布与迁移
4. 最终一致、允许丢点，是位置类系统的合理取舍

## Connects To
- **Ch 16**：静态 POI 的姊妹问题，geohash 在此用于随机附近人。
- **Ch 12**：WebSocket 有状态服务器与 service discovery 管理同源。
- **Ch 5**：一致性哈希减少 channel 迁移。
- **Ch 19**：pub/sub 消息系统的完整设计。
