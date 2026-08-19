---
name: system-design-interview-notes
description: "Use when designing scalable systems, preparing for system-design interviews, estimating capacity, or looking up architecture patterns. A practical knowledge base synthesized from System Design Interview notes covering 28 designs."
slug: system-design-interview-notes-nuaib
displayName: System Design Interview Notes
version: 3.0.2
homepage: https://github.com/znuaibot/system-design-interview-notes
summary: "从《System Design Interview》两卷笔记合成的可推理知识库，覆盖 28 个系统设计案例、21 个可复用架构模式、估算速查与 110+ 术语表。内置双重视角（从业者落地 + 面试话术），用于实际设计、面试训练与架构模式查询。"
license: MIT
---

<!-- argument-hint: [system, pattern, topic, or chapter number] -->

# System Design Interview Notes
**Source basis**: Notes based on *System Design Interview — An Insider's Guide*, Vol. 1 and 2 | **Chapters**: 28 | **Version**: 3.0.0

## How to Use This Skill
- **实际设计**：给出业务量、SLO、数据语义和约束；先读本页流程，再加载对应案例章节。
- **面试训练**：指定系统和时间限制；按“澄清→估算→高层设计→深挖→总结”模拟。
- **概念查询**：按 Topic Index 定位章节；比较算法时同时读取 `patterns.md` 和 `cheatsheet.md`。
- **章节学习**：指定 `chNN`，加载该章的 decision rules、worked example 和 checklist。

## Core Workflow
1. **Clarify**：写出用户、核心用例、API 边界、规模、延迟、可用性、一致性、持久性、安全与 out-of-scope。
2. **Estimate**：估平均/峰值 QPS、对象大小、保留期、存储、带宽、缓存和分区数量；每个数字标单位与假设。
3. **Model**：定义实体、主键、索引、状态机与权威数据源；说明哪些数据可重建。
4. **Draw flows**：分别画写路径、读路径和失败路径；先保持最少组件。
5. **Scale deliberately**：只针对已识别瓶颈引入缓存、异步、复制、分片、预计算或多地域。
6. **Define semantics**：明确顺序范围、交付保证、幂等键、冲突处理、超时与重试。
7. **Operate**：给出 SLI/SLO、容量告警、回压、降级、灾备、数据校验和迁移方案。
8. **Close**：总结关键权衡、容量上限、单点、替代方案和下一阶段。

## Core Frameworks & Mental Models
- **约束驱动设计**：组件不是答案。先用吞吐、延迟、数据语义、故障域和成本缩小解空间。
- **简单起步、证据演进**：单机→分离数据层→无状态横向扩展→缓存/CDN→队列→复制/分片。每次演进必须对应可测瓶颈。
- **Back-of-the-envelope**：数量级估算用于否决不可能方案和定位主导成本；始终写假设、单位、平均值与峰值。
- **权威状态与派生状态**：每份数据标注 source of truth。缓存、索引、物化视图和 feed 应能重建。
- **分区内语义**：顺序、事务和 exactly-once 往往只能在键、分区或账户内成立；不要暗示全局保证。
- **At-least-once + idempotency**：跨网络操作默认可能重复。用稳定幂等键、去重记录和状态机获得业务上的单次效果。
- **异步解耦**：队列吸收突发并隔离失败，但引入积压、重复、乱序和延迟；必须设计重试、死信与可观测性。
- **复制不是备份**：复制提高可用性，也会复制逻辑错误。恢复还需版本、快照、审计日志和定期演练。
- **热点优先于平均**：均匀分片不消除 celebrity、hot key、热门 cell 或热门 partition；单独识别和缓解热点。
- **故障路径是一等公民**：每个组件都回答超时、部分成功、重试、降级、恢复和数据校验。

## Fast Routing
> 表格中的 `chNN` 指 `chapters/` 目录下同名文件（如 `ch03` → `chapters/ch03-a-framework-for-system-design-interviews.md`），按编号用 Read 跟随下方 Chapter Index 的链接加载对应章节，再展开 `patterns.md` 与 `cheatsheet.md`。

| 任务 | 先读 | 再读 |
|---|---|---|
| 从零设计任意系统 | ch03 | ch02 + 对应案例 |
| 高吞吐数据平台 | ch19 | ch20, ch21 |
| 强正确性交易系统 | ch22 | ch26, ch27, ch28 |
| 实时社交/推送 | ch12 | ch10, ch11, ch17 |
| 地理位置系统 | ch16 | ch17, ch18 |
| 大对象与媒体 | ch24 | ch14, ch15 |
| 分布式存储 | ch06 | ch05, ch19, ch24 |

## Chapter Index
| # | Title | Key Decisions |
|---|---|---|
| [01](chapters/ch01-scale-from-zero-to-millions-of-users.md) | Scale from Zero to Millions of Users | 读多先加缓存与 CDN；写或容量成为瓶颈时再扩数据库, Web 层保持无状态，状态放入共享数据层 |
| [02](chapters/ch02-back-of-the-envelope-estimation.md) | Back-of-the-Envelope Estimation | 先估 QPS、峰值 QPS、存储、带宽和内存, 所有数字写单位，先取整再计算 |
| [03](chapters/ch03-a-framework-for-system-design-interviews.md) | A Framework for System Design Interviews | 先问功能、规模、延迟、一致性和可用性目标, 先画端到端数据流，再选择 1–2 个关键点深入 |
| [04](chapters/ch04-design-a-rate-limiter.md) | Design a Rate Limiter | Token Bucket 适合允许受控突发, Leaky Bucket 适合平滑输出 |
| [05](chapters/ch05-design-consistent-hashing.md) | Design Consistent Hashing | 节点增删频繁时用哈希环减少重映射, 用足够虚拟节点平滑分布 |
| [06](chapters/ch06-design-a-key-value-store.md) | Design a Key-Value Store | 用一致性哈希分区并跨故障域复制, 用 N/R/W 配置读写一致性等级 |
| [07](chapters/ch07-design-a-unique-id-generator-in-distributed-systems.md) | Design a Unique ID Generator in Distributed Systems | 只需唯一且不排序可用 UUID, 需要趋势递增和高吞吐可用 Snowflake |
| [08](chapters/ch08-design-a-url-shortener.md) | Design a URL Shortener | 301 适合永久跳转和客户端缓存，302 便于统计与动态控制, Base62 编码 ID 可避免随机哈希冲突 |
| [09](chapters/ch09-design-a-web-crawler.md) | Design a Web Crawler | 按主机分队列控制抓取频率, URL 与内容分别去重 |
| [10](chapters/ch10-design-a-notification-system.md) | Design a Notification System | 每个通道独立队列与 worker，隔离故障, 发送前检查退订、频控和设备令牌 |
| [11](chapters/ch11-design-a-news-feed-system.md) | Design a News Feed System | 普通用户发布时预计算收件箱, 名人用户内容读取时合并，避免写放大 |
| [12](chapters/ch12-design-a-chat-system.md) | Design a Chat System | WebSocket 适合双向实时通信, 每会话或分片内定义消息顺序，而非全局顺序 |
| [13](chapters/ch13-design-a-search-autocomplete-system.md) | Design a Search Autocomplete System | Cached Trie 节点直存 Top-K 保 <100ms, 写路径离线化：logs→聚合→周期重建 trie |
| [14](chapters/ch14-design-youtube.md) | Design YouTube | 上传原始文件后异步转码多个 codec/分辨率, 用 DAG 调度可并行和有依赖的任务 |
| [15](chapters/ch15-design-google-drive.md) | Design Google Drive | 大文件分块并支持断点续传, 元数据与 blob 分开存储 |
| [16](chapters/ch16-proximity-service.md) | Proximity Service | 静态或中等更新 POI 可用 geohash 前缀索引, 密度高度不均时考虑 Quadtree |
| [17](chapters/ch17-nearby-friends.md) | Nearby Friends | 位置放带 TTL 的内存存储, 按用户关系建立 pub/sub 传播路径 |
| [18](chapters/ch18-google-maps.md) | Google Maps | 地图渲染使用按 zoom 分层的 tiles 并通过 CDN 分发, 道路图按 routing tile 分区 |
| [19](chapters/ch19-distributed-message-queue.md) | Distributed Message Queue | WAL 分段顺序写 + 三处 batching 扛吞吐, ISR + ACK 等级调节持久性与延迟 |
| [20](chapters/ch20-metrics-monitoring-and-alerting-system.md) | Metrics Monitoring and Alerting System | 采集端先做缓冲和可控聚合, 时间序列按 metric+labels 标识并按时间分区 |
| [21](chapters/ch21-ad-click-event-aggregation.md) | Ad Click Event Aggregation | event time + watermark + 日终对账保计费准确, Kappa 单路径 + 原始数据支持重算 |
| [22](chapters/ch22-hotel-reservation-system.md) | Hotel Reservation System | 库存表按 hotel/room_type/date 建模, 预订时以条件更新或锁原子扣减库存 |
| [23](chapters/ch23-distributed-email-service.md) | Distributed Email Service | 发送请求先持久化再异步投递, 按目标域限速并对临时错误退避 |
| [24](chapters/ch24-s3-like-object-storage.md) | S3-like Object Storage | 大对象使用 multipart upload 并行与续传, 数据分片跨故障域复制或纠删码 |
| [25](chapters/ch25-real-time-gaming-leaderboard.md) | Real-time Gaming Leaderboard | Redis sorted set 全操作 O(logN) 是默认答案, 分数只接受游戏服务器上报防作弊 |
| [26](chapters/ch26-payment-system.md) | Payment System | 所有创建支付请求携带 idempotency key, 资金变化进入 double-entry ledger，借贷恒等 |
| [27](chapters/ch27-digital-wallet.md) | Digital Wallet | 单数据库优先本地事务与双式账本, 跨分片可用 Try-Confirm/Cancel 预留资金 |
| [28](chapters/ch28-stock-exchange.md) | Stock Exchange | sequencer 全局定序保证确定性撮合与重放, 单机 mmap 事件总线 + 绑核循环压微秒延迟 |

## Topic Index
- **API 与范围** → ch03, ch08, ch16, ch22, ch25, ch26, ch28
- **CAP 与一致性** → ch06, ch19, ch21, ch22, ch26, ch27
- **CDN 与静态分发** → ch01, ch14, ch18
- **Event Sourcing** → ch27, ch28
- **Exactly-once 与幂等** → ch19, ch21, ch22, ch26, ch27
- **Geohash 与空间索引** → ch16, ch17, ch18
- **分区与一致性哈希** → ch05, ch06, ch19, ch20
- **复制与故障恢复** → ch01, ch06, ch19, ch24, ch28
- **实时连接与推送** → ch10, ch12, ch17
- **容量估算** → ch02 及所有案例章节
- **对象存储与分块** → ch14, ch15, ch24
- **异步队列** → ch01, ch09, ch10, ch14, ch19
- **排序与排行榜** → ch13, ch25, ch28
- **流处理与窗口** → ch20, ch21
- **缓存** → ch01, ch08, ch11, ch13, ch16
- **账本与对账** → ch26, ch27
- **限流与背压** → ch04, ch09, ch10, ch19

## Supporting Files
- [glossary.md](glossary.md) — 关键术语与章节导航
- [patterns.md](patterns.md) — 可复用架构模式、使用条件与权衡
- [cheatsheet.md](cheatsheet.md) — 一页式决策规则、估算与面试流程

## Scope & Limits
这是从公开笔记合成的学习和推理工具，不是原书替代品。容量数字是练习基线而非生产承诺；真实系统还需结合业务、法规、团队能力、云服务限制和项目代码验证。支付、钱包、交易所等章节只讨论架构，不构成金融或合规建议。
