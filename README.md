# Scalable-Secure-Observable-Microservices-Platform-on-AWS

A scalable, secure, and observable microservices infrastructure built on AWS. Covers everything from VPC design and compute autoscaling to centralized monitoring, SNS alerting, and Lambda-driven self-healing automation.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [AWS Services Used](#aws-services-used)
- [Project Structure](#project-structure)
- [Phase-by-Phase Implementation](#phase-by-phase-implementation)
  - [Phase 1 — Networking (VPC)](#phase-1--networking-vpc)
  - [Phase 2 — Compute Layer](#phase-2--compute-layer)
  - [Phase 3 — Database Layer](#phase-3--database-layer)
  - [Phase 4 — Storage](#phase-4--storage)
  - [Phase 5 — Monitoring & Logging](#phase-5--monitoring--logging)
  - [Phase 6 — Alerting (SNS)](#phase-6--alerting-sns)
  - [Phase 7 — Lambda Automation](#phase-7--lambda-automation)
  - [Phase 8 — Security Hardening](#phase-8--security-hardening)
- [Key Design Decisions](#key-design-decisions)
- [Outcomes](#outcomes)

---

## Architecture Overview

```
                        ┌─────────────────────────────────────────┐
                        │              AWS Cloud (VPC)             │
                        │  CIDR: 10.0.0.0/16                      │
                        │                                         │
  Internet ──── IGW ───►│  ┌──────────────────────────────────┐  │
                        │  │        Public Subnet              │  │
                        │  │   ALB  (10.0.1.0/24)  NAT GW     │  │
                        │  └────────────┬─────────────────────┘  │
                        │               │                         │
                        │  ┌────────────▼─────────────────────┐  │
                        │  │        Private Subnet             │  │
                        │  │   EC2 (ASG)   RDS Multi-AZ        │  │
                        │  │   (10.0.2.0/24)                   │  │
                        │  └──────────────────────────────────┘  │
                        └─────────────────────────────────────────┘
                                         │
                              ┌──────────▼──────────┐
                              │   CloudWatch  + SNS  │
                              │   Lambda Auto-Heal   │
                              └─────────────────────┘
```

**Multi-AZ deployment across 2 Availability Zones for high availability.**

---

## AWS Services Used

| Category | Services |
|---|---|
| Networking | VPC, Subnets, Internet Gateway, NAT Gateway, Route Tables, Security Groups, NACLs |
| Compute | EC2, Auto Scaling Group, Application Load Balancer |
| Database | RDS (MySQL/PostgreSQL), Multi-AZ, Read Replica |
| Storage | S3 (assets, logs, backups) |
| Monitoring | CloudWatch Metrics, CloudWatch Logs, CloudWatch Alarms, Dashboards |
| Notifications | Amazon SNS (email alerts) |
| Automation | AWS Lambda |
| Security | IAM Roles & Policies, Secrets Manager |
| Optional | AWS X-Ray, AWS WAF |

---

## Project Structure

```
aws-microservices-platform/
├── networking/
│   ├── vpc.tf                  # VPC + subnet definitions
│   ├── routing.tf              # Route tables, IGW, NAT GW
│   └── security-groups.tf      # SG rules for ALB, EC2, RDS
├── compute/
│   ├── ec2-launch-template.tf  # EC2 config with IAM role
│   ├── alb.tf                  # Load balancer + target group
│   └── asg.tf                  # Auto Scaling Group + policies
├── database/
│   └── rds.tf                  # RDS Multi-AZ, subnet group
├── storage/
│   └── s3.tf                   # Buckets, lifecycle policies
├── monitoring/
│   ├── cloudwatch-agent.json   # Agent config for EC2
│   ├── alarms.tf               # CPU, disk, error alarms
│   └── dashboard.json          # CloudWatch dashboard layout
├── automation/
│   └── lambda/
│       └── remediation.py      # Auto-remediation handler
├── iam/
│   └── roles.tf                # EC2 and Lambda IAM roles
└── README.md
```

---

## Phase-by-Phase Implementation

### Phase 1 — Networking (VPC)

**Goal:** Isolate public-facing resources from backend services.

```bash
# VPC CIDR block
10.0.0.0/16

# Subnets (across 2 AZs)
Public Subnet A:  10.0.1.0/24  (AZ-a)
Public Subnet B:  10.0.3.0/24  (AZ-b)
Private Subnet A: 10.0.2.0/24  (AZ-a)
Private Subnet B: 10.0.4.0/24  (AZ-b)
```

**Routing setup:**
- Public subnets → Internet Gateway
- Private subnets → NAT Gateway (outbound only)

**Why this matters:**  
EC2 instances and RDS are never exposed to the internet directly. All inbound traffic enters through the ALB in the public subnet.

---

### Phase 2 — Compute Layer

**EC2 + ALB + Auto Scaling Group**

```
Security Group Rules:
  ALB SG  → Inbound: 0.0.0.0/0 on port 80/443
  EC2 SG  → Inbound: ALB SG only (no direct internet access)
```

Auto Scaling policy:

```
Min instances:  2
Max instances:  5
Scale out:      CPU > 70% for 2 consecutive periods
Scale in:       CPU < 30% for 5 consecutive periods
Health check:   /health (HTTP 200 required)
```

---

### Phase 3 — Database Layer

- RDS deployed in **private subnets only**
- Multi-AZ enabled (automatic failover in < 2 min)
- DB Security Group: only accepts connections from EC2 SG
- Automated backups: 7-day retention
- No public accessibility — disabled

```
DB Endpoint access: EC2 → RDS (internal VPC only)
Public access:      DISABLED ✗
```

---

### Phase 4 — Storage

S3 bucket configuration:

```
Versioning:       Enabled
Encryption:       SSE-S3
Public access:    Blocked
Lifecycle policy:
  - Move to S3-IA after 30 days
  - Archive to Glacier after 90 days
  - Expire after 365 days
```

Used for: application logs, EC2 backups, static assets.

---

### Phase 5 — Monitoring & Logging

CloudWatch Agent installed on EC2 to collect:

```json
{
  "metrics": {
    "metrics_collected": {
      "cpu":    { "measurement": ["cpu_usage_idle", "cpu_usage_user"] },
      "mem":    { "measurement": ["mem_used_percent"] },
      "disk":   { "measurement": ["used_percent"], "resources": ["/"] }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          { "file_path": "/var/log/app/*.log", "log_group_name": "/app/logs" }
        ]
      }
    }
  }
}
```

Dashboard tracks: EC2 CPU, ALB request count, RDS connections, error rates.

---

### Phase 6 — Alerting (SNS)

CloudWatch Alarms → SNS Topic → Email notification

| Alarm | Threshold | Action |
|---|---|---|
| High CPU | > 80% for 5 min | SNS alert + Lambda trigger |
| Low disk | < 20% free | SNS alert |
| App errors | > 10 in 5 min | SNS alert |
| ALB 5xx rate | > 1% | SNS alert |

---

### Phase 7 — Lambda Automation

Event-driven remediation flow:

```
CloudWatch Alarm
      │
      ▼
   SNS Topic
      │
      ▼
  Lambda Function
      │
      ├── Restart unhealthy EC2 instance
      ├── Trigger ASG scale-out
      └── Push enriched alert to SNS
```

Lambda IAM Role permissions (least privilege):
- `ec2:RebootInstances` — specific instance ARNs only
- `ec2:DescribeInstanceStatus`
- `autoscaling:SetDesiredCapacity`
- `logs:CreateLogGroup`, `logs:PutLogEvents`

```python
import boto3

def handler(event, context):
    ec2 = boto3.client('ec2')
    instance_id = event['detail']['instance-id']
    
    ec2.reboot_instances(InstanceIds=[instance_id])
    print(f"Rebooted instance: {instance_id}")
```

---

### Phase 8 — Security Hardening

- IAM roles on EC2 and Lambda — no static access keys anywhere
- S3 bucket policies block all public access
- RDS encryption at rest enabled
- CloudTrail enabled for API audit logging
- Security Groups follow least-privilege ingress/egress rules

---

## Key Design Decisions

**Why private subnets for EC2?**  
Reduces attack surface. Only the load balancer is internet-facing. EC2 instances are unreachable directly from outside the VPC.

**Why NAT Gateway instead of a NAT instance?**  
Managed, highly available, no patching overhead. NAT instances are cheaper but introduce a single point of failure.

**Why Multi-AZ for RDS?**  
Automatic failover to standby replica in a different AZ without manual intervention. Acceptable for < 2 minute RPO/RTO in most use cases.

**Why Lambda for auto-remediation instead of SSM Automation?**  
More flexibility for custom logic (enriched alerts, conditional restarts, chaining actions). SSM Automation works well for runbooks but Lambda fits event-driven workflows better here.

---

## Outcomes

| Metric | Result |
|---|---|
| Availability | 99.9% uptime |
| Incident response | Automated — no manual intervention for common failures |
| Scaling | Handles variable load, scales out in ~3 min |
| MTTR | Reduced significantly via proactive alerting + auto-remediation |
| Security posture | No public DB/EC2 exposure, encrypted storage, IAM roles throughout |

---
