# အခန်း (၂) - Terraform နှင့် AWS ပတ်ဝန်းကျင် ပြင်ဆင်ခြင်း

---

## 2.1 Terraform Installation (Windows, macOS, Linux)

### Windows တွင် Install လုပ်ခြင်း

**နည်းလမ်း (၁) - Chocolatey Package Manager ဖြင့်**

```powershell
# Chocolatey ရှိပြီးသားဆိုရင်
choco install terraform

# Version စစ်ကြည့်
terraform version
```

**နည်းလမ်း (၂) - Manual Download**

1. https://developer.hashicorp.com/terraform/downloads သို့ သွားပါ
2. Windows AMD64 ကို Download လုပ်ပါ
3. Zip File ကို Extract လုပ်ပါ
4. `terraform.exe` ကို PATH Environment Variable ထဲ ထည့်ပါ

```powershell
# PATH ထဲ ထည့်ခြင်း (PowerShell Admin)
$terraformPath = "C:\terraform"
[System.Environment]::SetEnvironmentVariable("PATH", $env:PATH + ";$terraformPath", "Machine")

# Terminal ကို ပိတ်ပြီး ပြန်ဖွင့်ပါ
terraform version
# Terraform v1.7.x on windows_amd64
```

**နည်းလမ်း (၃) - winget ဖြင့်**

```powershell
winget install Hashicorp.Terraform

terraform version
```

### macOS တွင် Install လုပ်ခြင်း

```bash
# Homebrew ဖြင့် (အလွယ်ဆုံး)
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Version စစ်ကြည့်
terraform version
```

### Linux (Ubuntu/Debian) တွင် Install လုပ်ခြင်း

```bash
# HashiCorp GPG key ထည့်ခြင်း
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Repository ထည့်ခြင်း
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# Install လုပ်ခြင်း
sudo apt update && sudo apt install terraform

# Version စစ်ကြည့်
terraform version
```

### Install ပြီးပြီ စစ်ကြည့်ခြင်း

```bash
$ terraform version
Terraform v1.7.5
on windows_amd64

$ terraform -help
Usage: terraform [global options] <subcommand> [args]

The available commands for execution are listed below.

Main commands:
  init          Prepare your working directory
  validate      Check configuration validity
  plan          Show changes required by the current configuration
  apply         Create or update infrastructure
  destroy       Destroy previously-created infrastructure
```

---

## 2.2 AWS CLI Installation နှင့် Configuration

### AWS CLI Install လုပ်ခြင်း

**Windows:**
```powershell
# MSI Installer Download လုပ်ပြီး Install
# https://awscli.amazonaws.com/AWSCLIV2.msi

# သို့မဟုတ် winget ဖြင့်
winget install Amazon.AWSCLI

# Version စစ်ကြည့်
aws --version
# aws-cli/2.15.0 Python/3.11.6 Windows/10 exe/AMD64
```

**macOS:**
```bash
brew install awscli
```

**Linux:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

aws --version
```

### AWS CLI Configure လုပ်ခြင်း

```bash
$ aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: ap-southeast-1
Default output format [None]: json
```

Configure လုပ်ပြီးနောက် Credentials File ကို စစ်ကြည့်နိုင်ပါတယ်:

```bash
# Windows
cat ~/.aws/credentials

# Linux/macOS
cat ~/.aws/credentials
```

```ini
# ~/.aws/credentials
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# ~/.aws/config
[default]
region = ap-southeast-1
output = json
```

### Multiple Profiles သုံးခြင်း

```bash
# Profile အသစ် ထည့်ခြင်း
aws configure --profile dev
aws configure --profile staging
aws configure --profile production

# Profile ဖြင့် Command Run ခြင်း
aws s3 ls --profile dev
aws ec2 describe-instances --profile production
```

```ini
# ~/.aws/credentials
[default]
aws_access_key_id = AKIADEFAULT...
aws_secret_access_key = wDefault...

[dev]
aws_access_key_id = AKIADEV...
aws_secret_access_key = wDev...

[production]
aws_access_key_id = AKIAPROD...
aws_secret_access_key = wProd...
```

---

## 2.3 AWS IAM User နှင့် Access Key ပြင်ဆင်ခြင်း

### Terraform အတွက် IAM User ဖန်တီးခြင်း

> **⚠️ Security Best Practice:** Root Account ကို Terraform အတွက် မသုံးပါနှင့်။ Terraform အတွက် သီးသန့် IAM User ဖန်တီးပါ။

**AWS Console မှ IAM User ဖန်တီးခြင်း:**

1. **AWS Console** → **IAM** → **Users** → **Create User**
2. User Name: `terraform-user`
3. Permissions:
   - **Attach policies directly** ကို ရွေးပါ
   - `AdministratorAccess` (Learning အတွက်)
   - Production တွင် Least Privilege Principle အရ လိုအပ်တဲ့ Permission သာ ပေးပါ

4. **Security Credentials** Tab → **Create Access Key**
   - Use case: **Command Line Interface (CLI)**
   - Access Key ID နှင့် Secret Access Key ကို မှတ်ထားပါ

### Least Privilege Policy ဥပမာ

Production Environment တွင် `AdministratorAccess` မသုံးသင့်ပါ။ လိုအပ်သော Permission များသာ ပေးပါ:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TerraformEC2Access",
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "elasticloadbalancing:*",
        "autoscaling:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "TerraformS3Access",
      "Effect": "Allow",
      "Action": [
        "s3:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "TerraformVPCAccess",
      "Effect": "Allow",
      "Action": [
        "vpc:*",
        "ec2:CreateVpc",
        "ec2:DeleteVpc",
        "ec2:DescribeVpcs",
        "ec2:CreateSubnet",
        "ec2:DeleteSubnet",
        "ec2:DescribeSubnets"
      ],
      "Resource": "*"
    },
    {
      "Sid": "TerraformIAMAccess",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:CreateInstanceProfile",
        "iam:DeleteInstanceProfile",
        "iam:GetRole",
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 2.4 AWS Provider Configuration

### အခြေခံ Provider Configuration

Terraform Project Directory ဖန်တီးပြီး Provider Configure လုပ်ပါ:

```bash
mkdir my-first-terraform
cd my-first-terraform
```

```hcl
# providers.tf
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
  region = "ap-southeast-1"  # Singapore Region
}
```

### Provider Version Constraints

```hcl
# Version Constraint Syntax
required_providers {
  aws = {
    source  = "hashicorp/aws"

    # version = "5.31.0"      # တိတိကျကျ ဤ Version ကိုသာ
    # version = ">= 5.0"      # 5.0 နှင့်အထက်
    # version = "~> 5.0"      # 5.x (5.0 - 5.99) >= 5.0, < 6.0
    # version = ">= 5.0, < 6.0"  # 5.0 မှ 6.0 မတိုင်ခင်

    version = "~> 5.0"  # Recommended
  }
}
```

### Multiple Region Provider

```hcl
# Default Provider (Singapore)
provider "aws" {
  region = "ap-southeast-1"
}

# Alias Provider (US East - Virginia)
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

# Singapore Region တွင် S3 Bucket
resource "aws_s3_bucket" "singapore_bucket" {
  bucket = "my-singapore-bucket"
}

# US East Region တွင် S3 Bucket
resource "aws_s3_bucket" "us_bucket" {
  provider = aws.us_east
  bucket   = "my-us-east-bucket"
}
```

### Provider Authentication Methods

```hcl
# နည်းလမ်း (၁) - AWS CLI Default Profile (Recommended)
provider "aws" {
  region = "ap-southeast-1"
  # ~/.aws/credentials မှ default profile ကို အလိုအလျောက် သုံးမယ်
}

# နည်းလမ်း (၂) - Named Profile
provider "aws" {
  region  = "ap-southeast-1"
  profile = "dev"
}

# နည်းလမ်း (၃) - Environment Variables
# export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
# export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI..."
# export AWS_DEFAULT_REGION="ap-southeast-1"
provider "aws" {
  # Environment Variables ကို အလိုအလျောက် ဖတ်မယ်
}

# ❌ နည်းလမ်း (၄) - Hard-coded (မသုံးပါနှင့်!)
provider "aws" {
  region     = "ap-southeast-1"
  access_key = "AKIAIOSFODNN7EXAMPLE"     # ❌ Security Risk!
  secret_key = "wJalrXUtnFEMI..."          # ❌ Git ထဲ ဝင်သွားနိုင်!
}
```

> **⚠️ သတိပြုရန်:** Access Key နှင့် Secret Key ကို `.tf` File ထဲ Hard-code မလုပ်ပါနှင့်။ AWS CLI Profile သို့မဟုတ် Environment Variables ကို အသုံးပြုပါ။

---

## 2.5 VS Code Extensions နှင့် Development Tools

### VS Code Extensions

Terraform Development အတွက် အသုံးဝင်သော Extensions များ:

| Extension | Publisher | ရည်ရွယ်ချက် |
|-----------|-----------|-------------|
| **HashiCorp Terraform** | HashiCorp | Syntax highlighting, auto-complete, format |
| **Terraform doc snippets** | Run at Scale | Code snippets for resources |
| **AWS Toolkit** | Amazon Web Services | AWS Resource explorer |
| **GitLens** | GitKraken | Git history visualization |
| **indent-rainbow** | oderwat | Indentation visualization |

### HashiCorp Terraform Extension Install

```
VS Code → Extensions (Ctrl+Shift+X) → "HashiCorp Terraform" ရှာ → Install
```

**Extension ရဲ့ Feature များ:**
- ✅ Syntax Highlighting
- ✅ Auto-completion (Resources, Arguments)
- ✅ Go to Definition
- ✅ Hover Documentation
- ✅ Format on Save
- ✅ Validation

### VS Code Settings

```json
// .vscode/settings.json
{
  "[terraform]": {
    "editor.defaultFormatter": "hashicorp.terraform",
    "editor.formatOnSave": true,
    "editor.formatOnSaveMode": "file"
  },
  "[terraform-vars]": {
    "editor.defaultFormatter": "hashicorp.terraform",
    "editor.formatOnSave": true
  },
  "terraform.experimentalFeatures.validateOnSave": true,
  "terraform.languageServer.enable": true
}
```

### .gitignore File

```gitignore
# .gitignore - Terraform Projects အတွက်

# Local .terraform directories
**/.terraform/*

# .tfstate files
*.tfstate
*.tfstate.*

# Crash log files
crash.log
crash.*.log

# Override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# CLI configuration files
.terraformrc
terraform.rc

# Lock file (Optional - Team ပေါ်မူတည်)
# .terraform.lock.hcl

# Sensitive variable files
*.tfvars
!example.tfvars

# IDE
.vscode/
.idea/
```

---

## 2.6 ပထမဆုံး `terraform init`, `plan`, `apply` လုပ်ကြည့်ခြင်း

### Project Structure ပြင်ဆင်ခြင်း

```bash
mkdir my-first-terraform
cd my-first-terraform
```

အောက်ပါ File Structure ကို ဖန်တီးပါ:

```
my-first-terraform/
├── main.tf          # Main configuration
├── providers.tf     # Provider configuration
├── variables.tf     # Input variables
├── outputs.tf       # Output values
└── terraform.tfvars # Variable values
```

### providers.tf

```hcl
# providers.tf
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
  region = var.aws_region
}
```

### variables.tf

```hcl
# variables.tf
variable "aws_region" {
  description = "AWS Region for resources"
  type        = string
  default     = "ap-southeast-1"
}

variable "project_name" {
  description = "Project name for tagging"
  type        = string
  default     = "my-first-terraform"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}
```

### terraform.tfvars

```hcl
# terraform.tfvars
aws_region   = "ap-southeast-1"
project_name = "my-first-terraform"
environment  = "dev"
```

### main.tf

```hcl
# main.tf

# S3 Bucket ဖန်တီးခြင်း
resource "aws_s3_bucket" "my_first_bucket" {
  bucket = "${var.project_name}-${var.environment}-bucket"

  tags = {
    Name        = "${var.project_name}-bucket"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# S3 Bucket Versioning ဖွင့်ခြင်း
resource "aws_s3_bucket_versioning" "my_first_bucket_versioning" {
  bucket = aws_s3_bucket.my_first_bucket.id

  versioning_configuration {
    status = "Enabled"
  }
}

# S3 Bucket Public Access ပိတ်ခြင်း
resource "aws_s3_bucket_public_access_block" "my_first_bucket_public_access" {
  bucket = aws_s3_bucket.my_first_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### outputs.tf

```hcl
# outputs.tf
output "bucket_name" {
  description = "S3 Bucket Name"
  value       = aws_s3_bucket.my_first_bucket.bucket
}

output "bucket_arn" {
  description = "S3 Bucket ARN"
  value       = aws_s3_bucket.my_first_bucket.arn
}

output "bucket_region" {
  description = "S3 Bucket Region"
  value       = aws_s3_bucket.my_first_bucket.region
}
```

### Step 1: terraform init

```bash
$ terraform init

Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.31.0...
- Installed hashicorp/aws v5.31.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the
provider selections it made above.

Terraform has been successfully initialized!
```

`terraform init` လုပ်ပြီးနောက် ဖိုင် Structure:

```
my-first-terraform/
├── .terraform/               # ← Init မှ ဖန်တီးထားသော Directory
│   └── providers/
│       └── registry.terraform.io/
│           └── hashicorp/
│               └── aws/
│                   └── 5.31.0/
│                       └── ...
├── .terraform.lock.hcl       # ← Provider Version Lock File
├── main.tf
├── providers.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```

### Step 2: terraform validate

```bash
$ terraform validate
Success! The configuration is valid.
```

### Step 3: terraform plan

```bash
$ terraform plan

Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_s3_bucket.my_first_bucket will be created
  + resource "aws_s3_bucket" "my_first_bucket" {
      + acceleration_status         = (known after apply)
      + acl                         = (known after apply)
      + arn                         = (known after apply)
      + bucket                      = "my-first-terraform-dev-bucket"
      + bucket_domain_name          = (known after apply)
      + bucket_regional_domain_name = (known after apply)
      + force_destroy               = false
      + hosted_zone_id              = (known after apply)
      + id                          = (known after apply)
      + object_lock_enabled         = (known after apply)
      + region                      = (known after apply)
      + request_payer               = (known after apply)
      + tags                        = {
          + "Environment" = "dev"
          + "ManagedBy"   = "terraform"
          + "Name"        = "my-first-terraform-bucket"
        }
      + tags_all                    = {
          + "Environment" = "dev"
          + "ManagedBy"   = "terraform"
          + "Name"        = "my-first-terraform-bucket"
        }
    }

  # aws_s3_bucket_public_access_block.my_first_bucket_public_access will be created
  + resource "aws_s3_bucket_public_access_block" "my_first_bucket_public_access" {
      + block_public_acls       = true
      + block_public_policy     = true
      + bucket                  = (known after apply)
      + id                      = (known after apply)
      + ignore_public_acls      = true
      + restrict_public_buckets = true
    }

  # aws_s3_bucket_versioning.my_first_bucket_versioning will be created
  + resource "aws_s3_bucket_versioning" "my_first_bucket_versioning" {
      + bucket = (known after apply)
      + id     = (known after apply)
      + versioning_configuration {
          + status = "Enabled"
        }
    }

Plan: 3 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + bucket_arn    = (known after apply)
  + bucket_name   = "my-first-terraform-dev-bucket"
  + bucket_region = (known after apply)
```

### Step 4: terraform apply

```bash
$ terraform apply

# ... Plan output ...

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_s3_bucket.my_first_bucket: Creating...
aws_s3_bucket.my_first_bucket: Creation complete after 3s [id=my-first-terraform-dev-bucket]
aws_s3_bucket_versioning.my_first_bucket_versioning: Creating...
aws_s3_bucket_public_access_block.my_first_bucket_public_access: Creating...
aws_s3_bucket_versioning.my_first_bucket_versioning: Creation complete after 2s
aws_s3_bucket_public_access_block.my_first_bucket_public_access: Creation complete after 1s

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

bucket_arn    = "arn:aws:s3:::my-first-terraform-dev-bucket"
bucket_name   = "my-first-terraform-dev-bucket"
bucket_region = "ap-southeast-1"
```

### Step 5: စစ်ဆေးခြင်း

```bash
# AWS CLI ဖြင့် Bucket စစ်ဆေးခြင်း
$ aws s3 ls | grep my-first-terraform
2024-01-15 10:30:00 my-first-terraform-dev-bucket

# Terraform Output ကြည့်ခြင်း
$ terraform output
bucket_arn    = "arn:aws:s3:::my-first-terraform-dev-bucket"
bucket_name   = "my-first-terraform-dev-bucket"
bucket_region = "ap-southeast-1"

# Terraform State ကြည့်ခြင်း
$ terraform state list
aws_s3_bucket.my_first_bucket
aws_s3_bucket_public_access_block.my_first_bucket_public_access
aws_s3_bucket_versioning.my_first_bucket_versioning
```

### Step 6: terraform destroy (ရှင်းလင်းခြင်း)

```bash
$ terraform destroy

# ... Plan output ...

Do you want to really destroy all resources?
  Terraform will destroy all your managed infrastructure.

  Enter a value: yes

aws_s3_bucket_public_access_block.my_first_bucket_public_access: Destroying...
aws_s3_bucket_versioning.my_first_bucket_versioning: Destroying...
aws_s3_bucket_public_access_block.my_first_bucket_public_access: Destruction complete after 1s
aws_s3_bucket_versioning.my_first_bucket_versioning: Destruction complete after 1s
aws_s3_bucket.my_first_bucket: Destroying...
aws_s3_bucket.my_first_bucket: Destruction complete after 1s

Destroy complete! Resources: 3 destroyed.
```

> **💡 Tips:** Learning အတွက် စမ်းသုံးပြီးတိုင်း `terraform destroy` လုပ်ပါ။ AWS Resources များသည် Running နေသမျှ ငွေကုန်ကျပါသည်။

---

## အခန်း (၂) အကျဉ်းချုပ်

ဤအခန်းတွင် အောက်ပါ အချက်များကို လေ့လာခဲ့ပါသည်:

- ✅ **Terraform** ကို Windows/macOS/Linux တွင် Install လုပ်ခြင်း
- ✅ **AWS CLI** ကို Install နှင့် Configure လုပ်ခြင်း
- ✅ **IAM User** ဖန်တီးပြီး Access Key ရယူခြင်း
- ✅ **AWS Provider** Configuration (Region, Profile, Multiple Regions)
- ✅ **VS Code** Extensions နှင့် Development Tools Setup
- ✅ ပထမဆုံး **S3 Bucket** ကို Terraform ဖြင့် ဖန်တီးခြင်း
- ✅ `terraform init` → `plan` → `apply` → `destroy` Workflow

> **📖 နောက်အခန်းတွင်** HCL Language ၏ အခြေခံများ ဖြစ်သော Variables, Outputs, Data Sources နှင့် Functions များကို အသေးစိတ် လေ့လာကြပါမည်။

---

*[← အခန်း (၁)](chapter-01.md) | [အခန်း (၃) - HCL အခြေခံ →](chapter-03.md)*
