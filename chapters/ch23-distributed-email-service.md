# Chapter 23: Design a Distributed Email Service

## Core Idea
分布式邮件 = 元数据库（user_id 分区、CP 取向）+ 附件对象存储 + 双队列收发流水线 + 实时推送（WebSocket）；送达率靠 IP 信誉运营，搜索走 Elasticsearch 异步索引。

## Frameworks Introduced
- **邮件协议栈与 MX 解析**：SMTP（服务器间发送）、POP（收后删）、IMAP（收后留）、HTTPS（web 客户端，本题选用，换灵活性）；收件方域名经 DNS MX 记录定位邮件服务器。
  - When to use：邮件题开场领域知识展示。
  - How：跨服务器交换走 SMTP，自家 web 客户端走 HTTP REST（POST /v1/messages、GET /v1/folders/{id}/messages 等）。
- **送达率运营框架**：发信容易进收件箱难——dedicated IP、营销与事务邮件分流、IP 预热 2–6 周、快速封禁 spammer、ISP 反馈回路、SPF/DKIM 认证。
  - When to use：被问"为什么我们的邮件进垃圾箱"。
  - How：按清单逐条讲，强调这是领域知识而非架构技巧。

## Key Concepts
- **规模基线**：1B 用户、人均日发 10 封 → 100k 封/秒；日收 40 封×50KB 元数据 → 730PB/年；20% 带附件均 500KB → 1,460PB/年。
- **发送流水线**：LB 限流 → web server 校验（大小/spam，同域短路）→ 合格进出队列（附件引用对象存储）、不合格进错误队列 → outgoing workers 拉取、查毒、SMTP 投递目标服务器 → 存 Sent 文件夹；队列积压监控（对端不可用→指数退避重试；消费不足→扩 consumer）。
- **接收流水线**：SMTP LB → SMTP server 执行接收策略（无效邮件直接丢）→ 大附件转存 S3 → 处理 worker 初检后分发到存储/缓存/对象存储/实时服务器；离线用户上线后经 HTTP API 补拉。
- **元数据特征**：header 小且频访、body 只读一次、操作按用户隔离、只读近期邮件、零容忍丢失 → Gmail 级用自研库降 IOPS（BigTable 系）。
- **user_id 分区键**：一个用户的数据落一个 shard（放弃跨用户共享邮件的需求）。
- **timeuuid**：email_id 用时间 UUID，天然按创建时间排序。
- **读写表反范式**：Cassandra 不允许按非键列过滤，read/unread 拆成两张表而非查询时过滤。
- **会话线程头**：Message-Id / In-Reply-To / References 由邮件客户端重建会话树。
- **CP 取向**：一致性是硬要求，failover/分区期间受影响用户短暂不可写。
- **搜索双路径**：Elasticsearch（user_id 分区、变更经 Kafka 异步重索引、查询同步）；自研则 LSM tree（内存攒写、阈值后合并下沉磁盘，Cassandra/BigTable/RocksDB 同款）优化写多场景。
- **邮件搜索 vs 网页搜索**：范围是用户自己的邮箱、按属性排序、索引要快结果要准。

## Decision Rules
1. 发送先落队列再异步投递，持久化优先于速度
2. 按目标域限速，临时失败指数退避重试
3. 正文元数据进元数据库，附件进对象存储（S3），全文索引进搜索存储——三层各司其职
4. 元数据按 user_id 分区；read/unread 反范式拆表
5. 一致性优先：CP 取向，分区时牺牲部分可用
6. 索引用 Kafka 异步重索引，搜索查询同步
7. 送达率靠 IP 信誉运营清单，不靠架构补丁

## Mental Models
- **邮件是三个存储问题的组合**：小元数据（高频读）、大附件（低频大块）、索引（写多）——别指望一个存储全扛。
- **反范式是查询模式驱动的**：Cassandra 的过滤限制逼出 read/unread 双表——数据模型跟着查询走，不是反过来。
- **进收件箱是信誉问题**：技术上能发出 ≠ 对方愿意收，IP 预热与认证协议是运营资产。

## Anti-patterns
- **每封邮件一个文件存本地磁盘**：传统邮件服务器做法，磁盘 I/O 与可用性双瓶颈。
- **新 IP 直接大量发信**：必进垃圾箱，信誉要 2–6 周预热。
- **查询时在 Cassandra 里过滤 read/unread**：非键列过滤被禁或全表扫。
- **营销邮件与事务邮件共用发信通道**：营销被标垃圾连累验证码邮件。
- **搜索索引同步更新**：写路径被重索引拖垮，应 Kafka 异步。

## Worked Example
发送：Alice 发带 1MB 附件的邮件 → LB 限流后到 web server → 校验大小、spam 初检 → 附件传 S3 得引用，邮件主体进 outgoing queue → outgoing worker 查毒、DNS 查 gmail.com 的 MX 记录 → SMTP 投递 Gmail 服务器 → Alice 的 Sent 文件夹落库。若 Gmail 服务器宕机：队列积压告警，worker 指数退避重试。接收：Bob 的 Gmail 侧 SMTP server 收信 → 接收策略检查 → 处理 worker 写元数据（emails 表 timeuuid）、缓存近期邮件、推实时服务器 → Bob 在线则 WebSocket 推新邮件提醒，离线则上线后经 HTTP 拉取。搜索：Bob 搜 "from:alice invoice" → 查询同步打 Elasticsearch（user_id 分区到固定节点）；此前每封新邮件经 Kafka 异步触发重索引。

## Interview Lens
1. 开场报协议栈（SMTP/POP/IMAP）+ MX 记录，展示领域基础。
2. 报容量：100k 封/秒、730PB+1,460PB/年——附件存储主导。
3. 元数据库分析（五个特征→自研/BigTable 方向）+ user_id 分区 + timeuuid + 反范式，是数据层深挖主线。
4. 送达率清单（IP 预热/SPF/DKIM/反馈回路）主动讲，区分度极高。
5. 搜索讲 ES vs 自研 LSM 的三条权衡；承认"自研超出面试范围"是成熟表现。
6. 收尾提 GDPR、加密、附件去重等加分话题。

## Practitioner Checklist
- [ ] 收发双流水线与错误队列
- [ ] 附件对象存储 + 元数据/索引分离
- [ ] user_id 分区 + timeuuid + read/unread 拆表
- [ ] 队列积压监控与退避重试
- [ ] WebSocket 实时推 + 离线 HTTP 补拉
- [ ] 送达率运营清单（IP/认证/反馈）
- [ ] Kafka 异步搜索索引
- [ ] 多数据中心 + leader-follower failover
- [ ] PII 合规与加密

## Key Takeaways
1. 元数据/附件/索引三层存储分工是骨架
2. user_id 分区 + CP 取向 + 反范式适配邮件访问模式
3. 送达率 = IP 信誉运营（预热/分流/认证/反馈）
4. 搜索写多读少：ES 异步索引或自研 LSM

## Connects To
- **Ch 10**：通知推送与实时通道复用。
- **Ch 19**：收发队列与重试语义。
- **Ch 24**：附件的 S3 类存储完整设计。
- **Ch 21**：Kafka 异步管道与写多索引的 LSM 思想。
