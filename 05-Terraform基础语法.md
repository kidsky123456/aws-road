## Terraform 基础语法草案（初学者版）

这份草案涵盖了 Terraform 最核心的语法元素，每个部分都配有示例代码和简短说明。你可以把它当作速查手册，在实际编写配置时对照使用。

---

### 1. HCL 基础结构

Terraform 使用 **HCL（HashiCorp Configuration Language）**，文件扩展名为 `.tf`。基本语法规则：

- **块（Block）**：用 `{ }` 包裹，如 `resource "aws_instance" "example" { ... }`
- **参数（Argument）**：`参数名 = 值`，如 `ami = "ami-123456"`
- **注释**：`#` 单行注释，`/* */` 多行注释
- **字符串**：双引号 `"hello"`
- **数字**：直接写，如 `10`
- **布尔值**：`true` / `false`
- **列表（list）**：`["a", "b", "c"]`，索引从 0 开始
- **映射（map）**：`{ name = "John", age = 30 }`
- **引用**：`资源类型.资源名称.属性`，如 `aws_instance.web.id`

---

### 2. Provider（提供商）

定义使用哪个云平台或服务，以及其配置（如区域、认证信息）。

```hcl
# 基本用法
provider "aws" {
  region = "us-west-1"
}

# 多个 Provider 实例（例如不同区域）
provider "aws" {
  alias  = "virginia"
  region = "us-east-1"
}

# 使用带别名的 Provider
resource "aws_instance" "example" {
  provider = aws.virginia
  # ...
}
```

---

### 3. Resource（资源）

**核心块**，用来创建云资源（如 EC2 实例、S3 桶、RDS 数据库）。

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  tags = {
    Name = "MyWebServer"
  }
}

# 依赖其他资源（隐式依赖：引用属性即可）
resource "aws_eip" "ip" {
  instance = aws_instance.web.id   # 自动依赖 aws_instance.web
}

# 显式声明依赖（不常用）
resource "null_resource" "example" {
  depends_on = [aws_instance.web]
}
```

#### 元参数（Meta-Arguments）
- **`count`**：创建多个相似的资源（数量）
- **`for_each`**：根据 map 或 set 创建多个资源（更灵活）
- **`depends_on`**：显式指定依赖关系
- **`provider`**：指定使用哪个 Provider 实例（如 alias）
- **`lifecycle`**：控制资源行为（创建前删除、忽略变更等）

```hcl
# 使用 count
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-123"
  instance_type = "t2.micro"
  tags = {
    Name = "web-${count.index}"   # 引用索引
  }
}

# 使用 for_each
locals {
  instances = {
    web1 = "t2.micro"
    web2 = "t2.small"
  }
}
resource "aws_instance" "web" {
  for_each      = local.instances
  ami           = "ami-123"
  instance_type = each.value
  tags = {
    Name = each.key
  }
}
```

---

### 4. Variable（变量）

让配置可复用，将变化的值抽取出来。

```hcl
# 声明变量（通常在 variables.tf）
variable "instance_type" {
  description = "EC2 实例类型"
  type        = string
  default     = "t2.micro"
}

variable "env" {
  type    = string
  # 无默认值，使用时必须提供
}

variable "tags" {
  type    = map(string)
  default = { Environment = "dev" }
}

# 使用变量
resource "aws_instance" "web" {
  instance_type = var.instance_type
  tags          = var.tags
}
```

#### 变量赋值优先级（从低到高）
1. 默认值（`default`）
2. 环境变量 `TF_VAR_变量名`（如 `export TF_VAR_instance_type="t3.micro"`）
3. `terraform.tfvars` 或 `terraform.tfvars.json` 文件
4. `*.auto.tfvars` 文件
5. 命令行 `-var` 或 `-var-file`

```hcl
# terraform.tfvars 内容示例
instance_type = "t3.small"
env           = "production"
tags = {
  Environment = "prod"
  Owner       = "team-a"
}
```

---

### 5. Output（输出）

定义部署完成后想要显示的信息（如 IP 地址、资源 ID）。

```hcl
output "instance_id" {
  description = "EC2 实例 ID"
  value       = aws_instance.web.id
}

output "public_ip" {
  value = aws_instance.web.public_ip
  # 敏感信息可标记（不会在日志中明文显示）
  sensitive = true
}

# 输出列表/映射的全部或部分
output "all_ips" {
  value = aws_instance.web[*].public_ip   # 展开列表
}
```

运行 `terraform apply` 后会显示输出值；也可用 `terraform output` 命令单独查看。

---

### 6. Local Values（局部值）

在当前模块内部使用的临时计算值，避免重复计算。

```hcl
locals {
  common_tags = {
    Project   = "MyProject"
    ManagedBy = "Terraform"
  }
  instance_name = "${var.env}-web-server"
}

resource "aws_instance" "web" {
  tags = local.common_tags   # 引用局部值
  # ...
}
```

---

### 7. Data Source（数据源）

查询已存在的资源信息（不是创建），供当前配置引用。

```hcl
# 查找 AWS 可用区
data "aws_availability_zones" "available" {
  state = "available"
}

# 查找特定的 AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]   # Canonical
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

# 引用数据源属性
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  availability_zone = data.aws_availability_zones.available.names[0]
}
```

---

### 8. Module（模块）

封装一组资源，实现复用。

```hcl
# 调用本地模块（相对路径）
module "vpc" {
  source = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
}

# 调用远程模块（Terraform Registry）
module "rds" {
  source = "terraform-aws-modules/rds-aurora/aws"
  version = "9.0.0"
  
  name           = "my-db"
  engine         = "aurora-postgresql"
  # ...
}

# 引用模块的输出
resource "aws_instance" "web" {
  subnet_id = module.vpc.subnet_ids[0]
}
```

**模块内部结构**（目录 `modules/vpc`）：
- `main.tf` - 资源定义
- `variables.tf` - 输入变量
- `outputs.tf` - 输出值
- `versions.tf` - Provider 版本等

---

### 9. Backend（后端状态存储）

配置 `terraform.tfstate` 文件的存储位置。默认本地存储，生产环境推荐使用远程后端（如 S3、Azure Storage、Terraform Cloud）。

```hcl
# 本地后端（默认）
# 不用写任何配置，状态文件为 terraform.tfstate

# 远程后端（例：AWS S3 + DynamoDB 锁）
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/network/terraform.tfstate"
    region         = "us-west-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

**重要**：修改 backend 配置后必须运行 `terraform init -reconfigure` 或 `-migrate-state`。

---

### 10. Workspace（工作区）

管理多个环境（dev/staging/prod）的隔离状态，避免混用同一套配置。

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace select dev
```

在代码中使用当前工作区名：
```hcl
resource "aws_instance" "web" {
  tags = {
    Environment = terraform.workspace   # "dev" 或 "prod"
  }
}
```

> 工作区适合简单场景；复杂项目建议用目录分离（`/dev`, `/prod`）配合不同后端配置。

---

### 11. 常用函数

Terraform 内置了许多转换、字符串、集合等函数。

| 类别 | 函数示例 | 说明 |
|------|---------|------|
| 字符串 | `upper("hello")` → `"HELLO"` | 转大写 |
| | `format("Hello %s", "World")` → `"Hello World"` | 格式化 |
| | `join(", ", ["a","b"])` → `"a, b"` | 拼接 |
| 集合 | `length(["a","b","c"])` → `3` | 长度 |
| | `element(["a","b"], 1)` → `"b"` | 按索引取元素 |
| | `lookup({name="Tom"}, "name", "default")` → `"Tom"` | 从 map 取值 |
| 数字 | `max(5, 12, 9)` → `12` | 最大值 |
| | `ceil(5.1)` → `6` | 向上取整 |
| 类型转换 | `tostring(123)` → `"123"` | 转字符串 |
| | `tolist({a=1,b=2})` → `[1,2]` | 转列表 |
| 文件 | `file("user_data.sh")` | 读取文件内容 |
| | `templatefile("script.tpl", { name = "John" })` | 渲染模板 |

```hcl
# 示例：拼接安全组规则描述
resource "aws_security_group_rule" "example" {
  description = "${upper(var.env)}-allow-http"
  # ...
}

# 使用 templatefile
user_data = templatefile("init.tpl", { server_role = "web" })
```

---

### 12. 条件表达式

类似三元运算符：`条件 ? 真值 : 假值`

```hcl
instance_type = var.env == "prod" ? "t3.large" : "t2.micro"

# 与 try 函数结合处理可能为空的属性
output "public_ip" {
  value = try(aws_instance.web[0].public_ip, "无公网IP")
}
```

---

### 13. 版本约束

锁定 Provider 和 Terraform 核心版本，避免意外升级导致兼容性问题。

```hcl
terraform {
  required_version = ">= 1.0, < 2.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"      # 等价于 >=5.0, <6.0
    }
    random = {
      source  = "hashicorp/random"
      version = ">= 3.0"
    }
  }
}
```

---

### 14. 常用命令速查

| 命令 | 作用 |
|------|------|
| `terraform init` | 初始化工作目录，下载 Provider 和模块 |
| `terraform plan` | 预览执行计划 |
| `terraform apply` | 应用变更（创建/更新/删除） |
| `terraform destroy` | 销毁所有资源 |
| `terraform fmt` | 格式化代码 |
| `terraform validate` | 验证语法 |
| `terraform output` | 查看输出值 |
| `terraform state list` | 列出状态中的资源 |
| `terraform state show <资源>` | 查看特定资源的状态 |
| `terraform import <地址> <id>` | 将现有资源导入状态 |
| `terraform workspace list` | 列出所有工作区 |

---

### 15. 一个完整的入门示例

**文件结构**：
```
.
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```

**variables.tf**：
```hcl
variable "instance_name" {
  description = "EC2 实例名称"
  type        = string
  default     = "test-instance"
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}
```

**main.tf**：
```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-west-1"
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  tags = {
    Name = var.instance_name
  }
}

output "instance_public_ip" {
  value = aws_instance.web.public_ip
}
```

**terraform.tfvars**：
```hcl
instance_name = "my-web-server"
instance_type = "t3.micro"
```

**运行**：
```bash
terraform init
terraform plan
terraform apply
terraform output
terraform destroy
```

---

### 总结

掌握以上基础语法，你就可以编写常见的 Terraform 配置来管理 AWS、Azure、GCP 等云资源。遇到不熟悉的函数或参数，官方文档是最详细的参考，但这份草案覆盖了 90% 的日常使用场景。多动手实践，很快就能熟练运用。
