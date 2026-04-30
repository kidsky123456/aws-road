以下是您所需的中文版完整配置方案，涵盖 Terraform 代码、部署步骤、监控与排障建议。
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

与您现有的 Secret ARN 命名风格保持一致：`xxx/rds/sa/secret`

```hcl
resource "random_password" "db_master" {
  length  = 16
  special = false
}

resource "aws_secretsmanager_secret" "db_creds" {
  name = "xxx/rds/sa/secret"
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
  cluster_identifier = "tb-kk"
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
  name = "tb-xxx-rds-proxy-role"
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
  name                   = "tb-xxx-rds-proxy"
  role_arn               = aws_iam_role.rds_proxy.arn
  engine_family          = "POSTGRESQL"
  vpc_subnet_ids         = aws_subnet.database[*].id
  vpc_security_group_ids = [aws_security_group.rds_proxy.id]
  idle_client_timeout    = 1800                      # 30 分钟
  require_tls            = true
  debug_logging          = false

  auth {
    auth_scheme = "SECRETS"
