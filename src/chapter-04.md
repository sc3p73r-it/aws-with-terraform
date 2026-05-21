# အခန်း (၄) - VPC Networking တည်ဆောက်ခြင်း

---

## 4.1 VPC အခြေခံ သဘောတရားများ

### VPC ဆိုတာ ဘာလဲ?

**VPC (Virtual Private Cloud)** ဆိုတာ AWS Cloud ထဲမှာ ကိုယ်ပိုင် **Virtual Network** တစ်ခု ဖန်တီးခြင်း ဖြစ်ပါတယ်။ ကိုယ့်အိမ်မှာ Router ထားပြီး ကိုယ်ပိုင် Network ရှိသလိုမျိုးပဲ - AWS ထဲမှာ ကိုယ်ပိုင် Isolated Network ဖန်တီးပေးတာပါ။

### VPC Architecture Overview

```
┌─────────────────────────── AWS Region (ap-southeast-1) ────────────────────────┐
│                                                                                 │
│  ┌──────────────────────── VPC (10.0.0.0/16) ───────────────────────────────┐   │
│  │                                                                           │   │
│  │  ┌─── AZ-1a ───────────────┐    ┌─── AZ-1b ───────────────┐             │   │
│  │  │                          │    │                          │             │   │
│  │  │  ┌── Public Subnet ──┐  │    │  ┌── Public Subnet ──┐  │             │   │
│  │  │  │   10.0.1.0/24     │  │    │  │   10.0.2.0/24     │  │             │   │
│  │  │  │   [Web Server]    │  │    │  │   [Web Server]    │  │             │   │
│  │  │  └───────────────────┘  │    │  └───────────────────┘  │             │   │
│  │  │                          │    │                          │             │   │
│  │  │  ┌── Private Subnet ─┐  │    │  ┌── Private Subnet ─┐  │             │   │
│  │  │  │   10.0.3.0/24     │  │    │  │   10.0.4.0/24     │  │             │   │
│  │  │  │   [App Server]    │  │    │  │   [App Server]    │  │             │   │
│  │  │  └───────────────────┘  │    │  └───────────────────┘  │             │   │
│  │  │                          │    │                          │             │   │
│  │  │  ┌── DB Subnet ──────┐  │    │  ┌── DB Subnet ──────┐  │             │   │
│  │  │  │   10.0.5.0/24     │  │    │  │   10.0.6.0/24     │  │             │   │
│  │  │  │   [Database]      │  │    │  │   [Database]      │  │             │   │
│  │  │  └───────────────────┘  │    │  └───────────────────┘  │             │   │
│  │  └──────────────────────────┘    └──────────────────────────┘             │   │
│  │                                                                           │   │
│  │  ┌─── Internet Gateway ───┐    ┌─── NAT Gateway ───────┐                │   │
│  │  │   (Public Internet)     │    │   (Private → Internet) │                │   │
│  │  └─────────────────────────┘    └───────────────────────┘                │   │
│  └───────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### အဓိက Components

| Component | ရှင်းလင်းချက် |
|-----------|---------------|
| **VPC** | Virtual Private Cloud - ကိုယ်ပိုင် Network |
| **Subnet** | VPC ထဲမှ IP Address Range ခွဲခြားချက် |
| **Internet Gateway** | VPC ကို Public Internet နှင့် ချိတ်ဆက်ပေး |
| **NAT Gateway** | Private Subnet မှ Internet ကို Access ပေး (Outbound Only) |
| **Route Table** | Network Traffic ကို လမ်းညွှန်ပေး |
| **Security Group** | Instance Level Firewall (Stateful) |
| **Network ACL** | Subnet Level Firewall (Stateless) |

### Public vs Private Subnet

| Feature | Public Subnet | Private Subnet |
|---------|--------------|----------------|
| **Internet Access** | ✅ Direct (IGW) | ⚠️ Via NAT Gateway |
| **Public IP** | ✅ Auto-assign ရနိုင် | ❌ မရ |
| **Route Table** | IGW သို့ Route ရှိ | NAT GW သို့ Route ရှိ |
| **သင့်တော်** | Web Servers, ALB | App Servers, Database |

---

## 4.2 VPC, Subnets (Public/Private) ဖန်တီးခြင်း

### Project Structure

```
vpc-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
└── terraform.tfvars
```

### variables.tf

```hcl
# variables.tf
variable "aws_region" {
  description = "AWS Region"
  type        = string
  default     = "ap-southeast-1"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "myproject"
}

variable "environment" {
  description = "Environment"
  type        = string
  default     = "dev"
}

variable "vpc_cidr" {
  description = "VPC CIDR Block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "Public Subnet CIDR Blocks"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  description = "Private Subnet CIDR Blocks"
  type        = list(string)
  default     = ["10.0.3.0/24", "10.0.4.0/24"]
}

variable "database_subnet_cidrs" {
  description = "Database Subnet CIDR Blocks"
  type        = list(string)
  default     = ["10.0.5.0/24", "10.0.6.0/24"]
}
```

### main.tf - VPC ဖန်တီးခြင်း

```hcl
# main.tf

# ===================================
# Data Sources
# ===================================
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 2)

  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ===================================
# VPC
# ===================================
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-vpc"
  })
}

# ===================================
# Public Subnets
# ===================================
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true  # Auto-assign Public IP

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-public-${local.azs[count.index]}"
    Type = "public"
  })
}

# ===================================
# Private Subnets
# ===================================
resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-private-${local.azs[count.index]}"
    Type = "private"
  })
}

# ===================================
# Database Subnets
# ===================================
resource "aws_subnet" "database" {
  count = length(var.database_subnet_cidrs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.database_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-database-${local.azs[count.index]}"
    Type = "database"
  })
}
```

---

## 4.3 Internet Gateway နှင့် NAT Gateway

### Internet Gateway

```hcl
# ===================================
# Internet Gateway
# ===================================
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-igw"
  })
}
```

### Elastic IP for NAT Gateway

```hcl
# ===================================
# Elastic IP for NAT Gateway
# ===================================
resource "aws_eip" "nat" {
  count  = length(var.public_subnet_cidrs)
  domain = "vpc"

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-nat-eip-${count.index + 1}"
  })

  depends_on = [aws_internet_gateway.main]
}
```

### NAT Gateway

```hcl
# ===================================
# NAT Gateway (Public Subnet ထဲမှာ ထား)
# ===================================
resource "aws_nat_gateway" "main" {
  count = length(var.public_subnet_cidrs)

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-nat-gw-${count.index + 1}"
  })

  depends_on = [aws_internet_gateway.main]
}
```

> **💡 Cost Tip:** NAT Gateway သည် စရိတ်မြင့်ပါတယ် ($0.045/hr ≈ $32/month)။ Development Environment တွင် NAT Gateway **တစ်ခုတည်း** ထားပြီး Production တွင်သာ AZ တိုင်းမှာ ထားပါ။

### Cost-Optimized NAT Gateway (Dev Environment)

```hcl
# Dev Environment - NAT Gateway တစ်ခုတည်း
resource "aws_nat_gateway" "main" {
  count = var.environment == "prod" ? length(var.public_subnet_cidrs) : 1

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-nat-gw-${count.index + 1}"
  })
}
```

---

## 4.4 Route Tables နှင့် Route Associations

### Public Route Table

```hcl
# ===================================
# Public Route Table
# ===================================
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-public-rt"
  })
}

# Public Subnet များကို Public Route Table နှင့် ချိတ်ဆက်
resource "aws_route_table_association" "public" {
  count = length(var.public_subnet_cidrs)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

### Private Route Table

```hcl
# ===================================
# Private Route Tables (AZ တစ်ခုချင်းစီအတွက်)
# ===================================
resource "aws_route_table" "private" {
  count  = length(var.private_subnet_cidrs)
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[
      var.environment == "prod" ? count.index : 0
    ].id
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-private-rt-${count.index + 1}"
  })
}

# Private Subnet များကို Private Route Table နှင့် ချိတ်ဆက်
resource "aws_route_table_association" "private" {
  count = length(var.private_subnet_cidrs)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

### Database Route Table

```hcl
# ===================================
# Database Route Table (Internet Access မလို)
# ===================================
resource "aws_route_table" "database" {
  vpc_id = aws_vpc.main.id

  # Internet Route မထည့် - Database ကို Internet ကနေ Access မရစေချင်

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-database-rt"
  })
}

resource "aws_route_table_association" "database" {
  count = length(var.database_subnet_cidrs)

  subnet_id      = aws_subnet.database[count.index].id
  route_table_id = aws_route_table.database.id
}
```

---

## 4.5 Security Groups နှင့် Network ACLs

### Security Groups

Security Groups သည် **Instance Level** Firewall ဖြစ်ပြီး **Stateful** ဖြစ်ပါတယ်:

```hcl
# ===================================
# Web Server Security Group
# ===================================
resource "aws_security_group" "web" {
  name        = "${var.project_name}-${var.environment}-web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  # HTTP
  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTPS
  ingress {
    description = "HTTPS from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # SSH (ကိုယ့် IP ကိုသာ ခွင့်ပြု)
  ingress {
    description = "SSH from my IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["YOUR_IP/32"]  # ← ကိုယ့် IP ထည့်ပါ
  }

  # Outbound - အားလုံးခွင့်ပြု
  egress {
    description = "All outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-web-sg"
  })
}

# ===================================
# Application Server Security Group
# ===================================
resource "aws_security_group" "app" {
  name        = "${var.project_name}-${var.environment}-app-sg"
  description = "Security group for application servers"
  vpc_id      = aws_vpc.main.id

  # Web SG ကိုသာ ခွင့်ပြု (Port 8080)
  ingress {
    description     = "App port from web servers"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]  # ← SG Reference
  }

  # SSH from Bastion
  ingress {
    description     = "SSH from web servers"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-app-sg"
  })
}

# ===================================
# Database Security Group
# ===================================
resource "aws_security_group" "database" {
  name        = "${var.project_name}-${var.environment}-db-sg"
  description = "Security group for databases"
  vpc_id      = aws_vpc.main.id

  # MySQL - App SG ကိုသာ ခွင့်ပြု
  ingress {
    description     = "MySQL from app servers"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  # PostgreSQL
  ingress {
    description     = "PostgreSQL from app servers"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-db-sg"
  })
}
```

### Network ACLs

```hcl
# ===================================
# Public Subnet Network ACL
# ===================================
resource "aws_network_acl" "public" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.public[*].id

  # Inbound HTTP
  ingress {
    rule_no    = 100
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }

  # Inbound HTTPS
  ingress {
    rule_no    = 110
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }

  # Inbound SSH
  ingress {
    rule_no    = 120
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 22
    to_port    = 22
  }

  # Inbound Ephemeral Ports (Return Traffic)
  ingress {
    rule_no    = 140
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }

  # Outbound - All
  egress {
    rule_no    = 100
    protocol   = "-1"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-public-nacl"
  })
}
```

### Security Group vs Network ACL

| Feature | Security Group | Network ACL |
|---------|---------------|-------------|
| **Level** | Instance Level | Subnet Level |
| **Stateful** | ✅ Yes | ❌ No (Stateless) |
| **Rules** | Allow Rules Only | Allow + Deny Rules |
| **Evaluation** | All Rules Together | Rules in Order (Rule Number) |
| **Default** | Deny All Inbound | Allow All |
| **Association** | Instance/ENI | Subnet |

---

## 4.6 VPC Peering

### VPC Peering ဆိုတာ ဘာလဲ?

VPC Peering ဆိုတာ VPC ၂ ခုကို **Private IP** ဖြင့် ချိတ်ဆက်ပေးခြင်း ဖြစ်ပါတယ်။ Internet Gateway သို့မဟုတ် VPN မလိုဘဲ VPC များအကြား Traffic ကူးနိုင်ပါတယ်။

```hcl
# ===================================
# VPC Peering
# ===================================

# VPC A
resource "aws_vpc" "vpc_a" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "vpc-a"
  }
}

# VPC B
resource "aws_vpc" "vpc_b" {
  cidr_block = "10.1.0.0/16"

  tags = {
    Name = "vpc-b"
  }
}

# Peering Connection
resource "aws_vpc_peering_connection" "a_to_b" {
  vpc_id      = aws_vpc.vpc_a.id     # Requester VPC
  peer_vpc_id = aws_vpc.vpc_b.id     # Accepter VPC
  auto_accept = true                  # Same Account ဆိုရင် Auto Accept

  tags = {
    Name = "vpc-a-to-vpc-b-peering"
  }
}

# VPC A Route Table - VPC B CIDR ကို Peering Connection သို့
resource "aws_route" "a_to_b" {
  route_table_id            = aws_route_table.vpc_a_rt.id
  destination_cidr_block    = aws_vpc.vpc_b.cidr_block  # 10.1.0.0/16
  vpc_peering_connection_id = aws_vpc_peering_connection.a_to_b.id
}

# VPC B Route Table - VPC A CIDR ကို Peering Connection သို့
resource "aws_route" "b_to_a" {
  route_table_id            = aws_route_table.vpc_b_rt.id
  destination_cidr_block    = aws_vpc.vpc_a.cidr_block  # 10.0.0.0/16
  vpc_peering_connection_id = aws_vpc_peering_connection.a_to_b.id
}
```

---

## 4.7 VPC Flow Logs

### VPC Flow Logs ဖြင့် Network Traffic Monitoring

```hcl
# ===================================
# VPC Flow Logs
# ===================================

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "vpc_flow_logs" {
  name              = "/aws/vpc/${var.project_name}-${var.environment}-flow-logs"
  retention_in_days = 30

  tags = local.common_tags
}

# IAM Role for Flow Logs
resource "aws_iam_role" "vpc_flow_logs" {
  name = "${var.project_name}-${var.environment}-vpc-flow-logs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "vpc-flow-logs.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "vpc_flow_logs" {
  name = "vpc-flow-logs-policy"
  role = aws_iam_role.vpc_flow_logs.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ]
      Effect   = "Allow"
      Resource = "*"
    }]
  })
}

# VPC Flow Log
resource "aws_flow_log" "main" {
  vpc_id                   = aws_vpc.main.id
  traffic_type             = "ALL"  # ALL, ACCEPT, REJECT
  log_destination_type     = "cloud-watch-logs"
  log_destination          = aws_cloudwatch_log_group.vpc_flow_logs.arn
  iam_role_arn             = aws_iam_role.vpc_flow_logs.arn
  max_aggregation_interval = 60  # seconds

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-vpc-flow-log"
  })
}
```

---

## 4.8 လက်တွေ့ - Multi-AZ VPC Architecture တည်ဆောက်ခြင်း

### Complete outputs.tf

```hcl
# outputs.tf
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "VPC CIDR Block"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "Public Subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "Private Subnet IDs"
  value       = aws_subnet.private[*].id
}

output "database_subnet_ids" {
  description = "Database Subnet IDs"
  value       = aws_subnet.database[*].id
}

output "internet_gateway_id" {
  description = "Internet Gateway ID"
  value       = aws_internet_gateway.main.id
}

output "nat_gateway_ids" {
  description = "NAT Gateway IDs"
  value       = aws_nat_gateway.main[*].id
}

output "web_security_group_id" {
  description = "Web Security Group ID"
  value       = aws_security_group.web.id
}

output "app_security_group_id" {
  description = "App Security Group ID"
  value       = aws_security_group.app.id
}

output "database_security_group_id" {
  description = "Database Security Group ID"
  value       = aws_security_group.database.id
}
```

### Apply & Verify

```bash
$ terraform init
$ terraform plan
$ terraform apply

# Outputs:
vpc_id                     = "vpc-0a1b2c3d4e5f6g7h8"
vpc_cidr                   = "10.0.0.0/16"
public_subnet_ids          = ["subnet-pub1...", "subnet-pub2..."]
private_subnet_ids         = ["subnet-priv1...", "subnet-priv2..."]
database_subnet_ids        = ["subnet-db1...", "subnet-db2..."]
internet_gateway_id        = "igw-0a1b2c3d..."
nat_gateway_ids            = ["nat-0a1b2c3d..."]
web_security_group_id      = "sg-web..."
app_security_group_id      = "sg-app..."
database_security_group_id = "sg-db..."
```

---

## အခန်း (၄) အကျဉ်းချုပ်

ဤအခန်းတွင် အောက်ပါ အချက်များကို လေ့လာခဲ့ပါသည်:

- ✅ **VPC** - Virtual Private Cloud ဖန်တီးခြင်း
- ✅ **Subnets** - Public, Private, Database Subnets
- ✅ **Internet Gateway** - Public Internet ချိတ်ဆက်ခြင်း
- ✅ **NAT Gateway** - Private Subnet Internet Access
- ✅ **Route Tables** - Network Traffic Routing
- ✅ **Security Groups** - Instance Level Firewall (Web, App, DB)
- ✅ **Network ACLs** - Subnet Level Firewall
- ✅ **VPC Peering** - VPC များ ချိတ်ဆက်ခြင်း
- ✅ **VPC Flow Logs** - Network Traffic Monitoring

> **📖 နောက်အခန်းတွင်** EC2 Instances ကို Terraform ဖြင့် ဖန်တီးခြင်း၊ User Data Scripts, Launch Templates, Auto Scaling Groups များကို လေ့လာကြပါမည်။

---

*[← အခန်း (၃)](chapter-03.md) | [အခန်း (၅) - EC2 Instances →](chapter-05.md)*
