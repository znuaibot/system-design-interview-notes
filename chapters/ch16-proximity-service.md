# Chapter 16: Design Proximity Service

## Core Idea
附近商家搜索 = LBS（只读位置查询）+ Business Service（商家 CRUD 与详情）；空间索引五选一（2D 暴力/均匀网格/Geohash/Quadtree/S2），静态 POI 场景默认 Geohash + Redis 双层缓存。

## Frameworks Introduced
- **空间索引选型矩阵**：按实现难度、动态密度适应、查询类型、更新成本比较五种方案。
  - When to use：任何"二维数据要用一维索引"的位置服务。
  - How：Geohash（易实现、固定网格、更新易、不支持 k-nearest）vs Quadtree（随密度自适应、支持 k-nearest、更新可能要重建整树）vs S2（Hilbert 曲线球面 cell、min/max level + max cells、geofencing 强但最难）。
- **边界问题 + 邻居格查询**：geohash 前缀不保证空间邻近（赤道两侧可能零前缀共享），所以半径查询必须查本格 + 周围 8 邻格。
  - When to use：任何基于网格前缀的邻近检索。
  - How：算出中心 geohash，展开 9 格并行查，再按真实距离过滤排序。

## Key Concepts
- **规模基线**：100M DAU、200M 商家、人均 5 次搜索/天 → 搜索 QPS≈5,000；商家变更非实时（日批处理）。
- **两服务拆分**：LBS 只读、无状态、高峰 QPS 高；Business Service 管商家 CRUD 与详情。
- **Geohash**：经纬度交替位编码成 12 级精度的字母数字串；前缀越长越近；按半径查表选最短够用的长度（如 500m→长度 6）。
- **Quadtree**：内存树结构，递归四分直到每节点 ≤100 商家；200M 商家建树 nlogn 约几分钟、占 GB 级内存；适合 k-nearest；更新需增量重建（缓存失效多）或加锁在线更新。
- **Google S2**：Hilbert 曲线把球面映射为一维索引，可指定 min/max level 与 max cells，覆盖任意形状区域，是 geofencing 首选。
- **主从 DB 架构**：读多写少用 primary-replica；商家信息非实时更新，主从少量延迟可接受。
- **缓存键用 geohash 不用坐标**：GPS 坐标不准且用户移动导致命中率低；geohash→商家 ID 列表、business_id→详情两个缓存层。
- **Quadtree 运维**：启动建树期间不能服务流量，新版本要灰度发布到部分服务器。

## Decision Rules
1. 静态/低频更新 POI → Geohash（实现简单、更新容易、支持固定半径）
2. 密度高度不均且需要 k-nearest → Quadtree
3. 需要球面几何、任意围栏形状、层级 cell 精度控制 → S2
4. 候选召回后必须按真实球面距离过滤排序（网格只是粗筛）
5. business 表按 business_id 分片；geohash 索引表单机 + 读副本即可（放得下）
6. 缓存键选 geohash 与 business_id，不选原始坐标

## Mental Models
- **二维问题降一维**：所有空间索引的本质都是"把位置编码成一维可排序/可哈希的键"，区别只在编码方式。
- **前缀邻近 ≠ 空间邻近**：网格编码在边界处必然撒谎，邻居格查询是标配补丁。
- **读路径分层缓存**：geohash→ID 列表挡掉大部分请求，ID→详情缓存挡掉剩余，DB 是兜底。

## Anti-patterns
- **二维范围 SQL 扫描**：lat/long 各一维索引无法联合高效过滤，全表扫。
- **只查本格不查邻格**：边界附近的商家被漏掉。
- **用原始坐标做缓存键**：GPS 抖动与移动让命中率趋近零。
- **Quadtree 在线更新无锁**：并发修改破坏树结构。

## Worked Example
用户在旧金山搜 500m 内餐厅：客户端发 (37.776720, -122.416730, 500m) → LB → LBS 查表得 500m 对应 geohash 长度 6 → 算出本格 + 8 邻格共 9 个 geohash → 并行查 Geohash Redis 取各格商家 ID → 并行查 Business Info Redis 取详情 → 按真实距离排序返回。后台：商家日批量更新写入 primary DB 并重建索引；business 表按 business_id 分片，geohash 表单机 + 读副本。

## Interview Lens
1. 先报估算（100M DAU、5k QPS、200M 商家），再进 API 与数据模型。
2. 五方案对比矩阵是本章灵魂——Geohash/Quadtree/S2 的 pros/cons 要脱口而出。
3. 主动讲 geohash 边界问题与邻居格方案，这是深挖必问。
4. 缓存键选 geohash 而非坐标的论证，展示工程直觉。
5. 结尾完整走查"500m 餐厅"请求路径（geohash 长度→9 格→双 Redis→排序）。

## Practitioner Checklist
- [ ] 空间索引选型有矩阵论证
- [ ] 邻居格查询覆盖边界
- [ ] 候选按真实距离过滤排序
- [ ] 双层 Redis 缓存（geohash→IDs、ID→详情）
- [ ] business 表分片、索引表读副本
- [ ] Quadtree 灰度发布与更新策略（如选用）
- [ ] GDPR/CCPA 合规与高峰可用性

## Key Takeaways
1. Geohash 是静态 POI 的默认索引：易用、固定半径、更新简单
2. Quadtree 换来自适应密度与 k-nearest，代价是更新复杂
3. S2 是球面 geofencing 的最强方案
4. 邻居格 + 真实距离过滤是所有网格方案的标配

## Connects To
- **Ch 17**：nearby friends 是"位置持续变化"的姊妹问题，geohash 同样用于随机附近人频道。
- **Ch 18**：Google Maps 的完整位置系统。
- **Ch 5**：redis 集群用一致性哈希分 channel（见 ch17）。
- **patterns.md**：空间索引模式。
