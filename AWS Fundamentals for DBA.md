# AWS Fundamentals for DBA
## Guide 2 — Database Administrator এর জন্য AWS Core Concepts

> **লক্ষ্য:** AWS এর যে concepts গুলো সরাসরি database কাজে লাগে সেগুলো DBA-relevant depth এ বোঝা।
> **Environment:** AWS Free Tier Account + AWS CLI
> **পরবর্তী:** Guide 3 — AWS Cloud Native DBA (RDS, Aurora, এবং সব managed services)

---

## Table of Contents

0. **[Free Tier Setup এবং Sequential Practice Guide](#0-free-tier-setup-এবং-sequential-practice-guide)**
   - 0.1 [Free Tier কী কী পাবো — DBA এর জন্য](#01-free-tier-কী-কী-পাবো--dba-এর-জন্য)
   - 0.2 [Account Setup — প্রথমেই করো](#02-account-setup--প্রথমেই-করো)
   - 0.3 [Billing Alert Setup — Unexpected Charge এড়াতে](#03-billing-alert-setup--unexpected-charge-এড়াতে)
   - 0.4 [AWS CLI Install এবং Configure করো](#04-aws-cli-install-এবং-configure-করো)
   - 0.5 [Sequential Practice Plan — Step by Step](#05-sequential-practice-plan--step-by-step)
   - 0.6 [Free Tier এ কী সাবধান থাকতে হবে](#06-free-tier-এ-কী-সাবধান-থাকতে-হবে)
   - 0.7 [Practice শেষে Cleanup করো — Charge থেকে বাঁচো](#07-practice-শেষে-cleanup-করো--charge-থেকে-বাঁচো)

1. **[AWS Global Infrastructure](#1-aws-global-infrastructure)**
   - 1.1 [Region, Availability Zone, এবং Edge Location](#11-region-availability-zone-এবং-edge-location)
   - 1.2 [DBA এর জন্য কোন Region choose করবো](#12-dba-এর-জন্য-কোন-region-choose-করবো)

2. **[IAM — Identity and Access Management](#2-iam--identity-and-access-management)**
   - 2.1 [IAM কী এবং কেন](#21-iam-কী-এবং-কেন)
   - 2.2 [Users, Groups, Roles, Policies](#22-users-groups-roles-policies)
   - 2.3 [Policy Document — কীভাবে লেখে](#23-policy-document--কীভাবে-লেখে)
   - 2.4 [IAM Roles — EC2 এবং RDS এর জন্য](#24-iam-roles--ec2-এবং-rds-এর-জন্য)
   - 2.5 [IAM Best Practices](#25-iam-best-practices)
   - 2.6 [Hands-on: IAM Setup করো](#26-hands-on-iam-setup-করো)

3. **[VPC — Virtual Private Cloud](#3-vpc--virtual-private-cloud)**
   - 3.1 [VPC কী — Data Center এর Virtual Version](#31-vpc-কী--data-center-এর-virtual-version)
   - 3.2 [Subnet — Public এবং Private](#32-subnet--public-এবং-private)
   - 3.3 [Route Table](#33-route-table)
   - 3.4 [Internet Gateway এবং NAT Gateway](#34-internet-gateway-এবং-nat-gateway)
   - 3.5 [Security Group — Instance-level Firewall](#35-security-group--instance-level-firewall)
   - 3.6 [Network ACL — Subnet-level Firewall](#36-network-acl--subnet-level-firewall)
   - 3.7 [VPC Peering](#37-vpc-peering)
   - 3.8 [DBA এর জন্য Typical VPC Architecture](#38-dba-এর-জন্য-typical-vpc-architecture)
   - 3.9 [Hands-on: VPC এবং Security Group তৈরি করো](#39-hands-on-vpc-এবং-security-group-তৈরি-করো)

4. **[EC2 — Elastic Compute Cloud](#4-ec2--elastic-compute-cloud)**
   - 4.1 [EC2 কী — DBA এর কেন জানা দরকার](#41-ec2-কী--dba-এর-কেন-জানা-দরকার)
   - 4.2 [Instance Types — Database এর জন্য কোনটা](#42-instance-types--database-এর-জন্য-কোনটা)
   - 4.3 [AMI — Amazon Machine Image](#43-ami--amazon-machine-image)
   - 4.4 [Key Pair এবং SSH Access](#44-key-pair-এবং-ssh-access)
   - 4.5 [Bastion Host — Database এ Secure Access](#45-bastion-host--database-এ-secure-access)
   - 4.6 [User Data — Instance Start এ Automation](#46-user-data--instance-start-এ-automation)
   - 4.7 [Hands-on: Bastion Host তৈরি করো](#47-hands-on-bastion-host-তৈরি-করো)

5. **[EBS — Elastic Block Store](#5-ebs--elastic-block-store)**
   - 5.1 [EBS কী — Database Storage](#51-ebs-কী--database-storage)
   - 5.2 [Volume Types — Database এর জন্য কোনটা](#52-volume-types--database-এর-জন্য-কোনটা)
   - 5.3 [EBS Snapshot — Backup](#53-ebs-snapshot--backup)
   - 5.4 [EBS Optimization Tips](#54-ebs-optimization-tips)
   - 5.5 [Hands-on: EBS Volume Manage করো](#55-hands-on-ebs-volume-manage-করো)

6. **[S3 — Simple Storage Service](#6-s3--simple-storage-service)**
   - 6.1 [S3 কী — Database Backup এর জন্য](#61-s3-কী--database-backup-এর-জন্য)
   - 6.2 [Bucket, Object, এবং Key](#62-bucket-object-এবং-key)
   - 6.3 [Storage Classes — Cost Optimization](#63-storage-classes--cost-optimization)
   - 6.4 [S3 Lifecycle Policy](#64-s3-lifecycle-policy)
   - 6.5 [S3 দিয়ে pgBackRest Configure করো](#65-s3-দিয়ে-pgbackrest-configure-করো)
   - 6.6 [Hands-on: S3 Bucket এবং Backup Setup](#66-hands-on-s3-bucket-এবং-backup-setup)

7. **[CloudWatch — Monitoring এবং Alerting](#7-cloudwatch--monitoring-এবং-alerting)**
   - 7.1 [CloudWatch কী — DBA এর Monitoring Tool](#71-cloudwatch-কী--dba-এর-monitoring-tool)
   - 7.2 [Metrics — Database কী Monitor করে](#72-metrics--database-কী-monitor-করে)
   - 7.3 [Log Groups এবং Log Insights](#73-log-groups-এবং-log-insights)
   - 7.4 [Alarms — Alert Setup](#74-alarms--alert-setup)
   - 7.5 [Dashboards তৈরি করো](#75-dashboards-তৈরি-করো)
   - 7.6 [Hands-on: Database Alarm Setup করো](#76-hands-on-database-alarm-setup-করো)

8. **[Secrets Manager এবং Parameter Store](#8-secrets-manager-এবং-parameter-store)**
   - 8.1 [কেন দরকার — Password Management](#81-কেন-দরকার--password-management)
   - 8.2 [Secrets Manager — Database Password Rotation](#82-secrets-manager--database-password-rotation)
   - 8.3 [Parameter Store — Configuration Management](#83-parameter-store--configuration-management)
   - 8.4 [Hands-on: Database Password Secrets Manager এ রাখো](#84-hands-on-database-password-secrets-manager-এ-রাখো)

9. **[KMS — Key Management Service](#9-kms--key-management-service)**
   - 9.1 [KMS কী — Encryption at Rest](#91-kms-কী--encryption-at-rest)
   - 9.2 [CMK vs AWS Managed Key](#92-cmk-vs-aws-managed-key)
   - 9.3 [Database Encryption এ KMS](#93-database-encryption-এ-kms)
   - 9.4 [Hands-on: Encrypted RDS Setup](#94-hands-on-encrypted-rds-setup)

10. **[Route 53 — DNS Management](#10-route-53--dns-management)**
    - 10.1 [Route 53 কী — Database Endpoint এর জন্য](#101-route-53-কী--database-endpoint-এর-জন্য)
    - 10.2 [Record Types — A, CNAME, Alias](#102-record-types--a-cname-alias)
    - 10.3 [Failover Routing — HA এর জন্য](#103-failover-routing--ha-এর-জন্য)
    - 10.4 [Hands-on: Database Failover DNS Setup](#104-hands-on-database-failover-dns-setup)

11. **[AWS CLI — Command Line থেকে সব কাজ](#11-aws-cli--command-line-থেকে-সব-কাজ)**
    - 11.1 [Install এবং Configure করো](#111-install-এবং-configure-করো)
    - 11.2 [DBA Daily Use Commands](#112-dba-daily-use-commands)
    - 11.3 [AWS CLI দিয়ে Scripting](#113-aws-cli-দিয়ে-scripting)

12. **[Cost Awareness — DBA এর জন্য](#12-cost-awareness--dba-এর-জন্য)**
    - 12.1 [কোন Service কতটা খরচ করে](#121-কোন-service-কতটা-খরচ-করে)
    - 12.2 [Cost Optimization Tips](#122-cost-optimization-tips)
    - 12.3 [AWS Cost Explorer ব্যবহার করো](#123-aws-cost-explorer-ব্যবহার-করো)

13. **[Free Tier — Step-by-Step Hands-on Labs](#13-free-tier--step-by-step-hands-on-lab)**
    - 13.0 [Free Tier এ কী কী পাবে](#130-free-tier-এ-কী-কী-পাবে-এবং-সাবধানতা)
    - 13.1 [Lab 0 — Account Setup এবং Safety Net](#131-lab-0--account-setup-এবং-safety-net)
    - 13.2 [Lab 1 — IAM: Users, Groups, Policies](#132-lab-1--iam-users-groups-policies)
    - 13.3 [Lab 2 — VPC: Network তৈরি করো](#133-lab-2--vpc-network-তৈরি-করো)
    - 13.4 [Lab 3 — EC2: Bastion Host তৈরি করো](#134-lab-3--ec2-bastion-host-তৈরি-করো)
    - 13.5 [Lab 4 — S3: Backup Bucket তৈরি করো](#135-lab-4--s3-backup-bucket-তৈরি-করো)
    - 13.6 [Lab 5 — RDS: PostgreSQL Database তৈরি করো](#136-lab-5--rds-postgresql-database-তৈরি-করো)
    - 13.7 [Lab 6 — RDS Snapshot এবং Restore](#137-lab-6--rds-snapshot-এবং-restore)
    - 13.8 [Lab 7 — CloudWatch: Monitoring Setup](#138-lab-7--cloudwatch-monitoring-setup-করো)
    - 13.9 [Lab 8 — Parameter Store: Configuration Management](#139-lab-8--parameter-store-configuration-management)
    - 13.10 [Lab 9 — Self-managed PostgreSQL on EC2](#1310-lab-9--self-managed-postgresql-on-ec2)
    - 13.11 [Lab 10 — Streaming Replication on EC2](#1311-lab-10--streaming-replication-on-ec2)
    - 13.12 [Lab Cleanup — সব Resources Delete করো](#1312-lab-cleanup--সব-resources-delete-করো)
    - 13.13 [Lab Sequence — কোন Order এ করবো](#1313-lab-sequence--কোন-order-এ-করবো)
    - 13.14 [Common Free Tier Mistakes](#1314-common-free-tier-mistakes--এড়িয়ে-চলো)

14. **[Advanced Networking — VPC Flow Logs, CloudTrail, VPC Endpoints](#14-advanced-networking)**
    - 14.1 [VPC Flow Logs — Network Traffic Monitoring](#141-vpc-flow-logs--network-traffic-monitoring)
    - 14.2 [CloudTrail — AWS API Call Audit](#142-cloudtrail--aws-api-call-audit)
    - 14.3 [VPC Endpoints — S3 ছাড়া NAT Gateway](#143-vpc-endpoints--s3-access-without-nat-gateway)
    - 14.4 [Systems Manager Session Manager](#144-systems-manager-session-manager--ssh-key-ছাড়া-ec2-access)

15. **[Connectivity — VPN, Direct Connect, ALB, Transit Gateway](#15-connectivity)**
    - 15.1 [Site-to-Site VPN — On-premise to AWS](#151-site-to-site-vpn--on-premise-to-aws)
    - 15.2 [AWS Direct Connect — Dedicated Private Line](#152-aws-direct-connect--dedicated-private-line)
    - 15.3 [Transit Gateway — Multi-VPC Connectivity](#153-transit-gateway--multi-vpc-connectivity)
    - 15.4 [NLB — PostgreSQL Connection Load Balancing](#154-application-load-balancer-alb--database-routing)
    - 15.5 [AWS Organizations — Multi-account Setup](#155-aws-organizations--multi-account-setup)

---

# 1. AWS Global Infrastructure

## 1.1 Region, Availability Zone, এবং Edge Location

AWS এর physical infrastructure তিনটা layer এ ভাগ করা।

```
┌─────────────────────────────────────────────────────────┐
│                    AWS Region                           │
│               (e.g., ap-southeast-1)                   │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │     AZ-1a    │  │     AZ-1b    │  │     AZ-1c    │  │
│  │  Data Center │  │  Data Center │  │  Data Center │  │
│  │  (physical)  │  │  (physical)  │  │  (physical)  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│  AZ গুলো physically আলাদা কিন্তু                       │
│  low-latency fiber দিয়ে connected                      │
└─────────────────────────────────────────────────────────┘
           │
           │ Edge Locations (100+ globally)
           │ CloudFront CDN, Route 53 DNS
           ▼
       End Users
```

**Region:**
- একটা geographic area — যেমন: Singapore (`ap-southeast-1`), Ireland (`eu-west-1`), N. Virginia (`us-east-1`)
- প্রতিটা Region এ কমপক্ষে ২টা AZ থাকে
- Regions একে অপরের থেকে completely isolated (data sovereignty)
- Global AWS service গুলো (IAM, Route 53, CloudFront) region-independent

**Availability Zone (AZ):**
- Region এর মধ্যে একটা বা একাধিক physical data center
- একটা AZ fail করলে অন্যটা চালু থাকে
- RDS Multi-AZ deployment → primary এক AZ তে, standby অন্য AZ তে
- নাম: `ap-southeast-1a`, `ap-southeast-1b`, `ap-southeast-1c`

**Edge Location:**
- CloudFront CDN এবং Route 53 DNS এর জন্য
- Database সরাসরি এখানে থাকে না

---

## 1.2 DBA এর জন্য কোন Region Choose করবো

```
Decision factors:

1. Data Residency / Compliance:
   Bangladesh এর data Bangladesh এ রাখতে হবে?
   → Nearest: ap-south-1 (Mumbai) বা ap-southeast-1 (Singapore)
   → GDPR: eu-west-1 (Ireland) বা eu-central-1 (Frankfurt)

2. Latency:
   Application server কোথায়? Database কাছাকাছি রাখো।
   Application: Singapore → Database: ap-southeast-1
   Ping test: https://www.cloudping.info

3. Service Availability:
   সব AWS service সব region এ নেই।
   Aurora Serverless v2, newer instance types → us-east-1 এ আগে আসে
   Check: https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/

4. Cost:
   us-east-1 generally সস্তা।
   ap-southeast-1 (Singapore) 10-20% বেশি খরচ।

5. Disaster Recovery:
   Primary: ap-southeast-1 (Singapore)
   DR Site: ap-south-1 (Mumbai) — কাছাকাছি কিন্তু আলাদা region

DBA Recommendation:
  Bangladesh/South Asia workload:
  Primary: ap-southeast-1 (Singapore)
  DR: ap-south-1 (Mumbai)
```

```bash
# AWS CLI দিয়ে regions দেখো
aws ec2 describe-regions --output table

# Current configured region দেখো
aws configure get region

# Region change করো
aws configure set region ap-southeast-1
```

---

# 2. IAM — Identity and Access Management

## 2.1 IAM কী এবং কেন

```
On-premise এ:
  Database server এ OS user ছিল
  DB user ছিল (postgres, app_user)
  Firewall rule ছিল

AWS এ:
  কে কোন AWS resource access করতে পারবে → IAM
  "Alice কি RDS instance এর snapshot নিতে পারবে?"
  "EC2 instance কি S3 bucket এ backup রাখতে পারবে?"
  এই সব প্রশ্নের উত্তর IAM দেয়।

IAM = AWS এর Security এবং Access Control layer
     → Global service (region নেই)
     → Free of charge
```

---

## 2.2 Users, Groups, Roles, Policies

```
IAM এর চারটা মূল concept:

┌──────────────────────────────────────────────────────────┐
│  Policy (What can be done?)                              │
│  JSON document যে বলে কোন action allow বা deny          │
│  "S3 এ PutObject করতে পারবে"                            │
│  "RDS restart করতে পারবে না"                            │
└──────────────────────┬───────────────────────────────────┘
                       │ attached to
          ┌────────────┴───────────────┐
          ▼                            ▼
┌─────────────────┐         ┌─────────────────────┐
│  User           │         │  Role               │
│  (Human/API)    │         │  (AWS Service/App)  │
│  Alice, Bob     │         │  EC2 instance role  │
│  CI/CD pipeline │         │  Lambda role        │
└────────┬────────┘         └─────────────────────┘
         │ member of
         ▼
┌─────────────────┐
│  Group          │
│  dba-team       │
│  dev-team       │
└─────────────────┘
```

**User:**
- Human বা application এর identity
- Username + Password (Console) বা Access Key + Secret Key (CLI/API)
- Direct policy attach করা যায় কিন্তু best practice নয়

**Group:**
- Users এর collection
- Policy Group এ attach করো, User কে Group এ রাখো
- `dba-team` group → সব DBA user এখানে → একটা policy change সবার জন্য apply হয়

**Role:**
- AWS Service বা application এর temporary identity
- Password নেই, Access Key নেই
- EC2 instance কে S3 access দিতে → Role তৈরি করো → EC2 তে assign করো
- **DBA এর জন্য most important:** RDS কে S3 access দেওয়া, Lambda কে DB access দেওয়া

**Policy:**
- JSON document — কী করতে পারবে, কী পারবে না
- Managed Policy: AWS তৈরি (AmazonRDSFullAccess) বা Customer তৈরি
- Inline Policy: directly user/group/role এ attached

---

## 2.3 Policy Document — কীভাবে লেখে

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowRDSRead",
            "Effect": "Allow",
            "Action": [
                "rds:DescribeDBInstances",
                "rds:DescribeDBSnapshots",
                "rds:ListTagsForResource"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowRDSSnapshotCreate",
            "Effect": "Allow",
            "Action": [
                "rds:CreateDBSnapshot",
                "rds:CopyDBSnapshot"
            ],
            "Resource": "arn:aws:rds:ap-southeast-1:123456789012:db:mydb-prod"
        },
        {
            "Sid": "DenyRDSDelete",
            "Effect": "Deny",
            "Action": [
                "rds:DeleteDBInstance",
                "rds:DeleteDBCluster"
            ],
            "Resource": "*"
        }
    ]
}
```

**Policy এর elements:**

| Element | মানে | Example |
|---|---|---|
| `Version` | Policy language version | `"2012-10-17"` (always এটাই) |
| `Statement` | Rules এর array | Multiple rules থাকতে পারে |
| `Sid` | Statement ID (optional) | Human-readable name |
| `Effect` | Allow বা Deny | `"Allow"` বা `"Deny"` |
| `Action` | কোন API call | `"rds:CreateDBSnapshot"` |
| `Resource` | কোন resource এ | ARN বা `"*"` (সব) |
| `Condition` | Extra conditions | IP range, MFA required, etc. |

**ARN — Amazon Resource Name:**
```
arn:aws:rds:ap-southeast-1:123456789012:db:mydb-prod
     │    │       │               │          │    │
     │    │       │               │          │    └── resource name
     │    │       │               │          └─────── resource type
     │    │       │               └────────────────── account ID
     │    │       └────────────────────────────────── region
     │    └────────────────────────────────────────── service
     └─────────────────────────────────────────────── prefix
```

---

## 2.4 IAM Roles — EC2 এবং RDS এর জন্য

### EC2 Instance Role (Most Common DBA Use Case)

```
Problem: EC2 তে চলা application কে S3 তে backup রাখতে হবে।
Bad solution: Access Key/Secret Key hardcode করা (security risk!)
Good solution: EC2 Instance Role

EC2 → IAM Role → S3 Access Policy
Application automatically role এর credentials পায় (temporary, auto-rotated)
```

```bash
# AWS CLI দিয়ে role তৈরি করো
# Step 1: Trust Policy (EC2 এই role assume করতে পারবে)
cat > /tmp/ec2-trust-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

# Role তৈরি করো
aws iam create-role \
    --role-name pg-backup-role \
    --assume-role-policy-document file:///tmp/ec2-trust-policy.json

# Permission Policy তৈরি করো (S3 backup bucket এ write)
cat > /tmp/s3-backup-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-pg-backup-bucket",
                "arn:aws:s3:::my-pg-backup-bucket/*"
            ]
        }
    ]
}
EOF

# Policy তৈরি করো এবং role এ attach করো
aws iam create-policy \
    --policy-name PGBackupS3Policy \
    --policy-document file:///tmp/s3-backup-policy.json

aws iam attach-role-policy \
    --role-name pg-backup-role \
    --policy-arn arn:aws:iam::123456789012:policy/PGBackupS3Policy

# Instance Profile তৈরি করো (EC2 তে attach করার জন্য)
aws iam create-instance-profile \
    --instance-profile-name pg-backup-profile

aws iam add-role-to-instance-profile \
    --instance-profile-name pg-backup-profile \
    --role-name pg-backup-role

# EC2 তে attach করো
aws ec2 associate-iam-instance-profile \
    --instance-id i-1234567890abcdef0 \
    --iam-instance-profile Name=pg-backup-profile
```

### RDS Enhanced Monitoring Role

```bash
# RDS Enhanced Monitoring এর জন্য role দরকার
# AWS Managed Policy ব্যবহার করো:
aws iam create-role \
    --role-name rds-monitoring-role \
    --assume-role-policy-document '{
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Principal": {"Service": "monitoring.rds.amazonaws.com"},
            "Action": "sts:AssumeRole"
        }]
    }'

aws iam attach-role-policy \
    --role-name rds-monitoring-role \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole
```

---

## 2.5 IAM Best Practices

```
DBA এর জন্য IAM Best Practices:

1. Root Account কখনো daily কাজে ব্যবহার করো না
   Root = সব permission, no restrictions
   → Root দিয়ে শুধু billing এবং account-level settings
   → Root এ MFA enable করো

2. Least Privilege Principle
   শুধু দরকারি permission দাও
   DBA user কে: RDS full access, CloudWatch read, S3 backup bucket access
   DBA user কে না: EC2 terminate, IAM user create

3. Groups ব্যবহার করো, Individual user এ policy না
   dba-team group → policy attach
   নতুন DBA → group এ add করো

4. Access Key regularly rotate করো
   AWS Console: IAM → Users → Security credentials → Rotate
   অথবা CLI:
   aws iam create-access-key --user-name alice
   # নতুন key তৈরির পরে পুরনো key delete করো
   aws iam delete-access-key --user-name alice \
       --access-key-id AKIAIOSFODNN7EXAMPLE

5. MFA Enable করো (especially console access)
   IAM → Users → Security credentials → MFA

6. Programmatic access এর জন্য Role ব্যবহার করো
   EC2/Lambda/ECS → Role (not access key)
   CI/CD pipeline → Role (not access key)

7. Permission Boundary ব্যবহার করো (advanced)
   Maximum permission set করো — user এর বেশি permission নিতে পারবে না
```

---

## 2.6 Hands-on: IAM Setup করো

> 💚 **FREE TIER:** IAM সম্পূর্ণ free। User, Group, Role, Policy যত তৈরি করো — কোনো charge নেই।

```bash
# ─── AWS CLI configure করো ───
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: ap-southeast-1
# Default output format: json

# Verify
aws sts get-caller-identity
# {
#     "UserId": "AIDAEXAMPLE",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/alice"
# }

# ─── DBA Group এবং Policy তৈরি করো ───
# DBA Group
aws iam create-group --group-name dba-team

# DBA Policy (RDS full + CloudWatch read + S3 backup)
cat > /tmp/dba-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "RDSFullAccess",
            "Effect": "Allow",
            "Action": ["rds:*"],
            "Resource": "*"
        },
        {
            "Sid": "CloudWatchRead",
            "Effect": "Allow",
            "Action": [
                "cloudwatch:GetMetricStatistics",
                "cloudwatch:ListMetrics",
                "cloudwatch:GetMetricData",
                "cloudwatch:DescribeAlarms",
                "logs:GetLogEvents",
                "logs:DescribeLogGroups",
                "logs:DescribeLogStreams",
                "logs:FilterLogEvents"
            ],
            "Resource": "*"
        },
        {
            "Sid": "S3BackupAccess",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject", "s3:GetObject",
                "s3:DeleteObject", "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Resource": [
                "arn:aws:s3:::my-pg-backup-*",
                "arn:aws:s3:::my-pg-backup-*/*"
            ]
        },
        {
            "Sid": "SecretsManagerDBAccess",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "arn:aws:secretsmanager:*:*:secret:database/*"
        },
        {
            "Sid": "DenyDangerousActions",
            "Effect": "Deny",
            "Action": [
                "rds:DeleteDBInstance",
                "rds:DeleteDBCluster",
                "rds:DeleteDBSnapshot",
                "rds:DeleteDBClusterSnapshot"
            ],
            "Resource": "*",
            "Condition": {
                "StringNotEquals": {
                    "aws:RequestedRegion": "ap-southeast-1"
                }
            }
        }
    ]
}
EOF

aws iam create-policy \
    --policy-name DBATeamPolicy \
    --policy-document file:///tmp/dba-policy.json \
    --description "Policy for DBA team members"

# Group এ policy attach করো
aws iam attach-group-policy \
    --group-name dba-team \
    --policy-arn arn:aws:iam::123456789012:policy/DBATeamPolicy

# DBA User তৈরি করো
aws iam create-user --user-name dba-alice

# Group এ add করো
aws iam add-user-to-group \
    --user-name dba-alice \
    --group-name dba-team

# Console access এর জন্য password দাও
aws iam create-login-profile \
    --user-name dba-alice \
    --password 'TempPass@2024!' \
    --password-reset-required

# CLI access এর জন্য access key তৈরি করো
aws iam create-access-key --user-name dba-alice
# Output: AccessKeyId, SecretAccessKey → store করো securely

# ─── Verify ───
aws iam list-users
aws iam list-groups-for-user --user-name dba-alice
aws iam list-attached-group-policies --group-name dba-team

# Simulate: কোনো action allowed কিনা test করো
aws iam simulate-principal-policy \
    --policy-source-arn arn:aws:iam::123456789012:user/dba-alice \
    --action-names rds:DescribeDBInstances rds:DeleteDBInstance \
    --resource-arns "*"
# rds:DescribeDBInstances → "allowed"
# rds:DeleteDBInstance → "explicitDeny" (ভালো!)
```

---

# 3. VPC — Virtual Private Cloud

## 3.1 VPC কী — Data Center এর Virtual Version

```
On-premise এ তোমার নিজস্ব physical network ছিল।
AWS এ VPC = তোমার নিজস্ব virtual network।

VPC এর মধ্যে:
  - তোমার EC2 instances
  - RDS databases
  - Load Balancers
  - সব এরা একে অপরের সাথে কথা বলতে পারে

VPC এর বাইরে:
  - Internet (public)
  - অন্য AWS account
  - On-premise network

Default VPC:
  প্রতিটা AWS account এ প্রতিটা region এ একটা Default VPC আছে।
  Production এ নিজের VPC তৈরি করো — Default VPC ব্যবহার করো না।
```

**CIDR Block — VPC এর IP Range:**
```
VPC CIDR: 10.0.0.0/16
  মানে: 10.0.0.0 থেকে 10.0.255.255 = 65,536 IP addresses

Subnet CIDR: 10.0.1.0/24
  মানে: 10.0.1.0 থেকে 10.0.1.255 = 256 IP addresses
  (AWS 5টা reserve করে → 251 usable)

Common VPC CIDR:
  10.0.0.0/16  (65K IPs)
  172.16.0.0/16 (65K IPs)
  192.168.0.0/16 (65K IPs)
```

---

## 3.2 Subnet — Public এবং Private

```
VPC CIDR: 10.0.0.0/16

┌─────────────────────────────────────────────────────────┐
│                    VPC: 10.0.0.0/16                     │
│                                                         │
│  ┌──────────────────┐    ┌──────────────────┐           │
│  │  Public Subnet   │    │  Public Subnet   │           │
│  │  10.0.1.0/24     │    │  10.0.2.0/24     │           │
│  │  AZ-1a           │    │  AZ-1b           │           │
│  │                  │    │                  │           │
│  │  Bastion Host    │    │  NAT Gateway     │           │
│  │  Load Balancer   │    │                  │           │
│  └──────────────────┘    └──────────────────┘           │
│                                                         │
│  ┌──────────────────┐    ┌──────────────────┐           │
│  │  Private Subnet  │    │  Private Subnet  │           │
│  │  10.0.10.0/24    │    │  10.0.11.0/24    │           │
│  │  AZ-1a           │    │  AZ-1b           │           │
│  │                  │    │                  │           │
│  │  App Servers     │    │  App Servers     │           │
│  └──────────────────┘    └──────────────────┘           │
│                                                         │
│  ┌──────────────────┐    ┌──────────────────┐           │
│  │  DB Subnet       │    │  DB Subnet       │           │
│  │  10.0.20.0/24    │    │  10.0.21.0/24    │           │
│  │  AZ-1a           │    │  AZ-1b           │           │
│  │                  │    │                  │           │
│  │  RDS Primary     │    │  RDS Standby     │           │
│  └──────────────────┘    └──────────────────┘           │
└─────────────────────────────────────────────────────────┘
```

**Public Subnet:**
- Internet Gateway এর সাথে connected
- Resources এর public IP থাকতে পারে
- Internet থেকে directly accessible
- কী রাখবো: Bastion Host, Load Balancer, NAT Gateway

**Private Subnet:**
- Internet এ directly accessible নয়
- Internet এ যেতে NAT Gateway ব্যবহার করে (outbound only)
- কী রাখবো: Application Servers, Databases (most important!)

**DB Subnet Group (RDS এর জন্য):**
- কমপক্ষে ২টা AZ তে ২টা subnet দরকার
- RDS Multi-AZ deployment এর জন্য mandatory

---

## 3.3 Route Table

```
Route Table = Network traffic কোথায় যাবে সেটার নিয়ম

Public Subnet Route Table:
  Destination    Target
  10.0.0.0/16   local         (VPC internal traffic)
  0.0.0.0/0     igw-xxxxxx    (Internet Gateway — internet যেতে)

Private Subnet Route Table:
  Destination    Target
  10.0.0.0/16   local         (VPC internal traffic)
  0.0.0.0/0     nat-xxxxxx    (NAT Gateway — outbound internet only)

DB Subnet Route Table:
  Destination    Target
  10.0.0.0/16   local         (VPC internal traffic only)
  (no 0.0.0.0/0 — no internet access!)
```

---

## 3.4 Internet Gateway এবং NAT Gateway

**Internet Gateway (IGW):**
```
VPC ↔ Internet (bidirectional)
Public subnet এর traffic internet এ যায় IGW দিয়ে
এবং internet থেকে আসে IGW দিয়ে

একটা VPC তে একটাই IGW
Free of charge (data transfer charge আছে)
```

**NAT Gateway:**
```
Private subnet → Internet (outbound only)
Internet → Private subnet (না, block)

কেন দরকার?
Private subnet এর EC2/App Server:
  OS update ডাউনলোড করতে হবে
  External API call করতে হবে
  কিন্তু internet থেকে directly আসতে দেওয়া যাবে না

NAT Gateway:
  Public subnet এ থাকে
  Elastic IP (fixed public IP) থাকে
  Private subnet → NAT → Internet (তবে return traffic আসে)
  
Cost: ~$0.045/hour + data transfer
  Production এ significant cost হতে পারে
  DB subnet → NAT দরকার নেই (DB কে internet access দেওয়া উচিত না)
```

---

## 3.5 Security Group — Instance-level Firewall

```
Security Group = Virtual firewall for EC2/RDS instances
  Stateful: Inbound allow করলে outbound automatically allowed
  Default: সব inbound deny, সব outbound allow

Database Security Group Design:
┌──────────────────────────────────────────────────────┐
│  Security Group: sg-database                         │
│                                                      │
│  Inbound Rules:                                      │
│  Port 5432 (PostgreSQL)                             │
│    Source: sg-application (App server SG)           │
│    → শুধু app server থেকে DB access করতে পারবে    │
│                                                      │
│  Port 5432                                          │
│    Source: sg-bastion (Bastion Host SG)             │
│    → DBA bastion host থেকে access করতে পারবে       │
│                                                      │
│  Outbound Rules:                                    │
│  All traffic: Allow                                 │
│  (RDS এর জন্য outbound usually not needed)         │
└──────────────────────────────────────────────────────┘
```

**Security Group Rules:**

| Rule | Protocol | Port | Source/Destination | মানে |
|---|---|---|---|---|
| Inbound | TCP | 5432 | sg-app-servers | App server থেকে PostgreSQL |
| Inbound | TCP | 5432 | sg-bastion | Bastion থেকে DBA access |
| Inbound | TCP | 5432 | 10.0.0.0/16 | VPC internal (development) |
| Outbound | All | All | 0.0.0.0/0 | সব outbound |

**Important:** Security Group এ Source হিসেবে অন্য Security Group দাও — IP range নয়। কারণ IP change হলে rule update করতে হবে না।

---

## 3.6 Network ACL — Subnet-level Firewall

```
Network ACL (NACL):
  Subnet level এ firewall (Security Group = instance level)
  Stateless: Inbound allow করলে outbound আলাদাভাবে allow করতে হবে
  Rules: numbered, order এ evaluate হয়

Default NACL: সব allow
Custom NACL: explicitly allow/deny করতে হয়

DBA কখন NACL ব্যবহার করবে?
  Production DB subnet: নির্দিষ্ট IP range ছাড়া সব block
  Rule 100: Allow TCP 5432 from 10.0.0.0/16 (VPC)
  Rule 200: Deny TCP 5432 from 0.0.0.0/0 (internet)
  Rule *:   Deny all (default)

Security Group vs NACL:
  Security Group: Instance level, stateful, allow only
  NACL: Subnet level, stateless, allow and deny
  Production: দুটোই ব্যবহার করো (defense in depth)
```

---

## 3.7 VPC Peering

```
দুটো VPC একে অপরের সাথে privately communicate করতে পারে।

Use cases:
  ① Same account, different environment:
     VPC-prod ↔ VPC-dev (monitoring, staging)

  ② Different account:
     Company A এর DB VPC ↔ Company B এর App VPC

  ③ Cross-region:
     Singapore (production) ↔ Mumbai (DR)

Limitations:
  CIDR overlap থাকলে peering কাজ করে না
  Transitive peering নেই:
  A ↔ B ↔ C → A এবং C directly কথা বলতে পারবে না
  A ↔ C আলাদা peering দরকার

DBA perspective:
  Production DB → Monitoring VPC এ export করতে peering
  Cross-region replication setup এ peering
```

---

## 3.8 DBA এর জন্য Typical VPC Architecture

```bash
# ─── Production VPC Architecture ───

# VPC: 10.0.0.0/16
# Region: ap-southeast-1

# Subnets:
# Public AZ-1a:  10.0.1.0/24   (Bastion, NAT GW)
# Public AZ-1b:  10.0.2.0/24   (NAT GW HA)
# App AZ-1a:     10.0.10.0/24  (Application Servers)
# App AZ-1b:     10.0.11.0/24  (Application Servers)
# DB AZ-1a:      10.0.20.0/24  (RDS Primary)
# DB AZ-1b:      10.0.21.0/24  (RDS Standby)

# Security Groups:
# sg-bastion:     Inbound: 22/TCP from DBA office IP
# sg-app:         Inbound: 8080/TCP from ALB
# sg-database:    Inbound: 5432/TCP from sg-app, sg-bastion

# Traffic flow:
# DBA → Bastion (public, sg-bastion) → RDS (private, sg-database)
# User → Internet → ALB → App (private, sg-app) → RDS (private, sg-database)
# App → NAT GW → Internet (OS update, API calls)
# DB → No internet access
```

---

## 3.9 Hands-on: VPC এবং Security Group তৈরি করো

> 💚 **FREE TIER:** VPC, Subnet, Route Table, Internet Gateway, Security Group — সব free। **⚠️ NAT Gateway তৈরি করো না** ($0.045/hour charge)।

এই section এ Phase 2 এর সব steps করবো। সব resource এ `practice-` prefix এবং `Project=dba-practice` tag দাও।

```bash
# ─── Practice VPC তৈরি করো ───
# Region confirm করো
aws configure get region   # ap-southeast-1 হওয়া উচিত

# Step 1: VPC তৈরি করো
VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=practice-vpc},{Key=Project,Value=dba-practice}]' \
    --query 'Vpc.VpcId' --output text)
echo "VPC: $VPC_ID"

# DNS hostname enable করো (RDS endpoint resolution এর জন্য দরকার)
aws ec2 modify-vpc-attribute \
    --vpc-id $VPC_ID \
    --enable-dns-hostnames

aws ec2 modify-vpc-attribute \
    --vpc-id $VPC_ID \
    --enable-dns-support

# Step 2: Internet Gateway তৈরি এবং attach করো
IGW_ID=$(aws ec2 create-internet-gateway \
    --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=production-igw}]' \
    --query 'InternetGateway.InternetGatewayId' --output text)
echo "IGW: $IGW_ID"

aws ec2 attach-internet-gateway \
    --internet-gateway-id $IGW_ID \
    --vpc-id $VPC_ID

# Step 3: Subnets তৈরি করো
# Public Subnet AZ-1a
PUB_SUBNET_1A=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.1.0/24 \
    --availability-zone ap-southeast-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a}]' \
    --query 'Subnet.SubnetId' --output text)

# DB Subnet AZ-1a
DB_SUBNET_1A=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.20.0/24 \
    --availability-zone ap-southeast-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=db-1a}]' \
    --query 'Subnet.SubnetId' --output text)

# DB Subnet AZ-1b
DB_SUBNET_1B=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.21.0/24 \
    --availability-zone ap-southeast-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=db-1b}]' \
    --query 'Subnet.SubnetId' --output text)

echo "DB Subnets: $DB_SUBNET_1A, $DB_SUBNET_1B"

# Step 4: Route Table তৈরি করো
# Public route table (IGW দিয়ে internet)
PUB_RT=$(aws ec2 create-route-table \
    --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]' \
    --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route \
    --route-table-id $PUB_RT \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id $IGW_ID

aws ec2 associate-route-table \
    --subnet-id $PUB_SUBNET_1A \
    --route-table-id $PUB_RT

# Step 5: Security Groups তৈরি করো
# Bastion SG
SG_BASTION=$(aws ec2 create-security-group \
    --group-name sg-bastion \
    --description "Bastion Host Security Group" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)

# SSH from DBA office (replace with your IP)
aws ec2 authorize-security-group-ingress \
    --group-id $SG_BASTION \
    --protocol tcp --port 22 \
    --cidr 203.0.113.0/32  # DBA এর IP

# Database SG
SG_DB=$(aws ec2 create-security-group \
    --group-name sg-database \
    --description "Database Security Group" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)

# PostgreSQL from Bastion only
aws ec2 authorize-security-group-ingress \
    --group-id $SG_DB \
    --protocol tcp --port 5432 \
    --source-group $SG_BASTION

echo "Security Groups: Bastion=$SG_BASTION, DB=$SG_DB"

# Step 6: RDS DB Subnet Group তৈরি করো
aws rds create-db-subnet-group \
    --db-subnet-group-name production-db-subnet-group \
    --db-subnet-group-description "Production DB Subnet Group" \
    --subnet-ids $DB_SUBNET_1A $DB_SUBNET_1B

# ─── Verify ───
aws ec2 describe-vpcs --vpc-ids $VPC_ID --query 'Vpcs[0].[VpcId,CidrBlock,State]'
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
    --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
    --output table
```

---

# 4. EC2 — Elastic Compute Cloud

## 4.1 EC2 কী — DBA এর কেন জানা দরকার

```
EC2 = Virtual Server in the Cloud

DBA কেন EC2 জানবে?
  ① Self-managed PostgreSQL on EC2 চালাবে
  ② Bastion Host (EC2) দিয়ে RDS access করবে
  ③ pgBackRest server EC2 তে হতে পারে
  ④ Application server (EC2) এর সাথে DB connection troubleshoot করবে
  ⑤ Patroni/etcd cluster EC2 তে চলবে
```

---

## 4.2 Instance Types — Database এর জন্য কোনটা

```
Instance type naming: [Family][Generation].[Size]
Example: r7g.2xlarge

Family:
  m = General purpose (balanced CPU/memory)
  c = Compute optimized (high CPU)
  r = Memory optimized (high RAM) ← Database এর জন্য সবচেয়ে ভালো
  x = Extra memory (very high RAM)
  i = Storage optimized (high local NVMe SSD)

Generation:
  6, 7 = Latest (faster, cheaper)
  g = AWS Graviton (ARM, 40% cheaper, similar performance)

Size:
  nano, micro, small, medium, large
  xlarge, 2xlarge, 4xlarge, 8xlarge, 12xlarge, 16xlarge, 24xlarge

PostgreSQL এর জন্য:
  Development:  t3.medium বা t3.large (burstable, cheap)
  Production:   r7g.xlarge থেকে শুরু
  Large DB:     r7g.4xlarge বা r7g.8xlarge
  Very large:   x2gd.4xlarge (local NVMe + high RAM)
```

| Instance | vCPU | RAM | Network | Use Case |
|---|---|---|---|---|
| t3.medium | 2 | 4GB | Moderate | Dev/Test |
| r7g.large | 2 | 16GB | Up to 12.5Gbps | Small prod |
| r7g.xlarge | 4 | 32GB | Up to 12.5Gbps | Medium prod |
| r7g.2xlarge | 8 | 64GB | Up to 12.5Gbps | Large prod |
| r7g.4xlarge | 16 | 128GB | Up to 12.5Gbps | Very large |
| r7g.8xlarge | 32 | 256GB | 12.5Gbps | Enterprise |

---

## 4.3 AMI — Amazon Machine Image

```
AMI = Pre-configured OS image
  OS + Software + Configuration এর snapshot

Types:
  AWS Managed: Amazon Linux 2023, Ubuntu, RHEL
  Marketplace: PostgreSQL pre-installed images
  Custom: তোমার নিজের তৈরি (PostgreSQL installed, configured)

DBA এর জন্য:
  Rocky Linux 9 → RHEL compatible AMI খোঁজো
  Amazon Linux 2023 → PostgreSQL install করো (dnf)
  Custom AMI: একটা server configure করো, AMI তৈরি করো → সব নতুন server identical
```

```bash
# Amazon Linux 2023 এ PostgreSQL 16 install
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql16-server postgresql16-contrib

# Custom AMI তৈরি করো (configured server থেকে)
aws ec2 create-image \
    --instance-id i-1234567890abcdef0 \
    --name "PostgreSQL-16-Base-AMI-$(date +%Y%m%d)" \
    --description "PostgreSQL 16 on Amazon Linux 2023" \
    --no-reboot

# AMI ID পাবে → নতুন instance launch এ এই AMI ব্যবহার করো
```

---

## 4.4 Key Pair এবং SSH Access

```bash
# ─── Key Pair তৈরি করো ───
aws ec2 create-key-pair \
    --key-name dba-prod-key \
    --key-type rsa \
    --key-format pem \
    --query 'KeyMaterial' \
    --output text > ~/.ssh/dba-prod-key.pem

chmod 400 ~/.ssh/dba-prod-key.pem

# ─── EC2 তে SSH করো ───
ssh -i ~/.ssh/dba-prod-key.pem ec2-user@52.1.2.3  # Amazon Linux
ssh -i ~/.ssh/dba-prod-key.pem ubuntu@52.1.2.3     # Ubuntu

# ─── SSH Config file এ save করো ───
cat >> ~/.ssh/config << 'EOF'
Host bastion-prod
    HostName 52.1.2.3       # Bastion public IP
    User ec2-user
    IdentityFile ~/.ssh/dba-prod-key.pem
    ServerAliveInterval 60

Host rds-prod
    HostName 10.0.20.100    # RDS private IP (bastion এর মাধ্যমে)
    User ec2-user
    ProxyJump bastion-prod
    IdentityFile ~/.ssh/dba-prod-key.pem
EOF

# Direct access:
ssh bastion-prod

# RDS এ port forward:
ssh -L 5433:mydb.cluster-xxxx.ap-southeast-1.rds.amazonaws.com:5432 bastion-prod
# এখন local 5433 → RDS 5432
psql -h localhost -p 5433 -U postgres -d mydb
```

---

## 4.5 Bastion Host — Database এ Secure Access

```
Problem: RDS private subnet এ আছে, internet থেকে access নেই।
DBA কীভাবে connect করবে?

Solution: Bastion Host (Jump Server)
  Public subnet এ একটা small EC2
  DBA → Bastion (public, SSH) → RDS (private, psql)
  শুধু DBA এর IP Bastion এ allow করো

Architecture:
  Internet → [DBA PC] → SSH → [Bastion EC2, Public Subnet]
                                    ↓ psql
                              [RDS, Private Subnet]
```

```bash
# ─── Bastion Host Best Practices ───

# 1. Minimum size: t3.nano বা t3.micro (শুধু SSH forwarding)
# 2. No storage except OS
# 3. Security Group: শুধু DBA এর IP এবং port 22
# 4. No permanent data store করো না
# 5. Session Manager ব্যবহার করো (SSH Key ছাড়া) — AWS SSM

# AWS Systems Manager Session Manager (SSH key ছাড়া):
aws ssm start-session --target i-1234567890abcdef0

# Port forwarding SSM দিয়ে (no SSH key needed!):
aws ssm start-session \
    --target i-1234567890abcdef0 \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters '{
        "host": ["mydb.xxxx.ap-southeast-1.rds.amazonaws.com"],
        "portNumber": ["5432"],
        "localPortNumber": ["5433"]
    }'
# Local 5433 → RDS 5432 (SSH key ছাড়া!)
psql -h localhost -p 5433 -U postgres -d mydb
```

---

## 4.6 User Data — Instance Start এ Automation

```bash
# EC2 launch এর সময় automatically script চালানো

aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --instance-type r7g.xlarge \
    --key-name dba-prod-key \
    --security-group-ids $SG_DB \
    --subnet-id $DB_SUBNET_1A \
    --iam-instance-profile Name=pg-backup-profile \
    --user-data '#!/bin/bash
# PostgreSQL install এবং configure করো
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf -qy module disable postgresql
dnf install -y postgresql16-server postgresql16-contrib pgbackrest
/usr/pgsql-16/bin/postgresql-16-setup initdb
systemctl enable --now postgresql-16
sudo -u postgres psql -c "ALTER USER postgres PASSWORD '"'"'StrongPass@2024!'"'"';"
echo "PostgreSQL setup complete" > /tmp/setup.log
' \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=pg-primary}]'
```

---

## 4.7 Hands-on: EC2 Launch এবং PostgreSQL Install — Complete Step-by-Step

> 💚 **FREE TIER:** t3.micro instance = 750 hours/month free। Key Pair free। **Practice শেষে instance STOP করো** (terminate নয় — data থাকবে)।

এই section এ Phase 3 এর সব steps একসাথে করবো।

### Step 1: Amazon Linux 2023 AMI ID খোঁজো

```bash
# ap-southeast-1 region এ latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
    --owners amazon \
    --filters \
        "Name=name,Values=al2023-ami-2023*" \
        "Name=architecture,Values=x86_64" \
        "Name=state,Values=available" \
    --query "sort_by(Images, &CreationDate)[-1].ImageId" \
    --output text \
    --region ap-southeast-1)

echo "Latest Amazon Linux 2023 AMI: $AMI_ID"
# ami-0xxxxxxxxxxxxxxxxx দেখাবে
```

### Step 2: Key Pair তৈরি করো

```bash
# Key Pair তৈরি করো
aws ec2 create-key-pair \
    --key-name practice-keypair \
    --key-type rsa \
    --query 'KeyMaterial' \
    --output text \
    --region ap-southeast-1 > ~/.ssh/practice-keypair.pem

# Permission set করো (SSH এর জন্য 400 দরকার)
chmod 400 ~/.ssh/practice-keypair.pem
echo "Key saved: ~/.ssh/practice-keypair.pem"

# Verify
aws ec2 describe-key-pairs \
    --key-names practice-keypair \
    --query 'KeyPairs[0].[KeyName,KeyFingerprint]' \
    --output table
```

### Step 3: Security Group তৈরি করো

```bash
# Phase 2 (VPC hands-on) থেকে VPC ID নাও
VPC_ID=$(aws ec2 describe-vpcs \
    --filters "Name=tag:Name,Values=practice-vpc" \
    --query 'Vpcs[0].VpcId' \
    --output text)

echo "VPC ID: $VPC_ID"

# Security Group তৈরি করো
SG_ID=$(aws ec2 create-security-group \
    --group-name practice-sg-ec2 \
    --description "Practice EC2 Security Group" \
    --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=practice-sg-ec2},{Key=Project,Value=dba-practice}]' \
    --query 'GroupId' \
    --output text)

echo "Security Group: $SG_ID"

# তোমার IP দিয়ে SSH allow করো
MY_IP=$(curl -s https://checkip.amazonaws.com)/32
echo "Your IP: $MY_IP"

aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 22 \
    --cidr $MY_IP

echo "SSH (port 22) allowed from $MY_IP"
```

### Step 4: EC2 Instance Launch করো

```bash
# Public Subnet ID নাও
PUB_SUBNET=$(aws ec2 describe-subnets \
    --filters "Name=tag:Name,Values=practice-subnet-public-1a" \
    --query 'Subnets[0].SubnetId' \
    --output text)

echo "Public Subnet: $PUB_SUBNET"

# EC2 Launch করো
INSTANCE_ID=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.micro \
    --key-name practice-keypair \
    --security-group-ids $SG_ID \
    --subnet-id $PUB_SUBNET \
    --associate-public-ip-address \
    --tag-specifications \
        'ResourceType=instance,Tags=[{Key=Name,Value=practice-ec2-postgres},{Key=Project,Value=dba-practice}]' \
        'ResourceType=volume,Tags=[{Key=Name,Value=practice-ebs-root},{Key=Project,Value=dba-practice}]' \
    --user-data '#!/bin/bash
hostnamectl set-hostname practice-pg-server
echo "Instance setup started" >> /tmp/setup.log
' \
    --query 'Instances[0].InstanceId' \
    --output text)

echo "Instance launched: $INSTANCE_ID"

# Running হওয়ার অপেক্ষা করো
echo "Waiting for instance to be running..."
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
echo "Instance is running!"

# Public IP পাও
PUBLIC_IP=$(aws ec2 describe-instances \
    --instance-ids $INSTANCE_ID \
    --query 'Reservations[0].Instances[0].PublicIpAddress' \
    --output text)

echo "Public IP: $PUBLIC_IP"
echo "SSH: ssh -i ~/.ssh/practice-keypair.pem ec2-user@$PUBLIC_IP"
```

### Step 5: SSH করো এবং PostgreSQL Install করো

```bash
# SSH করো
ssh -i ~/.ssh/practice-keypair.pem ec2-user@$PUBLIC_IP

# ─── EC2 এর ভেতরে নিচের commands চালাও ───

# System update করো
sudo dnf update -y

# PostgreSQL 16 install করো
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql16-server postgresql16-contrib

# Initialize এবং start করো
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl start postgresql-16
sudo systemctl enable postgresql-16

# Password set করো
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'Practice@2024!';"

# Verify
sudo -u postgres psql -c "SELECT version();"
# PostgreSQL 16.x দেখাবে ✅

# Exit করো
exit
```

### Step 6: Instance এ IAM Role Attach করো (S3 Access)

```bash
# ─── Local machine এ ─── (EC2 থেকে বের হওয়ার পরে)

# IAM Role এর জন্য Trust Policy
cat > /tmp/trust-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "ec2.amazonaws.com"},
        "Action": "sts:AssumeRole"
    }]
}
EOF

# Role তৈরি করো
aws iam create-role \
    --role-name practice-ec2-role \
    --assume-role-policy-document file:///tmp/trust-policy.json \
    --tags Key=Project,Value=dba-practice

# S3 Read/Write permission দাও
aws iam attach-role-policy \
    --role-name practice-ec2-role \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
# (Practice এ FullAccess, Production এ specific bucket policy)

# CloudWatch Logs permission দাও
aws iam attach-role-policy \
    --role-name practice-ec2-role \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# Instance Profile তৈরি করো
aws iam create-instance-profile \
    --instance-profile-name practice-ec2-profile

aws iam add-role-to-instance-profile \
    --instance-profile-name practice-ec2-profile \
    --role-name practice-ec2-role

# EC2 তে attach করো
aws ec2 associate-iam-instance-profile \
    --instance-id $INSTANCE_ID \
    --iam-instance-profile Name=practice-ec2-profile

echo "IAM Role attached!"

# Verify — EC2 এ SSH করে test করো:
ssh -i ~/.ssh/practice-keypair.pem ec2-user@$PUBLIC_IP \
    "aws sts get-caller-identity"
# practice-ec2-role এর ARN দেখাবে ✅
```

### Step 7: সব কিছু Verify করো

```bash
# EC2 Status
aws ec2 describe-instances \
    --instance-ids $INSTANCE_ID \
    --query 'Reservations[0].Instances[0].[InstanceId,State.Name,PublicIpAddress,InstanceType,IamInstanceProfile.Arn]' \
    --output table

# SSH করে PostgreSQL check করো
ssh -i ~/.ssh/practice-keypair.pem ec2-user@$PUBLIC_IP << 'REMOTE'
echo "=== PostgreSQL Status ==="
sudo systemctl status postgresql-16 --no-pager | head -5
sudo -u postgres psql -c "\l"
sudo -u postgres psql -c "SELECT version();"

echo "=== IAM Role Test ==="
aws sts get-caller-identity
aws s3 ls 2>/dev/null && echo "S3 access: OK" || echo "S3: No buckets yet"
REMOTE
```

### Step 8: Practice শেষে Instance Stop করো

```bash
# ─── Practice শেষে ─── (terminate নয়, stop করো!)
aws ec2 stop-instances --instance-ids $INSTANCE_ID
echo "Instance stopped. Stopped instance এ EBS storage charge হয় কিন্তু compute charge হয় না।"

# পরে resume করতে:
aws ec2 start-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
NEW_IP=$(aws ec2 describe-instances \
    --instance-ids $INSTANCE_ID \
    --query 'Reservations[0].Instances[0].PublicIpAddress' \
    --output text)
echo "New IP (IP change হতে পারে): $NEW_IP"

# ─── IP যেন change না হয় — Elastic IP ─── (optional)
# EIP allocate করো
EIP_ALLOC=$(aws ec2 allocate-address \
    --domain vpc \
    --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=practice-eip},{Key=Project,Value=dba-practice}]' \
    --query 'AllocationId' --output text)

# Instance এ associate করো
aws ec2 associate-address \
    --instance-id $INSTANCE_ID \
    --allocation-id $EIP_ALLOC

EIP=$(aws ec2 describe-addresses \
    --allocation-ids $EIP_ALLOC \
    --query 'Addresses[0].PublicIp' --output text)

echo "Elastic IP: $EIP (এখন IP change হবে না)"
echo "⚠️ Practice শেষে release করো: aws ec2 release-address --allocation-id $EIP_ALLOC"
```

---

# 5. EBS — Elastic Block Store

## 5.1 EBS কী — Database Storage

```
EBS = Persistent block storage for EC2
  EC2 instance বন্ধ হলেও data থাকে
  Network attached (instance থেকে আলাদা)
  Snapshot নিয়ে S3 তে store হয়

PostgreSQL এর জন্য:
  /var/lib/pgsql → EBS volume
  pg_wal → Separate EBS volume (performance, I/O isolation)
  Backup/archive → S3 (EBS না)
```

---

## 5.2 Volume Types — Database এর জন্য কোনটা

| Type | Name | IOPS | Throughput | Use Case | Cost |
|---|---|---|---|---|---|
| `gp3` | General Purpose SSD v3 | 3,000-16,000 | 125-1,000 MB/s | Most workloads ✅ | $ |
| `gp2` | General Purpose SSD v2 | Up to 16,000 | 250 MB/s | Legacy (use gp3) | $$ |
| `io2` | Provisioned IOPS SSD | Up to 256,000 | 4,000 MB/s | High IOPS production | $$$ |
| `io1` | Provisioned IOPS SSD v1 | Up to 64,000 | 1,000 MB/s | Legacy (use io2) | $$$ |
| `st1` | Throughput HDD | 500 | 500 MB/s | Log, archive | $ |
| `sc1` | Cold HDD | 250 | 250 MB/s | Infrequent access | $ |

**DBA Recommendation:**
```
Development: gp3 (default, 3000 IOPS, 125 MB/s throughput)

Production PostgreSQL:
  OS Volume:   gp3, 30GB (boot only)
  Data Volume: gp3, provisioned IOPS as needed
               gp3 default 3000 IOPS → most workloads fine
               বাড়াতে হলে: $0.005/IOPS/month extra

High IOPS (> 16,000 IOPS):
  io2 → expensive কিন্তু consistent, low-latency
  RDS এ io2 Block Express → up to 256,000 IOPS

WAL Volume: আলাদা EBS volume দাও
  Data এবং WAL একই disk এ থাকলে I/O compete করে
  Separate volume → no contention
```

```bash
# ─── gp3 Volume তৈরি করো (PostgreSQL data) ───
DATA_VOL=$(aws ec2 create-volume \
    --size 500 \
    --volume-type gp3 \
    --iops 6000 \
    --throughput 250 \
    --availability-zone ap-southeast-1a \
    --encrypted \
    --kms-key-id alias/aws/ebs \
    --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=pg-data-volume}]' \
    --query 'VolumeId' --output text)

# EC2 তে attach করো
aws ec2 attach-volume \
    --volume-id $DATA_VOL \
    --instance-id $INSTANCE_ID \
    --device /dev/sdb

# EC2 তে mount করো
lsblk  # নতুন device দেখো (xvdb বা nvme1n1)
sudo mkfs.xfs /dev/xvdb
sudo mkdir -p /data/postgresql
sudo mount /dev/xvdb /data/postgresql
echo "/dev/xvdb /data/postgresql xfs defaults,noatime 0 0" | sudo tee -a /etc/fstab
sudo chown postgres:postgres /data/postgresql
```

---

## 5.3 EBS Snapshot — Backup

```bash
# ─── Manual Snapshot ───
SNAPSHOT_ID=$(aws ec2 create-snapshot \
    --volume-id $DATA_VOL \
    --description "PostgreSQL data backup $(date +%Y-%m-%d)" \
    --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=pg-data-snapshot},{Key=Env,Value=production}]' \
    --query 'SnapshotId' --output text)

# Snapshot complete হওয়ার অপেক্ষা করো
aws ec2 wait snapshot-completed --snapshot-ids $SNAPSHOT_ID
echo "Snapshot ready: $SNAPSHOT_ID"

# ─── Snapshot থেকে Restore ───
NEW_VOL=$(aws ec2 create-volume \
    --snapshot-id $SNAPSHOT_ID \
    --volume-type gp3 \
    --availability-zone ap-southeast-1a \
    --query 'VolumeId' --output text)

# ─── Automated Snapshot (DLM — Data Lifecycle Manager) ───
aws dlm create-lifecycle-policy \
    --description "Daily PostgreSQL EBS Snapshots" \
    --state ENABLED \
    --execution-role-arn arn:aws:iam::123456789012:role/AWSDataLifecycleManagerDefaultRole \
    --policy-details '{
        "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
        "ResourceTypes": ["VOLUME"],
        "TargetTags": [{"Key": "Backup", "Value": "daily"}],
        "Schedules": [{
            "Name": "Daily Snapshots",
            "CreateRule": {"Interval": 24, "IntervalUnit": "HOURS", "Times": ["02:00"]},
            "RetainRule": {"Count": 7},
            "TagsToAdd": [{"Key": "SnapshotType", "Value": "Automated"}]
        }]
    }'

# Volume এ tag দাও (DLM automatically snapshot নেবে)
aws ec2 create-tags \
    --resources $DATA_VOL \
    --tags Key=Backup,Value=daily
```

---

## 5.4 EBS Optimization Tips

```bash
# ─── I/O Performance Check ───
# iostat দিয়ে EBS I/O দেখো
iostat -x -d 1 10 /dev/xvdb

# fio দিয়ে benchmark করো
sudo dnf install -y fio
fio --name=randread --ioengine=libaio --iodepth=16 \
    --rw=randread --bs=4k --direct=1 \
    --size=1G --numjobs=4 --runtime=60 \
    --filename=/data/postgresql/test.bin

# ─── EBS Optimization Tips ───
# 1. Instance type EBS-optimized কিনা দেখো
aws ec2 describe-instance-types \
    --instance-types r7g.xlarge \
    --query 'InstanceTypes[0].EbsInfo'
# "EbsOptimizedSupport": "default" → always EBS-optimized

# 2. gp3 IOPS/Throughput tune করো (restart ছাড়া)
aws ec2 modify-volume \
    --volume-id $DATA_VOL \
    --iops 6000 \
    --throughput 500
# Live এ apply হয়, no downtime

# 3. Multi-attach (rare, special use case)
# একই volume multiple instances এ attach → XFS cluster

# 4. PostgreSQL specific:
# pg_wal আলাদা volume এ রাখো
aws ec2 create-volume --size 100 --volume-type gp3 \
    --iops 3000 --availability-zone ap-southeast-1a
# /var/lib/pgsql/16/data/pg_wal → symlink করো
```

---

## 5.5 Hands-on: EBS Volume Manage করো

> 💚 **FREE TIER:** 30GB gp2 EBS free। Snapshot কিছুটা storage নেয়। **Practice শেষে snapshot delete করো** যদি 30GB এর বেশি হয়।

```bash
# ─── Current storage দেখো ───
df -h
lsblk
aws ec2 describe-volumes \
    --filters "Name=attachment.instance-id,Values=$INSTANCE_ID" \
    --query 'Volumes[*].[VolumeId,Size,VolumeType,Iops,State]' \
    --output table

# ─── Volume বাড়াও (no downtime) ───
# Step 1: AWS এ volume modify করো
aws ec2 modify-volume \
    --volume-id $DATA_VOL \
    --size 1000   # 500GB → 1TB

# Step 2: Modification complete হওয়ার অপেক্ষা করো
aws ec2 describe-volumes-modifications \
    --volume-ids $DATA_VOL \
    --query 'VolumesModifications[0].ModificationState'
# "optimizing" → "completed"

# Step 3: EC2 তে filesystem resize করো (no unmount দরকার)
sudo xfs_growfs /data/postgresql
df -h /data/postgresql  # নতুন size দেখাবে

# ─── Volume Migration (gp2 → gp3) ───
# gp3 = same performance, ~20% cheaper
aws ec2 modify-volume \
    --volume-id $DATA_VOL \
    --volume-type gp3 \
    --iops 3000 \
    --throughput 125
# No downtime, transparent migration
```

---

# 6. S3 — Simple Storage Service

## 6.1 S3 কী — Database Backup এর জন্য

```
S3 = Object Storage
  File, না block storage (EBS এর মতো)
  Unlimited capacity
  99.999999999% (11 nines) durability
  Region specific (data stays in region)

DBA এর জন্য S3:
  ① Database backup store করো (pgBackRest, pg_dump)
  ② RDS automatic backup
  ③ WAL archive (pgBackRest → S3)
  ④ Data export/import
  ⑤ Audit logs store করো
```

---

## 6.2 Bucket, Object, এবং Key

```
Bucket:
  Container for objects
  Globally unique name (সারা AWS এ)
  Region specific
  "my-company-pg-backup-prod" → good name

Object:
  File stored in S3
  Max single object: 5TB (multipart upload দিয়ে)
  Metadata attached

Key:
  Object এর "path" (S3 এ directory নেই, শুধু key prefix)
  "backups/2024/03/08/mydb_full.dump" → এটাই key
  "/" = just part of the key name (logical folder)
```

---

## 6.3 Storage Classes — Cost Optimization

| Class | Use Case | Retrieval | Cost |
|---|---|---|---|
| **S3 Standard** | Frequent access | Instant | $$$ |
| **S3 Standard-IA** | Infrequent access, >30 days | Instant | $$ |
| **S3 One Zone-IA** | Infrequent, recreatable | Instant | $ |
| **S3 Glacier Instant** | Archive, few times/year | Instant | $$ |
| **S3 Glacier Flexible** | Archive, rarely | Minutes-hours | $ |
| **S3 Glacier Deep Archive** | Long-term, rarely | Hours | $ (cheapest) |

**DBA Backup Strategy:**
```
Fresh backups (last 7 days):   S3 Standard
Weekly backups (7-30 days):    S3 Standard-IA
Monthly backups (30-90 days):  S3 Glacier Instant
Yearly archive (90+ days):     S3 Glacier Deep Archive

Lifecycle policy দিয়ে automatically move করো
→ পুরনো backup cheaper storage এ যাবে, manually করতে হবে না
```

---

## 6.4 S3 Lifecycle Policy

```bash
# ─── Lifecycle Policy তৈরি করো ───
aws s3api put-bucket-lifecycle-configuration \
    --bucket my-pg-backup-prod \
    --lifecycle-configuration '{
        "Rules": [
            {
                "ID": "BackupRetentionPolicy",
                "Status": "Enabled",
                "Filter": {"Prefix": "backups/"},
                "Transitions": [
                    {
                        "Days": 7,
                        "StorageClass": "STANDARD_IA"
                    },
                    {
                        "Days": 30,
                        "StorageClass": "GLACIER_IR"
                    },
                    {
                        "Days": 90,
                        "StorageClass": "DEEP_ARCHIVE"
                    }
                ],
                "Expiration": {
                    "Days": 365
                }
            }
        ]
    }'

# ─── Bucket Versioning (accidental delete protection) ───
aws s3api put-bucket-versioning \
    --bucket my-pg-backup-prod \
    --versioning-configuration Status=Enabled
```

---

## 6.5 S3 দিয়ে pgBackRest Configure করো

```bash
# ─── S3 Bucket তৈরি করো ───
aws s3api create-bucket \
    --bucket my-pg-backup-prod \
    --region ap-southeast-1 \
    --create-bucket-configuration LocationConstraint=ap-southeast-1

# Encryption enable করো
aws s3api put-bucket-encryption \
    --bucket my-pg-backup-prod \
    --server-side-encryption-configuration '{
        "Rules": [{
            "ApplyServerSideEncryptionByDefault": {
                "SSEAlgorithm": "aws:kms"
            }
        }]
    }'

# Public access block করো
aws s3api put-public-access-block \
    --bucket my-pg-backup-prod \
    --public-access-block-configuration \
        BlockPublicAcls=true,IgnorePublicAcls=true,\
        BlockPublicPolicy=true,RestrictPublicBuckets=true
```

```ini
# pgbackrest.conf এ S3 configure করো
[global]
repo1-type            = s3
repo1-s3-bucket       = my-pg-backup-prod
repo1-s3-endpoint     = s3.amazonaws.com
repo1-s3-region       = ap-southeast-1
repo1-s3-role         = pg-backup-role    # IAM Role (access key না!)
repo1-path            = /pg-backup
repo1-retention-full  = 2
repo1-retention-diff  = 14
repo1-cipher-type     = aes-256-cbc
repo1-cipher-pass     = YourEncryptionKey2024!

[main]
pg1-path = /var/lib/pgsql/16/data
```

```bash
# ─── Test করো ───
sudo -u postgres pgbackrest --stanza=main check
sudo -u postgres pgbackrest --stanza=main --type=full backup
sudo -u postgres pgbackrest --stanza=main info

# S3 তে দেখো
aws s3 ls s3://my-pg-backup-prod/pg-backup/ --recursive | head -20
```

---

## 6.6 Hands-on: S3 Bucket এবং Backup Setup

> 💚 **FREE TIER:** 5GB S3 Standard free। pgBackRest practice এর জন্য যথেষ্ট। **Bucket naming:** globally unique লাগে — `practice-s3-backup-YOUR-ACCOUNT-ID` ব্যবহার করো।

```bash
# ─── Complete Backup Setup ───

BUCKET="my-pg-backup-$(aws sts get-caller-identity --query Account --output text)"
REGION="ap-southeast-1"

# Bucket তৈরি করো
aws s3api create-bucket \
    --bucket $BUCKET \
    --region $REGION \
    --create-bucket-configuration LocationConstraint=$REGION

# Tags দাও
aws s3api put-bucket-tagging \
    --bucket $BUCKET \
    --tagging 'TagSet=[{Key=Project,Value=database},{Key=Environment,Value=production}]'

# Lifecycle করো
aws s3api put-bucket-lifecycle-configuration \
    --bucket $BUCKET \
    --lifecycle-configuration '{
        "Rules": [{
            "ID": "PostgreSQLBackupRetention",
            "Status": "Enabled",
            "Filter": {},
            "Transitions": [
                {"Days": 7, "StorageClass": "STANDARD_IA"},
                {"Days": 30, "StorageClass": "GLACIER_IR"}
            ],
            "Expiration": {"Days": 90}
        }]
    }'

# pgBackRest configure করো
sudo tee /etc/pgbackrest/pgbackrest.conf << EOF
[global]
repo1-type        = s3
repo1-s3-bucket   = $BUCKET
repo1-s3-endpoint = s3.amazonaws.com
repo1-s3-region   = $REGION
repo1-path        = /pgbackup
repo1-retention-full = 2
repo1-retention-diff = 7

[main]
pg1-path = /var/lib/pgsql/16/data
EOF

# Stanza create এবং first backup
sudo -u postgres pgbackrest --stanza=main stanza-create
sudo -u postgres pgbackrest --stanza=main --type=full backup

# S3 তে verify করো
aws s3 ls s3://$BUCKET/pgbackup/ --recursive | head
echo "Backup size: $(aws s3 ls s3://$BUCKET/ --recursive --summarize | grep 'Total Size')"
```

---

# 7. CloudWatch — Monitoring এবং Alerting

## 7.1 CloudWatch কী — DBA এর Monitoring Tool

```
CloudWatch = AWS এর central monitoring service
  Metrics: Numbers over time (CPU, IOPS, connections)
  Logs: Text log data (PostgreSQL logs, application logs)
  Alarms: Metric threshold পার হলে alert
  Dashboards: Visual graphs
  Events/EventBridge: Schedule, automation trigger

DBA এর জন্য:
  RDS metrics → CloudWatch automatically আসে
  EC2 + PostgreSQL → CloudWatch Agent দিয়ে push করো
  PMM না থাকলে → CloudWatch দিয়ে basic monitoring করো
  PMM থাকলেও → CloudWatch Alarm দরকার (SNS → email/PagerDuty)
```

---

## 7.2 Metrics — Database কী Monitor করে

**RDS এ Automatic Metrics (no setup needed):**

| Metric | মানে | Alert Threshold |
|---|---|---|
| `CPUUtilization` | CPU % | > 80% |
| `DatabaseConnections` | Active connections | > 80% max |
| `FreeStorageSpace` | Remaining disk | < 20% |
| `FreeableMemory` | Available RAM | < 15% |
| `ReadIOPS` / `WriteIOPS` | Disk I/O ops/sec | Baseline থেকে 2x |
| `ReadLatency` / `WriteLatency` | Disk latency | > 20ms |
| `ReplicaLag` | Read replica lag | > 60s |
| `SwapUsage` | Swap used | > 100MB |
| `NetworkReceiveThroughput` | Network in | Baseline থেকে 3x |

**EC2 + PostgreSQL (CloudWatch Agent দরকার):**

```bash
# CloudWatch Agent install করো
sudo dnf install -y amazon-cloudwatch-agent

# Config তৈরি করো
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'EOF'
{
    "agent": {
        "metrics_collection_interval": 60,
        "logfile": "/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log"
    },
    "metrics": {
        "namespace": "PostgreSQL/Custom",
        "append_dimensions": {
            "InstanceId": "${aws:InstanceId}"
        },
        "metrics_collected": {
            "cpu": {
                "measurement": ["cpu_usage_idle", "cpu_usage_iowait", "cpu_usage_user"],
                "metrics_collection_interval": 60
            },
            "disk": {
                "measurement": ["used_percent", "inodes_free"],
                "resources": ["/", "/data/postgresql"],
                "metrics_collection_interval": 60
            },
            "diskio": {
                "measurement": ["io_time", "read_bytes", "write_bytes"],
                "resources": ["xvdb"],
                "metrics_collection_interval": 60
            },
            "mem": {
                "measurement": ["mem_used_percent"],
                "metrics_collection_interval": 60
            }
        }
    },
    "logs": {
        "logs_collected": {
            "files": {
                "collect_list": [
                    {
                        "file_path": "/var/lib/pgsql/16/data/log/postgresql-*.log",
                        "log_group_name": "/postgresql/mydb-prod",
                        "log_stream_name": "{instance_id}",
                        "timezone": "UTC"
                    }
                ]
            }
        }
    }
}
EOF

sudo systemctl enable --now amazon-cloudwatch-agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config -m ec2 \
    -s -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

---

## 7.3 Log Groups এবং Log Insights

```bash
# ─── PostgreSQL Logs CloudWatch এ দেখো ───

# Log Group দেখো
aws logs describe-log-groups \
    --log-group-name-prefix "/postgresql"

# Recent logs দেখো
aws logs get-log-events \
    --log-group-name "/postgresql/mydb-prod" \
    --log-stream-name "i-1234567890abcdef0" \
    --limit 20

# ─── CloudWatch Log Insights Query ───
# Console: CloudWatch → Log Insights

# Slow queries খোঁজো (last 1 hour)
aws logs start-query \
    --log-group-name "/postgresql/mydb-prod" \
    --start-time $(date -d '1 hour ago' +%s) \
    --end-time $(date +%s) \
    --query-string '
fields @timestamp, @message
| filter @message like /duration:/
| parse @message "duration: * ms" as duration_ms
| filter duration_ms > 1000
| sort duration_ms desc
| limit 20'

# Query ID পাবে
QUERY_ID="<query-id-from-above>"

# Results নাও
aws logs get-query-results --query-id $QUERY_ID

# Error count by type
# filter @message like /ERROR/
# | parse @message "ERROR:  *" as error_msg
# | stats count(*) as count by error_msg
# | sort count desc

# Connection attempts
# filter @message like /connection received/ or @message like /connection authorized/
# | stats count(*) by bin(5m)
```

---

## 7.4 Alarms — Alert Setup

```bash
# ─── SNS Topic তৈরি করো (notification channel) ───
SNS_ARN=$(aws sns create-topic \
    --name db-alerts \
    --query 'TopicArn' --output text)

# Email subscription
aws sns subscribe \
    --topic-arn $SNS_ARN \
    --protocol email \
    --notification-endpoint dba@company.com
# Email verify করো (AWS থেকে আসবে)

# Slack webhook (Lambda দিয়ে):
# SNS → Lambda → Slack

# ─── CloudWatch Alarms তৈরি করো ───

# 1. CPU High
aws cloudwatch put-metric-alarm \
    --alarm-name "RDS-CPU-High" \
    --alarm-description "RDS CPU > 80% for 5 minutes" \
    --namespace "AWS/RDS" \
    --metric-name "CPUUtilization" \
    --dimensions Name=DBInstanceIdentifier,Value=mydb-prod \
    --statistic Average \
    --period 300 \
    --evaluation-periods 2 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --alarm-actions $SNS_ARN \
    --ok-actions $SNS_ARN

# 2. Low Storage
aws cloudwatch put-metric-alarm \
    --alarm-name "RDS-Storage-Low" \
    --alarm-description "RDS Free Storage < 20GB" \
    --namespace "AWS/RDS" \
    --metric-name "FreeStorageSpace" \
    --dimensions Name=DBInstanceIdentifier,Value=mydb-prod \
    --statistic Average \
    --period 300 \
    --evaluation-periods 1 \
    --threshold 21474836480 \
    --comparison-operator LessThanThreshold \
    --alarm-actions $SNS_ARN

# 3. High Connections
aws cloudwatch put-metric-alarm \
    --alarm-name "RDS-Connections-High" \
    --alarm-description "RDS connections > 150" \
    --namespace "AWS/RDS" \
    --metric-name "DatabaseConnections" \
    --dimensions Name=DBInstanceIdentifier,Value=mydb-prod \
    --statistic Average \
    --period 60 \
    --evaluation-periods 3 \
    --threshold 150 \
    --comparison-operator GreaterThanThreshold \
    --alarm-actions $SNS_ARN

# 4. Replica Lag
aws cloudwatch put-metric-alarm \
    --alarm-name "RDS-Replica-Lag" \
    --alarm-description "Read Replica lag > 60 seconds" \
    --namespace "AWS/RDS" \
    --metric-name "ReplicaLag" \
    --dimensions Name=DBInstanceIdentifier,Value=mydb-read-replica \
    --statistic Average \
    --period 60 \
    --evaluation-periods 2 \
    --threshold 60 \
    --comparison-operator GreaterThanThreshold \
    --alarm-actions $SNS_ARN

# 5. Low Memory
aws cloudwatch put-metric-alarm \
    --alarm-name "RDS-Memory-Low" \
    --alarm-description "RDS Freeable Memory < 500MB" \
    --namespace "AWS/RDS" \
    --metric-name "FreeableMemory" \
    --dimensions Name=DBInstanceIdentifier,Value=mydb-prod \
    --statistic Average \
    --period 300 \
    --evaluation-periods 2 \
    --threshold 524288000 \
    --comparison-operator LessThanThreshold \
    --alarm-actions $SNS_ARN

# Alarm list দেখো
aws cloudwatch describe-alarms \
    --alarm-name-prefix "RDS-" \
    --query 'MetricAlarms[*].[AlarmName,StateValue]' \
    --output table
```

---

## 7.5 Dashboards তৈরি করো

```bash
# ─── Dashboard তৈরি করো ───
aws cloudwatch put-dashboard \
    --dashboard-name "PostgreSQL-Production" \
    --dashboard-body '{
        "widgets": [
            {
                "type": "metric",
                "properties": {
                    "title": "CPU Utilization",
                    "metrics": [
                        ["AWS/RDS", "CPUUtilization",
                         "DBInstanceIdentifier", "mydb-prod",
                         {"stat": "Average", "period": 300}]
                    ],
                    "period": 300,
                    "view": "timeSeries",
                    "width": 12, "height": 6
                }
            },
            {
                "type": "metric",
                "properties": {
                    "title": "Database Connections",
                    "metrics": [
                        ["AWS/RDS", "DatabaseConnections",
                         "DBInstanceIdentifier", "mydb-prod"]
                    ],
                    "width": 12, "height": 6
                }
            },
            {
                "type": "metric",
                "properties": {
                    "title": "Storage Space",
                    "metrics": [
                        ["AWS/RDS", "FreeStorageSpace",
                         "DBInstanceIdentifier", "mydb-prod",
                         {"stat": "Average"}]
                    ],
                    "width": 12, "height": 6
                }
            },
            {
                "type": "metric",
                "properties": {
                    "title": "Read/Write IOPS",
                    "metrics": [
                        ["AWS/RDS", "ReadIOPS", "DBInstanceIdentifier", "mydb-prod"],
                        ["AWS/RDS", "WriteIOPS", "DBInstanceIdentifier", "mydb-prod"]
                    ],
                    "width": 12, "height": 6
                }
            }
        ]
    }'

echo "Dashboard URL: https://ap-southeast-1.console.aws.amazon.com/cloudwatch/home#dashboards:name=PostgreSQL-Production"
```

---

## 7.6 Hands-on: Database Alarm Setup করো

> 💚 **FREE TIER:** SNS email notification free (1000/month)। CloudWatch Alarm 10টা free। Custom metrics: 10টা free।

```bash
# ─── Complete Alarm Setup Script ───
cat > /usr/local/bin/setup_cw_alarms.sh << 'SCRIPT'
#!/bin/bash
# Usage: ./setup_cw_alarms.sh mydb-prod dba@company.com

DB_INSTANCE=${1:-mydb-prod}
EMAIL=${2:-dba@company.com}
REGION=$(aws configure get region)

echo "Setting up CloudWatch Alarms for: $DB_INSTANCE"

# SNS Topic
SNS_ARN=$(aws sns create-topic --name db-alerts-${DB_INSTANCE} \
    --query 'TopicArn' --output text)
aws sns subscribe --topic-arn $SNS_ARN \
    --protocol email --notification-endpoint $EMAIL
echo "SNS Topic: $SNS_ARN (check email for subscription confirmation)"

# Create all alarms
for alarm in \
    "CPUUtilization:80:GreaterThanThreshold:2:300" \
    "DatabaseConnections:150:GreaterThanThreshold:3:60" \
    "FreeStorageSpace:21474836480:LessThanThreshold:1:300" \
    "FreeableMemory:524288000:LessThanThreshold:2:300" \
    "ReadLatency:0.02:GreaterThanThreshold:2:60" \
    "WriteLatency:0.02:GreaterThanThreshold:2:60"
do
    METRIC=$(echo $alarm | cut -d: -f1)
    THRESHOLD=$(echo $alarm | cut -d: -f2)
    OPERATOR=$(echo $alarm | cut -d: -f3)
    EVALS=$(echo $alarm | cut -d: -f4)
    PERIOD=$(echo $alarm | cut -d: -f5)

    aws cloudwatch put-metric-alarm \
        --alarm-name "RDS-${DB_INSTANCE}-${METRIC}" \
        --namespace "AWS/RDS" \
        --metric-name "$METRIC" \
        --dimensions Name=DBInstanceIdentifier,Value=$DB_INSTANCE \
        --statistic Average \
        --period $PERIOD \
        --evaluation-periods $EVALS \
        --threshold $THRESHOLD \
        --comparison-operator $OPERATOR \
        --alarm-actions $SNS_ARN \
        --ok-actions $SNS_ARN \
        --region $REGION 2>/dev/null && echo "  ✅ $METRIC alarm created"
done

echo "Done! Verify: aws cloudwatch describe-alarms --alarm-name-prefix 'RDS-${DB_INSTANCE}'"
SCRIPT
chmod +x /usr/local/bin/setup_cw_alarms.sh
```

---

# 8. Secrets Manager এবং Parameter Store

## 8.1 কেন দরকার — Password Management

```
Bad practice (still common):
  Database password → application config file এ hardcode
  Config file → Git repository এ push
  → Everyone has access to production DB password!

Good practice:
  Database password → AWS Secrets Manager
  Application → Secrets Manager API call → password নাও
  Password rotation → automatic, application এ change করতে হয় না
```

---

## 8.2 Secrets Manager — Database Password Rotation

```bash
# ─── Database Secret তৈরি করো ───
SECRET_ARN=$(aws secretsmanager create-secret \
    --name "database/production/postgresql" \
    --description "Production PostgreSQL credentials" \
    --secret-string '{
        "username": "postgres",
        "password": "StrongPass@2024!",
        "host": "mydb.xxxx.ap-southeast-1.rds.amazonaws.com",
        --port": 5432,
        "dbname": "mydb",
        "engine": "postgres"
    }' \
    --query 'ARN' --output text)

echo "Secret ARN: $SECRET_ARN"

# ─── Secret read করো ───
aws secretsmanager get-secret-value \
    --secret-id "database/production/postgresql" \
    --query 'SecretString' --output text | python3 -m json.tool

# ─── Auto Rotation setup করো (RDS native rotation) ───
aws secretsmanager rotate-secret \
    --secret-id "database/production/postgresql" \
    --rotation-rules AutomaticallyAfterDays=30
# RDS managed secrets এ built-in rotation আছে
# Custom rotation এ Lambda function দরকার

# ─── Application এ ব্যবহার করো (Python) ───
cat > /tmp/get_db_creds.py << 'EOF'
import boto3
import json

def get_db_credentials(secret_name, region="ap-southeast-1"):
    client = boto3.client('secretsmanager', region_name=region)
    response = client.get_secret_value(SecretId=secret_name)
    secret = json.loads(response['SecretString'])
    return secret

creds = get_db_credentials("database/production/postgresql")
print(f"Host: {creds['host']}")
print(f"DB:   {creds['dbname']}")
# Password কখনো print করো না log এ!
EOF
python3 /tmp/get_db_creds.py

# ─── CLI থেকে password পেয়ে psql connect করো ───
DB_PASSWORD=$(aws secretsmanager get-secret-value \
    --secret-id "database/production/postgresql" \
    --query 'SecretString' --output text | \
    python3 -c "import sys,json; print(json.load(sys.stdin)['password'])")

PGPASSWORD=$DB_PASSWORD psql \
    -h mydb.xxxx.ap-southeast-1.rds.amazonaws.com \
    -U postgres -d mydb
```

---

## 8.3 Parameter Store — Configuration Management

```bash
# ─── Parameter Store vs Secrets Manager ───
# Secrets Manager:
#   - Automatic rotation
#   - Cross-account access
#   - Expensive ($0.40/secret/month)
#   - Database passwords, API keys

# Parameter Store:
#   - Free (Standard tier)
#   - Configuration values
#   - Hierarchical naming
#   - Database connection strings, app config

# ─── Parameters তৈরি করো ───
# Standard (free)
aws ssm put-parameter \
    --name "/production/database/host" \
    --value "mydb.xxxx.ap-southeast-1.rds.amazonaws.com" \
    --type String

aws ssm put-parameter \
    --name "/production/database/port" \
    --value "5432" \
    --type String

# Encrypted (SecureString)
aws ssm put-parameter \
    --name "/production/database/password" \
    --value "StrongPass@2024!" \
    --type SecureString \
    --key-id alias/aws/ssm

# ─── Parameter read করো ───
aws ssm get-parameter \
    --name "/production/database/host" \
    --query 'Parameter.Value' --output text

aws ssm get-parameter \
    --name "/production/database/password" \
    --with-decryption \
    --query 'Parameter.Value' --output text

# সব parameters একসাথে
aws ssm get-parameters-by-path \
    --path "/production/database" \
    --with-decryption \
    --query 'Parameters[*].[Name,Value]' \
    --output table
```

---

## 8.4 Hands-on: Database Password Secrets Manager এ রাখো

> ⚠️ **FREE TIER:** Secrets Manager এ 30-day free trial। তারপর $0.40/secret/month। **Practice শেষে secret delete করো।** বিকল্প: Parameter Store (SecureString) সম্পূর্ণ free।

```bash
# ─── Complete Secrets Manager Setup ───

# 1. Secret তৈরি করো
aws secretsmanager create-secret \
    --name "database/myapp/prod" \
    --secret-string '{
        "username": "appuser",
        "password": "AppPass@2024!",
        --host": "mydb.cluster-xxxx.ap-southeast-1.rds.amazonaws.com",
        --port": 5432,
        --dbname": "mydb"
    }'

# 2. EC2 Role এ Secrets Manager access দাও
aws iam create-policy \
    --policy-name SecretsManagerDBAccess \
    --policy-document '{
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "arn:aws:secretsmanager:*:*:secret:database/*"
        }]
    }'

aws iam attach-role-policy \
    --role-name pg-backup-role \
    --policy-arn arn:aws:iam::123456789012:policy/SecretsManagerDBAccess

# 3. EC2 তে test করো (IAM Role দিয়ে, no credentials needed)
aws secretsmanager get-secret-value \
    --secret-id "database/myapp/prod" \
    --region ap-southeast-1

# 4. Secret rotation enable করো (RDS managed)
aws secretsmanager rotate-secret \
    --secret-id "database/myapp/prod" \
    --rotation-rules '{"AutomaticallyAfterDays": 30}'
```

---

# 9. KMS — Key Management Service

## 9.1 KMS কী — Encryption at Rest

```
KMS = AWS এর encryption key management
  Encryption keys তৈরি, rotate, এবং manage করো
  কোন key কে কোন resource encrypt করতে পারবে সেটা control করো

Database এ KMS:
  RDS at-rest encryption → KMS key
  EBS volume encryption → KMS key
  S3 backup encryption → KMS key
  Secrets Manager encryption → KMS key
```

---

## 9.2 CMK vs AWS Managed Key

```
AWS Managed Key (aws/rds, aws/ebs, aws/s3):
  AWS automatically তৈরি এবং manage করে
  Free of charge
  Annual automatic rotation
  Cross-account share করা যায় না
  কখন ব্যবহার করবো: Simple setup, compliance যথেষ্ট

Customer Managed Key (CMK):
  তুমি তৈরি করো এবং control করো
  $1/month per key
  Manual rotation control করতে পারো
  Cross-account share করা যায়
  Key disable করতে পারো (emergency)
  কখন ব্যবহার করবো: Compliance requirement, cross-account, key control দরকার
```

---

## 9.3 Database Encryption এ KMS

```bash
# ─── CMK তৈরি করো (database এর জন্য) ───
KEY_ARN=$(aws kms create-key \
    --description "PostgreSQL Production Encryption Key" \
    --key-usage ENCRYPT_DECRYPT \
    --query 'KeyMetadata.KeyId' --output text)

# Alias দাও
aws kms create-alias \
    --alias-name alias/postgresql-prod \
    --target-key-id $KEY_ARN

# Key rotation enable করো (annual)
aws kms enable-key-rotation --key-id $KEY_ARN

# Key policy দেখো
aws kms get-key-policy \
    --key-id $KEY_ARN \
    --policy-name default

# ─── DBA team কে key use করার permission দাও ───
aws kms create-grant \
    --key-id $KEY_ARN \
    --grantee-principal arn:aws:iam::123456789012:role/pg-backup-role \
    --operations Decrypt GenerateDataKey

# ─── Key দিয়ে RDS তৈরি করো (Guide 3 তে বিস্তারিত) ───
aws rds create-db-instance \
    --db-instance-identifier mydb-prod \
    --db-instance-class db.r7g.xlarge \
    --engine postgres \
    --engine-version "16.3" \
    --storage-encrypted \
    --kms-key-id alias/postgresql-prod \
    ... (other params)
```

---

## 9.4 Hands-on: Encrypted RDS Setup

> 💚 **FREE TIER:** AWS Managed KMS Key (alias/aws/rds) সম্পূর্ণ free। CMK (Customer Managed) = $1/month — practice এ AWS Managed Key ব্যবহার করো।

```bash
# ─── Encrypted EBS Volume তৈরি করো ───
aws ec2 create-volume \
    --size 500 \
    --volume-type gp3 \
    --encrypted \
    --kms-key-id alias/postgresql-prod \
    --availability-zone ap-southeast-1a

# ─── Encryption status verify করো ───
aws ec2 describe-volumes \
    --filters "Name=attachment.instance-id,Values=$INSTANCE_ID" \
    --query 'Volumes[*].[VolumeId,Encrypted,KmsKeyId]' \
    --output table

# ─── RDS instance encryption status ───
aws rds describe-db-instances \
    --db-instance-identifier mydb-prod \
    --query 'DBInstances[0].[StorageEncrypted,KmsKeyId]'

# Unencrypted RDS → Encrypted migration:
# 1. Snapshot নাও
aws rds create-db-snapshot \
    --db-instance-identifier mydb-prod \
    --db-snapshot-identifier mydb-unencrypted-snapshot

# 2. Encrypted copy করো
aws rds copy-db-snapshot \
    --source-db-snapshot-identifier mydb-unencrypted-snapshot \
    --target-db-snapshot-identifier mydb-encrypted-snapshot \
    --kms-key-id alias/postgresql-prod

# 3. Encrypted snapshot থেকে নতুন instance তৈরি করো
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier mydb-prod-encrypted \
    --db-snapshot-identifier mydb-encrypted-snapshot
```

---

# 10. Route 53 — DNS Management

## 10.1 Route 53 কী — Database Endpoint এর জন্য

```
Route 53 = AWS এর DNS service
  Domain registration
  DNS routing
  Health checks

DBA এর জন্য:
  Database endpoint হিসেবে friendly name দাও
  "db.internal.company.com" → RDS endpoint
  Failover routing:
    Primary down → DNS automatically Replica কে point করো
  Multi-region:
    Singapore down → Mumbai এ route করো
```

---

## 10.2 Record Types — A, CNAME, Alias

```
A Record:
  Name → IPv4 address
  "db.internal.com" → 10.0.20.15

CNAME Record:
  Name → another Name
  "db.internal.com" → "mydb.xxxx.ap-southeast-1.rds.amazonaws.com"
  ⚠️ Root domain এ CNAME নেই (RFC limitation)

Alias Record (AWS specific):
  Name → AWS resource (ELB, RDS, CloudFront)
  Root domain এও কাজ করে
  Free (CNAME এর charge থাকে)
  AWS resource এর IP change হলে automatically update হয়
  ✅ RDS endpoint এর জন্য CNAME এর চেয়ে Alias ভালো
```

---

## 10.3 Failover Routing — HA এর জন্য

```bash
# ─── Private Hosted Zone তৈরি করো ───
ZONE_ID=$(aws route53 create-hosted-zone \
    --name internal.company.com \
    --vpc VPCRegion=ap-southeast-1,VPCId=$VPC_ID \
    --caller-reference $(date +%s) \
    --hosted-zone-config Comment="Internal DNS",PrivateZone=true \
    --query 'HostedZone.Id' --output text)

# ─── Primary DB record তৈরি করো ───
aws route53 change-resource-record-sets \
    --hosted-zone-id $ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "db.internal.company.com",
                "Type": "CNAME",
                "TTL": 60,
                "ResourceRecords": [{
                    "Value": "mydb-prod.xxxx.ap-southeast-1.rds.amazonaws.com"
                }]
            }
        }]
    }'

# ─── Failover Routing (Primary/Secondary) ───
# Primary record
aws route53 change-resource-record-sets \
    --hosted-zone-id $ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "db.internal.company.com",
                "Type": "CNAME",
                "TTL": 30,
                "Failover": "PRIMARY",
                "SetIdentifier": "primary",
                "HealthCheckId": "health-check-id",
                "ResourceRecords": [{
                    "Value": "mydb-primary.xxxx.ap-southeast-1.rds.amazonaws.com"
                }]
            }
        }]
    }'

# Application সবসময় "db.internal.company.com" ব্যবহার করে।
# Primary fail হলে DNS automatically replica কে point করে।
# TTL=30 → 30 সেকেন্ডের মধ্যে failover হয়।
```

---

## 10.4 Hands-on: Database Failover DNS Setup

> ⚠️ **FREE TIER:** Route 53 Hosted Zone = $0.50/month। এটা free tier এ নেই। **Practice এর জন্য skip করতে পারো** — concept বুঝলেই হবে। অথবা practice শেষে Hosted Zone delete করো।

```bash
# ─── Health Check তৈরি করো ───
HC_ID=$(aws route53 create-health-check \
    --caller-reference $(date +%s) \
    --health-check-config '{
        "IPAddress": "10.0.20.15",
        "Port": 5432,
        "Type": "TCP",
        "RequestInterval": 10,
        "FailureThreshold": 2
    }' \
    --query 'HealthCheck.Id' --output text)

echo "Health Check: $HC_ID"

# ─── Simple CNAME তৈরি করো (most common for DBA) ───
ZONE_ID=$(aws route53 list-hosted-zones \
    --query "HostedZones[?Config.PrivateZone==\`true\`].Id" \
    --output text | head -1 | cut -d/ -f3)

# Primary DB endpoint
aws route53 change-resource-record-sets \
    --hosted-zone-id $ZONE_ID \
    --change-batch '{
        "Comment": "Database endpoint",
        "Changes": [{
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "postgres.internal.company.com",
                "Type": "CNAME",
                "TTL": 60,
                "ResourceRecords": [{"Value": "mydb-prod.xxxx.ap-southeast-1.rds.amazonaws.com"}]
            }
        }]
    }'

# ─── Verify DNS ───
# VPC এর ভেতরের EC2 থেকে:
nslookup postgres.internal.company.com
dig postgres.internal.company.com

# ─── Manual failover (Patroni এর মতো DNS এ) ───
# Primary fail হলে:
aws route53 change-resource-record-sets \
    --hosted-zone-id $ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "postgres.internal.company.com",
                "Type": "CNAME",
                "TTL": 30,
                "ResourceRecords": [{"Value": "mydb-replica.xxxx.ap-southeast-1.rds.amazonaws.com"}]
            }
        }]
    }'
echo "DNS updated to replica — applications will reconnect within 30s"
```

---

# 11. AWS CLI — Command Line থেকে সব কাজ

## 11.1 Install এবং Configure করো

```bash
# ─── Install ───
# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version   # aws-cli/2.x.x

# ─── Configure ───
aws configure
# AWS Access Key ID:     AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI...
# Default region:        ap-southeast-1
# Default output format: json  (json/yaml/table/text)

# ─── Multiple Profiles (dev/staging/production) ───
aws configure --profile production
aws configure --profile staging

# Profile ব্যবহার করো
aws s3 ls --profile production
export AWS_PROFILE=production  # session এ default করো

# ─── Named Profiles in ~/.aws/credentials ───
cat ~/.aws/credentials
# [default]
# aws_access_key_id = AKIAIOSFODNN7EXAMPLE
# aws_secret_access_key = wJalrXUtnFEMI...
#
# [production]
# aws_access_key_id = AKIAI...
# aws_secret_access_key = ...
#
# [staging]
# ...

# ─── EC2 Instance এ Role ব্যবহার করো ───
# IAM Role attach থাকলে credentials configure করতে হয় না!
# AWS CLI automatically instance metadata থেকে credentials নেয়
aws sts get-caller-identity  # Role এর ARN দেখাবে
```

---

## 11.2 DBA Daily Use Commands

```bash
# ═══ RDS Commands ═══

# সব RDS instances দেখো
aws rds describe-db-instances \
    --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,DBInstanceClass,EngineVersion]' \
    --output table

# Specific instance details
aws rds describe-db-instances \
    --db-instance-identifier mydb-prod \
    --query 'DBInstances[0]' | python3 -m json.tool

# Snapshot তৈরি করো
aws rds create-db-snapshot \
    --db-instance-identifier mydb-prod \
    --db-snapshot-identifier mydb-$(date +%Y%m%d-%H%M)

# Snapshot list দেখো
aws rds describe-db-snapshots \
    --db-instance-identifier mydb-prod \
    --query 'DBSnapshots[*].[DBSnapshotIdentifier,SnapshotCreateTime,Status,AllocatedStorage]' \
    --output table

# Latest snapshot
aws rds describe-db-snapshots \
    --db-instance-identifier mydb-prod \
    --query 'sort_by(DBSnapshots, &SnapshotCreateTime)[-1].[DBSnapshotIdentifier,SnapshotCreateTime]' \
    --output table

# RDS reboot করো
aws rds reboot-db-instance \
    --db-instance-identifier mydb-prod

# Parameter Group change
aws rds modify-db-instance \
    --db-instance-identifier mydb-prod \
    --db-parameter-group-name mydb-params \
    --apply-immediately

# ═══ EC2 Commands ═══

# Instances দেখো
aws ec2 describe-instances \
    --filters "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value|[0],InstanceType,PrivateIpAddress]' \
    --output table

# Instance start/stop
aws ec2 start-instances --instance-ids i-1234567890abcdef0
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# ═══ S3 Commands ═══

# Backup list
aws s3 ls s3://my-pg-backup/backups/ --recursive --human-readable \
    | sort -k1,2 | tail -20

# File copy
aws s3 cp /backup/mydb.dump s3://my-pg-backup/backups/$(date +%Y/%m/%d)/
aws s3 cp s3://my-pg-backup/backups/2024/03/08/mydb.dump /tmp/

# Sync directory
aws s3 sync /backup/ s3://my-pg-backup/backups/ --delete

# Storage used
aws s3 ls s3://my-pg-backup/ --recursive --summarize | grep "Total Size"

# ═══ CloudWatch Commands ═══

# Latest metrics দেখো (last 1 hour)
aws cloudwatch get-metric-statistics \
    --namespace AWS/RDS \
    --metric-name CPUUtilization \
    --dimensions Name=DBInstanceIdentifier,Value=mydb-prod \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 300 \
    --statistics Average \
    --query 'Datapoints[*].[Timestamp,Average]' \
    --output table

# Alarm status দেখো
aws cloudwatch describe-alarms \
    --query 'MetricAlarms[*].[AlarmName,StateValue,StateReason]' \
    --output table

# ═══ Secrets Manager ═══

# Password নাও
aws secretsmanager get-secret-value \
    --secret-id "database/myapp/prod" \
    --query 'SecretString' --output text | python3 -m json.tool

# ═══ IAM ═══

# Current identity দেখো
aws sts get-caller-identity

# My permissions দেখো
aws iam simulate-principal-policy \
    --policy-source-arn $(aws sts get-caller-identity --query Arn --output text) \
    --action-names rds:DescribeDBInstances \
    --query 'EvaluationResults[*].[EvalActionName,EvalDecision]'
```

---

## 11.3 AWS CLI দিয়ে Scripting

```bash
# ─── Daily Database Health Check Script ───
cat > /usr/local/bin/aws_db_check.sh << 'SCRIPT'
#!/bin/bash
REGION="ap-southeast-1"
DB_INSTANCE="mydb-prod"

echo "=== AWS Database Health Check - $(date) ==="

# RDS Status
STATUS=$(aws rds describe-db-instances \
    --db-instance-identifier $DB_INSTANCE \
    --query 'DBInstances[0].DBInstanceStatus' \
    --output text --region $REGION)
echo "RDS Status: $STATUS"

# CloudWatch Metrics (last 5 min)
CPU=$(aws cloudwatch get-metric-statistics \
    --namespace AWS/RDS \
    --metric-name CPUUtilization \
    --dimensions Name=DBInstanceIdentifier,Value=$DB_INSTANCE \
    --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 300 --statistics Average \
    --query 'Datapoints[*].Average' --output text --region $REGION | tail -1)
echo "CPU: ${CPU}%"

STORAGE=$(aws cloudwatch get-metric-statistics \
    --namespace AWS/RDS \
    --metric-name FreeStorageSpace \
    --dimensions Name=DBInstanceIdentifier,Value=$DB_INSTANCE \
    --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 300 --statistics Average \
    --query 'Datapoints[*].Average' --output text --region $REGION | tail -1)
STORAGE_GB=$(echo "scale=2; $STORAGE/1073741824" | bc)
echo "Free Storage: ${STORAGE_GB}GB"

CONNECTIONS=$(aws cloudwatch get-metric-statistics \
    --namespace AWS/RDS \
    --metric-name DatabaseConnections \
    --dimensions Name=DBInstanceIdentifier,Value=$DB_INSTANCE \
    --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 300 --statistics Average \
    --query 'Datapoints[*].Average' --output text --region $REGION | tail -1)
echo "Connections: $CONNECTIONS"

# Latest Snapshot
SNAPSHOT=$(aws rds describe-db-snapshots \
    --db-instance-identifier $DB_INSTANCE \
    --query 'sort_by(DBSnapshots, &SnapshotCreateTime)[-1].[DBSnapshotIdentifier,SnapshotCreateTime]' \
    --output text --region $REGION)
echo "Latest Snapshot: $SNAPSHOT"

# Alarms in ALARM state
ALARMS=$(aws cloudwatch describe-alarms \
    --state-value ALARM \
    --query 'MetricAlarms[?contains(AlarmName, `RDS`)].[AlarmName,StateReason]' \
    --output text --region $REGION)
if [ -n "$ALARMS" ]; then
    echo "⚠️  ACTIVE ALARMS:"
    echo "$ALARMS"
fi

echo "==================================="
SCRIPT
chmod +x /usr/local/bin/aws_db_check.sh

# Cron এ run করো
echo "0 9 * * * ec2-user /usr/local/bin/aws_db_check.sh | mail -s 'AWS DB Daily Report' dba@company.com" | \
    sudo tee /etc/cron.d/aws_db_check
```

---

# 12. Cost Awareness — DBA এর জন্য

## 12.1 কোন Service কতটা খরচ করে

```
Database related AWS costs:

RDS Instance:
  db.r7g.xlarge (4 vCPU, 32GB): ~$0.41/hour = ~$300/month
  db.r7g.2xlarge (8 vCPU, 64GB): ~$0.82/hour = ~$600/month
  Multi-AZ = 2x cost!
  Reserved Instances (1 year): ~40% discount
  Reserved Instances (3 year): ~60% discount

RDS Storage:
  gp3: $0.115/GB/month
  io2: $0.125/GB/month + $0.10/IOPS/month
  500GB gp3 = $57.50/month

RDS Backup:
  Free up to DB size
  Beyond: $0.095/GB/month

EC2 (self-managed):
  r7g.xlarge: ~$0.24/hour = ~$175/month (cheaper than RDS!)
  কিন্তু management overhead বেশি

EBS:
  gp3: $0.08/GB/month
  500GB: $40/month

S3 (backup):
  Standard: $0.023/GB/month
  Standard-IA: $0.0125/GB/month
  Glacier: $0.004/GB/month
  100GB backup = $2.30/month (Standard)

Data Transfer:
  EC2/RDS → Internet: $0.09/GB (first 10TB)
  Same region: Free
  Cross-region: $0.02/GB
  → Database থেকে data বাইরে পাঠানো expensive!

CloudWatch:
  Basic monitoring: Free
  Detailed metrics: $0.30/metric/month
  Custom metrics: $0.30/metric/month
  Log ingestion: $0.50/GB
  Log storage: $0.03/GB/month

Secrets Manager:
  $0.40/secret/month
  $0.05 per 10,000 API calls
```

---

## 12.2 Cost Optimization Tips

```
1. Reserved Instances (RI) — most impactful
   Production DB = always running
   1-year RI: 40% discount, 3-year RI: 60% discount
   Convertible RI: instance type change করা যায়

2. Right-sizing
   CloudWatch metrics দেখো:
   CPU সবসময় < 20%? → smaller instance
   Memory সবসময় > 80%? → larger instance
   AWS Compute Optimizer → recommendation দেবে

3. Storage Class Lifecycle
   S3 backup: Standard → IA → Glacier automatically
   পুরনো EBS snapshot → Delete করো (no longer needed)

4. Multi-AZ শুধু production এ
   Dev/Staging: Single-AZ (half the cost)
   Production: Multi-AZ

5. Read Replica শুধু দরকার হলে
   আলাদা billing আছে (same as primary)
   Read traffic আসলেই আলাদা করো

6. Data Transfer কমাও
   Application এবং Database একই region এ রাখো
   Same AZ রাখার চেষ্টা করো (free transfer)
   VPC Endpoint দিয়ে S3 access করো (data transfer charge নেই)

7. Aurora Serverless (variable workload এর জন্য)
   Traffic কম থাকলে scale down
   Idle: ~$0.06/hour (minimum)
   Peak: auto-scale up

8. Spot Instances (dev/test DB)
   On-demand এর 70-90% সস্তা
   Interruption possible → dev/test এ ok
   Production: না!
```

---

## 12.3 AWS Cost Explorer ব্যবহার করো

```bash
# ─── Cost Explorer API ───
# Last month এর database costs দেখো
aws ce get-cost-and-usage \
    --time-period Start=$(date -d 'last month' +%Y-%m-01),End=$(date +%Y-%m-01) \
    --granularity MONTHLY \
    --filter '{"Dimensions": {"Key": "SERVICE", "Values": ["Amazon Relational Database Service"]}}' \
    --metrics BlendedCost \
    --query 'ResultsByTime[0].Total.BlendedCost'

# Service by service breakdown
aws ce get-cost-and-usage \
    --time-period Start=$(date -d 'last month' +%Y-%m-01),End=$(date +%Y-%m-01) \
    --granularity MONTHLY \
    --group-by Type=DIMENSION,Key=SERVICE \
    --metrics BlendedCost \
    --query 'ResultsByTime[0].Groups[*].[Keys[0],Metrics.BlendedCost.Amount]' \
    --output table | sort -k2 -rn

# ─── Budget Alert তৈরি করো ───
aws budgets create-budget \
    --account-id $(aws sts get-caller-identity --query Account --output text) \
    --budget '{
        "BudgetName": "RDS-Monthly-Budget",
        "BudgetLimit": {"Amount": "500", "Unit": "USD"},
        "TimeUnit": "MONTHLY",
        "BudgetType": "COST",
        "CostFilters": {
            "Service": ["Amazon Relational Database Service"]
        }
    }' \
    --notifications-with-subscribers '[
        {
            "Notification": {
                "NotificationType": "ACTUAL",
                "ComparisonOperator": "GREATER_THAN",
                "Threshold": 80
            },
            "Subscribers": [{
                "SubscriptionType": "EMAIL",
                "Address": "dba@company.com"
            }]
        }
    ]'
```

---

*AWS Fundamentals for DBA | Guide 2*
*IAM · VPC · EC2 · EBS · S3 · CloudWatch · Secrets Manager · KMS · Route 53 · AWS CLI*
*Foundation for Guide 3: AWS Cloud Native DBA (RDS, Aurora, and Managed Services)*

---


# 13. Free Tier — Step-by-Step Hands-on Lab

## 13.0 Free Tier এ কী কী পাবে এবং সাবধানতা

```
AWS Free Tier এ DBA practice এর জন্য যা free (12 মাস):

EC2:
  ✅ t2.micro বা t3.micro — 750 hours/month
     1টা instance সারামাস চালালেও free

RDS:
  ✅ db.t3.micro বা db.t2.micro — 750 hours/month
     Single-AZ, 20GB storage, 20GB backup
     1টা RDS সারামাস চালালেও free

S3:
  ✅ 5GB storage
  ✅ 20,000 GET / 2,000 PUT requests

EBS:
  ✅ 30GB gp2/gp3 SSD storage

CloudWatch:
  ✅ Basic monitoring (5-min interval) — always free
  ✅ 10 custom metrics, 10 alarms
  ✅ 3 dashboards

Parameter Store (SSM):
  ✅ Standard tier — FREE
  → Secrets Manager এর বদলে এটা ব্যবহার করো

Secrets Manager:
  ❌ $0.40/secret/month — skip করো lab এ

KMS:
  ✅ AWS Managed Key — FREE
  ❌ Customer Managed Key: $1/month — skip

Route 53:
  ❌ $0.50/hosted zone/month — skip বা শেষে delete করো

⚠️ সাবধানতা:
  ① Lab শেষে সব resource DELETE করো
  ② Billing Alarm set করো ($1 threshold) — Lab 0 তে করবো
  ③ t2.micro/t3.micro ছাড়া অন্য instance চালু করো না
  ④ Multi-AZ RDS চালু করো না (2x cost!)
  ⑤ NAT Gateway তৈরি করো না ($32/month!)
```

---

## 13.1 Lab 0 — Account Setup এবং Safety Net

**সবার আগে এটা করো। একবারই।**
**Duration: ~30 minutes | Cost: Free**

---

### 🖥️ GUI Method

**Step 1: AWS Console এ Login করো**
```
Browser এ যাও: https://console.aws.amazon.com
→ Root user email দিয়ে sign in করো
→ Top right corner এ Region দেখো
→ Region dropdown click করো → "Asia Pacific (Singapore) ap-southeast-1" select করো
```

**Step 2: Root Account এ MFA Enable করো**
```
Top right: তোমার account name এ click করো
→ "Security credentials" click করো
→ "Multi-factor authentication (MFA)" section
→ "Assign MFA device" button click করো
→ Device name: "my-phone"
→ "Authenticator app" select → Continue
→ QR code দেখাবে → Google Authenticator বা Authy দিয়ে scan করো
→ দুটো consecutive code enter করো → "Add MFA" click
→ ✅ MFA enabled দেখাবে
```

**Step 3: Billing Alarm তৈরি করো**
```
⚠️ NOTE: Billing metrics শুধু us-east-1 region এ আছে
→ Top right: Region → "US East (N. Virginia) us-east-1" select করো

Services search bar: "CloudWatch" টাইপ করো → CloudWatch click করো
→ Left menu: "Alarms" → "All alarms"
→ "Create alarm" button click করো
→ "Select metric" click করো
→ Search: "Billing" → "Total Estimated Charge" → "USD" checkbox → "Select metric"

Metric configuration:
→ Statistic: Maximum
→ Period: 6 hours

Conditions:
→ Threshold type: Static
→ Whenever EstimatedCharges is: Greater than
→ than: 1   ← $1 হলেই alert

Next → "Create new topic"
→ Topic name: billing-alerts
→ Email: তোমার email address
→ "Create topic" click

→ Alarm name: "FreeTier-Billing-Alert"
→ "Create alarm" click

⚠️ Email verify করো! AWS থেকে confirmation email আসবে → Confirm subscription link এ click করো
```

**Step 4: Region ap-southeast-1 এ ফিরে যাও**
```
Top right: Region → "Asia Pacific (Singapore) ap-southeast-1"
এখন থেকে সব কাজ এই region এ করবো
```

**Step 5: IAM Admin User তৈরি করো**
```
Services search: "IAM" → IAM click করো
→ Left menu: "Users" → "Create user" button

User details:
→ User name: dba-admin
→ "Provide user access to the AWS Management Console" checkbox ✅
→ "I want to create an IAM user" select
→ Console password: Custom password → "DbaAdmin@2024!"
→ "Users must create a new password" → uncheck (lab এর জন্য)
→ Next

Set permissions:
→ "Attach policies directly" select
→ Search: "AdministratorAccess"
→ "AdministratorAccess" checkbox ✅
→ Next → "Create user"

Access Key তৈরি করো (CLI এর জন্য):
→ "dba-admin" user এ click করো
→ "Security credentials" tab
→ "Access keys" section → "Create access key"
→ Use case: "Command Line Interface (CLI)" select
→ Confirmation checkbox ✅ → Next
→ Description: "lab-cli-key"
→ "Create access key" click
→ ⚠️ Access key ID এবং Secret access key COPY করে রাখো!
   (এই page বন্ধ হলে secret আর দেখা যাবে না)
→ .csv file download করো
```

---

### 💻 CLI Method

```bash
# AWS CLI Install (Linux)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
aws --version

# Configure করো dba-admin এর credentials দিয়ে
aws configure --profile dbalab
# AWS Access Key ID:     [GUI তে তৈরি করা key]
# AWS Secret Access Key: [GUI তে তৈরি করা secret]
# Default region:        ap-southeast-1
# Default output format: json

# Default profile করো
export AWS_PROFILE=dbalab

# Verify
aws sts get-caller-identity
# "Arn": "arn:aws:iam::ACCOUNT_ID:user/dba-admin" দেখাবে

# Billing Alarm (us-east-1 এ করতে হবে)
SNS_ARN=$(aws sns create-topic --name billing-alerts \
    --region us-east-1 --query 'TopicArn' --output text)

aws sns subscribe --topic-arn $SNS_ARN \
    --protocol email \
    --notification-endpoint your-email@gmail.com \
    --region us-east-1

aws cloudwatch put-metric-alarm \
    --alarm-name "FreeTier-Billing-Alert" \
    --namespace "AWS/Billing" \
    --metric-name "EstimatedCharges" \
    --dimensions Name=Currency,Value=USD \
    --statistic Maximum \
    --period 21600 \
    --evaluation-periods 1 \
    --threshold 1 \
    --comparison-operator GreaterThanThreshold \
    --alarm-actions $SNS_ARN \
    --region us-east-1
echo "✅ Billing alarm created"
```

---

### ✅ Verify

```
GUI: Billing → Free Tier দেখো — usage 0% দেখাবে
CLI: aws sts get-caller-identity → dba-admin দেখাবে
```

---

## 13.2 Lab 1 — IAM: Users, Groups, Policies

**Duration: ~30 minutes | Cost: Free**

---

### 🖥️ GUI Method

**Step 1: Group তৈরি করো**
```
Services search: "IAM" → IAM
→ Left menu: "User groups" → "Create group"

Group name: dba-team
→ Attach permissions policies:
   Search: "AmazonRDSFullAccess" → checkbox ✅
   Search: "CloudWatchReadOnlyAccess" → checkbox ✅
→ "Create user group" click
```

**Step 2: Custom Policy তৈরি করো**
```
Left menu: "Policies" → "Create policy"
→ "JSON" tab click করো
→ সব text delete করো, নিচেরটা paste করো:
```

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "S3BackupAccess",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::*-pg-backup-*",
                "arn:aws:s3:::*-pg-backup-*/*"
            ]
        }
    ]
}
```

```
→ Next
→ Policy name: DBA-S3-Backup-Policy
→ Description: "DBA team S3 backup access"
→ "Create policy" click
```

**Step 3: Policy Group এ attach করো**
```
Left menu: "User groups" → "dba-team" click করো
→ "Permissions" tab → "Add permissions" dropdown → "Attach policies"
→ Search: "DBA-S3-Backup-Policy" → checkbox ✅
→ "Attach policies" click
```

**Step 4: Test User তৈরি করো**
```
Left menu: "Users" → "Create user"
→ User name: test-dba-user
→ Next

Set permissions:
→ "Add user to group" select
→ "dba-team" checkbox ✅
→ Next → "Create user"
```

**Step 5: Permission Simulate করো**
```
Left menu: "Policies" → "Simulate" নেই সরাসরি
→ "Users" → "test-dba-user" click করো
→ "Permissions" tab দেখো — inherited from dba-team

Policy Simulator:
→ https://policysim.aws.amazon.com (directly যাও)
→ Users, Groups, and Roles: test-dba-user select করো
→ Actions: "rds:CreateDBInstance" enter → Run Simulation
→ Result: allowed বা denied দেখাবে
```

**Step 6: Cleanup**
```
Users → test-dba-user → Delete
User groups → dba-team → Delete group
Policies → DBA-S3-Backup-Policy → Delete (Actions menu থেকে)
```

---

### 💻 CLI Method

```bash
# Group তৈরি করো
aws iam create-group --group-name dba-team
aws iam attach-group-policy \
    --group-name dba-team \
    --policy-arn arn:aws:iam::aws:policy/AmazonRDSFullAccess
aws iam attach-group-policy \
    --group-name dba-team \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess

# Custom Policy তৈরি করো
cat > /tmp/s3-backup-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": ["s3:PutObject","s3:GetObject","s3:ListBucket","s3:DeleteObject"],
        "Resource": ["arn:aws:s3:::*-pg-backup-*","arn:aws:s3:::*-pg-backup-*/*"]
    }]
}
EOF

ACCT=$(aws sts get-caller-identity --query Account --output text)
aws iam create-policy \
    --policy-name DBA-S3-Backup-Policy \
    --policy-document file:///tmp/s3-backup-policy.json

aws iam attach-group-policy \
    --group-name dba-team \
    --policy-arn arn:aws:iam::${ACCT}:policy/DBA-S3-Backup-Policy

# Test user তৈরি করো
aws iam create-user --user-name test-dba-user
aws iam add-user-to-group --user-name test-dba-user --group-name dba-team

# Permission simulate করো
aws iam simulate-principal-policy \
    --policy-source-arn arn:aws:iam::${ACCT}:user/test-dba-user \
    --action-names rds:DescribeDBInstances rds:DeleteDBInstance s3:PutObject \
    --query 'EvaluationResults[*].[EvalActionName,EvalDecision]' \
    --output table

# Cleanup
aws iam remove-user-from-group --user-name test-dba-user --group-name dba-team
aws iam delete-user --user-name test-dba-user
aws iam detach-group-policy \
    --group-name dba-team \
    --policy-arn arn:aws:iam::${ACCT}:policy/DBA-S3-Backup-Policy
aws iam detach-group-policy \
    --group-name dba-team \
    --policy-arn arn:aws:iam::aws:policy/AmazonRDSFullAccess
aws iam detach-group-policy \
    --group-name dba-team \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess
aws iam delete-group --group-name dba-team
aws iam delete-policy --policy-arn arn:aws:iam::${ACCT}:policy/DBA-S3-Backup-Policy
echo "✅ Lab 1 cleanup done"
```

---

## 13.3 Lab 2 — VPC: Network তৈরি করো

**Duration: ~45 minutes | Cost: Free**
**⚠️ NAT Gateway তৈরি করো না — $32/month charge হবে!**

---

### 🖥️ GUI Method

**Step 1: VPC Dashboard এ যাও**
```
Services search: "VPC" → VPC click করো
→ Top left: "VPC Dashboard" দেখাবে
→ Region confirm করো: ap-southeast-1 (top right)
```

**Step 2: VPC তৈরি করো**
```
Left menu: "Your VPCs" → "Create VPC" button (top right)

VPC settings:
→ Resources to create: "VPC only" select করো
  (VPC and more select করলে automatically সব তৈরি হয়
   কিন্তু আমরা manually করবো — শেখার জন্য)
→ Name tag: lab-vpc
→ IPv4 CIDR block: 10.0.0.0/16
→ IPv6: No IPv6 CIDR block
→ Tenancy: Default
→ "Create VPC" button click করো

✅ "lab-vpc" created দেখাবে

DNS Settings enable করো:
→ VPC list এ "lab-vpc" click করো
→ "Actions" dropdown → "Edit VPC settings"
→ "Enable DNS hostnames" checkbox ✅
→ "Enable DNS resolution" checkbox ✅
→ "Save" click করো
```

**Step 3: Internet Gateway তৈরি করো**
```
Left menu: "Internet gateways" → "Create internet gateway"
→ Name tag: lab-igw
→ "Create internet gateway" click করো

IGW Attach করো VPC তে:
→ "Attach to a VPC" button দেখাবে (top) → click করো
  অথবা: Actions → "Attach to VPC"
→ Available VPCs: "lab-vpc" select করো
→ "Attach internet gateway" click করো

✅ State: "Attached" দেখাবে
```

**Step 4: Subnets তৈরি করো**
```
Left menu: "Subnets" → "Create subnet"

Subnet 1 (Public):
→ VPC ID: "lab-vpc" select করো
→ Subnet name: lab-public-1a
→ Availability Zone: ap-southeast-1a
→ IPv4 CIDR block: 10.0.1.0/24
→ "Add new subnet" button click করো (আরো subnet add করবো)

Subnet 2 (DB - AZ-a):
→ Subnet name: lab-db-1a
→ Availability Zone: ap-southeast-1a
→ IPv4 CIDR block: 10.0.10.0/24
→ "Add new subnet" click করো

Subnet 3 (DB - AZ-b):
→ Subnet name: lab-db-1b
→ Availability Zone: ap-southeast-1b
→ IPv4 CIDR block: 10.0.11.0/24

→ "Create subnet" button click করো (সব তিনটা একসাথে তৈরি হবে)
✅ 3 subnets created দেখাবে
```

**Step 5: Route Table তৈরি করো (Public Subnet এর জন্য)**
```
Left menu: "Route tables" → "Create route table"
→ Name: lab-public-rt
→ VPC: "lab-vpc" select করো
→ "Create route table" click করো

Internet route যোগ করো:
→ "Routes" tab → "Edit routes" button
→ "Add route" click করো
→ Destination: 0.0.0.0/0
→ Target: "Internet Gateway" select করো → "lab-igw" select করো
→ "Save changes" click করো

Public Subnet এ associate করো:
→ "Subnet associations" tab
→ "Edit subnet associations" button
→ "lab-public-1a" checkbox ✅
→ "Save associations" click করো

✅ Route table associated with public subnet
```

**Step 6: Security Groups তৈরি করো**

*Bastion Security Group:*
```
Left menu: "Security Groups" → "Create security group"

Basic details:
→ Security group name: lab-bastion-sg
→ Description: Lab Bastion Host Security Group
→ VPC: "lab-vpc" select করো

Inbound rules → "Add rule":
→ Type: SSH
→ Protocol: TCP (auto)
→ Port range: 22 (auto)
→ Source: My IP   ← এটা select করলে তোমার current IP automatically আসবে
  (Lab এ "Anywhere-IPv4" 0.0.0.0/0 ও দেওয়া যায় কিন্তু insecure)
→ Description: DBA SSH access

Outbound rules: default রাখো (All traffic allow)

→ "Create security group" click করো
✅ lab-bastion-sg created
```

*Database Security Group:*
```
"Create security group" button আবার click করো

Basic details:
→ Security group name: lab-db-sg
→ Description: Lab Database Security Group
→ VPC: "lab-vpc" select করো

Inbound rules → "Add rule":
→ Type: PostgreSQL
→ Protocol: TCP (auto)
→ Port range: 5432 (auto)
→ Source: Custom → search box এ "lab-bastion-sg" type করো
   → "lab-bastion-sg" এর ID দেখাবে → select করো
→ Description: From Bastion only

→ "Create security group" click করো
✅ lab-db-sg created
```

**Step 7: RDS Subnet Group তৈরি করো**
```
Services search: "RDS" → RDS click করো
→ Left menu: "Subnet groups" → "Create DB subnet group"

Subnet group details:
→ Name: lab-db-subnet-group
→ Description: Lab DB Subnet Group
→ VPC: "lab-vpc" select করো

Add subnets:
→ Availability Zones: ap-southeast-1a এবং ap-southeast-1b দুটোই select করো
→ Subnets:
   → ap-southeast-1a থেকে: "10.0.10.0/24" (lab-db-1a) select করো
   → ap-southeast-1b থেকে: "10.0.11.0/24" (lab-db-1b) select করো

→ "Create" button click করো
✅ lab-db-subnet-group created
```

---

### 💻 CLI Method

```bash
REGION="ap-southeast-1"

# VPC
VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lab-vpc}]' \
    --query 'Vpc.VpcId' --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
echo "VPC: $VPC_ID"

# Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
    --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=lab-igw}]' \
    --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
echo "IGW: $IGW_ID"

# Subnets
PUB_1A=$(aws ec2 create-subnet --vpc-id $VPC_ID \
    --cidr-block 10.0.1.0/24 --availability-zone ${REGION}a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab-public-1a}]' \
    --query 'Subnet.SubnetId' --output text)
DB_1A=$(aws ec2 create-subnet --vpc-id $VPC_ID \
    --cidr-block 10.0.10.0/24 --availability-zone ${REGION}a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab-db-1a}]' \
    --query 'Subnet.SubnetId' --output text)
DB_1B=$(aws ec2 create-subnet --vpc-id $VPC_ID \
    --cidr-block 10.0.11.0/24 --availability-zone ${REGION}b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab-db-1b}]' \
    --query 'Subnet.SubnetId' --output text)
echo "Subnets: $PUB_1A | $DB_1A | $DB_1B"

# Route Table
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=lab-public-rt}]' \
    --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PUB_RT \
    --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --subnet-id $PUB_1A --route-table-id $PUB_RT

# Security Groups
SG_BASTION=$(aws ec2 create-security-group \
    --group-name lab-bastion-sg \
    --description "Lab Bastion Security Group" \
    --vpc-id $VPC_ID --query 'GroupId' --output text)
MY_IP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress \
    --group-id $SG_BASTION \
    --protocol tcp --port 22 --cidr ${MY_IP}/32
echo "Bastion SG: $SG_BASTION (allowed IP: $MY_IP)"

SG_DB=$(aws ec2 create-security-group \
    --group-name lab-db-sg \
    --description "Lab Database Security Group" \
    --vpc-id $VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress \
    --group-id $SG_DB \
    --protocol tcp --port 5432 --source-group $SG_BASTION
echo "DB SG: $SG_DB"

# RDS Subnet Group
aws rds create-db-subnet-group \
    --db-subnet-group-name lab-db-subnet-group \
    --db-subnet-group-description "Lab DB Subnet Group" \
    --subnet-ids $DB_1A $DB_1B
echo "✅ Lab 2 complete!"

# IDs save করো
cat > /tmp/lab-ids.env << EOF
export VPC_ID=$VPC_ID
export IGW_ID=$IGW_ID
export PUB_1A=$PUB_1A
export DB_1A=$DB_1A
export DB_1B=$DB_1B
export PUB_RT=$PUB_RT
export SG_BASTION=$SG_BASTION
export SG_DB=$SG_DB
EOF
echo "IDs saved → /tmp/lab-ids.env"
```

---

## 13.4 Lab 3 — EC2: Bastion Host তৈরি করো

**Duration: ~30 minutes | Cost: t3.micro Free Tier**

---

### 🖥️ GUI Method

**Step 1: Key Pair তৈরি করো**
```
Services search: "EC2" → EC2 click করো
→ Left menu: "Network & Security" → "Key Pairs"
→ "Create key pair" button

Key pair details:
→ Name: lab-key
→ Key pair type: RSA
→ Private key file format: .pem (Linux/Mac) অথবা .ppk (Windows PuTTY)
→ "Create key pair" click করো

⚠️ Automatically download হবে → safe জায়গায় রাখো
   chmod 400 ~/Downloads/lab-key.pem   (Linux/Mac terminal এ)
```

**Step 2: EC2 Instance Launch করো**
```
Left menu: "Instances" → "Launch instances" button (top right)

Name and tags:
→ Name: lab-bastion

Application and OS Images:
→ "Amazon Linux" tab নিশ্চিত করো
→ Amazon Linux 2023 AMI দেখাবে → এটাই রাখো (Free tier eligible)

Instance type:
→ t3.micro select করো (Free tier eligible দেখাবে)

Key pair:
→ "lab-key" select করো

Network settings → "Edit" click করো:
→ VPC: "lab-vpc" select করো
→ Subnet: "lab-public-1a" select করো
→ Auto-assign public IP: Enable
→ Firewall (security groups): "Select existing security group"
→ "lab-bastion-sg" checkbox ✅

Configure storage:
→ 8GB gp3 রাখো (default, free tier এ যথেষ্ট)

→ "Launch instance" button click করো
✅ Instance launching... দেখাবে

Instance ID এ click করো → Instances page এ দেখাবে
→ State: "Pending" → কিছুক্ষণ পরে "Running"
→ Public IPv4 address copy করো (e.g., 52.77.x.x)
```

**Step 3: SSH Connect করো**
```
Method A — Browser দিয়ে (EC2 Instance Connect):
→ Instance select করো (checkbox)
→ "Connect" button (top) click করো
→ "EC2 Instance Connect" tab
→ Username: ec2-user
→ "Connect" button click করো
→ Browser এ terminal খুলবে ✅

Method B — Local terminal দিয়ে:
Terminal এ:
ssh -i ~/Downloads/lab-key.pem ec2-user@52.77.x.x
(52.77.x.x = তোমার instance এর Public IPv4)

Windows (PowerShell):
ssh -i C:\Users\YourName\Downloads\lab-key.pem ec2-user@52.77.x.x
```

**Step 4: Bastion এ PostgreSQL Client Install করো**
```
Bastion terminal এ চালাও:
sudo dnf install -y postgresql16
psql --version
```

---

### 💻 CLI Method

```bash
source /tmp/lab-ids.env

# Key pair তৈরি করো
aws ec2 create-key-pair \
    --key-name lab-key \
    --key-type rsa \
    --query 'KeyMaterial' \
    --output text > ~/.ssh/lab-key.pem
chmod 400 ~/.ssh/lab-key.pem
echo "Key saved: ~/.ssh/lab-key.pem"

# Latest Amazon Linux 2023 AMI খোঁজো
AMI_ID=$(aws ec2 describe-images \
    --owners amazon \
    --filters \
        "Name=name,Values=al2023-ami-2023*x86_64*" \
        "Name=state,Values=available" \
    --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
    --output text)
echo "AMI: $AMI_ID"

# Bastion launch
BASTION_ID=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.micro \
    --key-name lab-key \
    --security-group-ids $SG_BASTION \
    --subnet-id $PUB_1A \
    --associate-public-ip-address \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-bastion}]' \
    --query 'Instances[0].InstanceId' --output text)
echo "Bastion: $BASTION_ID"

# Running হওয়ার অপেক্ষা
aws ec2 wait instance-running --instance-ids $BASTION_ID
echo "Instance running!"

# Public IP নাও
BASTION_IP=$(aws ec2 describe-instances \
    --instance-ids $BASTION_ID \
    --query 'Reservations[0].Instances[0].PublicIpAddress' \
    --output text)
echo "Bastion IP: $BASTION_IP"

# SSH Connect
ssh -i ~/.ssh/lab-key.pem ec2-user@$BASTION_IP

# IDs save
echo "export BASTION_ID=$BASTION_ID" >> /tmp/lab-ids.env
echo "export BASTION_IP=$BASTION_IP" >> /tmp/lab-ids.env
echo "export AMI_ID=$AMI_ID" >> /tmp/lab-ids.env
```

---

## 13.5 Lab 4 — S3: Backup Bucket তৈরি করো

**Duration: ~20 minutes | Cost: 5GB free**

---

### 🖥️ GUI Method

**Step 1: S3 Console এ যাও**
```
Services search: "S3" → S3 click করো
→ "Create bucket" button click করো
```

**Step 2: Bucket তৈরি করো**
```
General configuration:
→ Bucket name: lab-pg-backup-[তোমার-account-id]
  (S3 bucket name globally unique হতে হবে)
  Account ID দেখতে: top right → account name → copy Account ID
  Example: lab-pg-backup-123456789012
→ AWS Region: Asia Pacific (Singapore) ap-southeast-1

Object Ownership:
→ ACLs disabled (recommended) রাখো

Block Public Access settings:
→ সব checkboxes ✅ checked রাখো (default)
  "Block all public access" থাকবে — এটাই চাই

Bucket Versioning:
→ Enable select করো ✅
  (accidental delete protection)

Default encryption:
→ Server-side encryption: Amazon S3 managed keys (SSE-S3)
  (Free — KMS CMK নয়)

→ "Create bucket" button click করো
✅ Bucket created দেখাবে
```

**Step 3: Lifecycle Rule তৈরি করো**
```
Bucket name এ click করো → bucket ভেতরে যাও
→ "Management" tab click করো
→ "Create lifecycle rule" button

Lifecycle rule details:
→ Lifecycle rule name: BackupRetentionPolicy
→ Rule scope: "Apply to all objects in the bucket" select করো
→ Acknowledge: checkbox ✅

Lifecycle rule actions:
→ "Transition current versions of objects between storage classes" ✅

Transition configuration:
→ "Add transition" click করো
   → Storage class transition: Standard-IA
   → Days after object creation: 30
→ "Add transition" আবার click করো
   → Storage class transition: Glacier Instant Retrieval
   → Days after object creation: 90

→ "Expire current versions of objects" ✅
   → Days after object creation: 365

→ "Create rule" click করো
✅ Lifecycle rule created
```

**Step 4: Test File Upload করো**
```
"Objects" tab → "Upload" button
→ "Add files" click করো → একটা test file select করো
  (অথবা drag and drop)
→ "Upload" button click করো
✅ Upload succeeded দেখাবে

Download test:
→ File name এ click করো
→ "Download" button click করো
```

---

### 💻 CLI Method

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
BUCKET="lab-pg-backup-${ACCT}"

# Bucket তৈরি
aws s3api create-bucket \
    --bucket $BUCKET \
    --region ap-southeast-1 \
    --create-bucket-configuration LocationConstraint=ap-southeast-1

# Public access block
aws s3api put-public-access-block \
    --bucket $BUCKET \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Versioning
aws s3api put-bucket-versioning \
    --bucket $BUCKET \
    --versioning-configuration Status=Enabled

# Lifecycle
aws s3api put-bucket-lifecycle-configuration \
    --bucket $BUCKET \
    --lifecycle-configuration '{
        "Rules": [{
            "ID": "BackupRetentionPolicy",
            "Status": "Enabled",
            "Filter": {},
            "Transitions": [
                {"Days": 30, "StorageClass": "STANDARD_IA"},
                {"Days": 90, "StorageClass": "GLACIER_IR"}
            ],
            "Expiration": {"Days": 365}
        }]
    }'

# Test upload/download
echo "test backup $(date)" > /tmp/test.txt
aws s3 cp /tmp/test.txt s3://${BUCKET}/backups/test.txt
aws s3 ls s3://${BUCKET}/backups/
aws s3 cp s3://${BUCKET}/backups/test.txt /tmp/downloaded.txt
echo "Downloaded: $(cat /tmp/downloaded.txt)"

echo "export BUCKET=$BUCKET" >> /tmp/lab-ids.env
echo "✅ Lab 4 complete!"
```

---

## 13.6 Lab 5 — RDS: PostgreSQL Database তৈরি করো

**Duration: ~45 minutes (RDS creation ~10 min) | Cost: db.t3.micro Free Tier**
**⚠️ Multi-AZ: No! Single-AZ only রাখো**

---

### 🖥️ GUI Method

**Step 1: Parameter Group তৈরি করো**
```
Services search: "RDS" → RDS click করো
→ Left menu: "Parameter groups" → "Create parameter group"

Parameter group details:
→ Parameter group family: postgres16
→ Type: DB Parameter Group
→ Group name: lab-pg16-params
→ Description: Lab PostgreSQL 16 Parameters
→ "Create" button click করো

Parameters Edit করো:
→ "lab-pg16-params" এ click করো
→ "Edit parameters" button
→ Search: "log_min_duration_statement"
→ Value: 1000 (1 second এর বেশি slow query log হবে)
→ Search: "log_connections"
→ Value: 1

→ "Save changes" button click করো
```

**Step 2: RDS Instance তৈরি করো**
```
Left menu: "Databases" → "Create database" button

Choose a database creation method:
→ "Standard create" select করো

Engine options:
→ "PostgreSQL" click করো
→ Engine version: PostgreSQL 16.x (latest 16) select করো

Templates:
→ "Free tier" select করো ✅
  (এটা select করলে Multi-AZ automatically disabled হয়)

Settings:
→ DB instance identifier: lab-postgres
→ Master username: postgres
→ Credentials management: "Self managed"
→ Master password: LabPass@2024!
→ Confirm master password: LabPass@2024!

Instance configuration:
→ DB instance class: db.t3.micro (Free tier eligible ✅)

Storage:
→ Storage type: General Purpose SSD (gp2)
→ Allocated storage: 20 GB
→ Storage autoscaling: Disable (uncheck)
  ⚠️ Enable রাখলে storage বাড়লে charge হবে

Connectivity:
→ Virtual private cloud (VPC): "lab-vpc" select করো
→ DB subnet group: "lab-db-subnet-group" select করো
→ Public access: No ✅ (private রাখো)
→ VPC security group: "Choose existing"
   → "default" remove করো (X button)
   → "lab-db-sg" add করো
→ Availability Zone: ap-southeast-1a

Database authentication:
→ Password authentication রাখো

Additional configuration → click to expand:
→ Initial database name: labdb
→ DB parameter group: "lab-pg16-params" select করো
→ Backup retention period: 1 day (minimum)
→ Deletion protection: Disable (lab এ delete করতে হবে)

→ "Create database" button click করো

⏳ Status: "Creating" → ~10 minutes পর "Available"
```

**Step 3: Endpoint দেখো**
```
Databases → "lab-postgres" click করো
→ "Connectivity & security" tab
→ "Endpoint & port" section
→ Endpoint: lab-postgres.xxxx.ap-southeast-1.rds.amazonaws.com
→ Port: 5432
→ এই endpoint copy করো — পরে দরকার হবে
```

**Step 4: Bastion দিয়ে Connect করো**

*Local machine এর terminal এ:*
```bash
# SSH Tunnel তৈরি করো
# BASTION_IP = তোমার bastion এর public IP (Lab 3 থেকে)
# RDS_ENDPOINT = এইমাত্র copy করা endpoint

ssh -i ~/.ssh/lab-key.pem \
    -L 5433:lab-postgres.xxxx.ap-southeast-1.rds.amazonaws.com:5432 \
    -N ec2-user@BASTION_IP &

# Tunnel চলছে কিনা দেখো
echo "Tunnel running on local port 5433"

# psql দিয়ে connect করো
psql -h 127.0.0.1 -p 5433 -U postgres -d labdb
# Password: LabPass@2024!

# Connect হলে কিছু কাজ করো:
```

```sql
-- Version দেখো
SELECT version();

-- Database list দেখো
\l

-- Test table তৈরি করো
CREATE TABLE test_orders (
    id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    item    TEXT NOT NULL,
    amount  NUMERIC(10,2),
    created TIMESTAMPTZ DEFAULT NOW()
);

-- Data insert করো
INSERT INTO test_orders (item, amount) VALUES
    ('Laptop', 999.99),
    ('Mouse', 29.99),
    ('Keyboard', 79.99);

-- Query করো
SELECT * FROM test_orders;

-- Explain দেখো
EXPLAIN ANALYZE SELECT * FROM test_orders WHERE amount > 50;

-- RDS specific: pg_stat_activity দেখো
SELECT pid, usename, application_name, state
FROM pg_stat_activity;
```

---

### 💻 CLI Method

```bash
source /tmp/lab-ids.env

# Parameter Group
aws rds create-db-parameter-group \
    --db-parameter-group-name lab-pg16-params \
    --db-parameter-group-family postgres16 \
    --description "Lab PostgreSQL 16 Parameters"

aws rds modify-db-parameter-group \
    --db-parameter-group-name lab-pg16-params \
    --parameters \
        "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate" \
        "ParameterName=log_connections,ParameterValue=1,ApplyMethod=immediate"

# RDS Instance
aws rds create-db-instance \
    --db-instance-identifier lab-postgres \
    --db-instance-class db.t3.micro \
    --engine postgres \
    --engine-version "16.3" \
    --master-username postgres \
    --master-user-password 'LabPass@2024!' \
    --allocated-storage 20 \
    --storage-type gp2 \
    --no-storage-encrypted \
    --db-subnet-group-name lab-db-subnet-group \
    --vpc-security-group-ids $SG_DB \
    --db-parameter-group-name lab-pg16-params \
    --no-multi-az \
    --no-publicly-accessible \
    --backup-retention-period 1 \
    --db-name labdb \
    --no-deletion-protection \
    --tags Key=Name,Value=lab-postgres

echo "RDS creating... (~10 minutes)"
aws rds wait db-instance-available --db-instance-identifier lab-postgres
echo "RDS ready!"

RDS_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier lab-postgres \
    --query 'DBInstances[0].Endpoint.Address' --output text)
echo "Endpoint: $RDS_ENDPOINT"

# SSH Tunnel
ssh -i ~/.ssh/lab-key.pem \
    -L 5433:${RDS_ENDPOINT}:5432 \
    -N -f ec2-user@$BASTION_IP
echo "Tunnel active on port 5433"

# Connect
psql -h 127.0.0.1 -p 5433 -U postgres -d labdb

echo "export RDS_ENDPOINT=$RDS_ENDPOINT" >> /tmp/lab-ids.env
```

---

## 13.7 Lab 6 — RDS Snapshot এবং Restore

**Duration: ~30 minutes | Cost: Free (up to DB size)**

---

### 🖥️ GUI Method

**Step 1: Manual Snapshot নাও**
```
RDS → Databases → "lab-postgres" click করো
→ "Actions" dropdown → "Take snapshot"
→ Snapshot name: lab-postgres-manual-snap
→ "Take snapshot" click করো

Status: "Creating" → "Available" (~5 minutes)
→ Left menu: "Snapshots" এ দেখা যাবে
```

**Step 2: Snapshot থেকে Restore করো**
```
Snapshots → "lab-postgres-manual-snap" select করো
→ "Actions" dropdown → "Restore snapshot"

Settings:
→ DB instance identifier: lab-postgres-restored
→ DB instance class: db.t3.micro
→ VPC: lab-vpc
→ DB subnet group: lab-db-subnet-group
→ Public access: No
→ VPC security group: lab-db-sg
→ Multi-AZ: No

→ "Restore DB instance" click করো
⏳ ~10 minutes
```

**Step 3: Restored Instance এ Connect করো**
```
Databases → "lab-postgres-restored" → endpoint copy করো

New SSH tunnel (port 5434):
ssh -i ~/.ssh/lab-key.pem \
    -L 5434:RESTORED_ENDPOINT:5432 \
    -N ec2-user@BASTION_IP &

psql -h 127.0.0.1 -p 5434 -U postgres -d labdb
```

```sql
-- Original data আছে কিনা verify করো
SELECT * FROM test_orders;
-- ✅ Data দেখাবে
```

**Step 4: Snapshot Delete করো (cleanup)**
```
Snapshots → "lab-postgres-manual-snap" select করো
→ "Actions" → "Delete snapshot"
→ Confirm: "delete me" type করো
→ "Delete" click করো
```

**Step 5: Restored Instance Delete করো**
```
Databases → "lab-postgres-restored" select করো
→ "Actions" → "Delete"
→ Create final snapshot: No (uncheck)
→ "I acknowledge..." checkbox ✅
→ Type "delete me" in confirmation box
→ "Delete" click করো
```

---

### 💻 CLI Method

```bash
source /tmp/lab-ids.env

# Snapshot তৈরি
SNAP_ID="lab-snap-$(date +%Y%m%d-%H%M)"
aws rds create-db-snapshot \
    --db-instance-identifier lab-postgres \
    --db-snapshot-identifier $SNAP_ID

aws rds wait db-snapshot-completed --db-snapshot-identifier $SNAP_ID
echo "Snapshot ready: $SNAP_ID"

# Restore করো
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier lab-postgres-restored \
    --db-snapshot-identifier $SNAP_ID \
    --db-instance-class db.t3.micro \
    --db-subnet-group-name lab-db-subnet-group \
    --vpc-security-group-ids $SG_DB \
    --no-multi-az \
    --no-publicly-accessible

aws rds wait db-instance-available \
    --db-instance-identifier lab-postgres-restored
echo "Restored!"

# Verify এবং cleanup
RESTORED_EP=$(aws rds describe-db-instances \
    --db-instance-identifier lab-postgres-restored \
    --query 'DBInstances[0].Endpoint.Address' --output text)

# Cleanup
aws rds delete-db-instance \
    --db-instance-identifier lab-postgres-restored \
    --skip-final-snapshot --delete-automated-backups
aws rds delete-db-snapshot --db-snapshot-identifier $SNAP_ID
echo "✅ Lab 6 cleanup done"
```

---

## 13.8 Lab 7 — CloudWatch: Monitoring Setup করো

**Duration: ~30 minutes | Cost: Basic monitoring free**

---

### 🖥️ GUI Method

**Step 1: SNS Topic তৈরি করো**
```
Services search: "SNS" → Simple Notification Service
→ Left menu: "Topics" → "Create topic"

Details:
→ Type: Standard
→ Name: lab-db-alerts
→ "Create topic" click করো

Subscription তৈরি করো:
→ Topic ARN দেখাবে → "Create subscription" button
→ Protocol: Email
→ Endpoint: তোমার email address
→ "Create subscription" click করো

⚠️ Email verify করো! Confirmation email আসবে → link এ click করো
```

**Step 2: CPU Alarm তৈরি করো**
```
Services search: "CloudWatch" → CloudWatch
→ Left menu: "Alarms" → "All alarms"
→ "Create alarm" button

Step 1 — Select metric:
→ "Select metric" button
→ "AWS namespaces" → "RDS" → "Per-Database Metrics"
→ DBInstanceIdentifier = "lab-postgres", MetricName = "CPUUtilization"
→ checkbox ✅ → "Select metric"

Metric and conditions:
→ Statistic: Average
→ Period: 5 minutes
→ Threshold type: Static
→ Condition: Greater than
→ Value: 70
→ Next

Notification:
→ "In alarm" → "Select an existing SNS topic"
→ "lab-db-alerts" select করো
→ Next

Name and description:
→ Alarm name: Lab-RDS-CPU-High
→ "Create alarm" click করো
✅ Alarm created
```

**Step 3: Storage Alarm তৈরি করো**
```
"Create alarm" আবার click করো
→ "Select metric" → RDS → Per-Database Metrics
→ DBInstanceIdentifier = "lab-postgres", MetricName = "FreeStorageSpace"
→ Select

Metric:
→ Statistic: Average
→ Period: 5 minutes
→ Threshold: Lower/Equal → 5368709120 (5GB in bytes)
→ SNS: lab-db-alerts
→ Alarm name: Lab-RDS-Storage-Low
→ "Create alarm" click করো
```

**Step 4: Metrics Graph দেখো**
```
Left menu: "Metrics" → "All metrics"
→ "AWS namespaces" → "RDS"
→ "Per-Database Metrics"
→ DBInstanceIdentifier: lab-postgres
→ CPUUtilization checkbox ✅
→ DatabaseConnections checkbox ✅
→ FreeStorageSpace checkbox ✅

→ Top right: Time range → "1h" select করো
→ Graph দেখাবে
```

**Step 5: Dashboard তৈরি করো**
```
Left menu: "Dashboards" → "Create dashboard"
→ Dashboard name: Lab-PostgreSQL
→ "Create dashboard" click করো

Widget type: "Line" select → "Next"

Metrics:
→ RDS → Per-Database Metrics
→ lab-postgres এর CPUUtilization এবং DatabaseConnections select করো
→ "Create widget" click করো

আরো widget যোগ করো:
→ "Add widget" → Line
→ FreeStorageSpace metric যোগ করো

→ "Save dashboard" click করো
```

---

### 💻 CLI Method

```bash
source /tmp/lab-ids.env

# SNS Topic
SNS_ARN=$(aws sns create-topic \
    --name lab-db-alerts \
    --query 'TopicArn' --output text)
aws sns subscribe --topic-arn $SNS_ARN \
    --protocol email --notification-endpoint your-email@gmail.com
echo "SNS: $SNS_ARN (confirm email!)"

# CPU Alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "Lab-RDS-CPU-High" \
    --namespace "AWS/RDS" \
    --metric-name "CPUUtilization" \
    --dimensions Name=DBInstanceIdentifier,Value=lab-postgres \
    --statistic Average --period 300 \
    --evaluation-periods 1 --threshold 70 \
    --comparison-operator GreaterThanThreshold \
    --alarm-actions $SNS_ARN --ok-actions $SNS_ARN

# Storage Alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "Lab-RDS-Storage-Low" \
    --namespace "AWS/RDS" \
    --metric-name "FreeStorageSpace" \
    --dimensions Name=DBInstanceIdentifier,Value=lab-postgres \
    --statistic Average --period 300 \
    --evaluation-periods 1 --threshold 5368709120 \
    --comparison-operator LessThanThreshold \
    --alarm-actions $SNS_ARN

# Alarm status দেখো
aws cloudwatch describe-alarms \
    --alarm-name-prefix "Lab-RDS" \
    --query 'MetricAlarms[*].[AlarmName,StateValue]' \
    --output table

# CPU metric manually দেখো
aws cloudwatch get-metric-statistics \
    --namespace AWS/RDS \
    --metric-name CPUUtilization \
    --dimensions Name=DBInstanceIdentifier,Value=lab-postgres \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 300 --statistics Average \
    --query 'sort_by(Datapoints,&Timestamp)[*].[Timestamp,Average]' \
    --output table

echo "export SNS_ARN=$SNS_ARN" >> /tmp/lab-ids.env
echo "✅ Lab 7 complete!"
```

---

## 13.9 Lab 8 — Parameter Store: Config Management

**Duration: ~20 minutes | Cost: FREE**

---

### 🖥️ GUI Method

**Step 1: Parameter Store এ যাও**
```
Services search: "Systems Manager" → AWS Systems Manager
→ Left menu (scroll down): "Parameter Store"
→ "Create parameter" button
```

**Step 2: Parameters তৈরি করো**

*Database Host:*
```
Name: /lab/database/host
Description: Lab RDS endpoint
Tier: Standard (Free)
Type: String
Value: [তোমার RDS endpoint paste করো]
→ "Create parameter" click করো
```

*Database Port:*
```
"Create parameter" আবার click করো
Name: /lab/database/port
Type: String
Value: 5432
→ "Create parameter"
```

*Database Password (Encrypted):*
```
"Create parameter" click করো
Name: /lab/database/password
Type: SecureString
KMS key source: My current account
KMS Key ID: alias/aws/ssm  ← AWS managed key (FREE)
Value: LabPass@2024!
→ "Create parameter"
```

**Step 3: Parameters দেখো**
```
Parameter Store → সব parameters দেখাবে
→ /lab/database/password এ click করো
→ "Show" button click করো → decrypted value দেখাবে
```

**Step 4: Script এ ব্যবহার করো**
```bash
# Terminal এ:
DB_HOST=$(aws ssm get-parameter \
    --name "/lab/database/host" \
    --query 'Parameter.Value' --output text)
DB_PASS=$(aws ssm get-parameter \
    --name "/lab/database/password" \
    --with-decryption \
    --query 'Parameter.Value' --output text)

echo "Host: $DB_HOST"
# Password print করো না log এ!

# Connect করো (SSH tunnel active থাকলে)
PGPASSWORD=$DB_PASS psql -h 127.0.0.1 -p 5433 \
    -U postgres -d labdb -c "SELECT current_user, version();"
```

---

### 💻 CLI Method

```bash
source /tmp/lab-ids.env

# Parameters store করো
aws ssm put-parameter --name "/lab/database/host" \
    --value "$RDS_ENDPOINT" --type String
aws ssm put-parameter --name "/lab/database/port" \
    --value "5432" --type String
aws ssm put-parameter --name "/lab/database/dbname" \
    --value "labdb" --type String
aws ssm put-parameter --name "/lab/database/username" \
    --value "postgres" --type String
aws ssm put-parameter --name "/lab/database/password" \
    --value "LabPass@2024!" \
    --type SecureString \
    --key-id alias/aws/ssm  # AWS managed key = FREE

# সব parameters দেখো
aws ssm get-parameters-by-path \
    --path "/lab/database" \
    --with-decryption \
    --query 'Parameters[*].[Name,Value]' \
    --output table

echo "✅ Lab 8 complete!"
```

---

## 13.10 Lab 9 — Self-managed PostgreSQL on EC2

**Duration: ~1 hour | Cost: t3.micro Free Tier**
**Goal: Guide 1 এর সব concepts AWS EC2 তে practice করো**

---

### 🖥️ GUI Method

**Step 1: PostgreSQL EC2 Launch করো**
```
EC2 → "Launch instances"

Name: lab-pg-ec2
AMI: Amazon Linux 2023 (same as before)
Instance type: t3.micro

Key pair: lab-key

Network settings → Edit:
→ VPC: lab-vpc
→ Subnet: lab-db-1a (private subnet)
→ Auto-assign public IP: Disable
  (Private subnet — Bastion দিয়ে access করবো)
→ Security group: lab-db-sg

Advanced details → User data:
→ নিচের script paste করো:
```

```bash
#!/bin/bash
# PostgreSQL 16 auto-install
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf -qy module disable postgresql
dnf install -y postgresql16-server postgresql16-contrib
/usr/pgsql-16/bin/postgresql-16-setup initdb
systemctl enable --now postgresql-16
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'PGPass@2024!';"
sudo -u postgres psql -c "CREATE DATABASE practicedb;"
echo "PostgreSQL 16 ready at $(date)" > /tmp/pg-setup.log
```

```
→ "Launch instance" click করো
```

**Step 2: SSH দিয়ে Connect করো**

*ProxyJump দিয়ে (Bastion → EC2):*
```bash
# Private IP নাও
# EC2 → Instances → lab-pg-ec2 → Private IPv4 address copy করো
# Example: 10.0.10.25

# SSH config এ add করো
cat >> ~/.ssh/config << 'EOF'
Host lab-pg-ec2
    HostName 10.0.10.25        # তোমার private IP
    User ec2-user
    IdentityFile ~/.ssh/lab-key.pem
    ProxyJump lab-bastion      # Bastion দিয়ে jump করবে
    StrictHostKeyChecking no
EOF

# Connect করো
ssh lab-pg-ec2

# Setup হয়েছে কিনা দেখো (~2 min লাগে user-data শেষ হতে)
cat /tmp/pg-setup.log
systemctl status postgresql-16
```

**Step 3: PostgreSQL Practice করো**
```bash
# Port forward তৈরি করো (local 5435 → EC2's PostgreSQL)
ssh -L 5435:10.0.10.25:5432 lab-bastion -N -f

# Connect করো
psql -h 127.0.0.1 -p 5435 -U postgres -d practicedb
```

```sql
-- Guide 1 এর Architecture concepts practice করো

-- 1. Hidden MVCC columns দেখো
CREATE TABLE mvcc_demo (id INT, val TEXT);
INSERT INTO mvcc_demo VALUES (1, 'version1');
SELECT xmin, xmax, ctid, * FROM mvcc_demo;

-- UPDATE করো → new tuple তৈরি হবে
UPDATE mvcc_demo SET val = 'version2' WHERE id = 1;
SELECT xmin, xmax, ctid, * FROM mvcc_demo;  -- ctid বদলেছে

-- 2. WAL position দেখো
SELECT pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());

-- 3. Background processes
SELECT pid, backend_type, state
FROM pg_stat_activity ORDER BY backend_type;

-- 4. Buffer statistics
SELECT * FROM pg_stat_bgwriter;

-- 5. Shared buffers hit rate
SELECT
    ROUND(
        sum(heap_blks_hit) * 100.0 /
        NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0),
    2) AS hit_rate
FROM pg_statio_user_tables;

-- 6. Extensions install করো
CREATE EXTENSION pg_stat_statements;
CREATE EXTENSION pgcrypto;

-- 7. EXPLAIN দেখো
CREATE TABLE big_test AS
SELECT generate_series(1, 100000) AS id,
       md5(random()::text) AS val;
CREATE INDEX idx_big ON big_test(id);

EXPLAIN ANALYZE SELECT * FROM big_test WHERE id = 5000;
-- Index Scan দেখাবে

-- 8. Transaction এবং Lock
BEGIN;
SELECT * FROM mvcc_demo FOR UPDATE;
-- অন্য terminal থেকে same row update করার চেষ্টা করো → block হবে
COMMIT;

-- 9. Vacuum manually চালাও
VACUUM VERBOSE ANALYZE mvcc_demo;
```

---

### 💻 CLI Method

```bash
source /tmp/lab-ids.env

# EC2 Launch with user-data
USER_DATA='#!/bin/bash
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf -qy module disable postgresql
dnf install -y postgresql16-server postgresql16-contrib
/usr/pgsql-16/bin/postgresql-16-setup initdb
systemctl enable --now postgresql-16
sudo -u postgres psql -c "ALTER USER postgres PASSWORD '"'"'PGPass@2024!'"'"';"
sudo -u postgres psql -c "CREATE DATABASE practicedb;"
echo "done" > /tmp/pg-setup.log'

PG_EC2=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.micro \
    --key-name lab-key \
    --security-group-ids $SG_DB \
    --subnet-id $DB_1A \
    --no-associate-public-ip-address \
    --user-data "$USER_DATA" \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-pg-ec2}]' \
    --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $PG_EC2
PG_IP=$(aws ec2 describe-instances --instance-ids $PG_EC2 \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)
echo "PG EC2: $PG_EC2 ($PG_IP)"

# SSH config
cat >> ~/.ssh/config << EOF
Host lab-pg-ec2
    HostName $PG_IP
    User ec2-user
    IdentityFile ~/.ssh/lab-key.pem
    ProxyJump lab-bastion
    StrictHostKeyChecking no
EOF

# Port forward
ssh -L 5435:${PG_IP}:5432 lab-bastion -N -f

# ~2 minutes পরে connect করো
psql -h 127.0.0.1 -p 5435 -U postgres -d practicedb

echo "export PG_EC2=$PG_EC2" >> /tmp/lab-ids.env
echo "export PG_IP=$PG_IP" >> /tmp/lab-ids.env
```

---

## 13.11 Lab 10 — Streaming Replication on EC2

**Duration: ~1.5 hours | Cost: 2x t3.micro = Free Tier combined**

---

### 🖥️ GUI Method

**Step 1: Replica EC2 Launch করো**
```
EC2 → "Launch instances"

Name: lab-pg-replica
AMI: Amazon Linux 2023
Instance type: t3.micro
Key pair: lab-key

Network settings:
→ VPC: lab-vpc
→ Subnet: lab-db-1b  (different AZ from primary!)
→ Auto-assign public IP: Disable
→ Security group: lab-db-sg

Advanced → User data:
```

```bash
#!/bin/bash
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf -qy module disable postgresql
dnf install -y postgresql16-server postgresql16-contrib
echo "PG installed" > /tmp/replica-ready.log
```

```
→ "Launch instance"
→ Private IP নাও (e.g., 10.0.11.30)
```

**Step 2: SSH Config তৈরি করো**
```bash
cat >> ~/.ssh/config << 'EOF'
Host lab-pg-replica
    HostName 10.0.11.30       # Replica private IP
    User ec2-user
    IdentityFile ~/.ssh/lab-key.pem
    ProxyJump lab-bastion
    StrictHostKeyChecking no
EOF
```

**Step 3: Primary Configure করো**
```bash
ssh lab-pg-ec2 << REMOTE
# Replication settings
sudo -u postgres psql << 'SQL'
ALTER SYSTEM SET wal_level = replica;
ALTER SYSTEM SET max_wal_senders = 5;
ALTER SYSTEM SET wal_keep_size = '256MB';
ALTER SYSTEM SET hot_standby = on;
SQL

# pg_hba.conf এ replica allow
REPLICA_IP="10.0.11.30"
echo "host replication replicator ${REPLICA_IP}/32 md5" | \
    sudo tee -a /var/lib/pgsql/16/data/pg_hba.conf

# Replication user
sudo -u postgres psql -c \
    "CREATE USER replicator REPLICATION LOGIN PASSWORD 'ReplPass@2024!';"

sudo systemctl restart postgresql-16
echo "Primary configured!"
REMOTE
```

**Step 4: Replica Setup করো**
```bash
# Primary IP জানো (EC2 → lab-pg-ec2 → Private IP)
# Example: 10.0.10.25

ssh lab-pg-replica << REMOTE
# pg_basebackup চালাও
sudo -u postgres pg_basebackup \
    -h 10.0.10.25 \
    -U replicator \
    -D /var/lib/pgsql/16/data \
    -Xs -R -P -W
# Password: ReplPass@2024!

# Replica start করো
sudo -u postgres bash -c \
    "echo 'hot_standby = on' >> /var/lib/pgsql/16/data/postgresql.conf"
sudo systemctl start postgresql-16
sudo systemctl enable postgresql-16
sudo systemctl status postgresql-16 | grep Active
echo "Replica ready!"
REMOTE
```

**Step 5: Verify করো**
```bash
# Port forward Replica এ
ssh -L 5436:10.0.11.30:5432 lab-bastion -N -f

# Primary এ check
psql -h 127.0.0.1 -p 5435 -U postgres << 'SQL'
SELECT application_name, client_addr, state,
    pg_size_pretty(sent_lsn - replay_lsn) AS lag
FROM pg_stat_replication;
SQL

# Replica এ check
psql -h 127.0.0.1 -p 5436 -U postgres << 'SQL'
SELECT pg_is_in_recovery() AS is_standby,
    now() - pg_last_xact_replay_timestamp() AS lag;
SQL

# Live test
psql -h 127.0.0.1 -p 5435 -U postgres -d practicedb \
    -c "INSERT INTO mvcc_demo VALUES (99, 'replication test $(date)');"

# Replica তে দেখো
psql -h 127.0.0.1 -p 5436 -U postgres -d practicedb \
    -c "SELECT * FROM mvcc_demo ORDER BY id DESC LIMIT 3;"
# ✅ New row দেখাবে
```

---

### 💻 CLI Method

```bash
source /tmp/lab-ids.env

# Replica EC2
REPLICA_EC2=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.micro \
    --key-name lab-key \
    --security-group-ids $SG_DB \
    --subnet-id $DB_1B \
    --no-associate-public-ip-address \
    --user-data '#!/bin/bash
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf -qy module disable postgresql
dnf install -y postgresql16-server postgresql16-contrib
echo "ready" > /tmp/replica-ready.log' \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-pg-replica}]' \
    --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $REPLICA_EC2
REPLICA_IP=$(aws ec2 describe-instances --instance-ids $REPLICA_EC2 \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)
echo "Replica: $REPLICA_EC2 ($REPLICA_IP)"

# SSH config
cat >> ~/.ssh/config << EOF
Host lab-pg-replica
    HostName $REPLICA_IP
    User ec2-user
    IdentityFile ~/.ssh/lab-key.pem
    ProxyJump lab-bastion
    StrictHostKeyChecking no
EOF

# Primary configure (SSH দিয়ে)
ssh lab-pg-ec2 "
sudo -u postgres psql -c \"ALTER SYSTEM SET wal_level = replica;
ALTER SYSTEM SET max_wal_senders = 5;
ALTER SYSTEM SET wal_keep_size = '256MB';\"
echo 'host replication replicator ${REPLICA_IP}/32 md5' | \
    sudo tee -a /var/lib/pgsql/16/data/pg_hba.conf
sudo -u postgres psql -c \
    \"CREATE USER replicator REPLICATION LOGIN PASSWORD 'ReplPass@2024!';\"
sudo systemctl restart postgresql-16"

# Replica setup
ssh lab-pg-replica "
sudo -u postgres pg_basebackup -h ${PG_IP} -U replicator \
    -D /var/lib/pgsql/16/data -Xs -R -P -W <<< 'ReplPass@2024!'
echo 'hot_standby = on' | sudo -u postgres tee -a /var/lib/pgsql/16/data/postgresql.conf
sudo systemctl start postgresql-16 && sudo systemctl enable postgresql-16"

# Verify
ssh -L 5436:${REPLICA_IP}:5432 lab-bastion -N -f
psql -h 127.0.0.1 -p 5435 -U postgres -c "SELECT * FROM pg_stat_replication;"
psql -h 127.0.0.1 -p 5436 -U postgres -c "SELECT pg_is_in_recovery();"

echo "export REPLICA_EC2=$REPLICA_EC2" >> /tmp/lab-ids.env
echo "export REPLICA_IP=$REPLICA_IP" >> /tmp/lab-ids.env
echo "✅ Lab 10 complete!"
```

---

## 13.12 Lab Cleanup — সব Resources Delete করো

**⚠️ প্রতিটা lab শেষে অথবা সব lab শেষে এটা করো। Charge এড়াতে critical!**

---

### 🖥️ GUI Method — Manual Cleanup

**Step 1: EC2 Instances Terminate করো**
```
EC2 → Instances
→ সব lab instances select করো:
   lab-bastion, lab-pg-ec2, lab-pg-replica
→ Instance state → Terminate instance
→ "Terminate" confirm করো

⏳ State: "Shutting-down" → "Terminated"
```

**Step 2: RDS Delete করো**
```
RDS → Databases
→ "lab-postgres" select করো
→ "Actions" → "Delete"
→ Create final snapshot: No (uncheck)
→ "I acknowledge that upon instance deletion, automated backups,
   including system snapshots and point-in-time recovery..." ✅
→ Type "delete me" in the box
→ "Delete" click করো

⏳ ~5 minutes
```

**Step 3: RDS Snapshot Delete করো**
```
RDS → Snapshots
→ সব "lab-" prefix এর snapshots select করো
→ "Actions" → "Delete Snapshot"
```

**Step 4: Security Groups Delete করো**
```
EC2 → Security Groups
→ "lab-bastion-sg" select করো → "Actions" → "Delete security groups"
→ "lab-db-sg" same করো

Note: Referenced security group আগে delete হতে হবে
(lab-bastion-sg কে lab-db-sg reference করে)
তাই lab-db-sg আগে delete করো, তারপর lab-bastion-sg
```

**Step 5: Subnets Delete করো**
```
VPC → Subnets
→ lab-public-1a, lab-db-1a, lab-db-1b select করো
→ "Actions" → "Delete subnet"
```

**Step 6: Route Table Delete করো**
```
VPC → Route tables
→ "lab-public-rt" select করো
→ Subnet associations tab → Edit → Disassociate সব
→ "Actions" → "Delete route table"
```

**Step 7: Internet Gateway Delete করো**
```
VPC → Internet gateways
→ "lab-igw" select করো
→ "Actions" → "Detach from VPC" → confirm
→ "Actions" → "Delete internet gateway" → confirm
```

**Step 8: VPC Delete করো**
```
VPC → Your VPCs
→ "lab-vpc" select করো
→ "Actions" → "Delete VPC"
→ "delete" type করো → "Delete" click করো
```

**Step 9: S3 Bucket Empty করো এবং Delete করো**
```
S3 → Buckets → "lab-pg-backup-..." click করো
→ "Empty" button → "permanently delete" type করো → Empty

Back to bucket list:
→ Bucket select করো → "Delete"
→ Bucket name type করো → Delete
```

**Step 10: CloudWatch Alarms Delete করো**
```
CloudWatch → Alarms → All alarms
→ "Lab-RDS-CPU-High" এবং "Lab-RDS-Storage-Low" select করো
→ "Actions" → "Delete"
→ Dashboard: Dashboards → "Lab-PostgreSQL" → Delete
```

**Step 11: SNS Topic Delete করো**
```
SNS → Topics
→ "lab-db-alerts" select করো
→ "Delete" → "delete me" type করো
```

**Step 12: Parameter Store Cleanup করো**
```
Systems Manager → Parameter Store
→ /lab/database/host, port, dbname, username, password select করো
→ "Delete" button
```

**Step 13: RDS Resources Delete করো**
```
RDS → Subnet groups
→ "lab-db-subnet-group" → "Delete"

RDS → Parameter groups
→ "lab-pg16-params" → "Actions" → "Delete"
```

**Step 14: Key Pair Delete করো**
```
EC2 → Key Pairs
→ "lab-key" select করো → "Actions" → "Delete"
→ "Delete" confirm করো

Local machine এ:
rm -f ~/.ssh/lab-key.pem
```

**Step 15: Cost Verify করো**
```
Top right: Account → Billing Dashboard
→ "Bills" → current month দেখো
→ $0.00 বা very small amount দেখাবে
Free Tier: Free Tier → কতটুকু ব্যবহার হয়েছে দেখো
```

---

### 💻 CLI Method — Automated Cleanup

```bash
source /tmp/lab-ids.env

echo "=== Complete Lab Cleanup ==="

# 1. EC2 Terminate
for id in $PG_EC2 $REPLICA_EC2 $BASTION_ID; do
    [ -n "$id" ] && aws ec2 terminate-instances --instance-ids $id 2>/dev/null && \
        echo "Terminating: $id"
done
[ -n "$BASTION_ID" ] && aws ec2 wait instance-terminated \
    --instance-ids $BASTION_ID 2>/dev/null
echo "✅ EC2 terminated"

# 2. RDS Delete
aws rds delete-db-instance \
    --db-instance-identifier lab-postgres \
    --skip-final-snapshot \
    --delete-automated-backups 2>/dev/null && echo "RDS deleting..."

# 3. Snapshot delete
aws rds describe-db-snapshots \
    --filters "Name=db-instance-id,Values=lab-postgres" \
    --query 'DBSnapshots[*].DBSnapshotIdentifier' \
    --output text | tr '\t' '\n' | while read snap; do
    aws rds delete-db-snapshot --db-snapshot-identifier $snap 2>/dev/null
done

# 4. Key Pair
aws ec2 delete-key-pair --key-name lab-key 2>/dev/null
rm -f ~/.ssh/lab-key.pem
echo "✅ Key pair deleted"

# Wait for RDS deletion (~5 min)
sleep 30

# 5. Security Groups (DB first, then Bastion)
aws ec2 delete-security-group --group-id $SG_DB 2>/dev/null && echo "DB SG deleted"
sleep 5
aws ec2 delete-security-group --group-id $SG_BASTION 2>/dev/null && echo "Bastion SG deleted"

# 6. Subnets
for subnet in $PUB_1A $DB_1A $DB_1B; do
    aws ec2 delete-subnet --subnet-id $subnet 2>/dev/null && echo "Subnet deleted: $subnet"
done

# 7. Route Table
RT_ASSOC=$(aws ec2 describe-route-tables \
    --route-table-ids $PUB_RT \
    --query 'RouteTables[0].Associations[?!Main].RouteTableAssociationId' \
    --output text 2>/dev/null)
[ -n "$RT_ASSOC" ] && aws ec2 disassociate-route-table --association-id $RT_ASSOC 2>/dev/null
aws ec2 delete-route-table --route-table-id $PUB_RT 2>/dev/null && echo "Route table deleted"

# 8. IGW
aws ec2 detach-internet-gateway \
    --internet-gateway-id $IGW_ID --vpc-id $VPC_ID 2>/dev/null
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID 2>/dev/null && echo "IGW deleted"

# 9. VPC
aws ec2 delete-vpc --vpc-id $VPC_ID 2>/dev/null && echo "VPC deleted"

# 10. S3
aws s3 rm s3://${BUCKET} --recursive 2>/dev/null
aws s3api delete-bucket --bucket $BUCKET 2>/dev/null && echo "S3 bucket deleted"

# 11. RDS Resources
aws rds delete-db-subnet-group \
    --db-subnet-group-name lab-db-subnet-group 2>/dev/null
aws rds delete-db-parameter-group \
    --db-parameter-group-name lab-pg16-params 2>/dev/null

# 12. CloudWatch
aws cloudwatch delete-alarms \
    --alarm-names "Lab-RDS-CPU-High" "Lab-RDS-Storage-Low" 2>/dev/null
aws cloudwatch delete-dashboards \
    --dashboard-names "Lab-PostgreSQL" 2>/dev/null

# 13. SNS
aws sns delete-topic --topic-arn $SNS_ARN 2>/dev/null

# 14. Parameter Store
for p in host port dbname username password; do
    aws ssm delete-parameter --name "/lab/database/$p" 2>/dev/null
done

# 15. IAM
ACCT=$(aws sts get-caller-identity --query Account --output text)
aws iam remove-role-from-instance-profile \
    --instance-profile-name lab-ec2-profile \
    --role-name lab-ec2-role 2>/dev/null
aws iam delete-instance-profile \
    --instance-profile-name lab-ec2-profile 2>/dev/null
aws iam detach-role-policy \
    --role-name lab-ec2-role \
    --policy-arn arn:aws:iam::${ACCT}:policy/LabSSMPolicy 2>/dev/null
aws iam delete-role --role-name lab-ec2-role 2>/dev/null
aws iam delete-policy \
    --policy-arn arn:aws:iam::${ACCT}:policy/LabSSMPolicy 2>/dev/null

rm -f /tmp/lab-ids.env
echo "=== ✅ Complete cleanup done! ==="
echo "Check billing: https://console.aws.amazon.com/billing/"
```

---

## 13.13 Lab Sequence — কোন Order এ করবো

```
Day 1 (~2 hours):
  ✅ Lab 0: Account Setup + Billing Alarm (MANDATORY)
  ✅ Lab 1: IAM (30 min)
  ✅ Lab 2: VPC তৈরি (45 min)

Day 2 (~2 hours):
  ✅ Lab 3: Bastion Host (30 min)
  ✅ Lab 4: S3 Bucket (20 min)
  ✅ Lab 5: RDS PostgreSQL (45 min)

Day 3 (~2 hours):
  ✅ Lab 6: Snapshot + Restore (30 min)
  ✅ Lab 7: CloudWatch Monitoring (30 min)
  ✅ Lab 8: Parameter Store (20 min)

Day 4 (~2.5 hours):
  ✅ Lab 9: Self-managed PostgreSQL on EC2 (1 hour)
  ✅ Lab 10: Streaming Replication (1.5 hours)

Day 5:
  ✅ Full Cleanup (30 min)
  ✅ Billing দেখো — $0 হওয়া উচিত

প্রতিটা lab এ:
  GUI Method দিয়ে করো → AWS Console এ সব দেখো
  তারপর CLI Method দিয়ে করো → automation অভ্যাস হবে
  শেষে: Console এ গিয়ে CLI এর কাজ verify করো
```

---

## 13.14 Common Free Tier Mistakes — এড়িয়ে চলো

```
❌ Multi-AZ RDS চালু করা → 2x cost!
   ✅ Fix: Create database → Free tier template select করলে auto-disabled

❌ NAT Gateway তৈরি করা → $0.045/hour = $32/month
   ✅ Fix: Private subnet এ internet দরকার নেই lab এ
         Bastion দিয়ে SSH forward করো

❌ t3.micro এর চেয়ে বড় instance চালু করা
   ✅ Fix: Always t3.micro বা t2.micro

❌ Lab শেষে resource delete না করা
   ✅ Fix: Lab 12 এর cleanup script চালাও

❌ দুটো RDS একসাথে চালানো
   ✅ Free: 750 hours/month। দুটো = 1500 hours → charge হবে
   ✅ Fix: Restore lab শেষে restored instance delete করো

❌ Storage autoscaling RDS এ enable করা
   ✅ Fix: Create করার সময় disable করো

❌ CloudWatch Detailed Monitoring enable করা
   ✅ Basic (5-min): Free
   ❌ Detailed (1-min): $0.015/metric

❌ Secrets Manager ব্যবহার করা
   ✅ Fix: Parameter Store (free) ব্যবহার করো

❌ Customer Managed KMS Key তৈরি করা
   ✅ Fix: AWS managed key (alias/aws/ssm) ব্যবহার করো

✅ Good Habits:
   → প্রতি lab শুরুতে: Billing → Free Tier Usage দেখো
   → প্রতি lab শেষে: সেই lab এর resources delete করো
   → Weekly: Cost Explorer দেখো
   → Billing Alarm: $1 threshold set রাখো (Lab 0 তে করেছ)
```

# 14. Advanced Networking — VPC Flow Logs, CloudTrail, VPC Endpoints

## 14.1 VPC Flow Logs — Network Traffic Monitoring

```
VPC Flow Logs কী:
  VPC এর সব network traffic record করে
  কে কোথায় connect করল, allowed না rejected

DBA এর daily কাজে লাগে:
  "Application কি DB এ reach করছে?"
  "Port 5432 কোন IP থেকে reject হচ্ছে?"
  Security audit: unauthorized access detect
```

### 🖥️ GUI Method

```
VPC → Your VPCs → তোমার VPC select করো
→ Flow logs tab → Create flow log

Filter: All
Aggregation interval: 1 minute
Destination: CloudWatch Logs
Log group: /vpc/flowlogs/lab-vpc
IAM role: Create new role

→ Create flow log
```

**DB Connection Troubleshoot করো:**
```
CloudWatch → Log groups → /vpc/flowlogs/lab-vpc
→ Log Insights

Query:
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter dstPort = 5432
| filter action = "REJECT"
| stats count(*) by srcAddr
| sort count desc

→ কোন IP থেকে DB port reject হচ্ছে সেটা দেখাবে
→ Security Group rule missing → Fix করো
```

### 💻 CLI Method

```bash
# VPC Flow Logs enable
aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids $VPC_ID \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name /vpc/flowlogs/lab-vpc \
    --deliver-logs-permission-arn arn:aws:iam::ACCOUNT:role/vpc-flow-logs-role

# DB connection attempts query করো
aws logs start-query \
    --log-group-name /vpc/flowlogs/lab-vpc \
    --start-time $(date -d "1 hour ago" +%s) \
    --end-time $(date +%s) \
    --query-string 'fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter dstPort = 5432
| filter action = "REJECT"
| stats count(*) as rejections by srcAddr'
```

---

## 14.2 CloudTrail — AWS API Call Audit

```
CloudTrail কী:
  সব AWS API calls record করে
  কে কখন কোন action নিয়েছে

DBA এর জন্য:
  "কে RDS instance delete করল?"
  "কে parameter group modify করল?"
  Compliance: change audit trail
```

### 🖥️ GUI Method

```
Services → CloudTrail → Create trail

Trail name: lab-audit-trail
Storage: New S3 bucket
Events: Management events (Read + Write)

→ Create trail

Recent events দেখো:
CloudTrail → Event history
Filter: Event source = rds.amazonaws.com

→ সব RDS actions দেখাবে (who, when, what)
```

### 💻 CLI Method

```bash
# Recent RDS events
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventSource,AttributeValue=rds.amazonaws.com \
    --start-time $(date -d "24 hours ago" +%s) \
    --query "Events[*].[EventTime,EventName,Username]" \
    --output table | head -20

# কে parameter group change করেছে?
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=ModifyDBParameterGroup \
    --start-time $(date -d "7 days ago" +%s) \
    --query "Events[*].[EventTime,Username,CloudTrailEvent]" \
    --output text
```

---

## 14.3 VPC Endpoints — S3 Access without NAT Gateway

```
Problem:
  Private subnet → S3 (backup) → কীভাবে?
  Option 1: NAT Gateway → Internet → S3 ($32/month + data transfer)
  Option 2: VPC Gateway Endpoint → S3 (FREE, secure, faster)

VPC Gateway Endpoint:
  S3 এবং DynamoDB → FREE
  AWS internal network (no internet)
  Production এ সবসময় এটা use করো
```

### 🖥️ GUI Method

```
VPC → Endpoints → Create endpoint

Service category: AWS services
Service: com.amazonaws.ap-southeast-1.s3
Type: Gateway (এটা FREE)

VPC: lab-vpc
Route tables: private subnet এর route table ✅

→ Create endpoint

✅ Private subnet থেকে S3 freely accessible
   NAT Gateway ছাড়াই!
```

### 💻 CLI Method

```bash
# S3 Gateway Endpoint (FREE)
aws ec2 create-vpc-endpoint \
    --vpc-id $VPC_ID \
    --service-name com.amazonaws.ap-southeast-1.s3 \
    --route-table-ids $PUB_RT \
    --vpc-endpoint-type Gateway

# Verify
aws ec2 describe-vpc-endpoints \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query "VpcEndpoints[*].[ServiceName,State,VpcEndpointType]" \
    --output table

# Test (EC2 private subnet থেকে)
# S3 access এখন directly → No NAT needed
```

---

## 14.4 Systems Manager Session Manager — SSH Key ছাড়া EC2 Access

```
SSM Session Manager:
  SSH Key ছাড়া EC2 তে access করো
  No port 22 needed
  All sessions CloudTrail এ logged
  IAM permission দিয়ে access control

DBA এর জন্য:
  Bastion Host এর alternative
  More secure (no SSH key exposure)
  Better audit trail
```

### 🖥️ GUI Method

```
Prerequisites:
  EC2 Instance এ SSM Agent install থাকতে হবে
  (Amazon Linux 2023 তে by default আছে)
  IAM Role এ AmazonSSMManagedInstanceCore policy থাকতে হবে

EC2 → Instance select করো
→ "Connect" button
→ "Session Manager" tab
→ "Connect" button

→ Browser তে terminal খুলবে ✅
  (SSH key ছাড়া!)
```

### 💻 CLI Method

```bash
# SSM session start
aws ssm start-session --target i-INSTANCE_ID

# Port forwarding (RDS তে access)
aws ssm start-session \
    --target i-BASTION_ID \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters '{
        "host": ["mydb.xxxx.rds.amazonaws.com"],
        "portNumber": ["5432"],
        "localPortNumber": ["5433"]
    }'
# Local 5433 → RDS 5432 (SSH key ছাড়া!)

psql -h 127.0.0.1 -p 5433 -U postgres -d mydb
```

---

*AWS Fundamentals for DBA | Guide 2*
*IAM · VPC · EC2 · EBS · S3 · CloudWatch · Secrets Manager · KMS · Route 53 · AWS CLI*
*Chapter 13: Free Tier Hands-on Labs (GUI + CLI)*
*Foundation for Guide 3: AWS Cloud Native DBA (RDS, Aurora, and Managed Services)*


---

# 15. Connectivity — VPN, Direct Connect, ALB, Transit Gateway

## 15.1 Site-to-Site VPN — On-premise to AWS

```
Site-to-Site VPN:
  On-premise network ↔ AWS VPC
  Internet এর উপর দিয়ে encrypted tunnel
  Setup: ~30 minutes
  Cost: $0.05/hour per VPN connection = ~$36/month
  Bandwidth: up to 1.25 Gbps

DBA এর জন্য কখন লাগে:
  ① On-premise PostgreSQL → RDS migration এ
  ② Hybrid: কিছু DB on-prem, কিছু AWS
  ③ DBA office থেকে private RDS access (alternative to Bastion)
```

### 🖥️ GUI Method

```
VPC → Virtual private gateways → Create virtual private gateway
→ Name: prod-vpg
→ ASN: Amazon default
→ Create

VPC → তোমার VPC → Actions → Attach virtual private gateway
→ prod-vpg attach করো

VPC → Customer gateways → Create customer gateway
→ Name: office-gateway
→ BGP ASN: 65000 (বা তোমার router এর ASN)
→ IP address: [তোমার office এর public IP]
→ Create

VPC → Site-to-Site VPN connections → Create VPN connection
→ Virtual private gateway: prod-vpg
→ Customer gateway: office-gateway
→ Routing: Static
→ Static IP prefixes: 192.168.1.0/24 (তোমার office subnet)
→ Create VPN connection

→ Download configuration (তোমার router type select করো)
→ Office router এ এই configuration apply করো
```

### 💻 CLI Method

```bash
# Virtual Private Gateway
VPG_ID=$(aws ec2 create-vpn-gateway \
    --type ipsec.1 \
    --tag-specifications "ResourceType=vpn-gateway,Tags=[{Key=Name,Value=prod-vpg}]" \
    --query "VpnGateway.VpnGatewayId" --output text)

aws ec2 attach-vpn-gateway --vpn-gateway-id $VPG_ID --vpc-id $VPC_ID

# Customer Gateway (office router)
CGW_ID=$(aws ec2 create-customer-gateway \
    --type ipsec.1 \
    --bgp-asn 65000 \
    --ip-address 203.0.113.1 \  # office public IP
    --tag-specifications "ResourceType=customer-gateway,Tags=[{Key=Name,Value=office-gw}]" \
    --query "CustomerGateway.CustomerGatewayId" --output text)

# VPN Connection
aws ec2 create-vpn-connection \
    --type ipsec.1 \
    --customer-gateway-id $CGW_ID \
    --vpn-gateway-id $VPG_ID \
    --options StaticRoutesOnly=true

# VPN status দেখো
aws ec2 describe-vpn-connections \
    --query "VpnConnections[*].[VpnConnectionId,State,Routes[*].State]" \
    --output table
```

---

## 15.2 AWS Direct Connect — Dedicated Private Line

```
Direct Connect:
  On-premise → AWS direct fiber connection
  No internet (private, dedicated)
  Speed: 1Gbps, 10Gbps, 100Gbps
  Latency: very low, consistent
  Cost: Port charge + data transfer

vs VPN:
  VPN:           Internet over encryption, cheaper, setup easy
  Direct Connect: Dedicated line, expensive, very fast/reliable

DBA এর জন্য কখন লাগে:
  ① Large database migration (100GB+ → RDS)
     VPN দিয়ে কয়েকদিন লাগবে, Direct Connect কয়েক ঘন্টা
  ② Production hybrid: on-prem app → RDS (high bandwidth)
  ③ Regulatory: data must not go over public internet

Setup process (summary):
  1. AWS Direct Connect Console → Create connection
  2. AWS LOA-CFA (Letter of Authorization) নাও
  3. Data center provider এ order করো (datacenter co-location)
  4. Provider AWS এর cage এ cross-connect করবে
  5. Virtual Interface তৈরি করো
  → Setup takes weeks (physical installation!)

Cost: $0.30/Gbps-hour (data transfer) + port fee
```

---

## 15.3 Transit Gateway — Multi-VPC Connectivity

```
Problem without Transit Gateway:
  VPC-A ↔ VPC-B: 1 peering
  VPC-A ↔ VPC-C: 1 peering
  VPC-B ↔ VPC-C: 1 peering
  = N*(N-1)/2 peering connections (complex, doesn't scale)

Transit Gateway:
  সব VPC → Transit Gateway → Hub-and-spoke
  100s of VPCs এক জায়গায় connect করো

DBA এর জন্য কখন লাগে:
  ① Multiple environments (prod, staging, dev) → shared database VPC
  ② Multi-account setup → central database VPC
  ③ On-premise + multiple AWS VPCs
```

### 🖥️ GUI Method

```
VPC → Transit gateways → Create transit gateway
→ Name: central-tgw
→ ASN: 64512 (default)
→ Default route table: Enable
→ Create transit gateway

VPC → Transit gateway attachments → Create attachment
→ Transit gateway: central-tgw
→ Attachment type: VPC
→ VPC: prod-vpc
→ Subnets: [private subnets]
→ Create

আরো VPCs attach করো (staging, dev)

Route Tables:
→ Each VPC এর route table এ Transit Gateway route add করো
  Destination: 10.0.0.0/8 (অন্য VPCs এর CIDR)
  Target: transit gateway
```

---

## 15.4 Application Load Balancer (ALB) — Database Routing

```
ALB directly database front করে না
কিন্তু DBA এর কাজে কীভাবে লাগে:

1. Read/Write Splitting:
   App → ALB:5432 → 
     Rule: POST/PUT/DELETE → Primary RDS
     Rule: GET → Read Replica

2. Blue/Green deployment traffic shifting:
   ALB: 90% → Blue (old), 10% → Green (new)
   Gradually shift: 50/50, then 100% Green

3. Database-adjacent:
   PgBouncer cluster → ALB → Load balance connections

DBA এর জন্য ALB এর limited use:
   → Direct DB connection: psql, application JDBC → ALB through NLB (not ALB)
   → ALB = HTTP/HTTPS only
   → NLB (Network Load Balancer) = TCP, works for PostgreSQL port 5432
```

### 🖥️ GUI — NLB for PostgreSQL

```
EC2 → Load Balancers → Create load balancer
→ Network Load Balancer (NLB)

Basic configuration:
→ Name: pg-nlb
→ Scheme: Internal (private)
→ VPC: prod-vpc
→ Subnets: private subnets

Listeners and routing:
→ Protocol: TCP
→ Port: 5432
→ Target group: Create new
   → Target type: Instance (Bastion/PgBouncer instances)
   → Protocol: TCP
   → Port: 5432
   → Health check: TCP port 5432

→ Create load balancer

Application: NLB endpoint দিয়ে connect করো
pg-nlb.xxxx.ap-southeast-1.elb.amazonaws.com:5432
```

### 💻 CLI Method

```bash
# NLB for PostgreSQL connection load balancing
aws elbv2 create-load-balancer \
    --name pg-nlb \
    --type network \
    --scheme internal \
    --subnets $DB_1A $DB_1B

NLB_ARN=$(aws elbv2 describe-load-balancers \
    --names pg-nlb \
    --query "LoadBalancers[0].LoadBalancerArn" --output text)

# Target group
aws elbv2 create-target-group \
    --name pg-targets \
    --protocol TCP \
    --port 5432 \
    --vpc-id $VPC_ID \
    --health-check-protocol TCP \
    --target-type instance

TG_ARN=$(aws elbv2 describe-target-groups \
    --names pg-targets \
    --query "TargetGroups[0].TargetGroupArn" --output text)

# Listener তৈরি করো
aws elbv2 create-listener \
    --load-balancer-arn $NLB_ARN \
    --protocol TCP \
    --port 5432 \
    --default-actions Type=forward,TargetGroupArn=$TG_ARN

# Targets register করো (PgBouncer instances)
aws elbv2 register-targets \
    --target-group-arn $TG_ARN \
    --targets Id=$PGBOUNCER_INSTANCE_1 Id=$PGBOUNCER_INSTANCE_2
```

---

## 15.5 AWS Organizations — Multi-account Setup

```
AWS Organizations:
  Multiple AWS accounts → একটা organization এ manage করো

Why multiple accounts for DBA:
  ① Isolation: Production DB → separate account
  ② Security: Blast radius limit করো
  ③ Cost tracking: Per-account billing
  ④ Compliance: Separate audit trail per account

Common setup for DB team:
  Root (Management) Account
  ├── Production Account → RDS prod
  ├── Staging Account → RDS staging
  ├── Development Account → RDS dev
  └── Backup Account → S3 backup, cross-account snapshots

Cross-account RDS snapshot share:
```

```bash
# Production account এ snapshot share করো
aws rds modify-db-snapshot-attribute \
    --db-snapshot-identifier mydb-snapshot \
    --attribute-name restore \
    --values-to-add "arn:aws:iam::BACKUP_ACCOUNT:root"

# Backup account এ restore করো
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier mydb-restored \
    --db-snapshot-identifier "arn:aws:rds:ap-southeast-1:PROD_ACCOUNT:snapshot:mydb-snapshot" \
    ...

# AWS RAM (Resource Access Manager) দিয়েও share করা যায়
aws ram create-resource-share \
    --name pg-snapshot-share \
    --resource-arns "arn:aws:rds:..." \
    --principals "arn:aws:iam::BACKUP_ACCOUNT:root"
```
