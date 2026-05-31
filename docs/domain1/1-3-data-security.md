# Task 1.3：确定适当的数据安全控制

---

## 一、Task 考试地图

本 Task 的核心知识点按考试权重排列如下：

- 静态加密与传输中加密的服务选择
- AWS KMS、Customer Managed Key、AWS Managed Key、CloudHSM 的区别
- S3 加密方式：SSE-S3、SSE-KMS、SSE-C、客户端加密
- KMS Key Policy、IAM Policy、Grant、跨账号密钥访问
- ACM、TLS、证书管理与私有 CA
- 敏感数据发现、分类与审计：Macie、CloudTrail、Config
- 数据备份、防删除与保留：AWS Backup、Vault Lock、S3 Object Lock
- 数据泄漏防护：S3 Block Public Access、Bucket Policy、访问日志

> **本 Task 最高频混淆对：** KMS（托管密钥管理，AWS 服务集成广） vs CloudHSM（专用硬件 HSM）；SSE-S3（S3 托管密钥） vs SSE-KMS（KMS 托管并可审计/控权）。

---

## 二、核心概念解析

### 1. 数据保护的两条线：静态与传输中

静态加密（Encryption at Rest）保护存储在磁盘、对象存储、数据库、备份中的数据。S3、EBS、RDS、DynamoDB、EFS、Redshift 等服务都支持静态加密，通常由 KMS 管理密钥。

传输中加密（Encryption in Transit）保护网络传输过程中的数据，标准方案是 TLS/HTTPS。ACM 用于申请、管理和自动续期 TLS 证书，并可与 ALB、CloudFront、API Gateway 等服务集成。

> 考试判断：数据存储后如何保护 → 静态加密/KMS；客户端到服务之间如何保护 → TLS/ACM。

---

### 2. AWS KMS

AWS KMS 是托管密钥管理服务，用于创建、管理和审计加密密钥。它与大量 AWS 服务原生集成，是 SAA 考试中默认的密钥管理答案。

KMS Key 分为 AWS Owned Key、AWS Managed Key、Customer Managed Key。考试最常见的是 Customer Managed Key，因为它支持客户控制 Key Policy、启用/禁用、轮换、别名、审计和跨账号授权。KMS 自动记录 CloudTrail 事件，便于追踪谁使用了密钥。

KMS Envelope Encryption（信封加密）是标准模式：KMS 生成或解密数据密钥，数据本身由数据密钥在本地加密，避免把大量数据直接发给 KMS。

> 考试判断：需要控制密钥权限、审计密钥使用、跨账号共享加密数据 → Customer Managed Key。

---

### 3. CloudHSM

CloudHSM 提供单租户硬件安全模块，客户完全控制 HSM、密钥材料和加密操作。它适合严格合规要求，例如必须使用专用 HSM、FIPS 140-2 Level 3、或需要迁移已有 HSM 应用。

CloudHSM 的运维复杂度和成本高于 KMS。普通 AWS 服务加密、S3/RDS/EBS/DynamoDB 加密，优先 KMS；只有题目明确强调专用硬件控制、客户完全管理密钥材料、或合规强制 HSM 时才选 CloudHSM。

> 考试判断：大多数托管加密 → KMS；专用硬件 HSM/客户完全控制密钥 → CloudHSM。

---

### 4. S3 数据加密

S3 支持多种静态加密方式。**SSE-S3** 由 S3 托管密钥，最简单，适合默认加密。**SSE-KMS** 使用 KMS Key，支持细粒度权限控制、CloudTrail 审计和客户管理密钥。**SSE-C** 由客户提供密钥，AWS 不保存密钥，应用必须在每次请求中提供密钥。**客户端加密**是在数据上传前由客户端自行加密。

考试中，如果题目只要求"加密 S3 对象且管理简单"，SSE-S3 足够；如果要求密钥访问审计、密钥禁用、跨账号控制或客户管理密钥，选择 SSE-KMS。

> 考试判断：简单 S3 默认加密 → SSE-S3；需要 KMS 审计和访问控制 → SSE-KMS；客户自己提供密钥且 AWS 不保存 → SSE-C。

---

### 5. KMS 权限与跨账号访问

KMS 权限由 Key Policy、IAM Policy 和 Grant 共同决定。Key Policy 是密钥的根权限控制，没有合适的 Key Policy，仅 IAM Policy 允许通常不够。Grant 常用于临时委托 AWS 服务代表用户使用 KMS Key。

跨账号使用 KMS Key 时，密钥所在账号的 Key Policy 必须允许外部账号或角色使用，同时外部账号中的 IAM Policy 也要允许对应 KMS 动作。加密数据本身的资源访问权限也必须允许，例如 S3 Bucket Policy。

> 考试判断：跨账号无法解密 SSE-KMS 对象 → 同时检查 S3 权限、KMS Key Policy、调用方 IAM Policy。

---

### 6. ACM 与证书管理

AWS Certificate Manager 用于申请和管理 TLS 证书。公有证书可自动续期，并可直接用于 ALB、CloudFront、API Gateway 等服务。ACM Private CA 用于企业内部私有证书颁发机构。

CloudFront 使用 ACM 证书时，证书必须位于 us-east-1。ALB 使用 ACM 证书时，证书应在 ALB 所在 Region。

> 考试判断：托管 TLS 证书 → ACM；内部私有证书体系 → ACM Private CA；CloudFront 证书位置 → us-east-1。

---

### 7. 敏感数据发现与防泄漏

Amazon Macie 使用机器学习和模式匹配发现 S3 中的敏感数据，例如 PII、凭证、财务信息。它适合回答"如何发现 S3 Bucket 中是否存放敏感数据"。

S3 Block Public Access 是防止 S3 意外公开的关键控制，可以在账号级或 Bucket 级启用。Bucket Policy、ACL、Access Analyzer、S3 Server Access Logs、CloudTrail Data Events 可进一步帮助限制和审计访问。

> 考试判断：发现 S3 敏感数据 → Macie；防止 S3 被公开 → Block Public Access；审计对象级访问 → CloudTrail Data Events / S3 访问日志。

---

### 8. 备份保留与不可变性

AWS Backup 可集中管理 EBS、RDS、DynamoDB、EFS、FSx 等资源备份，并通过 Backup Plan 设置备份频率和保留周期。Backup Vault Lock 可实现 WORM（Write Once Read Many）不可变备份，防止备份在保留期内被删除。

S3 Object Lock 也提供 WORM 模式，支持 Governance Mode 和 Compliance Mode。Compliance Mode 下，即使 root 用户也不能在保留期到期前删除或缩短保留。

> 考试判断：集中跨服务备份 → AWS Backup；备份不可删除/防勒索 → Backup Vault Lock；S3 对象不可变保留 → S3 Object Lock。

---

## 三、服务对比速查表

| 对比维度 | KMS | CloudHSM |
|---------|-----|----------|
| 托管模式 | AWS 托管密钥服务 | 客户控制专用 HSM |
| 硬件租户 | 多租户托管服务 | 单租户硬件 |
| AWS 服务集成 | 非常广 | 需要更多自定义集成 |
| 运维复杂度 | 低 | 高 |
| 典型场景 | S3/EBS/RDS/DynamoDB 加密 | 专用 HSM、FIPS Level 3、传统 HSM 迁移 |

| 对比维度 | SSE-S3 | SSE-KMS |
|---------|--------|---------|
| 密钥管理 | S3 托管 | KMS 托管 |
| 权限控制 | 较简单 | Key Policy/IAM 细粒度控制 |
| 审计 | 基础 S3 审计 | CloudTrail 记录 KMS 使用 |
| 成本 | 较低 | 有 KMS 请求成本 |
| 典型场景 | 默认对象加密 | 合规审计、跨账号控权、客户管理密钥 |

| 对比维度 | AWS Managed Key | Customer Managed Key |
|---------|------------------|----------------------|
| 创建者 | AWS 服务自动创建 | 客户创建和管理 |
| Key Policy 控制 | 控制有限 | 客户可配置 |
| 轮换 | 服务管理 | 可配置自动轮换 |
| 跨账号使用 | 通常不适合 | 支持 |
| 典型场景 | 简单服务默认加密 | 细粒度权限、审计、跨账号 |

| 对比维度 | AWS Backup Vault Lock | S3 Object Lock |
|---------|------------------------|----------------|
| 保护对象 | AWS Backup 备份点 | S3 对象版本 |
| 核心能力 | 备份不可变保留 | 对象 WORM 保留 |
| 典型场景 | 跨服务备份防删除 | 合规归档、对象级防删除 |
| 防护目标 | 勒索/误删备份 | 合规保留对象 |

---

## 四、高频复合场景

**场景一：公司要求所有 S3 对象加密，并能审计谁使用了加密密钥。**

推理路径：仅要求简单加密可以用 SSE-S3，但题目强调审计密钥使用，需要 KMS 与 CloudTrail 记录。应使用 SSE-KMS，并选择 Customer Managed Key 以便控制权限和审计。

最终答案：为 S3 Bucket 启用 SSE-KMS，使用 Customer Managed Key，并配置 Key Policy。

**场景二：账号 A 需要读取账号 B 中使用 SSE-KMS 加密的 S3 对象，但访问失败。**

推理路径：跨账号访问 SSE-KMS 对象需要三层权限：S3 Bucket 允许读取对象，KMS Key Policy 允许外部主体使用密钥，外部主体 IAM Policy 允许 `kms:Decrypt` 和 S3 读取。

最终答案：同时配置 Bucket Policy、KMS Key Policy 和调用方 IAM Policy。

**场景三：金融机构要求密钥必须保存在客户独占的 FIPS 140-2 Level 3 硬件模块中。**

推理路径：题目明确要求专用 HSM 和严格硬件合规，这超出普通 KMS 托管模型。CloudHSM 提供单租户硬件 HSM，客户控制密钥材料。

最终答案：使用 AWS CloudHSM，而不是默认 KMS。

**场景四：公司要为 CloudFront 自定义域名配置 HTTPS 证书，但在本地区域创建的证书不可用。**

推理路径：CloudFront 是全球服务，但要求使用位于 us-east-1 的 ACM 证书。本地区域证书可用于 ALB/API Gateway 区域资源，但不能用于 CloudFront。

最终答案：在 us-east-1 申请或导入 ACM 证书，并绑定到 CloudFront Distribution。

**场景五：安全团队需要发现 S3 中是否存储了身份证号、信用卡号等敏感数据，并持续生成发现结果。**

推理路径：这是敏感数据发现和分类问题。Macie 专门用于扫描 S3 中的 PII 和敏感数据，并生成 Findings。

最终答案：启用 Amazon Macie 扫描目标 S3 Bucket。

**场景六：公司要求备份保留 7 年，任何管理员都不能在保留期内删除备份。**

推理路径：这是不可变备份和合规保留要求。AWS Backup 集中管理备份，Backup Vault Lock 可将备份库设置为不可变保留，Compliance 类模式下可防止提前删除。

最终答案：使用 AWS Backup 管理备份计划，并启用 Backup Vault Lock。

---

## 五、本 Task 一句话总结

- **静态加密看 KMS，传输中加密看 TLS/ACM。**
- **SSE-S3 简单默认加密，SSE-KMS 解决密钥控制和审计。**
- **KMS 是大多数 AWS 加密题默认答案，CloudHSM 只在专用硬件 HSM 要求下选择。**
- **跨账号解密 SSE-KMS 数据，要同时放通资源策略、Key Policy 和 IAM Policy。**
- **敏感数据发现用 Macie，不可变备份用 Backup Vault Lock 或 S3 Object Lock。**
