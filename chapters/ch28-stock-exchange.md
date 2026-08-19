# Chapter 28: Design a Stock Exchange

## Core Idea
中小规模交易所 = 三流分离（trading 关键路径 / market data / reporting）+ sequencer 给入站订单与出站成交打全局序号保证确定性撮合 + O(1) FIFO 订单簿 + 全部塞进单机用 mmap 事件总线与绑核循环把延迟从毫秒压到微秒；只支持 limit order、下单前预扣资金。

## Frameworks Introduced
- **三流分治**：trading flow（关键路径，延迟极致优化）、market data flow（executions→K 线/订单簿重建→data service）、reporter flow（写库供税务/合规/清算，准确优先于延迟）。
  - When to use：交易所架构的骨架划分。
  - How：关键路径剥离一切非必需职责（连日志都去掉），非关键路径用常规手段。
- **Sequencer 确定性机制**：给每个 inbound order 与 outbound fill 打 sequence ID，撮合严格按序执行。
  - When to use：高可用与可重放的根基——三个理由：timeliness & fairness、fast recovery/replay、exactly-once。
  - How：概念上可用 Kafka（它就是进出消息队列），但为更低延迟自研实现；深挖形态是"单写者先定序再转发事件存储"。

## Key Concepts
- **规模基线**：100 symbol、10 亿订单/天、交易时段 6.5h → 43k QPS、峰值 215k（开盘显著更高）；99.99% 可用（日停 8.64s）、99 分位毫秒级往返。
- **Limit Order Only**：限价单按固定价买/卖，可部分成交或挂簿；本题不支持 market order、不做盘后交易。
- **Funds Withheld（资金预扣）**：下单前校验钱包余额充足，待成交订单占用的资金冻结至订单终结。
- **Risk Checks**：order manager 按 risk manager 规则检查（如单用户单日 AAPL 限 100 万股）。
- **Bid/Ask**：买一价（最高买）/卖一价（最低卖）；L1（最优价量）/L2（多档价位）/L3（档位排队量）三级行情。
- **FIX 协议**：券商与交易所间证券交易信息交换的行业标准协议。
- **Broker / Colo Engine**：券商中介散户；机构大单需拆单避免冲击市场；colo 是券商租用交易所机房服务器的 VIP 低延迟服务。
- **订单簿数据结构**：PriceLevel(limitPrice, totalVolume, orders) → Book(side, limitMap) → OrderBook(buyBook, sellBook, bestBid, bestOffer, orderMap)；用双向链表：挂单 O(1)（尾插）、撮合 O(1)（头删）、撤单 O(1)（orderMap 定位 + 前驱指针删除）；同价位 FIFO。
- **撮合流程**：校验 sequenceId 连续 → 验证订单 → 买单对 sellBook 从对手价逐笔吃单，生成双向 fills。
- **mmap 事件总线**：单机内进程间以 mmap 文件作事件存储通信；文件建在 /dev/shm（共享内存）则零磁盘访问。
- **Application Loop**：while 循环执行关键任务并绑同一 CPU——无上下文切换、无锁竞争。
- **订单簿组件库化**：各组件内嵌 order manager 库副本，避免跨进程调用。
- **Ring Buffer**：预分配、首尾相接、无锁的固定队列；padding 让 sequence number 独占 cache line；K 线内存限量、其余落盘。
- **Multicast + Reliable UDP**：行情同时送达所有订阅者（防先后差异被操纵市场）；UDP 不可靠需重传增强。
- **Leader Election / Raft**：有状态组件备份实例可处理入站事件但非 leader 不发布出站；心跳检测主挂；Raft 选主与事件复制。
- **功能确定性 vs 延迟确定性**：前者由 sequencer 保证（事件实际发生时间不重要，序才重要）；后者靠监控 99/99.99 分位，警惕 GC 等尖峰。

## Decision Rules
1. 只支持 limit order + 下单/撤单；资金先预扣后撮合
2. sequencer 自研而非 Kafka：关键路径延迟优先
3. 订单簿用双向链表 + orderMap，挂单/撮合/撤单全 O(1)，价位内 FIFO
4. 全链路单机化：mmap（/dev/shm）事件总线 + 绑核 application loop，目标微秒级
5. 行情用 multicast + reliable UDP 保公平同时送达
6. 高可用：关键服务热备 + 自动故障切换；leader 可收事件但非 leader 不发
7. 数据零丢失：Raft 复制事件日志 + 高频备份；RTO 与降级能力先定义
8. 风控/资金检查在 order manager，撮合引擎只收精简订单字段省带宽

## Mental Models
- **反直觉的单机架构**：现代交易所常把一切放一台巨型服务器——网络跳数与磁盘是延迟之敌，进程边界让位于内存总线。这与"分布式扩展"的常规范式相反，是本章最值得记住的点。
- **序即真相**：撮合结果由 sequence ID 决定而非物理时间——确定性、重放、公平全部建立在"序"上。
- **关键路径做减法**：优化不是加组件，而是删职责（连日志都删）、删跳数、删锁。

## Anti-patterns
- **微服务化撮合链路**：服务间网络往返直接把微秒目标打回毫秒。
- **用物理时间决定撮合顺序**：不可重放、不可审计、不公平。
- **订单簿用平衡树或数组**：撤单找不到 O(1) 路径，延迟不稳。
- **行情 unicast 逐个推送**：订阅者收到时间有先后，构成信息不公平甚至操纵空间。
- **无资金预扣**：挂单不占资金，成交时无钱可扣。

## Worked Example
下单流：散户经 Robinhood 类券商 → POST /v1/order（symbol/side/price/quantity，limit）→ client gateway 认证限流（轻量，关键路径）→ order manager 风控检查（日限 100 万股 AAPL）+ 钱包校验并预扣资金 → 精简字段送 sequencer 打 sequence ID → matching engine：校验序号连续 → 买单对 sellBook 对手价位 FIFO 逐笔撮合 → 生成买卖双向 fills 经 sequencer 出站 → 回传券商。行情流：fills → market data publisher 重建订单簿与 K 线（ring buffer 预分配）→ data service → multicast reliable UDP 同时送达订阅者（L1/L2/L3 分级收费）。报表流：reporter 收集 client_id/price/quantity/filled/remaining 写库，供税务合规清算。性能形态：全部组件单机，mmap（/dev/shm）事件总线互联，application loop 绑核运行；order manager 以库形式内嵌各组件。故障：matching engine 热备实例收流不发，心跳判主挂 → 自动 failover；事件日志 Raft 复制，重放恢复到任意点。

## Interview Lens
1. 开场报数字：100 symbol、10 亿单/天、43k QPS/215k 峰值、99.99%、99 分位毫秒。
2. 业务 101 先过：broker/bid-ask/L1-L3/FIX/limit vs market——领域词汇定第一印象。
3. 三流划分 + "trading 是关键路径"的延迟分级是架构主线。
4. sequencer 三理由（公平/重放/exactly-once）与"为什么不用 Kafka"必答。
5. 订单簿双向链表 + orderMap 的三操作复杂度随手推导。
6. 单机 + mmap + /dev/shm + 绑核循环是性能深挖的完整链条，主动点破"与现代分布式范式相反"。
7. 行情 multicast 公平性、colo、DDoS 加固（可缓存 URL、隔离公私服务）收尾。

## Practitioner Checklist
- [ ] limit order + 资金预扣 + 风控规则
- [ ] sequencer 全局序号与自研理由
- [ ] 订单簿 O(1) 三操作 + FIFO
- [ ] 单机 mmap 总线 + 绑核 loop
- [ ] 三流分离与延迟分级
- [ ] multicast reliable UDP 行情分发
- [ ] 热备 + leader election + Raft 复制
- [ ] RTO/数据零丢失/备份频率定义
- [ ] 功能与延迟确定性分别监控
- [ ] DDoS 与 KYC 安全项

## Key Takeaways
1. sequencer 定序 = 确定性撮合、快速重放、公平与 exactly-once 的共同根基
2. 订单簿双向链表 + orderMap：挂单/撮合/撤单全 O(1)
3. 单机 + mmap（/dev/shm）+ 绑核循环把延迟从毫秒压到微秒——反分布式直觉
4. 行情 multicast 保公平；三流分离让关键路径做减法

## Connects To
- **Ch 27**：event sourcing、Raft 与资金预扣在钱包侧的完整实现。
- **Ch 26**：支付结算与幂等机制。
- **Ch 25**：有序结构与极端低延迟存储选型。
- **Ch 19**：Kafka 作为 sequencer 概念原型的对照。
