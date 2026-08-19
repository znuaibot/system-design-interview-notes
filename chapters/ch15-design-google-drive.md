# Chapter 15: Design Google Drive

## Core Idea
云盘 = 元数据与块存储分离（metadata DB + S3 存 4MB 块）；同步用 delta sync 只传变化块，冲突"先处理者赢"并保留双方副本，通知用 pub-sub + 离线队列兜底。

## Frameworks Introduced
- **块存储模型**：文件切成 ≤4MB 块，每块哈希寻址，独立存 S3，重建按序拼接。
  - When to use：大文件存储与增量同步的基础。
  - How：上传时 block server 切块+压缩+加密 → 块入 S3 → 元数据记块序；下载时按元数据拉块重组。
- **冲突解决双规则**：先处理的版本赢（first processed wins）；冲突副本单独保存交用户合并或覆盖。
  - When to use：多设备/多用户并发写同一文件。
  - How：服务端按到达顺序接受第一个，后者收到冲突提示与两个版本。

## Key Concepts
- **需求基线**：上传/下载、多设备同步、版本历史、共享权限、编辑/删除/分享通知；可靠性零容忍丢数据、同步要快、带宽省、10M DAU、高可用。
- **容量假设**：10GB 免费空间、单文件上限 10GB、平均上传 500KB、人均 2 文件/天、总存储 500PB。
- **Namespace 模型**：drive/ 根目录下每用户一个 namespace，文件由 namespace+相对路径唯一标识（单机起点）。
- **Resumable Upload**：先取 resumable URL，分块上传并监控状态，中断可续传（simple upload 用于小文件）。
- **Block Server**：切块、压缩、加密，上传块到 S3。
- **Delta Sync**：只传修改过的块而非整个文件——带宽效率的关键。
- **Metadata DB 四表**：user / file / block / file version（版本历史）。
- **Notification Service**：pub-sub 通知客户端文件增删改，用 long polling 异步推送。
- **Offline Backup Queue**：离线客户端的变更暂存，上线后补同步。
- **Cold Storage**：不活跃文件移入廉价存储（如 S3 Glacier）降成本。
- **去重**：账户级按块哈希去重。

## Decision Rules
1. 大文件分块（≤4MB）+ 断点续传，块独立存储
2. 元数据与 blob 分开：metadata DB + cache 管索引，S3 管块
3. 通知只提示"有变化"，客户端再拉元数据和块——权威状态永远在服务器
4. 冲突先处理者赢，冲突副本保留供用户合并，绝不静默覆盖
5. 同步只传 delta 块，按文件类型选压缩算法
6. 版本数设上限，高频编辑文件保近期版本；冷文件下沉 cold storage

## Mental Models
- **通知是提示不是数据**：push 只说"变了"，真实同步永远走"拉元数据→拉块"的权威路径，这样通知丢了也能自愈。
- **块是同步与去重的公共单位**：delta sync、账户去重、并行上传都建立在同一块模型上。
- **可靠性是分层兑现的**：每类故障（LB/block server/DB/S3/通知）各有预案，而不是一个"高可用"口号。

## Anti-patterns
- **整文件重传做同步**：改一个字节传 10GB，带宽与用户耐心双输。
- **静默覆盖冲突版本**：用户数据丢失，信任崩塌。
- **通知消息携带文件内容**：消息大小失控且顺序难保证。
- **元数据与文件存一起**：无法独立扩展索引层与存储层。

## Worked Example
上传：文件到 block server 切成 4MB 块、压缩加密传 S3；客户端并行把元数据发 API server 存为 pending；S3 完成回调把状态改 uploaded，通知服务推送相关用户。同步：用户 A 在线时文件被 B 修改 → 通知服务推给 A → A 拉新元数据 → 只下载变化的块重组。A 离线时：变更进 offline backup queue，A 上线后补拉。冲突：user1 与 user2 同时改同一文件，user1 先到被接受；user2 收到冲突，系统展示服务器版与本地版两份，由 user2 合并或择一覆盖。

## Interview Lens
1. 先画单机版（web server + metadata DB + namespace 目录）再演进到分布式——标准开场。
2. 块模型三件套（切块/哈希/重组）与 delta sync 是核心深挖。
3. 冲突解决讲"先处理者赢 + 双副本保留"，展示数据安全观。
4. 五类故障预案逐个点名（LB/block/DB/S3/通知），可靠性部分直接满分。
5. 报容量数字：10M DAU、500PB、10GB/用户、500KB 均值。

## Practitioner Checklist
- [ ] 块大小与哈希方案定义
- [ ] resumable upload 支持断点续传
- [ ] metadata DB 四表齐全且与存储分离
- [ ] delta sync + 压缩生效
- [ ] 冲突双副本保留策略
- [ ] pub-sub 通知 + 离线队列兜底
- [ ] 版本上限与 cold storage 策略
- [ ] 五类故障预案就位

## Key Takeaways
1. 4MB 块 + S3 + metadata DB 是骨架，delta sync 是效率关键
2. 冲突先处理者赢，副本保留交用户
3. 通知 pub-sub（long polling）+ 离线队列保证最终同步
4. 去重、版本限制、cold storage 控成本

## Connects To
- **Ch 10**：通知系统架构复用。
- **Ch 24**：S3 类对象存储完整设计。
- **Ch 6**：metadata DB 的分片与复制。
- **Ch 27**：块级账本思想与钱包分片记账相通。
