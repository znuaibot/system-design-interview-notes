# Chapter 27: Design a Digital Wallet

## Core Idea
百万 TPS 钱包 = 分布式事务（2PC/TC-C/Saga 三选一）+ event sourcing（命令→事件→状态机，全历史可重放）+ 单机极致优化（mmap 本地盘 + RocksDB）+ Raft 复制保事件日志可靠 + 多 raft group 分片编排；唯一必须可靠的是事件列表。

## Frameworks Introduced
- **分布式事务三方案**：2PC（prepare→commit，锁竞争重、协调者单点）/ TC-C（try 预留资源→confirm/cancel 补偿，两段是独立事务、可并行）/ Saga（线性本地事务序列 + 失败补偿回滚，orchestration 优于 choreography）。
  - When to use：转账跨分片/跨服务时的原子性。
  - How：要低延迟选 TC/C（可并行），流程长、步骤多选 Saga orchestration；2PC 因性能与单点基本被排除。
- **Event Sourcing 四概念**：command（意图，可失败、可随机，FIFO 队列）→ event（已发生的事实，不可变，FIFO）→ state（事件作用的结果）→ state machine（验证命令、应用事件，必须确定性）。
  - When to use：需要审计、任意时点余额、代码变更后证明正确性的金融系统。
  - How：重放不可变事件 + 确定性状态机 = 任意历史状态可重建；CQRS 用多个只读状态机服务查询。

## Key Concepts
- **规模基线**：1M TPS、99.99% 可用、双式记账（每转账 = 2 条腿 → 实际 2M TPS）；云关系库单节点 ~1000 TPS → 需提升单节点能力（100→2 万节点，10,000→200 节点）。
- **唯一 API**：POST /v1/wallet/balance_transfer（from/to/amount-string/currency/transaction_id 幂等键）。
- **内存分片起点**：Redis map<user_id, balance> + hashCode 分区 + Zookeeper 存分区配置；可扩展但无法原子跨片转账——引出分布式事务。
- **TC/C 阶段表**：Try：A 扣 $1、C NOP；Confirm：A NOP、C 加 $1；Cancel：A 加 $1、C NOP；协调者挂掉靠 phase status table（事务内容/try 状态/二阶段名/二阶段状态/乱序标记）恢复。
- **先扣后加原则**：中间态允许短暂不平衡，但必须先扣款后加款——用户无法花费尚未存在的钱；"先加后扣"与"同时加减"都无效（跨库无 2PC 无法原子）。
- **Out-of-order 边界**：cancel 可能先于 try 到达 → phase status table 置乱序标记，后到的 try 直接失败。
- **Saga 两种协调**：choreography（事件订阅，逻辑分散）vs orchestration（单协调者指挥，钱包系统首选）；TC/C 与 Saga 的关键差异是 TC/C 可并行。
- **mmap 优化**：命令与事件存本地盘（append-only 对 HDD 友好）+ 内存缓存，mmap 一体实现。
- **RocksDB 存状态**：LSM tree 写优化、缓存补读；SQLite 是备选。
- **Snapshot**：周期快照存 HDFS 大二进制文件，重放不必从头开始。
- **可靠性推导**：state/snapshot 可由事件重放再生；事件不能由命令再生（命令非确定性）→ 只需保证事件列表可靠 → Raft 复制（leader/follower，过半存活即可用，保序保不丢）。
- **CQRS + 反向代理**：只读状态机响应查询；轮询伤实时性与负载 → 反向代理代用户发命令并代收响应，状态机就绪后推给代理，用户感知实时。
- **最终形态**：数据分片到多个 raft group，跨组转账由 Saga/TC-C 协调器编排。

## Decision Rules
1. 单分片内用本地事务；跨分片按延迟要求选 TC/C（并行低延迟）或 Saga（线性长流程）
2. try 阶段永远先扣后加，容忍短暂中间态但杜绝"花未来的钱"
3. 协调者状态进 phase status table，支持崩溃恢复与乱序检测
4. event sourcing：命令可丢（非确定性），事件必须 Raft 复制保可靠
5. 存储本地化：mmap 追加日志 + RocksDB（LSM）+ 周期快照
6. 查询走 CQRS 只读状态机 + 反向代理推送，不做用户轮询
7. 金额 string、transaction_id 幂等键

## Mental Models
- **可靠性只给不可再生的数据**：能重放出来的（state/snapshot）不值得复制，重放不出来的（事件）才值得 Raft——先推导什么必须可靠，再花钱。
- **确定性是重放的前提**：状态机禁外部 IO 与随机；命令允许随机所以不可重放——这条边界划分了整个可靠性设计。
- **演进式答案**：Redis→2PC→event sourcing→本地化→Raft→分片，每一步解决上一步的一个缺陷——这是本章的叙事结构本身。

## Anti-patterns
- **2PC 扛高 TPS**：锁竞争 + 协调者单点，性能与可用性双输。
- **try 阶段先给收款方加钱**：中间态窗口里钱被凭空花掉。
- **用命令列表重建事件**：命令有 IO/校验随机性，重放结果不一致。
- **事件存外部 Kafka 当最终方案**：网络往返吃掉 TPS，本地追加 + Raft 才是高吞吐答案。
- **choreography 编排钱包转账**：业务逻辑散在异步订阅里，故障追踪噩梦。

## Worked Example
A→C 转 $1（最终架构）：请求带 transaction_id 到 Saga 协调器 → 协调器写 phase status 记录、定位 A、C 所在分区 → 分区 1 raft leader 收命令 A-1 → 验证余额 → 生成扣款事件 → Raft 复制到组内节点 → 只读状态机同步后推响应给协调器 → 记录操作 1 成功 → 同流程执行 C+1 → 两步成功，通知客户端。故障演练：协调器中途挂 → 重启读 phase status table 续做；C 分区 try 失败 → 对 A 发 cancel（+1 补偿）；cancel 先于 try 到达 → 乱序标记拒绝后到 try。审计：查"A 在 3 月 1 日余额"→ 从最近快照重放事件到该时点；代码升级正确性 → 新旧版本状态机同放事件流比对结果。性能：命令/事件 mmap 追加本地盘、状态读写 RocksDB、周期快照到 HDFS；单 raft group 到顶后按账户分片多组。

## Interview Lens
1. 开场算账：1M TPS×2 腿=2M、单库 1000 TPS → 节点数表，引出"提升单节点 TPS"主线。
2. 分布式事务三方案对比表（尤其 2PC vs TC/C 的"一阶段是否已提交"差异）是核心考点。
3. 先扣后加 + 乱序标记两个细节展示对中间态的掌控。
4. event sourcing 四概念 + "为什么只需复制事件"的推导是全章最亮的推理。
5. 演进路线（Redis→…→分片 raft）按步讲，每步说清解决什么缺陷。
6. CQRS + 反向代理推送解决查询实时性，收尾完整。

## Practitioner Checklist
- [ ] balance_transfer 幂等键与金额 string
- [ ] 分布式事务选型（TC/C vs Saga）有延迟论证
- [ ] phase status table 含乱序标记
- [ ] 先扣后加顺序强制
- [ ] event sourcing 四组件齐全、状态机确定性
- [ ] 事件列表 Raft 复制；state/snapshot 不复制
- [ ] mmap 本地日志 + RocksDB + 快照到 HDFS
- [ ] CQRS 只读状态机 + 反向代理推送
- [ ] 多 raft group 分片与协调器编排

## Key Takeaways
1. TC/C 并行低延迟、Saga 线性易编排，2PC 基本出局
2. event sourcing 让余额任意时点可证、代码变更可验证
3. 可靠性只投给事件列表（Raft），其余可重放
4. 单机性能靠 mmap+RocksDB+快照，规模靠多 raft group 分片

## Connects To
- **Ch 26**：支付系统的 ledger/幂等/对账是钱包的上层应用。
- **Ch 22**：Saga 补偿在预订退款的实例。
- **Ch 28**：交易所的确定性撮合与事件日志同源。
- **Ch 6**：Raft/共识与 quorum 的一致性基础。
