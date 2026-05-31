# Task 3.2：设计高性能弹性计算架构

---

## 一、Task 考试地图

本 Task 的核心知识点按考试权重排列如下：

- EC2 实例家族选择：通用、计算优化、内存优化、存储优化、加速计算
- Auto Scaling 策略与高性能扩展模式
- Lambda 的并发、冷启动、Provisioned Concurrency 与事件源扩展
- ECS、EKS、Fargate 在容器计算中的选择
- AWS Batch 与批处理队列化计算
- Placement Group 对低延迟和高吞吐的影响
- Graviton、GPU、Inferentia 等专用硬件在性能题中的定位

> **本 Task 最高频混淆对：** Lambda（事件驱动短任务） vs EC2（长期运行/自定义环境）；ECS/EKS（容器编排） vs AWS Batch（批处理作业调度）；Cluster Placement Group（低延迟） vs Spread Placement Group（隔离故障）。

---

## 二、核心概念解析

### 1. 计算平台选择的第一原则

高性能计算题先判断工作负载形态，而不是先选最大实例。

长期运行的 Web 应用、需要完整 OS 控制、需要特定软件或驱动时，EC2 是最灵活的选择。需要自动处理事件、任务短、负载突发且无需管理服务器时，Lambda 更合适。已经容器化的应用，优先考虑 ECS、EKS 或 Fargate。大规模排队执行的科学计算、渲染、转码、基因分析等任务，AWS Batch 更适合。

> 考试判断：短时事件驱动 → Lambda；长期服务或自定义系统 → EC2；容器微服务 → ECS/EKS；大量批处理作业 → AWS Batch。

---

### 2. EC2 实例类型选择

EC2 性能题经常通过负载特征考实例家族。

**通用型（M、T）** 平衡 CPU、内存和网络，适合 Web 服务、应用服务器、小型数据库。T 系列有 CPU Credit，适合突发型轻负载，不适合持续高 CPU。

**计算优化型（C）** 适合 CPU 密集任务，例如高性能 Web 服务、批处理、科学建模、游戏服务器。

**内存优化型（R、X、u）** 适合内存数据库、缓存、实时大数据分析、SAP HANA。

**存储优化型（I、D、H）** 适合高本地磁盘 I/O，例如分布式数据库、NoSQL、数据仓库临时处理。注意本地 Instance Store 数据随实例停止或终止丢失，不能当长期持久层。

**加速计算型（P、G、Inf、Trn）** 适合 GPU、机器学习训练/推理、图形渲染和专用加速。

> 考试判断：CPU 高 → C；内存高 → R/X；本地 I/O 高 → I/D；GPU/ML → P/G/Inf/Trn；通用 Web → M。

---

### 3. Auto Scaling 性能设计

Auto Scaling 不是只为高可用，也是性能扩展的核心。关键是选择与负载相关的指标。

Web 服务常用 ALB RequestCountPerTarget、CPUUtilization 或 Target Tracking。队列消费 Worker 不应只看 CPU，应根据 SQS ApproximateNumberOfMessagesVisible 或队列深度与实例数比例扩展。容器服务可以使用 ECS Service Auto Scaling，根据 CPU、内存或自定义指标扩展任务数。

**预热和启动时间** 是性能题隐藏考点。如果实例启动慢、应用预热慢，可以使用 Warm Pools、预烘焙 AMI、启动模板和计划扩展，在高峰前准备容量。

> 考试判断：不可预测流量 → Target Tracking；固定高峰 → Scheduled Scaling；Worker 处理队列 → 按 SQS 队列深度扩展；实例启动慢 → Warm Pool 或提前计划扩容。

---

### 4. Lambda 高性能设计

Lambda 自动扩展并发，适合事件驱动短任务，但性能题会考三个限制。

第一是**并发限制**。突发大量事件会增加并发执行数，如果超过账号或函数并发限制，会出现 throttling。可以使用 Reserved Concurrency 为关键函数预留并发，也可以限制某函数不要打爆下游。

第二是**冷启动**。对延迟敏感的同步 API，如果冷启动影响明显，可以使用 Provisioned Concurrency 预初始化执行环境。

第三是**下游连接压力**。Lambda 并发访问 RDS 会产生大量数据库连接，标准解法是 RDS Proxy，而不是让每个函数直接建立连接。

> 考试判断：Lambda 同步 API 冷启动影响用户 → Provisioned Concurrency；Lambda 并发打爆 RDS → RDS Proxy；需要控制函数最大并发 → Reserved Concurrency。

---

### 5. 容器计算：ECS、EKS、Fargate

ECS 是 AWS 原生容器编排服务，操作简单，和 ALB、IAM、CloudWatch 集成紧密。EKS 是托管 Kubernetes，适合已有 Kubernetes 生态、需要 Kubernetes API 或多云一致性的团队。

Fargate 是 Serverless 容器运行方式，不需要管理 EC2 节点，适合希望减少运维负担的容器服务。EC2 Launch Type 则适合需要控制实例类型、使用 GPU、本地存储、特殊网络或更精细成本优化的场景。

> 考试判断：简单 AWS 原生容器 → ECS；已有 Kubernetes → EKS；不想管理节点 → Fargate；需要特殊实例/GPU/本地盘 → EC2 Launch Type。

---

### 6. AWS Batch

AWS Batch 用于运行大量批处理作业。它管理作业队列、调度、计算环境和自动扩缩。用户提交作业，Batch 根据队列需求启动合适的 EC2 或 Fargate 资源执行。

它适合不需要常驻服务、但需要大量并行计算的场景，例如视频转码、金融风险模拟、基因分析、科学计算。相比自己写队列和 Worker，Batch 的优势是托管调度和计算环境管理。

> 考试判断：大量批处理作业排队执行、需要自动调度计算资源 → AWS Batch。

---

### 7. Placement Group

Placement Group 决定 EC2 实例在底层物理基础设施上的放置方式。

**Cluster Placement Group** 将实例放得更近，提供低延迟、高吞吐的实例间网络，适合 HPC、MPI、分布式训练。缺点是更容易受单个机架或底层故障影响，且容量要求高时可能启动失败。

**Spread Placement Group** 将实例分散到底层硬件，降低相关故障风险，适合少量关键实例隔离。

**Partition Placement Group** 将实例分成多个分区，每个分区隔离底层硬件，适合 HDFS、Cassandra、Kafka 等大型分布式系统。

> 考试判断：HPC 节点间低延迟 → Cluster；少量关键实例隔离故障 → Spread；大型分布式数据系统分区隔离 → Partition。

---

## 三、服务对比速查表

| 对比维度 | EC2 | Lambda |
|---------|-----|--------|
| 运行模式 | 长期运行服务器 | 事件驱动函数 |
| 控制能力 | 完整 OS 和运行时控制 | 运行时受托管平台限制 |
| 扩展方式 | Auto Scaling 扩实例 | 自动扩并发 |
| 适合场景 | Web 服务、自定义软件、长任务 | 短任务、异步事件、Serverless API |
| 高频限制 | 启动和容量管理 | 冷启动、并发、最长执行时间 |

| 对比维度 | ECS | EKS | AWS Batch |
|---------|-----|-----|-----------|
| 核心用途 | AWS 原生容器编排 | 托管 Kubernetes | 批处理作业调度 |
| 适合场景 | 微服务、后台服务 | Kubernetes 生态迁移 | 大量排队计算任务 |
| 管理模型 | Service/Task | Pod/Deployment | Job/Queue/Compute Environment |
| 考试关键词 | 容器、简单托管 | Kubernetes API | 批处理、作业队列 |

| 对比维度 | Cluster Placement Group | Spread Placement Group | Partition Placement Group |
|---------|--------------------------|------------------------|---------------------------|
| 设计目标 | 低延迟高吞吐 | 实例硬件隔离 | 分区级故障隔离 |
| 典型场景 | HPC、MPI、分布式训练 | 少量关键实例 | Kafka、Cassandra、HDFS |
| 主要取舍 | 性能优先 | 可用性隔离优先 | 大规模分布式隔离 |

---

## 四、高频复合场景

**场景一：金融建模程序需要多台 EC2 之间进行极低延迟通信，节点间网络性能是瓶颈。**

推理路径：这是实例间网络性能问题，不是单台实例 CPU 不足。多节点低延迟通信对应 Cluster Placement Group，并选择支持高网络带宽的实例类型。

最终答案：使用 Cluster Placement Group 部署计算节点，并选择计算优化或 HPC 适用实例。

**场景二：电商 API 使用 Lambda + API Gateway，偶发请求延迟很高，主要发生在函数长时间未调用后。**

推理路径：长时间未调用后首次请求慢，典型冷启动。同步 API 对延迟敏感，应使用 Provisioned Concurrency 预热函数。

最终答案：为关键 Lambda 配置 Provisioned Concurrency。

**场景三：图片处理任务数量随用户上传量变化，任务可排队并并行处理。**

推理路径：这是异步批量任务。可以用 SQS 缓冲任务，Worker 使用 Auto Scaling 按队列深度扩展；如果是大量作业调度和计算环境管理，也可用 AWS Batch。

最终答案：S3 上传事件触发任务写入 SQS，EC2/ECS Worker 按队列深度 Auto Scaling；复杂批处理可使用 AWS Batch。

**场景四：公司已有 Kubernetes 工作负载，希望迁移到 AWS 并继续使用 Kubernetes API 和生态工具。**

推理路径：明确 Kubernetes 生态要求，ECS 虽然能跑容器但不提供 Kubernetes API。应选择 EKS。

最终答案：使用 Amazon EKS 托管 Kubernetes 控制面。

**场景五：Lambda 大量并发处理请求时，RDS 数据库连接数被耗尽。**

推理路径：问题不是 Lambda 算力不足，而是 Serverless 并发造成数据库短连接过多。RDS Proxy 提供连接池和复用。

最终答案：在 Lambda 和 RDS 之间使用 RDS Proxy，并根据需要配置 Lambda 并发限制。

---

## 五、本 Task 一句话总结

- **计算题先判断负载形态**：长服务 EC2，短事件 Lambda，容器 ECS/EKS，批处理 Batch。
- **实例家族按瓶颈选**：CPU 用 C，内存用 R/X，本地 I/O 用 I/D，GPU/ML 用加速计算。
- **队列 Worker 扩展看队列深度**，不是只看 CPU。
- **Lambda 冷启动用 Provisioned Concurrency，并发打爆 RDS 用 RDS Proxy。**
- **低延迟节点通信用 Cluster Placement Group，故障隔离才用 Spread/Partition。**
