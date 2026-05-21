# အခန်း (၈) - RDS Database စီမံခန့်ခွဲခြင်း

---

## 8.1 RDS Instance ဖန်တီးခြင်း

### RDS ဆိုတာ ဘာလဲ?

**RDS (Relational Database Service)** ဆိုတာ AWS ၏ Managed Database Service ဖြစ်ပါတယ်။ Database Server ကို AWS က Monitor, Backup, Patch, Scale လုပ်ပေးပါတယ်။

**Support Database Engines:**
- MySQL
- PostgreSQL
- MariaDB
- Oracle
- Microsoft SQL Server
- Amazon Aurora (MySQL/PostgreSQL Compatible)

### MySQL RDS Instance

```hcl
# ===================================
# RDS MySQL Instance
# ===================================
resource "aws_db_instance" "mysql" {
  identifier = "${var.project_name}-${var.environment}-mysql"

  # Engine
  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = var.environment == "prod" ? "db.r5.large" : "db.t3.micro"

  # Storage
  allocated_storage     = 20
  max_allocated_storage = 100  # Auto Scaling
  storage_type          = "gp3"
  storage_encrypted     = true

  # Credentials
  db_name  = "appdb"
  username = var.db_username
  password = var.db_password

  # Network
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.database.id]
  publicly_accessible    = false

  # Backup
  backup_retention_period = var.environment == "prod" ? 30 : 7
  backup_window           = "03:00-04:00"  # UTC

  # Maintenance
  maintenance_window        = "Mon:04:00-Mon:05:00"
  auto_minor_version_upgrade = true

  # Deletion Protection
  deletion_protection = var.environment == "prod" ? true : false
  skip_final_snapshot = var.environment == "prod" ? false : true
  final_snapshot_identifier = var.environment == "prod" ? "${var.project_name}-final-snapshot" : null

  # Parameter Group
  parameter_group_name = aws_db_parameter_group.mysql.name

  # Monitoring
  monitoring_interval          = var.environment == "prod" ? 60 : 0
  monitoring_role_arn          = var.environment == "prod" ? aws_iam_role.rds_monitoring[0].arn : null
  performance_insights_enabled = var.environment == "prod" ? true : false

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-mysql"
  })
}
```

### PostgreSQL RDS Instance

```hcl
resource "aws_db_instance" "postgres" {
  identifier = "${var.project_name}-${var.environment}-postgres"

  engine         = "postgres"
  engine_version = "15.4"
  instance_class = var.environment == "prod" ? "db.r5.large" : "db.t3.micro"

  allocated_storage     = 20
  max_allocated_storage = 100
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = "appdb"
  username = var.db_username
  password = var.db_password

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.database.id]
  publicly_accessible    = false

  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "Mon:04:00-Mon:05:00"

  skip_final_snapshot = var.environment != "prod"
  deletion_protection = var.environment == "prod"

  parameter_group_name = aws_db_parameter_group.postgres.name

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-postgres"
  })
}
```

---

## 8.2 DB Subnet Groups

```hcl
# ===================================
# DB Subnet Group
# ===================================
resource "aws_db_subnet_group" "main" {
  name       = "${var.project_name}-${var.environment}-db-subnet-group"
  subnet_ids = aws_subnet.database[*].id

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-db-subnet-group"
  })
}
```

---

## 8.3 Parameter Groups နှင့် Option Groups

### Parameter Groups

```hcl
# ===================================
# MySQL Parameter Group
# ===================================
resource "aws_db_parameter_group" "mysql" {
  name   = "${var.project_name}-${var.environment}-mysql-params"
  family = "mysql8.0"

  parameter {
    name  = "character_set_server"
    value = "utf8mb4"
  }

  parameter {
    name  = "collation_server"
    value = "utf8mb4_unicode_ci"
  }

  parameter {
    name  = "max_connections"
    value = "200"
  }

  parameter {
    name  = "slow_query_log"
    value = "1"
  }

  parameter {
    name  = "long_query_time"
    value = "2"
  }

  parameter {
    name         = "innodb_buffer_pool_size"
    value        = "{DBInstanceClassMemory*3/4}"
    apply_method = "pending-reboot"
  }

  tags = local.common_tags
}

# ===================================
# PostgreSQL Parameter Group
# ===================================
resource "aws_db_parameter_group" "postgres" {
  name   = "${var.project_name}-${var.environment}-postgres-params"
  family = "postgres15"

  parameter {
    name  = "log_connections"
    value = "1"
  }

  parameter {
    name  = "log_disconnections"
    value = "1"
  }

  parameter {
    name  = "log_statement"
    value = "all"
  }

  parameter {
    name  = "log_min_duration_statement"
    value = "1000"  # 1 second
  }

  tags = local.common_tags
}
```

---

## 8.4 Multi-AZ Deployment

```hcl
# ===================================
# Multi-AZ RDS (Production)
# ===================================
resource "aws_db_instance" "mysql_ha" {
  identifier = "${var.project_name}-prod-mysql-ha"

  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.r5.large"

  allocated_storage     = 100
  max_allocated_storage = 500
  storage_type          = "gp3"
  iops                  = 3000
  storage_encrypted     = true

  db_name  = "appdb"
  username = var.db_username
  password = var.db_password

  # Multi-AZ ← ဒါကို true လုပ်ရုံပဲ
  multi_az = true

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.database.id]
  publicly_accessible    = false

  backup_retention_period = 30
  deletion_protection     = true
  skip_final_snapshot     = false

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-prod-mysql-ha"
  })
}
```

### Multi-AZ ၏ အလုပ်လုပ်ပုံ

```
┌─── AZ-1a ─────────────┐    ┌─── AZ-1b ─────────────┐
│                         │    │                         │
│  ┌── Primary DB ─────┐ │    │  ┌── Standby DB ────┐  │
│  │  Read + Write      │ │    │  │  Synchronous      │  │
│  │  (Active)          │─┼────┼─▶│  Replication       │  │
│  └────────────────────┘ │    │  │  (Passive)         │  │
│                         │    │  └────────────────────┘  │
└─────────────────────────┘    └──────────────────────────┘
                                        │
                         Primary Fail ──▶ Auto Failover
                                        (60-120 seconds)
```

---

## 8.5 Read Replicas

```hcl
# ===================================
# Read Replica
# ===================================
resource "aws_db_instance" "mysql_read_replica" {
  identifier = "${var.project_name}-${var.environment}-mysql-replica"

  replicate_source_db = aws_db_instance.mysql.identifier
  instance_class      = "db.t3.medium"

  # Read Replica Configuration
  publicly_accessible    = false
  vpc_security_group_ids = [aws_security_group.database.id]

  # Backup (Read Replica မှာ Backup Disable)
  backup_retention_period = 0

  skip_final_snapshot = true

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-mysql-replica"
    Role = "read-replica"
  })
}

# Cross-Region Read Replica
resource "aws_db_instance" "mysql_cross_region_replica" {
  provider = aws.us_east

  identifier          = "${var.project_name}-mysql-us-replica"
  replicate_source_db = aws_db_instance.mysql.arn  # ARN for cross-region
  instance_class      = "db.t3.medium"

  skip_final_snapshot = true

  tags = {
    Name = "${var.project_name}-mysql-us-replica"
    Role = "cross-region-read-replica"
  }
}
```

---

## 8.6 Automated Backups နှင့် Snapshots

```hcl
# Manual Snapshot (On-demand)
# AWS CLI: aws rds create-db-snapshot --db-instance-identifier xxx --db-snapshot-identifier xxx

# Restore from Snapshot
resource "aws_db_instance" "restored" {
  identifier = "${var.project_name}-restored"

  # Snapshot မှ Restore
  snapshot_identifier = "my-snapshot-id"

  instance_class         = "db.t3.micro"
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.database.id]

  skip_final_snapshot = true

  tags = {
    Name = "${var.project_name}-restored"
  }
}
```

---

## 8.7 RDS Proxy

```hcl
# ===================================
# RDS Proxy (Connection Pooling)
# ===================================

# Secrets Manager for DB Credentials
resource "aws_secretsmanager_secret" "db_credentials" {
  name = "${var.project_name}-${var.environment}-db-credentials"
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id

  secret_string = jsonencode({
    username = var.db_username
    password = var.db_password
  })
}

# RDS Proxy IAM Role
resource "aws_iam_role" "rds_proxy" {
  name = "${var.project_name}-${var.environment}-rds-proxy-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "rds.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "rds_proxy" {
  name = "rds-proxy-policy"
  role = aws_iam_role.rds_proxy.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ]
      Resource = aws_secretsmanager_secret.db_credentials.arn
    }]
  })
}

# RDS Proxy
resource "aws_db_proxy" "main" {
  name                   = "${var.project_name}-${var.environment}-proxy"
  debug_logging          = false
  engine_family          = "MYSQL"
  idle_client_timeout    = 1800
  require_tls            = true
  role_arn               = aws_iam_role.rds_proxy.arn
  vpc_security_group_ids = [aws_security_group.database.id]
  vpc_subnet_ids         = aws_subnet.database[*].id

  auth {
    auth_scheme = "SECRETS"
    description = "DB credentials"
    iam_auth    = "DISABLED"
    secret_arn  = aws_secretsmanager_secret.db_credentials.arn
  }

  tags = local.common_tags
}

# Proxy Target
resource "aws_db_proxy_default_target_group" "main" {
  db_proxy_name = aws_db_proxy.main.name

  connection_pool_config {
    max_connections_percent = 100
    max_idle_connections_percent = 50
    connection_borrow_timeout = 120
  }
}

resource "aws_db_proxy_target" "main" {
  db_proxy_name          = aws_db_proxy.main.name
  target_group_name      = aws_db_proxy_default_target_group.main.name
  db_instance_identifier = aws_db_instance.mysql.identifier
}
```

---

## 8.8 လက်တွေ့ - Production-Ready Database Setup

### Variables

```hcl
variable "db_username" {
  description = "Database administrator username"
  type        = string
  sensitive   = true
}

variable "db_password" {
  description = "Database administrator password"
  type        = string
  sensitive   = true

  validation {
    condition     = length(var.db_password) >= 12
    error_message = "Password must be at least 12 characters."
  }
}
```

### Outputs

```hcl
output "db_endpoint" {
  description = "RDS Endpoint"
  value       = aws_db_instance.mysql.endpoint
}

output "db_name" {
  description = "Database Name"
  value       = aws_db_instance.mysql.db_name
}

output "db_port" {
  description = "Database Port"
  value       = aws_db_instance.mysql.port
}

output "proxy_endpoint" {
  description = "RDS Proxy Endpoint"
  value       = aws_db_proxy.main.endpoint
}
```

---

## အခန်း (၈) အကျဉ်းချုပ်

- ✅ **RDS Instance** ဖန်တီးခြင်း (MySQL, PostgreSQL)
- ✅ **DB Subnet Groups** နှင့် **Parameter Groups**
- ✅ **Multi-AZ Deployment** (High Availability)
- ✅ **Read Replicas** (Performance, Cross-Region)
- ✅ **Automated Backups** နှင့် Snapshots
- ✅ **RDS Proxy** (Connection Pooling)

> **📖 နောက်အခန်းတွင်** Load Balancing နှင့် High Availability ကို လေ့လာကြပါမည်။

---

*[← အခန်း (၇)](chapter-07.md) | [အခန်း (၉) - Load Balancing →](chapter-09.md)*
