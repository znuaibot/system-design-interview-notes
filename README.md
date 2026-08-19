# System Design Interview Notes

一套基于《System Design Interview》（Alex Xu，Vol. 1 & 2）两卷笔记合成的**系统设计知识型 WorkBuddy 技能**。把面试与实战中最常考的 28 个设计案例工程化为可检索、标准化、双视角的知识库，而不是一份堆砌的大文档。

- **双视角设计**：同一份知识同时服务「真做系统的人」（Practitioner Checklist / 落地清单）与「准备面试的人」（Interview Lens / 话术节奏）。
- **轻入口 + 深知识**：`SKILL.md` 只做导航与索引，详细内容下沉到 `chapters/`、`patterns.md`、`cheatsheet.md`、`glossary.md`，按需加载。

> 本技能是笔记合成物，非原书替代。容量数字为练习基线，不构成任何金融 / 合规建议。

## 关联地址

- **SkillHub 技能页**：https://www.skillhub.cn/skills/user_38816413/system-design-interview-notes-nuaib
- **GitHub 源码仓库**：https://github.com/znuaibot/system-design-interview-notes

## 核心资产

| 文件 | 作用 |
| --- | --- |
| `SKILL.md` | 薄入口：触发条件 + Fast Routing + Chapter Index + Topic Index |
| `chapters/` | 28 个标准化章节，每章统一模板（Core Idea → Frameworks → Key Concepts → Decision Rules → Mental Models → Anti-patterns → Worked Example → Interview Lens → Checklist → Key Takeaways → Connects To） |
| `patterns.md` | 21 个跨章节可复用架构模式（Outbox、Fanout Hybrid、TTL as State 等），标注「哪些章节是同一思想的实例」 |
| `cheatsheet.md` | 45 分钟时间盒、Decision Rules、Smells（反模式清单）速查 |
| `glossary.md` | 110+ 条术语，每条带章节链接 |

## 章节索引（28 章）

**基础与方法论**
- ch01 从零扩展到百万级用户
- ch02 信封背面估算（Back-of-the-envelope）
- ch03 系统设计面试框架

**核心组件与算法**
- ch04 限流器 · ch05 一致性哈希 · ch06 KV 存储 · ch07 分布式唯一 ID 生成器 · ch13 搜索自动补全

**经典系统设计案例**
- ch08 短链服务 · ch09 网络爬虫 · ch10 通知系统 · ch11 新闻推送 · ch12 聊天系统
- ch14 YouTube 视频流 · ch15 Google Drive · ch16 邻近服务 · ch17 附近好友 · ch18 Google Maps

**数据、基础设施与分布式**
- ch19 分布式消息队列 · ch20 指标监控告警 · ch21 广告点击聚合 · ch24 S3 类对象存储

**业务系统**
- ch22 酒店预订 · ch23 分布式邮件服务 · ch25 实时游戏排行榜 · ch26 支付系统 · ch27 数字钱包 · ch28 股票交易所

## 安装方式

**方式一：本地放入 WorkBuddy 技能目录**（推荐，零依赖）
```bash
git clone https://github.com/znuaibot/system-design-interview-notes.git
# 将克隆出的目录整体复制到 ~/.workbuddy/skills/ 下，重启 WorkBuddy 即可被加载
```

**方式二：通过 SkillHub CLI 安装**
```bash
skillhub install system-design-interview-notes-nuaib
```

## 使用方式

技能加载后，agent 会根据你的提问自动路由：
- 按任务（Fast Routing）：如「高吞吐数据平台」→ 先读 ch19；「支付 / 金额一致性」→ ch26。
- 按主题（Topic Index）：如「CAP 与一致性」→ ch06、ch19、ch21、ch22、ch26、ch27。
- 按章节（Chapter Index）：直接读取 `chapters/` 下对应文件。

典型触发语：「帮我设计一个短链服务」「系统设计面试怎么准备」「一致性哈希怎么选」「支付系统如何保证不重复扣款」。

## 许可证

MIT
