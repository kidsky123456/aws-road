作为初学者，掌握 Terraform 的基础语法是构建 AWS 基础设施的第一步。

Terraform 使用一种名为 HCL (HashiCorp Configuration Language) 的声明式语言。它的核心逻辑是：你告诉 Terraform 你想要什么状态（比如“我要一个 RDS Proxy”），而不是告诉它具体怎么做。

以下是一份为你整理的 Terraform 基础语法草案，涵盖了从文件结构到核心代码块的所有基础要素。

项目文件结构 (标准化布局)

虽然 Terraform 可以读取目录下所有的 .tf 文件，但为了保持整洁和可维护性，业界有一套约定俗成的文件组织方式：

my-aws-infra/
├── main.tf             # 核心资源定义 (Provider, Resources)
├── variables.tf        # 变量定义 (输入参数)
├── outputs.tf          # 输出定义 (如 Proxy 的端点地址)
├── terraform.tfvars    # 变量的具体数值 (敏感数据通常放这里)
└── providers.tf        # 指定 Provider 版本 (可选，但推荐)

核心语法块 (Building Blocks)

Terraform 的代码主要由“块 (Blocks)”组成。以下是你必须掌握的四种核心块：

① Provider (提供者)
这是 Terraform 与云厂商（如 AWS）交互的插件。
告诉 Terraform 我们要用 AWS，并指定区域
provider "aws" {
  region = "cn-northwest-1" # 例如：宁夏区域
}

② Resource (资源)
这是最核心的部分，用来定义具体的设施（如 VPC, RDS, Proxy）。
语法格式： resource "资源类型" "本地唯一名称" { ... }

创建一个安全组
resource "aws_security_group" "proxy_sg" {
  name        = "rds-proxy-sg"
  description = "Allow traffic for RDS Proxy"
  vpc_id      = aws_vpc.main.id # 引用其他资源

  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

③ Variable (变量)
为了让代码通用，不要把硬编码写死，而是使用变量。
variable "db_instance_class" {
  description = "The instance class for the RDS instance"
  type        = string
  default     = "db.t3.medium"
}

④ Output (输出)
执行 terraform apply 后，你想在屏幕上看到的信息（比如 Proxy 的连接地址）。
output "proxy_endpoint" {
  value = aws_db_proxy.my_proxy.endpoint
}

HCL 基础语法细节

在编写上述块时，你需要用到以下数据类型和语法特性：

数据类型
类型   示例   说明
String   "hello"   字符串，必须用双引号包裹

Number   10, 3.14   数字，不需要引号

Bool   true, false   布尔值

List   ["a", "b"]   列表，用 [] 包裹

| Map | { key = "val" } | 键值对，用 {} 包裹 |

常用操作符
引用资源属性：使用点号 .
    例如：aws_vpc.main.id (引用名为 main 的 VPC 的 ID)。
引用变量：使用 var. 前缀
    例如：var.region。
条件判断 (三元运算符)：条件 ? 真值 : 假值
    例如：instance_type = var.is_prod ? "t3.large" : "t3.micro"
注释：
    # 单行注释
    /* 多行注释 */

实战草案：搭建 RDS Proxy 的基础代码骨架

结合你之前的需求，这是一个最基础的 Terraform 脚本草案，展示了如何定义 RDS Proxy 及其依赖。你可以直接参考这个结构来编写你的 main.tf。

定义 Provider
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "cn-northwest-1"
}

定义变量 (variables.tf 的内容)
variable "vpc_id" {
  type = string
}

variable "subnet_ids" {
  type = list(string)
}

定义资源 (main.tf 的内容)

假设你已经有了 RDS 实例，这里仅演示 Proxy 部分
resource "aws_db_proxy" "my_aurora_proxy" {
  name                   = "my-aurora-proxy"
  debug_logging          = false
  engine_family          = "POSTGRESQL" # 或者 MYSQL
  idle_client_timeout    = 1800
  require_tls            = true
  role_arn               = aws_iam_role.proxy_role.arn # 引用 IAM 角色

  # 连接数据库的凭证 (来自 Secrets Manager)
  auth {
    auth_scheme = "SECRETS"
    iam_auth    = "DISABLED"
    secret_arn  = "arn:aws:secretsmanager:..." 
  }

  # 连接池配置
  pools_config {
    max_connections_percent = 100
  }

  # 网络配置
  vpc_security_group_ids = [aws_security_group.proxy_sg.id]
  vpc_subnet_ids         = var.subnet_ids
}

定义输出 (outputs.tf 的内容)
output "proxy_endpoint_address" {
  value = aws_db_proxy.my_aurora_proxy.endpoint
}

常用工作流命令

写好代码后，你需要通过命令行来执行：

terraform init: 初始化。下载 AWS 插件，这是所有工作的开始。
terraform plan: 预演。它会告诉你“我打算创建什么”，但还不会真的动手。
terraform apply: 执行。真的去 AWS 上创建资源。
terraform destroy: 销毁。删除所有由该代码创建的资源（清理环境时用）。

这套草案涵盖了 Terraform 80% 的日常用法。你可以先按照这个结构把文件建起来，填入具体的 AWS 资源 ID，然后尝试运行 terraform plan 看看效果！
