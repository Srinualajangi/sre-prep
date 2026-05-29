# 🔥 INTERVIEW BIBLE V2 — Deep Technical Hands-On Guide
### Your Second Reference for ALL Technical Interviews
### Based on Your Resume @ ASM Technologies | Srinu — Linux & Cloud Infrastructure Engineer
### Created: May 29, 2026

> [!IMPORTANT]
> **How to use this:** This bible goes DEEPER than V1. Every section has **real code you must be able to write or explain on a whiteboard.**
> You have **2.5 days** — follow the study plan at the end. Practice typing these out, not just reading.
> This is your **permanent second reference** for all upcoming interviews.

---

## TABLE OF CONTENTS

1. [Terraform — Writing Real Infrastructure](#1-terraform-deep-dive)
2. [Docker — Writing Dockerfiles & Compose](#2-docker-deep-dive)
3. [Kubernetes — Writing Manifests & Debugging](#3-kubernetes-deep-dive)
4. [CI/CD — Building Complete Pipelines](#4-cicd-deep-dive)
5. [Git — Advanced Workflows & Commands](#5-git-mastery)
6. [Bash Scripting — Production Scripts](#6-bash-scripting)
7. [Python for DevOps — Real Scripts](#7-python-for-devops)
8. [Hybrid Cloud — Architecture & Scenarios](#8-hybrid-cloud-concepts)
9. [🎯 Scenario-Based Questions (The Hard Ones)](#9-scenario-based-questions)
10. [⏰ 2.5-Day Study Plan](#10-study-plan)

---

## 1. TERRAFORM DEEP DIVE

### 🎯 Your Story (Remind Them)
> "I manage 100% IaC using Terraform with reusable modules for VPC, EKS, IAM, RDS, and ALB across dev/QA/prod. All infrastructure changes go through PR review and `terraform plan` in CI before any apply."

---

### 1.1 — Writing a VPC from Scratch (MUST KNOW)

**"Write a Terraform config to create a VPC with public and private subnets"** — This is the #1 most asked Terraform question.

```hcl
# providers.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

```hcl
# variables.tf
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "prod"
}

variable "azs" {
  description = "Availability Zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b", "us-east-1c"]
}
```

```hcl
# main.tf — The Full VPC
# ===== VPC =====
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ===== INTERNET GATEWAY =====
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.environment}-igw"
  }
}

# ===== PUBLIC SUBNETS (one per AZ) =====
resource "aws_subnet" "public" {
  count                   = length(var.azs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)        # 10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.environment}-public-${var.azs[count.index]}"
    Tier = "public"
  }
}

# ===== PRIVATE SUBNETS (one per AZ) =====
resource "aws_subnet" "private" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)  # 10.0.10.0/24, 10.0.11.0/24, 10.0.12.0/24
  availability_zone = var.azs[count.index]

  tags = {
    Name = "${var.environment}-private-${var.azs[count.index]}"
    Tier = "private"
  }
}

# ===== ELASTIC IP for NAT =====
resource "aws_eip" "nat" {
  count  = length(var.azs)
  domain = "vpc"

  tags = {
    Name = "${var.environment}-nat-eip-${count.index}"
  }
}

# ===== NAT GATEWAY (one per AZ for HA) =====
resource "aws_nat_gateway" "main" {
  count         = length(var.azs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "${var.environment}-nat-${var.azs[count.index]}"
  }

  depends_on = [aws_internet_gateway.main]
}

# ===== ROUTE TABLES =====
# Public route table — routes to Internet Gateway
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.environment}-public-rt"
  }
}

# Private route tables — route to NAT Gateway
resource "aws_route_table" "private" {
  count  = length(var.azs)
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "${var.environment}-private-rt-${var.azs[count.index]}"
  }
}

# ===== ROUTE TABLE ASSOCIATIONS =====
resource "aws_route_table_association" "public" {
  count          = length(var.azs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(var.azs)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

```hcl
# outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
```

> [!TIP]
> **`cidrsubnet(var.vpc_cidr, 8, count.index)`** — This is a POWER FUNCTION. Know it.
> `cidrsubnet("10.0.0.0/16", 8, 0)` → `10.0.0.0/24`
> `cidrsubnet("10.0.0.0/16", 8, 1)` → `10.0.1.0/24`
> `cidrsubnet("10.0.0.0/16", 8, 10)` → `10.0.10.0/24`
> The `8` means "add 8 bits to the prefix" (16+8=24), and the last number is the subnet index.

---

### 1.2 — Writing an EC2 Instance with Security Group

```hcl
# Security Group — Allow SSH + HTTP
resource "aws_security_group" "web" {
  name        = "${var.environment}-web-sg"
  description = "Allow SSH and HTTP"
  vpc_id      = aws_vpc.main.id

  # SSH from specific IP only (YOUR IP)
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["203.0.113.0/32"]  # Your office IP
  }

  # HTTP from ALB security group
  ingress {
    description     = "HTTP from ALB"
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  # All outbound allowed
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.environment}-web-sg"
  }
}

# EC2 Instance
resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.medium"
  subnet_id              = aws_subnet.private[0].id
  vpc_security_group_ids = [aws_security_group.web.id]
  key_name               = var.key_name
  iam_instance_profile   = aws_iam_instance_profile.web.name

  root_block_device {
    volume_size = 30
    volume_type = "gp3"
    encrypted   = true
  }

  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>Hello from $(hostname)</h1>" > /var/www/html/index.html
  EOF

  tags = {
    Name        = "${var.environment}-web-01"
    Environment = var.environment
    Role        = "webserver"
  }
}

# Data source — latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

---

### 1.3 — Writing an S3 Bucket with Policies

```hcl
resource "aws_s3_bucket" "app_data" {
  bucket = "${var.environment}-app-data-${data.aws_caller_identity.current.account_id}"

  tags = {
    Name        = "${var.environment}-app-data"
    Environment = var.environment
  }
}

# Enable versioning
resource "aws_s3_bucket_versioning" "app_data" {
  bucket = aws_s3_bucket.app_data.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Enable server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "app_data" {
  bucket = aws_s3_bucket.app_data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

# Block all public access
resource "aws_s3_bucket_public_access_block" "app_data" {
  bucket = aws_s3_bucket.app_data.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle rule — move to IA after 90 days, Glacier after 180 days
resource "aws_s3_bucket_lifecycle_configuration" "app_data" {
  bucket = aws_s3_bucket.app_data.id

  rule {
    id     = "archive-old-data"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 180
      storage_class = "GLACIER"
    }

    expiration {
      days = 365
    }
  }
}

data "aws_caller_identity" "current" {}
```

---

### 1.4 — Writing an RDS Instance

```hcl
resource "aws_db_subnet_group" "main" {
  name       = "${var.environment}-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id

  tags = {
    Name = "${var.environment}-db-subnet-group"
  }
}

resource "aws_security_group" "rds" {
  name   = "${var.environment}-rds-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description     = "MySQL from app tier"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]  # Only app tier can access
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_db_instance" "main" {
  identifier     = "${var.environment}-mysql"
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.t3.medium"

  allocated_storage     = 50
  max_allocated_storage = 100   # Auto-scaling storage
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = "appdb"
  username = "admin"
  password = var.db_password   # From terraform.tfvars or Secrets Manager

  multi_az               = true   # HA — standby in another AZ
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:04:00-sun:05:00"

  skip_final_snapshot       = false
  final_snapshot_identifier = "${var.environment}-mysql-final"

  tags = {
    Name        = "${var.environment}-mysql"
    Environment = var.environment
  }
}
```

---

### 1.5 — Writing a Terraform Module (CRITICAL SKILL)

**They WILL ask: "How do you structure a Terraform module?"**

```
modules/
└── ec2-webserver/
    ├── main.tf          # Resource definitions
    ├── variables.tf     # Input variables
    ├── outputs.tf       # Output values
    └── README.md        # Usage documentation
```

```hcl
# modules/ec2-webserver/variables.tf
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.medium"

  validation {
    condition     = contains(["t3.small", "t3.medium", "t3.large", "m5.large"], var.instance_type)
    error_message = "Instance type must be one of: t3.small, t3.medium, t3.large, m5.large"
  }
}

variable "environment" {
  type = string
}

variable "subnet_id" {
  type = string
}

variable "security_group_ids" {
  type = list(string)
}

variable "instance_count" {
  type    = number
  default = 1
}
```

```hcl
# modules/ec2-webserver/main.tf
resource "aws_instance" "web" {
  count                  = var.instance_count
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.security_group_ids

  tags = {
    Name = "${var.environment}-web-${count.index + 1}"
  }
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

```hcl
# modules/ec2-webserver/outputs.tf
output "instance_ids" {
  value = aws_instance.web[*].id
}

output "private_ips" {
  value = aws_instance.web[*].private_ip
}
```

```hcl
# Calling the module from environments/prod/main.tf
module "web_servers" {
  source = "../../modules/ec2-webserver"

  instance_type      = "m5.large"
  instance_count     = 3
  environment        = "prod"
  subnet_id          = module.vpc.private_subnet_ids[0]
  security_group_ids = [aws_security_group.web.id]
}

# Accessing module outputs
output "web_server_ips" {
  value = module.web_servers.private_ips
}
```

---

### 1.6 — Terraform Functions You Must Know

```hcl
# ===== STRING FUNCTIONS =====
upper("hello")                          # "HELLO"
lower("HELLO")                          # "hello"
format("Hello, %s!", "Srinu")           # "Hello, Srinu!"
join(", ", ["a", "b", "c"])             # "a, b, c"
split(",", "a,b,c")                     # ["a", "b", "c"]
replace("hello world", "world", "tf")   # "hello tf"
trimspace("  hello  ")                  # "hello"

# ===== COLLECTION FUNCTIONS =====
length(["a", "b", "c"])                 # 3
element(["a", "b", "c"], 1)             # "b"
contains(["a", "b", "c"], "b")          # true
flatten([["a","b"], ["c"]])             # ["a", "b", "c"]
merge({a=1}, {b=2})                     # {a=1, b=2}
lookup({a=1, b=2}, "a", 0)             # 1
keys({a=1, b=2})                        # ["a", "b"]
values({a=1, b=2})                      # [1, 2]
zipmap(["a","b"], [1,2])                # {a=1, b=2}

# ===== NUMERIC FUNCTIONS =====
max(5, 12, 9)                           # 12
min(5, 12, 9)                           # 5
ceil(4.3)                               # 5
floor(4.9)                              # 4

# ===== NETWORK FUNCTIONS =====
cidrsubnet("10.0.0.0/16", 8, 0)         # "10.0.0.0/24"
cidrhost("10.0.1.0/24", 5)              # "10.0.1.5"
cidrnetmask("10.0.0.0/16")              # "255.255.0.0"

# ===== CONDITIONAL =====
var.environment == "prod" ? 3 : 1        # Ternary operator

# ===== for_each vs count =====
# count — use when instances are identical
resource "aws_instance" "web" {
  count         = 3
  instance_type = "t3.medium"
  tags = { Name = "web-${count.index + 1}" }
}

# for_each — use when instances differ (preferred)
variable "instances" {
  default = {
    web    = "t3.medium"
    api    = "t3.large"
    worker = "m5.xlarge"
  }
}

resource "aws_instance" "app" {
  for_each      = var.instances
  instance_type = each.value
  tags = { Name = each.key }
}
```

---

### 1.7 — Terraform Interview Q&A (Deep)

**Q: "What is `terraform taint` and when would you use it?"**
> "`terraform taint` marks a resource for destruction and recreation on the next `apply`. I use it when a resource is in a bad state — for example, an EC2 instance that's running but the user_data didn't execute properly. In newer Terraform versions (1.5+), `taint` is replaced by `terraform apply -replace=aws_instance.web` which is more explicit."

**Q: "Explain `count` vs `for_each`. When do you use which?"**
> "`count` creates resources by index number — `[0]`, `[1]`, `[2]`. The problem: if you delete index 1, resources at index 2+ get shuffled, causing unnecessary destroy/recreate. `for_each` creates resources by key — each resource is identified by a stable name, not a number. If you remove one, others are unaffected. **I always use `for_each` when resources might be added/removed independently.** I only use `count` for identical resources or conditional creation (`count = var.create_resource ? 1 : 0`)."

**Q: "How do you import existing AWS resources into Terraform?"**
> "Step 1: Write the resource block in your `.tf` file matching the existing resource's config. Step 2: Run `terraform import aws_instance.web i-0abc123def`. This adds the resource to state. Step 3: Run `terraform plan` to verify no changes are detected — meaning your config matches reality. If there are diffs, adjust your config. In Terraform 1.5+, you can use `import` blocks directly in HCL:
```hcl
import {
  to = aws_instance.web
  id = "i-0abc123def"
}
```
> Then run `terraform plan` with `-generate-config-out=generated.tf` to auto-generate the config."

**Q: "What are `data` sources in Terraform?"**
> "Data sources let you READ existing infrastructure that Terraform doesn't manage. For example, I use `data.aws_ami` to fetch the latest Amazon Linux AMI ID, `data.aws_vpc` to reference an existing VPC, or `data.aws_caller_identity` to get the current AWS account ID. Data sources are read-only — they never create or modify resources."

**Q: "How do you handle secrets in Terraform?"**
> "Never hardcode secrets in `.tf` files. I use three approaches: (1) `terraform.tfvars` files that are in `.gitignore`, (2) Environment variables like `TF_VAR_db_password`, (3) For production, I use `data.aws_secretsmanager_secret_version` to fetch secrets from AWS Secrets Manager at plan/apply time. The state file also contains sensitive values, so we encrypt it at rest in S3."

**Q: "What is a `lifecycle` block?"**
> "It controls resource behavior:
```hcl
lifecycle {
  create_before_destroy = true   # Create new before destroying old (zero-downtime)
  prevent_destroy       = true   # Prevent accidental deletion (use for RDS, S3)
  ignore_changes        = [tags] # Ignore changes made outside Terraform
}
```
> I use `prevent_destroy` on production databases and S3 buckets. I use `create_before_destroy` on resources behind load balancers for zero-downtime replacements."

**Q: "Explain Terraform workspaces"**
> "Workspaces let you manage multiple environments with the same configuration but separate state files. `terraform workspace new dev` creates a dev workspace. I can reference the workspace name with `terraform.workspace` in my config — for example, setting `instance_type = terraform.workspace == \"prod\" ? \"m5.large\" : \"t3.medium\"`. However, in my current setup at ASM, we use **separate directories** per environment instead of workspaces — it gives better isolation and clearer PR reviews."

---

## 2. DOCKER DEEP DIVE

### 🎯 Your Story
> "I containerized 5 legacy VM-based applications into Docker containers for our Kubernetes migration. I write multi-stage Dockerfiles to keep images small, use docker-compose for local development, and push images to ECR/ACR for production deployments."

---

### 2.1 — Writing Dockerfiles (You WILL Be Asked to Write One)

#### Python Flask Application

```dockerfile
# ===== Stage 1: Builder =====
FROM python:3.11-slim AS builder

WORKDIR /app

# Copy only requirements first (layer caching!)
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Copy application code
COPY . .

# ===== Stage 2: Runtime =====
FROM python:3.11-slim AS runtime

# Security: don't run as root
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /root/.local /root/.local
COPY --from=builder /app .

# Make sure scripts in .local are usable
ENV PATH=/root/.local/bin:$PATH

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:5000/health || exit 1

# Switch to non-root user
USER appuser

# Run the application
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "app:app"]
```

#### Node.js Application

```dockerfile
# ===== Stage 1: Build =====
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files first (layer caching)
COPY package*.json ./
RUN npm ci --only=production

# Copy source
COPY . .

# ===== Stage 2: Runtime =====
FROM node:20-alpine AS runtime

# Security
RUN addgroup -S appgroup && adduser -S appuser -G appgrou           │    let me add socket.settimeout(1)"         │
           │  - Add service name resolution              │
           │  - Add argparse for CLI                     │
           │  - Test it: scan localhost, scan a server   │
           │  - "Broooo it works! Look — port 22 SSH,    │
           │    port 80 HTTP, port 443 HTTPS"            │
           └─────────────────────────────────────────────┘

9:45 PM — 🖥️ SCREEN RECORD STOP

9:45 PM — 🔴 PHONE RECORD (closing for the day, face cam, 2 min)
           "Day 17 done. Built a port scanner from scratch.
            Connected it to the networking tools I learned.
            Tomorrow is Day 18 — disk I/O deep dive and 3Sum 
            which everyone says is a nightmare. We'll see.
            If you're also grinding, comment your day count below."
9:47 PM — 🔴 STOP

10:00 PM — Push code to GitHub. Done.
```

---

## 🎬 What You Now Have (Raw Material)

After this ONE day, sitting on your hard drive:

| Raw Clip | Duration | Effort to Record |
|---|---|---|
| Morning face intro | ~2 min | 0 extra effort |
| LeetCode screen recording (with voice) | ~30 min | 0 extra (you were solving anyway) |
| Quick LeetCode reaction | ~30 sec | 10 seconds setup |
| Phone notes from work | Text notes | 30 sec total |
| Linux networking study (screen + face) | ~60 min | 0 extra (you were studying anyway) |
| Python port scanner build (screen + face) | ~70 min | 0 extra (you were coding anyway) |
| Night closing face cam | ~2 min | 0 extra effort |
| GitHub commit | Code | 0 extra (you were pushing anyway) |

**Total extra time spent on content: ~5 minutes** (just pressing record and the two face cam clips)

---

## ✂️ How to Edit This Into Content (Weekend Batch)

> [!TIP]
> You DON'T edit daily. You batch edit on the weekend. Saturday morning = editing time.

### 📹 Content Piece 1: Main YouTube Video (8-12 min)

**Title:** `"Day 17: I Built a Port Scanner & Learned Linux Networking | Linux Admin → SRE"`

**Editing Roadmap (Cut from raw footage):**

```
TIMELINE OF FINAL VIDEO:

[0:00 - 0:30] HOOK
  Cut from morning face cam:
  "Day 17 of my FAANG prep. Today I built a port scanner 
   from scratch and realized I've been using the wrong 
   networking tool for 2 years."

[0:30 - 1:00] QUICK CONTEXT
  "I'm a Linux admin trying to become an SRE at FAANG.
   This is my daily grind. Day 17."
  Show the roadmap briefly on screen.

[1:00 - 4:00] LINUX NETWORKING TOOLS (condensed)
  Cut the 60-min study recording down to 3 minutes:
  - Best moment of ss command demo (~40 sec)
  - ss vs netstat speed comparison (~30 sec)  
  - ip route show moment (~30 sec)
  - iptables "my senior did this at work" story (~40 sec)
  - The "I feel stupid" real talk moment (~30 sec)

[4:00 - 4:30] TRANSITION
  "Now let me build something real with this."

[4:30 - 7:30] PORT SCANNER BUILD (condensed)
  Cut the 70-min build down to 3 minutes:
  - Show the starting empty file → "here's the plan" (~20 sec)
  - Fast-forward typing (speed up boring parts 4x)
  - Keep the error moments! "wait, timeout..." (~30 sec)
  - The "IT WORKS" moment — show terminal output (~40 sec)
  - Walk through the final code briefly (~60 sec)

[7:30 - 8:30] LEETCODE MOMENT
  "Oh, I also solved my first Medium today."
  Show the key insight from Top K Frequent Elements (~60 sec)
  "Heap. That's it. Don't overcomplicate."

[8:30 - 9:30] REAL TALK + CLOSE
  Night face cam:
  "Day 17 done. I feel good. This networking stuff 
   connects directly to my job. Tomorrow is I/O and 3Sum.
   Comment your day count. See you tomorrow."

[9:30 - 9:45] END SCREEN
  Subscribe + next video card
```

> [!NOTE]
> **Editing time for this: ~45-60 minutes** using DaVinci Resolve (free).  
> Mostly just cutting clips, adding text overlays, and speed-ramping the coding parts.

---

### 📱 Content Piece 2: YouTube Short / Instagram Reel (30-60 sec)

**Title:** `"ss vs netstat — I've been using the WRONG tool 😤"`

```
SCRIPT (speak to phone camera or voiceover on screen recording):

[0:00] "I've been using netstat for 2 years."
[0:03] Show terminal: netstat -tulnp (takes a moment to load)
[0:08] "Then I discovered ss."
[0:10] Show terminal: ss -tulnp (instant output)
[0:14] "It's literally faster because it reads directly 
        from kernel netlink sockets instead of parsing /proc."
[0:20] Show side-by-side comparison
[0:25] "ss -s gives you a complete socket summary in one line."
[0:30] Show the output
[0:33] "2 years. I was using the slow one for 2 years."
[0:37] Cut to face: "What tool have YOU been using wrong?"

Total: 40 seconds. Done.
```

**This short can be clipped DIRECTLY from your study recording. Extra effort: 10 minutes.**

---

### 📱 Content Piece 3: Second YouTube Short (30 sec)

**Title:** `"I built a port scanner in Python in 30 minutes 🔥"`

```
[0:00] Show the empty file
[0:02] Speed up the entire build process (8x speed with background music)
[0:20] Show the final terminal output — ports discovered
[0:25] "From zero to a working port scanner. 
        Day 17 of FAANG prep. Follow for daily builds."
[0:30] End

Extra effort: 5 minutes to cut this from existing footage.
```

---

### 📝 Content Piece 4: LinkedIn Post (Write in 5 min)

```
📍 Day 17/100 — Linux Admin → SRE at FAANG

Today I learned something embarrassing.

I've been using `netstat` for 2+ years at work.
Turns out `ss` does the same thing — but faster.

Why? netstat parses /proc/net files.
ss uses kernel netlink sockets directly.

Then I built a port scanner in Python using what I learned:
✅ Scans port ranges
✅ Resolves service names  
✅ CLI with argparse

The best way to learn tools? Build with them.

Code: [GitHub link]

Day 18 tomorrow: Disk I/O + 3Sum (the LeetCode problem 
everyone warns you about 😅)

#SRE #Linux #Python #100DaysOfCode #FAANG
---
```

**Time to write this: 5 minutes. You literally just summarize what you did.**

---

### 📝 Content Piece 5: Twitter/X Thread (Optional, 3 min)

```
🧵 Day 17/100 of my SRE prep. Thread:

1/ Today's topic: Linux networking tools 
   that every SRE MUST know.

2/ ss (Socket Statistics)
   - ss -tulnp → all listening ports
   - ss -s → socket summary
   - It's FASTER than netstat. Uses netlink, not /proc.

3/ ip command (replaced ifconfig)
   - ip addr show → your IPs
   - ip route show → routing table
   - ip link show → interface status

4/ iptables
   - iptables -L -n → list firewall rules
   - Used by my senior today to block a malicious IP

5/ Built a Python port scanner to tie it all together.
   [screenshot of terminal output]

6/ LeetCode: Solved Top K Frequent Elements (#347)
   Key insight: Heap. Don't sort everything.

Day 18 tomorrow → I/O deep dive 🔥
```

---

## 📊 Summary: One Day → Multiple Content Pieces

| Content Piece | Platform | Extra Time Needed | When to Publish |
|---|---|---|---|
| Main Video (8-12 min) | YouTube | 45-60 min editing (weekend) | Sunday/Monday |
| Short #1: ss vs netstat | YouTube Shorts / Reels | 10 min | Same day or next day |
| Short #2: Port scanner timelapse | YouTube Shorts / Reels | 5 min | Day after |
| LinkedIn Post | LinkedIn | 5 min writing | Same night |
| Twitter Thread | X/Twitter | 3 min writing | Same night |

### Total Extra Daily Time: ~10-15 minutes
### Weekend Editing Session: ~2-3 hours (for the full week's main videos)

---

## 🎥 Recording Setup (Zero Budget)

```
What you need:
├── Phone (you already have one)
│   └── Mount it on a ₹300 tripod or stack of books
├── OBS Studio (FREE)
│   └── Screen recording + optional small face cam overlay
├── DaVinci Resolve (FREE)
│   └── Video editing (weekend batch)
└── Canva (FREE tier)
    └── Thumbnails (5 min each)
```

### OBS Settings for Study Recording
```
Video:  1920x1080, 30fps
Audio:  Your laptop mic is fine to start
Layout: Full screen capture + small webcam circle in corner
Output: MKV format (doesn't corrupt if OBS crashes)
        Convert to MP4 after recording: File → Remux
```

---

## 🖼️ Thumbnail Strategy

Keep it **dead simple and consistent** — a recognizable template:

```
┌──────────────────────────────────────┐
│                                      │
│  [Your face     ]   DAY 17          │
│  [with reaction ]   ───────          │
│                     PORT SCANNER     │
│                     IN PYTHON 🐍     │
│                                      │
│  Terminal screenshot in background   │
│                                      │
│  "Linux Admin → SRE" badge          │
└──────────────────────────────────────┘

Colors: Dark background + bright accent (cyan/green terminal colors)
Font: Bold, 2-3 words max
Face: Surprised/excited expression
```

---

## 🧠 The Mindset Shift

```
❌ OLD THINKING:
   "Study for 4 hours" → exhausted → nothing to show → feel drained

✅ NEW THINKING:  
   "Study for 4 hours" → press record → same learning → 
   now you have a video, 2 shorts, a LinkedIn post, and a GitHub commit
   → feel PRODUCTIVE → energy goes UP
```

> [!IMPORTANT]
> **You're not adding work. You're adding a RECORD button to work you're already doing.**
> 
> The studying happens regardless. The only question is: do you press record or not?

---

## 🚫 What NOT To Do

| Don't | Why |
|---|---|
| Don't wait for perfect setup | Your phone + OBS is enough to start |
| Don't script everything | Raw, real reactions ARE the content |
| Don't edit every day | Batch edit on weekends |
| Don't compare to big YouTubers | They have teams. You have authenticity |
| Don't aim for viral | Aim for consistent. 2 videos/week > 1 viral/month |
| Don't hide mistakes | Errors, bugs, confusion = engagement gold |
| Don't wait to "know enough" | "I'm learning this live" IS the value proposition |

---

## 📅 Weekly Content Schedule

| Day | Content Activity | Time |
|---|---|---|
| Mon-Fri | Press record during study. Quick face cam clips. LinkedIn post. | ~10-15 min/day extra |
| Saturday | Batch edit week's footage → 2-3 main YouTube videos + 3-4 shorts | 2-3 hours |
| Sunday | Upload videos. Schedule posts. Plan next week's thumbnails. | 1-2 hours |

**Weekly total content creation time: ~5-6 hours**  
**But ~4 of those hours overlap with study time you were doing anyway.**  
**Net extra time: ~2-3 hours/week**

---

*This template is built around Day 17, but the structure works for ANY day in your 100-day roadmap. The topic changes, the format stays the same.*
