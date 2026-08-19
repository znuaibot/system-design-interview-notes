# Chapter 26: Design a Payment System

## Core Idea
支付系统 = payment service（编排）+ payment executor（执行）+ PSP（动钱）+ ledger/wallet（记账），吞吐极低（10 TPS）但强一致与幂等是生命线；不碰卡数据（hosted page），夜结对账兜底一切不一致。

## Frameworks Introduced
- **Exactly-once = at-least-once + 幂等**：重试保证不丢（at-least-once），idempotency key 保证不重（at-most-once），两者合起来才是 exactly-once。
  - When to use：任何"重复执行会造成金钱损失"的操作。
  - How：重试策略默认指数退避（服务端可用 Retry-After 头指定间隔）；幂等用 idempotency-key 头 + DB unique 约束（插入冲突→返回已有结果）；PSP 侧用同一 nonce 不重复处理。
- **借贷记账（Double-Entry Ledger）**：每笔资金变动同时记两个账户，一借一贷，所有分录总和恒为零。
  - When to use：一切资金流动的财务记录。
  - How：buyer 记 debit $1、seller 记 credit $1；提供端到端资金可追溯。

## Key Concepts
- **规模基线**：100 万笔/天 ≈ 10 TPS——吞吐不是考点，正确性才是。
- **六角色**：payment service（收事件、编排、风控 AML 检查）、payment executor（执行单个 payment order）、PSP（Stripe/Braintree/Square，真正动钱）、card schemes（Visa/Mastercard）、ledger（财务记录）、wallet（商户余额）。
- **Payment Event vs Payment Order**：一次下单（event，checkout_id 主键）可含多个 order（payment_order_id 主键，含 seller_account/amount/currency）；order 状态机 NOT_STARTED→EXECUTING→SUCCESS/FAILED，带 ledger_updated/wallet_updated 标记。
- **Idempotency Key**：payment_order_id 转发给 PSP 做去重，防止重复扣款。
- **金额用 string 不用 double**：浮点不能精确表示货币。
- **数据库选型**：性能不重要、强一致最重要 → 传统 SQL（ACID、DBA 人才市场成熟、金融机构背书、工具生态），不选 NoSQL。
- **Hosted Payment Page**：PSP 提供的收银页 widget，商户不存卡信息、规避 PCI 重监管；流程：注册支付得 token → 用户跳转 PSP 页付款 → 重定向回商户 → PSP webhook 异步通知结果。
- **Settlement File / Reconciliation**：PSP 每晚发清算文件，内外状态比对；差异分三类——可分类可自动调、可分类需财务手工调、不可分类需人工调查；也可查 ledger 与 wallet 的内部不一致。
- **Retry Queue / Dead-Letter Queue**：可重试失败进重试队列，终态失败进死信队列供排查。
- **3D Secure / 人工风控**：支付可能耗时数小时（额外验证或人工审查），用户看 pending 状态页，完成后邮件通知。
- **同步 vs 异步通信**：同步简单但耦合、无缓冲、故障不隔离；支付副作用多（ledger/wallet/通知）→ 多接收者异步模型。
- **读主库策略**：复制延迟会让用户看到不一致 → 读写都走主库，副本只做冗余与 failover；或 Paxos/Raft 同步副本，或 YugabyteDB/CockroachDB 类共识数据库。

## Decision Rules
1. 所有创建支付请求携带 idempotency key（payment_order_id 兼作 PSP nonce）
2. 资金变化全部进 double-entry ledger，借贷恒为零
3. PSP 结果以 webhook 为准（验签、去重、容忍乱序），无 webhook 则轮询
4. 夜间 settlement 文件对账，差异分级处理
5. 卡数据一律不落地，用 hosted payment page
6. 金额 string、状态机 + 后台 job 监控 in-flight 超时告警
7. 可重试进 retry queue、终态失败进 dead-letter queue
8. 一致性：exactly-once + 对账 + 读写主库（或共识副本）

## Mental Models
- **吞吐低 ≠ 简单**：10 TPS 的系统比 10 万 QPS 的系统更难，因为每个失败路径都牵涉真金白银。
- **信任边界决定架构**：卡数据进 PSP 边界、资金事实进 ledger 边界——自己不持有的数据就不承担它的合规与风险。
- **对账是最后防线**：所有异步、重试、webhook 的不确定性，最终由"每晚把两边账本摆在一起"兜底。

## Anti-patterns
- **用 double 存金额**：0.1+0.2≠0.3，分毫不差是财务底线。
- **自己存信用卡号**：PCI 合规重负 + 泄露风险，hosted page 是行业标准答案。
- **重试不去重**：PSP 已扣款但下游没记账，重试导致二次扣款。
- **只信同步响应不接 webhook**：长支付（3D Secure）永远等不到结果。
- **读写分离给支付查询**：复制延迟下用户看到"未支付"而实际已扣款。

## Worked Example
Pay-in：用户下单（3 个卖家商品）→ POST /v1/payments（buyer_info, checkout_id, 3 个 payment_orders，各带 payment_order_id）→ payment service 存 event、风控检查 → 每个 order 交 payment executor 存库并调 PSP（order_id 作幂等 nonce）→ PSP hosted page：注册得 token 存库 → 用户跳 PSP 页填卡 → PSP 扣款成功 → 重定向回商户 + webhook 异步通知 → payment service 记录 SUCCESS → 更新 wallet（卖家余额）→ 记 ledger（buyer debit / seller credit）→ 置 ledger_updated/wallet_updated。失败：PSP 拒付 → 按状态判断可重试 → retry queue 指数退避；三次失败 → dead-letter queue 人工排查。长支付：卡触发 3D Secure → 状态 EXECUTING 挂起，用户看 pending 页 → 数小时后 webhook 到达完成。夜对账：PSP settlement 文件 vs payment_orders 表，发现一笔"我方 FAILED 但 PSP 已扣款"→ 归为可分类差异，财务按标准流程退款调账。

## Interview Lens
1. 开场六问定范围（支付方式/是否自营处理/卡数据存储/币种/规模/对账），"不存卡数据"主动说。
2. 报 10 TPS 并点破"这题考正确性不考吞吐"——展示判断力。
3. exactly-once 的拆解（at-least-once 重试 + 幂等键）与五种重试策略、默认指数退避。
4. double-entry ledger 借贷表随手画，总和为零是灵魂。
5. hosted page 全流程（token→跳转→重定向→webhook）讲清楚，合规分拿满。
6. 对账三类差异分级 + 失败支付三机制（状态/retry/DLQ）是可靠性深挖。
7. 收尾话题池：监控、汇率、地域支付方式、现金支付、Apple/Google Pay。

## Practitioner Checklist
- [ ] event/order 双表与状态机完整
- [ ] idempotency key 贯穿商户→PSP
- [ ] 金额 string、DB 选 SQL 有论证
- [ ] hosted page 流程与 webhook 处理
- [ ] ledger 借贷恒等 + wallet 更新标记
- [ ] retry queue + dead-letter queue
- [ ] 夜对账与差异分级流程
- [ ] 长支付（3D Secure）pending 体验
- [ ] 安全清单：HTTPS/令牌化/PCI/反欺诈
- [ ] 读写主库或共识副本的一致性策略

## Key Takeaways
1. exactly-once = 重试（不丢）+ 幂等键（不重）
2. double-entry ledger 借贷恒为零，资金全程可追溯
3. 卡数据不落地：hosted page + PSP webhook
4. 对账兜底一切异步不确定性；失败走 retry/DLQ 分级
5. SQL + ACID + 读写主库是支付数据层默认

## Connects To
- **Ch 27**：数字钱包把 ledger/wallet 做成核心产品。
- **Ch 22**：酒店预订的同款幂等键与退款补偿。
- **Ch 19**：retry queue/DLQ 的消息队列基础。
- **Ch 21**：计费对账与日终 reconciliation 同源。
