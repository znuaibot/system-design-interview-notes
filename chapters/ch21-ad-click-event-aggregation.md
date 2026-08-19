# Chapter 21: Design Ad Click Event Aggregation

## Core Idea
广告点击聚合是典型流处理系统：Kafka 双队列 + MapReduce DAG 按分钟聚合，原始数据与聚合数据双存（Kappa 架构支持重算），计费用途要求 exactly-once、event time + watermark、日终对账。

## Frameworks Introduced
- **Lambda vs Kappa 架构**：Lambda = 批流双路径双代码库；Kappa = 单流处理路径，历史重算也走流引擎。
  - When to use：需要"实时聚合 + 出错后从原始数据重算"的系统。
  - How：本设计选 Kappa——重算服务从 raw storage 批读，送入专用聚合实例，结果进第二队列再更新聚合库，不影响实时链路。
- **MapReduce DAG**：map 节点（读源、清洗、按 ad_id 分发）→ aggregate 节点（内存按分钟计数/top-N 堆）→ reduce 节点（汇总终值）。
  - When to use：无界流的分布式聚合。
  - How：中间数据在内存，节点间 TCP/共享内存通信；map 层存在价值是清洗转换与不受上游分区方式约束。

## Key Concepts
- **规模基线**：10 亿点击/天、200 万广告、年增 30%；QPS 10,000、峰值 50,000；单事件 0.1KB → 日 100GB、月 3TB；端到端延迟可容忍几分钟（计费用），RTB 本身 <1 秒。
- **两个查询 API**：GET /v1/ads/{ad_id}/aggregated_count（from/to/filter）与 GET /v1/ads/popular_ads（count/window/filter）。
- **filter_id 维度表**：预定义过滤策略（region/IP/user_id 组合）编号，聚合结果按 (ad_id, click_minute, filter_id) 存 count。
- **Star Schema**：把过滤字段当 dimension 预聚合（ad_id, minute, country, count），查询快但桶数随维度膨胀。
- **双队列 exactly-once**：第一队列存原始点击事件，第二队列存聚合结果，用分布式事务原子提交 offset + 下游写入（或下游幂等则免事务）。
- **Event Time vs Processing Time**：异步链路下两者差异大；计费要准确 → 选 event time，代价是处理迟到事件（客户端时钟可能错或被伪造）。
- **Watermark**：聚合窗口刻意延长的部分，等迟到事件；短 watermark 低延迟易漏，长 watermark 少漏高延迟；漏网之鱼靠日终对账兜底。
- **窗口类型**：tumbling（固定，用于分钟计数）、hopping、sliding（用于"过去 M 分钟 top N"）、session。
- **热点广告处理**：聚合节点向 resource manager 申请资源，把事件拆给多个节点并行算再汇总；进阶有 Global-Local Aggregation、Split Distinct Aggregation。
- **快照容错**：聚合是内存态，节点挂即丢；每分钟对进行中聚合做快照，新节点从最新 consumer offset + 最新快照续算。
- **Reconciliation（日终对账）**：批任务从原始数据重算当日聚合，与聚合库比对。

## Decision Rules
1. 按 event time 聚合（计费准确性优先），用 watermark 容忍迟到
2. 按 ad_id 分区保证同广告事件落同一聚合节点
3. 计费数据必须 exactly-once：原子提交 offset+结果，或下游幂等
4. 原始数据冷存储保留（调试与重算），聚合数据热存快查
5. 过滤需求用 star schema 预聚合维度，别查询时现算
6. 分钟计数用 tumbling window，top-N 用 sliding window
7. 三组件（MQ/聚合服务/DB）解耦独立扩展；rebalance 放低峰

## Mental Models
- **原始数据是保险单**：聚合逻辑出 bug 时，raw data + 重算能力就是回滚方案——这是保留"看似冗余"数据的理由。
- **watermark 是延迟与准确的定价**：没有完美解，只有为迟到概率付多少延迟。
- **计费系统没有"差不多"**：小百分比误差 = 数百万美元差异，所以语义要 exactly-once、日终要对账。

## Anti-patterns
- **用 processing time 聚合**：网络与队列延迟让分钟边界错位，计费不准。
- **只做 at-least-once 不去重**：重复点击虚增账单。
- **聚合状态纯内存无快照**：节点故障丢整段中间态。
- **过滤维度查询时现算**：每次查询扫原始流，延迟不可控。
- **Lambda 双代码库无必要地引入**：维护成本翻倍，Kappa 能覆盖就选 Kappa。

## Worked Example
事件 ad001 点击流入：生产端写 Kafka 队列 1（ad_id, ts, user, ip, country）→ map 节点清洗并按 ad_id%3 分发 → aggregate 节点内存累计本分钟计数（top-N 维护堆）→ reduce 汇总，写第二队列 → 原子提交（offset + 聚合结果）→ Cassandra 聚合库 (ad_id, click_minute, filter_id, count)。迟到事件：watermark 延长窗口吸收大部分；仍漏的由日终 reconciliation 从 S3 原始数据重算比对修正。热点广告爆了：聚合节点向 resource manager 申请 3 个协处理节点，事件拆三份并行计数后回汇。重算：发现聚合 bug → 重算服务批读 raw storage → 专用聚合实例处理 → 第二队列 → 更新聚合库。

## Interview Lens
1. 开场澄清三大查询（ad 计数/top-N/过滤）与三个边界情况（迟到/重复/故障恢复）——问题清单本身就是得分。
2. 报数字：10 亿/天、50k 峰值 QPS、100GB/天。
3. event time + watermark + 对账三件套是时间语义的标准答案。
4. exactly-once 的原子提交推导（存 offset 的三种方式逐一排除）展示深度。
5. Lambda/Kappa 对比并说明选 Kappa 的理由。
6. 主动提替代方案（Hive+ES、ClickHouse/Druid OLAP），说明通才面试重在权衡而非工具内幕。

## Practitioner Checklist
- [ ] 双 API 与 filter_id 维度设计
- [ ] 原始+聚合双存储，raw 冷存
- [ ] event time + watermark 配置
- [ ] exactly-once 原子提交或下游幂等
- [ ] star schema 预聚合维度
- [ ] tumbling/sliding 窗口分工
- [ ] 热点拆分与资源申请机制
- [ ] 内存快照 + offset 容错恢复
- [ ] 日终对账与监控（延迟/积压/资源）

## Key Takeaways
1. Kappa 单路径 + 原始数据重算是计费级聚合的骨架
2. event time、watermark、对账解决时间三难
3. exactly-once = 原子提交 offset 与结果（或下游幂等）
4. star schema 预聚合换过滤查询速度
5. 三组件独立扩展 + 热点拆分支撑规模

## Connects To
- **Ch 19**：Kafka 双队列、offset 与交付语义。
- **Ch 20**：监控系统指标（队列积压 records-lag、节点资源）。
- **Ch 26**：支付场景同样要求幂等与对账。
- **patterns.md**：流式窗口聚合模式。
