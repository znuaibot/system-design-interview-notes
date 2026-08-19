# Chapter 1: Scale from Zero to Millions of Users

## Core Idea
扩展是一条逐步演进的路线：从单机出发，每次只解决当前瓶颈——分离数据库、加负载均衡、复制、缓存、CDN、无状态化、异步化，最后才分片。

## Frameworks Introduced
- **逐级演进路线图**：Single Server → Database Separation → Load Balancer → DB Replication → Cache/CDN → Stateless Web Tier → Multi-DC → Message Queue → Sharding。
  - When to use：规划任何 web 系统的扩展路径，或面试中展示"为什么现在加这个组件"。
  - How：每一步都对应一个具体瓶颈（读放大、写瓶颈、单点故障、静态内容延迟、会话粘连、突发流量），先证明瓶颈存在再引入组件。
- **NoSQL 选型四条件**：超低延迟、非结构化/无关系数据、只需序列化/反序列化（JSON/XML/YAML）、海量存储——满足才选 NoSQL，否则默认关系型。

## Key Concepts
- **Single Server Setup**：web、数据库、缓存全在一台机器，DNS 解析域名到服务器 IP，请求走 HTTP；一切扩展的起点。
- **Database Separation**：把数据库迁到独立服务器，让 web 层和数据层可以各自独立扩展。
- **Vertical Scaling**：加 CPU/RAM，受硬件上限约束且无冗余；适合小流量过渡。
- **Horizontal Scaling**：加服务器数量，需要负载均衡器分发请求，是大规模系统的正路。
- **Load Balancer**：把流量分发到多台服务器，提供冗余（故障机流量转移）与弹性（峰值时加机器）。
- **Master-Slave Replication**：master 承担全部写，slave 承担读；读多写少时 slave 数量通常多于 master。
- **Cache**：内存中存热数据降低数据库压力；需设计过期策略、一致性、多节点避免 SPOF、LRU 等驱逐策略。
- **CDN**：地理分布的边缘节点缓存静态内容（图片/CSS/JS）；关注成本、过期时长、故障回源、文件失效。
- **Stateless Web Tier**：会话状态移到共享数据存储，web 层才能水平扩展和按流量自动伸缩。
- **Sharding**：按 sharding key（如 user_id）把数据切到多个同构分片；三大挑战是 resharding、名人热点、跨分片 join（用反范式化规避）。

## Decision Rules
1. 读多写少 → 先加 slave 复制和缓存，写瓶颈出现才动 master
2. 静态内容占比高 → 上 CDN，并设计回源与失效
3. 会话粘住单机 → 状态外置，web 层无状态化
4. 单库容量或写达到上限 → 才分片，且优先选分布均匀的 key
5. 满足 NoSQL 四条件之一才放弃关系型数据库

## Mental Models
- **把每一层都当会坏的设计**：负载均衡去掉单机依赖、复制去掉数据库单点、多缓存节点去掉缓存 SPOF——冗余建在每一层。
- **演进顺序即论证顺序**：组件的引入顺序就是瓶颈出现的顺序，倒序堆组件就是过度设计。
- **缓存是临时层**：任何进缓存的数据都要回答"过期了怎么办、不一致了怎么办、节点挂了怎么办"。

## Anti-patterns
- **用垂直扩容硬扛长期增长**：硬件有上限、成本高、单点风险增大，最终仍要水平扩展。
- **会话存在单台 web 服务器**：该机器故障即丢会话，且请求被绑定无法水平扩展。
- **未测瓶颈就先分片**：resharding、热点和跨分片 join 的复杂度提前支付，而瓶颈可能在别处。
- **缓存不设过期与驱逐策略**：内存被永久数据占满，且数据修改后读到陈旧值。

## Worked Example
从单机到百万用户的典型路径：① 单机跑全部组件；② 数据库独立部署；③ 流量增长，加负载均衡 + 多台无状态 web；④ 读放大出现，建 master-slave 复制；⑤ 热点读进缓存（LRU + TTL），静态资源进 CDN；⑥ 跨地域部署，GeoDNS 把用户导向最近数据中心并跨中心复制数据；⑦ 慢任务（发邮件、生成缩略图）进消息队列异步化；⑧ 单库到顶，按 user_id 分片，为名人账号单独分片规避热点。

slave 全部挂掉时读流量临时回到 master；master 挂掉时提升 slave 为新 master，并用数据恢复脚本补齐落后的数据（multi-master/循环复制可缓解）。

## Interview Lens
1. 画出"单机 → 分片"的演进链，并为每一步说出触发瓶颈——这是本章最有区分度的表达。
2. 被问"为什么不用 NoSQL"时，用四条件反证而不是背组件名。
3. 主动提分片的三个代价（resharding、热点、join），显示你知道方案的边界。
4. 用缓存五问（用途/过期/一致性/故障/驱逐）展示工程深度。

## Practitioner Checklist
- [ ] 每一层（web、DB、cache、CDN）都有冗余方案
- [ ] web 层无状态，会话在共享存储
- [ ] 缓存有 TTL、驱逐策略和多节点部署
- [ ] CDN 有回源降级和文件失效手段
- [ ] 分片键经过均匀性验证，有 resharding 预案
- [ ] 日志、指标、自动化部署就位后再谈更大规模

## Key Takeaways
1. Keep the web tier stateless——web 层无状态是水平扩展的前提
2. Build redundancy at every tier——每一层都建冗余
3. 缓存与 CDN 是读性能的第一杠杆
4. 数据层最终靠 sharding 扩展，但要最后才用
5. 用消息队列解耦组件，换取灵活性

## Connects To
- **Ch 2**：每次演进前用粗略估算验证瓶颈量级。
- **Ch 5**：consistent hashing 缓解 resharding 问题。
- **Ch 6**：分片与复制在 KV 存储中的完整实现。
- **Ch 19**：消息队列的完整设计。
