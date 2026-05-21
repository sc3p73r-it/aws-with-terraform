# အခန်း (၁၆) - Complete Project: Three-Tier Web Application

---

## 16.1 Architecture Overview

### Three-Tier Architecture

ဤအခန်းတွင် Production-Grade Three-Tier Web Application ကို Terraform ဖြင့် အစမှ အဆုံး တည်ဆောက်ပါမည်။

```
                              ┌──── Internet ────┐
                              │                   │
                         ┌────▼────┐              │
                         │Route 53 │              │
                         │  DNS    │              │
                         └────┬────┘              │
                              │                   │
                         ┌────▼────┐              │
                         │CloudFront              │
                         │  CDN    │              │
                         └────┬────┘              │
                              │                   │
═══════════════════════ PRESENTATION TIER ════════════════
                              │                   │
                    ┌─────────▼──────────┐        │
                    │   ALB (Public)     │        │
                    │   Port 443/80      │        │
                    └─────────┬──────────┘        │
                     ┌────────┼────────┐          │
                     ▼        ▼        ▼          │
                  ┌─────┐ ┌─────┐ ┌─────┐        │
                  │Web 1│ │Web 2│ │Web 3│  Public │
                  └─────┘ └─────┘ └─────┘ Subnets │
                              │                   │
═══════════════════════ APPLICATION TIER ══════════════════
                              │                   │
                     ┌────────┼────────┐          │
                     ▼        ▼        ▼          │
                  ┌─────┐ ┌─────┐ ┌─────┐        │
                  │App 1│ │App 2│ │App 3│ Private │
                  └─────┘ └─────┘ └─────┘ Subnets │
                              │                   │
═══════════════════════ DATA TIER ════════════════════════
                              │                   │
                    ┌─────────▼──────────┐        │
                    │  RDS MySQL (HA)    │        │
                    │  Multi-AZ          │ Database│
                    └────────────────────┘ Subnets │
                                                  │
══════════════════════════════════════════════════════════
```

### Components

| Tier | Components |
|------|-----------|
| **Presentation** | Route 53, CloudFront, ALB, ACM Certificate |
| **Application** | EC2 (ASG), Launch Template, Security Groups |
| **Data** | RDS MySQL (Multi-AZ), Secrets Manager |
| **Networking** | VPC, Subnets (Public/Private/DB), NAT GW, IGW |
| **Monitoring** | CloudWatch, SNS, CloudTrail |

---

## 16.2 Project Structure

```
three-tier-app/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── templates/
│   │       └── user_data.sh
│   ├── database/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── loadbalancer/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── monitoring/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── providers.tf
│   │   ├── backend.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       ├── providers.tf
│       ├── backend.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── terraform.tfvars
│
├── .gitignore
├── Makefile
└── README.md
```

---

## 16.3 Main Configuration

### environments/dev/providers.tf

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}
```

### environments/dev/main.tf

```hcl
# ===================================
# VPC Module
# ===================================
module "vpc" {
  source = "../../modules/vpc"

  project_name          = var.project_name
  environment           = var.environment
  vpc_cidr              = var.vpc_cidr
  public_subnet_cidrs   = var.public_subnet_cidrs
  private_subnet_cidrs  = var.private_subnet_cidrs
  database_subnet_cidrs = var.database_subnet_cidrs
  enable_nat_gateway    = true
  single_nat_gateway    = var.environment != "prod"
}

# ===================================
# Load Balancer Module
# ===================================
module "alb" {
  source = "../../modules/loadbalancer"

  project_name      = var.project_name
  environment       = var.environment
  vpc_id            = module.vpc.vpc_id
  public_subnet_ids = module.vpc.public_subnet_ids
  health_check_path = "/health"
  # certificate_arn = var.certificate_arn  # Optional
}

# ===================================
# Compute Module (EC2 + ASG)
# ===================================
module "compute" {
  source = "../../modules/compute"

  project_name       = var.project_name
  environment        = var.environment
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnet_ids
  target_group_arn   = module.alb.target_group_arn
  alb_sg_id          = module.alb.security_group_id

  instance_type    = var.instance_type
  min_size         = var.min_size
  max_size         = var.max_size
  desired_capacity = var.desired_capacity

  db_endpoint = module.database.db_endpoint
  db_name     = var.db_name
}

# ===================================
# Database Module
# ===================================
module "database" {
  source = "../../modules/database"

  project_name        = var.project_name
  environment         = var.environment
  vpc_id              = module.vpc.vpc_id
  database_subnet_ids = module.vpc.database_subnet_ids
  app_sg_id           = module.compute.security_group_id

  db_name        = var.db_name
  db_username    = var.db_username
  db_password    = var.db_password
  instance_class = var.db_instance_class
  multi_az       = var.environment == "prod"
}

# ===================================
# Monitoring Module
# ===================================
module "monitoring" {
  source = "../../modules/monitoring"

  project_name = var.project_name
  environment  = var.environment
  alert_email  = var.alert_email

  asg_name               = module.compute.asg_name
  alb_arn_suffix         = module.alb.alb_arn_suffix
  db_instance_identifier = module.database.db_identifier
}
```

### environments/dev/variables.tf

```hcl
variable "aws_region" {
  default = "ap-southeast-1"
}

variable "project_name" {
  default = "three-tier-app"
}

variable "environment" {
  default = "dev"
}

# VPC
variable "vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  default = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  default = ["10.0.3.0/24", "10.0.4.0/24"]
}

variable "database_subnet_cidrs" {
  default = ["10.0.5.0/24", "10.0.6.0/24"]
}

# Compute
variable "instance_type" {
  default = "t2.micro"
}

variable "min_size" {
  default = 1
}

variable "max_size" {
  default = 3
}

variable "desired_capacity" {
  default = 2
}

# Database
variable "db_name" {
  default = "appdb"
}

variable "db_username" {
  sensitive = true
}

variable "db_password" {
  sensitive = true
}

variable "db_instance_class" {
  default = "db.t3.micro"
}

# Monitoring
variable "alert_email" {
  default = "admin@example.com"
}
```

### environments/dev/terraform.tfvars

```hcl
aws_region   = "ap-southeast-1"
project_name = "three-tier-app"
environment  = "dev"

# Compute
instance_type    = "t2.micro"
min_size         = 1
max_size         = 3
desired_capacity = 2

# Database
db_name           = "appdb"
db_instance_class = "db.t3.micro"

# Monitoring
alert_email = "admin@example.com"

# Sensitive values - use environment variables or secrets
# export TF_VAR_db_username="admin"
# export TF_VAR_db_password="secure-password-here"
```

### environments/dev/outputs.tf

```hcl
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "alb_dns_name" {
  description = "Application URL"
  value       = module.alb.alb_dns_name
}

output "db_endpoint" {
  description = "Database Endpoint"
  value       = module.database.db_endpoint
  sensitive   = true
}

output "asg_name" {
  value = module.compute.asg_name
}
```

---

## 16.4 Module Implementations

### Compute Module (modules/compute/main.tf)

```hcl
# AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

# Security Group
resource "aws_security_group" "app" {
  name   = "${var.project_name}-${var.environment}-app-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [var.alb_sg_id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project_name}-${var.environment}-app-sg" }
}

# IAM Role
resource "aws_iam_role" "app" {
  name = "${var.project_name}-${var.environment}-app-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.app.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "app" {
  name = "${var.project_name}-${var.environment}-app-profile"
  role = aws_iam_role.app.name
}

# Launch Template
resource "aws_launch_template" "app" {
  name_prefix   = "${var.project_name}-${var.environment}-"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  vpc_security_group_ids = [aws_security_group.app.id]

  iam_instance_profile {
    name = aws_iam_instance_profile.app.name
  }

  user_data = base64encode(templatefile("${path.module}/templates/user_data.sh", {
    project_name = var.project_name
    environment  = var.environment
    db_endpoint  = var.db_endpoint
    db_name      = var.db_name
  }))

  metadata_options {
    http_tokens = "required"
  }

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size = 20
      volume_type = "gp3"
      encrypted   = true
    }
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "${var.project_name}-${var.environment}-app"
    }
  }

  lifecycle { create_before_destroy = true }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "app" {
  name                = "${var.project_name}-${var.environment}-asg"
  desired_capacity    = var.desired_capacity
  min_size            = var.min_size
  max_size            = var.max_size
  vpc_zone_identifier = var.private_subnet_ids
  target_group_arns   = [var.target_group_arn]

  health_check_type         = "ELB"
  health_check_grace_period = 300

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 50
    }
  }

  tag {
    key                 = "Name"
    value               = "${var.project_name}-${var.environment}-app"
    propagate_at_launch = true
  }
}

# Target Tracking Policy
resource "aws_autoscaling_policy" "cpu" {
  name                   = "${var.project_name}-${var.environment}-cpu-tracking"
  policy_type            = "TargetTrackingScaling"
  autoscaling_group_name = aws_autoscaling_group.app.name

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 60.0
  }
}
```

### User Data Script (modules/compute/templates/user_data.sh)

```bash
#!/bin/bash
set -e
exec > >(tee /var/log/user-data.log) 2>&1

echo "=== Starting ${project_name} ${environment} setup ==="

# Install packages
yum update -y
yum install -y nginx jq

# Configure Nginx
cat > /usr/share/nginx/html/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head><title>${project_name} - ${environment}</title></head>
<body>
  <h1>${project_name}</h1>
  <p>Environment: ${environment}</p>
  <p>DB: ${db_endpoint}</p>
</body>
</html>
EOF

cat > /usr/share/nginx/html/health <<'EOF'
OK
EOF

systemctl start nginx
systemctl enable nginx

echo "=== Setup Complete ==="
```

---

## 16.5 Deploy & Verify

```bash
# Navigate to environment
cd environments/dev

# Set sensitive variables
export TF_VAR_db_username="admin"
export TF_VAR_db_password="MySecurePassword123!"

# Initialize
terraform init

# Plan
terraform plan -out=tfplan

# Apply
terraform apply tfplan

# Outputs
terraform output alb_dns_name
# → three-tier-app-dev-alb-123456789.ap-southeast-1.elb.amazonaws.com

# Test
curl http://$(terraform output -raw alb_dns_name)/health
# OK
```

---

## 16.6 Complete Code Walkthrough

### ဖြစ်ပေါ်လာမည့် AWS Resources

| # | Resource | Count |
|---|----------|-------|
| 1 | VPC | 1 |
| 2 | Public Subnets | 2 |
| 3 | Private Subnets | 2 |
| 4 | Database Subnets | 2 |
| 5 | Internet Gateway | 1 |
| 6 | NAT Gateway | 1 |
| 7 | Route Tables | 4 |
| 8 | Security Groups | 3 |
| 9 | ALB | 1 |
| 10 | Target Group | 1 |
| 11 | Launch Template | 1 |
| 12 | Auto Scaling Group | 1 |
| 13 | EC2 Instances | 2 |
| 14 | RDS Instance | 1 |
| 15 | IAM Roles | 2 |
| 16 | CloudWatch Alarms | 3 |
| 17 | SNS Topic | 1 |
| **Total** | | **~30 Resources** |

### Estimated Monthly Cost (Dev)

| Resource | Monthly Cost |
|----------|-------------|
| EC2 (2x t2.micro) | $0 (Free Tier) / ~$17 |
| RDS (db.t3.micro) | ~$15 |
| NAT Gateway | ~$32 |
| ALB | ~$16 |
| S3, CloudWatch | ~$2 |
| **Total (Dev)** | **~$65-82/month** |

### Cleanup

```bash
# ⚠️ Development ပြီးရင် ဖျက်ပါ (Cost Saving)
terraform destroy

# Confirm: yes
```

---

## အခန်း (၁၆) အကျဉ်းချုပ်

- ✅ **Three-Tier Architecture** ဒီဇိုင်း
- ✅ **Module-Based** Project Structure
- ✅ **VPC, ALB, ASG, RDS** ပေါင်းစပ်ခြင်း
- ✅ **Security Best Practices** (SG, IAM, Encryption)
- ✅ **Auto Scaling** (CPU-based)
- ✅ **Monitoring** (CloudWatch, SNS)
- ✅ **Cost Estimation**

---

# 🎉 စာအုပ် ပြီးဆုံးပါပြီ!

ဤစာအုပ်ကို ဖတ်ပြီးနောက် အောက်ပါ အရည်အချင်းများ ရရှိပါမည်:

1. ✅ Terraform ဖြင့် AWS Infrastructure ကို Code ဖြင့် စီမံခန့်ခွဲနိုင်ခြင်း
2. ✅ VPC, EC2, S3, IAM, RDS, ALB, Route 53, CloudFront စသည့် Core Services များ ကို Terraform ဖြင့် တည်ဆောက်နိုင်ခြင်း
3. ✅ Modules, State Management, Workspaces ဖြင့် Production-Grade Code ရေးသားနိုင်ခြင်း
4. ✅ CI/CD Pipeline ဖြင့် Infrastructure Deployment ကို Automate လုပ်နိုင်ခြင်း
5. ✅ Security, Monitoring, Cost Optimization Best Practices များ ကျင့်သုံးနိုင်ခြင်း

> **နောက်ထပ် လေ့လာရန်:**
> - [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
> - [AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
> - [Terraform Registry](https://registry.terraform.io/)
> - [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)

---

*ကျေးဇူးတင်ပါသည်! 🙏*

*[← အခန်း (၁၅)](chapter-15.md) | [မာတိကာ →](../README.md)*
