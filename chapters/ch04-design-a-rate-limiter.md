# Chapter 4: Design a Rate Limiter

## Core Idea
限流是对单位时间内资源消耗实施策略；算法选择取决于突发容忍度、精度、状态成本和分布式一致性，放置位置取决于控制点在哪一层。

## Frameworks Introduced
- **放置位置三分法**：client-side（不可信、难集中管理）、server-side/API gateway（集中控制点，最常用）、middleware 层（与业务逻辑解耦）。
  - When to use：限流器设计题的第一个架构决策。
  - How：先问"谁是执行者、状态放哪、失败谁兜底"，再选位置；生产系统通常放网关并用 Redis 存计数。
- **算法五选一矩阵**：按突发容忍、精度、内存成本、实现复杂度四个维度在五种算法间选择。

## Key Concepts
- **Benefits of Rate Limiting**：防恶意滥用与 DDoS、控制成本（API 按次计费）、保护下游不被流量打垮。
- **Token Bucket**：桶按固定速率补充 token，请求消耗 1 token，桶空即拒；桶容量决定允许的突发量。实现简单、内存小，业界默认选择（Amazon/Stripe 均用）。
- **Leaky Bucket**：请求入队，以恒定速率出队处理；输出绝对平滑但突发会被排队延迟，队列满则丢弃。
- **Fixed Window Counter**：固定时间窗内计数，超阈值拒绝；问题是窗口边界处两个窗口的尾部+头部可叠加出 2 倍突发。
- **Sliding Window Log**：为每个用户记录完整请求时间戳，统计窗口内数量；精度最高但内存与计算成本最大。
- **Sliding Window Counter**：当前窗口计数 + 上一窗口计数按重叠比例加权；近似精确且内存友好，是精度与成本的折中。
- **Redis**：分布式限流的共享计数存储，用 INCR/EXPIRE 或 Lua 脚本保证原子更新。
- **429 Too Many Requests**：限流触发时返回的标准状态码，应携带剩余额度与重试时间。
- **Rate Limiting Rules**：按用户/IP/API endpoint/订阅层级定义的不同策略。

## Decision Rules
1. 需要允许受控突发 → Token Bucket
2. 需要绝对平滑的输出 → Leaky Bucket
3. 实现要最简单、容忍边界误差 → Fixed Window Counter
4. 精度要求极高且规模小 → Sliding Window Log
5. 精度与内存折中（大规模默认）→ Sliding Window Counter
6. 分布式部署 → 计数放 Redis，用原子操作（Lua 脚本）避免竞态
7. 限流规则按 endpoint + 用户层级分别配置，不搞全局一个阈值

## Mental Models
- **限流是策略不是组件**：先定义"保护什么、允许什么、拒绝后怎么办"，算法只是实现。
- **精度是有价格的**：从 Fixed Window 到 Sliding Log 是一条内存/计算成本递增的精度光谱。
- **边界条件是算法的试金石**：评估任何计数算法先问窗口切换瞬间会发生什么。

## Anti-patterns
- **Fixed Window 不设边界保护**：窗口切换瞬间可放行 2 倍流量，打垮下游。
- **Sliding Window Log 用于大规模**：每请求一条时间戳，内存随用户数线性爆炸。
- **多节点各自本地计数**：每台机器独立限流，N 台机器等于阈值放大 N 倍，策略被绕过。
- **限流后不返回重试信息**：客户端盲目重试加剧风暴。

## Worked Example
为每用户每分钟 100 次 API 请求配置 Token Bucket：桶容量 100（允许瞬时 100 连发），补充速率 100/60≈1.67 token/s。请求到达时原子扣 token（Redis Lua 脚本），不足则返回 429 + Retry-After。演练三个故障：热点用户狂刷（桶被耗尽，稳定在 1.67 req/s）、Redis 故障（fail open 还是 fail closed 需按业务定）、时钟偏差（用服务端时钟统一计算）。

## Interview Lens
1. 开场先澄清"限谁、限什么、精度要求、分布式与否"——这四问决定后面一切。
2. 五种算法全报一遍名字，再选一个深入，展示知识宽度。
3. 主动画出 Fixed Window 的边界双倍突发场景，这是最有区分度的细节。
4. 分布式场景主动提出 Redis + 原子操作和一致性代价。

## Practitioner Checklist
- [ ] 限流维度明确（用户/IP/endpoint/层级）
- [ ] 算法选择有四个维度的论证
- [ ] 分布式计数有原子性方案
- [ ] 拒绝响应带 429 + 重试信息
- [ ] 有限流命中率、误杀率监控
- [ ] 依赖存储故障时的降级策略已定

## Key Takeaways
1. Token Bucket 是允许受控突发的默认选择
2. Sliding Window Log 精度最高但内存最贵，Sliding Window Counter 是大规模折中
3. 分布式限流的正确性依赖共享存储 + 原子更新
4. 限流规则、位置、算法是三个独立决策，按顺序做

## Connects To
- **Ch 3**：本题是四步框架的标准实例。
- **Ch 6**：Redis 作为共享状态存储的一致性问题。
- **Ch 10/12**：通知与消息系统中队列入口同样需要限流。
- **patterns.md**：背压与入口准入模式。
