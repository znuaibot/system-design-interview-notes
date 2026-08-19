# Chapter 14: Design YouTube

## Core Idea
视频系统 = 上传流（blob 存储 → DAG 转码 → CDN 分发）+ 播放流（CDN 边缘直出）；转码用 DAG 模型并行化，成本靠"热门走 CDN、冷门按需编码"控制。

## Frameworks Introduced
- **DAG 转码模型**：把转码拆成有向无环图任务（编码、缩略图、水印），高并行执行。
  - When to use：任何计算昂贵、可分解、有依赖关系的媒体处理。
  - How：Preprocessor 按 GOP 切块 → DAG Scheduler 分阶段入队 → Resource Manager 三队列调度 → Task Workers 执行 → 中间产物落临时存储供重试。
- **热度分层分发**：热门视频放 CDN，冷门视频放大容量源站、按需编码。
  - When to use：CDN 成本主导时（示例：5M DAU×5 视频×0.3GB×$0.02≈$150k/天）。
  - How：按播放量分层存储与分发，区域化热点内容。

## Key Concepts
- **规模基线（2020）**：20 亿 MAU、日播 50 亿次、占移动互联网流量 37%、80 种语言；设计假设 5M DAU、均 300MB/视频、上限 1GB、日存储 150TB。
- **上传并行双流**：视频本体传 original storage（blob），元数据并行写 metadata DB。
- **Transcoding Pipeline**：原片 → 转码服务器（多分辨率多格式）→ 完成后并行：转码片入 transcoded storage + 完成事件入 completion queue → CDN 分发 + handler 更新元数据通知用户。
- **Container & Codecs**：容器封装音视频与元数据（MP4/AVI）；编解码器压缩解压（H.264/VP9）。
- **流媒体协议**：MPEG-DASH、Apple HLS、Adobe HDS——不同协议支持不同编码与播放器。
- **Preprocessor 四职责**：按 GOP 对齐切分视频、为旧客户端切分、按配置文件生成 DAG、GOP 与元数据存临时存储供失败重试。
- **Resource Manager**：task queue（优先级任务）+ worker queue（worker 利用率）+ running queue（运行中配对）+ task scheduler 挑最优配对。
- **Pre-Signed URLs**：授权用户限时直传，API 不经手文件流。
- **DRM / AES / 水印**：FairPlay、Widevine 等版权保护手段。

## Decision Rules
1. 上传原始文件后异步转码多个 codec/分辨率/码率
2. 转码任务用 DAG 调度，能并行的并行、有依赖的排序
3. 播放走 CDN 边缘直出，源站只处理回源
4. 热门走 CDN，冷门存大容量服务器、按需编码，区域化分发
5. 上传分块并行 + 断点续传；上传中心就近部署（CDN 兼做上传入口）
6. 可恢复错误重试，损坏视频停止处理并返回错误码

## Mental Models
- **上传和播放是两个系统**：写路径优化吞吐与可靠性，读路径优化延迟与成本，用不同组件。
- **DAG 是任务编排的通用语言**：先画任务依赖图，再谈并行度与重试边界。
- **CDN 账单决定架构**：先算 CDN 成本（$150k/天的例子），再决定分层策略。

## Anti-patterns
- **所有视频全放 CDN**：长尾内容分发成本爆炸。
- **转码失败无中间存储**：整个转码链从头重来。
- **上传走 API 服务器中转**：大文件占满连接，应 pre-signed URL 直传 blob。
- **单一分辨率输出**：弱网用户体验崩坏，且浪费高带宽。

## Worked Example
上传流：客户端分块并行直传 original storage（pre-signed URL），并行调 API 写元数据（标题/大小/格式）→ 转码服务器拉原片，preprocessor 按 GOP 切块并生成 DAG → scheduler 把 video/audio/metadata 分阶段入队，video 再拆编码与缩略图两任务 → resource manager 调度 worker 执行 → 完成事件双路：转码片入 transcoded storage 并分发 CDN，completion handler 更新元数据通知用户。播放流：客户端按协议（HLS/DASH）从最近 CDN 边缘节点拉流，按网速切换码率。

## Interview Lens
1. 开场报真实量级（20 亿 MAU、日播 50 亿）与设计假设（5M DAU、150TB/天、$150k CDN/天）。
2. 上传与播放两条流分开画，转码 DAG 单独深挖——三段式最有结构感。
3. Resource Manager 三队列 + scheduler 是转码架构最细的得分点。
4. 成本优化四招（热门 CDN/按需编码/区域化/自建 CDN）主动讲。
5. 安全（pre-signed URL/DRM/AES/水印）展示工程完整度。

## Practitioner Checklist
- [ ] 上传双流（视频+元数据）并行
- [ ] 转码 DAG 各组件职责清晰
- [ ] 临时存储支撑失败重试
- [ ] 播放协议与多码率自适应
- [ ] CDN 分层策略有成本论证
- [ ] 上传鉴权（pre-signed URL）与内容保护
- [ ] 错误分可恢复/不可恢复两类处理

## Key Takeaways
1. 上传→转码→分发与 CDN 播放是两条独立优化的流
2. DAG 模型 + 三队列 resource manager 支撑高并行转码
3. 热度分层是 CDN 成本控制的核心
4. pre-signed URL、DRM、断点续传是工程必备

## Connects To
- **Ch 1**：CDN 的基础概念。
- **Ch 19**：completion queue 等消息队列设计。
- **Ch 24**：blob/对象存储的完整设计。
- **Ch 13**：视频搜索建议复用自动补全。
