# Domain 1：设计安全架构

**权重：30%（考试最高权重域）**

Domain 1 是 SAA-C03 中权重最高的部分，核心不是背安全服务清单，而是判断：**谁可以访问什么资源、网络边界如何控制、数据如何加密和保护、密钥和凭证如何管理**。考试题通常会给出一个看似可行但有安全隐患的架构，让你选择更符合 AWS 最佳实践的方案。

---

## 一、本域三大任务

| Task | 内容 | 核心问题 | 高频服务 |
|------|------|----------|----------|
| 1.1 | 设计对 AWS 资源的安全访问 | 谁能访问？如何授权？跨账号如何安全访问？ | IAM、STS、IAM Identity Center、Organizations、SCP、Cognito |
| 1.2 | 设计安全的工作负载与应用 | 网络和应用入口如何防护？凭证如何安全使用？ | VPC、Security Group、NACL、WAF、Shield、Secrets Manager、PrivateLink |
| 1.3 | 确定适当的数据安全控制 | 数据如何加密、分类、审计和防泄漏？ | KMS、CloudHSM、ACM、S3 加密、Macie、AWS Backup |

> **本域最高频混淆点：** IAM Role（临时授权） vs IAM User（长期凭证）；Security Group（实例级有状态防火墙） vs NACL（子网级无状态 ACL）；KMS（托管密钥管理） vs CloudHSM（专用硬件 HSM）。

---

## 二、Domain 1 的设计主线

### 1. 身份优先：先确认主体是谁

所有安全访问题的第一步是识别访问主体：是人、应用、AWS 服务、外部账号，还是移动/Web 用户。不同主体对应不同方案：员工访问多账号优先 IAM Identity Center；EC2/Lambda 访问 AWS 服务使用 IAM Role；跨账号访问使用 STS AssumeRole；外部用户登录应用使用 Cognito。

考试中看到 Access Key 写在代码、配置文件、EC2 本地文件、Lambda 环境变量里，通常都是反模式。正确答案往往是用 IAM Role 或 Secrets Manager 消除长期明文凭证。

---

### 2. 最小权限：只给完成任务所需的权限

最小权限（Least Privilege）是 Domain 1 的核心判断原则。策略应该限制 Action、Resource 和必要的 Condition，而不是直接授予 `AdministratorAccess` 或使用 root 账号。

SCP、Permission Boundary、IAM Policy、Resource Policy 都可能参与权限判断。考试不要求手写复杂策略，但要求理解边界：SCP 不授予权限，只设置账号或 OU 的权限上限；显式 Deny 永远优先于 Allow；资源策略常用于 S3、SQS、KMS 等跨账号访问。

---

### 3. 网络分层：入口、子网、实例逐层防护

网络安全题通常围绕 VPC 边界展开。公共子网放 ALB、NAT Gateway、Bastion 等入口资源；私有子网放 EC2、数据库、内部服务；数据库子网不直接暴露公网。Security Group 控制资源级入站/出站，NACL 控制子网级流量。

考试中看到数据库在 public subnet、EC2 直接开放 SSH/RDP 到 0.0.0.0/0、私有应用通过公网访问 AWS 服务，要优先考虑私有子网、Systems Manager Session Manager、VPC Endpoint 或 PrivateLink。

---

### 4. 数据保护：加密、密钥、证书、审计分开判断

数据安全题要分清四类问题：数据静态加密、传输中加密、密钥管理、敏感数据发现。S3/EBS/RDS/DynamoDB 等服务支持静态加密，TLS/ACM 解决传输中加密和证书管理，KMS 管理大多数加密密钥，Macie 发现 S3 中的敏感数据。

如果题目要求客户完全控制 HSM、满足 FIPS 140-2 Level 3、或使用单租户硬件安全模块，才考虑 CloudHSM。普通 AWS 服务加密场景优先 KMS。

---

## 三、Task 边界速查

| 判断问题 | 更偏 Task 1.1 | 更偏 Task 1.2 | 更偏 Task 1.3 |
|---------|---------------|---------------|---------------|
| 核心矛盾 | 权限、身份、跨账号访问 | 网络边界、应用防护、凭证使用 | 数据加密、密钥、证书、备份保护 |
| 常见关键词 | IAM Role、AssumeRole、SSO、SCP、Federation | VPC、SG、NACL、WAF、Shield、Private subnet | KMS、CMK、SSE、TLS、ACM、Macie |
| 典型错误选项 | 创建长期 IAM User / 使用 root | 数据库放公网 / 0.0.0.0/0 开 SSH | 自管密钥但无轮换 / 未加密敏感数据 |
| 标准思路 | 临时凭证 + 最小权限 | 分层网络 + 私有访问 + 托管防护 | 默认加密 + KMS 控制 + 审计发现 |

简单说：**1.1 管"谁能访问"，1.2 管"从哪里进来和怎么防护"，1.3 管"数据和密钥怎么保护"。**

---

## 四、本域核心服务判断表

| 场景关键词 | 首选服务/方案 | 判断理由 |
|-----------|---------------|----------|
| EC2 需要访问 S3 | IAM Role / Instance Profile | 临时凭证，避免 Access Key |
| Lambda 访问 DynamoDB | Lambda Execution Role | 服务角色授权 |
| 跨账号访问资源 | STS AssumeRole | 临时跨账号授权 |
| 企业员工单点登录 AWS | IAM Identity Center | 多账号 SSO 集中管理 |
| 限制整个 OU 能使用的服务 | Organizations SCP | 设置权限上限，不直接授权 |
| 移动/Web 用户登录应用 | Cognito | 面向应用终端用户的身份池/用户池 |
| Web 应用防 SQL 注入/XSS | AWS WAF | L7 Web 攻击防护 |
| DDoS 基础防护 | Shield Standard | 默认启用，无额外费用 |
| 私有访问 S3/DynamoDB | Gateway VPC Endpoint | 不经公网访问区域服务 |
| 私有访问第三方/自有服务 | PrivateLink | 私有连接到 Endpoint Service |
| 应用数据库密码管理 | Secrets Manager | 密钥轮换和托管读取 |
| 大多数 AWS 服务加密密钥 | AWS KMS | 托管密钥服务，集成广 |
| 专用硬件 HSM 合规要求 | CloudHSM | 单租户硬件控制 |
| TLS 证书申请和轮换 | ACM | 托管公有证书 |
| 发现 S3 敏感数据 | Macie | 识别 PII/敏感数据 |

---

## 五、本域考试速记

- **不要在代码、EC2 或配置文件里放 Access Key**，AWS 服务访问 AWS 服务优先用 IAM Role。
- **显式 Deny 永远优先**，SCP 只限制上限，不直接授予权限。
- **Security Group 有状态，NACL 无状态**；SG 绑资源，NACL 绑子网。
- **Web 攻击用 WAF，DDoS 用 Shield，网络私有访问用 VPC Endpoint/PrivateLink。**
- **KMS 是默认密钥答案，CloudHSM 只在专用 HSM/严格合规要求下出现。**
