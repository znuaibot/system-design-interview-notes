# Chapter 6: Design a Key-Value Store

## Core Idea
分布式 KV 存储的设计 = 一致性哈希分区 + 沿环复制 + Quorum 一致性 + vector clock 冲突检测 + gossip/hinted handoff/Merkle tree 故障处理；先明确 CAP 取舍再定每个组件。

## Frameworks Introduced
- **CAP 取舍框架**：网络分区不可避免，所以实际只在 C 与 A 之间选——CP 分区时拒绝写（如银行），AP 分区时接受读写但可能读到旧值（最终一致）。
  - When to use：任何分布式存储设计的第一步。
  - How：先声明"分区发生时我保哪边"，再推导 quorum、复制和修复机制。
- **Quorum 一致性公式**：N 副本、W 写确认、R 读响应，W+R>N 保证强一致。
  - When to use：调节读写延迟与一致性的平衡。
  - How：读优化 R=1/W=N；写优化 W=1/R=N；均衡 N=3, W=R=2。

## Key Concepts
- **CAP Theorem**：一致性、可用性、分区容忍三者最多取二；分布式系统必须容忍分区，故 CA 系统不存在。
- **Data Partitioning**：用一致性哈希均匀分布数据；虚拟节点数与机器容量成正比，支持异构与自动扩缩容。
- **Data Replication**：从键的归属节点沿环顺时针取前 N 个节点存副本，副本应分散在不同数据中心。
- **Quorum Consensus**：W+R>N 时读写集合必有交集，保证读到最新写；W+R≤N 只有最终一致。
- **Consistency Models**：强一致（读必见最新写）、弱一致（可能读不到）、最终一致（足够时间后收敛）。
- **Vector Clock**：数据项关联 [server, version] 向量；写时递增本机计数或新增条目；全小于等于则是祖先版本，存在互有大小则是 sibling 冲突，交由应用逻辑或客户端解决；向量会增长需裁剪。
- **Gossip Protocol**：每节点维护成员心跳计数，周期性向随机节点发心跳；多个独立来源都报告心跳停滞才判定节点下线。
- **Sloppy Quorum**：故障时不强制原 quorum，改选环上前 W/R 个健康节点临时读写。
- **Hinted Handoff**：代存节点记录 hint，故障节点恢复后把暂存的写回送，恢复一致性。
- **Merkle Tree**：键空间分桶，逐桶哈希再逐层合成根哈希；两副本根哈希相同即一致，不同则递归比较子哈希定位差异桶，只同步不一致的数据。
- **SSTable**：Sorted String Table，内存缓存满后刷盘的有序数据文件（Cassandra 写路径）。
- **Bloom Filter**：读路径中快速判断键可能在哪个 SSTable，避免无谓磁盘扫描。
- **Coordinator**：客户端与存储之间的代理节点；系统完全去中心化，每个节点职责相同，无单点。

## Decision Rules
1. 用一致性哈希分区并跨故障域复制
2. 用 N/R/W 配置一致性等级：默认 N=3、W=R=2
3. 并发写冲突用 vector clock 检测，解决策略交应用层
4. 临时故障 → sloppy quorum + hinted handoff；永久故障 → Merkle tree 对账修复
5. 写路径 commit log → memtable → SSTable；读路径 cache → bloom filter → SSTable
6. 跨数据中心复制应对机房级故障

## Mental Models
- **CAP 是分区时的选择题**：平时三者都在，只有网络分裂的瞬间才被迫在 C/A 间站队。
- **复制只是起点，修复才是系统**：没有对账与回送机制的复制会随时间漂移成多个真相。
- **读路径是分层的排除法**：cache 排除热数据、bloom filter 排除不存在的文件，磁盘是最后手段。

## Anti-patterns
- **把 CAP 误解为任何时候只能三选二**：忽略了分区罕见，实际是分区发生时的取舍，平时可以同时提供较好的 C 和 A。
- **只复制不做修复**：副本漂移无人发现，故障切换后读到旧数据。
- **未定义冲突解决语义**：sibling 版本出现时无从合并，数据静默丢失。
- **单点 gossip 判活**：一个节点说对方挂了就摘除，网络抖动造成误判——至少要两个独立来源。

## Worked Example
N=3、W=2、R=2：写 put(k,v) 由 coordinator 转发给环上 3 个副本，2 个确认即返回成功；读 get(k) 询问 2 个副本取版本最大者。因 2+2>3，读写集合必有交集，读到最新值。节点 s2 临时宕机：写走 sloppy quorum 由 s4 代存并记 hint；s2 恢复后 s4 回送 hint 数据。s2 永久损坏：新节点加入后与邻居建 Merkle tree 对账，只拉取差异桶。vector clock 示例：s1 与 s2 并发改同一 key，产生 [s1:1] 与 [s2:1] 两个 sibling，由客户端合并后写回 [s1:2, s2:2]。

## Interview Lens
1. 开场报设计约束（<10KB 小 KV、高可用、自动扩缩容、可调一致性、低延迟），再进入架构。
2. CAP 必须讲"分区发生时"的取舍，并举 CP/AP 例子——这是必考点。
3. Quorum 公式 W+R>N 要能现场推导，并说出三种配置对应的优化方向。
4. 故障处理按"检测（gossip）→ 临时（sloppy quorum + hinted handoff）→ 永久（Merkle tree）"三层讲，结构感最强。
5. 被问存储引擎时讲 Cassandra 读写路径（commit log/SSTable/bloom filter）。

## Practitioner Checklist
- [ ] CAP 立场明确且与业务匹配
- [ ] N/W/R 数值有延迟与一致性论证
- [ ] 副本跨数据中心分布
- [ ] 冲突检测（vector clock）与解决策略都有
- [ ] 三层故障处理（检测/临时/永久）齐全
- [ ] 读写路径能说出每一层的作用

## Key Takeaways
1. 一致性哈希分区 + 沿环 N 副本是骨架
2. W+R>N 是强一致性的数学保证，配置即权衡
3. vector clock 检测冲突，hinted handoff 与 Merkle tree 分别处理临时与永久故障
4. 去中心化、无单点：coordinator 只是代理，任何节点可接管

## Connects To
- **Ch 5**：分区与虚拟节点的直接应用。
- **Ch 24**：对象存储的元数据层类似 KV 设计。
- **Ch 25**：leaderboard 用 Redis 类 KV 结构。
- **patterns.md**：复制、quorum、对账模式。
