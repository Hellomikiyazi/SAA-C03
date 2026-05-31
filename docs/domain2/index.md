# Domain 2：设计弹性架构

**权重：26%**

Domain 2 关注的是系统在面对**流量变化、组件故障、可用区故障、区域级灾难**时，是否还能继续提供服务。它不是单纯考"哪些服务高可用"，而是考你能不能根据业务约束，在**可扩展性、松耦合、高可用、容错、恢复时间、成本**之间做正确取舍。

---

## 一、本域两大任务

| Task | 内容 | 核心问题 | 高频服务 |
|------|------|----------|----------|
| 2.1 | 设计可扩展与松耦合架构 | 流量来了如何扩？组件之间如何不互相拖垮？ | SQS、SNS、EventBridge、Auto Scaling、ALB、API Gateway、Lambda、ElastiCache、Kinesis |
| 2.2 | 设计高可用与容错架构 | 组件坏了怎么办？AZ/Region 故障后如何恢复？ | Route 53、ELB、Auto Scaling、RDS Multi-AZ、Read Replica、RDS Proxy、S3 CRR、AWS Backup |

> **本域最高频混淆点：** 2.1 解决"负载变化与组件解耦"，2.2 解决"故障恢复与服务连续性"。SQS、Auto Scaling、Read Replica 这类服务会跨两个 Task 出现，考试关键是判断题目问的是性能、解耦、可用性，还是灾难恢复。

---

## 二、Domain 2 的设计主线

### 1. 可扩展性：负载变大时系统不能崩

可扩展性（Scalability）回答的是："当请求量增加时，系统如何处理更多负载？"

AWS 架构中的标准答案通常是**水平扩展**：通过 Auto Scaling 增加更多 EC2、ECS Task 或 Lambda 并发，而不是只升级单台服务器。前端使用 ALB/NLB/API Gateway 承接流量，后端通过队列、缓存和数据库读扩展降低瓶颈。

考试中看到"突发流量"、"访问量持续增长"、"高峰期请求超时"、"需要自动增加处理能力"，首先从可扩展架构角度思考：入口层是否能分发流量，计算层是否能横向扩展，数据层是否有缓存或读扩展。

---

### 2. 松耦合：组件之间不能互相拖垮

松耦合（Loose Coupling）回答的是："一个组件慢了、挂了、扩缩了，是否会直接影响其他组件？"

紧耦合系统里，前端直接调用后端，后端慢就会让前端超时；下游服务挂掉，上游也会失败。松耦合架构会用 SQS、SNS、EventBridge、Step Functions 等服务隔离组件依赖，让生产者和消费者可以独立扩展、独立失败、独立恢复。

考试中看到"异步处理"、"削峰填谷"、"一个请求触发多个系统"、"事件驱动"、"下游服务处理速度不稳定"，通常要在 SQS、SNS、EventBridge、Step Functions 之间做选择。

---

### 3. 高可用：单个组件故障不能导致整体中断

高可用（High Availability）回答的是："某个实例、磁盘、AZ 出故障后，系统是否还能继续运行？"

高可用设计的基本套路是消除单点故障（SPOF）：EC2 不要单台部署，要放在多个 AZ 的 Auto Scaling Group 后面；入口不要暴露单台服务器，要用 ELB；数据库不要单 AZ，要启用 RDS Multi-AZ；NAT Gateway 不要只有一个 AZ，要每个 AZ 独立部署。

考试中看到"单台 EC2 宕机导致服务不可用"、"一个 AZ 故障影响全部应用"、"数据库主实例故障后需要自动恢复"，优先考虑 Multi-AZ、ELB、Auto Scaling、RDS Multi-AZ。

---

### 4. 容错与灾难恢复：故障发生后能恢复到什么程度

容错（Fault Tolerance）和灾难恢复（Disaster Recovery）回答的是："更大范围故障发生后，最多能丢多少数据，多久能恢复？"

这部分的判断核心是 RPO 和 RTO。RPO 决定最多能丢多少数据，RTO 决定最多能停多久。要求越严格，方案越接近 Warm Standby 或 Active-Active，成本也越高；要求越宽松，Backup and Restore 或 Pilot Light 就可能足够。

考试中看到"Region 故障"、"跨区域恢复"、"服务中断不能超过 X 分钟"、"数据丢失不能超过 X 秒/小时"，要立刻切到 DR 策略选择，而不是只回答 Multi-AZ。

---

## 三、Task 2.1 与 Task 2.2 的边界

| 判断问题 | 更偏 Task 2.1 | 更偏 Task 2.2 |
|---------|---------------|---------------|
| 题目核心矛盾 | 流量增长、处理变慢、系统耦合过紧 | 实例故障、AZ 故障、Region 灾难 |
| 主要目标 | 扩展吞吐、削峰填谷、异步处理 | 提高可用性、自动故障切换、灾后恢复 |
| 常见关键词 | scalable、decouple、asynchronous、fanout、burst traffic | highly available、fault tolerant、failover、RPO、RTO |
| 典型服务 | SQS、SNS、EventBridge、Auto Scaling、ElastiCache、Kinesis | Multi-AZ、Route 53 Failover、RDS Proxy、Backup、CRR |
| 常见答案模式 | SQS 缓冲 + Worker 扩展；SNS 扇出；EventBridge 事件路由 | Multi-AZ + ELB + ASG；RDS Multi-AZ；跨 Region DR |

简单说：**2.1 关心"扛不扛得住"，2.2 关心"坏了还能不能用"。**

---

## 四、本域核心服务判断表

| 场景关键词 | 首选服务/方案 | 判断理由 |
|-----------|---------------|----------|
| 后端处理慢，请求超时，需要异步 | SQS | 队列缓冲，生产者和消费者解耦 |
| 一条消息通知多个系统 | SNS | 发布订阅，消息扇出 |
| AWS 服务状态变化触发动作 | EventBridge | 基于事件规则过滤和路由 |
| 多步骤业务流程，有重试和补偿 | Step Functions | 编排有状态工作流 |
| Web 流量自动扩缩 | ALB + Auto Scaling Group | 分发流量并按指标扩容 |
| HTTP 路径/Host 路由 | ALB | 第 7 层负载均衡 |
| TCP/UDP、固定 IP、超低延迟 | NLB | 第 4 层高性能入口 |
| Serverless API、限流、API Key | API Gateway | 托管 API 入口 |
| Session 不能绑在单台 EC2 | ElastiCache / DynamoDB | 会话状态外置，实现无状态扩展 |
| 全球静态内容访问慢 | CloudFront | 边缘缓存降低延迟 |
| RDS 主库故障自动切换 | RDS Multi-AZ | 高可用，不提升读性能 |
| RDS 读压力过大 | Read Replica | 分担读请求，不是自动 HA |
| Lambda 并发打爆 RDS 连接数 | RDS Proxy | 连接池复用，改善故障切换 |
| 主 Region 故障自动切流 | Route 53 Failover | DNS 健康检查 + 主备切换 |
| 跨 Region 对象复制 | S3 Cross-Region Replication | 区域级数据冗余 |

---

## 五、高频组合架构

### 1. 典型 Web 三层弹性架构

标准组合是 Route 53 → ALB → 多 AZ Auto Scaling Group → RDS Multi-AZ。如果读压力很大，再加 RDS Read Replica；如果静态内容多，再加 S3 + CloudFront；如果 Session 需要共享，再加 ElastiCache 或 DynamoDB。

考试判断：这是解决"Web 应用既要扩展又要高可用"的默认框架。

---

### 2. 异步任务处理架构

标准组合是 Web/API 层 → SQS → Worker Auto Scaling Group 或 Lambda → 数据库/S3。SQS 负责缓冲任务，Worker 根据队列深度扩缩，DLQ 保存处理失败的消息。

考试判断：这是解决"突发请求、耗时任务、后端处理不过来"的默认框架。

---

### 3. 事件驱动扇出架构

标准组合是业务服务 → SNS Topic → 多个 SQS 队列或 Lambda。每个下游系统拥有自己的队列，可以独立重试、独立扩缩，不影响其他消费者。

考试判断：这是解决"一条业务事件触发多个独立系统"的默认框架。

---

### 4. 跨区域灾难恢复架构

标准组合是主 Region 运行生产系统，备用 Region 通过数据复制保持恢复能力，Route 53 Failover 负责故障时切换入口。根据 RPO/RTO 和成本要求，在 Backup and Restore、Pilot Light、Warm Standby、Active-Active 之间选择。

考试判断：这是解决"Region 故障、RPO/RTO 明确、用户仍需访问服务"的默认框架。

---

## 六、本域考试速记

- **先判断题目问负载还是故障**：负载问题看 2.1，故障问题看 2.2。
- **SQS 是削峰填谷和异步解耦**，SNS 是一对多广播，EventBridge 是事件规则路由。
- **Multi-AZ 是高可用，Read Replica 是读扩展**，不要把读副本当作自动故障切换。
- **无状态是水平扩展前提**：Session 外置，文件放 S3/EFS，实例可随时替换。
- **RPO/RTO 决定 DR 策略**：要求越低，成本越高，越接近 Active-Active。
- **消除 SPOF 的基础组合**：多 AZ + ELB + Auto Scaling + 托管数据库高可用。
