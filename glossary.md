# Glossary

**301/302 Redirect** — 永久跳转（浏览器缓存、丢统计）/临时跳转（每次过服务、可统计）（Ch 8）。
**3D Secure** — 信用卡额外验证流程，可使支付耗时数小时，需 pending 状态承接（Ch 26）。
**ACK 策略** — 生产者确认等级：all（ISR 全同步）/1（leader 收到）/0（不等），持久性与延迟的旋钮（Ch 19）。
**Adaptive Rerouting** — 导航中按实时路况重算路线，用 super tile 链定位受影响用户（Ch 18）。
**Alert Manager** — 按规则查询指标、创建告警事件并去重合并、保证至少一次投递的组件（Ch 20）。
**Analytics Log** — append-only 的原始查询日志，供周期聚合建 trie（Ch 13）。
**APNS/FCM** — Apple/Android 官方推送通道，通知系统的第三方投递端（Ch 10）。
**Application Loop** — 绑核运行的 while 循环处理关键任务，无上下文切换与锁竞争（Ch 28）。
**At-most/At-least/Exactly-once** — 交付语义三档：可能丢/可能重（需幂等）/不丢不重（最贵）；at-least-once 下消费者必须幂等（Ch 19, 21, 26）。
**Availability Nines** — 可用性等级，每加一个 9 年停机缩 10 倍（99.9%≈8.8h/年）（Ch 2）。
**Back-of-the-envelope** — 信封背面估算法：假设显式化 + 数量级计算定位主导成本（Ch 2）。
**Backpressure** — 下游饱和时限制上游生产速度，防止队列和内存无界增长（Ch 19, 20）。
**Base62** — 用 62 字符编码自增 ID 生成短码，无碰撞；7 位支持 3.5 万亿（Ch 8）。
**Bid/Ask** — 买一价（最高买）/卖一价（最低卖）；L1/L2/L3 为三级行情深度（Ch 28）。
**Block（4MB）** — Google Drive 文件切块单位，哈希寻址独立存储支撑 delta sync（Ch 15）。
**Bloom Filter** — 低内存概率集合，快速判定"肯定不存在/可能存在"（Ch 6, 8, 9）。
**Broker** — 承载 MQ partition 的服务器；也指证券交易中介券商（Ch 19, 28）。
**Bucket** — 对象存储的全局唯一名逻辑容器（Ch 24）。
**Cached Trie** — 每节点缓存子树 Top-K 的前缀树，自动补全 <100ms 的关键（Ch 13）。
**CAP Theorem** — 网络分区发生时，一致性与可用性之间必须取舍；CA 系统不存在（Ch 6）。
**CDN** — 在边缘缓存静态或媒体内容以降低延迟与源站负载（Ch 1, 14, 18）。
**CDC** — 数据库变更流式同步到缓存/下游（如 Debezium），异步最终一致（Ch 22）。
**Compaction** — GC 把存活对象从旧文件拷到新文件再换映射，回收空间（Ch 24）。
**Consistent Hashing** — 节点变化时仅迁移局部键的环形分区方法（Ch 5, 6）。
**Consumer Group** — 协作消费 topic 的消费者集合，offset 按组维护，组内一分区一消费者（Ch 19）。
**Coordinator** — MQ 组协调者（按组名哈希定位、分配分区）；也指分布式事务协调者（Ch 19, 27）。
**CQRS** — 命令与查询分离：多个只读状态机基于不可变事件服务查询（Ch 27）。
**Cur_max_message_id** — 设备本地消息游标，多设备同步只拉游标之后的增量（Ch 12）。
**DAG 模型** — 有向无环图任务编排，视频转码并行化的核心（Ch 14）。
**Dead-letter Queue** — 终态失败消息的隔离队列，供人工排查（Ch 10, 26）。
**Delta Encoding** — 存时间戳差值而非全量，时序数据压缩手段（Ch 20）。
**Delta Sync** — 只传修改过的块而非整文件（Ch 15）。
**Double-entry Ledger** — 每笔资金变动借贷双记、总和恒零，端到端可追溯（Ch 26, 27）。
**Down-sampling** — 高分辨率数据转低分辨率省存储（如 10s→1min）（Ch 20）。
**Drain** — WebSocket 服务器下线前停止接新连接、优雅关闭存量连接（Ch 17）。
**Erasure Coding** — 数据+校验片分散存储，8+4 方案 50% 开销换 11 个 9 持久（Ch 24）。
**Event Sourcing** — 以不可变事件日志作为状态来源，通过重放重建状态（Ch 27, 28）。
**Event Time vs Processing Time** — 事件发生时间 vs 服务器处理时间；计费场景选前者配 watermark（Ch 21）。
**Fanout on Write/Read** — 发布时推给所有接收者 vs 读取时现聚合；名人用混合（Ch 11, 17）。
**Filter_id/Star Schema** — 过滤维度编号预聚合，星型模型换过滤查询速度（Ch 21）。
**FIX 协议** — 证券交易信息交换的行业标准协议（Ch 28）。
**Funds Withheld** — 挂单占用资金预扣冻结至订单终结（Ch 28）。
**Geocoding** — 地址与经纬度互转；反向称 reverse geocoding（Ch 18）。
**Geohash** — 把经纬度编码为层级字符串前缀的空间索引；有边界问题需查邻格（Ch 16, 17, 18）。
**Geofencing** — 任意形状地理围栏，S2 的强项（Ch 16）。
**Gossip Protocol** — 节点间周期性随机心跳交换，多源判定成员存活（Ch 6）。
**Graph Database** — 存好友/社交关系，支撑 fanout 取好友列表（Ch 11）。
**Hash Ring** — 哈希值首尾相接的环，节点与键同环映射实现局部迁移（Ch 5）。
**Heartbeat** — 周期性心跳判活：presence 30s 失联即离线；集群心跳触发 failover（Ch 12, 17, 28）。
**Hinted Handoff** — 节点临时故障时由替代节点代存写入，恢复后回送（Ch 6）。
**Hosted Payment Page** — PSP 提供的收银页 widget，商户不碰卡数据规避 PCI（Ch 26）。
**Idempotency Key** — 重复请求共享的稳定键（UUID/订单号）+ unique 约束，使重试返回同一业务结果（Ch 22, 26）。
**IMAP/POP/SMTP** — 收邮件留服务端/收后删/服务器间发送的标准邮件协议（Ch 23）。
**ISR** — 与 leader 保持足够同步、可参与提交与选举的副本集合（Ch 19）。
**Kappa/Lambda 架构** — 单流处理路径（历史重算也走流）vs 批流双路径双代码库（Ch 21）。
**L1/L2/L3 行情** — 最优价量/多档价位/档位排队量三级市场数据（Ch 28）。
**Latency Numbers** — L1 0.5ns/内存 100ns/SSD 150µs/HDD 10ms/跨区 150ms 的常识表（Ch 2）。
**Leader Election** — 主挂后从副本选新主（Raft），非 leader 只收不发（Ch 27, 28）。
**Leaky Bucket** — 入队后恒速出队的平滑限流算法（Ch 4）。
**Limit Order** — 按固定价格买卖的订单，可部分成交或挂簿（Ch 28）。
**Line Protocol** — metric name+labels+timestamp+value 的时序数据格式（Ch 20）。
**Local Sequence Number** — 消息序号只在会话/群内唯一有序，比全局 ID 更省更对（Ch 12）。
**Long Polling** — 服务端挂起请求直到有数据再返回，减少空轮询（Ch 12, 15, 23）。
**LSM Tree** — 内存攒写、阈值后合并下沉磁盘的写优化结构（Cassandra/RocksDB）（Ch 23, 27）。
**Map Tile/Routing Tile** — 按 zoom 分层的地图渲染单元/按路口道路建图的路径计算单元（Ch 18）。
**MapReduce DAG** — map 分发、aggregate 内存计数、reduce 汇总的流式聚合管道（Ch 21）。
**Matching Engine** — 维护订单簿并按序撮合买卖生成双向 fills 的核心（Ch 28）。
**Merkle Tree** — 通过分层哈希高效比较副本差异，只同步不一致桶（Ch 6）。
**Message Sync Queue（inbox）** — 每接收者一个收件箱，汇集多发送者消息（Ch 12）。
**mmap** — 磁盘文件映射进内存的 UNIX 调用；/dev/shm 中则零磁盘访问（Ch 27, 28）。
**Multipart Upload** — 大文件分块独立上传（upload_id+etag），complete 时重组（Ch 24）。
**MX Record** — DNS 中指向域名邮件服务器的记录（Ch 23）。
**Namespace** — 每用户一个的文件根目录隔离（Ch 15）。
**Offset** — partition 内消息位置；消费者崩溃后从已提交 offset 续读（Ch 19）。
**Optimistic Locking** — version 列读后校验写，低竞争快；对比悲观锁与 DB 约束（Ch 6, 22）。
**Order Book** — 按价位组织的买卖订单列表；双向链表+orderMap 实现 O(1) 挂单/撮合/撤单（Ch 28）。
**Overbooking** — 超订策略（如 110% 库存），写入可用量公式（Ch 22）。
**Partition Key** — 决定消息落区的键（hash(key)%N），保局部顺序（Ch 19）。
**PCI Compliance** — 处理品牌信用卡的安全标准；不存卡号即可大幅规避（Ch 26）。
**Phase Status Table** — TC/C 协调者的事务阶段状态表，支持崩溃恢复与乱序检测（Ch 27）。
**Placement Service** — 维护集群拓扑、决定对象落哪些数据节点、心跳摘除故障（Ch 24）。
**Pre-signed URL** — 授权用户限时直传对象存储，API 不经手文件流（Ch 14）。
**PSP** — 支付服务提供商（Stripe 等），真正执行资金转移（Ch 26）。
**Pub/Sub** — 按 topic/频道发布订阅；轻量如 Redis pub/sub，重量如 Kafka topic（Ch 15, 17, 19）。
**Pull vs Push（采集/消费）** — 拉模型消费方控制速率、可调试；推模型低延迟但易压垮接收方（Ch 19, 20）。
**Quadtree** — 按密度递归四分二维空间的索引，支持 k-nearest（Ch 16）。
**Quorum（N/W/R）** — 副本数/写确认/读响应，W+R>N 保证强一致（Ch 6）。
**Raft** — leader/follower 共识复制，过半存活可用，保序保不丢（Ch 24, 27, 28）。
**Rebalancing** — 消费者组成员变化时重新分配 partition（心跳→选主→分配计划→广播）（Ch 19）。
**Reconciliation** — 定期用权威数据重算比对，修正聚合/支付差异（Ch 21, 26）。
**Resharding/Celebrity Problem** — 分片扩容迁移/热点键压垮单片，一致性哈希与专属分片缓解（Ch 1）。
**Resumable Upload** — 取上传 URL 分块传、中断可续（Ch 15）。
**Retry Queue + Backoff** — 可重试失败进队列，默认指数退避（Ch 10, 26）。
**Ring Buffer** — 预分配无锁循环队列，padding 独占 cache line（Ch 28）。
**Routing Layer 内嵌** — 分区路由逻辑嵌入 producer，少跳数且支持批量（Ch 19）。
**Saga** — 将长事务拆为本地事务及相应补偿动作；orchestration 优于 choreography（Ch 22, 27）。
**Sequencer** — 给入站订单与出站成交打全局序号，保证撮合确定性/可重放/公平（Ch 28）。
**Settlement File** — PSP 夜发清算文件，内外对账的依据（Ch 26）。
**Shard Map Manager** — 按前缀范围路由自动补全请求，热点再细分（Ch 13）。
**Skip List** — 有序链表+多级索引，支撑 sorted set 的 O(logN)（Ch 25）。
**Sliding Window Log/Counter** — 记全时间戳精度最高内存最贵 / 双窗加权近似折中（Ch 4）。
**Sloppy Quorum** — 故障时放宽 quorum，选环上健康节点临时读写（Ch 6）。
**Snapshot** — 周期保存聚合或重放的中间状态，恢复不必从头（Ch 21, 27）。
**Snowflake ID** — 由时间、节点和序列位段（1+41+5+5+12）组成的趋势递增分布式 ID（Ch 7）。
**Sorted Set** — Redis 有序集合：hash map + skip list，实时排行榜底座（Ch 25）。
**Spider Trap** — 生成无限链接的恶意页面，用 URL 长度限制规避（Ch 9）。
**SSTable** — 内存缓存刷盘的有序字符串表（Cassandra 写路径）（Ch 6）。
**State Machine（确定性）** — 验证命令、应用事件的引擎；禁外部 IO 与随机才可重放（Ch 27）。
**Stateless Web Tier** — 会话外置后 web 层可水平扩展与自动伸缩（Ch 1）。
**T/C/C（Try-Confirm/Cancel）** — 两阶段补偿事务：try 预留、confirm/cancel 落定或回滚，可并行（Ch 27）。
**Ticket Server** — 中心化发号器，严格递增但单点（Ch 7）。
**Time Series Database** — 时序专用库（InfluxDB/Prometheus），内存缓存+磁盘、按 label 索引（Ch 20）。
**Timeuuid** — 时间 UUID，天然按创建时间排序（Ch 23, 24）。
**Token Bucket** — 按速率补充 token 且允许桶容量内突发的限流算法（Ch 4）。
**Topic/Partition/Broker** — MQ 三级结构：逻辑频道/分片/承载服务器（Ch 19）。
**Trie** — 共享前缀的树结构，适合自动补全（Ch 13）。
**Tumbling Window** — 互不重叠的固定时间窗口（Ch 21）。
**URL Frontier** — 爬虫待抓队列：前队列管优先级、后队列管礼貌性（Ch 9）。
**Vector Clock** — 表示分布式版本因果关系并检测并发更新的向量（Ch 6）。
**Virtual Node** — 一台物理节点在哈希环上的多个位置，用于平衡分布（Ch 5）。
**WAL** — 只追加日志：MQ partition 分段存储、对象存储小文件合并的共用手段（Ch 19, 24）。
**Watermark** — 流处理器对事件时间进度的估计；聚合窗口延长量，长短权衡延迟与漏算（Ch 21）。
**WebSocket** — 双向持久连接，实时聊天/位置/导航推送的默认协议（Ch 12, 17, 18）。
**Write Sharding** — 分区键拼接 user_id%N 序号打散写热点（DynamoDB）（Ch 25）。
**Zookeeper** — 层级 KV 协调服务：配置、服务发现、leader 选举（Ch 12, 19, 20）。
