# Chapter 12: Design a Chat System

## Core Idea
实时聊天 = 无状态服务（登录/资料）+ 有状态 chat server（持 WebSocket 长连接）；消息顺序用 local sequence number 而非全局 ID，多设备同步靠 cur_max_message_id 游标。

## Frameworks Introduced
- **接收侧协议三选一**：Polling（轮询，冗余请求多）→ Long Polling（挂起连接等消息，不活跃用户浪费）→ WebSocket（双向持久连接，最终选择）。
  - When to use：实时通信设计题的开场论证。
  - How：发送侧 HTTP 即可，接收侧需要服务器主动推，逐个淘汰后选 WebSocket。
- **消息同步游标法**：每设备维护 cur_max_message_id，满足"收件人是当前用户 且 消息 ID > 游标"的即为新消息。
  - When to use：多设备同步与离线补拉。
  - How：设备上线时带游标查询 KV 存储，只拉增量。

## Key Concepts
- **需求基线**：单聊、群聊（≤100 人）、文本 ≤10 万字符、在线状态、多设备、推送通知、50M DAU、历史永久保存。
- **Stateless Services**：注册/登录/资料管理，结合 service discovery 推荐最优 chat server。
- **Stateful Services**：chat server 维持 WebSocket 长连接，负责投递与同步——有状态是扩展难点。
- **KV Store 存聊天记录**：易水平扩展、低延迟、避免关系库大索引下随机访问昂贵；Facebook Messenger 与 Discord 均如此。
- **Local Sequence Number**：消息 ID 只需在单聊/群内唯一有序，不必全局唯一——比全局 Snowflake 更省、更对。
- **群聊主键**：(channel_id, message_id) 复合键。
- **Message Sync Queue（inbox）**：每个接收者一个收件箱，汇集多个发送者的消息；群聊消息复制到每个成员 inbox。
- **Service Discovery（Zookeeper）**：按地理位置与服务器容量推荐最优 chat server，故障时重新分配。
- **Heartbeat**：客户端周期性向 presence server 发心跳，30 秒无心跳判离线。
- **Presence Fanout（pub-sub）**：每对好友一个 channel，状态变化发布到所有相关 channel——小群组有效，大群组需另策。
- **cur_max_message_id**：设备本地游标，同步只拉游标之后的消息。

## Decision Rules
1. 接收侧用 WebSocket（polling/long polling 逐一淘汰）
2. 消息顺序用 local sequence number（会话内有序即可），全局 ID 是过度设计
3. 离线用户走推送通知，上线后按游标补拉
4. 群聊消息复制到每个成员 inbox——100 人群可接受，更大群需读扩散
5. chat server 有状态，用 service discovery 管理分配与故障迁移
6. 聊天记录存 KV 存储而非关系库

## Mental Models
- **有状态与无状态分开治理**：无状态层随便扩，有状态层靠连接管理和再分配，混在一起就乱。
- **顺序是局部的**：聊天里"谁先说"只在会话内有意义，全局顺序是无谓成本。
- **游标代替全量**：增量同步的本质是每个设备记住"我读到哪里了"。

## Anti-patterns
- **用全局自增 ID 定消息顺序**：中心化发号成瓶颈，且跨会话顺序无意义。
- **polling 接收消息**：海量空请求浪费带宽与服务器。
- **presence 对大 V 做 per-friend 扇出**：百万好友的状态变化扇出百万事件。
- **关系库存聊天历史**：长尾数据索引膨胀，随机读昂贵。

## Worked Example
A→B 单聊：A 发消息到其 chat server 1 → server 分配 local message ID 存 KV → 查 B 在线（presence）→ B 挂在 chat server 2，经内部通道转发到 B 的 WebSocket → B 更新游标。若 B 离线：走推送通知；B 上线时带 cur_max_message_id 拉增量。群聊（50 人）：消息写入后复制到 50 个成员 inbox，各自设备按游标同步。在线状态：客户端每 N 秒心跳，30 秒失联标离线，变化经 pub-sub channel 推给好友。

## Interview Lens
1. polling→long polling→WebSocket 的淘汰论证是开场标准动作。
2. local vs global sequence number 的取舍是本章最有区分度的点，主动讲。
3. KV 存储的四个理由（扩展/延迟/长尾/业界实践）要完整。
4. 多设备同步用 cur_max_message_id 游标讲，一句话到位。
5. 主动提 50M DAU 下的有状态服务器管理与 service discovery。

## Practitioner Checklist
- [ ] 收发协议选择有淘汰论证
- [ ] 消息 ID 用 local sequence number
- [ ] KV 存储选型有理由
- [ ] 多设备同步游标机制实现
- [ ] 离线推送 + 上线补拉链路完整
- [ ] 心跳阈值与 presence 扇出范围定义
- [ ] chat server 故障时有再分配机制

## Key Takeaways
1. WebSocket 双向长连接是实时聊天的默认协议
2. local sequence number + (channel_id, message_id) 复合键解决排序
3. cur_max_message_id 游标实现多设备增量同步
4. 心跳 + pub-sub 维护在线状态；KV 存储承载历史

## Connects To
- **Ch 10**：离线推送复用通知系统架构。
- **Ch 6**：KV 存储设计细节。
- **Ch 7**：若用全局 ID 则回到 Snowflake。
- **Ch 19**：消息同步队列的完整 MQ 设计。
