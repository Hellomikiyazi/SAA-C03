S3 Gateway Endpoint 是一种 VPC Endpoint，作用是让 VPC 里的资源，比如 EC2、Lambda、ECS，在不经过公网、不需要 NAT Gateway、不需要 Internet Gateway 的情况下，私有访问 Amazon S3。

可以把它理解成：

VPC 内资源 → S3 Gateway Endpoint → Amazon S3

而不是：

VPC 内资源 → NAT Gateway / Internet Gateway → 公网 → Amazon S3

⸻

1. 它解决什么问题？

假设你有一个 EC2 实例在 private subnet 里，需要读取 S3 bucket 里的日志、图片、配置文件。

如果没有 S3 Gateway Endpoint，private subnet 里的 EC2 想访问 S3，通常需要：

EC2 → Route Table → NAT Gateway → Internet Gateway → S3

这会带来几个问题：

需要 NAT Gateway
产生 NAT Gateway 费用
流量路径经过公网出口
架构更复杂

使用 S3 Gateway Endpoint 后，路径变成：

EC2 → Route Table → S3 Gateway Endpoint → S3

EC2 仍然在 private subnet，S3 访问不需要公网出口。

⸻

2. 为什么叫 Gateway Endpoint？

VPC Endpoint 主要有两类常考：

Gateway Endpoint → 用于 S3 和 DynamoDB
Interface Endpoint → 用于大多数其他 AWS 服务，例如 SSM、Secrets Manager、CloudWatch、KMS、ECR 等

S3 Gateway Endpoint 的特点是：它不是一个 ENI，不会在子网里放网卡，而是通过 Route Table 路由表 工作。

你创建 S3 Gateway Endpoint 后，需要把它关联到相关 subnet 的 route table。之后访问 S3 的流量会通过这个 endpoint 走。

⸻

3. 它在架构里长什么样？

典型架构：

Private Subnet
  └─ EC2 Instance
        ↓
Route Table
  └─ Destination: S3 prefix list
     Target: S3 Gateway Endpoint
        ↓
Amazon S3 Bucket

Route Table 中会出现类似这种逻辑：

Destination: pl-xxxxxxxx  # S3 prefix list
Target: vpce-xxxxxxxx     # S3 Gateway Endpoint

这里的 pl-xxxxxxxx 是 S3 的 managed prefix list，代表 S3 的地址范围。

⸻

4. 它和 NAT Gateway 有什么区别？

NAT Gateway → 让 private subnet 资源访问互联网或公网 AWS 服务
S3 Gateway Endpoint → 只让 VPC 私有访问 S3

对比：

项目	NAT Gateway	S3 Gateway Endpoint
访问目标	互联网、AWS 公网服务	Amazon S3
是否需要公网出口	需要	不需要
是否适合 private subnet 访问 S3	可以，但不是最优	是最佳实践
成本	有小时费和流量处理费	Gateway Endpoint 本身通常无额外费用
路由方式	走 NAT Gateway	走 VPC route table 到 endpoint
SAA 考点	可用但成本更高	private access to S3 首选

考试中看到：

EC2 in private subnet needs to access S3
without internet
least cost
private connectivity

优先选：

S3 Gateway Endpoint

⸻

5. 它和 Interface Endpoint 有什么区别？

Gateway Endpoint → S3、DynamoDB
Interface Endpoint → 大多数其他 AWS 服务

对比：

项目	Gateway Endpoint	Interface Endpoint
支持服务	S3、DynamoDB	大多数 AWS 服务
实现方式	路由表	ENI + 私有 IP
是否放在 subnet 里	不放 ENI	每个 AZ/subnet 创建 ENI
是否收费	通常 endpoint 本身无额外费用	通常有小时费和流量费
安全控制	Endpoint policy、bucket policy	Security Group、endpoint policy
SAA 高频场景	S3 / DynamoDB 私有访问	SSM、Secrets Manager、CloudWatch、ECR 等私有访问

⸻

6. 权限怎么控制？

S3 Gateway Endpoint 只是提供“私有网络路径”，不代表自动拥有 S3 权限。

访问 S3 仍然要同时满足权限条件：

EC2 IAM Role 权限
S3 Bucket Policy
S3 Gateway Endpoint Policy

可以这样理解：

Gateway Endpoint → 控制网络路径
IAM Role → 控制身份权限
Bucket Policy → 控制 bucket 资源权限
Endpoint Policy → 控制通过这个 endpoint 能访问哪些 S3 资源

例如，可以限制某个 bucket 只能通过指定 VPC Endpoint 访问：

S3 Bucket Policy:
只允许 aws:sourceVpce = vpce-xxxx 的请求访问 bucket

这在考试中经常对应：

Only allow access to S3 from this VPC
Prevent public internet access to S3
Private access to S3

⸻

7. 一句话总结

S3 Gateway Endpoint = 让 VPC 内资源通过私有路径访问 S3 的网关型 VPC Endpoint。

考试判断链路：

EC2 / Lambda / ECS 在 VPC 内
→ 需要访问 S3
→ 不想走公网 / 不想用 NAT Gateway / 要最低成本
→ 创建 S3 Gateway Endpoint
→ 关联 Route Table
→ 配置 IAM / Bucket Policy / Endpoint Policy

记忆口诀：

私网访问 S3，不走 NAT，不出公网，选 Gateway Endpoint。
S3 和 DynamoDB 用 Gateway，大多数其他 AWS 服务用 Interface。