# အခန်း (၁၃) - Advanced HCL Features

---

## 13.1 `for_each` နှင့် `count` Meta-Arguments

### count

```hcl
# count - Number ဖြင့် Multiple Resources
resource "aws_instance" "web" {
  count = 3

  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"

  tags = {
    Name = "web-server-${count.index + 1}"
  }
}

# count.index = 0, 1, 2
# web-server-1, web-server-2, web-server-3

# Conditional Resource (count ဖြင့် on/off)
resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? 1 : 0

  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
}
```

### for_each (Map/Set)

```hcl
# for_each with Map
variable "instances" {
  default = {
    web = {
      instance_type = "t2.micro"
      subnet_index  = 0
    }
    api = {
      instance_type = "t3.small"
      subnet_index  = 1
    }
    worker = {
      instance_type = "t3.medium"
      subnet_index  = 0
    }
  }
}

resource "aws_instance" "servers" {
  for_each = var.instances

  ami           = data.aws_ami.amazon_linux.id
  instance_type = each.value.instance_type
  subnet_id     = aws_subnet.private[each.value.subnet_index].id

  tags = {
    Name = "${var.project_name}-${each.key}"
    Role = each.key
  }
}

# each.key   = "web", "api", "worker"
# each.value = { instance_type = "t2.micro", subnet_index = 0 }
```

### for_each with Set

```hcl
# for_each with Set (Simple List)
resource "aws_iam_user" "users" {
  for_each = toset(["alice", "bob", "charlie"])

  name = each.key  # each.key == each.value for sets
}
```

### count vs for_each

| Feature | count | for_each |
|---------|-------|----------|
| **Input** | Number | Map / Set |
| **Reference** | `[index]` | `["key"]` |
| **Delete Middle** | ❌ Index Shift | ✅ No Shift |
| **Conditional** | ✅ `0 or 1` | ❌ |
| **သင့်တော်** | Identical Resources | Unique Resources |

```hcl
# count ပြဿနာ - Middle Item ဖျက်ရင် Index Shift ဖြစ်
# count = 3: [0]=a, [1]=b, [2]=c
# b ကို ဖျက်ရင်: [0]=a, [1]=c ← c ကို Destroy+Recreate!

# for_each - Key Based ဖြစ်လို့ ပြဿနာ မရှိ
# {"a"=..., "b"=..., "c"=...}
# "b" ကို ဖျက်ရင်: {"a"=..., "c"=...} ← c မပြောင်းဘူး ✅
```

---

## 13.2 Dynamic Blocks

### Dynamic Block ဆိုတာ ဘာလဲ?

Nested Block များကို **Dynamically Generate** လုပ်ခြင်း ဖြစ်ပါတယ်။

```hcl
# Dynamic Block မသုံးခင် - Repetitive Code
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
}
```

```hcl
# Dynamic Block သုံးပြီး - Clean Code
variable "ingress_rules" {
  default = [
    {
      port        = 80
      description = "HTTP"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      port        = 443
      description = "HTTPS"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      port        = 22
      description = "SSH"
      cidr_blocks = ["10.0.0.0/8"]
    }
  ]
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress_rules

    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## 13.3 Conditional Expressions (Ternary)

```hcl
# Ternary: condition ? true_value : false_value

# Instance Type
instance_type = var.environment == "prod" ? "t3.large" : "t2.micro"

# Multi-AZ
multi_az = var.environment == "prod" ? true : false

# Conditional Resource Creation
resource "aws_nat_gateway" "main" {
  count = var.environment == "prod" ? 2 : 1
  # ...
}

# Conditional Output
output "db_endpoint" {
  value = var.create_database ? aws_db_instance.main[0].endpoint : "N/A"
}

# Nested Conditionals (ရှောင်ပါ - ရှုပ်လွန်းတယ်)
instance_type = (
  var.environment == "prod" ? "t3.large" :
  var.environment == "staging" ? "t3.medium" :
  "t2.micro"
)

# ပိုကောင်းတဲ့ နည်း - Map Lookup
locals {
  instance_types = {
    dev     = "t2.micro"
    staging = "t3.medium"
    prod    = "t3.large"
  }
}

instance_type = local.instance_types[var.environment]
```

---

## 13.4 `for` Expressions

```hcl
# List Transformation
locals {
  names = ["alice", "bob", "charlie"]

  # Upper case list
  upper_names = [for name in local.names : upper(name)]
  # ["ALICE", "BOB", "CHARLIE"]

  # Filtered list
  long_names = [for name in local.names : name if length(name) > 3]
  # ["alice", "charlie"]

  # List → Map
  name_map = { for name in local.names : name => upper(name) }
  # { alice = "ALICE", bob = "BOB", charlie = "CHARLIE" }
}

# Map Transformation
locals {
  instances = {
    web    = { type = "t2.micro", az = "a" }
    api    = { type = "t3.small", az = "b" }
    worker = { type = "t3.medium", az = "a" }
  }

  # Map Values Transform
  instance_types = { for k, v in local.instances : k => v.type }
  # { web = "t2.micro", api = "t3.small", worker = "t3.medium" }

  # Filtered Map
  az_a_instances = {
    for k, v in local.instances : k => v
    if v.az == "a"
  }
  # { web = {type="t2.micro", az="a"}, worker = {type="t3.medium", az="a"} }
}

# for with Resource Outputs
output "instance_ips" {
  value = {
    for k, v in aws_instance.servers : k => v.private_ip
  }
  # { web = "10.0.1.5", api = "10.0.2.10", worker = "10.0.1.15" }
}
```

---

## 13.5 Type Constraints နှင့် Validation Rules

```hcl
# Complex Type Constraints
variable "server_configs" {
  description = "Server configurations"
  type = list(object({
    name          = string
    instance_type = string
    disk_size     = number
    is_public     = bool
    ports         = list(number)
    tags          = map(string)
  }))

  default = [
    {
      name          = "web"
      instance_type = "t2.micro"
      disk_size     = 20
      is_public     = true
      ports         = [80, 443]
      tags          = { Role = "web" }
    }
  ]
}

# Optional Attributes (Terraform 1.3+)
variable "config" {
  type = object({
    name     = string
    port     = optional(number, 8080)  # Default = 8080
    enabled  = optional(bool, true)    # Default = true
    metadata = optional(map(string), {})
  })
}

# Multiple Validations
variable "instance_type" {
  type = string

  validation {
    condition     = can(regex("^t[23]\\.", var.instance_type))
    error_message = "Must be a t2 or t3 instance type."
  }

  validation {
    condition     = !contains(["t2.nano", "t3.nano"], var.instance_type)
    error_message = "Nano instances are not allowed."
  }
}

# Cross-variable Validation (Terraform 1.9+)
variable "min_size" {
  type = number
}

variable "max_size" {
  type = number

  validation {
    condition     = var.max_size >= var.min_size
    error_message = "max_size must be >= min_size."
  }
}
```

---

## 13.6 `depends_on`, `lifecycle` Meta-Arguments

### lifecycle

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  lifecycle {
    # Destroy မလုပ်ခိုင်း
    prevent_destroy = true

    # အသစ်အရင်ဖန်တီး → ပြီးမှ အဟောင်းဖျက်
    create_before_destroy = true

    # ဤ Attributes ပြောင်းလဲရင် Ignore (Terraform Plan ထဲ မပါ)
    ignore_changes = [
      tags,
      ami,              # AMI Update ကို Ignore
      user_data,        # User Data ပြောင်းလဲမှု Ignore
    ]

    # Replace Trigger - ဒီ Value ပြောင်းရင် Resource ကို Replace
    replace_triggered_by = [
      aws_launch_template.web.latest_version
    ]

    # Precondition / Postcondition (Terraform 1.2+)
    precondition {
      condition     = data.aws_ami.amazon_linux.architecture == "x86_64"
      error_message = "AMI must be x86_64 architecture."
    }

    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP."
    }
  }
}
```

---

## 13.7 Provisioners

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"

  # local-exec - Local Machine (Terraform Run ‌ေနတဲ့ Machine) မှာ Run
  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> ip_list.txt"
  }

  # remote-exec - Remote Machine (EC2 Instance) ‌ေပါ်မှာ Run
  provisioner "remote-exec" {
    connection {
      type        = "ssh"
      user        = "ec2-user"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }

    inline = [
      "sudo yum update -y",
      "sudo yum install -y nginx",
      "sudo systemctl start nginx"
    ]
  }

  # Destroy Time Provisioner
  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Instance ${self.id} is being destroyed' >> destroy_log.txt"
  }

  # On Failure
  provisioner "local-exec" {
    command    = "some_command"
    on_failure = continue  # continue or fail
  }
}
```

> **⚠️ သတိ:** Provisioners ကို **နောက်ဆုံးနည်းလမ်း** အဖြစ်သာ သုံးပါ။ User Data, Configuration Management Tools (Ansible, Chef) က ပိုကောင်းပါတယ်။

---

## 13.8 `templatefile()` Function

```hcl
# templates/nginx.conf.tpl
server {
    listen ${port};
    server_name ${domain};

    location / {
        proxy_pass http://${backend_host}:${backend_port};
    }

%{ for location in api_locations ~}
    location ${location.path} {
        proxy_pass http://${location.backend};
    }
%{ endfor ~}
}

# main.tf
resource "aws_instance" "web" {
  user_data = templatefile("${path.module}/templates/nginx.conf.tpl", {
    port         = 80
    domain       = var.domain_name
    backend_host = "10.0.1.100"
    backend_port = 8080
    api_locations = [
      { path = "/api/v1", backend = "10.0.1.101:8081" },
      { path = "/api/v2", backend = "10.0.1.102:8082" }
    ]
  })
}
```

### Template Directives

```
# String Interpolation
${variable_name}

# Conditional
%{ if condition }
  content
%{ endif }

# Loop
%{ for item in list }
  ${item}
%{ endfor }

# Trimming (~ removes newlines)
%{ for item in list ~}
${item}
%{ endfor ~}
```

---

## အခန်း (၁၃) အကျဉ်းချုပ်

- ✅ **for_each vs count** - Multiple Resources ဖန်တီးခြင်း
- ✅ **Dynamic Blocks** - Nested Blocks Generate
- ✅ **Conditional Expressions** - Ternary Operator
- ✅ **for Expressions** - List/Map Transformation
- ✅ **Type Constraints** နှင့် Validation
- ✅ **lifecycle** Meta-Arguments
- ✅ **Provisioners** - local-exec, remote-exec
- ✅ **templatefile()** - Template Rendering

---

*[← အခန်း (၁၂)](chapter-12.md) | [အခန်း (၁၄) - CI/CD Pipeline →](chapter-14.md)*
