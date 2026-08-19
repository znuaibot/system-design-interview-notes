# Chapter 9: Design a Web Crawler

## Core Idea
搜索引擎爬虫要在可扩展性、健壮性、礼貌性和可扩展性之间平衡；核心机制是 URL Frontier 的双层调度——前队列管优先级，后队列管礼貌性。

## Frameworks Introduced
- **URL Frontier 前后队列架构**：front queues 按优先级（PageRank、更新频率）分组，back queues 按主机分组保证每主机串行。
  - When to use：任何需要"既要抓重要的、又不能打爆单一站点"的调度问题。
  - How：Prioritizer 计算优先级选 front queue → Queue selector 按权重随机选队列 → Queue router 按主机映射到 back queue → 每个 worker 线程只串行消费一个主机队列，请求间加延迟。
- **四大设计挑战清单**：Scalability（并行化抓数十亿页）、Robustness（坏 HTML/崩溃/恶意链接）、Politeness（不打垮目标站）、Extensibility（新内容类型插件化）。

## Key Concepts
- **Seed URLs**：爬取起点，按地域或主题精选，好的种子能遍历尽可能多的链接。
- **URL Frontier**：待下载 URL 的缓冲结构，同时实现优先级与礼貌性。
- **DNS Resolver / DNS Cache**：URL 转 IP；缓存避免重复解析。
- **Content Seen?**：对页面内容做哈希比较，识别重复/镜像内容。
- **URL Seen? / Bloom Filter**：记录已访问 URL 避免重复抓取。
- **Robots.txt**：下载前必须遵守的站点爬取规则。
- **Spider Trap**：生成无限链接的恶意页面，用 URL 长度限制等手段规避。
- **BFS vs DFS**：web 是有向图，BFS 是默认遍历方式（深度不可控使 DFS 不理想），但标准 BFS 不区分页面重要性，故用优先级队列增强。
- **Freshness**：根据页面历史更新频率和重要性决定重抓周期。

## Decision Rules
1. 按主机分队列控制抓取频率（礼貌性），每主机同时只有一个请求
2. URL 与内容分别去重（URL seen 与 content seen 是两层）
3. 下载、解析、存储、链接发现解耦成独立组件
4. robots.txt 与黑名单过滤是下载的前置条件
5. 优先级用 PageRank/更新频率计算，高优先级队列被更高概率选中
6. 坏页面丢弃、异常隔离，任何单页错误不许崩掉整个爬虫

## Mental Models
- **礼貌性是调度约束不是限速器**：真正的礼貌是"每主机串行 + 间隔延迟"，靠队列拓扑实现而非全局 QPS。
- **两个 seen 检查各有职责**：URL 去重省下载，内容去重省存储与索引。
- **爬虫是插件系统**：内容类型扩展（PNG 下载器、版权监控）应以模块接入而非改主干。

## Anti-patterns
- **单一 FIFO 全局队列**：热门站点的链接挤满队列，且并发请求打爆单一主机。
- **DFS 遍历**：web 图深度不可控，递归深度失控。
- **无限重试坏链接或落入 spider trap**：资源被无价值 URL 耗尽。
- **不做内容哈希**：镜像站产生海量重复索引。

## Worked Example
规模：10 亿页/月 ≈ 400 页/秒（峰值 800 QPS），仅 HTML，保留 5 年 ≈ 30PB。流程：精选种子 URL 入 Frontier → downloader 经 DNS 缓存取页（遵守 robots.txt、短超时）→ parser 校验丢弃坏页 → content hash 查重 → 新内容入库（热数据放内存）→ URL extractor 提取链接 → filter 过滤黑名单 → URL seen 检查 → 新 URL 经 prioritizer 回到 Frontier。Frontier 内：front queues 按优先级分档，queue selector 加权随机；queue router 把 URL 按主机映射到 back queues，每 worker 绑定一个主机队列串行下载并加延迟。

## Interview Lens
1. 开场报四大挑战（scalability/robustness/politeness/extensibility），展示问题全景。
2. URL Frontier 的前后队列双层结构是本章灵魂，手绘 queue router/mapping table/queue selector/worker。
3. 能解释为什么 BFS + 优先级，而不是纯 BFS 或 DFS。
4. 主动提 spider trap、robots.txt、DNS 缓存，显示实战细节。

## Practitioner Checklist
- [ ] 种子 URL 有选择策略（地域/主题）
- [ ] Frontier 同时实现优先级与礼貌性
- [ ] URL 去重与内容去重两层都在
- [ ] robots.txt 合规检查在下载前
- [ ] spider trap 防护（URL 长度限制等）
- [ ] 单页异常隔离，不影响全局

## Key Takeaways
1. 礼貌性防止打垮目标站点，优先级保证重要页面先抓
2. URL Frontier 前后队列是调度核心：front 管优先级、back 管礼貌
3. 大规模存储（30PB/5 年）与错误处理是工程关键
4. 架构插件化支持新内容类型低成本接入

## Connects To
- **Ch 5**：一致性哈希在下载节点间分配负载。
- **Ch 6**：bloom filter 用于 URL 去重。
- **Ch 8**：URL 规范化与去重的共通问题。
- **Ch 11**：抓取的页面是 news feed/搜索索引的上游。
