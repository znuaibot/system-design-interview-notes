# Chapter 18: Design Google Maps

## Core Idea
地图系统三服务：Location Service（批量位置更新入 Cassandra + Kafka 流）、Navigation（geocoding→route planner→A* 最短路径→ETA→ranker）、Map Rendering（按 zoom 分层 tiles 静态走 CDN）；路径计算靠多级分辨率 routing tile 拼接。

## Frameworks Introduced
- **地图 101 概念栈**：projection（Web Mercator）→ geocoding（地址⇄坐标）→ geohash → tiling（256×256，每升一级 tile 数×4）→ routing tile（路口=节点、道路=边的图）。
  - When to use：任何地图题开场，展示领域知识。
  - How：先讲"世界如何变成可计算的图"，再进架构。
- **分层 routing tile**：世界不能整图跑 Dijkstra，按 tile 细分并互相引用；长距离用低分辨率 tile，算法按目的地距离选合适粒度。
  - When to use：大规模路径规划的性能核心。
  - How：A* 从起点 tile 出发遍历，按需加载相邻 tile 拼接大图，只载入起终点需要的部分。

## Key Concepts
- **规模基线**：1B DAU、周用 35 分钟 → 日 5B 分钟；GPS 批量上报 → 200k QPS、峰值 1M QPS；世界地图 tiles ~70PB（压缩相似 tile 后）；实时位置日更 25M 次。
- **Location Service**：客户端批量上报 (lat, long, timestamp) 数组；写重场景用 Cassandra（user_id 分区键、timestamp 聚类键），AP 优先；Kafka 把位置流分发给 ETA/路况/分析等下游。
- **Navigation Service**：GET /v1/nav?origin=…&destination=…，返回距离/时长/polyline/分段指令；允许少量延迟但必须准确。
- **Routing Tile**：离线管道把原始道路数据转成图 tile 存 S3（不用数据库特性）+ 激进缓存；邻接表压缩成二进制。
- **Geocoding DB**：lat/long⇄地点的 KV，读多写少用 Redis。
- **Map Tile**：预计算静态 tile 按 geohash 寻址存 CDN；tile URL 可客户端算（长期承诺需谨慎）或 API 代算（多一次调用）。
- **矢量 tile 优化**：下发路径/多边形矢量由客户端渲染，带宽大幅节省。
- **ETA Service**：ML 模型基于路况数据预测用时。
- **Ranker Service**：按用户过滤条件（避开收费站/高速）给候选路线排序。
- **Adaptive Rerouting**：导航中用户存"当前 tile + 多级 super tile"序列，事故 tile 命中即重路由；比存全程 tile 省存储。
- **推送协议选择**：WebSocket 优于 long-polling（服务端开销小、双向，利于最后一公里类功能）；移动推送 payload 受限且 web 不可用。

## Decision Rules
1. 地图渲染用 zoom 分层 tiles，静态存 CDN，客户端按位置+缩放拉取
2. 道路数据离线转 routing tile 存 S3 + 缓存，不用数据库
3. 路径计算用 A* + 多级分辨率 tile 拼接，只加载相关区域
4. 位置更新客户端批量上报，Cassandra 存、Kafka 流给下游
5. ETA = ML 模型 + 实时路况；路线按 ranker 与用户偏好排序
6. 主动推送选 WebSocket

## Mental Models
- **一切皆 tile**：渲染是图 tile、路径是图 tile、重路由定位还是 tile——统一的空间分块抽象贯穿全系统。
- **精度分层**：地图 zoom 分层、路径分辨率分层、ETA 实时/历史分层——每层按需取精度，成本随之分层。
- **离线管道养在线服务**：原始道路数据与位置流都在离线侧消化成在线可查的结构。

## Anti-patterns
- **全球道路一张图跑最短路**：内存与时间双爆炸，必须 tile 化。
- **动态渲染 tile**：服务器负载巨大且无法 CDN 缓存。
- **位置更新逐条实时上报**：QPS 翻数倍，批量是标配。
- **把 tile URL 算法写死在客户端且不规划长期兼容**：客户端升级困难，改算法即历史包袱。

## Worked Example
导航 "1355 Market St SF → Disneyland"：geocoding service 把两端地址转 lat/long（Redis 查）→ route planner 调 shortest-path service：坐标推 geohash 定位起点 routing tile → A* 遍历并按需加载相邻 tile，长路段切换低分辨率 tile → 到终点 tile 得路径 → ETA service 按路况 ML 预测用时 → ranker 按"避开收费站"过滤排序 → 返回 polyline + 分段指令。行驶中：用户导航状态存 (user, 当前 tile, super tile 链)；事故发生在 tile r_x → 反查路径含 r_x 的用户 → WebSocket 推送重路由。地图显示：客户端按 zoom 计算所需 tile 的 geohash URL，从最近 CDN POP 拉取拼接渲染（或拉矢量自绘）。

## Interview Lens
1. 先过 Map 101（projection/geocoding/geohash/tiling/routing tile）——领域词汇量直接区分候选人。
2. 报估算：1B DAU、200k QPS/峰值 1M、70PB tiles。
3. 分层 routing tile + A* 是路径规划的标准深挖，讲清"为什么不整图算"。
4. adaptive rerouting 的 super tile 存储技巧是高阶加分点。
5. 推送协议对比（push/long-polling/SSE/WebSocket）展示选型广度。

## Practitioner Checklist
- [ ] 三服务边界清晰（位置/导航/渲染）
- [ ] tile 分层与 CDN 分发策略
- [ ] routing tile 离线管道 + S3 + 缓存
- [ ] Cassandra 位置存储与 Kafka 流
- [ ] geocoding 缓存层
- [ ] A* + 多级分辨率拼接
- [ ] ETA/ranker/重路由链路
- [ ] WebSocket 推送通道

## Key Takeaways
1. tiling 抽象统一渲染与路径：zoom 分层、CDN 静态分发
2. 位置流：批量上报 → Cassandra（AP）→ Kafka 下游
3. A* + 分层 routing tile 解决大规模最短路
4. adaptive ETA/reroute 用 super tile 链省存储

## Connects To
- **Ch 16**：geohash 与空间索引基础。
- **Ch 17**：位置更新的实时链路同源。
- **Ch 1**：CDN 分发原理。
- **Ch 19**：Kafka 流处理设计。
