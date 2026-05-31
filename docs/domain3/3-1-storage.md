# Task 3.1：设计高性能存储架构

---

## 一、Task 考试地图

本 Task 的核心知识点按考试权重排列如下：

- S3、EBS、EFS、FSx 的存储模型差异
- EBS gp3、io2、st1、sc1 的 IOPS、吞吐和适用场景
- S3 性能优化：前缀并行、Multipart Upload、Transfer Acceleration、CloudFront
- EFS 性能模式与吞吐模式
- FSx for Windows、FSx for Lustre、FSx for NetApp ONTAP、FSx for OpenZFS 的选择逻辑
- Storage Gateway 在混合云存储中的作用
- 数据生命周期与归档层对性能和访问延迟的影响

> **本 Task 最高频混淆对：** EBS（单 EC2 块存储） vs EFS（多 EC2 共享文件系统）；S3（对象存储） vs EFS（文件存储）；FSx for Lustre（HPC 高吞吐） vs EFS（通用共享文件）。

---

## 二、核心概念解析

### 1. 存储类型选择的第一原则

高性能存储题不要先背服务名，而要先判断应用需要的是对象、块还是文件。

**对象存储** 适合以完整对象读写数据，例如图片、视频、备份、日志、静态网站文件。Amazon S3 是标准答案。它不是 POSIX 文件系统，也不能像磁盘一样被操作系统直接挂载为块设备。

**块存储** 适合作为单台 EC2 的磁盘，例如数据库数据盘、启动卷、低延迟随机 I/O 工作负载。Amazon EBS 是标准答案。EBS 卷通常一次只能挂载到同一个 AZ 内的一台实例，Multi-Attach 只是少数 io1/io2 特殊场景，不是通用共享文件方案。

**文件存储** 适合多台实例通过文件路径共享访问，例如内容管理系统、共享主目录、机器学习训练数据集。Linux/NFS 场景选 EFS，Windows/SMB 场景选 FSx for Windows，高性能并行计算选 FSx for Lustre。

> 考试判断：静态对象和备份 → S3；单台 EC2 磁盘 → EBS；多台 Linux EC2 共享文件 → EFS；Windows 文件共享 → FSx for Windows；HPC 并行文件系统 → FSx for Lustre。

---

### 2. Amazon S3 性能设计

S3 是高持久性、高扩展性的对象存储。它适合海量对象和高并发访问，常用于静态网站、数据湖、备份、日志归档和媒体文件。

S3 的性能优化通常有四个方向。

第一是**并行化访问**。S3 可以按前缀自动扩展请求速率，现代 S3 不再要求随机前缀命名来避免热点，但高吞吐应用仍应通过多个并发连接、分片对象和并行请求提升吞吐。

第二是**Multipart Upload**。大对象上传时应使用分段上传，让多个 part 并行上传，失败时只重传失败片段，提升速度并降低重试成本。

第三是**Transfer Acceleration**。如果全球用户需要跨远距离上传对象到同一个 S3 Bucket，可以通过 CloudFront 边缘网络加速上传路径。

第四是**CloudFront 缓存**。如果是全球用户下载静态内容，首选 CloudFront，而不是让所有用户直接访问 S3 区域端点。

> 考试判断：全球下载静态内容慢 → CloudFront；全球上传到 S3 慢 → S3 Transfer Acceleration；大文件上传不稳定 → Multipart Upload。

---

### 3. Amazon EBS 高性能卷

EBS 是 EC2 的持久块存储。考试通常要求根据 IOPS、吞吐、成本和工作负载选择卷类型。

**gp3** 是通用 SSD，适合大多数启动卷、应用服务器和中等数据库负载。gp3 可以独立配置 IOPS 和吞吐，不像 gp2 那样强依赖卷大小，是通用场景的默认选择。

**io2 / io2 Block Express** 是高性能 Provisioned IOPS SSD，适合关键数据库、低延迟高 IOPS、需要高持久性的工作负载。如果题目强调非常高 IOPS、关键业务数据库、亚毫秒级延迟，通常考虑 io2。

**st1** 是吞吐优化 HDD，适合大规模顺序读写，例如日志处理、数据仓库暂存、大数据处理。它不适合小块随机 I/O。

**sc1** 是冷 HDD，适合低频访问的大容量数据，性能最低但成本低。

EBS 性能还受 EC2 实例 EBS 带宽限制影响。即使卷本身配置了高 IOPS，如果实例类型不支持足够 EBS 带宽，也无法发挥性能。

> 考试判断：通用 SSD → gp3；关键数据库高 IOPS → io2；大规模顺序吞吐 → st1；冷数据低成本 → sc1；EBS 性能不达标还要检查 EC2 实例带宽。

---

### 4. Amazon EFS 性能设计

EFS 是托管 NFS 文件系统，多个 EC2、ECS、EKS、Lambda 可以同时挂载访问。它跨多个 AZ 提供高可用，适合共享内容、共享配置、用户主目录和容器持久卷。

EFS 有两类重要性能选择。

**性能模式** 包括 General Purpose 和 Max I/O。General Purpose 延迟更低，适合大多数 Web、CMS 和普通共享文件场景。Max I/O 支持更高并发和总吞吐，但单操作延迟更高，适合大规模并行访问。

**吞吐模式** 包括 Bursting、Provisioned 和 Elastic。Bursting 根据文件系统大小获得吞吐突增能力；Provisioned 适合容量小但需要稳定高吞吐的场景；Elastic 自动按工作负载扩展吞吐，适合访问模式不可预测。

> 考试判断：多台 Linux EC2 共享文件 → EFS；大量客户端并行访问 → Max I/O；容量小但需要高吞吐 → Provisioned/Elastic Throughput。

---

### 5. Amazon FSx 系列

FSx 是托管高性能文件系统家族，考试重点是按协议和工作负载选择。

**FSx for Windows File Server** 提供 SMB 文件共享，支持 Windows ACL 和 Active Directory 集成，适合 Windows 应用、用户共享目录、企业文件服务器迁移。

**FSx for Lustre** 是高性能并行文件系统，适合 HPC、机器学习训练、媒体渲染、金融建模等高吞吐场景。它可以和 S3 集成，把 S3 数据集呈现为高性能文件系统。

**FSx for NetApp ONTAP** 适合需要 ONTAP 特性、NFS/SMB/iSCSI、多协议访问、快照和数据管理能力的企业迁移场景。

**FSx for OpenZFS** 适合需要 ZFS 特性、低延迟 NFS、快照和克隆能力的 Linux 文件工作负载。

> 考试判断：Windows SMB/AD → FSx for Windows；HPC/ML 高吞吐并行文件 → FSx for Lustre；NetApp 迁移或多协议企业文件 → FSx for ONTAP。

---

### 6. Storage Gateway

Storage Gateway 用于混合云存储，把本地环境和 AWS 存储服务连接起来。它常见于本地应用还不能直接改造到云端，但希望使用 S3、EBS Snapshot 或云端归档。

**File Gateway** 通过 NFS/SMB 暴露文件接口，后端把文件作为对象存入 S3，适合本地文件应用逐步使用云存储。

**Volume Gateway** 以 iSCSI 块存储形式给本地服务器使用，并将数据快照到 AWS，适合本地块存储备份和恢复。

**Tape Gateway** 替代物理磁带库，把备份软件写入的虚拟磁带归档到 AWS。

> 考试判断：本地应用通过 NFS/SMB 写入并落到 S3 → File Gateway；本地块卷备份到 AWS → Volume Gateway；替代磁带备份 → Tape Gateway。

---

## 三、服务对比速查表

| 对比维度 | S3 | EBS | EFS |
|---------|----|-----|-----|
| 存储模型 | 对象存储 | 块存储 | 文件存储 |
| 典型挂载方式 | API/HTTP 访问 | 挂载到单台 EC2 | 多客户端 NFS 挂载 |
| 共享访问 | 通过对象 API 并发访问 | 通常不用于多实例共享 | 原生多实例共享 |
| 典型场景 | 静态内容、备份、数据湖 | 数据库磁盘、启动卷 | CMS、共享目录、容器持久卷 |
| 考试关键词 | 对象、Bucket、静态网站 | IOPS、块设备、单实例 | NFS、多 EC2 共享 |

| 对比维度 | EBS gp3 | EBS io2 | EBS st1 |
|---------|---------|---------|---------|
| 介质 | SSD | Provisioned IOPS SSD | HDD |
| 优化目标 | 通用性价比 | 极高 IOPS 和低延迟 | 高顺序吞吐 |
| 典型场景 | 启动卷、普通数据库 | 关键数据库、事务系统 | 日志、大数据顺序处理 |
| 不适合 | 极端 IOPS | 低成本冷数据 | 小块随机 I/O |

| 对比维度 | EFS | FSx for Lustre | FSx for Windows |
|---------|-----|---------------|-----------------|
| 协议 | NFS | Lustre | SMB |
| 主要平台 | Linux | HPC/Linux | Windows |
| 性能特点 | 通用共享文件 | 极高吞吐并行 I/O | Windows 文件共享 |
| 典型场景 | Web 共享内容 | ML/HPC/渲染 | AD 集成文件服务器 |

---

## 四、高频复合场景

**场景一：多台 EC2 运行同一个内容管理系统，需要共享用户上传文件，并且实例分布在多个 AZ。**

推理路径：多台实例需要共享同一套文件，排除 EBS。应用是 Linux Web/CMS 场景，通用共享文件系统优先 EFS。EFS 跨 AZ 可用，可以挂载到多个 EC2。

最终答案：使用 Amazon EFS，并在各 AZ 创建 Mount Target 供 EC2 挂载。

**场景二：金融交易数据库部署在 EC2 上，要求极高 IOPS、低延迟和高持久性。**

推理路径：数据库运行在 EC2 上，需要块存储。通用 gp3 不一定满足极高 IOPS 和关键业务持久性要求，应选择 Provisioned IOPS SSD。

最终答案：使用 EBS io2 或 io2 Block Express，并选择支持足够 EBS 带宽的 EC2 实例类型。

**场景三：机器学习团队的数据集存放在 S3，训练任务需要高吞吐并行读取文件。**

推理路径：S3 适合持久存放数据集，但训练时需要 POSIX 文件系统和高吞吐并行 I/O。FSx for Lustre 可以和 S3 集成，并为训练集提供高性能文件访问。

最终答案：使用 FSx for Lustre 关联 S3 数据集，训练任务从 FSx 读取。

**场景四：全球用户上传大视频到同一个 S3 Bucket，经常因为跨区域网络距离导致上传慢。**

推理路径：目标是跨距离上传到 S3，不是下载缓存。CloudFront 主要解决下载和内容分发，S3 Transfer Acceleration 使用边缘网络加速上传。

最终答案：为 S3 Bucket 启用 Transfer Acceleration，并使用加速端点上传。

**场景五：本地备份软件只能写磁带库，公司希望不再维护物理磁带并把归档放到 AWS。**

推理路径：本地备份软件接口是磁带，迁移目标是云端归档，不应改造应用直接写 S3。Storage Gateway 的 Tape Gateway 专门模拟虚拟磁带库。

最终答案：部署 Tape Gateway，将虚拟磁带归档到 AWS。

---

## 五、本 Task 一句话总结

- **对象用 S3，块设备用 EBS，共享文件用 EFS/FSx。**
- **EBS 是单 EC2 磁盘答案，不是多实例共享文件系统答案。**
- **gp3 是通用默认，io2 是关键高 IOPS，st1 是顺序吞吐。**
- **Linux 通用共享选 EFS，Windows SMB 选 FSx for Windows，HPC 高吞吐选 FSx for Lustre。**
- **全球下载 S3 内容用 CloudFront，全球上传到 S3 慢用 Transfer Acceleration。**
