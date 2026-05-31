# Task 3.5：设计高性能数据摄取与转换架构

---

## 一、Task 考试地图

本 Task 的核心知识点按考试权重排列如下：

- Kinesis Data Streams、Kinesis Data Firehose、MSK 的流数据边界
- 实时处理、近实时交付、批处理 ETL 的选择逻辑
- AWS Glue Data Catalog、Crawler、ETL Job 在数据湖中的作用
- EMR 与 Glue 的托管程度和大数据框架差异
- Lambda、SQS、Step Functions 在轻量数据处理管道中的组合
- 数据写入 S3、Redshift、OpenSearch 的常见路径
- 数据转换、分区、压缩、格式转换对分析性能的影响

> **本 Task 最高频混淆对：** Kinesis Data Streams（实时自定义消费/可重放） vs Kinesis Data Firehose（托管交付）；Glue（Serverless ETL） vs EMR（可控大数据集群）；MSK（Kafka 兼容） vs Kinesis（AWS 原生流服务）。

---

## 二、核心概念解析

### 1. 数据摄取选择的第一原则

数据管道题先问三个问题：数据是否持续流入、是否需要实时处理、是否需要复杂转换。

如果数据持续产生并需要多个消费者实时处理，选 Kinesis Data Streams。如果只是把流数据近实时送到 S3、Redshift、OpenSearch 或第三方端点，选 Kinesis Data Firehose。如果企业已有 Kafka 应用并要求 Kafka API 兼容，选 MSK。如果是批量 ETL、数据清洗、格式转换和 Data Catalog，选 Glue。如果需要自定义 Hadoop/Spark/Hive 集群能力，选 EMR。

> 考试判断：实时自定义消费 → Kinesis Data Streams；托管交付 → Firehose；Kafka 兼容 → MSK；Serverless ETL → Glue；自管大数据框架 → EMR。

---

### 2. Kinesis Data Streams

Kinesis Data Streams 用于实时摄取和处理流数据。数据写入 Stream 后按 Shard 分区保存，消费者可以用 Lambda、Kinesis Client Library、Flink 或自定义应用读取。数据在保留期内可以被重放，多个消费者可以读取同一份数据。

它适合点击流、IoT 事件、应用日志、实时指标、欺诈检测等场景。性能设计重点是 shard 数量、分区键分布和消费者读取能力。分区键不均匀会造成 hot shard，限制吞吐。

Enhanced Fan-Out 可以为每个消费者提供独立读取吞吐，适合多个实时消费者同时读取同一 Stream。

> 考试判断：需要实时处理、多个消费者、重放事件、自定义处理逻辑 → Kinesis Data Streams。

---

### 3. Kinesis Data Firehose

Kinesis Data Firehose 是托管数据交付服务。它可以接收流数据并近实时交付到 S3、Redshift、OpenSearch、Splunk 或 HTTP 端点。Firehose 可以做缓冲、压缩、加密、格式转换，也可以调用 Lambda 做轻量转换。

Firehose 的价值是不用管理消费者和扩展逻辑。缺点是它不适合复杂实时处理，也不强调多消费者重放。

> 考试判断：应用日志或点击流直接送到 S3/Redshift/OpenSearch，要求少运维 → Firehose；需要复杂实时计算或多消费者 → Data Streams。

---

### 4. Amazon MSK

Amazon MSK 是托管 Apache Kafka。它适合已有 Kafka 生态、应用依赖 Kafka API、需要保留 Kafka 客户端和工具链的场景。

考试中如果题目明确说"现有 Kafka 集群迁移到 AWS"或"必须兼容 Kafka API"，MSK 通常是答案。否则 AWS 原生新建流处理题更常选 Kinesis。

> 考试判断：Kafka 兼容、迁移 Kafka、自管 Kafka 运维负担 → Amazon MSK。

---

### 5. AWS Glue

Glue 是 Serverless 数据集成服务，常用于数据湖 ETL。核心组件包括 Data Catalog、Crawler 和 ETL Job。

**Glue Data Catalog** 是元数据目录，保存表、分区、Schema 等信息，可被 Athena、Redshift Spectrum、EMR 等服务使用。

**Crawler** 可以扫描 S3 等数据源，自动推断 Schema 并更新 Data Catalog。

**Glue ETL Job** 通常基于 Spark，适合清洗、转换、合并、格式转换和写回数据湖。例如把 JSON/CSV 转成 Parquet，并按日期分区写入 S3，以提升 Athena/Redshift Spectrum 查询性能。

> 考试判断：Serverless ETL、自动发现 Schema、数据湖元数据 → Glue；把原始数据转换成 Parquet/分区数据 → Glue ETL。

---

### 6. Amazon EMR

EMR 是托管大数据集群服务，支持 Spark、Hadoop、Hive、Presto、HBase 等框架。相比 Glue，EMR 给用户更多集群、框架、配置和运行环境控制，适合已有 Hadoop/Spark 作业、复杂依赖、长时间运行集群或需要特定开源组件版本的场景。

EMR 可以使用 EC2 集群，也可以使用 EMR Serverless。考试中如果强调"现有 Hadoop/Spark 集群迁移"或"需要控制大数据框架配置"，EMR 更贴切。

> 考试判断：已有 Hadoop/Spark 作业迁移、需要自定义集群配置 → EMR；普通 Serverless ETL → Glue。

---

### 7. 轻量数据处理组合

不是所有数据管道都需要大数据服务。轻量事件处理可以用 S3 Event、EventBridge、SQS、Lambda 和 Step Functions 组合。

例如文件上传到 S3 后触发 Lambda 生成缩略图；订单事件进入 SQS 后由 Lambda 清洗并写入 DynamoDB；多步骤数据处理流程用 Step Functions 编排重试、分支和错误处理。

> 考试判断：单文件轻量转换 → Lambda；多步骤流程和错误处理 → Step Functions；突发任务削峰 → SQS。

---

## 三、服务对比速查表

| 对比维度 | Kinesis Data Streams | Kinesis Data Firehose |
|---------|----------------------|-----------------------|
| 核心定位 | 实时流数据平台 | 托管数据交付 |
| 消费模型 | 自定义消费者读取 | Firehose 自动投递 |
| 是否可重放 | 支持保留期内重放 | 不强调重放 |
| 多消费者 | 支持，Enhanced Fan-Out 更强 | 不适合作为多消费者流平台 |
| 典型目标 | Lambda、Flink、自定义应用 | S3、Redshift、OpenSearch、HTTP |
| 考试关键词 | 实时处理、可重放、自定义消费 | 近实时交付、少运维、投递到 S3 |

| 对比维度 | Kinesis | MSK |
|---------|---------|-----|
| 生态 | AWS 原生 | Apache Kafka 兼容 |
| 运维复杂度 | 更托管，概念更少 | Kafka 生态能力强 |
| 典型场景 | 新建 AWS 原生流处理 | 迁移 Kafka 或保留 Kafka API |
| 考试关键词 | Shard、Stream、Firehose | Kafka、Topic、Broker |

| 对比维度 | Glue | EMR |
|---------|------|-----|
| 管理模型 | Serverless ETL | 托管大数据集群/Serverless |
| 控制程度 | 较少 | 更高 |
| 典型场景 | 数据湖 ETL、Catalog、Crawler | Hadoop/Spark 迁移、自定义框架 |
| 考试关键词 | Data Catalog、Crawler、Parquet | Hadoop、Spark、Hive、集群 |

---

## 四、高频复合场景

**场景一：网站点击流需要实时分析，同时欺诈检测和推荐系统都要读取同一份事件。**

推理路径：持续流数据、多消费者、实时处理、需要同一数据被多个系统读取。Kinesis Data Streams 符合，必要时使用 Enhanced Fan-Out 给消费者独立吞吐。

最终答案：使用 Kinesis Data Streams 摄取点击流，不同消费者分别处理实时分析、欺诈检测和推荐。

**场景二：应用日志需要近实时写入 S3，并转换为 Parquet 供 Athena 查询，团队不想管理消费者。**

推理路径：目标是托管交付到 S3，并做格式转换。Firehose 支持缓冲、压缩、格式转换和投递 S3。

最终答案：使用 Kinesis Data Firehose 将日志交付到 S3，并配置格式转换为 Parquet。

**场景三：公司已有 Kafka 应用和客户端库，希望迁移到 AWS 并减少 broker 运维。**

推理路径：明确 Kafka 兼容要求，Kinesis 不兼容 Kafka API。MSK 是托管 Kafka。

最终答案：使用 Amazon MSK 托管 Kafka 集群。

**场景四：数据湖中每天落地大量 CSV，需要自动发现 Schema、转换为按日期分区的 Parquet。**

推理路径：自动发现 Schema 是 Glue Crawler/Data Catalog；批量转换为 Parquet 和分区写入是 Glue ETL。

最终答案：使用 Glue Crawler 更新 Data Catalog，Glue ETL Job 将 CSV 转换为分区 Parquet 写入 S3。

**场景五：已有复杂 Spark 作业依赖特定开源库和集群配置，需要迁移到 AWS。**

推理路径：需要控制 Spark 运行环境和依赖，Glue 的 Serverless ETL 可能不够灵活。EMR 更适合自定义大数据框架。

最终答案：使用 Amazon EMR 运行 Spark 作业，并按需配置集群和依赖。

---

## 五、本 Task 一句话总结

- **实时自定义消费选 Kinesis Data Streams，托管交付选 Firehose。**
- **Kafka 兼容需求明确时选 MSK，不要强行改成 Kinesis。**
- **数据湖元数据和 Serverless ETL 选 Glue。**
- **复杂 Hadoop/Spark 迁移和自定义框架选 EMR。**
- **轻量文件处理用 Lambda，多步骤数据流程用 Step Functions，突发任务前面加 SQS。**
