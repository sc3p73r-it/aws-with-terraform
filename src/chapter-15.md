# အခန်း (၁၅) - Monitoring, Security နှင့် Best Practices

---

## 15.1 CloudWatch Alarms နှင့် Metrics

### CloudWatch Metric Alarms

```hcl
# ===================================
# EC2 CPU Alarm
# ===================================
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "${var.project_name}-${var.environment}-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300  # 5 minutes
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "CPU utilization > 80%"

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.web.name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]

  tags = local.common_tags
}

# ===================================
# RDS Free Storage Alarm
# ===================================
resource "aws_cloudwatch_metric_alarm" "db_storage" {
  alarm_name          = "${var.project_name}-${var.environment}-db-storage-low"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "FreeStorageSpace"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 5368709120  # 5GB in bytes
  alarm_description   = "RDS free storage < 5GB"

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.mysql.identifier
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

# ===================================
# ALB 5xx Error Rate
# ===================================
resource "aws_cloudwatch_metric_alarm" "alb_5xx" {
  alarm_name          = "${var.project_name}-${var.environment}-alb-5xx"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HTTPCode_ELB_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = 300
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "ALB 5xx errors > 10"

  dimensions = {
    LoadBalancer = aws_lb.web.arn_suffix
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

# ===================================
# CloudWatch Dashboard
# ===================================
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "${var.project_name}-${var.environment}"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["AWS/EC2", "CPUUtilization", "AutoScalingGroupName", aws_autoscaling_group.web.name]
          ]
          period = 300
          stat   = "Average"
          region = var.aws_region
          title  = "EC2 CPU Utilization"
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 0
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", aws_lb.web.arn_suffix]
          ]
          period = 60
          stat   = "Sum"
          region = var.aws_region
          title  = "ALB Request Count"
        }
      },
      {
        type   = "metric"
        x      = 0
        y      = 6
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["AWS/RDS", "CPUUtilization", "DBInstanceIdentifier", aws_db_instance.mysql.identifier],
            ["AWS/RDS", "DatabaseConnections", "DBInstanceIdentifier", aws_db_instance.mysql.identifier]
          ]
          period = 300
          region = var.aws_region
          title  = "RDS Metrics"
        }
      }
    ]
  })
}
```

---

## 15.2 SNS Notifications

```hcl
# ===================================
# SNS Topic
# ===================================
resource "aws_sns_topic" "alerts" {
  name = "${var.project_name}-${var.environment}-alerts"

  tags = local.common_tags
}

# Email Subscription
resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}

# Slack Notification (via Lambda)
resource "aws_sns_topic_subscription" "slack" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.slack_notifier.arn
}
```

---

## 15.3 CloudTrail Logging

```hcl
# ===================================
# CloudTrail
# ===================================
resource "aws_cloudtrail" "main" {
  name                          = "${var.project_name}-${var.environment}-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3"]
    }
  }

  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail.arn

  tags = local.common_tags
}

# CloudTrail S3 Bucket
resource "aws_s3_bucket" "cloudtrail" {
  bucket        = "${var.project_name}-${var.environment}-cloudtrail"
  force_destroy = var.environment != "prod"
}

resource "aws_s3_bucket_policy" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "AWSCloudTrailAclCheck"
        Effect    = "Allow"
        Principal = { Service = "cloudtrail.amazonaws.com" }
        Action    = "s3:GetBucketAcl"
        Resource  = aws_s3_bucket.cloudtrail.arn
      },
      {
        Sid       = "AWSCloudTrailWrite"
        Effect    = "Allow"
        Principal = { Service = "cloudtrail.amazonaws.com" }
        Action    = "s3:PutObject"
        Resource  = "${aws_s3_bucket.cloudtrail.arn}/*"
        Condition = {
          StringEquals = {
            "s3:x-amz-acl" = "bucket-owner-full-control"
          }
        }
      }
    ]
  })
}
```

---

## 15.4 AWS Config Rules

```hcl
# ===================================
# AWS Config
# ===================================
resource "aws_config_configuration_recorder" "main" {
  name     = "${var.project_name}-config"
  role_arn = aws_iam_role.config.arn

  recording_group {
    all_supported = true
  }
}

# Config Rule: S3 Encryption Check
resource "aws_config_config_rule" "s3_encrypted" {
  name = "s3-bucket-server-side-encryption-enabled"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
  }

  depends_on = [aws_config_configuration_recorder.main]
}

# Config Rule: EC2 Public IP Check
resource "aws_config_config_rule" "ec2_no_public_ip" {
  name = "ec2-instance-no-public-ip"

  source {
    owner             = "AWS"
    source_identifier = "EC2_INSTANCE_NO_PUBLIC_IP"
  }

  depends_on = [aws_config_configuration_recorder.main]
}

# Config Rule: RDS Encryption
resource "aws_config_config_rule" "rds_encrypted" {
  name = "rds-storage-encrypted"

  source {
    owner             = "AWS"
    source_identifier = "RDS_STORAGE_ENCRYPTED"
  }

  depends_on = [aws_config_configuration_recorder.main]
}
```

---

## 15.5 Terraform Security Best Practices

### Secret Management

```hcl
# ===================================
# AWS Secrets Manager
# ===================================
resource "aws_secretsmanager_secret" "app_secrets" {
  name        = "${var.project_name}/${var.environment}/app-secrets"
  description = "Application secrets"

  tags = local.common_tags
}

resource "aws_secretsmanager_secret_version" "app_secrets" {
  secret_id = aws_secretsmanager_secret.app_secrets.id

  secret_string = jsonencode({
    database_url = "mysql://${var.db_username}:${var.db_password}@${aws_db_instance.mysql.endpoint}/appdb"
    api_key      = var.api_key
    jwt_secret   = var.jwt_secret
  })
}

# ===================================
# SSM Parameter Store
# ===================================
resource "aws_ssm_parameter" "db_host" {
  name  = "/${var.project_name}/${var.environment}/db/host"
  type  = "String"
  value = aws_db_instance.mysql.address

  tags = local.common_tags
}

resource "aws_ssm_parameter" "db_password" {
  name  = "/${var.project_name}/${var.environment}/db/password"
  type  = "SecureString"  # KMS Encrypted
  value = var.db_password

  tags = local.common_tags
}
```

---

## 15.6 Cost Optimization Tips

```hcl
# ===================================
# Cost Optimization Strategies
# ===================================

# 1. Dev Environment Auto-Stop (EventBridge + Lambda)
resource "aws_scheduler_schedule" "stop_dev_instances" {
  name = "stop-dev-instances-nightly"

  schedule_expression = "cron(0 18 ? * MON-FRI *)"  # 6 PM Weekdays

  flexible_time_window {
    mode = "OFF"
  }

  target {
    arn      = "arn:aws:scheduler:::aws-sdk:ec2:stopInstances"
    role_arn = aws_iam_role.scheduler.arn

    input = jsonencode({
      InstanceIds = aws_instance.dev[*].id
    })
  }
}

# 2. Reserved/Spot Instances
resource "aws_autoscaling_group" "cost_optimized" {
  mixed_instances_policy {
    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.web.id
        version            = "$Latest"
      }

      override {
        instance_type = "t3.medium"
      }
      override {
        instance_type = "t3a.medium"
      }
    }

    instances_distribution {
      on_demand_base_capacity                  = 1      # 1 On-Demand
      on_demand_percentage_above_base_capacity = 25     # 25% On-Demand
      spot_allocation_strategy                 = "capacity-optimized"
    }
  }

  min_size         = 2
  max_size         = 10
  desired_capacity = 3
}

# 3. S3 Intelligent Tiering
resource "aws_s3_bucket_intelligent_tiering_configuration" "cost_optimization" {
  bucket = aws_s3_bucket.app_data.id
  name   = "entire-bucket"

  tiering {
    access_tier = "DEEP_ARCHIVE_ACCESS"
    days        = 180
  }

  tiering {
    access_tier = "ARCHIVE_ACCESS"
    days        = 90
  }
}
```

---

## 15.7 Tagging Strategy

```hcl
# ===================================
# Tagging Module
# ===================================
locals {
  required_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
    Owner       = var.team_name
    CostCenter  = var.cost_center
    CreatedBy   = "terraform"
  }

  # default_tags ကို Provider Level မှာ Set
}

# Provider Level Default Tags
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = local.required_tags
  }
}

# Resource Level Tags (Override / Additional)
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"

  tags = {
    Name = "web-server"  # Additional tag
    Role = "web"         # Additional tag
    # required_tags are automatically applied via default_tags
  }
}
```

---

## 15.8 Code Organization Best Practices

### Recommended Directory Structure

```
terraform-project/
├── modules/                    # Reusable Modules
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── ec2/
│   ├── rds/
│   ├── alb/
│   └── s3/
│
├── environments/               # Environment Configurations
│   ├── dev/
│   │   ├── main.tf            # Module calls
│   │   ├── providers.tf
│   │   ├── backend.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ... (same structure)
│   └── prod/
│       └── ... (same structure)
│
├── global/                     # Global Resources (IAM, Route53)
│   └── iam/
│       ├── main.tf
│       └── backend.tf
│
├── scripts/                    # Helper Scripts
│   ├── init-backend.sh
│   └── deploy.sh
│
├── .github/
│   └── workflows/
│       └── terraform.yml
│
├── .gitignore
├── .pre-commit-config.yaml
├── Makefile
└── README.md
```

### Naming Conventions

```hcl
# ✅ Good Naming
resource "aws_instance" "web_server" { }          # snake_case
resource "aws_s3_bucket" "application_data" { }    # descriptive
variable "instance_type" { }                       # clear meaning

# ❌ Bad Naming
resource "aws_instance" "instance1" { }            # non-descriptive
resource "aws_s3_bucket" "bucket" { }              # too generic
variable "it" { }                                  # unclear
```

### File Organization Rules

| File | ပါဝင်ရန် |
|------|-----------|
| `main.tf` | Primary Resources |
| `providers.tf` | Provider Configuration |
| `variables.tf` | All Variable Declarations |
| `outputs.tf` | All Output Declarations |
| `locals.tf` | Local Values |
| `data.tf` | Data Sources |
| `backend.tf` | Backend Configuration |
| `versions.tf` | Version Constraints |

---

## အခန်း (၁၅) အကျဉ်းချုပ်

- ✅ **CloudWatch** Alarms, Metrics, Dashboards
- ✅ **SNS** Notifications (Email, Slack)
- ✅ **CloudTrail** - API Activity Logging
- ✅ **AWS Config** - Compliance Rules
- ✅ **Secrets Manager** & **SSM Parameter Store**
- ✅ **Cost Optimization** (Spot, Auto-Stop, Tiering)
- ✅ **Tagging Strategy** (default_tags)
- ✅ **Code Organization** Best Practices

---

*[← အခန်း (၁၄)](chapter-14.md) | [အခန်း (၁၆) - Complete Project →](chapter-16.md)*
