# Domain 3：设计高性能架构

**权重：24%**

Domain 3 关注的是系统在满足业务功能之后，如何让存储、计算、数据库、网络和数据管道在正确的位置提供足够性能。考试不是单纯问"哪个服务最快"，而是问你能不能根据访问模式、延迟目标、吞吐量、并发量、数据规模和全球用户分布，选出最合适的托管能力。

---

## 一、本域五大任务

| Task | 内容 | 核心问题 | 高频服务 |
|------|------|----------|----------|
| 3.1 | 设计高性能存储 | 数据放在哪里，如何满足 IOPS、吞吐、共享访问和访问延迟？ | S3、EBS、EFS、FSx、Storage Gateway |
| 3.2 | 设计高性能弹性计算 | 计算资源如何匹配负载类型、启动速度和扩展方式？ | EC2、Auto Scaling、Lambda、ECS、EKS、Batch、Placement Group |
| 3.3 | 设计高性能数据库 | 读写压力、延迟、扩展和数据模型如何匹配数据库服务？ | RDS、Aurora、DynamoDB、ElastiCache、DAX、Redshift、OpenSearch |
| 3.4 | 设计高性能网络架构 | 用户和服务之间如何降低网络延迟并提高吞吐？ | CloudFront、Global Accelerator、Route 53、VPC Endpoint、Direct Connect、ELB |
| 3.5 | 设计数据摄取与转换 | 大规模数据如何进入、处理、转换并交付到分析系统？ | Kinesis、Firehose、Glue、EMR、Lambda、SQS、MSK |

> **本域最高频混淆点：** 性能优化不是统一升级规格。存储看访问模式，计算看负载形态，数据库看读写模型，网络看用户路径，数据摄取看实时性和转换复杂度。

---

## 二、Domain 3 的设计主线

### 1. 从访问模式反推服务

高性能题最常见的陷阱是只看"快"，不看数据如何被访问。对象存储、块存储、文件存储和数据库存储解决的是完全不同的问题。S3 适合海量对象和静态内容，EBS 适合单台 EC2 的低延迟块设备，EFS 适合多台 Linux 实例共享文件，FSx 适合高性能文件系统或 Windows 文件共享。

考试中看到"多个 EC2 同时访问同一文件系统"不能选 EBS；看到"静态网站全球访问慢"不能只升级 EC2；看到"数据库热点读压力大"通常应该先考虑缓存或读副本。

---

### 2. 计算性能来自匹配负载，而不是只加机器

计算题要先判断负载类型：长期运行的 Web 服务、突发事件驱动任务、容器化微服务、批处理作业、HPC 低延迟集群，答案会完全不同。

EC2 适合需要控制操作系统和实例类型的通用计算；Lambda 适合事件驱动和短时任务；ECS/EKS 适合容器；AWS Batch 适合队列化批处理；Placement Group 适合低延迟或高吞吐的实例间通信。

---

### 3. 数据库性能由数据模型和访问路径决定

关系型事务、键值访问、文档搜索、分析型查询和内存级读取不应使用同一种数据库。RDS/Aurora 适合关系型事务；DynamoDB 适合高并发键值/文档访问；ElastiCache 和 DAX 解决缓存；Redshift 解决数据仓库分析；OpenSearch 解决全文检索和日志分析。

考试中看到"毫秒级稳定高并发键值访问"优先 DynamoDB；看到"复杂 SQL 事务"优先 RDS/Aurora；看到"读取延迟要降到微秒级"通常是缓存题。

---

### 4. 网络性能要减少距离和公网绕路

全球用户访问慢，通常不是源站算力不足，而是用户离源站太远。CloudFront 通过边缘缓存减少距离；Global Accelerator 通过 AWS 全球网络优化 TCP/UDP 路径；Route 53 Latency-based Routing 把用户导向更近区域。

服务间网络性能也要看路径。私有访问 AWS 服务用 VPC Endpoint，混合云稳定专线用 Direct Connect，HTTP 路由用 ALB，极低延迟 TCP/UDP 用 NLB。

---

### 5. 数据摄取看实时性、托管程度和转换复杂度

数据管道题要先判断是实时流、近实时交付、批处理 ETL，还是 Kafka 生态迁移。Kinesis Data Streams 适合实时自定义消费和可重放；Firehose 适合托管交付到 S3/Redshift/OpenSearch；Glue 适合 Serverless ETL；EMR 适合自定义大数据框架；MSK 适合 Kafka 兼容。

---

## 三、Task 边界速查

| 判断问题 | 更偏 3.1 | 更偏 3.2 | 更偏 3.3 | 更偏 3.4 | 更偏 3.5 |
|---------|----------|----------|----------|----------|----------|
| 核心矛盾 | 存储介质、IOPS、吞吐、共享文件 | 计算类型、实例选择、扩展方式 | 数据模型、查询延迟、读写扩展 | 用户访问路径、网络延迟、连接方式 | 流数据、批处理、ETL、数据交付 |
| 常见关键词 | S3、EBS、EFS、FSx、IOPS | EC2、Lambda、ECS、Batch、Placement | RDS、Aurora、DynamoDB、Cache | CloudFront、GA、DX、Endpoint | Kinesis、Firehose、Glue、EMR、MSK |
| 标准思路 | 按对象/块/文件选择 | 按负载形态选择计算平台 | 按访问模型选择数据库 | 缩短路径并避开公网 | 按实时性和转换复杂度选管道 |

---

## 四、本域核心服务判断表

| 场景关键词 | 首选服务/方案 | 判断理由 |
|-----------|---------------|----------|
| 海量静态对象、图片、备份 | S3 | 高持久性对象存储，可配合 CloudFront |
| EC2 低延迟块存储 | EBS gp3/io2 | 单实例块设备，按 IOPS/吞吐调优 |
| 多台 Linux EC2 共享文件 | EFS | 托管 NFS，多 AZ 共享 |
| Windows 文件共享 | FSx for Windows File Server | SMB/AD 集成 |
| HPC 文件系统 | FSx for Lustre | 高吞吐并行文件系统 |
| HTTP Web 服务扩展 | ALB + Auto Scaling | 七层分发和水平扩展 |
| 突发短任务 | Lambda | 事件驱动自动扩展 |
| 容器化服务 | ECS/EKS | 容器编排 |
| HPC 节点低延迟通信 | Cluster Placement Group | 实例间低延迟高带宽 |
| 关系型事务 | RDS/Aurora | SQL、事务、一致性 |
| 高并发键值访问 | DynamoDB | Serverless NoSQL，低延迟高吞吐 |
| 数据库热点读 | ElastiCache / DAX | 内存缓存降低读取延迟 |
| 数据仓库分析 | Redshift | 列式 MPP 分析 |
| 全球静态内容访问慢 | CloudFront | 边缘缓存 |
| 全球 TCP/UDP 应用加速 | Global Accelerator | Anycast + AWS 全球网络 |
| 私有访问 AWS 服务 | VPC Endpoint | 不经公网 |
| 实时流式自定义处理 | Kinesis Data Streams | 可重放、多消费者 |
| 流数据托管交付到 S3 | Kinesis Data Firehose | 近实时投递，无需管理消费者 |
| Serverless ETL | AWS Glue | 托管 Spark ETL 和 Data Catalog |

---

## 五、本域考试速记

- **先识别瓶颈位置**：存储、计算、数据库、网络、数据管道不要混着答。
- **S3/EBS/EFS/FSx 的边界最重要**：对象、块、共享文件、专用文件系统分别对应不同题型。
- **数据库性能先看数据模型**：关系型用 RDS/Aurora，键值高并发用 DynamoDB，分析用 Redshift。
- **全球访问慢优先缩短网络路径**：CloudFront 缓存内容，Global Accelerator 加速动态 TCP/UDP。
- **流数据题先问是否要自定义实时消费**：要就 Kinesis Data Streams，只交付就 Firehose。
