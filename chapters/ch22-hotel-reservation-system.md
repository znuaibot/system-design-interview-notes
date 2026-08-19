# Chapter 22: Design a Hotel Reservation System

## Core Idea
酒店预订 = 关系库 ACID 保正确性 + 按 (hotel, room_type, date) 的库存表 + 幂等键防重复下单 + 乐观锁/约束防超卖并发；超订 10% 是业务规则，写进可用量公式。

## Frameworks Introduced
- **超卖防护三选一**：悲观锁（SELECT…FOR UPDATE，串行化但易死锁不可扩展）/ 乐观锁（version 列，低竞争快、高竞争回滚多）/ 数据库约束（CHECK total_inventory-total_reserved>=0，最易实现）。
  - When to use：库存并发扣减的防超卖设计。
  - How：本系统预订 TPS 低（~3/s），乐观锁或约束都合适；作者不推荐悲观锁（扩展性差）。
- **幂等键防重复下单**：reservationID 由客户端在填单时生成（全局唯一），reservation_id 列加 unique 约束，重复提交直接命中约束。
  - When to use：任何"用户可能点两次"的写接口。
  - How：客户端禁用按钮只是辅助（JS 可被关），幂等键才是服务端保障。

## Key Concepts
- **规模基线**：5000 酒店、100 万房间、70% 入住、平均住 3 天 → 日预订 ~240k → 平均 ~3 TPS；转化漏斗：1 预订≈10 详情页浏览≈100 列表页浏览。
- **room_type_inventory 表**：每行一个 (hotel_id, room_type_id, date)，含 total_inventory 与 total_reserved；CRON 每日预填未来两年；5000×20×2×365≈7300 万行，单库可扛。
- **超订公式**：total_reserved + N ≤ 110% × total_inventory 才允许预订。
- **room_type_rate 表**：房价按房型+日期存，每天变化，且随入住率浮动（rate service 独立）。
- **Idempotency Key**：reservationID，防同一用户重复提交。
- **Optimistic Locking**：version 列；读时记版本，写时 version+1 并要求新版本超过旧版本才成功；推荐用版本号而非时间戳（服务器时钟不准）。
- **CDC（Change Data Capture）**：数据库变更流式同步到缓存（如 Debezium→Redis），异步最终一致。
- **Saga**：跨服务长事务拆成本地事务 + 补偿动作，最终一致。
- **Two-Phase Commit**：跨节点原子提交协议，性能差（一节点慢全体阻塞）。
- **混合微服务策略**：reservation 与 inventory API 放同一服务共享关系库，借 ACID 保原子性，而非纯微服务每服务一库。

## Decision Rules
1. 库存表按 hotel/room_type/date 一行建模，查询与扣减都简单
2. 扣库存用乐观锁或 DB 约束，不用悲观锁（TPS 低、避免死锁与阻塞）
3. 幂等键（reservationID + unique 约束）防重复下单
4. 选关系库：读多写少、需 ACID 防负余额/重复扣款、数据结构清晰
5. 预订与库存放同一服务用关系事务保原子；跨服务才上 Saga/2PC
6. 库存缓存 Redis（key=hotel_roomType_date，TTL 过期历史日）+ CDC 异步同步；缓存与库短暂不一致可接受，DB 约束兜底
7. 扩容按 hash(hotel_id) 分片（查询必带 hotel_id）；历史预订移冷存储

## Mental Models
- **正确性靠数据库兜底，体验靠缓存加速**：缓存说"还有房"但库里没了——拒绝预订的是 DB 约束，用户刷新页面即可，这是可接受的不一致。
- **业务规则进公式**：超订 10% 不是补丁，是可用量判断的一部分。
- **架构纯度让位于一致性成本**：把强相关操作关进同一个事务边界，比追求"纯微服务"更务实。

## Anti-patterns
- **悲观锁扛预订并发**：长事务锁行阻塞所有后来者，旺季直接雪崩。
- **只做前端按钮置灰**：JS 被禁就失效，幂等必须服务端做。
- **非 serializable 隔离下先查后写**：两个事务都看到"还有 1 间"，双双提交成超卖。
- **预订与库存拆成两个服务两个库却无 Saga**：跨服务非原子操作产生不一致。

## Worked Example
用户订 hotel 211 房型 1001 两间（6/1–6/3）：POST /v1/reservations 带 roomTypeID、roomCount=2、reservationID=13422445 → reservation service 查 inventory 三天各 (total_inventory=100, total_reserved=80/82/86) → 每天校验 82+2≤110 ✓ → 事务内 UPDATE total_reserved+=2（version 乐观锁）→ 成功则 payment service 扣全款、更新预订状态；重复提交同 reservationID → unique 约束拒绝。并发：用户 1/2 同时订最后 1 间，事务 1 读到 version=5 提交成功使 version=6，事务 2 的 version=5 校验失败回滚。扩容场景：30,000 QPS 时按 hash(hotel_id)%16 分片，每片 1875 QPS 单 MySQL 集群可扛；库存读走 Redis，Debezium CDC 异步同步，过去日期 TTL 自动清。

## Interview Lens
1. 开场澄清五问（规模/付款时机/渠道/取消/超订），特别是超订 10%——这是业务理解得分点。
2. 报漏斗估算：3 TPS 预订对应 300 详情页浏览，解释"为什么写少读多"。
3. 库存表设计（每 hotel×type×date 一行 + CRON 预填 + 73M 行估算）是数据模型核心。
4. 防超卖三方案对比并给选择理由（TPS 低→乐观锁/约束），展示并发功底。
5. 微服务一致性讲"混合策略 + 为什么不纯拆"，比背 Saga 更有判断力。

## Practitioner Checklist
- [ ] 库存表三列建模 + 每日预填任务
- [ ] 超订系数进可用量公式
- [ ] reservationID 幂等键 + unique 约束
- [ ] 乐观锁 version 列或 CHECK 约束
- [ ] 关系库 ACID 覆盖预订+库存原子操作
- [ ] Redis 库存缓存 + CDC 同步 + TTL
- [ ] 分片键 hotel_id、冷存储归档策略
- [ ] 支付成功才确认预订的状态机

## Key Takeaways
1. (hotel, room_type, date) 库存表 + 超订公式是领域模型核心
2. 幂等键防重复下单，乐观锁/约束防并发超卖
3. 关系库 ACID 是预订系统的正确性底座
4. 缓存最终一致可接受，DB 约束是最后防线
5. 扩容 = hash(hotel_id) 分片 + Redis 缓存

## Connects To
- **Ch 26**：支付与幂等键的完整设计。
- **Ch 27**：Saga 与补偿事务在钱包系统的应用。
- **Ch 6**：乐观锁版本机制与 vector clock 思想同源。
- **patterns.md**：幂等与库存扣减模式。
