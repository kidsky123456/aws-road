以下是您所需的中文版完整配置方案，完全基于您提供的英文内容翻译并重新组织，涵盖 Terraform 代码、部署步骤、监控与排障建议。

---

## 📘 参照 ROAD 方案在 AWS 上部署 Aurora + RDS Proxy

您的现有基础设施 `tb-kk-rds-proxy`（PostgreSQL）已具备读写和只读两个代理端点，并与 Secrets Manager 集成。下面提供一套与 ROAD 自动化 CI/CD 流程完全兼容的 Terraform 配置，使您能够用代码复现这套成熟架构。

---

## 一、Terraform 配置步骤

### 1️⃣ `providers.tf` – 定义 AWS Provider

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-west-1"   # 与您的截图区域一致
}
```

### 2️⃣ `vpc-and-database-subnets.tf` – 网络与数据库子网

**核心要点**：必须排除不支持 RDS Proxy 的可用区（如 `eu-west-1a`），否则 Terraform 会持续产生配置漂移。

```hcl
data "aws_vpc" "main" {
  tags = { Name = "<your-vpc-name>" }   # 请替换为实际 VPC 名称
}

data "aws_availability_zones" "available" {
  state = "available"
}

# 重要：过滤掉不支持 RDS Proxy 的 AZ（根据您账号实际情况调整）
locals {
  unsupported_azs   = ["eu-west-1a"]   # 示例，请确认后填写
  supported_subnets = [
    for az in data.aws_availability_zones.available.names :
    az if !contains(local.unsupported_azs, az)
  ]
}

# 仅在支持的可用区中创建子网
resource "aws_subnet" "database" {
  count             = length(local.supported_subnets)
  vpc_id            = data.aws_vpc.main.id
  cidr_block        = cidrsubnet(data.aws_vpc.main.cidr_block, 8, count.index + 20)
  availability_zone = local.supported_subnets[count.index]
  tags = {
    Name = "database-subnet-${count.index + 1}"
  }
}

# Aurora 和 RDS Proxy 都需要使用的子网组
resource "aws_db_subnet_group" "aurora" {
  name        = "aurora-subnet-group"
  subnet_ids  = aws_subnet.database[*].id
  tags = {
    Name = "aurora-subnet-group"
  }
}
```

### 3️⃣ `database-credentials.tf` – 数据库凭证（Secrets Manager）

与您现有的 Secret ARN 命名风格保持一致：`fxps/fdr/rds/sa/secret`

```hcl
resource "random_password" "db_master" {
  length  = 16
  special = false
}

resource "aws_secretsmanager_secret" "db_creds" {
  name = "fxps/fdr/rds/sa/secret"
}

resource "aws_secretsmanager_secret_version" "db_creds" {
  secret_id = aws_secretsmanager_secret.db_creds.id
  secret_string = jsonencode({
    username              = "db_admin"
    password              = random_password.db_master.result
    engine                = "postgres"
    host                  = aws_rds_cluster.aurora.endpoint
    port                  = 5432
    dbClusterIdentifier   = aws_rds_cluster.aurora.cluster_identifier
  })
}
```

### 4️⃣ `aurora-cluster.tf` – Aurora Serverless v2 集群

**注意**：Serverless v2 使用 `engine_mode = "provisioned"`，实例类为 `db.serverless`。

```hcl
resource "aws_rds_cluster" "aurora" {
  cluster_identifier = "tb-fdr-eu"
  engine             = "aurora-postgresql"
  engine_mode        = "provisioned"          # Serverless v2 必须
  engine_version     = "16.4"
  database_name      = "fxdb"
  master_username    = jsondecode(aws_secretsmanager_secret_version.db_creds.secret_string)["username"]
  master_password    = jsondecode(aws_secretsmanager_secret_version.db_creds.secret_string)["password"]
  db_subnet_group_name   = aws_db_subnet_group.aurora.name
  vpc_security_group_ids = [aws_security_group.aurora.id]
  storage_encrypted  = true
  backup_retention_period = 30

  serverlessv2_scaling_configuration {
    max_capacity = 8.0    # 根据负载调整
    min_capacity = 0.5    # Serverless v2 最低 0.5 ACU
  }
}

# 写实例（可增加读副本）
resource "aws_rds_cluster_instance" "writer" {
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.aurora.engine
  engine_version     = aws_rds_cluster.aurora.engine_version
  publicly_accessible = false
}

# 如需添加读副本（与您的截图对应），取消下面的注释
/*
resource "aws_rds_cluster_instance" "reader" {
  count              = 1
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.aurora.engine
  engine_version     = aws_rds_cluster.aurora.engine_version
  publicly_accessible = false
}
*/
```

### 5️⃣ `rds-proxy.tf` – RDS Proxy（核心配置）

包含 IAM 角色、代理本身、默认目标组以及只读端点，完全匹配您的截图。

```hcl
# IAM 角色 – 允许 RDS Proxy 访问 Secrets Manager
resource "aws_iam_role" "rds_proxy" {
  name = "tb-fdr-eu-rds-proxy-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "rds.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_policy" "secrets_manager" {
  name = "rds-proxy-secrets-policy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["secretsmanager:GetSecretValue"]
        Resource = [aws_secretsmanager_secret.db_creds.arn]
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "attach" {
  role       = aws_iam_role.rds_proxy.name
  policy_arn = aws_iam_policy.secrets_manager.arn
}

# RDS Proxy 主体 – 名称与您的截图一致
resource "aws_db_proxy" "main" {
  name                   = "tb-fdr-eu-rds-proxy"
  role_arn               = aws_iam_role.rds_proxy.arn
  engine_family          = "POSTGRESQL"
  vpc_subnet_ids         = aws_subnet.database[*].id
  vpc_security_group_ids = [aws_security_group.rds_proxy.id]
  idle_client_timeout    = 1800                      # 30 分钟
  require_tls            = true
  debug_logging          = false

  auth {
    auth_scheme = "SECRETS"
    secret_arn  = aws_secretsmanager_secret.db_creds.arn
    iam_auth    = "DISABLED"
  }
}

# 默认目标组 – 连接池配置
resource "aws_db_proxy_default_target_group" "main" {
  db_proxy_name = aws_db_proxy.main.name
  connection_pool_config {
    max_connections_percent      = 100
    max_idle_connections_percent = 50
    connection_borrow_timeout    = 120
    session_pinning_filters      = ["EXCLUDE_VARIABLE_SETS"]
  }
}

# 将代理与 Aurora 集群关联
resource "aws_db_proxy_target" "aurora_cluster" {
  db_proxy_name          = aws_db_proxy.main.name
  target_group_name      = aws_db_proxy_default_target_group.main.name
  db_cluster_identifier  = aws_rds_cluster.aurora.id
}

# 只读端点 – 对应截图中的第二个端点
resource "aws_db_proxy_endpoint" "read_only" {
  db_proxy_name          = aws_db_proxy.main.name
  db_proxy_endpoint_name = "read-only-endpoint"
  vpc_subnet_ids         = aws_subnet.database[*].id
  target_role            = "READ_ONLY"
}
```

### 6️⃣ `security-groups.tf` – 安全组（网络隔离）

**关键点**：RDS Proxy 安全组出方向必须开放 **443 端口**，以便访问 Secrets Manager。

```hcl
# Aurora 安全组：只允许来自 RDS Proxy 的访问
resource "aws_security_group" "aurora" {
  name   = "aurora-cluster-sg"
  vpc_id = data.aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.rds_proxy.id]
  }
}

# RDS Proxy 安全组
resource "aws_security_group" "rds_proxy" {
  name   = "rds-proxy-sg"
  vpc_id = data.aws_vpc.main.id

  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = [data.aws_vpc.main.cidr_block]   # 允许 VPC 内应用访问
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]   # 必须：允许代理调用 Secrets Manager API
  }
}
```

---

## 二、应用连接示例（Node.js）

演示如何通过 Secrets Manager 获取凭证，并连接到读写端点和只读端点。

```javascript
const { Client } = require('pg');
const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');

const getSecret = async () => {
  const client = new SecretsManagerClient({ region: 'eu-west-1' });
  const command = new GetSecretValueCommand({ SecretId: 'fxps/fdr/rds/sa/secret' });
  const response = await client.send(command);
  return JSON.parse(response.SecretString);
};

// 获取写/读端点（代理的主端点）
const getWriteClient = async () => {
  const { username, password, host } = await getSecret();
  // host 为: tb-fdr-eu-rds-proxy.proxy-xxxxxx.eu-west-1.rds.amazonaws.com
  return new Client({
    host, port: 5432, user: username, password, database: 'fxdb', ssl: true
  });
};

// 只读端点（需自行拼接）
const getReadOnlyClient = async () => {
  const { username, password } = await getSecret();
  const readOnlyHost = "tb-fdr-eu-rds-proxy.read-only.endpoint.proxy-xxxxxx.eu-west-1.rds.amazonaws.com";
  return new Client({
    host: readOnlyHost, port: 5432, user: username, password, database: 'fxdb', ssl: true
  });
};
```

---

## 三、通过 ROAD 自动部署

1. **确认权限**：您已获得 `ADFS-RoadAutoDeploy` 角色以及对应环境（Dev/PreProd/Prod）的 Entitlement。申请链接见您提供的 ServiceNow URL【IMG_3139†L8-L20】。

2. **提交代码**：将上述 Terraform 文件推送到 ROAD 监控的仓库：
   ```
   https://alm-github.systems.uk.hsbc/TFX/tfps-foundation-teraform
   ```

3. **CI/CD 流水线**：CodePipeline `tfp-foundation-teraform-cd-pipeline` 会自动触发，执行 `terraform plan` 并等待批准（生产环境需手动审核）【IMG_3139†L34】。

4. **审批与应用**：在 CodePipeline 控制台中批准 “Deploy” 阶段。

---

## 四、推荐的监控与告警

| 监控指标 | 服务 | 推荐阈值 | 说明 |
|---------|------|----------|------|
| `DatabaseConnections` | RDS Proxy | < 数据库 `max_connections` 的 80% | 连接池效率低时会出现尖刺 |
| `ClientConnections` | RDS Proxy | 与运行应用实例数匹配 | 高值 + 低 DatabaseConnections 表示代理工作正常 |
| `DatabaseConnectionRequestsPerSecond` | RDS Proxy | 留意突发尖峰 | 可能是应用配置错误导致的“连接风暴” |
| `CPUUtilization` | Aurora 集群 | < 75% | 高 CPU 通常来自低效查询，而非连接问题 |
| `ServerlessDatabaseCapacity` | Aurora 集群 | 避免频繁扩缩 | 剧烈波动说明 `min_capacity` 可能设置过低 |

**建议告警规则**：
- `DatabaseConnections` > 80% 持续 15 分钟 → 触发警告
- `CPUUtilization` > 80% 持续 10 分钟 → 触发警告

---

## 五、常见 Terraform 错误及解决

### ❌ 错误：`aws_db_proxy_target` 创建卡在 `PENDING_PROXY_CAPACITY`

**原因**：RDS Proxy 无法出站访问 AWS Secrets Manager（需要 HTTPS 443 端口）。

**解决**：
- 检查 RDS Proxy 安全组出方向是否允许 **443 端口**；
- 如果代理位于私有子网且无 NAT 网关，必须创建 **VPC Endpoint for Secrets Manager**。

### ❌ 错误：`terraform plan` 持续报告代理配置漂移

**原因**：数据库子网中包含不支持 RDS Proxy 的可用区（如 `eu-west-1a`）。

**解决**：
- 在 `vpc-and-database-subnets.tf` 的 `local.unsupported_azs` 中明确列出不支持的可用区，确保 Terraform 不会使用它们创建子网。

---

以上配置已完整复现您提供的 AWS 控制台截图中的 Aurora + RDS Proxy 环境，并完全兼容 ROAD 的 GitOps 流程。您只需将代码中的 VPC 名称、可用区列表等个性化信息替换为实际值，即可直接使用。
