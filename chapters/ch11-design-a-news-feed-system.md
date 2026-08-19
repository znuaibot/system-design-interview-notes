# Chapter 11: Design a News Feed System

## Core Idea
Feed 系统拆成两条流：发布流（写帖 + fanout 传播到好友 feed）与拉取流（聚合好友帖子按时间倒序展示）；核心权衡是 fanout on write vs fanout on read，名人用混合方案。

## Frameworks Introduced
- **Fanout 三选一**：写扩散（发布时推给所有好友 feed）vs 读扩散（读取时现聚合）vs 混合（普通用户推、名人拉）。
  - When to use：feed 发布设计的第一决策。
  - How：写扩散实时且读快但大 V 发布写放大严重；读扩散对不活跃用户省资源但读慢；混合按用户连接数分流——这正是"热点优先于平均"的实例。
- **五层缓存架构**：news feed / content / social graph / action / counter 各司其职。
  - When to use：feed 读路径深挖。
  - How：feed 缓存只存 post_id 列表，实体在 content cache，关系在 graph cache，互动与计数分开缓存。

## Key Concepts
- **需求基线**：web+mobile、发布与浏览、时间倒序、好友上限 5,000、10M DAU、含文本图片视频。
- **Feed Publishing API**：POST /v1/me/feed（content + auth_token）。
- **News Feed Retrieval API**：GET /v1/me/feed（auth_token）。
- **Fanout on Write**：发布时把帖子推入好友 feed 缓存；实时、读快，但对好友多的用户资源消耗大。
- **Fanout on Read**：读取时现拉好友最新帖子合并；对不活跃用户高效，但读延迟高。
- **Hybrid Approach**：普通用户写扩散，高连接用户（名人）读扩散。
- **Fanout Service 流程**：从 graph database 取好友列表 → 用缓存中的设置过滤（屏蔽/分组可见）→ 好友列表+post ID 进消息队列 → fanout workers 更新好友 feed 缓存。
- **News Feed Cache 结构**：存 <post_id, user_id> 映射而非完整对象省内存；设可配置上限只保留近期帖子。
- **Graph Database**：存好友关系，支撑 fanout 取好友列表。

## Decision Rules
1. 普通用户发布时用 fanout on write 预计算收件箱
2. 名人/大 V 内容读时合并（fanout on read），避免写放大
3. feed 缓存只存 post ID 列表，实体数据放 content cache 分层缓存
4. fanout 走消息队列 + worker 池，削峰并解耦
5. 发布前先按用户设置（屏蔽、分组）过滤好友
6. web 层无状态、数据库分片 + 读副本应对 10M DAU

## Mental Models
- **写放大是 fanout 的核心代价**：一个 5000 好友的用户发一帖就是 5000 次缓存写——连接数分布决定方案。
- **缓存存指针不存实体**：feed 里放 post ID，实体按需从 content cache/DB 取，内存效率数量级差异。
- **两条流独立演进**：发布流优化写吞吐，拉取流优化读延迟，瓶颈和手段完全不同。

## Anti-patterns
- **对名人做写扩散**：一次发布扇出百万次写，队列积压、缓存风暴。
- **feed 缓存存完整帖子对象**：内存爆炸且更新一致性复杂。
- **不做好友过滤就 fanout**：屏蔽关系泄露，隐私事故。
- **同步 fanout 阻塞发布请求**：发布延迟被最慢的好友写拖垮。

## Worked Example
用户 A（500 好友）发帖：POST /v1/me/feed → LB → web server 鉴权 + 限流 → post service 存 DB 与 content cache → fanout service 从 graph DB 取 500 好友，按设置过滤掉 20 个屏蔽者 → 480 个 <post_id, user_id> 进 MQ → workers 把 post_id 追加到各好友 feed 缓存（保留最近 N 条）。好友 B 刷 feed：GET /v1/me/feed → news feed service 从 feed 缓存取 post ID 列表 → 批量查 content cache 取实体 → 倒序返回。若发帖者是百万粉名人：跳过写扩散，B 刷 feed 时读路径实时合并该名人最新帖。

## Interview Lens
1. 开场就拆两条流（publishing/building），这是本章的骨架表达。
2. fanout 三方案对比 + 混合方案是必考深挖，主动画出名人例外。
3. 能说出五层缓存各自存什么，展示读路径工程深度。
4. 主动提 feed 缓存存 ID、MQ 异步 fanout、上限截断三个省内存技巧。

## Practitioner Checklist
- [ ] 发布与拉取两条流分别有架构图
- [ ] fanout 策略按用户连接数分流
- [ ] 好友过滤（屏蔽/分组）在 fanout 前
- [ ] feed 缓存只存 ID 且有长度上限
- [ ] MQ + worker 承接 fanout 洪峰
- [ ] QPS、延迟、缓存命中率有监控

## Key Takeaways
1. fanout on write/read/hybrid 是 feed 系统的核心权衡，名人必须例外
2. feed 缓存存 post_id，实体分层缓存
3. graph DB 存关系，MQ 解耦 fanout
4. 数据库分片 + 读副本 + 无状态 web 层支撑 10M DAU

## Connects To
- **Ch 17**：nearby friends 也面临 fanout 与位置结合的变体。
- **Ch 19**：fanout 用的消息队列完整设计。
- **Ch 6**：social graph 缓存与 KV 存储。
- **patterns.md**：读写扩散模式。
