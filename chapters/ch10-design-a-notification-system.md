# Chapter 10: Design a Notification System

## Core Idea
通知系统 = 多通道（push/SMS/email）软实时投递；演进关键是拆掉单体 notification server——数据库外置、多实例水平扩展、消息队列解耦、worker 池对接第三方供应商。

## Frameworks Introduced
- **初版→改进版演进路径**：单体 server（SPOF、难扩展、性能瓶颈）→ 拆出 DB/cache、多 server 水平扩展、MQ 缓冲、独立 worker。
  - When to use：任何"先画简单版再主动演进"的设计题，这是标准示范。
  - How：先画 trigger→server→3rd party 的直线流程，主动指出三个缺陷，再逐个用组件解决。

## Key Concepts
- **三通道供应商栈**：iOS 推送走 APNS，Android 走 FCM，SMS 用 Twilio/Nexmo，邮件用 SendGrid/Mailchimp——全部第三方，不自建。
- **Contact Info Gathering**：安装/注册时收集 device token、手机号、邮箱；device tokens 表存推送令牌，user 表存邮件与电话。
- **Trigger Services**：产生通知事件的源头——微服务、cron job 或分布式系统（如账单提醒、物流更新）。
- **Notification Server**：提供发送 API、做基础校验（邮箱/手机号格式）、查库/缓存取渲染数据。
- **Message Queues**：解耦各组件并在高峰时充当缓冲；每通道独立队列。
- **Workers**：从队列拉事件、调用对应第三方服务投递。
- **Notification Log DB**：持久化通知数据防丢失，支撑重试。
- **Deduplication**：按 event ID 判重，见过的直接丢弃。
- **Notification Templates**：预格式化模板保证一致性与发送效率。
- **Notification Settings**：用户对每通道 opt-in/opt-out，独立设置表。
- **Rate Limiting**：限制对单用户的通知频率。
- **AppKey/AppSecret**：推送 API 的鉴权安全机制。

## Decision Rules
1. 每个通道独立队列与 worker，隔离故障与限速差异
2. 发送前检查退订设置、频控和设备令牌有效性
3. 第三方失败要重试 + 退避，永久失败进死信队列
4. 事件先落 notification log 再进队列，保证不丢
5. event ID 去重在入队时做，不在发送时做
6. 监控队列积压动态扩 worker

## Mental Models
- **第三方是能力边界**：推送/短信/邮件都外包，系统的职责是路由、编排与可靠性，不是传输本身。
- **队列是缓冲不是管道**：高峰洪峰时队列吸收尖峰，worker 按自己的速度消费。
- **通知是有状态的**：每条通知有 accepted/delivered/failed 生命周期，可追踪、可重试、可统计打开率。

## Anti-patterns
- **业务服务直接调短信/推送供应商**：每个业务各存密钥、各做重试，重复且无法统一频控。
- **重试不幂等**：第三方超时重发导致用户收到重复通知——必须按 event ID 去重。
- **忽视无效 token 和退订状态**：向已卸载设备反复推送被供应商降权，向退订用户发送违反合规。
- **单一 notification server**：一挂全挂，且无法独立扩展校验与发送。

## Worked Example
规模：push 1000 万/天、SMS 100 万/天、邮件 500 万/天，软实时。订单发货触发流程：订单服务调 notification server API → 校验并查设置表（用户关了 SMS 则跳过该通道）→ 生成 event（ID=evt123）写 log 库 → 推入 push 队列与 email 队列 → worker 拉取，查模板渲染，按幂等键调 FCM/SendGrid → 记录 delivered/failed；evt123 再次出现时入队即丢弃。FCM 瞬时失败指数退避重试 3 次，仍失败进死信队列人工排查。

## Interview Lens
1. 先画单体版再主动说出三个缺陷（SPOF/扩展性/性能）——展示演进思维。
2. 三通道对应的供应商名字（APNS/FCM/Twilio/SendGrid）要脱口而出。
3. 可靠性两件套（log 持久化 + event ID 去重）是深挖得分点。
4. 主动补 rate limiting、模板、设置表、监控，展示产品级完整性。

## Practitioner Checklist
- [ ] 三通道队列与 worker 隔离
- [ ] device token/联系方式收集链路完整
- [ ] opt-out 设置在发送前检查
- [ ] 通知落库后再进队列
- [ ] event ID 去重生效
- [ ] 重试、退避、死信策略定义
- [ ] AppKey/AppSecret 鉴权就位

## Key Takeaways
1. 水平扩展 + MQ 解耦 + worker 池是标准架构
2. 防丢（log 持久化）与去重（event ID）是可靠性双支柱
3. 模板、设置、频控、监控让系统产品化
4. 第三方供应商承担最后一公里投递

## Connects To
- **Ch 19**：消息队列的完整设计。
- **Ch 12**：聊天系统的离线推送复用本架构。
- **Ch 4**：对用户的频控即 rate limiter 应用。
- **Ch 23**：邮件服务的发送侧细节。
