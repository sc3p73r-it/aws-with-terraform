# အခန်း (၃) - HCL (HashiCorp Configuration Language) အခြေခံ

---

## 3.1 HCL Syntax အခြေခံ

HCL (HashiCorp Configuration Language) သည် Terraform ရဲ့ Configuration Language ဖြစ်ပါတယ်။ JSON ကို လူသား ဖတ်ရ ပိုလွယ်အောင် ဒီဇိုင်းလုပ်ထားတဲ့ Language ဖြစ်ပါတယ်။

### HCL ၏ အခြေခံ ဖွဲ့စည်းပုံ

```hcl
# Comment တစ်ကြောင်းတည်း
// Comment တစ်ကြောင်းတည်း (ဒါလည်း ရတယ်)

/*
Comment
အကြောင်းများစွာ
*/

# Block Syntax
block_type "label_1" "label_2" {
  argument_name = "argument_value"

  nested_block {
    nested_argument = "value"
  }
}
```

### Terraform File Types

| File Extension | ရည်ရွယ်ချက် |
|---------------|-------------|
| `.tf` | Terraform Configuration Files |
| `.tf.json` | JSON Format Configuration |
| `.tfvars` | Variable Values Files |
| `.tfstate` | State Files |
| `.terraform.lock.hcl` | Provider Lock File |

### File Organization Best Practice

```
project/
├── main.tf          # အဓိက Resources
├── providers.tf     # Provider Configuration
├── variables.tf     # Variable Declarations
├── outputs.tf       # Output Declarations
├── terraform.tfvars # Variable Values
├── locals.tf        # Local Values
├── data.tf          # Data Sources
└── versions.tf      # Version Constraints
```

> **💡 Tip:** Terraform သည် Directory တစ်ခုအတွင်းရှိ `.tf` File အားလုံးကို ပေါင်း၍ ဖတ်ပါသည်။ File Name ကို ဘာပဲ ပေးပေး အလုပ်လုပ်ပါတယ်။ သို့သော် Convention အတိုင်း ခွဲရေးခြင်းက Code Organization အတွက် အကောင်းဆုံး ဖြစ်ပါတယ်။

---

## 3.2 Blocks, Arguments, Expressions

### Block Types

Terraform တွင် Block Types အဓိက ၅ မျိုး ရှိပါတယ်:

#### 1. Resource Block

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "WebServer"
  }
}
# resource  = Block Type
# "aws_instance" = Resource Type (provider_resourcetype)
# "web_server"   = Resource Name (local identifier)
```

#### 2. Data Block

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  owners = ["099720109477"]  # Canonical
}
# data = Block Type
# "aws_ami" = Data Source Type
# "ubuntu"  = Data Source Name
```

#### 3. Variable Block

```hcl
variable "instance_type" {
  description = "EC2 Instance Type"
  type        = string
  default     = "t2.micro"
}
```

#### 4. Output Block

```hcl
output "instance_ip" {
  description = "Public IP of the instance"
  value       = aws_instance.web_server.public_ip
}
```

#### 5. Locals Block

```hcl
locals {
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

### Arguments vs Blocks

```hcl
resource "aws_security_group" "web_sg" {
  # Arguments (key = value)
  name        = "web-sg"               # String Argument
  description = "Web Security Group"    # String Argument
  vpc_id      = aws_vpc.main.id        # Reference Argument

  # Nested Block (Sub-block)
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]       # List Argument
  }

  # Nested Block (ထပ်ထည့်နိုင်တယ်)
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Map Argument
  tags = {
    Name = "web-sg"
  }
}
```

### Expressions

```hcl
# String Interpolation
name = "server-${var.environment}"

# Arithmetic
count = var.instance_count * 2

# Conditional (Ternary)
instance_type = var.environment == "prod" ? "t3.large" : "t2.micro"

# Function Call
upper_name = upper(var.project_name)

# Reference
subnet_id = aws_subnet.public.id
```

---

## 3.3 Variables (Input Variables) အမျိုးမျိုး

### Variable Types

Terraform တွင် Variable Type **၆ မျိုး** ရှိပါတယ်:

#### 1. String

```hcl
variable "instance_name" {
  description = "Name of the EC2 instance"
  type        = string
  default     = "my-server"
}

# အသုံးပြုပုံ
resource "aws_instance" "example" {
  tags = {
    Name = var.instance_name
  }
}
```

#### 2. Number

```hcl
variable "instance_count" {
  description = "Number of instances to create"
  type        = number
  default     = 2
}

# အသုံးပြုပုံ
resource "aws_instance" "example" {
  count         = var.instance_count
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

#### 3. Bool

```hcl
variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = false
}

# အသုံးပြုပုံ
resource "aws_instance" "example" {
  monitoring = var.enable_monitoring
}
```

#### 4. List

```hcl
variable "availability_zones" {
  description = "List of AZs"
  type        = list(string)
  default     = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
}

# အသုံးပြုပုံ
resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  availability_zone = var.availability_zones[count.index]
  cidr_block        = "10.0.${count.index + 1}.0/24"
  vpc_id            = aws_vpc.main.id
}

# List ဥပမာများ
variable "ports" {
  type    = list(number)
  default = [80, 443, 8080]
}

variable "mixed_list" {
  type    = list(any)
  default = ["hello", 42, true]
}
```

#### 5. Map

```hcl
variable "instance_types" {
  description = "Instance types per environment"
  type        = map(string)
  default = {
    dev     = "t2.micro"
    staging = "t2.small"
    prod    = "t3.large"
  }
}

# အသုံးပြုပုံ
resource "aws_instance" "example" {
  instance_type = var.instance_types[var.environment]
  # var.environment = "dev" ဆိုရင် "t2.micro" ရမယ်
}

# Map of Numbers
variable "port_mapping" {
  type = map(number)
  default = {
    http  = 80
    https = 443
    ssh   = 22
  }
}
```

#### 6. Object

```hcl
variable "server_config" {
  description = "Server configuration"
  type = object({
    instance_type = string
    ami_id        = string
    disk_size     = number
    monitoring    = bool
    tags          = map(string)
  })
  default = {
    instance_type = "t2.micro"
    ami_id        = "ami-0c55b159cbfafe1f0"
    disk_size     = 20
    monitoring    = true
    tags = {
      Environment = "dev"
    }
  }
}

# အသုံးပြုပုံ
resource "aws_instance" "example" {
  ami           = var.server_config.ami_id
  instance_type = var.server_config.instance_type
  monitoring    = var.server_config.monitoring

  root_block_device {
    volume_size = var.server_config.disk_size
  }

  tags = var.server_config.tags
}
```

### Variable ကို Value ပေးနည်းများ

```bash
# နည်းလမ်း (၁) - terraform.tfvars File
# terraform.tfvars
instance_type = "t2.small"
environment   = "staging"

# နည်းလမ်း (၂) - Custom .tfvars File
# prod.tfvars
instance_type = "t3.large"
environment   = "prod"
$ terraform plan -var-file="prod.tfvars"

# နည်းလမ်း (၃) - Command Line
$ terraform plan -var="instance_type=t2.small" -var="environment=staging"

# နည်းလမ်း (၄) - Environment Variable
$ export TF_VAR_instance_type="t2.small"
$ export TF_VAR_environment="staging"
$ terraform plan

# နည်းလမ်း (၅) - Default Value (variable block ထဲမှာ)
variable "instance_type" {
  default = "t2.micro"
}
```

**Variable Value Precedence (ဦးစားပေး အစဉ်):**

```
1. -var command line flag           (အမြင့်ဆုံး)
2. -var-file command line flag
3. *.auto.tfvars files
4. terraform.tfvars file
5. TF_VAR_ environment variables
6. Default value in variable block  (အနိမ့်ဆုံး)
```

### Variable Validation

```hcl
variable "environment" {
  description = "Environment name"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be 'dev', 'staging', or 'prod'."
  }
}

variable "instance_type" {
  description = "EC2 Instance Type"
  type        = string

  validation {
    condition     = can(regex("^t[23]\\.", var.instance_type))
    error_message = "Instance type must start with 't2.' or 't3.'."
  }
}

variable "cidr_block" {
  description = "VPC CIDR Block"
  type        = string

  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block (e.g., 10.0.0.0/16)."
  }
}
```

### Sensitive Variables

```hcl
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true  # Plan/Apply Output တွင် ပြမည်မဟုတ်
}

# Output ကိုလည်း Sensitive မှတ်နိုင်
output "db_password" {
  value     = var.db_password
  sensitive = true
}
```

```bash
$ terraform plan
# db_password = (sensitive value)  ← Value မပြဘူး
```

---

## 3.4 Output Values

### Output ၏ ရည်ရွယ်ချက်

Output Values ကို အောက်ပါ အကြောင်းများကြောင့် သုံးပါတယ်:

1. **Apply ပြီးနောက် Information ပြခြင်း** (IP Address, URL, etc.)
2. **Module များအကြား Data Sharing**
3. **Remote State** မှ တခြား Terraform Config က ဖတ်ခြင်း

### Output ဥပမာများ

```hcl
# အခြေခံ Output
output "instance_id" {
  description = "EC2 Instance ID"
  value       = aws_instance.web_server.id
}

# Multiple Values Output
output "instance_public_ips" {
  description = "Public IPs of all instances"
  value       = aws_instance.web_server[*].public_ip
}

# Conditional Output
output "load_balancer_dns" {
  description = "ALB DNS name (only in prod)"
  value       = var.environment == "prod" ? aws_lb.main[0].dns_name : "N/A"
}

# Formatted Output
output "connection_string" {
  description = "Database connection string"
  value       = "postgresql://${var.db_username}:${var.db_password}@${aws_db_instance.main.endpoint}/${var.db_name}"
  sensitive   = true
}

# Map Output
output "instance_details" {
  description = "Instance details map"
  value = {
    id         = aws_instance.web_server.id
    public_ip  = aws_instance.web_server.public_ip
    private_ip = aws_instance.web_server.private_ip
    az         = aws_instance.web_server.availability_zone
  }
}
```

### Output Commands

```bash
# Output အားလုံး ကြည့်ခြင်း
$ terraform output

# Output တစ်ခုတည်း ကြည့်ခြင်း
$ terraform output instance_id

# JSON Format ဖြင့်
$ terraform output -json

# Raw Value (quotes မပါ)
$ terraform output -raw instance_id
```

---

## 3.5 Local Values

### Local Values ဆိုတာ ဘာလဲ?

Local Values (Locals) ကို **Variable** လိုမျိုး Value သိမ်းဖို့ သုံးပါတယ်။ ကွာခြားချက်က Locals ကို **အပြင်ကနေ override မလုပ်နိုင်ပါ**။ Module/Config အတွင်းမှာ ထပ်ခါထပ်ခါ သုံးမယ့် Value များကို Locals ဖြင့် Define လုပ်ပါ။

```hcl
# locals.tf
locals {
  # Common Tags - Resource အားလုံးမှာ ထပ်ခါထပ်ခါ မရေးချင်ရင်
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
    Owner       = "devops-team"
    CostCenter  = "engineering"
  }

  # Computed Values
  name_prefix = "${var.project_name}-${var.environment}"

  # Conditional Values
  is_production = var.environment == "prod"
  instance_type = local.is_production ? "t3.large" : "t2.micro"

  # Merged Maps
  all_tags = merge(local.common_tags, {
    CreatedAt = timestamp()
  })
}

# အသုံးပြုပုံ
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.instance_type

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web"
    Role = "web-server"
  })
}

resource "aws_s3_bucket" "assets" {
  bucket = "${local.name_prefix}-assets"

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-assets"
  })
}
```

### Variables vs Locals

| Feature | Variable | Local |
|---------|----------|-------|
| **အပြင်မှ Override** | ✅ Yes | ❌ No |
| **tfvars ဖြင့် Set** | ✅ Yes | ❌ No |
| **Computed Values** | ❌ No | ✅ Yes |
| **ရည်ရွယ်ချက်** | User Input | Internal Computation |

---

## 3.6 Data Sources

### Data Source ဆိုတာ ဘာလဲ?

Data Sources ကို **ရှိပြီးသား** AWS Resources (သို့) External Data ကို Terraform Config ထဲမှာ **ဖတ်ဖို့ (Read-Only)** သုံးပါတယ်။ Resource Block ကတော့ ဖန်တီးခြင်း/ပြင်ဆင်ခြင်း ဖြစ်ပြီး Data Block ကတော့ ဖတ်ခြင်း သာ ဖြစ်ပါတယ်။

### Data Source ဥပမာများ

#### 1. Latest AMI ရှာခြင်း

```hcl
# Amazon Linux 2023 AMI အသစ်ဆုံးကို ရှာခြင်း
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Ubuntu 22.04 AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

# အသုံးပြုပုံ
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"
}
```

#### 2. Availability Zones ရယူခြင်း

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

# အသုံးပြုပုံ
resource "aws_subnet" "public" {
  count             = length(data.aws_availability_zones.available.names)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```

#### 3. Caller Identity (Current Account Info)

```hcl
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

output "current_region" {
  value = data.aws_region.current.name
}
```

#### 4. Existing VPC ကို Reference လုပ်ခြင်း

```hcl
# ရှိပြီးသား VPC ကို ID ဖြင့် ရှာခြင်း
data "aws_vpc" "existing" {
  id = "vpc-0a1b2c3d4e5f6g7h8"
}

# Tag ဖြင့် ရှာခြင်း
data "aws_vpc" "by_tag" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

# Default VPC
data "aws_vpc" "default" {
  default = true
}
```

#### 5. IAM Policy Document

```hcl
data "aws_iam_policy_document" "s3_read_only" {
  statement {
    sid    = "S3ReadOnly"
    effect = "Allow"

    actions = [
      "s3:GetObject",
      "s3:ListBucket",
    ]

    resources = [
      aws_s3_bucket.my_bucket.arn,
      "${aws_s3_bucket.my_bucket.arn}/*",
    ]
  }
}

resource "aws_iam_policy" "s3_read" {
  name   = "s3-read-only"
  policy = data.aws_iam_policy_document.s3_read_only.json
}
```

---

## 3.7 Resource References နှင့် Dependencies

### Implicit Dependencies (အလိုအလျောက်)

Terraform သည် Resource References ကိုကြည့်ပြီး Dependencies ကို **အလိုအလျောက်** သိပါတယ်:

```hcl
# VPC ကို အရင် ဖန်တီးရမယ်
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# Subnet က VPC ကို Reference လုပ်ထားတယ် → VPC ပြီးမှ Subnet ဖန်တီးမယ်
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id    # ← Implicit Dependency
  cidr_block = "10.0.1.0/24"
}

# Security Group ကလည်း VPC ကို Reference လုပ်ထားတယ်
resource "aws_security_group" "web" {
  vpc_id = aws_vpc.main.id       # ← Implicit Dependency
  name   = "web-sg"
}

# EC2 Instance က Subnet နှင့် Security Group ကို Reference လုပ်ထားတယ်
resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.public.id         # ← Implicit
  vpc_security_group_ids = [aws_security_group.web.id]  # ← Implicit
}
```

### Resource Reference Syntax

```hcl
# Resource Reference
<resource_type>.<resource_name>.<attribute>

# ဥပမာများ
aws_vpc.main.id                    # VPC ID
aws_subnet.public.id               # Subnet ID
aws_instance.web.public_ip         # EC2 Public IP
aws_s3_bucket.my_bucket.arn        # S3 Bucket ARN
aws_db_instance.main.endpoint      # RDS Endpoint

# Data Source Reference
data.<data_source_type>.<name>.<attribute>

# ဥပမာ
data.aws_ami.ubuntu.id
data.aws_availability_zones.available.names
data.aws_caller_identity.current.account_id
```

### Explicit Dependencies (depends_on)

တခါတရံ Terraform က Dependency ကို မသိတဲ့ အချိန် ရှိပါတယ်။ ထိုအခါ `depends_on` ကို သုံးပါ:

```hcl
resource "aws_iam_role" "ec2_role" {
  name = "ec2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ec2_s3" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"

  # IAM Role ကို Attach ပြီးမှ Instance ဖန်တီးရမယ်
  depends_on = [aws_iam_role_policy_attachment.ec2_s3]
}
```

---

## 3.8 Built-in Functions များ

Terraform တွင် Built-in Functions **100+** ရှိပါတယ်။ အသုံးများသော Functions များကို ဖော်ပြပါမည်:

### String Functions

```hcl
locals {
  # upper - စာလုံးအကြီး ပြောင်း
  upper_name = upper("hello")  # "HELLO"

  # lower - စာလုံးအသေး ပြောင်း
  lower_name = lower("HELLO")  # "hello"

  # title - စာလုံး ပထမလုံးကြီး
  title_name = title("hello world")  # "Hello World"

  # format - String Format
  formatted = format("Hello, %s! You have %d items.", "User", 5)
  # "Hello, User! You have 5 items."

  # join - List ကို String ပေါင်း
  joined = join(", ", ["a", "b", "c"])  # "a, b, c"
  joined_dash = join("-", ["web", "server", "01"])  # "web-server-01"

  # split - String ကို List ခွဲ
  parts = split(",", "a,b,c")  # ["a", "b", "c"]

  # replace - String Replace
  replaced = replace("hello-world", "-", "_")  # "hello_world"

  # substr - Substring
  sub = substr("hello world", 0, 5)  # "hello"

  # trimspace - Space ဖယ်
  trimmed = trimspace("  hello  ")  # "hello"

  # regex - Regular Expression
  matched = regex("[a-z]+", "123abc456")  # "abc"
}
```

### Collection Functions

```hcl
locals {
  # length - List/Map Size
  list_len = length(["a", "b", "c"])        # 3
  map_len  = length({ a = 1, b = 2 })       # 2

  # concat - List ပေါင်းစပ်
  combined = concat(["a", "b"], ["c", "d"])  # ["a", "b", "c", "d"]

  # merge - Map ပေါင်းစပ်
  merged = merge(
    { a = 1, b = 2 },
    { c = 3, d = 4 }
  )  # { a = 1, b = 2, c = 3, d = 4 }

  # lookup - Map Lookup with Default
  value = lookup(
    { dev = "t2.micro", prod = "t3.large" },
    "staging",
    "t2.small"  # default value
  )  # "t2.small"

  # element - List Element by Index
  first = element(["a", "b", "c"], 0)  # "a"

  # flatten - Nested List ကို Flat လုပ်
  flat = flatten([["a", "b"], ["c", "d"]])  # ["a", "b", "c", "d"]

  # keys / values - Map ရဲ့ Keys/Values
  my_map  = { a = 1, b = 2, c = 3 }
  k       = keys(local.my_map)    # ["a", "b", "c"]
  v       = values(local.my_map)  # [1, 2, 3]

  # contains - List ထဲမှာ ရှိ/မရှိ
  has_a = contains(["a", "b", "c"], "a")  # true

  # distinct - Duplicates ဖယ်
  unique = distinct(["a", "b", "a", "c"])  # ["a", "b", "c"]

  # sort - အစဉ်လိုက် စီ
  sorted = sort(["c", "a", "b"])  # ["a", "b", "c"]

  # zipmap - Keys List နှင့် Values List ကို Map ပြောင်း
  zipped = zipmap(
    ["name", "age"],
    ["Alice", "30"]
  )  # { name = "Alice", age = "30" }
}
```

### Numeric Functions

```hcl
locals {
  # min / max
  minimum = min(1, 5, 3)     # 1
  maximum = max(1, 5, 3)     # 5

  # ceil / floor
  ceiling = ceil(4.3)         # 5
  floored = floor(4.7)        # 4

  # abs
  absolute = abs(-5)          # 5

  # pow
  power = pow(2, 3)           # 8

  # signum
  sign = signum(-5)           # -1
}
```

### IP Network Functions

```hcl
locals {
  # cidrsubnet - CIDR ခွဲခြားခြင်း
  subnet_1 = cidrsubnet("10.0.0.0/16", 8, 0)   # "10.0.0.0/24"
  subnet_2 = cidrsubnet("10.0.0.0/16", 8, 1)   # "10.0.1.0/24"
  subnet_3 = cidrsubnet("10.0.0.0/16", 8, 2)   # "10.0.2.0/24"

  # cidrhost - CIDR ထဲက Host IP
  host_ip = cidrhost("10.0.1.0/24", 5)          # "10.0.1.5"

  # cidrnetmask
  netmask = cidrnetmask("10.0.0.0/16")          # "255.255.0.0"
}
```

### Type Conversion Functions

```hcl
locals {
  # tostring, tonumber, tobool
  str_val  = tostring(42)       # "42"
  num_val  = tonumber("42")     # 42
  bool_val = tobool("true")     # true

  # tolist, toset, tomap
  my_list = tolist(["a", "b", "c"])
  my_set  = toset(["a", "b", "a"])   # ["a", "b"] (unique)
  my_map  = tomap({ a = 1, b = 2 })

  # jsonencode / jsondecode
  json_str = jsonencode({
    name = "server"
    port = 8080
  })
  # '{"name":"server","port":8080}'

  json_obj = jsondecode("{\"name\":\"server\"}")
  # { name = "server" }
}
```

### terraform console ဖြင့် Function စမ်းသုံးခြင်း

```bash
$ terraform console

> upper("hello")
"HELLO"

> length([1, 2, 3])
3

> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"

> merge({a=1}, {b=2})
{
  "a" = 1
  "b" = 2
}

> format("Hello, %s!", "Terraform")
"Hello, Terraform!"

> exit
```

---

## အခန်း (၃) အကျဉ်းချုပ်

ဤအခန်းတွင် အောက်ပါ အချက်များကို လေ့လာခဲ့ပါသည်:

- ✅ **HCL Syntax** - Blocks, Arguments, Expressions
- ✅ **Variable Types** - string, number, bool, list, map, object
- ✅ **Variable Validation** နှင့် **Sensitive Variables**
- ✅ **Output Values** - Apply ပြီးနောက် Data ပြခြင်း
- ✅ **Local Values** - Internal Computation အတွက်
- ✅ **Data Sources** - ရှိပြီးသား Resources ဖတ်ခြင်း
- ✅ **Dependencies** - Implicit vs Explicit (depends_on)
- ✅ **Built-in Functions** - String, Collection, Numeric, IP, Type Conversion

> **📖 နောက်အခန်းတွင်** AWS VPC Networking ကို Terraform ဖြင့် အသေးစိတ် တည်ဆောက်ကြပါမည်။ Public/Private Subnets, Internet Gateway, NAT Gateway, Route Tables, Security Groups များ ပါဝင်ပါမည်။

---

*[← အခန်း (၂)](chapter-02.md) | [အခန်း (၄) - VPC Networking →](chapter-04.md)*
