## 一、为何采用 Aurora + RDS Proxy 架构

在现代云原生应用中，数据库连接管理常常成为性能瓶颈。如果你的应用包含以下特征，Aurora + RDS Proxy 的组合尤其适合：

- **Serverless/Lambda 架构**：函数频繁启动和关闭，每次调用都试图建立新的数据库连接
- **不可预测的流量突发**：连接数量短期内急剧增长，可能耗尽数据库连接池
- **长连接但大量空闲**：SaaS 或电商应用保持大量空闲连接，造成资源浪费
- **对高可用性有严格要求**：需要快速故障转移，对底层故障透明容忍

RDS Proxy 作为数据库连接代理，主要提供三大核心价值：

1. **连接池化**：RDS Proxy 维护一个已建立的数据库连接池。当应用请求到达时，从池中借用现有连接而不是每次都创建新的，从而减少数据库端建立新连接的内存和计算压力。

2. **故障转移加速与保护**：RDS Proxy 在数据库故障期间自动将流量路由到新实例并保留应用连接，可将故障转移时间缩短多达 **66%**。Proxy 还会绕过 DNS 缓存，进一步加速故障转移过程。

3. **安全加固与凭证管理**：RDS Proxy 可与 AWS IAM 认证集成，支持端到端的 IAM 认证机制（适用于 MySQL、MariaDB 和 PostgreSQL），并集中管理 Secrets Manager 中的数据库凭证。用户不再需要在 Lambda 代码中处理数据库凭证。

---

## 二、整体架构蓝图与网络设计原则

> 以下各节提供架构思路和关键配置要点，完整代码请根据实际环境填充变量后整合使用。

在部署 Aurora 和 RDS Proxy 之前，必须首先建立正确的网络基础。以下是核心架构原则：

### 2.1 子网与可用区策略

> 关键提示：**并非所有可用区都支持 DB Proxy**。如果配置了不支持 Proxy 的可用区子网，Terraform 不会报错，但会在每次 `plan` 时持续报告配置漂移，因为 AWS API 只会返回支持 Proxy 的可用区子网。

最推荐的解决方案是通过 `aws_availability_zones` data source 动态过滤掉不支持 Proxy 的可用区。以下以 us-east-1 为例，确保排除那些不支持的可用区（如 use1-az3）：

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

# 使用 filter 过滤不支持 RDS Proxy 的可用区名称
# 注意：AZ名称跨账号可能不同，建议使用 AZ ID（如 use1-az1）进行精确过滤
locals {
  unsupported_az_names = ["use1-az3"]  # 根据实际区域预先识别
  supported_azs = [for az in data.aws_availability_zones.available.names :
    az if !contains(local.unsupported_az_names, az)
  ]
}

# 创建数据库子网，只选择经过验证支持的可用区
resource "aws_subnet" "database" {
  count             = length(local.supported_azs)
  vpc_id            = data.aws_vpc.main.id
  cidr_block        = cidrsubnet(data.aws_vpc.main.cidr_block, 8, count.index + 20)
  availability_zone = local.supported_azs[count.index]
  tags              = { Name = "database-subnet-${count.index + 1}" }
}

# DB Subnet Group
resource "aws_db_subnet_group" "aurora" {
  name        = "aurora-subnet-group"
  subnet_ids  = aws_subnet.database[*].id
  description = "Subnet group for Aurora cluster"
}
```

### 2.2 安全组配置要点

RDS Proxy 与 Aurora 各自需要明确的安全组规则。以下表格总结了关键配置：

| 组件 | 方向 | 端口 | 协议 | 说明 |
|------|------|------|------|------|
| Aurora 安全组 | 入站 | 5432 (PostgreSQL) 或 3306 (MySQL) | TCP | 仅允许 RDS Proxy 安全组的访问 |
| RDS Proxy 安全组 | 出站 | 5432 (或 3306) | TCP | 允许访问 Aurora 数据库端口 |
| RDS Proxy 安全组 | 出站 | 443 | TCP | **关键**：允许访问 AWS Secrets Manager（HTTPS 端点） |
| 应用安全组 | 出站 | Proxy 端口 | TCP | 允许应用通过 Proxy 端点访问 |

>  **出站 443 端口是关键配置**：RDS Proxy 必须通过 HTTPS 连接到 AWS Secrets Manager 来获取数据库凭证。如果 Proxy 位于私有子网，路径上必须具备 NAT Gateway 或 VPC Endpoint，否则 Proxy 会卡在 PENDING_PROXY_CAPACITY 状态。

---

## 三、Aurora 数据库集群部署

### 3.1 Aurora Serverless v2 配置

Aurora Serverless v2 是目前推荐的生产级 Serverless 数据库方案。与 Serverless v1 不同，Serverless v2 使用 cluster mode `provisioned`，然后通过 `instance_class` 将单个实例配置为 Serverless 实例。容量不能降至 0 ACU（最低 0.5 ACU），这是与 v1 的一个关键差异。

以下展示 Aurora PostgreSQL 17 Serverless v2 集群的核心配置结构：

```hcl
# 数据库凭证由 Secrets Manager 统一管理（应在 Proxy 配置中引用）
resource "aws_secretsmanager_secret" "db_creds" {
  name = "aurora-db-credentials"
}

resource "aws_secretsmanager_secret_version" "db_creds" {
  secret_id     = aws_secretsmanager_secret.db_creds.id
  secret_string = jsonencode({
    username = var.db_username          # 从变量或 random_password 获取
    password = var.db_password
    port     = "5432"
  })
}

# Aurora Serverless v2 集群
resource "aws_rds_cluster" "aurora" {
  cluster_identifier = "aurora-serverless-cluster"
  engine             = "aurora-postgresql"
  engine_version     = "17.5"
  engine_mode        = "provisioned"       # Serverless v2 使用 provisioned 模式
  database_name      = var.db_name
  master_username    = var.db_username
  master_password    = var.db_password
  storage_encrypted  = true

  db_subnet_group_name   = aws_db_subnet_group.aurora.name
  vpc_security_group_ids = [aws_security_group.aurora.id]

  backup_retention_period = 7
  preferred_backup_window = "03:00-04:00"

  enabled_cloudwatch_logs_exports = ["postgresql"]
}

# Serverless v2 实例（Writer）
resource "aws_rds_cluster_instance" "writer" {
  identifier         = "aurora-writer"
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.serverless"    # Serverless v2 的关键配置
  engine             = aws_rds_cluster.aurora.engine
  engine_version     = aws_rds_cluster.aurora.engine_version
  publicly_accessible = false

  # 可选：添加读副本实例（同样配置 db.serverless）
  # count = var.read_replicas
}

# RDS Proxy 的 IAM 角色（用于访问 Secrets Manager）
resource "aws_iam_role" "rds_proxy" {
  name = "rds-proxy-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "rds.amazonaws.com" }
    }]
  })
}

resource "aws_iam_policy" "secrets_manager" {
  name = "rds-proxy-secrets-policy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["secretsmanager:GetSecretValue"]
      Resource = [aws_secretsmanager_secret.db_creds.arn]
    }]
  })
}

resource "aws_iam_role_policy_attachment" "attach" {
  role       = aws_iam_role.rds_proxy.name
  policy_arn = aws_iam_policy.secrets_manager.arn
}
```

### 3.2 Aurora 参数组优化

针对高并发场景，建议对数据库参数进行调优：

```
# 以 PostgreSQL 为例的关键参数
shared_buffers = 25% of DB memory
max_connections = 1000-5000（根据实例大小和 Proxy 连接池设计）
work_mem = 4MB-16MB
……
```

在 Terraform 中通过 `aws_rds_cluster_parameter_group` 和 `aws_db_parameter_group` 配置。

---

## 四、RDS Proxy 核心配置

### 4.1 创建 RDS Proxy

以下配置涵盖了生产级 RDS Proxy 的核心参数：

```hcl
# 获取 Aruora 集群信息（用于 Proxy 的目标关联）
data "aws_rds_cluster" "aurora" {
  cluster_identifier = aws_rds_cluster.aurora.cluster_identifier
}

# 使用社区模块创建 RDS Proxy（需要提前获取 module 源码）
# 参考模块：terraform-aws-modules/rds-proxy/aws
module "rds_proxy" {
  source = "terraform-aws-modules/rds-proxy/aws"

  name                   = "aurora-proxy"
  iam_role_name         = aws_iam_role.rds_proxy.name        # 使用已创建的 IAM 角色
  vpc_subnet_ids        = aws_subnet.database[*].id
  vpc_security_group_ids = [aws_security_group.rds_proxy.id]

  engine_family          = "POSTGRESQL"
  target_db_cluster      = true
  db_cluster_identifier  = data.aws_rds_cluster.aurora.cluster_identifier

  # 连接池优化配置
  connection_borrow_timeout = 120      # 等待连接可用的秒数（默认120）
  max_connections_percent    = 100     # 连接池最大大小占目标组最大连接的百分比
  max_idle_connections_percent = 50    # 空闲连接池百分比（仅当连接数超过此值时关闭）
  session_pinning_filters   = ["EXCLUDE_VARIABLE_SETS"]  # 减少 session pinning

  # 代理行为
  idle_client_timeout = 1800           # 客户端空闲超时（30分钟）
  require_tls         = true
  debug_logging       = false
```

**关键参数说明**：

| 参数 | 取值范围 | 说明 |
|------|---------|------|
| `max_connections_percent` | 1-100 | 控制 Proxy 可占用的数据库最大连接数的百分比。设置为 100 允许完全使用，但建议预留 10%-20% 给直接数据库管理操作 |
| `max_idle_connections_percent` | 1-100 | 控制 Proxy 积极关闭空闲连接的阈值；建议设置 40-60 之间，以平衡复用和资源释放 |
| `session_pinning_filters` | ["EXCLUDE_VARIABLE_SETS"] | 减少因 SET 等操作导致的 session pinning，提高连接复用率 |

### 4.2 认证方式选择

RDS Proxy 支持两种主要认证方式：

**方式一：Secrets Manager（推荐所有密码验证场景）**

凭证以 JSON 格式存储在 Secrets Manager 中，Proxy 通过 IAM 角色获取：

```hcl
resource "aws_secretsmanager_secret_version" "proxy_creds" {
  secret_id = aws_secretsmanager_secret.db_creds.id
  secret_string = jsonencode({
    username = var.db_username
    password = var.db_password
    engine   = "postgres"
    host     = aws_rds_cluster.aurora.endpoint
    port     = 5432
  })
}

module "rds_proxy" {
  # ... 其他配置
  auth = {
    "database_user" = {
      description = "Database credentials for proxy"
      secret_arn = aws_secretsmanager_secret_version.proxy_creds.arn
    }
  }
}
```

**方式二：IAM 认证（推荐 Serverless/Lambda 场景）**

对于 RDS Proxy 与 MySQL、MariaDB、PostgreSQL，目前支持端到端 IAM 认证。设置 `iam_auth = "ENABLED"` 后，Proxy 将使用 IAM 认证向数据库验证身份。

```hcl
auth {
  auth_scheme = "IAM_AUTH"
  iam_auth    = "ENABLED"
}
```

**适合场景比较**：

| 场景 | 推荐认证方式 | 原因 |
|------|-------------|------|
| Lambda + Serverless 应用 | IAM 认证 | 无需在代码中存储凭证，使用执行角色 |
| 传统应用/服务 | Secrets Manager | 与现有密码体系兼容，迁移成本低 |
| 混合场景 | 两者同时配置 | 不同应用可选择不同认证方式 |

---

## 五、目标组与连接池优化

RDS Proxy 通过目标组（Target Group）管理连接行为。关键的连接池配置参数直接影响多租户和高并发环境下的性能：

### 5.1 连接池核心配置

下表中的参数可针对 `aws_db_proxy_default_target_group` 进行配置：

| 参数 | 值 | 说明 |
|------|---|------|
| `connection_borrow_timeout` | 120-300 秒 | 代理等待连接从池中可用的最大时间。超过后连接失败。高并发场景建议调整此值匹配应用超时设计 |
| `max_connections_percent` | 70-90% | 代理允许占用的数据库最大连接百分比。**生产建议预留 10-30% 给管理用途**。示例：若数据库 max_connections=1000，设置 70% 则代理用 700 个连接 |
| `max_idle_connections_percent` | 50-70% | 代理积极关闭空闲连接的阈值；数值越高保持更多空闲连接 |
| `init_query` | `SET app_name='myapp'` | 代理每次打开连接时运行的 SQL 语句；可用于设置会话上下文 |
| `session_pinning_filters` | `["EXCLUDE_VARIABLE_SESSIONS"]` | 控制连接何时必须保持固定（pinned），减少不必要的固定可大幅提高复用率 |

### 5.2 连接层级与双重池化

必须注意：RDS Proxy 创建了两层连接：
- **App → Proxy**：应用端保持的连接（应用层面可能也有连接池）
- **Proxy → DB**：Proxy 与数据库之间的连接池

**双重池化反模式**：如果应用层连接池和 RDS Proxy 都设置过大，会导致资源浪费。建议**降低应用端的连接池大小**，让它与 Proxy 的最大连接数相匹配，而不是无限扩大。

### 5.3 高并发多租户场景建议

在多租户架构中，不同租户可能共享 Proxy 但需要不同的连接需求。针对每种租户类型可以创建独立的 Proxy 端点（endpoint）：

```hcl
resource "aws_db_proxy_endpoint" "read_only" {
  db_proxy_name          = module.rds_proxy.name
  db_proxy_endpoint_name = "read-only-endpoint"
  vpc_subnet_ids         = aws_subnet.database[*].id
  target_role            = "READ_ONLY"                # 使用只读副本
}
```

---

## 六、常见故障排查与最佳实践

### 6.1 核心故障解决指南

**问题：PENDING_PROXY_CAPACITY（部署失败）**

这是 RDS Proxy 部署中最常见的问题，通常由于 Proxy 无法访问 Secrets Manager 导致的。

**解决方案**：
1. 安全组出站必须允许 **TCP 443** 出站流量；
2. 私有子网的 Proxy 需要 **NAT Gateway** 或 **VPC Endpoint for Secrets Manager**；
3. Secrets Manager 中的凭证必须以**完整 JSON 格式**存储（包含 username、password、engine、port 等）。

**问题：持续配置漂移（Terraform plan 总是提示变化）**

如 2.1 节所述，根本原因是指定了不支持 Proxy 的可用区子网。解决方案已在第 2.1 节给出。

**问题：连接失败（Connection timeout/refused）**

- 安全组规则是否允许应用 → Proxy → Aurora 的完整连接路径；
- Proxy 的连接借用超时（connection_borrow_timeout）设置是否过短（默认 120 秒）；
- 数据库的 max_connections 参数是否过小，导致 Proxy 无法建立新连接。

### 6.2 生产级最佳实践清单

**准备阶段**：
1. 确定哪些可用区支持 RDS Proxy（在目标区域执行测试或查阅官方文档）
2. 规划两层连接池的大小适配
3. 决定认证方式（IAM 认证更安全，Secrets Manager 兼容性更广）

**配置阶段**：
4. 启动 Proxy 前确保 Secrets Manager 中的凭证 JSON 格式正确
5. 安全组配置：确保 Proxy 安全组**同时**开放 5432（数据库）和 443（Secrets Manager）
6. 设置 max_connections_percent < 100%，留有余量（建议 80-90%）
7. 启用 require_tls = true，强制加密传输

**运维阶段**：
8. 在 CloudWatch 中监控 `DatabaseConnections`、`ClientConnections`、`MaxConnections` 等指标
9. 定期轮换凭据（通过 Secrets Manager 自动轮换）
10. 考虑设置多个 Proxy 端点为不同应用类型提供差异化连接池配置

---

> **完整代码参考**：完整的 Terraform 模块化实现可参考 [terraform-aws-modules/rds-proxy/aws](https://github.com/terraform-aws-modules/terraform-aws-rds-proxy)，其中包含了所有上述配置的综合示例。
