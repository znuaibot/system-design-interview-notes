# Chapter 20: Design a Metrics Monitoring and Alerting System

## Core Idea
监控系统五组件（采集/传输/存储/告警/可视化）；核心是时序数据库（InfluxDB/Prometheus 类）+ 分辨率分层降采样（7d 原始/30d 1min/1y 1h）+ Kafka 缓冲管道；采集 pull vs push 无绝对赢家。

## Frameworks Introduced
- **五组件骨架**：Data Collection → Transmission → Storage → Alerting → Visualization。
  - When to use：监控题的标准开场结构。
  - How：先问清范围（ infra 指标还是日志/trace？内部还是 SaaS？），再逐组件设计。
- **聚合位置三选一**：采集 agent（简单聚合、省流量）/ ingestion pipeline（Flink 预聚合、省写量但丢原始精度）/ query side（无损但查询慢）。
  - When to use：写量与查询性能的权衡点。
  - How：按保留策略与查询模式决定在哪一层付出精度或算力。

## Key Concepts
- **规模基线**：100M DAU、1000 池×100 机×~100 指标≈10M 指标、保留 1 年；原始 7 天 → 1min 30 天 → 1h 1 年。
- **Time Series 数据模型**：metric name + labels/tags + timestamp + value；line protocol 格式（Prometheus/OpenTSDB 通用）。
- **Label 基数控制**：标签唯一值数量必须保持低，否则索引爆炸。
- **时序数据库**：不用通用 DB/NoSQL 硬套；OpenTSDB（依赖 HBase）、MetricsDB（Twitter）、Timestream（Amazon）、InfluxDB 与 Prometheus（最流行，内存缓存+磁盘存储）；InfluxDB 参考值 8 核 32GB 可 250k 写/秒。
- **Pull vs Push 采集**：pull（Prometheus）易调试（/metrics 随时看）、自带健康检查、数据真实；push（CloudWatch/Graphite）适合短命任务、跨防火墙网络、UDP 低延迟，但需鉴权防伪造——大型组织两个都要。
- **一致性哈希分配采集目标**：多 collector 实例与服务器同上一致性哈希环，避免重复采集。
- **Kafka 传输管道**：collector→Kafka→流处理（Storm/Flink/Spark）→时序 DB；防 DB 宕机丢数据、解耦采集与处理；可按 metric name/标签分区。
- **Down-sampling**：高分辨率转低分辨率省盘（10s→30s 取均值等），规则可配置。
- **Delta 编码压缩**：存时间戳差值而非全量，显著减容。
- **85%/26h 规律**：Facebook 研究 ~85% 查询落在最近 26 小时——存储与缓存向热窗口倾斜。
- **Alert Manager**：规则（YAML，如 up==0 for 5m）加载进缓存，定时查 query service；职责含过滤/合并/去重、访问控制、至少一次投递重试。
- **Alert Store**：Cassandra 类 KV 存告警状态，触发后进 Kafka，alert consumer 分发到 Email/SMS/PagerDuty/webhook。
- **Query Service**：隔离可视化/告警与 DB，可加缓存；也可直接用 DB 自带查询接口（如 Flux 语言比 SQL 适合时序）。

## Decision Rules
1. 采集端允许 fire-and-forget，偶发丢失可接受
2. 时序按 metric+labels 标识；存储选专用时序 DB，不硬套通用数据库
3. 多 collector 用一致性哈希分采集目标，避免重复
4. 传输加 Kafka 缓冲：防丢、解耦、可再分区聚合
5. 保留策略 = 分辨率分层降采样（7d/30d/1y）+ 冷数据下沉冷存储
6. 告警必须去重、合并、至少一次投递；告警与可视化优先用现成方案（Alertmanager/Grafana）
7. 查询热点（近 26h）用缓存倾斜

## Mental Models
- **监控系统自己是最高优先级客户**：它挂了没人知道——传输层的 Kafka 缓冲和告警的 at-least-once 都是为自身可靠性。
- **精度是货币**：分辨率分层就是"用旧数据的精度换存储成本"的定价表。
- **pull/push 是组织形态的镜子**：网络简单、要可调试选 pull；跨网段、短命任务选 push——别站队，看场景。

## Anti-patterns
- **label 高基数**（把 user_id、request_id 当标签）：索引与存储爆炸。
- **直写时序 DB 不加缓冲**：DB 故障即丢数据。
- **告警不去重不合并**：一次故障千条告警，值班麻木。
- **原始分辨率存一年**：存储成本失控，查询也慢。
- **自建可视化/告警**：Grafana/Alertmanager 已足够好，自研难自证。

## Worked Example
规模与链路：1000×100×100=10M 指标持续写入；collector（auto-scaling，一致性哈希分工）按 pull 拉各机 /metrics（或 agent push）→ 写 Kafka（按 metric name 分区）→ Flink 消费、按规则预聚合 → 写 InfluxDB。保留策略自动执行：>7d 的数据降采样到 1min，>30d 降到 1h。告警：YAML 规则 "up==0 for 5m severity=page" 载入缓存；alert manager 每分钟查 query service，命中则创建事件、去重合并后写 alert store 并发 Kafka；consumer 发 PagerDuty + 邮件。仪表盘 Grafana 查 query service（带缓存），85% 查询命中近 26h 热区。

## Interview Lens
1. 开场用澄清问题界定范围（infra 指标、内部使用、无日志无 trace）——监控题最容易跑偏。
2. 报 10M 指标与 7d/30d/1y 分辨率策略，保留策略就是容量答案。
3. pull vs push 对比表六维度讲全，结论"都要"，展示无偏见。
4. Kafka 缓冲的三个理由（防丢/解耦/分区聚合）是传输层得分点。
5. 主动报 85%/26h 与 delta 编码、降采样三个优化。
6. 告警讲去重合并 + at-least-once；可视化/告警承认用现成方案是成熟表现。

## Practitioner Checklist
- [ ] 五组件边界清晰
- [ ] 时序 DB 选型与 label 基数约束
- [ ] 采集模型（pull/push/混合）与分工（一致性哈希）
- [ ] Kafka 缓冲与流处理聚合
- [ ] 分辨率分层 + 冷存储策略
- [ ] 告警规则、去重、重试、多通道分发
- [ ] 查询缓存向热窗口倾斜

## Key Takeaways
1. 时序数据库 + line protocol + 低基数标签是数据层答案
2. 分辨率分层降采样是保留一年成本的解法
3. Kafka 管道防丢解耦；聚合位置按精度/算力权衡
4. 告警与可视化优先用现成生态

## Connects To
- **Ch 19**：Kafka 传输管道与分区设计。
- **Ch 5**：collector 分工的一致性哈希。
- **Ch 10**：告警多通道分发复用通知系统。
- **Ch 21**：指标聚合与广告点击聚合的窗口计算同源。
