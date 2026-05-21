# အခန်း (၁၂) - State Management နှင့် Workspaces

---

## 12.1 Terraform State ဆိုတာ ဘာလဲ?

### State File ၏ ရည်ရွယ်ချက်

**Terraform State** ဆိုတာ Terraform က Management လုပ်နေတဲ့ Infrastructure ရဲ့ **လက်ရှိ အခြေအနေ မှတ်တမ်း** ဖြစ်ပါတယ်။ `terraform.tfstate` ဟူသော JSON File ဖြစ်ပါတယ်။

```
Config (.tf files)     State (tfstate)      Real World (AWS)
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Desired State│     │Current State │     │ Actual State │
│ "ဘာ ရချင်လဲ"│────▶│"ဘာ ရှိလဲ"   │────▶│"AWS မှာ ဘာရှိ│
└──────────────┘     └──────────────┘     │    တကယ်"    │
                                          └──────────────┘
```

**Plan လုပ်ရင်:**
1. Config ကို ဖတ် (Desired State)
2. State File ကို ဖတ် (Current State)
3. ၂ ခု ယှဉ်ကြည့် (Diff)
4. ကွာခြားချက်ကို Plan အဖြစ် ပြ

### State File ဥပမာ

```json
{
  "version": 4,
  "terraform_version": "1.7.5",
  "serial": 15,
  "lineage": "abc123...",
  "outputs": {
    "vpc_id": {
      "value": "vpc-0a1b2c3d4e5f6",
      "type": "string"
    }
  },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_vpc",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "vpc-0a1b2c3d4e5f6",
            "cidr_block": "10.0.0.0/16",
            "enable_dns_hostnames": true,
            "tags": {
              "Name": "myproject-dev-vpc"
            }
          }
        }
      ]
    }
  ]
}
```

> **⚠️ သတိပြုရန်:** State File တွင် Database Password, Access Keys စသည့် **Sensitive Data** ပါဝင်နိုင်ပါသည်။

---

## 12.2 Remote State (S3 + DynamoDB Backend)

### ဘာကြောင့် Remote State သုံးရသလဲ?

| Feature | Local State | Remote State (S3) |
|---------|------------|-------------------|
| **Team Sharing** | ❌ | ✅ |
| **State Locking** | ❌ | ✅ (DynamoDB) |
| **Backup** | ❌ Manual | ✅ Versioning |
| **Encryption** | ❌ | ✅ (SSE) |
| **Security** | Local Disk | IAM Controlled |

### S3 Backend Configuration

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "myproject-terraform-state"
    key            = "environments/dev/terraform.tfstate"
    region         = "ap-southeast-1"
    dynamodb_table = "myproject-terraform-locks"
    encrypt        = true

    # Optional - KMS Encryption
    # kms_key_id = "arn:aws:kms:ap-southeast-1:123456789:key/xxx"
  }
}
```

### State Key Organization

```
myproject-terraform-state/          (S3 Bucket)
├── global/
│   └── iam/
│       └── terraform.tfstate       # IAM Resources
├── environments/
│   ├── dev/
│   │   └── terraform.tfstate       # Dev Environment
│   ├── staging/
│   │   └── terraform.tfstate       # Staging Environment
│   └── prod/
│       └── terraform.tfstate       # Production
└── shared/
    └── networking/
        └── terraform.tfstate       # Shared VPC
```

### Remote State Data Source

```hcl
# VPC State ကို ခြား Config မှ ဖတ်ခြင်း
data "terraform_remote_state" "vpc" {
  backend = "s3"

  config = {
    bucket = "myproject-terraform-state"
    key    = "shared/networking/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

# Remote State Output ကို အသုံးပြုခြင်း
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"
  subnet_id     = data.terraform_remote_state.vpc.outputs.public_subnet_ids[0]
}
```

---

## 12.3 State Locking

### DynamoDB State Locking

State Locking ဆိုတာ Team Member ၂ ယောက် **တစ်ပြိုင်နက်** `terraform apply` မလုပ်နိုင်အောင် Lock ခတ်ခြင်း ဖြစ်ပါတယ်။

```
User A: terraform apply          User B: terraform apply
   │                                │
   ▼                                ▼
Lock Acquired ✅                  Lock Failed ❌
   │                              "Error acquiring the state lock"
   ▼
Apply Running...
   │
   ▼
Lock Released
                                 User B: terraform apply
                                    │
                                    ▼
                                 Lock Acquired ✅
```

### Force Unlock (Emergency)

```bash
# Lock ID ကို ရယူ (Error Message ထဲမှာ ပါတယ်)
$ terraform force-unlock LOCK_ID

# ⚠️ သတိ: Force Unlock ကို Emergency သာ သုံးပါ
# ပုံမှန် Terraform Process ပြီးရင် Lock အလိုအလျောက် ပြန်ဖွင့်ပါတယ်
```

---

## 12.4 `terraform import` - ရှိပြီးသား Resources ခေါ်ယူခြင်း

### Import ဆိုတာ ဘာလဲ?

Manual (Console) ဖြင့် ဖန်တီးထားသော AWS Resources ကို Terraform State ထဲ ထည့်သွင်းခြင်း ဖြစ်ပါတယ်။

### Import Workflow

```bash
# Step 1: Config ရေး (Resource Block)
# main.tf
resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket-name"
}

# Step 2: Import Command
$ terraform import aws_s3_bucket.existing my-existing-bucket-name

# Step 3: Plan စစ် (Config နဲ့ Actual ကွာခြားချက် ရှိမရှိ)
$ terraform plan

# Step 4: Config ကို Actual State နဲ့ ကိုက်ညီအောင် ပြင်
```

### Import ဥပမာများ

```bash
# EC2 Instance
$ terraform import aws_instance.web i-0123456789abcdef0

# VPC
$ terraform import aws_vpc.main vpc-0123456789abcdef0

# S3 Bucket
$ terraform import aws_s3_bucket.data my-bucket-name

# Security Group
$ terraform import aws_security_group.web sg-0123456789abcdef0

# RDS Instance
$ terraform import aws_db_instance.main my-rds-instance

# Route 53 Record
$ terraform import aws_route53_record.www Z12345_example.com_A

# IAM Role
$ terraform import aws_iam_role.app my-role-name
```

### Import Block (Terraform 1.5+)

```hcl
# Terraform 1.5+ Import Block
import {
  to = aws_s3_bucket.existing
  id = "my-existing-bucket-name"
}

resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket-name"
}

# terraform plan -generate-config-out=generated.tf
# ← Config ကို Auto Generate လုပ်ပေးမယ်
```

---

## 12.5 `terraform state` Commands

### State Commands

```bash
# State ထဲ Resources စာရင်း
$ terraform state list
aws_vpc.main
aws_subnet.public[0]
aws_subnet.public[1]
module.ec2.aws_instance.web

# Resource Details ကြည့်
$ terraform state show aws_vpc.main

# Resource ကို State ထဲမှ ဖယ်ထုတ် (AWS ထဲမှာ မဖျက်ဘူး)
$ terraform state rm aws_instance.old_server
# ← Terraform စီမံခန့်ခွဲမှုမှ ဖယ်ထုတ်ခြင်း

# Resource Rename / Move
$ terraform state mv aws_instance.web aws_instance.web_server
# ← Code ထဲမှာ Resource Name ပြောင်းပြီးရင် State ထဲမှာလည်း ပြောင်းပေး

# Module Resource Move
$ terraform state mv aws_instance.web module.ec2.aws_instance.web
# ← Resource ကို Module ထဲ ရွှေ့ခြင်း

# State Pull (Remote State Download)
$ terraform state pull > terraform.tfstate.backup

# State Push (Upload State - ⚠️ Dangerous)
$ terraform state push terraform.tfstate
```

### moved Block (Terraform 1.1+)

```hcl
# Resource Rename ကို Code ထဲမှာ ကြေညာ
moved {
  from = aws_instance.web
  to   = aws_instance.web_server
}

resource "aws_instance" "web_server" {
  # ... (renamed from web → web_server)
}

# Module Move
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}
```

---

## 12.6 Workspaces ဖြင့် Multi-Environment စီမံခြင်း

### Workspace ဆိုတာ ဘာလဲ?

Workspace ဆိုတာ **တူညီသော Config** ဖြင့် **ကွဲပြားသော State** ကို စီမံခြင်း ဖြစ်ပါတယ်။ Dev, Staging, Prod Environment တွေကို Workspace ဖြင့် ခွဲနိုင်ပါတယ်။

```bash
# Workspace Commands
$ terraform workspace list
* default

$ terraform workspace new dev
Created and switched to workspace "dev"!

$ terraform workspace new staging
$ terraform workspace new prod

$ terraform workspace list
  default
  dev
* staging
  prod

$ terraform workspace select dev
Switched to workspace "dev".

$ terraform workspace show
dev
```

### Workspace ကို Config ထဲ အသုံးပြုခြင်း

```hcl
# terraform.workspace ကို Variable အဖြစ် သုံး
locals {
  environment = terraform.workspace

  instance_types = {
    dev     = "t2.micro"
    staging = "t3.small"
    prod    = "t3.large"
  }

  instance_counts = {
    dev     = 1
    staging = 2
    prod    = 3
  }
}

resource "aws_instance" "web" {
  count         = local.instance_counts[local.environment]
  ami           = data.aws_ami.amazon_linux.id
  instance_type = local.instance_types[local.environment]

  tags = {
    Name        = "web-${local.environment}-${count.index + 1}"
    Environment = local.environment
  }
}
```

### S3 Backend with Workspaces

```hcl
# Workspace သုံးရင် State Key အလိုအလျောက် ကွဲသွားမယ်
terraform {
  backend "s3" {
    bucket         = "myproject-terraform-state"
    key            = "terraform.tfstate"
    region         = "ap-southeast-1"
    dynamodb_table = "myproject-terraform-locks"
    encrypt        = true

    # Workspace ပေါ်မူတည်ပြီး State Path ကွဲမယ်
    # env:/dev/terraform.tfstate
    # env:/staging/terraform.tfstate
    # env:/prod/terraform.tfstate
  }
}
```

---

## 12.7 State ဖိုင် Sensitive Data ကာကွယ်ခြင်း

```hcl
# S3 Backend Encryption
terraform {
  backend "s3" {
    bucket     = "myproject-terraform-state"
    key        = "terraform.tfstate"
    region     = "ap-southeast-1"
    encrypt    = true
    kms_key_id = "arn:aws:kms:ap-southeast-1:123456789:key/xxx"
  }
}

# S3 Bucket Policy - State Access ကန့်သတ်
resource "aws_s3_bucket_policy" "state" {
  bucket = aws_s3_bucket.terraform_state.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyUnencryptedObjectUploads"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:PutObject"
        Resource  = "${aws_s3_bucket.terraform_state.arn}/*"
        Condition = {
          StringNotEquals = {
            "s3:x-amz-server-side-encryption" = "aws:kms"
          }
        }
      },
      {
        Sid       = "DenyNonSSLRequests"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.terraform_state.arn,
          "${aws_s3_bucket.terraform_state.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }
    ]
  })
}
```

### .gitignore

```gitignore
# State Files ကို Git ထဲ COMMIT မလုပ်ပါနှင့်
*.tfstate
*.tfstate.*
*.tfvars
!example.tfvars
```

---

## 12.8 လက်တွေ့ - Dev/Staging/Prod Environments

### Directory Structure (Workspace Approach)

```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── backend.tf
├── dev.tfvars
├── staging.tfvars
└── prod.tfvars
```

### Usage

```bash
# Dev
terraform workspace select dev
terraform plan -var-file="dev.tfvars"
terraform apply -var-file="dev.tfvars"

# Staging
terraform workspace select staging
terraform plan -var-file="staging.tfvars"
terraform apply -var-file="staging.tfvars"

# Prod
terraform workspace select prod
terraform plan -var-file="prod.tfvars"
terraform apply -var-file="prod.tfvars"
```

---

## အခန်း (၁၂) အကျဉ်းချုပ်

- ✅ **Terraform State** - Infrastructure ၏ လက်ရှိ အခြေအနေ
- ✅ **Remote State** - S3 + DynamoDB Backend
- ✅ **State Locking** - DynamoDB ဖြင့် Concurrent Access ကာကွယ်
- ✅ **terraform import** - ရှိပြီးသား Resources ခေါ်ယူခြင်း
- ✅ **State Commands** - list, show, mv, rm
- ✅ **Workspaces** - Multi-Environment Management
- ✅ **State Security** - Encryption, Access Control

---

*[← အခန်း (၁၁)](chapter-11.md) | [အခန်း (၁၃) - Advanced HCL →](chapter-13.md)*
