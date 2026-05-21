# အခန်း (၁၁) - Modules ဖြင့် Code ပြန်သုံးခြင်း

---

## 11.1 Module ဆိုတာ ဘာလဲ? ဘာကြောင့် သုံးရသလဲ?

### Module ဆိုတာ ဘာလဲ?

**Module** ဆိုတာ **ပြန်သုံးနိုင်သော Terraform Code အစုအဝေး** ဖြစ်ပါတယ်။ Programming Language တွေမှာ Function/Class လို Terraform မှာ Module ရှိပါတယ်။

### Module မသုံးရင် ဖြစ်နိုင်တဲ့ ပြဿနာ

```
# Module မသုံးဘူးဆိုရင်...
dev/
├── main.tf          (VPC + EC2 + S3 + RDS = 500+ lines)
├── variables.tf
└── outputs.tf

staging/
├── main.tf          (တူညီသော Code 500+ lines COPY)
├── variables.tf
└── outputs.tf

prod/
├── main.tf          (ထပ်ပြီး COPY 500+ lines)
├── variables.tf
└── outputs.tf

# ပြဿနာ: VPC Config ပြောင်းချင်ရင် ၃ နေရာ ပြင်ရတယ်!
```

### Module သုံးရင်

```
modules/
├── vpc/             # VPC Module (ရေးထားတဲ့ Code)
├── ec2/             # EC2 Module
├── rds/             # RDS Module
└── s3/              # S3 Module

environments/
├── dev/
│   └── main.tf      # module "vpc" { source = "../../modules/vpc" }
├── staging/
│   └── main.tf      # module "vpc" { source = "../../modules/vpc" }
└── prod/
    └── main.tf      # module "vpc" { source = "../../modules/vpc" }

# VPC ပြောင်းချင်ရင် modules/vpc/ တစ်နေရာတည်းမှာ ပြင်ရုံပဲ!
```

---

## 11.2 Local Modules ဖန်တီးခြင်း

### VPC Module

```
modules/vpc/
├── main.tf
├── variables.tf
├── outputs.tf
└── README.md
```

**modules/vpc/variables.tf:**

```hcl
variable "project_name" {
  description = "Project name"
  type        = string
}

variable "environment" {
  description = "Environment (dev, staging, prod)"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "Public subnet CIDR blocks"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  description = "Private subnet CIDR blocks"
  type        = list(string)
  default     = ["10.0.3.0/24", "10.0.4.0/24"]
}

variable "enable_nat_gateway" {
  description = "Enable NAT Gateway"
  type        = bool
  default     = true
}

variable "single_nat_gateway" {
  description = "Use single NAT Gateway (cost saving)"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Additional tags"
  type        = map(string)
  default     = {}
}
```

**modules/vpc/main.tf:**

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, length(var.public_subnet_cidrs))

  common_tags = merge(var.tags, {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
  })
}

# VPC
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-vpc"
  })
}

# Public Subnets
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-public-${local.azs[count.index]}"
    Type = "public"
  })
}

# Private Subnets
resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-private-${local.azs[count.index]}"
    Type = "private"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-igw"
  })
}

# NAT Gateway
resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : length(var.public_subnet_cidrs)) : 0
  domain = "vpc"

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-nat-eip-${count.index + 1}"
  })
}

resource "aws_nat_gateway" "this" {
  count = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : length(var.public_subnet_cidrs)) : 0

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-nat-${count.index + 1}"
  })

  depends_on = [aws_internet_gateway.this]
}

# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-public-rt"
  })
}

resource "aws_route_table_association" "public" {
  count          = length(var.public_subnet_cidrs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private Route Tables
resource "aws_route_table" "private" {
  count  = var.enable_nat_gateway ? length(var.private_subnet_cidrs) : 0
  vpc_id = aws_vpc.this.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.this[var.single_nat_gateway ? 0 : count.index].id
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-private-rt-${count.index + 1}"
  })
}

resource "aws_route_table_association" "private" {
  count          = var.enable_nat_gateway ? length(var.private_subnet_cidrs) : 0
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

**modules/vpc/outputs.tf:**

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.this.id
}

output "vpc_cidr" {
  description = "VPC CIDR"
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "Public Subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "Private Subnet IDs"
  value       = aws_subnet.private[*].id
}

output "internet_gateway_id" {
  description = "Internet Gateway ID"
  value       = aws_internet_gateway.this.id
}

output "nat_gateway_ids" {
  description = "NAT Gateway IDs"
  value       = aws_nat_gateway.this[*].id
}
```

---

## 11.3 Module Inputs နှင့် Outputs

### Module ကို ခေါ်သုံးခြင်း

```hcl
# environments/dev/main.tf

module "vpc" {
  source = "../../modules/vpc"

  project_name         = "myapp"
  environment          = "dev"
  vpc_cidr             = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.3.0/24", "10.0.4.0/24"]
  enable_nat_gateway   = true
  single_nat_gateway   = true  # Dev: Cost saving

  tags = {
    Team = "platform"
  }
}

# Module Output ကို ခေါ်သုံးခြင်း
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"
  subnet_id     = module.vpc.public_subnet_ids[0]  # ← Module Output

  tags = {
    Name = "web-server"
  }
}

output "vpc_id" {
  value = module.vpc.vpc_id  # ← Module Output
}
```

### Production Environment

```hcl
# environments/prod/main.tf

module "vpc" {
  source = "../../modules/vpc"

  project_name         = "myapp"
  environment          = "prod"
  vpc_cidr             = "10.1.0.0/16"      # Different CIDR
  public_subnet_cidrs  = ["10.1.1.0/24", "10.1.2.0/24", "10.1.3.0/24"]
  private_subnet_cidrs = ["10.1.4.0/24", "10.1.5.0/24", "10.1.6.0/24"]
  enable_nat_gateway   = true
  single_nat_gateway   = false  # Prod: High Availability

  tags = {
    Team = "platform"
  }
}
```

---

## 11.4 Module Composition (Nested Modules)

```hcl
# infrastructure/main.tf - Multiple Modules ပေါင်းစပ်
module "vpc" {
  source = "../modules/vpc"
  # ... VPC config
}

module "ec2" {
  source = "../modules/ec2"

  vpc_id             = module.vpc.vpc_id              # VPC Module Output
  subnet_ids         = module.vpc.private_subnet_ids   # VPC Module Output
  security_group_ids = [module.security.app_sg_id]     # Security Module Output

  instance_type = "t3.medium"
  instance_count = 3
}

module "rds" {
  source = "../modules/rds"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids

  engine         = "mysql"
  instance_class = "db.r5.large"
  db_name        = "appdb"
  username       = var.db_username
  password       = var.db_password
}

module "alb" {
  source = "../modules/alb"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.public_subnet_ids

  target_group_port = 8080
  health_check_path = "/api/health"
}
```

---

## 11.5 Terraform Registry မှ Public Modules

### VPC Module (Popular)

```hcl
# Official AWS VPC Module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.1"

  name = "${var.project_name}-${var.environment}"
  cidr = "10.0.0.0/16"

  azs             = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = var.environment != "prod"
  enable_vpn_gateway = false

  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = local.common_tags
}
```

### Security Group Module

```hcl
module "web_sg" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "5.1.0"

  name        = "${var.project_name}-web-sg"
  description = "Web server security group"
  vpc_id      = module.vpc.vpc_id

  ingress_with_cidr_blocks = [
    {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = "0.0.0.0/0"
    },
    {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = "0.0.0.0/0"
    }
  ]

  egress_with_cidr_blocks = [
    {
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = "0.0.0.0/0"
    }
  ]

  tags = local.common_tags
}
```

---

## 11.6 Module Versioning

```hcl
# Exact Version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.1"
}

# Version Range
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"   # 5.x (>= 5.0, < 6.0)
}

# Git Source with Tag
module "vpc" {
  source = "git::https://github.com/org/terraform-modules.git//vpc?ref=v1.2.0"
}

# Local Path
module "vpc" {
  source = "../../modules/vpc"
}
```

---

## 11.7 လက်တွေ့ - VPC Module နှင့် EC2 Module ဖန်တီးခြင်း

### EC2 Module

**modules/ec2/variables.tf:**
```hcl
variable "project_name" {
  type = string
}

variable "environment" {
  type = string
}

variable "vpc_id" {
  type = string
}

variable "subnet_ids" {
  type = list(string)
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}

variable "instance_count" {
  type    = number
  default = 1
}

variable "key_name" {
  type    = string
  default = ""
}

variable "security_group_ids" {
  type    = list(string)
  default = []
}
```

**modules/ec2/main.tf:**
```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_instance" "this" {
  count = var.instance_count

  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_ids[count.index % length(var.subnet_ids)]
  vpc_security_group_ids = var.security_group_ids
  key_name               = var.key_name != "" ? var.key_name : null

  tags = {
    Name        = "${var.project_name}-${var.environment}-${count.index + 1}"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

**modules/ec2/outputs.tf:**
```hcl
output "instance_ids" {
  value = aws_instance.this[*].id
}

output "private_ips" {
  value = aws_instance.this[*].private_ip
}

output "public_ips" {
  value = aws_instance.this[*].public_ip
}
```

### Module ပေါင်းစပ် အသုံးပြုခြင်း

```hcl
# environments/dev/main.tf
module "vpc" {
  source       = "../../modules/vpc"
  project_name = "myapp"
  environment  = "dev"
}

module "web_servers" {
  source = "../../modules/ec2"

  project_name       = "myapp"
  environment        = "dev"
  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.public_subnet_ids
  instance_type      = "t2.micro"
  instance_count     = 2
  security_group_ids = [aws_security_group.web.id]
}

output "web_server_ips" {
  value = module.web_servers.public_ips
}
```

---

## အခန်း (၁၁) အကျဉ်းချုပ်

- ✅ **Module** ဆိုတာ ပြန်သုံးနိုင်သော Terraform Code Package
- ✅ **Local Modules** ဖန်တီးခြင်း (VPC, EC2)
- ✅ **Module Inputs/Outputs** ဖြင့် Data Sharing
- ✅ **Module Composition** - Modules ပေါင်းစပ်
- ✅ **Public Modules** - Terraform Registry
- ✅ **Module Versioning**

---

*[← အခန်း (၁၀)](chapter-10.md) | [အခန်း (၁၂) - State Management →](chapter-12.md)*
