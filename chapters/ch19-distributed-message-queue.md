# Chapter 19: Design a Distributed Message Queue

## Core Idea
支持长期保留、重复消费与顺序保证的消息队列 = topic/partition/broker 骨架 + WAL 分段顺序存储 + 批量读写 + 消费者组重平衡 + ISR 复制与可配 ACK；元数据与状态外置 ZooKeeper。

## Frameworks Introduced
- **point-to-point vs publish-subscribe 双模型**：传统 MQ 是点对点（消费即删、无保留），event streaming 是 pub-sub（按 topic 订阅、可重复消费）；本设计取后者并加保留与顺序。
  - When to use：开场界定"我们设计的是哪类系统"——严格说 Kafka/Pulsar 是 event streaming platform 而非传统 MQ，两者功能在趋同。
  - How：需要重复消费、两周保留、顺序保证 → 必须 pub-sub + 持久化日志模型。
- **三大性能设计选择**：① 利用现代 HDD 顺序写与 OS 页缓存的磁盘数据结构（WAL）② 消息结构不可变避免拷贝 ③ 一切围绕 batching（小 I/O 是高吞吐之敌）。
  - When to use：解释"为什么 MQ 能跑到百万级吞吐"的深挖。
  - How：生产者/队列/消费者三处都批量；顺序写 WAL 段文件；schema 三端一致零拷贝。

## Key Concepts
- **Topic / Partition / Broker**：topic 按 partition 分片扩展；broker 是承载 partition 的服务器；partition 内 FIFO 保序；消息位置 = topic + partition + offset 三元组定位。
- **Partition Key**：hash(key) % numPartitions 决定落区（如 user_id 保同一用户有序）；key 可重复、可缺失，与 KV 存储不同；生产者可覆盖默认键控制路由。
- **Consumer Group**：共同消费 topic 的消费者集合；消费进度按 group 而非个人维护各自 offset；组内并行提吞吐但破坏顺序——约束"一个 partition 同时只给组内一个消费者"，故组内消费者数 ≤ partition 数。
- **WAL（Write-Ahead Log）**：只追加的日志文件；partition 切成 segment 避免单文件过大，旧 segment 只读、只有最新 segment 接受写。
- **Offset**：partition 内消息位置；consumer 崩溃后新 consumer 从已提交 offset 续读。
- **Coordinator（组协调者）**：按 group 名哈希定位 broker，负责确认入组、分配 partition（round-robin/range 等策略）；注意与 ZooKeeper 协调服务不同。
- **Consumer Rebalancing**：成员加入/离开/心跳超时触发；coordinator 选组内 leader，leader 计算 partition 分配计划上报，coordinator 广播。
- **ISR（In-Sync Replicas）**：与 leader 保持同步（滞后 ≤ replica.lag.max.messages）的副本集合；消息在 ISR 全部同步后才 commit。
- **ACK 策略**：ACK=all（ISR 全同步，最慢最持久）/ ACK=1（leader 收到即确认，快但可能丢）/ ACK=0（不等确认，最快最易丢）。
- **State Storage**：partition→consumer 映射与 offset；读写频繁量小、随机访问、要求一致 → ZooKeeper 类 KV。
- **Metadata Storage**：partition 数、保留期、副本分布等配置；变化少、强一致 → ZooKeeper。
- **Routing Layer 内嵌生产者**：独立路由层多一跳且无法批量；嵌入 producer 后少跳数、可控分区、内存 buffer 批量发送。

## Decision Rules
1. 按业务键分区保持局部顺序（partition 内 FIFO，跨 partition 无序）
2. consumer group 内一个 partition 同一时刻只分配给一个消费者
3. 存储选 WAL 分段追加写，不选数据库（读写双重的系统数据库不擅长）
4. pull 模型优于 push：消费者控制速率、不被打垮、适合批量；空轮询用 long polling 缓解
5. ACK 与最小 ISR 数是持久性/延迟的配置旋钮
6. 路由逻辑嵌入 producer 换取少跳数与批量能力
7. 交付语义按场景配置：at-most-once（发了不重试、取后即提交）/ at-least-once（重试 + 处理后提交 offset，消费者需幂等）/ exactly-once（极贵）

## Mental Models
- **顺序是分区内的局部性质**：要全局顺序就要单 partition，吞吐归零——顺序与并发的兑换率就是 partition 数。
- **HDD 慢是误解**：慢在随机寻道，顺序读写能跑满带宽——WAL 正是把一切变成顺序。
- **commit 时机定义语义**：先提交 offset 再处理 = at-most-once；先处理再提交 = at-least-once——交付语义就是两行代码的顺序。

## Anti-patterns
- **push 模型不看消费能力**：生产快消费慢时消费者被压垮。
- **组内消费者数超过 partition 数**：多出来的消费者空转。
- **小消息逐条发**：网络往返与磁盘 I/O 开销摊不平，吞吐塌方。
- **副本全放同一节点/机房**：节点故障即全副本丢失，跨机房复制或用 data mirroring 兜底。
- **用"删消息"思维做保留**：传统 MQ 消费即删，无法支撑重放与审计。

## Worked Example
生产者发送：SDK 内嵌路由层从 metadata store 读副本计划并缓存 → 按 hash(user_id)%N 选 partition → 消息进内存 buffer 攒批 → 单请求发往该 partition 的 leader broker → leader 追加 WAL 当前 segment → follower 拉取，ISR 全同步（ACK=all）→ commit 并回复。消费者：订阅 topic A、加入 group 1 → 按 group 名哈希找到 coordinator broker → 确认入组并分到 partition 2 → 从 state storage 读上次 offset 拉取消息块 → 处理后提交 offset。partition 2 的 broker 宕机：ISR 中选新 leader，coordinator 把 partition 重分给现有副本，follower 追平后进 ISR。新消费者加入：coordinator 借心跳广播重平衡信号 → 重选组 leader → 新分配计划广播 → 各 consumer 按新计划续读。延迟消息（如 30 分钟后查支付结果）：消息先入 broker 临时存储（特殊 topic），time wheel 到点移入目标 partition。

## Interview Lens
1. 开场先辨析 MQ vs event streaming（Kafka/Pulsar 严格说是后者），再列三项增强需求（保留/重复消费/顺序）。
2. "为什么用 WAL 不用数据库"三连：写读双高、无更新删除、顺序访问 + OS 页缓存——标准深挖。
3. batching 要讲三处（producer/broker/consumer）与延迟-吞吐权衡曲线。
4. push vs pull 对比 + 选 pull 的理由是必考。
5. rebalancing 全流程（心跳→选 leader→分配计划→广播）讲出来就是 senior。
6. 交付语义用"offset 提交时机"一句话点破。

## Practitioner Checklist
- [ ] topic/partition/key 路由策略明确
- [ ] WAL 分段与保留期策略
- [ ] 三处 batching 配置与延迟权衡
- [ ] consumer group ≤ partition 数约束
- [ ] ISR、ACK、最小 ISR 配置
- [ ] rebalancing 流程与心跳超时
- [ ] 元数据/状态外置 ZooKeeper
- [ ] 交付语义可配置且消费者幂等
- [ ] 副本跨 broker/机房分布

## Key Takeaways
1. WAL 分段 + 顺序写 + OS 缓存是高吞吐存储的答案
2. partition 内有序、组内一分区一消费者、offset 按组维护
3. pull 模型 + long polling 是消费侧默认
4. ISR + ACK 等级是持久性与延迟的调节旋钮
5. 过滤（tag）、延迟/定时消息（time wheel）、重试 topic、历史归档（HDFS/S3）是进阶话题

## Connects To
- **Ch 1/10/11/14**：各章消息队列组件的完整实现。
- **Ch 6**：quorum 与 ISR 的一致性思想同源。
- **Ch 21**：Kafka 上的流式聚合消费。
- **patterns.md**：发布订阅与背压模式。
