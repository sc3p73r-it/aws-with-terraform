# အခန်း (၁) - Terraform ဆိုတာ ဘာလဲ?

---

## 1.1 Infrastructure as Code (IaC) ၏ အဓိပ္ပါယ်နှင့် အရေးပါပုံ

### Infrastructure as Code ဆိုတာ ဘာလဲ?

ရိုးရိုး ပြောရရင်တော့ **Infrastructure as Code (IaC)** ဆိုတာ Server များ၊ Network များ၊ Database များ စသည့် IT Infrastructure တွေကို **Code** ရေးပြီး တည်ဆောက်တဲ့ နည်းလမ်း ဖြစ်ပါတယ်။

အရင်တုန်းက Server တစ်လုံး တည်ဆောက်ချင်ရင် -
- AWS Console ကို Login ဝင်
- Mouse နဲ့ Click လုပ်
- Form တွေ ဖြည့်
- ခလုတ်တွေ နှိပ်

ဒီနည်းကို **Manual Process** (သို့) **ClickOps** လို့ ခေါ်ပါတယ်။

**ClickOps ရဲ့ ပြဿနာများ:**

| ပြဿနာ | ရှင်းလင်းချက် |
|---------|---------------|
| **လူသား အမှား** | Click မှားရင်၊ Value မှားရင် ပြဿနာ ဖြစ်နိုင် |
| **ပြန်လုပ်ရ ခက်ခြင်း** | Server ၁၀ လုံး အတူတူ လုပ်ချင်ရင် ၁၀ ခါ ထပ်ခါထပ်ခါ Click ရတယ် |
| **မှတ်တမ်း မရှိခြင်း** | ဘယ်သူ ဘာပြောင်းခဲ့လဲ Track လုပ်ဖို့ ခက်တယ် |
| **Consistency မရှိခြင်း** | Dev, Staging, Production Environment တွေ မတူညီနိုင် |
| **Speed နှေးခြင်း** | Infrastructure ကြီးလာလေ Manual Process ပိုနှေးလေ |

### IaC ဖြင့် ဖြေရှင်းခြင်း

IaC ကို အသုံးပြုခြင်းဖြင့် အထက်ပါ ပြဿနာအားလုံးကို ဖြေရှင်းနိုင်ပါတယ်:

```
# ClickOps - Manual
AWS Console → EC2 → Launch Instance → Configure → Review → Launch
(ခလုတ် ၂၀ ကျော် နှိပ်ရမယ်)

# IaC - Terraform
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  tags = {
    Name = "WebServer"
  }
}
# terraform apply ဆိုရင် ပြီးသွားပြီ!
```

### IaC ၏ အဓိက အားသာချက်များ

1. **Version Control** - Git ဖြင့် Infrastructure ပြောင်းလဲမှုတွေကို Track လုပ်နိုင်
2. **Reproducibility** - တူညီတဲ့ Environment ကို အကြိမ်ကြိမ် ဖန်တီးနိုင်
3. **Automation** - CI/CD Pipeline ထဲ ထည့်သွင်း၍ Auto Deploy လုပ်နိုင်
4. **Documentation** - Code ကိုယ်တိုင်က Documentation ဖြစ်
5. **Cost Reduction** - Manual Error လျော့ကျ၊ Speed မြန်ဆန်
6. **Collaboration** - Team Members များ အတူတကွ Infrastructure ကို ရေးသား/Review လုပ်နိုင်

---

## 1.2 Terraform ၏ သမိုင်းကြောင်းနှင့် HashiCorp အကြောင်း

### HashiCorp ကုမ္ပဏီ

**HashiCorp** ကို **Mitchell Hashimoto** နှင့် **Armon Dadgar** တို့က **2012** ခုနှစ်တွင် တည်ထောင်ခဲ့ပါတယ်။ DevOps နှင့် Cloud Infrastructure အတွက် Open Source Tools များ ဖန်တီးပေးသည့် ကုမ္ပဏီ ဖြစ်ပါတယ်။

**HashiCorp ရဲ့ Product များ:**

| Product | ရည်ရွယ်ချက် |
|---------|-------------|
| **Terraform** | Infrastructure Provisioning |
| **Vault** | Secrets Management |
| **Consul** | Service Discovery & Mesh |
| **Nomad** | Workload Orchestration |
| **Packer** | Machine Image Building |
| **Vagrant** | Development Environments |
| **Waypoint** | Application Deployment |
| **Boundary** | Secure Remote Access |

### Terraform ၏ သမိုင်းကြောင်း

- **2014** - Terraform v0.1 ကို Open Source အဖြစ် ထုတ်ဝေ
- **2017** - Terraform v0.10 - Provider/Provisioner separation
- **2019** - Terraform v0.12 - HCL2 (Major syntax improvements)
- **2020** - Terraform v0.13 - Module `for_each`, custom validation
- **2021** - Terraform v1.0 - Production-ready stable release
- **2023** - License change (BSL), OpenTofu fork ပေါ်ပေါက်
- **2024** - Terraform v1.7+ - Test framework, removed backends

> **🔑 သိထားရန်:** 2023 ခုနှစ်တွင် HashiCorp သည် Terraform ၏ License ကို MPL 2.0 မှ BSL (Business Source License) သို့ ပြောင်းလဲခဲ့ပါတယ်။ ထို့ကြောင့် Community မှ **OpenTofu** ဟူသော Open Source Fork ကို ဖန်တီးခဲ့ပါတယ်။ သို့သော် ဤစာအုပ်တွင် Terraform ကိုသာ အသုံးပြုသွားပါမည်။

---

## 1.3 Terraform vs CloudFormation vs Pulumi - နှိုင်းယှဉ်ချက်

IaC Tools အမျိုးမျိုးရှိပါတယ်။ ထင်ရှားတဲ့ ၃ ခုကို နှိုင်းယှဉ်ကြည့်ရအောင်:

### နှိုင်းယှဉ်ဇယား

| Feature | Terraform | CloudFormation | Pulumi |
|---------|-----------|----------------|--------|
| **Developer** | HashiCorp | AWS | Pulumi Inc. |
| **Language** | HCL | JSON/YAML | Python, JS, Go, etc. |
| **Multi-Cloud** | ✅ Yes | ❌ AWS Only | ✅ Yes |
| **State Management** | Remote/Local | AWS Managed | Pulumi Cloud/Local |
| **Learning Curve** | အလယ်အလတ် | မြင့် | နိမ့် (Devs အတွက်) |
| **Community** | အလွန်ကြီးမား | ကြီးမား | ကြီးထွားနေ |
| **Open Source** | BSL License | ❌ No | ✅ Yes |
| **Drift Detection** | Plan ဖြင့် | Drift Detection | Preview ဖြင့် |
| **Modularity** | Modules | Nested Stacks | Components |

### Terraform ကို ဘာကြောင့် ရွေးချယ်သင့်သလဲ?

**1. Multi-Cloud Support**
```hcl
# AWS Resource
provider "aws" {
  region = "ap-southeast-1"
}

# Azure Resource (တူညီသော Tool ဖြင့်)
provider "azurerm" {
  features {}
}

# GCP Resource
provider "google" {
  project = "my-project"
}
```
CloudFormation က AWS Only ဖြစ်ပေမယ့် Terraform ကတော့ AWS, Azure, GCP, Kubernetes, GitHub, Datadog စသဖြင့် Provider **3000+** ကို Support လုပ်ပါတယ်။

**2. HCL Language ရိုးရှင်းခြင်း**
```hcl
# Terraform (HCL) - ဖတ်ရ လွယ်ကူ
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

```json
// CloudFormation (JSON) - ရှုပ်ထွေး
{
  "Resources": {
    "Example": {
      "Type": "AWS::EC2::Instance",
      "Properties": {
        "ImageId": "ami-0c55b159cbfafe1f0",
        "InstanceType": "t2.micro"
      }
    }
  }
}
```

**3. Plan Feature**
```bash
$ terraform plan

# Plan ဆိုတာ Apply မလုပ်ခင် ဘာတွေ ပြောင်းလဲမယ်ဆိုတာ Preview ကြည့်ခြင်း
+ aws_instance.example will be created
  + ami           = "ami-0c55b159cbfafe1f0"
  + instance_type = "t2.micro"

Plan: 1 to add, 0 to change, 0 to destroy.
```

**4. ကြီးမားသော Community နှင့် Ecosystem**
- Terraform Registry တွင် Modules **15,000+** ရှိ
- Providers **3,000+** ရှိ
- GitHub Stars **40,000+**
- Stack Overflow Questions **50,000+**

---

## 1.4 Terraform ၏ အလုပ်လုပ်ပုံ (Architecture Overview)

### Terraform Core Architecture

Terraform ၏ Architecture ကို နားလည်ရန် အဓိက အစိတ်အပိုင်း ၃ ခု ရှိပါတယ်:

```
┌─────────────────────────────────────────────────┐
│                  Terraform Core                  │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐   │
│  │  Config    │  │   State   │  │    Plan   │   │
│  │  Parser    │  │  Manager  │  │  Builder  │   │
│  └───────────┘  └───────────┘  └───────────┘   │
└────────────────────┬────────────────────────────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │   AWS    │ │  Azure   │ │   GCP    │
   │ Provider │ │ Provider │ │ Provider │
   └──────────┘ └──────────┘ └──────────┘
         │           │           │
         ▼           ▼           ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │  AWS API │ │Azure API │ │ GCP API  │
   └──────────┘ └──────────┘ └──────────┘
```

### ၁. Terraform Core
- HCL Config Files ကို ဖတ်
- State File ကို စီမံ
- Execution Plan ကို တည်ဆောက်
- Provider များနှင့် ဆက်သွယ်

### ၂. Providers
- Cloud Platform တစ်ခုချင်းစီအတွက် Plugin
- API Calls များ ပြုလုပ်
- Resources များ CRUD Operations လုပ်ဆောင်

### ၃. State
- Infrastructure ၏ လက်ရှိ အခြေအနေ မှတ်တမ်း
- JSON Format ဖြင့် သိမ်းဆည်း
- Plan ပြုလုပ်ရာတွင် Desired State နှင့် Current State ကို နှိုင်းယှဉ်

### Terraform Workflow

Terraform ၏ အဓိက Workflow ကို **Write → Plan → Apply** ဟု မှတ်ပါ:

```
    ┌──────────┐     ┌──────────┐     ┌──────────┐
    │  Write   │────▶│   Plan   │────▶│  Apply   │
    │  (.tf)   │     │ (Preview)│     │ (Execute)│
    └──────────┘     └──────────┘     └──────────┘
         │                │                │
    Config ရေးသား    ပြောင်းလဲမှု      Infrastructure
    ပြီးရင်          Preview ကြည့်    ဖန်တီး/ပြင်ဆင်
```

**Step 1: Write** - `.tf` Files ရေးသား
```hcl
# main.tf
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-terraform-bucket-2024"
}
```

**Step 2: Init** - Provider Download & Initialize
```bash
$ terraform init
Initializing the backend...
Initializing provider plugins...
- Finding latest version of hashicorp/aws...
- Installing hashicorp/aws v5.31.0...
Terraform has been successfully initialized!
```

**Step 3: Plan** - ဘာတွေ ပြောင်းမယ်ဆိုတာ Preview ကြည့်
```bash
$ terraform plan
Terraform will perform the following actions:
  # aws_s3_bucket.my_bucket will be created
  + resource "aws_s3_bucket" "my_bucket" {
      + bucket = "my-terraform-bucket-2024"
      + id     = (known after apply)
      + arn    = (known after apply)
    }
Plan: 1 to add, 0 to change, 0 to destroy.
```

**Step 4: Apply** - Plan ကို အတည်ပြု၍ Infrastructure ဖန်တီး
```bash
$ terraform apply
Do you want to perform these actions?
  Enter a value: yes

aws_s3_bucket.my_bucket: Creating...
aws_s3_bucket.my_bucket: Creation complete after 2s [id=my-terraform-bucket-2024]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

**Step 5: Destroy** - Infrastructure ဖျက်သိမ်း (လိုအပ်ပါက)
```bash
$ terraform destroy
```

### State File ၏ အရေးပါပုံ

```json
// terraform.tfstate (ဥပမာ)
{
  "version": 4,
  "terraform_version": "1.7.0",
  "resources": [
    {
      "type": "aws_s3_bucket",
      "name": "my_bucket",
      "instances": [
        {
          "attributes": {
            "bucket": "my-terraform-bucket-2024",
            "arn": "arn:aws:s3:::my-terraform-bucket-2024"
          }
        }
      ]
    }
  ]
}
```

> **⚠️ သတိပြုရန်:** State File တွင် Sensitive Data (passwords, keys) ပါဝင်နိုင်သောကြောင့် Git တွင် Commit မလုပ်ပါနှင့်။ `.gitignore` တွင် `*.tfstate` ထည့်ထားပါ။

---

## 1.5 Declarative vs Imperative ချဉ်းကပ်နည်းများ

### Imperative (အဆင့်ဆင့် ညွှန်ကြားခြင်း)

Imperative ချဉ်းကပ်နည်းတွင် **"ဘယ်လို လုပ်ရမလဲ"** ဆိုတာကို အဆင့်ဆင့် ပြောပြရပါတယ်:

```bash
# Bash Script (Imperative)
#!/bin/bash

# Step 1: VPC ဖန်တီး
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)

# Step 2: Subnet ဖန်တီး
SUBNET_ID=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --output text)

# Step 3: Internet Gateway ဖန်တီး
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)

# Step 4: IGW ကို VPC နှင့် ချိတ်ဆက်
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# ပြဿနာ: Script ကို ၂ ကြိမ် Run ရင် Error ဖြစ်မယ်!
# ရှိပြီးသား Resource ကို ထပ်ဖန်တီးမိမယ်!
```

### Declarative (ရလဒ် ဖော်ပြခြင်း)

Declarative ချဉ်းကပ်နည်းတွင် **"ဘာ ရချင်လဲ"** ဆိုတာကိုသာ ဖော်ပြရပါတယ်:

```hcl
# Terraform (Declarative)
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id
}

# ၂ ကြိမ်၊ ၃ ကြိမ် Apply လုပ်လုပ် ပြဿနာ မရှိ!
# Terraform က ရှိပြီးသားဆိုရင် Skip သွားမယ်
```

### နှိုင်းယှဉ်ချက်

| Feature | Imperative | Declarative (Terraform) |
|---------|-----------|------------------------|
| **ဖော်ပြပုံ** | "ဘယ်လို လုပ်ရမလဲ" | "ဘာ ရချင်လဲ" |
| **Idempotent** | ❌ ကိုယ်တိုင် Handle ရ | ✅ Automatic |
| **Learning Curve** | နိမ့် | အလယ်အလတ် |
| **Error Handling** | ကိုယ်တိုင် ရေးရ | Built-in |
| **State Tracking** | ❌ မရှိ | ✅ State File |
| **ဥပမာ Tools** | Bash, Ansible | Terraform, CloudFormation |

### Idempotency ဆိုတာ ဘာလဲ?

**Idempotent** ဆိုတာ Operation တစ်ခုကို **အကြိမ်ကြိမ် လုပ်ဆောင်ပေမယ့် ရလဒ် တူညီခြင်း** ဖြစ်ပါတယ်:

```bash
# Apply ပထမအကြိမ် - Resource ဖန်တီးမယ်
$ terraform apply
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

# Apply ဒုတိယအကြိမ် - ပြောင်းလဲမှု မရှိတော့ဘူး
$ terraform apply
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

# ရလဒ် - Infrastructure အခြေအနေ တူညီ ✅
```

---

## အခန်း (၁) အကျဉ်းချုပ်

ဤအခန်းတွင် အောက်ပါ အချက်များကို လေ့လာခဲ့ပါသည်:

- ✅ **Infrastructure as Code** ဆိုတာ Infrastructure ကို Code ဖြင့် စီမံခန့်ခွဲခြင်း ဖြစ်ပါတယ်
- ✅ **Terraform** ကို HashiCorp က 2014 ခုနှစ်တွင် ဖန်တီးခဲ့ပြီး Multi-Cloud Support ရှိပါတယ်
- ✅ CloudFormation ထက် **Multi-Cloud** နှင့် **HCL** ကြောင့် Terraform က ပိုအားသာပါတယ်
- ✅ Terraform သည် **Write → Plan → Apply** Workflow ဖြင့် အလုပ်လုပ်ပါတယ်
- ✅ **Declarative** ချဉ်းကပ်နည်းသည် Imperative ထက် **Idempotent** ဖြစ်၍ ပိုလုံခြုံပါတယ်

> **📖 နောက်အခန်းတွင်** Terraform နှင့် AWS ပတ်ဝန်းကျင်ကို ကွန်ပျူတာတွင် Setup လုပ်ပြီး ပထမဆုံး Infrastructure ကို ဖန်တီးကြပါမည်။

---

*[အခန်း (၂) - Terraform နှင့် AWS ပတ်ဝန်းကျင် ပြင်ဆင်ခြင်း →](chapter-02.md)*
