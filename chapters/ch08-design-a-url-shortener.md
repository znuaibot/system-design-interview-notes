# Chapter 8: Design a URL Shortener

## Core Idea
短链系统 = 低频创建（写）+ 高吞吐重定向（读 10:1）；核心决策是短码生成方式（Base62 vs 哈希+冲突解决）、重定向语义（301 vs 302）和读路径缓存。

## Frameworks Introduced
- **两种短码生成对比框架**：Base62(ID) vs Hash+Collision Resolution，按四个维度选择。
  - When to use：短链设计题的必考深挖点。
  - How：比较 ① 长度是否固定 ② 是否需要独立 ID 生成器 ③ 是否有碰撞 ④ 能否预测下一个短码（安全性），再按业务选。

## Key Concepts
- **需求基线**：1 亿次/天生成、10 年支持、读写比 10:1、365B 条约 365TB 存储、短码唯一且尽量短。
- **API 设计**：POST api/v1/data/shorten（longUrl→shortURL）；GET api/v1/shortUrl（→longURL 重定向）。
- **301 Redirect**：永久跳转，浏览器缓存结果，后续请求不再经过短链服务——省服务流量但丢失统计。
- **302 Redirect**：临时跳转，每次都经过服务——可做点击统计与动态控制，但服务承担全部流量。
- **Base62 Conversion**：用 [0-9a-zA-Z] 共 62 字符编码自增 ID；7 位短码空间 62^7≈3.5 万亿，远超 3650 亿需求；示例 ID 2009215674938 → zn9edcu。
- **Hash + Collision Resolution**：CRC32/MD5/SHA-1 取前 7 字符；碰撞时拼接预定义串递归重哈希（昂贵），用 Bloom Filter 加速存在性检查。
- **Data Model**：关系表 (id, shortURL, longURL)，shortURL 短小节省内存与索引。
- **Bloom Filter**：碰撞检测时快速判断短码是否已存在，避免全表查询。
- **Shortening Flow**：先查 longURL 是否已存在（存在则复用短码）→ 否则分布式 ID 生成器发号 → Base62 转换 → 存映射。
- **Redirecting Flow**：先查缓存 → 未命中查库 → 回填缓存 → 302/301 跳转。

## Decision Rules
1. 要统计点击或动态控制 → 302；纯省流量且不需统计 → 301
2. 默认选 Base62：无碰撞、实现简单；接受长度随 ID 增长
3. 需要固定长度且能承担碰撞处理成本 → 哈希 + bloom filter
4. 读多写少（10:1）→ 缓存 shortURL→longURL 映射，热点链接长 TTL
5. 相同 longURL 复用已有短码，避免重复记录
6. web 层无状态水平扩展，数据层复制 + 分片

## Mental Models
- **短码空间是容量规划的核心**：62^7≈3.5 万亿这个数字要能现场算出来，它决定短码长度。
- **301/302 是流量模型的开关**：选哪个直接决定服务要不要承担重定向流量，进而决定容量设计。
- **安全性藏在可预测性里**：Base62 顺序 ID 可被枚举扫描，需要随机化或鉴权保护。

## Anti-patterns
- **截断哈希不处理碰撞**：两个长链指向同一短码，用户被导向错误页面。
- **无 rate limiter**：恶意用户批量创建短链或扫描枚举短码。
- **未定义过期、删除和别名字义**：存储无限增长，删除后短码复用语义混乱。
- **用 301 却想统计点击**：浏览器缓存后统计必然失真。

## Worked Example
创建流程：POST 收到 longURL → 查库已存在则直接返回旧短码 → 否则 snowflake 发号 ID=2009215674938 → Base62 得 zn9edcu → 存 (id, zn9edcu, longURL) → 预热缓存。重定向：GET /zn9edcu → 缓存命中直接 302 → 否则查库回填。容量核对：日增 1 亿条、读写 10:1 → 读 QPS≈11,500（峰值×2≈23,000）、写 QPS≈1,150；10 年 365B 条、365TB，数据层按 shortURL 哈希分片。

## Interview Lens
1. 先报需求数字（1 亿/天、10:1、365TB），再进 API 设计——估算即开场。
2. 301 vs 302 必须讲出对流量模型和统计的影响，不是背状态码定义。
3. Base62 vs 哈希的四维对比表是本章最有区分度的输出。
4. 主动提 rate limiter、防枚举、过期策略，展示工程完整性。

## Practitioner Checklist
- [ ] 短码长度有空间论证（62^7 vs 需求）
- [ ] 301/302 选择与统计需求一致
- [ ] 读写路径都有缓存层
- [ ] 碰撞或冲突有明确处理（Base62 天然无碰撞）
- [ ] rate limiter 与防扫描就位
- [ ] 过期/删除/复用语义已定义

## Key Takeaways
1. Base62 编码自增 ID 是无碰撞的默认方案，7 位支持 3.5 万亿 URL
2. 301 省流量丢统计，302 保统计扛流量
3. 读写比 10:1 决定缓存是读路径核心
4. 哈希方案必须配碰撞解决与 bloom filter

## Connects To
- **Ch 7**：Base62 方案的 ID 来源。
- **Ch 4**：rate limiter 防滥用。
- **Ch 6**：bloom filter 在读路径的应用。
- **Ch 9**：crawler 抓取也会用到 URL 归一化与去重。
