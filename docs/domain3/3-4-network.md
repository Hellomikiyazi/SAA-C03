# Task 3.4：设计高性能网络架构

---

## 一、Task 考试地图

本 Task 的核心知识点按考试权重排列如下：

- CloudFront、Global Accelerator、Route 53 延迟路由的区别
- ALB、NLB、Gateway Load Balancer 的性能与协议边界
- VPC Endpoint、PrivateLink 对私有访问和路径优化的作用
- Direct Connect、VPN、Transit Gateway 在混合网络中的选择
- ENA、增强联网、实例网络带宽对 EC2 性能的影响
- 跨 AZ、跨 Region、跨公网路径对延迟和数据传输的影响
- API Gateway、CloudFront、ALB 在 API 加速中的定位

> **本 Task 最高频混淆对：** CloudFront（内容缓存/CDN） vs Global Accelerator（动态 TCP/UDP 加速）；Route 53 Latency-based Routing（DNS 就近解析） vs Global Accelerator（Anycast 固定入口和 AWS 骨干网加速）；ALB（HTTP 七层） vs NLB（TCP/UDP 四层）。

---

## 二、核心概念解析

### 1. 网络性能优化的第一原则

网络性能题要先判断慢在哪里：用户离源站远、内容可缓存、动态请求路径差、服务间走公网、混合云链路不稳定，还是负载均衡器协议选错。

如果内容可缓存，优先 CloudFront。如果是动态 TCP/UDP 应用且全球用户路径不稳定，考虑 Global Accelerator。如果只是希望 DNS 把用户解析到最近 Region，可用 Route 53 Latency-based Routing。如果私有子网访问 AWS 服务绕公网，应使用 VPC Endpoint。如果本地到 AWS 需要稳定带宽和低抖动，应使用 Direct Connect。

> 考试判断：能缓存的全球内容 → CloudFront；不能缓存的全球动态 TCP/UDP → Global Accelerator；DNS 就近选 Region → Route 53 Latency-based；私有访问 AWS 服务 → VPC Endpoint。

---

### 2. CloudFront

CloudFront 是 CDN，通过全球边缘节点缓存内容并终止 TLS，减少用户到源站的距离。源站可以是 S3、ALB、EC2、API Gateway 或自定义 HTTP 服务器。

CloudFront 最适合静态内容，例如图片、视频、CSS、JavaScript、下载文件。它也可以加速动态内容，因为 TCP/TLS 连接可以在边缘优化，但如果内容完全不可缓存且是非 HTTP TCP/UDP，Global Accelerator 通常更贴切。

CloudFront 还常用于保护 S3 源站。通过 Origin Access Control（OAC）让用户只能经 CloudFront 访问 S3，而不能直接访问 Bucket。

> 考试判断：全球用户访问 S3 静态网站慢 → CloudFront；降低 ALB/S3 源站压力 → CloudFront 缓存；限制用户直接访问 S3 → CloudFront + OAC。

---

### 3. Global Accelerator

Global Accelerator 提供两个 Anycast 静态 IP，把用户流量引入最近的 AWS 边缘位置，然后走 AWS 全球骨干网到后端 ALB、NLB、EC2 或 Elastic IP。它适合对网络路径稳定性和动态请求延迟敏感的全球应用。

它与 CloudFront 的根本区别是：CloudFront 是 HTTP CDN，核心是缓存和内容分发；Global Accelerator 是网络层加速，核心是 Anycast 入口、快速故障切换和优化动态流量路径。

> 考试判断：全球游戏、VoIP、动态 API、TCP/UDP 应用需要固定入口 IP 和低延迟 → Global Accelerator。

---

### 4. Route 53 延迟路由

Route 53 Latency-based Routing 根据用户到 AWS Region 的延迟，把 DNS 查询结果返回给延迟最低的区域端点。它适合多 Region 部署的应用，希望用户访问最近区域。

它和 Global Accelerator 的区别在于：Route 53 是 DNS 层决策，返回不同端点，受 DNS 缓存和客户端行为影响；Global Accelerator 提供固定 Anycast IP，连接级别通过 AWS 网络优化路径并能更快切换端点。

> 考试判断：应用已经多 Region 部署，只需 DNS 层就近解析 → Route 53 Latency-based；需要固定 IP、快速切换、动态流量加速 → Global Accelerator。

---

### 5. ELB 选择：ALB、NLB、GWLB

**ALB** 工作在第 7 层，理解 HTTP/HTTPS/gRPC，支持路径、Host、Header 路由，适合 Web、微服务、容器和 HTTP API。

**NLB** 工作在第 4 层，支持 TCP、UDP、TLS，面向极高吞吐、超低延迟和静态 IP。适合非 HTTP 协议、游戏、金融协议、IoT TCP 入口。

**Gateway Load Balancer（GWLB）** 用于部署和横向扩展第三方网络虚拟设备，例如防火墙、入侵检测、深度包检测。它不是普通 Web 负载均衡器。

> 考试判断：HTTP 路径路由 → ALB；TCP/UDP/静态 IP/低延迟 → NLB；透明插入安全设备 → GWLB。

---

### 6. VPC Endpoint 与 PrivateLink

VPC Endpoint 让 VPC 内资源私有访问 AWS 服务或 Endpoint Service，不需要经过 Internet Gateway、NAT Gateway 或公网 IP。

**Gateway Endpoint** 支持 S3 和 DynamoDB，路由表中添加目标即可，常用于私有子网访问 S3/DynamoDB。

**Interface Endpoint** 基于 AWS PrivateLink，为大多数 AWS 服务、第三方服务或自建 Endpoint Service 提供私有弹性网卡入口。

性能和安全角度看，Endpoint 可以减少公网绕路、降低 NAT Gateway 依赖，并保持流量在 AWS 网络内。

> 考试判断：私有子网访问 S3/DynamoDB → Gateway Endpoint；私有访问其他 AWS 服务或第三方服务 → Interface Endpoint/PrivateLink。

---

### 7. Direct Connect 与 VPN

Site-to-Site VPN 通过互联网建立加密隧道，部署快、成本低，但延迟和抖动受公网影响。Direct Connect 提供专用网络连接，带宽稳定、延迟更可预测，适合高吞吐、稳定混合云连接。

Direct Connect 本身不加密链路，如果题目要求专线稳定且传输加密，可以在 Direct Connect 上叠加 VPN 或使用应用层加密。

Transit Gateway 用于连接多个 VPC、VPN 和 Direct Connect，作为中心路由枢纽，适合大规模网络拓扑。

> 考试判断：快速低成本连接本地和 AWS → VPN；稳定高带宽低抖动 → Direct Connect；大量 VPC 和本地网络集中互联 → Transit Gateway。

---

## 三、服务对比速查表

| 对比维度 | CloudFront | Global Accelerator |
|---------|------------|-------------------|
| 工作层次 | CDN/HTTP 内容分发 | 网络层 Anycast 加速 |
| 核心价值 | 缓存内容，降低源站压力 | 优化动态流量路径和故障切换 |
| 适合协议 | HTTP/HTTPS | TCP/UDP |
| 入口 | 域名 | 两个静态 Anycast IP |
| 考试关键词 | 静态内容、边缘缓存、S3/ALB 源站 | 固定 IP、全球动态应用、游戏、低延迟 |

| 对比维度 | Route 53 Latency-based | Global Accelerator |
|---------|------------------------|-------------------|
| 决策层 | DNS 解析 | 网络连接路径 |
| 端点暴露 | 返回不同区域端点 | 固定 Anycast IP |
| 切换速度 | 受 DNS TTL 影响 | 更快端点故障切换 |
| 适合场景 | 多 Region DNS 就近 | 全球动态应用路径优化 |

| 对比维度 | ALB | NLB | GWLB |
|---------|-----|-----|------|
| 层级 | L7 | L4 | L3/L4 设备插入 |
| 协议 | HTTP/HTTPS/gRPC | TCP/UDP/TLS | GENEVE |
| 主要用途 | Web/微服务路由 | 高性能网络入口 | 第三方防火墙/检测设备 |
| 高频关键词 | Path/Host/Header | 静态 IP、低延迟 | 网络虚拟设备 |

| 对比维度 | VPN | Direct Connect |
|---------|-----|----------------|
| 链路 | 公网加密隧道 | 专用网络连接 |
| 部署速度 | 快 | 较慢，需要线路交付 |
| 性能稳定性 | 受互联网影响 | 更稳定、低抖动 |
| 典型场景 | 快速混合连接 | 稳定高带宽企业连接 |

---

## 四、高频复合场景

**场景一：全球用户访问 S3 中的图片和视频延迟高，源站请求量也很大。**

推理路径：内容是静态对象，可以被缓存。全球访问慢和源站压力大都符合 CloudFront。若源站是私有 S3，还应配置 OAC。

最终答案：使用 CloudFront 分发 S3 内容，并通过 OAC 限制直接访问 S3。

**场景二：全球在线游戏使用 UDP，用户需要固定入口 IP 和更低网络抖动。**

推理路径：UDP 动态流量不可用 CloudFront 缓存，固定 Anycast IP 和路径优化是 Global Accelerator 的优势。

最终答案：使用 AWS Global Accelerator，将端点指向区域内 NLB 或 EC2。

**场景三：私有子网中的 EC2 需要大量读取 S3 对象，目前流量经过 NAT Gateway。**

推理路径：私有访问 S3 不需要走 NAT 或公网。Gateway VPC Endpoint 可让流量在 AWS 私有网络内到达 S3，同时降低 NAT 成本和瓶颈。

最终答案：创建 S3 Gateway VPC Endpoint，并更新私有子网路由表。

**场景四：企业本地数据中心每天向 AWS 传输 TB 级数据，VPN 经常抖动，业务要求稳定带宽。**

推理路径：公网 VPN 不能保证稳定吞吐和低抖动。Direct Connect 是专用连接，适合高带宽稳定混合云。

最终答案：部署 AWS Direct Connect；如需加密，可叠加 VPN 或使用应用层 TLS。

**场景五：微服务平台需要按 URL 路径把请求路由到不同服务，所有服务都是 HTTP。**

推理路径：HTTP 路径路由属于七层能力，ALB 是标准答案。NLB 虽然性能高，但不理解 HTTP 路径。

最终答案：使用 Application Load Balancer，配置 Path-based Routing 到不同 Target Group。

---

## 五、本 Task 一句话总结

- **可缓存内容全球慢用 CloudFront，动态 TCP/UDP 全球慢用 Global Accelerator。**
- **Route 53 Latency-based 是 DNS 就近，Global Accelerator 是固定 Anycast 入口和网络路径优化。**
- **HTTP 选 ALB，TCP/UDP/静态 IP 选 NLB，网络设备插入选 GWLB。**
- **私有访问 S3/DynamoDB 用 Gateway Endpoint，其他服务多用 Interface Endpoint。**
- **混合云稳定高带宽选 Direct Connect，快速低成本连接选 VPN。**
