# အခန်း (၆) - S3 Storage စီမံခန့်ခွဲခြင်း

---

## 6.1 S3 Bucket ဖန်တီးခြင်းနှင့် Configuration

### S3 ဆိုတာ ဘာလဲ?

**S3 (Simple Storage Service)** ဆိုတာ AWS ၏ **Object Storage** Service ဖြစ်ပါတယ်။ File, Image, Video, Backup, Log စသည်တို့ကို **Unlimited** သိမ်းဆည်းနိုင်ပါတယ်။

### အခြေခံ S3 Bucket

```hcl
# ===================================
# S3 Bucket ဖန်တီးခြင်း
# ===================================
resource "aws_s3_bucket" "app_data" {
  bucket = "${var.project_name}-${var.environment}-app-data"

  # Bucket ကို Destroy လုပ်ရင် Object တွေပါ ဖျက်ပေးမယ်
  force_destroy = var.environment != "prod" ? true : false

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-app-data"
  })
}

# Bucket Ownership Controls
resource "aws_s3_bucket_ownership_controls" "app_data" {
  bucket = aws_s3_bucket.app_data.id

  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}
```

---

## 6.2 Bucket Policies နှင့် ACLs

### Bucket Policy

```hcl
# ===================================
# Bucket Policy - Read Only Access
# ===================================
resource "aws_s3_bucket_policy" "app_data" {
  bucket = aws_s3_bucket.app_data.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "AllowEC2RoleAccess"
        Effect    = "Allow"
        Principal = {
          AWS = aws_iam_role.web.arn
        }
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.app_data.arn,
          "${aws_s3_bucket.app_data.arn}/*"
        ]
      },
      {
        Sid       = "DenyUnencryptedUploads"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:PutObject"
        Resource  = "${aws_s3_bucket.app_data.arn}/*"
        Condition = {
          StringNotEquals = {
            "s3:x-amz-server-side-encryption" = "AES256"
          }
        }
      }
    ]
  })
}

# ===================================
# Public Access Block (Security Best Practice)
# ===================================
resource "aws_s3_bucket_public_access_block" "app_data" {
  bucket = aws_s3_bucket.app_data.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Bucket Encryption

```hcl
# ===================================
# Server-Side Encryption
# ===================================
resource "aws_s3_bucket_server_side_encryption_configuration" "app_data" {
  bucket = aws_s3_bucket.app_data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
      # KMS Key သုံးချင်ရင်
      # sse_algorithm     = "aws:kms"
      # kms_master_key_id = aws_kms_key.s3_key.arn
    }
    bucket_key_enabled = true
  }
}
```

---

## 6.3 Versioning နှင့် Lifecycle Rules

### Versioning

```hcl
# ===================================
# Bucket Versioning
# ===================================
resource "aws_s3_bucket_versioning" "app_data" {
  bucket = aws_s3_bucket.app_data.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

### Lifecycle Rules

```hcl
# ===================================
# Lifecycle Rules - Cost Optimization
# ===================================
resource "aws_s3_bucket_lifecycle_configuration" "app_data" {
  bucket = aws_s3_bucket.app_data.id

  # Rule 1: Log Files
  rule {
    id     = "log-lifecycle"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    # 30 ရက်နောက် Infrequent Access သို့
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    # 90 ရက်နောက် Glacier သို့
    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    # 365 ရက်နောက် ဖျက်ပစ်
    expiration {
      days = 365
    }

    # Old Versions ကို 30 ရက်နောက် ဖျက်
    noncurrent_version_expiration {
      noncurrent_days = 30
    }
  }

  # Rule 2: Temp Files
  rule {
    id     = "temp-cleanup"
    status = "Enabled"

    filter {
      prefix = "tmp/"
    }

    expiration {
      days = 7
    }
  }

  # Rule 3: All Objects
  rule {
    id     = "general-lifecycle"
    status = "Enabled"

    filter {
      prefix = ""
    }

    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }

    noncurrent_version_expiration {
      noncurrent_days = 90
    }
  }

  depends_on = [aws_s3_bucket_versioning.app_data]
}
```

### S3 Storage Classes

| Storage Class | သင့်တော် | Cost (GB/month) |
|--------------|-----------|-----------------|
| **Standard** | Frequent Access | $0.023 |
| **Standard-IA** | Infrequent Access (30+ days) | $0.0125 |
| **One Zone-IA** | Non-critical, Infrequent | $0.01 |
| **Glacier Instant** | Archive, Milliseconds Access | $0.004 |
| **Glacier Flexible** | Archive, Minutes-Hours | $0.0036 |
| **Glacier Deep** | Long-term Archive, 12 Hours | $0.00099 |

---

## 6.4 Static Website Hosting

```hcl
# ===================================
# Static Website S3 Bucket
# ===================================
resource "aws_s3_bucket" "website" {
  bucket = "${var.project_name}-${var.environment}-website"

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-website"
  })
}

# Website Configuration
resource "aws_s3_bucket_website_configuration" "website" {
  bucket = aws_s3_bucket.website.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}

# Public Access ခွင့်ပြုခြင်း
resource "aws_s3_bucket_public_access_block" "website" {
  bucket = aws_s3_bucket.website.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

# Bucket Policy - Public Read
resource "aws_s3_bucket_policy" "website" {
  bucket = aws_s3_bucket.website.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "PublicReadGetObject"
      Effect    = "Allow"
      Principal = "*"
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.website.arn}/*"
    }]
  })

  depends_on = [aws_s3_bucket_public_access_block.website]
}

# Upload Files
resource "aws_s3_object" "index_html" {
  bucket       = aws_s3_bucket.website.id
  key          = "index.html"
  content      = <<-HTML
    <!DOCTYPE html>
    <html>
    <head><title>${var.project_name}</title></head>
    <body>
      <h1>Welcome to ${var.project_name}!</h1>
      <p>Deployed with Terraform</p>
    </body>
    </html>
  HTML
  content_type = "text/html"
}

resource "aws_s3_object" "error_html" {
  bucket       = aws_s3_bucket.website.id
  key          = "error.html"
  content      = <<-HTML
    <!DOCTYPE html>
    <html>
    <head><title>Error - ${var.project_name}</title></head>
    <body>
      <h1>404 - Page Not Found</h1>
    </body>
    </html>
  HTML
  content_type = "text/html"
}

# Output
output "website_url" {
  value = aws_s3_bucket_website_configuration.website.website_endpoint
}
```

---

## 6.5 S3 Backend for Terraform State

### Terraform State ကို S3 တွင် သိမ်းခြင်း

```hcl
# ===================================
# State Backend Infrastructure
# ===================================

# S3 Bucket for State
resource "aws_s3_bucket" "terraform_state" {
  bucket = "${var.project_name}-terraform-state"

  lifecycle {
    prevent_destroy = true  # ဖျက်ခံရမှာ ကာကွယ်
  }

  tags = {
    Name      = "Terraform State"
    ManagedBy = "terraform"
  }
}

# Versioning (State History)
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Public Access Block
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# DynamoDB Table for State Locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "${var.project_name}-terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name      = "Terraform State Lock"
    ManagedBy = "terraform"
  }
}
```

### Backend Configuration

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "myproject-terraform-state"
    key            = "dev/terraform.tfstate"
    region         = "ap-southeast-1"
    dynamodb_table = "myproject-terraform-locks"
    encrypt        = true
  }
}
```

---

## 6.6 Cross-Region Replication

```hcl
# ===================================
# Cross-Region Replication
# ===================================

# Source Bucket (Singapore)
resource "aws_s3_bucket" "source" {
  bucket   = "${var.project_name}-source"
  provider = aws
}

resource "aws_s3_bucket_versioning" "source" {
  bucket = aws_s3_bucket.source.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Destination Bucket (US East)
resource "aws_s3_bucket" "destination" {
  bucket   = "${var.project_name}-destination"
  provider = aws.us_east
}

resource "aws_s3_bucket_versioning" "destination" {
  provider = aws.us_east
  bucket   = aws_s3_bucket.destination.id
  versioning_configuration {
    status = "Enabled"
  }
}

# IAM Role for Replication
resource "aws_iam_role" "replication" {
  name = "${var.project_name}-s3-replication-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "s3.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "replication" {
  name = "s3-replication-policy"
  role = aws_iam_role.replication.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "s3:GetReplicationConfiguration",
          "s3:ListBucket"
        ]
        Effect   = "Allow"
        Resource = aws_s3_bucket.source.arn
      },
      {
        Action = [
          "s3:GetObjectVersionForReplication",
          "s3:GetObjectVersionAcl",
          "s3:GetObjectVersionTagging"
        ]
        Effect   = "Allow"
        Resource = "${aws_s3_bucket.source.arn}/*"
      },
      {
        Action = [
          "s3:ReplicateObject",
          "s3:ReplicateDelete",
          "s3:ReplicateTags"
        ]
        Effect   = "Allow"
        Resource = "${aws_s3_bucket.destination.arn}/*"
      }
    ]
  })
}

# Replication Configuration
resource "aws_s3_bucket_replication_configuration" "replication" {
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.source.id

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.destination.arn
      storage_class = "STANDARD_IA"
    }
  }

  depends_on = [
    aws_s3_bucket_versioning.source,
    aws_s3_bucket_versioning.destination
  ]
}
```

---

## 6.7 S3 Event Notifications

```hcl
# ===================================
# S3 Event → SNS Notification
# ===================================
resource "aws_sns_topic" "s3_notifications" {
  name = "${var.project_name}-s3-notifications"
}

resource "aws_sns_topic_policy" "s3_notifications" {
  arn = aws_sns_topic.s3_notifications.arn

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowS3Publish"
      Effect    = "Allow"
      Principal = { Service = "s3.amazonaws.com" }
      Action    = "SNS:Publish"
      Resource  = aws_sns_topic.s3_notifications.arn
      Condition = {
        ArnLike = {
          "aws:SourceArn" = aws_s3_bucket.app_data.arn
        }
      }
    }]
  })
}

resource "aws_s3_bucket_notification" "app_data" {
  bucket = aws_s3_bucket.app_data.id

  topic {
    topic_arn     = aws_sns_topic.s3_notifications.arn
    events        = ["s3:ObjectCreated:*"]
    filter_prefix = "uploads/"
    filter_suffix = ".csv"
  }

  topic {
    topic_arn = aws_sns_topic.s3_notifications.arn
    events    = ["s3:ObjectRemoved:*"]
  }
}
```

---

## 6.8 လက်တွေ့ - Static Website Deploy

```bash
# Deploy & Verify
$ terraform init
$ terraform apply

# Website URL ကို ရယူ
$ terraform output website_url
# myproject-dev-website.s3-website-ap-southeast-1.amazonaws.com

# Browser ဖြင့် ဖွင့်ကြည့်ပါ
```

---

## အခန်း (၆) အကျဉ်းချုပ်

- ✅ **S3 Bucket** ဖန်တီးခြင်း နှင့် Encryption
- ✅ **Bucket Policies** နှင့် Public Access Block
- ✅ **Versioning** နှင့် **Lifecycle Rules** (Cost Optimization)
- ✅ **Static Website Hosting**
- ✅ **S3 Backend** for Terraform State (Remote State)
- ✅ **Cross-Region Replication**
- ✅ **S3 Event Notifications**

> **📖 နောက်အခန်းတွင်** IAM ကို အသေးစိတ် လေ့လာကြပါမည်။

---

*[← အခန်း (၅)](chapter-05.md) | [အခန်း (၇) - IAM →](chapter-07.md)*
