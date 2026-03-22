# AWS Cloud Native DBA
## Guide 3 — RDS, Aurora, এবং Managed Database Services

> **Prerequisites:** Guide 2 (AWS Fundamentals for DBA) complete করা থাকতে হবে
> **লক্ষ্য:** AWS এ managed PostgreSQL (RDS + Aurora) দিয়ে production-grade database চালানো
> **Environment:** AWS Console + AWS CLI, ap-southeast-1 region

---

## Table of Contents

1. **[RDS vs Aurora vs Self-managed — কোনটা কখন](#1-rds-vs-aurora-vs-self-managed)**
   - 1.1 [তিনটার পার্থক্য এবং tradeoffs](#11-তিনটার-পার্থক্য)
   - 1.2 [Decision Guide — তোমার workload এর জন্য কোনটা](#12-decision-guide)

2. **[Amazon RDS for PostgreSQL — Deep Dive](#2-amazon-rds-for-postgresql)**
   - 2.1 [RDS Architecture — On-premise এর সাথে পার্থক্য](#21-rds-architecture)
   - 2.2 [Parameter Groups — postgresql.conf এর RDS equivalent](#22-parameter-groups)
   - 2.3 [Option Groups এবং Extensions](#23-option-groups-এবং-extensions)
   - 2.4 [Storage Types এবং Sizing](#24-storage-types-এবং-sizing)
   - 2.5 [Multi-AZ Deployment — HA Setup](#25-multi-az-deployment)
   - 2.6 [Read Replicas — Read Scaling](#26-read-replicas)
   - 2.7 [RDS Backup — Automated এবং Manual](#27-rds-backup)
   - 2.8 [Point-in-Time Recovery (PITR) on RDS](#28-point-in-time-recovery-on-rds)
   - 2.9 [RDS Upgrade — Minor এবং Major Version](#29-rds-upgrade)
   - 2.10 [RDS Maintenance Window](#210-rds-maintenance-window)
   - 2.11 [Hands-on: Production-ready RDS Setup](#211-hands-on-production-ready-rds-setup)

3. **[Amazon Aurora PostgreSQL — Deep Dive](#3-amazon-aurora-postgresql)**
   - 3.1 [Aurora Architecture — কীভাবে আলাদা](#31-aurora-architecture)
   - 3.2 [Aurora Cluster — Writer এবং Reader Endpoints](#32-aurora-cluster)
   - 3.3 [Aurora Storage — Distributed এবং Auto-scaling](#33-aurora-storage)
   - 3.4 [Aurora Replicas — Fast Failover](#34-aurora-replicas)
   - 3.5 [Aurora Serverless v2 — Variable Workload](#35-aurora-serverless-v2)
   - 3.6 [Aurora Global Database — Multi-region](#36-aurora-global-database)
   - 3.7 [Aurora Backup এবং Backtrack](#37-aurora-backup-এবং-backtrack)
   - 3.8 [Hands-on: Aurora Cluster Setup](#38-hands-on-aurora-cluster-setup)

4. **[RDS Proxy — Connection Pooling Managed](#4-rds-proxy)**
   - 4.1 [RDS Proxy কী এবং কেন](#41-rds-proxy-কী-এবং-কেন)
   - 4.2 [RDS Proxy Setup করো](#42-rds-proxy-setup-করো)
   - 4.3 [Hands-on: RDS Proxy Configure করো](#43-hands-on-rds-proxy-configure-করো)

5. **[Performance Insights — Query Analytics](#5-performance-insights)**
   - 5.1 [Performance Insights কী](#51-performance-insights-কী)
   - 5.2 [Dashboard ব্যবহার করো](#52-dashboard-ব্যবহার-করো)
   - 5.3 [Top SQL এবং Wait Events](#53-top-sql-এবং-wait-events)
   - 5.4 [Hands-on: Slow Query Identify করো](#54-hands-on-slow-query-identify-করো)

6. **[Enhanced Monitoring এবং CloudWatch](#6-enhanced-monitoring-এবং-cloudwatch)**
   - 6.1 [Basic vs Enhanced Monitoring](#61-basic-vs-enhanced-monitoring)
   - 6.2 [Key RDS Metrics এবং Thresholds](#62-key-rds-metrics-এবং-thresholds)
   - 6.3 [CloudWatch Logs — Database Logs](#63-cloudwatch-logs--database-logs)
   - 6.4 [Hands-on: Complete Monitoring Setup](#64-hands-on-complete-monitoring-setup)

7. **[RDS Security — Best Practices](#7-rds-security)**
   - 7.1 [Encryption at Rest এবং in Transit](#71-encryption-at-rest-এবং-in-transit)
   - 7.2 [IAM Database Authentication](#72-iam-database-authentication)
   - 7.3 [Secrets Manager + RDS Integration](#73-secrets-manager--rds-integration)
   - 7.4 [Security Groups এবং Network Isolation](#74-security-groups-এবং-network-isolation)
   - 7.5 [Hands-on: Secure RDS Setup](#75-hands-on-secure-rds-setup)

8. **[Database Migration Service (DMS)](#8-database-migration-service-dms)**
   - 8.1 [DMS কী এবং কখন ব্যবহার করবো](#81-dms-কী-এবং-কখন-ব্যবহার-করবো)
   - 8.2 [On-premise PostgreSQL → RDS Migration](#82-on-premise-postgresql--rds-migration)
   - 8.3 [MySQL → Aurora PostgreSQL Migration](#83-mysql--aurora-postgresql-migration)
   - 8.4 [Hands-on: DMS Replication Task](#84-hands-on-dms-replication-task)

9. **[Cost Optimization — RDS এবং Aurora](#9-cost-optimization)**
   - 9.1 [Reserved Instances — সবচেয়ে বড় saving](#91-reserved-instances)
   - 9.2 [Right-sizing — Correct Instance Type](#92-right-sizing)
   - 9.3 [Storage Optimization](#93-storage-optimization)
   - 9.4 [Aurora Serverless vs Provisioned](#94-aurora-serverless-vs-provisioned)

10. **[Troubleshooting — RDS এবং Aurora](#10-troubleshooting)**
    - 10.1 [Connection Issues](#101-connection-issues)
    - 10.2 [Performance Issues](#102-performance-issues)
    - 10.3 [Storage Issues](#103-storage-issues)
    - 10.4 [Failover Issues](#104-failover-issues)
    - 10.5 [Common Error Messages](#105-common-error-messages)

11. **[Production Runbook — Day-to-Day Operations](#11-production-runbook)**
    - 11.1 [Daily Checks](#111-daily-checks)
    - 11.2 [Weekly Tasks](#112-weekly-tasks)
    - 11.3 [Monthly Tasks](#113-monthly-tasks)
    - 11.4 [Emergency Procedures](#114-emergency-procedures)

12. **[Free Tier — Hands-on Labs (GUI + CLI)](#12-free-tier--hands-on-labs)**
    - 12.0 [Free Tier Cost Guide এবং সাবধানতা](#120-free-tier-cost-guide-এবং-সাবধানতা)
    - 12.1 [Lab 1 — RDS PostgreSQL তৈরি এবং Connect করো](#121-lab-1--rds-postgresql-তৈরি-এবং-connect-করো)
    - 12.2 [Lab 2 — Parameter Group Tune করো](#122-lab-2--parameter-group-tune-করো)
    - 12.3 [Lab 3 — Read Replica তৈরি এবং Test করো](#123-lab-3--read-replica-তৈরি-এবং-test-করো)
    - 12.4 [Lab 4 — Snapshot, PITR, এবং Restore](#124-lab-4--snapshot-pitr-এবং-restore)
    - 12.5 [Lab 5 — Performance Insights দিয়ে Slow Query খোঁজো](#125-lab-5--performance-insights-দিয়ে-slow-query-খোঁজো)
    - 12.6 [Lab 6 — CloudWatch Alarms এবং Dashboard](#126-lab-6--cloudwatch-alarms-এবং-dashboard)
    - 12.7 [Lab 7 — RDS Security Hardening](#127-lab-7--rds-security-hardening)
    - 12.8 [Lab 8 — Aurora Cluster (Serverless v2)](#128-lab-8--aurora-cluster-serverless-v2)
    - 12.9 [Lab 9 — RDS Upgrade (Minor এবং Major)](#129-lab-9--rds-upgrade-minor-এবং-major)
    - 12.10 [Lab Cleanup — সব Resources Delete করো](#1210-lab-cleanup--সব-resources-delete-করো)

13. **[Blue/Green Deployments — Zero-Downtime Upgrade](#13-bluegreen-deployments)**
    - 13.1 [Blue/Green কী এবং কেন](#131-bluegreen-কী-এবং-কেন)
    - 13.2 [Blue/Green Setup এবং Switchover](#132-bluegreen-setup-এবং-switchover)

14. **[RDS Event Subscriptions — Automatic Notifications](#14-rds-event-subscriptions)**
    - 14.1 [Event Subscriptions কী](#141-event-subscriptions-কী)
    - 14.2 [Event Subscriptions Setup](#142-event-subscriptions-setup)

15. **[VPC Flow Logs এবং CloudTrail — Audit](#15-vpc-flow-logs-এবং-cloudtrail--audit)**
    - 15.1 [VPC Flow Logs — Network Traffic Audit](#151-vpc-flow-logs--network-traffic-audit)
    - 15.2 [CloudTrail — AWS API Call Audit](#152-cloudtrail--aws-api-call-audit)
    - 15.3 [VPC Endpoints — S3 Access ছাড়া NAT Gateway](#153-vpc-endpoints--s3-access-ছাড়া-nat-gateway)

16. **[AWS Backup — Centralized Backup Management](#16-aws-backup--centralized-backup-management)**
    - 16.1 [AWS Backup কী](#161-aws-backup-কী)
    - 16.2 [Backup Plan তৈরি করো](#162-backup-plan-তৈরি-করো)

17. **[Final Summary — Complete Guide Overview](#17-final-summary--complete-guide-overview)**

18. **[RDS Custom — OS Access সহ Managed RDS](#18-rds-custom--os-access-সহ-managed-rds)**
    - 18.1 [RDS Custom কী](#181-rds-custom-কী)
    - 18.2 [RDS Custom Setup এবং Access](#182-rds-custom-setup-এবং-access)

19. **[Trusted Language Extensions (TLE)](#19-trusted-language-extensions-tle)**
    - 19.1 [TLE কী](#191-tle-কী)
    - 19.2 [TLE Setup এবং Use](#192-tle-setup-এবং-use)

20. **[Zero-ETL — Aurora to Redshift Direct](#20-zero-etl--aurora-to-redshift-direct-integration)**
    - 20.1 [Zero-ETL কী](#201-zero-etl-কী)
    - 20.2 [Zero-ETL Setup](#202-zero-etl-setup)

21. **[Aurora PostgreSQL Limitless — Horizontal Sharding](#21-aurora-postgresql-limitless--horizontal-sharding)**
    - 21.1 [Aurora Limitless কী](#211-aurora-limitless-কী)
    - 21.2 [Limitless Setup Concept](#212-limitless-setup-concept)

22. **[pgvector on RDS/Aurora](#22-pgvector-on-rdsaurora)**
    - 22.1 [pgvector on Managed Services](#221-pgvector-on-managed-services)

23. **[Adjacent Services — Redshift, ElastiCache, DynamoDB](#23-adjacent-services--redshift-elasticache-dynamodb)**
    - 23.1 [Amazon Redshift — Data Warehouse](#231-amazon-redshift--data-warehouse)
    - 23.2 [Amazon ElastiCache — In-memory Caching](#232-amazon-elasticache--in-memory-caching)
    - 23.3 [Amazon DynamoDB — NoSQL](#233-amazon-dynamodb--nosql-when-postgresql-isnt-enough)
    - 23.4 [When to Use What — Decision Guide](#234-when-to-use-what--complete-decision-guide)
    - 23.5 [Monitoring — সব Services একসাথে](#235-monitoring--সব-services-একসাথে)
    - 23.6 [Cost Comparison](#236-cost-comparison--service-selection-এ-cost-factor)

---

# 1. RDS vs Aurora vs Self-managed

## 1.1 তিনটার পার্থক্য

```
┌─────────────────────────────────────────────────────────────────────┐
│                    তিনটা Option এর Comparison                       │
├──────────────────┬──────────────────┬───────────────────────────────┤
│ Self-managed EC2 │  Amazon RDS      │  Amazon Aurora PostgreSQL     │
├──────────────────┼──────────────────┼───────────────────────────────┤
│ OS access আছে   │ OS access নেই    │ OS access নেই                 │
│ postgresql.conf  │ Parameter Groups │ Parameter Groups              │
│ Manual backup    │ Automated backup │ Automated backup              │
│ Manual HA        │ Multi-AZ managed │ Built-in HA, faster failover  │
│ Manual upgrade   │ Managed upgrade  │ Managed upgrade               │
│ pg_hba.conf      │ Security Groups  │ Security Groups               │
│ Custom extensions│ Limited extensions│ Limited extensions           │
│ Cheapest         │ Middle cost      │ More expensive (worth it)     │
│ Most control     │ Less control     │ Least control                 │
│ Most ops burden  │ Less ops burden  │ Least ops burden              │
└──────────────────┴──────────────────┴───────────────────────────────┘
```

**Self-managed EC2:**
```
Pros:
  ✅ Full control — pg_hba.conf, postgresql.conf, extensions সব
  ✅ Latest PostgreSQL version সাথে সাথে
  ✅ Custom kernel settings, huge pages, OS tuning
  ✅ সস্তা (EC2 r7g.xlarge vs db.r7g.xlarge — ~30% কম)
  ✅ pg_repack, custom pg_stat_* setups সব চলবে

Cons:
  ❌ সব কিছু নিজে করতে হবে: backup, HA, upgrade, patching
  ❌ OS patching, security updates
  ❌ DBA সময় বেশি লাগে operations এ
  ❌ হঠাৎ disk full, OOM — নিজে handle করতে হবে
```

**Amazon RDS:**
```
Pros:
  ✅ Automated backup, PITR built-in
  ✅ Multi-AZ — automatic failover (~1-2 min)
  ✅ Minor version auto-upgrade
  ✅ Enhanced monitoring, Performance Insights
  ✅ AWS manages OS, hardware
  ✅ Read Replicas easy setup
  ✅ Free storage for automated backups (up to DB size)

Cons:
  ❌ OS access নেই — kernel tuning সম্ভব না
  ❌ pg_hba.conf directly edit করা যায় না
  ❌ কিছু extensions নেই (pg_repack CLI নেই)
  ❌ Aurora এর চেয়ে failover slow (~1-2 min vs ~30s)
  ❌ বড় extension list নেই
```

**Amazon Aurora PostgreSQL:**
```
Pros:
  ✅ 6-way replication across 3 AZs — storage level
  ✅ Failover ~30 seconds (RDS এর চেয়ে 4x fast)
  ✅ Up to 15 read replicas (RDS: 5)
  ✅ Aurora Serverless v2 — auto-scale CPU/RAM
  ✅ Global Database — multi-region <1s latency
  ✅ Backtrack — seconds এ যেকোনো সময়ে rewind
  ✅ Clone — production এর copy instant (শুধু changed blocks নেয়)
  ✅ Storage auto-grows (max 128TB)

Cons:
  ❌ Most expensive (~20% বেশি than RDS)
  ❌ Aurora specific features — vendor lock-in কিছুটা
  ❌ Small instance types কম available
  ❌ PostgreSQL latest version delay হয় (Aurora লাগে সময়)
```

---

## 1.2 Decision Guide

```
কোনটা choose করবো?

তোমার team ছোট (<5 people), DBA full-time নয়:
  → Aurora PostgreSQL
  Reason: Least operational overhead, most managed

তোমার team বড়, DBA আছে, cost important:
  → RDS PostgreSQL
  Reason: Good balance of managed + control + cost

তোমার compliance requirement আছে, custom config দরকার:
  → Self-managed on EC2
  Reason: Full control needed

Variable traffic (peak/off-peak very different):
  → Aurora Serverless v2
  Reason: Pay for what you use

Multi-region, global users:
  → Aurora Global Database
  Reason: Low latency reads globally

Large team, existing on-premise PG investment:
  → RDS + pgBackRest on S3
  Reason: Familiar tooling, good control

Development/Testing:
  → RDS (Free Tier) বা self-managed EC2 t3.micro
  Reason: Cost minimum

Simple rule:
  Startup/Small team         → Aurora Serverless v2
  Medium business            → RDS Multi-AZ
  Large enterprise           → Aurora (provisioned)
  Need full OS control       → Self-managed EC2
  Uncertain traffic          → Aurora Serverless v2
```

---

# 2. Amazon RDS for PostgreSQL — Deep Dive

## 2.1 RDS Architecture — On-premise এর সাথে পার্থক্য

```
On-premise PostgreSQL:
  তোমার server → তুমি OS control করো
  → postgresql.conf directly edit
  → pg_hba.conf directly edit
  → OS: kernel parameters, huge pages, I/O scheduler
  → Backup: তুমি configure করো
  → HA: তুমি Patroni setup করো

Amazon RDS:
  AWS managed server → তুমি DB layer control করো
  → Parameter Groups (postgresql.conf equivalent)
  → Security Groups (network access control)
  → OS: AWS manages (তুমি access পাবে না)
  → Backup: AWS automatically নেয়
  → HA: Multi-AZ enable করলে AWS manages

RDS এ কী পাবে না:
  ❌ SSH/OS access
  ❌ /var/lib/pgsql directly access
  ❌ pg_hba.conf edit (Security Groups দিয়ে করো)
  ❌ Custom OS-level tuning
  ❌ superuser (rds_superuser role পাবে, কিন্তু true superuser নয়)

RDS এ কী বেশি পাবে:
  ✅ Automated daily backup + 35 days retention
  ✅ Point-in-time recovery (5-minute granularity)
  ✅ One-click Multi-AZ
  ✅ Performance Insights (built-in query analytics)
  ✅ Enhanced Monitoring (OS metrics without access)
  ✅ Automatic minor version upgrades
  ✅ Event notifications (SNS)
```

---

## 2.2 Parameter Groups — postgresql.conf এর RDS Equivalent

```
On-premise:                     RDS:
postgresql.conf                 Parameter Group
  shared_buffers = 4GB    →       shared_buffers = {DBInstanceClassMemory/4}
  work_mem = 64MB          →       work_mem = 65536  (bytes এ)
  max_connections = 200    →       max_connections = LEAST({DBInstanceClassMemory/9531392},5000)
```

**Parameter Group Types:**
```
Static Parameter:
  Change করতে Reboot দরকার
  Example: shared_buffers, max_connections
  Apply method: pending-reboot

Dynamic Parameter:
  Running এ apply হয় (no reboot)
  Example: work_mem, log_min_duration_statement
  Apply method: immediate
```

### Hands-on: Parameter Group তৈরি এবং Tune করো

**🖥️ GUI:**
```
RDS → Parameter groups → Create parameter group

Parameter group family: postgres16
Type: DB Parameter Group
Group name: prod-pg16-optimized
Description: Production PostgreSQL 16 Optimized

→ Create

Parameter Group এ click করো → Edit parameters:

Search "work_mem":
  → work_mem: 65536  (64MB in KB নয়, bytes এ)
  Apply: immediate

Search "log_min_duration_statement":
  → Value: 1000  (1 second)
  Apply: immediate

Search "shared_preload_libraries":
  → Value: pg_stat_statements,pgaudit
  Apply: pending-reboot

Search "log_connections":
  → Value: 1
  Apply: immediate

Search "log_lock_waits":
  → Value: 1
  Apply: immediate

→ Save changes
```

**💻 CLI:**
```bash
# Parameter Group তৈরি
aws rds create-db-parameter-group \
    --db-parameter-group-name prod-pg16-optimized \
    --db-parameter-group-family postgres16 \
    --description "Production PostgreSQL 16 Optimized"

# Parameters tune করো
aws rds modify-db-parameter-group \
    --db-parameter-group-name prod-pg16-optimized \
    --parameters \
        "ParameterName=work_mem,ParameterValue=65536,ApplyMethod=immediate" \
        "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate" \
        "ParameterName=shared_preload_libraries,ParameterValue=pg_stat_statements,ApplyMethod=pending-reboot" \
        "ParameterName=log_connections,ParameterValue=1,ApplyMethod=immediate" \
        "ParameterName=log_lock_waits,ParameterValue=1,ApplyMethod=immediate" \
        "ParameterName=log_temp_files,ParameterValue=0,ApplyMethod=immediate" \
        "ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=pending-reboot"

# Parameters দেখো
aws rds describe-db-parameters \
    --db-parameter-group-name prod-pg16-optimized \
    --query 'Parameters[?Source==`user`].[ParameterName,ParameterValue,ApplyMethod]' \
    --output table
```

**RDS Important Parameters:**

| Parameter | Recommended Value | মানে |
|---|---|---|
| `shared_buffers` | `{DBInstanceClassMemory/32768}` | Auto (25% RAM) |
| `work_mem` | `65536` (64MB) | Sort/Hash memory |
| `maintenance_work_mem` | `524288` (512MB) | VACUUM/Index memory |
| `max_connections` | `LEAST({DBInstanceClassMemory/9531392},5000)` | Auto-calculated |
| `log_min_duration_statement` | `1000` | 1s slow query log |
| `shared_preload_libraries` | `pg_stat_statements` | Query tracking |
| `rds.force_ssl` | `1` | SSL required |
| `log_connections` | `1` | Connection logging |
| `log_lock_waits` | `1` | Lock wait logging |
| `idle_in_transaction_session_timeout` | `600000` | 10 min (ms) |
| `statement_timeout` | `0` | (app level এ set করো) |

---

## 2.3 Option Groups এবং Extensions

```
RDS PostgreSQL এ available extensions দেখো:
```

```sql
-- RDS তে connect করে:
SELECT name, default_version, installed_version, comment
FROM pg_available_extensions
ORDER BY name;

-- Common extensions যেগুলো RDS এ আছে:
-- pg_stat_statements  ✅
-- pgcrypto            ✅
-- uuid-ossp           ✅
-- pg_trgm             ✅
-- PostGIS             ✅
-- pg_partman          ✅ (RDS 14+)
-- pgaudit             ✅
-- hstore              ✅
-- ltree               ✅

-- Install করো
CREATE EXTENSION pg_stat_statements;
CREATE EXTENSION pgcrypto;
CREATE EXTENSION postgis;
```

```
RDS এ নেই বা limited:
  ❌ pg_repack (CLI tool, RDS এ চলে না)
  ❌ Custom C extensions
  ❌ timescaledb (Aurora তে আছে কিন্তু RDS তে নেই)
  ❌ pg_cron (RDS PostgreSQL 13+ এ আছে, enable করতে হয়)

pg_cron on RDS enable করো:
  Parameter Group: rds.enable_pg_cron = 1 (pending-reboot)
  তারপর: CREATE EXTENSION pg_cron;
```

---

## 2.4 Storage Types এবং Sizing

```
RDS Storage Types:

gp2 (General Purpose SSD v2) — Legacy:
  Baseline: 3 IOPS/GB (100-16,000 IOPS)
  Burst: Up to 3,000 IOPS (< 1TB volumes)
  Max throughput: 250 MB/s
  → নতুন এ ব্যবহার করো না, gp3 ব্যবহার করো

gp3 (General Purpose SSD v3) — Recommended:
  Baseline: 3,000 IOPS (regardless of size)
  Max: 16,000 IOPS (extra charge)
  Max throughput: 1,000 MB/s
  20% cheaper than gp2
  → Most workloads এর জন্য ✅

io1 (Provisioned IOPS) — High IOPS:
  1,000-64,000 IOPS
  Consistent, predictable IOPS
  → High-transaction, latency-sensitive

io2 Block Express — Highest Performance:
  Up to 256,000 IOPS
  Sub-millisecond latency
  → Mission-critical, highest performance
  → Expensive

magnetic — Legacy:
  → এটা ব্যবহার করো না

Storage Auto Scaling:
  Enable করলে threshold cross হলে automatically বাড়ে
  Production এ enable করো + maximum threshold set করো
  কমে না (once grown, stays grown)
```

**Storage Sizing Guide:**
```
Initial sizing:
  Data size = current DB size × 1.5 (growth buffer)
  + 20% for temporary files, indexes
  Minimum: 100GB (gp3 এর জন্য sufficient IOPS)

IOPS calculation:
  gp3 default 3,000 IOPS = adequate for most
  Heavy write workload: increase to 6,000-12,000
  Very high IOPS: io2

Throughput:
  gp3 default 125 MB/s → increase to 500-1000 MB/s if needed
  Analytics workload এ বেশি throughput দরকার হয়
```

---

## 2.5 Multi-AZ Deployment — HA Setup

```
Multi-AZ কীভাবে কাজ করে:

Primary (AZ-a) ──── Synchronous Replication ──── Standby (AZ-b)
      │                                                │
      │ Writes go here                                │ DNS failover এ
      │ Reads go here (single endpoint)               │ এটা Primary হয়
      │                                               │
      └──── RDS Endpoint ────────────────────────────┘
            (DNS name stays same, IP changes on failover)

Failover কখন হয়:
  ① Primary DB instance failure
  ② AZ failure
  ③ DB instance type change
  ④ OS patching
  ⑤ Manual failover (reboot with failover)

Failover time: ~60-120 seconds
  DNS TTL: 5 seconds (application reconnect fast)
  Application: connection retry logic থাকতে হবে

Multi-AZ vs Read Replica:
  Multi-AZ Standby:
    Synchronous replication
    Not readable (শুধু failover target)
    Same region, different AZ
    HA purpose

  Read Replica:
    Asynchronous replication
    Readable (read scaling)
    Same or different region
    Performance scaling purpose
```

**🖥️ GUI — Multi-AZ Enable করো:**
```
RDS → Databases → তোমার DB → Modify

Availability & durability:
→ Multi-AZ deployment: "Create a standby instance" select করো
→ Apply immediately বা During next maintenance window

→ Continue → Modify DB instance

Status: "Modifying" → "Available" (~10 minutes)
```

**💻 CLI:**
```bash
# Multi-AZ enable করো
aws rds modify-db-instance \
    --db-instance-identifier mydb-prod \
    --multi-az \
    --apply-immediately

# Multi-AZ status দেখো
aws rds describe-db-instances \
    --db-instance-identifier mydb-prod \
    --query 'DBInstances[0].[MultiAZ,AvailabilityZone,SecondaryAvailabilityZone]'

# Manual failover test করো (planned)
aws rds reboot-db-instance \
    --db-instance-identifier mydb-prod \
    --force-failover
# ~60-120 seconds এ Primary AZ change হবে
```

---

## 2.6 Read Replicas — Read Scaling

```
Read Replica architecture:

           Primary (write + read)
               │
               │ Async replication
               ▼
    ┌──────────────────────────┐
    │  Read Replica 1 (AZ-b)  │ ← read traffic
    │  Read Replica 2 (AZ-c)  │ ← reporting queries
    │  Read Replica 3 (Mumbai)│ ← cross-region DR
    └──────────────────────────┘

Use cases:
  ① Read-heavy workload: app এর read queries replica তে
  ② Analytics/Reporting: heavy queries primary এ না গিয়ে replica তে
  ③ DR: cross-region replica → promote করলে new primary হয়
  ④ Testing: production data দিয়ে test (no impact on prod)
```

**🖥️ GUI — Read Replica তৈরি করো:**
```
RDS → Databases → তোমার DB select করো
→ Actions → "Create read replica"

Settings:
→ DB instance identifier: mydb-replica-1
→ DB instance class: db.r7g.large  (Primary এর চেয়ে ছোট ok)
→ Storage: gp3, auto-scaling enable
→ Multi-AZ: No (replica নিজে HA না হলে ok)
→ Destination region: same region রাখো
→ Public access: No
→ VPC: same VPC

→ "Create read replica" click করো
```

**💻 CLI:**
```bash
# Read Replica তৈরি
aws rds create-db-instance-read-replica \
    --db-instance-identifier mydb-replica-1 \
    --source-db-instance-identifier mydb-prod \
    --db-instance-class db.r7g.large \
    --availability-zone ap-southeast-1b \
    --no-publicly-accessible \
    --storage-type gp3

# Replica lag monitor করো
aws cloudwatch get-metric-statistics \
    --namespace AWS/RDS \
    --metric-name ReplicaLag \
    --dimensions Name=DBInstanceIdentifier,Value=mydb-replica-1 \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 60 --statistics Average \
    --query 'Datapoints[*].[Timestamp,Average]' \
    --output table

# Replica promote করো (DR scenario)
aws rds promote-read-replica \
    --db-instance-identifier mydb-replica-1
# Replica → Standalone DB হয়ে যাবে
```

```sql
-- Replica এ lag দেখো
SELECT now() - pg_last_xact_replay_timestamp() AS lag;

-- Replica তে write করার চেষ্টা → error
INSERT INTO test VALUES (1);
-- ERROR: cannot execute INSERT in a read-only transaction
```

---

## 2.7 RDS Backup — Automated এবং Manual

```
Automated Backup:
  Daily full snapshot + transaction logs (5-minute interval)
  Retention: 0-35 days (default 7)
  Backup window: specify করো (low-traffic time)
  Storage: Free up to DB size, তারপর charge

Manual Snapshot:
  তুমি manually নাও
  Never expires (তুমি delete না করলে)
  Cross-region copy করা যায়
  Share করা যায় (অন্য AWS account এ)

কোনটা কখন:
  Automated: daily regular backup
  Manual: major change এর আগে, migration এর আগে
```

**🖥️ GUI — Backup Window Set করো:**
```
RDS → Databases → DB → Modify

Backup:
→ Backup retention period: 7 days (বা 14, 30)
→ Backup window: 02:00-03:00 UTC (রাত ২-৩টা)
  (low traffic time choose করো)

→ Modify DB instance
```

**🖥️ GUI — Manual Snapshot:**
```
RDS → Databases → DB select করো
→ Actions → "Take snapshot"
→ Snapshot name: mydb-before-migration-20240308
→ "Take snapshot"

Snapshot list দেখো:
RDS → Snapshots → Manual
```

**💻 CLI:**
```bash
# Backup settings modify করো
aws rds modify-db-instance \
    --db-instance-identifier mydb-prod \
    --backup-retention-period 7 \
    --preferred-backup-window "18:00-19:00" \
    --apply-immediately

# Manual snapshot
aws rds create-db-snapshot \
    --db-instance-identifier mydb-prod \
    --db-snapshot-identifier mydb-before-migration-$(date +%Y%m%d)

# Latest automated backup দেখো
aws rds describe-db-snapshots \
    --db-instance-identifier mydb-prod \
    --snapshot-type automated \
    --query 'sort_by(DBSnapshots, &SnapshotCreateTime)[-1].[DBSnapshotIdentifier,SnapshotCreateTime,Status]' \
    --output table

# Cross-region copy
aws rds copy-db-snapshot \
    --source-db-snapshot-identifier arn:aws:rds:ap-southeast-1:123456789012:snapshot:mydb-snap \
    --target-db-snapshot-identifier mydb-snap-mumbai \
    --region ap-south-1
```

---

## 2.8 Point-in-Time Recovery on RDS

**On-premise PITR vs RDS PITR:**
```
On-premise:
  Base backup + WAL archive manually configure করতে হয়
  restore_command, recovery.signal manually
  Complex, error-prone

RDS PITR:
  Automatic — কোনো setup নেই
  Last 5 minutes পর্যন্ত যেকোনো সময়ে restore
  Retention period এর মধ্যে যেকোনো point
  New DB instance তৈরি করে (original intact থাকে)
```

**🖥️ GUI — PITR করো:**
```
RDS → Databases → তোমার DB → Actions
→ "Restore to point in time"

Restore time:
→ "Custom date and time" select করো
→ Date: 2024-03-08
→ Time: 09:59:59 UTC  (disaster এর ঠিক আগে)

Settings:
→ DB instance identifier: mydb-restored-pitr
→ DB instance class: same as original বা smaller
→ VPC, subnet, security group: same configure করো
→ Parameter group: same

→ "Restore to point in time" click করো
⏳ ~15-30 minutes

Restore হলে:
→ Verify করো data correct আছে কিনা
→ Application point করো নতুন endpoint এ
→ পুরনো instance delete করো বা রেখে দাও
```

**💻 CLI:**
```bash
# PITR restore
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier mydb-prod \
    --target-db-instance-identifier mydb-pitr-restore \
    --restore-time 2024-03-08T09:59:59Z \
    --db-instance-class db.r7g.xlarge \
    --db-subnet-group-name prod-db-subnet-group \
    --vpc-security-group-ids sg-xxxxxxxx \
    --no-publicly-accessible \
    --no-multi-az

# Available হওয়ার অপেক্ষা
aws rds wait db-instance-available \
    --db-instance-identifier mydb-pitr-restore

# Endpoint নাও
aws rds describe-db-instances \
    --db-instance-identifier mydb-pitr-restore \
    --query 'DBInstances[0].Endpoint.Address' \
    --output text

# Data verify করো
psql -h RESTORED_ENDPOINT -U postgres -d mydb -c "SELECT COUNT(*) FROM orders;"
```

---

## 2.9 RDS Upgrade — Minor এবং Major Version

**Minor Version Upgrade (e.g., 16.1 → 16.3):**
```
Automatic:
  Parameter: Auto minor version upgrade = Yes
  Maintenance window এ automatically হয়
  ~5-10 minutes downtime (Multi-AZ: ~60s)

Manual:
  RDS → DB → Modify → Engine version
  Apply: immediately বা maintenance window
```

**Major Version Upgrade (e.g., 15 → 16):**
```
Cannot be undone automatically — snapshot নাও আগে!
~30-60 minutes downtime (Multi-AZ এও)
Pre-upgrade checks AWS করে

Steps:
1. Manual snapshot নাও
2. Test upgrade on non-prod first
3. Maintenance window এ apply
4. Post-upgrade: extension compatibility check
```

**🖥️ GUI — Major Upgrade:**
```
RDS → Databases → DB → Modify

Engine:
→ Engine version: PostgreSQL 16.x select করো
→ Apply: During next scheduled maintenance window
  (অথবা Immediately for dev/test)

→ Modify DB instance

⚠️ Before upgrade:
  Snapshots → Take snapshot of current instance
  Check: AWS will show pre-check results
```

**💻 CLI:**
```bash
# Pre-upgrade: snapshot নাও
aws rds create-db-snapshot \
    --db-instance-identifier mydb-prod \
    --db-snapshot-identifier mydb-before-upgrade-to-16

# Major upgrade
aws rds modify-db-instance \
    --db-instance-identifier mydb-prod \
    --engine-version "16.3" \
    --allow-major-version-upgrade \
    --apply-immediately

# Status monitor করো
aws rds describe-db-instances \
    --db-instance-identifier mydb-prod \
    --query 'DBInstances[0].[DBInstanceStatus,EngineVersion,PendingModifiedValues]'

# Upgrade হলে extensions check করো
psql -h ENDPOINT -U postgres -d mydb -c "
SELECT name, default_version, installed_version,
    CASE WHEN default_version = installed_version THEN 'OK'
         ELSE 'UPDATE NEEDED'
    END AS status
FROM pg_available_extensions
WHERE installed_version IS NOT NULL;"
```

---

## 2.10 RDS Maintenance Window

```
Maintenance Window:
  AWS weekly maintenance tasks এখানে হয়:
  - Minor version upgrade (if enabled)
  - OS patching
  - Hardware maintenance

Set করার time:
  Low traffic time choose করো
  Multi-AZ: failover হয়, ~60s downtime
  Single-AZ: ~5-10 minutes downtime

Format: ddd:hh24:mi-ddd:hh24:mi (UTC)
Example: sun:18:00-sun:19:00 (Sunday 6-7 PM UTC)
```

**🖥️ GUI:**
```
RDS → DB → Modify

Maintenance:
→ Auto minor version upgrade: Enable (recommended)
→ Maintenance window: Select window
  → Choose: sun 18:00 - sun 19:00

→ Modify DB instance
```

**💻 CLI:**
```bash
# Maintenance window set করো
aws rds modify-db-instance \
    --db-instance-identifier mydb-prod \
    --preferred-maintenance-window "sun:18:00-sun:19:00" \
    --auto-minor-version-upgrade \
    --apply-immediately

# Pending maintenance দেখো
aws rds describe-pending-maintenance-actions \
    --resource-identifier arn:aws:rds:ap-southeast-1:123456789012:db:mydb-prod

# Immediate maintenance apply করো (emergency patch)
aws rds apply-pending-maintenance-action \
    --resource-identifier arn:aws:rds:ap-southeast-1:123456789012:db:mydb-prod \
    --apply-action system-update \
    --opt-in-type immediate
```

---

## 2.11 Hands-on: Production-ready RDS Setup

**Duration: ~1 hour | Cost: db.t3.micro Free Tier অথবা বন্ধ করলে ~$0**

### 🖥️ GUI Method

**Step 1: Production VPC আছে ধরে নিচ্ছি (Lab 2 থেকে)**

**Step 2: Parameter Group তৈরি করো**
```
RDS → Parameter groups → Create parameter group
→ Family: postgres16
→ Name: prod-pg16
→ Create

Edit parameters:
  log_min_duration_statement = 1000
  log_connections = 1
  log_lock_waits = 1
  shared_preload_libraries = pg_stat_statements
  idle_in_transaction_session_timeout = 600000
→ Save changes
```

**Step 3: RDS Create করো**
```
RDS → Create database

Method: Standard create
Engine: PostgreSQL 16.x
Template: Production (Multi-AZ, যদি charge afford করতে পারো)
         অথবা Free tier (lab এ)

Settings:
→ DB identifier: prod-postgres
→ Master username: postgres
→ Master password: [strong password]

Instance:
→ db.r7g.xlarge (Production) বা db.t3.micro (Lab)
→ Multi-AZ: Yes (Production), No (Lab)

Storage:
→ gp3, 100GB (Production) বা 20GB (Lab)
→ Storage autoscaling: Enable, max 500GB
→ Provisioned IOPS: 3000 (default ok)

Connectivity:
→ VPC: prod-vpc
→ Subnet group: prod-db-subnet-group
→ Public access: No
→ Security group: prod-db-sg
→ Availability zone: ap-southeast-1a
→ Database port: 5432

Database authentication:
→ Password authentication
→ IAM database authentication: Enable (for app connections)

Additional configuration:
→ Initial DB name: proddb
→ Parameter group: prod-pg16
→ Backup retention: 7 days
→ Backup window: 18:00-19:00 UTC
→ Maintenance window: sun:20:00-sun:21:00
→ Auto minor version upgrade: Enable
→ Performance Insights: Enable, 7 days retention
→ Enhanced monitoring: Enable, 60 seconds
→ Log exports: postgresql, upgrade (CloudWatch)
→ Deletion protection: Enable ✅ (important!)

→ Create database
```

**Step 4: Extensions Install করো**
```sql
-- Connect করে:
CREATE EXTENSION pg_stat_statements;
CREATE EXTENSION pgcrypto;
CREATE EXTENSION "uuid-ossp";
CREATE EXTENSION pg_trgm;

-- Verify
SELECT name, installed_version FROM pg_available_extensions
WHERE installed_version IS NOT NULL;
```

---

### 💻 CLI Method

```bash
# Parameter Group
aws rds create-db-parameter-group \
    --db-parameter-group-name prod-pg16 \
    --db-parameter-group-family postgres16 \
    --description "Production PostgreSQL 16"

aws rds modify-db-parameter-group \
    --db-parameter-group-name prod-pg16 \
    --parameters \
        "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate" \
        "ParameterName=log_connections,ParameterValue=1,ApplyMethod=immediate" \
        "ParameterName=log_lock_waits,ParameterValue=1,ApplyMethod=immediate" \
        "ParameterName=shared_preload_libraries,ParameterValue=pg_stat_statements,ApplyMethod=pending-reboot" \
        "ParameterName=idle_in_transaction_session_timeout,ParameterValue=600000,ApplyMethod=immediate"

# RDS Instance Create
aws rds create-db-instance \
    --db-instance-identifier prod-postgres \
    --db-instance-class db.r7g.xlarge \
    --engine postgres \
    --engine-version "16.3" \
    --master-username postgres \
    --master-user-password 'ProdPass@2024!' \
    --allocated-storage 100 \
    --storage-type gp3 \
    --storage-encrypted \
    --kms-key-id alias/aws/rds \
    --db-subnet-group-name prod-db-subnet-group \
    --vpc-security-group-ids sg-xxxxxxxxx \
    --db-parameter-group-name prod-pg16 \
    --multi-az \
    --no-publicly-accessible \
    --backup-retention-period 7 \
    --preferred-backup-window "18:00-19:00" \
    --preferred-maintenance-window "sun:20:00-sun:21:00" \
    --auto-minor-version-upgrade \
    --enable-performance-insights \
    --performance-insights-retention-period 7 \
    --monitoring-interval 60 \
    --monitoring-role-arn arn:aws:iam::ACCOUNT:role/rds-monitoring-role \
    --enable-cloudwatch-logs-exports postgresql upgrade \
    --enable-iam-database-authentication \
    --db-name proddb \
    --deletion-protection \
    --tags Key=Environment,Value=production Key=Team,Value=dba

echo "Creating RDS instance (~15 min)..."
aws rds wait db-instance-available --db-instance-identifier prod-postgres
echo "RDS ready!"

ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier prod-postgres \
    --query 'DBInstances[0].Endpoint.Address' --output text)
echo "Endpoint: $ENDPOINT"
```

---

# 3. Amazon Aurora PostgreSQL — Deep Dive

## 3.1 Aurora Architecture — কীভাবে আলাদা

```
RDS Multi-AZ:
  Primary ──sync replication──► Standby
  (block-level replication, one standby)
  Failover: ~60-120 seconds

Aurora:
  Primary ──────────────────────────────────────────────────────┐
      │                                                         │
      │  6 copies of data across 3 AZs (storage layer)        │
      │                                                         │
  ┌───▼───────────────────────────────────────────────────────┐ │
  │  Aurora Storage (Distributed, Auto-scaling)               │ │
  │  AZ-a: copy1, copy2 ─┐                                   │ │
  │  AZ-b: copy3, copy4 ─┤── Quorum writes (4 of 6)         │ │
  │  AZ-c: copy5, copy6 ─┘   Reads (3 of 6)                 │ │
  └───────────────────────────────────────────────────────────┘ │
      │                                                         │
  Aurora Replicas (up to 15) ◄─────── reads from storage ──────┘
  (no replication lag — all share same storage!)

Why Aurora is faster failover:
  Storage already replicated → no data to copy
  New primary just needs to be promoted
  Failover: ~30 seconds (vs ~60-120s RDS)

Aurora Storage:
  Starts: 10GB
  Auto-grows: 10GB increments
  Maximum: 128TB
  No pre-provisioning needed
```

---

## 3.2 Aurora Cluster — Writer এবং Reader Endpoints

```
Aurora Cluster এর Endpoints:

1. Cluster Endpoint (Writer):
   mydb.cluster-xxxx.ap-southeast-1.rds.amazonaws.com
   → সবসময় current Writer instance কে point করে
   → Writes এই endpoint এ
   → Failover হলে automatically new writer এ point করে

2. Reader Endpoint:
   mydb.cluster-ro-xxxx.ap-southeast-1.rds.amazonaws.com
   → সব Reader instances এ load balance করে
   → Reads এই endpoint এ (scale out)

3. Instance Endpoint:
   mydb-instance-1.xxxx.ap-southeast-1.rds.amazonaws.com
   → Specific instance এ direct
   → Debugging এ useful

Application setup:
  Write connection: cluster endpoint
  Read connection:  reader endpoint
  (pgBouncer বা RDS Proxy দিয়ে এই routing manage করো)
```

---

## 3.3 Aurora Storage — Distributed এবং Auto-scaling

```
Aurora Storage Model:
  Traditional DB: Storage on instance → limit by instance size
  Aurora: Storage layer separate from compute
    → Storage auto-grows (no pre-provisioning)
    → Storage billed per GB-month ($0.10/GB/month)
    → Compute (instance) আলাদা charge

Aurora Storage এর সুবিধা:
  ① Clone করা সহজ (just metadata copy — same storage)
  ② Restore fast (no data copy for snapshot restore)
  ③ Backtrack (storage level rewind — no restore needed)

Cost consideration:
  Storage: $0.10/GB-month
  100GB DB = $10/month storage
  RDS gp3: $0.115/GB-month (similar)
  কিন্তু Aurora এ free backup storage নেই beyond 1x DB size
```

---

## 3.4 Aurora Replicas — Fast Failover

```
Aurora Replica vs RDS Read Replica:

RDS Read Replica:
  Async replication (WAL streaming)
  Lag: seconds to minutes
  Failover: manual promotion (~minutes)
  Max: 5 replicas

Aurora Replica:
  Same storage (no replication at all!)
  Lag: ~10ms (just applying WAL records)
  Failover: automatic, ~30 seconds
  Max: 15 replicas

Failover Priority:
  0 = highest priority (failover first)
  15 = lowest priority
  Multiple same priority = random selection
```

**🖥️ GUI — Aurora Replica তৈরি করো:**
```
RDS → Databases → তোমার Aurora cluster → Actions
→ "Add reader"

Settings:
→ DB instance identifier: mydb-reader-1
→ DB instance class: db.r7g.large
→ Availability zone: ap-southeast-1b
→ Failover priority: 1 (lower number = higher priority)
→ Performance Insights: Enable

→ "Add reader" click করো
```

**💻 CLI:**
```bash
# Aurora Reader যোগ করো
aws rds create-db-instance \
    --db-instance-identifier mydb-reader-1 \
    --db-cluster-identifier mydb-aurora-cluster \
    --db-instance-class db.r7g.large \
    --engine aurora-postgresql \
    --availability-zone ap-southeast-1b \
    --no-auto-minor-version-upgrade \
    --promotion-tier 1

# Failover test (planned)
aws rds failover-db-cluster \
    --db-cluster-identifier mydb-aurora-cluster \
    --target-db-instance-identifier mydb-reader-1
# Reader → Writer হয়ে যাবে
```

---

## 3.5 Aurora Serverless v2 — Variable Workload

```
Aurora Serverless v2:
  CPU/RAM automatically scales based on load
  Unit: Aurora Capacity Units (ACU)
  1 ACU = 2GB RAM (approx)
  Range: 0.5 ACU to 128 ACU (configure করো)

Cost:
  $0.12/ACU-hour (ap-southeast-1)
  Min 0.5 ACU: $0.06/hour = ~$43/month
  vs db.t3.micro: ~$0.02/hour = ~$15/month
  কিন্তু serverless auto-scales → high peak তে sufficient

কখন ব্যবহার করবো:
  ✅ Variable/unpredictable traffic
  ✅ Development databases (idle when not used)
  ✅ Infrequent, intermittent use
  ✅ New application (don't know expected load)

কখন না:
  ❌ Steady, predictable high load → provisioned cheaper
  ❌ Very latency-sensitive (scale-up takes ~1s)
```

**🖥️ GUI — Aurora Serverless v2 Create:**
```
RDS → Create database

Engine: Aurora (PostgreSQL Compatible)
Template: Serverless (বা Production → then choose Serverless)

Capacity settings:
→ Minimum ACUs: 0.5
→ Maximum ACUs: 8  (adjust based on expected peak)

Settings:
→ Cluster identifier: mydb-serverless
→ Master username: postgres
→ Password: [strong password]

Connectivity:
→ VPC, subnet group, security group same configure

→ Create cluster
```

**💻 CLI:**
```bash
# Aurora Serverless v2 Cluster
aws rds create-db-cluster \
    --db-cluster-identifier mydb-serverless \
    --engine aurora-postgresql \
    --engine-version "16.2" \
    --master-username postgres \
    --master-user-password 'ServerlessPass@2024!' \
    --db-subnet-group-name prod-db-subnet-group \
    --vpc-security-group-ids sg-xxxxxxxxx \
    --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=8 \
    --storage-encrypted \
    --backup-retention-period 7 \
    --enable-cloudwatch-logs-exports postgresql

# Writer instance তৈরি করো
aws rds create-db-instance \
    --db-instance-identifier mydb-serverless-writer \
    --db-cluster-identifier mydb-serverless \
    --db-instance-class db.serverless \
    --engine aurora-postgresql

# Scaling activity দেখো
aws rds describe-db-clusters \
    --db-cluster-identifier mydb-serverless \
    --query 'DBClusters[0].ServerlessV2ScalingConfiguration'
```

---

## 3.6 Aurora Global Database — Multi-region

```
Aurora Global Database:
  Primary region: writes happen here
  Secondary regions: replicated copies (read-only)
  Replication lag: typically <1 second (physical replication)
  Failover: ~1 minute (promote secondary to primary)

Use cases:
  ① Global application: users in different continents → low latency reads
  ② Disaster recovery: region failure → failover to secondary
  ③ Data locality compliance: data in specific region

Cost:
  Primary cluster: normal Aurora cost
  Secondary cluster: additional charge per ACU
  Replication: charged per GB replicated
  → Expensive, only for global/DR requirements
```

**🖥️ GUI — Global Database:**
```
RDS → Global databases → Create global database

Primary cluster:
→ Existing cluster: select তোমার existing Aurora cluster

→ "Create global database" click করো

Secondary region যোগ করো:
→ Global database → Actions → "Add region"
→ Secondary region: ap-south-1 (Mumbai)
→ DB instance class: same বা smaller
→ Configure: subnet group, security group in secondary region

→ "Add region" click করো
⏳ ~30 minutes
```

---

## 3.7 Aurora Backup এবং Backtrack

**Backup (same as RDS):**
```bash
# Automated backup automatically হয়
# Manual snapshot:
aws rds create-db-cluster-snapshot \
    --db-cluster-identifier mydb-aurora-cluster \
    --db-cluster-snapshot-identifier mydb-snapshot-$(date +%Y%m%d)
```

**Backtrack — Aurora Unique Feature:**
```
Backtrack:
  Storage level rewind — কোনো restore দরকার নেই!
  "ওই সময়ে ফিরে যাও" — same cluster, same endpoint
  Speed: seconds (না minutes)
  Backtrack window: up to 72 hours
  Cost: $0.012/GB-hour retained

vs PITR:
  PITR: নতুন instance তৈরি করে → endpoint change → app update
  Backtrack: same instance → endpoint same → instant
```

**🖥️ GUI — Backtrack Enable:**
```
Aurora cluster create করার সময়:
Additional configuration:
→ Enable Backtrack: ✅
→ Target backtrack window: 24 hours

Backtrack করো (disaster হলে):
RDS → Databases → cluster select করো
→ Actions → "Backtrack"
→ Backtrack to: custom date and time
→ 2024-03-08 09:59:00 UTC
→ "Backtrack cluster"

⚠️ This will affect the cluster immediately!
   Application temporarily unavailable during backtrack
   Reads/Writes paused during the process
```

**💻 CLI:**
```bash
# Cluster তৈরিতে backtrack enable
aws rds create-db-cluster \
    --db-cluster-identifier mydb-aurora \
    --engine aurora-postgresql \
    --engine-version "16.2" \
    --backtrack-window 86400 \  # 24 hours in seconds
    ...

# Backtrack করো
aws rds backtrack-db-cluster \
    --db-cluster-identifier mydb-aurora \
    --backtrack-to "2024-03-08T09:59:00+00:00" \
    --no-force-apply-earliest-time-on-point-in-time-unavailable

# Backtrack status দেখো
aws rds describe-db-cluster-backtracks \
    --db-cluster-identifier mydb-aurora
```

---

## 3.8 Hands-on: Aurora Cluster Setup

**Duration: ~45 minutes | Cost: ~$0.10/hour (stop করলে storage only)**

### 🖥️ GUI Method

**Step 1: Aurora Cluster তৈরি করো**
```
RDS → Create database

Choose a database creation method:
→ Standard create

Engine options:
→ Amazon Aurora
→ Amazon Aurora PostgreSQL-Compatible Edition

Engine version:
→ Aurora PostgreSQL 16.x (latest)

Templates:
→ Dev/Test (lab এর জন্য)
  (Production select করলে Multi-AZ automatically)

Settings:
→ DB cluster identifier: lab-aurora-cluster
→ Master username: postgres
→ Master password: AuroraLab@2024!

Instance configuration:
→ DB instance class: db.t3.medium
  (Aurora Serverless নয়, Provisioned)

Availability & durability:
→ Create an Aurora Replica/Reader node in a different AZ: No
  (cost বাঁচাতে, production এ Yes করো)

Connectivity:
→ VPC: lab-vpc
→ DB subnet group: lab-db-subnet-group
→ Public access: No
→ VPC security group: lab-db-sg

Additional configuration:
→ Initial DB name: labdb
→ Backup retention: 1 day
→ Backtrack: Enable, 1 hour (lab এর জন্য)
→ Performance Insights: Enable
→ Enhanced monitoring: Enable, 60 seconds
→ Log exports: postgresql
→ Deletion protection: No (lab, delete করতে হবে)

→ "Create database" click করো
⏳ ~10-15 minutes
```

**Step 2: Endpoints দেখো**
```
RDS → Databases → "lab-aurora-cluster" click করো

Endpoints:
  Writer: lab-aurora-cluster.cluster-xxxx.ap-southeast-1.rds.amazonaws.com:5432
  Reader: lab-aurora-cluster.cluster-ro-xxxx.ap-southeast-1.rds.amazonaws.com:5432

→ Writer endpoint copy করো
```

**Step 3: Connect এবং Test করো**
```bash
# SSH tunnel Writer এ
ssh -i ~/.ssh/lab-key.pem \
    -L 5440:WRITER_ENDPOINT:5432 \
    -N ec2-user@BASTION_IP &

# Connect করো
psql -h 127.0.0.1 -p 5440 -U postgres -d labdb
```

```sql
-- Aurora specific: ইটা RDS না Aurora দেখো
SELECT aurora_version();
-- Aurora PostgreSQL এ এই function আছে

-- Aurora storage status
SELECT * FROM aurora_db_instance_maintenance_status();

-- Cluster info
SELECT server_id, session_id, aurora_version()
FROM aurora_replica_status();

-- Test data তৈরি করো
CREATE TABLE aurora_test (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    val TEXT,
    created TIMESTAMPTZ DEFAULT NOW()
);
INSERT INTO aurora_test (val)
SELECT 'row-' || generate_series FROM generate_series(1,1000);

-- Performance Insights verify করো
SELECT pg_sleep(2);  -- slow query তৈরি করো
```

**Step 4: Backtrack Test করো**
```sql
-- Current time note করো
SELECT NOW();  -- e.g., 2024-03-08 14:30:00+00

-- Data delete করো (simulate mistake)
DELETE FROM aurora_test WHERE id < 500;
SELECT COUNT(*) FROM aurora_test;  -- 501 rows
```

```
RDS → Databases → lab-aurora-cluster
→ Actions → "Backtrack"
→ Custom date and time: [14:29:00 — deletion এর আগে]
→ "Backtrack cluster"

⏳ ~30 seconds
```

```sql
-- Reconnect করো এবং verify
SELECT COUNT(*) FROM aurora_test;  -- 1000 rows ফিরে আসবে ✅
```

---

### 💻 CLI Method

```bash
# Aurora Cluster Create
aws rds create-db-cluster \
    --db-cluster-identifier lab-aurora-cluster \
    --engine aurora-postgresql \
    --engine-version "16.2" \
    --master-username postgres \
    --master-user-password 'AuroraLab@2024!' \
    --db-subnet-group-name lab-db-subnet-group \
    --vpc-security-group-ids $SG_DB \
    --database-name labdb \
    --backup-retention-period 1 \
    --backtrack-window 3600 \
    --enable-cloudwatch-logs-exports postgresql \
    --no-deletion-protection

# Writer instance
aws rds create-db-instance \
    --db-instance-identifier lab-aurora-writer \
    --db-cluster-identifier lab-aurora-cluster \
    --db-instance-class db.t3.medium \
    --engine aurora-postgresql

aws rds wait db-instance-available \
    --db-instance-identifier lab-aurora-writer
echo "Aurora ready!"

# Endpoints
WRITER=$(aws rds describe-db-clusters \
    --db-cluster-identifier lab-aurora-cluster \
    --query 'DBClusters[0].Endpoint' --output text)
READER=$(aws rds describe-db-clusters \
    --db-cluster-identifier lab-aurora-cluster \
    --query 'DBClusters[0].ReaderEndpoint' --output text)
echo "Writer: $WRITER"
echo "Reader: $READER"
```

---

# 4. RDS Proxy — Connection Pooling Managed

## 4.1 RDS Proxy কী এবং কেন

```
Problem:
  Lambda functions, serverless apps → thousands of short connections
  RDS max_connections = 200-1000
  Too many connections → "too many connections" error

RDS Proxy:
  Managed connection pooler (pgBouncer এর মতো কিন্তু AWS managed)
  App → RDS Proxy → RDS/Aurora
  Connection reuse, multiplexing
  Automatic failover handling

কখন ব্যবহার করবো:
  ✅ Lambda + RDS (classic use case)
  ✅ Serverless applications
  ✅ Many short-lived connections
  ✅ Seamless failover (proxy handles reconnection)

Cost:
  $0.015/vCPU-hour of the protected database
  db.r7g.xlarge (4 vCPU) → 4 × $0.015 = $0.06/hour ≈ $43/month
  → Expensive for small workloads, Lambda তে worth it
```

---

## 4.2 RDS Proxy Setup করো

**🖥️ GUI:**
```
RDS → Proxies → Create proxy

Proxy configuration:
→ Proxy identifier: prod-pg-proxy
→ Engine family: PostgreSQL
→ Require TLS: Yes ✅

Target group configuration:
→ Database: mydb-prod (RDS বা Aurora cluster)
→ Connection pool max: 100% (of max_connections)
→ Connection borrow timeout: 120 seconds

Connectivity:
→ Secrets Manager secret: database/prod/postgres
  (Proxy Secrets Manager থেকে credentials নেয়)
→ IAM role: Create new (অথবা existing)
→ VPC: prod-vpc
→ Subnets: private subnets select করো
→ VPC security group: prod-db-sg

→ Create proxy
⏳ ~5 minutes

Proxy endpoint পাবে:
prod-pg-proxy.proxy-xxxx.ap-southeast-1.rds.amazonaws.com
```

**💻 CLI:**
```bash
# Secret তৈরি করো (Proxy Secrets Manager থেকে নেয়)
SECRET_ARN=$(aws secretsmanager create-secret \
    --name "database/prod/postgres" \
    --secret-string '{
        "username": "postgres",
        "password": "ProdPass@2024!",
        "engine": "postgres",
        "host": "mydb-prod.xxxx.ap-southeast-1.rds.amazonaws.com",
        "port": 5432,
        "dbname": "proddb"
    }' --query 'ARN' --output text)

# IAM Role for Proxy
PROXY_ROLE=$(aws iam create-role \
    --role-name rds-proxy-role \
    --assume-role-policy-document '{
        "Version":"2012-10-17",
        "Statement":[{
            "Effect":"Allow",
            "Principal":{"Service":"rds.amazonaws.com"},
            "Action":"sts:AssumeRole"
        }]
    }' --query 'Role.Arn' --output text)

aws iam attach-role-policy \
    --role-name rds-proxy-role \
    --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite

# Proxy তৈরি
aws rds create-db-proxy \
    --db-proxy-name prod-pg-proxy \
    --engine-family POSTGRESQL \
    --auth AuthScheme=SECRETS,SecretArn=$SECRET_ARN,IAMAuth=DISABLED \
    --role-arn $PROXY_ROLE \
    --vpc-subnet-ids $DB_1A $DB_1B \
    --vpc-security-group-ids $SG_DB \
    --require-tls

# Proxy endpoint
aws rds describe-db-proxies \
    --db-proxy-name prod-pg-proxy \
    --query 'DBProxies[0].Endpoint' --output text
```

---

## 4.3 Hands-on: RDS Proxy Configure করো

```bash
# Proxy endpoint দিয়ে connect করো
# (Bastion দিয়ে proxy তে connect)
PROXY_ENDPOINT="prod-pg-proxy.proxy-xxxx.ap-southeast-1.rds.amazonaws.com"

ssh -i ~/.ssh/lab-key.pem \
    -L 5445:${PROXY_ENDPOINT}:5432 \
    -N ec2-user@$BASTION_IP &

psql -h 127.0.0.1 -p 5445 -U postgres -d proddb

# Proxy statistics দেখো
aws rds describe-db-proxy-target-groups \
    --db-proxy-name prod-pg-proxy

# Proxy logs (CloudWatch এ)
aws logs describe-log-groups \
    --log-group-name-prefix "/aws/rds/proxy/prod-pg-proxy"
```

---

# 5. Performance Insights — Query Analytics

## 5.1 Performance Insights কী

```
Performance Insights = AWS এর built-in query analytics
  pg_stat_statements এর managed version
  DBLoad (Database Load) metric — key concept

DBLoad:
  Average number of active sessions
  DBLoad = 1 মানে: 1 session সবসময় active
  DBLoad > vCPU count → bottleneck!

Wait Events:
  Sessions কী কারণে wait করছে?
  CPU: CPU bound
  IO:bufferIO: Disk I/O waiting
  Lock: Lock waiting
  Client: Application slow to read results
```

---

## 5.2 Dashboard ব্যবহার করো

**🖥️ GUI:**
```
RDS → Databases → DB → "Monitoring" tab
→ "Performance Insights" section → "View in Performance Insights"

অথবা:
Performance Insights → select your DB

Dashboard:
  Top section: DBLoad over time (graph)
  Bottom section: Top SQL, Wait events, Hosts, Users

Time range:
  Last 1 hour / 3 hours / 12 hours / custom

Counter metrics (right side):
  db.SQL.tup_fetched (rows fetched/sec)
  db.SQL.tup_inserted (rows inserted/sec)
  db.Transactions.active (active transactions)
  db.Transactions.idle_in_transaction
```

---

## 5.3 Top SQL এবং Wait Events

**🖥️ GUI:**
```
Performance Insights Dashboard:

Top SQL tab:
  Sorted by: DB Load, Executions/sec, Avg latency
  Click on any query → detail দেখাবে:
    SQL text
    Statistics: avg latency, executions, rows examined
    Wait breakdown

Load by wait tab:
  কোন wait event এ সবচেয়ে বেশি time?
  IO:bufferIO বেশি → shared_buffers বাড়াও
  Lock বেশি → lock contention investigate করো
  CPU বেশি → query optimize করো

Filters:
  Filter by: specific SQL, user, host, database
```

---

## 5.4 Hands-on: Slow Query Identify করো

```sql
-- RDS তে connect করে slow queries তৈরি করো
-- (Performance Insights এ দেখা যাবে)

-- Slow query 1: Sequential scan
CREATE TABLE perf_test AS
SELECT generate_series(1,500000) AS id,
       md5(random()::text) AS data;

-- Index ছাড়া query (slow)
SELECT * FROM perf_test WHERE id = 250000;
-- এটা ~2 seconds লাগবে → Performance Insights এ দেখা যাবে

-- Slow query 2: Aggregation
SELECT COUNT(*), AVG(length(data))
FROM perf_test
WHERE data LIKE 'a%';

-- Fix করো
CREATE INDEX idx_perf_id ON perf_test(id);

-- এখন fast
EXPLAIN ANALYZE SELECT * FROM perf_test WHERE id = 250000;
```

```
Performance Insights এ দেখো:
→ Top SQL এ slow queries দেখাবে
→ Wait events: আগে IO:bufferIO বেশি ছিল
→ Index এর পরে: CPU bound হবে (IO কমে যাবে)
→ DBLoad কমবে
```

---

# 6. Enhanced Monitoring এবং CloudWatch

## 6.1 Basic vs Enhanced Monitoring

```
Basic Monitoring (Free):
  5-minute interval
  DB-level metrics শুধু
  CloudWatch এ automatically আসে
  CPU, Memory, Storage, IOPS

Enhanced Monitoring (charge: $0.015/GB log):
  1/5/10/15/30/60 second interval
  OS-level metrics:
    CPU breakdown (user, system, wait, nice, steal)
    Memory: free, cached, buffers
    Disk I/O per device
    Network throughput
    PostgreSQL process metrics
  CloudWatch Logs এ যায়
  Requires: monitoring IAM role
```

---

## 6.2 Key RDS Metrics এবং Thresholds

| Metric | Healthy | Warning | Critical | Action |
|---|---|---|---|---|
| `CPUUtilization` | < 70% | 70-85% | > 85% | Scale up / optimize |
| `DatabaseConnections` | < 70% max | 70-85% | > 85% | RDS Proxy / pgBouncer |
| `FreeStorageSpace` | > 25% | 10-25% | < 10% | Increase storage |
| `FreeableMemory` | > 25% | 15-25% | < 15% | Scale up RAM |
| `ReadLatency` | < 5ms | 5-20ms | > 20ms | Check queries / IOPS |
| `WriteLatency` | < 5ms | 5-20ms | > 20ms | Check queries / IOPS |
| `ReplicaLag` | < 5s | 5-60s | > 60s | Check replica load |
| `DiskQueueDepth` | < 1 | 1-10 | > 10 | Increase IOPS |
| `SwapUsage` | 0 | < 256MB | > 256MB | Scale up RAM |
| `BurstBalance` | > 50% | 20-50% | < 20% | Upgrade to gp3 |

---

## 6.3 CloudWatch Logs — Database Logs

```
RDS Logs CloudWatch এ export করো:
  postgresql: main PostgreSQL log
  upgrade: major version upgrade log

Log এ কী দেখবে:
  Slow queries (log_min_duration_statement)
  Connections/disconnections
  Lock waits
  Autovacuum
  Errors
```

**🖥️ GUI:**
```
RDS → Databases → DB → Logs & events tab
→ "Log exports" section
→ PostgreSQL log, upgrade log এ ✅

CloudWatch তে দেখো:
CloudWatch → Log groups
→ /aws/rds/instance/mydb-prod/postgresql

Log Insights query:
→ CloudWatch → Log Insights
→ Select: /aws/rds/instance/mydb-prod/postgresql

Query:
fields @timestamp, @message
| filter @message like /duration:/
| parse @message "duration: * ms" as duration_ms
| filter duration_ms > 1000
| sort duration_ms desc
| limit 20
```

**💻 CLI:**
```bash
# Log export enable
aws rds modify-db-instance \
    --db-instance-identifier mydb-prod \
    --cloudwatch-logs-export-configuration \
        EnableLogTypes=postgresql,upgrade \
    --apply-immediately

# Logs দেখো
aws logs get-log-events \
    --log-group-name /aws/rds/instance/mydb-prod/postgresql \
    --log-stream-name $(aws logs describe-log-streams \
        --log-group-name /aws/rds/instance/mydb-prod/postgresql \
        --query 'logStreams[-1].logStreamName' --output text) \
    --limit 20 \
    --query 'events[*].message' \
    --output text

# Slow queries search
aws logs start-query \
    --log-group-name /aws/rds/instance/mydb-prod/postgresql \
    --start-time $(date -d '1 hour ago' +%s) \
    --end-time $(date +%s) \
    --query-string 'fields @message | filter @message like /duration:/ | parse @message "duration: * ms" as ms | filter ms > 1000 | sort ms desc | limit 10'
```

---

## 6.4 Hands-on: Complete Monitoring Setup

**🖥️ GUI:**
```
Step 1: Enhanced Monitoring Enable করো
RDS → DB → Modify
→ Monitoring → Enable Enhanced monitoring: ✅
→ Monitoring interval: 60 seconds
→ Monitoring role: create new
→ Modify

Step 2: Log exports enable করো
→ Log exports → postgresql, upgrade ✅

Step 3: Performance Insights enable করো
→ Performance Insights → Enable ✅
→ Retention: 7 days (free)

Step 4: CloudWatch Alarms তৈরি করো (Lab 7 থেকে)
→ CPU, Storage, Connections, ReplicaLag

Step 5: Dashboard তৈরি করো
CloudWatch → Dashboards → Create dashboard
→ Add widgets for:
   RDS CPUUtilization
   RDS DatabaseConnections
   RDS FreeStorageSpace
   RDS ReadLatency / WriteLatency
   RDS ReplicaLag (if replica exists)
```

---

# 7. RDS Security — Best Practices

## 7.1 Encryption at Rest এবং in Transit

```
Encryption at Rest:
  Enable at creation time only (cannot enable later)
  KMS key: AWS managed (free) বা CMK ($1/month)
  Encrypts: data, automated backups, replicas, snapshots

Encryption in Transit (SSL/TLS):
  By default available
  Force SSL: rds.force_ssl = 1 (Parameter Group)
  Certificate: download from AWS
  Connection: sslmode=require বা verify-full
```

**SSL Connection:**
```bash
# AWS RDS CA certificate download করো
wget https://truststore.pki.rds.amazonaws.com/ap-southeast-1/ap-southeast-1-bundle.pem

# psql দিয়ে SSL connect
psql "host=mydb.xxxx.ap-southeast-1.rds.amazonaws.com \
      port=5432 \
      user=postgres \
      dbname=mydb \
      sslmode=verify-full \
      sslrootcert=ap-southeast-1-bundle.pem"

# SSL connection verify
psql -c "SELECT ssl, version FROM pg_stat_ssl WHERE pid = pg_backend_pid();"
```

---

## 7.2 IAM Database Authentication

```
IAM Database Authentication:
  Password ছাড়া IAM token দিয়ে connect করো
  Token 15 minutes valid
  IAM Policy দিয়ে access control
  RDS এ SSL required (automatically)

Use cases:
  EC2 instances (role দিয়ে)
  Lambda functions
  ECS/Fargate containers
  No password management

Limitations:
  Max 200 connections/second (token generation)
  PostgreSQL: rds_iam role দরকার
```

**Setup:**
```sql
-- RDS তে user তৈরি করো
CREATE USER app_iam_user;
GRANT rds_iam TO app_iam_user;
GRANT CONNECT ON DATABASE mydb TO app_iam_user;
GRANT USAGE ON SCHEMA public TO app_iam_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_iam_user;
```

```bash
# IAM Policy (EC2 Role এ attach করো)
cat > /tmp/rds-iam-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": ["rds-db:connect"],
        "Resource": "arn:aws:rds-db:ap-southeast-1:123456789012:dbuser:db-XXXX/app_iam_user"
    }]
}

EOF

# Token generate করো (EC2 তে, IAM Role থাকলে)
TOKEN=$(aws rds generate-db-auth-token \
    --hostname mydb.xxxx.ap-southeast-1.rds.amazonaws.com \
    --port 5432 \
    --region ap-southeast-1 \
    --username app_iam_user)

# Token দিয়ে connect করো
PGPASSWORD=$TOKEN psql \
    "host=mydb.xxxx.ap-southeast-1.rds.amazonaws.com \
     port=5432 user=app_iam_user dbname=mydb \
     sslmode=verify-full sslrootcert=ap-southeast-1-bundle.pem"
```

---

## 7.3 Secrets Manager + RDS Integration

```
RDS Managed Secrets (Native Integration):
  RDS automatically creates and rotates the secret
  Application pulls password from Secrets Manager
  No hardcoded passwords!
  Rotation: automatic, configurable (7-30 days)
```

**🖥️ GUI:**
```
RDS → Databases → DB → Modify

Credentials:
→ Manage master credentials in AWS Secrets Manager: ✅
→ KMS key: aws/secretsmanager (default, free)

→ Modify

Rotation configure করো:
Secrets Manager → তোমার secret → Rotation → Edit rotation
→ Automatic rotation: Enable ✅
→ Rotation schedule: 30 days
→ Rotation function: use default Lambda
→ Save
```

**Application এ:**
```python
import boto3, json, psycopg2

def get_db_connection():
    client = boto3.client("secretsmanager", region_name="ap-southeast-1")
    secret = json.loads(
        client.get_secret_value(
            SecretId="rds!db-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
        )["SecretString"]
    )
    return psycopg2.connect(
        host=secret["host"], port=secret["port"],
        database=secret["dbname"],
        user=secret["username"], password=secret["password"],
        sslmode="require"
    )
```

---

## 7.4 Security Groups এবং Network Isolation

```
Best Practice Network Setup:

Internet → [No direct DB access]
App Servers (EC2/ECS) → sg-app → Port 5432 → RDS (sg-db)
DBA Bastion → sg-bastion → Port 5432 → RDS (sg-db)
Lambda → sg-lambda → Port 5432 → RDS (sg-db)

sg-db Inbound Rules:
  5432 from sg-app      (application servers)
  5432 from sg-bastion  (DBA access)
  5432 from sg-lambda   (Lambda functions)
  5432 from sg-proxy    (RDS Proxy)

What NOT to do:
  Never: 5432 from 0.0.0.0/0
  Never: Public access = Yes
  Never: No encryption in transit
```

---

## 7.5 Hands-on: Secure RDS Checklist

```bash
DB="mydb-prod"

echo "=== Security Checklist ==="

echo "1. Encryption at rest:"
aws rds describe-db-instances --db-instance-identifier $DB \
    --query "DBInstances[0].StorageEncrypted"

echo "2. SSL forced (check parameter group):"
aws rds describe-db-parameters \
    --db-parameter-group-name prod-pg16 \
    --query "Parameters[?ParameterName==\`rds.force_ssl\`].ParameterValue" \
    --output text

echo "3. Public access disabled:"
aws rds describe-db-instances --db-instance-identifier $DB \
    --query "DBInstances[0].PubliclyAccessible"

echo "4. IAM auth enabled:"
aws rds describe-db-instances --db-instance-identifier $DB \
    --query "DBInstances[0].IAMDatabaseAuthenticationEnabled"

echo "5. Deletion protection:"
aws rds describe-db-instances --db-instance-identifier $DB \
    --query "DBInstances[0].DeletionProtection"

echo "6. Multi-AZ:"
aws rds describe-db-instances --db-instance-identifier $DB \
    --query "DBInstances[0].MultiAZ"
```

---

# 8. Database Migration Service (DMS)

## 8.1 DMS কী এবং কখন ব্যবহার করবো

```
AWS DMS:
  Source DB → DMS → Target DB
  Schema + data migrate + ongoing replication (CDC)
  Near-zero downtime migration possible

Use cases:
  On-premise PostgreSQL → RDS PostgreSQL
  MySQL → Aurora PostgreSQL (heterogeneous)
  Oracle → PostgreSQL
  One RDS → Another RDS

Migration Types:
  Full Load: existing data একবার copy
  CDC: ongoing changes replicate
  Full Load + CDC: সবচেয়ে common
```

---

## 8.2 On-premise PostgreSQL → RDS Migration

**🖥️ GUI — Step by Step:**
```
Step 1: Source DB prepare করো
  On-premise PG তে:
  postgresql.conf: wal_level = logical
  pg_hba.conf: DMS instance IP allow করো
  Restart PostgreSQL

Step 2: DMS Console → Replication instances → Create
  Name: pg-migration-dms
  Class: dms.t3.medium
  Storage: 50GB
  Multi-AZ: No
  VPC: prod-vpc (source reach করতে পারে)
  → Create

Step 3: Endpoints → Create endpoint
  Source endpoint:
    Type: Source
    Engine: PostgreSQL
    Server: 172.16.x.x
    Port: 5432
    Database: mydb
    Username/Password: [credentials]

  Target endpoint:
    Type: Target
    Engine: PostgreSQL (বা Aurora PostgreSQL)
    Server: RDS endpoint
    Database/credentials

Step 4: Test connections (both endpoints)
  → Test connection button

Step 5: Migration tasks → Create task
  Task: migrate-mydb
  Replication instance: pg-migration-dms
  Source: source endpoint
  Target: target endpoint
  Migration type: Migrate existing data and replicate ongoing changes

  Table mappings → Add rule:
    Schema: % (all)
    Table: % (all)
    Action: Include

  → Create task → Start automatically

Step 6: Monitor
  Task → Table statistics → rows loaded, rows inserts/updates/deletes
  Full Load Complete → CDC begins
  Lag = 0 → ready for cutover

Step 7: Cutover
  Stop writes on source (maintenance mode)
  Wait for lag = 0
  Update application connection string → target
  Done!
```

---

## 8.3 MySQL → Aurora PostgreSQL Migration

```
Heterogeneous migration needs schema conversion.

AWS Schema Conversion Tool (SCT):
  Download: https://docs.aws.amazon.com/SchemaConversionTool
  Connect to MySQL source
  Convert schema → PostgreSQL compatible
  Review conversion issues (red/orange items)
  Apply to Aurora PostgreSQL target

Common MySQL → PostgreSQL conversion issues:
  AUTO_INCREMENT → IDENTITY
  DATETIME → TIMESTAMPTZ
  TINYINT(1) → BOOLEAN
  IFNULL() → COALESCE()
  LIMIT x,y → LIMIT y OFFSET x
  Backticks → double quotes

After schema conversion:
  Use DMS for data migration (Full Load + CDC)
  Test application thoroughly before cutover
```

---

## 8.4 Hands-on: DMS Quick Setup

```bash
# Replication Instance
aws dms create-replication-instance \
    --replication-instance-identifier pg-migration-dms \
    --replication-instance-class dms.t3.medium \
    --allocated-storage 50 \
    --no-multi-az \
    --no-publicly-accessible \
    --replication-subnet-group-identifier prod-db-subnet-group

# Source Endpoint
aws dms create-endpoint \
    --endpoint-identifier source-postgres \
    --endpoint-type source \
    --engine-name postgres \
    --server-name 172.16.x.x \
    --port 5432 \
    --database-name mydb \
    --username postgres \
    --password "SourcePass@2024!"

# Target Endpoint
aws dms create-endpoint \
    --endpoint-identifier target-rds \
    --endpoint-type target \
    --engine-name aurora-postgresql \
    --server-name mydb.cluster-xxxx.rds.amazonaws.com \
    --port 5432 \
    --database-name mydb \
    --username postgres \
    --password "TargetPass@2024!"

# Migration Task
aws dms create-replication-task \
    --replication-task-identifier migrate-mydb \
    --source-endpoint-arn SOURCE_ARN \
    --target-endpoint-arn TARGET_ARN \
    --replication-instance-arn REPL_ARN \
    --migration-type full-load-and-cdc \
    --table-mappings file:///tmp/table-mappings.json

# Task status
aws dms describe-replication-tasks \
    --filters Name=replication-task-id,Values=migrate-mydb \
    --query "ReplicationTasks[0].[Status,ReplicationTaskStats]"
```

---

# 9. Cost Optimization — RDS এবং Aurora

## 9.1 Reserved Instances — সবচেয়ে বড় Saving

```
On-demand vs Reserved (db.r7g.xlarge, ap-southeast-1):
  On-demand:  $0.409/hour = $296/month
  1-year RI:  $0.248/hour = $179/month  (39% discount)
  3-year RI:  $0.171/hour = $124/month  (58% discount)

Multi-AZ (2x instance):
  On-demand:  $592/month
  1-year RI:  $359/month

When to reserve:
  Production DB running 24/7 → Always reserve
  Dev/Test → On-demand (can stop/start)
```

**🖥️ GUI:**
```
RDS → Reserved instances → Purchase reserved DB instance

→ Engine: PostgreSQL
→ Instance class: db.r7g.xlarge
→ Multi-AZ: Yes/No
→ Term: 1 Year
→ Offering type: Partial Upfront (best balance)
→ Quantity: 1
→ Purchase
```

---

## 9.2 Right-sizing

```bash
# AWS Compute Optimizer দেখো
# Console → Compute Optimizer → RDS

# CLI এ recommendations
aws compute-optimizer get-rds-instance-recommendations \
    --account-ids $(aws sts get-caller-identity --query Account --output text) \
    --query "rdsInstanceRecommendations[*].[
        currentInstanceType,
        recommendationOptions[0].instanceType,
        recommendationOptions[0].estimatedMonthlySavings.value
    ]" \
    --output table 2>/dev/null || echo "Check AWS Console → Compute Optimizer"
```

---

## 9.3 Storage Optimization

```bash
# gp2 → gp3 migration (20% cheaper, no downtime)
aws rds modify-db-instance \
    --db-instance-identifier mydb-prod \
    --storage-type gp3 \
    --iops 3000 \
    --storage-throughput 125 \
    --apply-immediately

# Storage autoscaling enable
aws rds modify-db-instance \
    --db-instance-identifier mydb-prod \
    --max-allocated-storage 500 \
    --apply-immediately

# Old snapshots cleanup (30+ days পুরনো)
CUTOFF=$(date -d '30 days ago' +%Y-%m-%d)
aws rds describe-db-snapshots \
    --db-instance-identifier mydb-prod \
    --snapshot-type manual \
    --query "DBSnapshots[?SnapshotCreateTime<='${CUTOFF}'].[DBSnapshotIdentifier]" \
    --output text | while read snap; do
    echo "Deleting old snapshot: $snap"
    aws rds delete-db-snapshot --db-snapshot-identifier $snap
done
```

---

## 9.4 Aurora Serverless vs Provisioned Decision

```
Aurora Serverless v2 wins when:
  Average ACU usage < 60% of minimum provisioned
  Variable traffic (peak >> off-peak)
  Dev databases (idle most of the time)

Provisioned + Reserved wins when:
  Steady, predictable load
  CPU > 70% consistently
  Known traffic pattern

Quick calculation:
  If avg monthly hours at full capacity < 500 hours → Serverless
  If avg monthly hours at full capacity > 500 hours → Provisioned + RI

Serverless scaling check:
```

```sql
-- Aurora Serverless current ACU
SELECT aurora_version(),
    current_setting('max_connections') AS max_conn;

-- Monitor serverless scaling in CloudWatch:
-- Metric: ServerlessDatabaseCapacity (ACU)
```

---

# 10. Troubleshooting — RDS এবং Aurora

## 10.1 Connection Issues

```bash
# Systematic diagnosis

# 1. RDS status
aws rds describe-db-instances \
    --db-instance-identifier mydb-prod \
    --query "DBInstances[0].[DBInstanceStatus,PubliclyAccessible,MultiAZ]"

# 2. Security group check
aws ec2 describe-security-groups \
    --group-ids SG_ID \
    --query "SecurityGroups[0].IpPermissions"

# 3. Network reachability (from Bastion)
# ssh lab-bastion
# nc -zv RDS_ENDPOINT 5432

# 4. Recent events
aws rds describe-events \
    --source-identifier mydb-prod \
    --source-type db-instance \
    --duration 60 \
    --query "Events[*].[Date,Message]" \
    --output table

# 5. SSL test
# psql "host=ENDPOINT sslmode=require ..."
```

---

## 10.2 Performance Issues

```sql
-- RDS তে connect করে diagnose

-- Active slow queries
SELECT pid, now()-query_start AS duration,
    state, wait_event_type, wait_event,
    LEFT(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle' AND now()-query_start > interval '5s'
ORDER BY duration DESC;

-- Lock contention
SELECT blocked.pid, LEFT(blocked.query,60) AS blocked_query,
    blocking.pid AS blocker, LEFT(blocking.query,60) AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- Top slow queries
SELECT LEFT(query,80) AS query, calls,
    ROUND(mean_exec_time::numeric,0)||'ms' AS avg_ms,
    ROUND(total_exec_time::numeric,0)||'ms' AS total_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 10;

-- Buffer hit rate (99%+ হওয়া উচিত)
SELECT ROUND(
    sum(heap_blks_hit)*100.0/
    NULLIF(sum(heap_blks_hit)+sum(heap_blks_read),0),2
) AS hit_rate FROM pg_statio_user_tables;

-- XID wraparound check
SELECT datname, age(datfrozenxid),
    2000000000-age(datfrozenxid) AS safe_left
FROM pg_database ORDER BY age(datfrozenxid) DESC;
```

---

## 10.3 Storage Issues

```sql
-- Large tables
SELECT schemaname, relname,
    pg_size_pretty(pg_total_relation_size(schemaname||"."||relname)) AS size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname||"."||relname) DESC
LIMIT 10;

-- Bloated tables
SELECT relname, n_dead_tup,
    ROUND(n_dead_tup*100.0/NULLIF(n_live_tup+n_dead_tup,0),1)||"%" AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000 ORDER BY n_dead_tup DESC LIMIT 5;

-- Temp file usage
SELECT datname, temp_files, pg_size_pretty(temp_bytes)
FROM pg_stat_database WHERE temp_files > 0;
```

```bash
# Storage autoscaling enable করো (emergency)
aws rds modify-db-instance \
    --db-instance-identifier mydb-prod \
    --max-allocated-storage 1000 \
    --apply-immediately
```

---

## 10.4 Failover Issues

```bash
# Failover status
aws rds describe-db-instances \
    --db-instance-identifier mydb-prod \
    --query "DBInstances[0].[AvailabilityZone,SecondaryAvailabilityZone,MultiAZ]"

# Recent failover events
aws rds describe-events \
    --source-identifier mydb-prod \
    --source-type db-instance \
    --duration 1440 \
    --query "Events[?contains(Message,'failover')].[Date,Message]" \
    --output table

# Force failover test
aws rds reboot-db-instance \
    --db-instance-identifier mydb-prod \
    --force-failover
```

```python
# Application level: connection retry
import psycopg2, time

def connect_with_retry(dsn, max_retries=5):
    for i in range(max_retries):
        try:
            return psycopg2.connect(dsn, connect_timeout=5)
        except psycopg2.OperationalError as e:
            if i < max_retries - 1:
                time.sleep(2 ** i)  # 1, 2, 4, 8, 16 seconds
            else:
                raise
```

---

## 10.5 Common Error Messages

| Error | মানে | Fix |
|---|---|---|
| `password authentication failed` | Wrong credentials | Secrets Manager verify করো |
| `no pg_hba.conf entry` | Security Group blocked | SG inbound rule check করো |
| `SSL SYSCALL error` | SSL issue | sslmode=require দাও |
| `Connection timed out` | Network/SG block | Bastion দিয়ে test করো |
| `too many connections` | max_connections exceeded | RDS Proxy add করো |
| `remaining connection slots reserved` | Superuser slots only | Connection pool optimize করো |
| `canceling statement due to conflict` | Standby query conflict | hot_standby_feedback=on করো |
| `disk full` | Storage exhausted | Storage autoscaling enable করো |
| `the database system is starting up` | DB rebooting | Retry করো |
| `could not resize shared memory` | Parameter change needs reboot | Maintenance window এ reboot করো |

---

# 11. Production Runbook — Day-to-Day Operations

## 11.1 Daily Checks

```bash
DB="mydb-prod"
REGION="ap-southeast-1"

echo "=== RDS Daily Check - $(date) ==="

# Status
aws rds describe-db-instances \
    --db-instance-identifier $DB \
    --query "DBInstances[0].[DBInstanceStatus,EngineVersion,MultiAZ]" \
    --output table

# Key metrics (last 30 min)
for metric in CPUUtilization DatabaseConnections FreeStorageSpace SwapUsage; do
    VAL=$(aws cloudwatch get-metric-statistics \
        --namespace AWS/RDS \
        --metric-name $metric \
        --dimensions Name=DBInstanceIdentifier,Value=$DB \
        --start-time $(date -u -d "30 minutes ago" +%Y-%m-%dT%H:%M:%SZ) \
        --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
        --period 1800 --statistics Average \
        --query "Datapoints[0].Average" --output text 2>/dev/null)
    echo "  $metric: $VAL"
done

# Active alarms
ALARMS=$(aws cloudwatch describe-alarms \
    --state-value ALARM \
    --alarm-name-prefix "RDS-${DB}" \
    --query "MetricAlarms[*].[AlarmName,StateReason]" \
    --output text 2>/dev/null)
[ -n "$ALARMS" ] && echo "ACTIVE ALARMS: $ALARMS" || echo "No active alarms"

# Latest snapshot
aws rds describe-db-snapshots \
    --db-instance-identifier $DB \
    --query "sort_by(DBSnapshots, &SnapshotCreateTime)[-1].[DBSnapshotIdentifier,SnapshotCreateTime]" \
    --output table
```

**🖥️ Console Daily Check:**
```
RDS → Dashboard:
  → All instances "Available" status?
  → Any "Maintenance required" banners?
  → Storage warnings?

CloudWatch → Dashboards → PostgreSQL-Production:
  → CPU trending up?
  → Connection count normal?
  → Storage remaining > 25%?

Performance Insights:
  → Any new slow queries?
  → DBLoad normal?
```

---

## 11.2 Weekly Tasks

```bash
# 1. Manual snapshot
aws rds create-db-snapshot \
    --db-instance-identifier mydb-prod \
    --db-snapshot-identifier weekly-$(date +%Y-W%V)

# 2. Old snapshot cleanup (30 days+)
CUTOFF=$(date -d "30 days ago" +%Y-%m-%d)
aws rds describe-db-snapshots \
    --db-instance-identifier mydb-prod \
    --snapshot-type manual \
    --query "DBSnapshots[?SnapshotCreateTime<='${CUTOFF}T00:00:00Z'].[DBSnapshotIdentifier]" \
    --output text | while read snap; do
    echo "Deleting: $snap"
    aws rds delete-db-snapshot --db-snapshot-identifier "$snap"
done

# 3. Cost check
aws ce get-cost-and-usage \
    --time-period Start=$(date -d "7 days ago" +%Y-%m-%d),End=$(date +%Y-%m-%d) \
    --granularity DAILY \
    --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Relational Database Service"]}}' \
    --metrics BlendedCost \
    --query "ResultsByTime[*].[TimePeriod.Start,Total.BlendedCost.Amount]" \
    --output table
```

---

## 11.3 Monthly Tasks

```bash
# 1. Security audit
echo "=== Monthly Security Audit ==="

echo "Public access check (should be empty):"
aws rds describe-db-instances \
    --query "DBInstances[?PubliclyAccessible==\`true\`].[DBInstanceIdentifier]" \
    --output text

echo "Unencrypted instances (should be empty):"
aws rds describe-db-instances \
    --query "DBInstances[?StorageEncrypted!=\`true\`].[DBInstanceIdentifier]" \
    --output text

echo "No deletion protection (should be empty for production):"
aws rds describe-db-instances \
    --query "DBInstances[?DeletionProtection!=\`true\` && contains(DBInstanceIdentifier, \`prod\`)].[DBInstanceIdentifier]" \
    --output text

# 2. Right-sizing review
echo "Compute Optimizer recommendations:"
aws compute-optimizer get-rds-instance-recommendations \
    --account-ids $(aws sts get-caller-identity --query Account --output text) \
    --query "rdsInstanceRecommendations[*].[currentInstanceType,recommendationOptions[0].instanceType,recommendationOptions[0].estimatedMonthlySavings.value]" \
    --output table 2>/dev/null

# 3. Cost summary
aws ce get-cost-and-usage \
    --time-period Start=$(date -d "last month" +%Y-%m-01),End=$(date +%Y-%m-01) \
    --granularity MONTHLY \
    --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Relational Database Service"]}}' \
    --metrics BlendedCost \
    --query "ResultsByTime[0].Total.BlendedCost"
```

---

## 11.4 Emergency Procedures

### Emergency 1: Database Unavailable

```bash
DB="mydb-prod"

# Status check
aws rds describe-db-instances \
    --db-instance-identifier $DB \
    --query "DBInstances[0].[DBInstanceStatus]"

# Recent events
aws rds describe-events \
    --source-identifier $DB \
    --source-type db-instance \
    --duration 60 \
    --query "Events[*].[Date,Message]" \
    --output table

# Reboot
aws rds reboot-db-instance --db-instance-identifier $DB
aws rds wait db-instance-available --db-instance-identifier $DB
echo "DB available!"
```

### Emergency 2: Accidental Data Loss

```bash
# PITR (RDS)
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier mydb-prod \
    --target-db-instance-identifier mydb-emergency-restore \
    --restore-time $(date -u -d "10 minutes ago" +%Y-%m-%dT%H:%M:%SZ) \
    --db-instance-class db.r7g.xlarge \
    --db-subnet-group-name prod-db-subnet-group \
    --vpc-security-group-ids $SG_DB
aws rds wait db-instance-available --db-instance-identifier mydb-emergency-restore

# Backtrack (Aurora only)
aws rds backtrack-db-cluster \
    --db-cluster-identifier mydb-aurora \
    --backtrack-to $(date -u -d "10 minutes ago" +%Y-%m-%dT%H:%M:%SZ)
```

### Emergency 3: Storage Almost Full

```bash
# Immediate: enable autoscaling
aws rds modify-db-instance \
    --db-instance-identifier mydb-prod \
    --max-allocated-storage 1000 \
    --apply-immediately
echo "Autoscaling enabled → storage will grow automatically"

# Identify what is taking space (connect to DB)
# psql: SELECT schemaname, relname, pg_size_pretty(pg_total_relation_size(...))

# Archive old data
# psql: CREATE TABLE archive AS SELECT * FROM logs WHERE created < '2023-01-01';
#       DELETE FROM logs WHERE created < '2023-01-01';
#       VACUUM logs;
```

---

*AWS Cloud Native DBA | Guide 3*
*RDS PostgreSQL · Aurora PostgreSQL · RDS Proxy · Performance Insights*
*Enhanced Monitoring · Security · DMS Migration · Cost Optimization*
*Production Runbook · Troubleshooting*


---

# 12. Free Tier — Hands-on Labs (GUI + CLI)

## 12.0 Free Tier Cost Guide এবং সাবধানতা

```
Guide 3 Labs এ কী free এবং কী charge হবে:

RDS (Free Tier — 12 months):
  ✅ db.t3.micro — 750 hours/month
  ✅ 20GB gp2/gp3 storage
  ✅ 20GB automated backup storage
  ❌ Multi-AZ — charge হবে (skip করো lab এ)
  ❌ Read Replica — আলাদা instance = আলাদা charge
     t3.micro Read Replica → 750 hours free এর মধ্যে পড়ে যদি
     primary + replica একসাথে চালাও না

Aurora:
  ❌ Aurora Free Tier নেই সরাসরি
  ✅ Aurora Serverless v2 — minimum 0.5 ACU = $0.06/hour
     Lab শেষে STOP করো → ACU pause হয়
     Storage charge থাকে (~$0.10/GB-month)

Performance Insights:
  ✅ 7 days retention — FREE
  ❌ Longer retention — charge

Enhanced Monitoring:
  ✅ 60-second interval — minimal cost (~$0.015/GB log)

RDS Proxy:
  ❌ $0.015/vCPU-hour — skip করো lab এ

⚠️ Lab Rules:
  ① সব lab শেষে resources STOP বা DELETE করো
  ② Aurora lab: stop করো (storage charge only)
  ③ Read Replica: Primary চালানোর সময়ই করো, তারপর delete
  ④ Billing Alarm $1 set থাকা উচিত (Guide 2 Lab 0)
  ⑤ Lab শেষে: AWS Console → Billing → Free Tier Usage দেখো
```

**Lab Prerequisites:**
```
Guide 2 Lab 0-3 complete হয়ে থাকলে:
  ✅ VPC, Subnets, Security Groups তৈরি আছে
  ✅ Bastion Host চলছে
  ✅ AWS CLI configured

না থাকলে Guide 2 এর Lab 2 এবং Lab 3 করো আগে।

Environment variables:
source /tmp/lab-ids.env
# VPC_ID, SG_DB, SG_BASTION, DB_1A, DB_1B, BASTION_IP set থাকবে
```

---

## 12.1 Lab 1 — RDS PostgreSQL তৈরি এবং Connect করো

**Duration: ~45 minutes | Cost: db.t3.micro Free Tier**

---

### 🖥️ GUI Method

**Step 1: Parameter Group তৈরি করো**
```
RDS → Parameter groups → Create parameter group

Details:
→ Parameter group family: postgres16
→ Type: DB Parameter Group
→ Group name: lab3-pg16
→ Description: Guide 3 Lab PostgreSQL 16
→ Create

lab3-pg16 এ click করো → Edit parameters:

Search করো এবং value set করো:
  log_min_duration_statement  → 500    (500ms slow query log)
  log_connections             → 1      (connection log)
  log_lock_waits              → 1      (lock wait log)
  log_temp_files              → 0      (temp file log)
  idle_in_transaction_session_timeout → 300000  (5 minutes)

→ Save changes
```

**Step 2: RDS Instance তৈরি করো**
```
RDS → Create database

Choose creation method:
→ Standard create

Engine:
→ PostgreSQL
→ PostgreSQL 16.x

Template:
→ Free tier  ✅
  (Multi-AZ automatically disabled)

Settings:
→ DB instance identifier: lab3-postgres
→ Master username: postgres
→ Credentials management: Self managed
→ Master password: Lab3Pass@2024!
→ Confirm password: Lab3Pass@2024!

Instance configuration:
→ DB instance class: db.t3.micro  (Free tier eligible ✅)

Storage:
→ Storage type: General Purpose SSD (gp3)
→ Allocated storage: 20 GB
→ Storage autoscaling: ✅ Enable
→ Maximum storage threshold: 30 GB

Connectivity:
→ VPC: lab-vpc  (Guide 2 এ তৈরি করা)
→ DB subnet group: lab-db-subnet-group
→ Public access: No
→ VPC security group:
   → Remove "default"
   → Add "lab-db-sg"
→ Availability zone: ap-southeast-1a

Additional configuration (expand করো):
→ Initial database name: lab3db
→ DB parameter group: lab3-pg16
→ Backup retention period: 1 day
→ Backup window: 19:00-20:00  (UTC, রাত ১টা Bangladesh time)
→ Maintenance window: sun:20:00-sun:21:00
→ Performance Insights: ✅ Enable
   → Retention period: 7 days (free)
→ Enhanced monitoring: ✅ Enable
   → Granularity: 60 seconds
→ Log exports: ✅ PostgreSQL log
→ Auto minor version upgrade: ✅ Enable
→ Deletion protection: ❌ Disable  (lab এ delete করতে হবে)

→ Create database button click করো

⏳ Status: Creating → ~10 minutes → Available
```

**Step 3: Endpoint নাও**
```
RDS → Databases → lab3-postgres click করো

Connectivity & security tab:
→ Endpoint: lab3-postgres.xxxx.ap-southeast-1.rds.amazonaws.com
→ Port: 5432
→ এই endpoint copy করো
```

**Step 4: SSH Tunnel দিয়ে Connect করো**
```
Local terminal এ:
ssh -i ~/.ssh/lab-key.pem \
    -L 5450:lab3-postgres.xxxx.ap-southeast-1.rds.amazonaws.com:5432 \
    -N ec2-user@BASTION_IP &

psql -h 127.0.0.1 -p 5450 -U postgres -d lab3db
Password: Lab3Pass@2024!
```

```sql
-- Connect হলে verify করো
SELECT version();
-- PostgreSQL 16.x on x86_64-pc-linux-gnu দেখাবে

-- Parameter verify করো
SHOW log_min_duration_statement;  -- 500ms
SHOW log_connections;              -- on

-- RDS specific function
SELECT rds_tools.engine_version();  -- রুলে error না হলে RDS তে আছো

-- Test table তৈরি করো
CREATE TABLE lab3_orders (
    id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    item    TEXT NOT NULL,
    amount  NUMERIC(10,2) NOT NULL,
    status  TEXT DEFAULT 'pending',
    created TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO lab3_orders (item, amount) VALUES
    ('Laptop', 999.99), ('Mouse', 29.99),
    ('Keyboard', 79.99), ('Monitor', 349.99),
    ('Headphones', 149.99);

SELECT * FROM lab3_orders;
```

---

### 💻 CLI Method

```bash
source /tmp/lab-ids.env

# Parameter Group
aws rds create-db-parameter-group \
    --db-parameter-group-name lab3-pg16 \
    --db-parameter-group-family postgres16 \
    --description "Guide 3 Lab PostgreSQL 16"

aws rds modify-db-parameter-group \
    --db-parameter-group-name lab3-pg16 \
    --parameters \
        "ParameterName=log_min_duration_statement,ParameterValue=500,ApplyMethod=immediate" \
        "ParameterName=log_connections,ParameterValue=1,ApplyMethod=immediate" \
        "ParameterName=log_lock_waits,ParameterValue=1,ApplyMethod=immediate" \
        "ParameterName=log_temp_files,ParameterValue=0,ApplyMethod=immediate" \
        "ParameterName=idle_in_transaction_session_timeout,ParameterValue=300000,ApplyMethod=immediate"

# RDS Instance
aws rds create-db-instance \
    --db-instance-identifier lab3-postgres \
    --db-instance-class db.t3.micro \
    --engine postgres \
    --engine-version "16.3" \
    --master-username postgres \
    --master-user-password "Lab3Pass@2024!" \
    --allocated-storage 20 \
    --storage-type gp3 \
    --max-allocated-storage 30 \
    --db-subnet-group-name lab-db-subnet-group \
    --vpc-security-group-ids $SG_DB \
    --db-parameter-group-name lab3-pg16 \
    --no-multi-az \
    --no-publicly-accessible \
    --backup-retention-period 1 \
    --preferred-backup-window "19:00-20:00" \
    --preferred-maintenance-window "sun:20:00-sun:21:00" \
    --auto-minor-version-upgrade \
    --enable-performance-insights \
    --performance-insights-retention-period 7 \
    --monitoring-interval 60 \
    --enable-cloudwatch-logs-exports postgresql \
    --no-deletion-protection \
    --db-name lab3db \
    --tags Key=Lab,Value=guide3

echo "Creating... (~10 minutes)"
aws rds wait db-instance-available --db-instance-identifier lab3-postgres
echo "Ready!"

LAB3_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier lab3-postgres \
    --query "DBInstances[0].Endpoint.Address" --output text)
echo "Endpoint: $LAB3_ENDPOINT"
echo "export LAB3_ENDPOINT=$LAB3_ENDPOINT" >> /tmp/lab-ids.env

# SSH Tunnel
ssh -i ~/.ssh/lab-key.pem \
    -L 5450:${LAB3_ENDPOINT}:5432 \
    -N -f ec2-user@$BASTION_IP
echo "Tunnel active on port 5450"

psql -h 127.0.0.1 -p 5450 -U postgres -d lab3db
```

---

## 12.2 Lab 2 — Parameter Group Tune করো

**Duration: ~20 minutes | Cost: Free**

---

### 🖥️ GUI Method

**Step 1: Current Parameters দেখো**
```
RDS → Parameter groups → lab3-pg16 click করো

Filter by: "current" বা specific parameter নাম
→ সব user-defined parameters দেখাবে (Source: user)

Parameters explore করো:
→ shared_buffers দেখো → value কী? {DBInstanceClassMemory/32768}
   এটা auto-calculated — t3.micro এর RAM এ ~96MB হবে
→ work_mem → default কত?
→ max_connections → value কী?
```

**Step 2: work_mem বাড়াও**
```
lab3-pg16 → Edit parameters

Search: work_mem
→ Current value: 4096 (4MB default)
→ New value: 16384  (16MB — t3.micro এর জন্য reasonable)
→ Apply method: immediate

→ Save changes
```

**Step 3: pg_stat_statements enable করো**
```
Search: shared_preload_libraries
→ Value: pg_stat_statements
→ Apply method: pending-reboot

→ Save changes

RDS → Databases → lab3-postgres → Actions → Reboot
→ Reboot DB instance (no failover — single AZ)
→ Confirm
⏳ ~2 minutes
```

**Step 4: Extension install এবং Verify করো**
```sql
-- Reconnect করো
CREATE EXTENSION pg_stat_statements;
CREATE EXTENSION pgcrypto;
CREATE EXTENSION "uuid-ossp";

-- Verify
SELECT name, installed_version FROM pg_available_extensions
WHERE installed_version IS NOT NULL;

-- work_mem verify
SHOW work_mem;  -- 16MB দেখাবে

-- pg_stat_statements test
SELECT * FROM lab3_orders WHERE amount > 100;
SELECT * FROM lab3_orders WHERE item LIKE '%top%';

SELECT LEFT(query, 80) AS query, calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 5;
```

---

### 💻 CLI Method

```bash
# work_mem বাড়াও
aws rds modify-db-parameter-group \
    --db-parameter-group-name lab3-pg16 \
    --parameters \
        "ParameterName=work_mem,ParameterValue=16384,ApplyMethod=immediate" \
        "ParameterName=shared_preload_libraries,ParameterValue=pg_stat_statements,ApplyMethod=pending-reboot"

# Reboot (pending-reboot parameters এর জন্য)
aws rds reboot-db-instance --db-instance-identifier lab3-postgres
aws rds wait db-instance-available --db-instance-identifier lab3-postgres
echo "Reboot complete!"

# Verify
psql -h 127.0.0.1 -p 5450 -U postgres -d lab3db << 'SQL'
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
SHOW work_mem;
SELECT name, installed_version FROM pg_available_extensions
WHERE installed_version IS NOT NULL;
SQL

echo "✅ Lab 2 complete!"
```

---

## 12.3 Lab 3 — Read Replica তৈরি এবং Test করো

**Duration: ~30 minutes | Cost: Read Replica = আরেকটা t3.micro → 750h এর মধ্যে রাখো**
**⚠️ এই lab শেষে Read Replica DELETE করো**

---

### 🖥️ GUI Method

**Step 1: Read Replica তৈরি করো**
```
RDS → Databases → lab3-postgres select করো
→ Actions → Create read replica

Settings:
→ DB instance identifier: lab3-postgres-replica
→ DB instance class: db.t3.micro  (free tier)
→ Availability zone: ap-southeast-1b
  (Primary এর চেয়ে different AZ — best practice)

Storage:
→ gp3, 20GB, autoscaling enable

Connectivity:
→ VPC: lab-vpc
→ DB subnet group: lab-db-subnet-group
→ Public access: No
→ Security group: lab-db-sg

Additional:
→ Performance Insights: Enable (7 days)

→ Create read replica
⏳ ~10 minutes
```

**Step 2: Endpoints দেখো**
```
RDS → Databases:
  lab3-postgres         → Role: Instance (Primary)
  lab3-postgres-replica → Role: Read replica (shows lag)

lab3-postgres-replica click করো:
→ Endpoint copy করো
→ "Replication" tab দেখো → Replication source, lag
```

**Step 3: Replica তে SSH Tunnel করো**
```bash
# New tunnel on port 5451
ssh -i ~/.ssh/lab-key.pem \
    -L 5451:REPLICA_ENDPOINT:5432 \
    -N ec2-user@BASTION_IP &
```

**Step 4: Primary vs Replica Test করো**
```sql
-- PRIMARY এ (port 5450):
SELECT pg_is_in_recovery();  -- f (false = primary)

INSERT INTO lab3_orders (item, amount)
VALUES ('Test Item', 999.99)
RETURNING id;
-- id note করো

-- REPLICA এ (port 5451):
SELECT pg_is_in_recovery();  -- t (true = read replica)

SELECT now() - pg_last_xact_replay_timestamp() AS lag;
-- Lag দেখাবে (milliseconds to seconds)

SELECT * FROM lab3_orders ORDER BY id DESC LIMIT 3;
-- Primary তে insert করা row দেখা যাবে ✅

-- Write test
INSERT INTO lab3_orders (item, amount) VALUES ('Should fail', 1);
-- ERROR: cannot execute INSERT in a read-only transaction ✅
```

**Step 5: Replica Lag Monitoring**
```
RDS → Databases → lab3-postgres-replica
→ Monitoring tab
→ "Replica Lag" metric দেখো (seconds)

CloudWatch:
→ RDS → ReplicaLag metric graph দেখো
```

**Step 6: Replica Delete করো (cleanup)**
```
RDS → Databases → lab3-postgres-replica select করো
→ Actions → Delete
→ Create final snapshot: No
→ Acknowledge checkbox ✅
→ Type "delete me" → Delete
```

---

### 💻 CLI Method

```bash
source /tmp/lab-ids.env

# Read Replica তৈরি
aws rds create-db-instance-read-replica \
    --db-instance-identifier lab3-postgres-replica \
    --source-db-instance-identifier lab3-postgres \
    --db-instance-class db.t3.micro \
    --availability-zone ap-southeast-1b \
    --no-publicly-accessible \
    --enable-performance-insights \
    --performance-insights-retention-period 7

aws rds wait db-instance-available --db-instance-identifier lab3-postgres-replica
REPLICA_EP=$(aws rds describe-db-instances \
    --db-instance-identifier lab3-postgres-replica \
    --query "DBInstances[0].Endpoint.Address" --output text)
echo "Replica: $REPLICA_EP"

# SSH Tunnel to replica
ssh -i ~/.ssh/lab-key.pem -L 5451:${REPLICA_EP}:5432 -N -f ec2-user@$BASTION_IP

# Test
psql -h 127.0.0.1 -p 5450 -U postgres -d lab3db \
    -c "INSERT INTO lab3_orders (item, amount) VALUES ('Replica Test', 1.99) RETURNING id;"

sleep 2

psql -h 127.0.0.1 -p 5451 -U postgres -d lab3db \
    -c "SELECT id, item FROM lab3_orders ORDER BY id DESC LIMIT 3;"

# Lag check
aws cloudwatch get-metric-statistics \
    --namespace AWS/RDS \
    --metric-name ReplicaLag \
    --dimensions Name=DBInstanceIdentifier,Value=lab3-postgres-replica \
    --start-time $(date -u -d "15 minutes ago" +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 60 --statistics Average \
    --query "Datapoints[*].[Timestamp,Average]" --output table

# Cleanup
aws rds delete-db-instance \
    --db-instance-identifier lab3-postgres-replica \
    --skip-final-snapshot \
    --delete-automated-backups
echo "✅ Replica deleted"
```

---

## 12.4 Lab 4 — Snapshot, PITR, এবং Restore

**Duration: ~45 minutes | Cost: Snapshot storage free (up to DB size)**

---

### 🖥️ GUI Method

**Step 1: Test Data তৈরি করো**
```sql
-- Current time note করো
SELECT NOW();
-- e.g., 2024-03-08 14:30:00+00

-- Important data insert করো
INSERT INTO lab3_orders (item, amount, status) VALUES
    ('PITR Test Item 1', 500.00, 'important'),
    ('PITR Test Item 2', 750.00, 'important'),
    ('PITR Test Item 3', 250.00, 'important');

-- Row count note করো
SELECT COUNT(*) FROM lab3_orders;
-- e.g., 8 rows
```

**Step 2: Manual Snapshot নাও**
```
RDS → Databases → lab3-postgres select করো
→ Actions → Take snapshot

Snapshot identifier:
→ lab3-before-delete-YYYYMMDD-HHMM
→ Take snapshot

⏳ ~5 minutes → Status: Available
```

**Step 3: Disaster Simulate করো**
```sql
-- গুরুত্বপূর্ণ data delete করো (accident simulate)
DELETE FROM lab3_orders WHERE status = 'important';
SELECT COUNT(*) FROM lab3_orders;
-- 5 rows (important items gone!)
```

**Step 4: PITR দিয়ে Restore করো**
```
RDS → Databases → lab3-postgres → Actions
→ Restore to point in time

Restore time:
→ Custom date and time select করো
→ Date এবং Time: [Step 1 এর time এর 1 minute আগে]
  e.g., 2024-03-08 14:29:00 UTC

New DB instance settings:
→ DB instance identifier: lab3-pitr-restore
→ DB instance class: db.t3.micro
→ VPC: lab-vpc
→ DB subnet group: lab-db-subnet-group
→ Security group: lab-db-sg
→ Public access: No

→ Restore to point in time
⏳ ~15 minutes
```

**Step 5: Restored Data Verify করো**
```bash
# Restored instance endpoint নাও
# RDS → lab3-pitr-restore → Endpoint copy করো

# SSH Tunnel
ssh -i ~/.ssh/lab-key.pem \
    -L 5452:RESTORED_ENDPOINT:5432 \
    -N ec2-user@BASTION_IP &
```

```sql
-- Restored instance এ connect করো (port 5452)
SELECT COUNT(*) FROM lab3_orders;
-- 8 rows ফিরে এসেছে ✅

SELECT * FROM lab3_orders WHERE status = 'important';
-- Deleted items ফিরে এসেছে ✅
```

**Step 6: Snapshot থেকে Restore করো (alternative)**
```
RDS → Snapshots → Manual → lab3-before-delete-... select করো
→ Actions → Restore snapshot

→ DB instance identifier: lab3-snap-restore
→ db.t3.micro, same VPC/SG settings
→ Restore DB instance
⏳ ~10 minutes
```

**Step 7: Cleanup**
```
RDS → Databases:
→ lab3-pitr-restore → Delete (skip final snapshot)
→ lab3-snap-restore → Delete (skip final snapshot)

RDS → Snapshots → Manual:
→ lab3-before-delete-... → Delete
```

---

### 💻 CLI Method

```bash
source /tmp/lab-ids.env

# Test data এবং snapshot
psql -h 127.0.0.1 -p 5450 -U postgres -d lab3db << 'SQL'
INSERT INTO lab3_orders (item, amount, status) VALUES
    ('PITR Test 1', 500.00, 'important'),
    ('PITR Test 2', 750.00, 'important');
SELECT NOW() AS snapshot_time;
SQL

SNAP_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)
SNAP_ID="lab3-snap-$(date +%Y%m%d-%H%M)"

aws rds create-db-snapshot \
    --db-instance-identifier lab3-postgres \
    --db-snapshot-identifier $SNAP_ID
aws rds wait db-snapshot-completed --db-snapshot-identifier $SNAP_ID
echo "Snapshot ready: $SNAP_ID"

# Disaster simulate
psql -h 127.0.0.1 -p 5450 -U postgres -d lab3db \
    -c "DELETE FROM lab3_orders WHERE status = 'important';"
echo "Data deleted! Rows remaining:"
psql -h 127.0.0.1 -p 5450 -U postgres -d lab3db \
    -c "SELECT COUNT(*) FROM lab3_orders;"

# PITR restore
RESTORE_TIME=$(date -u -d "5 minutes ago" +%Y-%m-%dT%H:%M:%SZ)
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier lab3-postgres \
    --target-db-instance-identifier lab3-pitr-restore \
    --restore-time $RESTORE_TIME \
    --db-instance-class db.t3.micro \
    --db-subnet-group-name lab-db-subnet-group \
    --vpc-security-group-ids $SG_DB \
    --no-publicly-accessible \
    --no-multi-az

aws rds wait db-instance-available --db-instance-identifier lab3-pitr-restore
echo "PITR restore complete!"

PITR_EP=$(aws rds describe-db-instances \
    --db-instance-identifier lab3-pitr-restore \
    --query "DBInstances[0].Endpoint.Address" --output text)

ssh -i ~/.ssh/lab-key.pem -L 5452:${PITR_EP}:5432 -N -f ec2-user@$BASTION_IP

psql -h 127.0.0.1 -p 5452 -U postgres -d lab3db \
    -c "SELECT COUNT(*), COUNT(CASE WHEN status='important' THEN 1 END) AS important FROM lab3_orders;"

# Cleanup
aws rds delete-db-instance \
    --db-instance-identifier lab3-pitr-restore \
    --skip-final-snapshot --delete-automated-backups
aws rds delete-db-snapshot --db-snapshot-identifier $SNAP_ID
echo "✅ Lab 4 complete!"
```

---

## 12.5 Lab 5 — Performance Insights দিয়ে Slow Query খোঁজো

**Duration: ~30 minutes | Cost: Free (7-day retention)**

---

### 🖥️ GUI Method

**Step 1: Test Data এবং Slow Queries তৈরি করো**
```sql
-- Large dataset তৈরি করো
CREATE TABLE perf_test AS
SELECT
    generate_series(1, 100000) AS id,
    md5(random()::text) AS product_name,
    (random() * 1000)::NUMERIC(10,2) AS price,
    (random() * 100)::INTEGER AS quantity,
    NOW() - (random() * INTERVAL '365 days') AS created_at;

-- এটা ~10 seconds লাগবে (slow query তৈরি হবে)

-- Index ছাড়া queries (slow)
SELECT * FROM perf_test WHERE id = 50000;
SELECT COUNT(*), AVG(price) FROM perf_test WHERE price > 500;
SELECT product_name FROM perf_test WHERE product_name LIKE 'a%'
ORDER BY price DESC LIMIT 10;

-- আরো কিছু slow queries
SELECT * FROM perf_test WHERE quantity > 90 ORDER BY price DESC;
SELECT DATE(created_at), COUNT(*), SUM(price)
FROM perf_test GROUP BY DATE(created_at) ORDER BY 1;
```

**Step 2: Performance Insights খোলো**
```
RDS → Databases → lab3-postgres click করো
→ "Monitoring" tab
→ "Performance Insights" section এর নিচে
  "View in Performance Insights" button click করো

অথবা directly:
→ Left menu: Performance Insights
→ Select database: lab3-postgres
```

**Step 3: Dashboard Explore করো**
```
Dashboard এ যা দেখবে:

Top graph: DBLoad (AAS — Average Active Sessions)
  → X-axis: time
  → Y-axis: concurrent active sessions
  → Color: wait event type অনুযায়ী

Wait events breakdown:
  → io:DataFileRead (disk I/O)
  → CPU (query processing)
  → Lock (lock waits)
  → Client:ClientRead (network)

আমাদের slow queries এর সময়:
  DBLoad বাড়বে
  io:DataFileRead বেশি দেখাবে (index নেই, disk scan)
```

**Step 4: Top SQL দেখো**
```
Dashboard এর নিচে "Top SQL" tab:

Columns:
  SQL: query text (truncated)
  DB Load: database load (higher = worse)
  Executions/sec: frequency
  Avg Latency (ms): per execution time

Click on any slow query:
  → SQL tab: full query text
  → Statistics tab: call count, avg time, total time
  → Wait states: কোন wait এ সময় যাচ্ছে

সবচেয়ে slow query identify করো
```

**Step 5: Index দিয়ে Fix করো**
```sql
-- Index তৈরি করো
CREATE INDEX idx_perf_id ON perf_test(id);
CREATE INDEX idx_perf_price ON perf_test(price);
CREATE INDEX idx_perf_created ON perf_test(created_at);

-- Same queries আবার চালাও
SELECT * FROM perf_test WHERE id = 50000;          -- এখন fast
SELECT COUNT(*) FROM perf_test WHERE price > 500;  -- এখন fast
```

**Step 6: Before/After Compare করো**
```
Performance Insights:
→ Time range: last 1 hour
→ Index তৈরির আগে vs পরে DBLoad compare করো
→ Top SQL এ same queries এর avg latency কমেছে দেখবে
→ Wait events এ io:DataFileRead কমবে, CPU বাড়বে (ভালো!)
```

---

### 💻 CLI Method

```bash
# Slow queries generate করো
psql -h 127.0.0.1 -p 5450 -U postgres -d lab3db << 'SQL'
CREATE TABLE IF NOT EXISTS perf_test AS
SELECT generate_series(1,100000) AS id,
    md5(random()::text) AS product_name,
    (random()*1000)::NUMERIC(10,2) AS price,
    (random()*100)::INTEGER AS quantity,
    NOW() - (random()*INTERVAL '365 days') AS created_at;

-- Slow queries
SELECT * FROM perf_test WHERE id = 50000;
SELECT COUNT(*), AVG(price) FROM perf_test WHERE price > 500;
SELECT product_name FROM perf_test
WHERE product_name LIKE 'a%' ORDER BY price DESC LIMIT 10;
SQL

echo "Slow queries executed — check Performance Insights in console"

# pg_stat_statements দিয়ে দেখো (CLI alternative)
psql -h 127.0.0.1 -p 5450 -U postgres -d lab3db << 'SQL'
SELECT LEFT(query, 80) AS query,
    calls,
    ROUND(mean_exec_time::numeric, 0)||'ms' AS avg_ms,
    ROUND(total_exec_time::numeric, 0)||'ms' AS total_ms
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat%'
ORDER BY total_exec_time DESC LIMIT 5;
SQL

# Index তৈরি করো
psql -h 127.0.0.1 -p 5450 -U postgres -d lab3db << 'SQL'
CREATE INDEX idx_perf_id    ON perf_test(id);
CREATE INDEX idx_perf_price ON perf_test(price);

-- After index: performance check
EXPLAIN ANALYZE SELECT * FROM perf_test WHERE id = 50000;
-- Index Scan দেখাবে (Seq Scan থেকে fast)
SQL

# Performance Insights metrics (CLI)
aws pi get-resource-metrics \
    --service-type RDS \
    --identifier db-$(aws rds describe-db-instances \
        --db-instance-identifier lab3-postgres \
        --query "DBInstances[0].DbiResourceId" --output text) \
    --metric-queries "[{\"Metric\": \"db.load.avg\"}]" \
    --start-time $(date -u -d "30 minutes ago" +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period-in-seconds 60 \
    --query "MetricList[0].DataPoints[*].[Timestamp,Value]" \
    --output table 2>/dev/null || echo "Check Performance Insights in Console"

echo "✅ Lab 5 complete!"
```

---

## 12.6 Lab 6 — CloudWatch Alarms এবং Dashboard

**Duration: ~30 minutes | Cost: 10 alarms free**

---

### 🖥️ GUI Method

**Step 1: SNS Topic তৈরি করো**
```
Services → SNS → Topics → Create topic
→ Type: Standard
→ Name: lab3-db-alerts
→ Create topic

Subscription:
→ Create subscription
→ Protocol: Email
→ Endpoint: তোমার email
→ Create subscription
→ Email confirm করো!
```

**Step 2: CPU Alarm**
```
CloudWatch → Alarms → Create alarm
→ Select metric → RDS → Per-Database Metrics
→ DBInstanceIdentifier = lab3-postgres, CPUUtilization
→ Select metric

Conditions:
→ Threshold: > 70
→ Period: 5 minutes
→ Evaluation periods: 1

Notification:
→ In alarm: lab3-db-alerts SNS topic

Alarm name: Lab3-CPU-High
→ Create alarm
```

**Step 3: Connections Alarm**
```
Create alarm → RDS → DatabaseConnections
→ Threshold: > 10  (t3.micro এ 20 max)
→ Period: 1 minute
→ Alarm name: Lab3-Connections-High
```

**Step 4: Storage Alarm**
```
Create alarm → RDS → FreeStorageSpace
→ Threshold: < 5368709120  (5GB in bytes)
→ Alarm name: Lab3-Storage-Low
```

**Step 5: Dashboard তৈরি করো**
```
CloudWatch → Dashboards → Create dashboard
→ Name: Lab3-RDS-Monitoring

Add widget → Line:
→ RDS → CPUUtilization (lab3-postgres)
→ RDS → DatabaseConnections (lab3-postgres)
→ Create widget

Add another widget → Line:
→ RDS → FreeStorageSpace (lab3-postgres)
→ RDS → ReadLatency, WriteLatency

Save dashboard

Dashboard এ দেখো:
→ Graphs update হচ্ছে
→ Time range change করো: 1h, 3h, 12h
```

**Step 6: Alarm Test করো**
```sql
-- High CPU generate করো
-- এটা CPU spike করবে
DO $$ DECLARE i INTEGER := 0;
BEGIN
  WHILE i < 50000 LOOP
    PERFORM COUNT(*) FROM perf_test WHERE id > i;
    i := i + 100;
  END LOOP;
END $$;
-- এখন CloudWatch → CPU alarm "In alarm" হবে
-- Email notification আসবে
```

---

### 💻 CLI Method

```bash
source /tmp/lab-ids.env

SNS_ARN=$(aws sns create-topic --name lab3-db-alerts --query "TopicArn" --output text)
aws sns subscribe --topic-arn $SNS_ARN --protocol email \
    --notification-endpoint your-email@gmail.com

# 3 alarms
for alarm_config in \
    "Lab3-CPU-High:CPUUtilization:70:GreaterThanThreshold:300:1" \
    "Lab3-Connections-High:DatabaseConnections:10:GreaterThanThreshold:60:2" \
    "Lab3-Storage-Low:FreeStorageSpace:5368709120:LessThanThreshold:300:1"; do
    NAME=$(echo $alarm_config | cut -d: -f1)
    METRIC=$(echo $alarm_config | cut -d: -f2)
    THRESHOLD=$(echo $alarm_config | cut -d: -f3)
    OPERATOR=$(echo $alarm_config | cut -d: -f4)
    PERIOD=$(echo $alarm_config | cut -d: -f5)
    EVALS=$(echo $alarm_config | cut -d: -f6)

    aws cloudwatch put-metric-alarm \
        --alarm-name $NAME \
        --namespace "AWS/RDS" \
        --metric-name $METRIC \
        --dimensions Name=DBInstanceIdentifier,Value=lab3-postgres \
        --statistic Average \
        --period $PERIOD \
        --evaluation-periods $EVALS \
        --threshold $THRESHOLD \
        --comparison-operator $OPERATOR \
        --alarm-actions $SNS_ARN
    echo "Created: $NAME"
done

# Dashboard
aws cloudwatch put-dashboard \
    --dashboard-name "Lab3-RDS" \
    --dashboard-body '{
        "widgets": [
            {"type":"metric","properties":{
                "title":"CPU & Connections",
                "metrics":[
                    ["AWS/RDS","CPUUtilization","DBInstanceIdentifier","lab3-postgres"],
                    ["AWS/RDS","DatabaseConnections","DBInstanceIdentifier","lab3-postgres"]
                ],"period":60,"width":24,"height":6
            }}
        ]
    }'

aws cloudwatch describe-alarms --alarm-name-prefix "Lab3-" \
    --query "MetricAlarms[*].[AlarmName,StateValue]" --output table

echo "export SNS_LAB3_ARN=$SNS_ARN" >> /tmp/lab-ids.env
echo "✅ Lab 6 complete!"
```

---

## 12.7 Lab 7 — RDS Security Hardening

**Duration: ~20 minutes | Cost: Free**

---

### 🖥️ GUI Method

**Step 1: SSL Force করো**
```
RDS → Parameter groups → lab3-pg16
→ Edit parameters
→ Search: rds.force_ssl
→ Value: 1
→ Apply: pending-reboot
→ Save changes

RDS → lab3-postgres → Actions → Reboot
```

**Step 2: SSL দিয়ে Connect Test করো**
```bash
# SSL Certificate download করো
wget https://truststore.pki.rds.amazonaws.com/ap-southeast-1/ap-southeast-1-bundle.pem

# SSL verify করে connect
psql "host=127.0.0.1 port=5450 user=postgres dbname=lab3db \
      sslmode=verify-full sslrootcert=ap-southeast-1-bundle.pem"
```

```sql
-- SSL verify
SELECT ssl, version, cipher FROM pg_stat_ssl
WHERE pid = pg_backend_pid();
-- ssl: t ← SSL active!
```

**Step 3: IAM Database Auth Enable করো**
```
RDS → lab3-postgres → Modify

Database authentication:
→ Password and IAM database authentication ✅

→ Continue → Apply immediately
```

```sql
-- IAM user তৈরি করো
CREATE USER lab3_iam_user;
GRANT rds_iam TO lab3_iam_user;
GRANT CONNECT ON DATABASE lab3db TO lab3_iam_user;
GRANT USAGE ON SCHEMA public TO lab3_iam_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO lab3_iam_user;
```

```bash
# IAM Token generate করো
TOKEN=$(aws rds generate-db-auth-token \
    --hostname 127.0.0.1 \
    --port 5450 \
    --region ap-southeast-1 \
    --username lab3_iam_user)

# Token দিয়ে connect করো (SSL required)
PGPASSWORD=$TOKEN psql \
    "host=127.0.0.1 port=5450 user=lab3_iam_user dbname=lab3db \
     sslmode=require"
```

**Step 4: Deletion Protection Enable করো**
```
RDS → lab3-postgres → Modify

Additional settings:
→ Deletion protection: ✅ Enable

→ Modify DB instance

Test: RDS → lab3-postgres → Actions → Delete
→ Error দেখাবে: "Cannot delete DB instance with deletion protection enabled"
✅ Protection কাজ করছে

Disable করো lab cleanup এর আগে:
→ Modify → Deletion protection uncheck → Apply immediately
```

---

### 💻 CLI Method

```bash
# Force SSL
aws rds modify-db-parameter-group \
    --db-parameter-group-name lab3-pg16 \
    --parameters "ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=pending-reboot"

aws rds reboot-db-instance --db-instance-identifier lab3-postgres
aws rds wait db-instance-available --db-instance-identifier lab3-postgres

# IAM auth enable
aws rds modify-db-instance \
    --db-instance-identifier lab3-postgres \
    --enable-iam-database-authentication \
    --apply-immediately

# Deletion protection enable
aws rds modify-db-instance \
    --db-instance-identifier lab3-postgres \
    --deletion-protection \
    --apply-immediately

# Security status verify
aws rds describe-db-instances \
    --db-instance-identifier lab3-postgres \
    --query "DBInstances[0].{
        IAMAuth:IAMDatabaseAuthenticationEnabled,
        DeletionProtection:DeletionProtection,
        StorageEncrypted:StorageEncrypted,
        PubliclyAccessible:PubliclyAccessible
    }"

echo "✅ Lab 7 complete!"
```

---

## 12.8 Lab 8 — Aurora Cluster (Serverless v2)

**Duration: ~30 minutes | Cost: ~$0.06/hour minimum, STOP করো শেষে**

---

### 🖥️ GUI Method

**Step 1: Aurora Serverless v2 তৈরি করো**
```
RDS → Create database

Engine:
→ Amazon Aurora
→ Amazon Aurora PostgreSQL-Compatible Edition
→ Engine version: Aurora PostgreSQL 16.x

Templates:
→ Dev/Test  (কম cost)

Settings:
→ DB cluster identifier: lab3-aurora
→ Master username: postgres
→ Master password: AuroraLab3@2024!

Instance configuration:
→ DB instance class: Serverless
→ Minimum ACUs: 0.5
→ Maximum ACUs: 2  (lab এ low)

Connectivity:
→ VPC: lab-vpc
→ DB subnet group: lab-db-subnet-group
→ Public access: No
→ Security group: lab-db-sg

Additional:
→ Initial DB name: aurora3db
→ Backtrack: Enable, 1 hour (unique Aurora feature!)
→ Deletion protection: No
→ Performance Insights: Enable

→ Create database
⏳ ~10 minutes
```

**Step 2: Aurora Endpoints দেখো**
```
RDS → Databases → lab3-aurora cluster click করো

Endpoints tab:
  Writer: lab3-aurora.cluster-xxxx.ap-southeast-1.rds.amazonaws.com
  Reader: lab3-aurora.cluster-ro-xxxx.ap-southeast-1.rds.amazonaws.com

Writer endpoint copy করো
```

**Step 3: Connect এবং Aurora-specific Features Test করো**
```bash
# SSH Tunnel to Aurora Writer
ssh -i ~/.ssh/lab-key.pem \
    -L 5455:AURORA_WRITER_ENDPOINT:5432 \
    -N ec2-user@BASTION_IP &

psql -h 127.0.0.1 -p 5455 -U postgres -d aurora3db
Password: AuroraLab3@2024!
```

```sql
-- Aurora version দেখো
SELECT aurora_version();
-- Aurora PostgreSQL এর version দেখাবে

-- Aurora specific: replica status
SELECT * FROM aurora_replica_status();

-- Test data তৈরি করো
CREATE TABLE aurora_test (
    id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    val     TEXT,
    created TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO aurora_test (val)
SELECT 'Row-' || generate_series FROM generate_series(1, 100);

SELECT COUNT(*) FROM aurora_test;  -- 100
```

**Step 4: Backtrack Feature Test করো (Aurora Unique!)**
```sql
-- Current time note করো
SELECT NOW();
-- e.g., 2024-03-08 15:00:00+00

-- Data delete করো
DELETE FROM aurora_test WHERE id > 50;
SELECT COUNT(*) FROM aurora_test;  -- 50 rows
```

```
RDS → Databases → lab3-aurora (cluster) → Actions
→ Backtrack

Backtrack cluster:
→ "Backtrack to a specific date" select করো
→ Date/time: [1 minute আগের time]
  e.g., 2024-03-08 14:59:00 UTC

→ Backtrack cluster

⏳ ~30 seconds!
```

```sql
-- Reconnect করো (connection dropped during backtrack)
SELECT COUNT(*) FROM aurora_test;
-- 100 rows ফিরে এসেছে! ✅
-- PITR এর চেয়ে 100x faster!
```

**Step 5: Aurora Cluster STOP করো (cost save)**
```
RDS → Databases → lab3-aurora cluster select করো
→ Actions → Stop temporarily

Confirm → Stop DB cluster

⏳ Stopped state এ storage charge শুধু ($0.10/GB-month)
Instance charge নেই

7 দিন পরে AWS automatically restart করবে
Lab cleanup এ DELETE করো
```

---

### 💻 CLI Method

```bash
source /tmp/lab-ids.env

# Aurora Cluster
aws rds create-db-cluster \
    --db-cluster-identifier lab3-aurora \
    --engine aurora-postgresql \
    --engine-version "16.2" \
    --master-username postgres \
    --master-user-password "AuroraLab3@2024!" \
    --db-subnet-group-name lab-db-subnet-group \
    --vpc-security-group-ids $SG_DB \
    --database-name aurora3db \
    --backup-retention-period 1 \
    --backtrack-window 3600 \
    --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=2 \
    --no-deletion-protection

# Writer instance
aws rds create-db-instance \
    --db-instance-identifier lab3-aurora-writer \
    --db-cluster-identifier lab3-aurora \
    --db-instance-class db.serverless \
    --engine aurora-postgresql

aws rds wait db-instance-available --db-instance-identifier lab3-aurora-writer
echo "Aurora ready!"

AURORA_WRITER=$(aws rds describe-db-clusters \
    --db-cluster-identifier lab3-aurora \
    --query "DBClusters[0].Endpoint" --output text)
echo "Writer: $AURORA_WRITER"

# SSH Tunnel
ssh -i ~/.ssh/lab-key.pem -L 5455:${AURORA_WRITER}:5432 -N -f ec2-user@$BASTION_IP

# Test
psql -h 127.0.0.1 -p 5455 -U postgres -d aurora3db << 'SQL'
SELECT aurora_version();
CREATE TABLE aurora_test (id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY, val TEXT);
INSERT INTO aurora_test (val) SELECT 'Row-'||g FROM generate_series(1,100) g;
SELECT COUNT(*) FROM aurora_test;
SQL

# Backtrack test
psql -h 127.0.0.1 -p 5455 -U postgres -d aurora3db \
    -c "DELETE FROM aurora_test WHERE id > 50;"

aws rds backtrack-db-cluster \
    --db-cluster-identifier lab3-aurora \
    --backtrack-to $(date -u -d "2 minutes ago" +%Y-%m-%dT%H:%M:%SZ)

sleep 45
psql -h 127.0.0.1 -p 5455 -U postgres -d aurora3db \
    -c "SELECT COUNT(*) FROM aurora_test;"
# 100 দেখাবে!

# Stop Aurora (cost save)
aws rds stop-db-cluster --db-cluster-identifier lab3-aurora
echo "export AURORA_WRITER=$AURORA_WRITER" >> /tmp/lab-ids.env
echo "✅ Lab 8 complete! Aurora stopped to save cost."
```

---

## 12.9 Lab 9 — RDS Upgrade (Minor এবং Major)

**Duration: ~30 minutes | Cost: Free (db.t3.micro)**

---

### 🖥️ GUI Method

**Step 1: Current Version দেখো**
```sql
SELECT version();
-- PostgreSQL 16.x দেখাবে
```

**Step 2: Minor Version Upgrade (Automatic Check)**
```
RDS → lab3-postgres → Configuration tab

Engine version: PostgreSQL 16.x
Auto minor version upgrade: Yes/No দেখাবে

Minor upgrade manually trigger করো:
RDS → lab3-postgres → Modify
→ Engine version: latest 16.x select করো (newer patch)
→ Apply immediately: Yes
→ Modify DB instance
⏳ ~5-10 minutes (brief downtime)
```

**Step 3: Pending Maintenance দেখো**
```
RDS → lab3-postgres → Maintenance & backups tab
→ "Pending maintenance" section দেখো
→ System updates থাকলে schedule দেখাবে

RDS → Recommendations এও দেখো
→ Upgrade recommendations
```

**Step 4: Pre-upgrade Check করো**
```sql
-- Extensions compatibility check
SELECT name, default_version, installed_version
FROM pg_available_extensions
WHERE installed_version IS NOT NULL;

-- Deprecated features check
SELECT name, setting FROM pg_settings
WHERE name = 'standard_conforming_strings';

-- Database size (upgrade time estimate)
SELECT pg_size_pretty(pg_database_size('lab3db'));
```

**Step 5: Parameter Group compatibility দেখো**
```
RDS → Parameter groups → lab3-pg16

একটা major upgrade (16 → 17) হলে:
→ নতুন Parameter Group লাগবে (postgres17 family)
→ আগে নতুন group তৈরি করো
→ তারপর upgrade করো
```

---

### 💻 CLI Method

```bash
# Current version
aws rds describe-db-instances \
    --db-instance-identifier lab3-postgres \
    --query "DBInstances[0].[EngineVersion,AutoMinorVersionUpgrade]"

# Available upgrade versions
aws rds describe-db-engine-versions \
    --engine postgres \
    --engine-version "16" \
    --query "DBEngineVersions[*].[EngineVersion,ValidUpgradeTarget[*].EngineVersion]" \
    --output table

# Minor version upgrade
LATEST_16=$(aws rds describe-db-engine-versions \
    --engine postgres \
    --query "DBEngineVersions[?starts_with(EngineVersion, '16')].EngineVersion | sort(@) | [-1]" \
    --output text)
echo "Latest 16.x: $LATEST_16"

aws rds modify-db-instance \
    --db-instance-identifier lab3-postgres \
    --engine-version $LATEST_16 \
    --apply-immediately
aws rds wait db-instance-available --db-instance-identifier lab3-postgres
echo "Minor upgrade complete!"

# Post-upgrade verify
psql -h 127.0.0.1 -p 5450 -U postgres -d lab3db << 'SQL'
SELECT version();

-- Extension compatibility
SELECT name, default_version, installed_version,
    CASE WHEN default_version = installed_version THEN 'OK'
         ELSE 'UPDATE: ALTER EXTENSION ' || name || ' UPDATE;'
    END AS status
FROM pg_available_extensions
WHERE installed_version IS NOT NULL;
SQL

# Extension update if needed
psql -h 127.0.0.1 -p 5450 -U postgres -d lab3db \
    -c "ALTER EXTENSION pg_stat_statements UPDATE;"

echo "✅ Lab 9 complete!"
```

---

## 12.10 Lab Cleanup — সব Resources Delete করো

**⚠️ সব lab শেষে এটা চালাও। Charge এড়াতে critical!**

---

### 🖥️ GUI Method — Step by Step

**Step 1: Deletion Protection Disable করো**
```
RDS → lab3-postgres → Modify
→ Additional settings → Deletion protection: ❌ Disable
→ Apply immediately → Modify
```

**Step 2: Aurora Cluster Delete করো**
```
RDS → Databases → lab3-aurora-writer
→ Actions → Delete
→ Create final snapshot: No
→ Acknowledge ✅
→ Type "delete me" → Delete

তারপর:
RDS → Databases → lab3-aurora (cluster)
→ Actions → Delete
→ Acknowledge ✅
→ Delete

⏳ ~5 minutes
```

**Step 3: RDS Instance Delete করো**
```
RDS → Databases → lab3-postgres
→ Actions → Delete
→ Create final snapshot: No
→ "I acknowledge..." ✅
→ "delete me" type করো
→ Delete button
⏳ ~5 minutes
```

**Step 4: Snapshots Delete করো**
```
RDS → Snapshots → Manual
→ সব lab3- prefix snapshots select করো
→ Actions → Delete snapshot
→ Delete

RDS → Snapshots → Automated
→ lab3-postgres এর automated backups automatically delete হবে
```

**Step 5: Parameter Groups Delete করো**
```
RDS → Parameter groups
→ lab3-pg16 select করো
→ Actions → Delete
→ Delete button
```

**Step 6: CloudWatch Cleanup করো**
```
CloudWatch → Alarms → All alarms
→ Lab3-CPU-High, Lab3-Connections-High, Lab3-Storage-Low select করো
→ Actions → Delete
→ Delete

CloudWatch → Dashboards
→ Lab3-RDS → Delete dashboard
```

**Step 7: SNS Topic Delete করো**
```
SNS → Topics
→ lab3-db-alerts select করো
→ Delete → "delete me" type করো
```

**Step 8: EC2/VPC Cleanup (Guide 2 এর resources)**
```
যদি Guide 2 এর resources এখনো চলছে:
EC2 → lab-bastion → Instance state → Terminate

VPC resources (Guide 2 Lab 12 এর cleanup script চালাও)
```

**Step 9: Cost Verify করো**
```
Billing → Free Tier
→ RDS Hours: কতটুকু ব্যবহার হয়েছে?
→ 750 এর মধ্যে থাকা উচিত

Billing → Bills → Current month
→ RDS charges দেখো
→ $0 বা very small হওয়া উচিত
```

---

### 💻 CLI Method — Automated Cleanup

```bash
source /tmp/lab-ids.env

echo "=== Guide 3 Lab Cleanup ==="

# 1. Deletion protection disable
aws rds modify-db-instance \
    --db-instance-identifier lab3-postgres \
    --no-deletion-protection \
    --apply-immediately 2>/dev/null
sleep 10

# 2. Aurora Cluster delete
aws rds delete-db-instance \
    --db-instance-identifier lab3-aurora-writer \
    --skip-final-snapshot 2>/dev/null && echo "Aurora writer deleting..."

sleep 30

aws rds delete-db-cluster \
    --db-cluster-identifier lab3-aurora \
    --skip-final-snapshot 2>/dev/null && echo "Aurora cluster deleting..."

# 3. RDS delete
aws rds delete-db-instance \
    --db-instance-identifier lab3-postgres \
    --skip-final-snapshot \
    --delete-automated-backups 2>/dev/null && echo "RDS deleting..."

echo "Waiting for deletions (~5 min)..."
sleep 60

# 4. Snapshots
aws rds describe-db-snapshots \
    --query "DBSnapshots[?contains(DBInstanceIdentifier,'lab3')].[DBSnapshotIdentifier]" \
    --output text | tr '\t' '\n' | while read snap; do
    [ -n "$snap" ] && aws rds delete-db-snapshot \
        --db-snapshot-identifier "$snap" 2>/dev/null && echo "Snapshot deleted: $snap"
done

# 5. Parameter groups
aws rds delete-db-parameter-group \
    --db-parameter-group-name lab3-pg16 2>/dev/null && echo "Parameter group deleted"

# 6. CloudWatch
aws cloudwatch delete-alarms \
    --alarm-names "Lab3-CPU-High" "Lab3-Connections-High" "Lab3-Storage-Low" 2>/dev/null
aws cloudwatch delete-dashboards --dashboard-names "Lab3-RDS" 2>/dev/null
echo "CloudWatch cleaned"

# 7. SNS
[ -n "$SNS_LAB3_ARN" ] && aws sns delete-topic \
    --topic-arn $SNS_LAB3_ARN 2>/dev/null && echo "SNS deleted"

# 8. Guide 2 cleanup (Bastion, VPC, etc.)
echo ""
echo "Guide 2 resources (VPC, Bastion, etc.) এখনো আছে।"
echo "Guide 2 এর cleanup script চালাও:"
echo "source /tmp/lab-ids.env && run guide2 cleanup"

echo ""
echo "=== ✅ Guide 3 Cleanup Complete! ==="
echo "Check: https://console.aws.amazon.com/billing/home#/freetier"
```

---

## Lab Sequence — কোন Order এ করবো

```
Day 1 (~2 hours):
  ✅ Lab 1: RDS তৈরি + Connect (45 min)
  ✅ Lab 2: Parameter Group Tune (20 min)
  ✅ Lab 6: CloudWatch Alarms + Dashboard (30 min)

Day 2 (~2 hours):
  ✅ Lab 5: Performance Insights (30 min)
  ✅ Lab 4: Snapshot + PITR (45 min)
  ✅ Lab 7: Security Hardening (20 min)

Day 3 (~2 hours):
  ✅ Lab 3: Read Replica (30 min — delete শেষে)
  ✅ Lab 8: Aurora Serverless v2 (30 min — stop শেষে)
  ✅ Lab 9: RDS Upgrade (30 min)

Day 4:
  ✅ Lab 10: Complete Cleanup
  ✅ Billing verify করো

প্রতিটা Lab এ:
  GUI দিয়ে করো → Console এ visualize করো
  তারপর CLI দিয়ে করো → same result দেখো
  শেষে: Console এ CLI এর কাজ verify করো
```


---

# 13. Blue/Green Deployments — Zero-Downtime Upgrade

## 13.1 Blue/Green কী এবং কেন

```
Traditional RDS Upgrade:
  DB stop → upgrade → DB start
  Downtime: 5-60 minutes

Blue/Green Deployment:
  Blue = current production (চলছে)
  Green = new version copy (parallel)
  
  Steps:
  ① Green environment তৈরি হয় (clone of Blue)
  ② Green এ upgrade/change apply হয়
  ③ Green এ test করো
  ④ Switchover: Blue → Green (seconds!)
  ⑤ Blue কিছুদিন রাখো (rollback এর জন্য)
  ⑥ Blue delete করো

Benefits:
  ✅ Near-zero downtime (~seconds switchover)
  ✅ Easy rollback (Blue intact থাকে)
  ✅ Test before production cutover
  ✅ Major version upgrade safe করে

Available for:
  RDS PostgreSQL 13.4+
  RDS MySQL 5.7, 8.0
  Aurora MySQL (not Aurora PostgreSQL yet)
```

---

## 13.2 Blue/Green Setup এবং Switchover

### 🖥️ GUI Method

**Step 1: Blue/Green Deployment তৈরি করো**
```
RDS → Databases → তোমার Production DB select করো
→ Actions → "Create Blue/Green Deployment"

Blue/Green deployment identifier:
→ mydb-bg-deployment

Green environment settings:
→ DB engine version: PostgreSQL 16.x (upgrade target)
  অথবা same version (schema change, parameter change)
→ DB instance class: same বা different
→ Parameter group: নতুন তৈরি করা group (if changed)

→ "Create Blue/Green Deployment"
⏳ ~15-30 minutes (Green তৈরি হতে)

Status দেখো:
RDS → Blue/Green deployments
→ Replication status: in-sync হলে ready
→ Green এর lag দেখো (0 হওয়া উচিত)
```

**Step 2: Green এ Test করো**
```
Blue/Green deployments → তোমার deployment
→ Green instance endpoint copy করো

Green এ connect করো (Bastion দিয়ে):
```

```sql
-- Green এ version verify করো
SELECT version();  -- নতুন version দেখাবে

-- Data intact আছে কিনা
SELECT COUNT(*) FROM orders;  -- Same as Blue

-- Application test করো Green endpoint দিয়ে
```

**Step 3: Switchover করো**
```
RDS → Blue/Green deployments → deployment select করো
→ "Switch over" button

Switch over settings:
→ Timeout: 300 seconds (5 minutes max downtime allowed)
  (Actual switchover ~30 seconds)

→ "Switch over" click করো

⚠️ এই মুহূর্তে:
  Existing connections gracefully closed
  New connections Green এ redirect হয়
  DNS update হয় (same endpoint name, new IP)
  
~30 seconds পরে:
  Green = Production
  Blue = Old (read-only, available for rollback)
```

**Step 4: Verify এবং Cleanup**
```sql
-- Production এ নতুন version confirm
SELECT version();  -- New version ✅
```

```
কয়েকদিন পরে Blue delete করো:
RDS → Blue/Green deployments → Delete deployment
→ Old Blue instance automatically delete হবে
```

---

### 💻 CLI Method

```bash
# Blue/Green তৈরি
aws rds create-blue-green-deployment \
    --blue-green-deployment-name mydb-bg \
    --source arn:aws:rds:ap-southeast-1:ACCOUNT:db:mydb-prod \
    --target-engine-version "16.3" \
    --target-db-parameter-group-name prod-pg16-new

# Status monitor করো
aws rds describe-blue-green-deployments \
    --filters Name=blue-green-deployment-name,Values=mydb-bg \
    --query "BlueGreenDeployments[0].[Status,SwitchoverDetails]"

# Lag check (switchover এর আগে 0 হওয়া উচিত)
aws rds describe-blue-green-deployments \
    --filters Name=blue-green-deployment-name,Values=mydb-bg \
    --query "BlueGreenDeployments[0].Tasks[*].[Name,Status]" \
    --output table

# Switchover করো
BG_ARN=$(aws rds describe-blue-green-deployments \
    --filters Name=blue-green-deployment-name,Values=mydb-bg \
    --query "BlueGreenDeployments[0].BlueGreenDeploymentIdentifier" \
    --output text)

aws rds switchover-blue-green-deployment \
    --blue-green-deployment-identifier $BG_ARN \
    --switchover-timeout 300

# Switchover complete হলে status দেখো
aws rds describe-blue-green-deployments \
    --filters Name=blue-green-deployment-name,Values=mydb-bg \
    --query "BlueGreenDeployments[0].Status"
# "SWITCHOVER_COMPLETED" হলে done

# Cleanup (Blue delete)
aws rds delete-blue-green-deployment \
    --blue-green-deployment-identifier $BG_ARN \
    --delete-target
```

---

# 14. RDS Event Subscriptions — Automatic Notifications

## 14.1 Event Subscriptions কী

```
RDS automatically generates events:
  DB instance: started, stopped, failover, backup, restore
  DB snapshot: created, deleted, restored
  DB parameter group: modified
  DB security group: modified

Event Subscriptions:
  এই events এ SNS notification পাঠাও
  → Email, SMS, Lambda, SQS

DBA এর জন্য critical events:
  failover:        Primary failed, standby promoted
  backup:          Backup started/completed/failed
  recovery:        DB recovered from failure
  low-storage:     Storage almost full
  maintenance:     Maintenance started/completed
  configuration-change: Parameter change applied
```

---

## 14.2 Event Subscriptions Setup

### 🖥️ GUI Method

```
RDS → Event subscriptions → Create event subscription

Subscription name: prod-db-events

Target:
→ Send notifications to: "New SNS topic"
   বা existing topic select করো
→ Topic name: rds-events (নতুন হলে)

Source:
→ Source type: Instances
→ Instances to include: All instances
   বা specific: mydb-prod

Event categories:
→ Select: failover, failure, low storage, maintenance,
          recovery, restoration, backup

→ Create

⚠️ Email verify করো
```

### 💻 CLI Method

```bash
# Event subscription তৈরি করো
SNS_ARN=$(aws sns create-topic --name rds-events --query "TopicArn" --output text)
aws sns subscribe --topic-arn $SNS_ARN --protocol email \
    --notification-endpoint dba@company.com

aws rds create-event-subscription \
    --subscription-name prod-db-events \
    --sns-topic-arn $SNS_ARN \
    --source-type db-instance \
    --event-categories failover failure "low storage" maintenance recovery restoration backup \
    --source-ids mydb-prod \
    --enabled

# All event categories দেখো
aws rds describe-event-categories \
    --source-type db-instance \
    --query "EventCategoriesMapList[0].EventCategories"

# Recent events দেখো
aws rds describe-events \
    --source-identifier mydb-prod \
    --source-type db-instance \
    --duration 1440 \
    --query "Events[*].[Date,EventCategories,Message]" \
    --output table
```

```sql
-- RDS তে connect করে PostgreSQL log events দেখো
-- (CloudWatch Logs)

-- Backup started/completed
-- Failover events
-- এগুলো RDS → Events page এ দেখা যাবে
```

---

# 15. VPC Flow Logs এবং CloudTrail — Audit এবং Troubleshooting

## 15.1 VPC Flow Logs — Network Traffic Audit

```
VPC Flow Logs কী:
  VPC এর সব network traffic record করে
  কে কোথায় connect করল, port কত, allowed/rejected
  S3 বা CloudWatch Logs এ store করে

DBA এর জন্য কেন দরকার:
  ① Database connection troubleshoot করো
     "Application কি DB তে connect করছে?"
     "Connection reject হচ্ছে কিনা?"
  ② Security audit
     "কোন IP থেকে DB access হয়েছে?"
     "Unauthorized access attempt?"
  ③ Incident investigation
     "কখন কে কোন port এ connect করেছিল?"
```

### 🖥️ GUI Method

**VPC Flow Logs Enable করো:**
```
VPC → Your VPCs → তোমার VPC select করো
→ "Flow logs" tab
→ "Create flow log"

Settings:
→ Filter: All (Accept + Reject দুটোই)
→ Maximum aggregation interval: 1 minute
→ Destination: Send to CloudWatch Logs
→ Destination log group: /vpc/flowlogs/prod-vpc (নতুন তৈরি হবে)
→ IAM role: Create new role বা existing

→ "Create flow log"
```

**Flow Logs থেকে DB Connection দেখো:**
```
CloudWatch → Log groups → /vpc/flowlogs/prod-vpc
→ Log Insights

Query:
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter dstPort = 5432
| filter action = "REJECT"
| sort @timestamp desc
| limit 20

→ এটা দেখাবে কোন IP থেকে DB port এ reject হয়েছে
→ "Why can't app connect?" এর answer এখানে আছে
```

### 💻 CLI Method

```bash
# Flow Logs enable করো
aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids $VPC_ID \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name /vpc/flowlogs/lab-vpc \
    --deliver-logs-permission-arn arn:aws:iam::ACCOUNT:role/vpc-flow-logs-role

# DB connections দেখো (port 5432)
aws logs start-query \
    --log-group-name /vpc/flowlogs/lab-vpc \
    --start-time $(date -d "1 hour ago" +%s) \
    --end-time $(date +%s) \
    --query-string '
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter dstPort = 5432
| stats count(*) as connections by srcAddr, action
| sort connections desc'

# Query results
QUERY_ID=$(aws logs start-query ... --query "queryId" --output text)
sleep 10
aws logs get-query-results --query-id $QUERY_ID
```

---

## 15.2 CloudTrail — AWS API Call Audit

```
CloudTrail কী:
  সব AWS API calls record করে
  কে (IAM user/role) কখন কোন action নিয়েছে
  Console, CLI, SDK — সব record হয়

DBA এর জন্য কেন দরকার:
  ① "কে RDS instance delete করল?"
  ② "কে parameter group modify করল?"
  ③"Snapshot কখন তৈরি হয়েছে?"
  ④ Compliance audit: "কে database এ কোন change করেছে?"
  ⑤ Security: unauthorized API calls detect করো

CloudTrail Events:
  Management events: AWS resource management (create, delete, modify)
  Data events: S3 object access, Lambda, RDS (expensive to enable)
```

### 🖥️ GUI Method

**CloudTrail Enable করো:**
```
Services → CloudTrail
→ Dashboard → "Create trail"

Trail settings:
→ Trail name: prod-audit-trail
→ Storage location: S3 bucket (নতুন বা existing)
  → New bucket: cloudtrail-logs-ACCOUNT-ID
→ SSE-KMS encryption: optional

Events:
→ Management events: ✅ Read + Write
→ Data events: Skip (expensive)

→ Create trail

CloudTrail automatically সব regions এ apply হবে
```

**Recent RDS Events দেখো:**
```
CloudTrail → Event history

Filter:
→ Attribute: Event name
→ Value: CreateDBInstance, DeleteDBInstance, ModifyDBInstance,
         CreateDBSnapshot, RestoreDBInstance

→ সব RDS related actions দেখাবে
→ Each event এ click করো → Who, When, What details
```

**Who Modified My Parameter Group?**
```
CloudTrail → Event history

Filter:
→ Event name: ModifyDBParameterGroup
→ Time range: last 30 days

→ দেখাবে কে কখন parameter change করেছে
→ User identity, timestamp, parameters changed
```

### 💻 CLI Method

```bash
# Trail তৈরি করো
BUCKET="cloudtrail-$(aws sts get-caller-identity --query Account --output text)"

aws s3api create-bucket --bucket $BUCKET \
    --region ap-southeast-1 \
    --create-bucket-configuration LocationConstraint=ap-southeast-1

# Bucket policy (CloudTrail needs write access)
aws s3api put-bucket-policy --bucket $BUCKET --policy "{
    "Version": "2012-10-17",
    "Statement": [{
        "Sid": "CloudTrailWrite",
        "Effect": "Allow",
        "Principal": {"Service": "cloudtrail.amazonaws.com"},
        "Action": "s3:PutObject",
        "Resource": "arn:aws:s3:::${BUCKET}/AWSLogs/*"
    },{
        "Sid": "CloudTrailAcl",
        "Effect": "Allow",
        "Principal": {"Service": "cloudtrail.amazonaws.com"},
        "Action": "s3:GetBucketAcl",
        "Resource": "arn:aws:s3:::${BUCKET}"
    }]
}"

aws cloudtrail create-trail \
    --name prod-audit-trail \
    --s3-bucket-name $BUCKET \
    --is-multi-region-trail \
    --include-global-service-events

aws cloudtrail start-logging --name prod-audit-trail

# Recent RDS events দেখো
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventSource,AttributeValue=rds.amazonaws.com \
    --start-time $(date -d "24 hours ago" +%s) \
    --query "Events[*].[EventTime,EventName,Username,CloudTrailEvent]" \
    --output table | head -30

# Specific event: কে snapshot নিয়েছে?
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=CreateDBSnapshot \
    --start-time $(date -d "7 days ago" +%s) \
    --query "Events[*].[EventTime,Username,CloudTrailEvent]" \
    --output text

# Specific user এর actions
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=Username,AttributeValue=dba-alice \
    --start-time $(date -d "24 hours ago" +%s) \
    --query "Events[*].[EventTime,EventName,EventSource]" \
    --output table
```

---

## 15.3 VPC Endpoints — S3 Access ছাড়া NAT Gateway

```
Problem:
  RDS/EC2 (private subnet) → S3 backup → কীভাবে যাবে?
  
  Option 1 (Bad): NAT Gateway → Internet → S3
    Cost: $0.045/hour NAT + data transfer
    Security: Traffic public internet দিয়ে যায়

  Option 2 (Good): VPC Endpoint → S3
    Cost: S3 Gateway Endpoint = FREE
    Security: Traffic AWS network এর ভেতরে থাকে
    Speed: Faster (no internet latency)

VPC Endpoint Types:
  Gateway Endpoint: S3, DynamoDB — FREE
  Interface Endpoint: Other AWS services — Charge ($0.01/hour)

DBA এর জন্য:
  S3 Gateway Endpoint → pgBackRest S3 backup, RDS export
  Free এবং secure → always use this!
```

### 🖥️ GUI Method

```
VPC → Endpoints → Create endpoint

Service category:
→ AWS services

Services:
→ Search: "s3"
→ "com.amazonaws.ap-southeast-1.s3" select করো
→ Type: Gateway ← এটা FREE

VPC:
→ lab-vpc (বা prod-vpc)

Configure route tables:
→ Private subnet এর route tables select করো ✅
  (public RT তেও করতে পারো)

→ "Create endpoint"

✅ এখন private subnet থেকে S3 directly access হবে
   NAT Gateway ছাড়াই!
```

### 💻 CLI Method

```bash
# S3 Gateway Endpoint (FREE)
aws ec2 create-vpc-endpoint \
    --vpc-id $VPC_ID \
    --service-name com.amazonaws.ap-southeast-1.s3 \
    --route-table-ids $PUB_RT \
    --vpc-endpoint-type Gateway \
    --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=s3-endpoint}]"

# Endpoint created — verify
aws ec2 describe-vpc-endpoints \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query "VpcEndpoints[*].[VpcEndpointId,ServiceName,State]" \
    --output table

# Test: EC2 থেকে S3 access করো (NAT Gateway ছাড়া)
# ssh lab-pg-ec2
# aws s3 ls s3://your-bucket/  ← এটা VPC Endpoint দিয়ে যাবে

# Interface Endpoint example (Secrets Manager — বাড়তি cost)
# aws ec2 create-vpc-endpoint \
#     --vpc-id $VPC_ID \
#     --service-name com.amazonaws.ap-southeast-1.secretsmanager \
#     --vpc-endpoint-type Interface \
#     --subnet-ids $DB_1A $DB_1B \
#     --security-group-ids $SG_DB
```

---

# 16. AWS Backup — Centralized Backup Management

## 16.1 AWS Backup কী

```
AWS Backup:
  সব AWS services (RDS, EBS, DynamoDB, EFS) এর
  backup একটা জায়গা থেকে manage করো

  Centralized backup policy
  Cross-account, cross-region backup
  Compliance reporting

vs RDS Native Backup:
  RDS native: শুধু RDS
  AWS Backup: RDS + EBS + S3 + EFS + Aurora একসাথে

DBA কখন AWS Backup ব্যবহার করবে:
  Multiple DB instances → একটা policy সবার জন্য
  Cross-account backup (production → backup account)
  Compliance: backup audit report দরকার
  Retention policy: 7 days dev, 30 days staging, 365 days prod
```

---

## 16.2 Backup Plan তৈরি করো

### 🖥️ GUI Method

```
Services → AWS Backup
→ "Create backup plan"

Start options:
→ "Build a new plan"

Plan name: prod-database-backup

Backup rule:
→ Rule name: daily-backup
→ Backup vault: Default
→ Backup frequency: Daily
→ Backup window: 02:00 AM UTC (রাত ৮টা BD)
→ Retention period: 30 days
→ Copy to region: ap-south-1 (Mumbai — DR)  [optional]

→ "Create plan"

Resource assignments:
→ Assignment name: rds-instances
→ Resource type: RDS
→ Tag-based: Key=Backup, Value=yes
  (সব DB instance এ এই tag দাও)

→ "Assign resources"
```

**DB Instance এ Tag দাও:**
```
RDS → DB Instance → Tags → Add tag
Key: Backup
Value: yes

এখন AWS Backup automatically এই instance এর backup নেবে
```

### 💻 CLI Method

```bash
# Backup Plan তৈরি
aws backup create-backup-plan \
    --backup-plan '{
        "BackupPlanName": "prod-database-backup",
        "Rules": [{
            "RuleName": "daily-backup",
            "TargetBackupVaultName": "Default",
            "ScheduleExpression": "cron(0 2 * * ? *)",
            "StartWindowMinutes": 60,
            "CompletionWindowMinutes": 180,
            "Lifecycle": {"DeleteAfterDays": 30},
            "RecoveryPointTags": {"Type": "AutomatedBackup"}
        }]
    }'

# Resource assignment (tag-based)
PLAN_ID=$(aws backup list-backup-plans \
    --query "BackupPlansList[?BackupPlanName=='prod-database-backup'].BackupPlanId" \
    --output text)

aws backup create-backup-selection \
    --backup-plan-id $PLAN_ID \
    --backup-selection '{
        "SelectionName": "rds-tagged",
        "IamRoleArn": "arn:aws:iam::ACCOUNT:role/AWSBackupDefaultServiceRole",
        "ListOfTags": [{
            "ConditionType": "STRINGEQUALS",
            "ConditionKey": "Backup",
            "ConditionValue": "yes"
        }]
    }'

# RDS instance এ tag দাও
aws rds add-tags-to-resource \
    --resource-name arn:aws:rds:ap-southeast-1:ACCOUNT:db:mydb-prod \
    --tags Key=Backup,Value=yes

# Backup jobs দেখো
aws backup list-backup-jobs \
    --by-resource-type RDS \
    --query "BackupJobs[*].[BackupJobId,State,CreationDate,ResourceArn]" \
    --output table
```

---

# 17. Final Summary — Complete Guide Overview

## তোমার এখন কী আছে

```
Guide 1 — PostgreSQL DBA (14,000+ lines):
  ✅ Architecture: MVCC, WAL, Checkpoint, Crash Recovery
  ✅ Configuration: All parameters, OS tuning
  ✅ Data Types, Indexes (B-tree, GIN, GiST, BRIN, Partial)
  ✅ Transactions, Locking, Deadlock Prevention
  ✅ Query Optimization: EXPLAIN, pg_stat_statements, indexes
  ✅ Backup: pg_dump, pgBackRest, Barman, PITR
  ✅ Monitoring: pg_stat_*, PMM, CloudWatch Agent
  ✅ Security: RLS, pgAudit, SSL, pgcrypto
  ✅ HA: Streaming Replication, Patroni, pgBouncer
  ✅ Percona: pg_stat_monitor, PMM
  ✅ PL/pgSQL: Functions, Procedures, Triggers, Cursors
  ✅ Upgrades: pg_dump method, pg_upgrade, Logical Replication
  ✅ Extensions: pg_cron, pg_partman, FTS, FDW, PostGIS
  ✅ Troubleshooting: 15+ scenarios, emergency procedures
  ✅ Lock Contention: Deep dive, zero-downtime DDL
  ✅ Sequence Management: Gap, exhaustion, reset
  ✅ Advanced: Dynamic SQL, LISTEN/NOTIFY, Extended Statistics

Guide 2 — AWS Fundamentals (5,000+ lines):
  ✅ IAM: Users, Groups, Roles, Policies
  ✅ VPC: Subnets, Route Tables, IGW, NAT, Security Groups
  ✅ EC2: Instance types, AMI, Key Pairs, Bastion Host
  ✅ EBS: Volume types, Snapshots, Optimization
  ✅ S3: Buckets, Lifecycle, pgBackRest integration
  ✅ CloudWatch: Metrics, Logs, Alarms, Dashboards
  ✅ Secrets Manager + Parameter Store
  ✅ KMS: Encryption
  ✅ Route 53: DNS, Failover
  ✅ AWS CLI: Daily commands, scripting
  ✅ Cost Awareness: Pricing, optimization
  ✅ Free Tier Labs: 10 labs with GUI + CLI

Guide 3 — AWS Cloud Native DBA (4,000+ lines):
  ✅ RDS vs Aurora vs Self-managed: Decision guide
  ✅ RDS PostgreSQL: Parameter Groups, Multi-AZ, Read Replicas
  ✅ RDS Backup: Automated, Manual, PITR
  ✅ RDS Upgrade: Minor, Major, Blue/Green
  ✅ Aurora: Architecture, Serverless v2, Backtrack
  ✅ Aurora Global Database: Multi-region
  ✅ RDS Proxy: Connection pooling managed
  ✅ Performance Insights: Slow query, Wait events
  ✅ Enhanced Monitoring + CloudWatch
  ✅ RDS Security: SSL, IAM Auth, Secrets Manager
  ✅ DMS: On-premise → RDS, MySQL → Aurora
  ✅ Cost Optimization: Reserved Instances, Right-sizing
  ✅ Troubleshooting: Connection, Performance, Storage, Failover
  ✅ Production Runbook: Daily/Weekly/Monthly/Emergency
  ✅ Blue/Green Deployments: Zero-downtime upgrade
  ✅ Event Subscriptions: Automatic notifications
  ✅ VPC Flow Logs: Network audit
  ✅ CloudTrail: API audit
  ✅ VPC Endpoints: S3 without NAT Gateway
  ✅ AWS Backup: Centralized backup
  ✅ Free Tier Labs: 9 labs with GUI + CLI
```

## Learning Path Recommendation

```
Week 1-2: Guide 1 Chapters 1-4
  Architecture, Configuration, Data Types, Indexes
  Hands-on: Rocky Linux 9 এ PostgreSQL install করো
  Practice: EXPLAIN, index optimization

Week 3-4: Guide 1 Chapters 5-8
  Transactions, Query Optimization, Backup, Monitoring
  Hands-on: pgBackRest setup, slow query investigation

Week 5-6: Guide 1 Chapters 9-11
  Security, HA/Replication, Percona
  Hands-on: Patroni cluster, pgBouncer setup

Week 7-8: Guide 1 Chapters 12-16
  PL/pgSQL, Upgrades, Extensions, Troubleshooting
  Hands-on: Function writing, version upgrade practice

Week 9-10: Guide 2 All Chapters
  AWS Fundamentals
  Hands-on: Free Tier Labs 0-10

Week 11-12: Guide 3 All Chapters
  RDS, Aurora, Production
  Hands-on: Free Tier Labs 1-9
  Practice: Real RDS performance investigation

Certification path (optional):
  AWS Certified Database Specialty
  → covers RDS, Aurora, DynamoDB, Redshift, Neptune
  → Guide 2 + Guide 3 এর পরে ready হওয়া উচিত
```

*AWS Cloud Native DBA | Guide 3 — Complete*
*RDS · Aurora · RDS Proxy · Performance Insights · DMS*
*Blue/Green Deployments · Event Subscriptions · CloudTrail*
*VPC Flow Logs · VPC Endpoints · AWS Backup*
*Free Tier Hands-on Labs (GUI + CLI)*


---

# 18. RDS Custom — OS Access সহ Managed RDS

## 18.1 RDS Custom কী

```
RDS Custom = RDS + EC2 OS access combined
  AWS manages: hardware, storage, networking
  You manage: OS, database engine config, custom software

কখন দরকার:
  ① pg_repack CLI চালাতে হবে (table reorganization)
  ② Custom kernel parameters (huge pages, IO scheduler)
  ③ Custom PostgreSQL builds (specific patches)
  ④ Third-party tools install করতে হবে
  ⑤ OS-level monitoring agents

RDS Custom vs RDS vs Self-managed:
  RDS:        No OS access, fully managed
  RDS Custom: OS access + AWS managed (backup, failover)
  Self-managed EC2: Full OS access, manual everything

Cost:
  More expensive than regular RDS (~20-30% more)
  Instance + RDS Custom fee
```

---

## 18.2 RDS Custom Setup এবং Access

```bash
# RDS Custom instance তৈরি করো
aws rds create-db-instance \
    --db-instance-identifier mydb-custom \
    --db-instance-class db.r7g.xlarge \
    --engine custom-postgres \
    --engine-version "16.3" \
    --master-username postgres \
    --master-user-password "CustomPass@2024!" \
    --allocated-storage 100 \
    --storage-type gp3 \
    --db-subnet-group-name prod-db-subnet-group \
    --vpc-security-group-ids sg-xxxxxxxxx \
    --custom-iam-instance-profile AmazonRDSCustomInstanceProfileForRdsCustomInstance

# OS এ SSH করো (EC2 instance এর মতো)
aws rds describe-db-instances \
    --db-instance-identifier mydb-custom \
    --query "DBInstances[0].Endpoint.Address"

# SSM Session Manager দিয়ে (preferred)
aws ssm start-session \
    --target $(aws rds describe-db-instances \
        --db-instance-identifier mydb-custom \
        --query "DBInstances[0].DbiResourceId" --output text)
```

```bash
# OS এ যা করতে পারবে:
# pg_repack install এবং চালাও
sudo dnf install -y pg_repack_16
pg_repack -h localhost -U postgres -d mydb -t large_table

# Custom postgresql.conf settings
sudo nano /var/lib/postgresql/data/postgresql.conf
# huge_pages = on
# shared_buffers = 8GB

# Custom monitoring agent install
sudo rpm -i datadog-agent.rpm

# Kernel parameters
sudo sysctl -w vm.nr_hugepages=2048

# সব করার পরে RDS এর backup/HA এখনো কাজ করে!
```

```
RDS Custom vs Self-managed comparison:
  Feature              RDS Custom    Self-managed
  OS Access            ✅            ✅
  Automated backup     ✅            ❌ (manual)
  Multi-AZ failover    ✅            ❌ (Patroni setup)
  AWS manages HW       ✅            ✅
  Custom kernel        ✅            ✅
  Cost                 $$$           $$
  Ops burden           Medium        High
```

---

# 19. Trusted Language Extensions (TLE) — Custom Code on RDS

## 19.1 TLE কী

```
TLE (Trusted Language Extensions):
  RDS/Aurora তে C extensions ছাড়া custom languages এ functions লেখো
  Supported: JavaScript, PL/Rust, PL/Perl, PL/Python
  "Trusted" = safe sandbox (OS access নেই)

vs Regular Extensions:
  Regular: C code → unsafe → RDS তে নেই
  TLE: JavaScript/Rust → sandboxed → RDS তে allowed

Use cases:
  ① Complex business logic
  ② External API calls (pg_tle_http)
  ③ Custom data validation
  ④ JSON transformation
```

---

## 19.2 TLE Setup এবং Use

```sql
-- TLE extension enable করো
CREATE EXTENSION pg_tle;

-- ─── JavaScript Functions ───
SELECT pgtle.install_extension(
    'my_utils',     -- extension name
    '1.0',          -- version
    'My utility functions',
    $js$
    -- JavaScript code
    exports.double_it = function(n) {
        return n * 2;
    };

    exports.format_phone = function(phone) {
        // Clean and format phone number
        var clean = phone.replace(/[^0-9]/g, '');
        if (clean.length === 11) {
            return '+' + clean.slice(0,2) + '-' + clean.slice(2,7) + '-' + clean.slice(7);
        }
        return phone;
    };
    $js$,
    'plv8'  -- language: plv8 (JavaScript)
);

CREATE EXTENSION my_utils;

-- Use করো
SELECT my_utils.double_it(21);        -- 42
SELECT my_utils.format_phone('01712345678');  -- +01-71234-5678

-- ─── PL/Rust Functions (fast, type-safe) ───
CREATE EXTENSION plrust;

CREATE OR REPLACE FUNCTION fibonacci(n INTEGER)
RETURNS INTEGER
LANGUAGE plrust AS $$
    let mut a: i32 = 0;
    let mut b: i32 = 1;
    for _ in 0..n {
        let temp = a + b;
        a = b;
        b = temp;
    }
    Ok(Some(a))
$$;

SELECT fibonacci(10);  -- 55

-- ─── Available TLE Languages ───
-- plv8:     JavaScript V8 engine
-- plrust:   Rust (fastest, type-safe)
-- plpython: Python 3 (most libraries)
-- plperl:   Perl
```

---

# 20. Zero-ETL — Aurora to Redshift Direct Integration

## 20.1 Zero-ETL কী

```
Traditional Analytics Pipeline:
  Aurora (OLTP) → ETL job → S3 → Redshift (OLAP)
  Latency: hours
  Complexity: ETL pipeline maintain করতে হয়
  Cost: ETL compute, S3 storage

Zero-ETL:
  Aurora → Redshift (direct, near real-time)
  No ETL pipeline
  Latency: seconds to minutes
  AWS manages everything

Use cases:
  ① Real-time dashboards (business metrics)
  ② Operational analytics (orders, inventory)
  ③ Machine learning (fresh training data)
  ④ Compliance reporting (near real-time)
```

---

## 20.2 Zero-ETL Setup

```bash
# ─── Prerequisites ───
# Aurora PostgreSQL cluster (source)
# Amazon Redshift Serverless বা provisioned (target)
# Same AWS account (বা cross-account with RAM)

# ─── Step 1: Aurora cluster enable করো ───
aws rds modify-db-cluster \
    --db-cluster-identifier mydb-aurora \
    --enable-http-endpoint  # Aurora Serverless v2 এর জন্য
    # Aurora Provisioned তে আলাদা step নেই

# ─── Step 2: Redshift তে integration authorize করো ───
# Redshift Console:
# Integrations → Create integration
# → Source: Aurora cluster
# → Target: Redshift Serverless/Cluster
# → Integration name: aurora-to-redshift

# ─── Step 3: Redshift তে database তৈরি হবে ───
# Redshift Query Editor:
# CREATE DATABASE aurora_data FROM INTEGRATION 'integration-id';

# ─── Step 4: Query করো ───
# Redshift তে (near real-time Aurora data):
# SELECT * FROM aurora_data.public.orders
# WHERE created_at > GETDATE() - INTERVAL '1 hour';
```

```sql
-- Aurora তে INSERT করো
INSERT INTO orders (user_id, total, status)
VALUES (1, 500.00, 'completed');

-- Seconds পরে Redshift এ দেখাবে
-- SELECT * FROM aurora_data.public.orders ORDER BY created_at DESC LIMIT 1;
```

```
Zero-ETL limitations:
  ✅ DML (INSERT, UPDATE, DELETE) replicate হয়
  ❌ DDL (CREATE TABLE, ALTER TABLE) partial support
  ❌ Large objects (BLOB, BYTEA)
  ❌ Some data types (JSON, ARRAY partially)
  → Schema changes carefully manage করতে হবে

Cost:
  Aurora: normal charge
  Redshift: normal charge
  Zero-ETL: no extra charge (2024 এ free)
```

---

# 21. Aurora PostgreSQL Limitless — Horizontal Sharding

## 21.1 Aurora Limitless কী

```
Traditional Aurora:
  Max: ~128TB storage, 1 writer
  For very large workloads: sharding manually

Aurora Limitless Database:
  Horizontal sharding built-in (AWS manages shards)
  Millions of writes/second
  Petabytes of data
  Single endpoint (application এ কোনো change নেই!)
  GA: 2024

Architecture:
  Application → Limitless Router
                    ↓
  Shard 1 | Shard 2 | Shard 3 | ... | Shard N
  (All appear as one database)

কখন দরকার:
  ✅ > 128TB data
  ✅ > 200K writes/second
  ✅ Single Aurora কে outgrow করেছো

কখন না:
  Regular Aurora enough হলে → Limitless ব্যবহার করো না
  Complex transactions (cross-shard transactions slow)
  Limitless expensive এবং complex
```

---

## 21.2 Limitless Setup Concept

```bash
# Aurora Limitless cluster তৈরি
aws rds create-db-cluster \
    --db-cluster-identifier mydb-limitless \
    --engine aurora-postgresql \
    --engine-version "16.4" \
    --db-cluster-mode limitless \  # ← Limitless mode
    --master-username postgres \
    --master-user-password "LimitlessPass@2024!" \
    --db-subnet-group-name prod-db-subnet-group \
    --vpc-security-group-ids sg-xxxxxxxxx
```

```sql
-- Sharding policy define করো
-- Table কোন column এ shard করবে
CREATE TABLE orders (
    id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id     INTEGER,
    total       NUMERIC(10,2),
    created_at  TIMESTAMPTZ DEFAULT NOW()
) DISTRIBUTED BY (user_id);  -- user_id এ shard
-- Same user_id → same shard (co-location)

-- Reference table (সব shards এ copy)
CREATE TABLE products (
    id    INTEGER PRIMARY KEY,
    name  TEXT,
    price NUMERIC
) USING citus_reference_table;

-- Query করো (normal SQL, routing transparent)
SELECT o.id, p.name, o.total
FROM orders o JOIN products p ON o.product_id = p.id
WHERE o.user_id = 5;
-- Router automatically shard 3 এ যাবে (user_id=5 এখানে)
```

---

# 22. pgvector on RDS/Aurora

## 22.1 pgvector on Managed Services

```sql
-- RDS PostgreSQL 15+ এ pgvector আছে
-- Aurora PostgreSQL 15.2+ এ pgvector আছে

-- Enable করো
CREATE EXTENSION vector;

-- Version check
SELECT extversion FROM pg_extension WHERE extname = 'vector';
-- 0.7.0 বা higher দেখাবে

-- RDS/Aurora তে HNSW index supported:
CREATE TABLE embeddings (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    content   TEXT,
    embedding VECTOR(1536)
);

CREATE INDEX ON embeddings USING hnsw (embedding vector_cosine_ops);

-- Performance Insights এ vector query দেখাবে
-- Wait event: CPU বা IO হবে depending on index hit/miss
```

```bash
# pgvector কাজ করছে কিনা RDS তে verify
psql -h RDS_ENDPOINT -U postgres -d mydb << 'SQL'
CREATE EXTENSION IF NOT EXISTS vector;
SELECT '[1,2,3]'::vector <-> '[1,2,4]'::vector AS l2_distance;
-- 1.0 দেখাবে
SQL
```


---

# 23. Adjacent Services — Redshift, ElastiCache, DynamoDB

> এই chapter টা DBA হিসেবে তোমাকে সম্পূর্ণ করবে। এই তিনটা service নিজে DBA manage নাও করতে পারো, কিন্তু architecture discussion, team collaboration, এবং "কোন service কোথায় use করব" — এই decision এ contribute করতে হলে জানা দরকার।

---

## 23.1 Amazon Redshift — Data Warehouse

### Redshift কী এবং PostgreSQL থেকে কতটা আলাদা

```
PostgreSQL:              Redshift:
OLTP (transactions)      OLAP (analytics)
Row-based storage        Column-based storage
Many small queries       Few large queries
Millions of rows         Billions of rows
Indexes optimize reads   Distribution/Sort keys optimize scans
VACUUM = dead tuple      VACUUM = sort order + space reclaim
pg_stat_statements       SVL_QLOG, SVL_QUERY_SUMMARY

Redshift এর ভেতরে PostgreSQL 8.x base আছে
কিন্তু behavior অনেক আলাদা!
```

**DBA কে কখন Redshift জানতে হয়:**
```
① Analytics team: "RDS তে heavy reporting query চালাচ্ছি, DB slow হয়ে যাচ্ছে"
   → Solution: Redshift এ move করো, RDS কে OLTP এর জন্য রাখো

② Zero-ETL: Aurora → Redshift automatic (Ch 20 এ cover হয়েছে)
   → DBA কে Redshift এর table structure বুঝতে হবে

③ Architecture decision: "এই data কি PostgreSQL এ রাখব নাকি Redshift এ?"
   → Decision making এ contribute করতে হবে
```

---

### Redshift Architecture — Key Concepts

```
Redshift Cluster:
  Leader Node: Query plan তৈরি করে, client connect করে এখানে
  Compute Nodes: Actual data store এবং process করে
  Node Slices: প্রতিটা Compute Node এ multiple slices

Columnar Storage:
  PostgreSQL: একটা row এর সব columns একসাথে store
  Redshift:   একটা column এর সব values একসাথে store

  কেন faster for analytics?
  "SELECT SUM(revenue) FROM sales" →
  PostgreSQL: সব rows এর সব columns read করে, revenue নেয়
  Redshift:   শুধু revenue column read করে → 10-100x faster
```

**Distribution Style — Data কোন Node এ যাবে:**
```sql
-- EVEN: সব rows সমানভাবে distribute
CREATE TABLE logs (id BIGINT, msg TEXT)
DISTSTYLE EVEN;
-- ভালো: parallel processing
-- খারাপ: JOIN এ data shuffle দরকার হয়

-- KEY: নির্দিষ্ট column এর value অনুযায়ী distribute
CREATE TABLE orders (
    order_id BIGINT,
    user_id  INTEGER,
    total    NUMERIC
) DISTKEY(user_id);
-- user_id same → same node তে থাকে
-- users table ও user_id তে distribute করলে → co-located JOIN (fast!)

-- ALL: সব nodes এ full copy রাখো
CREATE TABLE countries (code CHAR(2), name TEXT)
DISTSTYLE ALL;
-- ছোট lookup table এর জন্য
-- JOIN এ shuffle লাগে না

-- AUTO: Redshift নিজে decide করে (recommended for beginners)
CREATE TABLE products (...) DISTSTYLE AUTO;
```

**Sort Keys — Query কে Fast করে:**
```sql
-- SORTKEY: data physically এই order এ store হয়
-- WHERE clause এ এই column থাকলে → Zone Map দিয়ে blocks skip করে

CREATE TABLE sales (
    sale_date DATE,
    region    TEXT,
    revenue   NUMERIC
)
SORTKEY(sale_date);  -- date range query fast করে

-- Compound Sort Key (multiple columns)
CREATE TABLE events (
    event_date DATE,
    user_id    INTEGER,
    event_type TEXT
)
COMPOUND SORTKEY(event_date, user_id);
-- WHERE event_date = '2024-03-08' AND user_id = 5 → fast

-- Interleaved Sort Key (equal weight to all)
-- Rarely used, more maintenance
```

---

### Redshift Maintenance — VACUUM এবং ANALYZE

```sql
-- Redshift এ VACUUM PostgreSQL থেকে আলাদা!

-- VACUUM FULL: sort order restore করো + space reclaim
-- (DELETE করলে space marked as deleted, VACUUM এ reclaim হয়)
VACUUM FULL sales;
-- Long running, table lock নেয়

-- VACUUM SORT ONLY: শুধু sort করো (faster)
VACUUM SORT ONLY sales;

-- VACUUM DELETE ONLY: শুধু deleted rows cleanup
VACUUM DELETE ONLY sales;

-- ANALYZE: statistics update (PostgreSQL এর মতোই)
ANALYZE sales;
ANALYZE sales(sale_date, revenue);  -- specific columns

-- Auto VACUUM এবং Auto ANALYZE: Redshift automatically করে
-- কিন্তু large table এ manually trigger করা ভালো

-- ─── Monitoring ───
-- Table এর vacuum status দেখো
SELECT "table", unsorted, stats_off, tbl_rows,
    skew_rows, estimated_visible_rows
FROM svv_table_info
WHERE "table" = 'sales';
-- unsorted > 10%: VACUUM SORT দরকার
-- stats_off > 10%: ANALYZE দরকার

-- Query performance দেখো
SELECT query, trim(querytxt) AS sql,
    starttime, endtime,
    datediff(seconds, starttime, endtime) AS duration_sec
FROM stl_query
WHERE userid > 1
ORDER BY starttime DESC LIMIT 20;
```

---

### Redshift vs PostgreSQL — DBA এর জন্য Key Differences

| Feature | PostgreSQL | Redshift |
|---|---|---|
| Storage | Row-based | Columnar |
| Indexes | B-Tree, GIN, etc. | নেই (Sort Keys এর পরিবর্তে) |
| VACUUM | Dead tuple remove | Sort + space reclaim |
| Transactions | Full ACID | Limited (DDL autocommit) |
| UPDATE/DELETE | Fast | Slow (columns store এ expensive) |
| JOIN | Index-assisted | Distribution key co-location |
| Extensions | pg_stat_statements, PostGIS, etc. | Limited |
| Connections | Thousands | Limited (~500) |
| Use case | OLTP | OLAP/Analytics |
| Cost | EC2/RDS pricing | Per node per hour |

```sql
-- Redshift তে কাজ করার সময় PostgreSQL habits এ সাবধান:

-- ❌ Redshift এ এগুলো avoid করো:
SELECT * FROM large_table;  -- সব columns scan → slow
UPDATE large_table SET col = val WHERE ...;  -- very slow
CREATE INDEX idx ON large_table(col);  -- indexes নেই Redshift এ!

-- ✅ Redshift এ এভাবে করো:
SELECT specific_col1, specific_col2 FROM large_table;  -- needed columns only
-- DELETE + INSERT (UPDATE এর বদলে)
-- DISTKEY/SORTKEY দিয়ে optimize
```

---

### Redshift Serverless — Cost-effective Option

```bash
# Serverless: capacity automatically manage করে
# RPU (Redshift Processing Units): 8-512 RPU
# Pay per query (not per hour like provisioned)
# Good for: variable workload, infrequent queries

aws redshift-serverless create-namespace \
    --namespace-name analytics-ns \
    --admin-username admin \
    --admin-user-password "RedshiftPass@2024!" \
    --db-name analytics

aws redshift-serverless create-workgroup \
    --workgroup-name analytics-wg \
    --namespace-name analytics-ns \
    --base-capacity 8 \
    --max-capacity 64 \
    --subnet-ids $DB_1A $DB_1B \
    --security-group-ids $SG_DB

# Endpoint নাও
aws redshift-serverless get-workgroup \
    --workgroup-name analytics-wg \
    --query "workgroup.endpoint.address"
```

---

## 23.2 Amazon ElastiCache — In-memory Caching

### ElastiCache কী এবং কেন DBA জানবে

```
ElastiCache:
  In-memory key-value store
  Redis বা Memcached managed service
  Microsecond latency (vs millisecond for RDS)

DBA এর সাথে কীভাবে relate করে:

Problem: "Application এর DB query ৫০০ms লাগছে, users অনেক"
DBA Solution 1: Query optimize করো, index দাও → ৫০ms
DBA Solution 2: Frequently accessed data cache করো
               → First request: DB থেকে নাও, cache তে রাখো
               → Subsequent requests: cache থেকে নাও → 1ms

DBA জানতে হবে:
  ① "কোন queries cache করব?" — high read, low write data
  ② "Cache invalidation" — DB update হলে cache কখন clear হবে?
  ③ "Cache hit rate" — cache কতটা effective?
  ④ "ElastiCache + RDS connection" — architecture বোঝা
```

---

### Redis vs Memcached — কোনটা কখন

```
Redis (recommended for most cases):
  ✅ Data structures: String, List, Set, Hash, Sorted Set
  ✅ Persistence: AOF/RDB (restart এ data থাকে)
  ✅ Replication: Primary + Replica
  ✅ Cluster mode: horizontal scaling
  ✅ Pub/Sub: LISTEN/NOTIFY এর মতো
  ✅ Transactions: MULTI/EXEC
  ✅ TTL per key (expiry)

Memcached:
  ✅ Simple key-value only
  ✅ Multi-threaded (more CPU cores use করে)
  ✅ Slightly faster for simple get/set
  ❌ No persistence
  ❌ No replication

Choose Redis for:
  Database query caching
  Session storage
  Rate limiting
  Leaderboards (Sorted Sets)
  Real-time features

Choose Memcached for:
  Simple object caching at very high scale
  Multi-threaded performance critical
```

---

### Cache Patterns — DBA এর জন্য

```python
# ─── Pattern 1: Cache-Aside (Lazy Loading) ───
# সবচেয়ে common pattern

import redis
import psycopg2
import json

r = redis.Redis(host='elasticache-endpoint', port=6379)
db = psycopg2.connect("host=rds-endpoint ...")

def get_user(user_id: int):
    cache_key = f"user:{user_id}"

    # Cache চেক করো
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)  # Cache hit ✅

    # Cache miss → DB থেকে নাও
    cur = db.cursor()
    cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    user = cur.fetchone()

    if user:
        # Cache তে রাখো (1 hour TTL)
        r.setex(cache_key, 3600, json.dumps(user))

    return user

# Cache invalidation (user update হলে)
def update_user(user_id: int, data: dict):
    cur = db.cursor()
    cur.execute("UPDATE users SET name=%s WHERE id=%s",
                (data['name'], user_id))
    db.commit()

    # Cache clear করো
    r.delete(f"user:{user_id}")

# ─── Pattern 2: Write-Through ───
# DB write এর সাথে সাথে cache update করো
def update_user_write_through(user_id: int, data: dict):
    cur = db.cursor()
    cur.execute("UPDATE users SET name=%s WHERE id=%s RETURNING *",
                (data['name'], user_id))
    updated_user = cur.fetchone()
    db.commit()

    # Cache ও update করো
    r.setex(f"user:{user_id}", 3600, json.dumps(updated_user))
```

---

### ElastiCache Setup

**🖥️ GUI:**
```
Services → ElastiCache → Redis OSS Caches → Create

Configuration:
→ Deployment option: Serverless (simplest) বা Design your own
→ Cluster name: prod-cache
→ Location: AWS Cloud
→ Cluster mode: Disabled (single shard, simpler)
   বা Enabled (multi-shard, horizontal scale)

Node type: cache.r7g.large (production)
           cache.t3.micro (Free tier এর কাছাকাছি — cheapest)

Replicas: 1 (Multi-AZ এর জন্য)

Connectivity:
→ VPC: prod-vpc
→ Subnet group: create new (private subnets select করো)
→ Security group: create new
   → Inbound: Port 6379 from app security group

→ Create

Endpoint:
prod-cache.xxxxx.apne1.cache.amazonaws.com:6379
```

**💻 CLI:**
```bash
# Subnet Group
aws elasticache create-cache-subnet-group \
    --cache-subnet-group-name prod-cache-subnet \
    --cache-subnet-group-description "Cache Subnet Group" \
    --subnet-ids $DB_1A $DB_1B

# Security Group
SG_CACHE=$(aws ec2 create-security-group \
    --group-name sg-cache \
    --description "ElastiCache Security Group" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress \
    --group-id $SG_CACHE \
    --protocol tcp --port 6379 \
    --source-group $SG_APP  # App server SG থেকে allow

# Redis Cluster
aws elasticache create-replication-group \
    --replication-group-id prod-cache \
    --description "Production Redis Cache" \
    --num-cache-clusters 2 \
    --cache-node-type cache.r7g.large \
    --engine redis \
    --engine-version "7.2" \
    --cache-subnet-group-name prod-cache-subnet \
    --security-group-ids $SG_CACHE \
    --automatic-failover-enabled \
    --at-rest-encryption-enabled \
    --transit-encryption-enabled

# Endpoint দেখো
aws elasticache describe-replication-groups \
    --replication-group-id prod-cache \
    --query 'ReplicationGroups[0].NodeGroups[0].PrimaryEndpoint'
```

---

### Cache Monitoring — DBA এর জন্য Key Metrics

```bash
# CloudWatch এ দেখো
aws cloudwatch get-metric-statistics \
    --namespace AWS/ElastiCache \
    --metric-name CacheHitRate \
    --dimensions Name=ReplicationGroupId,Value=prod-cache \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 300 --statistics Average \
    --query 'Datapoints[*].[Timestamp,Average]' \
    --output table
```

| Metric | Healthy | Action needed |
|---|---|---|
| `CacheHitRate` | > 80% | < 80% → cache strategy review |
| `CacheMisses` | Low | High → TTL বাড়াও বা more data cache করো |
| `DatabaseMemoryUsagePercentage` | < 80% | > 80% → node size বাড়াও |
| `EngineCPUUtilization` | < 70% | High → cluster mode enable করো |
| `CurrConnections` | < 10k | High → connection pooling review |
| `Evictions` | 0 | > 0 → memory কম, node বাড়াও |

```
Cache Hit Rate কম হলে diagnose:
  ① TTL বেশি short? → বাড়াও
  ② Cache key design ভুল? → review করো
  ③ Application cache miss handle করছে না? → fix করো
  ④ Memory eviction হচ্ছে? → node size বাড়াও

DBA + Cache team collaboration:
  DBA: "এই query ১০ লক্ষ বার run হচ্ছে daily, result same থাকে ১ hour"
  → Cache করলে DB এ ১০ লক্ষ → ২৪ query reduction!
  Cache team: key design করে
  DBA: invalidation strategy বলে দেয়
```

---

## 23.3 Amazon DynamoDB — NoSQL When PostgreSQL Isn't Enough

### DynamoDB কী এবং কখন PostgreSQL এর পরিবর্তে

```
DynamoDB:
  Serverless NoSQL key-value / document database
  Single-digit millisecond at any scale
  Fully managed (no server, no cluster to manage)
  Auto-scales seamlessly

কখন DynamoDB PostgreSQL এর চেয়ে better:
  ① Millions of requests/second — PostgreSQL এ এত connection handle করা কঠিন
  ② Flexible schema — প্রতিটা item আলাদা attributes থাকতে পারে
  ③ Global tables — multi-region active-active (PostgreSQL এ কঠিন)
  ④ Serverless apps — Lambda + DynamoDB = no connection management
  ⑤ Simple access pattern — always by primary key বা known index

কখন PostgreSQL DynamoDB এর চেয়ে better:
  ① Complex queries — JOIN, GROUP BY, window functions
  ② ACID transactions spanning many tables
  ③ Ad-hoc analytics
  ④ Relational data model
  ⑤ Team already knows SQL
```

---

### DynamoDB Core Concepts

```
Table:
  DynamoDB এর সবকিছু এক table এ থাকে
  No schema — প্রতিটা item আলাদা shape হতে পারে

Item:
  PostgreSQL এর row equivalent
  JSON-like document

Primary Key (mandatory):
  Partition Key (PK): item কোন partition এ যাবে decide করে
  Sort Key (SK): optional — same partition এ items sort করে
  Together: Composite Primary Key

Access Pattern:
  DynamoDB তে আগে decide করতে হয় "কীভাবে query করব?"
  তারপর table design করতে হয়
  PostgreSQL এর মতো flexible query এর পরে index নয়!
```

**DynamoDB Data Modeling Example:**

```
PostgreSQL way (normalized):
  users table: id, name, email
  orders table: id, user_id, total, status
  items table: id, order_id, product, quantity

DynamoDB way (denormalized, single table):
  PK              | SK                | Attributes
  USER#123        | USER#123          | name, email, created_at
  USER#123        | ORDER#2024-001    | total, status, created_at
  USER#123        | ORDER#2024-002    | total, status
  ORDER#2024-001  | ITEM#1            | product, quantity, price
  ORDER#2024-001  | ITEM#2            | product, quantity, price

Access patterns এখন:
  "Get user 123" → PK=USER#123, SK=USER#123
  "Get all orders of user 123" → PK=USER#123, SK begins_with ORDER#
  "Get all items of order 2024-001" → PK=ORDER#2024-001, SK begins_with ITEM#
```

---

### DynamoDB Operations — SQL থেকে NoSQL

```python
import boto3
from boto3.dynamodb.conditions import Key, Attr

dynamodb = boto3.resource('dynamodb', region_name='ap-southeast-1')
table = dynamodb.Table('UserOrders')

# ─── CREATE / INSERT ───
# SQL: INSERT INTO users VALUES (1, 'Alice', 'alice@example.com')
table.put_item(
    Item={
        'PK': 'USER#1',
        'SK': 'USER#1',
        'name': 'Alice',
        'email': 'alice@example.com',
        'created_at': '2024-03-08T14:30:00Z'
    }
)

# ─── READ ───
# SQL: SELECT * FROM users WHERE id = 1
response = table.get_item(
    Key={'PK': 'USER#1', 'SK': 'USER#1'}
)
user = response.get('Item')

# ─── QUERY (same partition) ───
# SQL: SELECT * FROM orders WHERE user_id = 1
response = table.query(
    KeyConditionExpression=
        Key('PK').eq('USER#1') & Key('SK').begins_with('ORDER#')
)
orders = response['Items']

# ─── UPDATE ───
# SQL: UPDATE users SET name = 'Alice New' WHERE id = 1
table.update_item(
    Key={'PK': 'USER#1', 'SK': 'USER#1'},
    UpdateExpression='SET #n = :name',
    ExpressionAttributeNames={'#n': 'name'},
    ExpressionAttributeValues={':name': 'Alice New'}
)

# ─── DELETE ───
# SQL: DELETE FROM users WHERE id = 1
table.delete_item(
    Key={'PK': 'USER#1', 'SK': 'USER#1'}
)

# ─── SCAN (avoid! expensive) ───
# SQL: SELECT * FROM users WHERE email LIKE '%@gmail.com'
# DynamoDB: সব items read করে filter → very expensive!
response = table.scan(
    FilterExpression=Attr('email').contains('@gmail.com')
)
# বরং: GSI (Global Secondary Index) তৈরি করো email এ
```

---

### DynamoDB vs PostgreSQL — Decision Matrix

```
তোমার use case কী? সেই অনুযায়ী choose করো:

Access Pattern:
  Always by known key (user_id, order_id)  → DynamoDB
  Ad-hoc queries, JOINs, analytics         → PostgreSQL

Scale:
  > 100K requests/second                   → DynamoDB
  < 100K requests/second                   → PostgreSQL fine

Data Model:
  Flexible schema, document-like           → DynamoDB
  Relational, normalized                   → PostgreSQL

Team Skill:
  SQL জানে                                → PostgreSQL
  NoSQL/document DB experience             → DynamoDB

Cost at scale:
  Very high read/write volume              → DynamoDB (pay per request)
  Moderate volume                          → RDS/Aurora (pay per hour)

Special features needed:
  ACID transactions, complex queries       → PostgreSQL
  Global tables, offline sync              → DynamoDB
  Time-series at scale                     → DynamoDB বা TimescaleDB
  Vector search                            → pgvector (PostgreSQL)
  Full-text search                         → PostgreSQL FTS বা OpenSearch
```

---

### DynamoDB Setup

**🖥️ GUI:**
```
Services → DynamoDB → Tables → Create table

Table details:
→ Table name: UserOrders
→ Partition key: PK (String)
→ Sort key: SK (String)

Table settings:
→ Default settings (recommended for start)
   Capacity mode: On-demand (pay per request — good for variable)
   বা Provisioned (fixed RCU/WCU — cheaper for predictable)

→ Create table

Table ready হলে:
→ Explore items → Create item
→ Add attributes manually (flexible schema)
```

**💻 CLI:**
```bash
# Table তৈরি করো
aws dynamodb create-table \
    --table-name UserOrders \
    --attribute-definitions \
        AttributeName=PK,AttributeType=S \
        AttributeName=SK,AttributeType=S \
    --key-schema \
        AttributeName=PK,KeyType=HASH \
        AttributeName=SK,KeyType=RANGE \
    --billing-mode PAY_PER_REQUEST \
    --tags Key=Project,Value=demo

# Table ready হওয়ার অপেক্ষা
aws dynamodb wait table-exists --table-name UserOrders

# Item insert করো
aws dynamodb put-item \
    --table-name UserOrders \
    --item '{
        "PK": {"S": "USER#1"},
        "SK": {"S": "USER#1"},
        "name": {"S": "Alice"},
        "email": {"S": "alice@example.com"}
    }'

# Query করো
aws dynamodb query \
    --table-name UserOrders \
    --key-condition-expression "PK = :pk AND begins_with(SK, :sk_prefix)" \
    --expression-attribute-values '{
        ":pk": {"S": "USER#1"},
        ":sk_prefix": {"S": "ORDER#"}
    }' \
    --query 'Items'

# Table info দেখো
aws dynamodb describe-table \
    --table-name UserOrders \
    --query 'Table.[TableName,TableStatus,ItemCount,BillingModeSummary]'

# Cleanup
aws dynamodb delete-table --table-name UserOrders
```

---

## 23.4 When to Use What — Complete Decision Guide

```
তোমার কাছে data আছে। কোথায় রাখবে?

Step 1: Data কি structured (table/row/column)?
  No → S3 (raw files, logs), বা DynamoDB (document)
  Yes → next step

Step 2: Relationships আছে (JOIN দরকার)?
  No → DynamoDB (simple access patterns)
  Yes → next step

Step 3: Query pattern কী?
  Always by primary key → DynamoDB
  Complex queries, analytics → next step

Step 4: Read/Write ratio কী?
  Mostly reads, occasional writes → ElastiCache সামনে রাখো
  Mixed reads/writes → next step

Step 5: Scale কতটা?
  < 10K req/sec, OLTP → RDS PostgreSQL বা Aurora
  > 100K req/sec → DynamoDB বা Aurora Limitless
  Analytics (large scans) → Redshift

Step 6: Access frequency?
  Hot data (every request) → ElastiCache
  Warm data (frequent) → RDS/Aurora
  Cold data (rare) → S3 + Athena

Real-world example:
  E-commerce app:
    User sessions     → ElastiCache Redis (fast, temporary)
    Orders/Products   → Aurora PostgreSQL (relational, ACID)
    Analytics         → Redshift (via Zero-ETL from Aurora)
    Product catalog   → DynamoDB (high read, simple access)
    Product images    → S3
```

---

## 23.5 Monitoring — সব Services একসাথে

```bash
# ─── CloudWatch একসাথে সব দেখো ───
cat > /usr/local/bin/aws_db_stack_check.sh << 'SCRIPT'
#!/bin/bash
REGION="ap-southeast-1"
RDS_ID="mydb-prod"
CACHE_ID="prod-cache"

echo "=== Database Stack Health Check — $(date) ==="

# RDS
echo ""
echo "[ RDS PostgreSQL ]"
STATUS=$(aws rds describe-db-instances \
    --db-instance-identifier $RDS_ID \
    --query "DBInstances[0].DBInstanceStatus" --output text 2>/dev/null)
echo "  Status: $STATUS"

CPU=$(aws cloudwatch get-metric-statistics \
    --namespace AWS/RDS \
    --metric-name CPUUtilization \
    --dimensions Name=DBInstanceIdentifier,Value=$RDS_ID \
    --start-time $(date -u -d "10 minutes ago" +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 600 --statistics Average \
    --query "Datapoints[0].Average" --output text 2>/dev/null)
echo "  CPU: ${CPU}%"

# ElastiCache
echo ""
echo "[ ElastiCache Redis ]"
HIT=$(aws cloudwatch get-metric-statistics \
    --namespace AWS/ElastiCache \
    --metric-name CacheHitRate \
    --dimensions Name=ReplicationGroupId,Value=$CACHE_ID \
    --start-time $(date -u -d "10 minutes ago" +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 600 --statistics Average \
    --query "Datapoints[0].Average" --output text 2>/dev/null)
echo "  Cache Hit Rate: ${HIT}%"
[ "$(echo "$HIT < 80" | bc 2>/dev/null)" = "1" ] && echo "  ⚠️  Cache hit rate low!"

# DynamoDB (if any tables)
echo ""
echo "[ DynamoDB ]"
aws dynamodb list-tables \
    --query "TableNames" --output text 2>/dev/null | tr '\t' '\n' | while read tbl; do
    THROTTLE=$(aws cloudwatch get-metric-statistics \
        --namespace AWS/DynamoDB \
        --metric-name ThrottledRequests \
        --dimensions Name=TableName,Value=$tbl \
        --start-time $(date -u -d "10 minutes ago" +%Y-%m-%dT%H:%M:%SZ) \
        --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
        --period 600 --statistics Sum \
        --query "Datapoints[0].Sum" --output text 2>/dev/null)
    echo "  Table: $tbl — Throttles: ${THROTTLE:-0}"
done

echo ""
echo "==================================="
SCRIPT
chmod +x /usr/local/bin/aws_db_stack_check.sh
```

---

## 23.6 Cost Comparison — Service Selection এ Cost Factor

```
Monthly cost estimate (moderate production workload):

PostgreSQL/Aurora:
  db.r7g.xlarge Multi-AZ:    ~$600/month
  1-year Reserved:           ~$360/month (40% discount)
  Storage 500GB gp3:         ~$58/month
  Total:                     ~$418/month

ElastiCache Redis:
  cache.r7g.large (2 nodes): ~$300/month
  1-year Reserved:           ~$180/month

Redshift:
  ra3.xlplus (2 nodes):      ~$700/month
  Redshift Serverless:       Pay per query ($0.375/RPU-hour)

DynamoDB:
  On-demand: $1.25/million writes + $0.25/million reads
  Provisioned: WCU/RCU purchase
  10M writes + 100M reads/month: ~$37.50/month (On-demand)
  Same with Provisioned + RI: ~$10/month

Total typical stack:
  Aurora + ElastiCache + Redshift Serverless + DynamoDB:
  ~$600-800/month (without Reserved Instances)
  ~$350-500/month (with 1-year Reserved Instances)

Cost optimization:
  ① Reserved Instances for steady workloads (RDS, ElastiCache)
  ② DynamoDB On-demand for unpredictable (no upfront cost)
  ③ Redshift Serverless for occasional analytics
  ④ ElastiCache Serverless for variable cache needs
```
