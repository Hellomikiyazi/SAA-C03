# Task 3.3：设计高性能数据库架构

---

## 一、Task 考试地图

本 Task 的核心知识点按考试权重排列如下：

- RDS、Aurora、DynamoDB、Redshift、OpenSearch 的数据模型边界
- RDS Read Replica、Aurora Replica 与 Multi-AZ 的性能和可用性差异
- DynamoDB 分区键设计、容量模式、Global Tables、DAX
- ElastiCache for Redis/Memcached 的缓存场景
- 数据库连接池与 RDS Proxy
- Redshift 在 OLAP 数据仓库中的定位
- OpenSearch 在全文检索和日志分析中的定位

> **本 Task 最高频混淆对：** RDS Read Replica（读扩展） vs Multi-AZ（高可用）；DynamoDB DAX（DynamoDB 专用缓存） vs ElastiCache（通用缓存）；Redshift（分析型数据仓库） vs RDS（事务型数据库）。

---

## 二、核心概念解析

### 1. 数据库选择的第一原则

数据库性能题首先看数据模型和访问模式。

如果需要 SQL、事务、复杂 JOIN、强一致关系模型，选 RDS 或 Aurora。如果访问模式是高并发键值查询、按主键读写、无服务器扩展，选 DynamoDB。如果需要缓存热点数据，选 ElastiCache 或 DAX。如果需要对 TB/PB 级数据做分析型查询，选 Redshift。如果需要全文检索、日志搜索、模糊匹配，选 OpenSearch。

> 考试判断：OLTP 事务 → RDS/Aurora；键值高并发 → DynamoDB；热点读延迟 → Cache/DAX；OLAP 分析 → Redshift；全文检索 → OpenSearch。

---

### 2. RDS 与 Aurora 性能扩展

RDS 是托管关系型数据库，支持 MySQL、PostgreSQL、MariaDB、Oracle、SQL Server 等。Aurora 是 AWS 云原生关系数据库，兼容 MySQL/PostgreSQL，通常提供更高性能、分布式存储和更快副本扩展。

关系型数据库性能扩展常见手段有三类。

第一是**垂直扩展**，升级实例规格，适合 CPU、内存或网络不足的单点瓶颈。

第二是**读扩展**，使用 Read Replica 或 Aurora Replica 把读请求分流。应用需要显式把读流量发到副本端点。Multi-AZ 不是读扩展，不能用来分担读请求。

第三是**连接管理**，使用 RDS Proxy 复用连接，尤其适合 Lambda 或短连接高并发场景。

Aurora 还提供 Reader Endpoint，把读流量负载均衡到多个 Aurora Replica；Aurora Serverless v2 适合容量变化频繁但仍需要关系型能力的场景。

> 考试判断：关系型读压力大 → Read Replica/Aurora Replica；主库故障切换 → Multi-AZ；Lambda 连接数过多 → RDS Proxy；容量波动关系型 → Aurora Serverless v2。

---

### 3. DynamoDB 高性能设计

DynamoDB 是 Serverless NoSQL 数据库，适合高并发、低延迟、可预测访问模式的键值和文档数据。性能设计重点是分区键、容量模式和热点避免。

**分区键设计** 决定数据和请求如何分布。如果大量请求集中到同一个分区键，会形成 hot partition，导致吞吐受限。高基数、访问分布均匀的分区键更适合高性能。

**容量模式** 包括 On-Demand 和 Provisioned。On-Demand 适合不可预测流量，无需提前配置吞吐。Provisioned 适合可预测流量，可以配合 Auto Scaling 控制读写容量。

**Global Secondary Index（GSI）** 支持按不同访问模式查询，但 GSI 本身也需要容量和良好键设计。不能把 DynamoDB 当作任意字段随意查询的数据库。

**Global Tables** 提供多 Region 主动-主动复制，适合全球用户低延迟读写和区域级可用性。

> 考试判断：DynamoDB 性能差且某个 key 特别热 → 重新设计分区键或写分片；流量不可预测 → On-Demand；全球多区域低延迟读写 → Global Tables。

---

### 4. DAX 与 ElastiCache

DynamoDB Accelerator（DAX）是 DynamoDB 专用的全托管内存缓存。它兼容 DynamoDB API，可把读取延迟从毫秒级降到微秒级，适合读密集、最终一致性可接受、访问热点明显的 DynamoDB 工作负载。

ElastiCache 是通用内存缓存服务，支持 Redis 和 Memcached。Redis 更常见，支持复杂数据结构、持久化、复制、Pub/Sub、排行榜、Session 等。Memcached 简单高性能，适合纯缓存、简单 key-value。

两者边界很明确：DAX 只服务 DynamoDB；ElastiCache 可缓存 RDS、API 响应、Session 或任意应用数据。

> 考试判断：DynamoDB 读延迟要微秒级 → DAX；RDS 热点查询缓存或 Session → ElastiCache；排行榜/复杂数据结构 → Redis。

---

### 5. Redshift

Redshift 是托管数据仓库，面向 OLAP 分析，不适合高并发小事务。它使用列式存储和 MPP 架构，适合对大量历史数据做聚合、报表和 BI 查询。

Redshift Spectrum 可以直接查询 S3 数据湖中的数据，不必全部加载进 Redshift。RA3 节点将计算和存储解耦，适合需要灵活扩展分析能力的场景。

> 考试判断：大量数据分析、BI 报表、复杂聚合 → Redshift；事务型订单系统 → RDS/Aurora；直接分析 S3 数据湖 → Redshift Spectrum 或 Athena。

---

### 6. OpenSearch

OpenSearch 适合全文检索、日志分析、模糊搜索和近实时搜索。它不是关系型事务数据库，也不是数据仓库替代品。

典型场景包括应用日志搜索、网站搜索框、商品全文检索、安全日志分析。日志管道常见组合是 CloudWatch Logs/Kinesis Data Firehose → OpenSearch，或应用日志 → Kinesis → OpenSearch。

> 考试判断：全文搜索、模糊匹配、日志检索 → OpenSearch；不要用 RDS 的 LIKE 查询去承担大规模搜索场景。

---

## 三、服务对比速查表

| 对比维度 | RDS/Aurora | DynamoDB | Redshift | OpenSearch |
|---------|------------|----------|----------|------------|
| 数据模型 | 关系型 SQL | Key-value/Document | 列式数据仓库 | 搜索索引 |
| 核心场景 | OLTP 事务 | 高并发低延迟访问 | OLAP 分析报表 | 全文检索/日志搜索 |
| 扩展方式 | 实例、副本、Aurora 架构 | 分区和吞吐自动扩展 | MPP 集群/Serverless | 分片和节点 |
| 考试关键词 | JOIN、事务、SQL | 分区键、毫秒延迟 | BI、聚合、数据仓库 | Search、日志、模糊查询 |

| 对比维度 | Multi-AZ | Read Replica |
|---------|----------|--------------|
| 设计目的 | 高可用故障切换 | 读性能扩展 |
| 是否服务读请求 | 通常不服务 | 可以服务 |
| 复制方式 | 同步或托管高可用复制 | 异步复制 |
| 应用是否改读端点 | 不需要 | 需要 |
| 考试关键词 | 自动 failover | 读多写少、查询压力 |

| 对比维度 | DAX | ElastiCache |
|---------|-----|--------------|
| 服务范围 | DynamoDB 专用 | 通用缓存 |
| API 兼容 | DynamoDB API | Redis/Memcached API |
| 典型目标 | DynamoDB 微秒级读取 | RDS/API/Session 热点缓存 |
| 高频场景 | DynamoDB 读密集 | Session、排行榜、数据库缓存 |

---

## 四、高频复合场景

**场景一：订单系统使用 Aurora PostgreSQL，写入正常但报表查询拖慢主库，业务需要分担读请求。**

推理路径：这是关系型数据库读压力问题，不是高可用问题。应创建 Aurora Replica 或 Read Replica，把报表和只读查询导向 Reader Endpoint/副本端点。

最终答案：添加 Aurora Replica，并将只读查询路由到 Reader Endpoint。

**场景二：游戏应用需要以极低延迟读取玩家资料，访问模式是按 player_id 读取，流量峰值很高。**

推理路径：按主键高并发访问，适合 DynamoDB。若读取非常频繁且要求微秒级，可在 DynamoDB 前加 DAX。

最终答案：使用 DynamoDB，以 player_id 作为高基数分区键；读延迟要求更低时使用 DAX。

**场景三：Web 应用频繁读取 RDS 中相同商品信息，数据库 CPU 很高，但数据可以接受短时间缓存。**

推理路径：热点读造成数据库压力，且数据可缓存。ElastiCache 是通用缓存答案，应用先读缓存，未命中再读 RDS。

最终答案：在应用和 RDS 之间加入 ElastiCache for Redis。

**场景四：公司希望对多年销售数据做 BI 聚合分析，当前用生产 MySQL 跑报表导致性能下降。**

推理路径：BI 聚合分析是 OLAP，不应压在 OLTP 生产库上。Redshift 是数据仓库标准答案。

最终答案：将数据 ETL 到 Amazon Redshift，用 Redshift 执行分析查询。

**场景五：电商网站需要支持商品标题和描述的模糊搜索、排序和过滤。**

推理路径：全文搜索不是关系型数据库强项，也不是 DynamoDB 主键访问模式。OpenSearch 专门处理搜索索引和文本相关性。

最终答案：将商品数据索引到 OpenSearch Service，搜索请求由 OpenSearch 处理。

---

## 五、本 Task 一句话总结

- **关系型事务用 RDS/Aurora，高并发键值用 DynamoDB，分析仓库用 Redshift，全文搜索用 OpenSearch。**
- **Multi-AZ 管故障切换，Read Replica 管读扩展。**
- **DynamoDB 性能关键是分区键，不是后期随便加索引就能解决所有查询。**
- **DAX 只缓存 DynamoDB，ElastiCache 是通用缓存。**
- **Lambda 大并发访问 RDS，优先想 RDS Proxy。**
