# Chapter 25: Design a Real-time Gaming Leaderboard

## Core Idea
实时排行榜 = Redis sorted set（hash map + skip list，全程有序）承载排名，MySQL 存用户资料与胜负事件供重建；分数更新必须经游戏服务器验证，客户端直报是安全漏洞。

## Frameworks Introduced
- **三方案演进**：关系库（ORDER BY 全表扫或 score 索引+LIMIT，查中游用户排名要嵌套 COUNT 子查询，不扩展）→ Redis sorted set（ZADD/ZINCRBY/ZRANGE/ZREVRANK 全 O(logN)）→ NoSQL（DynamoDB GSI + write sharding，但精确排名难）。
  - When to use：排名存储选型的标准论证链。
  - How：中小规模关系库够用；实时 + 百万级 → Redis；写极重且可接受百分位 → NoSQL。
- **Skip List 原理**：有序链表 + 多级索引；64 节点查找基链表要 62 步、skip list 只要 11 步——这是 sorted set O(logN) 的来源。
  - When to use：被追问"Redis 怎么做到实时排名"。
  - How：内部 = hash map（user_id→score）+ skip list（score→user 有序）。

## Key Concepts
- **规模基线**：5M DAU/25M MAU、人均日 10 局 → 得分 QPS 500、峰值 2,500；榜单查询 ~50 QPS；每月新赛季新榜。
- **安全边界**：POST /v1/scores 只对游戏服务器开放，客户端直报可被中间人改分；服务器托管游戏逻辑时服务器自动记胜。
- **Redis Sorted Set**：始终按分数有序；ZADD（插入/改分）、ZINCRBY（加分，不存在从 0 起）、ZREVRANGE（取 top N 带分数）、ZREVRANK（取用户名次），均 O(logN)（ZRANGE 为 O(logN+M)）。
- **月度榜生命周期**：key 如 leaderboard_feb_2021，新月开新榜，旧榜移历史存储。
- **附近玩家**：ZREVRANK 得位置后 ZREVRANGE pos-4 pos+4 取上下四名。
- **存储估算**：25M MAU 全参与、26B/条 ≈ 650MB，skip list 翻倍也轻松装下；2,500 QPS 单 Redis 实例可扛。
- **MySQL 辅助双表**：用户资料表（username 等）+ 胜负事件表（基础设施故障时重建排行榜的依据）。
- **范围分片（range partition）**：按分数段分片（如 [900–1000]），user_id→shard 映射存 MySQL/缓存；top10 查最高分段；用户排名 = 片内 rank + 更高分片总人数（info keyspace O(1)）。
- **哈希分片（Redis Cluster）**：top10 需各片取 top10 再应用层归并；K 大或片多时延迟涨、排名无直解——作者倾向范围固定分片。
- **DynamoDB 方案**：score 做 sort key；按月分区会热点 → write sharding（user_id % N 拼分区号）；分区数 = 写扩展 vs scatter-gather 读成本的权衡，需基准测试。
- **百分位降级**：分片后精确排名难，CRON 分析分数分布给出百分位区间（如 top 10% = score≥6500）。
- **平局处理**：同分同名次；可加"最后对局时间"做次级排序破平。
- **Serverless 选项**：API Gateway + Lambda 免运维自动扩缩，从零起建推荐。

## Decision Rules
1. 单赛季中小规模直接用 Redis sorted set（650MB/2500 QPS 单实例够）
2. 分数更新只接受游戏服务器调用，拒绝客户端直报
3. MySQL 记胜负事件与资料：展示靠 Redis，重建靠 MySQL
4. 10 倍规模 → 按分数范围分片（top10 与排名都可解），不选哈希分片
5. 写极重场景可用 DynamoDB write sharding，接受百分位替代精确排名
6. top10 用户资料缓存（高频访问）
7. Redis 配副本 + 持久化；写重节点预留 2 倍内存给快照

## Mental Models
- **排名 = 始终有序**：关系库"查询时排序"与 sorted set"写入时有序"的本质区别，决定实时性。
- **快数据与真数据分层**：Redis 是展示层（快、可丢可重建），MySQL 是事实层（慢、权威、可重建 Redis）。
- **分片方式决定查询能力**：范围分片保住 top-N 与排名语义，哈希分片只保写入均衡——先想查询再选分片。

## Anti-patterns
- **客户端直接上报分数**：代理改分作弊，必须服务器验证。
- **关系库算中游用户排名**：嵌套 COUNT 子查询随规模劣化。
- **哈希分片后硬算精确排名**：没有直解，徒增延迟。
- **按月做 DynamoDB 分区键**：当月分区成绝对热点。
- **Redis 无副本无持久化**：宕机即榜单蒸发，只能靠 MySQL 慢慢重建。

## Worked Example
得分流：mary1934 赢一局 → 客户端报游戏服务器 → 验证对局有效 → 调 leaderboard service POST /v1/scores → ZINCRBY leaderboard_feb_2021 1 'mary1934'（O(logN)）→ 同写 MySQL 胜负事件。看榜：GET /v1/scores → ZREVRANGE leaderboard_feb_2021 0 9 WITHSCORES → 批量查 MySQL/缓存补 username → 返回 top10。查自己：GET /v1/scores/user5 → ZREVRANK 得 rank 6 → 返回；附近玩家 ZREVRANGE 2 10。扩容 500M DAU（65GB、250k QPS）：按分数范围分片，user_id→shard 映射入缓存；top10 只查最高分片；用户排名 = 片内 ZREVRANK + 更高分片人数（info keyspace）。灾难恢复：Redis 集群挂 → 脚本回放 MySQL 胜负事件重建 sorted set。

## Interview Lens
1. 开场澄清六问（计分/范围/月度/展示内容/规模/平局/实时性）——实时性是方案分水岭。
2. 报估算：500 QPS 均/2500 峰、650MB——先证明单实例够，再谈扩展。
3. 安全点主动讲：分数只能服务器上报，中间人攻击是送分也是送命题。
4. sorted set 内部结构（hash map + skip list）与四命令复杂度要脱口而出。
5. 扩展讲范围分片并解释为什么不选哈希分片——有对比才有说服力。
6. NoSQL 替代 + 百分位降级 + 平局处理 + MySQL 重建，收尾话题池充足。

## Practitioner Checklist
- [ ] 分数上报只接受游戏服务器
- [ ] sorted set 命令与复杂度明确
- [ ] 月度榜 key 设计与历史归档
- [ ] MySQL 双表（资料+事件）与重建脚本
- [ ] Redis 副本 + 持久化 + 2 倍内存余量
- [ ] top10 用户资料缓存
- [ ] 扩展方案：范围分片 + 映射表
- [ ] 平局与附近玩家查询

## Key Takeaways
1. Redis sorted set 是实时排行榜的默认答案（O(logN) 全操作）
2. 服务器验证分数是安全底线
3. MySQL 事件流是 Redis 的重建保险
4. 范围分片保排名语义，哈希分片只保写均衡；超大写量降级百分位

## Connects To
- **Ch 13**：Top-K 缓存与热度排序思想同源。
- **Ch 6**：KV/NoSQL 选型对比。
- **Ch 19**：可选的游戏结果消息队列扇出。
- **patterns.md**：有序结构与排名模式。
