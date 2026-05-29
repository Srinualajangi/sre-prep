# 🔥 INTERVIEW BIBLE — Quick Revision Guide
### Based on Your Resume | Read this before EVERY interview
### Last Updated: May 22, 2026

> [!IMPORTANT]
> **How to use this:** Read each section as if YOU are explaining YOUR current work.
> Every answer below is framed as "In my current organization at ASM Technologies..."
> Internalize the narrative. You lived this. Own it.

---

## TABLE OF CONTENTS

1. [Hybrid Cloud Management (1000+ Instances)](#1-hybrid-cloud-management)
2. [Ansible Automation Framework](#2-ansible-automation-framework)
3. [Terraform IaC — Modules & Infrastructure](#3-terraform-iac)
4. [Python + Lambda + EventBridge Automation](#4-python-lambda-eventbridge)
5. [Kubernetes Migration (EKS/AKS)](#5-kubernetes-migration)
6. [CI/CD with DevSecOps (Jenkins + GitHub Actions)](#6-cicd-with-devsecops)
7. [⭐ Prometheus + Grafana + Zabbix Observability (Deep Dive)](#7-observability-deep-dive)
8. [AWS Auto-Scaling & Security](#8-aws-auto-scaling--security)
9. [Linux Kernel Tuning (RHEL 7/8/9)](#9-linux-kernel-tuning)
10. [Networking & Security Fundamentals](#10-networking--security)
11. [Shell Scripting & Automation](#11-shell-scripting)
12. [Rapid-Fire Interview Q&A](#12-rapid-fire-qa)

---

## 1. HYBRID CLOUD MANAGEMENT

### 🎯 Your Story
> "At ASM Technologies, I manage a hybrid infrastructure of over 1000 instances — roughly 600 on AWS (EC2), about 200 on Azure VMs, and around 200 on-premise RHEL servers. My responsibility is ensuring 99.9% uptime across all environments."

### What This Means in Practice

```
Your Infrastructure Map:
┌─────────────────────────────────────────────────────┐
│                    AWS (Primary)                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Prod VPC │  │  QA VPC  │  │ Dev VPC  │          │
│  │ ~300 EC2 │  │ ~150 EC2 │  │ ~150 EC2 │          │
│  │ 3 EKS    │  │ 1 EKS   │  │ 1 EKS   │          │
│  │ RDS Multi│  │ RDS     │  │ RDS     │          │
│  └──────────┘  └──────────┘  └──────────┘          │
├─────────────────────────────────────────────────────┤
│                  Azure (Secondary)                   │
│  ┌──────────┐  ┌──────────┐                         │
│  │ Prod     │  │ DR       │                         │
│  │ ~150 VMs │  │ ~50 VMs  │                         │
│  │ AKS      │  │ AKS      │                         │
│  └──────────┘  └──────────┘                         │
├─────────────────────────────────────────────────────┤
│                On-Premise (Legacy)                   │
│  ┌──────────────────────────────────┐               │
│  │ ~200 RHEL 7/8/9 Servers          │               │
│  │ VMware ESXi + KVM Hypervisors    │               │
│  │ NFS, DNS, DHCP services          │               │
│  └──────────────────────────────────┘               │
└─────────────────────────────────────────────────────┘
```

### Key Concepts You MUST Know

**99.9% Uptime = 8.76 hours of downtime/year**
```
99%    = 3.65 days downtime/year    (unacceptable for prod)
99.9%  = 8.76 hours/year            (YOUR SLA ✅)
99.95% = 4.38 hours/year            (FAANG target)
99.99% = 52.6 minutes/year          (Google-level)
```

**How you achieve 99.9%:**
- Multi-AZ deployments (instances spread across us-east-1a, 1b, 1c)
- Auto-scaling groups with health checks
- RDS Multi-AZ for database HA
- Route53 health checks with failover routing
- Proactive monitoring with Prometheus/Grafana alerts
- Automated patching during maintenance windows

### Interview Q&A

**Q: "How do you manage 1000+ instances day-to-day?"**
> "We use a combination of tools. Ansible for configuration management and patching — I run playbooks against inventory groups (prod, QA, dev). Terraform for provisioning any new infrastructure. Prometheus + Grafana for monitoring health and performance metrics. We have Zabbix for the on-premise RHEL servers. Everything is tagged by environment, team, and application — so I can filter and manage logically. For example, if I need to patch all RHEL 8 prod servers, I target the Ansible inventory group `[prod_rhel8]` and run the patching playbook with a rolling strategy to avoid downtime."

**Q: "What happens when a server goes down at 2 AM?"**
> "We have tiered alerting. Prometheus fires alerts to PagerDuty/Slack based on severity. P1 (prod down) pages the on-call engineer immediately. P2 (degraded) notifies the Slack channel. When I get paged, I follow our incident response runbook — first check the CloudWatch/Grafana dashboard to identify scope, then SSH into the instance or check logs in ELK. Most common issues: OOM kills (check `dmesg | grep -i oom`), disk full (`df -h`), or service crash (`systemctl status <service>`). After resolution, I write a postmortem documenting root cause, impact, and preventive action."

**Q: "How do you ensure consistency across 1000 instances?"**
> "Three layers: (1) Terraform ensures infrastructure is provisioned identically — same VPC, same security groups, same instance types per environment. (2) Ansible ensures OS-level configuration is consistent — same packages, same kernel parameters, same hardening baselines. (3) We use AMIs (Amazon Machine Images) as golden images — every new instance launches from a pre-hardened, pre-configured AMI that we rebuild monthly."

---

## 2. ANSIBLE AUTOMATION FRAMEWORK

### 🎯 Your Story
> "I built an Ansible automation framework that handles three critical things: OS patching, security baseline hardening, and compliance checks. This reduced our operational toil by 60% — from about 200 hours/month of manual work to about 80."

### Architecture

```
Ansible Control Node (your machine or CI server)
│
├── inventory/
│   ├── production          # 600+ hosts grouped by role
│   ├── staging             
│   └── development
│
├── playbooks/
│   ├── patching.yml        # OS patching (yum/apt)
│   ├── hardening.yml       # CIS benchmark hardening
│   ├── compliance.yml      # Check & report compliance
│   └── user_mgmt.yml      # User access management
│
├── roles/
│   ├── base_hardening/     # SSH, firewall, SELinux
│   ├── monitoring_agent/   # Install Prometheus node_exporter
│   ├── patching/           # Rolling OS updates
│   └── compliance_check/   # CIS benchmark validation
│
└── group_vars/
    ├── all.yml             # Global variables
    ├── production.yml      # Prod-specific vars
    └── development.yml     # Dev-specific vars
```

### Key Ansible Concepts & Commands

```bash
# Inventory file structure
[prod_web]
web01.prod.internal ansible_host=10.0.1.10
web02.prod.internal ansible_host=10.0.1.11

[prod_db]
db01.prod.internal ansible_host=10.0.2.10

[prod:children]
prod_web
prod_db

# Run patching playbook on prod with rolling updates
ansible-playbook -i inventory/production playbooks/patching.yml \
  --limit prod_web \
  -e "reboot_after_patch=true" \
  --forks 5

# Dry run (check mode) — see what WOULD change
ansible-playbook -i inventory/production playbooks/hardening.yml --check --diff

# Ad-hoc command — check uptime on all prod servers
ansible prod -i inventory/production -m command -a "uptime"

# Ad-hoc — check disk space
ansible prod -i inventory/production -m shell -a "df -h /"
```

### Sample Patching Playbook (Know This)

```yaml
---
- name: Rolling OS Patching
  hosts: "{{ target_hosts | default('all') }}"
  become: yes
  serial: "25%"         # Rolling: 25% of hosts at a time
  max_fail_percentage: 10  # Stop if >10% fail

  pre_tasks:
    - name: Check current kernel
      command: uname -r
      register: pre_kernel

  tasks:
    - name: Update all packages (RHEL/CentOS)
      yum:
        name: "*"
        state: latest
        exclude: kernel*    # Exclude kernel unless specified
      when: ansible_os_family == "RedHat"

    - name: Update all packages (Ubuntu)
      apt:
        upgrade: dist
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: Reboot if required
      reboot:
        reboot_timeout: 300
        msg: "Ansible patching reboot"
      when: reboot_after_patch | default(false) | bool

  post_tasks:
    - name: Verify services are running
      service:
        name: "{{ item }}"
        state: started
      loop:
        - sshd
        - prometheus-node-exporter
```

### Interview Q&A

**Q: "Explain serial and max_fail_percentage"**
> "`serial: 25%` means Ansible processes 25% of hosts at a time — so if I have 100 servers, it patches 25, waits for them to complete, then the next 25. This ensures we don't take down all servers simultaneously. `max_fail_percentage: 10` means if more than 10% of hosts fail during a batch, Ansible stops the entire playbook — so a bad patch doesn't cascade to all servers."

**Q: "How do you handle secrets in Ansible?"**
> "We use Ansible Vault for sensitive data. I encrypt variables like database passwords and API keys using `ansible-vault encrypt_string`. The vault password is stored in a file referenced by `--vault-password-file` in our CI pipeline. We never commit secrets in plaintext."

**Q: "Difference between Ansible and Terraform?"**
> "Terraform is for **provisioning infrastructure** — creating VPCs, EC2 instances, load balancers, RDS databases. It's declarative and tracks state. Ansible is for **configuring what's already provisioned** — installing packages, configuring services, patching OS, hardening security. In our workflow: Terraform creates the instance → Ansible configures it."

---

## 3. TERRAFORM IaC

### 🎯 Your Story
> "All our AWS infrastructure is 100% managed through Terraform. I built reusable modules for VPC, EKS, IAM, RDS, ALB, and ASG. We have three environments — dev, QA, prod — all using the same modules with different variable files. This ensures zero configuration drift."

### Architecture

```
terraform/
├── modules/                    # Reusable modules
│   ├── vpc/
│   │   ├── main.tf            # VPC, subnets, IGW, NAT
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── eks/
│   │   ├── main.tf            # EKS cluster, node groups
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── rds/
│   ├── alb/
│   └── iam/
│
├── environments/
│   ├── dev/
│   │   ├── main.tf            # Calls modules with dev vars
│   │   ├── terraform.tfvars   # dev-specific values
│   │   └── backend.tf         # S3 + DynamoDB state
│   ├── qa/
│   └── prod/
│
└── global/
    └── iam/                    # Global IAM roles
```

### Key Terraform Concepts

```hcl
# VPC Module Call (in environments/prod/main.tf)
module "vpc" {
  source = "../../modules/vpc"
  
  vpc_cidr        = "10.0.0.0/16"
  environment     = "prod"
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  single_nat_gateway = false  # One NAT per AZ for HA in prod
}

# State Backend (CRITICAL — know this)
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "prod/vpc/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"  # State locking
    encrypt        = true
  }
}
```

### Essential Commands

```bash
terraform init          # Download providers, initialize backend
terraform plan          # Preview changes (ALWAYS run before apply)
terraform apply         # Apply changes
terraform destroy       # Tear down infrastructure

terraform state list    # List all managed resources
terraform state show aws_instance.web  # Show specific resource
terraform import aws_instance.web i-1234  # Import existing resource

terraform workspace list     # List workspaces
terraform workspace select prod  # Switch to prod workspace

terraform fmt           # Format code
terraform validate      # Validate syntax
```

### Interview Q&A

**Q: "How do you handle Terraform state?"**
> "We store state in S3 with encryption enabled and use a DynamoDB table for state locking. This prevents two engineers from running `terraform apply` simultaneously, which could corrupt state. Each environment (dev/QA/prod) has its own state file with a separate S3 key path. We also enable versioning on the S3 bucket so we can recover previous state if needed."

**Q: "What is Terraform drift and how do you handle it?"**
> "Drift happens when someone manually changes infrastructure outside Terraform — like modifying a security group rule in the AWS console. We detect drift by running `terraform plan` regularly in our CI pipeline. If the plan shows unexpected changes, we investigate. To prevent drift, we enforce a policy that all changes go through Terraform PRs — no manual console changes allowed in prod."

**Q: "Explain modules and why you use them"**
> "Modules are reusable packages of Terraform configuration. Instead of writing VPC configuration three times for dev/QA/prod, I write it once as a module and call it with different variables. This ensures consistency — all environments have the same architecture (3 AZs, public/private subnets, NAT gateways) but with different CIDR ranges and instance sizes. When I update the module, all environments get the improvement."

**Q: "What happens if `terraform apply` fails midway?"**
> "Terraform is designed to handle this. It updates the state file after each resource is created, so if it fails midway, the state reflects what was actually created. On the next `terraform apply`, it picks up where it left off — only creating/modifying resources that haven't been applied yet. It won't duplicate already-created resources."

---

## 4. PYTHON + LAMBDA + EVENTBRIDGE

### 🎯 Your Story
> "I automated cloud cost optimization using Python Lambda functions triggered by EventBridge schedules. Three main automations: EC2 rightsizing recommendations, S3 lifecycle tiering, and EBS snapshot management. Together they save about $15K annually — a 35% cost reduction."

### Architecture

```
                    EventBridge (Cron Triggers)
                    ┌───────────────────────┐
                    │ cron(0 2 * * ? *)     │  ← Runs daily at 2 AM
                    │ cron(0 0 1 * ? *)     │  ← Runs 1st of each month
                    └───────────┬───────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                   │
    ┌─────────▼──────┐ ┌───────▼────────┐ ┌───────▼────────┐
    │ EC2 Rightsizer │ │ S3 Tiering     │ │ EBS Snapshot   │
    │ Lambda         │ │ Lambda         │ │ Cleanup Lambda │
    │                │ │                │ │                │
    │ • Check CPU    │ │ • Find objects │ │ • Find snaps   │
    │   utilization  │ │   >90 days     │ │   >30 days     │
    │   via CW       │ │ • Move to IA   │ │ • Delete old   │
    │ • Flag under-  │ │ • >180 days    │ │   ones         │
    │   utilized     │ │   → Glacier    │ │ • Keep tagged  │
    │ • Send report  │ │                │ │   ones         │
    │   to Slack     │ │                │ │                │
    └────────────────┘ └────────────────┘ └────────────────┘
```

### Sample Lambda Code (EC2 Rightsizing)

```python
import boto3
from datetime import datetime, timedelta

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    cloudwatch = boto3.client('cloudwatch')
    
    # Get all running instances
    instances = ec2.describe_instances(
        Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
    )
    
    underutilized = []
    
    for reservation in instances['Reservations']:
        for instance in reservation['Instances']:
            instance_id = instance['InstanceId']
            instance_type = instance['InstanceType']
            
            # Get average CPU over last 7 days
            cpu_stats = cloudwatch.get_metric_statistics(
                Namespace='AWS/EC2',
                MetricName='CPUUtilization',
                Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
                StartTime=datetime.utcnow() - timedelta(days=7),
                EndTime=datetime.utcnow(),
                Period=86400,  # 1 day
                Statistics=['Average']
            )
            
            if cpu_stats['Datapoints']:
                avg_cpu = sum(d['Average'] for d in cpu_stats['Datapoints']) / len(cpu_stats['Datapoints'])
                
                if avg_cpu < 10:  # Less than 10% CPU = underutilized
                    underutilized.append({
                        'InstanceId': instance_id,
                        'Type': instance_type,
                        'AvgCPU': round(avg_cpu, 2),
                        'Recommendation': 'Downsize or terminate'
                    })
    
    # Send report to SNS → Slack
    if underutilized:
        sns = boto3.client('sns')
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456:cost-alerts',
            Subject=f'EC2 Rightsizing: {len(underutilized)} underutilized instances',
            Message=str(underutilized)
        )
    
    return {'statusCode': 200, 'underutilized_count': len(underutilized)}
```

### Key Concepts

```
EventBridge Rule:
  - cron(0 2 * * ? *)  = Every day at 2:00 AM UTC
  - cron(0 0 1 * ? *)  = First day of every month at midnight
  - Rate: rate(1 hour)  = Every hour

Lambda Limits (know these):
  - Max execution time:  15 minutes
  - Max memory:          10,240 MB
  - Max package size:    250 MB (unzipped)
  - Concurrent:          1000 (default, can request increase)

IAM Role for Lambda:
  - Lambda needs an execution role with permissions to:
    - Read EC2, CloudWatch, S3
    - Write to SNS (for notifications)
    - Write to CloudWatch Logs (for its own logs)
```

### Interview Q&A

**Q: "How does EventBridge trigger Lambda?"**
> "I create an EventBridge rule with a cron expression — for example, `cron(0 2 * * ? *)` runs daily at 2 AM UTC. The rule's target is the Lambda function's ARN. When the cron fires, EventBridge invokes the Lambda function with an event payload. I also set up a dead-letter queue (SQS) for failed invocations so we don't silently lose failed runs."

**Q: "How did you calculate the $15K savings?"**
> "Three sources: (1) EC2 rightsizing — we identified ~40 instances running at under 10% CPU average over 30 days. Downsizing from m5.xlarge to m5.large saved about $8K/year. (2) S3 tiering — moving 5TB of objects older than 90 days to S3-IA and 180+ days to Glacier saved about $4K/year. (3) EBS snapshot cleanup — deleting 2000+ orphaned snapshots older than 30 days saved about $3K/year."

---

## 5. KUBERNETES MIGRATION

### 🎯 Your Story
> "We migrated 5 legacy VM-based services to Kubernetes microservices on EKS and AKS. I containerized the applications with Docker, created Helm charts with multi-environment overlays, and deployed across dev/QA/prod. This reduced deployment time from 45 minutes to about 5 minutes — a 40% improvement overall."

### Architecture

```
               ┌──────────────────────────────────────┐
               │        EKS Cluster (AWS)              │
               │                                       │
               │  ┌─────────┐  ┌─────────┐            │
               │  │  App A  │  │  App B  │            │
               │  │ 3 pods  │  │ 5 pods  │            │
               │  └────┬────┘  └────┬────┘            │
               │       │            │                  │
               │  ┌────▼────────────▼────┐            │
               │  │    ClusterIP Svc     │            │
               │  └──────────┬───────────┘            │
               │             │                         │
               │  ┌──────────▼───────────┐            │
               │  │   Ingress (ALB)      │            │
               │  └──────────────────────┘            │
               └──────────────────────────────────────┘
```

### Key K8s Concepts & Commands

```bash
# Cluster management
kubectl get nodes                          # List nodes
kubectl get pods -A                        # All 
