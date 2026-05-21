# အခန်း (၇) - IAM (Identity and Access Management)

---

## 7.1 IAM Users, Groups, Roles

### IAM ဆိုတာ ဘာလဲ?

**IAM (Identity and Access Management)** ဆိုတာ AWS Resources များကို **ဘယ်သူ (Identity)** က **ဘာလုပ်ခွင့်ရှိ (Access)** ဆိုတာ ထိန်းချုပ်ပေးတဲ့ Service ဖြစ်ပါတယ်။

```
┌──────────────────── IAM Overview ────────────────────┐
│                                                       │
│  Users ─────┐                                         │
│              ├──── Groups ──── Policies ──── AWS       │
│  Users ─────┘                              Resources  │
│                                                       │
│  Applications ──── Roles ──── Policies ──── AWS       │
│  Services          │                       Resources  │
│                    └── Temporary Credentials           │
└───────────────────────────────────────────────────────┘
```

### IAM Users

```hcl
# ===================================
# IAM Users
# ===================================
resource "aws_iam_user" "developers" {
  for_each = toset(["alice", "bob", "charlie"])

  name = each.key
  path = "/developers/"

  tags = merge(local.common_tags, {
    Team = "development"
  })
}

# Console Access (Password)
resource "aws_iam_user_login_profile" "developers" {
  for_each = aws_iam_user.developers

  user                    = each.value.name
  password_reset_required = true
}

# Programmatic Access (Access Keys)
resource "aws_iam_access_key" "developers" {
  for_each = aws_iam_user.developers

  user = each.value.name
}
```

### IAM Groups

```hcl
# ===================================
# IAM Groups
# ===================================

# Developer Group
resource "aws_iam_group" "developers" {
  name = "developers"
  path = "/teams/"
}

# Admin Group
resource "aws_iam_group" "admins" {
  name = "admins"
  path = "/teams/"
}

# DevOps Group
resource "aws_iam_group" "devops" {
  name = "devops"
  path = "/teams/"
}

# Group Membership
resource "aws_iam_group_membership" "developers" {
  name  = "developers-membership"
  group = aws_iam_group.developers.name
  users = [for user in aws_iam_user.developers : user.name]
}
```

### IAM Roles

```hcl
# ===================================
# IAM Roles
# ===================================

# EC2 Role
resource "aws_iam_role" "ec2_app_role" {
  name = "${var.project_name}-${var.environment}-ec2-app-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "EC2AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })

  max_session_duration = 3600  # 1 hour

  tags = local.common_tags
}

# Lambda Role
resource "aws_iam_role" "lambda_role" {
  name = "${var.project_name}-${var.environment}-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "LambdaAssumeRole"
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })

  tags = local.common_tags
}

# Cross-Account Role
resource "aws_iam_role" "cross_account" {
  name = "${var.project_name}-cross-account-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "CrossAccountAssumeRole"
      Effect    = "Allow"
      Principal = {
        AWS = "arn:aws:iam::123456789012:root"  # Other Account ID
      }
      Action    = "sts:AssumeRole"
      Condition = {
        StringEquals = {
          "sts:ExternalId" = "unique-external-id"
        }
      }
    }]
  })

  tags = local.common_tags
}
```

---

## 7.2 IAM Policies (JSON Policy Document)

### Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "StatementIdentifier",
      "Effect": "Allow | Deny",
      "Action": "service:Action",
      "Resource": "arn:aws:...",
      "Condition": { }
    }
  ]
}
```

### Custom Policies with data source

```hcl
# ===================================
# Custom IAM Policies
# ===================================

# S3 Access Policy
data "aws_iam_policy_document" "s3_access" {
  statement {
    sid    = "S3BucketAccess"
    effect = "Allow"

    actions = [
      "s3:ListBucket",
      "s3:GetBucketLocation"
    ]

    resources = [
      aws_s3_bucket.app_data.arn
    ]
  }

  statement {
    sid    = "S3ObjectAccess"
    effect = "Allow"

    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject"
    ]

    resources = [
      "${aws_s3_bucket.app_data.arn}/*"
    ]
  }

  statement {
    sid    = "DenyDeleteBucket"
    effect = "Deny"

    actions = [
      "s3:DeleteBucket"
    ]

    resources = ["*"]
  }
}

resource "aws_iam_policy" "s3_access" {
  name        = "${var.project_name}-${var.environment}-s3-access"
  description = "S3 access policy for application"
  policy      = data.aws_iam_policy_document.s3_access.json

  tags = local.common_tags
}

# EC2 Limited Access Policy
data "aws_iam_policy_document" "ec2_limited" {
  statement {
    sid    = "EC2ReadOnly"
    effect = "Allow"

    actions = [
      "ec2:Describe*",
      "ec2:Get*",
      "ec2:List*"
    ]

    resources = ["*"]
  }

  statement {
    sid    = "EC2ManageOwnInstances"
    effect = "Allow"

    actions = [
      "ec2:StartInstances",
      "ec2:StopInstances",
      "ec2:RebootInstances"
    ]

    resources = ["*"]

    condition {
      test     = "StringEquals"
      variable = "ec2:ResourceTag/Environment"
      values   = [var.environment]
    }
  }
}

resource "aws_iam_policy" "ec2_limited" {
  name   = "${var.project_name}-${var.environment}-ec2-limited"
  policy = data.aws_iam_policy_document.ec2_limited.json
}
```

---

## 7.3 Policy Attachments

```hcl
# ===================================
# Policy Attachments
# ===================================

# Managed Policy → Role
resource "aws_iam_role_policy_attachment" "ec2_s3" {
  role       = aws_iam_role.ec2_app_role.name
  policy_arn = aws_iam_policy.s3_access.arn
}

resource "aws_iam_role_policy_attachment" "ec2_cloudwatch" {
  role       = aws_iam_role.ec2_app_role.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

resource "aws_iam_role_policy_attachment" "ec2_ssm" {
  role       = aws_iam_role.ec2_app_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# Managed Policy → Group
resource "aws_iam_group_policy_attachment" "developers_ec2" {
  group      = aws_iam_group.developers.name
  policy_arn = aws_iam_policy.ec2_limited.arn
}

resource "aws_iam_group_policy_attachment" "developers_s3" {
  group      = aws_iam_group.developers.name
  policy_arn = aws_iam_policy.s3_access.arn
}

# AWS Managed Policy → Group
resource "aws_iam_group_policy_attachment" "admins_admin" {
  group      = aws_iam_group.admins.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}

# Inline Policy → Role
resource "aws_iam_role_policy" "lambda_logs" {
  name = "lambda-cloudwatch-logs"
  role = aws_iam_role.lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ]
      Resource = "arn:aws:logs:*:*:*"
    }]
  })
}
```

---

## 7.4 Instance Profiles

```hcl
# ===================================
# Instance Profile (EC2 → IAM Role ချိတ်ဆက်)
# ===================================
resource "aws_iam_instance_profile" "ec2_app" {
  name = "${var.project_name}-${var.environment}-ec2-profile"
  role = aws_iam_role.ec2_app_role.name

  tags = local.common_tags
}

# EC2 Instance တွင် အသုံးပြုခြင်း
resource "aws_instance" "app" {
  ami                  = data.aws_ami.amazon_linux_2023.id
  instance_type        = "t2.micro"
  iam_instance_profile = aws_iam_instance_profile.ec2_app.name

  tags = {
    Name = "${var.project_name}-app"
  }
}
```

---

## 7.5 Cross-Account Access

```hcl
# ===================================
# Account A: Role ဖန်တီးခြင်း
# ===================================
resource "aws_iam_role" "cross_account_s3" {
  name = "cross-account-s3-access"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = {
        AWS = "arn:aws:iam::${var.trusted_account_id}:root"
      }
      Action    = "sts:AssumeRole"
      Condition = {
        StringEquals = {
          "sts:ExternalId" = var.external_id
        }
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "cross_account_s3" {
  role       = aws_iam_role.cross_account_s3.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}

# ===================================
# Account B: Assume Role Policy
# ===================================
data "aws_iam_policy_document" "assume_cross_account" {
  statement {
    effect = "Allow"
    actions = ["sts:AssumeRole"]

    resources = [
      "arn:aws:iam::${var.target_account_id}:role/cross-account-s3-access"
    ]
  }
}
```

---

## 7.6 Service-Linked Roles

```hcl
# AWS Service-Linked Roles (Auto Scaling, ELB, etc.)
resource "aws_iam_service_linked_role" "autoscaling" {
  aws_service_name = "autoscaling.amazonaws.com"
  description      = "Service-linked role for Auto Scaling"
}

resource "aws_iam_service_linked_role" "elasticloadbalancing" {
  aws_service_name = "elasticloadbalancing.amazonaws.com"
  description      = "Service-linked role for ELB"
}
```

---

## 7.7 IAM Best Practices

### Password Policy

```hcl
resource "aws_iam_account_password_policy" "strict" {
  minimum_password_length        = 14
  require_lowercase_characters   = true
  require_uppercase_characters   = true
  require_numbers                = true
  require_symbols                = true
  allow_users_to_change_password = true
  max_password_age               = 90
  password_reuse_prevention      = 12
  hard_expiry                    = false
}
```

### Best Practices Summary

| Practice | ရှင်းလင်းချက် |
|----------|---------------|
| **Least Privilege** | လိုအပ်သော Permission သာ ပေးပါ |
| **MFA Enable** | Admin Users အားလုံး MFA ဖွင့်ပါ |
| **Root Account** | Root Account ကို နေ့စဉ် မသုံးပါနှင့် |
| **Access Keys Rotate** | Access Keys ကို ပုံမှန် ပြောင်းပါ |
| **Groups** | Users တိုက်ရိုက် Policy မတပ်ပါနှင့်၊ Groups ဖြင့် စီမံပါ |
| **Roles** | EC2, Lambda စသည် Services အတွက် Roles သုံးပါ |
| **Conditions** | Policy Conditions ဖြင့် ကန့်သတ်ပါ (IP, Time, MFA, Tags) |
| **Audit** | CloudTrail ဖြင့် IAM Actions ကို Monitor လုပ်ပါ |

---

## 7.8 လက်တွေ့ - Organization-level IAM Structure

```hcl
# outputs.tf
output "developer_group_arn" {
  value = aws_iam_group.developers.arn
}

output "ec2_role_arn" {
  value = aws_iam_role.ec2_app_role.arn
}

output "instance_profile_name" {
  value = aws_iam_instance_profile.ec2_app.name
}

output "developer_access_keys" {
  value     = { for k, v in aws_iam_access_key.developers : k => v.id }
  sensitive = true
}
```

---

## အခန်း (၇) အကျဉ်းချုပ်

- ✅ **IAM Users, Groups, Roles** ဖန်တီးခြင်း
- ✅ **Custom Policies** (`data.aws_iam_policy_document`)
- ✅ **Policy Attachments** (Managed, Inline, Group-level)
- ✅ **Instance Profiles** (EC2 ← IAM Role)
- ✅ **Cross-Account Access** (Assume Role)
- ✅ **IAM Best Practices** (Least Privilege, MFA, Password Policy)

> **📖 နောက်အခန်းတွင်** RDS Database ကို Terraform ဖြင့် စီမံခန့်ခွဲခြင်းကို လေ့လာကြပါမည်။

---

*[← အခန်း (၆)](chapter-06.md) | [အခန်း (၈) - RDS Database →](chapter-08.md)*
