# Chapter 9: The Skeletal System (Infrastructure as Code)

## Introduction: Infrastructure You Can Version, Test, and Deploy

Just as a skeleton provides structure, support, and a framework for an organism, Infrastructure as Code (IaC) provides the foundational structure for your infrastructure - defined, versioned, and managed through code rather than manual processes.

**What You'll Learn:**
- Infrastructure as Code principles and benefits
- Terraform for cloud resource provisioning
- Configuration management patterns
- GitOps workflows for infrastructure
- Testing and validation strategies
- Disaster recovery and reproducibility
- Security and compliance as code

**Real-World Scenario:**

Your infrastructure needs:
- Reproducible environments (dev, staging, production)
- Version control for infrastructure changes
- Automated provisioning and deployment
- Rollback capability for failures
- Compliance and security enforcement
- Multi-cloud or multi-region deployment
- Self-documentation

This chapter provides production-ready solutions using Infrastructure as Code.

---

## The Evolution of Infrastructure Management

### Phase 1: Manual Configuration (2000s)

```
Infrastructure Setup Process:

1. Request server from IT (wait 2 weeks)
2. SSH into server
3. Manually install packages:
   $ apt-get install nginx
   $ apt-get install postgresql
4. Manually edit configuration files
5. Manually configure firewall rules
6. Write down what you did (maybe)
7. Hope you remember for next time

Problems:
❌ No reproducibility
❌ No version control
❌ Configuration drift (servers diverge over time)
❌ No testing before production
❌ Knowledge lives in people's heads
❌ Slow disaster recovery
```

### Phase 2: Configuration Management (2010s)

```
Tools: Puppet, Chef, Ansible

Ansible Playbook:
---
- hosts: webservers
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
    - name: Start nginx
      service:
        name: nginx
        state: started

Improvements:
✅ Repeatable configuration
✅ Version controlled
✅ Idempotent (safe to run multiple times)

Limitations:
⚠️ Assumes servers already exist
⚠️ Doesn't handle cloud resources
⚠️ State management challenges
```

### Phase 3: Infrastructure as Code (2015+)

```
Tools: Terraform, CloudFormation, Pulumi

Terraform Configuration:
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }
}

Advantages:
✅ Define entire infrastructure
✅ Declarative (describe desired state)
✅ State management (tracks reality)
✅ Plan before apply (preview changes)
✅ Cloud-agnostic (multi-cloud)
```

### Phase 4: GitOps (2020+)

```
Git as Single Source of Truth:

┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│   Git Repo   │──────>│  CI/CD       │──────>│ Infrastructure│
│              │       │  Pipeline    │       │               │
│  Terraform   │       │  (Auto)      │       │  AWS/GCP/K8s │
│  K8s YAML    │       │              │       │               │
└──────────────┘       └──────────────┘       └──────────────┘
       │
       │ Pull Request
       ▼
   Code Review + Tests

Modern Advantages:
✅ Full audit trail (every change in git)
✅ Automated deployment
✅ Easy rollback (git revert)
✅ Peer review (pull requests)
✅ Testing in pipeline
✅ Self-documenting
```

---

## Core IaC Principles

### Principle 1: Declarative Over Imperative

```
Imperative (How to do it):
1. Create VPC
2. Wait for VPC
3. Create subnet in VPC
4. Wait for subnet
5. Create EC2 instance in subnet
6. Configure security group
... (sequence matters!)

Declarative (What you want):
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "web" {
  ami           = "ami-123456"
  subnet_id     = aws_subnet.public.id
  instance_type = "t3.micro"
}

# Tool figures out the order!
# Dependencies handled automatically!
```

### Principle 2: Idempotent Operations

```
Idempotent: Running multiple times = same result

# First run: Creates database
terraform apply
# Creates: database-prod

# Second run: No changes
terraform apply
# Output: "No changes. Infrastructure is up-to-date."

# Third run: Still no changes
terraform apply
# Output: "No changes. Infrastructure is up-to-date."

Safe to run repeatedly!
```

### Principle 3: Immutable Infrastructure

```
Old Way (Mutable):
Server A → Update in place → Server A' (modified)
Problems:
- Configuration drift
- Hard to debug
- Can't rollback easily

New Way (Immutable):
Server A → Deploy Server B → Delete Server A
Benefits:
✅ No configuration drift
✅ Easy rollback (keep old version)
✅ Predictable behavior
```

### Principle 4: Version Everything

```
.git/
├── infrastructure/
│   ├── terraform/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── kubernetes/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── ansible/
│       └── playbook.yml

Every change tracked:
- Who changed what?
- When did they change it?
- Why did they change it? (commit message)
- Can revert to any previous state
```

---

## Building Block 1: Terraform

Terraform is the industry-standard tool for infrastructure provisioning.

### Terraform Architecture

```
┌─────────────────────────────────────────┐
│         Terraform Workflow              │
│                                         │
│  1. Write Configuration (.tf files)    │
│          ↓                              │
│  2. terraform init                      │
│     (Download providers)                │
│          ↓                              │
│  3. terraform plan                      │
│     (Preview changes)                   │
│          ↓                              │
│  4. terraform apply                     │
│     (Execute changes)                   │
│          ↓                              │
│  5. terraform.tfstate                   │
│     (Track current state)               │
└─────────────────────────────────────────┘

Providers:
├── AWS
├── GCP
├── Azure
├── Kubernetes
├── Docker
├── GitHub
└── 3000+ others
```

### Complete AWS Infrastructure Example

**Directory Structure:**

```
infrastructure/
├── main.tf           # Main configuration
├── variables.tf      # Input variables
├── outputs.tf        # Output values
├── terraform.tfvars  # Variable values
├── modules/          # Reusable modules
│   ├── vpc/
│   ├── ec2/
│   └── rds/
└── environments/     # Per-environment configs
    ├── dev/
    ├── staging/
    └── production/
```

**main.tf:**

```hcl
# Configure Terraform
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # Store state in S3 (recommended for teams)
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# Configure AWS Provider
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "Terraform"
      Project     = var.project_name
    }
  }
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}

# Public Subnets
resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project_name}-public-${var.availability_zones[count.index]}"
    Type = "public"
  }
}

# Private Subnets
resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 100)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.project_name}-private-${var.availability_zones[count.index]}"
    Type = "private"
  }
}

# NAT Gateway (for private subnets to access internet)
resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"

  tags = {
    Name = "${var.project_name}-nat-eip-${count.index + 1}"
  }
}

resource "aws_nat_gateway" "main" {
  count = length(var.availability_zones)

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "${var.project_name}-nat-${count.index + 1}"
  }

  depends_on = [aws_internet_gateway.main]
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

resource "aws_route_table" "private" {
  count  = length(var.availability_zones)
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "${var.project_name}-private-rt-${count.index + 1}"
  }
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count = length(var.availability_zones)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count = length(var.availability_zones)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# Security Groups
resource "aws_security_group" "web" {
  name_prefix = "${var.project_name}-web-"
  vpc_id      = aws_vpc.main.id
  description = "Security group for web servers"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow HTTP"
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow HTTPS"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }

  tags = {
    Name = "${var.project_name}-web-sg"
  }

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_security_group" "database" {
  name_prefix = "${var.project_name}-db-"
  vpc_id      = aws_vpc.main.id
  description = "Security group for database"

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]
    description     = "Allow PostgreSQL from web servers"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }

  tags = {
    Name = "${var.project_name}-db-sg"
  }

  lifecycle {
    create_before_destroy = true
  }
}

# RDS PostgreSQL
resource "aws_db_subnet_group" "main" {
  name       = "${var.project_name}-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id

  tags = {
    Name = "${var.project_name}-db-subnet-group"
  }
}

resource "aws_db_instance" "main" {
  identifier        = "${var.project_name}-db"
  engine            = "postgres"
  engine_version    = "15.3"
  instance_class    = var.db_instance_class
  allocated_storage = var.db_allocated_storage
  storage_encrypted = true

  db_name  = var.db_name
  username = var.db_username
  password = var.db_password

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.database.id]

  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "mon:04:00-mon:05:00"

  skip_final_snapshot       = var.environment != "production"
  final_snapshot_identifier = "${var.project_name}-final-snapshot-${formatdate("YYYY-MM-DD-hhmm", timestamp())}"

  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  tags = {
    Name = "${var.project_name}-db"
  }
}

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "${var.project_name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.web.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = var.environment == "production"
  enable_http2              = true
  enable_cross_zone_load_balancing = true

  tags = {
    Name = "${var.project_name}-alb"
  }
}

resource "aws_lb_target_group" "main" {
  name     = "${var.project_name}-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200"
  }

  deregistration_delay = 30

  tags = {
    Name = "${var.project_name}-tg"
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
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

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = var.ssl_certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.main.arn
  }
}

# Launch Template for EC2 instances
resource "aws_launch_template" "app" {
  name_prefix   = "${var.project_name}-"
  image_id      = var.ami_id
  instance_type = var.instance_type

  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = base64encode(templatefile("${path.module}/user-data.sh", {
    db_host     = aws_db_instance.main.endpoint
    db_name     = var.db_name
    db_username = var.db_username
    db_password = var.db_password
  }))

  iam_instance_profile {
    name = aws_iam_instance_profile.app.name
  }

  monitoring {
    enabled = true
  }

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
  }

  tag_specifications {
    resource_type = "instance"

    tags = {
      Name = "${var.project_name}-app"
    }
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "app" {
  name                = "${var.project_name}-asg"
  vpc_zone_identifier = aws_subnet.private[*].id
  target_group_arns   = [aws_lb_target_group.main.arn]
  health_check_type   = "ELB"
  health_check_grace_period = 300

  min_size         = var.asg_min_size
  max_size         = var.asg_max_size
  desired_capacity = var.asg_desired_capacity

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  enabled_metrics = [
    "GroupDesiredCapacity",
    "GroupInServiceInstances",
    "GroupMaxSize",
    "GroupMinSize",
    "GroupPendingInstances",
    "GroupStandbyInstances",
    "GroupTerminatingInstances",
    "GroupTotalInstances",
  ]

  tag {
    key                 = "Name"
    value               = "${var.project_name}-app"
    propagate_at_launch = true
  }
}

# Auto Scaling Policies
resource "aws_autoscaling_policy" "scale_up" {
  name                   = "${var.project_name}-scale-up"
  autoscaling_group_name = aws_autoscaling_group.app.name
  adjustment_type        = "ChangeInCapacity"
  scaling_adjustment     = 2
  cooldown              = 300
}

resource "aws_autoscaling_policy" "scale_down" {
  name                   = "${var.project_name}-scale-down"
  autoscaling_group_name = aws_autoscaling_group.app.name
  adjustment_type        = "ChangeInCapacity"
  scaling_adjustment     = -1
  cooldown              = 300
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "${var.project_name}-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 70

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.app.name
  }

  alarm_actions = [aws_autoscaling_policy.scale_up.arn]
}

resource "aws_cloudwatch_metric_alarm" "cpu_low" {
  alarm_name          = "${var.project_name}-cpu-low"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 30

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.app.name
  }

  alarm_actions = [aws_autoscaling_policy.scale_down.arn]
}

# IAM Role for EC2 instances
resource "aws_iam_role" "app" {
  name_prefix = "${var.project_name}-app-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "app_ssm" {
  role       = aws_iam_role.app.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "app" {
  name_prefix = "${var.project_name}-app-"
  role        = aws_iam_role.app.name
}
```

**variables.tf:**

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}

variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}

variable "project_name" {
  description = "Project name"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
  default     = ["us-west-2a", "us-west-2b", "us-west-2c"]
}

variable "db_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.t3.micro"
}

variable "db_allocated_storage" {
  description = "RDS allocated storage (GB)"
  type        = number
  default     = 20
}

variable "db_name" {
  description = "Database name"
  type        = string
}

variable "db_username" {
  description = "Database username"
  type        = string
  sensitive   = true
}

variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}

variable "ami_id" {
  description = "AMI ID for EC2 instances"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "asg_min_size" {
  description = "Auto Scaling Group minimum size"
  type        = number
  default     = 2
}

variable "asg_max_size" {
  description = "Auto Scaling Group maximum size"
  type        = number
  default     = 10
}

variable "asg_desired_capacity" {
  description = "Auto Scaling Group desired capacity"
  type        = number
  default     = 2
}

variable "ssl_certificate_arn" {
  description = "ARN of SSL certificate"
  type        = string
}
```

**outputs.tf:**

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "alb_dns_name" {
  description = "ALB DNS name"
  value       = aws_lb.main.dns_name
}

output "alb_zone_id" {
  description = "ALB zone ID"
  value       = aws_lb.main.zone_id
}

output "database_endpoint" {
  description = "Database endpoint"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}

output "asg_name" {
  description = "Auto Scaling Group name"
  value       = aws_autoscaling_group.app.name
}
```

**terraform.tfvars:**

```hcl
aws_region   = "us-west-2"
environment  = "production"
project_name = "myapp"

vpc_cidr           = "10.0.0.0/16"
availability_zones = ["us-west-2a", "us-west-2b", "us-west-2c"]

db_name             = "myapp"
db_username         = "admin"
db_instance_class   = "db.t3.small"
db_allocated_storage = 100

ami_id        = "ami-0abcdef1234567890"
instance_type = "t3.small"

asg_min_size         = 2
asg_max_size         = 10
asg_desired_capacity = 3

ssl_certificate_arn = "arn:aws:acm:us-west-2:123456789012:certificate/abc-123"
```

### Terraform Workflow

```bash
# Initialize (download providers)
terraform init

# Format code
terraform fmt -recursive

# Validate configuration
terraform validate

# Plan changes (preview)
terraform plan -out=tfplan

# Review plan
less tfplan

# Apply changes
terraform apply tfplan

# View state
terraform show

# List resources
terraform state list

# View specific resource
terraform state show aws_vpc.main

# Destroy everything (careful!)
terraform destroy
```

### Terraform Modules

Create reusable modules for common patterns.

**modules/web-app/main.tf:**

```hcl
# Reusable web application module

variable "name" {
  type = string
}

variable "vpc_id" {
  type = string
}

variable "subnet_ids" {
  type = list(string)
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "min_size" {
  type    = number
  default = 2
}

variable "max_size" {
  type    = number
  default = 10
}

# Security Group
resource "aws_security_group" "this" {
  name_prefix = "${var.name}-"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Launch Template
resource "aws_launch_template" "this" {
  name_prefix   = "${var.name}-"
  image_id      = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type

  vpc_security_group_ids = [aws_security_group.this.id]
}

# Auto Scaling Group
resource "aws_autoscaling_group" "this" {
  name                = "${var.name}-asg"
  vpc_zone_identifier = var.subnet_ids
  min_size            = var.min_size
  max_size            = var.max_size
  desired_capacity    = var.min_size

  launch_template {
    id      = aws_launch_template.this.id
    version = "$Latest"
  }
}

# Outputs
output "security_group_id" {
  value = aws_security_group.this.id
}

output "asg_name" {
  value = aws_autoscaling_group.this.name
}

data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

**Use the module:**

```hcl
module "web_app" {
  source = "./modules/web-app"

  name          = "myapp"
  vpc_id        = aws_vpc.main.id
  subnet_ids    = aws_subnet.private[*].id
  instance_type = "t3.small"
  min_size      = 3
  max_size      = 20
}

output "web_app_asg" {
  value = module.web_app.asg_name
}
```

---

## Building Block 2: Configuration Management

### Ansible for Server Configuration

**Directory Structure:**

```
ansible/
├── inventory/
│   ├── production.ini
│   └── staging.ini
├── group_vars/
│   ├── all.yml
│   ├── webservers.yml
│   └── databases.yml
├── roles/
│   ├── common/
│   ├── nginx/
│   ├── postgresql/
│   └── monitoring/
├── playbooks/
│   ├── site.yml
│   ├── webservers.yml
│   └── databases.yml
└── ansible.cfg
```

**inventory/production.ini:**

```ini
[webservers]
web1.example.com
web2.example.com
web3.example.com

[databases]
db1.example.com
db2.example.com

[monitoring]
monitor.example.com

[production:children]
webservers
databases
monitoring
```

**playbooks/webservers.yml:**

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: yes

  roles:
    - common
    - nginx
    - monitoring

  tasks:
    - name: Install application dependencies
      apt:
        name:
          - python3
          - python3-pip
          - python3-venv
          - git
        state: present
        update_cache: yes

    - name: Create application user
      user:
        name: appuser
        system: yes
        create_home: yes
        home: /opt/app
        shell: /bin/bash

    - name: Clone application repository
      git:
        repo: https://github.com/myorg/myapp.git
        dest: /opt/app/myapp
        version: "{{ app_version }}"
        force: yes
      become_user: appuser
      notify: Restart application

    - name: Install Python dependencies
      pip:
        requirements: /opt/app/myapp/requirements.txt
        virtualenv: /opt/app/venv
        virtualenv_python: python3
      become_user: appuser

    - name: Copy application configuration
      template:
        src: app-config.j2
        dest: /opt/app/myapp/config.yml
        owner: appuser
        group: appuser
        mode: '0600'
      notify: Restart application

    - name: Copy systemd service file
      template:
        src: app.service.j2
        dest: /etc/systemd/system/myapp.service
        owner: root
        group: root
        mode: '0644'
      notify:
        - Reload systemd
        - Restart application

    - name: Enable and start application service
      systemd:
        name: myapp
        enabled: yes
        state: started
        daemon_reload: yes

    - name: Configure log rotation
      copy:
        src: logrotate-app.conf
        dest: /etc/logrotate.d/myapp
        owner: root
        group: root
        mode: '0644'

  handlers:
    - name: Reload systemd
      systemd:
        daemon_reload: yes

    - name: Restart application
      systemd:
        name: myapp
        state: restarted
```

**roles/nginx/tasks/main.yml:**

```yaml
---
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes

- name: Remove default nginx site
  file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  notify: Reload nginx

- name: Copy nginx configuration
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    validate: 'nginx -t -c %s'
  notify: Reload nginx

- name: Copy site configuration
  template:
    src: site.conf.j2
    dest: /etc/nginx/sites-available/{{ site_name }}
    owner: root
    group: root
    mode: '0644'
    validate: 'nginx -t -c /etc/nginx/nginx.conf'
  notify: Reload nginx

- name: Enable site
  file:
    src: /etc/nginx/sites-available/{{ site_name }}
    dest: /etc/nginx/sites-enabled/{{ site_name }}
    state: link
  notify: Reload nginx

- name: Ensure nginx is running
  systemd:
    name: nginx
    state: started
    enabled: yes
```

**Run Ansible:**

```bash
# Check connectivity
ansible all -i inventory/production.ini -m ping

# Run playbook (dry run)
ansible-playbook -i inventory/production.ini playbooks/webservers.yml --check

# Run playbook
ansible-playbook -i inventory/production.ini playbooks/webservers.yml

# Run specific tasks
ansible-playbook -i inventory/production.ini playbooks/webservers.yml --tags="nginx"

# Limit to specific hosts
ansible-playbook -i inventory/production.ini playbooks/webservers.yml --limit="web1.example.com"
```

---

## Building Block 3: GitOps with ArgoCD

GitOps: Git as single source of truth for infrastructure.

### ArgoCD Setup

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Application Definition

**argocd/application.yaml:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default

  # Source: Git repository
  source:
    repoURL: https://github.com/myorg/myapp-k8s
    targetRevision: main
    path: k8s/production

    # Kustomize configuration
    kustomize:
      namePrefix: prod-
      commonLabels:
        environment: production

  # Destination: Kubernetes cluster
  destination:
    server: https://kubernetes.default.svc
    namespace: production

  # Sync policy
  syncPolicy:
    automated:
      prune: true      # Delete resources not in git
      selfHeal: true   # Force sync if manual changes detected
      allowEmpty: false

    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true

    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # Health checks
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # Ignore replica count (HPA manages this)
```

### Multi-Environment Setup

**Git Repository Structure:**

```
myapp-k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   ├── replicas.yaml
│   │   └── resources.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   ├── replicas.yaml
│   │   └── resources.yaml
│   └── production/
│       ├── kustomization.yaml
│       ├── replicas.yaml
│       └── resources.yaml
└── argocd/
    ├── dev-app.yaml
    ├── staging-app.yaml
    └── production-app.yaml
```

**base/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app: myapp
```

**overlays/production/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namePrefix: prod-

commonLabels:
  environment: production

replicas:
  - name: myapp
    count: 5

images:
  - name: myapp
    newTag: v1.2.3

patchesStrategicMerge:
  - replicas.yaml
  - resources.yaml

configMapGenerator:
  - name: app-config
    env: production.env
```

**GitOps Workflow:**

```
Developer makes change:
    1. git commit -m "Update deployment"
    2. git push origin main
       ↓
    3. ArgoCD detects change (polls every 3 minutes)
       ↓
    4. ArgoCD applies changes to Kubernetes
       ↓
    5. Kubernetes rolls out update
       ↓
    6. ArgoCD reports status (Success/Failed)

If manual change in cluster:
    1. kubectl scale deployment myapp --replicas=10
       ↓
    2. ArgoCD detects drift (selfHeal enabled)
       ↓
    3. ArgoCD reverts to git state
       ↓
    4. Git is source of truth!
```

---

## Testing Infrastructure Code

### Terraform Testing

**tests/vpc_test.go:**

```go
package test

import (
    "testing"

    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestVPCCreation(t *testing.T) {
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../infrastructure",
        Vars: map[string]interface{}{
            "project_name": "test",
            "environment":  "test",
            "vpc_cidr":     "10.1.0.0/16",
        },
        NoColor: true,
    })

    defer terraform.Destroy(t, terraformOptions)

    terraform.InitAndApply(t, terraformOptions)

    // Test VPC CIDR
    vpcCIDR := terraform.Output(t, terraformOptions, "vpc_cidr")
    assert.Equal(t, "10.1.0.0/16", vpcCIDR)

    // Test number of subnets
    publicSubnets := terraform.OutputList(t, terraformOptions, "public_subnet_ids")
    assert.Len(t, publicSubnets, 3)

    privateSubnets := terraform.OutputList(t, terraformOptions, "private_subnet_ids")
    assert.Len(t, privateSubnets, 3)
}

func TestHighAvailability(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../infrastructure",
    }

    terraform.InitAndApply(t, terraformOptions)

    // Test resources are spread across AZs
    asgName := terraform.Output(t, terraformOptions, "asg_name")

    // Verify ASG spans multiple AZs
    // (implementation depends on cloud provider SDK)
}
```

**Run tests:**

```bash
cd tests
go test -v -timeout 30m
```

### Pre-commit Hooks

**.pre-commit-config.yaml:**

```yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.5
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_docs
      - id: terraform_tflint
        args:
          - '--args=--only=terraform_deprecated_interpolation'
          - '--args=--only=terraform_deprecated_index'
          - '--args=--only=terraform_unused_declarations'
          - '--args=--only=terraform_comment_syntax'
          - '--args=--only=terraform_documented_outputs'
          - '--args=--only=terraform_documented_variables'
          - '--args=--only=terraform_typed_variables'
          - '--args=--only=terraform_module_pinned_source'
          - '--args=--only=terraform_naming_convention'
          - '--args=--only=terraform_required_version'
          - '--args=--only=terraform_required_providers'
          - '--args=--only=terraform_standard_module_structure'
          - '--args=--only=terraform_workspace_remote'
      - id: terraform_tfsec
        args:
          - >-
            --args=--minimum-severity=MEDIUM
```

**Install and use:**

```bash
pip install pre-commit
pre-commit install
pre-commit run --all-files
```

---

## CI/CD Pipeline for Infrastructure

### GitHub Actions Example

**.github/workflows/terraform.yml:**

```yaml
name: Terraform

on:
  push:
    branches: [main]
    paths:
      - 'infrastructure/**'
  pull_request:
    branches: [main]
    paths:
      - 'infrastructure/**'

env:
  TF_VERSION: 1.6.0

jobs:
  validate:
    name: Validate
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        run: terraform fmt -check -recursive
        working-directory: infrastructure

      - name: Terraform Init
        run: terraform init
        working-directory: infrastructure
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Terraform Validate
        run: terraform validate
        working-directory: infrastructure

  plan:
    name: Plan
    runs-on: ubuntu-latest
    needs: validate
    if: github.event_name == 'pull_request'

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init
        working-directory: infrastructure
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        working-directory: infrastructure
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        continue-on-error: true

      - name: Comment PR
        uses: actions/github-script@v6
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const output = `#### Terraform Plan 📖

            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`

            *Pusher: @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

  apply:
    name: Apply
    runs-on: ubuntu-latest
    needs: validate
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init
        working-directory: infrastructure
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Terraform Apply
        run: terraform apply -auto-approve
        working-directory: infrastructure
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

## Disaster Recovery

### Backup Strategy

**backup-script.sh:**

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DATE=$(date +%Y-%m-%d-%H%M%S)
S3_BUCKET="myapp-backups"

# Backup Terraform state
echo "Backing up Terraform state..."
aws s3 cp \
  s3://myapp-terraform-state/production/terraform.tfstate \
  s3://${S3_BUCKET}/terraform/${BACKUP_DATE}/terraform.tfstate

# Backup Kubernetes resources
echo "Backing up Kubernetes resources..."
kubectl get all --all-namespaces -o yaml > k8s-backup-${BACKUP_DATE}.yaml
gzip k8s-backup-${BACKUP_DATE}.yaml
aws s3 cp k8s-backup-${BACKUP_DATE}.yaml.gz s3://${S3_BUCKET}/kubernetes/${BACKUP_DATE}/

# Backup databases
echo "Backing up databases..."
pg_dump -h ${DB_HOST} -U ${DB_USER} ${DB_NAME} | gzip > db-backup-${BACKUP_DATE}.sql.gz
aws s3 cp db-backup-${BACKUP_DATE}.sql.gz s3://${S3_BUCKET}/database/${BACKUP_DATE}/

# Backup secrets
echo "Backing up secrets..."
kubectl get secrets --all-namespaces -o yaml > secrets-backup-${BACKUP_DATE}.yaml
gpg --encrypt --recipient ops@example.com secrets-backup-${BACKUP_DATE}.yaml
aws s3 cp secrets-backup-${BACKUP_DATE}.yaml.gpg s3://${S3_BUCKET}/secrets/${BACKUP_DATE}/

# Cleanup local files
rm -f k8s-backup-${BACKUP_DATE}.yaml.gz db-backup-${BACKUP_DATE}.sql.gz secrets-backup-${BACKUP_DATE}.yaml*

echo "✅ Backup complete: ${BACKUP_DATE}"
```

### Restore Procedure

**restore-procedure.md:**

```markdown
# Disaster Recovery Procedure

## Prerequisites
- AWS CLI configured
- kubectl configured
- Terraform installed
- GPG key for decrypting secrets

## Restore Steps

### 1. Restore Terraform State
```bash
BACKUP_DATE=2024-01-15-120000
aws s3 cp s3://myapp-backups/terraform/${BACKUP_DATE}/terraform.tfstate ./terraform.tfstate
terraform state push terraform.tfstate
```

### 2. Recreate Infrastructure
```bash
cd infrastructure
terraform init
terraform plan
terraform apply
```

### 3. Restore Kubernetes Resources
```bash
aws s3 cp s3://myapp-backups/kubernetes/${BACKUP_DATE}/k8s-backup-${BACKUP_DATE}.yaml.gz ./
gunzip k8s-backup-${BACKUP_DATE}.yaml.gz
kubectl apply -f k8s-backup-${BACKUP_DATE}.yaml
```

### 4. Restore Database
```bash
aws s3 cp s3://myapp-backups/database/${BACKUP_DATE}/db-backup-${BACKUP_DATE}.sql.gz ./
gunzip db-backup-${BACKUP_DATE}.sql.gz
psql -h ${DB_HOST} -U ${DB_USER} ${DB_NAME} < db-backup-${BACKUP_DATE}.sql
```

### 5. Restore Secrets
```bash
aws s3 cp s3://myapp-backups/secrets/${BACKUP_DATE}/secrets-backup-${BACKUP_DATE}.yaml.gpg ./
gpg --decrypt secrets-backup-${BACKUP_DATE}.yaml.gpg > secrets-backup.yaml
kubectl apply -f secrets-backup.yaml
```

### 6. Verify
- [ ] All pods running
- [ ] Database accessible
- [ ] Application responding
- [ ] Monitoring working
- [ ] DNS resolving
```

---

## Best Practices

### DO ✅

1. **Version Control Everything:**
```
All infrastructure code in Git
All configuration in Git
All documentation in Git
→ Single source of truth
```

2. **Use Remote State:**
```hcl
terraform {
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-locks"  # Locking!
  }
}
```

3. **Implement State Locking:**
```bash
# Prevents concurrent modifications
# Uses DynamoDB for locking in AWS
# Automatically handled by Terraform
```

4. **Use Modules for Reusability:**
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

5. **Tag All Resources:**
```hcl
default_tags {
  tags = {
    Environment = "production"
    ManagedBy   = "Terraform"
    Project     = "myapp"
    Owner       = "platform-team"
    CostCenter  = "engineering"
  }
}
```

6. **Separate Environments:**
```
infrastructure/
├── dev/
├── staging/
└── production/  # Never share state!
```

7. **Use Data Sources:**
```hcl
# Don't hardcode AMI IDs
data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*"]
  }
}
```

8. **Validate Before Apply:**
```bash
terraform validate
terraform plan
# Review plan carefully!
terraform apply
```

9. **Use Workspaces for Environments:**
```bash
terraform workspace new production
terraform workspace new staging
terraform workspace select production
```

10. **Implement Naming Conventions:**
```hcl
resource "aws_instance" "web" {
  # Good: descriptive, consistent
  tags = {
    Name = "${var.project_name}-${var.environment}-web-${count.index + 1}"
  }
}
```

### DON'T ❌

1. **Don't Store Secrets in Code:**
```hcl
# Bad:
variable "db_password" {
  default = "supersecret123"  # ❌
}

# Good:
variable "db_password" {
  type      = string
  sensitive = true
  # Pass via environment or secrets manager
}
```

2. **Don't Skip Planning:**
```bash
# Bad:
terraform apply -auto-approve  # ❌ Dangerous!

# Good:
terraform plan -out=tfplan
# Review plan
terraform apply tfplan
```

3. **Don't Modify State Manually:**
```bash
# Bad:
vim terraform.tfstate  # ❌ Never!

# Good:
terraform state rm aws_instance.old
terraform import aws_instance.new i-123456
```

4. **Don't Use Count for Critical Resources:**
```hcl
# Bad: Reordering list recreates resources
resource "aws_instance" "app" {
  count = length(var.instance_names)
  # ...
}

# Good: Use for_each with map
resource "aws_instance" "app" {
  for_each = toset(var.instance_names)
  # ...
}
```

5. **Don't Ignore Drift:**
```bash
# Check for configuration drift regularly
terraform plan

# If drift detected, investigate and fix
```

6. **Don't Share State Files:**
```
# Each environment needs its own state
production.tfstate  # ✅
staging.tfstate     # ✅
shared.tfstate      # ❌ Don't do this
```

7. **Don't Hardcode Values:**
```hcl
# Bad:
resource "aws_instance" "web" {
  instance_type = "t3.micro"  # ❌ Hardcoded
}

# Good:
resource "aws_instance" "web" {
  instance_type = var.instance_type  # ✅ Variable
}
```

8. **Don't Skip Documentation:**
```hcl
# Add descriptions to everything
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}
```

9. **Don't Deploy Without Testing:**
```bash
# Test in dev first
terraform workspace select dev
terraform apply

# Then staging
terraform workspace select staging
terraform apply

# Finally production
terraform workspace select production
terraform apply
```

10. **Don't Forget Destroy Protection:**
```hcl
resource "aws_db_instance" "production" {
  # Prevent accidental deletion
  deletion_protection = true

  lifecycle {
    prevent_destroy = true
  }
}
```

---

## Further Reading

**Books:**
- "Terraform: Up & Running" by Yevgeniy Brikman
- "Infrastructure as Code" by Kief Morris
- "Site Reliability Engineering" by Google

**Documentation:**
- Terraform: https://www.terraform.io/docs
- Ansible: https://docs.ansible.com
- ArgoCD: https://argo-cd.readthedocs.io

**Online Resources:**
- Terraform Registry: https://registry.terraform.io
- Terraform Best Practices: https://www.terraform-best-practices.com
- GitOps Guide: https://www.gitops.tech

---

## Summary

In this chapter, we built infrastructure as code:

1. **Terraform** for infrastructure provisioning
2. **Ansible** for configuration management
3. **GitOps** for declarative operations
4. **Testing** for infrastructure validation
5. **CI/CD** for automated deployment
6. **Disaster Recovery** for business continuity

**Key Takeaways:**
- Infrastructure as code enables reproducibility
- Version control provides audit trail
- Declarative approach reduces complexity
- Automation reduces human error
- Testing catches issues early
- GitOps ensures consistency

**Next Chapter Preview:**

Chapter 10 concludes with **The Future of Infrastructure** - emerging patterns, trends, and the evolution toward self-managing, intelligent infrastructure systems.

Ready to explore the future? Let's go! 🚀

---

## Apex Infrastructure-as-Code Practices

Codify your organism's skeleton with the industry's strongest bones:

- **Modular Terraform/OpenTofu** — Structure modules with versioned registries, test via Terratest or Kitchen-Terraform, and lint with terraform-docs + tflint to guarantee reusable building blocks.
- **Drift Prevention** — Run Atlantis or Spacelift with mandatory plan reviews, integrate driftctl or Terraform Cloud drift detection, and alert via Slack when reality diverges from desired state.
- **Multi-Cloud Composition** — Use Crossplane or AWS Proton to expose opinionated infrastructure APIs, while Pulumi or CDKTF handles polyglot use cases where imperative languages shine.
- **Policy & Compliance as Code** — Enforce Sentinel, OPA/Conftest, or HashiCorp Cloud Platform Policy Sets to block risky changes, backed by Regula/tfsec scans and cloud provider configuration rules (Azure Policy, AWS Config).
- **Secrets & Config Hygiene** — Store state in encrypted backends (AWS KMS + S3, Google Cloud KMS + GCS), wrap Terraform with SOPS or Chamber, and rotate credentials automatically through Vault or AWS IAM Roles Anywhere.
- **Pipeline Automation** — Embed `terraform fmt/validate`, `infracost breakdown`, and security scanners in CI, then promote plans via GitOps controllers (Argo CD, Flux) or environment promotion workflows (Spacelift stacks, Env0).

Adopting these practices keeps your infrastructure definitions auditable, composable, and resilient to human error.