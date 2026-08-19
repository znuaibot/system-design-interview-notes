# Chapter 24: Design S3-like Object Storage

## Core Idea
对象存储 = 元数据与数据分离（类 UNIX inode 思想）+ 对象不可变写一次读多次；数据节点用 WAL 合并小文件、SQLite 本地映射表，持久性靠跨故障域三副本或 8+4 纠删码，列表查询用反范式表。

## Frameworks Introduced
- **三类存储对比**：Block（可变、贵、快、VM/数据库）/ File（层级目录、中成本）/ Object（不可变、扁平、便宜、海量冷数据、REST 访问）。
  - When to use：开场定位"为什么选对象存储"。
  - How：对象存储牺牲性能换持久性、规模与成本；S3 标准 SLA 11 个 9 持久性、99.9% 可用。
- **复制 vs 纠删码**：3 副本 = 200% 开销、6 个 9 持久、读快实现简单；8+4 纠删码 = 50% 开销、11 个 9 持久、读要聚多点、计算与实现复杂。
  - When to use：持久性方案的深挖权衡。
  - How：延迟敏感选复制，成本与持久性优先选纠删码（跨故障域分布 12 片）。

## Key Concepts
- **规模基线**：100PB/年、6 个 9 持久、4 个 9 可用；对象分布 20% 小(<1MB)/60% 中(1–64MB)/20% 大(>64MB)；按中位估算 ~6.8 亿对象、元数据 1KB/对象≈0.68TB；HDD 100–150 IOPS 是瓶颈约束。
- **术语**：Bucket（全局唯一名的逻辑容器）、Object（数据+元数据）、Versioning、URI、SLA。
- **对象不可变**：只有删除与替换，没有更新；写一次读多次（LinkedIn 研究 95% 操作是读）。
- **五组件**：LB → API service（无状态编排 + IAM 鉴权）→ metadata store + data store。
- **Placement Service**：维护虚拟集群图决定对象落哪些数据节点，心跳摘除故障节点；关键服务跑 5/7 副本用 Paxos/Raft 共识（7 节点容忍 3 挂）。
- **Data Routing Service**：无状态路由读写到数据节点，水平扩展。
- **持久化流**：对象按 UUID 一致性哈希定复制组 → 主节点本地存 + 复制 2 个副节点后才返回（强一致换延迟）。
- **WAL 合并小文件**：每对象一文件会浪费 4KB 块与 inode，改为追加写大文件（几 GB 一卷）；写串行化问题用"文件绑定特定核"规避锁竞争。
- **对象映射表**：object_id → filename + file_offset + object_size；低写高读，SQLite 直接部在数据节点内（避免独立 DB 集群的网络延迟与扩展压力）。
- **元数据分片**：bucket 表小可单库；object 表按 hash(bucket_name, object_name) 分片（查询都按名字；按 bucket_id 分会热点）。
- **列表优化**：分片后前缀列表要扫所有 shard 且分页困难 → 反范式列表表按 bucket ID 分片，牺牲写换列表速度（对象存储列表本就不求快）。
- **版本控制**：object_version 用 TIMEUUID 可排序；每新版本新 object_id；删除 = 写一个特殊删除标记版本，查询返回 404。
- **Multipart Upload**：发起得 upload_id → 分块独立上传各返 etag（MD5）→ complete 请求带齐 part 号与 etags → 重组；垃圾块由 GC 清理。
- **Garbage Collection + Compaction**：垃圾来源 = 懒删除/孤儿上传块/校验损坏数据；GC 把存活对象从旧文件拷到新文件，事务更新映射表，超阈值文件才做 compaction。
- **Checksums**：每文件每对象存校验和，检测静默损坏；纠删码要逐片取回逐个验证。

## Decision Rules
1. 大对象用 multipart upload 并行上传 + 断点续传
2. 持久性：跨故障域 3 副本（HDD 年故障率 0.81%，三副本达 6 个 9）或 8+4 纠删码（50% 开销、11 个 9）
3. 小文件 WAL 合并成大卷，文件绑核避免写锁竞争
4. 对象映射表用 SQLite 内置数据节点，不建独立 DB 集群
5. object 表按 hash(bucket_name, object_name) 分片；列表用反范式表
6. 删除用懒删除 + GC compaction，不做同步物理删
7. 每对象/文件 checksum，后台校验防静默损坏

## Mental Models
- **inode 思想**：元数据（指针）与内容（块）分离，各自独立扩展——这是所有存储系统的第一性设计。
- **不可变是简化的礼物**：没有更新就没有并发写冲突、没有部分写、GC 只需标记不需锁。
- **持久性是概率题**：单盘故障率 × 副本独立性 → 目标 9 的个数；故障域隔离保证"独立"成立。

## Anti-patterns
- **每对象一文件**：inode 爆炸 + 块浪费，海量小文件文件系统直接崩。
- **副本全放同一机架**：一次断电全丢，故障域隔离是 6 个 9 的前提。
- **按 bucket_id 分片对象表**：大桶成热点。
- **分片后硬做跨片前缀列表 + 分页**：每 shard 独立 offset 噩梦，应反范式。
- **同步物理删除**：阻塞请求且副本清理复杂，懒删除 + GC 是正解。

## Worked Example
上传 script.txt（4.5KB）到 bucket-to-share：PUT /bucket-to-share/script.txt 带鉴权头 → LB → API service 查 IAM 权限 → 数据路由问 placement 得复制组（一致性哈希 UUID）→ 主数据节点追加到当前卷文件 /data/c → 复制 2 副本成功 → SQLite 插映射记录 (object_id, /data/c, offset, size) → 元数据库写 (object_id, bucket_id, name…) → 返回。下载 GET：IAM 校验 → 元数据查 UUID → 数据节点按映射表定位卷内 offset 读出。大文件 1GB：multipart 切 100 块并行传各得 etag，complete 重组；中途失败的块由 GC 回收。桶内列表：查反范式列表表（按 bucket ID 分片，单实例完成）。版本：同名覆盖生成新 object_id 新版本；删除写删除标记，后续 GET 返 404，真实数据 GC 回收并跨副本/12 纠删片清理。

## Interview Lens
1. 开场三类存储对比表 + 对象存储四特性（不可变/KV/写一次读多次/大小通吃）。
2. 报估算：6.8 亿对象、0.68TB 元数据、IOPS 瓶颈意识。
3. 数据节点内部设计（WAL 合并 + 绑核 + SQLite 映射）是最细的深挖点。
4. 复制 vs 纠删码对比（开销/持久性/读性能/复杂度）必须量化。
5. 分片键选择（hash(bucket_name, object_name)）与列表反范式展示查询驱动建模。
6. GC + compaction + checksum 讲全，运维视角加分。

## Practitioner Checklist
- [ ] 元数据/数据分离与独立扩展
- [ ] placement 服务 Raft 多副本
- [ ] WAL 卷文件 + 绑核 + SQLite 映射
- [ ] 跨故障域复制或纠删码选型
- [ ] checksum 与后台校验
- [ ] hash(bucket_name, object_name) 分片
- [ ] 反范式列表表
- [ ] TIMEUUID 版本 + 删除标记
- [ ] multipart upload + GC compaction

## Key Takeaways
1. 元数据与数据分离、对象不可变是对象存储两大基石
2. 小文件合并成卷 + 本地映射表解决海量对象存储
3. 三副本 vs 8+4 纠删码：200%/50% 开销，6/11 个 9
4. 列表靠反范式，版本靠 TIMEUUID，空间靠 GC compaction

## Connects To
- **Ch 15**：Google Drive 块存储建立在同类对象存储上。
- **Ch 14**：视频转码产物与原始片存储。
- **Ch 5**：复制组的一致性哈希选择。
- **Ch 19**：WAL 分段思想同源。
