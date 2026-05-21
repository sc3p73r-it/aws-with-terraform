# အခန်း (၉) - Load Balancing နှင့် High Availability

---

## 9.1 Application Load Balancer (ALB) တည်ဆောက်ခြင်း

### ALB ဆိုတာ ဘာလဲ?

**ALB (Application Load Balancer)** ဆိုတာ Incoming Traffic ကို Backend Servers များသို့ **ညီမျှစွာ ခွဲခြမ်းပေး**တဲ့ Service ဖြစ်ပါတယ်။ Layer 7 (HTTP/HTTPS) Level မှာ အလုပ်လုပ်ပါတယ်။

```
                   ┌─── Target Group ────┐
                   │                      │
Client ──▶ ALB ──▶ │  ┌── Instance 1 ──┐ │
                   │  └─────────────────┘ │
                   │  ┌── Instance 2 ──┐ │
                   │  └─────────────────┘ │
                   │  ┌── Instance 3 ──┐ │
                   │  └─────────────────┘ │
                   └──────────────────────┘
```

### ALB ဖန်တီးခြင်း

```hcl
# ===================================
# Application Load Balancer
# ===================================
resource "aws_lb" "web" {
  name               = "${var.project_name}-${var.environment}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = var.environment == "prod" ? true : false
  enable_http2               = true
  idle_timeout               = 60

  # Access Logs
  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "alb-logs"
    enabled = true
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-alb"
  })
}

# ALB Security Group
resource "aws_security_group" "alb" {
  name        = "${var.project_name}-${var.environment}-alb-sg"
  description = "Security group for ALB"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-alb-sg"
  })
}

# ALB Logs S3 Bucket
resource "aws_s3_bucket" "alb_logs" {
  bucket        = "${var.project_name}-${var.environment}-alb-logs"
  force_destroy = true

  tags = local.common_tags
}
```

---

## 9.2 Target Groups နှင့် Health Checks

```hcl
# ===================================
# Target Group
# ===================================
resource "aws_lb_target_group" "web" {
  name     = "${var.project_name}-${var.environment}-web-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  # Target Type
  target_type = "instance"  # instance, ip, lambda

  # Health Check
  health_check {
    enabled             = true
    healthy_threshold   = 3
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    matcher             = "200"
  }

  # Stickiness (Session Affinity)
  stickiness {
    type            = "lb_cookie"
    cookie_duration = 86400  # 1 day
    enabled         = false
  }

  # Deregistration Delay
  deregistration_delay = 30

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-web-tg"
  })
}

# API Target Group
resource "aws_lb_target_group" "api" {
  name     = "${var.project_name}-${var.environment}-api-tg"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    path    = "/api/health"
    matcher = "200"
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-api-tg"
  })
}
```

---

## 9.3 Listener Rules နှင့် Path-Based Routing

```hcl
# ===================================
# HTTP Listener (→ HTTPS Redirect)
# ===================================
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.web.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"

    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# ===================================
# HTTPS Listener
# ===================================
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.web.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

# ===================================
# Path-Based Routing Rules
# ===================================

# /api/* → API Target Group
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }
}

# /static/* → Fixed Response
resource "aws_lb_listener_rule" "static" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 200

  action {
    type = "fixed-response"

    fixed_response {
      content_type = "text/plain"
      message_body = "Static content served from S3/CloudFront"
      status_code  = "200"
    }
  }

  condition {
    path_pattern {
      values = ["/static/*"]
    }
  }
}

# Host-Based Routing
resource "aws_lb_listener_rule" "admin" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 50

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    host_header {
      values = ["admin.example.com"]
    }
  }
}
```

---

## 9.4 SSL/TLS Certificates (ACM)

```hcl
# ===================================
# ACM Certificate
# ===================================
resource "aws_acm_certificate" "main" {
  domain_name       = var.domain_name
  validation_method = "DNS"

  subject_alternative_names = [
    "*.${var.domain_name}"  # Wildcard
  ]

  lifecycle {
    create_before_destroy = true
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-cert"
  })
}

# DNS Validation Records
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = aws_route53_zone.main.zone_id
}

# Certificate Validation
resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}
```

---

## 9.5 Network Load Balancer (NLB)

```hcl
# ===================================
# Network Load Balancer (Layer 4)
# ===================================
resource "aws_lb" "network" {
  name               = "${var.project_name}-${var.environment}-nlb"
  internal           = true
  load_balancer_type = "network"
  subnets            = aws_subnet.private[*].id

  enable_cross_zone_load_balancing = true

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-nlb"
  })
}

resource "aws_lb_target_group" "tcp" {
  name     = "${var.project_name}-${var.environment}-tcp-tg"
  port     = 3306
  protocol = "TCP"
  vpc_id   = aws_vpc.main.id

  health_check {
    enabled  = true
    protocol = "TCP"
    port     = "traffic-port"
  }
}

resource "aws_lb_listener" "tcp" {
  load_balancer_arn = aws_lb.network.arn
  port              = 3306
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tcp.arn
  }
}
```

### ALB vs NLB

| Feature | ALB (Application) | NLB (Network) |
|---------|-------------------|---------------|
| **Layer** | 7 (HTTP/HTTPS) | 4 (TCP/UDP) |
| **Routing** | Path, Host, Header | Port-based |
| **Performance** | Good | Ultra-high |
| **Static IP** | ❌ | ✅ |
| **SSL Termination** | ✅ | ✅ |
| **WebSocket** | ✅ | ✅ |
| **သင့်တော်** | Web Apps, APIs | Gaming, IoT, DB Proxy |

---

## 9.6 ALB + Auto Scaling Group ပေါင်းစပ်ခြင်း

```hcl
# ===================================
# ASG ကို ALB Target Group နှင့် ချိတ်ဆက်
# ===================================
resource "aws_autoscaling_group" "web" {
  name                = "${var.project_name}-${var.environment}-web-asg"
  desired_capacity    = 2
  min_size            = 2
  max_size            = 6
  vpc_zone_identifier = aws_subnet.private[*].id

  # ALB Target Group ချိတ်ဆက်
  target_group_arns = [aws_lb_target_group.web.arn]

  # Health Check - ELB Type (ALB Health Check သုံး)
  health_check_type         = "ELB"
  health_check_grace_period = 300

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "${var.project_name}-${var.environment}-web"
    propagate_at_launch = true
  }
}

# ALB Request Count Based Scaling
resource "aws_autoscaling_policy" "request_count" {
  name                   = "${var.project_name}-${var.environment}-request-scaling"
  policy_type            = "TargetTrackingScaling"
  autoscaling_group_name = aws_autoscaling_group.web.name

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
      resource_label         = "${aws_lb.web.arn_suffix}/${aws_lb_target_group.web.arn_suffix}"
    }
    target_value = 1000  # Target 1000 requests per instance
  }
}
```

---

## 9.7 လက်တွေ့ - Highly Available Web Application

### Outputs

```hcl
output "alb_dns_name" {
  description = "ALB DNS Name"
  value       = aws_lb.web.dns_name
}

output "alb_zone_id" {
  description = "ALB Zone ID"
  value       = aws_lb.web.zone_id
}

output "target_group_arn" {
  description = "Target Group ARN"
  value       = aws_lb_target_group.web.arn
}

output "app_url" {
  description = "Application URL"
  value       = "https://${var.domain_name}"
}
```

---

## အခန်း (၉) အကျဉ်းချုပ်

- ✅ **ALB** တည်ဆောက်ခြင်း (Application Load Balancer)
- ✅ **Target Groups** နှင့် Health Checks
- ✅ **Listener Rules** - Path-Based & Host-Based Routing
- ✅ **ACM Certificates** - SSL/TLS
- ✅ **NLB** (Network Load Balancer) - Layer 4
- ✅ **ALB + ASG** ပေါင်းစပ်ခြင်း
- ✅ **Request-Based Auto Scaling**

---

*[← အခန်း (၈)](chapter-08.md) | [အခန်း (၁၀) - Route 53 & CloudFront →](chapter-10.md)*
