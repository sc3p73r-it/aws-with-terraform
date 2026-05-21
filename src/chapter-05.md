# အခန်း (၅) - EC2 Instances စီမံခန့်ခွဲခြင်း

---

## 5.1 AMI ရွေးချယ်ခြင်းနှင့် Data Source အသုံးပြုခြင်း

### AMI ဆိုတာ ဘာလဲ?

**AMI (Amazon Machine Image)** ဆိုတာ EC2 Instance ဖန်တီးဖို့ လိုအပ်တဲ့ **Template** ဖြစ်ပါတယ်။ OS, Software, Configuration တွေ ပါဝင်ပါတယ်။

### Data Source ဖြင့် AMI ရှာခြင်း

```hcl
# Amazon Linux 2023 - Latest
data "aws_ami" "amazon_linux_2023" {
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

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
}

# Ubuntu 22.04 LTS - Latest
data "aws_ami" "ubuntu_22" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Windows Server 2022 - Latest
data "aws_ami" "windows_2022" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["Windows_Server-2022-English-Full-Base-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Output ဖြင့် AMI ID ကြည့်ခြင်း
output "amazon_linux_ami_id" {
  value = data.aws_ami.amazon_linux_2023.id
}

output "ubuntu_ami_id" {
  value = data.aws_ami.ubuntu_22.id
}
```

---

## 5.2 EC2 Instance ဖန်တီးခြင်း

### အခြေခံ EC2 Instance

```hcl
# ===================================
# SSH Key Pair
# ===================================
resource "aws_key_pair" "deployer" {
  key_name   = "${var.project_name}-${var.environment}-key"
  public_key = file("~/.ssh/id_rsa.pub")  # ← ကိုယ့် Public Key Path

  tags = local.common_tags
}

# ===================================
# EC2 Instance
# ===================================
resource "aws_instance" "web_server" {
  ami                    = data.aws_ami.amazon_linux_2023.id
  instance_type          = var.instance_type
  key_name               = aws_key_pair.deployer.key_name
  subnet_id              = aws_subnet.public[0].id
  vpc_security_group_ids = [aws_security_group.web.id]

  # Root Volume
  root_block_device {
    volume_type           = "gp3"
    volume_size           = 20
    encrypted             = true
    delete_on_termination = true

    tags = {
      Name = "${var.project_name}-${var.environment}-web-root"
    }
  }

  # Instance Metadata Service v2 (Security Best Practice)
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # IMDSv2 Only
    http_put_response_hop_limit = 1
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-web-server"
    Role = "web"
  })
}
```

### Instance Types Reference

```
┌──────────────────────────────────────────────────────────────┐
│                    EC2 Instance Types                        │
├──────────────┬──────────┬────────┬───────────────────────────┤
│ Family       │ Type     │ vCPU   │ Memory (GiB)             │
├──────────────┼──────────┼────────┼───────────────────────────┤
│ General      │ t2.micro │ 1      │ 1    (Free Tier)         │
│ Purpose      │ t2.small │ 1      │ 2                        │
│              │ t3.medium│ 2      │ 4                        │
│              │ t3.large │ 2      │ 8                        │
│              │ m5.large │ 2      │ 8                        │
│              │ m5.xlarge│ 4      │ 16                       │
├──────────────┼──────────┼────────┼───────────────────────────┤
│ Compute      │ c5.large │ 2      │ 4                        │
│ Optimized    │ c5.xlarge│ 4      │ 8                        │
├──────────────┼──────────┼────────┼───────────────────────────┤
│ Memory       │ r5.large │ 2      │ 16                       │
│ Optimized    │ r5.xlarge│ 4      │ 32                       │
├──────────────┼──────────┼────────┼───────────────────────────┤
│ Storage      │ i3.large │ 2      │ 15.25                    │
│ Optimized    │ d2.xlarge│ 4      │ 30.5                     │
└──────────────┴──────────┴────────┴───────────────────────────┘
```

### Variable-based Instance Type

```hcl
variable "instance_type" {
  description = "EC2 Instance Type"
  type        = string

  validation {
    condition     = contains(["t2.micro", "t2.small", "t3.medium", "t3.large"], var.instance_type)
    error_message = "Allowed instance types: t2.micro, t2.small, t3.medium, t3.large"
  }
}

# Environment အလိုက် Instance Type ပြောင်းခြင်း
locals {
  instance_types = {
    dev     = "t2.micro"
    staging = "t3.small"
    prod    = "t3.large"
  }
  selected_instance_type = local.instance_types[var.environment]
}
```

---

## 5.3 User Data Scripts ဖြင့် Bootstrap လုပ်ခြင်း

### User Data ဆိုတာ ဘာလဲ?

User Data ဆိုတာ EC2 Instance **ပထမဆုံး Boot** တဲ့အခါ Auto Run မယ့် Script ဖြစ်ပါတယ်။ Software Install, Configuration, Service Start စတာတွေ Auto လုပ်နိုင်ပါတယ်။

### Inline User Data

```hcl
resource "aws_instance" "web_server" {
  ami                    = data.aws_ami.amazon_linux_2023.id
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.public[0].id
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOF
    #!/bin/bash
    # System Update
    yum update -y

    # Install Nginx
    yum install -y nginx

    # Custom HTML Page
    cat > /usr/share/nginx/html/index.html <<'HTML'
    <!DOCTYPE html>
    <html>
    <head><title>My Terraform Server</title></head>
    <body>
      <h1>Hello from Terraform!</h1>
      <p>This server was provisioned by Terraform.</p>
      <p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>
      <p>Availability Zone: $(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)</p>
    </body>
    </html>
    HTML

    # Start & Enable Nginx
    systemctl start nginx
    systemctl enable nginx
  EOF

  user_data_replace_on_change = true  # User Data ပြောင်းရင် Instance ပြန်ဖန်တီး

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-web-server"
  })
}
```

### templatefile() ဖြင့် User Data

```hcl
# templates/user_data.sh
#!/bin/bash
set -e

# Logging
exec > >(tee /var/log/user-data.log) 2>&1
echo "=== User Data Script Started at $(date) ==="

# System Update
yum update -y

# Install packages
yum install -y nginx jq awscli

# Configure Nginx
cat > /usr/share/nginx/html/index.html <<'HTML'
<!DOCTYPE html>
<html>
<head><title>${project_name} - ${environment}</title></head>
<body>
  <h1>Welcome to ${project_name}</h1>
  <p>Environment: ${environment}</p>
  <p>Server Port: ${server_port}</p>
</body>
</html>
HTML

# Start Nginx
systemctl start nginx
systemctl enable nginx

echo "=== User Data Script Completed at $(date) ==="
```

```hcl
# main.tf
resource "aws_instance" "web_server" {
  ami           = data.aws_ami.amazon_linux_2023.id
  instance_type = "t2.micro"

  user_data = templatefile("${path.module}/templates/user_data.sh", {
    project_name = var.project_name
    environment  = var.environment
    server_port  = 80
  })

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-web"
  })
}
```

---

## 5.4 EBS Volumes ထည့်သွင်းခြင်း

### Additional EBS Volume

```hcl
# ===================================
# EBS Volume ဖန်တီးခြင်း
# ===================================
resource "aws_ebs_volume" "data_volume" {
  availability_zone = aws_instance.web_server.availability_zone
  size              = 50        # GB
  type              = "gp3"
  iops              = 3000
  throughput         = 125      # MB/s
  encrypted         = true

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-data-volume"
  })
}

# ===================================
# EC2 Instance နှင့် Attach လုပ်ခြင်း
# ===================================
resource "aws_volume_attachment" "data_volume_attachment" {
  device_name = "/dev/sdf"
  volume_id   = aws_ebs_volume.data_volume.id
  instance_id = aws_instance.web_server.id
}
```

### EBS Volume Types

| Type | IOPS | Throughput | သင့်တော် |
|------|------|------------|-----------|
| **gp3** | 3,000 - 16,000 | 125 - 1,000 MB/s | General Purpose (Recommended) |
| **gp2** | 100 - 16,000 | - | General Purpose (Legacy) |
| **io1/io2** | 64,000 | 1,000 MB/s | High Performance DB |
| **st1** | 500 | 500 MB/s | Big Data, Logs |
| **sc1** | 250 | 250 MB/s | Archive, Infrequent Access |

---

## 5.5 Elastic IP Addresses

```hcl
# ===================================
# Elastic IP
# ===================================
resource "aws_eip" "web_server" {
  domain = "vpc"

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-web-eip"
  })
}

# EC2 Instance နှင့် Associate
resource "aws_eip_association" "web_server" {
  instance_id   = aws_instance.web_server.id
  allocation_id = aws_eip.web_server.id
}

output "web_server_elastic_ip" {
  description = "Web Server Elastic IP"
  value       = aws_eip.web_server.public_ip
}
```

---

## 5.6 Launch Templates

### Launch Template ဆိုတာ ဘာလဲ?

Launch Template ဆိုတာ EC2 Instance ဖန်တီးဖို့ **Template** ဖြစ်ပါတယ်။ AMI, Instance Type, Security Groups, User Data စသည့် Configuration များကို Template အဖြစ် သိမ်းထားပြီး Auto Scaling Group, EC2 Fleet တို့မှာ ပြန်သုံးနိုင်ပါတယ်။

```hcl
# ===================================
# Launch Template
# ===================================
resource "aws_launch_template" "web" {
  name_prefix   = "${var.project_name}-${var.environment}-web-"
  description   = "Launch template for web servers"
  image_id      = data.aws_ami.amazon_linux_2023.id
  instance_type = local.selected_instance_type
  key_name      = aws_key_pair.deployer.key_name

  # Network
  vpc_security_group_ids = [aws_security_group.web.id]

  # EBS Root Volume
  block_device_mappings {
    device_name = "/dev/xvda"

    ebs {
      volume_size           = 20
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }

  # Additional Data Volume
  block_device_mappings {
    device_name = "/dev/sdf"

    ebs {
      volume_size           = 50
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }

  # IMDSv2
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
  }

  # Monitoring
  monitoring {
    enabled = var.environment == "prod" ? true : false
  }

  # IAM Instance Profile
  iam_instance_profile {
    name = aws_iam_instance_profile.web.name
  }

  # User Data
  user_data = base64encode(templatefile("${path.module}/templates/user_data.sh", {
    project_name = var.project_name
    environment  = var.environment
    server_port  = 80
  }))

  # Tags
  tag_specifications {
    resource_type = "instance"
    tags = merge(local.common_tags, {
      Name = "${var.project_name}-${var.environment}-web"
      Role = "web-server"
    })
  }

  tag_specifications {
    resource_type = "volume"
    tags = merge(local.common_tags, {
      Name = "${var.project_name}-${var.environment}-web-volume"
    })
  }

  tags = local.common_tags

  lifecycle {
    create_before_destroy = true
  }
}
```

---

## 5.7 Auto Scaling Groups (ASG)

### Auto Scaling Group ဆိုတာ ဘာလဲ?

ASG ဆိုတာ EC2 Instances အရေအတွက်ကို Load/Demand ပေါ်မူတည်ပြီး **အလိုအလျောက် တိုး/လျှော့** ပေးတဲ့ Service ဖြစ်ပါတယ်။

```
Load နည်း ──▶ Instances 2 လုံး  (Min)
Load များ  ──▶ Instances 6 လုံး  (Scale Out)
Load ပြန်ကျ ──▶ Instances 2 လုံး  (Scale In)
```

### Auto Scaling Group Configuration

```hcl
# ===================================
# Auto Scaling Group
# ===================================
resource "aws_autoscaling_group" "web" {
  name                = "${var.project_name}-${var.environment}-web-asg"
  desired_capacity    = 2
  min_size            = 1
  max_size            = 6
  vpc_zone_identifier = aws_subnet.public[*].id  # Multiple AZs

  # Launch Template
  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  # Health Check
  health_check_type         = "ELB"  # EC2 or ELB
  health_check_grace_period = 300    # seconds

  # Instance Refresh (Rolling Update)
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 50
      instance_warmup        = 120
    }
  }

  # Tags
  tag {
    key                 = "Name"
    value               = "${var.project_name}-${var.environment}-web-asg"
    propagate_at_launch = true
  }

  tag {
    key                 = "Environment"
    value               = var.environment
    propagate_at_launch = true
  }

  tag {
    key                 = "ManagedBy"
    value               = "terraform"
    propagate_at_launch = true
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

### Auto Scaling Policies

```hcl
# ===================================
# Scale Out Policy (Instance တိုးခြင်း)
# ===================================
resource "aws_autoscaling_policy" "scale_out" {
  name                   = "${var.project_name}-${var.environment}-scale-out"
  scaling_adjustment     = 1
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300
  autoscaling_group_name = aws_autoscaling_group.web.name
}

# CPU > 70% ဆိုရင် Scale Out
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "${var.project_name}-${var.environment}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 70
  alarm_description   = "Scale out when CPU > 70%"
  alarm_actions       = [aws_autoscaling_policy.scale_out.arn]

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.web.name
  }
}

# ===================================
# Scale In Policy (Instance လျှော့ခြင်း)
# ===================================
resource "aws_autoscaling_policy" "scale_in" {
  name                   = "${var.project_name}-${var.environment}-scale-in"
  scaling_adjustment     = -1
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300
  autoscaling_group_name = aws_autoscaling_group.web.name
}

# CPU < 30% ဆိုရင် Scale In
resource "aws_cloudwatch_metric_alarm" "low_cpu" {
  alarm_name          = "${var.project_name}-${var.environment}-low-cpu"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 30
  alarm_description   = "Scale in when CPU < 30%"
  alarm_actions       = [aws_autoscaling_policy.scale_in.arn]

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.web.name
  }
}

# ===================================
# Target Tracking Policy (ပိုလွယ်ကူ)
# ===================================
resource "aws_autoscaling_policy" "target_tracking" {
  name                   = "${var.project_name}-${var.environment}-target-tracking"
  policy_type            = "TargetTrackingScaling"
  autoscaling_group_name = aws_autoscaling_group.web.name

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 50.0  # CPU 50% ဝန်းကျင်မှာ ထိန်းထား
  }
}
```

---

## 5.8 လက်တွေ့ - Web Server Fleet တည်ဆောက်ခြင်း

### IAM Role for EC2

```hcl
# ===================================
# IAM Role for EC2 Instances
# ===================================
resource "aws_iam_role" "web" {
  name = "${var.project_name}-${var.environment}-web-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })

  tags = local.common_tags
}

# S3 Read Access
resource "aws_iam_role_policy_attachment" "web_s3_read" {
  role       = aws_iam_role.web.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}

# CloudWatch Agent
resource "aws_iam_role_policy_attachment" "web_cloudwatch" {
  role       = aws_iam_role.web.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

# SSM (Session Manager - SSH Alternative)
resource "aws_iam_role_policy_attachment" "web_ssm" {
  role       = aws_iam_role.web.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# Instance Profile
resource "aws_iam_instance_profile" "web" {
  name = "${var.project_name}-${var.environment}-web-profile"
  role = aws_iam_role.web.name
}
```

### Complete Outputs

```hcl
# outputs.tf
output "launch_template_id" {
  description = "Launch Template ID"
  value       = aws_launch_template.web.id
}

output "asg_name" {
  description = "Auto Scaling Group Name"
  value       = aws_autoscaling_group.web.name
}

output "asg_arn" {
  description = "Auto Scaling Group ARN"
  value       = aws_autoscaling_group.web.arn
}
```

---

## အခန်း (၅) အကျဉ်းချုပ်

ဤအခန်းတွင် အောက်ပါ အချက်များကို လေ့လာခဲ့ပါသည်:

- ✅ **AMI Data Sources** - OS Image များ ရှာခွေခြင်း
- ✅ **EC2 Instance** - Server ဖန်တီးခြင်း (Key Pairs, Security Groups)
- ✅ **User Data** - Bootstrap Scripts ဖြင့် Auto Configuration
- ✅ **EBS Volumes** - Additional Storage ထည့်သွင်းခြင်း
- ✅ **Elastic IP** - Static IP Address
- ✅ **Launch Templates** - Instance Configuration Templates
- ✅ **Auto Scaling Groups** - Auto Scale Out/In
- ✅ **IAM Roles** - EC2 Instance Permissions

> **📖 နောက်အခန်းတွင်** S3 Storage ကို Terraform ဖြင့် အသေးစိတ် စီမံခန့်ခွဲခြင်းကို လေ့လာကြပါမည်။

---

*[← အခန်း (၄)](chapter-04.md) | [အခန်း (၆) - S3 Storage →](chapter-06.md)*
