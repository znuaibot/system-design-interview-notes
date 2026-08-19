# Chapter 13: Design a Search Autocomplete System

## Core Idea
自动补全 = Data Gathering Service（离线聚合查询频率、周期性建 trie）+ Query Service（Cached Trie 前缀检索返回 Top-5）；写路径离线化、读路径极致快（<100ms）。

## Frameworks Introduced
- **两服务拆分**：Data Gathering（采集聚合）与 Query（在线查询）分离。
  - When to use：任何"写多但可延迟、读要求极快"的建议系统。
  - How：用户查询先落 analytics logs（append-only），aggregator 周期性聚合成 frequency table，worker 异步重建 trie；在线侧只读 trie。
- **Cached Trie**：每个 trie 节点直接缓存其子树的 Top-K 查询。
  - When to use：前缀检索的性能核心——避免每次查询遍历整个子树再排序。
  - How：查询时定位前缀节点，直接读节点缓存的 Top-K 列表返回。

## Key Concepts
- **需求基线**：Top-5 建议、按查询热度排序、仅小写英文、<100ms 响应、10M DAU、峰值 48,000 QPS、日增 0.4GB 查询数据、高可用。
- **Trie（前缀树）**：层级存储前缀压缩冗余，节点携带频率信息；Top-K 检索三步：定位前缀节点 → 遍历子树收集候选 → 排序取前 K。
- **Cached Trie**：节点缓存 Top-K，把遍历+排序变成一次读。
- **Frequency Table**：查询串→频率的聚合表，是 trie 的数据来源。
- **Analytics Logs**：原始查询日志，append-only 不索引，供周聚合。
- **Aggregators**：把日志处理成 frequency table；实时性要求高的场景（如 Twitter）缩短聚合间隔，一般周级足够。
- **Filter Layer**：删除有害/违规建议（如仇恨言论）；规则灵活，物理删除异步执行。
- **Trie Cache**：分布式缓存把 trie 放内存加速读。
- **Trie DB 两种存法**：文档存储（MongoDB，周快照序列化存储）或 KV 存储（前缀→key、节点数据→value 的哈希映射）。
- **Shard Map Manager**：按前缀范围分片（a–m / n–z），片内再细分（aa–ag / ah–an）平衡不均，路由请求到对应服务器。

## Decision Rules
1. 实时更新 trie 不可行（十亿级日查询、Top 建议变化慢）→ 离线聚合 + 周期重建（默认周级，实时场景缩短间隔）
2. 读路径用 Cached Trie，节点直存 Top-K
3. 前缀长度上限 ~50 字符（用户很少打更长），缩小搜索空间
4. 分片按前缀范围 + shard map manager 路由，热点前缀再细分
5. 过滤层独立于 trie，规则可插拔、删除异步
6. 客户端用 AJAX 轻量请求 + 浏览器缓存常见前缀

## Mental Models
- **写慢读快的系统都长这样**：把实时性要求低的一侧离线化（logs→聚合→重建），把在线侧简化成纯内存查表。
- **缓存下沉到数据结构节点**：不是给 trie 外面套缓存，而是让每个节点自带答案——缓存粒度跟着查询模式走。
- **分片键选前缀范围**：查询天然按前缀聚集，范围分片让路由可预测；不均匀就递归细分。

## Anti-patterns
- **每次查询实时遍历子树排序**：48k QPS 下遍历+排序直接超时——必须 Cached Trie。
- **每条用户查询实时更新 trie**：十亿级日写入，且 Top-K 基本不变，纯属浪费。
- **按 a-z 平均 26 分片**：字母分布极不均匀，需要 shard map manager 动态细分。
- **把过滤规则硬编码进 trie 构建**：新规则要重建整棵树。

## Worked Example
用户输入 "tw"：AJAX 请求 → shard map manager 按前缀路由到 a–m 片服务器 → trie cache 定位 "tw" 节点 → 直接返回节点缓存的 Top-5（twitter、two player games…）→ <100ms。后台：查询落 analytics logs；aggregator 每周聚合成 frequency table；worker 重建 trie 并序列化存 Trie DB（KV 存 prefix→节点数据），同时预热 trie cache。过滤层按规则标记有害建议，异步物理删除。扩容：a–m 片过载 → 细分为 aa–ag、ah–an…，由 shard map manager 更新路由。

## Interview Lens
1. 开场报全数字：10M DAU、48k 峰值 QPS、0.4GB/天、Top-5、<100ms、仅小写英文。
2. 两服务拆分 + "实时建 trie 为什么不行"的论证是演进得分点。
3. Cached Trie 必须点名——只说"用 trie"不够，说清节点缓存 Top-K 才是深挖。
4. 分片讲 a–m/n–z + 再细分 + shard map manager，展示热点处理。
5. 存储双选项（MongoDB 快照 vs KV 前缀映射）主动对比。

## Practitioner Checklist
- [ ] 数字基线完整（QPS/DAU/延迟/增量）
- [ ] 采集与查询两服务分离
- [ ] trie 离线重建周期已定
- [ ] Cached Trie + 前缀长度上限
- [ ] 过滤层独立可配
- [ ] 分片与路由（shard map manager）就位
- [ ] trie cache 与 Trie DB 双层存储

## Key Takeaways
1. Cached Trie（节点缓存 Top-K）是 <100ms 的关键
2. 写路径离线化：logs → aggregator → 周期重建 trie
3. 前缀范围分片 + shard map manager 处理不均
4. 存储可选 MongoDB 快照或 KV 前缀映射；多语言/地区可用独立 trie 与趋势加权扩展

## Connects To
- **Ch 6**：KV 存储用于 trie 持久化。
- **Ch 14/15**：YouTube/Drive 的搜索建议复用本架构。
- **Ch 25**：热度排序与 leaderboard 的排名问题相通。
- **patterns.md**：离线预计算 + 在线查表模式。
