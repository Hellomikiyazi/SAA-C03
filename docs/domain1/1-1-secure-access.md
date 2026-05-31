# Task 1.1：设计对 AWS 资源的安全访问

---

## 一、Task 考试地图

本 Task 的核心知识点按考试权重排列如下：

- IAM User、Group、Role、Policy 的职责边界
- IAM Role 与 STS AssumeRole 的临时凭证模型
- 最小权限原则与策略评估逻辑
- Identity-Based Policy、Resource-Based Policy、Permission Boundary、SCP 的区别
- 跨账号访问与第三方访问的安全设计
- 企业联合身份认证：IAM Identity Center、SAML、Active Directory
- 应用终端用户身份：Amazon Cognito
- Root 账号保护与 MFA

> **本 Task 最高频混淆对：** IAM Role（临时凭证/服务与跨账号访问） vs IAM User（长期凭证/人或遗留程序）；SCP（权限上限） vs IAM Policy（实际授权）。

---

## 二、核心概念解析

### 1. IAM 核心对象

IAM 是 AWS 权限体系的基础，解决"谁可以对什么资源做什么操作"。IAM User 代表长期身份，拥有长期凭证；Group 用于批量管理用户权限；Role 是可被临时扮演的身份，不绑定固定人或长期密钥；Policy 是 JSON 权限文档，定义允许或拒绝的 Action、Resource 和 Condition。

考试中最重要的判断是：**AWS 服务访问 AWS 服务，不要使用 IAM User Access Key，而要使用 IAM Role**。例如 EC2 访问 S3 应创建 IAM Role 并通过 Instance Profile 挂到 EC2；Lambda 访问 DynamoDB 应给 Lambda Execution Role 授权。

> 考试判断：看到"在 EC2 上保存 Access Key"、"在应用代码中硬编码密钥" → 排除；正确答案通常是 IAM Role。

---

### 2. IAM Role 与 STS

IAM Role 的本质是临时身份。主体通过 AWS STS 调用 AssumeRole 获得短期安全凭证，凭证到期自动失效。Role 常用于三类场景：AWS 服务代表你访问其他 AWS 服务、跨账号访问、联合身份认证后访问 AWS。

跨账号访问的标准模式是：目标账号创建 Role，并在信任策略中允许源账号主体 AssumeRole；源账号主体获得临时凭证后访问目标资源。这样不需要在目标账号创建 IAM User，也不需要分发长期 Access Key。

第三方 SaaS 代表客户访问 AWS 时，还应使用 External ID 防止混乱代理人问题。External ID 是第三方和客户之间约定的唯一值，用于确保第三方不能被其他客户诱导去访问错误账号。

> 考试判断：跨账号访问 → STS AssumeRole；第三方代管访问 → AssumeRole + External ID；服务到服务访问 → IAM Role。

---

### 3. 最小权限与策略评估

最小权限原则要求只授予完成任务所需的最小权限集合。策略应尽量限定具体 Action、Resource 和 Condition，而不是使用 `*` 或 AdministratorAccess。

IAM 策略评估遵循固定优先级：**显式 Deny > 显式 Allow > 默认隐式 Deny**。只要任意策略中出现适用的显式 Deny，请求一定失败。没有 Allow 时默认拒绝。

权限来源可能包括 Identity-Based Policy、Resource-Based Policy、Permission Boundary、Session Policy、SCP。考试通常不会要求完整计算所有策略，但会考一个关键事实：边界类策略（Permission Boundary、SCP）不会直接授予权限，只会限制最大可用权限。

> 考试判断：题目问"为什么 IAM Policy 已允许但仍无法访问" → 检查显式 Deny、SCP、Permission Boundary、资源策略或 KMS Key Policy。

---

### 4. Identity-Based Policy 与 Resource-Based Policy

Identity-Based Policy 附加在 IAM User、Group 或 Role 上，描述这个身份能做什么。Resource-Based Policy 附加在资源上，描述谁能访问这个资源。S3 Bucket Policy、SQS Queue Policy、KMS Key Policy、Lambda Resource Policy 都属于资源策略。

跨账号访问常常需要两侧配合：调用方身份要有权限，资源方也要允许访问。对于 S3 这类支持资源策略的服务，可以通过 Bucket Policy 直接授权外部账号；对于不支持资源策略的服务，通常使用 AssumeRole。

KMS Key Policy 是特殊高频点。即使 IAM Policy 允许使用某个 KMS Key，如果 Key Policy 没有允许该主体或账号使用，访问也可能失败。

> 考试判断：谁能访问某个具体资源 → Resource-Based Policy；某个身份能做哪些动作 → Identity-Based Policy；跨账号 KMS 加密访问 → 同时检查 IAM Policy 和 Key Policy。

---

### 5. AWS Organizations 与 SCP

AWS Organizations 用于管理多账号环境。账号可以放入 OU，统一应用服务控制策略（SCP）。SCP 是权限上限，不是授权策略。即使账号内 IAM Policy 允许某操作，如果 SCP 不允许，最终仍会被拒绝。

SCP 最适合做组织级限制，例如禁止所有成员账号关闭 CloudTrail、禁止使用未批准的 Region、禁止删除安全审计资源。它不适合给某个用户授予 S3 访问权限，因为 SCP 不能直接 Allow 出权限。

AWS Control Tower 建立在 Organizations 之上，提供 Landing Zone、Guardrails 和账号治理模板，适合快速搭建多账号安全基线。

> 考试判断：限制整个 OU/账号能用哪些服务 → SCP；快速建立多账号治理环境 → Control Tower。

---

### 6. 联合身份认证与 IAM Identity Center

企业通常已经有身份源，如 Active Directory、Okta、Azure AD，不应在每个 AWS 账号里手工创建 IAM User。IAM Identity Center 是 AWS 推荐的多账号单点登录方案，可以把企业身份源映射到多个 AWS 账号和权限集。

SAML 2.0 Federation 适合已有 SAML 身份提供商并需要直接联合到 AWS 的场景。Directory Service 可用于连接或托管 Microsoft Active Directory。

> 考试判断：员工使用企业账号登录多个 AWS 账号 → IAM Identity Center；已有 SAML IdP → SAML Federation；不要为每个员工创建 IAM User。

---

### 7. Amazon Cognito

Cognito 面向的是应用终端用户，而不是 AWS 管理员或公司员工。User Pool 负责用户注册、登录、MFA、社交登录；Identity Pool 可以为已认证用户或匿名用户换取临时 AWS 凭证，用于访问 S3、AppSync 等资源。

考试中如果场景是移动 App 或 Web App 的用户登录、社交身份登录、用户注册和令牌管理，Cognito 通常是答案。它和 IAM Identity Center 的边界很清楚：Cognito 管应用用户，IAM Identity Center 管员工访问 AWS 账号。

> 考试判断：移动/Web 应用用户认证 → Cognito；员工登录 AWS Console → IAM Identity Center。

---

### 8. Root 账号与 MFA

Root 账号拥有账号内所有权限，不受 IAM Policy 限制。最佳实践是只在必须使用 root 的少数操作中登录，例如修改账号支持计划、关闭账号等。日常管理应创建管理员角色或用户，并启用 MFA。

Root 账号必须开启 MFA，并删除 Root Access Key。所有高权限用户也应启用 MFA。IAM Policy 可以通过 Condition 要求 MFA 存在后才允许执行敏感操作。

> 考试判断：选项中出现"使用 root 账号日常操作"或"为 root 创建 Access Key" → 通常错误。

---

## 三、服务对比速查表

| 对比维度 | IAM User | IAM Role |
|---------|----------|----------|
| 凭证类型 | 长期凭证 | STS 临时凭证 |
| 是否绑定实体 | 绑定具体用户或应用 | 被服务、用户、账号临时扮演 |
| 典型场景 | 遗留程序或少量人类用户 | EC2/Lambda 访问 AWS、跨账号访问 |
| 安全性 | 需要管理密钥轮换 | 凭证自动过期，更安全 |
| 考试关键词 | Access Key、长期凭证 | 临时凭证、AssumeRole、Instance Profile |

| 对比维度 | IAM Policy | SCP |
|---------|------------|-----|
| 作用对象 | IAM 身份或资源 | Organization OU/Account |
| 是否授予权限 | 可以授予 Allow | 不能直接授予权限 |
| 核心作用 | 定义实际可用权限 | 定义权限上限 |
| 典型场景 | 允许 Role 访问 S3 | 禁止整个 OU 使用某 Region |

| 对比维度 | IAM Identity Center | Cognito |
|---------|---------------------|---------|
| 面向对象 | 企业员工/管理员 | 应用终端用户 |
| 主要用途 | 多账号 AWS Console/CLI SSO | Web/Mobile 用户注册登录 |
| 身份源 | AD、Okta、Azure AD 等 | 用户池、社交 IdP、SAML/OIDC |
| 考试关键词 | 员工单点登录 AWS | App 用户登录、社交登录 |

| 对比维度 | Identity-Based Policy | Resource-Based Policy |
|---------|----------------------|-----------------------|
| 附加位置 | IAM User/Group/Role | S3、SQS、KMS、Lambda 等资源 |
| 回答问题 | 这个身份能做什么 | 谁能访问这个资源 |
| 跨账号访问 | 常与 AssumeRole 组合 | 可直接指定外部主体 |
| 典型服务 | IAM Role Policy | S3 Bucket Policy、KMS Key Policy |

---

## 四、高频复合场景

**场景一：EC2 实例需要读取 S3 对象，开发者计划把 Access Key 写入配置文件。**

推理路径：EC2 是 AWS 服务上的工作负载，访问 S3 应使用 IAM Role 提供临时凭证。Access Key 写入配置文件是长期凭证泄漏风险，且需要人工轮换。

最终答案：创建最小权限 IAM Role，通过 Instance Profile 关联到 EC2，授予所需 S3 读权限。

**场景二：账号 A 的 Lambda 需要读取账号 B 的 DynamoDB 表。**

推理路径：这是跨账号访问。账号 B 创建 IAM Role，信任账号 A 的 Lambda 执行角色或账号 A 指定主体；账号 A 的 Lambda 调用 STS AssumeRole 获取临时凭证，再访问 DynamoDB。

最终答案：使用 STS AssumeRole 跨账号访问，不在账号 B 中创建长期 IAM User。

**场景三：公司有多个 AWS 账号，希望员工用企业 AD 账号登录不同账号，并按岗位获得不同权限。**

推理路径：这是企业员工访问 AWS 多账号，标准方案是 IAM Identity Center。它可以连接企业身份源，为不同账号分配 Permission Set。

最终答案：配置 IAM Identity Center 对接企业身份源，并通过 Permission Set 授权。

**场景四：安全团队希望禁止某个 OU 下所有账号在未批准 Region 创建资源。**

推理路径：这是组织级权限上限控制，不是给某个用户授权。应使用 Organizations SCP 限制可用 Region。SCP 不授予权限，只阻止不合规操作。

最终答案：在目标 OU 上应用 SCP，通过 Condition 限制 `aws:RequestedRegion`。

**场景五：移动 App 用户需要使用 Google 登录，并上传自己的头像到 S3。**

推理路径：这是应用终端用户认证，不是员工访问 AWS。Cognito User Pool 管理登录和联邦身份，Identity Pool 可为用户换取临时 AWS 凭证访问指定 S3 前缀。

最终答案：使用 Cognito User Pool + Identity Pool，并用 IAM Role 限制用户只能访问自己的 S3 路径。

---

## 五、本 Task 一句话总结

- **AWS 服务访问 AWS 服务用 IAM Role**，不要硬编码 Access Key。
- **跨账号访问用 STS AssumeRole**，第三方访问加 External ID。
- **显式 Deny 优先于一切 Allow**，SCP 和 Permission Boundary 只是权限上限。
- **员工 SSO 用 IAM Identity Center，应用用户登录用 Cognito。**
- **Root 账号只用于极少数账号级操作，必须启用 MFA 并删除 Access Key。**
