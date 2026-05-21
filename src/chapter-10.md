# အခန်း (၁၀) - Route 53 DNS နှင့် CloudFront CDN

---

## 10.1 Hosted Zones ဖန်တီးခြင်း

### Route 53 ဆိုတာ ဘာလဲ?

**Route 53** သည် AWS ၏ DNS (Domain Name System) Service ဖြစ်ပါတယ်။ Domain Name (example.com) ကို IP Address (1.2.3.4) သို့ ပြောင်းပေးခြင်း ဖြစ်ပါတယ်။

```hcl
# ===================================
# Public Hosted Zone
# ===================================
resource "aws_route53_zone" "main" {
  name    = var.domain_name
  comment = "${var.project_name} public hosted zone"

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.domain_name}"
  })
}

# ===================================
# Private Hosted Zone (VPC Internal)
# ===================================
resource "aws_route53_zone" "private" {
  name    = "internal.${var.domain_name}"
  comment = "Private hosted zone for ${var.project_name}"

  vpc {
    vpc_id = aws_vpc.main.id
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-internal"
  })
}
```

---

## 10.2 DNS Records

```hcl
# ===================================
# A Record (ALB Alias)
# ===================================
resource "aws_route53_record" "web" {
  zone_id = aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_lb.web.dns_name
    zone_id                = aws_lb.web.zone_id
    evaluate_target_health = true
  }
}

# www → root domain redirect
resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.${var.domain_name}"
  type    = "A"

  alias {
    name                   = aws_lb.web.dns_name
    zone_id                = aws_lb.web.zone_id
    evaluate_target_health = true
  }
}

# ===================================
# CNAME Record
# ===================================
resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.${var.domain_name}"
  type    = "CNAME"
  ttl     = 300
  records = [aws_lb.web.dns_name]
}

# ===================================
# MX Record (Email)
# ===================================
resource "aws_route53_record" "mx" {
  zone_id = aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "MX"
  ttl     = 300

  records = [
    "10 mail1.example.com",
    "20 mail2.example.com"
  ]
}

# ===================================
# TXT Record (SPF, DKIM, etc.)
# ===================================
resource "aws_route53_record" "txt" {
  zone_id = aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "TXT"
  ttl     = 300

  records = [
    "v=spf1 include:amazonses.com ~all"
  ]
}

# ===================================
# Private Zone Records
# ===================================
resource "aws_route53_record" "db_internal" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "db.internal.${var.domain_name}"
  type    = "CNAME"
  ttl     = 60
  records = [aws_db_instance.mysql.address]
}
```

---

## 10.3 Routing Policies

### Weighted Routing (Traffic ခွဲခြားခြင်း)

```hcl
# 80% Traffic → Primary
resource "aws_route53_record" "weighted_primary" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "app.${var.domain_name}"
  type           = "A"
  set_identifier = "primary"

  weighted_routing_policy {
    weight = 80
  }

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

# 20% Traffic → Canary (New Version)
resource "aws_route53_record" "weighted_canary" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "app.${var.domain_name}"
  type           = "A"
  set_identifier = "canary"

  weighted_routing_policy {
    weight = 20
  }

  alias {
    name                   = aws_lb.canary.dns_name
    zone_id                = aws_lb.canary.zone_id
    evaluate_target_health = true
  }
}
```

### Failover Routing

```hcl
# Health Check
resource "aws_route53_health_check" "primary" {
  fqdn              = "primary.${var.domain_name}"
  port               = 443
  type               = "HTTPS"
  resource_path      = "/health"
  failure_threshold  = 3
  request_interval   = 30

  tags = {
    Name = "primary-health-check"
  }
}

# Primary Record
resource "aws_route53_record" "failover_primary" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "app.${var.domain_name}"
  type           = "A"
  set_identifier = "primary"

  failover_routing_policy {
    type = "PRIMARY"
  }

  health_check_id = aws_route53_health_check.primary.id

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

# Secondary Record (Failover Target)
resource "aws_route53_record" "failover_secondary" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "app.${var.domain_name}"
  type           = "A"
  set_identifier = "secondary"

  failover_routing_policy {
    type = "SECONDARY"
  }

  alias {
    name                   = aws_lb.secondary.dns_name
    zone_id                = aws_lb.secondary.zone_id
    evaluate_target_health = true
  }
}
```

### Latency-Based Routing

```hcl
# Singapore Region
resource "aws_route53_record" "latency_sg" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "app.${var.domain_name}"
  type           = "A"
  set_identifier = "singapore"

  latency_routing_policy {
    region = "ap-southeast-1"
  }

  alias {
    name                   = aws_lb.singapore.dns_name
    zone_id                = aws_lb.singapore.zone_id
    evaluate_target_health = true
  }
}

# US East Region
resource "aws_route53_record" "latency_us" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "app.${var.domain_name}"
  type           = "A"
  set_identifier = "us-east"

  latency_routing_policy {
    region = "us-east-1"
  }

  alias {
    name                   = aws_lb.us_east.dns_name
    zone_id                = aws_lb.us_east.zone_id
    evaluate_target_health = true
  }
}
```

---

## 10.4 Health Checks

```hcl
# ===================================
# Health Checks
# ===================================
resource "aws_route53_health_check" "web" {
  fqdn              = var.domain_name
  port               = 443
  type               = "HTTPS"
  resource_path      = "/health"
  failure_threshold  = 3
  request_interval   = 30
  measure_latency    = true
  regions            = ["us-east-1", "eu-west-1", "ap-southeast-1"]

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-web-health-check"
  })
}

# Health Check Alarm
resource "aws_cloudwatch_metric_alarm" "health_check" {
  alarm_name          = "${var.project_name}-health-check-alarm"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "HealthCheckStatus"
  namespace           = "AWS/Route53"
  period              = 60
  statistic           = "Minimum"
  threshold           = 1
  alarm_description   = "Health check failed"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    HealthCheckId = aws_route53_health_check.web.id
  }
}
```

---

## 10.5 CloudFront Distribution ဖန်တီးခြင်း

### CloudFront ဆိုတာ ဘာလဲ?

**CloudFront** သည် AWS ၏ **CDN (Content Delivery Network)** ဖြစ်ပါတယ်။ ကမ္ဘာတစ်ဝှမ်းရှိ Edge Locations များမှ Content ကို Cache လုပ်ပြီး Users များအနီးဆုံး Location မှ Serve ပေးပါတယ်။

---

## 10.6 CloudFront + S3 Origin

```hcl
# ===================================
# CloudFront + S3 Static Website
# ===================================

# Origin Access Control
resource "aws_cloudfront_origin_access_control" "s3" {
  name                              = "${var.project_name}-s3-oac"
  description                       = "OAC for S3"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# CloudFront Distribution
resource "aws_cloudfront_distribution" "website" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "${var.project_name} website distribution"
  default_root_object = "index.html"
  price_class         = "PriceClass_200"  # Global except South America
  aliases             = [var.domain_name, "www.${var.domain_name}"]

  # S3 Origin
  origin {
    domain_name              = aws_s3_bucket.website.bucket_regional_domain_name
    origin_id                = "S3-${aws_s3_bucket.website.id}"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id
  }

  # Default Cache Behavior
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-${aws_s3_bucket.website.id}"

    forwarded_values {
      query_string = false

      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600     # 1 hour
    max_ttl                = 86400    # 24 hours
    compress               = true
  }

  # Custom Error Responses (SPA)
  custom_error_response {
    error_code         = 404
    response_code      = 200
    response_page_path = "/index.html"
  }

  custom_error_response {
    error_code         = 403
    response_code      = 200
    response_page_path = "/index.html"
  }

  # SSL Certificate
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  # Restrictions
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-cdn"
  })
}

# S3 Bucket Policy for CloudFront
resource "aws_s3_bucket_policy" "website_cdn" {
  bucket = aws_s3_bucket.website.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowCloudFront"
      Effect    = "Allow"
      Principal = { Service = "cloudfront.amazonaws.com" }
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.website.arn}/*"
      Condition = {
        StringEquals = {
          "AWS:SourceArn" = aws_cloudfront_distribution.website.arn
        }
      }
    }]
  })
}

# Route 53 → CloudFront
resource "aws_route53_record" "cdn" {
  zone_id = aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.website.domain_name
    zone_id                = aws_cloudfront_distribution.website.hosted_zone_id
    evaluate_target_health = false
  }
}
```

---

## 10.7 CloudFront + ALB Origin

```hcl
# ===================================
# CloudFront + ALB (Dynamic Content)
# ===================================
resource "aws_cloudfront_distribution" "app" {
  enabled         = true
  is_ipv6_enabled = true
  comment         = "${var.project_name} app distribution"
  aliases         = ["app.${var.domain_name}"]

  # ALB Origin
  origin {
    domain_name = aws_lb.web.dns_name
    origin_id   = "ALB-${aws_lb.web.id}"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  # S3 Origin (Static Assets)
  origin {
    domain_name              = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id                = "S3-assets"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id
  }

  # Default → ALB (Dynamic)
  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "ALB-${aws_lb.web.id}"

    forwarded_values {
      query_string = true
      headers      = ["Host", "Origin", "Authorization"]

      cookies {
        forward = "all"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 0      # No cache for dynamic
    max_ttl                = 0
  }

  # /static/* → S3 (Cached)
  ordered_cache_behavior {
    path_pattern     = "/static/*"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-assets"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 86400   # 1 day
    max_ttl                = 604800  # 7 days
    compress               = true
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  tags = local.common_tags
}
```

---

## 10.8 Outputs

```hcl
output "nameservers" {
  description = "Route53 Nameservers"
  value       = aws_route53_zone.main.name_servers
}

output "cloudfront_domain" {
  description = "CloudFront Distribution Domain"
  value       = aws_cloudfront_distribution.website.domain_name
}

output "cloudfront_distribution_id" {
  description = "CloudFront Distribution ID"
  value       = aws_cloudfront_distribution.website.id
}
```

---

## အခန်း (၁၀) အကျဉ်းချုပ်

- ✅ **Route 53 Hosted Zones** (Public / Private)
- ✅ **DNS Records** (A, CNAME, MX, TXT, Alias)
- ✅ **Routing Policies** (Weighted, Failover, Latency)
- ✅ **Health Checks** နှင့် Alarms
- ✅ **CloudFront + S3** (Static Website CDN)
- ✅ **CloudFront + ALB** (Dynamic Content CDN)

---

*[← အခန်း (၉)](chapter-09.md) | [အခန်း (၁၁) - Terraform Modules →](chapter-11.md)*
