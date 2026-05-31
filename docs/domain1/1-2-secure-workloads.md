# Task 1.2：设计安全的工作负载与应用

---

## 一、Task 考试地图

本 Task 的核心知识点按考试权重排列如下：

- VPC 公有/私有子网设计与安全边界
- Security Group 与 Network ACL 的区别
- Bastion Host、Session Manager、VPN、Direct Connect 的安全访问选择
- VPC Endpoint、Gateway Endpoint、Interface Endpoint、PrivateLink
- AWS WAF、Shield、Firewall Manager 的应用层与 DDoS 防护
- Secrets Manager、Parameter Store 在应用凭证管理中的选择
- ALB/NLB/API Gateway 的安全入口设计
- 日志审计与检测：CloudTrail、Config、GuardDuty、Security Hub

> **本 Task 最高频混淆对：** Security Group（有状态/资源级） vs NACL（无状态/子网级）；Gateway VPC Endpoint（S3/DynamoDB） vs Interface Endpoint/PrivateLink（私有访问服务）。

---

## 二、核心概念解析

### 1. VPC 分层安全架构

VPC 是工作负载网络安全的基础。典型安全架构会把入口层、应用层、数据层分开：ALB、NAT Gateway 等需要公网入口的资源放在公有子网；应用 EC2/ECS 放在私有子网；数据库放在私有数据库子网，并禁止公网访问。

公有子网的定义不是"里面的资源一定公开"，而是路由表中有到 Internet Gateway 的默认路由。私有子网没有直接到 IGW 的路由，出站访问互联网通常通过 NAT Gateway。

> 考试判断：数据库放 public subnet、EC2 直接暴露公网 SSH/RDP、应用服务器有 Public IP → 通常不是最佳实践；应改为私有子网 + 受控入口。

---

### 2. Security Group 与 NACL

Security Group 是资源级虚拟防火墙，绑定到 ENI、EC2、ALB、RDS 等资源。它是有状态的：允许入站请求后，响应流量会自动允许返回。SG 只支持 Allow 规则，不支持 Deny。

Network ACL 是子网级访问控制列表，作用于进入和离开子网的流量。它是无状态的：入站和出站都必须显式允许。NACL 支持 Allow 和 Deny，并按规则号从小到大匹配。

考试中，绝大多数应用访问控制优先使用 Security Group，因为它更细粒度、更贴近资源。NACL 更适合做子网级粗粒度拦截，例如阻断已知恶意 IP。

> 考试判断：实例级端口访问控制 → Security Group；子网级显式拒绝某 IP → NACL；临时端口返回流量被挡 → 检查 NACL 出站/入站规则。

---

### 3. 安全管理访问：Session Manager、Bastion、VPN

传统做法是通过 Bastion Host 跳板机 SSH 到私有子网实例，但它需要开放入站端口、管理密钥和维护跳板机。更推荐的方案是 AWS Systems Manager Session Manager，它不需要入站 SSH/RDP，也不需要 Bastion Public IP，可以通过 IAM、CloudTrail 和日志实现访问控制与审计。

Site-to-Site VPN 用于企业数据中心和 VPC 之间的加密网络连接，部署快但依赖互联网。Direct Connect 提供专线连接，延迟更稳定、带宽更可控，但不自带加密，如需加密可叠加 VPN。

> 考试判断：安全登录私有 EC2 且不开放 SSH → Session Manager；快速连接本地数据中心 → Site-to-Site VPN；稳定高带宽专线 → Direct Connect。

---

### 4. VPC Endpoint 与 PrivateLink

VPC Endpoint 允许私有子网中的资源不经过公网访问 AWS 服务。Gateway Endpoint 只支持 S3 和 DynamoDB，通过路由表生效，不产生 ENI。Interface Endpoint 基于 AWS PrivateLink，在子网中创建 ENI，支持大量 AWS 服务、私有 API 和第三方服务。

PrivateLink 的核心价值是让服务通过 AWS 私有网络暴露给其他 VPC 或账号，而不需要 VPC Peering、NAT Gateway、Internet Gateway 或公网 IP。它常用于 SaaS 服务私有接入、跨账号服务暴露、避免网络重叠问题。

> 考试判断：私有访问 S3/DynamoDB → Gateway Endpoint；私有访问其他 AWS 服务或 SaaS → Interface Endpoint/PrivateLink；要求流量不走公网 → VPC Endpoint。

---

### 5. AWS WAF、Shield 与 Firewall Manager

AWS WAF 是 Web 应用防火墙，工作在第 7 层，用于防护 SQL 注入、XSS、恶意 Bot、路径规则、IP 黑名单等 HTTP/HTTPS 攻击。它可以关联 CloudFront、ALB、API Gateway、AppSync。

AWS Shield Standard 默认为所有客户提供基础 DDoS 防护。Shield Advanced 提供更高级的 DDoS 防护、成本保护和 24×7 DRT 支持，适合关键互联网应用。

AWS Firewall Manager 用于在 AWS Organizations 多账号环境中集中管理 WAF、Shield Advanced、Security Group、Network Firewall 等安全策略。

> 考试判断：SQL 注入/XSS → WAF；DDoS → Shield；多账号统一下发 WAF/Shield 策略 → Firewall Manager。

---

### 6. Secrets Manager 与 Parameter Store

应用不应把数据库密码、API Key、OAuth Secret 写在代码或 AMI 中。Secrets Manager 用于管理敏感凭证，支持自动轮换，特别适合数据库密码。Systems Manager Parameter Store 可以存储配置参数和 SecureString，成本更低，适合一般配置和不需要复杂轮换的密钥。

两者都可以与 KMS 集成加密。考试中如果明确要求自动轮换数据库凭证，Secrets Manager 是标准答案；如果只是集中存储配置参数，Parameter Store 通常足够。

> 考试判断：数据库密码自动轮换 → Secrets Manager；普通配置和分层参数 → Parameter Store；硬编码凭证 → 错误。

---

### 7. 安全入口：ALB、API Gateway、CloudFront

安全入口设计通常把公网流量集中到托管入口层，而不是直接暴露后端实例。ALB 适合 Web 应用和微服务，可与 WAF、ACM、Security Group 集成。API Gateway 适合发布 API，支持认证授权、限流、API Key、请求验证和 WAF。CloudFront 可以作为全球边缘入口，与 WAF、Origin Access Control、签名 URL/Cookie 结合保护源站。

> 考试判断：后端 EC2 不应直接暴露公网；公网入口放 ALB/API Gateway/CloudFront，后端放私有子网。

---

### 8. 检测、审计与合规

CloudTrail 记录 AWS API 调用，是审计"谁在什么时候做了什么"的核心服务。AWS Config 记录资源配置变更，并可用规则检查合规性。GuardDuty 使用威胁情报和机器学习检测恶意行为，如异常 API 调用、可疑网络通信、凭证泄露迹象。Security Hub 汇总多个安全服务发现，提供统一安全态势视图。

> 考试判断：API 审计 → CloudTrail；资源配置合规 → Config；威胁检测 → GuardDuty；集中安全发现 → Security Hub。

---

## 三、服务对比速查表

| 对比维度 | Security Group | NACL |
|---------|----------------|------|
| 作用层级 | 资源/ENI 级 | 子网级 |
| 状态 | 有状态 | 无状态 |
| 规则类型 | 只支持 Allow | 支持 Allow 和 Deny |
| 规则评估 | 所有规则共同生效 | 按规则号顺序匹配 |
| 典型场景 | 控制 EC2/RDS/ALB 访问 | 子网级封禁 IP |

| 对比维度 | Gateway Endpoint | Interface Endpoint |
|---------|------------------|--------------------|
| 支持服务 | S3、DynamoDB | 大量 AWS 服务、私有服务、SaaS |
| 实现方式 | 路由表目标 | 子网 ENI + PrivateLink |
| 成本 | 通常无 Endpoint 小时费 | 有 ENI/数据处理费用 |
| 典型场景 | 私有访问 S3/DynamoDB | 私有访问 Secrets Manager、CloudWatch、第三方服务 |

| 对比维度 | WAF | Shield |
|---------|-----|--------|
| 防护层级 | 第 7 层 Web 攻击 | DDoS 攻击 |
| 典型攻击 | SQL 注入、XSS、Bot、路径规则 | L3/L4/L7 DDoS |
| 关联资源 | CloudFront、ALB、API Gateway | CloudFront、Route 53、ELB、Global Accelerator 等 |
| 考试关键词 | Web ACL、托管规则 | DDoS、DRT、成本保护 |

| 对比维度 | Secrets Manager | Parameter Store |
|---------|-----------------|-----------------|
| 核心用途 | 敏感凭证管理 | 配置与参数管理 |
| 自动轮换 | 原生支持 | 需自定义 |
| 典型场景 | RDS 密码、API Secret | 环境配置、普通 SecureString |
| 成本 | 较高 | 标准参数成本低 |

---

## 四、高频复合场景

**场景一：公司要求运维人员访问私有子网 EC2，但不能开放 22 端口，也不能维护 Bastion Host。**

推理路径：目标是安全管理访问，且明确不能开放 SSH 或维护跳板机。Session Manager 通过 SSM Agent 出站连接 AWS 服务，使用 IAM 控制权限，并可记录会话日志。

最终答案：使用 Systems Manager Session Manager，并为实例配置 SSM Role 和必要 VPC Endpoint。

**场景二：私有子网中的 EC2 需要访问 S3，安全团队要求流量不能经过公网。**

推理路径：访问目标是 S3，且要求不走公网。S3 支持 Gateway VPC Endpoint，通过路由表让私有流量直达 S3。

最终答案：创建 S3 Gateway VPC Endpoint，更新私有子网路由表和必要的 Endpoint Policy。

**场景三：Web 应用频繁遭遇 SQL 注入和 XSS 攻击，部署在 ALB 后面。**

推理路径：SQL 注入和 XSS 是第 7 层 Web 攻击，应使用 WAF。WAF 可以直接关联 ALB，并使用 AWS Managed Rules 快速防护常见攻击。

最终答案：在 ALB 上关联 AWS WAF Web ACL，启用托管规则和必要自定义规则。

**场景四：数据库密码被硬编码在 Lambda 环境变量中，公司要求自动轮换密码。**

推理路径：硬编码敏感凭证是安全反模式；自动轮换是 Secrets Manager 的核心能力。Lambda 应在运行时读取 Secret，并通过 IAM Role 获得最小权限。

最终答案：将数据库密码迁移到 Secrets Manager，启用自动轮换，Lambda 使用执行角色读取 Secret。

**场景五：多账号环境中，安全团队需要统一给所有互联网 ALB 配置 WAF 托管规则。**

推理路径：这是 Organizations 下多账号集中安全策略管理。单个账号手工配置 WAF 不可扩展，Firewall Manager 可以集中下发和强制执行 WAF 策略。

最终答案：使用 AWS Firewall Manager 在组织范围内管理 WAF 策略。

---

## 五、本 Task 一句话总结

- **Security Group 有状态、绑资源；NACL 无状态、绑子网。**
- **私有访问 S3/DynamoDB 用 Gateway Endpoint，私有访问多数服务用 Interface Endpoint/PrivateLink。**
- **Web 攻击用 WAF，DDoS 用 Shield，多账号统一策略用 Firewall Manager。**
- **不要开放 SSH/RDP 管理私有实例，优先用 Session Manager。**
- **应用密钥不要硬编码，自动轮换用 Secrets Manager。**
